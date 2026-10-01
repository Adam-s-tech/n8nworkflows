Approve Slack expense receipts and post invoices to Xero with Textract and OpenAI

https://n8nworkflows.xyz/workflows/approve-slack-expense-receipts-and-post-invoices-to-xero-with-textract-and-openai-20018


# Approve Slack expense receipts and post invoices to Xero with Textract and OpenAI

### 1. Workflow Overview

This workflow automates the end-to-end processing, validation, management approval, and accounting integration of expense receipts shared via Slack. When an employee posts a receipt image in Slack, the system extracts critical financial data using AWS Textract, classifies the expense category utilizing OpenAI, checks extraction confidence thresholds, requests manager approval via Slack, and upon approval, automatically generates an authorized ACCPAY invoice in Xero with the original receipt image attached and logs the transaction in Google Sheets. Rejected expenses trigger a user notification and are logged accordingly.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Filtering:** Captures files shared in Slack and filters for valid receipt images.
- **1.2 AI & OCR Extraction:** Analyzes the receipt image using AWS Textract, extracts key summary fields, and passes them to OpenAI for categorization.
- **1.3 Normalization & Validation:** Combines Textract and OpenAI outputs into a structured schema and validates confidence scores to decide between requesting a clearer photo or proceeding to manager approval.
- **1.4 Approval Workflow:** Fetches manager details, sends an interactive Slack approval request, and handles approval or rejection branching.
- **1.5 Xero Accounting Integration & Auditing:** Maps categories to Xero account codes, routes transactions as bills or expense claims, posts to Xero, attaches the receipt image, and records outcomes in Google Sheets.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Filtering
- **Overview:** Receives incoming messages and files from Slack, filtering out non-image uploads to ensure only valid receipt graphics proceed to processing.
- **Nodes Involved:** 
  - `When Slack Message Received`
  - `Filter Receipt Images`

- **Node Details:**
  - **When Slack Message Received**
    - *Type & Technical Role:* `n8n-nodes-base.slackTrigger` (Trigger)
    - *Configuration Choices:* Watches the workspace for `file_share` events and downloads shared files as binary data.
    - *Key Expressions/Variables:* Uses webhook identifier `43a83d31-9e01-4327-af48-c3a11bd29f82`.
    - *Input/Output Connections:* Input: None (Trigger); Output: `Filter Receipt Images`.
    - *Version-specific Requirements:* Requires Slack App bot token with `files:read` and `channels:history` scopes.
    - *Edge Cases / Potential Failures:* Rate limiting by Slack or missing file download permissions.

  - **Filter Receipt Images**
    - *Type & Technical Role:* `n8n-nodes-base.filter` (Flow Control)
    - *Configuration Choices:* Evaluates conditions where `{{ $json.files?.[0]?.url_private }}` is not empty and `{{ $json.files?.[0]?.mimetype }}` starts with `image/`.
    - *Key Expressions/Variables:* `={{ $json.files?.[0]?.url_private }}` (Not Empty), `={{ $json.files?.[0]?.mimetype }}` (Starts with `image/`).
    - *Input/Output Connections:* Input: `When Slack Message Received`; Output: `Textract Expense Analysis`.
    - *Version-specific Requirements:* Type Version 2.3.
    - *Edge Cases / Potential Failures:* Non-image files (e.g., PDFs, text documents) dropped silently.

---

#### Block 1.2: AI & OCR Extraction
- **Overview:** Submits the binary receipt image to AWS Textract for expense OCR analysis, parses key-value pairs, and classifies the expense type via OpenAI.
- **Nodes Involved:**
  - `Textract Expense Analysis`
  - `Extract Textract Data`
  - `AI Expense Categorization`

- **Node Details:**
  - **Textract Expense Analysis**
    - *Type & Technical Role:* `n8n-nodes-base.awsTextract` (AI / OCR Service)
    - *Configuration Choices:* Uses `file_0` binary property name and invokes the `AnalyzeExpense` operation.
    - *Key Expressions/Variables:* Binary property: `file_0`.
    - *Input/Output Connections:* Input: `Filter Receipt Images`; Output: `Extract Textract Data`.
    - *Edge Cases / Potential Failures:* AWS IAM credential permission errors, unsupported image encodings, or service timeouts.

  - **Extract Textract Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Iterates through `ExpenseDocuments` summary fields to extract vendor name, confidence scores, amount, date, and currency.
    - *Key Expressions/Variables:* Custom JavaScript parsing `SummaryFields` looking for types: `VENDOR_NAME`, `TOTAL`, `INVOICE_RECEIPT_DATE`, and `CURRENCY`.
    - *Input/Output Connections:* Input: `Textract Expense Analysis`; Output: `AI Expense Categorization`.
    - *Edge Cases / Potential Failures:* Unstructured or missing fields resulting in `null` values.

  - **AI Expense Categorization**
    - *Type & Technical Role:* `n8n-nodes-base.openAi` (AI / LLM Service)
    - *Configuration Choices:* Uses OpenAI Chat resource with a strict prompt limiting output strictly to one category from a predefined list (`Travel`, `Meals`, `Office Supplies`, `Software`, `Client Entertainment`, `Other`). Temperature set to `0`.
    - *Key Expressions/Variables:* Prompt references `{{ $json.vendor }}` and `{{ $json.amount }}`.
    - *Input/Output Connections:* Input: `Extract Textract Data`; Output: `Normalize Expense Data`.
    - *Edge Cases / Potential Failures:* OpenAI API key limits, insufficient credits, or unexpected chat completion formats.

---

#### Block 1.3: Normalization & Validation
- **Overview:** Combines AWS OCR results with OpenAI classification into a single normalized data structure, verifying confidence scores to decide if human intervention is required.
- **Nodes Involved:**
  - `Normalize Expense Data`
  - `Check Validation Confidence`

- **Node Details:**
  - **Normalize Expense Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Sanitizes the category string, parses numeric amounts, extracts Slack metadata, and evaluates a boolean `confident` status flag.
    - *Key Expressions/Variables:* Evaluates vendor confidence > 80, amount confidence > 80, and presence of a valid parsed amount.
    - *Input/Output Connections:* Input: `AI Expense Categorization`; Output: `Check Validation Confidence`.
    - *Edge Cases / Potential Failures:* Regex string sanitization failures on malformed currency strings.

  - **Check Validation Confidence**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether `{{$json.confident}}` equals `true`.
    - *Key Expressions/Variables:* Left value: `={{$json.confident}}`, Operator: Equals `true`.
    - *Input/Output Connections:* Input: `Normalize Expense Data`; Output True: `Read Manager from Sheets`; Output False: `Request Clearer Photo in Slack`.
    - *Edge Cases / Potential Failures:* Boolean type coercion mismatches.

---

#### Block 1.4: Approval Workflow
- **Overview:** Handles low-confidence rejections by requesting a clearer photo in Slack, or retrieves manager context and dispatches an interactive approval request requiring managerial sign-off.
- **Nodes Involved:**
  - `Request Clearer Photo in Slack`
  - `Read Manager from Sheets`
  - `Request Approval via Slack`
  - `If Approval Granted`
  - `Announce Rejection in Slack`
  - `Append Rejected to Sheets`

- **Node Details:**
  - **Request Clearer Photo in Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging)
    - *Configuration Choices:* Sends a static text alert to the designated Slack channel (`C0C0UEMPD5E`) asking for a clearer upload.
    - *Input/Output Connections:* Input: `Check Validation Confidence` (False branch); Output: None (Terminal node for low-confidence branch).

  - **Read Manager from Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Store)
    - *Configuration Choices:* Reads rows from Google Document ID `1yozEO983mOlmmCm0ujIVUsU46yop5D6HsN-VvL0dU2w`, sheet tab `ExpenseAudit`.
    - *Input/Output Connections:* Input: `Check Validation Confidence` (True branch); Output: `Request Approval via Slack`.

  - **Request Approval via Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Approval Workflow)
    - *Configuration Choices:* Uses the `sendAndWait` operation with double-approval options. Automatically pauses workflow execution until interactive buttons in Slack are clicked.
    - *Key Expressions/Variables:* Message interpolates submitter mention, vendor, amount, currency, and category from upstream nodes.
    - *Input/Output Connections:* Input: `Read Manager from Sheets`; Output: `If Approval Granted`.
    - *Edge Cases / Potential Failures:* Workflow timeout if managers do not respond within configured Slack app limitations.

  - **If Approval Granted**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Checks `={{$json.data.approved}}` equals `true`.
    - *Input/Output Connections:* Input: `Request Approval via Slack`; Output True: `Load Approval Data`; Output False: `Announce Rejection in Slack`.

  - **Announce Rejection in Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging)
    - *Configuration Choices:* Sends a direct message to the user (`slackUserId`) notifying them that their receipt was rejected.
    - *Key Expressions/Variables:* Uses `slackUserId` and receipt details.
    - *Input/Output Connections:* Input: `If Approval Granted` (False branch); Output: `Append Rejected to Sheets`.

  - **Append Rejected to Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Store)
    - *Configuration Choices:* Appends rejection audit records to the `ExpenseAudit` sheet tab.
    - *Input/Output Connections:* Input: `Announce Rejection in Slack`; Output: None.

---

#### Block 1.5: Xero Accounting Integration & Auditing
- **Overview:** Maps approved expenses to correct Xero account codes, routes records as either bills or expense claims, submits them to Xero, reattaches the original receipt image, and records successful transactions in Google Sheets.
- **Nodes Involved:**
  - `Load Approval Data`
  - `Route by Expense Type`
  - `Submit Expense Claim to Xero`
  - `Submit Bill to Xero`
  - `Rebuild Receipt Binary`
  - `Link Receipt to Xero Invoice`
  - `Append Approved to Sheets`

- **Node Details:**
  - **Load Approval Data**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Sanitizes vendor/category strings and evaluates mapping rules for Xero account codes (e.g., Travel -> `493`, Meals/Client Entertainment -> `420`, Software -> `485`, Office Supplies -> `453`, Default -> `429`). Determines routing destination (`expense` vs `bill`).
    - *Input/Output Connections:* Input: `If Approval Granted` (True branch); Output: `Route by Expense Type`.

  - **Route by Expense Type**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control)
    - *Configuration Choices:* Evaluates expression `={{ $json.route === 'expense' ? 0 : 1 }}` across 2 outputs.
    - *Input/Output Connections:* Input: `Load Approval Data`; Output 0: `Submit Expense Claim to Xero`; Output 1: `Submit Bill to Xero`.

  - **Submit Expense Claim to Xero**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration Choices:* `PUT` request to `https://api.xero.com/api.xro/2.0/Invoices` with JSON payload configuring ACCPAY invoice, tax types (`INPUT`), and account codes. Includes required Xero Tenant ID header (`2456a90a-71ac-4df1-a5fb-fb0234672663`).
    - *Input/Output Connections:* Input: `Route by Expense Type` (Output 0); Output: `Rebuild Receipt Binary`.
    - *Edge Cases / Potential Failures:* Xero OAuth token expiration or tenant ID mismatch.

  - **Submit Bill to Xero**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration Choices:* Identical payload structure and HTTP method (`PUT`) to Xero Invoices endpoint, serving bill routing logic.
    - *Input/Output Connections:* Input: `Route by Expense Type` (Output 1); Output: `Rebuild Receipt Binary`.

  - **Rebuild Receipt Binary**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Reattaches binary file payload from the original Slack trigger node to the incoming invoice JSON payload.
    - *Input/Output Connections:* Input: `Submit Expense Claim to Xero` / `Submit Bill to Xero`; Output: `Link Receipt to Xero Invoice`.

  - **Link Receipt to Xero Invoice**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration Choices:* `POST` request to Xero Attachment endpoint using `file_0` input data field name.
    - *Key Expressions/Variables:* Dynamically extracts `InvoiceID` and encodes filename.
    - *Input/Output Connections:* Input: `Rebuild Receipt Binary`; Output: `Append Approved to Sheets`.

  - **Append Approved to Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Store)
    - *Configuration Choices:* Appends successful audit row data to Google Sheet `n8n poc` under tab `ExpenseAudit`.
    - *Input/Output Connections:* Input: `Link Receipt to Xero Invoice`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Canvas documentation | None | None | ## Expense Receipt Approval (Slack -> Textract -> Manager Approval -> Xero)<br><br>### How it works<br><br>This workflow starts when a receipt image is posted in Slack, verifies that it is an image, and uses AWS Textract plus GPT to extract, categorize, normalize, and validate the expense details. Valid receipts are routed to the employee’s manager for Slack approval, while unclear submissions receive a request for a better photo. Approved expenses are converted into either a Xero bill or expense claim, the receipt is attached to the Xero record, and the outcome is logged; rejected expenses are also notified and logged.<br><br>### Setup steps<br><br>- Configure the Slack Trigger with the correct workspace, app permissions, event subscription, and channel or file-upload source for receipt images.<br>- Add AWS credentials with permission to run Textract AnalyzeExpense on uploaded receipt files.<br>- Configure OpenAI credentials for the GPT categorization step and review the prompt/category mapping used by the node.<br>- Connect Google Sheets credentials and prepare sheets for manager lookup plus approved/rejected audit logging with the expected columns.<br>- Configure Xero API/OAuth access for the HTTP Request nodes, including tenant ID, authorization headers, account codes, tax rates, and endpoints for bills, expense claims, and attachments.<br>- Verify the custom Code nodes preserve required fields such as employee, amount, date, merchant, category, Slack file binary data, and Xero record IDs across branches.<br><br>### Customization<br><br>Adjust the validation rules, GPT expense categories, manager lookup sheet, approval message format, and the switch logic that decides whether an approved item becomes a Xero bill or an expense claim. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Capture Slack receipt<br><br>Receives new Slack activity and filters the incoming item so only receipt image uploads continue into processing. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Extract receipt fields<br><br>Runs AWS Textract AnalyzeExpense on the receipt image and parses the returned OCR expense fields into structured data. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Categorize and normalize<br><br>Uses GPT to classify the expense, then merges AI output with extracted receipt data into a normalized record for downstream checks. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Validate and request approval<br><br>Checks whether the receipt data is complete enough to proceed, asks the submitter for a clearer photo when validation fails, or looks up the manager and sends a Slack approval request when valid. |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Prepare Xero routing<br><br>Loads the approved expense context and decides whether the approved item should be handled as a Xero expense claim or a bill. |
| **Sticky Note6** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Create Xero record<br><br>Creates the appropriate financial record in Xero, either an expense claim or a bill, based on the routing decision. |
| **Sticky Note7** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Attach receipt and log<br><br>Restores the receipt binary, attaches the image to the created Xero record, and records the approved transaction in Google Sheets. |
| **Sticky Note8** | `n8n-nodes-base.stickyNote` | Section documentation | None | None | ## Handle rejected expenses<br><br>Sends a Slack rejection notification and logs the rejected expense in Google Sheets in a separate lower branch of the canvas. |
| **When Slack Message Received** | `n8n-nodes-base.slackTrigger` | Trigger for Slack file shares | None | Filter Receipt Images | |
| **Filter Receipt Images** | `n8n-nodes-base.filter` | Filters non-image files | When Slack Message Received | Textract Expense Analysis | |
| **Textract Expense Analysis** | `n8n-nodes-base.awsTextract` | Extracts OCR expense data | Filter Receipt Images | Extract Textract Data | Uses binary file_0 from Slack Trigger (downloadFiles enabled). Attach AWS credentials on this node. |
| **Extract Textract Data** | `n8n-nodes-base.code` | Parses Textract output JSON | Textract Expense Analysis | AI Expense Categorization | |
| **AI Expense Categorization** | `n8n-nodes-base.openAi` | Classifies expense category | Extract Textract Data | Normalize Expense Data | |
| **Normalize Expense Data** | `n8n-nodes-base.code` | Merges OCR and AI metadata | AI Expense Categorization | Check Validation Confidence | |
| **Check Validation Confidence** | `n8n-nodes-base.if` | Validates extraction confidence | Normalize Expense Data | Read Manager from Sheets, Request Clearer Photo in Slack | |
| **Request Clearer Photo in Slack** | `n8n-nodes-base.slack` | Asks user for better image | Check Validation Confidence | None | |
| **Read Manager from Sheets** | `n8n-nodes-base.googleSheets` | Looks up manager record | Check Validation Confidence | Request Approval via Slack | |
| **Request Approval via Slack** | `n8n-nodes-base.slack` | Sends interactive approval | Read Manager from Sheets | If Approval Granted | Uses n8n's built-in Slack 'Send and Wait for Approval' operation - handles the button callback and pause/resume automatically. |
| **If Approval Granted** | `n8n-nodes-base.if` | Branches on manager approval | Request Approval via Slack | Load Approval Data, Announce Rejection in Slack | |
| **Load Approval Data** | `n8n-nodes-base.code` | Prepares Xero account routing | If Approval Granted | Route by Expense Type | |
| **Route by Expense Type** | `n8n-nodes-base.switch` | Routes by expense type | Load Approval Data | Submit Expense Claim to Xero, Submit Bill to Xero | |
| **Submit Expense Claim to Xero** | `n8n-nodes-base.httpRequest` | Creates Xero expense record | Route by Expense Type | Rebuild Receipt Binary | |
| **Submit Bill to Xero** | `n8n-nodes-base.httpRequest` | Creates Xero bill record | Route by Expense Type | Rebuild Receipt Binary | ACCPAY bill via HTTP so Contact.Name works (native node needs Contact UUID). Uses INTUZ tenant. |
| **Rebuild Receipt Binary** | `n8n-nodes-base.code` | Restores receipt binary data | Submit Expense Claim to Xero, Submit Bill to Xero | Link Receipt to Xero Invoice | |
| **Link Receipt to Xero Invoice** | `n8n-nodes-base.httpRequest` | Attaches receipt to Xero | Rebuild Receipt Binary | Append Approved to Sheets | |
| **Append Approved to Sheets** | `n8n-nodes-base.googleSheets` | Logs approved transaction | Link Receipt to Xero Invoice | None | |
| **Announce Rejection in Slack** | `n8n-nodes-base.slack` | Notifies user of rejection | If Approval Granted | Append Rejected to Sheets | |
| **Append Rejected to Sheets** | `n8n-nodes-base.googleSheets` | Logs rejected transaction | Announce Rejection in Slack | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Slack Trigger Node**
   - Node Type: `n8n-nodes-base.slackTrigger`
   - Name: `When Slack Message Received`
   - Configuration: Set trigger to `file_share`, enable `downloadFiles: true`, and watch the workspace.
   - Credentials: Connect Slack OAuth2 API.

2. **Create Filter Node**
   - Node Type: `n8n-nodes-base.filter`
   - Name: `Filter Receipt Images`
   - Configuration: Add condition where `{{ $json.files?.[0]?.url_private }}` is not empty, and `{{ $json.files?.[0]?.mimetype }}` starts with `image/`.
   - Connection: Connect `When Slack Message Received` output to this node.

3. **Create AWS Textract Node**
   - Node Type: `n8n-nodes-base.awsTextract`
   - Name: `Textract Expense Analysis`
   - Configuration: Set binary property name to `file_0`, set simple mode to `false`.
   - Credentials: Connect AWS IAM credentials with Textract permissions.
   - Connection: Connect `Filter Receipt Images` output to this node.

4. **Create Code Node (Extract Textract Data)**
   - Node Type: `n8n-nodes-base.code`
   - Name: `Extract Textract Data`
   - Configuration: Add JavaScript logic to parse `ExpenseDocuments[0].SummaryFields` for vendor, total, date, and currency.
   - Connection: Connect `Textract Expense Analysis` output to this node.

5. **Create OpenAI Node**
   - Node Type: `n8n-nodes-base.openAi`
   - Name: `AI Expense Categorization`
   - Configuration: Resource: `chat`, prompt: `Vendor: {{ $json.vendor }}. Amount: {{ $json.amount }}. Pick exactly one category from [Travel, Meals, Office Supplies, Software, Client Entertainment, Other]. Reply with only the category name.`, temperature: `0`.
   - Credentials: Connect OpenAI API credentials.
   - Connection: Connect `Extract Textract Data` output to this node.

6. **Create Code Node (Normalize Expense Data)**
   - Node Type: `n8n-nodes-base.code`
   - Name: `Normalize Expense Data`
   - Configuration: Add JavaScript logic to clean up AI output strings, parse float amounts, and calculate `confident` status flag.
   - Connection: Connect `AI Expense Categorization` output to this node.

7. **Create IF Node (Check Validation Confidence)**
   - Node Type: `n8n-nodes-base.if`
   - Name: `Check Validation Confidence`
   - Configuration: Condition: `{{$json.confident}}` equals `true`.
   - Connection: Connect `Normalize Expense Data` output to this node.

8. **Create Slack Node (Request Clearer Photo)**
   - Node Type: `n8n-nodes-base.slack`
   - Name: `Request Clearer Photo in Slack`
   - Configuration: Send text message to channel `C0C0UEMPD5E`.
   - Credentials: Connect Slack credentials.
   - Connection: Connect `Check Validation Confidence` **false** output to this node.

9. **Create Google Sheets Node (Read Manager)**
   - Node Type: `n8n-nodes-base.googleSheets`
   - Name: `Read Manager from Sheets`
   - Configuration: Operation: Read/Get, Document ID: `1yozEO983mOlmmCm0ujIVUsU46yop5D6HsN-VvL0dU2w`, Sheet Name: `ExpenseAudit`.
   - Credentials: Connect Google Sheets OAuth2 API.
   - Connection: Connect `Check Validation Confidence` **true** output to this node.

10. **Create Slack Node (Request Approval)**
    - Node Type: `n8n-nodes-base.slack`
    - Name: `Request Approval via Slack`
    - Configuration: Operation: `sendAndWait`, Approval Options type: `double`, channel: `C0C0UEMPD5E`.
    - Connection: Connect `Read Manager from Sheets` output to this node.

11. **Create IF Node (If Approval Granted)**
    - Node Type: `n8n-nodes-base.if`
    - Name: `If Approval Granted`
    - Configuration: Condition: `{{$json.data.approved}}` equals `true`.
    - Connection: Connect `Request Approval via Slack` output to this node.

12. **Create Code Node (Load Approval Data)**
    - Node Type: `n8n-nodes-base.code`
    - Name: `Load Approval Data`
    - Configuration: Add JavaScript to assign Xero account codes (e.g., Travel -> `493`, Meals -> `420`, etc.) and set `route` to `expense` or `bill`.
    - Connection: Connect `If Approval Granted` **true** output to this node.

13. **Create Switch Node (Route by Expense Type)**
    - Node Type: `n8n-nodes-base.switch`
    - Name: `Route by Expense Type`
    - Configuration: Mode: `expression`, output expression: `={{ $json.route === 'expense' ? 0 : 1 }}` (2 outputs).
    - Connection: Connect `Load Approval Data` output to this node.

14. **Create HTTP Request Node (Submit Expense Claim / Bill to Xero)**
    - Node Type: `n8n-nodes-base.httpRequest`
    - Names: `Submit Expense Claim to Xero` & `Submit Bill to Xero`
    - Configuration: Method: `PUT`, URL: `https://api.xero.com/api.xro/2.0/Invoices`, specify body as JSON with payload containing ACCPAY invoice details, Line Items, AccountCode, and Status `AUTHORISED`. Set headers: `Xero-Tenant-Id` (`2456a90a-71ac-4df1-a5fb-fb0234672663`) and `Accept: application/json`.
    - Credentials: Connect Xero OAuth2 API.
    - Connections: Connect Switch outputs 0 and 1 respectively.

15. **Create Code Node (Rebuild Receipt Binary)**
    - Node Type: `n8n-nodes-base.code`
    - Name: `Rebuild Receipt Binary`
    - Configuration: Re-inject binary payload from `When Slack Message Received` into output json/binary structure.
    - Connection: Connect both Xero submission HTTP request nodes to this node.

16. **Create HTTP Request Node (Link Receipt to Xero Invoice)**
    - Node Type: `n8n-nodes-base.httpRequest`
    - Name: `Link Receipt to Xero Invoice`
    - Configuration: Method: `POST`, URL: `https://api.xero.com/api.xro/2.0/Invoices/{{ $json.Invoices?.[0]?.InvoiceID }}/Attachments/...`, Content-Type: binaryData, Input data field name: `file_0`. Set `Xero-Tenant-Id` header.
    - Credentials: Connect Xero OAuth2 API.
    - Connection: Connect `Rebuild Receipt Binary` output to this node.

17. **Create Google Sheets Node (Append Approved)**
    - Node Type: `n8n-nodes-base.googleSheets`
    - Name: `Append Approved to Sheets`
    - Configuration: Operation: `append`, Document ID: `1yozEO983mOlmmCm0ujIVUsU46yop5D6HsN-VvL0dU2w`, Sheet Name: `ExpenseAudit`.
    - Credentials: Connect Google Sheets credentials.
    - Connection: Connect `Link Receipt to Xero Invoice` output to this node.

18. **Create Slack Node (Announce Rejection)**
    - Node Type: `n8n-nodes-base.slack`
    - Name: `Announce Rejection in Slack`
    - Configuration: Select: `user`, user: `={{$('Normalize Expense Data').item.json.slackUserId}}`.
    - Credentials: Connect Slack OAuth2.
    - Connection: Connect `If Approval Granted` **false** output to this node.

19. **Create Google Sheets Node (Append Rejected)**
    - Node Type: `n8n-nodes-base.googleSheets`
    - Name: `Append Rejected to Sheets`
    - Configuration: Operation: `append`, Document ID: `1yozEO983mOlmmCm0ujIVUsU46yop5D6HsN-VvL0dU2w`, Sheet Name: `ExpenseAudit`.
    - Credentials: Connect Google Sheets credentials.
    - Connection: Connect `Announce Rejection in Slack` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Website & Templates | [Intuz n8n Templates](https://www.intuz.com/n8n-workflow-automation-templates/) |
| Support Contact Email | getstarted@intuz.com |
| Company LinkedIn Profile | [Intuz LinkedIn](https://www.linkedin.com/company/intuz) |
| Partner Links & Sign Up | [n8n Partner Link](https://n8n.partnerlinks.io/intuz) |
| Custom Workflow Automation Requests | [Intuz Get Started](https://www.intuz.com/get-started/) |