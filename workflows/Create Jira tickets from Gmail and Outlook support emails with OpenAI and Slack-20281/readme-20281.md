Create Jira tickets from Gmail and Outlook support emails with OpenAI and Slack

https://n8nworkflows.xyz/workflows/create-jira-tickets-from-gmail-and-outlook-support-emails-with-openai-and-slack-20281


# Create Jira tickets from Gmail and Outlook support emails with OpenAI and Slack

### 1. Workflow Overview

This workflow automates the intake, triage, escalation, and logging of customer support emails received via Gmail and Microsoft Outlook. It uses OpenAI to analyze and categorize incoming messages, maps high-confidence requests to Jira issues (creating new ones or updating existing threads), notifies support teams via Slack and optional Microsoft Teams, acknowledges customers, and writes an audit log entry to Google Sheets.

The workflow architecture uses a main execution pipeline combined with a recursive sub-workflow pattern. Depending on execution parameters, individual nodes act as triggers, sub-workflow routers, or action executors. The logic is divided into the following blocks:

- **1.1 Email Reception and Normalization:** Monitors Gmail and Outlook inboxes, pulls raw payloads every minute, and normalizes them into a unified object structure.
- **1.2 Configuration and Sub-Workflow Dispatcher:** Injects global workflow settings and acts as the entry point for nested sub-workflow steps via an internal router (`Route by Workflow Step`).
- **1.3 Pre-filtering and AI Classification:** Filters out internal or automated email addresses, sanitizes message bodies by removing quoted reply history, and leverages OpenAI with structured output parsing to categorize the request and assess sentiment/priority.
- **1.4 Jira Routing and Escalation:** Determines the appropriate Jira issue type and project routing based on classification, checks for existing open tickets via thread identifiers, and either updates an existing issue with a comment or creates a new ticket.
- **1.5 Team Notifications:** Constructs and dispatches alerts to designated Slack channels and optionally pushes adaptive cards to Microsoft Teams.
- **1.6 Customer Acknowledgement:** Formats and sends or drafts acknowledgement replies back to the customer via Gmail or Outlook depending on the configuration mode.
- **1.7 Review Queue, Audit Logging, and Error Handling:** Routes low-confidence or unclassified emails to a human review queue, appends structured audit records to Google Sheets, and broadcasts workflow execution failures to a Slack alert channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Normalization
- **Overview:** Monitors external inbox providers on a 1-minute schedule, retrieves raw incoming messages, and normalizes them into a standard data schema.
- **Nodes Involved:** 
  - `When Email Received in Gmail`
  - `When Email Received in Outlook`
  - `Normalize Gmail Email`
  - `Normalize Outlook Email`

- **Node Details:**
  - **When Email Received in Gmail** (`n8n-nodes-base.gmailTrigger`)
    - *Role:* Triggers every minute for new inbox messages matching exclusion filters (ignoring sent mail, promotions, social, updates, and forums).
    - *Config:* Polls every minute (`item.mode: everyMinute`), queries `-from:me -category:promotions -category:social -category:updates -category:forums`, label `INBOX`.
    - *Connections:* Input: None (Trigger). Output: `Normalize Gmail Email`.
    - *Edge Cases:* API rate limits, revoked OAuth2 token authorizations, or missing Gmail credentials.
  - **When Email Received in Outlook** (`n8n-nodes-base.microsoftOutlookTrigger`)
    - *Role:* Triggers every minute to fetch raw messages from Microsoft Graph.
    - *Config:* Polls every minute, output set to raw.
    - *Connections:* Input: None (Trigger). Output: `Normalize Outlook Email`.
    - *Edge Cases:* Microsoft Graph API throttling or expired token refresh parameters.
  - **Normalize Gmail Email** (`n8n-nodes-base.code`)
    - *Role:* Transforms raw Gmail JSON payloads into a uniform email schema (source, messageId, threadId, receivedAt, fromName, fromEmail, to, subject, body, link, hasAttachments).
    - *Config:* JavaScript execution parsing nested addresses and stripping inline HTML tags.
    - *Connections:* Input: `When Email Received in Gmail`. Output: `Set Workflow Configuration`.
    - *Edge Cases:* Malformed address fields causing undefined array indexing.
  - **Normalize Outlook Email** (`n8n-nodes-base.code`)
    - *Role:* Converts raw Microsoft Graph messages into the identical shared email schema.
    - *Config:* Extracts content types, maps conversation IDs to `threadId`, and standardizes datetime stamps.
    - *Connections:* Input: `When Email Received in Outlook`. Output: `Set Workflow Configuration`.
    - *Edge Cases:* Missing body content properties or missing sender object arrays.

---

#### 2.2 Configuration and Sub-Workflow Dispatcher
- **Overview:** Injects global operational variables into the execution context and directs sub-workflow executions based on the active step parameter.
- **Nodes Involved:**
  - `Set Workflow Configuration`
  - `Prepare Step for Classification`
  - `Route by Workflow Step`

- **Node Details:**
  - **Set Workflow Configuration** (`n8n-nodes-base.set`)
    - *Role:* Establishes default parameters including internal domains, ignored sender patterns, confidence thresholds, Jira routing keys, Slack channels, and acknowledgement modes.
    - *Config:* Sets static configuration assignments (e.g., `internalDomains`, `minConfidence`, `jiraProjectKey`, `ackMode`).
    - *Connections:* Input: `Normalize Gmail Email`, `Normalize Outlook Email`. Output: `Prepare Step for Classification`.
    - *Edge Cases:* Missing required configuration parameters if values are cleared during setup.
  - **Prepare Step for Classification** (`n8n-nodes-base.set`)
    - *Role:* Appends a `step: classify` property to the payload to initiate the sub-workflow routing branch.
    - *Config:* Assigns `step = "classify"`.
    - *Connections:* Input: `Set Workflow Configuration`. Output: `Execute Email Classification`.
  - **Route by Workflow Step** (`n8n-nodes-base.switch`)
    - *Role:* Acts as the primary sub-workflow switchboard, dispatching execution to appropriate logic chains depending on whether `step` equals `classify`, `escalate`, `acknowledge`, `review`, or `log`.
    - *Config:* Strict string matching on `{{ $json.step }}` across 5 output branches.
    - *Connections:* Input: Internal sub-workflow entry point. Outputs: `Filter and Prepare for AI`, `Build Jira Routing Decision`, `Build Acknowledgement Reply`, `Alert for Email Review`, `Generate Audit Log Entry`.

---

#### 2.3 Pre-filtering and AI Classification
- **Overview:** Evaluates sender validity, strips email reply chains, and runs structured AI classification to categorize tickets and assess confidence.
- **Nodes Involved:**
  - `Filter and Prepare for AI`
  - `Check AI Classification Needed`
  - `AI Email Classifier`
  - `OpenAI GPT-4.1 Mini`
  - `Parse Classification Result`
  - `Combine Email and Classification`
  - `Record Skipped AI Classification`

- **Node Details:**
  - **Filter and Prepare for AI** (`n8n-nodes-base.code`)
    - *Role:* Identifies internal domains or blacklisted sender patterns, strips quoted history/signatures, and truncates overly long bodies.
    - *Config:* JavaScript filtering script checking `internalDomains` and `ignoreSenderPatterns`.
    - *Connections:* Input: `Route by Workflow Step` (classify branch). Output: `Check AI Classification Needed`.
  - **Check AI Classification Needed** (`n8n-nodes-base.if`)
    - *Role:* Determines whether the email should proceed to OpenAI based on the pre-filter flag (`skipAI`).
    - *Config:* Evaluates `{{ !$json.skipAI }}`.
    - *Connections:* Input: `Filter and Prepare for AI`. True Output: `AI Email Classifier`. False Output: `Record Skipped AI Classification`.
  - **AI Email Classifier** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* LangChain model execution node taking structured parameters to analyze support emails.
    - *Config:* System prompt instructing B2B support triage analysis, categorization, priority assignment, and confidence scoring. Configured with error handling (`onError: continueRegularOutput`, `maxTries: 3`).
    - *Connections:* Input: `Check AI Classification Needed`. AI Model Input: `OpenAI GPT-4.1 Mini`. Output Parser: `Parse Classification Result`. Output: `Combine Email and Classification`.
    - *Edge Cases:* OpenAI API rate limits, payload token limits, or malformed JSON responses.
  - **OpenAI GPT-4.1 Mini** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Provides the underlying chat model infrastructure for the LangChain classifier.
    - *Config:* Model: `gpt-4.1-mini`, Temperature: `0.1`.
    - *Connections:* Output: Connected to `AI Email Classifier`.
    - *Credentials:* Requires OpenAI API credentials.
  - **Parse Classification Result** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Role:* Enforces a strict JSON schema on OpenAI's output, validating keys such as `isCustomerRequest`, `category`, `priority`, `confidence`, and `summary`.
    - *Config:* Manual JSON schema definition enforcing enumerated category and priority values.
    - *Connections:* Output: Connected to `AI Email Classifier`.
  - **Combine Email and Classification** (`n8n-nodes-base.code`)
    - *Role:* Merges original email properties with AI triage results.
    - *Config:* JavaScript mapper handling missing classifications safely.
    - *Connections:* Input: `AI Email Classifier`. Output: `Route by Classification Outcome`.
  - **Record Skipped AI Classification** (`n8n-nodes-base.code`)
    - *Role:* Generates a fallback triage record for emails skipped by the pre-filter.
    - *Config:* Assigns default category `Other` and sets `needsReview` to false.
    - *Connections:* Input: `Check AI Classification Needed` (false branch). Output: `Route by Workflow Step` (log branch via router).

---

#### 2.4 Jira Routing and Escalation
- **Overview:** Evaluates high-confidence requests, queries Jira for existing open issues matching email threads, and either adds comments or creates new tickets.
- **Nodes Involved:**
  - `Route by Classification Outcome`
  - `Prepare Step for Escalation`
  - `Execute Escalation Workflow`
  - `Build Jira Routing Decision`
  - `Search Open Jira Tickets`
  - `Check If Ticket Exists`
  - `Comment on Existing Ticket`
  - `Record Ticket Follow-Up`
  - `Post New Jira Issue`

- **Node Details:**
  - **Route by Classification Outcome** (`n8n-nodes-base.switch`)
    - *Role:* Routes classified items to Escalation, Review, or Logging branches based on confidence score and customer request flags.
    - *Config:* Condition checking whether `isCustomerRequest` is true, category is valid, and confidence meets or exceeds `minConfidence`.
    - *Connections:* Input: `Combine Email and Classification`. Outputs: `Prepare Step for Escalation`, `Prepare Step for Review`, `Prepare Step for Logging`.
  - **Prepare Step for Escalation** (`n8n-nodes-base.set`)
    - *Role:* Sets execution mode parameter to invoke the escalation sub-workflow.
    - *Config:* Assigns `step = "escalate"`.
    - *Connections:* Input: `Route by Classification Outcome`. Output: `Execute Escalation Workflow`.
  - **Execute Escalation Workflow** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Recursively triggers the current workflow ID to process the escalation branch.
    - *Config:* Mode: each, wait for sub-workflow: true.
    - *Connections:* Input: `Prepare Step for Escalation`. Output: `Prepare Step for Acknowledgement`.
  - **Build Jira Routing Decision** (`n8n-nodes-base.code`)
    - *Role:* Generates Jira field payloads, computes a stable hash-based thread label for deduplication, and formats ticket description text.
    - *Config:* JavaScript mapping script mapping categories to configured Jira issue types and priorities.
    - *Connections:* Input: `Route by Workflow Step` (escalate branch). Output: `Search Open Jira Tickets`.
  - **Search Open Jira Tickets** (`n8n-nodes-base.jira`)
    - *Role:* Searches Jira for open tickets matching the calculated thread label.
    - *Config:* Operation: `getAll`, JQL filter: `labels = "{threadLabel}" AND statusCategory != Done ORDER BY created DESC`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Build Jira Routing Decision`. Output: `Check If Ticket Exists`.
    - *Edge Cases:* Jira API timeouts or invalid JQL syntax.
  - **Check If Ticket Exists** (`n8n-nodes-base.if`)
    - *Role:* Evaluates whether an open Jira issue was found for the incoming email thread.
    - *Config:* Condition checking `dedupEnabled` and presence of issue key.
    - *Connections:* Input: `Search Open Jira Tickets`. True Output: `Comment on Existing Ticket`. False Output: `Post New Jira Issue`.
  - **Comment on Existing Ticket** (`n8n-nodes-base.jira`)
    - *Role:* Appends the incoming email content as a follow-up comment on an existing open ticket.
    - *Config:* Issue comment resource targeting `issueKey`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Check If Ticket Exists` (true branch). Output: `Record Ticket Follow-Up`.
  - **Record Ticket Follow-Up** (`n8n-nodes-base.code`)
    - *Role:* Exposes the existing ticket key downstream as if it were newly handled.
    - *Config:* Extracts ticket key and flags `isFollowUp: true`.
    - *Connections:* Input: `Comment on Existing Ticket`. Output: `Create Notification Messages`.
  - **Post New Jira Issue** (`n8n-nodes-base.httpRequest`)
    - *Role:* Sends a POST request to Jira REST API to create a new support issue.
    - *Config:* URL: `{{ jiraBaseUrl }}/rest/api/2/issue`, Method: POST, Authentication: Predefined Jira Cloud API credentials.
    - *Connections:* Input: `Check If Ticket Exists` (false branch). Output: `Create Notification Messages`.
    - *Edge Cases:* Required custom fields missing in Jira schema causing validation failures.

---

#### 2.5 Team Notifications
- **Overview:** Formats Slack and Microsoft Teams notification messages and broadcasts them to configured communication channels.
- **Nodes Involved:**
  - `Create Notification Messages`
  - `Send Slack Notification`
  - `Check Teams Webhook Availability`
  - `Send Teams Notification`
  - `Return to Main Pipeline`

- **Node Details:**
  - **Create Notification Messages** (`n8n-nodes-base.code`)
    - *Role:* Builds formatted Markdown strings for Slack and Adaptive Card structures for Microsoft Teams based on ticket creation outcomes.
    - *Config:* JavaScript script assembling notification blocks, icons, and urgency tags.
    - *Connections:* Input: `Record Ticket Follow-Up`, `Post New Jira Issue`. Output: `Send Slack Notification`.
  - **Send Slack Notification** (`n8n-nodes-base.slack`)
    - *Role:* Posts the formatted alert to the category-specific Slack channel.
    - *Config:* Selects channel by name (`route.slackChannel`), with link unfurling disabled. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Create Notification Messages`. Output: `Check Teams Webhook Availability`.
  - **Check Teams Webhook Availability** (`n8n-nodes-base.if`)
    - *Role:* Checks whether a Microsoft Teams incoming webhook URL has been provided in the configuration.
    - *Config:* Evaluates `{{ !!teamsWebhookUrl }}`.
    - *Connections:* Input: `Send Slack Notification`. True Output: `Send Teams Notification`. False Output: `Return to Main Pipeline`.
  - **Send Teams Notification** (`n8n-nodes-base.httpRequest`)
    - *Role:* Posts the adaptive card payload to Microsoft Teams via webhook.
    - *Config:* Method: POST, URL bound to `teamsWebhookUrl`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Check Teams Webhook Availability` (true branch). Output: `Return to Main Pipeline`.
  - **Return to Main Pipeline** (`n8n-nodes-base.code`)
    - *Role:* Consolidates notification results and passes execution back to the caller.
    - *Config:* JavaScript cleanup script assigning `escalationStatus` and `slackSent` flags.
    - *Connections:* Input: `Send Teams Notification`, `Check Teams Webhook Availability` (false branch). Output: `Prepare Step for Acknowledgement` (via main execution flow).

---

#### 2.6 Customer Acknowledgement
- **Overview:** Determines whether to reply to customers and routes acknowledgements via Gmail or Microsoft Outlook.
- **Nodes Involved:**
  - `Prepare Step for Acknowledgement`
  - `Execute Customer Acknowledgement`
  - `Build Acknowledgement Reply`
  - `Route by Reply Channel`
  - `Send Gmail Reply`
  - `Draft Gmail Reply`
  - `Send Outlook Reply`
  - `Conclude Acknowledgement Process`

- **Node Details:**
  - **Prepare Step for Acknowledgement** (`n8n-nodes-base.set`)
    - *Role:* Prepares execution parameters for the acknowledgement sub-workflow step.
    - *Config:* Assigns `step = "acknowledge"`.
    - *Connections:* Input: `Return to Main Pipeline` (or escalation branch). Output: `Execute Customer Acknowledgement`.
  - **Execute Customer Acknowledgement** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Triggers the acknowledgement sub-workflow execution branch.
    - *Config:* Mode: each, wait for sub-workflow: true.
    - *Connections:* Input: `Prepare Step for Acknowledgement`. Output: `Prepare Step for Logging`.
  - **Build Acknowledgement Reply** (`n8n-nodes-base.code`)
    - *Role:* Evaluates acknowledgement mode settings (`off`, `draft`, `send`) and constructs personalized reply texts and HTML bodies.
    - *Config:* JavaScript script checking mode rules and calculating expected response times based on priority.
    - *Connections:* Input: `Route by Workflow Step` (acknowledge branch). Output: `Route by Reply Channel`.
  - **Route by Reply Channel** (`n8n-nodes-base.switch`)
    - *Role:* Directs the message to Gmail send, Gmail draft, Outlook reply, or skip branches.
    - *Config:* Strict string matching on `{{ $json.ackRoute }}`.
    - *Connections:* Input: `Build Acknowledgement Reply`. Outputs: `Send Gmail Reply`, `Draft Gmail Reply`, `Send Outlook Reply`, `Conclude Acknowledgement Process` (via fallback).
  - **Send Gmail Reply** (`n8n-nodes-base.gmail`)
    - *Role:* Sends a direct reply to the customer thread via Gmail.
    - *Config:* Operation: reply, reply to sender only, referencing `messageId`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Route by Reply Channel` (Gmail send). Output: `Conclude Acknowledgement Process`.
    - *Credentials:* Requires Gmail OAuth2 credentials.
  - **Draft Gmail Reply** (`n8n-nodes-base.gmail`)
    - *Role:* Creates a draft reply in Gmail for human review instead of sending automatically.
    - *Config:* Resource: draft, referencing `threadId` and `fromEmail`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Route by Reply Channel` (Gmail draft). Output: `Conclude Acknowledgement Process`.
  - **Send Outlook Reply** (`n8n-nodes-base.microsoftOutlook`)
    - *Role:* Sends or drafts a reply via Microsoft Outlook graph integration.
    - *Config:* Operation: reply, referencing `messageId`, with draft saving conditional on `ackMode`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Route by Reply Channel` (Outlook). Output: `Conclude Acknowledgement Process`.
    - *Credentials:* Requires Microsoft Outlook OAuth2 credentials.
  - **Conclude Acknowledgement Process** (`n8n-nodes-base.code`)
    - *Role:* Normalizes acknowledgement status results before returning to the caller.
    - *Config:* JavaScript script cleaning up temporary text variables.
    - *Connections:* Input: `Send Gmail Reply`, `Draft Gmail Reply`, `Send Outlook Reply`, `Route by Reply Channel` (fallback). Output: `Prepare Step for Logging` (via main execution flow).

---

#### 2.7 Review Queue, Audit Logging, and Error Handling
- **Overview:** Manages uncertain AI classifications by sending them to a human review channel, appends execution results to Google Sheets, and reports workflow execution failures to Slack.
- **Nodes Involved:**
  - `Prepare Step for Review`
  - `Execute Review Queue Workflow`
  - `Alert for Email Review`
  - `Send Review Request to Slack`
  - `Return Review Outcome`
  - `Prepare Step for Logging`
  - `Execute Audit Log Workflow`
  - `Generate Audit Log Entry`
  - `Add Entry to Triage Log`
  - `Create Error Alert Message`
  - `Send Failure Alert to Slack`

- **Node Details:**
  - **Prepare Step for Review** (`n8n-nodes-base.set`)
    - *Role:* Sets the sub-workflow step parameter for human triage review.
    - *Config:* Assigns `step = "review"`.
    - *Connections:* Input: `Route by Classification Outcome`. Output: `Execute Review Queue Workflow`.
  - **Execute Review Queue Workflow** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Triggers the sub-workflow execution for review handling.
    - *Config:* Mode: each, wait for sub-workflow: true.
    - *Connections:* Input: `Prepare Step for Review`. Output: `Prepare Step for Logging`.
  - **Alert for Email Review** (`n8n-nodes-base.code`)
    - *Role:* Formats a Slack alert message for low-confidence or failed classifications.
    - *Config:* JavaScript script building triage review markdown.
    - *Connections:* Input: `Route by Workflow Step` (review branch). Output: `Send Review Request to Slack`.
  - **Send Review Request to Slack** (`n8n-nodes-base.slack`)
    - *Role:* Posts the review alert to the designated triage review Slack channel.
    - *Config:* Channel ID bound to `reviewSlackChannel`. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Alert for Email Review`. Output: `Return Review Outcome`.
  - **Return Review Outcome** (`n8n-nodes-base.code`)
    - *Role:* Returns review alert status to the calling pipeline.
    - *Config:* JavaScript helper mapping review status flags.
    - *Connections:* Input: `Send Review Request to Slack`. Output: `Prepare Step for Logging` (via main execution flow).
  - **Prepare Step for Logging** (`n8n-nodes-base.set`)
    - *Role:* Sets the sub-workflow execution parameter for audit logging.
    - *Config:* Assigns `step = "log"`.
    - *Connections:* Input: `Execute Customer Acknowledgement`, `Execute Review Queue Workflow`, `Record Skipped AI Classification` (routed). Output: `Execute Audit Log Workflow`.
  - **Execute Audit Log Workflow** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Invokes the logging branch of the sub-workflow.
    - *Config:* Mode: each, wait for sub-workflow: true.
    - *Connections:* Input: `Prepare Step for Logging`. Output: None (Terminal branch).
  - **Generate Audit Log Entry** (`n8n-nodes-base.code`)
    - *Role:* Flattens all collected email, classification, Jira, and acknowledgement properties into a single spreadsheet row object.
    - *Config:* JavaScript mapping script formatting audit headers.
    - *Connections:* Input: `Route by Workflow Step` (log branch). Output: `Add Entry to Triage Log`.
  - **Add Entry to Triage Log** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends the audit row to the configured Google Sheets spreadsheet.
    - *Config:* Operation: append, document ID and sheet name configured via n8n parameter selectors. Configured with `onError: continueRegularOutput`.
    - *Connections:* Input: `Generate Audit Log Entry`. Output: None (Terminal node).
    - *Credentials:* Requires Google Sheets OAuth2 / Service Account credentials.
  - **Create Error Alert Message** (`n8n-nodes-base.code`)
    - *Role:* Error-trigger handler formatting readable execution failure notifications.
    - *Config:* JavaScript script parsing execution metadata (`$json.execution`, `$json.workflow`).
    - *Connections:* Input: Error Trigger (implicit). Output: `Send Failure Alert to Slack`.
  - **Send Failure Alert to Slack** (`n8n-nodes-base.slack`)
    - *Role:* Broadcasts workflow execution errors to the designated error-alert Slack channel.
    - *Config:* Posts markdown failure text to channel.
    - *Connections:* Input: `Create Error Alert Message`. Output: None (Terminal node).
    - *Credentials:* Requires Slack API credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation overview and setup instructions | None | None | ## Create Jira tickets from Gmail and Outlook support emails with OpenAI and Slack... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Gmail intake documentation | None | None | ## Gmail email intake<br><br>Receives new Gmail support messages and normalizes them into the common email shape used by the rest of the workflow. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Outlook intake documentation | None | None | ## Outlook email intake<br><br>Receives new Outlook support messages and normalizes Microsoft Graph email data into the shared email structure. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Classification routing documentation | None | None | ## Configure classification route<br><br>Adds workflow configuration, starts the classification step, and routes the main pipeline based on the returned outcome. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Escalation and acknowledgement documentation | None | None | ## Escalate and acknowledge<br><br>Invokes the escalation sub-workflow for high-priority emails, then calls the acknowledgement step to reply to the customer when appropriate. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Review queue documentation | None | None | ## Send for review<br><br>Routes uncertain classification results into the review sub-workflow for human triage. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Audit logging documentation | None | None | ## Call audit logging<br><br>Triggers the audit-log sub-workflow after the main outcome has been processed. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Sub-workflow dispatcher documentation | None | None | ## Sub-workflow dispatcher<br><br>Entry point for reusable sub-workflow calls; dispatches execution to classification, escalation, acknowledgement, review, or logging logic based on the step value. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Pre-filter documentation | None | None | ## Classification pre-filter<br><br>Filters out internal or automated senders before incurring AI usage and decides whether the message should be sent to the model. |
| Sticky Note9 | n8n-nodes-base.stickyNote | AI classification documentation | None | None | ## AI email classification<br><br>Uses an OpenAI chat model with structured parsing to classify eligible emails, then merges the AI result back with the original email data. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Skipped classification documentation | None | None | ## Skipped classification result<br><br>Produces a logged result for messages that are intentionally not sent to AI. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Jira routing documentation | None | None | ## Resolve Jira routing<br><br>Builds the Jira routing decision, searches for an existing open ticket for the email thread, and branches between follow-up and new issue handling. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Update ticket documentation | None | None | ## Update existing ticket<br><br>Adds the incoming email as a follow-up comment on an existing Jira ticket and formats the result like a newly handled issue. |
| Sticky Note13 | n8n-nodes-base.stickyNote | Create issue documentation | None | None | ## Create Jira issue<br><br>Creates a new Jira issue when no open ticket exists for the email thread. |
| Sticky Note14 | n8n-nodes-base.stickyNote | Team notifications documentation | None | None | ## Build team notifications<br><br>Builds escalation messages, posts the main notification to Slack, and checks whether a Teams webhook is configured. |
| Sticky Note15 | n8n-nodes-base.stickyNote | Escalation status documentation | None | None | ## Return escalation status<br><br>Optionally posts a Microsoft Teams card and returns the final escalation result to the main pipeline. |
| Sticky Note16 | n8n-nodes-base.stickyNote | Acknowledgement path documentation | None | None | ## Choose acknowledgement path<br><br>Prepares the customer acknowledgement text and selects the appropriate email reply channel or no-reply outcome. |
| Sticky Note17 | n8n-nodes-base.stickyNote | Customer replies documentation | None | None | ## Send customer replies<br><br>Sends or drafts Gmail acknowledgements, sends Outlook replies, and returns acknowledgement status to the caller. |
| Sticky Note18 | n8n-nodes-base.stickyNote | Review alert documentation | None | None | ## Post review alert<br><br>Builds a human-review Slack alert for low-confidence emails and returns the review routing result. |
| Sticky Note19 | n8n-nodes-base.stickyNote | Audit row documentation | None | None | ## Append audit row<br><br>Flattens the processed email outcome into an audit-log row and appends it to Google Sheets. |
| Sticky Note20 | n8n-nodes-base.stickyNote | Error notification documentation | None | None | ## Notify workflow failures<br><br>Handles workflow-level errors by formatting a readable alert and posting it to Slack. |
| When Email Received in Gmail | n8n-nodes-base.gmailTrigger | Triggers on new Gmail messages | None | Normalize Gmail Email | ## Gmail email intake<br><br>Receives new Gmail support messages and normalizes them into the common email shape used by the rest of the workflow. |
| When Email Received in Outlook | n8n-nodes-base.microsoftOutlookTrigger | Triggers on new Microsoft Outlook messages | None | Normalize Outlook Email | ## Outlook email intake<br><br>Receives new Outlook support messages and normalizes Microsoft Graph email data into the shared email structure. |
| Normalize Gmail Email | n8n-nodes-base.code | Normalizes Gmail JSON payload | When Email Received in Gmail | Set Workflow Configuration | ## Gmail email intake<br><br>Receives new Gmail support messages and normalizes them into the common email shape used by the rest of the workflow. |
| Normalize Outlook Email | n8n-nodes-base.code | Normalizes Outlook JSON payload | When Email Received in Outlook | Set Workflow Configuration | ## Outlook email intake<br><br>Receives new Outlook support messages and normalizes Microsoft Graph email data into the shared email structure. |
| Set Workflow Configuration | n8n-nodes-base.set | Assigns global workflow configuration variables | Normalize Gmail Email, Normalize Outlook Email | Prepare Step for Classification | ## Configure classification route<br><br>Adds workflow configuration, starts the classification step, and routes the main pipeline based on the returned outcome. |
| Prepare Step for Classification | n8n-nodes-base.set | Sets step parameter to classify | Set Workflow Configuration | Execute Email Classification | ## Configure classification route<br><br>Adds workflow configuration, starts the classification step, and routes the main pipeline based on the returned outcome. |
| Execute Email Classification | n8n-nodes-base.executeWorkflow | Recursively executes classification sub-workflow | Prepare Step for Classification | Route by Classification Outcome | ## Configure classification route<br><br>Adds workflow configuration, starts the classification step, and routes the main pipeline based on the returned outcome. |
| Route by Classification Outcome | n8n-nodes-base.switch | Routes based on confidence and request flags | Execute Email Classification | Prepare Step for Escalation, Prepare Step for Review, Prepare Step for Logging | ## Configure classification route<br><br>Adds workflow configuration, starts the classification step, and routes the main pipeline based on the returned outcome. |
| Prepare Step for Escalation | n8n-nodes-base.set | Sets step parameter to escalate | Route by Classification Outcome | Execute Escalation Workflow | ## Escalate and acknowledge<br><br>Invokes the escalation sub-workflow for high-priority emails, then calls the acknowledgement step to reply to the customer when appropriate. |
| Execute Escalation Workflow | n8n-nodes-base.executeWorkflow | Recursively executes escalation sub-workflow | Prepare Step for Escalation | Prepare Step for Acknowledgement | ## Escalate and acknowledge<br><br>Invokes the escalation sub-workflow for high-priority emails, then calls the acknowledgement step to reply to the customer when appropriate. |
| Prepare Step for Acknowledgement | n8n-nodes-base.set | Sets step parameter to acknowledge | Execute Escalation Workflow | Execute Customer Acknowledgement | ## Escalate and acknowledge<br><br>Invokes the escalation sub-workflow for high-priority emails, then calls the acknowledgement step to reply to the customer when appropriate. |
| Execute Customer Acknowledgement | n8n-nodes-base.executeWorkflow | Recursively executes acknowledgement sub-workflow | Prepare Step for Acknowledgement | Prepare Step for Logging | ## Escalate and acknowledge<br><br>Invokes the escalation sub-workflow for high-priority emails, then calls the acknowledgement step to reply to the customer when appropriate. |
| Prepare Step for Review | n8n-nodes-base.set | Sets step parameter to review | Route by Classification Outcome | Execute Review Queue Workflow | ## Send for review<br><br>Routes uncertain classification results into the review sub-workflow for human triage. |
| Execute Review Queue Workflow | n8n-nodes-base.executeWorkflow | Recursively executes review sub-workflow | Prepare Step for Review | Prepare Step for Logging | ## Send for review<br><br>Routes uncertain classification results into the review sub-workflow for human triage. |
| Prepare Step for Logging | n8n-nodes-base.set | Sets step parameter to log | Execute Customer Acknowledgement, Execute Review Queue Workflow | Execute Audit Log Workflow | ## Call audit logging<br><br>Triggers the audit-log sub-workflow after the main outcome has been processed. |
| Execute Audit Log Workflow | n8n-nodes-base.executeWorkflow | Recursively executes audit logging sub-workflow | Prepare Step for Logging | None (Terminal) | ## Call audit logging<br><br>Triggers the audit-log sub-workflow after the main outcome has been processed. |
| Route by Workflow Step | n8n-nodes-base.switch | Dispatches sub-workflow execution steps | Sub-workflow entry point | Filter and Prepare for AI, Build Jira Routing Decision, Build Acknowledgement Reply, Alert for Email Review, Generate Audit Log Entry | ## Sub-workflow dispatcher<br><br>Entry point for reusable sub-workflow calls; dispatches execution to classification, escalation, acknowledgement, review, or logging logic based on the step value. |
| Filter and Prepare for AI | n8n-nodes-base.code | Pre-filters senders and strips quoted replies | Route by Workflow Step | Check AI Classification Needed | ## Classification pre-filter<br><br>Filters out internal or automated senders before incurring AI usage and decides whether the message should be sent to the model. |
| Check AI Classification Needed | n8n-nodes-base.if | Checks whether AI processing is required | Filter and Prepare for AI | AI Email Classifier, Record Skipped AI Classification | ## Classification pre-filter<br><br>Filters out internal or automated senders before incurring AI usage and decides whether the message should be sent to the model. |
| AI Email Classifier | @n8n/n8n-nodes-langchain.chainLlm | LangChain model execution for email triage | Check AI Classification Needed | Combine Email and Classification | ## AI email classification<br><br>Uses an OpenAI chat model with structured parsing to classify eligible emails, then merges the AI result back with the original email data. |
| OpenAI GPT-4.1 Mini | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend for classifier | None | AI Email Classifier | ## AI email classification<br><br>Uses an OpenAI chat model with structured parsing to classify eligible emails, then merges the AI result back with the original email data. |
| Parse Classification Result | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output schema for classification | None | AI Email Classifier | ## AI email classification<br><br>Uses an OpenAI chat model with structured parsing to classify eligible emails, then merges the AI result back with the original email data. |
| Combine Email and Classification | n8n-nodes-base.code | Merges original email with AI results | AI Email Classifier | Route by Classification Outcome | ## AI email classification<br><br>Uses an OpenAI chat model with structured parsing to classify eligible emails, then merges the AI result back with the original email data. |
| Record Skipped AI Classification | n8n-nodes-base.code | Creates fallback record for skipped emails | Check AI Classification Needed | Route by Workflow Step (log branch) | ## Skipped classification result<br><br>Produces a logged result for messages that are intentionally not sent to AI. |
| Build Jira Routing Decision | n8n-nodes-base.code | Builds Jira payload and thread deduplication JQL | Route by Workflow Step | Search Open Jira Tickets | ## Resolve Jira routing<br><br>Builds the Jira routing decision, searches for an existing open ticket for the email thread, and branches between follow-up and new issue handling. |
| Search Open Jira Tickets | n8n-nodes-base.jira | Searches Jira for existing thread tickets | Build Jira Routing Decision | Check If Ticket Exists | ## Resolve Jira routing<br><br>Builds the Jira routing decision, searches for an existing open ticket for the email thread, and branches between follow-up and new issue handling. |
| Check If Ticket Exists | n8n-nodes-base.if | Evaluates whether open thread ticket exists | Search Open Jira Tickets | Comment on Existing Ticket, Post New Jira Issue | ## Resolve Jira routing<br><br>Builds the Jira routing decision, searches for an existing open ticket for the email thread, and branches between follow-up and new issue handling. |
| Comment on Existing Ticket | n8n-nodes-base.jira | Adds email follow-up comment to existing ticket | Check If Ticket Exists | Record Ticket Follow-Up | ## Update existing ticket<br><br>Adds the incoming email as a follow-up comment on an existing Jira ticket and formats the result like a newly handled issue. |
| Record Ticket Follow-Up | n8n-nodes-base.code | Exposes existing ticket key downstream | Comment on Existing Ticket | Create Notification Messages | ## Update existing ticket<br><br>Adds the incoming email as a follow-up comment on an existing Jira ticket and formats the result like a newly handled issue. |
| Post New Jira Issue | n8n-nodes-base.httpRequest | Creates new Jira support issue via REST API | Check If Ticket Exists | Create Notification Messages | ## Create Jira issue<br><br>Creates a new Jira issue when no open ticket exists for the email thread. |
| Create Notification Messages | n8n-nodes-base.code | Constructs Slack and Teams notification payloads | Record Ticket Follow-Up, Post New Jira Issue | Send Slack Notification | ## Build team notifications<br><br>Builds escalation messages, posts the main notification to Slack, and checks whether a Teams webhook is configured. |
| Send Slack Notification | n8n-nodes-base.slack | Posts escalation alert to Slack | Create Notification Messages | Check Teams Webhook Availability | ## Build team notifications<br><br>Builds escalation messages, posts the main notification to Slack, and checks whether a Teams webhook is configured. |
| Check Teams Webhook Availability | n8n-nodes-base.if | Checks if Microsoft Teams webhook URL is set | Send Slack Notification | Send Teams Notification, Return to Main Pipeline | ## Build team notifications<br><br>Builds escalation messages, posts the main notification to Slack, and checks whether a Teams webhook is configured. |
| Send Teams Notification | n8n-nodes-base.httpRequest | Posts adaptive card to Microsoft Teams webhook | Check Teams Webhook Availability | Return to Main Pipeline | ## Return escalation status<br><br>Optionally posts a Microsoft Teams card and returns the final escalation result to the main pipeline. |
| Return to Main Pipeline | n8n-nodes-base.code | Normalizes escalation status for caller | Send Teams Notification, Check Teams Webhook Availability | Prepare Step for Acknowledgement | ## Return escalation status<br><br>Optionally posts a Microsoft Teams card and returns the final escalation result to the main pipeline. |
| Build Acknowledgement Reply | n8n-nodes-base.code | Builds customer acknowledgement text and reply routes | Route by Workflow Step | Route by Reply Channel | ## Choose acknowledgement path<br><br>Prepares the customer acknowledgement text and selects the appropriate email reply channel or no-reply outcome. |
| Route by Reply Channel | n8n-nodes-base.switch | Routes reply to Gmail, Outlook, or skip | Build Acknowledgement Reply | Send Gmail Reply, Draft Gmail Reply, Send Outlook Reply, Conclude Acknowledgement Process | ## Choose acknowledgement path<br><br>Prepares the customer acknowledgement text and selects the appropriate email reply channel or no-reply outcome. |
| Send Gmail Reply | n8n-nodes-base.gmail | Sends direct email reply via Gmail | Route by Reply Channel | Conclude Acknowledgement Process | ## Send customer replies<br><br>Sends or drafts Gmail acknowledgements, sends Outlook replies, and returns acknowledgement status to the caller. |
| Draft Gmail Reply | n8n-nodes-base.gmail | Creates draft reply in Gmail | Route by Reply Channel | Conclude Acknowledgement Process | ## Send customer replies<br><br>Sends or drafts Gmail acknowledgements, sends Outlook replies, and returns acknowledgement status to the caller. |
| Send Outlook Reply | n8n-nodes-base.microsoftOutlook | Sends or drafts reply via Microsoft Outlook | Route by Reply Channel | Conclude Acknowledgement Process | ## Send customer replies<br><br>Sends or drafts Gmail acknowledgements, sends Outlook replies, and returns acknowledgement status to the caller. |
| Conclude Acknowledgement Process | n8n-nodes-base.code | Normalizes acknowledgement status | Send Gmail Reply, Draft Gmail Reply, Send Outlook Reply, Route by Reply Channel | Prepare Step for Logging | ## Send customer replies<br><br>Sends or drafts Gmail acknowledgements, sends Outlook replies, and returns acknowledgement status to the caller. |
| Alert for Email Review | n8n-nodes-base.code | Formats review alert text for uncertain emails | Route by Workflow Step | Send Review Request to Slack | ## Post review alert<br><br>Builds a human-review Slack alert for low-confidence emails and returns the review routing result. |
| Send Review Request to Slack | n8n-nodes-base.slack | Posts review request to Slack | Alert for Email Review | Return Review Outcome | ## Post review alert<br><br>Builds a human-review Slack alert for low-confidence emails and returns the review routing result. |
| Return Review Outcome | n8n-nodes-base.code | Returns review status to caller | Send Review Request to Slack | Prepare Step for Logging | ## Post review alert<br><br>Builds a human-review Slack alert for low-confidence emails and returns the review routing result. |
| Generate Audit Log Entry | n8n-nodes-base.code | Flattens execution state into audit log row | Route by Workflow Step | Add Entry to Triage Log | ## Append audit row<br><br>Flattens the processed email outcome into an audit-log row and appends it to Google Sheets. |
| Add Entry to Triage Log | n8n-nodes-base.googleSheets | Appends audit row to Google Sheets | Generate Audit Log Entry | None (Terminal) | ## Append audit row<br><br>Flattens the processed email outcome into an audit-log row and appends it to Google Sheets. |
| Create Error Alert Message | n8n-nodes-base.code | Formats workflow execution error alerts | Error Trigger | Send Failure Alert to Slack | ## Notify workflow failures<br><br>Handles workflow-level errors by formatting a readable alert and posting it to Slack. |
| Send Failure Alert to Slack | n8n-nodes-base.slack | Posts execution error alert to Slack | Create Error Alert Message | None (Terminal) | ## Notify workflow failures<br><br>Handles workflow-level errors by formatting a readable alert and posting it to Slack. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow in n8n manually, follow these ordered steps:

1. **Create Email Triggers & Normalization Nodes**
   - Add **When Email Received in Gmail** (`n8n-nodes-base.gmailTrigger`) configured with query `-from:me -category:promotions -category:social -category:updates -category:forums` and label `INBOX`. Connect credentials.
   - Add **When Email Received in Outlook** (`n8n-nodes-base.microsoftOutlookTrigger`) configured with raw output and poll time every minute. Connect credentials.
   - Add two **Code** nodes (`Normalize Gmail Email` and `Normalize Outlook Email`) to transform incoming payloads into a shared schema containing `source`, `messageId`, `threadId`, `receivedAt`, `fromName`, `fromEmail`, `subject`, `body`, `link`, and `hasAttachments`.
   - Connect both trigger nodes to their respective code normalizer nodes.

2. **Configure Global Settings & Sub-Workflow Router**
   - Add a **Set** node (`Set Workflow Configuration`) connected to both normalizer nodes. Populate assignment variables: `internalDomains` (string), `ignoreSenderPatterns` (string), `minConfidence` (number, `0.6`), `jiraBaseUrl` (string), `jiraProjectKey` (string), `bugIssueType` (`Bug`), `issueIssueType` (`Task`), `requirementIssueType` (`Story`), `bugSlackChannel`, `issueSlackChannel`, `requirementSlackChannel`, `reviewSlackChannel`, and `ackMode` (`off`).
   - Add a **Set** node (`Prepare Step for Classification`) setting `step = "classify"`. Connect it after configuration.
   - Add an **Execute Workflow** node (`Execute Email Classification`) targeting the current workflow ID (`{{ $workflow.id }}`) with `waitForSubWorkflow: true`.
   - Add a **Switch** node (`Route by Classification Outcome`) evaluating whether `isCustomerRequest === true`, category is in `['New Requirement','Issue','Bug']`, and `confidence >= minConfidence`. Set fallback output to `Ignore`.
   - Add a **Switch** node (`Route by Workflow Step`) at the sub-workflow entry point. Configure string matching on `{{ $json.step }}` for values: `classify`, `escalate`, `acknowledge`, `review`, and `log`.

3. **Build the AI Classification Chain**
   - Connect the `classify` output of `Route by Workflow Step` to a **Code** node (`Filter and Prepare for AI`) to filter internal/automated senders and trim body text.
   - Connect to an **If** node (`Check AI Classification Needed`) evaluating `{{ !$json.skipAI }}`.
   - For the true branch, add an **Advanced AI LangChain Classifier** (`@n8n/n8n-nodes-langchain.chainLlm`) linked to an **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`, model: `gpt-4.1-mini`, temperature: `0.1`) and a **Structured Output Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`) enforcing the triage schema (`isCustomerRequest`, `category`, `priority`, `confidence`, `productArea`, `title`, `summary`, `customerName`, `customerCompany`, `sentiment`, `stepsToReproduce`, `reasoning`). Connect OpenAI API credentials.
   - Add a **Code** node (`Combine Email and Classification`) to merge the original email with AI results.
   - For the false branch of the pre-filter check, add a **Code** node (`Record Skipped AI Classification`) producing fallback metadata.

4. **Build Jira Escalation Branch**
   - From `Route by Classification Outcome` (Escalate branch), add a **Set** node (`Prepare Step for Escalation`) setting `step = "escalate"`.
   - Add an **Execute Workflow** node (`Execute Escalation Workflow`) targeting the workflow ID.
   - From `Route by Workflow Step` (escalate branch), add a **Code** node (`Build Jira Routing Decision`) to generate issue payloads and thread hashes.
   - Add a **Jira** node (`Search Open Jira Tickets`) configured with operation `getAll` and JQL filter `labels = "{threadLabel}" AND statusCategory != Done ORDER BY created DESC`. Set `onError: continueRegularOutput`.
   - Add an **If** node (`Check If Ticket Exists`) checking if an open ticket key exists.
   - For the true branch, add a **Jira** node (`Comment on Existing Ticket`) to add follow-up comments, followed by a **Code** node (`Record Ticket Follow-Up`).
   - For the false branch, add an **HTTP Request** node (`Post New Jira Issue`) sending a POST request to `{{ jiraBaseUrl }}/rest/api/2/issue` using Jira Cloud API credentials.
   - Add a **Code** node (`Create Notification Messages`) to build Slack markdown and Teams adaptive card payloads.
   - Add a **Slack** node (`Send Slack Notification`) posting to `{{ $json.route.slackChannel }}`. Connect Slack credentials.
   - Add an **If** node (`Check Teams Webhook Availability`) evaluating `{{ !!teamsWebhookUrl }}`. If true, add an **HTTP Request** node (`Send Teams Notification`) posting to the webhook.
   - Add a **Code** node (`Return to Main Pipeline`) to consolidate escalation results.

5. **Build Customer Acknowledgement Branch**
   - From the escalation flow, add a **Set** node (`Prepare Step for Acknowledgement`) setting `step = "acknowledge"`.
   - Add an **Execute Workflow** node (`Execute Customer Acknowledgement`) targeting the workflow ID.
   - From `Route by Workflow Step` (acknowledge branch), add a **Code** node (`Build Acknowledgement Reply`) to construct acknowledgement messages based on `ackMode`.
   - Add a **Switch** node (`Route by Reply Channel`) evaluating `ackRoute` (`gmail-send`, `gmail-draft`, `outlook`).
   - Add **Gmail** (`Send Gmail Reply`, `Draft Gmail Reply`) and **Microsoft Outlook** (`Send Outlook Reply`) nodes configured with respective OAuth2 credentials.
   - Add a **Code** node (`Conclude Acknowledgement Process`) to standardize acknowledgement status.

6. **Build Review Queue, Audit Logging, and Error Handling Branches**
   - For review handling, add **Set** (`Prepare Step for Review`, `step = "review"`), **Execute Workflow** (`Execute Review Queue Workflow`), **Code** (`Alert for Email Review`), **Slack** (`Send Review Request to Slack`), and **Code** (`Return Review Outcome`) nodes.
   - For audit logging, add **Set** (`Prepare Step for Logging`, `step = "log"`), **Execute Workflow** (`Execute Audit Log Workflow`), **Code** (`Generate Audit Log Entry`), and **Google Sheets** (`Add Entry to Triage Log`, operation `append`) nodes connected with Google Sheets credentials.
   - For error handling, create an Error Trigger event linked to a **Code** node (`Create Error Alert Message`) and a **Slack** node (`Send Failure Alert to Slack`) posting failure details to the configured error channel. Set this workflow as its own error workflow in n8n workflow settings.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create Jira tickets from Gmail and Outlook support emails with OpenAI and Slack | Primary workflow title and functional overview |
| n8n Documentation & Sub-Workflow Architecture | [n8n Workflow Documentation](https://docs.n8n.io/) |