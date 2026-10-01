Capture LINE receipt expenses with Gemini, Google Drive and Google Sheets

https://n8nworkflows.xyz/workflows/capture-line-receipt-expenses-with-gemini--google-drive-and-google-sheets-20073


# Capture LINE receipt expenses with Gemini, Google Drive and Google Sheets

### 1. Workflow Overview

This workflow automates the collection and processing of expense receipts sent by users via the LINE messaging platform. It extracts structured accounting data using Google Gemini, archives the physical receipt images in Google Drive, validates Japanese Qualified Invoice System T-numbers against the National Tax Agency (NTA) Web-API, logs the final records to Google Sheets, and replies to the user with a formatted confirmation in Japanese.

The logic is organized into the following functional blocks:

- **1.1 Input Reception & Routing:** Receives inbound webhooks from LINE, extracts event metadata, and branches based on whether the message type is an image or text/other.
- **1.2 Media Preparation & Archiving:** Downloads the image binary from LINE, concurrently uploads it to Google Drive as an archive, and converts it to Base64 format for AI consumption.
- **1.3 AI Extraction & Validation:** Submits the Base64 image to Google Gemini to classify whether it is a receipt and extract key-value accounting fields, returning a retry prompt if validation fails.
- **1.4 Japanese Invoice Verification (NTA):** Checks if a valid 13-digit Japanese T-number exists and queries the NTA invoice validity Web-API using the transaction date.
- **1.5 Ledger Recording & User Confirmation:** Appends the consolidated financial record to Google Sheets and sends an itemized confirmation message back to the user via the LINE Messaging API.

---

### 2. Block-by-Block Analysis

---

### 1.1 Input Reception & Routing
**Overview:**  
This block ingests the webhook payload from LINE, parses essential message tokens and IDs, and determines whether the incoming interaction is an image (receipt) or standard text requiring instructional guidance.

**Nodes Involved:**
- `When LINE Webhook Triggered`
- `Set LINE Message Data`
- `If LINE Message is Image`
- `Reply with Text Guide`

**Node Details:**

- **When LINE Webhook Triggered**
  - **Type & Role:** `n8n-nodes-base.webhook` (Trigger) — Listens for incoming HTTP POST payloads from LINE.
  - **Configuration:** HTTP Method `POST`, Path set to `line-expense-webhook`.
  - **Connections:** Output connects to `Set LINE Message Data`.
  - **Edge Cases:** Invalid endpoint configurations or missing network access from LINE servers will prevent trigger execution.

- **Set LINE Message Data**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Extracts specific fields from the nested LINE webhook JSON structure.
  - **Configuration:** Maps expression properties (`messageType`, `messageId`, `replyToken`, `userId`, `text`) using optional chaining on `{{ $json.body.events?.[0] }}`.
  - **Connections:** Input from `When LINE Webhook Triggered`; output to `If LINE Message is Image`.
  - **Edge Cases:** Malformed JSON events or missing event indexes result in empty string outputs.

- **If LINE Message is Image**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control) — Evaluates whether the extracted message type equals `image`.
  - **Configuration:** Condition evaluates `{{ $json.messageType }}` equals `image`.
  - **Connections:** Input from `Set LINE Message Data`. True branch goes to `Fetch LINE Image Content`; False branch goes to `Reply with Text Guide`.

- **Reply with Text Guide**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Integration) — Replies to non-image messages with instructions on how to submit a receipt.
  - **Configuration:** POST request to `https://api.line.me/v2/bot/message/reply` using generic header authentication (LINE Channel Access Token). Sends a Japanese help text message.
  - **Connections:** Input from the false branch of `If LINE Message is Image`.
  - **Edge Cases:** Expired LINE reply tokens (valid only for a short time after receipt) will cause API rejection.

---

### 1.2 Media Preparation & Archiving
**Overview:**  
Downloads the raw image binary associated with the LINE message ID, uploads an archived copy to Google Drive, and transforms the image buffer into a Base64 string for multimodal AI analysis.

**Nodes Involved:**
- `Fetch LINE Image Content`
- `Store Receipt in Google Drive`
- `Convert Image to Base64`

**Node Details:**

- **Fetch LINE Image Content**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Integration) — Downloads binary image content from LINE servers.
  - **Configuration:** Endpoint uses `https://api-data.line.me/v2/bot/message/{{ $json.messageId }}/content`, response format configured as `file`, authenticated via generic header auth.
  - **Connections:** Input from `If LINE Message is Image` (True branch); outputs to both `Store Receipt in Google Drive` and `Convert Image to Base64`.
  - **Edge Cases:** LINE message content retention policies mean older messages cannot be downloaded.

- **Store Receipt in Google Drive**
  - **Type & Role:** `n8n-nodes-base.googleDrive` (Storage Integration) — Uploads the receipt binary into a designated Google Drive folder.
  - **Configuration:** File name dynamically constructed using `Receipt_{{ userId }}_{{ timestamp }}.jpg`. Target folder set via `folderId` placeholder (`YOUR_DRIVE_FOLDER_ID`).
  - **Connections:** Input from `Fetch LINE Image Content`; output connects as input index `1` to `Merge Receipt Data`.
  - **Edge Cases:** Insufficient Google Drive permission scopes or invalid folder IDs will halt execution.

- **Convert Image to Base64**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript Transformation) — Reads binary buffer data from the preceding HTTP request and constructs a Base64 encoded property.
  - **Configuration:** Custom JS code utilizing `this.helpers.getBinaryDataBuffer(0, 'data')` to populate `item.json.imageBase64`.
  - **Connections:** Input from `Fetch LINE Image Content`; output connects to `Submit to Gemini for Analysis`.
  - **Edge Cases:** Missing binary data objects will trigger code execution exceptions.

---

### 1.3 AI Extraction & Validation
**Overview:**  
Submits the Base64 image payload to Google Gemini with a specialized system prompt, parses the structured JSON output, and checks whether the AI identified a valid receipt.

**Nodes Involved:**
- `Submit to Gemini for Analysis`
- `Extract Receipt Details`
- `Validate Receipt Presence`
- `Merge Receipt Data`
- `Notify Invalid Receipt in LINE`

**Node Details:**

- **Submit to Gemini for Analysis**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (AI Integration) — Calls the Google Gemini REST API (`gemini-3.8-flash:generateContent`).
  - **Configuration:** POST request with JSON payload containing prompt instructions for structured accounting extraction, inline image data, and `responseMimeType: 'application/json'`. Authenticated via generic header auth.
  - **Connections:** Input from `Convert Image to Base64`; output connects to `Extract Receipt Details`.
  - **Edge Cases:** API rate limits, quota exhaustion, or safety blockages by the model.

- **Extract Receipt Details**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript Transformation) — Cleans markdown code blocks from the Gemini response, parses the JSON string, and validates T-number formatting.
  - **Configuration:** Sanitizes raw text, runs `JSON.parse()`, cleans potential invoice strings, and determines `hasValidTaxFormat` based on a 13-digit sequence following a 'T' prefix.
  - **Connections:** Input from `Submit to Gemini for Analysis`; output connects to `Validate Receipt Presence`.
  - **Edge Cases:** Malformed or incomplete JSON output from the model is caught and safely handled as an invalid receipt.

- **Validate Receipt Presence**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control) — Evaluates whether `is_receipt` is true.
  - **Configuration:** Condition checks `{{ $json.is_receipt === true }}`.
  - **Connections:** Input from `Extract Receipt Details`. True branch connects to `Merge Receipt Data`; False branch connects to `Notify Invalid Receipt in LINE`.

- **Merge Receipt Data**
  - **Type & Role:** `n8n-nodes-base.merge` (Data Combination) — Combines parsed receipt data from the AI analysis branch with file metadata from the Google Drive upload branch.
  - **Configuration:** Mode configured to `combine` by position.
  - **Connections:** Input index `0` from `Validate Receipt Presence` (True); input index `1` from `Store Receipt in Google Drive`. Output connects to `Check for Invoice Number`.

- **Notify Invalid Receipt in LINE**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Integration) — Sends an automated Japanese notification telling the user the photo could not be recognized as a receipt.
  - **Configuration:** POST request to LINE reply API with guidance text asking for a clearer photo.
  - **Connections:** Input from the false branch of `Validate Receipt Presence`.

---

### 1.4 Japanese Invoice Verification (NTA)
**Overview:**  
Inspects the extracted data for a Qualified Invoice System T-number and, if present, queries the National Tax Agency (NTA) Web-API to verify registration status on the receipt date.

**Nodes Involved:**
- `Check for Invoice Number`
- `Verify Invoice via NTA API`
- `Build Valid Record`

**Node Details:**

- **Check for Invoice Number**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control) — Determines if a valid tax format number was extracted.
  - **Configuration:** Evaluates `{{ $json.hasValidTaxFormat }}` equals `true`.
  - **Connections:** Input from `Merge Receipt Data`. True branch goes to `Verify Invoice via NTA API`; False branch bypasses verification and goes directly to `Build Valid Record`.

- **Verify Invoice via NTA API**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Integration) — Queries the NTA Web-API endpoint to verify invoice status.
  - **Configuration:** URL uses `https://web-api.invoice-kohyo.nta.go.jp/1/valid?id=YOUR_NTA_APP_ID&number=T{{ $json.cleanTaxNumber }}&day={{ $json.receipt_date }}&type=21`. Error handling set to `continueRegularOutput`.
  - **Connections:** Input from the true branch of `Check for Invoice Number`; output connects to `Build Valid Record`.
  - **Edge Cases:** Network timeouts, invalid NTA application IDs, or API downtime will be caught gracefully via error settings.

- **Build Valid Record**
  - **Type & Role:** `n8n-nodes-base.code` (JavaScript Transformation) — Aggregates data across previous execution branches to assemble the final unified expense dataset.
  - **Configuration:** Combines Gemini receipt details, NTA validation response, Google Drive link references, and LINE user tokens.
  - **Connections:** Inputs from `Verify Invoice via NTA API` (and bypassed items from `Check for Invoice Number`). Output connects to `Append Expense to Sheets`.

---

### 1.5 Ledger Recording & User Confirmation
**Overview:**  
Appends the completed expense record to Google Sheets and sends an itemized confirmation message back to the user via LINE.

**Nodes Involved:**
- `Append Expense to Sheets`
- `Post Expense Receipt to LINE`

**Node Details:**

- **Append Expense to Sheets**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Productivity Integration) — Inserts a new row into the target accounting spreadsheet.
  - **Configuration:** Operation set to `append`, mapping document ID (`YOUR_GOOGLE_SHEETS_ID`) and sheet name (`YOUR_GOOGLE_SHEETS_NAME`). Explicitly maps columns including user ID, category, vendor name, amounts, invoice status, and Drive URL.
  - **Connections:** Input from `Build Valid Record`; output connects to `Post Expense Receipt to LINE`.
  - **Edge Cases:** Schema mismatches (missing spreadsheet columns) will result in API insertion errors.

- **Post Expense Receipt to LINE**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Integration) — Sends an itemized confirmation message to the submitting user in LINE.
  - **Configuration:** POST request to LINE reply API (`https://api.line.me/v2/bot/message/reply`) using generic header auth. Formats a structured Japanese summary including store name, payment date, amounts, tax breakdown, category, invoice number, verification status, and description.
  - **Connections:** Input from `Append Expense to Sheets`.
  - **Edge Cases:** Expired reply tokens or restricted bot permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Workflow documentation & setup guide | None | None | ## Capture LINE Receipt Expenses with Gemini and Verify Japanese Invoice Numbers... |
| Section 1 | n8n-nodes-base.stickyNote | Visual grouping for webhook & routing | None | None | ## 1. Receive and route LINE events... |
| Section 2 | n8n-nodes-base.stickyNote | Visual grouping for media prep & archive | None | None | ## 2. Prepare and archive receipt media... |
| Section 3 | n8n-nodes-base.stickyNote | Visual grouping for AI extraction & check | None | None | ## 3. Extract and validate receipt data... |
| Section 4 | n8n-nodes-base.stickyNote | Visual grouping for NTA invoice check | None | None | ## 4. Verify the Japanese invoice number... |
| Section 5 | n8n-nodes-base.stickyNote | Visual grouping for logging & confirmation | None | None | ## 5. Record and confirm the expense... |
| When LINE Webhook Triggered | n8n-nodes-base.webhook | Receives incoming webhook events | None | Set LINE Message Data | ## 1. Receive and route LINE events... |
| Set LINE Message Data | n8n-nodes-base.set | Extracts properties from webhook payload | When LINE Webhook Triggered | If LINE Message is Image | ## 1. Receive and route LINE events... |
| If LINE Message is Image | n8n-nodes-base.if | Routes flow based on image message type | Set LINE Message Data | Fetch LINE Image Content, Reply with Text Guide | ## 1. Receive and route LINE events... |
| Fetch LINE Image Content | n8n-nodes-base.httpRequest | Downloads image binary from LINE | If LINE Message is Image | Store Receipt in Google Drive, Convert Image to Base64 | ## 2. Prepare and archive receipt media... |
| Store Receipt in Google Drive | n8n-nodes-base.googleDrive | Archives receipt image in Google Drive | Fetch LINE Image Content | Merge Receipt Data | ## 2. Prepare and archive receipt media... |
| Convert Image to Base64 | n8n-nodes-base.code | Converts binary buffer to Base64 format | Fetch LINE Image Content | Submit to Gemini for Analysis | ## 2. Prepare and archive receipt media... |
| Submit to Gemini for Analysis | n8n-nodes-base.httpRequest | Calls Gemini API for OCR and extraction | Convert Image to Base64 | Extract Receipt Details | ## 3. Extract and validate receipt data... |
| Extract Receipt Details | n8n-nodes-base.code | Parses AI JSON and validates T-number | Submit to Gemini for Analysis | Validate Receipt Presence | ## 3. Extract and validate receipt data... |
| Check for Invoice Number | n8n-nodes-base.if | Checks if a valid T-number format exists | Merge Receipt Data | Verify Invoice via NTA API, Build Valid Record | ## 4. Verify the Japanese invoice number... |
| Verify Invoice via NTA API | n8n-nodes-base.httpRequest | Queries National Tax Agency API | Check for Invoice Number | Build Valid Record | ## 4. Verify the Japanese invoice number... |
| Build Valid Record | n8n-nodes-base.code | Combines data into a unified expense record | Check for Invoice Number, Verify Invoice via NTA API | Append Expense to Sheets | ## 4. Verify the Japanese invoice number... |
| Append Expense to Sheets | n8n-nodes-base.googleSheets | Logs expense record to Google Sheets | Build Valid Record | Post Expense Receipt to LINE | ## 5. Record and confirm the expense... |
| Post Expense Receipt to LINE | n8n-nodes-base.httpRequest | Sends confirmation message to LINE user | Append Expense to Sheets | None | ## 5. Record and confirm the expense... |
| Reply with Text Guide | n8n-nodes-base.httpRequest | Sends text instructions for non-images | If LINE Message is Image | None | ## 1. Receive and route LINE events... |
| Validate Receipt Presence | n8n-nodes-base.if | Checks whether AI confirmed valid receipt | Extract Receipt Details | Merge Receipt Data, Notify Invalid Receipt in LINE | ## 3. Extract and validate receipt data... |
| Merge Receipt Data | n8n-nodes-base.merge | Joins receipt data with Drive file metadata | Store Receipt in Google Drive, Validate Receipt Presence | Check for Invoice Number | ## 3. Extract and validate receipt data... |
| Notify Invalid Receipt in LINE | n8n-nodes-base.httpRequest | Sends retry guidance for invalid receipts | Validate Receipt Presence | None | ## 3. Extract and validate receipt data... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger**
   - Add a **Webhook** node named `When LINE Webhook Triggered`.
   - Set HTTP Method to `POST` and Path to `line-expense-webhook`.

2. **Extract LINE Event Metadata**
   - Add a **Set** node named `Set LINE Message Data`.
   - Create string assignments:
     - `messageType`: `={{ $json.body.events?.[0]?.message?.type || '' }}`
     - `messageId`: `={{ $json.body.events?.[0]?.message?.id || '' }}`
     - `replyToken`: `={{ $json.body.events?.[0]?.replyToken || '' }}`
     - `userId`: `={{ $json.body.events?.[0]?.source?.userId || '' }}`
     - `text`: `={{ $json.body.events?.[0]?.message?.text || '' }}`

3. **Branch by Message Type**
   - Add an **If** node named `If LINE Message is Image`.
   - Configure condition: `{{ $json.messageType }}` equals `image`.
   - Connect `Set LINE Message Data` output to this node.

4. **Handle Non-Image Messages (False Branch)**
   - Add an **HTTP Request** node named `Reply with Text Guide`.
   - Set Method to `POST`, URL to `https://api.line.me/v2/bot/message/reply`, and configure generic header authentication (LINE Channel Access Token).
   - Set JSON body to send a welcome/instructional message using `replyToken`.

5. **Download Receipt Media (True Branch)**
   - Add an **HTTP Request** node named `Fetch LINE Image Content`.
   - Set Method to `GET`, URL to `=https://api-data.line.me/v2/bot/message/{{ $json.messageId }}/content`, response format to `File`, and configure LINE HTTP header authentication.

6. **Archive and Convert Media Concurrently**
   - **Google Drive Node:** Add a **Google Drive** node named `Store Receipt in Google Drive`. Set operation to upload, specify target folder ID (`YOUR_DRIVE_FOLDER_ID`), and name file `=Receipt_{{ $('Set LINE Message Data').item.json.userId }}_{{$now.setZone('Asia/Tokyo').toFormat('yyyyMMdd_HHmmss') }}.jpg`. Connect input from `Fetch LINE Image Content`.
   - **Code Node:** Add a **Code** node named `Convert Image to Base64`. Insert JavaScript to extract the binary buffer and assign `item.json.imageBase64` and `item.json.mimeType`. Connect input from `Fetch LINE Image Content`.

7. **Call Google Gemini for OCR**
   - Add an **HTTP Request** node named `Submit to Gemini for Analysis`.
   - Set Method to `POST`, URL to `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.8-flash:generateContent`, and configure generic header auth for the Gemini API key.
   - Send JSON body structured with a system prompt instructing the model to return JSON with accounting fields and inline image data.

8. **Parse AI Output**
   - Add a **Code** node named `Extract Receipt Details`.
   - Insert JavaScript to strip markdown formatting, parse JSON safely, validate T-number length (13 digits), and output structured attributes (`is_receipt`, `receipt_date`, `vendor_name`, `total_amount`, `tax_amount`, `invoice_number`, `cleanTaxNumber`, `hasValidTaxFormat`, `category`, `description`).

9. **Validate Receipt Presence**
   - Add an **If** node named `Validate Receipt Presence`.
   - Condition checks `{{ $json.is_receipt }}` equals `true`.
   - **False branch:** Connects to an **HTTP Request** node (`Notify Invalid Receipt in LINE`) replying to the user in LINE with retry instructions.
   - **True branch:** Connects to `Merge Receipt Data`.

10. **Merge Metadata and Receipt Data**
    - Add a **Merge** node named `Merge Receipt Data` configured to combine by position.
    - Connect input `0` from `Validate Receipt Presence` and input `1` from `Store Receipt in Google Drive`.

11. **Check for Invoice Number**
    - Add an **If** node named `Check for Invoice Number`.
    - Condition checks `{{ $json.hasValidTaxFormat }}` equals `true`.
    - **True branch:** Connects to `Verify Invoice via NTA API`.
    - **False branch:** Connects directly to `Build Valid Record`.

12. **Verify NTA Invoice Status**
    - Add an **HTTP Request** node named `Verify Invoice via NTA API`.
    - Set URL to `=https://web-api.invoice-kohyo.nta.go.jp/1/valid?id=YOUR_NTA_APP_ID&number=T{{ $json.cleanTaxNumber }}&day={{ $json.receipt_date }}&type=21`.
    - Set error handling parameter `OnError` to `continueRegularOutput`.

13. **Build Consolidated Expense Record**
    - Add a **Code** node named `Build Valid Record`.
    - Insert JavaScript to combine receipt properties, NTA verification announcements, Google Sheets formatting timestamps, and reply tokens.

14. **Record to Google Sheets and Reply in LINE**
    - **Google Sheets Node:** Add a **Google Sheets** node named `Append Expense to Sheets`. Set operation to `append`, configure document ID (`YOUR_GOOGLE_SHEETS_ID`) and sheet name (`YOUR_GOOGLE_SHEETS_NAME`), and map all extracted financial columns.
    - **HTTP Request Node:** Add an **HTTP Request** node named `Post Expense Receipt to LINE` connected after the Sheets node. Send a POST request to `https://api.line.me/v2/bot/message/reply` with a detailed Japanese confirmation message containing expense itemization and tax validation results.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Disclaimer: The provided text originates exclusively from an automated n8n workflow complying with content policies and utilizing public data. | Workflow metadata and compliance disclaimer |