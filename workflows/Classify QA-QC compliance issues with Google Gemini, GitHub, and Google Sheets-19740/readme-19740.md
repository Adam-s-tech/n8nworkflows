Classify QA/QC compliance issues with Google Gemini, GitHub, and Google Sheets

https://n8nworkflows.xyz/workflows/classify-qa-qc-compliance-issues-with-google-gemini--github--and-google-sheets-19740


# Classify QA/QC compliance issues with Google Gemini, GitHub, and Google Sheets

### 1. Workflow Overview

This workflow acts as an automated QA/QC compliance and non-conformance RAG (Retrieval-Augmented Generation) copilot. It ingests chat-based user queries concerning quality assurance procedures, validates the input length, leverages an advanced AI agent backed by Google Gemini to query a controlled GitHub repository of quality documents, extracts structured compliance assessments, and routes decisions accordingly. Depending on the AI's output, it logs the results to Google Sheets and triggers automated escalation or manual review notifications via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Validation:** Captures incoming chat messages, assigns session and timing metadata, and verifies that the query meets length constraints.
- **1.2 AI RAG Processing:** Utilizes Google Gemini, connected tools (GitHub repository file listing and file retrieval), and a structured output parser to evaluate compliance against approved knowledge-base documents.
- **1.3 Normalization & Manual Review Routing:** Standardizes the AI output payload and branches between automated audit logging and manual review processing.
- **1.4 Automated Decision Logging & Escalation:** Appends valid QA decisions to Google Sheets, evaluates whether risk or severity mandates escalation, dispatches escalation notices, and updates tracking statuses.
- **1.5 Manual Review Handling:** Logs unresolved or uncertain cases requiring human intervention into Google Sheets and notifies the QA team via email.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Validation
- **Overview:** This block receives the user's chat input, enriches it with session tracking and localized timestamps, and validates that the prompt is long enough to be processed as a legitimate query.
- **Nodes Involved:** 
  - `When Chat Message Received`
  - `Set Query Metadata`
  - `If Valid QA Query`

##### Node Details:
- **When Chat Message Received**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.chatTrigger` – Acts as the chat entry point for users interacting with the copilot.
  - **Configuration:** Default options.
  - **Key Expressions / Variables:** Outputs `chatInput` and `sessionId`.
  - **Connections:** Input: None (Trigger); Output: Main connection to `Set Query Metadata`.
  - **Edge Cases / Potential Failures:** Disconnected chat sessions or malformed payloads from the UI widget.

- **Set Query Metadata**
  - **Type & Technical Role:** `n8n-nodes-base.set` – Enriches the trigger output with structured metadata fields.
  - **Configuration:** Assigns custom variables (`query`, `session_id`, `timestamp`, `source`) while retaining other input fields.
  - **Key Expressions / Variables:** 
    - `query`: `={{$json.chatInput}}`
    - `session_id`: `={{$json.sessionId}}`
    - `timestamp`: `={{$now.setZone('Asia/Karachi').toISO()}}`
    - `source`: `chat`
  - **Connections:** Input: `When Chat Message Received`; Output: Main connection to `If Valid QA Query`.
  - **Edge Cases / Potential Failures:** Timezone evaluation failures if the target zone string is unsupported.

- **If Valid QA Query**
  - **Type & Technical Role:** `n8n-nodes-base.if` – Conditional router ensuring input quality before engaging resource-heavy LLM calls.
  - **Configuration:** Strict type validation checking whether the query exists and has a trimmed length of 10 characters or more.
  - **Key Expressions / Variables:** `={{ !!$json.query && $json.query.trim().length >= 10 }}`
  - **Connections:** Input: `Set Query Metadata`; Output (True): Main connection to `QA/QC Compliance Agent`; Output (False): None (stops execution).
  - **Edge Cases / Potential Failures:** Empty strings or whitespace-only inputs bypassed if trimming logic fails.

---

#### Block 1.2: AI RAG Processing
- **Overview:** Executes a LangChain-based agent powered by Google Gemini, querying a GitHub repository containing approved QA/QC documentation to generate a strictly grounded compliance assessment.
- **Nodes Involved:** 
  - `QA/QC Compliance Agent`
  - `Google Gemini Chatbot`
  - `Fetch QA Knowledge Files`
  - `Get QA Document`
  - `Parse Structured Output`

##### Node Details:
- **QA/QC Compliance Agent**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` – Core reasoning engine coordinating tool use and prompt instructions.
  - **Configuration:** Defined prompt type with a specialized QA/QC system message restricting the model to retrieved knowledge-base documents only. Enforces output parsing.
  - **Key Expressions / Variables:** Text input: `={{$json.query}}`
  - **Connections:** Input: `If Valid QA Query` (Main), `Google Gemini Chatbot` (AI Model), `Fetch QA Knowledge Files` & `Get QA Document` (AI Tools), `Parse Structured Output` (AI Output Parser); Output: Main connection to `Normalize Compliance Data`.
  - **Edge Cases / Potential Failures:** Tool execution timeouts, hallucinated filenames, or LLM refusal responses.

- **Google Gemini Chatbot**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` – Language model provider node.
  - **Configuration:** Model name set to `models/gemini-3.5-flash-lite`.
  - **Connections:** Output: AI Language Model connection to `QA/QC Compliance Agent`.
  - **Edge Cases / Potential Failures:** API rate limits, authentication token expiration, or model deprecation.

- **Fetch QA Knowledge Files**
  - **Type & Technical Role:** `n8n-nodes-base.githubTool` – Tool allowing the agent to list documents in the repository.
  - **Configuration:** Accesses owner `fahimjilani`, repository `qa-qc-rag-knowledge-base`, targeting path `docs` with operation `list`.
  - **Connections:** Output: AI Tool connection to `QA/QC Compliance Agent`.
  - **Edge Cases / Potential Failures:** GitHub API rate limits or repository permission errors.

- **Get QA Document**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequestTool` – HTTP request tool utilized by the agent to fetch raw file contents from GitHub.
  - **Configuration:** Dynamic URL construction pointing to the raw repository content using the filename returned by the previous tool.
  - **Key Expressions / Variables:** `={{ 'https://raw.githubusercontent.com/fahimjilani/qa-qc-rag-knowledge-base/main/docs/' + $fromAI('filename', '...') }}`
  - **Connections:** Output: AI Tool connection to `QA/QC Compliance Agent`.
  - **Edge Cases / Potential Failures:** 404 errors if the AI selects a non-existent filename or `README.md`.

- **Parse Structured Output**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` – Enforces a strict JSON schema for the agent's final response.
  - **Configuration:** Configured with a schema defining properties such as `classification`, `severity_score`, `requirement_violated`, `recommended_action`, `escalation_required`, `capa_required`, `manual_review_required`, `source_files`, and `reason`.
  - **Connections:** Output: AI Output Parser connection to `QA/QC Compliance Agent`.
  - **Edge Cases / Potential Failures:** Parsing errors if the LLM output violates the expected data types.

---

#### Block 1.3: Normalization & Manual Review Routing
- **Overview:** Extracts parsed JSON fields from the agent's output, re-attaches session metadata, and evaluates whether human intervention is mandatory.
- **Nodes Involved:** 
  - `Normalize Compliance Data`
  - `Check Manual Review Requirement`

##### Node Details:
- **Normalize Compliance Data**
  - **Type & Technical Role:** `n8n-nodes-base.set` – Standardizes fields from the structured AI parser and merges them with original query metadata.
  - **Configuration:** Assignments mapping all individual output properties (`classification`, `severity_score`, `requirement_violated`, etc.) alongside original query tracking data.
  - **Key Expressions / Variables:** Maps attributes from `={{$json.output...}}` and references upstream query data via `={{ $('Set Query Metadata').item.json... }}`.
  - **Connections:** Input: `QA/QC Compliance Agent`; Output: Main connection to `Check Manual Review Requirement`.
  - **Edge Cases / Potential Failures:** Undefined property references if upstream parsing fails to emit expected keys.

- **Check Manual Review Requirement**
  - **Type & Technical Role:** `n8n-nodes-base.if` – Conditional gate checking if `manual_review_required` is true.
  - **Configuration:** Evaluates boolean truthiness of `manual_review_required`.
  - **Key Expressions / Variables:** `={{$json.manual_review_required}}`
  - **Connections:** Input: `Normalize Compliance Data`; Output (True): Main connection to `Append Manual Review to Sheets`; Output (False): Main connection to `Route by Classification`.
  - **Edge Cases / Potential Failures:** Type coercion discrepancies if boolean values are returned as strings.

---

#### Block 1.4: Automated Decision Logging & Escalation
- **Overview:** Routes standard classifications (Compliant, Observation, Minor/Major NCR), logs audit records into Google Sheets, assesses escalation requirements, sends alerts, and updates tracking statuses.
- **Nodes Involved:** 
  - `Route by Classification`
  - `Append QA Decision to Sheets`
  - `Check Escalation Requirement`
  - `Send Escalation Email`
  - `Update Escalation Status`

##### Node Details:
- **Route by Classification**
  - **Type & Technical Role:** `n8n-nodes-base.switch` – Multi-branch router for standard QA outcomes.
  - **Configuration:** Four output rules matching exact string values: `Compliant`, `Observation`, `Minor Non-Conformance`, and `Major Non-Conformance`.
  - **Key Expressions / Variables:** `={{$json.classification}}`
  - **Connections:** Input: `Check Manual Review Requirement` (False branch); Outputs (All 4 branches): Main connections to `Append QA Decision to Sheets`.
  - **Edge Cases / Potential Failures:** Unhandled classification strings causing items to be dropped if they do not match any rule.

- **Append QA Decision to Sheets**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` – Appends audit entries to the decision log spreadsheet.
  - **Configuration:** Operation `append` on document `QA_QC_RAG_Compliance_Log`, sheet `QA_Decision_Log`. Maps columns including generated unique record IDs (`QA-<timestamp>`), final status (`Closed` for compliant, `Open` otherwise), and escalation statuses.
  - **Key Expressions / Variables:** 
    - `Record_ID`: `={{ 'QA-' + $now.toMillis() }}`
    - `Final_Status`: `={{ $json.classification === 'Compliant' ? 'Closed' : 'Open' }}`
  - **Connections:** Input: `Route by Classification`; Output: Main connection to `Check Escalation Requirement`.
  - **Edge Cases / Potential Failures:** Google Sheets API rate limits, quota limits, or column header mismatch.

- **Check Escalation Requirement**
  - **Type & Technical Role:** `n8n-nodes-base.if` – Evaluates whether the logged decision requires escalation.
  - **Configuration:** Checks if `escalation_required` from the upstream normalization node evaluates to true.
  - **Key Expressions / Variables:** `={{ $('Normalize Compliance Data').item.json.escalation_required }}`
  - **Connections:** Input: `Append QA Decision to Sheets`; Output (True): Main connection to `Send Escalation Email`; Output (False): None.
  - **Edge Cases / Potential Failures:** Missing node reference lookups if execution paths become complex.

- **Send Escalation Email**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` – Sends an HTML escalation notification to the QA team.
  - **Configuration:** Recipient set to `user@example.com`, subject and HTML body dynamically constructed using sheet record values.
  - **Key Expressions / Variables:** Uses fields like `{{$json.Record_ID}}`, `{{$json.Classification}}`, and `{{$json.Severity_Score}}`.
  - **Connections:** Input: `Check Escalation Requirement` (True branch); Output: Main connection to `Update Escalation Status`.
  - **Edge Cases / Potential Failures:** Gmail authentication expiration, daily sending quotas, or invalid recipient addresses.

- **Update Escalation Status**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` – Updates the existing audit log row to mark escalation as completed.
  - **Configuration:** Operation `update` on document `QA_QC_RAG_Compliance_Log`, sheet `QA_Decision_Log`, matching rows by `Record_ID`.
  - **Key Expressions / Variables:** 
    - `Record_ID`: `={{ $('Append QA Decision to Sheets').item.json.Record_ID }}`
    - `Escalation_Status`: `Sent`
  - **Connections:** Input: `Send Escalation Email`; Output: None (Terminal node).
  - **Edge Cases / Potential Failures:** Failure to match the `Record_ID` causing update failures.

---

#### Block 1.5: Manual Review Handling
- **Overview:** Logs unclassified or uncertain cases into Google Sheets under a manual review status and alerts reviewers via Gmail.
- **Nodes Involved:** 
  - `Append Manual Review to Sheets`
  - `Send Review Alert Email`

##### Node Details:
- **Append Manual Review to Sheets**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` – Appends a pending manual review record to the decision log.
  - **Configuration:** Operation `append` on document `QA_QC_RAG_Compliance_Log`, sheet `QA_Decision_Log`. Sets classification to `Pending Manual Review` and final status to `Under Review`.
  - **Key Expressions / Variables:** 
    - `Record_ID`: `={{ 'QA-' + $now.toMillis() }}`
    - `Source_Files`: `={{ $('Normalize Compliance Data').item.json.source_files.join(', ') }}`
  - **Connections:** Input: `Check Manual Review Requirement` (True branch); Output: Main connection to `Send Review Alert Email`.
  - **Edge Cases / Potential Failures:** Google Sheets authentication or schema validation errors.

- **Send Review Alert Email**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` – Dispatches an alert email informing reviewers that manual evaluation is required.
  - **Configuration:** Recipient set to `user@example.com`, HTML body detailing the unresolved query and reason for manual review.
  - **Key Expressions / Variables:** References properties from `Normalize Compliance Data` and `Append Manual Review to Sheets`.
  - **Connections:** Input: `Append Manual Review to Sheets`; Output: None (Terminal node).
  - **Edge Cases / Potential Failures:** Email delivery failures or incorrect recipient configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Title & overview documentation | None | None | ## QA/QC Compliance & Non-Conformance RAG Copilot<br><br>### How it works<br><br>This workflow acts as a QA/QC compliance copilot that receives a chat question, validates it, and uses a RAG agent to retrieve relevant quality documents and produce a structured non-conformance assessment. It normalizes the AI result, decides whether manual review is needed, then routes automated NCR classifications into a decision log. Higher-risk outcomes trigger escalation emails, while uncertain cases are logged separately and sent for manual review.<br><br>### Setup steps<br><br>- Configure the chat trigger so users can submit QA/QC compliance or non-conformance questions.<br>- Add credentials for Google Gemini, GitHub, Google Sheets, and Gmail.<br>- Configure the QA knowledge repository or document retrieval endpoint used by the agent tools.<br>- Map the Google Sheets nodes to the QA decision log used for automated decisions and manual-review cases.<br>- Set Gmail recipients, escalation criteria, and manual review alert recipients before activation.<br><br>### Customization<br><br>Adjust the validation condition, structured output schema, severity thresholds, NCR classification routes, and escalation rules to match the organization’s QA/QC procedures. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Capture and validate query<br><br>Receives the chat request, enriches it with request metadata, and checks whether it is a valid QA/QC query before invoking the RAG process. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Run QA RAG agent<br><br>Uses Gemini with document retrieval, GitHub knowledge-file listing, and structured parsing tools to answer the QA/QC query and extract compliance-focused fields. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Normalize and route outcome<br><br>Standardizes the RAG output into fields such as classification, severity, violated requirement, and recommended action, then branches between manual review and automated NCR classification routing. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Log automated decision<br><br>Records classified QA decisions in Google Sheets and checks whether the logged decision meets escalation criteria. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Send escalation notice<br><br>Sends an escalation email for qualifying QA decisions and updates the sheet to show the escalation was sent. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Handle manual review<br><br>Logs cases requiring human QA review in a separate sheet and sends a manual review alert email to the responsible reviewers. |
| When Chat Message Received | `@n8n/n8n-nodes-langchain.chatTrigger` | Trigger chat interface session | None | Set Query Metadata | |
| Set Query Metadata | `n8n-nodes-base.set` | Enrich payload with metadata & timestamps | When Chat Message Received | If Valid QA Query | |
| If Valid QA Query | `n8n-nodes-base.if` | Validate query existence and minimum length | Set Query Metadata | QA/QC Compliance Agent | |
| QA/QC Compliance Agent | `@n8n/n8n-nodes-langchain.agent` | Core RAG processing agent | If Valid QA Query | Normalize Compliance Data | |
| Google Gemini Chatbot | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM model provider | None | QA/QC Compliance Agent | |
| Fetch QA Knowledge Files | `n8n-nodes-base.githubTool` | GitHub tool to list documentation | None | QA/QC Compliance Agent | |
| Get QA Document | `n8n-nodes-base.httpRequestTool` | HTTP tool to retrieve raw file text | None | QA/QC Compliance Agent | |
| Parse Structured Output | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output schema | None | QA/QC Compliance Agent | |
| Normalize Compliance Data | `n8n-nodes-base.set` | Standardizes output variables | QA/QC Compliance Agent | Check Manual Review Requirement | |
| Check Manual Review Requirement | `n8n-nodes-base.if` | Determines if human review is needed | Normalize Compliance Data | Append Manual Review to Sheets, Route by Classification | |
| Route by Classification | `n8n-nodes-base.switch` | Branches by NCR classification type | Check Manual Review Requirement | Append QA Decision to Sheets (x4) | |
| Append QA Decision to Sheets | `n8n-nodes-base.googleSheets` | Logs audit records to Google Sheets | Route by Classification | Check Escalation Requirement | |
| Check Escalation Requirement | `n8n-nodes-base.if` | Checks if escalation email is required | Append QA Decision to Sheets | Send Escalation Email | |
| Send Escalation Email | `n8n-nodes-base.gmail` | Sends escalation notice via email | Check Escalation Requirement | Update Escalation Status | |
| Update Escalation Status | `n8n-nodes-base.googleSheets` | Updates sheet escalation status to Sent | Send Escalation Email | None | |
| Append Manual Review to Sheets | `n8n-nodes-base.googleSheets` | Logs manual review cases to Sheets | Check Manual Review Requirement | Send Review Alert Email | |
| Send Review Alert Email | `n8n-nodes-base.gmail` | Sends manual review alert via email | Append Manual Review to Sheets | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **When Chat Message Received** node (`@n8n/n8n-nodes-langchain.chatTrigger`). Keep default options.
2. **Add Metadata Set Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Query Metadata`.
   - Connect **When Chat Message Received** to **Set Query Metadata**.
   - Configure assignments: `query` (`={{$json.chatInput}}`), `session_id` (`={{$json.sessionId}}`), `timestamp` (`={{$now.setZone('Asia/Karachi').toISO()}}`), and `source` (`chat`). Enable "Include Other Fields".
3. **Add Query Validation Gate:**
   - Add an **If** node (`n8n-nodes-base.if`) named `If Valid QA Query`.
   - Connect `Set Query Metadata` to `If Valid QA Query`.
   - Set condition (Expression): `={{ !!$json.query && $json.query.trim().length >= 10 }}`.
4. **Configure LLM and AI Tools:**
   - Add a **Google Gemini Chatbot** node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) with model name `models/gemini-3.5-flash-lite`. Set up Google AI credentials.
   - Add a **GitHub Tool** node (`n8n-nodes-base.githubTool`) named `Fetch QA Knowledge Files`. Set repository owner to `fahimjilani`, repository name to `qa-qc-rag-knowledge-base`, path to `docs`, and operation to `list`.
   - Add an **HTTP Request Tool** node (`n8n-nodes-base.httpRequestTool`) named `Get QA Document`. Set the URL expression to `={{ 'https://raw.githubusercontent.com/fahimjilani/qa-qc-rag-knowledge-base/main/docs/' + $fromAI('filename', 'Return only the exact filename from Fetch QA Knowledge Files that is relevant to the user question. Use only one of the approved .txt files returned by that tool. Never use README.md and never invent a filename.') }}`.
   - Add an **Output Parser Structured** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) named `Parse Structured Output`. Provide a JSON schema example containing `classification`, `severity_score`, `requirement_violated`, `recommended_action`, `escalation_required`, `capa_required`, `manual_review_required`, `source_files`, and `reason`.
5. **Create the Core AI Agent:**
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`) named `QA/QC Compliance Agent`.
   - Connect `If Valid QA Query` (True branch) to the agent's main input.
   - Connect **Google Gemini Chatbot** to the agent's AI Language Model input.
   - Connect `Fetch QA Knowledge Files` and `Get QA Document` to the agent's AI Tool inputs.
   - Connect `Parse Structured Output` to the agent's AI Output Parser input.
   - Set prompt type to **Define** with text `={{$json.query}}` and configure the system message restricting the model to approved knowledge-base documents and requiring audit-friendly evidence grounding.
6. **Normalize AI Output:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Normalize Compliance Data`.
   - Connect `QA/QC Compliance Agent` to `Normalize Compliance Data`.
   - Map output fields (`classification`, `severity_score`, `requirement_violated`, `recommended_action`, `escalation_required`, `capa_required`, `manual_review_required`, `source_files`, `reason`) from `={{$json.output...}}` and re-inject query, session ID, and timestamp from `Set Query Metadata`.
7. **Add Manual Review Branching:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Check Manual Review Requirement`.
   - Connect `Normalize Compliance Data` to `Check Manual Review Requirement`.
   - Set condition to evaluate `={{$json.manual_review_required}}`.
8. **Configure Manual Review Path (True Branch):**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append Manual Review to Sheets`. Connect the True output of `Check Manual Review Requirement` here.
   - Set operation to `append`, select document `QA_QC_RAG_Compliance_Log` and sheet `QA_Decision_Log`. Map columns (`Record_ID`, `Timestamp`, `Session_ID`, `Reporter`, `Department`, `Issue_Description`, `Classification`, `Manual_Review_Required`, `Source_Files`, `Reason`, `Escalation_Status`, `Final_Status`) with appropriate expressions.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Send Review Alert Email`. Connect `Append Manual Review to Sheets` to this node. Configure recipient (`user@example.com`), subject, and HTML body. Set up Gmail OAuth2 credentials.
9. **Configure Automated Routing Path (False Branch):**
   - Add a **Switch** node (`n8n-nodes-base.switch`) named `Route by Classification`. Connect the False output of `Check Manual Review Requirement` here.
   - Configure 4 rules matching exact string values: `Compliant`, `Observation`, `Minor Non-Conformance`, and `Major Non-Conformance`.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append QA Decision to Sheets`. Connect all 4 outputs of `Route by Classification` to this node.
   - Set operation to `append`, target document `QA_QC_RAG_Compliance_Log` and sheet `QA_Decision_Log`. Map full schema columns including `Record_ID`, `Severity_Score`, `Requirement_Violated`, `Recommended_Action`, `Escalation_Required`, `CAPA_Required`, `Final_Status`, etc. Set up Google Sheets OAuth2 credentials.
10. **Configure Escalation Handling:**
    - Add an **If** node (`n8n-nodes-base.if`) named `Check Escalation Requirement`. Connect `Append QA Decision to Sheets` here.
    - Set condition to check `={{ $('Normalize Compliance Data').item.json.escalation_required }}`.
    - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Send Escalation Email`. Connect the True output of `Check Escalation Requirement` here. Configure recipient (`user@example.com`), subject, and HTML escalation body.
    - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Update Escalation Status`. Connect `Send Escalation Email` here.
    - Set operation to `update`, select document `QA_QC_RAG_Compliance_Log` and sheet `QA_Decision_Log`. Match by column `Record_ID` and update `Escalation_Status` to `Sent`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GitHub Repository Knowledge Base | [fahimjilani/qa-qc-rag-knowledge-base](https://github.com/fahimjilani/qa-qc-rag-knowledge-base) |
| GitHub Profile Link | [fahimjilani](https://github.com/fahimjilani) |