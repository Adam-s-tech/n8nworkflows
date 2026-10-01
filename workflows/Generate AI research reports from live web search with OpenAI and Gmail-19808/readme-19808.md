Generate AI research reports from live web search with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-ai-research-reports-from-live-web-search-with-openai-and-gmail-19808


# Generate AI research reports from live web search with OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the generation of professional, structured, and sourced research reports using live web search data, AI-driven analysis, and AI-driven drafting. It collects a research inquiry via an interactive form, queries Google Search via the Serper API across multiple angles, compiles and deduplicates findings, extracts structured research intelligence through an AI agent, drafts a comprehensive multi-section report using a second AI agent, logs metadata into Google Sheets, and finally delivers the complete report via email.

The logic is divided into four functional blocks:
- **1.1 Input Reception & Configuration:** Collects parameters via an internal n8n form trigger and initializes core configuration constants (API keys, target emails, document identifiers).
- **1.2 Multi-Angle Web Data Gathering:** Executes parallel or sequential queries to Google via the Serper API (main topic query and expert/data angle query), combining and deduplicating results into a structured reference text block.
- **1.3 Two-Stage AI Intelligence & Drafting:** Utilizes an Advanced AI Agent pipeline where the first agent extracts structured analytical intelligence (findings, gaps, confidence scores) and the second agent drafts the final report sections.
- **1.4 Logging & Delivery:** Formats the compiled assets into an email body, records execution metrics to a Google Sheets database, and triggers the final email dispatch.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** This block captures the user’s research parameters via a web form and standardizes variables (such as API keys, email addresses, and sheet parameters) for downstream operations.
- **Nodes Involved:** 
  - `Research request form`
  - `Set config`

- **Node Details:**
  - **Research request form**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to generate a standalone web form titled "AI Research Report Request". Collects five fields: Research Topic (required), Target Audience (required), Report Goal (required), Specific Angle or Focus (optional), and Your Name (optional).
    - *Key Expressions or Variables:* Captures form values via standard n8n input handling (`$json['Research Topic']`, etc.).
    - *Input and Output Connections:* Input: None (Trigger). Output: Connects to `Set config`.
    - *Version-Specific Requirements:* TypeVersion 2.2.
    - *Edge Cases / Potential Failure Types:* Incomplete submissions if required fields are left blank by the user.
  - **Set config**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Assignment)
    - *Configuration Choices:* Sets static placeholders and expression mappings for Serper API keys, recipient addresses, sender names, Google Sheet IDs, sheet names, and pre-formatted query strings (`mainQuery`, `expertQuery`, and date variables).
    - *Key Expressions or Variables:* 
      - `serperApiKey`: `"YOUR_SERPER_API_KEY"`
      - `recipientEmail`: `"YOUR_EMAIL_ADDRESS"`
      - `senderName`: `"YOUR_NAME"`
      - `sheetId`: `"YOUR_GOOGLE_SHEET_ID"`
      - `mainQuery`: `={{ $json['Research Topic'] }}`
      - `expertQuery`: `={{ $json['Research Topic'] + ' expert insights data research 2026' }}`
      - `today`: `={{ $now.toFormat('dd MMM yyyy') }}`
    - *Input and Output Connections:* Input: `Research request form`. Output: Connects to `Search the main topic`.
    - *Version-Specific Requirements:* TypeVersion 3.4.
    - *Edge Cases / Potential Failure Types:* Missing or invalid credential placeholders causing downstream API errors.

---

#### Block 1.2: Multi-Angle Web Data Gathering
- **Overview:** This block fetches up to 15 unique, deduplicated search results from Google using the Serper API by combining a general query and an expert/data-focused query.
- **Nodes Involved:**
  - `Search the main topic`
  - `Search the expert angle`
  - `Combine the sources`

- **Node Details:**
  - **Search the main topic**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Sends a POST request to `https://google.serper.dev/search` requesting 10 organic results (`num: 10`) for the US region (`gl: "us"`, `hl: "en"`).
    - *Key Expressions or Variables:* Uses `={{ $json.mainQuery }}` for the search query body and `={{ $json.serperApiKey }}` for authorization headers. Configured with `onError: "continueRegularOutput"` and `alwaysOutputData: true`.
    - *Input and Output Connections:* Input: `Set config`. Output: Connects to `Search the expert angle`.
    - *Version-Specific Requirements:* TypeVersion 4.2.
    - *Edge Cases / Potential Failure Types:* Network timeout, invalid Serper API key, or quota exhaustion resulting in empty search payloads.
  - **Search the expert angle**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Sends a second POST request to `https://google.serper.dev/search` using the auto-generated expert query string to surface papers, statistics, and industry reports.
    - *Key Expressions or Variables:* Uses `={{ $('Set config').item.json.expertQuery }}` and `={{ $('Set config').item.json.serperApiKey }}`.
    - *Input and Output Connections:* Input: `Search the main topic`. Output: Connects to `Combine the sources`.
    - *Version-Specific Requirements:* TypeVersion 4.2.
    - *Edge Cases / Potential Failure Types:* API rate limiting or zero matching search results.
  - **Combine the sources**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration Choices:* Executes custom JavaScript to parse the top 8 results from the main search and the top 7 results from the expert search, filters out duplicate URLs, assigns sequential source numbers, and builds structured text blocks (`researchContent` and `sourceList`).
    - *Key Expressions or Variables:* References `$input.first().json`, `$('Search the main topic').item.json`, and `$('Set config').item.json`.
    - *Input and Output Connections:* Input: `Search the expert angle`. Output: Connects to `Analyse the sources`.
    - *Version-Specific Requirements:* TypeVersion 2.
    - *Edge Cases / Potential Failure Types:* Unexpected JSON response structure from Serper API causing undefined property reading errors.

---

#### Block 1.3: Two-Stage AI Intelligence & Drafting
- **Overview:** This block executes a two-stage LLM pipeline. The first AI agent analyzes raw web snippets into structured intelligence metrics (key findings, confidence scores, gaps), and the second agent consumes this structured output to draft a professional multi-section report.
- **Nodes Involved:**
  - `Analyse the sources`
  - `Chat model - Analysis`
  - `Output Parser - Analysis`
  - `Write the report`
  - `Chat model - Writing`
  - `Output Parser - Report`

- **Node Details:**
  - **Analyse the sources**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (Advanced AI Agent)
    - *Configuration Choices:* Configured with a system prompt instructing the agent to act as a senior research analyst. Utilizes an output parser to ensure strictly typed JSON output containing 7 required analytical fields.
    - *Key Expressions or Variables:* Injects topic parameters, source counts, and raw research content from `Combine the sources`.
    - *Input and Output Connections:* Input: `Combine the sources`. Output: Connects to `Write the report`. AI model input connected to `Chat model - Analysis`. Output parser connected to `Output Parser - Analysis`.
    - *Version-Specific Requirements:* TypeVersion 1.7.
    - *Edge Cases / Potential Failure Types:* LLM hallucination or output failing schema validation.
  - **Chat model - Analysis**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Integration)
    - *Configuration Choices:* Uses model `gpt-4o-mini`, temperature `0.2` (for factual extraction), and `maxTokens: 1200`.
    - *Key Expressions or Variables:* Uses OpenAI credentials.
    - *Input and Output Connections:* Connected to `Analyse the sources`.
    - *Version-Specific Requirements:* TypeVersion 1.2.
    - *Edge Cases / Potential Failure Types:* OpenAI API authentication failures or rate limits.
  - **Output Parser - Analysis**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser)
    - *Configuration Choices:* Enforces a manual JSON schema requiring arrays for `keyFindings`, `dataPoints`, `expertPositions`, `sourceCredibility`, `knowledgeGaps`, `contradictions`, and a number for `researchConfidence`.
    - *Input and Output Connections:* Connected to `Analyse the sources`.
    - *Version-Specific Requirements:* TypeVersion 1.3.
    - *Edge Cases / Potential Failure Types:* Model returning markdown backticks or malformed JSON that fails validation.
  - **Write the report**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (Advanced AI Agent)
    - *Configuration Choices:* Configured to act as a professional research report writer, taking structured intelligence metrics from the previous agent and compiling them into an 8-section report structure.
    - *Key Expressions or Variables:* Dynamically pulls parsed JSON arrays and metrics from `Analyse the sources`.
    - *Input and Output Connections:* Input: `Analyse the sources`. Output: Connects to `Build the report email`. AI model connected to `Chat model - Writing`. Output parser connected to `Output Parser - Report`.
    - *Version-Specific Requirements:* TypeVersion 1.7.
    - *Edge Cases / Potential Failure Types:* Truncation due to token limits.
  - **Chat model - Writing**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Integration)
    - *Configuration Choices:* Uses model `gpt-4o-mini`, temperature `0.5` (for balanced creative professional writing), and `maxTokens: 1500`.
    - *Key Expressions or Variables:* Uses OpenAI credentials.
    - *Input and Output Connections:* Connected to `Write the report`.
    - *Version-Specific Requirements:* TypeVersion 1.2.
    - *Edge Cases / Potential Failure Types:* API timeouts or insufficient token limits.
  - **Output Parser - Report**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser)
    - *Configuration Choices:* Enforces a manual JSON schema requiring fields: `reportTitle`, `executiveSummary`, `background`, `keyInsights` (array), `detailedFindings`, `implications`, `recommendedActions` (array), and `conclusion`.
    - *Input and Output Connections:* Connected to `Write the report`.
    - *Version-Specific Requirements:* TypeVersion 1.3.
    - *Edge Cases / Potential Failure Types:* Schema mismatch or parsing errors.

---

#### Block 1.4: Logging & Delivery
- **Overview:** This block formats the final email payload, records metadata into a Google Sheets logging tab, and sends the finished report to the recipient via Gmail.
- **Nodes Involved:**
  - `Build the report email`
  - `Log the request`
  - `Email the report`

- **Node Details:**
  - **Build the report email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation)
    - *Configuration Choices:* Combines data from Agent 2 (`$input.first().json.output`), Agent 1 (`$('Analyse the sources').item.json.output`), and configuration variables (`$('Combine the sources').item.json`) to construct a comprehensive plain-text email body and structured logging variables.
    - *Key Expressions or Variables:* Maps email subject lines, confidence level indicators, timestamp strings (`loggedAt`), and concatenated text blocks.
    - *Input and Output Connections:* Input: `Write the report`. Output: Connects to `Log the request`.
    - *Version-Specific Requirements:* TypeVersion 2.
    - *Edge Cases / Potential Failure Types:* Missing AI output properties throwing runtime evaluation errors.
  - **Log the request**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Integration)
    - *Configuration Choices:* Appends a new row (`operation: "append"`) to the designated spreadsheet using cell format `USER_ENTERED`. Maps columns: Date, Goal, Topic, Audience, Logged At, Key Findings, Report Title, Requested By, Sources Used, and Confidence Level.
    - *Key Expressions or Variables:* Dynamically references sheet name (`={{ $json.sheetName }}`), document ID (`={{ $json.sheetId }}`), and respective data fields.
    - *Input and Output Connections:* Input: `Build the report email`. Output: Connects to `Email the report`.
    - *Version-Specific Requirements:* TypeVersion 4.5.
    - *Edge Cases / Potential Failure Types:* Missing Google Sheets OAuth2 credentials, incorrect spreadsheet ID, or missing sheet tab name.
  - **Email the report**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration Choices:* Sends an email via the connected Gmail account with attribution appending disabled (`appendAttribution: false`).
    - *Key Expressions or Variables:* 
      - Send To: `={{ $('Build the report email').item.json.recipientEmail }}`
      - Subject: `={{ $('Build the report email').item.json.emailSubject }}`
      - Message: `={{ $('Build the report email').item.json.emailBody }}`
    - *Input and Output Connections:* Input: `Log the request`. Output: None (Terminal Node).
    - *Version-Specific Requirements:* TypeVersion 2.1.
    - *Edge Cases / Potential Failure Types:* Gmail API token expiration or sending limits exceeded.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Research request form` | `n8n-nodes-base.formTrigger` | Trigger Node | None | `Set config` | ## 1. Take the request<br>A form asks for the topic, audience and goal of the report. One config node holds your search API key, email and sheet. |
| `Set config` | `n8n-nodes-base.set` | Configuration Mapping | `Research request form` | `Search the main topic` | ## 1. Take the request<br>A form asks for the topic, audience and goal of the report. One config node holds your search API key, email and sheet. |
| `Search the main topic` | `n8n-nodes-base.httpRequest` | API Request | `Set config` | `Search the expert angle` | ## 2. Search the web twice<br>One search covers the topic itself, a second looks for expert and data angles. Duplicates are dropped and up to fifteen numbered sources are combined. |
| `Search the expert angle` | `n8n-nodes-base.httpRequest` | API Request | `Search the main topic` | `Combine the sources` | ## 2. Search the web twice<br>One search covers the topic itself, a second looks for expert and data angles. Duplicates are dropped and up to fifteen numbered sources are combined. |
| `Combine the sources` | `n8n-nodes-base.code` | Data Transformation | `Search the expert angle` | `Analyse the sources` | ## 2. Search the web twice<br>One search covers the topic itself, a second looks for expert and data angles. Duplicates are dropped and up to fifteen numbered sources are combined. |
| `Analyse the sources` | `@n8n/n8n-nodes-langchain.agent` | AI Agent (Analysis) | `Combine the sources` | `Write the report` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Chat model - Analysis` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Integration | None | `Analyse the sources` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Output Parser - Analysis` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output Schema Validation | None | `Analyse the sources` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Write the report` | `@n8n/n8n-nodes-langchain.agent` | AI Agent (Drafting) | `Analyse the sources` | `Build the report email` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Chat model - Writing` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Integration | None | `Write the report` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Output Parser - Report` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output Schema Validation | None | `Write the report` | ## 3. Analyse, then write<br>The first agent pulls out findings, data points and disagreements with source numbers. The second writes the finished report for your audience and goal. |
| `Build the report email` | `n8n-nodes-base.code` | Data Transformation | `Write the report` | `Log the request` | ## 4. Log and deliver<br>The report is formatted as an email with its sources listed, logged to your sheet first, then sent. |
| `Log the request` | `n8n-nodes-base.googleSheets` | Database Logging | `Build the report email` | `Email the report` | ## 4. Log and deliver<br>The report is formatted as an email with its sources listed, logged to your sheet first, then sent. |
| `Email the report` | `n8n-nodes-base.gmail` | Email Dispatch | `Log the request` | None | ## 4. Log and deliver<br>The report is formatted as an email with its sources listed, logged to your sheet first, then sent. |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to manually rebuild the workflow in n8n:

1. **Create the Form Trigger Node:**
   - Add a **Form Trigger** node named `Research request form`.
   - Set form title to `AI Research Report Request` and add form fields: *Research Topic* (Required), *Target Audience* (Required), *Report Goal* (Required), *Specific Angle or Focus* (Optional), and *Your Name* (Optional).

2. **Create the Configuration Node:**
   - Add a **Set** node named `Set config`.
   - Configure assignments to set string values for `serperApiKey`, `recipientEmail`, `senderName`, `sheetId`, and `sheetName`.
   - Add assignments mapping form fields and expressions:
     - `mainTopic`: `={{ $json['Research Topic'] }}`
     - `targetAudience`: `={{ $json['Target Audience'] }}`
     - `reportGoal`: `={{ $json['Report Goal'] }}`
     - `specificAngle`: `={{ $json['Specific Angle or Focus'] || 'None specified' }}`
     - `requestedBy`: `={{ $json['Your Name'] || 'Unknown' }}`
     - `mainQuery`: `={{ $json['Research Topic'] }}`
     - `expertQuery`: `={{ $json['Research Topic'] + ' expert insights data research 2026' }}`
     - `today`: `={{ $now.toFormat('dd MMM yyyy') }}`
   - Connect `Research request form` to `Set config`.

3. **Create the Main Web Search Node:**
   - Add an **HTTP Request** node named `Search the main topic`.
   - Set Method to `POST`, URL to `https://google.serper.dev/search`.
   - Set Header Parameters: `X-API-KEY` = `={{ $json.serperApiKey }}`, `Content-Type` = `application/json`.
   - Set Body (JSON) to:
     ```json
     ={
       "q": "{{ $json.mainQuery }}",
       "num": 10,
       "gl": "us",
       "hl": "en"
     }
     ```
   - Enable `Continue On Fail` (`continueRegularOutput`) and `Always Output Data`.
   - Connect `Set config` to `Search the main topic`.

4. **Create the Expert Angle Search Node:**
   - Add an **HTTP Request** node named `Search the expert angle`.
   - Set Method to `POST`, URL to `https://google.serper.dev/search`.
   - Set Header Parameters: `X-API-KEY` = `={{ $('Set config').item.json.serperApiKey }}`, `Content-Type` = `application/json`.
   - Set Body (JSON) to:
     ```json
     ={
       "q": "{{ $('Set config').item.json.expertQuery }}",
       "num": 10,
       "gl": "us",
       "hl": "en"
     }
     ```
   - Connect `Search the main topic` to `Search the expert angle`.

5. **Create the Source Combination Code Node:**
   - Add a **Code** node named `Combine the sources`.
   - Insert JavaScript to slice top 8 results from search 1, top 7 results from search 2, deduplicate URLs, assign sequential numbering, and format `researchContent` and `sourceList`.
   - Connect `Search the expert angle` to `Combine the sources`.

6. **Build the Analysis AI Agent Subtree:**
   - Add an **Advanced AI Agent** node named `Analyse the sources`. Set prompt type to define and enable output parser.
   - Add a **Language Model (OpenAI)** node named `Chat model - Analysis`, select model `gpt-4o-mini`, set temperature to `0.2`, and connect OpenAI API credentials. Link to the AI Agent's language model input.
   - Add a **Structured Output Parser** node named `Output Parser - Analysis` with a JSON schema defining required properties: `keyFindings`, `dataPoints`, `expertPositions`, `sourceCredibility`, `knowledgeGaps`, `contradictions`, and `researchConfidence`. Link to the AI Agent's output parser input.
   - Connect `Combine the sources` to `Analyse the sources`.

7. **Build the Report Writing AI Agent Subtree:**
   - Add an **Advanced AI Agent** node named `Write the report`. Set prompt type to define and enable output parser.
   - Add a **Language Model (OpenAI)** node named `Chat model - Writing`, select model `gpt-4o-mini`, set temperature to `0.5`, and connect OpenAI API credentials. Link to the AI Agent's language model input.
   - Add a **Structured Output Parser** node named `Output Parser - Report` with a JSON schema defining required properties: `reportTitle`, `executiveSummary`, `background`, `keyInsights`, `detailedFindings`, `implications`, `recommendedActions`, and `conclusion`. Link to the AI Agent's output parser input.
   - Connect `Analyse the sources` to `Write the report`.

8. **Create the Email & Data Assembly Code Node:**
   - Add a **Code** node named `Build the report email`.
   - Insert JavaScript to assemble the 8-section plain-text email body and extract fields for Google Sheets logging.
   - Connect `Write the report` to `Build the report email`.

9. **Create the Google Sheets Logging Node:**
   - Add a **Google Sheets** node named `Log the request`.
   - Set Operation to `Append`, select document ID and sheet name (`Research Log`) using expressions referencing `$json.sheetId` and `$json.sheetName`.
   - Map columns (`Date`, `Goal`, `Topic`, `Audience`, `Logged At`, `Key Findings`, `Report Title`, `Requested By`, `Sources Used`, `Confidence Level`) to corresponding JSON properties. Configure Google Sheets OAuth2 credentials.
   - Connect `Build the report email` to `Log the request`.

10. **Create the Gmail Notification Node:**
    - Add a **Gmail** node named `Email the report`.
    - Configure Send To (`={{ $('Build the report email').item.json.recipientEmail }}`), Subject (`={{ $('Build the report email').item.json.emailSubject }}`), and Message (`={{ $('Build the report email').item.json.emailBody }}`). Configure Gmail OAuth2 credentials.
    - Connect `Log the request` to `Email the report`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Serper API Provider | [serper.dev](https://serper.dev) — Free tier provides 2,500 searches per month (workflow consumes 2 searches per execution). |
| Google Cloud Console | [console.cloud.google.com](https://console.cloud.google.com) — Required to enable Google Sheets API and Gmail API for OAuth2 authentication. |
| OpenAI Platform | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) — Required for generating API keys for the GPT-4o-mini chat models. |