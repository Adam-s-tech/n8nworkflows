Triage Gmail support inquiries with Google Gemini, Calendar, Sheets, and Discord

https://n8nworkflows.xyz/workflows/triage-gmail-support-inquiries-with-google-gemini--calendar--sheets--and-discord-19683


# Triage Gmail support inquiries with Google Gemini, Calendar, Sheets, and Discord

### 1. Workflow Overview

This workflow automates the triage, logging, and scheduling of incoming customer support emails received via Gmail. It parses inquiry content using Google Gemini, evaluates urgency to calculate dynamic Service Level Agreement (SLA) deadlines, creates calendar blocks, logs ticket metadata in a spreadsheet, drafts email responses, and dispatches real-time alerts to Discord for high-priority issues.

The workflow logic is categorized into four functional blocks:

- **1.1 Input Reception & AI Parsing:** Polls Gmail for incoming unread support emails and passes their content to Google Gemini along with a structured output parser to extract triage metadata (category, priority, summary, and a reply draft).
- **1.2 SLA & Deadline Calculation:** Normalizes raw fields and executes custom JavaScript logic to compute a target deadline based on the assigned priority level.
- **1.3 Scheduling, Logging & Drafting:** Provisions a Google Calendar event for the SLA window, appends the ticket details to a Google Sheets worksheet, and creates a corresponding draft reply inside the original Gmail thread.
- **1.4 Urgent Alerting:** Evaluates the priority level and routes critical incidents to a Discord webhook channel if flagged as "High".

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & AI Parsing
- **Overview:** This block watches for new incoming messages matching specific criteria, feeds the email metadata into an LLM chain, and enforces structured JSON output using Gemini.
- **Nodes Involved:** 
  - `When Email Arrives`
  - `Language Model Processing`
  - `Gemini AI Conversation`
  - `Parse Structured Output`

- **Node Details:**
  - **When Email Arrives**
    - *Type & Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger). Polls the connected Gmail account for unread messages.
    - *Configuration:* Filters set to look for unread status and query string `subject:"Inquiry Received"`. Polling interval set to run every minute.
    - *Expressions:* None.
    - *Connections:* Input: None (Trigger). Output: `Language Model Processing`.
    - *Edge Cases / Failures:* Authentication revocation or missing mailbox permissions will pause execution.

  - **Language Model Processing**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI / Action). Executes the prompt combining sender, subject, and body against the configured LLM and parser.
    - *Configuration:* Prompt type set to define. Uses input variables extracted from the trigger node.
    - *Key Expressions:* 
      ```javascript
      =Please analyze the following incoming customer inquiry email and return the result strictly in the specified JSON schema.
      [Sender]: {{ $json.from?.text || $json.from?.value?.[0]?.address || $json.from || '' }}
      [Subject]: {{ $json.subject || '' }}
      [Body]: {{ $json.text || $json.snippet || '' }}
      ```
    - *Connections:* Input: `When Email Arrives`. AI Model input from `Gemini AI Conversation`. Output parser linked from `Parse Structured Output`. Output: `Calculate SLA & Deadlines`.
    - *Edge Cases / Failures:* API rate limits, invalid schema responses, or empty email bodies causing parse errors.

  - **Gemini AI Conversation**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Model Provider). Provides the underlying Google Gemini chat model configuration.
    - *Configuration:* Default options. Requires active Google AI credentials.
    - *Connections:* Output (`ai_languageModel`): Connected to `Language Model Processing`.

  - **Parse Structured Output**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Parser). Enforces a strict JSON schema on the LLM response.
    - *Configuration:* Manual schema enforcing four properties: `category` (string: Billing, Technical, Account, Other), `priority` (enum: High, Medium, Low), `summary` (string), and `reply_draft` (string). All four fields are marked as required.
    - *Connections:* Output (`ai_outputParser`): Connected to `Language Model Processing`.

---

#### 2.2 SLA & Deadline Calculation
- **Overview:** Normalizes unstructured or varied email address objects, processes structured model outputs, and calculates dynamic resolution target timestamps based on urgency.
- **Nodes Involved:**
  - `Calculate SLA & Deadlines`

- **Node Details:**
  - **Calculate SLA & Deadlines**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Executes custom JavaScript to derive SLA dates, format human-readable deadlines, and assemble a consolidated item payload.
    - *Configuration:* JavaScript execution environment (v2).
    - *Key Expressions / Code Logic:* Implements dynamic SLA branching:
      - **High:** Current time + 4 hours.
      - **Medium:** Next calendar day at 18:00.
      - **Low:** 3 calendar days out at 18:00.
    - *Connections:* Input: `Language Model Processing`. Output: Parallel execution to `Add to Google Calendar` and `Check Priority Level`.
    - *Edge Cases / Failures:* Script failure if upstream properties are completely missing or malformed; mitigated by fallback default assignment (`Medium`, `General Inquiry`).

---

#### 2.3 Scheduling, Logging & Drafting
- **Overview:** Takes the processed ticket details to create a calendar block, record row data in a tracking sheet, and prepare a draft email response in Gmail.
- **Nodes Involved:**
  - `Add to Google Calendar`
  - `Append Details to Sheets`
  - `Create Gmail Draft`

- **Node Details:**
  - **Add to Google Calendar**
    - *Type & Technical Role:* `n8n-nodes-base.googleCalendar` (Action). Creates a calendar event representing the SLA window.
    - *Configuration:* Operation set to create an event on the primary calendar.
    - *Key Expressions:* 
      - Start: `={{ $json.startTime }}`
      - End: `={{ $json.deadlineTime }}`
      - Summary: `={{ $json.taskTitle }}`
      - Description includes summary, priority, deadline, sender, subject, and thread ID.
    - *Connections:* Input: `Calculate SLA & Deadlines`. Output: `Append Details to Sheets`.
    - *Edge Cases / Failures:* Calendar permission scopes or timezone mismatches.

  - **Append Details to Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action). Appends a row of ticket metrics to a specified Google Sheet.
    - *Configuration:* Operation set to append rows. Mapping mode set to define below.
    - *Key Expressions:* Maps fields such as `sender`, `subject`, `summary`, `category`, `deadline`, `priority`, received timestamp (`$now.toFormat('yyyy-MM-dd HH:mm:ss')`), and calendar event URL (`={{ $json.htmlLink || '' }}`). Note: References upstream data using explicit node traversal (`$('Calculate SLA & Deadlines').item.json...`).
    - *Connections:* Input: `Add to Google Calendar`. Output: `Create Gmail Draft`.
    - *Edge Cases / Failures:* Incorrect document ID, sheet name mismatch, or missing column headers in the destination sheet.

  - **Create Gmail Draft**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Action). Generates an email draft inside the original thread.
    - *Configuration:* Resource set to `draft`. 
    - *Key Expressions:* Message body bound to `={{ $('Calculate SLA & Deadlines').item.json.replyDraft }}` and thread ID bound to `={{ $('Calculate SLA & Deadlines').item.json.threadId }}`.
    - *Connections:* Input: `Append Details to Sheets`. Output: None (Terminal node in this branch).
    - *Edge Cases / Failures:* Invalid thread ID reference or missing Gmail API scopes.

---

#### 2.4 Urgent Alerting
- **Overview:** Evaluates whether an inquiry's priority meets the threshold for urgent interruption and fires a formatted notification to Discord if criteria are met.
- **Nodes Involved:**
  - `Check Priority Level`
  - `Send Urgent Discord Alert`

- **Node Details:**
  - **Check Priority Level**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates conditional logic against incoming priority strings.
    - *Configuration:* Condition set to check if `={{ $('Calculate SLA & Deadlines').item.json.priority }}` equals string `High`.
    - *Connections:* Input: `Calculate SLA & Deadlines`. Output (Main branch): `Send Urgent Discord Alert`.
    - *Edge Cases / Failures:* Case sensitivity mismatches in priority strings.

  - **Send Urgent Discord Alert**
    - *Type & Technical Role:* `n8n-nodes-base.discord` (Action). Sends a rich-text notification via webhook.
    - *Configuration:* Authentication via webhook URL.
    - *Key Expressions:* Formats a markdown-based message including deadline, category, sender, subject, and summary.
    - *Connections:* Input: `Check Priority Level`. Output: None (Terminal node).
    - *Edge Cases / Failures:* Invalid webhook URL or rate limiting by Discord.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Guide | None | None | ## Customer Support Triage & SLA Calendar Dispatcher<br><br>Automate incoming customer inquiry triage from Gmail with Google Gemini. The system classifies requests, calculates dynamic SLA deadlines based on urgency, schedules follow-up blocks in Google Calendar, logs ticket records in Google Sheets, generates AI reply drafts, and sends instant Discord alerts for high-priority incidents.<br><br>Who's it for: Customer support leads, IT service desks, and operations teams needing automated triage and SLA tracking without manual data entry.<br><br>### How it works<br>1. When Email Arrives polls unread emails with the subject "Inquiry Received".<br>2. Language Model Processing and Gemini parse the email body into structured JSON (priority, category, summary, draft).<br>3. Calculate SLA & Deadlines computes resolution target times (High: 4h, Medium: 1 day, Low: 3 days).<br>4. Add to Google Calendar creates a deadline block, Append Details to Sheets logs the record, and Create Gmail Draft saves a reply.<br>5. If priority is High, Check Priority Level routes a critical alert to Send Urgent Discord Alert.<br><br>### Setup<br>1. Connect your Gmail, Google Gemini, Google Calendar, and Google Sheets credentials.<br>2. Prepare a Google Sheet titled `form` with headers: received_at, priority, category, sender, subject, summary, deadline, calendar_event_url.<br>3. Configure your Discord Webhook URL in Send Urgent Discord Alert.<br><br>### Customization tips<br>- Adjust dynamic SLA hours directly in the JavaScript code inside Calculate SLA & Deadlines.<br>- Modify the prompt in Language Model Processing to adapt categorization tags to your domain. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section 1 Documentation | None | None | ## 1. Receive and classify email<br><br>Captures new support emails from Gmail and uses the LLM chain, Gemini chat model, and structured parser to extract triage details in a predictable format. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section 2 Documentation | None | None | ## 2. Calculate SLA deadlines<br><br>Uses custom code to convert the structured triage output into SLA dates, deadlines, and priority metadata before splitting into downstream actions. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section 3 Documentation | None | None | ## 3. Schedule and draft response<br><br>Creates a Google Calendar item for the SLA follow-up, logs the triaged case in Google Sheets, and prepares a Gmail draft response for the support team. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section 4 Documentation | None | None | ## 4. Send urgent alerts<br><br>Checks whether the calculated priority meets the urgent criteria and sends a Discord notification for cases that need immediate attention. |
| When Email Arrives | n8n-nodes-base.gmailTrigger | Polls unread inquiry emails | None | Language Model Processing | ## 1. Receive and classify email<br><br>Captures new support emails from Gmail and uses the LLM chain, Gemini chat model, and structured parser to extract triage details in a predictable format. |
| Language Model Processing | @n8n/n8n-nodes-langchain.chainLlm | Processes email text via LLM | When Email Arrives | Calculate SLA & Deadlines | ## 1. Receive and classify email<br><br>Captures new support emails from Gmail and uses the LLM chain, Gemini chat model, and structured parser to extract triage details in a predictable format. |
| Gemini AI Conversation | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini model configuration | None | Language Model Processing | ## 1. Receive and classify email<br><br>Captures new support emails from Gmail and uses the LLM chain, Gemini chat model, and structured parser to extract triage details in a predictable format. |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces strict JSON schema | None | Language Model Processing | ## 1. Receive and classify email<br><br>Captures new support emails from Gmail and uses the LLM chain, Gemini chat model, and structured parser to extract triage details in a predictable format. |
| Calculate SLA & Deadlines | n8n-nodes-base.code | Computes SLA and normalizes data | Language Model Processing | Add to Google Calendar, Check Priority Level | ## 2. Calculate SLA deadlines<br><br>Uses custom code to convert the structured triage output into SLA dates, deadlines, and priority metadata before splitting into downstream actions. |
| Add to Google Calendar | n8n-nodes-base.googleCalendar | Creates SLA calendar event | Calculate SLA & Deadlines | Append Details to Sheets | ## 3. Schedule and draft response<br><br>Creates a Google Calendar item for the SLA follow-up, logs the triaged case in Google Sheets, and prepares a Gmail draft response for the support team. |
| Append Details to Sheets | n8n-nodes-base.googleSheets | Logs ticket data to sheet | Add to Google Calendar | Create Gmail Draft | ## 3. Schedule and draft response<br><br>Creates a Google Calendar item for the SLA follow-up, logs the triaged case in Google Sheets, and prepares a Gmail draft response for the support team. |
| Create Gmail Draft | n8n-nodes-base.gmail | Drafts reply in email thread | Append Details to Sheets | None | ## 3. Schedule and draft response<br><br>Creates a Google Calendar item for the SLA follow-up, logs the triaged case in Google Sheets, and prepares a Gmail draft response for the support team. |
| Check Priority Level | n8n-nodes-base.if | Evaluates high priority threshold | Calculate SLA & Deadlines | Send Urgent Discord Alert | ## 4. Send urgent alerts<br><br>Checks whether the calculated priority meets the urgent criteria and sends a Discord notification for cases that need immediate attention. |
| Send Urgent Discord Alert | n8n-nodes-base.discord | Sends webhook alert to Discord | Check Priority Level | None | ## 4. Send urgent alerts<br><br>Checks whether the calculated priority meets the urgent criteria and sends a Discord notification for cases that need immediate attention. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`).
   - Name it `When Email Arrives`.
   - Set polling intervals to every minute.
   - Configure filters: Query string set to `subject:"Inquiry Received"` and Read Status set to `unread`.
   - Connect required Gmail credentials.

2. **Set Up the AI Components:**
   - Add a **Google Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) and name it `Gemini AI Conversation`. Connect Google AI credentials.
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) and name it `Parse Structured Output`. Configure the manual JSON schema with properties `category`, `priority` (Enum: High, Medium, Low), `summary`, and `reply_draft`. Ensure all four fields are added to the required list.
   - Add an **Advanced AI / LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) and name it `Language Model Processing`. Set prompt type to define and input the expression referencing the sender, subject, and body from `When Email Arrives`. Connect `Gemini AI Conversation` to its AI Model input and `Parse Structured Output` to its Output Parser input.
   - Connect `When Email Arrives` output to `Language Model Processing`.

3. **Add Data Transformation Code:**
   - Add a **Code** node (`n8n-nodes-base.code`) and name it `Calculate SLA & Deadlines`.
   - Paste the JavaScript code logic to extract the structured outputs, parse raw sender/subject fields, evaluate dynamic SLA timelines (4h for High, next day 18:00 for Medium, 3 days 18:00 for Low), and output the standardized JSON object containing deadlines, display strings, task titles, and thread IDs.
   - Connect `Language Model Processing` output to `Calculate SLA & Deadlines`.

4. **Build the Calendar and Logging Branch:**
   - Add a **Google Calendar** node (`n8n-nodes-base.googleCalendar`) named `Add to Google Calendar`. Set operation to create an event on the `primary` calendar. Map Start time to `{{ $json.startTime }}`, End time to `={{ $json.deadlineTime }}`, Summary to `={{ $json.taskTitle }}`, and populate the Description field with inquiry metadata. Connect Google Calendar credentials.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append Details to Sheets`. Set operation to append rows. Configure document ID and sheet name parameters. Map row columns (`sender`, `subject`, `summary`, `category`, `deadline`, `priority`, `received_at`, `calendar_event_url`) using expressions pointing back to `Calculate SLA & Deadlines` and the sheet output link (`{{ $json.htmlLink || '' }}`). Connect Google Sheets credentials.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Create Gmail Draft`. Set resource to `draft`, set subject to `=Re: {{ $('Calculate SLA & Deadlines').item.json.subject }}`, message body to `={{ $('Calculate SLA & Deadlines').item.json.replyDraft }}`, and options threadId to `={{ $('Calculate SLA & Deadlines').item.json.threadId }}`.
   - Connect execution order: `Calculate SLA & Deadlines` $\rightarrow$ `Add to Google Calendar` $\rightarrow$ `Append Details to Sheets` $\rightarrow$ `Create Gmail Draft`.

5. **Build the Urgent Alerting Branch:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Check Priority Level`. Configure condition: Left Value `={{ $('Calculate SLA & Deadlines').item.json.priority }}`, Operator `equals`, Right Value `High`.
   - Add a **Discord** node (`n8n-nodes-base.discord`) named `Send Urgent Discord Alert`. Set authentication to webhook, paste your webhook URL, and input the markdown-formatted alert content.
   - Connect execution order: `Calculate SLA & Deadlines` $\rightarrow$ `Check Priority Level` (Main branch output) $\rightarrow$ `Send Urgent Discord Alert`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Customer Support Triage & SLA Calendar Dispatcher template overview and architecture instructions. | Workflow template metadata. |