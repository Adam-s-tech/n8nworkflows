Send TDS, GST and ROC compliance reminders via WhatsApp, Sheets and GPT-4o-mini

https://n8nworkflows.xyz/workflows/send-tds--gst-and-roc-compliance-reminders-via-whatsapp--sheets-and-gpt-4o-mini-20031


# Send TDS, GST and ROC compliance reminders via WhatsApp, Sheets and GPT-4o-mini

### 1. Workflow Overview

This workflow automates statutory compliance deadline tracking and client communication for Indian Chartered Accountant (CA) firms across three distinct functional blocks. It utilizes Google Sheets as a central database, the WhatsApp Business API for outbound reminders and inbound client interaction, Google Drive for document storage, and OpenAI’s GPT-4o-mini model for automated document classification and checklist matching.

- **1.1 Deadline Generator Pipeline:** Executes bi-monthly (1st and 15th at 6:00 AM) to read active client rosters and existing tracker entries, calculates upcoming statutory deadlines (TDS, GST, ROC) for a rolling 60-day window, filters out duplicates, and appends new tasks to the Compliance Tracker.
- **1.2 Daily Reminder Pipeline:** Executes daily at 9:00 AM to scan open tasks, evaluate reminder intervals based on due dates and missing documents, compile personalized WhatsApp messages per client, dispatch them via Meta's Graph API, and update the tracker to prevent duplicate notifications.
- **1.3 Inbound Message Routing & Processing Pipeline:** Triggers instantly upon receiving an inbound WhatsApp message, resolves the sender's phone number against the client database, branches execution based on message type (text vs. file/media), downloads media attachments, uploads them to the client's designated Google Drive folder, uses GPT-4o-mini to reconcile documents against pending compliance tasks, updates checklist statuses, logs the operation, and responds automatically via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Deadline Generator

- **Overview:** Automatically scans active clients on the 1st and 15th of every month, computes statutory Indian tax and corporate deadlines (GST, TDS, ROC), and adds new records to the compliance tracker sheet while preventing duplicate insertions.
- **Nodes Involved:**
  - `Every 1st & 15th (6 AM)1`
  - `Read Clients1`
  - `Read Tracker (Generator)1`
  - `Generate Upcoming Deadlines1`
  - `Append New Tasks1`

- **Node Details:**
  - **Every 1st & 15th (6 AM)1**
    - *Type & Role:* Schedule Trigger node executing on a cron expression.
    - *Configuration:* Cron expression `0 6 1,15 * *` (runs at 6:00 AM on the 1st and 15th of every month).
    - *Connections:* Input: None; Output: `Read Clients1`.
    - *Failure Types:* None specific to the node; scheduling depends on the n8n instance worker status.
  - **Read Clients1**
    - *Type & Role:* Google Sheets node fetching the active client directory.
    - *Configuration:* Operation: `get` / `getAll`, Sheet Name: `Clients`, Document ID: `REPLACE_WITH_SHEET_ID`.
    - *Connections:* Input: `Every 1st & 15th (6 AM)1`; Output: `Read Tracker (Generator)1`.
    - *Failure Types:* API rate limits, invalid sheet IDs, missing authentication scopes (Google OAuth2).
  - **Read Tracker (Generator)1**
    - *Type & Role:* Google Sheets node fetching existing compliance records to avoid duplicates.
    - *Configuration:* Operation: `getAll`, Sheet Name: `Compliance_Tracker`, Document ID: `REPLACE_WITH_SHEET_ID`, setting `executeOnce: true` and `alwaysOutputData: true`.
    - *Connections:* Input: `Read Clients1`; Output: `Generate Upcoming Deadlines1`.
    - *Failure Types:* Large sheet size timeouts, OAuth token expiration.
  - **Generate Upcoming Deadlines1**
    - *Type & Role:* Code node (JavaScript) computing statutory compliance dates.
    - *Configuration:* Processes raw client data, evaluates GST frequency (`monthly`, `qrmp`, `none`), TDS applicability, entity types (`company`, `llp`, `other`), and calculates due dates within a 60-day horizon. Generates composite `task_id` strings (`client_id|form|period`).
    - *Key Expressions:* `$now.setZone('Asia/Kolkata')`, `$input.all()`, `$('Read Clients1').all()`.
    - *Connections:* Input: `Read Tracker (Generator)1`; Output: `Append New Tasks1`.
    - *Failure Types:* JavaScript runtime exceptions caused by malformed date inputs in the source sheet.
  - **Append New Tasks1**
    - *Type & Role:* Google Sheets node writing newly generated tasks to the sheet.
    - *Configuration:* Operation: `append`, Sheet Name: `Compliance_Tracker`, Document ID: `REPLACE_WITH_SHEET_ID`, Mapping Mode: `autoMapInputData`.
    - *Connections:* Input: `Generate Upcoming Deadlines1`; Output: None.
    - *Failure Types:* Schema mismatches between code output keys and sheet column headers.

---

#### 2.2 Daily WhatsApp Reminders

- **Overview:** Runs every morning at 9:00 AM to identify tasks due within configured windows, groups pending obligations by client, formats reminders, sends them via WhatsApp, and logs sent status updates.
- **Nodes Involved:**
  - `Daily 9 AM1`
  - `Read Tracker (Reminders)1`
  - `Build Reminder Payloads1`
  - `Send WhatsApp Reminder1`
  - `Prepare Reminder Update1`
  - `Mark Reminder Sent1`

- **Node Details:**
  - **Daily 9 AM1**
    - *Type & Role:* Schedule Trigger node executing daily.
    - *Configuration:* Cron expression `0 9 * * *` (9:00 AM daily).
    - *Connections:* Input: None; Output: `Read Tracker (Reminders)1`.
  - **Read Tracker (Reminders)1**
    - *Type & Role:* Google Sheets node loading current compliance tracker rows.
    - *Configuration:* Operation: `getAll`, Sheet Name: `Compliance_Tracker`, Document ID: `REPLACE_WITH_SHEET_ID`.
    - *Connections:* Input: `Daily 9 AM1`; Output: `Build Reminder Payloads1`.
  - **Build Reminder Payloads1**
    - *Type & Role:* Code node (JavaScript) grouping tasks by client and constructing Meta Graph API request payloads.
    - *Configuration:* Evaluates reminder intervals (`REMIND_DAYS = [7, 3, 1, 0]`), checks whether notifications were already sent today, and structures either an approved Meta template payload (`compliance_reminder`) or a fallback text message.
    - *Connections:* Input: `Read Tracker (Reminders)1`; Output: `Send WhatsApp Reminder1`.
  - **Send WhatsApp Reminder1**
    - *Type & Role:* HTTP Request node executing outbound calls to the Meta WhatsApp Cloud API.
    - *Configuration:* Method: `POST`, URL: `https://graph.facebook.com/v21.0/REPLACE_WITH_PHONE_NUMBER_ID/messages`, Authentication: Generic Credential Type (`httpHeaderAuth`), Error handling: `continueRegularOutput`.
    - *Key Expressions:* `={{ JSON.stringify($json.payload) }}`.
    - *Connections:* Input: `Build Reminder Payloads1`; Output: `Prepare Reminder Update1`.
    - *Failure Types:* HTTP 400/401 errors from invalid phone number IDs, expired access tokens, or template parameter mismatches.
  - **Prepare Reminder Update1**
    - *Type & Role:* Code node validating successful API responses before updating the tracker.
    - *Configuration:* Filters items where Meta successfully returned an array of sent messages (`it.json.messages`).
    - *Connections:* Input: `Send WhatsApp Reminder1`; Output: `Mark Reminder Sent1`.
  - **Mark Reminder Sent1**
    - *Type & Role:* Google Sheets node updating reminder metrics in the tracker.
    - *Configuration:* Operation: `update`, Sheet Name: `Compliance_Tracker`, Matching Columns: `task_id`, updating `last_reminder_sent` and incrementing `reminder_count`.
    - *Connections:* Input: `Prepare Reminder Update1`; Output: None.

---

#### 2.3 Inbound WhatsApp Processing & Routing

- **Overview:** Captures inbound webhook events from WhatsApp, maps senders to registered clients, routes text messages to status replies, and routes file uploads through cloud storage retrieval, AI classification, sheet updating, logging, and confirmation dispatch.
- **Nodes Involved:**
  - `WhatsApp Trigger1`
  - `Read Clients (Inbound)1`
  - `Read Tracker (Inbound)1`
  - `Route Incoming Message1`
  - `Switch by Route1`
  - `Get Media URL1`
  - `Download Media1`
  - `Upload to Client Drive Folder1`
  - `Classify Document1`
  - `Apply Checklist Update1`
  - `Split Tracker Updates1`
  - `Update Tracker Checklist1`
  - `Prepare Upload Log1`
  - `Append Upload Log1`
  - `Send WhatsApp Reply1`

- **Node Details:**
  - **WhatsApp Trigger1**
    - *Type & Role:* Webhook trigger listening for incoming Meta WhatsApp events.
    - *Configuration:* Watches for `messages` updates. Webhook ID: `b0d462f3-0147-403e-a5b5-0bfc04f0cf2e`.
    - *Connections:* Input: None; Output: `Read Clients (Inbound)1`.
  - **Read Clients (Inbound)1**
    - *Type & Role:* Google Sheets node fetching client phone number mappings.
    - *Configuration:* Sheet Name: `Clients`, Document ID: `REPLACE_WITH_SHEET_ID`.
    - *Connections:* Input: `WhatsApp Trigger1`; Output: `Read Tracker (Inbound)1`.
  - **Read Tracker (Inbound)1**
    - *Type & Role:* Google Sheets node fetching open compliance tasks for matching.
    - *Configuration:* Sheet Name: `Compliance_Tracker`, `executeOnce: true`, `alwaysOutputData: true`.
    - *Connections:* Input: `Read Clients (Inbound)1`; Output: `Route Incoming Message1`.
  - **Route Incoming Message1**
    - *Type & Role:* Code node (JavaScript) performing sender lookup, parsing message payloads, and establishing classification prompts.
    - *Configuration:* Normalizes phone numbers (last 10 digits), checks authorization, handles text vs. document/image distinction, and constructs the JSON prompt for OpenAI. Sets routing parameter to either `media` or `reply`.
    - *Connections:* Input: `Read Tracker (Inbound)1`; Output: `Switch by Route1`.
  - **Switch by Route1**
    - *Type & Role:* Switch node routing execution based on message evaluation.
    - *Configuration:* Rules branch into output `media` (for files/images) or `reply` (for text messages or unregistered senders).
    - *Connections:* Input: `Route Incoming Message1`; Outputs: `Get Media URL1` (Branch 0), `Send WhatsApp Reply1` (Branch 1).
  - **Get Media URL1**
    - *Type & Role:* HTTP Request node querying Meta graph API to retrieve temporary media download links.
    - *Configuration:* Method: `GET`, URL: `=https://graph.facebook.com/v21.0/{{ $json.media_id }}`, Auth: `httpHeaderAuth`.
    - *Connections:* Input: `Switch by Route1`; Output: `Download Media1`.
  - **Download Media1**
    - *Type & Role:* HTTP Request node fetching the actual binary media file from Meta servers.
    - *Configuration:* Method: `GET`, URL: `={{ $json.url }}`, Response Format: `file`, Auth: `httpHeaderAuth`.
    - *Connections:* Input: `Get Media URL1`; Output: `Upload to Client Drive Folder1`.
  - **Upload to Client Drive Folder1**
    - *Type & Role:* Google Drive node storing client documents in cloud storage.
    - *Configuration:* Operation: `create`, Folder ID resolved dynamically via expression: `={{ $('Route Incoming Message1').item.json.client.drive_folder_id || 'REPLACE_WITH_DRIVE_ROOT_FOLDER_ID' }}`.
    - *Connections:* Input: `Download Media1`; Output: `Classify Document1`.
    - *Failure Types:* Invalid folder IDs, missing Google Drive OAuth2 write scopes, quota exceeded.
  - **Classify Document1**
    - *Type & Role:* OpenAI Chat Model node (LangChain integration) running `gpt-4o-mini`.
    - *Configuration:* Model ID: `gpt-4o-mini`, Temperature: `0`, JSON Output mode enabled. Uses system prompt instructing the model to match documents to pending checklist items using strict ID/document matching rules. User prompt uses `={{ $('Route Incoming Message1').item.json.ai_prompt }}`.
    - *Connections:* Input: `Upload to Client Drive Folder1`; Output: `Apply Checklist Update1`.
    - *Failure Types:* API rate limits, invalid OpenAI credentials, malformed JSON responses from LLM.
  - **Apply Checklist Update1**
    - *Type & Role:* Code node (JavaScript) parsing LLM classification output, reconciling matched compliance items, and preparing downstream sheet updates and confirmation texts.
    - *Connections:* Input: `Classify Document1`; Outputs: `Split Tracker Updates1`, `Prepare Upload Log1`, `Send WhatsApp Reply1`.
  - **Split Tracker Updates1**
    - *Type & Role:* Code node extracting individual task updates into separate items.
    - *Connections:* Input: `Apply Checklist Update1`; Output: `Update Tracker Checklist1`.
  - **Update Tracker Checklist1**
    - *Type & Role:* Google Sheets node updating compliance tracker rows upon successful document matching.
    - *Configuration:* Operation: `update`, Matching Columns: `task_id`, Sheet Name: `Compliance_Tracker`.
    - *Connections:* Input: `Split Tracker Updates1`; Output: None.
  - **Prepare Upload Log1**
    - *Type & Role:* Code node isolating audit logging data.
    - *Connections:* Input: `Apply Checklist Update1`; Output: `Append Upload Log1`.
  - **Append Upload Log1**
    - *Type & Role:* Google Sheets node recording upload audit trails.
    - *Configuration:* Operation: `append`, Sheet Name: `Upload_Log`.
    - *Connections:* Input: `Prepare Upload Log1`; Output: None.
  - **Send WhatsApp Reply1**
    - *Type & Role:* HTTP Request node dispatching confirmation or checklist status messages to clients via WhatsApp API.
    - *Configuration:* Method: `POST`, URL: `https://graph.facebook.com/v21.0/REPLACE_WITH_PHONE_NUMBER_ID/messages`, Auth: `httpHeaderAuth`.
    - *Connections:* Input: `Switch by Route1` (for text queries) and `Apply Checklist Update1` (for media processing confirmations); Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `📌 Overview – How This Workflow Works` | `n8n-nodes-base.stickyNote` | High-level architectural overview and deployment steps | None | None | ## 🗓️ CA Firm Compliance Calendar – TDS, GST & ROC WhatsApp Reminders... |
| `Section – Deadline Generator` | `n8n-nodes-base.stickyNote` | Groups bi-monthly deadline generation logic | None | None | ## 📅 Deadline Generator (1st & 15th)... |
| `Section – Daily WhatsApp Reminders` | `n8n-nodes-base.stickyNote` | Groups daily morning reminder sequence | None | None | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Section – Inbound Message Routing` | `n8n-nodes-base.stickyNote` | Groups inbound webhook message routing logic | None | None | ## 📥 Inbound WhatsApp Message Routing... |
| `Section – Document Classification & Tracker Update` | `n8n-nodes-base.stickyNote` | Groups document ingestion, AI matching, storage, and logging logic | None | None | ## 🤖 Document Classification & Tracker Update... |
| `🔑 Credentials & Configuration` | `n8n-nodes-base.stickyNote` | Outlines mandatory external credentials | None | None | ## 🔑 Credentials Required... |
| `Every 1st & 15th (6 AM)1` | `n8n-nodes-base.scheduleTrigger` | Triggers deadline calculation bi-monthly | None | `Read Clients1` | ## 📅 Deadline Generator (1st & 15th)... |
| `Read Clients1` | `n8n-nodes-base.googleSheets` | Fetches active client roster | `Every 1st & 15th (6 AM)1` | `Read Tracker (Generator)1` | ## 📅 Deadline Generator (1st & 15th)... |
| `Read Tracker (Generator)1` | `n8n-nodes-base.googleSheets` | Fetches existing tracker tasks to avoid duplicates | `Read Clients1` | `Generate Upcoming Deadlines1` | ## 📅 Deadline Generator (1st & 15th)... |
| `Generate Upcoming Deadlines1` | `n8n-nodes-base.code` | Computes tax and corporate deadlines | `Read Tracker (Generator)1` | `Append New Tasks1` | ## 📅 Deadline Generator (1st & 15th)... |
| `Append New Tasks1` | `n8n-nodes-base.googleSheets` | Appends new tasks to Compliance Tracker | `Generate Upcoming Deadlines1` | None | ## 📅 Deadline Generator (1st & 15th)... |
| `Daily 9 AM1` | `n8n-nodes-base.scheduleTrigger` | Triggers daily reminder execution | None | `Read Tracker (Reminders)1` | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Read Tracker (Reminders)1` | `n8n-nodes-base.googleSheets` | Loads tracker rows for reminder checks | `Daily 9 AM1` | `Build Reminder Payloads1` | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Build Reminder Payloads1` | `n8n-nodes-base.code` | Filters items and builds WhatsApp payloads | `Read Tracker (Reminders)1` | `Send WhatsApp Reminder1` | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Send WhatsApp Reminder1` | `n8n-nodes-base.httpRequest` | Sends WhatsApp reminder messages | `Build Reminder Payloads1` | `Prepare Reminder Update1` | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Prepare Reminder Update1` | `n8n-nodes-base.code` | Validates API delivery before updating sheet | `Send WhatsApp Reminder1` | `Mark Reminder Sent1` | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `Mark Reminder Sent1` | `n8n-nodes-base.googleSheets` | Updates last reminder timestamp and count | `Prepare Reminder Update1` | None | ## 📲 Daily WhatsApp Reminders (9AM)... |
| `WhatsApp Trigger1` | `n8n-nodes-base.whatsAppTrigger` | Inbound webhook for WhatsApp messages | None | `Read Clients (Inbound)1` | ## 📥 Inbound WhatsApp Message Routing... |
| `Read Clients (Inbound)1` | `n8n-nodes-base.googleSheets` | Fetches client phone directory for inbound routing | `WhatsApp Trigger1` | `Read Tracker (Inbound)1` | ## 📥 Inbound WhatsApp Message Routing... |
| `Read Tracker (Inbound)1` | `n8n-nodes-base.googleSheets` | Fetches active compliance tasks | `Read Clients (Inbound)1` | `Route Incoming Message1` | ## 📥 Inbound WhatsApp Message Routing... |
| `Route Incoming Message1` | `n8n-nodes-base.code` | Identifies client and determines processing route | `Read Tracker (Inbound)1` | `Switch by Route1` | ## 📥 Inbound WhatsApp Message Routing... |
| `Switch by Route1` | `n8n-nodes-base.switch` | Branches workflow based on message type | `Route Incoming Message1` | `Get Media URL1`, `Send WhatsApp Reply1` | ## 📥 Inbound WhatsApp Message Routing... |
| `Get Media URL1` | `n8n-nodes-base.httpRequest` | Fetches temporary media download URL from Meta | `Switch by Route1` | `Download Media1` | ## 🤖 Document Classification & Tracker Update... |
| `Download Media1` | `n8n-nodes-base.httpRequest` | Downloads binary media attachment | `Get Media URL1` | `Upload to Client Drive Folder1` | ## 🤖 Document Classification & Tracker Update... |
| `Upload to Client Drive Folder1` | `n8n-nodes-base.googleDrive` | Uploads file to client's Google Drive folder | `Download Media1` | `Classify Document1` | ## 🤖 Document Classification & Tracker Update... |
| `Classify Document1` | `@n8n/n8n-nodes-langchain.openAi` | Classifies document against checklist via GPT-4o-mini | `Upload to Client Drive Folder1` | `Apply Checklist Update1` | ## 🤖 Document Classification & Tracker Update... |
| `Apply Checklist Update1` | `n8n-nodes-base.code` | Parses AI match and updates checklist items | `Classify Document1` | `Split Tracker Updates1`, `Prepare Upload Log1`, `Send WhatsApp Reply1` | ## 🤖 Document Classification & Tracker Update... |
| `Split Tracker Updates1` | `n8n-nodes-base.code` | Splits checklist updates for individual row writes | `Apply Checklist Update1` | `Update Tracker Checklist1` | ## 🤖 Document Classification & Tracker Update... |
| `Update Tracker Checklist1` | `n8n-nodes-base.googleSheets` | Updates checklist completion status in sheet | `Split Tracker Updates1` | None | ## 🤖 Document Classification & Tracker Update... |
| `Prepare Upload Log1` | `n8n-nodes-base.code` | Formats upload audit log row | `Apply Checklist Update1` | `Append Upload Log1` | ## 🤖 Document Classification & Tracker Update... |
| `Append Upload Log1` | `n8n-nodes-base.googleSheets` | Appends record to Upload Log sheet | `Prepare Upload Log1` | None | ## 🤖 Document Classification & Tracker Update... |
| `Send WhatsApp Reply1` | `n8n-nodes-base.httpRequest` | Sends automated reply or confirmation to client | `Switch by Route1`, `Apply Checklist Update1` | None | ## 📥 Inbound WhatsApp Message Routing... / ## 🤖 Document Classification & Tracker Update... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Spreadsheet Resources & Setup Sheets
1. Create a Google Sheets spreadsheet containing three sheets:
   - **Clients**: Columns needed: `client_id`, `client_name`, `whatsapp_number`, `active`, `gst_filing`, `gst_3b_due_day`, `tds_applicable`, `entity_type`, `gstr9_applicable`, `roc_applicable`, `agm_date`, `msme_applicable`, `drive_folder_id`.
   - **Compliance_Tracker**: Columns needed: `task_id`, `client_id`, `client_name`, `whatsapp_number`, `compliance_type`, `form`, `period`, `due_date`, `status`, `docs_required`, `docs_received`, `checklist_pct`, `last_reminder_sent`, `reminder_count`.
   - **Upload_Log**: Columns needed: `timestamp`, `client_id`, `client_name`, `file_name`, `drive_file_id`, `drive_link`, `matched_items`, `ai_reason`, `whatsapp_media_id`.

#### Step 2: Build Pipeline 1 – Deadline Generator
1. **Schedule Trigger**: Create node `Every 1st & 15th (6 AM)1`. Set Rule type to Cron expression: `0 6 1,15 * *`.
2. **Google Sheets (Read Clients)**: Create node `Read Clients1`. Connect credential (Google Sheets OAuth2). Set Sheet Name to `Clients` and Document ID to your spreadsheet ID.
3. **Google Sheets (Read Tracker)**: Create node `Read Tracker (Generator)1`. Set Sheet Name to `Compliance_Tracker`. Configure options: `executeOnce: true`, `alwaysOutputData: true`.
4. **Code (Generate Deadlines)**: Create node `Generate Upcoming Deadlines1`. Insert the JavaScript code provided in the JSON configuration for deadline generation.
5. **Google Sheets (Append Tasks)**: Create node `Append New Tasks1`. Operation: `append`, Sheet Name: `Compliance_Tracker`, Mapping Mode: `autoMapInputData`.
6. Connect nodes: `Every 1st & 15th (6 AM)1` ➔ `Read Clients1` ➔ `Read Tracker (Generator)1` ➔ `Generate Upcoming Deadlines1` ➔ `Append New Tasks1`.

#### Step 3: Build Pipeline 2 – Daily WhatsApp Reminders
1. **Schedule Trigger**: Create node `Daily 9 AM1`. Cron expression: `0 9 * * *`.
2. **Google Sheets (Read Tracker)**: Create node `Read Tracker (Reminders)1`. Sheet Name: `Compliance_Tracker`.
3. **Code (Build Payloads)**: Create node `Build Reminder Payloads1`. Insert the reminder generation JavaScript code.
4. **HTTP Request (Send Reminder)**: Create node `Send WhatsApp Reminder1`. Method: `POST`, URL: `https://graph.facebook.com/v21.0/REPLACE_WITH_PHONE_NUMBER_ID/messages`. Auth: Generic Credential Type (`httpHeaderAuth`). Set Body to JSON stringified payload. Set Error Handling to `continueRegularOutput`.
5. **Code (Prepare Update)**: Create node `Prepare Reminder Update1`. Insert reminder validation JavaScript code.
6. **Google Sheets (Mark Sent)**: Create node `Mark Reminder Sent1`. Operation: `update`, Sheet Name: `Compliance_Tracker`, Matching Columns: `task_id`.
7. Connect nodes: `Daily 9 AM1` ➔ `Read Tracker (Reminders)1` ➔ `Build Reminder Payloads1` ➔ `Send WhatsApp Reminder1` ➔ `Prepare Reminder Update1` ➔ `Mark Reminder Sent1`.

#### Step 4: Build Pipeline 3 – Inbound Message Routing & Processing
1. **WhatsApp Trigger**: Create node `WhatsApp Trigger1`. Listen for updates: `messages`. Copy the generated Webhook URL to your Meta App dashboard.
2. **Google Sheets (Read Clients)**: Create node `Read Clients (Inbound)1`. Sheet Name: `Clients`.
3. **Google Sheets (Read Tracker)**: Create node `Read Tracker (Inbound)1`. Sheet Name: `Compliance_Tracker`. `executeOnce: true`, `alwaysOutputData: true`.
4. **Code (Route Message)**: Create node `Route Incoming Message1`. Insert inbound parsing and routing JavaScript code.
5. **Switch**: Create node `Switch by Route1`. Configure two output rules: Output 0 where `{{ $json.route }}` equals `media`, Output 1 where `{{ $json.route }}` equals `reply`.
6. **HTTP Request (Get Media URL)**: Create node `Get Media URL1`. Method: `GET`, URL: `=https://graph.facebook.com/v21.0/{{ $json.media_id }}`. Auth: `httpHeaderAuth`. Connect from Switch Output 0.
7. **HTTP Request (Download Media)**: Create node `Download Media1`. Method: `GET`, URL: `={{ $json.url }}`, Response format: `file`. Auth: `httpHeaderAuth`.
8. **Google Drive (Upload)**: Create node `Upload to Client Drive Folder1`. Operation: `create`, Folder ID: `={{ $('Route Incoming Message1').item.json.client.drive_folder_id || 'REPLACE_WITH_DRIVE_ROOT_FOLDER_ID' }}`.
9. **OpenAI (Classify Document)**: Create node `Classify Document1`. Model: `gpt-4o-mini`, Temperature: `0`, JSON output enabled. Configure system prompt and user expression prompt (`={{ $('Route Incoming Message1').item.json.ai_prompt }}`).
10. **Code (Apply Update)**: Create node `Apply Checklist Update1`. Insert checklist reconciliation JavaScript code.
11. **Code (Split Updates)**: Create node `Split Tracker Updates1`. Insert array mapping code for tracker updates.
12. **Google Sheets (Update Checklist)**: Create node `Update Tracker Checklist1`. Operation: `update`, Matching Columns: `task_id`, Sheet Name: `Compliance_Tracker`.
13. **Code (Prepare Log)**: Create node `Prepare Upload Log1`. Insert log isolation code.
14. **Google Sheets (Append Log)**: Create node `Append Upload Log1`. Operation: `append`, Sheet Name: `Upload_Log`.
15. **HTTP Request (Send Reply)**: Create node `Send WhatsApp Reply1`. Method: `POST`, URL: `https://graph.facebook.com/v21.0/REPLACE_WITH_PHONE_NUMBER_ID/messages`. Auth: `httpHeaderAuth`. Connect inputs from both Switch Output 1 and `Apply Checklist Update1`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WhatsApp Business API configuration requirements | Ensure valid HTTP Header Auth credentials are created with your Meta permanent/temporary access token, and replace `REPLACE_WITH_PHONE_NUMBER_ID` in all WhatsApp HTTP request URLs. |
| Google Sheets and Drive OAuth2 Setup | Connect Google OAuth2 credentials across all Sheets nodes and the Drive upload node. Ensure proper spreadsheet ID variables and folder IDs are mapped correctly. |
| OpenAI API Configuration | Connect an OpenAI API credential with access to `gpt-4o-mini` to power automated document classification. |