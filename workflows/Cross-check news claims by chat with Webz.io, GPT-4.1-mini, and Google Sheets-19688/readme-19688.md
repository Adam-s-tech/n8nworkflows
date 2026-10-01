Cross-check news claims by chat with Webz.io, GPT-4.1-mini, and Google Sheets

https://n8nworkflows.xyz/workflows/cross-check-news-claims-by-chat-with-webz-io--gpt-4-1-mini--and-google-sheets-19688


# Cross-check news claims by chat with Webz.io, GPT-4.1-mini, and Google Sheets

### 1. Workflow Overview

This workflow is a chat-driven global news cross-checking and verification engine. Its primary purpose is to take a user question about current events, query Webz.io for recent news coverage using GPT-4.1-mini, de-duplicate and verify independent source domains, and archive the resulting structured claims to Google Sheets before replying in the chat interface.

The logical blocks are organized as follows:
- **1.1 Input Reception & Configuration:** Receives the chat message and injects global research parameters.
- **1.2 Intent Routing & AI Processing:** Classifies the query depth and executes either a quick lookup or a deep research agent equipped with thought-planning, web search, memory, and structured output parsing.
- **1.3 Corroboration & Filtering:** Normalizes and counts independent publisher domains per claim against a threshold, determining whether findings are corroborated or single-source.
- **1.4 Output Generation & Archiving:** Formats a markdown-styled briefing, archives records row-by-row into Google Sheets, and outputs the final response or alternative search angles if no coverage is found.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Captures incoming chat queries and initializes global configuration settings (such as default language, lookback window, and corroboration thresholds) used throughout the execution.
- **Nodes Involved:** 
  - `When chat message received`
  - `Research settings`

- **Node Details:**
  - **When chat message received**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chatTrigger` (Chat Trigger) - Acts as the entry point for chat interactions.
    - *Configuration Choices:* Standard options configuration.
    - *Key Expressions:* Accesses message text via `{{ $json.chatInput }}` in downstream nodes.
    - *Input/Output:* Output connects to `Research settings`.
    - *Version Requirements:* v1.4.
    - *Edge Cases/Failures:* Fails if the chat session disconnects or webhooks are misconfigured.

  - **Research settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Set) - Defines environment and threshold parameters.
    - *Configuration Choices:* Manual assignment mode.
    - *Key Expressions:* Sets `defaultLanguage` ("english"), `lookbackDays` (7), `minSourcesToCorroborate` (2), `maxFindings` (6).
    - *Input/Output:* Input from `When chat message received`; output connects to `Route by research depth`.
    - *Version Requirements:* v3.5.
    - *Edge Cases/Failures:* Typographical errors in JSON assignment structures can break downstream evaluations.

---

#### Block 1.2: Intent Routing & AI Processing
- **Overview:** Evaluates the user query to determine research complexity, then routes execution to specialized AI agents that query news articles via Webz.io, utilize memory, plan research angles, and enforce structured outputs.
- **Nodes Involved:**
  - `Route by research depth`
  - `Answer the quick lookup`
  - `Run the deep research`
  - `OpenAI Chat Model`
  - `Conversation memory`
  - `Plan the research`
  - `Webz.io news search`
  - `Enforce the finding format`

- **Node Details:**
  - **Route by research depth**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.textClassifier` (Text Classifier) - Categorizes queries into quick lookups or deep research paths.
    - *Configuration Choices:* Fallback category set to "other". Categories configured for "Quick lookup" and "Deep research".
    - *Key Expressions:* `={{ $json.chatInput }}`
    - *Input/Output:* Input from `Research settings`; outputs connect to `Answer the quick lookup` and `Run the deep research`.
    - *Version Requirements:* v1.1.
    - *Edge Cases/Failures:* Misclassification may route complex queries to the quick path, reducing search depth.

  - **Answer the quick lookup**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) - Handles simple factual lookups with a single search.
    - *Configuration Choices:* Uses system message enforcing tool usage, language arrays, and retry limits. Max tries: 2.
    - *Key Expressions:* Uses template strings mapping `{{ $json.chatInput }}`, `{{ $json.defaultLanguage }}`, and `{{ $json.lookbackDays }}`.
    - *Input/Output:* Inputs from classifier/settings, OpenAI model, memory, search tool, and output parser; output connects to `Merge research paths`.
    - *Version Requirements:* v3.1.
    - *Edge Cases/Failures:* API timeouts or missing search results return empty findings.

  - **Run the deep research**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) - Executes multi-angle comparative research.
    - *Configuration Choices:* Configured with think-tool support and strict instructions for handling independent outlets. Max tries: 2.
    - *Key Expressions:* Uses interpolation for lookback days, language, and corroboration thresholds (`{{ $json.minSourcesToCorroborate }}`).
    - *Input/Output:* Inputs from classifier/settings, OpenAI model, memory, think tool, search tool, and output parser; output connects to `Merge research paths`.
    - *Version Requirements:* v3.1.
    - *Edge Cases/Failures:* High token consumption or search rate-limit exhaustion from Webz.io.

  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Chat Model) - Provides the underlying language model (`gpt-4.1-mini`).
    - *Configuration Choices:* Model selected via list: `gpt-4.1-mini`.
    - *Key Expressions:* None.
    - *Input/Output:* Connects model outputs to the classifier and both agents.
    - *Credentials:* Requires an OpenAI API credential.
    - *Version Requirements:* v1.3.
    - *Edge Cases/Failures:* Invalid API key, credit exhaustion, or upstream OpenAI outages.

  - **Conversation memory**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Window Buffer Memory) - Retains chat conversation history within the session.
    - *Configuration Choices:* Default window settings.
    - *Input/Output:* Connects to both research agents.
    - *Version Requirements:* v1.4.

  - **Plan the research**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.toolThink` (Think Tool) - Scratch space for the deep research agent to plan search angles.
    - *Configuration Choices:* Description set to guide planning actions.
    - *Input/Output:* Connects as an AI tool to `Run the deep research`.
    - *Version Requirements:* v1.1.

  - **Webz.io news search**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.mcpClientTool` (MCP Client Tool) - Executes news searches via Webz.io's Model Context Protocol endpoint.
    - *Configuration Choices:* Endpoint URL set to `https://news-search-mcp.webz.io/mcp`, authentication set to bearerAuth, transport set to httpStreamable.
    - *Input/Output:* Connects as a tool to both `Answer the quick lookup` and `Run the deep research`.
    - *Credentials:* Requires a Webz.io Bearer Auth credential.
    - *Version Requirements:* v1.4.
    - *Edge Cases/Failures:* Invalid API bearer token or network timeout.

  - **Enforce the finding format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser) - Forces the agent response into a validated JSON schema.
    - *Configuration Choices:* Manual schema defining `summary`, `no_results`, and an array of `findings` containing claims and source objects.
    - *Input/Output:* Connects to the output parser ports of both agents.
    - *Version Requirements:* v1.3.
    - *Edge Cases/Failures:* Model hallucinations failing schema validation.

---

#### Block 1.3: Corroboration & Filtering
- **Overview:** Consolidates agent outputs, resolves article URLs to registrable domains to eliminate duplicate newsroom sources, counts independent publishers, and evaluates corroboration status.
- **Nodes Involved:**
  - `Merge research paths`
  - `Check source corroboration`
  - `Any claims to report?`

- **Node Details:**
  - **Merge research paths**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Merge) - Combines outputs from whichever research agent branch executed.
    - *Configuration Choices:* Number of inputs set to 2.
    - *Input/Output:* Inputs from quick and deep research agents; output connects to `Check source corroboration`.
    - *Version Requirements:* v3.2.

  - **Check source corroboration**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code) - Executes custom JavaScript to normalize URLs, strip `www.`, map unique registrable domains, count independent sources, and calculate statistics.
    - *Configuration Choices:* Custom JS code utilizing `URL` parsing and filtering logic.
    - *Input/Output:* Input from `Merge research paths`; output connects to `Any claims to report?`.
    - *Version Requirements:* v2.
    - *Edge Cases/Failures:* Malformed URLs throwing exceptions inside the domain-extraction try/catch block.

  - **Any claims to report?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If) - Checks whether any verified findings were returned.
    - *Configuration Choices:* Condition set to check if `{{ $json.stats.finding_count }}` is greater than 0.
    - *Input/Output:* Input from `Check source corroboration`; true branch connects to `Compose the sourced briefing`, false branch connects to `Suggest a different angle`.
    - *Version Requirements:* v2.3.

---

#### Block 1.4: Output Generation & Archiving
- **Overview:** Formats findings into a structured markdown briefing, archives each claim individually to Google Sheets, and returns the final response to the user or suggests alternative search angles.
- **Nodes Involved:**
  - `Compose the sourced briefing`
  - `Split out one row per claim`
  - `Archive the briefing to Google Sheets`
  - `Reply with the briefing`
  - `Suggest a different angle`

- **Node Details:**
  - **Compose the sourced briefing**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code) - Generates markdown text for the chat interface and structures payload arrays for archiving.
    - *Configuration Choices:* Custom JavaScript constructing markdown lists and timestamping via `new Date().toISOString()`.
    - *Input/Output:* Input from `Any claims to report?` (true branch); output connects to `Split out one row per claim`.
    - *Version Requirements:* v2.

  - **Split out one row per claim**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Split Out) - Flattens the array of findings so each claim is processed as a separate item.
    - *Configuration Choices:* Field to split out set to `findings`.
    - *Input/Output:* Input from `Compose the sourced briefing`; output connects to `Archive the briefing to Google Sheets`.
    - *Version Requirements:* v1.

  - **Archive the briefing to Google Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) - Appends claim records to a Google Sheet.
    - *Configuration Choices:* Operation set to `append`, mapping columns (`saved_at`, `question`, `claim`, `status`, `source_count`, `publishers`, `urls`).
    - *Input/Output:* Input from `Split out one row per claim`; output connects to `Reply with the briefing`.
    - *Credentials:* Requires Google Sheets OAuth2 / Service Account credentials.
    - *Version Requirements:* v4.7.
    - *Edge Cases/Failures:* Missing sheet headers, permission errors, or invalid document ID configurations.

  - **Reply with the briefing**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Set) - Prepares the final chat response payload containing the markdown briefing.
    - *Configuration Choices:* Manual assignment mode mapping `output` to `={{ $('Compose the sourced briefing').first().json.briefing }}`. Execute once enabled.
    - *Input/Output:* Input from `Archive the briefing to Google Sheets`.
    - *Version Requirements:* v3.5.

  - **Suggest a different angle**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Set) - Provides fallback suggestions when no search results are returned.
    - *Configuration Choices:* Manual assignment setting `output` text detailing alternative search strategies.
    - *Input/Output:* Input from `Any claims to report?` (false branch).
    - *Version Requirements:* v3.5.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template overview | stickyNote | High-level documentation and setup guide | None | None | ## Cross-check global news by chat with Webz.io, GPT-4.1-mini and Google Sheets... [Setup notes and full filter reference](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |
| Step 1 note | stickyNote | Explains chat input and research settings | None | None | ### 1. Ask a question<br><br>Open the chat panel at the bottom of the canvas and type in plain language... |
| Step 2 note | stickyNote | Explains query routing by depth | None | None | ### 2. Route by depth<br><br>Not every question deserves the same spend... |
| Step 3 note | stickyNote | Explains the quick and deep research paths | None | None | ### 3. Research<br><br>Both agents return the same shape: a list of claims... |
| Step 4 note | stickyNote | Explains verification and source corroboration | None | None | ### 4. Verify across sources<br><br>The part that makes this more than a chatbot... |
| Step 5 note | stickyNote | Explains empty result handling | None | None | ### 5. Anything to report?<br><br>An empty result is a normal outcome, not a failure... |
| Step 6 note | stickyNote | Explains briefing composition and archiving | None | None | ### 6. Compose, archive, reply<br><br>Corroborated claims lead the briefing, every outlet is linked... |
| When chat message received | chatTrigger | Captures user chat input | None | Research settings | ### 1. Ask a question<br><br>Open the chat panel at the bottom of the canvas and type in plain language... |
| Research settings | set | Defines global research variables | When chat message received | Route by research depth | ### 1. Ask a question<br><br>Open the chat panel at the bottom of the canvas and type in plain language... |
| Route by research depth | textClassifier | Classifies query complexity | Research settings | Answer the quick lookup, Run the deep research | ### 2. Route by depth<br><br>Not every question deserves the same spend... |
| Answer the quick lookup | agent | Executes single-search information retrieval | Route by research depth, OpenAI Chat Model, Conversation memory, Webz.io news search, Enforce the finding format | Merge research paths | ### 3. Research<br><br>Both agents return the same shape: a list of claims... |
| Run the deep research | agent | Executes multi-angle deep research | Route by research depth, OpenAI Chat Model, Conversation memory, Plan the research, Webz.io news search, Enforce the finding format | Merge research paths | ### 3. Research<br><br>Both agents return the same shape: a list of claims... |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT-4.1-mini language model | None | Route by research depth, Answer the quick lookup, Run the deep research | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |
| Conversation memory | memoryBufferWindow | Maintains chat session history | None | Answer the quick lookup, Run the deep research | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |
| Plan the research | toolThink | Scratch space tool for deep research planning | None | Run the deep research | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |
| Webz.io news search | mcpClientTool | MCP tool for fetching news articles | None | Answer the quick lookup, Run the deep research | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |
| Enforce the finding format | outputParserStructured | Validates structured JSON output schema | None | Answer the quick lookup, Run the deep research | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |
| Merge research paths | merge | Combines agent execution branches | Answer the quick lookup, Run the deep research | Check source corroboration | None |
| Check source corroboration | code | Normalizes domains and counts independent sources | Merge research paths | Any claims to report? | ### 4. Verify across sources<br><br>The part that makes this more than a chatbot... |
| Any claims to report? | if | Evaluates if findings exist | Check source corroboration | Compose the sourced briefing, Suggest a different angle | ### 5. Anything to report?<br><br>An empty result is a normal outcome, not a failure... |
| Compose the sourced briefing | code | Formats markdown briefing and structured payload | Any claims to report? | Split out one row per claim | ### 6. Compose, archive, reply<br><br>Corroborated claims lead the briefing, every outlet is linked... |
| Split out one row per claim | splitOut | Flattens findings array into individual items | Compose the sourced briefing | Archive the briefing to Google Sheets | ### 6. Compose, archive, reply<br><br>Corroborated claims lead the briefing, every outlet is linked... |
| Archive the briefing to Google Sheets | googleSheets | Appends claim records to Google Sheets | Split out one row per claim | Reply with the briefing | ### 6. Compose, archive, reply<br><br>Corroborated claims lead the briefing, every outlet is linked... |
| Reply with the briefing | set | Returns the markdown briefing to chat | Archive the briefing to Google Sheets | None | ### 6. Compose, archive, reply<br><br>Corroborated claims lead the briefing, every outlet is linked... |
| Suggest a different angle | set | Provides alternative search advice on empty results | Any claims to report? | None | ### 5. Anything to report?<br><br>An empty result is a normal outcome, not a failure... |
| Sub-node note | stickyNote | Explains underlying models, memory, and tools | None | None | ### Model, memory, and tools<br><br>- **OpenAI Chat Model** drives the classifier and both agents... |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the entire workflow manually in n8n:

1. **Create the Entry Point:**
   - Add a **Chat Trigger** node (`@n8n/n8n-nodes-langchain.chatTrigger`) named `When chat message received`. Leave options at default.
2. **Configure Research Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Research settings`.
   - Connect `When chat message received` to `Research settings`.
   - Add manual assignments:
     - `defaultLanguage` (String): `english`
     - `lookbackDays` (Number): `7`
     - `minSourcesToCorroborate` (Number): `2`
     - `maxFindings` (Number): `6`
3. **Add Model and Infrastructure Nodes (Standalone):**
   - **OpenAI Chat Model:** Add `@n8n/n8n-nodes-langchain.lmChatOpenAi`. Set model to `gpt-4.1-mini`. Configure your OpenAI API credentials.
   - **Conversation memory:** Add `@n8n/n8n-nodes-langchain.memoryBufferWindow`.
   - **Plan the research:** Add `@n8n/n8n-nodes-langchain.toolThink` with description `"Scratch space for planning..."`.
   - **Webz.io news search:** Add `@n8n/n8n-nodes-langchain.mcpClientTool`. Set endpoint URL to `https://news-search-mcp.webz.io/mcp`, authentication to `bearerAuth`, and configure your Webz.io Bearer Token credential.
   - **Enforce the finding format:** Add `@n8n/n8n-nodes-langchain.outputParserStructured`. Set schema type to manual and paste the JSON schema defining `summary`, `no_results`, and `findings` (with `claim` and `sources` array).
4. **Configure Intent Routing:**
   - Add a **Text Classifier** node (`@n8n/n8n-nodes-langchain.textClassifier`) named `Route by research depth`.
   - Connect `Research settings` main output to `Route by research depth`.
   - Set input text to `={{ $json.chatInput }}` and configure fallback to `other` with categories `Quick lookup` and `Deep research`.
   - Connect OpenAI Chat Model to the `ai_languageModel` input of the classifier.
5. **Configure Research Agents:**
   - **Answer the quick lookup:** Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`). Connect OpenAI model (`ai_languageModel`), Conversation memory (`ai_memory`), Webz.io news search (`ai_tool`), and Enforce the finding format (`ai_outputParser`). Connect the first output of `Route by research depth` to it.
   - **Run the deep research:** Add another **AI Agent** node. Connect OpenAI model (`ai_languageModel`), Conversation memory (`ai_memory`), Plan the research (`ai_tool`), Webz.io news search (`ai_tool`), and Enforce the finding format (`ai_outputParser`). Connect the second output of `Route by research depth` to it. (Note: The third fallback route also connects to `Answer the quick lookup`).
6. **Merge and Process Corroboration:**
   - Add a **Merge** node (`n8n-nodes-base.merge`) named `Merge research paths` with `numberInputs: 2`. Connect both agent outputs to inputs 0 and 1.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Check source corroboration`. Connect `Merge research paths` to it. Paste the JavaScript code that extracts registrable domains, calculates source counts, and determines the `corroborated` or `single-source` status.
7. **Evaluate Findings & Handle Empty Results:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Any claims to report?`. Connect `Check source corroboration` to it. Configure condition: `{{ $json.stats.finding_count }}` greater than `0`.
   - **False branch:** Add a **Set** node named `Suggest a different angle` with manual assignment returning alternative search advice.
8. **Compose Briefing and Archive:**
   - **True branch:** Add a **Code** node named `Compose the sourced briefing`. Paste the briefing rendering script.
   - Add a **Split Out** node (`n8n-nodes-base.splitOut`) named `Split out one row per claim` targeting the `findings` field. Connect briefing node to it.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Archive the briefing to Google Sheets`. Connect split-out node to it. Set operation to `append`, configure credentials, and map columns (`saved_at`, `question`, `claim`, `status`, `source_count`, `publishers`, `urls`).
9. **Final Reply Generation:**
   - Add a **Set** node named `Reply with the briefing`. Connect Google Sheets node to it. Map `output` to `={{ $('Compose the sourced briefing').first().json.briefing }}` with `executeOnce` enabled.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Setup notes and full filter reference | [GitHub Repository](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |
| Webz.io API Token Requirement | [Webz.io Official Website](https://webz.io) |