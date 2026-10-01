Process, validate, and approve invoices with OpenAI GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/process--validate--and-approve-invoices-with-openai-gpt-4o-and-google-sheets-19739


# Process, validate, and approve invoices with OpenAI GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the intake, extraction, validation, duplicate checking, risk assessment, approval routing, and storage of invoices received as PDF files. Designed for finance and accounts payable teams, it minimizes manual data entry and review overhead while maintaining strict compliance thresholds.

The logical execution is grouped into the following functional blocks:
- **1.1 Intake & Extraction:** Receives invoice payloads via webhook, captures metadata, extracts text from the PDF, and utilizes an AI agent with structured outputs to parse invoice fields. Includes global error handling for operational failures.
- **1.2 Validation, Duplicate Check & Risk Analysis:** Cleans and normalizes extracted data, validates mandatory fields, verifies invoice uniqueness against Google Sheets, evaluates risk via a secondary AI agent, and checks total amounts.
- **1.3 Approval Routing & Storage:** Automatically approves low-risk/low-value invoices or routes high-risk/high-value ones for manual human review via email and a webhook resume mechanism. Stores finalized records in Google Sheets and dispatches confirmation notifications.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Intake & Extraction

#### Overview
This block serves as the entry point for incoming invoice submissions, capturing raw webhook data, extracting text from attached PDF binaries, and leveraging OpenAI to parse unstructured invoice text into a structured JSON schema. It also contains an independent error-handling branch for administrative notifications.

#### Nodes Involved
- `Webhook - Receive Invoice`
- `Set - Prepare Invoice Metadata`
- `Extract From File - Get Invoice Text`
- `AI - Extract Invoice Data`
- `AI - Extract Invoice Data - Chat Model`
- `AI - Extract Invoice Data - Output Parser`
- `Error Trigger - Workflow Failure`
- `Send Email - Notify Administrator`

#### Node Details

- **Webhook - Receive Invoice**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Trigger) — Listens for incoming HTTP POST requests containing vendor information and invoice PDF files.
  - **Configuration Choices:** Configured for method `POST` on path `invoice-intake`.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: External HTTP POST. Output: `Set - Prepare Invoice Metadata`.
  - **Version Requirements:** Type version 2.
  - **Edge Cases & Failure Types:** Payload parsing errors if the body is malformed; missing binary files if the client fails to upload the PDF correctly.

- **Set - Prepare Invoice Metadata**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation) — Assigns initial workflow metadata and normalizes submission properties.
  - **Configuration Choices:** Creates custom assignments including `vendorNameSubmitted`, `vendorEmailSubmitted`, `receivedAt` (current ISO timestamp), and `sourceRequestId` (falls back to execution ID). Retains other input fields.
  - **Key Expressions:** `={{$json.body.vendorName}}`, `={{$json.body.vendorEmail}}`, `={{$now.toISO()}}`, `={{$json.headers['x-request-id'] || $execution.id}}`.
  - **Input/Output Connections:** Input: `Webhook - Receive Invoice`. Output: `Extract From File - Get Invoice Text`.
  - **Version Requirements:** Type version 3.4.
  - **Edge Cases & Failure Types:** Missing body properties if upstream client does not send expected fields.

- **Extract From File - Get Invoice Text**
  - **Type & Technical Role:** `n8n-nodes-base.extractFromFile` (Utility) — Extracts plain text content from a binary PDF file.
  - **Configuration Choices:** Operation set to `pdf`, targeting binary property name `invoiceFile`.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: `Set - Prepare Invoice Metadata`. Output: `AI - Extract Invoice Data`.
  - **Version Requirements:** Type version 1.
  - **Edge Cases & Failure Types:** Fails if the binary property name is incorrect, if the file is password-protected, or if the PDF contains only non-searchable scanned images without OCR.

- **AI - Extract Invoice Data**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Processes raw invoice text combined with vendor metadata to extract explicit invoice values.
  - **Configuration Choices:** Prompt type defined manually with a system message forcing strict adherence to extracted text without inventing values. Enforces an output parser.
  - **Key Expressions:** `=Vendor submitted on the form: {{$('Set - Prepare Invoice Metadata').item.json.vendorNameSubmitted}} / {{$('Set - Prepare Invoice Metadata').item.json.vendorEmailSubmitted}}\n\nInvoice text:\n{{$json.text}}`.
  - **Input/Output Connections:** Input: `Extract From File - Get Invoice Text`. Output: `Code - Clean and Normalize Data`. Model and parser connected via AI sub-ports.
  - **Version Requirements:** Type version 1.7. Requires `@n8n/n8n-nodes-langchain`.
  - **Edge Cases & Failure Types:** API timeouts, rate limits, or hallucinations if the LLM encounters ambiguous document formats.

- **AI - Extract Invoice Data - Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Model Provider) — Provides the underlying LLM engine for extraction.
  - **Configuration Choices:** Model set to `gpt-4o-mini` with temperature set to `0` for deterministic outputs.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Connected to `AI - Extract Invoice Data` via the `ai_languageModel` port.
  - **Version Requirements:** Type version 1.

- **AI - Extract Invoice Data - Output Parser**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (Data Formatter) — Enforces a strict JSON schema on the LLM's response.
  - **Configuration Choices:** Manual schema defining required properties (`invoice_number`, `vendor_name`, `invoice_date`, `total_amount`) and optional line items and payment info.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Connected to `AI - Extract Invoice Data` via the `ai_outputParser` port.
  - **Version Requirements:** Type version 1.2.

- **Error Trigger - Workflow Failure**
  - **Type & Technical Role:** `n8n-nodes-base.errorTrigger` (Trigger) — Activates whenever any node in the workflow encounters an unhandled execution error.
  - **Configuration Choices:** Default setup.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: Global workflow error event. Output: `Send Email - Notify Administrator`.
  - **Version Requirements:** Type version 1.

- **Send Email - Notify Administrator**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Notification) — Emails the system administrator detailing workflow failures.
  - **Configuration Choices:** Text email format sent to a designated recipient address.
  - **Key Expressions:** `=The invoice processing workflow failed.\n\nWorkflow: {{$json.workflow.name}}\nExecution ID: {{$json.execution.id}}\nError: {{$json.execution.error?.message || 'Unknown error'}}\nNode: {{$json.execution.lastNodeExecuted}}`.
  - **Input/Output Connections:** Input: `Error Trigger - Workflow Failure`. Output: None.
  - **Version Requirements:** Type version 2.1. Requires valid Gmail OAuth2 credentials.

---

### Block 1.2: Validation, Duplicate Check & Risk Analysis

#### Overview
This block normalizes AI extraction outputs, validates mandatory fields, queries Google Sheets to prevent duplicate invoice processing, performs an AI-driven risk and categorization assessment, and verifies arithmetic consistency between subtotals, taxes, and totals.

#### Nodes Involved
- `Code - Clean and Normalize Data`
- `Code - Validate Required Fields`
- `IF - Is Invoice Valid`
- `Send Email - Request Missing Info`
- `Google Sheets - Search Existing Invoice`
- `IF - Is Duplicate Invoice`
- `Google Sheets - Update Status Duplicate`
- `Send Email - Notify Finance Duplicate`
- `AI - Analyze and Categorize Invoice`
- `AI - Analyze and Categorize Invoice - Chat Model`
- `AI - Analyze and Categorize Invoice - Output Parser`
- `Code - Verify Invoice Total`

#### Node Details

- **Code - Clean and Normalize Data**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Parses raw AI extraction payloads, strips currency symbols from numeric fields, and maps values to a consistent schema.
  - **Configuration Choices:** Custom JavaScript handling robust JSON parsing fallback and numeric cleaning helper functions.
  - **Key Expressions:** References `$('Set - Prepare Invoice Metadata').first().json`.
  - **Input/Output Connections:** Input: `AI - Extract Invoice Data`. Output: `Code - Validate Required Fields`.
  - **Version Requirements:** Type version 2.

- **Code - Validate Required Fields**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Validation) — Checks normalized invoice records for mandatory fields (`invoice_number`, `vendor_name`, `invoice_date`, `total_amount`).
  - **Configuration Choices:** JavaScript returning `is_valid` boolean flag and an array of `missing_fields`.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: `Code - Clean and Normalize Data`. Output: `IF - Is Invoice Valid`.
  - **Version Requirements:** Type version 2.

- **IF - Is Invoice Valid**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control) — Branches execution based on whether all mandatory fields are present.
  - **Configuration Choices:** Evaluates `{{$json.is_valid}}` equals `true`.
  - **Key Expressions:** `={{$json.is_valid}}`.
  - **Input/Output Connections:** Input: `Code - Validate Required Fields`. Output True: `Google Sheets - Search Existing Invoice`. Output False: `Send Email - Request Missing Info`.
  - **Version Requirements:** Type version 2.

- **Send Email - Request Missing Info**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Notification) — Emails the vendor requesting resubmission due to missing mandatory data.
  - **Configuration Choices:** Text email sent to vendor's email address.
  - **Key Expressions:** `={{$json.vendor_email}}`, `={{$json.vendor_name || 'there'}}`, `={{$json.missing_fields.join(', ')}}`, `={{$json.invoice_number || 'Unknown'}}`.
  - **Input/Output Connections:** Input: `IF - Is Invoice Valid` (False branch). Output: None.
  - **Version Requirements:** Type version 2.1. Requires Gmail credentials.

- **Google Sheets - Search Existing Invoice**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Database Lookup) — Queries the `Invoices` sheet to determine if an invoice with the same number already exists.
  - **Configuration Choices:** Operation set to lookup rows matching `invoice_number`. Document ID configured via placeholder `YOUR_GOOGLE_SHEET_ID`.
  - **Key Expressions:** `={{$json.invoice_number}}`.
  - **Input/Output Connections:** Input: `IF - Is Invoice Valid` (True branch). Output: `IF - Is Duplicate Invoice`.
  - **Version Requirements:** Type version 4.5. Requires Google Sheets OAuth2 credentials.

- **IF - Is Duplicate Invoice**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control) — Checks if any rows were returned by the Google Sheets lookup.
  - **Configuration Choices:** Evaluates if item count (`{{$items().length}}`) is greater than `0`.
  - **Key Expressions:** `={{$items().length}}`.
  - **Input/Output Connections:** Input: `Google Sheets - Search Existing Invoice`. Output True: `Google Sheets - Update Status Duplicate`. Output False: `AI - Analyze and Categorize Invoice`.
  - **Version Requirements:** Type version 2.

- **Google Sheets - Update Status Duplicate**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Database Update) — Updates the duplicate invoice record status to "Duplicate".
  - **Configuration Choices:** Operation set to update matching rows on `invoice_number`.
  - **Key Expressions:** `={{$('Code - Validate Required Fields').item.json.invoice_number}}`.
  - **Input/Output Connections:** Input: `IF - Is Duplicate Invoice` (True branch). Output: `Send Email - Notify Finance Duplicate`.
  - **Version Requirements:** Type version 4.5.

- **Send Email - Notify Finance Duplicate**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Notification) — Notifies the finance team that a duplicate invoice was intercepted.
  - **Configuration Choices:** Text email sent to finance recipient.
  - **Key Expressions:** References upstream validation node data for invoice details.
  - **Input/Output Connections:** Input: `Google Sheets - Update Status Duplicate`. Output: None.
  - **Version Requirements:** Type version 2.1.

- **AI - Analyze and Categorize Invoice**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Assesses invoice category, priority, risk level, and whether manual approval is required.
  - **Configuration Choices:** System prompt configured for an accounts-payable analyst persona with strict structured output parsing.
  - **Key Expressions:** Passes vendor name, email, total amount, serialized line items, and payment info.
  - **Input/Output Connections:** Input: `IF - Is Duplicate Invoice` (False branch). Output: `Code - Verify Invoice Total`. Model and parser connected via AI sub-ports.
  - **Version Requirements:** Type version 1.7.

- **AI - Analyze and Categorize Invoice - Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Model Provider) — OpenAI chat model provider (`gpt-4o-mini`, temperature `0`).
  - **Configuration Choices:** Standard LLM setup.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Connected via `ai_languageModel` to the analysis agent.
  - **Version Requirements:** Type version 1.

- **AI - Analyze and Categorize Invoice - Output Parser**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (Data Formatter) — Enforces schema requiring `category`, `priority` (low/medium/high), `risk_level` (low/medium/high), `approval_required` (boolean), and `reason`.
  - **Configuration Choices:** Manual JSON schema definition.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Connected via `ai_outputParser` to the analysis agent.
  - **Version Requirements:** Type version 1.2.

- **Code - Verify Invoice Total**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Calculation) — Verifies whether `subtotal + tax` equals the stated `total_amount` within a 1-cent tolerance.
  - **Configuration Choices:** JavaScript calculating expected totals and flagging discrepancies (`total_mismatch`).
  - **Key Expressions:** Merges data from validation and AI analysis nodes.
  - **Input/Output Connections:** Input: `AI - Analyze and Categorize Invoice`. Output: `Switch - Determine Approval Path`.
  - **Version Requirements:** Type version 2.

---

### Block 1.3: Approval Routing & Storage

#### Overview
This block routes invoices based on amount, risk level, discrepancy flags, and approval requirements. Low-risk invoices are automatically approved, while others trigger notification emails and enter a wait state until an external approval decision webhook is received. Finally, all records are stored in Google Sheets and confirmation is sent to finance.

#### Nodes Involved
- `Switch - Determine Approval Path`
- `Code - Auto Approve Invoice`
- `Send Email - Approval Request`
- `Wait - Wait for Approval Decision`
- `Code - Process Approval Decision`
- `Google Sheets - Store Invoice Record`
- `Send Email - Final Notification to Finance`

#### Node Details

- **Switch - Determine Approval Path**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (Flow Control) — Routes invoices into three distinct paths based on financial and risk criteria.
  - **Configuration Choices:** 
    - Rule 0 (Auto-Approve): Total amount $\le 1000$, risk level `low`, `approval_required` is `false`, and `total_mismatch` is `false`.
    - Rule 1 (Finance Review): Risk level `high` or `total_mismatch` is `true`.
    - Rule 2 (Manager Approval): Total amount $> 1000$.
  - **Key Expressions:** Evaluates JSON fields from total verification node.
  - **Input/Output Connections:** Input: `Code - Verify Invoice Total`. Output 0: `Code - Auto Approve Invoice`. Output 1 & 2: `Send Email - Approval Request`.
  - **Version Requirements:** Type version 3.2.

- **Code - Auto Approve Invoice**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Marks qualifying low-risk invoices as approved automatically.
  - **Configuration Choices:** Sets `approval_status` to `Approved` and records processing timestamp.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: `Switch - Determine Approval Path` (Auto-Approve branch). Output: `Google Sheets - Store Invoice Record`.
  - **Version Requirements:** Type version 2.

- **Send Email - Approval Request**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Notification) — Sends review and approval requests to finance or management inboxes.
  - **Configuration Choices:** Dynamically selects recipient (`finance-review@yourcompany.com` for high risk, `manager-approvals@yourcompany.com` otherwise).
  - **Key Expressions:** Includes evaluation details and a custom approval link containing request parameters.
  - **Input/Output Connections:** Input: `Switch - Determine Approval Path` (Review/Manager branches). Output: `Wait - Wait for Approval Decision`.
  - **Version Requirements:** Type version 2.1.

- **Wait - Wait for Approval Decision**
  - **Type & Technical Role:** `n8n-nodes-base.wait` (Flow Control) — Pauses workflow execution until an external webhook resume signal is received.
  - **Configuration Choices:** Resume mechanism set to `webhook`.
  - **Key Expressions:** None.
  - **Input/Output Connections:** Input: `Send Email - Approval Request`. Output: `Code - Process Approval Decision`.
  - **Version Requirements:** Type version 1.1.
  - **Edge Cases & Failure Types:** Execution timeouts if human reviewers fail to act within n8n retention limits.

- **Code - Process Approval Decision**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation) — Parses the incoming resume webhook payload containing the reviewer's decision.
  - **Configuration Choices:** Extracts `decision` (`approved` or `rejected`), `approver`, and comments.
  - **Key Expressions:** References resume webhook body and prior branch data.
  - **Input/Output Connections:** Input: `Wait - Wait for Approval Decision`. Output: `Google Sheets - Store Invoice Record`.
  - **Version Requirements:** Type version 2.

- **Google Sheets - Store Invoice Record**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Database Upsert) — Appends or updates the final invoice record in Google Sheets.
  - **Operation & Config:** Operation set to `appendOrUpdate`, matching on `invoice_number`. Maps all financial, category, status, and timestamp fields.
  - **Key Expressions:** Mapped to corresponding JSON properties (`approval_status`, `total_amount`, `category`, etc.).
  - **Input/Output Connections:** Input: `Code - Auto Approve Invoice` or `Code - Process Approval Decision`. Output: `Send Email - Final Notification to Finance`.
  - **Version Requirements:** Type version 4.5.

- **Send Email - Final Notification to Finance**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Notification) — Sends completion notices to finance regarding processed invoices.
  - **Configuration Choices:** Text email format sent to notification inbox.
  - **Key Expressions:** Summarizes vendor, invoice number, amount, category, status, and processing time.
  - **Input/Output Connections:** Input: `Google Sheets - Store Invoice Record`. Output: None.
  - **Version Requirements:** Type version 2.1.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Receive Invoice | n8n-nodes-base.webhook | Receive raw invoice payload | External HTTP | Set - Prepare Invoice Metadata | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| Set - Prepare Invoice Metadata | n8n-nodes-base.set | Assign initial metadata | Webhook - Receive Invoice | Extract From File - Get Invoice Text | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| Extract From File - Get Invoice Text | n8n-nodes-base.extractFromFile | Extract PDF plain text | Set - Prepare Invoice Metadata | AI - Extract Invoice Data | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| AI - Extract Invoice Data | @n8n/n8n-nodes-langchain.agent | AI invoice data extraction | Extract From File - Get Invoice Text | Code - Clean and Normalize Data | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| AI - Extract Invoice Data - Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for extraction | None (AI sub-port) | AI - Extract Invoice Data | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| AI - Extract Invoice Data - Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce extraction schema | None (AI sub-port) | AI - Extract Invoice Data | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| Code - Clean and Normalize Data | n8n-nodes-base.code | Normalize AI extraction output | AI - Extract Invoice Data | Code - Validate Required Fields | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| Code - Validate Required Fields | n8n-nodes-base.code | Validate mandatory fields | Code - Clean and Normalize Data | IF - Is Invoice Valid | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| IF - Is Invoice Valid | n8n-nodes-base.if | Branch on validity | Code - Validate Required Fields | Google Sheets - Search Existing Invoice, Send Email - Request Missing Info | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Send Email - Request Missing Info | n8n-nodes-base.gmail | Request missing vendor data | IF - Is Invoice Valid | None | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Google Sheets - Search Existing Invoice | n8n-nodes-base.googleSheets | Check duplicate invoices | IF - Is Invoice Valid | IF - Is Duplicate Invoice | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| IF - Is Duplicate Invoice | n8n-nodes-base.if | Branch on duplicate status | Google Sheets - Search Existing Invoice | Google Sheets - Update Status Duplicate, AI - Analyze and Categorize Invoice | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Google Sheets - Update Status Duplicate | n8n-nodes-base.googleSheets | Mark sheet row duplicate | IF - Is Duplicate Invoice | Send Email - Notify Finance Duplicate | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Send Email - Notify Finance Duplicate | n8n-nodes-base.gmail | Notify finance of duplicate | Google Sheets - Update Status Duplicate | None | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| AI - Analyze and Categorize Invoice | @n8n/n8n-nodes-langchain.agent | AI risk and category assessment | IF - Is Duplicate Invoice | Code - Verify Invoice Total | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| AI - Analyze and Categorize Invoice - Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for analysis | None (AI sub-port) | AI - Analyze and Categorize Invoice | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| AI - Analyze and Categorize Invoice - Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce analysis schema | None (AI sub-port) | AI - Analyze and Categorize Invoice | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Code - Verify Invoice Total | n8n-nodes-base.code | Verify total calculation | AI - Analyze and Categorize Invoice | Switch - Determine Approval Path | **2. Validate, Analyze & Route**<br>Checks required fields and duplicate invoices, uses AI to assess category/priority/risk, then routes to auto-approval, finance review, or manager approval based on amount and risk. |
| Switch - Determine Approval Path | n8n-nodes-base.switch | Route approval path | Code - Verify Invoice Total | Code - Auto Approve Invoice, Send Email - Approval Request | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Code - Auto Approve Invoice | n8n-nodes-base.code | Auto-approve qualified invoices | Switch - Determine Approval Path | Google Sheets - Store Invoice Record | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Send Email - Approval Request | n8n-nodes-base.gmail | Send approval email | Switch - Determine Approval Path | Wait - Wait for Approval Decision | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Wait - Wait for Approval Decision | n8n-nodes-base.wait | Wait for decision webhook | Send Email - Approval Request | Code - Process Approval Decision | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Code - Process Approval Decision | n8n-nodes-base.code | Parse approval webhook resume | Wait - Wait for Approval Decision | Google Sheets - Store Invoice Record | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Google Sheets - Store Invoice Record | n8n-nodes-base.googleSheets | Upsert final invoice record | Code - Auto Approve Invoice, Code - Process Approval Decision | Send Email - Final Notification to Finance | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Send Email - Final Notification to Finance | n8n-nodes-base.gmail | Notify finance of completion | Google Sheets - Store Invoice Record | None | **3. Approval & Storage**<br>Low-risk, low-value invoices are auto-approved. Everything else waits for a human decision via webhook. Either way, the final record is stored in Google Sheets and finance gets a confirmation email. |
| Error Trigger - Workflow Failure | n8n-nodes-base.errorTrigger | Catch global workflow errors | Global Error Event | Send Email - Notify Administrator | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |
| Send Email - Notify Administrator | n8n-nodes-base.gmail | Email admin on failure | Error Trigger - Workflow Failure | None | **1. Intake & Extract Data**<br>Receives the invoice via webhook, extracts the PDF text, and uses AI to pull structured invoice data. Includes global error handling — any workflow failure sends an alert email. |

---

### 4. Reproducing the Workflow from Scratch

Rebuild the workflow manually in your n8n instance by following these steps in order:

1. **Create Trigger Node:**
   - Add a **Webhook** node named `Webhook - Receive Invoice`. Set HTTP Method to `POST` and path to `invoice-intake`.
2. **Setup Metadata Transformation:**
   - Add a **Set** node named `Set - Prepare Invoice Metadata`. Assign `vendorNameSubmitted` (`={{$json.body.vendorName}}`), `vendorEmailSubmitted` (`={{$json.body.vendorEmail}}`), `receivedAt` (`={{$now.toISO()}}`), and `sourceRequestId` (`={{$json.headers['x-request-id'] || $execution.id}}`). Enable inclusion of other fields. Connect `Webhook - Receive Invoice` to this node.
3. **Configure PDF Text Extraction:**
   - Add an **Extract From File** node named `Extract From File - Get Invoice Text`. Set operation to `pdf` and binary property name to `invoiceFile`. Connect `Set - Prepare Invoice Metadata` to this node.
4. **Set Up Extraction AI Agent:**
   - Add an **AI Agent** node named `AI - Extract Invoice Data`. Set prompt type to define and enable output parser. 
   - Attach an **OpenAI Chat Model** node named `AI - Extract Invoice Data - Chat Model` (set to `gpt-4o-mini`, temperature `0`) to its language model input. Configure OpenAI API credentials.
   - Attach a **Structured Output Parser** node named `AI - Extract Invoice Data - Output Parser` to its output parser input, configured with the JSON schema specifying `invoice_number`, `vendor_name`, `invoice_date`, `total_amount`, line items, and payment info. Connect `Extract From File - Get Invoice Text` to this node.
5. **Normalize and Validate Data:**
   - Add a **Code** node named `Code - Clean and Normalize Data` to clean extraction outputs. Connect `AI - Extract Invoice Data` here.
   - Add a **Code** node named `Code - Validate Required Fields` to check mandatory fields (`invoice_number`, `vendor_name`, `invoice_date`, `total_amount`). Connect `Code - Clean and Normalize Data` here.
6. **Implement Validity Check:**
   - Add an **IF** node named `IF - Is Invoice Valid`. Set condition to check `{{$json.is_valid}} === true`. Connect `Code - Validate Required Fields` here.
   - For the `false` output branch, add a **Gmail** node named `Send Email - Request Missing Info` configured to email `{{$json.vendor_email}}` with missing fields. Connect Gmail OAuth2 credentials.
7. **Configure Duplicate Detection:**
   - For the `true` output branch of the validity IF node, add a **Google Sheets** node named `Google Sheets - Search Existing Invoice`. Set operation to lookup rows matching `invoice_number` in document ID `YOUR_GOOGLE_SHEET_ID` under sheet `Invoices`. Connect Google Sheets OAuth2 credentials.
   - Add an **IF** node named `IF - Is Duplicate Invoice` checking if `{{$items().length}} > 0`. Connect `Google Sheets - Search Existing Invoice` here.
   - For the `true` (duplicate) branch, add a **Google Sheets** node named `Google Sheets - Update Status Duplicate` to update the row status to `Duplicate`. Follow this with a **Gmail** node named `Send Email - Notify Finance Duplicate` targeting the finance team.
8. **Configure Analysis AI Agent:**
   - For the `false` (non-duplicate) branch of the duplicate IF node, add an **AI Agent** node named `AI - Analyze and Categorize Invoice`.
   - Attach an **OpenAI Chat Model** node named `AI - Analyze and Categorize Invoice - Chat Model` (`gpt-4o-mini`, temperature `0`).
   - Attach a **Structured Output Parser** node named `AI - Analyze and Categorize Invoice - Output Parser` with schema requiring `category`, `priority`, `risk_level`, `approval_required`, and `reason`.
9. **Verify Invoice Totals:**
   - Add a **Code** node named `Code - Verify Invoice Total` to calculate expected totals (`subtotal + tax`) and flag discrepancies. Connect `AI - Analyze and Categorize Invoice` here.
10. **Configure Approval Routing Switch:**
    - Add a **Switch** node named `Switch - Determine Approval Path`. Connect `Code - Verify Invoice Total` here. Configure three rules:
      - Rule 0 (Auto-Approve): `total_amount` $\le 1000$, `risk_level` equals `low`, `approval_required` is `false`, `total_mismatch` is `false`.
      - Rule 1 (Finance Review): `risk_level` equals `high` OR `total_mismatch` is `true`.
      - Rule 2 (Manager Approval): `total_amount` $> 1000$.
11. **Configure Auto-Approval Branch:**
    - Connect Output 0 of the switch to a **Code** node named `Code - Auto Approve Invoice` setting status to `Approved`. Connect this to a **Google Sheets** node named `Google Sheets - Store Invoice Record` (operation `appendOrUpdate`, matching `invoice_number`).
12. **Configure Manual Approval Branch:**
    - Connect Outputs 1 and 2 of the switch to a **Gmail** node named `Send Email - Approval Request` targeting reviewers based on risk level.
    - Connect this to a **Wait** node named `Wait - Wait for Approval Decision` set to resume via `webhook`.
    - Connect the wait node to a **Code** node named `Code - Process Approval Decision` to parse resume payloads.
    - Connect the process decision node to the same **Google Sheets - Store Invoice Record** node.
13. **Configure Completion Notification:**
    - Connect `Google Sheets - Store Invoice Record` to a **Gmail** node named `Send Email - Final Notification to Finance` to email processing confirmation to finance.
14. **Configure Global Error Handling:**
    - Add an **Error Trigger** node named `Error Trigger - Workflow Failure`.
    - Connect it to a **Gmail** node named `Send Email - Notify Administrator` to alert admins on workflow failures.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Invoice Processing and Automation Overview | Designed to handle automated invoice ingestion, AI extraction, duplicate checking, validation, risk categorization, and multi-tier approval routing. |
| External Integration Requirements | Requires configured active credentials for OpenAI (LLM extraction and analysis), Gmail (OAuth2 for vendor, approver, and finance notifications), and Google Sheets (for duplicate lookup and record storage). |
| Approval Webhook Implementation Note | The Wait node requires an external HTTP endpoint to POST back resume payloads containing the `decision` field (`approved` or `rejected`) to the workflow's wait URL. |