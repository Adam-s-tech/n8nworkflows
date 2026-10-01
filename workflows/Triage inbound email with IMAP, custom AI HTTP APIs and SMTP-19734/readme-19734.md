Triage inbound email with IMAP, custom AI HTTP APIs and SMTP

https://n8nworkflows.xyz/workflows/triage-inbound-email-with-imap--custom-ai-http-apis-and-smtp-19734


# Triage inbound email with IMAP, custom AI HTTP APIs and SMTP

### 1. Workflow Overview

This workflow automates the triage, classification, drafting, and response management of inbound emails. It supports dual entry methods—scheduled IMAP inbox polling or an incoming JSON webhook—to capture messages, standardizes them into a unified internal format, and passes them to external AI endpoints for categorization and response generation. Based on confidence scores and category rules, the workflow routes the generated reply either to automatic sending via SMTP with database logging, or to a human review queue with notifications dispatched to an external channel (e.g., Slack or email). An auxiliary callback webhook endpoint is also included to handle downstream human approval decisions.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures inbound emails via a 15-minute scheduled IMAP poll or an alternative HTTP webhook, then standardizes payload structures into a unified schema.
- **1.2 AI Processing & Drafting:** Sends normalized email payloads to external HTTP AI APIs for strict JSON classification and context-aware reply drafting.
- **1.3 Approval Gate & Routing:** Evaluates business rules (such as complaint detection and confidence scores) to branch the workflow between automatic dispatch and human review queues.
- **1.4 Execution & Logging (Auto-Send Path):** Delivers approved replies via SMTP and posts audit logs to a database endpoint.
- **1.5 Queue & Notification (Review Path):** Formats pending review items, triggers human notifications via webhook, and logs pending statuses.
- **1.6 Approval Callback Handling:** Receives external approval or rejection commands via a secondary webhook endpoint.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization
- **Overview:** Captures incoming messages from an IMAP mailbox on a fixed schedule or from an external push webhook, then flattens varying data structures into a clean, consistent internal format.
- **Nodes Involved:** 
  - `Poll Every 15 Minutes`
  - `Webhook for Email Inbound`
  - `Fetch Unseen Emails`
  - `Normalize Email Message`

- **Node Details:**
  - **Poll Every 15 Minutes**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) — Acts as the primary timed trigger for polling email inboxes.
    - *Configuration:* Interval set to every 15 minutes (`days: 0`, `hours: 0`, `minutes: 15`) with timezone set to `Asia/Shanghai`.
    - *Expressions / Variables:* None.
    - *Connections:* Input: None (Trigger); Output: `Fetch Unseen Emails`.
    - *Edge Cases / Failures:* Timezone drift or execution overlap if run frequencies are set too aggressively for large inboxes.
  - **Webhook for Email Inbound**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (v1.2) — Provides an alternative push-based entry point for external systems (e.g., Gmail API, Outlook forwarders, form builders).
    - *Configuration:* HTTP Method `POST`, response mode set to `onReceived`, path configured via environment variable placeholder `{{EMAIL_INBOUND_WEBHOOK_PATH}}`.
    - *Expressions / Variables:* `{{WEBHOOK_ID_EMAIL_INBOUND}}`, `{{EMAIL_INBOUND_WEBHOOK_PATH}}`.
    - *Connections:* Input: None (Trigger); Output: `Normalize Email Message`.
    - *Edge Cases / Failures:* Unauthorized payloads or network timeouts if upstream systems fail to reach the n8n instance.
  - **Fetch Unseen Emails**
    - *Type and Technical Role:* `n8n-nodes-base.emailReadImap` (v1.2) — Connects to an IMAP mail server to fetch unread messages.
    - *Configuration:* Mailbox set to `INBOX`, format set to `resolved`, post-process action set to `read` (marks messages as read).
    - *Credentials:* Requires an IMAP credential linked via `{{IMAP_CREDENTIAL_ID}}`.
    - *Expressions / Variables:* `{{IMAP_HOST}}`, `{{IMAP_USER}}`, `{{IMAP_PASSWORD}}`.
    - *Connections:* Input: `Poll Every 15 Minutes`; Output: `Normalize Email Message`.
    - *Edge Cases / Failures:* Authentication errors, SSL handshake failures, IMAP server connection timeouts, or missing credential configurations.
  - **Normalize Email Message**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that maps fields from disparate inputs (IMAP vs. Webhook) into a uniform schema.
    - *Configuration:* Normalizes body fields (`body`, `text`, `content`, `message`), sender fields (`from`, `sender`), subject fields, and timestamps. Limits body length to 20,000 characters.
    - *Expressions / Variables:* Uses standard JavaScript array mapping over `$input.all()`.
    - *Connections:* Input: `Fetch Unseen Emails`, `Webhook for Email Inbound`; Output: `Classify Email Content`.
    - *Edge Cases / Failures:* Unexpected payload shapes lacking standard properties, leading to empty or fallback values (`(no subject)`).

---

#### Block 1.2: AI Processing & Drafting
- **Overview:** Submits the normalized email content to external AI services via HTTP requests to classify intent/sentiment and generate a tailored response draft.
- **Nodes Involved:**
  - `Classify Email Content`
  - `Draft AI Reply`

- **Node Details:**
  - **Classify Email Content**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Sends normalized email details to a classification service.
    - *Configuration:* Method `POST`, timeout set to 30,000ms, sends a JSON body containing `message_id`, `channel`, `from`, `subject`, `body`, and `language`.
    - *Expressions / Variables:* `{{CLASSIFY_API_URL}}`, `{{CLASSIFY_LANGUAGE}}`, `={{ $json.messageId }}`, `={{ $json.channel }}`, `={{ $json.from }}`, `={{ $json.subject }}`, `={{ $json.body }}`.
    - *Connections:* Input: `Normalize Email Message`; Output: `Draft AI Reply`.
    - *Edge Cases / Failures:* HTTP 4xx/5xx errors from the external API, connection timeouts, or missing authentication headers if the API requires keys.
  - **Draft AI Reply**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Requests a response draft from an external AI drafting service based on classification metrics.
    - *Configuration:* Method `POST`, timeout set to 30,000ms, sends structured JSON including classification outputs (`category`, `sentiment`, `key_points`) and tone parameters.
    - *Expressions / Variables:* `{{DRAFT_API_URL}}`, `{{CLASSIFY_LANGUAGE}}`, `{{REPLY_TONE}}`, `={{ $json.data.category ?? $json.category }}`.
    - *Connections:* Input: `Classify Email Content`; Output: `Determine Approval Requirement`.
    - *Edge Cases / Failures:* Malformed JSON responses from the AI endpoint or missing fields in the classification output object.

---

#### Block 1.3: Approval Gate & Routing
- **Overview:** Evaluates business logic to determine if an email response can be sent automatically or requires human intervention.
- **Nodes Involved:**
  - `Determine Approval Requirement`
  - `If Auto Send Approved`

- **Node Details:**
  - **Determine Approval Requirement**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node executing routing logic and policy checks.
    - *Configuration:* Enforces a default `autoSend = false` switch. Forces review if the category is a `complaint` or if confidence is `< 0.6`.
    - *Expressions / Variables:* Accesses `inputItem.category`, `inputItem.confidence`, and `inputItem.data?.draft`.
    - *Connections:* Input: `Draft AI Reply`; Output: `If Auto Send Approved`.
    - *Edge Cases / Failures:* Missing nested properties resulting in NaN comparisons during confidence checks.
  - **If Auto Send Approved**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.1) — Conditional router dividing flow based on the evaluated auto-send status.
    - *Configuration:* Evaluates boolean condition where `{{ $json.autoSend }}` equals `true`.
    - *Expressions / Variables:* `={{ $json.autoSend }}`.
    - *Connections:* Input: `Determine Approval Requirement`; Outputs: 
      - True branch -> `Send SMTP Reply`
      - False branch -> `Queue for Human Approval`
    - *Edge Cases / Failures:* Type coercion errors where string values ("true" / "false") are incorrectly evaluated instead of strict booleans.

---

#### Block 1.4: Execution & Logging (Auto-Send Path)
- **Overview:** Handles messages meeting auto-send criteria by sending the reply back through the email server and recording the action in a logging database.
- **Nodes Involved:**
  - `Send SMTP Reply`
  - `Log Sent Email`

- **Node Details:**
  - **Send SMTP Reply**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (v1.2) — Sends the final response text back to the original sender using SMTP server settings while preserving threading headers.
    - *Configuration:* Format set to `text`, uses dynamic subject and body mappings.
    - *Credentials:* Requires an SMTP credential linked via `{{SMTP_CREDENTIAL_ID}}`.
    - *Expressions / Variables:* `{{SMTP_FROM_EMAIL}}`, `={{ $json.draft }}`, `={{ $json.subject }}`, `={{ $json.from }}`.
    - *Connections:* Input: `If Auto Send Approved` (True branch); Output: `Log Sent Email`.
    - *Edge Cases / Failures:* SMTP authentication rejection, relay restrictions, invalid recipient addresses, or connection timeouts.
  - **Log Sent Email**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Posts audit log entries for successfully sent automated emails to a remote database API.
    - *Configuration:* Method `POST`, timeout set to 30,000ms, JSON body containing message metadata, draft text, and status set to `sent`.
    - *Expressions / Variables:* `{{DB_WRITE_API_URL}}`, `={{ $json.messageId }}`, `={{ $json.channel }}`, `={{ $json.from }}`, `={{ $json.subject }}`, `={{ $json.body }}`, `={{ $json.category }}`, `={{ $json.draft }}`.
    - *Connections:* Input: `Send SMTP Reply`; Output: None (Terminal node).
    - *Edge Cases / Failures:* Database write errors, network drops, or schema mismatches at the logging endpoint.

---

#### Block 1.5: Queue & Notification (Review Path)
- **Overview:** Formats messages requiring human review, registers a queue tracking ID, logs pending states, and dispatches external notifications.
- **Nodes Involved:**
  - `Queue for Human Approval`
  - `Notify Approver for Review`
  - `Log Awaiting Approval`

- **Node Details:**
  - **Queue for Human Approval**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that constructs unique queue identifiers and formats notification strings.
    - *Configuration:* Generates a `queueId` prefixed with `q-` using message IDs or timestamps, sets `dbStatus` to `pending_review`, and slices notification text to 2,000 characters.
    - *Expressions / Variables:* JavaScript runtime expressions accessing `$input.first().json`.
    - *Connections:* Input: `If Auto Send Approved` (False branch); Outputs: `Notify Approver for Review`, `Log Awaiting Approval`.
    - *Edge Cases / Failures:* Missing message IDs resulting in non-unique fallback timestamps.
  - **Notify Approver for Review**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Dispatches review notifications to an external channel (e.g., Slack webhook or email notification API).
    - *Configuration:* Method `POST`, timeout set to 30,000ms, sends notification payload containing formatted text and queue IDs.
    - *Expressions / Variables:* `{{APPROVAL_NOTIFY_WEBHOOK_URL}}`, `={{ $json.notifyText }}`, `={{ $json.queueId }}`.
    - *Connections:* Input: `Queue for Human Approval`; Output: None (Terminal node).
    - *Edge Cases / Failures:* Webhook rate limits, invalid target URLs, or external notification service downtime.
  - **Log Awaiting Approval**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Records pending review items to the database logging API.
    - *Configuration:* Method `POST`, timeout set to 30,000ms, sends JSON payload with status set to `pending_review` and includes the assigned `queue_id`.
    - *Expressions / Variables:* `{{DB_WRITE_API_URL}}`, `={{ $json.messageId }}`, `={{ $json.channel }}`, `={{ $json.from }}`, `={{ $json.subject }}`, `={{ $json.body }}`, `={{ $json.category }}`, `={{ $json.draft }}`, `={{ $json.queueId }}`, `={{ $json.dbStatus }}`.
    - *Connections:* Input: `Queue for Human Approval`; Output: None (Terminal node).
    - *Edge Cases / Failures:* Database connection errors or API timeout exceptions.

---

#### Block 1.6: Approval Callback Handling
- **Overview:** Receives external approval decisions (approve/reject) via a standalone webhook endpoint to process downstream actions.
- **Nodes Involved:**
  - `Approval Callback Webhook`

- **Node Details:**
  - **Approval Callback Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (v1.2) — Standalone trigger endpoint waiting for callback actions from human reviewers.
    - *Configuration:* HTTP Method `POST`, response mode set to `onReceived`, path configured via `{{APPROVE_WEBHOOK_PATH}}`. Note: This node is intentionally left disconnected in the template skeleton.
    - *Expressions / Variables:* `{{WEBHOOK_ID_APPROVE}}`, `{{APPROVE_WEBHOOK_PATH}}`.
    - *Connections:* Input: None (Trigger); Output: None (Disconnected in skeleton).
    - *Edge Cases / Failures:* Unauthenticated callback attempts or payloads missing required `queue_id` parameters.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Poll Every 15 Minutes** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution on a scheduled interval. | None | Fetch Unseen Emails | Scheduled email intake |
| **Webhook for Email Inbound** | `n8n-nodes-base.webhook` | Receives inbound email payloads via an HTTP webhook. | None | Normalize Email Message | Optional webhook intake |
| **Fetch Unseen Emails** | `n8n-nodes-base.emailReadImap` | Fetches unread emails from an IMAP mailbox. | Poll Every 15 Minutes | Normalize Email Message | Scheduled email intake |
| **Normalize Email Message** | `n8n-nodes-base.code` | Normalizes varying input payloads into a standard schema. | Fetch Unseen Emails, Webhook for Email Inbound | Classify Email Content | Normalize incoming message |
| **Classify Email Content** | `n8n-nodes-base.httpRequest` | Sends email content to an external AI service for classification. | Normalize Email Message | Draft AI Reply | Classify and draft |
| **Draft AI Reply** | `n8n-nodes-base.httpRequest` | Requests a response draft from an AI service based on classification results. | Classify Email Content | Determine Approval Requirement | Classify and draft |
| **Determine Approval Requirement** | `n8n-nodes-base.code` | Evaluates business rules and confidence scores for auto-send eligibility. | Draft AI Reply | If Auto Send Approved | Approval routing gate |
| **If Auto Send Approved** | `n8n-nodes-base.if` | Branches execution based on whether auto-send is permitted. | Determine Approval Requirement | Send SMTP Reply, Queue for Human Approval | Approval routing gate |
| **Send SMTP Reply** | `n8n-nodes-base.emailSend` | Sends approved reply emails back to senders via SMTP. | If Auto Send Approved (True) | Log Sent Email | Send and log reply |
| **Log Sent Email** | `n8n-nodes-base.httpRequest` | Records successful automated replies to a logging database. | Send SMTP Reply | None | Send and log reply |
| **Queue for Human Approval** | `n8n-nodes-base.code` | Assigns queue IDs and formats notification strings for review items. | If Auto Send Approved (False) | Notify Approver for Review, Log Awaiting Approval | Queue human review |
| **Notify Approver for Review** | `n8n-nodes-base.httpRequest` | Dispatches review notifications to external webhooks (e.g., Slack). | Queue for Human Approval | None | Queue human review |
| **Log Awaiting Approval** | `n8n-nodes-base.httpRequest` | Records pending review items to the database logging API. | Queue for Human Approval | None | Queue human review |
| **Approval Callback Webhook** | `n8n-nodes-base.webhook` | Listens for external approval/rejection actions. | None | None | Approval callback endpoint |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the workflow manually in n8n:

1. **Create Triggers (Choose one active entry method in production):**
   - **Schedule Trigger:** Add a Schedule Trigger node named `Poll Every 15 Minutes`. Configure interval parameters to run every 15 minutes in timezone `Asia/Shanghai`.
   - **Webhook Trigger:** Add a Webhook node named `Webhook for Email Inbound`. Set HTTP Method to `POST`, response mode to `onReceived`, and define an appropriate webhook path variable.
   - **Callback Webhook:** Add a separate Webhook node named `Approval Callback Webhook` for handling human decisions (`POST`, response mode `onReceived`). Leave this node disconnected from the main pipeline.

2. **Configure Email Intake:**
   - Add an IMAP Email Read node named `Fetch Unseen Emails`. Connect `Poll Every 15 Minutes` to its input.
   - Set mailbox to `INBOX`, format to `resolved`, and post-process action to `read`.
   - Create or select an IMAP credential containing your server host, port, username, password, and SSL settings.

3. **Add Data Normalization:**
   - Add a Code node named `Normalize Email Message`. Connect both `Fetch Unseen Emails` and `Webhook for Email Inbound` to its input.
   - Populate the JavaScript code block to map incoming fields (`body`, `text`, `content`, `message`, `from`, `sender`, `subject`, etc.) into a consistent JSON schema with keys: `messageId`, `channel`, `from`, `to`, `subject`, `body`, and `receivedAt`.

4. **Set Up AI Processing HTTP Requests:**
   - Add an HTTP Request node named `Classify Email Content`. Connect `Normalize Email Message` to its input. Set method to `POST`, timeout to `30000ms`, and specify a JSON body including `message_id`, `channel`, `from`, `subject`, `body`, and `language`.
   - Add an HTTP Request node named `Draft AI Reply`. Connect `Classify Email Content` to its input. Set method to `POST`, timeout to `30000ms`, and configure the JSON body to include classification outputs (`category`, `sentiment`, `key_points`) along with tone settings.

5. **Configure Approval Logic and Routing:**
   - Add a Code node named `Determine Approval Requirement`. Connect `Draft AI Reply` to its input. Insert JavaScript code to set `autoSend = false` by default, flag review requirements for complaints or low confidence (`< 0.6`), and output the consolidated object.
   - Add an If node named `If Auto Send Approved`. Connect `Determine Approval Requirement` to its input. Configure a boolean condition verifying that `{{ $json.autoSend }}` equals `true`.

6. **Build the Auto-Send Branch (True Path):**
   - Add an Email Send (SMTP) node named `Send SMTP Reply`. Connect the `true` output of `If Auto Send Approved` to its input. Configure text format, subject, and recipient expressions.
   - Create or select an SMTP credential.
   - Add an HTTP Request node named `Log Sent Email`. Connect `Send SMTP Reply` to its input. Set method to `POST`, timeout to `30000ms`, and configure the JSON body to log action status as `sent`.

7. **Build the Human Review Branch (False Path):**
   - Add a Code node named `Queue for Human Approval`. Connect the `false` output of `If Auto Send Approved` to its input. Add JavaScript to generate a unique `queueId` (prefixed with `q-`), set `dbStatus` to `pending_review`, and format `notifyText`.
   - From `Queue for Human Approval`, create two parallel output connections:
     - **Notification Call:** Connect to an HTTP Request node named `Notify Approver for Review`. Set method to `POST`, timeout to `30000ms`, and point it to your Slack or notification webhook URL.
     - **Database Logging:** Connect to an HTTP Request node named `Log Awaiting Approval`. Set method to `POST`, timeout to `30000ms`, and configure the JSON body to record the item with status `pending_review` and its associated `queue_id`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Template Skeleton Notice** | This workflow is a working skeleton template and requires explicit credential and endpoint configuration before deployment. |
| **Trigger Selection Guidance** | Production environments must use only a single entry point (either scheduled IMAP polling or inbound webhooks) to prevent duplicate message processing. |
| **Human Approval Default** | The `autoSend` parameter defaults to `false` to route all drafts through the review queue during initial testing phases. |