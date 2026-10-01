Run win-back drip email campaigns with Google Sheets and SMTP

https://n8nworkflows.xyz/workflows/run-win-back-drip-email-campaigns-with-google-sheets-and-smtp-19787


# Run win-back drip email campaigns with Google Sheets and SMTP

### 1. Workflow Overview

This workflow automates a win-back email drip campaign for inactive or "cold" leads stored in a Google Sheets tracker. It operates on two distinct schedules: a daily check to process and send staged reactivation emails, and a weekly rollup to report overall campaign statistics to the firm. 

The execution logic is divided into the following functional blocks:
- **1.1 Daily Drip Trigger & Ingestion:** Triggers every morning to read the tracker data.
- **1.2 Drip Calculation & Content Assembly:** Computes which leads are due for the next sequence stage and builds personalized email payloads.
- **1.3 Compliance & Conditional Dispatch:** Evaluates outgoing emails through an external compliance sub-workflow, sending approved messages via SMTP and updating the tracker.
- **1.4 Weekly Rollup & Reporting:** Triggers every Monday to aggregate lead status metrics and dispatch a summary report via SMTP.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Daily Drip Trigger & Ingestion
- **Overview:** Initiates the daily win-back sequence at 9:00 AM and pulls the current lead dataset from Google Sheets.
- **Nodes Involved:** 
  - `Every Morning at 9am`
  - `Read Win-Back Tracker Sheet`

- **Node Details:**
  - **Every Morning at 9am**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Acts as the cron-based entry point for the daily drip sequence.
    - *Configuration:* Cron expression set to `0 9 * * *` (Daily at 9:00 AM).
    - *Connections:* Output connects to `Read Win-Back Tracker Sheet`.
    - *Edge Cases:* Server timezone misconfigurations may cause emails to send at unintended local hours.
  - **Read Win-Back Tracker Sheet**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Reads rows from the designated tracking spreadsheet.
    - *Configuration:* Operation: `read`, Sheet Name: `Win-Back Tracker`, Document ID retrieved from environment variable `={{ $vars.WIN_BACK_TRACKER_SHEET_ID }}`.
    - *Connections:* Input from `Every Morning at 9am`; output to `Calculate Due Drip Stage`.
    - *Edge Cases:* Authentication expiration or missing spreadsheet columns (`lead_id`, `email`, `status`, `marked_cold_at`, `last_stage_sent`) will throw execution errors.

---

#### Block 1.2: Drip Calculation & Content Assembly
- **Overview:** Evaluates lead activity timelines against configured campaign stages and generates formatted HTML/text email content.
- **Nodes Involved:**
  - `Calculate Due Drip Stage`
  - `Build Win-Back Drip Email Content`

- **Node Details:**
  - **Calculate Due Drip Stage**
    - *Type & Role:* `n8n-nodes-base.code` (Code Node) — JavaScript execution block that filters active leads and determines progression through the drip stages.
    - *Configuration:* Parses `WIN_BACK_TRACKER_SHEET_ID` and `WIN_BACK_DRIP_STAGES` variables (default: `'0|check-in;14|resources;30|last-call'`). Computes elapsed days since `marked_cold_at` and assigns the next applicable stage label.
    - *Input/Output:* Ingests multiple lead items; outputs filtered items containing `due_stage_label` and `days_since_cold`.
    - *Edge Cases:* Invalid date strings in `marked_cold_at` will fail the date parsing check and skip the lead.
  - **Build Win-Back Drip Email Content**
    - *Type & Role:* `n8n-nodes-base.code` (Code Node) — Generates localized subject lines, plain text bodies, and styled HTML email structures.
    - *Configuration:* Uses stage-keyed mapping objects to generate context-aware messages incorporating the lead's name and practice area.
    - *Input/Output:* Ingests evaluated lead items; outputs structured objects containing `recipient`, `subject`, `message`, and `email_html`.
    - *Edge Cases:* Unescaped special characters in lead names or practice areas are handled via an internal `escapeHtml` utility function to prevent XSS or formatting distortion.

---

#### Block 1.3: Compliance & Conditional Dispatch
- **Overview:** Validates messages against a compliance guardrail sub-workflow, conditionally sends approved emails via SMTP, and logs sent stages back to Google Sheets.
- **Nodes Involved:**
  - `Execute Compliance Workflow`
  - `If Drip Email Is Approved`
  - `Send Win-Back Drip Email`
  - `Update Tracker With Sent Stage`
  - `Skip Drip Email Lead Opted Out`

- **Node Details:**
  - **Execute Compliance Workflow**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Execute Workflow) — Invokes an external guardrail sub-workflow for message review.
    - *Configuration:* Target workflow ID dynamic expression `={{ $vars.GUARDRAIL_WORKFLOW_ID }}`. Passes parameters: `channel`, `message`, `recipient`, and `template_id`.
    - *Sub-Workflow Reference:* Invokes the compliance guardrail workflow, expecting an `approved` boolean response.
    - *Edge Cases:* Sub-workflow timeout or failure will halt the individual lead's drip progression.
  - **If Drip Email Is Approved**
    - *Type & Role:* `n8n-nodes-base.if` (If Node) — Branching logic based on the compliance evaluation.
    - *Configuration:* Evaluates `={{ $json.approved }}` equals `true`.
    - *Connections:* True branch goes to `Send Win-Back Drip Email`; False branch goes to `Skip Drip Email Lead Opted Out`.
  - **Send Win-Back Drip Email**
    - *Type & Role:* `n8n-nodes-base.emailSend` (Email Send) — Dispatches the approved email via SMTP.
    - *Configuration:* Uses expression references pointing back to the preceding content node (`$('Build Win-Back Drip Email Content').item.json`). Sender address mapped to `={{ $vars.FIRM_FROM_EMAIL }}`.
    - *Credentials:* SMTP Account ID `1`.
    - *Edge Cases:* SMTP connection drops, authentication failures, or invalid recipient formats will trigger node errors.
  - **Update Tracker With Sent Stage**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Updates tracking data in Google Sheets upon successful email delivery.
    - *Configuration:* Operation: `appendOrUpdate`, matching columns: `lead_id`. Updates `last_stage_sent` and `last_sent_at`.
    - *Connections:* Input from `Send Win-Back Drip Email`.
  - **Skip Drip Email Lead Opted Out**
    - *Type & Role:* `n8n-nodes-base.noOp` (No Operation) — Terminal placeholder for unapproved or opted-out leads.
    - *Connections:* Input from the false branch of `If Drip Email Is Approved`.

---

#### Block 1.4: Weekly Rollup & Reporting
- **Overview:** Executes a weekly analysis of the tracking sheet and sends an administrative summary report to the firm.
- **Nodes Involved:**
  - `Every Monday at 8am`
  - `Read Complete Win-Back Tracker`
  - `Summarize Weekly Win-Back Status`
  - `Build Weekly Win-Back Email`
  - `Send Weekly Win-Back Email`

- **Node Details:**
  - **Every Monday at 8am**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Cron-based entry point for weekly reporting.
    - *Configuration:* Cron expression set to `0 8 * * 1` (Every Monday at 8:00 AM).
  - **Read Complete Win-Back Tracker**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Reads all entries in the tracker regardless of status.
    - *Configuration:* Operation: `read`, Sheet Name: `Win-Back Tracker`, Document ID: `={{ $vars.WIN_BACK_TRACKER_SHEET_ID }}`.
  - **Summarize Weekly Win-Back Status**
    - *Type & Role:* `n8n-nodes-base.code` (Code Node) — Aggregates counts for active, reactivated, and unsubscribed statuses, and flags completed sequences.
    - *Output:* Returns a single summary object containing total lead counts, breakdown metrics, and completed sequence tallies.
  - **Build Weekly Win-Back Email**
    - *Type & Role:* `n8n-nodes-base.code` (Code Node) — Compiles administrative summary text and responsive HTML report layouts.
    - *Configuration:* Sets recipient address to `={{ $vars.FIRM_EMAIL }}`.
  - **Send Weekly Win-Back Email**
    - *Type & Role:* `n8n-nodes-base.emailSend` (Email Send) — Delivers the weekly status report via SMTP.
    - *Credentials:* SMTP Account ID `1`.
    - *Edge Cases:* Failure in SMTP transport prevents administrative visibility into weekly campaign performance.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Win-Back / Reactivation Drip<br><br>### How it works<br><br>1. The workflow triggers a daily check to identify leads due for win-back drip emails and builds personalized emails.<br>2. It checks email compliance and conditionally sends approved emails while updating the tracker or skips opted-out leads.<br>3. Separately, it runs a weekly summary rollup to analyze the overall campaign status of active and inactive leads.<br>4. The weekly summary email is built and sent out to report on the win-back campaign's performance.<br><br><br>### Setup steps<br><br>- - [ ] Configure Google Sheets credentials and link sheets containing the win-back tracker data.<br>- - [ ] Set up email credentials for sending drip and summary emails.<br>- - [ ] Customize the compliance check workflow and link it appropriately.<br>- - [ ] Schedule triggers appropriately for daily drip checks and weekly rollup summaries. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual grouping for trigger and ingestion | None | None | ## Daily drip trigger and fetch<br><br>Scheduled trigger to start the daily win-back drip process by fetching the current win-back tracker data. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual grouping for drip calculation | None | None | ## Determine and build drip emails<br><br>Processes leads to compute which drip stage they are due for and builds personalized win-back drip emails. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual grouping for compliance and conditional sending | None | None | ## Compliance and conditional sending<br><br>Calls a compliance sub-workflow and decides to send drip emails or skip leads who opted out. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual grouping for weekly trigger | None | None | ## Weekly rollup trigger and fetch<br><br>Scheduled trigger to start the weekly win-back campaign rollup summary and fetch the full tracker data. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual grouping for weekly summary email | None | None | ## Compute and send weekly summary<br><br>Computes campaign summary based on tracker data, builds the weekly summary email, and sends it out. |
| Every Morning at 9am | n8n-nodes-base.scheduleTrigger | Daily execution trigger | None | Read Win-Back Tracker Sheet | |
| Read Win-Back Tracker Sheet | n8n-nodes-base.googleSheets | Fetches lead tracker data | Every Morning at 9am | Calculate Due Drip Stage | |
| Calculate Due Drip Stage | n8n-nodes-base.code | Computes drip progression stage | Read Win-Back Tracker Sheet | Build Win-Back Drip Email Content | |
| Build Win-Back Drip Email Content | n8n-nodes-base.code | Generates personalized email bodies and HTML | Calculate Due Drip Stage | Execute Compliance Workflow | |
| Execute Compliance Workflow | n8n-nodes-base.executeWorkflow | Validates message via guardrail sub-workflow | Build Win-Back Drip Email Content | If Drip Email Is Approved | |
| If Drip Email Is Approved | n8n-nodes-base.if | Evaluates compliance approval status | Execute Compliance Workflow | Send Win-Back Drip Email,<br>Skip Drip Email Lead Opted Out | |
| Send Win-Back Drip Email | n8n-nodes-base.emailSend | Sends approved drip email via SMTP | If Drip Email Is Approved | Update Tracker With Sent Stage | |
| Update Tracker With Sent Stage | n8n-nodes-base.googleSheets | Logs sent stage and timestamp in sheet | Send Win-Back Drip Email | None | |
| Skip Drip Email Lead Opted Out | n8n-nodes-base.noOp | Fallback endpoint for opted-out leads | If Drip Email Is Approved | None | |
| Every Monday at 8am | n8n-nodes-base.scheduleTrigger | Weekly execution trigger | None | Read Complete Win-Back Tracker | |
| Read Complete Win-Back Tracker | n8n-nodes-base.googleSheets | Reads full tracker dataset | Every Monday at 8am | Summarize Weekly Win-Back Status | |
| Summarize Weekly Win-Back Status | n8n-nodes-base.code | Aggregates status counts and completion metrics | Read Complete Win-Back Tracker | Build Weekly Win-Back Email | |
| Build Weekly Win-Back Email | n8n-nodes-base.code | Compiles weekly summary report format | Summarize Weekly Win-Back Status | Send Weekly Win-Back Email | |
| Send Weekly Win-Back Email | n8n-nodes-base.emailSend | Sends weekly summary report via SMTP | Build Weekly Win-Back Email | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Workflow:** Open n8n, create a new workflow, and name it `Win-Back / Reactivation Drip`.
2. **Add Daily Trigger:** Create a **Schedule Trigger** node named `Every Morning at 9am`. Set the interval rule to Cron Expression `0 9 * * *`.
3. **Add Daily Sheet Reader:** Create a **Google Sheets** node named `Read Win-Back Tracker Sheet`. Set operation to `Read`, sheet name to `Win-Back Tracker`, and Document ID to `={{ $vars.WIN_BACK_TRACKER_SHEET_ID }}`. Connect `Every Morning at 9am` to this node.
4. **Add Stage Calculation Code:** Create a **Code** node named `Calculate Due Drip Stage`. Insert the JavaScript snippet that filters active leads and calculates day offsets based on `WIN_BACK_DRIP_STAGES`. Connect `Read Win-Back Tracker Sheet` to this node.
5. **Add Content Builder Code:** Create a **Code** node named `Build Win-Back Drip Email Content`. Insert the templating script that generates subject lines, plain text, and HTML payloads. Connect `Calculate Due Drip Stage` to this node.
6. **Add Compliance Execution Node:** Create an **Execute Workflow** node named `Execute Compliance Workflow`. Set the workflow ID parameter to `={{ $vars.GUARDRAIL_WORKFLOW_ID }}` and map input variables (`channel`, `message`, `recipient`, `template_id`). Connect `Build Win-Back Drip Email Content` to this node.
7. **Add Approval Condition Node:** Create an **If** node named `If Drip Email Is Approved`. Configure a condition where `={{ $json.approved }}` equals `true`. Connect `Execute Compliance Workflow` to this node.
8. **Add Email Dispatcher:** Create an **Email Send** (SMTP) node named `Send Win-Back Drip Email`. Configure subject, HTML, and text fields using references to the content node items. Set `fromEmail` to `={{ $vars.FIRM_FROM_EMAIL }}` and select your SMTP credential. Connect the true output of `If Drip Email Is Approved` to this node.
9. **Add Sheet Updater:** Create a **Google Sheets** node named `Update Tracker With Sent Stage`. Set operation to `Append or Update`, matching columns to `lead_id`, and update fields for `last_stage_sent` and `last_sent_at`. Connect `Send Win-Back Drip Email` to this node.
10. **Add Opt-Out Fallback:** Create a **No Operation** node named `Skip Drip Email Lead Opted Out`. Connect the false output of `If Drip Email Is Approved` to this node.
11. **Add Weekly Trigger:** Create a **Schedule Trigger** node named `Every Monday at 8am`. Set the interval rule to Cron Expression `0 8 * * 1`.
12. **Add Weekly Sheet Reader:** Create a **Google Sheets** node named `Read Complete Win-Back Tracker`. Set operation to `Read`, sheet name to `Win-Back Tracker`, and Document ID to `={{ $vars.WIN_BACK_TRACKER_SHEET_ID }}`. Connect `Every Monday at 8am` to this node.
13. **Add Weekly Summarizer Code:** Create a **Code** node named `Summarize Weekly Win-Back Status`. Insert the aggregation script that calculates active, reactivated, unsubscribed, and completed sequence totals. Connect `Read Complete Win-Back Tracker` to this node.
14. **Add Weekly Email Builder:** Create a **Code** node named `Build Weekly Win-Back Email`. Insert the script that formats the administrative weekly HTML report. Connect `Summarize Weekly Win-Back Status` to this node.
15. **Add Weekly Email Dispatcher:** Create an **Email Send** (SMTP) node named `Send Weekly Win-Back Email`. Set `toEmail` to `={{ $json.recipient }}`, `fromEmail` to `={{ $vars.FIRM_FROM_EMAIL }}`, and configure credential bindings. Connect `Build Weekly Win-Back Email` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Compliance Guardrail Integration | Relies on an external sub-workflow referenced via `GUARDRAIL_WORKFLOW_ID` to enforce marketing compliance and opt-out checks. |
| Operational Variables Required | Ensure environment variables `WIN_BACK_TRACKER_SHEET_ID`, `WIN_BACK_DRIP_STAGES`, `GUARDRAIL_WORKFLOW_ID`, `FIRM_NAME`, `FIRM_FROM_EMAIL`, and `FIRM_EMAIL` are configured in your n8n instance. |
| Lead State Management | Lead statuses (`active`, `reactivated`, `unsubscribed`) and lead additions must be managed directly within the Google Sheets tracker. |