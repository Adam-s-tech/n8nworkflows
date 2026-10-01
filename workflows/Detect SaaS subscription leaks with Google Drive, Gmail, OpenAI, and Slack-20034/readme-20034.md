Detect SaaS subscription leaks with Google Drive, Gmail, OpenAI, and Slack

https://n8nworkflows.xyz/workflows/detect-saas-subscription-leaks-with-google-drive--gmail--openai--and-slack-20034


# Detect SaaS subscription leaks with Google Drive, Gmail, OpenAI, and Slack

### 1. Workflow Overview

This workflow is an automated Financial Operations (FinOps) audit tool designed to identify, analyze, and report hidden SaaS subscription waste within an organization. It executes either on-demand via a manual trigger or on a recurring weekly schedule (every Monday at 9:00 AM). 

The architecture aggregates data from three disparate sources in parallel—corporate card CSV statements from Google Drive, invoice emails from Gmail processed through an OpenAI language model, and a master SaaS inventory spreadsheet from Google Sheets. A central detection engine cross-references this information to uncover anomalies such as zombie charges, duplicate billings, unused software licenses, seat waste, price creep, overlapping tool categories, and shadow IT subscriptions. If financial leaks are detected, an AI model builds a prioritized mitigation plan, which is then logged to a Google Sheets audit report, emailed to financial stakeholders, and published to a team messaging channel.

The logic is grouped into five functional blocks:
- **1.1 Trigger & Configuration:** Initializes the audit run and centralizes global parameters (folder IDs, sheet IDs, search filters, and alert thresholds).
- **1.2 Parallel Data Collection:** Concurrently pulls corporate card statements from Google Drive, fetches and extracts line items from invoice emails via OpenAI, and reads the active SaaS inventory spreadsheet.
- **1.3 Leak Detection & AI Analysis:** Consolidates data streams to calculate financial waste metrics, evaluates business logic rules, and passes findings to OpenAI to generate an actionable optimization strategy.
- **1.4 Conditional Evaluation & Routing:** Determines whether waste was identified to either branch into mitigation reporting or terminate cleanly.
- **1.5 Report Delivery & Logging:** Disperses findings simultaneously to Google Sheets, Gmail, and Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
- **Overview:** Establishes entry points for manual execution or automated weekly scheduling, feeding initial contextual settings into the core pipeline.
- **Nodes Involved:** `Run Manually`, `Weekly Monday 9 AM`, `Config`

- **Node Details:**
  - **Run Manually**
    - Type and technical role: `n8n-nodes-base.manualTrigger` (Manual Trigger)
    - Configuration choices: Default parameters.
    - Key expressions or variables: None.
    - Input and output connections: Output connects to `Config`.
    - Edge cases or potential failure types: None.
  - **Weekly Monday 9 AM**
    - Type and technical role: `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - Configuration choices: Configured to trigger weekly on Mondays at 09:00.
    - Key expressions or variables: None.
    - Input and output connections: Output connects to `Config`.
    - Edge cases or potential failure types: None.
  - **Config**
    - Type and technical role: `n8n-nodes-base.set` (Edit Fields)
    - Configuration choices: Establishes static operational variables including folder IDs, Google Sheet IDs, email filter queries, currency, date formats, and evaluation thresholds (`unusedDays`, `minSeatUtilization`, `reportEmail`, `slackChannel`).
    - Key expressions or variables: Maps properties such as `statementsFolderId`, `inventorySheetId`, `gmailQuery`, `currency`, `dateFormat`, `debitsAreNegative`, `unusedDays`, `minSeatUtilization`, `reportEmail`, and `slackChannel`.
    - Input and output connections: Inputs from `Run Manually` and `Weekly Monday 9 AM`. Outputs broadcast to `Find Card Statements`, `Fetch Invoice Emails`, and `Read SaaS Inventory`.
    - Edge cases or potential failure types: Invalid IDs or misconfigured folder/sheet permissions will fail downstream API calls.

---

#### 2.2 Parallel Data Collection
- **Overview:** Simultaneously gathers raw transaction histories, vendor invoices, and internal software inventories from three distinct platforms.
- **Nodes Involved:** `Find Card Statements`, `Download Statement`, `Parse Statement CSV`, `Normalize Card Transactions`, `Fetch Invoice Emails`, `Prepare Invoice Batch`, `Extract Invoice Data`, `Parse Invoice Data`, `Read SaaS Inventory`

- **Node Details:**
  - **Find Card Statements**
    - Type and technical role: `n8n-nodes-base.googleDrive` (Google Drive)
    - Configuration choices: Resource set to file/folder, using a custom query string to filter CSV files inside the configured parent folder.
    - Key expressions or variables: `='{{ $json.statementsFolderId }}' in parents and (mimeType='text/csv' or name contains '.csv') and trashed=false`
    - Input and output connections: Input from `Config`. Output connects to `Download Statement`.
    - Edge cases or potential failure types: Authentication failures or invalid folder IDs return empty lists or errors.
  - **Download Statement**
    - Type and technical role: `n8n-nodes-base.googleDrive` (Google Drive)
    - Configuration choices: Operation set to download file by ID. Error handling set to continue regular output.
    - Key expressions or variables: File ID: `={{ $json.id }}`
    - Input and output connections: Input from `Find Card Statements`. Output connects to `Parse Statement CSV`.
    - Edge cases or potential failure types: Missing files or corrupted binaries handled via error continuation.
  - **Parse Statement CSV**
    - Type and technical role: `n8n-nodes-base.extractFromFile` (Extract from File)
    - Configuration choices: Extracts tabular data from binary files. Error handling set to continue regular output.
    - Input and output connections: Input from `Download Statement`. Output connects to `Normalize Card Transactions`.
    - Edge cases or potential failure types: Unstructured or non-standard CSV schemas may yield unparseable rows.
  - **Normalize Card Transactions**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: JavaScript parser that normalizes multi-format banking and credit card CSV exports into a standard schema, filtering out refunds and cleaning merchant names.
    - Key expressions or variables: Reads configuration settings from `$('Config').first().json`.
    - Input and output connections: Input from `Parse Statement CSV`. Output connects input index 0 of `Wait for All Sources`.
    - Edge cases or potential failure types: Non-standard date formats or missing amount columns can cause rows to be skipped.
  - **Fetch Invoice Emails**
    - Type and technical role: `n8n-nodes-base.gmail` (Gmail)
    - Configuration choices: Operation set to retrieve messages (`getAll`), returning up to 80 emails using a dynamic search query.
    - Key expressions or variables: Filter query: `={{ $json.gmailQuery }}`
    - Input and output connections: Input from `Config`. Output connects to `Prepare Invoice Batch`.
    - Edge cases or potential failure types: Rate limits or OAuth scope restrictions.
  - **Prepare Invoice Batch**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Strips HTML tags, sanitizes characters, and aggregates up to 80 email payloads into a single structured array item to optimize AI requests.
    - Input and output connections: Input from `Fetch Invoice Emails`. Output connects to `Extract Invoice Data`.
    - Edge cases or potential failure types: Extremely large email payloads could trigger size limits.
  - **Extract Invoice Data**
    - Type and technical role: `@n8n/n8n-nodes-langchain.openAi` (OpenAI)
    - Configuration choices: Uses model `gpt-4o-mini` with strict JSON output mode and a low temperature (0.1) to extract SaaS billing line items.
    - Key expressions or variables: Passes serialized email array using `{{ JSON.stringify($json.emails) }}`.
    - Input and output connections: Input from `Prepare Invoice Batch`. Output connects to `Parse Invoice Data`.
    - Edge cases or potential failure types: API timeouts, rate limits, or non-compliant model outputs.
  - **Parse Invoice Data**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Parses the JSON response from OpenAI, filtering out non-SaaS vendors and normalizing line-item attributes.
    - Input and output connections: Input from `Extract Invoice Data`. Output connects input index 1 of `Wait for All Sources`.
    - Edge cases or potential failure types: Malformed JSON strings returned by the model.
  - **Read SaaS Inventory**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets)
    - Configuration choices: Reads rows from the "SaaS Inventory" worksheet.
    - Key expressions or variables: Document ID: `={{ $json.inventorySheetId }}`
    - Input and output connections: Input from `Config`. Output connects input index 2 of `Wait for All Sources`.
    - Edge cases or potential failure types: Missing sheet tabs or permission revocation.

---

#### 1.3 Leak Detection & AI Analysis
- **Overview:** Synchronizes all collected data streams, executes deterministic financial rule checks, and leverages an LLM to generate prioritization and optimization plans.
- **Nodes Involved:** `Wait for All Sources`, `Detect Leaks`, `Any Leaks Found?`, `Draft Cancellation Plan`, `Parse AI Plan`

- **Node Details:**
  - **Wait for All Sources**
    - Type and technical role: `n8n-nodes-base.merge` (Merge)
    - Configuration choices: Mode set to combine three input streams (Card Transactions, Invoices, and Inventory).
    - Input and output connections: Inputs from `Normalize Card Transactions`, `Parse Invoice Data`, and `Read SaaS Inventory`. Output connects to `Detect Leaks`.
    - Edge cases or potential failure types: Execution stalls if any input branch fails to emit data.
  - **Detect Leaks**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Comprehensive FinOps analytics engine that matches tool aliases, checks for zombie accounts, identifies duplicate charges, detects low-seat utilization, monitors price increases, flags tool category overlaps, and discovers shadow subscriptions.
    - Key expressions or variables: Evaluates thresholds such as `unusedDays`, `minSeatUtilization`, and `currency`.
    - Input and output connections: Input from `Wait for All Sources`. Output connects to `Any Leaks Found?`.
    - Edge cases or potential failure types: Memory constraints if transaction sets are excessively large.
  - **Any Leaks Found?**
    - Type and technical role: `n8n-nodes-base.if` (If)
    - Configuration choices: Evaluates whether the number of identified findings is greater than zero.
    - Key expressions or variables: `={{ $json.findingsCount }}` > 0
    - Input and output connections: Input from `Detect Leaks`. True branch connects to `Draft Cancellation Plan`; false branch connects to `No Leaks - Done`.
    - Edge cases or potential failure types: None.
  - **Draft Cancellation Plan**
    - Type and technical role: `@n8n/n8n-nodes-langchain.openAi` (OpenAI)
    - Configuration choices: Uses model `gpt-4o-mini` with JSON output mode to convert raw findings into a clear executive summary and prioritized remediation steps.
    - Key expressions or variables: Passes currency, monthly spend totals, and stringified findings.
    - Input and output connections: Input from `Any Leaks Found?` (True branch). Output connects to `Parse AI Plan`.
    - Edge cases or potential failure types: API rate limits or invalid model responses.
  - **Parse AI Plan**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Merges AI recommendations with raw detection results and calculates confirmed monthly and annual savings metrics.
    - Input and output connections: Input from `Draft Cancellation Plan`. Output fans out to `Build Report Rows`, `Build Email Report`, and `Slack Alert`.
    - Edge cases or potential failure types: Malformed JSON from the AI block.

---

#### 1.4 Conditional Evaluation & Routing
- **Overview:** Handles workflow branching based on leak detection status, gracefully terminating executions that find no financial waste.
- **Nodes Involved:** `No Leaks - Done`

- **Node Details:**
  - **No Leaks - Done**
    - Type and technical role: `n8n-nodes-base.noOp` (No Operation)
    - Configuration choices: Null operation used as a clean terminal point.
    - Input and output connections: Input from `Any Leaks Found?` (False branch). No output connections.
    - Edge cases or potential failure types: None.

---

#### 1.5 Report Delivery & Logging
- **Overview:** Simultaneously records audit findings into a tracking spreadsheet, compiles and dispatches an HTML email report to finance, and posts a high-level summary alert to a Slack channel.
- **Nodes Involved:** `Build Report Rows`, `Log to Leak Report`, `Build Email Report`, `Email Finance Report`, `Slack Alert`

- **Node Details:**
  - **Build Report Rows**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Transforms finding objects and AI recommendations into discrete tabular rows suitable for spreadsheet insertion.
    - Input and output connections: Input from `Parse AI Plan`. Output connects to `Log to Leak Report`.
    - Edge cases or potential failure types: Schema mismatches against spreadsheet column headers.
  - **Log to Leak Report**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets)
    - Configuration choices: Appends rows to the "Leak Report" sheet using auto-mapping.
    - Key expressions or variables: Document ID: `={{ $('Config').first().json.inventorySheetId }}`
    - Input and output connections: Input from `Build Report Rows`. No subsequent nodes.
    - Edge cases or potential failure types: Sheet permission errors or missing target sheet names.
  - **Build Email Report**
    - Type and technical role: `n8n-nodes-base.code` (Code)
    - Configuration choices: Generates a responsive HTML email layout containing executive summaries, prioritized recommendations, and itemized findings.
    - Input and output connections: Input from `Parse AI Plan`. Output connects to `Email Finance Report`.
    - Edge cases or potential failure types: Unsupported HTML rendering in specific email clients.
  - **Email Finance Report**
    - Type and technical role: `n8n-nodes-base.gmail` (Gmail)
    - Configuration choices: Sends an email message with attribution disabled.
    - Key expressions or variables: Recipient: `={{ $('Config').first().json.reportEmail }}`; Subject: `={{ $json.subject }}`; Body: `={{ $json.html }}`
    - Input and output connections: Input from `Build Email Report`. No subsequent nodes.
    - Edge cases or potential failure types: Invalid recipient addresses or SMTP/OAuth token expiration.
  - **Slack Alert**
    - Type and technical role: `n8n-nodes-base.slack` (Slack)
    - Configuration choices: Publishes formatted Markdown alerts to a designated Slack channel.
    - Key expressions or variables: Channel ID: `={{ $('Config').first().json.slackChannel }}`; Text: Formatted alert message including finding counts and potential savings.
    - Input and output connections: Input from `Parse AI Plan`. No subsequent nodes.
    - Edge cases or potential failure types: Bot missing permissions to post in the specified channel.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 📌 Overview – How This Workflow Works | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## 💸 SaaS Subscription Leak Detector – Automated FinOps Audit<br><br>### How it works<br>This workflow runs every Monday to audit your company's SaaS spend for hidden waste. It pulls data from three sources in parallel: corporate card statements from Google Drive (CSV), invoice emails from Gmail (parsed by GPT-4o-mini), and your SaaS inventory sheet. A leak detection engine cross-references all three to flag four types of waste: zombie charges (cancelled tools still billing), duplicate subscriptions, unused tools (no logins in 30+ days), and low-utilisation seats. GPT-4o-mini then generates a prioritised cancellation and optimisation plan. Results are logged to a Google Sheets report, emailed to the finance team, and posted to Slack — with the total confirmed monthly savings amount.<br><br>### Setup steps<br>1. Connect **Google Drive OAuth2** to `Find Card Statements` and `Download Statement` — set `YOUR_GOOGLE_DRIVE_FOLDER_ID` in `Config`.<br>2. Connect **Gmail OAuth2** to `Fetch Invoice Emails` and `Email Finance Report`.<br>3. Connect **OpenAI API** to `Extract Invoice Data` and `Draft Cancellation Plan`.<br>4. Connect **Google Sheets OAuth2** to `Read SaaS Inventory` and `Log to Leak Report` — set `YOUR_GOOGLE_SHEET_ID` in `Config`.<br>5. Connect **Slack OAuth2** to `Slack Alert` — set `slackChannel` in `Config`.<br>6. Update `Config`: set `reportEmail`, `unusedDays`, `minSeatUtilization`, and `currency`.<br>7. Run manually to test, then activate the weekly schedule. |
| Section – Trigger & Config | `n8n-nodes-base.stickyNote` | Section grouping for triggers and config | None | None | ## ⚙️ Trigger & Config<br>Two entry points share the same pipeline: a manual trigger for on-demand runs and a weekly Monday 9AM schedule for automated audits. The `Config` node centralises all settings — folder IDs, sheet IDs, thresholds, email, Slack channel, and currency — so you only change one place. |
| Section – Parallel Data Collection | `n8n-nodes-base.stickyNote` | Section grouping for parallel data gathering | None | None | ## 📊 Parallel Data Collection<br>Three sources run simultaneously from `Config`. Card statements are found in Google Drive, downloaded, and CSV-parsed. Invoice emails are fetched from Gmail and extracted by GPT-4o-mini into structured line items. The SaaS inventory sheet is read directly. All three merge before analysis. |
| Section – Leak Detection & AI Analysis | `n8n-nodes-base.stickyNote` | Section grouping for analysis and detection | None | None | ## 🔍 Leak Detection & AI Analysis<br>The leak engine cross-references card charges, invoices, and the SaaS inventory to flag: zombie charges, duplicates, unused tools, and low-utilisation seats. If findings exist, GPT-4o-mini generates a prioritised cancellation and optimisation plan with confirmed savings amounts. |
| Section – Report Delivery | `n8n-nodes-base.stickyNote` | Section grouping for report distribution | None | None | ## 📤 Report Delivery<br>Three parallel outputs fire when leaks are found: findings are logged row-by-row to Google Sheets; an HTML summary email is sent to the finance team; a Slack alert posts total findings and confirmed monthly savings. No leaks exits cleanly via the `No Leaks - Done` node. |
| 🔑 Credentials & Configuration | `n8n-nodes-base.stickyNote` | Documents required credentials | None | None | ## 🔑 Credentials Required<br>- **Google Drive OAuth2** — statement folder access<br>- **Gmail OAuth2** — invoice emails + finance report<br>- **OpenAI API** — invoice extraction + cancellation plan<br>- **Google Sheets OAuth2** — SaaS inventory + leak report log<br>- **Slack OAuth2** — savings alert |
| Run Manually | `n8n-nodes-base.manualTrigger` | Initiates manual workflow execution | None | Config | |
| Weekly Monday 9 AM | `n8n-nodes-base.scheduleTrigger` | Initiates scheduled weekly execution | None | Config | |
| Config | `n8n-nodes-base.set` | Centralizes workflow configuration parameters | Run Manually, Weekly Monday 9 AM | Find Card Statements, Fetch Invoice Emails, Read SaaS Inventory | |
| Find Card Statements | `n8n-nodes-base.googleDrive` | Queries Google Drive for CSV statements | Config | Download Statement | |
| Download Statement | `n8n-nodes-base.googleDrive` | Downloads statement files | Find Card Statements | Parse Statement CSV | |
| Parse Statement CSV | `n8n-nodes-base.extractFromFile` | Parses tabular data from CSV files | Download Statement | Normalize Card Transactions | |
| Normalize Card Transactions | `n8n-nodes-base.code` | Normalizes transaction rows | Parse Statement CSV | Wait for All Sources | |
| Fetch Invoice Emails | `n8n-nodes-base.gmail` | Retrieves invoice and receipt emails | Config | Prepare Invoice Batch | |
| Prepare Invoice Batch | `n8n-nodes-base.code` | Bundles email payloads into batches | Fetch Invoice Emails | Extract Invoice Data | |
| Extract Invoice Data | `@n8n/n8n-nodes-langchain.openAi` | Extracts SaaS invoice data via LLM | Prepare Invoice Batch | Parse Invoice Data | |
| Parse Invoice Data | `n8n-nodes-base.code` | Structures LLM invoice output | Extract Invoice Data | Wait for All Sources | |
| Read SaaS Inventory | `n8n-nodes-base.googleSheets` | Loads SaaS inventory spreadsheet data | Config | Wait for All Sources | |
| Wait for All Sources | `n8n-nodes-base.merge` | Merges parallel data streams | Normalize Card Transactions, Parse Invoice Data, Read SaaS Inventory | Detect Leaks | |
| Detect Leaks | `n8n-nodes-base.code` | Executes FinOps subscription leak rules | Wait for All Sources | Any Leaks Found? | |
| Any Leaks Found? | `n8n-nodes-base.if` | Evaluates if findings exist | Detect Leaks | Draft Cancellation Plan, No Leaks - Done | |
| Draft Cancellation Plan | `@n8n/n8n-nodes-langchain.openAi` | Generates AI optimization plan | Any Leaks Found? | Parse AI Plan | |
| No Leaks - Done | `n8n-nodes-base.noOp` | Terminal node when no leaks are found | Any Leaks Found? | None | |
| Parse AI Plan | `n8n-nodes-base.code` | Merges AI plan with detection metrics | Draft Cancellation Plan | Build Report Rows, Build Email Report, Slack Alert | |
| Build Report Rows | `n8n-nodes-base.code` | Formats data for spreadsheet logging | Parse AI Plan | Log to Leak Report | |
| Log to Leak Report | `n8n-nodes-base.googleSheets` | Appends findings to Google Sheets | Build Report Rows | None | |
| Build Email Report | `n8n-nodes-base.code` | Generates HTML email report | Parse AI Plan | Email Finance Report | |
| Email Finance Report | `n8n-nodes-base.gmail` | Sends HTML audit report via email | Build Email Report | None | |
| Slack Alert | `n8n-nodes-base.slack` | Publishes savings alert to Slack | Parse AI Plan | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers and Configuration:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) named `Run Manually`.
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Weekly Monday 9 AM` configured to run weekly on Mondays at 9:00 AM.
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`) named `Config`. Configure assignments with string, boolean, and number types for:
     - `statementsFolderId`: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
     - `inventorySheetId`: `YOUR_GOOGLE_SHEET_ID`
     - `gmailQuery`: `(subject:(invoice OR receipt OR "payment received" OR "your subscription" OR "billing") OR from:(billing OR invoice OR receipts)) newer_than:35d -in:spam`
     - `currency`: `USD`
     - `dateFormat`: `DMY`
     - `debitsAreNegative`: `false`
     - `unusedDays`: `30`
     - `minSeatUtilization`: `0.8`
     - `reportEmail`: `user@example.com`
     - `slackChannel`: `#finance-ops`
   - Connect both triggers to `Config`.

2. **Build Parallel Data Collection - Google Drive Branch:**
   - Create a **Google Drive** node named `Find Card Statements`. Set resource to `fileFolder`, operation to list/search, and use query: `='{{ $json.statementsFolderId }}' in parents and (mimeType='text/csv' or name contains '.csv') and trashed=false`. Connect `Config` to this node.
   - Create a second **Google Drive** node named `Download Statement`. Set operation to `download`, specifying file ID `={{ $json.id }}`. Enable error handling ("Continue Regular Output"). Connect `Find Card Statements` to it.
   - Create an **Extract from File** node named `Parse Statement CSV`. Enable error handling ("Continue Regular Output"). Connect `Download Statement` to it.
   - Create a **Code** node named `Normalize Card Transactions` with JavaScript logic to normalize row headers, parse amounts and dates, and filter out credits. Connect `Parse Statement CSV` to it.

3. **Build Parallel Data Collection - Gmail Branch:**
   - Create a **Gmail** node named `Fetch Invoice Emails`. Set operation to `getAll`, limit to `80`, and filter query to `={{ $json.gmailQuery }}`. Connect `Config` to this node.
   - Create a **Code** node named `Prepare Invoice Batch` to strip HTML and combine email payloads into a single array item. Connect `Fetch Invoice Emails` to it.
   - Create an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`) named `Extract Invoice Data`. Set model to `gpt-4o-mini`, temperature to `0.1`, enable JSON output mode, and configure system and user messages to parse invoice line items. Connect `Prepare Invoice Batch` to it. Configure with valid **OpenAI API** credentials.
   - Create a **Code** node named `Parse Invoice Data` to validate and structure model responses. Connect `Extract Invoice Data` to it.

4. **Build Parallel Data Collection - Google Sheets Inventory Branch:**
   - Create a **Google Sheets** node named `Read SaaS Inventory`. Set operation to read, document ID to `={{ $json.inventorySheetId }}` and sheet name to `SaaS Inventory`. Connect `Config` to this node. Configure with valid **Google Sheets OAuth2** credentials.

5. **Merge and Analyze Data:**
   - Create a **Merge** node named `Wait for All Sources` with `numberInputs` set to `3`. Connect `Normalize Card Transactions`, `Parse Invoice Data`, and `Read SaaS Inventory` to inputs 0, 1, and 2 respectively.
   - Create a **Code** node named `Detect Leaks` containing the FinOps rule engine JavaScript code. Connect `Wait for All Sources` to it.
   - Create an **If** node named `Any Leaks Found?` evaluating `={{ $json.findingsCount }}` > 0. Connect `Detect Leaks` to it.

6. **Build AI Remediation and Reporting Branches:**
   - For the True branch of `Any Leaks Found?`:
     - Create an **OpenAI** node named `Draft Cancellation Plan`. Set model to `gpt-4o-mini`, temperature to `0.1`, JSON output mode, and prompt rules for generating optimization steps.
     - Create a **Code** node named `Parse AI Plan` to merge detection results with AI recommendations and compute confirmed savings.
   - For the False branch of `Any Leaks Found?`:
     - Create a **No Operation** node named `No Leaks - Done`.

7. **Build Output Destinations:**
   - From `Parse AI Plan`, branch to three output pipelines:
     - **Spreadsheet Logging:** Create a **Code** node named `Build Report Rows` to format rows, followed by a **Google Sheets** node named `Log to Leak Report` set to append mode with document ID `={{ $('Config').first().json.inventorySheetId }}` and sheet name `Leak Report`.
     - **Email Notification:** Create a **Code** node named `Build Email Report` to construct responsive HTML, followed by a **Gmail** node named `Email Finance Report` sending to `={{ $('Config').first().json.reportEmail }}`.
     - **Slack Notification:** Create a **Slack** node named `Slack Alert` set to post text messages to channel `={{ $('Config').first().json.slackChannel }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automated FinOps Audit Template | SaaS Subscription Leak Detector Workflow |
| Required Integrations | Google Drive, Gmail, OpenAI API, Google Sheets, Slack |