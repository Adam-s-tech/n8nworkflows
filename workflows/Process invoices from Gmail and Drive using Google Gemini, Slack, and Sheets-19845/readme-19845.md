Process invoices from Gmail and Drive using Google Gemini, Slack, and Sheets

https://n8nworkflows.xyz/workflows/process-invoices-from-gmail-and-drive-using-google-gemini--slack--and-sheets-19845


# Process invoices from Gmail and Drive using Google Gemini, Slack, and Sheets

### 1. Workflow Overview

This workflow automates the ingestion, validation, extraction, and ledger management of invoices and receipts received via Gmail attachments and Google Drive. It leverages Google Gemini for AI-driven data extraction and expense categorization, validates financial totals and schema completeness, handles duplicate detection against a Google Sheets ledger, and routes high-value invoices through a Slack interactive approval process.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Filtering:** Monitors Gmail and Google Drive for new invoice documents, filters for supported file types (PDFs and images), and processes unsupported formats.
- **1.2 AI Extraction & Validation:** Sends valid files to Google Gemini for structured data extraction, merges sources, parses responses, and validates math and required fields.
- **1.3 Duplicate Check & Categorization:** Verifies whether an invoice already exists in Google Sheets to prevent duplicates, and uses Google Gemini to assign a standardized expense category.
- **1.4 Approval Routing & Ledger:** Evaluates the invoice total against a threshold ($500), logs low-value invoices as auto-approved, and routes high-value invoices to Slack for interactive approval before writing to Google Sheets.
- **1.5 Failure Alerts:** Captures global workflow execution errors and sends an alert message to a designated Slack channel.

---

### 2. Block-by-Block Analysis

---

### 2.1 Input Reception & Filtering
**Overview:** This block polls Gmail and a Google Drive folder every minute for new invoice documents, validates that attachments/files are PDFs or images, marks processed or skipped items accordingly, and tags the data source.

**Nodes Involved:**
- `Watch Invoice Emails`
- `Watch Invoice Drive Folder`
- `Download Drive File`
- `Email Attachment Is PDF or Image?`
- `Drive File Is PDF or Image?`
- `Extract Invoice Data (Email)`
- `Extract Invoice Data (Drive)`
- `Mark Unsupported Email As Read`
- `Notify Unsupported File Type`
- `Tag Source Email`
- `Tag Source Drive`
- `Mark Email As Read`

**Node Details:**

- **Watch Invoice Emails**
  - **Type & Role:** `n8n-nodes-base.gmailTrigger` (Trigger). Polls Gmail every minute for unread messages matching search criteria.
  - **Configuration:** Polls every minute; filter query `q` set to `has:attachment (invoice OR receipt OR bill)` with unread status. Downloads attachments with prefix `attachment_`.
  - **Expressions:** None.
  - **Connections:** Input: None (Trigger); Output: `Email Attachment Is PDF or Image?`.
  - **Edge Cases:** Auth token expiration, rate limiting by Google APIs.

- **Watch Invoice Drive Folder**
  - **Type & Role:** `n8n-nodes-base.googleDriveTrigger` (Trigger). Polls a specific Google Drive folder for newly created files.
  - **Configuration:** Triggers on `fileCreated` every minute.
  - **Expressions:** None.
  - **Connections:** Input: None (Trigger); Output: `Download Drive File`.
  - **Edge Cases:** Missing folder selection, insufficient folder permissions.

- **Download Drive File**
  - **Type & Role:** `n8n-nodes-base.googleDrive` (Action). Downloads the file binary from Google Drive.
  - **Configuration:** Operation: `download`; binary property set to `data`.
  - **Expressions:** File ID: `={{ $json.id }}`
  - **Connections:** Input: `Watch Invoice Drive Folder`; Output: `Drive File Is PDF or Image?`.
  - **Edge Cases:** File deleted before download, permission errors.

- **Email Attachment Is PDF or Image?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Evaluates the mime type of the email attachment.
  - **Configuration:** Strict validation checking if `={{ $binary.attachment_0.mimeType }}` matches regex `^application/pdf|^image/`.
  - **Expressions:** `={{ $binary.attachment_0.mimeType }}`
  - **Connections:** Input: `Watch Invoice Emails`; Outputs: True -> `Extract Invoice Data (Email)`, False -> `Mark Unsupported Email As Read`.
  - **Edge Cases:** Missing attachment property resulting in evaluation errors.

- **Drive File Is PDF or Image?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Evaluates the mime type of the downloaded Drive file.
  - **Configuration:** Loose validation checking if `={{ $binary.data.mimeType }}` matches regex `^application/pdf|^image/`.
  - **Expressions:** `={{ $binary.data.mimeType }}`
  - **Connections:** Input: `Download Drive File`; Outputs: True -> `Extract Invoice Data (Drive)`, False -> `Notify Unsupported File Type`.
  - **Edge Cases:** Undefined binary properties.

- **Extract Invoice Data (Email)**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / Data Extraction). Extracts structured fields from the email attachment binary using Gemini.
  - **Configuration:** Resource: `document`; input type: binary (`attachment_0`); model: `models/gemini-3.5-flash-lite`; max output tokens: 2000. Retries up to 3 times on failure with a 5000ms wait.
  - **Expressions:** Prompt requests JSON extraction of vendor_name, invoice_number, invoice_date, due_date, line_items, subtotal, tax, total, currency, and payment_terms.
  - **Connections:** Input: `Email Attachment Is PDF or Image?`; Output: `Tag Source Email`.
  - **Credentials:** Google Gemini (PaLM) API account.
  - **Edge Cases:** Gemini returning invalid JSON structure, API timeouts.

- **Extract Invoice Data (Drive)**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / Data Extraction). Extracts structured fields from the Drive file binary using Gemini.
  - **Configuration:** Resource: `document`; input type: binary (`data`); model: `models/gemini-3.6-flash`; max output tokens: 2000. Retries up to 3 times on failure with a 5000ms wait.
  - **Expressions:** Prompt requests JSON extraction of standard invoice fields.
  - **Connections:** Input: `Drive File Is PDF or Image?`; Output: `Tag Source Drive`.
  - **Credentials:** Google Gemini (PaLM) API account.
  - **Edge Cases:** Unsupported document structure, API rate limits.

- **Mark Unsupported Email As Read**
  - **Type & Role:** `n8n-nodes-base.gmail` (Action). Marks an unsupported email as read.
  - **Configuration:** Operation: `markAsRead`; error handling set to continue regular output.
  - **Expressions:** Message ID: `={{ $('Watch Invoice Emails').item.json.id }}`
  - **Connections:** Input: `Email Attachment Is PDF or Image?` (False branch); Output: `Notify Unsupported File Type`.
  - **Edge Cases:** Email already read or deleted.

- **Notify Unsupported File Type**
  - **Type & Role:** `n8n-nodes-base.slack` (Action). Sends a notification when an unsupported file type is encountered.
  - **Configuration:** Select: channel.
  - **Expressions:** Text alert.
  - **Connections:** Inputs: `Mark Unsupported Email As Read`, `Drive File Is PDF or Image?` (False branch); Output: None.
  - **Credentials:** Slack account.

- **Tag Source Email**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation). Appends a source metadata tag to the data item.
  - **Configuration:** Assigns `source = "email"`.
  - **Expressions:** None.
  - **Connections:** Input: `Extract Invoice Data (Email)`; Outputs: `Combine Extraction Results`, `Mark Email As Read`.

- **Tag Source Drive**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation). Appends a source metadata tag to the data item.
  - **Configuration:** Assigns `source = "drive"`.
  - **Expressions:** None.
  - **Connections:** Input: `Extract Invoice Data (Drive)`; Output: `Combine Extraction Results`.

- **Mark Email As Read**
  - **Type & Role:** `n8n-nodes-base.gmail` (Action). Marks a successfully processed email as read.
  - **Configuration:** Operation: `markAsRead`; error handling set to continue regular output.
  - **Expressions:** Message ID: `={{ $('Watch Invoice Emails').item.json.id }}`
  - **Connections:** Input: `Tag Source Email`; Output: None.
  - **Edge Cases:** API errors when marking message.

---

### 2.2 AI Extraction & Validation
**Overview:** Merges incoming branches from Gmail and Google Drive, parses raw text outputs, strips markdown formatting, validates JSON integrity, checks arithmetic consistency of financial totals, and routes items for manual review if validation fails.

**Nodes Involved:**
- `Combine Extraction Results`
- `Parse & Validate Extraction`
- `Extraction Valid?`
- `Notify Extraction Needs Review`

**Node Details:**

- **Combine Extraction Results**
  - **Type & Role:** `n8n-nodes-base.merge` (Flow Control). Combines data streams from Gmail and Google Drive intake paths.
  - **Configuration:** Default merge mode.
  - **Expressions:** None.
  - **Connections:** Inputs: `Tag Source Email`, `Tag Source Drive`; Output: `Parse & Validate Extraction`.

- **Parse & Validate Extraction**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation / Scripting). Cleans Gemini raw outputs, parses JSON, validates mathematical accuracy (`subtotal + tax === total`), and checks for required fields.
  - **Configuration:** JavaScript execution environment.
  - **Expressions:** Iterates through `$input.all()`, cleans markdown code blocks, parses JSON, computes validation flags (`isValid`, `needsReview`).
  - **Connections:** Input: `Combine Extraction Results`; Output: `Extraction Valid?`.
  - **Edge Cases:** Malformed JSON strings causing parsing exceptions.

- **Extraction Valid?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Checks whether the extraction passed validation.
  - **Configuration:** Strict boolean validation on `={{ $json.isValid }}` equals `true`.
  - **Expressions:** `={{ $json.isValid }}`
  - **Connections:** Input: `Parse & Validate Extraction`; Outputs: True -> `Check Duplicate Invoice`, False -> `Notify Extraction Needs Review`.

- **Notify Extraction Needs Review**
  - **Type & Role:** `n8n-nodes-base.slack` (Action). Alerts the team via Slack when an invoice fails validation or parsing.
  - **Configuration:** Select: channel.
  - **Expressions:** Formatted message containing vendor name, invoice number, source, and failure reason.
  - **Connections:** Input: `Extraction Valid?` (False branch); Output: None.
  - **Credentials:** Slack account.

---

### 2.3 Duplicate Check & Categorization
**Overview:** Queries the Google Sheets ledger to detect duplicate invoice numbers, notifies if a duplicate exists, and leverages Google Gemini to categorize the expense type.

**Nodes Involved:**
- `Check Duplicate Invoice`
- `Duplicate Found?`
- `Notify Duplicate Invoice`
- `Classify Expense Category`

**Node Details:**

- **Check Duplicate Invoice**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action). Looks up rows in Google Sheets by invoice number.
  - **Configuration:** Operation: lookup; filter column `invoice_number` matched against incoming invoice number; always output data enabled.
  - **Expressions:** Lookup value: `={{ $json.invoice_number }}`
  - **Connections:** Input: `Extraction Valid?` (True branch); Output: `Duplicate Found?`.
  - **Credentials:** Google Sheets account.

- **Duplicate Found?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Determines whether an existing ledger entry matched the invoice number.
  - **Configuration:** Checks if lookup result for `invoice_number` is not empty.
  - **Expressions:** `={{ $json.invoice_number }}`
  - **Connections:** Input: `Check Duplicate Invoice`; Outputs: True -> `Notify Duplicate Invoice`, False -> `Classify Expense Category`.

- **Notify Duplicate Invoice**
  - **Type & Role:** `n8n-nodes-base.slack` (Action). Sends a notification when a duplicate invoice is skipped.
  - **Configuration:** Select: channel.
  - **Expressions:** Message referencing vendor, invoice number, and total using node data traversal (`$('Parse & Validate Extraction')`).
  - **Connections:** Input: `Duplicate Found?` (True branch); Output: None.
  - **Credentials:** Slack account.

- **Classify Expense Category**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / Categorization). Assigns an expense category using Google Gemini.
  - **Configuration:** Model: `models/gemini-3.1-flash-lite`; temperature: 0.1; max output tokens: 20. Retries up to 3 times on failure.
  - **Expressions:** Prompt classifies expense into Software, Utilities, Contractor, Office Supplies, Travel, or Other based on vendor name and line items.
  - **Connections:** Input: `Duplicate Found?` (False branch); Output: `Determine Approval Route`.
  - **Credentials:** Google Gemini (PaLM) API account.

---

### 2.4 Approval Routing & Ledger
**Overview:** Determines if an invoice requires manual approval based on a financial threshold ($500), logs it to Google Sheets as Approved or Pending Review, initiates a Slack interactive approval workflow when needed, and updates the ledger based on the response.

**Nodes Involved:**
- `Determine Approval Route`
- `Needs Approval?`
- `Mark Pending Review`
- `Mark Auto Approved`
- `Log Pending Invoice`
- `Log Invoice To Ledger`
- `Request Manual Approval`
- `Set Approval Decision`
- `Update Ledger Status`

**Node Details:**

- **Determine Approval Route**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation / Scripting). Combines extraction data with Gemini category classification and calculates approval requirements.
  - **Configuration:** JavaScript code block.
  - **Expressions:** Evaluates category text from Gemini, checks if `total > 500`, and sets `needsApproval` boolean flag.
  - **Connections:** Input: `Classify Expense Category`; Output: `Needs Approval?`.

- **Needs Approval?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control). Routes invoices based on the approval threshold.
  - **Configuration:** Strict boolean comparison on `={{ $json.needsApproval }}` equals `true`.
  - **Expressions:** `={{ $json.needsApproval }}`
  - **Connections:** Input: `Determine Approval Route`; Outputs: True -> `Mark Pending Review`, False -> `Mark Auto Approved`.

- **Mark Pending Review**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation). Sets the invoice status to "Pending Review".
  - **Configuration:** Assigns `status = "Pending Review"`.
  - **Expressions:** None.
  - **Connections:** Input: `Needs Approval?` (True branch); Output: `Log Pending Invoice`.

- **Mark Auto Approved**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation). Sets the invoice status to "Approved".
  - **Configuration:** Assigns `status = "Approved"`.
  - **Expressions:** None.
  - **Connections:** Input: `Needs Approval?` (False branch); Output: `Log Invoice To Ledger`.

- **Log Pending Invoice**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action). Appends pending invoice records to Google Sheets.
  - **Configuration:** Operation: `append`; sheet name: `invoices`; error handling set to continue regular output.
  - **Expressions:** Auto-map input data.
  - **Connections:** Input: `Mark Pending Review`; Output: `Request Manual Approval`.
  - **Credentials:** Google Sheets account.

- **Log Invoice To Ledger**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action). Appends auto-approved invoice records to Google Sheets.
  - **Configuration:** Operation: `append`; sheet name: `invoices`; error handling set to continue regular output.
  - **Expressions:** Auto-map input data.
  - **Connections:** Input: `Mark Auto Approved`; Output: None.
  - **Credentials:** Google Sheets account.

- **Request Manual Approval**
  - **Type & Role:** `n8n-nodes-base.slack` (Action). Sends an interactive approval card to Slack with Approve and Reject buttons.
  - **Configuration:** Operation: `sendAndWait`; approval type: double; approve label: `✅ Approve`; disapprove label: `❌ Reject`; limit wait time: 3 days; captures responder.
  - **Expressions:** Message content populated via node traversal from `Mark Pending Review`.
  - **Connections:** Input: `Log Pending Invoice`; Output: `Set Approval Decision`.
  - **Credentials:** Slack account.
  - **Edge Cases:** Slack interaction timeout after 3 days, missing webhook responder URL configuration.

- **Set Approval Decision**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation). Interprets the Slack interaction response and assigns the final status.
  - **Configuration:** Assigns `invoice_number` and computes `status` (`Approved`, `Rejected`, or `No Response`).
  - **Expressions:** `={{ $('Mark Pending Review').item.json.invoice_number }}`, evaluates `$json.data.approved`.
  - **Connections:** Input: `Request Manual Approval`; Output: `Update Ledger Status`.

- **Update Ledger Status**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Action). Updates the ledger row status based on the Slack decision.
  - **Configuration:** Operation: `update`; mapping mode: define below; matching columns: `invoice_number`.
  - **Expressions:** `status = {{ $json.status }}`, `invoice_number = {{ $json.invoice_number }}`.
  - **Connections:** Input: `Set Approval Decision`; Output: None.
  - **Credentials:** Google Sheets account.
  - **Edge Cases:** Row mismatch if invoice number was altered.

---

### 2.5 Failure Alerts
**Overview:** Global error handling block that triggers whenever any node in the workflow fails, posting a failure notification to Slack.

**Nodes Involved:**
- `On Workflow Error`
- `Alert Workflow Failure`

**Node Details:**

- **On Workflow Error**
  - **Type & Role:** `n8n-nodes-base.errorTrigger` (Trigger). Listens for workflow execution failures.
  - **Configuration:** Global error trigger.
  - **Expressions:** None.
  - **Connections:** Input: None (Trigger); Output: `Alert Workflow Failure`.

- **Alert Workflow Failure**
  - **Type & Role:** `n8n-nodes-base.slack` (Action). Posts an error alert to a Slack channel.
  - **Configuration:** Select: channel.
  - **Expressions:** `={{ $json.workflow.name }}`, `={{ $json.execution.id }}`.
  - **Connections:** Input: `On Workflow Error`; Output: None.
  - **Credentials:** Slack account.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | ## AI Invoice & Receipt Processing with Gemini, Slack Approvals & Google Sheets<br><br>### How it works<br>1. Watches Gmail for unread emails with invoice or receipt attachments, and a Google Drive folder for new files.<br>2. Only PDFs and images continue. Gemini reads the file and returns JSON (vendor, invoice number, dates, line items, subtotal, tax, total, currency).<br>3. A Code node validates the data. Missing fields or totals that don't add up are sent to Slack for review.<br>4. The ledger is searched by invoice number, so repeat invoices are skipped.<br>5. Gemini assigns an expense category.<br>6. Invoices up to $500 are logged as *Approved*. Larger ones are logged as *Pending Review* and posted to Slack with **Approve / Reject** buttons. The click updates the row to *Approved* or *Rejected* (*No Response* after 3 days) and replaces the buttons with the outcome.<br>7. An Error Trigger posts a Slack alert if the workflow fails.<br><br>### Setup steps (~15 min)<br>1. Add credentials for Gmail, Google Drive, Google Sheets, Google Gemini and Slack.<br>2. Create a Google Sheet with an `invoices` tab and these headers: `vendor_name, invoice_number, invoice_date, due_date, subtotal, tax, total, currency, category, source, needsApproval, status`. Select it in all four Google Sheets nodes.<br>3. Choose your folder in **Watch Invoice Drive Folder** and your channel in every Slack node.<br>4. For in-Slack approvals, create a Slack app with the `chat:write` scope, enable Interactivity and set the Request URL to `https://<your-n8n-domain>/webhook-waiting-slack`. Put its bot token and Signing Secret (Signature Secret field) in the Slack credential, then invite the bot to your channel.<br>5. In workflow **Settings**, set **Error Workflow** to this workflow so **On Workflow Error** fires.<br>6. Activate the workflow.<br><br>### Customization<br>- Change the $500 limit in **Determine Approval Route**.<br>- Edit the categories in **Classify Expense Category**.<br>- Adjust the search query in **Watch Invoice Emails**. |
| **Note - Intake** | `n8n-nodes-base.stickyNote` | Visual grouping for intake nodes | None | None | ## 1. Intake: Gmail & Google Drive |
| **Note - Extraction** | `n8n-nodes-base.stickyNote` | Visual grouping for extraction and validation nodes | None | None | ## 2. AI extraction & validation |
| **Note - Dedup** | `n8n-nodes-base.stickyNote` | Visual grouping for duplicate check and categorization | None | None | ## 3. Duplicate check & categorisation |
| **Note - Approval** | `n8n-nodes-base.stickyNote` | Visual grouping for approval routing and ledger | None | None | ## 4. Approval routing & ledger |
| **Note - Error Handling** | `n8n-nodes-base.stickyNote` | Visual grouping for failure alerts | None | None | ## 5. Failure alerts |
| **Watch Invoice Emails** | `n8n-nodes-base.gmailTrigger` | Polls Gmail for unread emails with invoice attachments | None | Email Attachment Is PDF or Image? | ## 1. Intake: Gmail & Google Drive |
| **Watch Invoice Drive Folder** | `n8n-nodes-base.googleDriveTrigger` | Polls Google Drive for new files in a specific folder | None | Download Drive File | ## 1. Intake: Gmail & Google Drive |
| **On Workflow Error** | `n8n-nodes-base.errorTrigger` | Triggers when any workflow execution fails | None | Alert Workflow Failure | ## 5. Failure alerts |
| **Download Drive File** | `n8n-nodes-base.googleDrive` | Downloads the binary content of a Google Drive file | Watch Invoice Drive Folder | Drive File Is PDF or Image? | ## 1. Intake: Gmail & Google Drive |
| **Drive File Is PDF or Image?** | `n8n-nodes-base.if` | Validates if the Drive file is a PDF or image | Download Drive File | Extract Invoice Data (Drive), Notify Unsupported File Type | ## 1. Intake: Gmail & Google Drive |
| **Extract Invoice Data (Drive)** | `@n8n/n8n-nodes-langchain.googleGemini` | Extracts structured invoice fields from Drive document binary | Drive File Is PDF or Image? | Tag Source Drive | ## 1. Intake: Gmail & Google Drive |
| **Tag Source Drive** | `n8n-nodes-base.set` | Adds source metadata ('drive') to the item | Extract Invoice Data (Drive) | Combine Extraction Results | ## 1. Intake: Gmail & Google Drive |
| **Notify Unsupported File Type** | `n8n-nodes-base.slack` | Sends Slack notification for skipped unsupported files | Drive File Is PDF or Image?, Mark Unsupported Email As Read | None | ## 1. Intake: Gmail & Google Drive |
| **Email Attachment Is PDF or Image?** | `n8n-nodes-base.if` | Validates if the email attachment is a PDF or image | Watch Invoice Emails | Extract Invoice Data (Email), Mark Unsupported Email As Read | ## 1. Intake: Gmail & Google Drive |
| **Extract Invoice Data (Email)** | `@n8n/n8n-nodes-langchain.googleGemini` | Extracts structured invoice fields from email attachment binary | Email Attachment Is PDF or Image? | Tag Source Email | ## 1. Intake: Gmail & Google Drive |
| **Tag Source Email** | `n8n-nodes-base.set` | Adds source metadata ('email') to the item | Extract Invoice Data (Email) | Combine Extraction Results, Mark Email As Read | ## 1. Intake: Gmail & Google Drive |
| **Mark Email As Read** | `n8n-nodes-base.gmail` | Marks processed Gmail message as read | Tag Source Email | None | ## 1. Intake: Gmail & Google Drive |
| **Mark Unsupported Email As Read** | `n8n-nodes-base.gmail` | Marks unsupported Gmail message as read | Email Attachment Is PDF or Image? | Notify Unsupported File Type | ## 1. Intake: Gmail & Google Drive |
| **Combine Extraction Results** | `n8n-nodes-base.merge` | Merges email and Drive data streams | Tag Source Email, Tag Source Drive | Parse & Validate Extraction | ## 2. AI extraction & validation |
| **Parse & Validate Extraction** | `n8n-nodes-base.code` | Parses Gemini output, cleans JSON, validates totals and required fields | Combine Extraction Results | Extraction Valid? | ## 2. AI extraction & validation |
| **Extraction Valid?** | `n8n-nodes-base.if` | Routes based on whether extraction data is valid | Parse & Validate Extraction | Check Duplicate Invoice, Notify Extraction Needs Review | ## 2. AI extraction & validation |
| **Notify Extraction Needs Review** | `n8n-nodes-base.slack` | Alerts Slack when extraction fails or is incomplete | Extraction Valid? | None | ## 2. AI extraction & validation |
| **Check Duplicate Invoice** | `n8n-nodes-base.googleSheets` | Looks up invoice number in Google Sheets ledger | Extraction Valid? | Duplicate Found? | ## 3. Duplicate check & categorisation |
| **Duplicate Found?** | `n8n-nodes-base.if` | Checks if duplicate invoice exists in ledger | Check Duplicate Invoice | Notify Duplicate Invoice, Classify Expense Category | ## 3. Duplicate check & categorisation |
| **Notify Duplicate Invoice** | `n8n-nodes-base.slack` | Sends Slack notification for skipped duplicate invoices | Duplicate Found? | None | ## 3. Duplicate check & categorisation |
| **Classify Expense Category** | `@n8n/n8n-nodes-langchain.googleGemini` | Assigns an expense category using Gemini | Duplicate Found? | Determine Approval Route | ## 3. Duplicate check & categorisation |
| **Determine Approval Route** | `n8n-nodes-base.code` | Combines classification and checks approval threshold ($500) | Classify Expense Category | Needs Approval? | ## 4. Approval routing & ledger |
| **Needs Approval?** | `n8n-nodes-base.if` | Routes invoices based on total amount threshold | Determine Approval Route | Mark Pending Review, Mark Auto Approved | ## 4. Approval routing & ledger |
| **Mark Pending Review** | `n8n-nodes-base.set` | Sets invoice status to 'Pending Review' | Needs Approval? | Log Pending Invoice | ## 4. Approval routing & ledger |
| **Mark Auto Approved** | `n8n-nodes-base.set` | Sets invoice status to 'Approved' | Needs Approval? | Log Invoice To Ledger | ## 4. Approval routing & ledger |
| **Log Pending Invoice** | `n8n-nodes-base.googleSheets` | Appends pending invoice row to Google Sheets | Mark Pending Review | Request Manual Approval | ## 4. Approval routing & ledger |
| **Log Invoice To Ledger** | `n8n-nodes-base.googleSheets` | Appends approved invoice row to Google Sheets | Mark Auto Approved | None | ## 4. Approval routing & ledger |
| **Request Manual Approval** | `n8n-nodes-base.slack` | Sends interactive approval card with buttons to Slack | Log Pending Invoice | Set Approval Decision | ## 4. Approval routing & ledger |
| **Set Approval Decision** | `n8n-nodes-base.set` | Parses Slack interactive response into final status | Request Manual Approval | Update Ledger Status | ## 4. Approval routing & ledger |
| **Update Ledger Status** | `n8n-nodes-base.googleSheets` | Updates ledger row status based on Slack response | Set Approval Decision | None | ## 4. Approval routing & ledger |
| **Alert Workflow Failure** | `n8n-nodes-base.slack` | Sends Slack alert when workflow execution errors | On Workflow Error | None | ## 5. Failure alerts |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Prerequisites & Credentials:**
   - Create and configure credentials for: **Gmail**, **Google Drive**, **Google Sheets**, **Google Gemini (PaLM) API**, and **Slack**.
   - Prepare a Google Sheet with a worksheet named `invoices` containing headers: `vendor_name`, `invoice_number`, `invoice_date`, `due_date`, `subtotal`, `tax`, `total`, `currency`, `category`, `source`, `needsApproval`, `status`.

2. **Intake Nodes:**
   - **Step 1:** Create a **Gmail Trigger** node (`Watch Invoice Emails`). Set poll time to every minute. Filter query `q` = `has:attachment (invoice OR receipt OR bill)`, read status = unread, download attachments enabled with prefix `attachment_`.
   - **Step 2:** Create a Google Drive Trigger node (`Watch Invoice Drive Folder`). Set event to `fileCreated`, trigger on specific folder, and select your target watch folder.
   - **Step 3:** Create a Google Drive node (`Download Drive File`). Operation: `download`, file ID: `={{ $json.id }}`, binary property: `data`. Connect `Watch Invoice Drive Folder` to this node.
   - **Step 4:** Create an **If** node (`Drive File Is PDF or Image?`). Condition checks regex `^application/pdf|^image/` on `={{ $binary.data.mimeType }}`. Connect `Download Drive File` here.
   - **Step 5:** Create an **If** node (`Email Attachment Is PDF or Image?`). Condition checks regex `^application/pdf|^image/` on `={{ $binary.attachment_0.mimeType }}`. Connect `Watch Invoice Emails` here.

3. **AI Extraction & Source Tagging:**
   - **Step 6:** Create a **Google Gemini** node (`Extract Invoice Data (Email)`). Resource: `document`, input type: binary (`attachment_0`), model: `models/gemini-3.5-flash-lite`, max tokens: 2000, retry on fail. Connect True branch of `Email Attachment Is PDF or Image?`.
   - **Step 7:** Create a **Google Gemini** node (`Extract Invoice Data (Drive)`). Resource: `document`, input type: binary (`data`), model: `models/gemini-3.6-flash`, max tokens: 2000, retry on fail. Connect True branch of `Drive File Is PDF or Image?`.
   - **Step 8:** Create a **Set** node (`Tag Source Email`). Assign `source` = `email`. Connect `Extract Invoice Data (Email)`.
   - **Step 9:** Create a **Set** node (`Tag Source Drive`). Assign `source` = `drive`. Connect `Extract Invoice Data (Drive)`.
   - **Step 10:** Create Gmail nodes (`Mark Email As Read` & `Mark Unsupported Email As Read`) for handling Gmail messages. Connect appropriately to tag and false branches.
   - **Step 11:** Create a **Slack** node (`Notify Unsupported File Type`) for skipped non-PDF/image files.

4. **Validation & Deduplication:**
   - **Step 12:** Create a **Merge** node (`Combine Extraction Results`). Connect `Tag Source Email` and `Tag Source Drive`.
   - **Step 13:** Create a **Code** node (`Parse & Validate Extraction`). Add JavaScript to clean markdown, parse JSON, and compute validation math. Connect from `Combine Extraction Results`.
   - **Step 14:** Create an **If** node (`Extraction Valid?`). Check `={{ $json.isValid === true }}`. Connect a **Slack** node (`Notify Extraction Needs Review`) to the false branch.
   - **Step 15:** Create a **Google Sheets** node (`Check Duplicate Invoice`). Operation: lookup, filter column `invoice_number` matched to `={{ $json.invoice_number }}`. Connect from True branch of `Extraction Valid?`.
   - **Step 16:** Create an **If** node (`Duplicate Found?`). Check if lookup result is not empty. Connect a **Slack** node (`Notify Duplicate Invoice`) to the true branch.

5. **Categorization & Approval Routing:**
   - **Step 17:** Create a **Google Gemini** node (`Classify Expense Category`). Model: `models/gemini-3.1-flash-lite`, temperature: 0.1, prompt categorization. Connect from False branch of `Duplicate Found?`.
   - **Step 18:** Create a **Code** node (`Determine Approval Route`). Combines category classification and checks if `total > 500` to set `needsApproval`. Connect from `Classify Expense Category`.
   - **Step 19:** Create an **If** node (`Needs Approval?`). Check `={{ $json.needsApproval === true }}`.
   - **Step 20:** Create **Set** nodes (`Mark Pending Review` status="Pending Review", `Mark Auto Approved` status="Approved") connected to the respective branches of `Needs Approval?`.
   - **Step 21:** Create **Google Sheets** nodes (`Log Pending Invoice`, `Log Invoice To Ledger`) to append rows to the ledger.
   - **Step 22:** Create a **Slack** node (`Request Manual Approval`). Operation: `sendAndWait`, approval type: double, configure approve/reject labels, limit wait time to 3 days. Connect from `Log Pending Invoice`.
   - **Step 23:** Create a **Set** node (`Set Approval Decision`) to parse Slack response data into approved/rejected/no response status, followed by a **Google Sheets** node (`Update Ledger Status`) configured to update ledger rows by `invoice_number`.

6. **Error Handling:**
   - **Step 24:** Create an **Error Trigger** node (`On Workflow Error`) and connect it to a **Slack** node (`Alert Workflow Failure`) to notify on workflow execution errors. Set this workflow as the error workflow in workflow settings.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Purpose & Architecture** | Complete end-to-end invoice automation system utilizing Gmail/Drive triggers, multimodal AI extraction with Google Gemini, duplicate ledger checking via Google Sheets, and interactive Slack approvals. |
| **Setup & Integration Requirements** | Requires active OAuth2/API credentials for Gmail, Google Drive, Google Sheets, Google Gemini (PaLM), and a configured Slack App with interactivity and bot token scopes (`chat:write`). |