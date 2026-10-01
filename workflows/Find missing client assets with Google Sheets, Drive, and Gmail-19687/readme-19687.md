Find missing client assets with Google Sheets, Drive, and Gmail

https://n8nworkflows.xyz/workflows/find-missing-client-assets-with-google-sheets--drive--and-gmail-19687


# Find missing client assets with Google Sheets, Drive, and Gmail

### 1. Workflow Overview

This workflow is designed for client service and creative agency teams to identify missing project assets, track human reviews, and prepare outbound communication drafts via Gmail. It bridges a Google Sheets project checklist with a staff-only Google Drive intake folder, automatically comparing expected deliverables against submitted files based on strict naming conventions. The system identifies files requiring human approval, detects missing deliverables, and generates batched client email drafts alongside an internal team readiness digest.

The workflow supports two distinct operational modes: a **Demo mode** utilizing built-in mock data for zero-configuration testing, and a **Live mode** connecting directly to real Google Sheets, Google Drive, and Gmail accounts. 

The logic is organized into the following functional blocks:
- **1.1 Initialization and Configuration:** Entry points (manual or scheduled) that feed raw template configurations into a centralized validation layer.
- **1.2 Branching and Data Acquisition:** Evaluates whether to use fictional demo datasets or perform live API calls to Google Sheets, Google Drive, and Gmail.
- **1.3 Evaluation and Readiness Assessment:** A robust JavaScript engine that parses incoming sheet rows and file listings, matches assets using standardized naming conventions, verifies business rules, checks duplicate safeguards/cooldown periods, and compiles consolidated action payloads.
- **1.4 Action Execution and Receipt Logging:** Conditionally executes side effects (creating Gmail drafts or sending internal digests) when permitted, records transaction provider IDs, and appends audit receipts back to Google Sheets.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Initialization and Configuration

#### Overview
This block establishes the execution entry points for the workflow, standardizes global settings via static variable assignments, and strictly validates operational parameters, execution context, and limits before proceeding.

#### Nodes Involved
- Run a preview
- Check every morning
- Edit template settings
- Validate settings

#### Node Details

##### Run a preview
- **Type and Technical Role:** `n8n-nodes-base.manualTrigger` — Acts as a manual entry point for on-demand testing and dry runs.
- **Configuration Choices:** Default configuration with no parameters required.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Outputs to `Edit template settings`.
- **Version-Specific Requirements:** Type Version 1.
- **Edge Cases / Potential Failure Types:** None.

##### Check every morning
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` — Automated cron-like trigger firing daily at 09:00 AM (configured to timezone Asia/Bangkok).
- **Configuration Choices:** Configured to repeat every 1 day at hour 9, minute 0.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Outputs to `Edit template settings`.
- **Version-Specific Requirements:** Type Version 1.2.
- **Edge Cases / Potential Failure Types:** Timezone mismatches or instance downtime during the scheduled execution window.

##### Edit template settings
- **Type and Technical Role:** `n8n-nodes-base.set` — Initializes operational parameters, flags, and resource identifiers (Spreadsheet ID, Drive Folder ID, lookahead thresholds).
- **Configuration Choices:** Sets `dataMode` to `"demo"`, `enableEffects` to `false`, `allowManualEffects` to `false`, mock IDs for spreadsheets and folders, an internal recipient email, lookahead window (3 days), and draft cooldown (3 days).
- **Key Expressions or Variables:** Manual static assignments.
- **Input and Output Connections:** Inputs from `Run a preview` or `Check every morning`; outputs to `Validate settings`.
- **Version-Specific Requirements:** Type Version 3.4.
- **Edge Cases / Potential Failure Types:** Leaving placeholder strings (`REPLACE_SPREADSHEET_ID`) in live mode will trigger validation errors.

##### Validate settings
- **Type and Technical Role:** `n8n-nodes-base.code` — Validates environment modes (`test` vs `production`), ensures date/clock integrity, checks boolean flags, and enforces limits on lookahead and cooldown intervals.
- **Configuration Choices:** JavaScript execution running once for all items using `$input.first().json`, `$now`, and `$execution`.
- **Key Expressions or Variables:** `{{ $input.first().json }}` with runtime bindings for `$now.toISO()`, `$now.toISODate()`, and `$execution.mode`.
- **Input and Output Connections:** Input from `Edit template settings`; output to `Use fictional demo?`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** Throws explicit errors if production runs with demo data mode or if setting configurations fail regex validation checks (e.g., malformed email addresses).

---

### Block 1.2: Branching and Data Acquisition

#### Overview
This block checks whether the user selected the fictional demo setup or live data mode. Depending on the path, it either generates mock records instantly or queries Google Sheets, Google Drive, and Gmail APIs concurrently to retrieve the complete project snapshot.

#### Nodes Involved
- Use fictional demo?
- Load fictional demo
- Read requirements
- Read receipt history
- Check intake folder
- Validate intake folder
- List intake files
- List open draft IDs
- Prepare live inputs

#### Node Details

##### Use fictional demo?
- **Type and Technical Role:** `n8n-nodes-base.if` — Conditional router that checks if `dataMode` equals `"demo"`.
- **Configuration Choices:** Evaluates boolean expression on data mode.
- **Key Expressions or Variables:** `={{ $json.dataMode === "demo" }}`
- **Input and Output Connections:** Input from `Validate settings`; True branch outputs to `Load fictional demo`, False branch outputs to `Read requirements`.
- **Version-Specific Requirements:** Type Version 2.2.
- **Edge Cases / Potential Failure Types:** None.

##### Load fictional demo
- **Type and Technical Role:** `n8n-nodes-base.code` — Generates a fully populated mock dataset of requirements, files, and history logs without requiring external API authentication.
- **Configuration Choices:** Executes standalone JavaScript generating dummy project assets.
- **Key Expressions or Variables:** Uses configuration JSON from the previous node.
- **Input and Output Connections:** Input from `Use fictional demo?` (True); output directly to `Evaluate readiness`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** None.

##### Read requirements
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — HTTP GET request interacting with the Google Sheets API v4 to fetch data from the `Requirements!A:M` range.
- **Configuration Choices:** Uses predefined Google Sheets OAuth2 credentials, requests JSON response format with unformatted values.
- **Key Expressions or Variables:** `={{ 'https://sheets.googleapis.com/v4/spreadsheets/' + $('Validate settings').first().json.spreadsheetId + '/values/Requirements!A:M' }}`
- **Input and Output Connections:** Input from `Use fictional demo?` (False); output to `Read receipt history`.
- **Version-Specific Requirements:** Type Version 4.2.
- **Edge Cases / Potential Failure Types:** Authentication token expiration, incorrect spreadsheet ID, or missing sheet tabs.

##### Read receipt history
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — HTTP GET request fetching historical receipt logs from the `History!A:E` range of the Google Sheet.
- **Configuration Choices:** Uses Google Sheets OAuth2 credentials with rows major dimension.
- **Key Expressions or Variables:** `={{ 'https://sheets.googleapis.com/v4/spreadsheets/' + $('Validate settings').first().json.spreadsheetId + '/values/History!A:E' }}`
- **Input and Output Connections:** Input from `Read requirements`; output to `Check intake folder`.
- **Version-Specific Requirements:** Type Version 4.2.
- **Edge Cases / Potential Failure Types:** Sheet not found or permission restrictions.

##### Check intake folder
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — HTTP GET request querying Google Drive API v3 to retrieve metadata for the designated intake folder.
- **Configuration Choices:** Uses Google Drive OAuth2 credentials, queries file metadata (`id`, `mimeType`, `trashed`, `driveId`), supports shared drives.
- **Key Expressions or Variables:** `={{ 'https://www.googleapis.com/drive/v3/files/' + $('Validate settings').first().json.intakeFolderId }}`
- **Input and Output Connections:** Input from `Read receipt history`; output to `Validate intake folder`.
- **Version-Specific Requirements:** Type Version 4.2.
- **Edge Cases / Potential Failure Types:** Folder trashed, deleted, or lacks read permissions.

##### Validate intake folder
- **Type and Technical Role:** `n8n-nodes-base.code` — Validates the folder metadata response and builds a query object to list all valid child files inside the intake folder.
- **Configuration Choices:** JavaScript execution verifying MIME type (`application/vnd.google-apps.folder`) and constructing a Drive search query string.
- **Key Expressions or Variables:** Reads payload from `Check intake folder` and configuration from `Validate settings`.
- **Input and Output Connections:** Input from `Check intake folder`; output to `List intake files`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** Throws errors if the target ID points to a standard file rather than a folder.

##### List intake files
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — HTTP GET request calling Google Drive API v3 `/files` endpoint using the generated query filter.
- **Configuration Choices:** Uses Google Drive OAuth2 credentials with a JSON query payload.
- **Key Expressions or Variables:** `={{ JSON.stringify($('Validate intake folder').first().json.query) }}`
- **Input and Output Connections:** Input from `Validate intake folder`; output to `List open draft IDs`.
- **Version-Specific Requirements:** Type Version 4.2.
- **Edge Cases / Potential Failure Types:** Pagination limits exceeded (>1,000 files) or incomplete searches.

##### List open draft IDs
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — HTTP GET request calling the Gmail API (`/users/me/drafts`) to retrieve active uncommitted mailbox drafts.
- **Configuration Choices:** Uses Gmail OAuth2 credentials, limits results to 500 drafts, requests ID fields and next page tokens.
- **Key Expressions or Variables:** None (static endpoint).
- **Input and Output Connections:** Input from `List intake files`; output to `Prepare live inputs`.
- **Version-Specific Requirements:** Type Version 4.2.
- **Edge Cases / Potential Failure Types:** Mailbox pagination limits exceeded or API scope missing.

##### Prepare live inputs
- **Type and Technical Role:** `n8n-nodes-base.code` — Normalizes and validates API responses from Google Sheets, Google Drive, and Gmail into a cohesive payload structure.
- **Configuration Choices:** JavaScript execution validating array structures, header alignments, and pagination tokens.
- **Key Expressions or Variables:** Pulls data from `List open draft IDs`, `List intake files`, `Read requirements`, and `Read receipt history`.
- **Input and Output Connections:** Input from `List open draft IDs`; output to `Evaluate readiness`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** Throws explicit errors on header mismatches or truncated/paginated file/draft lists.

---

### Block 1.3: Evaluation and Readiness Assessment

#### Overview
This block executes the core evaluation logic. It parses requirements, validates file naming conventions (`project_id__requirement_id__description.ext`), maps file versions against recorded human approvals, calculates project readiness, detects duplicate prevention rules, and builds planned actions for client drafts and internal team digests.

#### Nodes Involved
- Evaluate readiness
- Effects enabled and planned?
- Preview and run report

#### Node Details

##### Evaluate readiness
- **Type and Technical Role:** `n8n-nodes-base.code` — Comprehensive JavaScript business logic engine that processes items, enforces limits, tracks history logs, computes state statuses (`approved`, `missing`, `needs_changes`, `awaiting_review`), and outputs action lists.
- **Configuration Choices:** Standalone execution once for all items containing helper functions for date parsing, email regex validation, history parsing, requirement matching, and draft building.
- **Key Expressions or Variables:** Processes inputs from either `Load fictional demo` or `Prepare live inputs`.
- **Input and Output Connections:** Input from either demo or live preparation nodes; output to `Effects enabled and planned?`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** Enforces strict limits (max 200 requirements, 1,000 files, 5,000 history rows, 20 clients). Exceeding limits throws an immediate error.

##### Effects enabled and planned?
- **Type and Technical Role:** `n8n-nodes-base.if` — Conditional router checking whether external effects are permitted (`effectsAllowed === true`) and if there are pending actions to execute (`actions.length > 0`).
- **Configuration Choices:** Evaluates boolean condition on combined properties.
- **Key Expressions or Variables:** `={{ $json.effectsAllowed && $json.actions.length > 0 }}`
- **Input and Output Connections:** Input from `Evaluate readiness`; True branch outputs to `Expand approved actions`, False branch outputs to `Preview and run report`.
- **Version-Specific Requirements:** Type Version 2.2.
- **Edge Cases / Potential Failure Types:** None.

##### Preview and run report
- **Type and Technical Role:** `n8n-nodes-base.code` — Generates a final summary object detailing scan results, error logs, warnings, suppression reasons, and readiness metrics without writing or sending external communications.
- **Configuration Choices:** JavaScript execution filtering evaluation outputs into a clean reporting structure.
- **Key Expressions or Variables:** References output data from `Evaluate readiness`.
- **Input and Output Connections:** Input from `Effects enabled and planned?` (False).
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** None.

---

### Block 1.4: Action Execution and Receipt Logging

#### Overview
When side effects are active, this block expands the planned action queue, processes items sequentially (batch size 1), routes them to either create a client draft or send an internal digest via Gmail, captures the provider acknowledgement receipt, and appends an audit log entry to the Google Sheets History tab.

#### Nodes Involved
- Expand approved actions
- Process one action
- Completed run report
- Create a client draft?
- Create client draft
- Send internal digest
- Record receipt
- Append acknowledged receipt

#### Node Details

##### Expand approved actions
- **Type and Technical Role:** `n8n-nodes-base.code` — Flattens the array of planned actions into individual items for sequential processing.
- **Configuration Choices:** JavaScript mapping over `$input.first().json.actions`.
- **Key Expressions or Variables:** `{{ $input.first().json.actions }}`
- **Input and Output Connections:** Input from `Effects enabled and planned?` (True); output to `Process one action`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** None.

##### Process one action
- **Type and Technical Role:** `n8n-nodes-base.splitInBatches` — Flow controller that processes items sequentially one by one to prevent rate limits and race conditions during API writes.
- **Configuration Choices:** Batch size set to 1.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input from `Expand approved actions` or `Append acknowledged receipt`; outputs loop back or proceed to completion depending on batch state. Output 0 goes to `Completed run report`, Output 1 goes to `Create a client draft?`.
- **Version-Specific Requirements:** Type Version 3.
- **Edge Cases / Potential Failure Types:** Infinite loops if batch indexing is mismanaged, though handled natively by n8n split-in-batches logic.

##### Completed run report
- **Type and Technical Role:** `n8n-nodes-base.code` — Summarizes completed actions after the batch queue finishes execution.
- **Configuration Choices:** JavaScript extracting acknowledgement counts and scan metrics.
- **Key Expressions or Variables:** References output from `Evaluate readiness`.
- **Input and Output Connections:** Input from `Process one action` (when batch completes); terminal node.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** None.

##### Create a client draft?
- **Type and Technical Role:** `n8n-nodes-base.if` — Conditional router checking if the current action kind equals `"draft"`.
- **Configuration Choices:** Evaluates string equality on `$json.kind`.
- **Key Expressions or Variables:** `={{ $json.kind === "draft" }}`
- **Input and Output Connections:** Input from `Process one action`; True branch outputs to `Create client draft`, False branch outputs to `Send internal digest`.
- **Version-Specific Requirements:** Type Version 2.2.
- **Edge Cases / Potential Failure Types:** None.

##### Create client draft
- **Type and Technical Role:** `n8n-nodes-base.gmail` — Gmail integration node configured to create a text-format draft message in the user's mailbox.
- **Configuration Choices:** Resource: `draft`, Operation: `create`, Email Type: `text`, recipient mapped from `$json.recipient`, subject and body mapped from payload.
- **Key Expressions or Variables:** 
  - Send To: `={{ $json.recipient }}`
  - Subject: `={{ $json.subject }}`
  - Message: `={{ $json.body }}`
- **Input and Output Connections:** Input from `Create a client draft?` (True); output to `Record receipt`.
- **Version-Specific Requirements:** Type Version 2.1.
- **Edge Cases / Potential Failure Types:** Gmail OAuth authentication failure, quota exhaustion, or invalid recipient formatting.

##### Send internal digest
- **Type and Technical Role:** `n8n-nodes-base.gmail` — Gmail integration node configured to send an email message containing the internal readiness digest.
- **Configuration Choices:** Resource: `message`, Operation: `send`, Email Type: `text`, attribution appending disabled.
- **Key Expressions or Variables:** 
  - Send To: `={{ $json.recipient }}`
  - Subject: `={{ $json.subject }}`
  - Message: `={{ $json.body }}`
- **Input and Output Connections:** Input from `Create a client draft?` (False); output to `Record receipt`.
- **Version-Specific Requirements:** Type Version 2.1.
- **Edge Cases / Potential Failure Types:** Authentication failures or sending to unverified external domains in test mode.

##### Record receipt
- **Type and Technical Role:** `n8n-nodes-base.code` — Normalizes Gmail provider responses into structured history audit records containing event type, recipient, event key, provider ID, and ISO timestamp.
- **Configuration Choices:** JavaScript execution running once for each item, parsing response ID from Gmail API output.
- **Key Expressions or Variables:** References `$('Process one action').item.json`, `$json` (Gmail response), and `$now.toISO()`.
- **Input and Output Connections:** Input from either `Create client draft` or `Send internal digest`; output to `Append acknowledged receipt`.
- **Version-Specific Requirements:** Type Version 2.
- **Edge Cases / Potential Failure Types:** Throws an error if the Gmail response lacks a valid provider message/draft ID.

##### Append acknowledged receipt
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` — Appends the normalized audit receipt row into the `History` sheet tab.
- **Configuration Choices:** Resource: `sheet`, Operation: `append`, sheet name: `History`, cell format: `RAW`, mapping mode: `autoMapInputData`. Document ID retrieved dynamically from validation settings.
- **Key Expressions or Variables:** Document ID: `={{ $('Validate settings').first().json.spreadsheetId }}`
- **Input and Output Connections:** Input from `Record receipt`; output loops back to `Process one action` to continue the batch queue.
- **Version-Specific Requirements:** Type Version 4.6.
- **Edge Cases / Potential Failure Types:** Google Sheets write locks, rate limiting, or column misalignment.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Run a preview** | `n8n-nodes-base.manualTrigger` | Manual trigger for dry runs | None | Edit template settings | Start here — free client asset tracker |
| **Check every morning** | `n8n-nodes-base.scheduleTrigger` | Automated daily trigger at 09:00 | None | Edit template settings | Start here — free client asset tracker |
| **Edit template settings** | `n8n-nodes-base.set` | Initializes global variables and IDs | Run a preview, Check every morning | Validate settings | 1. Configure and preview |
| **Validate settings** | `n8n-nodes-base.code` | Validates environment and config rules | Edit template settings | Use fictional demo? | 1. Configure and preview |
| **Use fictional demo?** | `n8n-nodes-base.if` | Routes demo vs live execution | Validate settings | Load fictional demo, Read requirements | 1. Configure and preview |
| **Load fictional demo** | `n8n-nodes-base.code` | Generates mock data structures | Use fictional demo? | Evaluate readiness | 1. Configure and preview |
| **Read requirements** | `n8n-nodes-base.httpRequest` | Fetches Requirements sheet tab | Use fictional demo? | Read receipt history | 2. Read a complete snapshot |
| **Read receipt history** | `n8n-nodes-base.httpRequest` | Fetches History audit tab | Read requirements | Check intake folder | 2. Read a complete snapshot |
| **Check intake folder** | `n8n-nodes-base.httpRequest` | Retrieves Google Drive folder metadata | Read receipt history | Validate intake folder | 2. Read a complete snapshot |
| **Validate intake folder** | `n8n-nodes-base.code` | Validates folder and builds search query | Check intake folder | List intake files | 2. Read a complete snapshot |
| **List intake files** | `n8n-nodes-base.httpRequest` | Lists direct files in intake folder | Validate intake folder | List open draft IDs | 2. Read a complete snapshot |
| **List open draft IDs** | `n8n-nodes-base.httpRequest` | Fetches existing Gmail draft IDs | List intake files | Prepare live inputs | 2. Read a complete snapshot |
| **Prepare live inputs** | `n8n-nodes-base.code` | Normalizes live API responses | List open draft IDs | Evaluate readiness | 2. Read a complete snapshot |
| **Evaluate readiness** | `n8n-nodes-base.code` | Core business logic and readiness evaluator | Load fictional demo, Prepare live inputs | Effects enabled and planned? | 3. Decide what needs attention |
| **Effects enabled and planned?** | `n8n-nodes-base.if` | Checks if side effects are permitted | Evaluate readiness | Expand approved actions, Preview and run report | 3. Decide what needs attention |
| **Preview and run report** | `n8n-nodes-base.code` | Outputs dry-run diagnostic report | Effects enabled and planned? | None | 3. Decide what needs attention |
| **Expand approved actions** | `n8n-nodes-base.code` | Flattens action queue for batch processing | Effects enabled and planned? | Process one action | 4. Drafts, internal email and receipts |
| **Process one action** | `n8n-nodes-base.splitInBatches` | Sequential batch processor (batch size 1) | Expand approved actions, Append acknowledged receipt | Completed run report, Create a client draft? | 4. Drafts, internal email and receipts |
| **Completed run report** | `n8n-nodes-base.code` | Summarizes completed execution run | Process one action | None | 4. Drafts, internal email and receipts |
| **Create a client draft?** | `n8n-nodes-base.if` | Routes draft creation vs digest mailing | Process one action | Create client draft, Send internal digest | 4. Drafts, internal email and receipts |
| **Create client draft** | `n8n-nodes-base.gmail` | Creates Gmail draft for client | Create a client draft? | Record receipt | 4. Drafts, internal email and receipts |
| **Send internal digest** | `n8n-nodes-base.gmail` | Sends team digest email | Create a client draft? | Record receipt | 4. Drafts, internal email and receipts |
| **Record receipt** | `n8n-nodes-base.code` | Normalizes API receipt response | Create client draft, Send internal digest | Append acknowledged receipt | 4. Drafts, internal email and receipts |
| **Append acknowledged receipt** | `n8n-nodes-base.googleSheets` | Appends receipt into History sheet | Record receipt | Process one action | 4. Drafts, internal email and receipts |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in an n8n instance:

#### Step 1: Create Triggers and Configuration Nodes
1. Create a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Run a preview`.
2. Create a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) named `Check every morning`. Configure interval to 1 day at `09:00` (Timezone: `Asia/Bangkok`).
3. Create a **Set** node (`n8n-nodes-base.set`) named `Edit template settings`. Connect both triggers to this node. Configure assignments:
   - `dataMode` (String): `demo`
   - `enableEffects` (Boolean): `false`
   - `allowManualEffects` (Boolean): `false`
   - `spreadsheetId` (String): `REPLACE_SPREADSHEET_ID`
   - `intakeFolderId` (String): `REPLACE_INTAKE_FOLDER_ID`
   - `internalRecipient` (String): `user@example.com`
   - `lookaheadDays` (Number): `3`
   - `draftCooldownDays` (Number): `3`
4. Create a **Code** node (`n8n-nodes-base.code`) named `Validate settings`. Connect `Edit template settings` to it. (Use the validation JavaScript provided in the source workflow).

#### Step 2: Implement Data Acquisition Branching
1. Create an **If** node (`n8n-nodes-base.if`) named `Use fictional demo?`. Connect `Validate settings` to it. Set condition: `{{ $json.dataMode === "demo" }}`.
2. **True Branch (Demo):** Create a **Code** node named `Load fictional demo`. Connect the True output of `Use fictional demo?` to it.
3. **False Branch (Live):** 
   - Create an **HTTP Request** node named `Read requirements`. Connect the False output of `Use fictional demo?` to it. Configure GET request to Google Sheets API using predefined credential `googleSheetsOAuth2Api`.
   - Create an **HTTP Request** node named `Read receipt history`. Connect `Read requirements` to it. Configure GET request for History tab.
   - Create an **HTTP Request** node named `Check intake folder`. Connect `Read receipt history` to it. Configure GET request to Google Drive API using credential `googleDriveOAuth2Api`.
   - Create a **Code** node named `Validate intake folder`. Connect `Check intake folder` to it.
   - Create an **HTTP Request** node named `List intake files`. Connect `Validate intake folder` to it. Configure GET request to Google Drive API.
   - Create an **HTTP Request** node named `List open draft IDs`. Connect `List intake files` to it. Configure GET request to Gmail API using credential `gmailOAuth2`.
   - Create a **Code** node named `Prepare live inputs`. Connect `List open draft IDs` to it.

#### Step 3: Implement the Readiness Engine
1. Create a **Code** node named `Evaluate readiness`. Connect the output of both `Load fictional demo` and `Prepare live inputs` into this node. (Paste the core evaluation JavaScript logic).
2. Create an **If** node named `Effects enabled and planned?`. Connect `Evaluate readiness` to it. Set condition: `{{ $json.effectsAllowed && $json.actions.length > 0 }}`.
3. **False Output:** Create a **Code** node named `Preview and run report`. Connect the False output here.

#### Step 4: Implement Action Execution and Side Effects
1. **True Output:** Connect the True output of `Effects enabled and planned?` to a **Code** node named `Expand approved actions`.
2. Create a **Split In Batches** node named `Process one action` with batch size `1`. Connect `Expand approved actions` to it.
3. Connect Output 0 of `Process one action` to a **Code** node named `Completed run report`.
4. Connect Output 1 of `Process one action` to an **If** node named `Create a client draft?`. Set condition: `{{ $json.kind === "draft" }}`.
5. **True Output of Draft Check:** Create a **Gmail** node named `Create client draft`. Configure Resource: `draft`, Operation: `create`, Send To: `{{ $json.recipient }}`, Subject: `={{ $json.subject }}`, Message: `={{ $json.body }}` using credential `gmailOAuth2`.
6. **False Output of Draft Check:** Create a **Gmail** node named `Send internal digest`. Configure Resource: `message`, Operation: `send`, Send To: `{{ $json.recipient }}`, Subject: `={{ $json.subject }}`, Message: `={{ $json.body }}` using credential `gmailOAuth2`.
7. Create a **Code** node named `Record receipt`. Connect both Gmail nodes to this node.
8. Create a **Google Sheets** node named `Append acknowledged receipt`. Connect `Record receipt` to it. Configure Resource: `sheet`, Operation: `append`, Document ID: `={{ $('Validate settings').first().json.spreadsheetId }}`, Sheet Name: `History`.
9. Connect the output of `Append acknowledged receipt` back to the input of `Process one action` to close the batch loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Spreadsheet Structure Setup** | Create a Google Sheets spreadsheet with two tabs: `Requirements` and `History`. Populate headers and seed rows according to canvas instructions. |
| **Drive Folder Configuration** | Use a single, flat, staff-only Google Drive intake folder. Name files strictly using the `project_id__requirement_id__description.ext` convention. |
| **Credentials Required** | Requires pre-configured OAuth2 credentials in n8n for **Google Sheets**, **Google Drive**, and **Gmail**. Ensure the same service account/user profile is used across read and write operations for each service. |