Track and manage construction RFIs with Gmail, OpenAI, and Data Tables

https://n8nworkflows.xyz/workflows/track-and-manage-construction-rfis-with-gmail--openai--and-data-tables-19815


# Track and manage construction RFIs with Gmail, OpenAI, and Data Tables

### 1. Workflow Overview

This workflow automates the tracking, classification, routing, and management of Requests for Information (RFIs) within a construction management environment using Gmail, OpenAI, and n8n Data Tables. It handles incoming emails from subcontractors and suppliers, extracts details (including text from PDF attachments), correlates them with active construction projects, detects duplicate inquiries, routes new questions to design teams, routes design team answers through project manager approvals if impacts are flagged, and runs daily sweeps to chase overdue RFIs and compile project digests.

The workflow logic is grouped into the following functional blocks:
- **1.1 Database Initialization:** Creates required storage tables (`rfi_projects`, `rfi_log`) and inserts a sample project record via manual execution.
- **1.2 Inbound Email Ingestion & Settings:** Triggers on incoming emails matching targeted criteria, establishes global intake settings, loops through messages, downloads attachments, and extracts text from attached PDFs.
- **1.3 AI Classification & Parsing:** Utilizes an AI Agent powered by an OpenAI Language Model and a database tool to read, classify, and parse email contents and attachments into structured RFI data fields.
- **1.4 Email Routing & Project Matching:** Evaluates the classified email type (`new_question`, `design_answer`, or `other`) and retrieves matching project and RFI log information from Data Tables.
- **1.5 Duplicate Detection & New Question Logging:** Evaluates whether a new question matches a previously answered RFI using token and keyword scoring, logs unique or duplicate items, replies directly to repeat inquiries, and dispatches new questions to design teams while acknowledging receipt to the sender.
- **1.6 Design Answer Processing & Approval Workflow:** Validates whether incoming design team answers correspond to open RFIs, checks for cost or schedule impacts, requests Project Manager approval via email when impacts exist, and updates the RFI status accordingly.
- **1.7 Daily Sweep, Escalate & Digest:** Operates on a scheduled daily trigger to evaluate overdue open RFIs, send reminders to design teams, escalate aged items to Project Managers, and distribute comprehensive per-project digests.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Database Initialization

#### Overview
This block runs a one-time setup routine to create the underlying `rfi_projects` and `rfi_log` n8n Data Tables with appropriate schemas and seeds them with a sample project record.

#### Nodes Involved
- Run Once: Set Up RFI Tables
- Create RFI Projects Table
- Create RFI Log Table
- Add Sample Project

#### Node Details

- **Run Once: Set Up RFI Tables**
  - **Type & Role:** `n8n-nodes-base.manualTrigger` — Acts as the manual entry point to build storage infrastructure.
  - **Configuration:** No parameters required.
  - **Expressions:** None.
  - **Connections:** Output connects to `Create RFI Projects Table`.
  - **Version Requirements:** v1.
  - **Edge Cases:** Can only be triggered manually by an administrator during initial setup.

- **Create RFI Projects Table**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Creates the `rfi_projects` table if it does not already exist, defining columns for project metadata.
  - **Configuration:** Resource set to `table`, operation set to `create`, table name set to `rfi_projects`, with columns defined for `project_code`, `project_name`, `design_team_email`, `pm_email`, `response_days`, and `status`.
  - **Expressions:** None.
  - **Connections:** Input from `Run Once: Set Up RFI Tables`; output connects to `Create RFI Log Table`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Fails if permissions are restricted or if a table with a conflicting schema exists.

- **Create RFI Log Table**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Creates the `rfi_log` table to store individual RFI records, tracking status, dates, numbers, impacts, and source IDs.
  - **Configuration:** Resource set to `table`, operation set to `create`, table name set to `rfi_log`, executing once with columns for `rfi_number`, `project_code`, `project_name`, `trade`, `asked_by_company`, `asked_by_email`, `subject`, `question`, `spec_section`, `drawing_ref`, `status`, `date_received`, `date_due`, `date_answered`, `answer`, `cost_impact`, `schedule_impact`, `design_team_email`, `pm_email`, `duplicate_of`, and `source_message_id`.
  - **Expressions:** None.
  - **Connections:** Input from `Create RFI Projects Table`; output connects to `Add Sample Project`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Table creation errors if schema conflicts occur.

- **Add Sample Project**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Inserts a placeholder project record into the `rfi_projects` table.
  - **Configuration:** Operation set to record creation with mapping mode set to define values below for `project_code` (`RMOB-2026`), `project_name` (`Riverside Medical Office Building`), `design_team_email`, `pm_email`, `response_days` (`7`), and `status` (`open`). Executed once.
  - **Expressions:** None.
  - **Connections:** Input from `Create RFI Log Table`; no downstream connections in this initialization block.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Duplicate insertions if run repeatedly without clearing the table.

---

### Block 1.2: Inbound Email Ingestion & Settings

#### Overview
This block monitors the Gmail inbox for RFI-related emails, injects global intake configuration variables, iterates through incoming batches, downloads attachments, and extracts text from attached PDF documents.

#### Nodes Involved
- When an RFI Email Arrives
- RFI Intake Settings
- Loop Over RFI Emails
- Extract Attached PDF Text

#### Node Details

- **When an RFI Email Arrives**
  - **Type & Role:** `n8n-nodes-base.gmailTrigger` — Polling trigger that monitors Gmail for messages matching specified filters.
  - **Configuration:** Filters set to search query `subject:(RFI OR question OR clarification OR information) -from:me` with read status set to `both`. Options enabled to download attachments with prefix `attachment_`. Polls every minute.
  - **Expressions:** None.
  - **Connections:** Output connects to `RFI Intake Settings`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v1.4.
  - **Edge Cases:** API rate limits or OAuth token expiration.

- **RFI Intake Settings**
  - **Type & Role:** `n8n-nodes-base.set` — Establishes global configuration variables for the RFI processing logic.
  - **Configuration:** Assigns `company_name` (`Your Construction Co.`), `project_team_name` (`Project Team`), `review_inbox` (`user@example.com`), `min_match_confidence` (`0.7`), and `duplicate_threshold` (`0.45`). Includes other incoming fields.
  - **Expressions:** None.
  - **Connections:** Input from `When an RFI Email Arrives`; output connects to `Loop Over RFI Emails`.
  - **Version Requirements:** v3.4.
  - **Edge Cases:** Missing variables if downstream nodes reference incorrect keys.

- **Loop Over RFI Emails**
  - **Type & Role:** `n8n-nodes-base.splitInBatches` — Iterates through the list of retrieved emails one by one.
  - **Configuration:** Standard batch loop configuration.
  - **Expressions:** None.
  - **Connections:** Input from `RFI Intake Settings`; outputs branch based on item availability (Item 0 connects back to loop continuation/done; Item 1 connects to `Extract Attached PDF Text`).
  - **Version Requirements:** v3.

- **Extract Attached PDF Text**
  - **Type & Role:** `n8n-nodes-base.extractFromFile` — Extracts textual content from an email attachment.
  - **Configuration:** Operation set to `pdf`, binary property name set to `attachment_0`, options set to join pages. Error handling set to continue regular output.
  - **Expressions:** None.
  - **Connections:** Input from `Loop Over RFI Emails`; output connects to `Read and Classify the RFI`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Encrypted, password-protected, or image-only scanned PDFs where text extraction fails or returns empty strings.

---

### Block 1.3: AI Classification & Parsing

#### Overview
This block employs an AI Agent backed by an OpenAI language model and a Data Table tool to analyze email text and extracted PDF content, classifying the communication and extracting structured RFI parameters.

#### Nodes Involved
- Read and Classify the RFI
- OpenAI RFI Model
- Look Up Active Projects
- RFI Fields Parser

#### Node Details

- **Read and Classify the RFI**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` — Advanced AI agent node that orchestrates tool calls and output parsing.
  - **Configuration:** Maximum tries set to 3 with retry on fail (3000ms wait). System message instructs the model to sort RFI mail, query active projects first, categorize into `new_question`, `design_answer`, or `other`, match project codes, extract question details, summary rulings, and cost/schedule impact flags. Uses structured output parsing.
  - **Expressions:** Text prompt constructs an analysis payload using:
    - `=Email subject: {{ $('Loop Over RFI Emails').first().json.subject }}`
    - `From: {{ $('Loop Over RFI Emails').first().json.from.text }}`
    - Body slice: `{{ String($('Loop Over RFI Emails').first().json.text || '').slice(0, 4000) }}`
    - Attachment text slice: `{{ String($json.text || '').slice(0, 6000) }}`
  - **Connections:** Connected to `OpenAI RFI Model` (AI Language Model), `Look Up Active Projects` (AI Tool), and `RFI Fields Parser` (AI Output Parser). Output connects to `Route by Email Type`.
  - **Version Requirements:** v3.1.
  - **Edge Cases:** Hallucination of project codes or extraction failures on unstructured email threads.

- **OpenAI RFI Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the underlying LLM capability to the AI Agent.
  - **Configuration:** Model configured to `gpt-5-mini`.
  - **Expressions:** None.
  - **Connections:** Linked to `Read and Classify the RFI`.
  - **Credentials:** OpenAI API account.
  - **Version Requirements:** v1.3.
  - **Edge Cases:** OpenAI API downtime, rate limits, or token quota exhaustion.

- **Look Up Active Projects**
  - **Type & Role:** `n8n-nodes-base.dataTableTool` — Exposes active projects as a tool for the AI Agent.
  - **Configuration:** Operation set to `get`, returning all rows from data table `rfi_projects` filtered by `status` equals `open`.
  - **Expressions:** None.
  - **Connections:** Linked to `Read and Classify the RFI` as an AI tool.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Empty project table causing project matching to fail.

- **RFI Fields Parser**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a strict JSON schema on the AI Agent's output.
  - **Configuration:** JSON schema example defines expected fields including `email_type`, `project_code`, `match_confidence`, `asker_company`, `asker_email`, `subject`, `question`, `spec_section`, `drawing_ref`, `trade`, `rfi_number`, `answer`, `has_cost_impact`, `has_schedule_impact`, and `reasoning`.
  - **Expressions:** None.
  - **Connections:** Linked to `Read and Classify the RFI` as an AI output parser.
  - **Version Requirements:** v1.3.
  - **Edge Cases:** Parsing errors if model output violates the defined JSON schema.

---

### Block 1.4: Email Routing & Project Matching

#### Overview
This block evaluates the classification result from the AI agent, routing the execution path to handle new questions, design team answers, or flagging unmatchable emails for manual review.

#### Nodes Involved
- Route by Email Type
- Get Matched Project
- Get RFIs for This Project

#### Node Details

- **Route by Email Type**
  - **Type & Role:** `n8n-nodes-base.switch` — Routes items based on classified email type and confidence thresholds.
  - **Configuration:** Rule 1 (`New question`): Evaluates `email_type` equals `new_question`, project code length greater than 0, and `match_confidence` greater than or equal to `{{ $('RFI Intake Settings').first().json.min_match_confidence }}`. Rule 2 (`Design answer`): Evaluates `email_type` equals `design_answer` and RFI number length greater than 0. Fallback output (`Needs review`) handles unmatched items.
  - **Expressions:** Uses conditional expressions referencing `{{ $json.output.email_type }}`, `{{ $json.output.project_code }}`, `{{ $json.output.match_confidence }}`, and `{{ $json.output.rfi_number }}`.
  - **Connections:** Input from `Read and Classify the RFI`. Output 1 connects to `Get Matched Project`; Output 2 connects to `Get the Logged RFI`; Fallback output connects to `Flag Email for Review`.
  - **Version Requirements:** v3.4.
  - **Edge Cases:** Emails falling into fallback if confidence is borderline.

- **Get Matched Project**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Retrieves project details from the `rfi_projects` table for matched questions.
  - **Configuration:** Operation set to `get`, limit 1, filtering where `project_code` matches the AI output.
  - **Expressions:** Filter value: `={{ $json.output.project_code }}`.
  - **Connections:** Input from `Route by Email Type`; output connects to `Get RFIs for This Project`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Missing project record returning empty arrays.

- **Get RFIs for This Project**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Loads existing RFI records for the target project to check history and assign numbers.
  - **Configuration:** Operation set to `get`, returning all rows from `rfi_log` filtered by `project_code`. Always outputs data.
  - **Expressions:** Filter value: `={{ $json.project_code }}`.
  - **Connections:** Input from `Get Matched Project`; output connects to `Number and Check for Duplicates`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Empty result sets on brand new projects with zero prior RFIs.

---

### Block 1.5: Duplicate Detection & New Question Logging

#### Overview
This block runs custom JavaScript logic to evaluate duplicate questions using token overlap, spec section matching, and drawing references, assigns sequential RFI numbers and due dates, logs items into the RFI database, and dispatches responses or notifications via Gmail.

#### Nodes Involved
- Number and Check for Duplicates
- Already Logged?
- Already Answered Before?
- Log the Duplicate RFI
- Reply With the Previous Answer
- Log the New RFI
- Send RFI to Design Team
- Acknowledge to Subcontractor

#### Node Details

- **Number and Check for Duplicates**
  - **Type & Role:** `n8n-nodes-base.code` — Executes JavaScript to compute duplicate scores, check source message IDs to prevent duplicate processing, assign sequential RFI numbers, and set due dates.
  - **Configuration:** Custom JS parsing tokens, calculating wording similarity, checking spec sections and drawing references against previously answered RFIs, and evaluating duplicate thresholds.
  - **Expressions:** JavaScript logic processes inputs from AI classification, matched project data, email payload, intake settings, and existing RFI logs.
  - **Connections:** Input from `Get RFIs for This Project`; output connects to `Already Logged?`.
  - **Version Requirements:** v2.
  - **Edge Cases:** False positive duplicate matches if threshold is set too low.

- **Already Logged?**
  - **Type & Role:** `n8n-nodes-base.if` — Checks if the email message ID has already been logged.
  - **Configuration:** Evaluates if `already_logged` is `true`.
  - **Expressions:** Condition: `={{ $json.already_logged === true }}`.
  - **Connections:** Input from `Number and Check for Duplicates`; True branch connects to `Loop Over RFI Emails` (skipping); False branch connects to `Already Answered Before?`.
  - **Version Requirements:** v2.3.
  - **Edge Cases:** Re-processing identical message IDs if state tracking fails.

- **Already Answered Before?**
  - **Type & Role:** `n8n-nodes-base.if` — Determines whether the question is a duplicate of a previously answered RFI.
  - **Configuration:** Evaluates if `is_duplicate` is `true`.
  - **Expressions:** Condition: `={{ $json.is_duplicate === true }}`.
  - **Connections:** Input from `Already Logged?`; True branch connects to `Log the Duplicate RFI`; False branch connects to `Log the New RFI`.
  - **Version Requirements:** v2.3.
  - **Edge Cases:** Edge cases in wording variations leading to missed duplicate detection.

- **Log the Duplicate RFI**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Records the duplicate RFI entry into the `rfi_log` table.
  - **Configuration:** Operation set to `create` with mapping mode defining values from current JSON fields (`trade`, `answer`, `status`, `subject`, `date_due`, `pm_email`, `question`, `rfi_number`, `cost_impact`, `drawing_ref`, `duplicate_of`, `project_code`, `project_name`, `spec_section`, `date_answered`, `date_received`, `asked_by_email`, `schedule_impact`, `asked_by_company`, `design_team_email`, `source_message_id`).
  - **Expressions:** Mapped field values referencing `{{ $json.<field> }}`.
  - **Connections:** Input from `Already Answered Before?`; output connects to `Reply With the Previous Answer`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Database write failures.

- **Reply With the Previous Answer**
  - **Type & Role:** `n8n-nodes-base.gmail` — Sends a reply email to the subcontractor providing the prior answer.
  - **Configuration:** Operation set to `reply`, message type text, appending no attribution, using original message ID.
  - **Expressions:** Message body references `{{ $('Number and Check for Duplicates').first().json.asked_by_company }}`, `project_name`, `duplicate_of`, `answer`, and project team settings. Message ID references `{{ $('Loop Over RFI Emails').first().json.id }}`.
  - **Connections:** Input from `Log the Duplicate RFI`; output connects to `Loop Over RFI Emails`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Missing message IDs or Gmail send failures.

- **Log the New RFI**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Logs a brand new open RFI into the `rfi_log` table.
  - **Configuration:** Operation set to `create` mapping all required fields (`trade`, `answer`, `status`, `subject`, `date_due`, `pm_email`, `question`, `rfi_number`, `cost_impact`, `drawing_ref`, `duplicate_of`, `project_code`, `project_name`, `spec_section`, `date_answered`, `date_received`, `asked_by_email`, `schedule_impact`, `asked_by_company`, `design_team_email`, `source_message_id`).
  - **Expressions:** Field values mapped from `={{ $json.<field> }}`.
  - **Connections:** Input from `Already Answered Before?`; output connects to `Send RFI to Design Team`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Table write errors or schema mismatches.

- **Send RFI to Design Team**
  - **Type & Role:** `n8n-nodes-base.gmail` — Emails the new RFI inquiry to the design team.
  - **Configuration:** Operation set to `send`, email type text, no attribution.
  - **Expressions:** Recipient: `={{ $('Number and Check for Duplicates').first().json.design_team_email }}`. Subject: `={{ 'RFI ' + ... }}`. Message body includes RFI metadata, question text, and instructions.
  - **Connections:** Input from `Log the New RFI`; output connects to `Acknowledge to Subcontractor`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Invalid design team email addresses.

- **Acknowledge to Subcontractor**
  - **Type & Role:** `n8n-nodes-base.gmail` — Sends an acknowledgement receipt email back to the subcontractor.
  - **Configuration:** Operation set to `reply`, email type text, using original message ID.
  - **Expressions:** Message body references RFI number, project name, and due date. Message ID references `{{ $('Loop Over RFI Emails').first().json.id }}`.
  - **Connections:** Input from `Send RFI to Design Team`; output connects to `Loop Over RFI Emails`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Gmail API errors.

---

### Block 1.6: Design Answer Processing & Approval Workflow

#### Overview
This block processes incoming design team answers, verifies that the target RFI is still open, evaluates cost and schedule impacts, triggers a Project Manager approval workflow when impacts are present, updates table statuses, and dispatches answers to subcontractors.

#### Nodes Involved
- Get the Logged RFI
- Is the RFI Still Open?
- Cost or Schedule Impact?
- PM Approves the Answer
- Answer Approved?
- Send the Answer to the Sub
- Close the RFI
- Hold the RFI for Review
- Flag Email for Review

#### Node Details

- **Get the Logged RFI**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Retrieves the RFI log record matching an incoming design team answer.
  - **Configuration:** Operation set to `get`, limit 1, filtering by `rfi_number`. Always outputs data.
  - **Expressions:** Filter value: `={{ String($json.output.rfi_number || '').trim() }}`.
  - **Connections:** Input from `Route by Email Type` (Design answer branch); output connects to `Is the RFI Still Open?`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Unmatched RFI numbers returning empty records.

- **Is the RFI Still Open?**
  - **Type & Role:** `n8n-nodes-base.if` — Verifies that the retrieved RFI exists and is currently in an `open` status.
  - **Configuration:** Evaluates if record ID > 0 and status equals `open`.
  - **Expressions:** Conditions evaluate `={{ Number($json.id) || 0 }}` and `={{ $json.status }}`.
  - **Connections:** Input from `Get the Logged RFI`; True branch connects to `Cost or Schedule Impact?`; False branch connects to `Flag Email for Review`.
  - **Version Requirements:** v2.3.
  - **Edge Cases:** Already closed or non-existent RFIs routed to review.

- **Cost or Schedule Impact?**
  - **Type & Role:** `n8n-nodes-base.if` — Checks whether the design answer flags cost or schedule impacts.
  - **Configuration:** Evaluates if either `has_cost_impact` or `has_schedule_impact` is true.
  - **Expressions:** Conditions check `={{ $('Read and Classify the RFI').first().json.output.has_cost_impact === true }}` and `={{ $('Read and Classify the RFI').first().json.output.has_schedule_impact === true }}` (Combined with OR logic).
  - **Connections:** Input from `Is the RFI Still Open?`; True branch connects to `PM Approves the Answer`; False branch connects to `Send the Answer to the Sub`.
  - **Version Requirements:** v2.3.
  - **Edge Cases:** Misclassification by AI leading to bypassed PM approvals.

- **PM Approves the Answer**
  - **Type & Role:** `n8n-nodes-base.gmail` — Sends an interactive approval email to the Project Manager.
  - **Configuration:** Operation set to `sendAndWait`, approval options configured with double approval type (`Send the answer` vs `Hold for review`). Limit wait time set to resume after 3 days.
  - **Expressions:** Recipient: `={{ $('Get the Logged RFI').first().json.pm_email }}`. Subject: `={{ 'Approve answer for RFI ' + ... }}`. Message body includes question, answer, and impact flags.
  - **Connections:** Input from `Cost or Schedule Impact?`; output connects to `Answer Approved?`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** PM failing to respond within the wait timeout window.

- **Answer Approved?**
  - **Type & Role:** `n8n-nodes-base.if` — Evaluates the PM's response from the approval email.
  - **Configuration:** Checks if approval data indicates approval equals true.
  - **Expressions:** Condition: `={{ $json.data?.approved === true }}`.
  - **Connections:** Input from `PM Approves the Answer`; True branch connects to `Send the Answer to the Sub`; False branch connects to `Hold the RFI for Review`.
  - **Version Requirements:** v2.3.
  - **Edge Cases:** Missing or corrupted approval payload data.

- **Send the Answer to the Sub**
  - **Type & Role:** `n8n-nodes-base.gmail` — Emails the approved design answer to the subcontractor.
  - **Configuration:** Operation set to `send`, email type text, no attribution.
  - **Expressions:** Recipient: `={{ $('Get the Logged RFI').first().json.asked_by_email }}`. Subject and message body reference RFI number, project name, and design answer.
  - **Connections:** Inputs from `Cost or Schedule Impact?` (False branch) and `Answer Approved?` (True branch); output connects to `Close the RFI`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Invalid subcontractor email addresses.

- **Close the RFI**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Updates the RFI log entry to `answered` status and logs answer details.
  - **Configuration:** Operation set to `update`, matching by `rfi_number`. Updates `answer` (`{{ $('Read and Classify the RFI').first().json.output.answer }}`), `status` (`answered`), `cost_impact`, `date_answered`, and `schedule_impact`.
  - **Expressions:** Field values mapped from AI output and current timestamps (`$now.toFormat('yyyy-MM-dd')`).
  - **Connections:** Input from `Send the Answer to the Sub`; output connects to `Loop Over RFI Emails`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Database update failures.

- **Hold the RFI for Review**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Updates the RFI log status to `pm_review` when PM holds the answer.
  - **Configuration:** Operation set to `update`, matching by `rfi_number`. Updates `answer` and sets `status` to `pm_review`.
  - **Expressions:** Field values mapped from AI output.
  - **Connections:** Input from `Answer Approved?` (False branch); output connects to `Loop Over RFI Emails`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Database update errors.

- **Flag Email for Review**
  - **Type & Role:** `n8n-nodes-base.gmail` — Forwards unmatchable or invalid emails to the review inbox.
  - **Configuration:** Operation set to `send`, email type text, no attribution.
  - **Expressions:** Recipient: `={{ $('RFI Intake Settings').first().json.review_inbox }}`. Subject and message body reference sender, subject, and AI reasoning.
  - **Connections:** Inputs from `Route by Email Type` (Fallback) and `Is the RFI Still Open?` (False branch); output connects to `Loop Over RFI Emails`.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Gmail send errors.

---

### Block 1.7: Daily Sweep, Escalate & Digest

#### Overview
This block runs daily at 8:00 AM to review open RFIs, calculate lateness, send reminders to design teams for due/overdue items, escalate severely overdue items to Project Managers, and compile and email per-project open RFI digests.

#### Nodes Involved
- Daily RFI Sweep 8am
- Sweep Settings
- Load Open RFIs
- Age and Route Open RFIs
- Route by RFI Age
- Remind the Design Team
- Escalate to Project Manager
- Build the Open RFI Digest
- Email the Open RFI Digest

#### Node Details

- **Daily RFI Sweep 8am**
  - **Type & Role:** `n8n-nodes-base.scheduleTrigger` — Cron-based schedule trigger running daily at 8 AM.
  - **Configuration:** Rule configured to trigger at hour 8.
  - **Expressions:** None.
  - **Connections:** Output connects to `Sweep Settings`.
  - **Version Requirements:** v1.3.
  - **Edge Cases:** Server timezone misconfigurations.

- **Sweep Settings**
  - **Type & Role:** `n8n-nodes-base.set` — Defines configuration parameters for the daily sweep.
  - **Configuration:** Assigns `company_name`, `project_team_name`, and `escalate_after_days` (`2`).
  - **Expressions:** None.
  - **Connections:** Input from `Daily RFI Sweep 8am`; output connects to `Load Open RFIs`.
  - **Version Requirements:** v3.4.
  - **Edge Cases:** Missing parameter keys.

- **Load Open RFIs**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Loads all open RFI records from the `rfi_log` table.
  - **Configuration:** Operation set to `get`, returning all rows where `status` equals `open`.
  - **Expressions:** None.
  - **Connections:** Input from `Sweep Settings`; outputs branch to both `Age and Route Open RFIs` and `Build the Open RFI Digest`.
  - **Version Requirements:** v1.1.
  - **Edge Cases:** Empty data tables returning no records.

- **Age and Route Open RFIs**
  - **Type & Role:** `n8n-nodes-base.code` — Executes JavaScript to calculate days overdue and assign actions (`wait`, `chase`, or `escalate`).
  - **Configuration:** Custom JS parsing due dates using Luxon (`DateTime.fromISO`), calculating difference against current date, and applying escalation windows.
  - **Expressions:** JavaScript processing records from `Load Open RFIs`.
  - **Connections:** Input from `Load Open RFIs`; output connects to `Route by RFI Age`.
  - **Version Requirements:** v2.
  - **Edge Cases:** Invalid date format strings causing parsing errors.

- **Route by RFI Age**
  - **Type & Role:** `n8n-nodes-base.switch` — Routes RFI items based on calculated action requirements.
  - **Configuration:** Rule 1 (`Chase`): Evaluates `action` equals `chase`. Rule 2 (`Escalate`): Evaluates `action` equals `escalate`.
  - **Expressions:** Condition references `={{ $json.action }}`.
  - **Connections:** Input from `Age and Route Open RFIs`; Output 1 connects to `Remind the Design Team`; Output 2 connects to `Escalate to Project Manager`.
  - **Version Requirements:** v3.4.
  - **Edge Cases:** Items categorized as `wait` are filtered out.

- **Remind the Design Team**
  - **Type & Role:** `n8n-nodes-base.gmail` — Sends a reminder email to the design team for due or slightly overdue RFIs.
  - **Configuration:** Operation set to `send`, email type text, no attribution.
  - **Expressions:** Recipient: `={{ $json.design_team_email }}`. Subject and message body include RFI number, project name, subject, and due date.
  - **Connections:** Input from `Route by RFI Age` (Chase branch); no downstream connections.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Invalid design team email addresses.

- **Escalate to Project Manager**
  - **Type & Role:** `n8n-nodes-base.gmail` — Sends an escalation notice to the Project Manager for severely overdue RFIs.
  - **Configuration:** Operation set to `send`, email type text, no attribution.
  - **Expressions:** Recipient: `={{ $json.pm_email }}`. Subject and message body include RFI number, project name, days overdue, and design team contact.
  - **Connections:** Input from `Route by RFI Age` (Escalate branch); no downstream connections.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Invalid project manager email addresses.

- **Build the Open RFI Digest**
  - **Type & Role:** `n8n-nodes-base.code` — Compiles and formats open RFIs into a structured HTML digest grouped by project.
  - **Configuration:** Custom JS grouping open rows by project code, calculating lateness, sorting by overdue days descending, and generating HTML tables.
  - **Expressions:** JavaScript processing records from `Load Open RFIs`.
  - **Connections:** Input from `Load Open RFIs`; output connects to `Email the Open RFI Digest`.
  - **Version Requirements:** v2.
  - **Edge Cases:** Missing project emails resulting in un-routable digest objects.

- **Email the Open RFI Digest**
  - **Type & Role:** `n8n-nodes-base.gmail` — Emails the compiled HTML open RFI digest to each Project Manager.
  - **Configuration:** Operation set to `send`, plain/HTML message rendering, no attribution.
  - **Expressions:** Recipient: `={{ $json.pm_email }}`. Subject references `={{ 'Open RFIs on ' + $json.project_name + ': ' + $json.open_count + ' open, ' + $json.overdue_count + ' overdue' }}`. Message body includes project name, counts, and `digest_html`.
  - **Connections:** Input from `Build the Open RFI Digest`; no downstream connections.
  - **Credentials:** Gmail OAuth2 account.
  - **Version Requirements:** v2.2.
  - **Edge Cases:** Email client rendering issues with HTML tables.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Once: Set Up RFI Tables | n8n-nodes-base.manualTrigger | Manual entry point to initialize setup | None | Create RFI Projects Table | ## Create the RFI tables<br><br>Run once to create the rfi_projects and rfi_log Data Tables with a sample project. |
| Create RFI Projects Table | n8n-nodes-base.dataTable | Creates rfi_projects data table | Run Once: Set Up RFI Tables | Create RFI Log Table | ## Create the RFI tables<br><br>Run once to create the rfi_projects and rfi_log Data Tables with a sample project. |
| Create RFI Log Table | n8n-nodes-base.dataTable | Creates rfi_log data table | Create RFI Projects Table | Add Sample Project | ## Create the RFI tables<br><br>Run once to create the rfi_projects and rfi_log Data Tables with a sample project. |
| Add Sample Project | n8n-nodes-base.dataTable | Inserts sample project record | Create RFI Log Table | None | ## Create the RFI tables<br><br>Run once to create the rfi_projects and rfi_log Data Tables with a sample project. |
| When an RFI Email Arrives | n8n-nodes-base.gmailTrigger | Triggers on incoming RFI emails | None | RFI Intake Settings | ## Keep the Gmail search tight<br><br>This workflow replies to senders automatically. The query already skips your own mail; narrow it further if this mailbox gets a lot of unrelated post. |
| RFI Intake Settings | n8n-nodes-base.set | Sets global configuration variables | When an RFI Email Arrives | Loop Over RFI Emails | None |
| Loop Over RFI Emails | n8n-nodes-base.splitInBatches | Iterates through email batch | RFI Intake Settings, Reply With the Previous Answer, Acknowledge to Subcontractor, Close the RFI, Hold the RFI for Review, Flag Email for Review, Already Logged? | Extract Attached PDF Text | None |
| Extract Attached PDF Text | n8n-nodes-base.extractFromFile | Extracts text from attached PDF | Loop Over RFI Emails | Read and Classify the RFI | None |
| Read and Classify the RFI | @n8n/n8n-nodes-langchain.agent | AI agent for mail classification and parsing | Extract Attached PDF Text | Route by Email Type | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| OpenAI RFI Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend for AI agent | None | Read and Classify the RFI | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| Look Up Active Projects | n8n-nodes-base.dataTableTool | Exposes open projects as AI tool | None | Read and Classify the RFI | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| RFI Fields Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output structure for AI | None | Read and Classify the RFI | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| Route by Email Type | n8n-nodes-base.switch | Routes based on classification type | Read and Classify the RFI | Get Matched Project, Get the Logged RFI, Flag Email for Review | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| Get Matched Project | n8n-nodes-base.dataTable | Retrieves project data from table | Route by Email Type | Get RFIs for This Project | None |
| Get RFIs for This Project | n8n-nodes-base.dataTable | Loads project RFI log history | Get Matched Project | Number and Check for Duplicates | None |
| Number and Check for Duplicates | n8n-nodes-base.code | Scores duplicates and assigns RFI numbers | Get RFIs for This Project | Already Logged? | ## Number and check for repeats<br><br>Skips mail already logged, assigns the next RFI number, and scores the question against RFIs this project has already answered. |
| Already Logged? | n8n-nodes-base.if | Checks if email was already logged | Number and Check for Duplicates | Loop Over RFI Emails, Already Answered Before? | ## Number and check for repeats<br><br>Skips mail already logged, assigns the next RFI number, and scores the question against RFIs this project has already answered. |
| Already Answered Before? | n8n-nodes-base.if | Checks if question is a duplicate | Already Logged? | Log the Duplicate RFI, Log the New RFI | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Log the Duplicate RFI | n8n-nodes-base.dataTable | Records duplicate RFI entry | Already Answered Before? | Reply With the Previous Answer | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Reply With the Previous Answer | n8n-nodes-base.gmail | Replies to sub with prior answer | Log the Duplicate RFI | Loop Over RFI Emails | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Log the New RFI | n8n-nodes-base.dataTable | Logs new open RFI record | Already Answered Before? | Send RFI to Design Team | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Send RFI to Design Team | n8n-nodes-base.gmail | Emails new RFI to design team | Log the New RFI | Acknowledge to Subcontractor | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Acknowledge to Subcontractor | n8n-nodes-base.gmail | Sends receipt ack to sub | Send RFI to Design Team | Loop Over RFI Emails | ## Log and route the question<br><br>Repeat questions get the earlier ruling straight away; new ones go to the design team with an acknowledgement to the sub. |
| Get the Logged RFI | n8n-nodes-base.dataTable | Retrieves RFI log record for answers | Route by Email Type | Is the RFI Still Open? | ## File the design answer<br><br>Matches the reply to its RFI number and checks whether the answer carries a cost or schedule impact. |
| Is the RFI Still Open? | n8n-nodes-base.if | Verifies RFI status is open | Get the Logged RFI | Cost or Schedule Impact?, Flag Email for Review | ## File the design answer<br><br>Matches the reply to its RFI number and checks whether the answer carries a cost or schedule impact. |
| Cost or Schedule Impact? | n8n-nodes-base.if | Checks for cost or schedule impacts | Is the RFI Still Open? | PM Approves the Answer, Send the Answer to the Sub | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| PM Approves the Answer | n8n-nodes-base.gmail | Sends interactive approval to PM | Cost or Schedule Impact? | Answer Approved? | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| Answer Approved? | n8n-nodes-base.if | Evaluates PM approval result | PM Approves the Answer | Send the Answer to the Sub, Hold the RFI for Review | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| Send the Answer to the Sub | n8n-nodes-base.gmail | Emails design answer to sub | Cost or Schedule Impact?, Answer Approved? | Close the RFI | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| Close the RFI | n8n-nodes-base.dataTable | Updates RFI status to answered | Send the Answer to the Sub | Loop Over RFI Emails | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| Hold the RFI for Review | n8n-nodes-base.dataTable | Updates RFI status to pm_review | Answer Approved? | Loop Over RFI Emails | ## Approve and close the RFI<br><br>Impacts wait for the project manager; everything else goes to the sub and the RFI is closed. |
| Flag Email for Review | n8n-nodes-base.gmail | Forwards unmatchable emails to review | Route by Email Type, Is the RFI Still Open? | Loop Over RFI Emails | ## Read and classify incoming mail<br><br>Every email is handled one at a time and sorted into a new question, a design answer, or something to review by hand. |
| Daily RFI Sweep 8am | n8n-nodes-base.scheduleTrigger | Triggers daily at 8 AM | None | Sweep Settings | ## Sweep open RFIs each morning<br><br>Works out how late each open RFI is and picks the next action. |
| Sweep Settings | n8n-nodes-base.set | Sets sweep configuration variables | Daily RFI Sweep 8am | Load Open RFIs | ## Sweep open RFIs each morning<br><br>Works out how late each open RFI is and picks the next action. |
| Load Open RFIs | n8n-nodes-base.dataTable | Loads all open RFIs | Sweep Settings | Age and Route Open RFIs, Build the Open RFI Digest | ## Sweep open RFIs each morning<br><br>Works out how late each open RFI is and picks the next action. |
| Age and Route Open RFIs | n8n-nodes-base.code | Calculates lateness and actions | Load Open RFIs | Route by RFI Age | ## Sweep open RFIs each morning<br><br>Works out how late each open RFI is and picks the next action. |
| Route by RFI Age | n8n-nodes-base.switch | Routes by chase or escalate action | Age and Route Open RFIs | Remind the Design Team, Escalate to Project Manager | ## Chase and escalate<br><br>Reminds the design team when an answer is due and escalates the badly overdue ones to the project manager. |
| Remind the Design Team | n8n-nodes-base.gmail | Emails reminder to design team | Route by RFI Age | None | ## Chase and escalate<br><br>Reminds the design team when an answer is due and escalates the badly overdue ones to the project manager. |
| Escalate to Project Manager | n8n-nodes-base.gmail | Emails escalation notice to PM | Route by RFI Age | None | ## Chase and escalate<br><br>Reminds the design team when an answer is due and escalates the badly overdue ones to the project manager. |
| Build the Open RFI Digest | n8n-nodes-base.code | Compiles HTML digest by project | Load Open RFIs | Email the Open RFI Digest | ## Email the daily digest<br><br>Sends each project manager a list of open RFIs with the late ones highlighted. |
| Email the Open RFI Digest | n8n-nodes-base.gmail | Emails open RFI digest to PM | Build the Open RFI Digest | None | ## Email the daily digest<br><br>Sends each project manager a list of open RFIs with the late ones highlighted. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n.

#### Step 1: Database Setup
1. Create a **Manual Trigger** node (`Run Once: Set Up RFI Tables`).
2. Create a **Data Table** node (`Create RFI Projects Table`): Set resource to `table`, operation to `create`, table name to `rfi_projects`, and add columns: `project_code` (string), `project_name` (string), `design_team_email` (string), `pm_email` (string), `response_days` (number), `status` (string). Connect `Run Once` to this node.
3. Create a **Data Table** node (`Create RFI Log Table`): Set resource to `table`, operation to `create`, table name to `rfi_log`, execute once, and add columns: `rfi_number`, `project_code`, `project_name`, `trade`, `asked_by_company`, `asked_by_email`, `subject`, `question`, `spec_section`, `drawing_ref`, `status`, `date_received`, `date_due`, `date_answered`, `answer`, `cost_impact`, `schedule_impact`, `design_team_email`, `pm_email`, `duplicate_of`, `source_message_id` (all strings). Connect previous table node to this.
4. Create a **Data Table** node (`Add Sample Project`): Set operation to create record, data table ID to `rfi_projects`, mapping mode to define below with sample values (`RMOB-2026`, Riverside Medical Office Building, user@example.com, user@example.com, 7, open). Execute once. Connect from log table creation.

#### Step 2: Email Trigger & Intake Settings
1. Create a **Gmail Trigger** node (`When an RFI Email Arrives`): Configure filters with query `subject:(RFI OR question OR clarification OR information) -from:me`, download attachments enabled with prefix `attachment_`, and polling interval set to every minute. Connect Gmail OAuth2 credentials.
2. Create a **Set** node (`RFI Intake Settings`): Assign variables `company_name` (`Your Construction Co.`), `project_team_name` (`Project Team`), `review_inbox` (`user@example.com`), `min_match_confidence` (`0.7`), and `duplicate_threshold` (`0.45`). Include other fields. Connect trigger to this node.
3. Create a **Split In Batches** node (`Loop Over RFI Emails`). Connect intake settings to this node.
4. Create an **Extract From File** node (`Extract Attached PDF Text`): Set operation to `pdf`, binary property name `attachment_0`, join pages enabled, and error handling set to continue regular output. Connect batch loop item output to this node.

#### Step 3: AI Agent & Tools Setup
1. Create an **OpenAI Chat Model** node (`OpenAI RFI Model`): Select model `gpt-5-mini` and configure OpenAI API credentials.
2. Create a **Data Table Tool** node (`Look Up Active Projects`): Set table name to `rfi_projects`, operation `get`, returning all rows filtered by `status` equals `open`.
3. Create a **Structured Output Parser** node (`RFI Fields Parser`): Provide the JSON schema example matching fields (`email_type`, `project_code`, `match_confidence`, `asker_company`, `asker_email`, `subject`, `question`, `spec_section`, `drawing_ref`, `trade`, `rfi_number`, `answer`, `has_cost_impact`, `has_schedule_impact`, `reasoning`).
4. Create an **AI Agent** node (`Read and Classify the RFI`): Set max tries to 3, configure system prompt instructing classification into `new_question`, `design_answer`, or `other`, and set prompt expression referencing email subject, sender, body, and attachment text. Connect model, data table tool, and output parser to this agent. Connect PDF extractor output to the agent.

#### Step 4: Routing & Project Lookup
1. Create a **Switch** node (`Route by Email Type`): Configure Rule 1 (`New question` where `email_type == 'new_question'`, project code length > 0, confidence >= minimum threshold) and Rule 2 (`Design answer` where `email_type == 'design_answer'`, RFI number length > 0), with fallback `Needs review`. Connect AI agent output here.
2. Create a **Data Table** node (`Get Matched Project`): Operation `get`, limit 1, filter `project_code` equals `={{ $json.output.project_code }}`. Connect Rule 1 output here.
3. Create a **Data Table** node (`Get RFIs for This Project`): Operation `get`, returning all rows from `rfi_log` filtered by `project_code`. Always output data. Connect project lookup here.

#### Step 5: Deduplication, Numbering & Logging (New Questions)
1. Create a **Code** node (`Number and Check for Duplicates`): Insert custom JavaScript to parse tokens, score wording overlap against answered RFIs, check message IDs, assign sequential RFI numbers, and compute due dates. Connect project RFIs lookup here.
2. Create an **If** node (`Already Logged?`): Condition `={{ $json.already_logged === true }}`. True branch loops back to `Loop Over RFI Emails`. False branch proceeds.
3. Create an **If** node (`Already Answered Before?`): Condition `={{ $json.is_duplicate === true }}`.
   - **True Branch:**
     - Create **Data Table** node (`Log the Duplicate RFI`): Create record in `rfi_log` mapping all fields.
     - Create **Gmail** node (`Reply With the Previous Answer`): Reply operation using original message ID, sending prior answer. Connect output back to `Loop Over RFI Emails`.
   - **False Branch:**
     - Create **Data Table** node (`Log the New RFI`): Create record in `rfi_log` mapping all fields.
     - Create **Gmail** node (`Send RFI to Design Team`): Send operation to design team email with RFI details.
     - Create **Gmail** node (`Acknowledge to Subcontractor`): Reply operation acknowledging receipt to subcontractor. Connect output back to `Loop Over RFI Emails`.

#### Step 6: Design Answer Processing & Approval
1. Create a **Data Table** node (`Get the Logged RFI`): Operation `get`, limit 1, filtered by `rfi_number`. Always output data. Connect Route by Email Type Rule 2 output here.
2. Create an **If** node (`Is the RFI Still Open?`): Check if ID > 0 and status equals `open`. False branch connects to `Flag Email for Review`.
3. Create an **If** node (`Cost or Schedule Impact?`): Check if either `has_cost_impact` or `has_schedule_impact` is true.
   - **True Branch (Impact Exists):**
     - Create **Gmail** node (`PM Approves the Answer`): Send and wait operation (`double` approval type) to PM email.
     - Create **If** node (`Answer Approved?`): Check if `{{ $json.data?.approved === true }}`.
       - True branch connects to `Send the Answer to the Sub`.
       - False branch connects to a **Data Table** node (`Hold the RFI for Review`) updating status to `pm_review`, then looping back to `Loop Over RFI Emails`.
   - **False Branch (No Impact):**
     - Connects directly to `Send the Answer to the Sub`.
4. Create **Gmail** node (`Send the Answer to the Sub`): Send operation to subcontractor email.
5. Create **Data Table** node (`Close the RFI`): Update operation on `rfi_log` matching `rfi_number`, setting status to `answered`, updating answer text, impacts, and answered date. Connect output back to `Loop Over RFI Emails`.
6. Create **Gmail** node (`Flag Email for Review`): Send operation to review inbox for unmatchable items. Connect output back to `Loop Over RFI Emails`.

#### Step 7: Daily Sweep, Escalation & Digest
1. Create a **Schedule Trigger** node (`Daily RFI Sweep 8am`): Trigger at hour 8.
2. Create a **Set** node (`Sweep Settings`): Assign `company_name`, `project_team_name`, and `escalate_after_days` (`2`).
3. Create a **Data Table** node (`Load Open RFIs`): Operation `get`, returning all rows where status equals `open`.
4. Branch 1 (Chase & Escalate):
   - Create **Code** node (`Age and Route Open RFIs`): Calculate days overdue and action (`wait`, `chase`, `escalate`).
   - Create **Switch** node (`Route by RFI Age`): Route by action.
     - Chase output connects to **Gmail** node (`Remind the Design Team`) sending reminders to design team email.
     - Escalate output connects to **Gmail** node (`Escalate to Project Manager`) sending escalation notices to PM email.
5. Branch 2 (Daily Digest):
   - Create **Code** node (`Build the Open RFI Digest`): Group open RFIs by project code, compute lateness, and generate HTML digest tables.
   - Create **Gmail** node (`Email the Open RFI Digest`): Send operation to PM email containing the formatted HTML digest.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Keep the Gmail search tight | The workflow replies to senders automatically; ensure the Gmail query excludes your own mail and unrelated correspondence. |
| Run initial database setup | Execute the manual trigger node once to create the required Data Tables (`rfi_projects` and `rfi_log`) before activating the workflow. |
| Connect required credentials | Ensure valid Gmail OAuth2 credentials and OpenAI API credentials are created and attached to all respective nodes. |