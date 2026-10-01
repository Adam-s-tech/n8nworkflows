Create technical interview study guides with OpenAI, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/create-technical-interview-study-guides-with-openai--google-sheets--and-gmail-19806


# Create technical interview study guides with OpenAI, Google Sheets, and Gmail

### 1. Workflow Overview

This workflow is designed to automate the collection, analysis, storage, and distribution of technical interview preparation materials. Its primary target use cases include job seekers and technical educators who want to generate structured study guides dynamically, maintain a searchable archive in a spreadsheet, and receive email summaries for offline review.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Receives authenticated form submissions containing technical interview queries and assigns downstream global operational parameters.
- **1.2 Data Validation & AI Processing:** Sanitizes inputs, validates payload boundaries, and interacts with the OpenAI Chat Completions API using a structured system prompt to return a JSON-formatted study guide.
- **1.3 Persistence & Notification:** Parses and validates the AI payload, calculates a review schedule, appends the finalized record to a Google Sheets worksheet, and dispatches an email summary via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** This block captures the raw user input securely via an authenticated web form, sets the foundational environment parameters (such as targeted spreadsheet IDs and model selection), and routes the combined data downstream.
- **Nodes Involved:** 
  - `Submit interview question`
  - `Configure study settings`

- **Node Details:**
  - **Submit interview question**
    - *Type and technical role:* `n8n-nodes-base.formTrigger` (Trigger node). Acts as a public or secured form entry point, gathering interview inputs and pausing execution until submitted.
    - *Configuration choices:* Uses Basic Authentication (`basicAuth`) to restrict access. Configures fields for "Interview question" (textarea, required), "Target role" (text, required), and "Your draft answer" (textarea, optional). Sets response mode to return data from the last node upon completion.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: None (Trigger); Output: `Configure study settings`.
    - *Version-specific requirements:* Version 2.2.
    - *Edge cases or potential failure types:* Unauthorized access attempts if Basic Auth is misconfigured; empty submissions if required fields are bypassed.
  - **Configure study settings**
    - *Type and technical role:* `n8n-nodes-base.set` (Data transformation/assignment node). Hardcodes or initializes contextual configuration settings required by subsequent nodes.
    - *Configuration choices:* Sets workflow-wide string assignments: `spreadsheetId`, `worksheetName`, `recipientEmail`, and `openaiModel`. Retains other incoming fields from the form trigger.
    - *Key expressions or variables:* Explicit string assignments (e.g., `REPLACE_WITH_SPREADSHEET_ID`, `gpt-4o-mini`).
    - *Input and output connections:* Input: `Submit interview question`; Output: `Validate study input`.
    - *Version-specific requirements:* Version 3.4.
    - *Edge cases or potential failure types:* Placeholder values (`REPLACE_...`) left unconfigured will cause downstream validation errors.

---

#### 1.2 Data Validation & AI Processing
- **Overview:** This block validates user input limits, prevents injection or format errors, queries the OpenAI Chat Completions API with a constrained JSON system prompt, and parses the returned model output.
- **Nodes Involved:** 
  - `Validate study input`
  - `Analyze question with OpenAI`
  - `Prepare study record`

- **Node Details:**
  - **Validate study input**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript processing node). Validates types, trims whitespace, enforces character length constraints, and checks for unconfigured placeholder values.
    - *Configuration choices:* Executes native JavaScript code evaluating limits (question max 8,000 chars, draft max 12,000 chars, role max 200 chars) and ensuring valid email formatting.
    - *Key expressions or variables:* Accesses `$input.first().json` properties including `Interview question`, `Target role`, `Your draft answer`, `spreadsheetId`, and `recipientEmail`.
    - *Input and output connections:* Input: `Configure study settings`; Output: `Analyze question with OpenAI`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Throws explicit runtime errors if inputs exceed maximum lengths, if required fields are missing, or if placeholder values persist.
  - **Analyze question with OpenAI**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (External API integration node). Sends a POST request to OpenAI to generate study guide JSON content.
    - *Configuration choices:* Targets `https://api.openai.com/v1/chat/completions` with a 90-second timeout. Sends a JSON body configured for JSON mode (`response_format: {type: "json_object"}`), a system prompt defining tutor parameters, and a user prompt containing the validated question, role, and draft. Configures retry parameters (`maxTries: 2`, `waitBetweenTries: 3000`). Authenticated via predefined credentials (`openAiApi`).
    - *Key expressions or variables:* Uses `{{ JSON.stringify({ model: $json.openaiModel, messages: [...], ... }) }}` to dynamically pass the selected model and user/system payloads.
    - *Input and output connections:* Input: `Validate study input`; Output: `Prepare study record`.
    - *Version-specific requirements:* Version 4.2.
    - *Edge cases or potential failure types:* API rate limits, authentication token errors, request timeouts on large models, or non-JSON responses.
  - **Prepare study record**
    - *Type and technical role:* `n8n-nodes-base.code` (Custom JavaScript processing node). Parses the OpenAI response, ensures structural completeness, escapes potential CSV injection characters, calculates review timelines, and formats the plain-text email body.
    - *Configuration choices:* Executes native JavaScript validating `finish_reason === 'stop'`, parsing JSON string outputs, checking for required keys (`topic`, `difficulty`, `reference_answer`, `key_concepts`, `common_mistakes`, `follow_up_questions`, `draft_feedback`), generating a review timestamp 3 days in the future, and assembling a plain-text email template.
    - *Key expressions or variables:* Accesses `$input.first().json.choices` and references the upstream `Validate study input` node using `$('Validate study input').first().json`.
    - *Input and output connections:* Input: `Analyze question with OpenAI`; Output: `Save study guide to Google Sheets`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases or potential failure types:* Fails if OpenAI returns an incomplete response, cuts off generation prematurely, or omits required JSON keys.

---

#### 1.3 Persistence & Notification
- **Overview:** This block persists the processed study guide data into a designated Google spreadsheet and distributes the summary directly to the user via Gmail.
- **Nodes Involved:** 
  - `Save study guide to Google Sheets`
  - `Email study guide with Gmail`

- **Node Details:**
  - **Save study guide to Google Sheets**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` (App integration node). Appends a new formatted row containing all study guide attributes to an existing Google spreadsheet.
    - *Configuration choices:* Operation set to `append` using raw cell formatting (`cellFormat: "RAW"`). Maps columns (`created_at`, `question`, `target_role`, `draft_answer`, `topic`, `difficulty`, `reference_answer`, `key_concepts`, `common_mistakes`, `follow_up_questions`, `draft_feedback`, `review_on`, `status`) to incoming payload variables. Uses expressions to dynamically resolve spreadsheet ID and worksheet name from configuration settings.
    - *Key expressions or variables:* Uses `={{ $('Configure study settings').first().json.spreadsheetId }}`, `={{ $('Configure study settings').first().json.worksheetName }}`, and maps fields via `={{ $json.row.<field_name> }}`.
    - *Input and output connections:* Input: `Prepare study record`; Output: `Email study guide with Gmail`.
    - *Version-specific requirements:* Version 4.5.
    - *Edge cases or potential failure types:* Authentication expiration, missing spreadsheet columns, or permission issues blocking append operations.
  - **Email study guide with Gmail**
    - *Type and technical role:* `n8n-nodes-base.gmail` (App integration node). Dispatches the compiled text-based study guide summary to the designated recipient email address.
    - *Configuration choices:* Sends a plain text message (`emailType: "text"`) with resource set to `message`, operation set to `send`, and `appendAttribution` disabled.
    - *Key expressions or variables:* Uses `={{ $('Configure study settings').first().json.recipientEmail }}` for the recipient and `={{ $('Prepare study record').first().json.emailBody }}` for the message content.
    - *Input and output connections:* Input: `Save study guide to Google Sheets`; Output: None (Terminal node).
    - *Version-specific requirements:* Version 2.1.
    - *Edge cases or potential failure types:* OAuth scope limitations, invalid recipient address formats, or network-level SMTP/API blocks.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Submit interview question | `n8n-nodes-base.formTrigger` | Triggers the workflow via an authenticated web form submission. | None | Configure study settings | Job seekers who want a searchable study log of technical interview questions and clear explanations for later practice.<br><br>Submit a question, target role, and optional draft answer through an n8n form. OpenAI creates a study guide covering key concepts, a reference answer, common mistakes, follow-up questions, and feedback on the draft. The workflow validates the response, saves it to Google Sheets, and sends a plain-text summary to your configured Gmail recipient. Each row includes a suggested review date three days later; this does not schedule a reminder.<br><br>Connect Form Basic Auth, OpenAI, Google Sheets, and Gmail credentials. In Configure study settings, set your spreadsheet ID, worksheet name, recipient email, and an available OpenAI model. Create a sheet using the supplied column headers. Test with a practice question before activating the workflow.<br><br>Requires an n8n instance, an OpenAI API account with available credit, a Google spreadsheet, and Gmail. Customize the prompt and review interval for your study routine. AI explanations may be incorrect: verify technical claims. Submit only questions you are permitted to share with OpenAI and Google.<br><br>Created by [SkillCopilotAI](https://skillcopilotai.com/), an AI interview assistance product. |
| Configure study settings | `n8n-nodes-base.set` | Assigns global environment variables (Spreadsheet ID, Worksheet Name, Email, Model). | Submit interview question | Validate study input | 1. Configure before testing<br>Connect Form Basic Auth and keep access private. Set spreadsheetId, worksheetName, recipientEmail and openaiModel in Configure study settings. The recipient is fixed by the owner, never taken from public form input. Import sheet-headers.csv into your worksheet.<br><br>Use Execute Workflow and the Form Test URL. Keep the workflow inactive until a successful end-to-end test. |
| Validate study input | `n8n-nodes-base.code` | Sanitizes strings and validates length, placeholders, and email syntax. | Configure study settings | Analyze question with OpenAI | 1. Configure before testing<br>Connect Form Basic Auth and keep access private. Set spreadsheetId, worksheetName, recipientEmail and openaiModel in Configure study settings. The recipient is fixed by the owner, never taken from public form input. Import sheet-headers.csv into your worksheet.<br><br>Use Execute Workflow and the Form Test URL. Keep the workflow inactive until a successful end-to-end test. |
| Analyze question with OpenAI | `n8n-nodes-base.httpRequest` | Calls OpenAI API to generate structured technical study guides in JSON format. | Validate study input | Prepare study record | 2. Connect OpenAI<br>Select your OpenAI credential in the HTTP Request node. No API key is included in this template. JSON output is validated before writing to Sheets. Incomplete responses stop the execution. A failed API request may retry once and incur additional usage.<br><br>This template uses the OpenAI API through the standard HTTP Request node. |
| Prepare study record | `n8n-nodes-base.code` | Parses AI JSON output, applies security sanitization, and formats email bodies. | Analyze question with OpenAI | Save study guide to Google Sheets | 2. Connect OpenAI<br>Select your OpenAI credential in the HTTP Request node. No API key is included in this template. JSON output is validated before writing to Sheets. Incomplete responses stop the execution. A failed API request may retry once and incur additional usage.<br><br>This template uses the OpenAI API through the standard HTTP Request node. |
| Save study guide to Google Sheets | `n8n-nodes-base.googleSheets` | Appends structured study guide items into the designated Google Sheets spreadsheet. | Prepare study record | Email study guide with Gmail | 3. Connect Google Sheets and Gmail<br>Use the supplied exact headers. Connect your own credentials in both nodes. The email is plain text. Review dates are recorded only; no scheduled reminder is created.<br><br>If Gmail fails after Sheets succeeds, retry only Gmail. Re-running the full workflow appends another row.<br><br>Draft: static checks only; run an end-to-end test in your n8n instance before submitting for publication. |
| Email study guide with Gmail | `n8n-nodes-base.gmail` | Dispatches the plain-text study guide email summary to the configured recipient. | Save study guide to Google Sheets | None | 3. Connect Google Sheets and Gmail<br>Use the supplied exact headers. Connect your own credentials in both nodes. The email is plain text. Review dates are recorded only; no scheduled reminder is created.<br><br>If Gmail fails after Sheets succeeds, retry only Gmail. Re-running the full workflow appends another row.<br><br>Draft: static checks only; run an end-to-end test in your n8n instance before submitting for publication. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Add a `Form Trigger` node (`n8n-nodes-base.formTrigger`, v2.2).
   - Set Authentication to `Basic Auth` and configure valid credentials.
   - Set the form title to `Technical interview study log`.
   - Add three form fields:
     - Field 1: Type `Textarea`, Label `Interview question`, Required: `true`.
     - Field 2: Type `Text`, Label `Target role`, Required: `true`.
     - Field 3: Type `Textarea`, Label `Your draft answer`, Required: `false`.
   - Set the post-submission text (`formSubmittedText`) to `Your study guide has been saved and emailed.`

2. **Create the Configuration Node:**
   - Add a `Set` node (`n8n-nodes-base.set`, v3.4), and connect it from the Form Trigger.
   - Define four string assignments:
     - `spreadsheetId`: `REPLACE_WITH_SPREADSHEET_ID` (replace with your actual Google Sheet ID).
     - `worksheetName`: `Interview Prep`.
     - `recipientEmail`: `you@example.com` (replace with your destination email).
     - `openaiModel`: `gpt-4o-mini`.

3. **Create the Input Validation Node:**
   - Add a `Code` node (`n8n-nodes-base.code`, v2), and connect it from the Set node.
   - Paste JavaScript logic to validate payload presence, string length limits (question ≤ 8000, draft ≤ 12000, role ≤ 200), and configuration placeholders.

4. **Create the OpenAI HTTP Request Node:**
   - Add an `HTTP Request` node (`n8n-nodes-base.httpRequest`, v4.2), and connect it from the Code validation node.
   - Set method to `POST` and URL to `https://api.openai.com/v1/chat/completions`.
   - Configure Authentication using predefined OpenAI credentials (`openAiApi`).
   - Set Request Body Type to `JSON`, enabling retry parameters (max retries: 2, wait time: 3000ms).
   - Populate the JSON body expression incorporating system constraints, JSON response format configuration, and dynamic user variables.

5. **Create the Data Preparation Node:**
   - Add a second `Code` node (`n8n-nodes-base.code`, v2), and connect it from the OpenAI HTTP Request node.
   - Insert JavaScript code to parse choices, validate JSON completion status, verify required analytical fields (`topic`, `difficulty`, `reference_answer`, `key_concepts`, `common_mistakes`, `follow_up_questions`, `draft_feedback`), escape CSV injection risks, compute a review timestamp (`+3 days`), and compile the text email body.

6. **Create the Google Sheets Integration Node:**
   - Add a `Google Sheets` node (`n8n-nodes-base.googleSheets`, v4.5), and connect it from the Preparation Code node.
   - Authenticate via Google OAuth2 credentials.
   - Set operation to `Append` and cell format to `RAW`.
   - Link Document ID and Sheet Name dynamically to the upstream configuration node values.
   - Map all 13 sheet columns (`created_at`, `question`, `target_role`, `draft_answer`, `topic`, `difficulty`, `reference_answer`, `key_concepts`, `common_mistakes`, `follow_up_questions`, `draft_feedback`, `review_on`, `status`) to their corresponding `$json.row` properties.

7. **Create the Gmail Delivery Node:**
   - Add a `Gmail` node (`n8n-nodes-base.gmail`, v2.1), and connect it from the Google Sheets node.
   - Authenticate via Gmail OAuth2 credentials.
   - Set Resource to `Message` and Operation to `Send`.
   - Set Email Type to `Text`.
   - Map the recipient (`sendTo`) to `{{ $('Configure study settings').first().json.recipientEmail }}` and message body to `{{ $('Prepare study record').first().json.emailBody }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by SkillCopilotAI, an AI interview assistance product. | [SkillCopilotAI Website](https://skillcopilotai.com/) |