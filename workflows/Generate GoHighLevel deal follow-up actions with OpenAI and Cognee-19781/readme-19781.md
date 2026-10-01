Generate GoHighLevel deal follow-up actions with OpenAI and Cognee

https://n8nworkflows.xyz/workflows/generate-gohighlevel-deal-follow-up-actions-with-openai-and-cognee-19781


# Generate GoHighLevel deal follow-up actions with OpenAI and Cognee

### 1. Workflow Overview

This workflow automates the follow-up process whenever a new sales opportunity (deal) is opened in GoHighLevel (GHL). It extracts opportunity metadata, retrieves human-readable pipeline and stage designations, and passes the context to an AI agent powered by OpenAI GPT-4o-mini. The agent queries a Cognee knowledge graph to pull relevant company SOPs, sales playbooks, or past deal histories, and then compiles a customized 3–6 step follow-up checklist. Finally, the workflow dispatches this checklist in parallel to Slack, Gmail, and a Google Sheets audit log, while providing centralized error handling across all execution branches.

The system logic is divided into the following functional blocks:

- **1.1 Input Reception & Data Intake:** Gathers incoming opportunity triggers (webhook, daily schedule, or manual run), pulls global pipeline metadata from the GoHighLevel API, and queries open leads.
- **1.2 Data Normalization & Merging:** Combines raw opportunity payloads with pipeline definitions and flattens them into clean, standardized JSON properties.
- **1.3 AI Processing & Knowledge Retrieval:** Utilizes an OpenAI agent backed by a Cognee tool (`Search Company Brain`) to query internal documentation and generate deal-specific action items.
- **1.4 Multi-Channel Output & Distribution:** Parses agent results into formatted payloads and sends notifications to Slack, drafts and sends an email via Gmail, and appends a structured entry to Google Sheets.
- **1.5 Error Handling & Alerting:** Captures exceptions across primary nodes and formats localized alerts for delivery to a dedicated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Intake
- **Overview:** Serves as the multi-source entry point for the pipeline, fetching raw open opportunities alongside pipeline metadata from GoHighLevel.
- **Nodes Involved:** 
  - `GHL Deal Opened (Webhook)`
  - `Schedule Trigger`
  - `Manual Trigger`
  - `Get GHL Pipelines`
  - `Fetch Leads from GHL`
- **Node Details:**
  - **`GHL Deal Opened (Webhook)`**
    - *Type and Role:* `n8n-nodes-base.webhook` — Triggers the execution when an external HTTP POST request is received from GoHighLevel.
    - *Configuration Choices:* Configured to listen on path `b8c7672e-63c9-46fd-9a77-275cdc9cfb80` using HTTP method `POST`.
    - *Input/Output:* No inputs; outputs to `Get GHL Pipelines` and `Merge Deal + Pipelines`.
    - *Edge Cases:* Unauthenticated external triggers if GHL payload security isn't enforced upstream.
  - **`Schedule Trigger`**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` — Periodic trigger for periodic batch checks of newly opened leads.
    - *Configuration Choices:* Default recurring interval settings.
    - *Input/Output:* No inputs; outputs to `Fetch Leads from GHL`.
  - **`Manual Trigger`**
    - *Type and Role:* `n8n-nodes-base.manualTrigger` — On-demand testing entry point.
    - *Configuration Choices:* Standard click-to-run setup.
    - *Input/Output:* No inputs; outputs to `Fetch Leads from GHL`.
  - **`Get GHL Pipelines`**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Fetches metadata for all sales pipelines and stages.
    - *Configuration Choices:* GET request to `https://services.leadconnectorhq.com/opportunities/pipelines` with HTTP header `Version: v3` and Bearer token authorization. Requires a `locationId` query parameter.
    - *Input/Output:* Receives triggers from Webhook; outputs primary data to `Merge Deal + Pipelines` (Input 1) and error handling to `Format Error Message`.
    - *Edge Cases:* API authentication errors (`401`), rate limits, or missing `locationId` values.
  - **`Fetch Leads from GHL`**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Retrieves open leads/opportunities from the GHL system.
    - *Configuration Choices:* GET request to `https://services.leadconnectorhq.com/opportunities/search` filtering by `status=open`. Requires `Version: v3` and Bearer token auth along with `locationId`.
    - *Input/Output:* Triggered by Schedule or Manual triggers; outputs to `Get GHL Pipelines`, `Merge Deal + Pipelines` (Input 0), and `Format Error Message`.
    - *Edge Cases:* Network timeouts or malformed response arrays when zero open leads exist.

#### 2.2 Data Normalization & Merging
- **Overview:** Merges pipeline stage definitions with raw opportunity payloads and transforms nested structures into flat, accessible variables.
- **Nodes Involved:**
  - `Merge Deal + Pipelines`
  - `Parse Deal Data`
- **Node Details:**
  - **`Merge Deal + Pipelines`**
    - *Type and Role:* `n8n-nodes-base.merge` — Combines opportunity objects and pipeline definitions into a unified dataset.
    - *Configuration Choices:* Mode set to `combine` using `combineByPosition`.
    - *Input/Output:* Input 0 receives data from `Fetch Leads from GHL`; Input 1 receives data from `Get GHL Pipelines`. Outputs to `Parse Deal Data`.
  - **`Parse Deal Data`**
    - *Type and Role:* `n8n-nodes-base.code` — JavaScript execution node that cleans up disparate payload shapes and correlates ID fields with their human-readable stage and pipeline titles.
    - *Configuration Choices:* Handles extraction from arrays (`opportunities`), singular objects (`opportunity`), or flat records. Falls back safely to default placeholder values (`'Unnamed Deal'`, `'Unknown Pipeline'`).
    - *Key Expressions/Variables:* `$json.opportunities?.[0] ?? $json.opportunity ?? $json`.
    - *Input/Output:* Input from `Merge Deal + Pipelines`; outputs structured flat deal objects to `Follow-up Actions Agent`.

#### 2.3 AI Processing & Knowledge Retrieval
- **Overview:** Leverages an LLM agent supplemented with a RAG vector tool to query company SOPs and generate a targeted follow-up checklist.
- **Nodes Involved:**
  - `Follow-up Actions Agent`
  - `OpenAI Chat Model`
  - `Search Company Brain (Cognee)`
- **Node Details:**
  - **`Follow-up Actions Agent`**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` — LangChain agent orchestrating reasoning loops between the LLM and external tools.
    - *Configuration Choices:* System prompt enforces querying company playbooks first, relying on general best practices only if search yields no results, and restricting output strictly to a bullet list format starting with `- `.
    - *Input/Output:* Connected to `Parse Deal Data` for prompt text generation; links to `OpenAI Chat Model` (AI Language Model) and `Search Company Brain (Cognee)` (AI Tool). Outputs to `Parse Agent Output` and `Format Error Message`.
  - **`OpenAI Chat Model`**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the foundational language intelligence layer.
    - *Configuration Choices:* Model selected is `gpt-4o-mini`.
    - *Input/Output:* Connects directly as an AI Language Model parameter to `Follow-up Actions Agent`.
  - **`Search Company Brain (Cognee)`**
    - *Type and Role:* `@n8n/n8n-nodes-cognee.cogneeTool` — Knowledge graph RAG lookup utility.
    - *Configuration Choices:* Queries dataset `cognee_2_workflow_dats_et_` with `topK` set to 5. Uses dynamic queries derived via `$fromAI()`.
    - *Input/Output:* Connected as an AI tool to `Follow-up Actions Agent`.
    - *Edge Cases:* Cognee dataset unavailability or empty search response contexts.

#### 2.4 Multi-Channel Output & Distribution
- **Overview:** Converts raw LLM output into clean UI strings and broadcasts the resulting actions across Slack, Gmail, and Google Sheets simultaneously.
- **Nodes Involved:**
  - `Parse Agent Output`
  - `Post Follow-up to Slack`
  - `Gmail - Send Follow-up Email`
  - `Log Deal to Google Sheet`
- **Node Details:**
  - **`Parse Agent Output`**
    - *Type and Role:* `n8n-nodes-base.code` — JavaScript code node parsing bullet points from LLM text and formatting currency values.
    - *Key Expressions/Variables:* Uses `Intl.NumberFormat` for currency parsing, builds rich `slackText` templates, and formats HTML strings for emails.
    - *Input/Output:* Input from `Follow-up Actions Agent`; outputs parsed properties to Slack, Gmail, and Google Sheets nodes.
  - **`Post Follow-up to Slack`**
    - *Type and Role:* `n8n-nodes-base.slack` — Posts notifications to a designated sales channel.
    - *Configuration Choices:* Selects channel type, referencing channel ID `C0AN1UGL0RM` (`your-sales-alerts-channel`). Uses expression `{{ $json.slackText }}`.
    - *Input/Output:* Input from `Parse Agent Output`; error flow connects to `Format Error Message`.
  - **`Gmail - Send Follow-up Email`**
    - *Type and Role:* `n8n-nodes-base.gmail` — Sends the HTML summary email via OAuth2.
    - *Configuration Choices:* Recipient set to `user@example.com` (to be updated), using `{{ $json.emailSubject }}` and `{{ $json.emailBody }}`.
    - *Input/Output:* Input from `Parse Agent Output`; error flow connects to `Format Error Message`.
  - **`Log Deal to Google Sheet`**
    - *Type and Role:* `n8n-nodes-base.googleSheets` — Appends deal metrics and action items into a tabular log.
    - *Configuration Choices:* Operation set to `append`. Document ID: `1r85OvP38XCr-WbFgiFs01sC-zy5jblCgkmJItA2DgZM`, Sheet: `Deal Follow-ups` (ID: `430066971`). Maps schema fields (`Timestamp`, `Deal Name`, `Deal ID`, `Pipeline`, `Stage`, `Value`, `Contact`, `Contact Email`, `Owner`, `Follow-up Actions`).
    - *Input/Output:* Input from `Parse Agent Output`; error flow connects to `Format Error Message`.

#### 2.5 Error Handling & Alerting
- **Overview:** Intercepts failures from all upstream nodes, standardizes error metadata, and publishes alerts to Slack.
- **Nodes Involved:**
  - `Format Error Message`
  - `Notify Error in Slack`
- **Node Details:**
  - **`Format Error Message`**
    - *Type and Role:* `n8n-nodes-base.set` — Normalizes exception payloads into clean variables.
    - *Key Expressions/Variables:* Extracts properties using fallback logic: `{{ $json.error?.message ?? $json.execution?.error?.message ?? $json.message ?? 'Unknown error' }}`. Captures workflow name and execution timestamps via `$workflow.name` and `$now.toISO()`.
    - *Input/Output:* Receives error output flags from HTTP, Agent, Sheets, Slack, and Gmail nodes; outputs to `Notify Error in Slack`.
  - **`Notify Error in Slack`**
    - *Type and Role:* `n8n-nodes-base.slack` — Sends operational failure alerts to the sales monitoring channel.
    - *Configuration Choices:* Channel ID `C0AN1UGL0RM`, formatting failure details with node names and error text.
    - *Input/Output:* Input from `Format Error Message`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `📌 Overview – How This Workflow Works` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🤝 Deal Opened – AI Follow-up Actions Agent (Cognee RAG)<br><br>### How it works<br>This workflow fires automatically when a new deal is opened in GoHighLevel — via a webhook trigger, a daily schedule, or a manual run. It fetches the deal and pipeline details from GHL, parses them into a structured object, and passes everything to an AI Agent powered by GPT-4o-mini and a Cognee tool. The agent searches your company's knowledge base (past deals, client context, sales playbooks) to generate a tailored set of follow-up actions specific to this deal. The output is parsed and dispatched in parallel: a Slack message goes to the sales channel, a follow-up email is drafted and sent via Gmail, and the deal + action items are logged to a Google Sheet. Any errors are caught, formatted, and posted to Slack for visibility.<br><br>### Setup steps<br>1. Replace `YOUR_GHL_API_TOKEN` in `Get GHL Pipelines` and `Fetch Leads from GHL`.<br>2. Connect **Google Sheets OAuth2** to `Log Deal to Google Sheet` — replace the sheet ID.<br>3. Connect **OpenAI API** to `OpenAI Chat Model`.<br>4. Connect **Cognee API** to `Search Company Brain (Cognee)` — replace with your Cognee account.<br>5. Connect **Slack API** to both Slack nodes — replace `YOUR_SLACK_CHANNEL_ID`.<br>6. Connect **Gmail OAuth2** to `Gmail - Send Follow-up Email` — replace `sales@yourcompany.com`.<br>7. Set the Webhook URL as the trigger in your GHL workflow.<br>8. Activate all three triggers. |
| `Section – Triple-Trigger Deal Intake` | `n8n-nodes-base.stickyNote` | Intake documentation grouping | None | None | ## 🎯 Triple-Trigger Deal Intake<br>Three entry points feed the same pipeline: a GHL webhook fires on every new deal, a daily schedule sweeps deals opened in the last 24 hours, and a manual trigger for on-demand runs. All three converge at `Get GHL Pipelines` and `Merge Deal + Pipelines`. |
| `Section – AI Follow-up Agent with Cognee RAG` | `n8n-nodes-base.stickyNote` | AI block documentation grouping | None | None | ## 🧠 AI Follow-up Agent with Cognee RAG<br>The parsed deal data — name, pipeline, stage, contact, value — is passed to a GPT-4o-mini agent with access to the `Search Company Brain` Cognee tool. The agent searches your company knowledge base to generate context-aware, deal-specific follow-up action items. |
| `Section – Multi-Channel Output` | `n8n-nodes-base.stickyNote` | Output block documentation grouping | None | None | ## 📤 Multi-Channel Output<br>Once the agent responds, a code node parses the output into structured Slack text, email body, and sheet rows. All three fire in parallel: a Slack message to the sales channel, a follow-up email via Gmail, and a new row appended to the Deal Follow-ups Google Sheet. |
| `Section – Error Handling` | `n8n-nodes-base.stickyNote` | Error block documentation grouping | None | None | ## ⚠️ Error Handling<br>Every node in the pipeline has an error path routed to `Format Error Message`. This Set node captures the workflow name, failed node, error message, and timestamp, then posts a structured Slack alert so no failure goes unnoticed. |
| `🔑 Credentials & Configuration` | `n8n-nodes-base.stickyNote` | Credentials checklist | None | None | ## 🔑 Credentials Required<br>- **GHL Private Integration Token** → `YOUR_GHL_API_TOKEN`<br>- **Google Sheets OAuth2** — deal log sheet<br>- **OpenAI API** — GPT-4o-mini agent<br>- **Cognee API** — company brain search<br>- **Slack API** — follow-up + error alerts<br>- **Gmail OAuth2** — follow-up email send |
| `Get GHL Pipelines` | `n8n-nodes-base.httpRequest` | Fetch pipeline metadata | `GHL Deal Opened (Webhook)` | `Merge Deal + Pipelines`, `Format Error Message` | |
| `Merge Deal + Pipelines` | `n8n-nodes-base.merge` | Combine opportunity and pipeline lists | `Get GHL Pipelines`, `Fetch Leads from GHL` | `Parse Deal Data` | |
| `GHL Deal Opened (Webhook)` | `n8n-nodes-base.webhook` | Webhook entry point | None | `Get GHL Pipelines`, `Merge Deal + Pipelines` | |
| `Parse Deal Data` | `n8n-nodes-base.code` | Normalize raw deal payloads | `Merge Deal + Pipelines` | `Follow-up Actions Agent` | |
| `Parse Agent Output` | `n8n-nodes-base.code` | Format bullet list for channels | `Follow-up Actions Agent` | `Post Follow-up to Slack`, `Log Deal to Google Sheet`, `Gmail - Send Follow-up Email` | |
| `Log Deal to Google Sheet` | `n8n-nodes-base.googleSheets` | Append audit log row | `Parse Agent Output` | `Format Error Message` | |
| `Follow-up Actions Agent` | `@n8n/n8n-nodes-langchain.agent` | Generate deal checklist via LLM | `Parse Deal Data` | `Parse Agent Output`, `Format Error Message` | |
| `OpenAI Chat Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provide LLM engine | None | `Follow-up Actions Agent` | |
| `Post Follow-up to Slack` | `n8n-nodes-base.slack` | Send deal notification to Slack | `Parse Agent Output` | `Format Error Message` | |
| `Format Error Message` | `n8n-nodes-base.set` | Standardize failure context | `Get GHL Pipelines`, `Fetch Leads from GHL`, `Follow-up Actions Agent`, `Post Follow-up to Slack`, `Log Deal to Google Sheet`, `Gmail - Send Follow-up Email` | `Notify Error in Slack` | |
| `Notify Error in Slack` | `n8n-nodes-base.slack` | Post error alerts | `Format Error Message` | None | |
| `Schedule Trigger` | `n8n-nodes-base.scheduleTrigger` | Periodic execution trigger | None | `Fetch Leads from GHL` | |
| `Gmail - Send Follow-up Email` | `n8n-nodes-base.gmail` | Send follow-up HTML email | `Parse Agent Output` | `Format Error Message` | |
| `Fetch Leads from GHL` | `n8n-nodes-base.httpRequest` | Fetch open opportunities | `Schedule Trigger`, `Manual Trigger` | `Get GHL Pipelines`, `Merge Deal + Pipelines`, `Format Error Message` | |
| `Search Company Brain (Cognee)` | `n8n-nodes-cognee.cogneeTool` | RAG search tool for AI agent | None | `Follow-up Actions Agent` | |
| `Manual Trigger` | `n8n-nodes-base.manualTrigger` | On-demand test execution trigger | None | `Fetch Leads from GHL` | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Entry Point Triggers
1. **Webhook Trigger:** Add a **Webhook** node named `GHL Deal Opened (Webhook)`. Set the HTTP Method to `POST`.
2. **Schedule Trigger:** Add a **Schedule Trigger** named `Schedule Trigger`. Keep default interval settings.
3. **Manual Trigger:** Add a **Manual Trigger** node named `Manual Trigger`.

#### Step 2: Set Up GoHighLevel API Data Ingestion
1. **Fetch Leads:** Create an **HTTP Request** node named `Fetch Leads from GHL`. 
   - Method: `GET`
   - URL: `https://services.leadconnectorhq.com/opportunities/search`
   - Query Parameters: `status` = `open`, `getTasks` = `false`, `getNotes` = `false`, `getCalendarEvents` = `false`, and add a parameter named `locationId`.
   - Headers: `Accept` (`application/json`), `Version` (`v3`), `Authorization` (`Bearer YOUR_TOKEN_HERE`).
   - Enable `Continue On Fail` (`onError: continueErrorOutput`).
   - Connect both **Schedule Trigger** and **Manual Trigger** outputs to this node.
2. **Fetch Pipelines:** Create an **HTTP Request** node named `Get GHL Pipelines`.
   - Method: `GET`
   - URL: `https://services.leadconnectorhq.com/opportunities/pipelines`
   - Query Parameters: Add a parameter named `locationId`.
   - Headers: `Accept` (`application/json`), `Version` (`v3`), `Authorization` (`Bearer YOUR_TOKEN_HERE`).
   - Enable `Continue On Fail`.
   - Connect outputs from `GHL Deal Opened (Webhook)` to this node.

#### Step 3: Combine and Normalize Data
1. **Merge Node:** Add a **Merge** node named `Merge Deal + Pipelines`. Set mode to `combine` and combination method to `combineByPosition`.
   - Connect Input 0 (`main`) from `Fetch Leads from GHL`.
   - Connect Input 1 (`main`) from `Get GHL Pipelines`.
2. **Code Node (Normalization):** Add a **Code** node named `Parse Deal Data`. Paste JavaScript code to parse opportunities, match pipeline names with stage definitions, and handle fallbacks for monetary values and assignees.
   - Connect input from `Merge Deal + Pipelines`.

#### Step 4: Configure AI Agent & RAG Tools
1. **Chat Model:** Add an **OpenAI Chat Model** node (`OpenAI Chat Model`). Select model `gpt-4o-mini`.
2. **Cognee Tool:** Add a **Cognee Tool** node (`Search Company Brain (Cognee)`). Set Dataset to `cognee_2_workflow_dats_et_`, Top K to `5`, and Resource to `search`.
3. **AI Agent:** Add an **Advanced AI Agent** node named `Follow-up Actions Agent`.
   - Prompt Type: Define.
   - Prompt Text: Pass deal metadata variables (`dealName`, `pipelineName`, `stageName`, etc.) with instructions to search the Company Brain.
   - System Message: Direct the agent to query the brain first, avoid hallucinating company policy, and output solely a 3–6 item bullet list starting with `- `.
   - Enable `Continue On Fail`.
   - Connect `OpenAI Chat Model` to the agent's `ai_languageModel` input.
   - Connect `Search Company Brain (Cognee)` to the agent's `ai_tool` input.
   - Connect input from `Parse Deal Data`.

#### Step 5: Format and Distribute Outputs
1. **Parse Agent Output:** Add a **Code** node named `Parse Agent Output`. Paste JavaScript to convert the raw bullet list into structured HTML, plain text, and currency objects.
   - Connect input from `Follow-up Actions Agent`.
2. **Slack Notification:** Add a **Slack** node named `Post Follow-up to Slack`. Set resource to `channel`, select channel ID `C0AN1UGL0RM`, and reference expression `{{ $json.slackText }}`. Enable `Continue On Fail`.
3. **Google Sheets Logging:** Add a **Google Sheets** node named `Log Deal to Google Sheet`. 
   - Operation: `append`.
   - Document ID: `1r85OvP38XCr-WbFgiFs01sC-zy5jblCgkmJItA2DgZM`.
   - Sheet: `Deal Follow-ups` (ID: `430066971`).
   - Map columns to respective item properties (`Timestamp`, `Deal Name`, `Deal ID`, `Pipeline`, `Stage`, `Value`, `Contact`, `Contact Email`, `Owner`, `Follow-up Actions`).
   - Enable `Continue On Fail`.
4. **Gmail Dispatch:** Add a **Gmail** node named `Gmail - Send Follow-up Email`.
   - Set recipient to `user@example.com`, subject to `{{ $json.emailSubject }}`, and message body to `{{ $json.emailBody }}`.
   - Enable `Continue On Fail`.
   - Connect all three output nodes (`Post Follow-up to Slack`, `Log Deal to Google Sheet`, `Gmail - Send Follow-up Email`) to receive parallel main data feeds from `Parse Agent Output`.

#### Step 6: Configure Centralized Error Handling
1. **Set Error Context:** Add a **Set** node named `Format Error Message`.
   - Assign expressions capturing error messages, failed node names, workflow name ($workflow.name), and timestamps ($now.toISO()).
2. **Slack Error Alert:** Add a **Slack** node named `Notify Error in Slack`.
   - Set channel ID to `C0AN1UGL0RM` and format a structured error alert message template.
   - Connect input from `Format Error Message`.
3. **Route Errors:** Connect the error output (`error` path) of all fallible nodes (`Get GHL Pipelines`, `Fetch Leads from GHL`, `Follow-up Actions Agent`, `Post Follow-up to Slack`, `Log Deal to Google Sheet`, `Gmail - Send Follow-up Email`) into `Format Error Message`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GoHighLevel API Version & Tokens | Ensure GHL private integration tokens have proper permission scopes for opportunity and pipeline reads, utilizing API version `v3`. |
| Cognee RAG Dataset Configuration | Verify that your Cognee knowledge store vector database contains the target dataset ID (`cognee_2_workflow_dats_et_`) populated with relevant sales playbooks prior to activation. |
| Notification Channel Routing | Update Slack channel ID references (`C0AN1UGL0RM`) to match production sales alerting channels across both notification and error handler nodes. |