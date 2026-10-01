Extract and validate invoice PDFs into Excel with OpenAI GPT-4o-mini

https://n8nworkflows.xyz/workflows/extract-and-validate-invoice-pdfs-into-excel-with-openai-gpt-4o-mini-19810


# Extract and validate invoice PDFs into Excel with OpenAI GPT-4o-mini

### 1. Workflow Overview

This workflow automates the daily ingestion, AI-powered data extraction, validation, and archiving of invoice PDFs from a local filesystem. It evaluates environment constraints, structures invoice headers and line items using OpenAI (`gpt-4o-mini`), validates business rules (such as required fields and subtotal math), generates individual Excel outputs per invoice, archives source files into corresponding subdirectories, and compiles a comprehensive run execution report.

The system logic is organized into the following functional blocks:
- **1.1 Schedule & Environment Initialization:** Triggers daily execution, detects local filesystem access availability, and validates the required folder structure.
- **1.2 Configuration & Initial Reporting:** Sets up drive paths, prepares the initial execution tracking metrics, and generates the base execution report file.
- **1.3 File Ingestion & Batch Processing:** Reads PDFs from the local disk input directory and iterates through them one by one using a batch loop.
- **1.4 AI Data Extraction & Error Handling:** Extracts text from PDFs, routes unreadable files to error storage, cleans valid text, and processes it through OpenAI with a strict JSON schema.
- **1.5 Normalization & Validation:** Parses the AI response, runs business validation checks on required fields and line-item totals, and formats data into invoice header and line-item rows.
- **1.6 Output Generation & Archiving:** Converts validated data to Excel spreadsheets, writes them to the output folder, and archives source PDFs into `valid`, `invalid`, or `error` folders while removing them from the input queue.
- **1.7 Final Execution Wrap-Up:** Updates final counters, appends completion timestamps, and overwrites the execution report with final results.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Environment Initialization
- **Overview:** Initiates the workflow on a daily schedule, tests whether the runtime environment has local filesystem capabilities, and verifies that the base invoices directory exists.
- **Nodes Involved:** 
  - `When Scheduled at 4am`
  - `Identify Deployment Mode`
  - `Check Local Deployment Status`
  - `Verify Invoices Directory`
  - `Confirm Invoices Directory Exists`
- **Node Details:**
  - **When Scheduled at 4am**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Starts the workflow execution daily at 4:00 AM.
    - *Configuration:* Trigger rule configured for hour 4.
    - *Connections:* Input: None; Output: `Identify Deployment Mode`.
    - *Edge Cases / Failure Types:* Missed runs if the n8n instance is offline.
  - **Identify Deployment Mode**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Determines if n8n is running in a local self-hosted instance with `fs` access or in n8n Cloud.
    - *Configuration:* Evaluates `require('fs')` access.
    - *Expressions:* Returns `{ deployment_mode: 'local' | 'cloud' }`.
    - *Connections:* Input: `When Scheduled at 4am`; Output: `Check Local Deployment Status`.
  - **Check Local Deployment Status**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional Router) — Verifies if the deployment mode is set to local.
    - *Configuration:* Checks `{{$json.deployment_mode}}` equals `local`.
    - *Connections:* Input: `Identify Deployment Mode`; Output (True): `Verify Invoices Directory`.
  - **Verify Invoices Directory**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Checks if the base directory `/invoices` exists on the filesystem.
    - *Configuration:* Uses `fs.existsSync('/invoices')`.
    - *Connections:* Input: `Check Local Deployment Status`; Output: `Confirm Invoices Directory Exists`.
  - **Confirm Invoices Directory Exists**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional Router) — Confirms directory existence before configuration.
    - *Configuration:* Checks `{{$json.exists}}` equals `true`.
    - *Connections:* Input: `Verify Invoices Directory`; Output (True): `Configure Local Drive Settings`.

---

#### Block 1.2: Configuration & Initial Reporting
- **Overview:** Assembles drive folder paths, sets up initial metadata for tracking batch execution counters, and writes the initial execution report to disk.
- **Nodes Involved:**
  - `Configure Local Drive Settings`
  - `Compile Final Configuration`
  - `Set Initial Configurations`
  - `Retrieve Execution Info`
  - `Initialize Exec Report Content`
  - `Route by Input Source`
  - `Transform Data to File Format`
  - `Generate Exec Report File`
  - `Handle Missing Input Source`
- **Node Details:**
  - **Configure Local Drive Settings**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Sets the input source scope to `local_drive`.
    - *Connections:* Input: `Confirm Invoices Directory Exists`; Output: `Compile Final Configuration`.
  - **Compile Final Configuration**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Builds object paths for input, output, valid, invalid, and error subfolders.
    - *Connections:* Input: `Configure Local Drive Settings`; Output: `Set Initial Configurations`.
  - **Set Initial Configurations**
    - *Type & Role:* `n8n-nodes-base.set` (Set/Edit Fields) — Passes configuration states downstream.
    - *Connections:* Input: `Compile Final Configuration`; Output: `Retrieve Execution Info`.
  - **Retrieve Execution Info**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Generates timestamp stamps and initializes execution counters in workflow static data (`exec_info`).
    - *Connections:* Input: `Set Initial Configurations`; Output: `Initialize Exec Report Content`.
  - **Initialize Exec Report Content**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Reads static execution metrics into item scope.
    - *Connections:* Input: `Retrieve Execution Info`; Output: `Route by Input Source`.
  - **Route by Input Source**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch Router) — Routes execution based on whether `input_source` equals `local_drive`.
    - *Connections:* Input: `Initialize Exec Report Content`; Output 0: `Transform Data to File Format`, Output 1: `Handle Missing Input Source`.
  - **Transform Data to File Format**
    - *Type & Role:* `n8n-nodes-base.convertToFile` (Convert to File) — Formats execution tracking data into an Excel sheet structure (`Execution Report`).
    - *Connections:* Input: `Route by Input Source`; Output: `Generate Exec Report File`.
  - **Generate Exec Report File**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Writes the initial empty execution report Excel file to the output folder.
    - *Connections:* Input: `Transform Data to File Format`; Output: `Load PDFs from Disk`.
  - **Handle Missing Input Source**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Throws an explicit error if the input source configuration is missing or invalid.
    - *Connections:* Input: `Route by Input Source`; Output: None (Terminal error).

---

#### Block 1.3: File Ingestion & Batch Processing
- **Overview:** Scans the local input folder for PDF invoices, checks if any files are available, and initiates a batch loop to process files sequentially.
- **Nodes Involved:**
  - `Load PDFs from Disk`
  - `Check Local Files Availability`
  - `Loop Over File Batches`
- **Node Details:**
  - **Load PDFs from Disk**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Reads all `.pdf` files matching the input directory selector pattern.
    - *Configuration:* File selector uses `{{ $('Set Initial Configurations').first().json.config.local_drive.invoice_input_folder }}/*.pdf`.
    - *Connections:* Input: `Generate Exec Report File`; Output: `Check Local Files Availability`.
  - **Check Local Files Availability**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional Router) — Determines if the file list is empty.
    - *Configuration:* Checks if `Object.keys($json).length === 0`.
    - *Connections:* Input: `Load PDFs from Disk`; Output (True): `Setup Exec Report Initial Content`, Output (False): `Loop Over File Batches`.
  - **Loop Over File Batches**
    - *Type & Role:* `n8n-nodes-base.splitInBatches` (Split In Batches) — Iterates through the file list item by item.
    - *Connections:* Input: `Check Local Files Availability`; Output 1 (Looping): `Increment Total File Counter`, Output 2 (Done): `Setup Exec Report Initial Content`.

---

#### Block 1.4: AI Data Extraction & Error Handling
- **Overview:** Increments batch counters, extracts text from the PDF, catches unreadable file errors, cleans extracted text, and sends it to OpenAI for structured JSON extraction.
- **Nodes Involved:**
  - `Increment Total File Counter`
  - `Extract Data from PDF`
  - `Clean Extracted Text`
  - `OpenAI Invoice Processing`
  - `Increment Error File Counter`
  - `Log Error Files to Disk`
  - `Remove Error Files from Input`
- **Node Details:**
  - **Increment Total File Counter**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Increments `total_files` inside global static data.
    - *Connections:* Input: `Loop Over File Batches`; Output: `Extract Data from PDF`.
  - **Extract Data from PDF**
    - *Type & Role:* `n8n-nodes-base.extractFromFile` (Extract from File) — Extracts text content from PDF binary data.
    - *Configuration:* Operation: `pdf`. Error Handling: `Continue Using Error Output` (`onError: continueErrorOutput`).
    - *Connections:* Input: `Increment Total File Counter`; Output 1 (Success): `Clean Extracted Text`, Output 2 (Error): `Increment Error File Counter`.
  - **Clean Extracted Text**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Cleans whitespace, line breaks, and determines character length / scan flags.
    - *Connections:* Input: `Extract Data from PDF`; Output: `OpenAI Invoice Processing`.
  - **OpenAI Invoice Processing**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (OpenAI) — Queries the `gpt-4o-mini` model with structured JSON schema constraints to extract invoice metadata and line items.
    - *Configuration:* Model: `gpt-4o-mini`. Response format enforced via JSON Schema (`invoice_output_schema`). System prompt strictly prohibits value hallucination and mandates accurate currency and subtotal detection.
    - *Credentials:* OpenAI account credential (`nx6fC2VKPhBpLz0a`).
    - *Connections:* Input: `Clean Extracted Text`; Output: `Normalize AI Output`.
    - *Edge Cases / Failure Types:* Rate limits, API timeouts, invalid API keys, or JSON schema violations.
  - **Increment Error File Counter**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Increments `error_files` counter on PDF read/extraction failure.
    - *Connections:* Input: `Extract Data from PDF` (Error branch); Output: `Log Error Files to Disk`.
  - **Log Error Files to Disk**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Writes the failed source PDF into the local error folder.
    - *Connections:* Input: `Increment Error File Counter`; Output: `Remove Error Files from Input`.
  - **Remove Error Files from Input**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Deletes the error-flagged PDF from the input directory using `fs.unlinkSync`.
    - *Connections:* Input: `Log Error Files to Disk`; Output: `Loop Over File Batches` (Loops back for next file).

---

#### Block 1.5: Normalization & Validation
- **Overview:** Normalizes raw AI output structures, validates required invoice fields, cross-checks subtotal and line-item totals, and formats results into tabular row schemas.
- **Nodes Involved:**
  - `Normalize AI Output`
  - `Validate Extracted Invoice`
  - `Split JSON Output`
- **Node Details:**
  - **Normalize AI Output**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Recursively searches the AI response object for canonical invoice structures.
    - *Connections:* Input: `OpenAI Invoice Processing`; Output: `Validate Extracted Invoice`.
  - **Validate Extracted Invoice**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Validates required fields (`vendor_name`, `invoice_number`, `invoice_date`, `total`) and verifies that line item totals match stated invoice subtotals within tolerance ($0.01). Assigns `validation_status` (`VALID` or `INVALID`).
    - *Connections:* Input: `Normalize AI Output`; Output: `Split JSON Output`.
  - **Split JSON Output**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Flattens the invoice header and line items into separate uniform items for spreadsheet export.
    - *Connections:* Input: `Validate Extracted Invoice`; Output: `Convert JSON to File`.

---

#### Block 1.6: Output Generation & Archiving
- **Overview:** Converts structured invoice data into Excel files, saves them to disk, routes source PDFs to valid/invalid folders, deletes originals from input, and merges processing streams.
- **Nodes Involved:**
  - `Convert JSON to File`
  - `Manage Files on Disk`
  - `Verify Invoice Validity`
  - `Count Valid Files`
  - `Store Valid Files Locally`
  - `Remove Valid Files from Input`
  - `Increment Invalid File Counter`
  - `Archive Invalid Files Locally`
  - `Clear Invalid Files from Input`
  - `Combine File Streams`
- **Node Details:**
  - **Convert JSON to File**
    - *Type & Role:* `n8n-nodes-base.convertToFile` (Convert to File) — Converts JSON objects into Excel (`.xlsx`) binary format.
    - *Connections:* Input: `Split JSON Output`; Output: `Manage Files on Disk`.
  - **Manage Files on Disk**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Writes the invoice extraction Excel output to the local output folder.
    - *Connections:* Input: `Convert JSON to File`; Output: `Verify Invoice Validity`.
  - **Verify Invoice Validity**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional Router) — Branches based on whether the validation status is `VALID`.
    - *Configuration:* Checks `{{ $('Validate Extracted Invoice').item.json.validation_status }}` equals `VALID`.
    - *Connections:* Input: `Manage Files on Disk`; Output (True): `Count Valid Files`, Output (False): `Increment Invalid File Counter`.
  - **Count Valid Files**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Increments `valid_files` in workflow static data.
    - *Connections:* Input: `Verify Invoice Validity` (True); Output: `Store Valid Files Locally`.
  - **Store Valid Files Locally**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Writes the valid source PDF to the local valid folder.
    - *Connections:* Input: `Count Valid Files`; Output: `Remove Valid Files from Input`.
  - **Remove Valid Files from Input**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Deletes the processed valid PDF from the input directory using `fs.unlinkSync`.
    - *Connections:* Input: `Store Valid Files Locally`; Output: `Combine File Streams`.
  - **Increment Invalid File Counter**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Increments `invalid_files` in workflow static data.
    - *Connections:* Input: `Verify Invoice Validity` (False); Output: `Archive Invalid Files Locally`.
  - **Archive Invalid Files Locally**
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Writes the invalid source PDF to the local invalid folder.
    - *Connections:* Input: `Increment Invalid File Counter`; Output: `Clear Invalid Files from Input`.
  - **Clear Invalid Files from Input**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Deletes the processed invalid PDF from the input directory using `fs.unlinkSync`.
    - *Connections:* Input: `Archive Invalid Files Locally`; Output: `Combine File Streams`.
  - **Combine File Streams**
    - *Type & Role:* `n8n-nodes-base.merge` (Merge) — Merges the valid and invalid file processing branches back into a single stream.
    - *Connections:* Input 1: `Remove Valid Files from Input`, Input 2: `Clear Invalid Files from Input`; Output: `Loop Over File Batches` (Loops back for next file).

---

#### Block 1.7: Final Execution Wrap-Up
- **Overview:** Compiles final run counts and timestamps after all files are processed, and overwrites the execution report with end-of-run summaries.
- **Nodes Involved:**
  - `Setup Exec Report Initial Content`
  - `Reformat to File Format`
  - `Save Exec Report to File`
  - `Setup Exec Report Initial Content` (Alternate empty run branch)
  - `Reformat to File Format` (Alternate empty run branch)
  - `Save Exec Report to File` (Alternate empty run branch)
- **Node Details:**
  - **Setup Exec Report Initial Content** (Both branches)
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript Code) — Captures execution end time and gathers final counts from static storage (`exec_info`).
    - *Connections:* Input: `Loop Over File Batches` (Done) or `Check Local Files Availability` (True); Output: `Reformat to File Format`.
  - **Reformat to File Format** (Both branches)
    - *Type & Role:* `n8n-nodes-base.convertToFile` (Convert to File) — Formats final execution statistics into Excel spreadsheet structure.
    - *Connections:* Input: `Setup Exec Report Initial Content`; Output: `Save Exec Report to File`.
  - **Save Exec Report to File** (Both branches)
    - *Type & Role:* `n8n-nodes-base.readWriteFile` (Read/Write Files from Disk) — Overwrites the execution report Excel file on disk with finalized counters and timestamps.
    - *Connections:* Input: `Reformat to File Format`; Output: None (Terminal nodes).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note covering full workflow overview, setup, and customization. | None | None | InvoiceExtractor-Local |
| Sticky Note1 | n8n-nodes-base.stickyNote | Note describing scheduled execution trigger branch. | None | None | Scheduled invoice run |
| Sticky Note2 | n8n-nodes-base.stickyNote | Note describing local deployment environment check. | None | None | Detect local deployment |
| Sticky Note3 | n8n-nodes-base.stickyNote | Note describing drive configuration assembly. | None | None | Build final configuration |
| Sticky Note4 | n8n-nodes-base.stickyNote | Note describing execution report initialization. | None | None | Prepare execution report |
| Sticky Note5 | n8n-nodes-base.stickyNote | Note describing input source routing branch. | None | None | Route input source |
| Sticky Note6 | n8n-nodes-base.stickyNote | Note describing initial report generation and local PDF loading. | None | None | Load local PDFs |
| Sticky Note7 | n8n-nodes-base.stickyNote | Note describing file batch loop management. | None | None | Manage file batches |
| Sticky Note8 | n8n-nodes-base.stickyNote | Note describing alternate report generation for empty runs. | None | None | Write empty-run report |
| Sticky Note9 | n8n-nodes-base.stickyNote | Note describing file counter increment and PDF text extraction. | None | None | Count and extract PDF |
| Sticky Note10 | n8n-nodes-base.stickyNote | Note describing PDF extraction error handling and routing. | None | None | Handle PDF extraction errors |
| Sticky Note11 | n8n-nodes-base.stickyNote | Note describing text cleaning and OpenAI processing. | None | None | Clean and classify text |
| Sticky Note12 | n8n-nodes-base.stickyNote | Note describing AI output normalization and validation. | None | None | Normalize validated invoice |
| Sticky Note13 | n8n-nodes-base.stickyNote | Note describing file conversion and validity checking. | None | None | Persist extraction result |
| Sticky Note14 | n8n-nodes-base.stickyNote | Note describing valid invoice archiving and counter update. | None | None | Process valid invoices |
| Sticky Note15 | n8n-nodes-base.stickyNote | Note describing invalid invoice archiving and counter update. | None | None | Process invalid invoices |
| Sticky Note16 | n8n-nodes-base.stickyNote | Note describing loop branch rejoining. | None | None | Rejoin processing loop |
| Clean Extracted Text | n8n-nodes-base.code | Cleans raw extracted text and builds metadata properties. | Extract Data from PDF | OpenAI Invoice Processing | Clean and classify text |
| OpenAI Invoice Processing | @n8n/n8n-nodes-langchain.openAi | Extracts structured invoice fields and line items using GPT-4o-mini with schema enforcement. | Clean Extracted Text | Normalize AI Output | Clean and classify text |
| Normalize AI Output | n8n-nodes-base.code | Recursively normalizes AI response into canonical invoice JSON. | OpenAI Invoice Processing | Validate Extracted Invoice | Normalize validated invoice |
| Split JSON Output | n8n-nodes-base.code | Flattens invoice headers and line items into rows for export. | Validate Extracted Invoice | Convert JSON to File | Normalize validated invoice |
| Extract Data from PDF | n8n-nodes-base.extractFromFile | Extracts text content from PDF binary input with error fallback. | Increment Total File Counter | Clean Extracted Text, Increment Error File Counter | Count and extract PDF |
| When Scheduled at 4am | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 4:00 AM. | None | Identify Deployment Mode | Scheduled invoice run |
| Initialize Exec Report Content | n8n-nodes-base.code | Retrieves initial execution info from static storage. | Retrieve Execution Info | Route by Input Source | Prepare execution report |
| Generate Exec Report File | n8n-nodes-base.readWriteFile | Writes initial execution report Excel file to disk. | Transform Data to File Format | Load PDFs from Disk | Load local PDFs |
| Load PDFs from Disk | n8n-nodes-base.readWriteFile | Loads all PDF files from the configured local input directory. | Generate Exec Report File | Check Local Files Availability | Load local PDFs |
| Loop Over File Batches | n8n-nodes-base.splitInBatches | Iterates through loaded PDF files sequentially. | Check Local Files Availability, Combine File Streams | Setup Exec Report Initial Content, Increment Total File Counter | Manage file batches |
| Convert JSON to File | n8n-nodes-base.convertToFile | Converts structured JSON records to Excel file format. | Split JSON Output | Manage Files on Disk | Persist extraction result |
| Manage Files on Disk | n8n-nodes-base.readWriteFile | Writes individual invoice Excel outputs to disk. | Convert JSON to File | Verify Invoice Validity | Persist extraction result |
| Set Initial Configurations | n8n-nodes-base.set | Passes configuration object downstream. | Compile Final Configuration | Retrieve Execution Info | Prepare execution report |
| Transform Data to File Format | n8n-nodes-base.convertToFile | Formats execution report data into Excel table structure. | Route by Input Source | Generate Exec Report File | Prepare execution report |
| Setup Exec Report Initial Content | n8n-nodes-base.code | Prepares final execution report content with timestamps and metrics. | Check Local Files Availability, Loop Over File Batches | Reformat to File Format | Write empty-run report, Manage file batches |
| Reformat to File Format | n8n-nodes-base.convertToFile | Converts final execution metrics into Excel sheet format. | Setup Exec Report Initial Content | Save Exec Report to File | Write empty-run report, Manage file batches |
| Save Exec Report to File | n8n-nodes-base.readWriteFile | Overwrites execution report Excel file on disk with final data. | Reformat to File Format | None | Write empty-run report, Manage file batches |
| Retrieve Execution Info | n8n-nodes-base.code | Initializes workflow static execution metadata and timestamps. | Set Initial Configurations | Initialize Exec Report Content | Prepare execution report |
| Increment Total File Counter | n8n-nodes-base.code | Increments total processed file counter in static storage. | Loop Over File Batches | Extract Data from PDF | Count and extract PDF |
| Increment Error File Counter | n8n-nodes-base.code | Increments error file counter in static storage upon failure. | Extract Data from PDF | Log Error Files to Disk | Handle PDF extraction errors |
| Log Error Files to Disk | n8n-nodes-base.readWriteFile | Writes failed source PDF to the local error directory. | Increment Error File Counter | Remove Error Files from Input | Handle PDF extraction errors |
| Validate Extracted Invoice | n8n-nodes-base.code | Validates required fields and line item math consistency. | Normalize AI Output | Split JSON Output | Normalize validated invoice |
| Check Local Files Availability | n8n-nodes-base.if | Checks whether the file input list is empty. | Load PDFs from Disk | Setup Exec Report Initial Content, Loop Over File Batches | Manage file batches |
| Verify Invoice Validity | n8n-nodes-base.if | Branches processing based on invoice validation status (VALID/INVALID). | Manage Files on Disk | Count Valid Files, Increment Invalid File Counter | Persist extraction result |
| Count Valid Files | n8n-nodes-base.code | Increments valid file counter in static storage. | Verify Invoice Validity | Store Valid Files Locally | Process valid invoices |
| Store Valid Files Locally | n8n-nodes-base.readWriteFile | Writes valid source PDF to local valid folder. | Count Valid Files | Remove Valid Files from Input | Process valid invoices |
| Remove Valid Files from Input | n8n-nodes-base.code | Deletes processed valid PDF from input directory. | Store Valid Files Locally | Combine File Streams | Process valid invoices |
| Combine File Streams | n8n-nodes-base.merge | Merges valid and invalid file processing streams. | Remove Valid Files from Input, Clear Invalid Files from Input | Loop Over File Batches | Rejoin processing loop |
| Increment Invalid File Counter | n8n-nodes-base.code | Increments invalid file counter in static storage. | Verify Invoice Validity | Archive Invalid Files Locally | Process invalid invoices |
| Clear Invalid Files from Input | n8n-nodes-base.code | Deletes processed invalid PDF from input directory. | Archive Invalid Files Locally | Combine File Streams | Process invalid invoices |
| Archive Invalid Files Locally | n8n-nodes-base.readWriteFile | Writes invalid source PDF to local invalid folder. | Increment Invalid File Counter | Clear Invalid Files from Input | Process invalid invoices |
| Route by Input Source | n8n-nodes-base.switch | Routes execution based on input source configuration. | Initialize Exec Report Content | Transform Data to File Format, Handle Missing Input Source | Route input source |
| Remove Error Files from Input | n8n-nodes-base.code | Deletes error PDF from input directory. | Log Error Files to Disk | Loop Over File Batches | Handle PDF extraction errors |
| Configure Local Drive Settings | n8n-nodes-base.code | Sets input source to local drive configuration. | Confirm Invoices Directory Exists | Compile Final Configuration | Build final configuration |
| Check Local Deployment Status | n8n-nodes-base.if | Checks if deployment mode equals local. | Identify Deployment Mode | Verify Invoices Directory | Detect local deployment |
| Verify Invoices Directory | n8n-nodes-base.code | Checks if base `/invoices` directory exists on disk. | Check Local Deployment Status | Confirm Invoices Directory Exists | Detect local deployment |
| Confirm Invoices Directory Exists | n8n-nodes-base.if | Confirms `/invoices` directory exists before configuration. | Verify Invoices Directory | Configure Local Drive Settings | Detect local deployment |
| Identify Deployment Mode | n8n-nodes-base.code | Detects whether n8n runs locally or in cloud. | When Scheduled at 4am | Check Local Deployment Status | Scheduled invoice run |
| Compile Final Configuration | n8n-nodes-base.code | Assembles local drive folder paths configuration object. | Configure Local Drive Settings | Set Initial Configurations | Build final configuration |
| Handle Missing Input Source | n8n-nodes-base.code | Throws error if input source is unconfigured. | Route by Input Source | None | Route input source |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Environment Setup:** Ensure your self-hosted n8n instance has local filesystem access enabled (e.g., set environment variable `NODE_FUNCTION_ALLOW_BUILTIN=fs,path` and restrict access with `N8N_RESTRICT_FILE_ACCESS_TO=/invoices`). Create a local directory `/invoices` with subfolders: `input`, `output`, `valid`, `invalid`, and `error`.
2. **Trigger Creation:**
   - Create a **Schedule Trigger** node named `When Scheduled at 4am`. Set the trigger rule to run at hour `4`.
3. **Environment Detection & Configuration Nodes:**
   - Create a **Code** node named `Identify Deployment Mode` to check `fs` module availability. Connect `When Scheduled at 4am` to it.
   - Create an **If** node named `Check Local Deployment Status` (`{{$json.deployment_mode}}` equals `local`). Connect `Identify Deployment Mode` to it.
   - Create a **Code** node named `Verify Invoices Directory` to check if `/invoices` exists via `fs.existsSync`. Connect `Check Local Deployment Status` (True branch) to it.
   - Create an **If** node named `Confirm Invoices Directory Exists` (`{{$json.exists}}` equals `true`). Connect `Verify Invoices Directory` to it.
   - Create a **Code** node named `Configure Local Drive Settings` returning `{ input_source: 'local_drive' }`. Connect `Confirm Invoices Directory Exists` (True branch) to it.
   - Create a **Code** node named `Compile Final Configuration` to build paths (`invoice_input_folder`, `invoice_valid_folder`, etc.). Connect `Configure Local Drive Settings` to it.
   - Create a **Set** node named `Set Initial Configurations` (include other fields). Connect `Compile Final Configuration` to it.
4. **Initial Execution Reporting:**
   - Create a **Code** node named `Retrieve Execution Info` to initialize timestamps and static `exec_info` counters. Connect `Set Initial Configurations` to it.
   - Create a **Code** node named `Initialize Exec Report Content`. Connect `Retrieve Execution Info` to it.
   - Create a **Switch** node named `Route by Input Source` checking if `input_source` equals `local_drive`. Connect `Initialize Exec Report Content` to it.
   - Create a **Code** node named `Handle Missing Input Source` to throw an error on unsupported input sources. Connect switch route 1 to it.
   - Create a **Convert to File** node named `Transform Data to File Format` (Excel format, sheet name `Execution Report`). Connect switch route 0 to it.
   - Create a **Read/Write Files from Disk** node named `Generate Exec Report File` to write the initial report Excel. Connect `Transform Data to File Format` to it.
5. **PDF Ingestion & Batching:**
   - Create a **Read/Write Files from Disk** node named `Load PDFs from Disk` reading `{{ $('Set Initial Configurations').first().json.config.local_drive.invoice_input_folder }}/*.pdf`. Connect `Generate Exec Report File` to it.
   - Create an **If** node named `Check Local Files Availability` (`{{ Object.keys($json).length === 0 }}`). Connect `Load PDFs from Disk` to it.
   - Create a **Split In Batches** node named `Loop Over File Batches`. Connect `Check Local Files Availability` (False branch) to it.
6. **Error Path & Text Extraction:**
   - Create a **Code** node named `Increment Total File Counter` to increment `total_files` in static storage. Connect `Loop Over File Batches` (Looping output) to it.
   - Create an **Extract from File** node named `Extract Data from PDF` (Operation: PDF, set `onError` to `continueErrorOutput`). Connect `Increment Total File Counter` to it.
   - **Error Sub-branch:** Connect error output to a **Code** node named `Increment Error File Counter`, then to a **Read/Write Files from Disk** node named `Log Error Files to Disk` (writing to error folder), then to a **Code** node named `Remove Error Files from Input` (`fs.unlinkSync`), and loop back to `Loop Over File Batches`.
7. **AI Processing & Normalization:**
   - From the success output of `Extract Data from PDF`, connect to a **Code** node named `Clean Extracted Text`.
   - Create an **OpenAI** node named `OpenAI Invoice Processing`. Set model to `gpt-4o-mini`, response format to JSON Schema (`invoice_output_schema`), and link your OpenAI API credential (`nx6fC2VKPhBpLz0a`). Connect `Clean Extracted Text` to it.
   - Create a **Code** node named `Normalize AI Output` to locate canonical invoice JSON. Connect `OpenAI Invoice Processing` to it.
   - Create a **Code** node named `Validate Extracted Invoice` to run required field and subtotal validation logic. Connect `Normalize AI Output` to it.
   - Create a **Code** node named `Split JSON Output` to flatten headers and line items into rows. Connect `Validate Extracted Invoice` to it.
8. **File Persisting & Archiving:**
   - Create a **Convert to File** node named `Convert JSON to File` (Excel format). Connect `Split JSON Output` to it.
   - Create a **Read/Write Files from Disk** node named `Manage Files on Disk` to write invoice Excel outputs. Connect `Convert JSON to File` to it.
   - Create an **If** node named `Verify Invoice Validity` (`validation_status` equals `VALID`). Connect `Manage Files on Disk` to it.
   - **Valid Branch:** Connect to **Code** (`Count Valid Files`) $\rightarrow$ **Read/Write Files from Disk** (`Store Valid Files Locally`) $\rightarrow$ **Code** (`Remove Valid Files from Input`).
   - **Invalid Branch:** Connect to **Code** (`Increment Invalid File Counter`) $\rightarrow$ **Read/Write Files from Disk** (`Archive Invalid Files Locally`) $\rightarrow$ **Code** (`Clear Invalid Files from Input`).
   - Create a **Merge** node named `Combine File Streams`. Connect both `Remove Valid Files from Input` and `Clear Invalid Files from Input` into it, and route the output back to `Loop Over File Batches`.
9. **Execution Wrap-Up:**
   - Create three nodes (`Setup Exec Report Initial Content`, `Reformat to File Format`, `Save Exec Report to File`) for both the empty file availability check (True branch) and the batch loop completion (Done output), linking them to overwrite the execution report Excel file on disk with final metrics.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| InvoiceExtractor-Local Workflow Reference | Complete automated invoice extraction, validation, and archiving workflow utilizing n8n and OpenAI GPT-4o-mini. |
| Filesystem Environment Requirements | Requires self-hosted n8n instance with `NODE_FUNCTION_ALLOW_BUILTIN=fs,path` enabled and local volume mounting for `/invoices`. |