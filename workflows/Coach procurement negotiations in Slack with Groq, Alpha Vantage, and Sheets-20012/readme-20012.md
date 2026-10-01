Coach procurement negotiations in Slack with Groq, Alpha Vantage, and Sheets

https://n8nworkflows.xyz/workflows/coach-procurement-negotiations-in-slack-with-groq--alpha-vantage--and-sheets-20012


# Coach procurement negotiations in Slack with Groq, Alpha Vantage, and Sheets

### 1. Workflow Overview

This workflow functions as a real-time AI negotiation co-pilot for procurement teams. It connects Slack interactions (slash commands, button clicks, and modal submissions) with external data sources—such as live commodity pricing via Alpha Vantage and historical supplier negotiation logs in Google Sheets. It leverages Groq-hosted LLMs to evaluate leverage, generate tactical scenarios, and produce live counter-arguments mid-call.

The workflow is divided into five logical blocks:

- **1.1 Input Reception & Routing:** Receives incoming webhook events from Slack and routes them to appropriate sub-branches based on whether the payload is a slash command, an interactive button click, or a modal form submission.
- **1.2 Context Aggregation & Profile Analysis:** Fetches live global aluminum prices from Alpha Vantage and historical deal records for a specific supplier from Google Sheets, combines them, and prompts the Groq LLM to extract a structured behavioral and pricing profile.
- **1.3 Scenario Generation & UI Rendering:** Uses the analyzed supplier profile to generate cooperative, moderate, and aggressive pushback scenarios with buyer counter-tactics, formatting the output into an interactive Slack Block Kit message.
- **1.4 Live Interactivity Handling:** Intercepts button interaction events, posts a loading indicator message, and opens a secure Slack modal to capture live verbal pushback from the supplier during an active call.
- **1.5 Counter-Tactic Generation & Session Logging:** Processes the modal input through an LLM using the previously established strategic context to generate a real-time counter-tactic, then logs the entire session outcome into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing
- **Overview:** Captures incoming HTTP POST requests from Slack webhooks and uses a conditional switch node to direct traffic according to the action type (slash command, button interaction, or modal submission).
- **Nodes Involved:** 
  - `Slack Event Listener`
  - `Route Slack Payload`
- **Node Details:**
  - **Slack Event Listener**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`). Listens for incoming POST payloads from Slack.
    - *Configuration Choices:* Method set to `POST`, webhook path set to `slack/negotiate`.
    - *Key Expressions or Variables:* Captures raw request body (`$json.body`).
    - *Input/Output:* Input: External webhook request from Slack. Output: Passes JSON body to routing switch.
    - *Edge Cases / Failures:* Authentication or signature verification issues if Slack endpoint verification fails; timeout if the response takes longer than 3 seconds (Slack requirement).
  - **Route Slack Payload**
    - *Type and Technical Role:* Switch node (`n8n-nodes-base.switch`). Evaluates incoming payload structures to route execution.
    - *Configuration Choices:* Three rules configured:
      1. Route index 0 (Slash Command): Checks if `$json.body.command` exists.
      2. Route index 1 (Interactive Button): Checks if `$json.body.payload` contains `block_actions`.
      3. Route index 2 (Modal Submission): Checks if `$json.body.payload` contains `view_submission`.
    - *Key Expressions or Variables:* Evaluates properties within `$json.body`.
    - *Input/Output:* Input: Output of `Slack Event Listener`. Output: Branch 0 (Fetch price/history), Branch 1 (Extract response URL / Launch modal), Branch 2 (Parse modal payload).
    - *Edge Cases / Failures:* Malformed payloads cause items to fall through to unmatched outputs.

---

#### 2.2 Context Aggregation & Profile Analysis
- **Overview:** Parallelizes fetching of live commodity pricing and historical deal records, merges the data sets, and prompts Groq to analyze the supplier's historical behavior relative to current market conditions.
- **Nodes Involved:**
  - `Fetch Aluminum Price`
  - `Extract Price Data`
  - `Fetch Supplier History`
  - `Format History Data`
  - `Combine Context Sources`
  - `Analyze Supplier Profile`
  - `Groq Model (Profile)`
  - `Parse Profile Output`
  - `Set Profile Variables`
- **Node Details:**
  - **Fetch Aluminum Price**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Queries Alpha Vantage for live aluminum market pricing.
    - *Configuration Choices:* GET request to `https://www.alphavantage.co/query` with query parameters `function=ALUMINUM`, `interval=monthly`, and `apikey`.
    - *Input/Output:* Input: Triggered by Route index 0. Output: Raw market data JSON.
    - *Edge Cases / Failures:* Rate limiting or invalid API key will cause request failure.
  - **Extract Price Data**
    - *Type and Technical Role:* Set (Edit Fields) node (`n8n-nodes-base.set`). Extracts the latest price value from the Alpha Vantage response.
    - *Configuration Choices:* Assigns `latest_aluminum_price` from `={{ $json.data[0].value }}`.
    - *Input/Output:* Input: `Fetch Aluminum Price`. Output: Cleaned price data object.
  - **Fetch Supplier History**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Queries historical negotiation logs.
    - *Configuration Choices:* Operation set to lookup rows; filters by `Supplier_ID` matching `={{ $('Slack Event Listener').item.json.body.text }}`.
    - *Input/Output:* Input: Triggered by Route index 0. Output: Matching spreadsheet rows.
    - *Edge Cases / Failures:* Missing columns or incorrect document IDs cause API errors.
  - **Format History Data**
    - *Type and Technical Role:* Aggregate node (`n8n-nodes-base.aggregate`). Aggregates all items returned from Google Sheets into a single array.
    - *Configuration Choices:* Aggregate mode set to `aggregateAllItemData`.
    - *Input/Output:* Input: `Fetch Supplier History`. Output: Consolidated array of historical deal logs.
  - **Combine Context Sources**
    - *Type and Technical Role:* Merge node (`n8n-nodes-base.merge`). Combines market pricing and historical supplier data into a unified item.
    - *Configuration Choices:* Mode set to `combine` using `combineByPosition`.
    - *Input/Output:* Input: Inputs from `Extract Price Data` and `Format History Data`. Output: Combined dataset.
  - **Analyze Supplier Profile**
    - *Type and Technical Role:* Basic LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`). Sends market and supplier data to Groq for analysis.
    - *Configuration Choices:* Prompt instructs the LLM to compare global aluminum prices with historical logs to determine leverage. Uses structured output parser.
    - *Input/Output:* Input: `Combine Context Sources`. Output: AI-generated supplier profile string.
  - **Groq Model (Profile)**
    - *Type and Technical Role:* Chat Model node (`@n8n/n8n-nodes-langchain.lmChatGroq`). Provides the underlying LLM engine.
    - *Configuration Choices:* Model configured to `openai/gpt-oss-120b`. Requires Groq API credentials.
    - *Input/Output:* Connected to `Analyze Supplier Profile`.
  - **Parse Profile Output**
    - *Type and Technical Role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`). Enforces a strict JSON schema for the profile analysis.
    - *Configuration Choices:* Schema requires `pricing_flexibility_percentage`, `primary_concession_tendency`, `resistance_triggers`, and `recommended_tactic`.
    - *Input/Output:* Connected to `Analyze Supplier Profile`.
  - **Set Profile Variables**
    - *Type and Technical Role:* Set (Edit Fields) node (`n8n-nodes-base.set`). Maps parsed profile fields to individual workflow variables.
    - *Configuration Choices:* Assigns extracted JSON output fields to workflow context variables.
    - *Input/Output:* Input: `Parse Profile Output`. Output: Variables ready for scenario generation.

---

#### 2.3 Scenario Generation & UI Rendering
- **Overview:** Generates three negotiation pushback scenarios via AI based on the supplier profile, then formats and posts an interactive Slack Block Kit card.
- **Nodes Involved:**
  - `Generate Scenarios`
  - `Groq Model (Scenarios)`
  - `Parse Scenario Output`
  - `Set Scenario Variables`
  - `Render Block Kit UI`
- **Node Details:**
  - **Generate Scenarios**
    - *Type and Technical Role:* Basic LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`). Generates reactions and counter-tactics.
    - *Configuration Choices:* Prompts the model to generate cooperative, moderate, and aggressive scenarios given a 12% discount anchor. Uses structured output parsing.
    - *Input/Output:* Input: `Set Profile Variables`. Output: Generated scenarios.
  - **Groq Model (Scenarios)**
    - *Type and Technical Role:* Chat Model node (`@n8n/n8n-nodes-langchain.lmChatGroq`).
    - *Configuration Choices:* Model configured to `openai/gpt-oss-20b`.
  - **Parse Scenario Output**
    - *Type and Technical Role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`). Enforces schema validation for the three scenarios.
  - **Set Scenario Variables**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`). Maps scenario objects to workflow variables.
  - **Render Block Kit UI**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Sends an interactive message card to Slack.
    - *Configuration Choices:* POST request to `response_url` from the initial Slack event. Body contains a full Slack Block Kit JSON structure featuring summary metrics, scenario dividers, and an interactive button (`button_counteroffer`) embedding serialized strategy metadata.

---

#### 2.4 Live Interactivity Handling
- **Overview:** Intercepts Slack button interaction events, acknowledges the action with a loading state message, and opens an interactive modal to capture live feedback.
- **Nodes Involved:**
  - `Extract Response URL`
  - `Post Loading Message`
  - `Launch Slack Modal`
- **Node Details:**
  - **Extract Response URL**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`). Extracts the Slack `response_url` from button payloads.
    - *Configuration Choices:* Assigns `url` to `={{ JSON.parse($json.body.payload).response_url }}`.
  - **Post Loading Message**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Posts a temporary notification to Slack.
    - *Configuration Choices:* POST request containing text `*Drafting live counteroffer...*`.
  - **Launch Slack Modal**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Invokes the Slack API to open a modal view.
    - *Configuration Choices:* POST request to `https://slack.com/api/views.open`. Requires Bot Token Authorization header. Embeds state metadata (supplier ID, strategy, scenarios) into `private_metadata` and includes a multiline text input block (`supplier_response_input`).

---

#### 2.5 Counter-Tactic Generation & Session Logging
- **Overview:** Parses the modal submission containing the supplier's live response, generates an immediate counter-tactic using Groq, and logs the entire session into Google Sheets.
- **Nodes Involved:**
  - `Parse Modal Payload`
  - `Generate Live Tactic`
  - `Groq Model (Tactic)`
  - `Log Negotiation Session`
- **Node Details:**
  - **Parse Modal Payload**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`). Parses the submitted modal JSON payload.
  - **Generate Live Tactic**
    - *Type and Technical Role:* Basic LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`). Evaluates supplier feedback against strategic baseline context.
    - *Configuration Choices:* Prompt extracts strategy and baseline scenarios from `private_metadata` and instructs the model to produce a punchy 1-2 sentence counter-tactic.
  - **Groq Model (Tactic)**
    - *Type and Technical Role:* Chat Model node (`@n8n/n8n-nodes-langchain.lmChatGroq`). Configured to `openai/gpt-oss-120b`.
  - **Log Negotiation Session**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Appends or updates session records.
    - *Configuration Choices:* Operation set to `appendOrUpdate`, matching on `User ID`. Mapped columns include User ID, Timestamp, Session ID, Supplier ID, Supplier Response, Generated AI Tactic, and Recommended Strategy.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Slack Event Listener | `n8n-nodes-base.webhook` | Receives incoming POST requests from Slack | External Webhook | Route Slack Payload | ## Trigger & Traffic Routing<br>Captures incoming Slack payloads and routes traffic based on slash commands, interactive button clicks, or live modal submissions. |
| Route Slack Payload | `n8n-nodes-base.switch` | Routes payload based on command, button click, or modal submission | Slack Event Listener | Fetch Aluminum Price, Fetch Supplier History, Extract Response URL, Parse Modal Payload | ## Trigger & Traffic Routing<br>Captures incoming Slack payloads and routes traffic based on slash commands, interactive button clicks, or live modal submissions. |
| Fetch Aluminum Price | `n8n-nodes-base.httpRequest` | Queries Alpha Vantage for live aluminum prices | Route Slack Payload | Extract Price Data | ## Context Aggregation<br>Fetches live aluminum market prices and retrieves historical supplier negotiation logs from Google Sheets, merging them for AI context. |
| Extract Price Data | `n8n-nodes-base.set` | Extracts price string from API response | Fetch Aluminum Price | Combine Context Sources | ## Context Aggregation<br>Fetches live aluminum market prices and retrieves historical supplier negotiation logs from Google Sheets, merging them for AI context. |
| Fetch Supplier History | `n8n-nodes-base.googleSheets` | Queries historical deal records from Google Sheets | Route Slack Payload | Format History Data | ## Context Aggregation<br>Fetches live aluminum market prices and retrieves historical supplier negotiation logs from Google Sheets, merging them for AI context. |
| Format History Data | `n8n-nodes-base.aggregate` | Aggregates spreadsheet items into a single array | Fetch Supplier History | Combine Context Sources | ## Context Aggregation<br>Fetches live aluminum market prices and retrieves historical supplier negotiation logs from Google Sheets, merging them for AI context. |
| Combine Context Sources | `n8n-nodes-base.merge` | Merges market price and supplier history | Extract Price Data, Format History Data | Analyze Supplier Profile | ## Context Aggregation<br>Fetches live aluminum market prices and retrieves historical supplier negotiation logs from Google Sheets, merging them for AI context. |
| Analyze Supplier Profile | `@n8n/n8n-nodes-langchain.chainLlm` | Analyzes combined data to extract supplier profile via Groq | Combine Context Sources | Set Profile Variables | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Groq Model (Profile) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides LLM engine (gpt-oss-120b) for profile analysis | None (AI Model) | Analyze Supplier Profile | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Parse Profile Output | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema on profile analysis output | None (Output Parser) | Analyze Supplier Profile | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Set Profile Variables | `n8n-nodes-base.set` | Sets profile attributes into workflow variables | Parse Profile Output | Generate Scenarios | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Generate Scenarios | `@n8n/n8n-nodes-langchain.chainLlm` | Generates 3 pushback scenarios using Groq | Set Profile Variables | Set Scenario Variables | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Groq Model (Scenarios) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides LLM engine (gpt-oss-20b) for scenarios | None (AI Model) | Generate Scenarios | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Parse Scenario Output | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema on scenario output | None (Output Parser) | Generate Scenarios | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Set Scenario Variables | `n8n-nodes-base.set` | Maps scenario outputs to workflow variables | Parse Scenario Output | Render Block Kit UI | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Render Block Kit UI | `n8n-nodes-base.httpRequest` | Posts formatted Slack Block Kit report and interactive button | Set Scenario Variables | None (Terminal Response) | ## AI Strategy & UI Render<br>Analyzes supplier data to formulate a primary negotiation strategy, generates three pushback scenarios, and renders the interactive Slack interface |
| Extract Response URL | `n8n-nodes-base.set` | Extracts response URL from button action payload | Route Slack Payload | Post Loading Message, Launch Slack Modal | ## Live Interactivity Trigger<br>Acknowledges the user's button click with a loading message and simultaneously pops open the hidden data-rich Slack modal. |
| Post Loading Message | `n8n-nodes-base.httpRequest` | Posts loading indicator message to Slack | Extract Response URL | None | ## Live Interactivity Trigger<br>Acknowledges the user's button click with a loading message and simultaneously pops open the hidden data-rich Slack modal. |
| Launch Slack Modal | `n8n-nodes-base.httpRequest` | Opens Slack interactive modal view with embedded metadata | Extract Response URL | None | ## Live Interactivity Trigger<br>Acknowledges the user's button click with a loading message and simultaneously pops open the hidden data-rich Slack modal. |
| Parse Modal Payload | `n8n-nodes-base.set` | Parses submitted modal form payload | Route Slack Payload | Generate Live Tactic | ## Counter-Tactic & Logging<br>Parses the user's live modal input, generates a real-time counter-strategy via AI, posts the tactic, and logs the session. |
| Generate Live Tactic | `@n8n/n8n-nodes-langchain.chainLlm` | Generates real-time counter-tactic using Groq based on supplier response | Parse Modal Payload | Log Negotiation Session | ## Counter-Tactic & Logging<br>Parses the user's live modal input, generates a real-time counter-strategy via AI, posts the tactic, and logs the session. |
| Groq Model (Tactic) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides LLM engine (gpt-oss-120b) for live tactics | None (AI Model) | Generate Live Tactic | ## Counter-Tactic & Logging<br>Parses the user's live modal input, generates a real-time counter-strategy via AI, posts the tactic, and logs the session. |
| Log Negotiation Session | `n8n-nodes-base.googleSheets` | Appends or updates session log data in Google Sheets | Generate Live Tactic | None (Terminal Action) | ## Counter-Tactic & Logging<br>Parses the user's live modal input, generates a real-time counter-strategy via AI, posts the tactic, and logs the session. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Entry Point:**
   - Add a **Webhook** node named `Slack Event Listener`. Set HTTP Method to `POST` and path to `slack/negotiate`.
2. **Add Routing Logic:**
   - Connect a **Switch** node named `Route Slack Payload` to the webhook. Configure three rules:
     - Rule 0: Checks if `{{ $json.body.command }}` exists.
     - Rule 1: Checks if `{{ $json.body.payload }}` contains `block_actions`.
     - Rule 2: Checks if `{{ $json.body.payload }}` contains `view_submission`.
3. **Build Context Aggregation Branch (Rule 0 Output 1):**
   - **Fetch Aluminum Price:** Add an **HTTP Request** node (`GET` to `https://www.alphavantage.co/query` with query parameters `function=ALUMINUM`, `interval=monthly`, and `apikey`).
   - **Extract Price Data:** Add a **Set** node to assign `latest_aluminum_price` = `={{ $json.data[0].value }}`.
   - **Fetch Supplier History:** Add a **Google Sheets** node (Operation: Look Up Rows). Set Document ID and Sheet Name (`Sheet1`). Configure filter where `Supplier_ID` matches `={{ $('Slack Event Listener').item.json.body.text }}`.
   - **Format History Data:** Add an **Aggregate** node with aggregation mode set to `aggregateAllItemData`.
   - **Combine Context Sources:** Add a **Merge** node set to combine by position, taking inputs from `Extract Price Data` (input 0) and `Format History Data` (input 1).
4. **Build AI Profile Analysis Sub-Branch:**
   - **Analyze Supplier Profile:** Add a **Basic LLM Chain** node. Provide prompt comparing global aluminum prices to historical logs. Connect an **Output Parser Structured** node enforcing schema for `pricing_flexibility_percentage`, `primary_concession_tendency`, `resistance_triggers`, and `recommended_tactic`.
   - **Groq Model (Profile):** Add a **Chat Model Groq** node configured to `openai/gpt-oss-120b` linked to the chain's AI model input. Configure Groq API credentials.
   - **Set Profile Variables:** Add a **Set** node to map parsed JSON outputs to workflow variables.
5. **Build Scenario Generation & Rendering Sub-Branch:**
   - **Generate Scenarios:** Add a **Basic LLM Chain** node with prompt to generate Cooperative, Moderate, and Aggressive pushback scenarios. Connect a structured output parser enforcing schema with nested reaction and counter-tactic objects.
   - **Groq Model (Scenarios):** Add a **Chat Model Groq** node configured to `openai/gpt-oss-20b` linked to the chain.
   - **Set Scenario Variables:** Add a **Set** node to map scenario outputs.
   - **Render Block Kit UI:** Add an **HTTP Request** node (`POST` to `={{ $('Slack Event Listener').item.json.body.response_url }}`) sending a JSON body containing Slack Block Kit elements, including an interactive button (`button_counteroffer`) with serialized metadata.
6. **Build Live Interactivity Branch (Rule 1 Output 2):**
   - **Extract Response URL:** Add a **Set** node to assign `url` = `={{ JSON.parse($json.body.payload).response_url }}`.
   - **Post Loading Message:** Add an **HTTP Request** node (`POST` to `={{ $json.url }}`) sending `{ "replace_original": false, "text": " *Drafting live counteroffer...*" }`.
   - **Launch Slack Modal:** Add an **HTTP Request** node (`POST` to `https://slack.com/api/views.open`). Set `Authorization` header to your Slack OAuth token. Send JSON body opening a modal view (`counteroffer_modal`) with `private_metadata` containing channel ID and strategy context, and a text input block (`supplier_response_input`).
7. **Build Modal Submission & Logging Branch (Rule 2 Output 3):**
   - **Parse Modal Payload:** Add a **Set** node to assign `modal_data` = `={{ JSON.parse($json.body.payload) }}`.
   - **Generate Live Tactic:** Add a **Basic LLM Chain** node. Configure prompt to read supplier response from modal input and output a 1-2 sentence counter-tactic based on stored strategy context.
   - **Groq Model (Tactic):** Add a **Chat Model Groq** node configured to `openai/gpt-oss-120b` linked to the chain.
   - **Log Negotiation Session:** Add a **Google Sheets** node (Operation: `appendOrUpdate`, matching on `User ID`). Set Document ID and Sheet Name (`Logs`). Map columns: `User ID`, `Timestamp`, `Session ID`, `Supplier ID`, `Supplier Response`, `Generated AI Tactic`, and `Recommended Strategy`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Need help setting up custom Slack integrations, tuning Groq AI prompts, or configuring CRM add-ons? | [Contact WeblineIndia](https://www.weblineindia.com/contact-us.html) |