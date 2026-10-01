Track startup funding rounds with Webz.io, GPT-4.1-mini, Slack and Sheets

https://n8nworkflows.xyz/workflows/track-startup-funding-rounds-with-webz-io--gpt-4-1-mini--slack-and-sheets-20030


# Track startup funding rounds with Webz.io, GPT-4.1-mini, Slack and Sheets

### 1. Workflow Overview

This workflow automates the discovery, processing, filtering, and archiving of recent startup funding announcements. Designed for venture capitalists, sales teams, and market researchers, it removes the need to manually scan tech news sources. The process runs daily, queries global news providers, validates and structures the returned data using an AI model, removes duplicates, posts a compiled notification to Slack, and writes individual records into Google Sheets.

The logical execution is organized into the following functional blocks:
- **1.1 Initialization and Configuration:** Sets execution schedules and centralizes user-defined parameters (sectors, funding stages, financial thresholds, and output targets).
- **1.2 Data Extraction and AI Processing:** Iterates through target sectors, executes external news searches via tool-calling agents, and enforces strict schema constraints on the extracted data.
- **1.3 Deduplication and Filtering:** Compares incoming results against a persistent memory cache, filters out entries below minimum financial thresholds, and flags execution errors.
- **1.4 Output Distribution:** Evaluates whether new records exist, dispatches aggregated summaries to Slack, and fans out items for archival in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration
- **Overview:** Establishes the daily execution cadence and provides a centralized configuration interface for user-defined parameters such as target sectors, funding stages, and alert channels.
- **Nodes Involved:** `Every morning at 08:00`, `Funding tracker settings`, `One item per sector`
- **Node Details:**
  - **Every morning at 08:00**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration Choices:* Configured to fire daily at hour 8.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Output connects to `Funding tracker settings`.
    - *Version-specific Requirements:* Version 1.4. Requires proper workflow timezone configuration in n8n settings to align with local execution times (defaults to UTC).
    - *Edge Cases / Potential Failure Types:* Failure to set the workflow timezone can lead to triggers executing at unintended local times.
  - **Funding tracker settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation)
    - *Configuration Choices:* Manual assignment mode. Defines manual string and number pairs for sectors, stages, minimum USD amounts, lookback days, article counts, and the target Slack channel.
    - *Key Expressions:* Static assignments (e.g., sectors: `AI, fintech, climate tech, cybersecurity`, minAmountUsd: `1000000`).
    - *Input/Output Connections:* Input from `Every morning at 08:00`; Output connects to `One item per sector`.
    - *Version-specific Requirements:* Version 3.5.
    - *Edge Cases / Potential Failure Types:* Typographical errors in parameter names or invalid data types can cause downstream processing blocks to fail.
  - **One item per sector**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Splits a comma-separated string of sectors into individual execution items.
    - *Key Expressions:* Parses `settings.sectors`, trims whitespace, filters empty strings, and maps each sector into an individual JSON payload containing tracker configurations.
    - *Input/Output Connections:* Input from `Funding tracker settings`; Output connects to `Find funding rounds`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Throws a runtime error if the sector string is empty or unparseable.

#### 2.2 Data Extraction and AI Processing
- **Overview:** Processes each sector individually by invoking an AI agent equipped with external tool access and a strict output parser to structure news search results.
- **Nodes Involved:** `Find funding rounds`, `OpenAI Chat Model`, `Webz.io news search`, `Enforce the round format`, `Report the failure`
- **Node Details:**
  - **Find funding rounds**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration Choices:* Tool-calling agent configured with up to 2 retry attempts and continuation logic on error output routing.
    - *Key Expressions:* Uses dynamic prompt interpolation: `{{ $json.sector }}`, `{{ $json.stages }}`, `{{ $json.lookbackDays }}`, and `{{ $json.articleCount }}` to instruct the model on querying external tools.
    - *Input/Output Connections:* Input from `One item per sector`. Connected to `OpenAI Chat Model` (AI Language Model), `Webz.io news search` (AI Tool), and `Enforce the round format` (AI Output Parser). Main output branches to `Drop rounds already seen` (success) and `Report the failure` (error output).
    - *Version-specific Requirements:* Version 3.1.
    - *Edge Cases / Potential Failure Types:* API rate limits, upstream tool downtime, or hallucinated responses that fail validation against the schema parser.
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Sub-node / Language Model)
    - *Configuration Choices:* Utilizes the `gpt-4.1-mini` model.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Connects exclusively to the `ai_languageModel` input of `Find funding rounds`.
    - *Version-specific Requirements:* Version 1.3. Requires valid OpenAI credentials.
    - *Edge Cases / Potential Failure Types:* Invalid API keys, model deprecation, or quota exhaustion.
  - **Webz.io news search**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.mcpClientTool` (AI Sub-node / Tool)
    - *Configuration Choices:* Model Context Protocol (MCP) client tool connecting to `https://news-search-mcp.webz.io/mcp` via HTTP Streamable transport and Bearer Token authentication.
    - *Key Expressions:* None.
    - *Input/Output Connections:* Connects to the `ai_tool` input of `Find funding rounds`.
    - *Version-specific Requirements:* Version 1.4. Requires valid Bearer Auth credentials.
    - *Edge Cases / Potential Failure Types:* Authentication errors due to expired tokens, network timeouts, or malformed MCP endpoint responses.
  - **Enforce the round format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Sub-node / Output Parser)
    - *Configuration Choices:* Manual JSON schema definition ensuring strict adherence to required fields (`company`, `round_type`, `amount_usd`, `lead_investors`, `company_description`, `headline`, `publisher`, `announced`, `url`) and enumerating valid `round_type` values (`pre_seed`, `seed`, `series_a`, `series_b`, `series_c`, `series_d_plus`, `growth`, `strategic`, `undisclosed`).
    - *Key Expressions:* None.
    - *Input/Output Connections:* Connects to the `ai_outputParser` input of `Find funding rounds`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases / Potential Failure Types:* Validation failures if the language model returns data that violates the strict schema types or required fields.
  - **Report the failure**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (Error handling)
    - *Configuration Choices:* Configured to halt execution and output a readable error message when a sector run fails.
    - *Key Expressions:* Static error messaging pointing to credential or execution issues.
    - *Input/Output Connections:* Input from the error output of `Find funding rounds`.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases / Potential Failure Types:* Halts the entire workflow run upon encountering an unrecoverable extraction error in any single sector.

#### 2.3 Deduplication and Filtering
- **Overview:** Processes raw extraction outputs to filter out small funding rounds, prevent duplicate entries across sectors and historical runs using workflow static data, and compile structured alert texts.
- **Nodes Involved:** `Drop rounds already seen`, `Any new rounds?`
- **Node Details:**
  - **Drop rounds already seen**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Reads persistent workflow static data (`$getWorkflowStaticData('global')`), tracks seen company/round keys up to a maximum limit of 1,000 items, applies minimum funding thresholds (`minAmountUsd`), formats financial values, and aggregates items into sector-based groupings.
    - *Key Expressions:* Accesses global static data and settings from `Funding tracker settings`.
    - *Input/Output Connections:* Input from `Find funding rounds`; Output connects to `Any new rounds?`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Static data persistence is restricted to production environments; manual test executions do not persist state across runs.
  - **Any new rounds?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control)
    - *Configuration Choices:* Evaluates whether the boolean flag `hasNew` is true.
    - *Key Expressions:* `={{ $json.hasNew }}`
    - *Input/Output Connections:* Input from `Drop rounds already seen`; True branch connects to `Send the funding alert to Slack` and `Split into sheet rows`. False branch terminates without action.
    - *Version-specific Requirements:* Version 2.3.
    - *Edge Cases / Potential Failure Types:* Type mismatch if `hasNew` evaluates to a non-boolean value.

#### 2.4 Output Distribution
- **Overview:** Dispatches aggregated alert notifications to Slack and transforms batch records into individual rows for insertion into Google Sheets.
- **Nodes Involved:** `Send the funding alert to Slack`, `Split into sheet rows`, `Add rounds to the lead list`
- **Node Details:**
  - **Send the funding alert to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Action)
    - *Configuration Choices:* Posts a formatted text message to a designated channel name.
    - *Key Expressions:* `={{ $json.alertText }}` (message body) and `={{ $json.slackChannel }}` (target channel).
    - *Input/Output Connections:* Input from the true branch of `Any new rounds?`.
    - *Version-specific Requirements:* Version 2.7. Requires valid Slack bot credentials with `chat:write` and `channels:read` scopes.
    - *Edge Cases / Potential Failure Types:* Authentication errors, permission issues if the bot is not invited to the target channel, or API rate limits.
  - **Split into sheet rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration Choices:* Flattens the array of newly identified rounds into individual items, appending a timestamp (`found_at`) to each record.
    - *Key Expressions:* Reads `input.rounds` and returns individual row objects.
    - *Input/Output Connections:* Input from the true branch of `Any new rounds?`; Output connects to `Add rounds to the lead list`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Malformed or empty arrays will return empty outputs, preventing unnecessary downstream append operations.
  - **Add rounds to the lead list**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration Choices:* Appends rows using defined column mappings matching fields: `found_at`, `company`, `round_type`, `amount_usd`, `lead_investors`, `sector`, `company_description`, `headline`, `publisher`, `announced`, and `url`.
    - *Key Expressions:* Maps incoming JSON properties directly to sheet columns (e.g., `={{ $json.url }}`, `={{ $json.sector }}`, etc.).
    - *Input/Output Connections:* Input from `Split into sheet rows`.
    - *Version-specific Requirements:* Version 4.7. Requires Google OAuth2 or Service Account credentials, along with a pre-configured spreadsheet and sheet name.
    - *Edge Cases / Potential Failure Types:* Mismatched column headers, missing sheet access permissions, or API throttling by Google.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template overview | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ## Find startups raising money and build a funding lead list with Webz.io and GPT-4.1-mini... [Setup notes and full filter reference](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |
| Step 1 note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### 1. Runs every morning<br><br>Fires at 08:00 in the workflow timezone.<br><br>n8n defaults to UTC, so set **Workflow Settings → Timezone** before you trust the hour. |
| Step 2 note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### 2. Edit your settings here<br><br>Sectors and stages are comma-separated lists.<br><br>Lookback window, article count, minimum round size, and Slack channel live here too. |
| Step 3 note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### 3. One search per sector, structured output<br><br>**One item per sector** splits the sector list, and the agent runs once per sector.<br><br>It searches Webz.io news for funding announcements and the **Structured Output Parser** forces each round into company, round_type, amount_usd, lead_investors, and url. |
| Step 4 note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### 4. Dedupe, filter, fail loudly<br><br>**Drop rounds already seen** remembers company+round keys in workflow static data, filters below your minimum amount, and dedupes across sectors in the same run.<br><br>A failed sector run stops at **Report the failure** with a readable error. |
| Step 5 note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### 5. Alert, archive, or stay quiet<br><br>New rounds go to Slack and each round is appended to Google Sheets.<br><br>A quiet run sends nothing and writes no rows.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |
| Every morning at 08:00 | `n8n-nodes-base.scheduleTrigger` | Trigger daily execution | None | Funding tracker settings | |
| Funding tracker settings | `n8n-nodes-base.set` | Define configuration parameters | Every morning at 08:00 | One item per sector | |
| One item per sector | `n8n-nodes-base.code` | Fan out sectors into items | Funding tracker settings | Find funding rounds | |
| Find funding rounds | `@n8n/n8n-nodes-langchain.agent` | AI Agent for data extraction | One item per sector | Drop rounds already seen, Report the failure | |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM provider for AI agent | None | Find funding rounds | |
| Webz.io news search | `@n8n/n8n-nodes-langchain.mcpClientTool` | MCP tool for news retrieval | None | Find funding rounds | |
| Enforce the round format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structure extraction schema | None | Find funding rounds | |
| Report the failure | `n8n-nodes-base.stopAndError` | Halt on sector extraction error | Find funding rounds | None | |
| Drop rounds already seen | `n8n-nodes-base.code` | Deduplicate and filter records | Find funding rounds | Any new rounds? | |
| Any new rounds? | `n8n-nodes-base.if` | Check for new funding entries | Drop rounds already seen | Send the funding alert to Slack, Split into sheet rows | |
| Send the funding alert to Slack | `n8n-nodes-base.slack` | Post alerts to Slack channel | Any new rounds? | None | |
| Split into sheet rows | `n8n-nodes-base.code` | Flatten items for sheet insertion | Any new rounds? | Add rounds to the lead list | |
| Add rounds to the lead list | `n8n-nodes-base.googleSheets` | Append rows to Google Sheets | Split into sheet rows | None | |
| Sub-node note | `n8n-nodes-base.stickyNote` | Documentation / Reference | None | None | ### Model and tools<br><br>- **OpenAI Chat Model** drives the agent — add your OpenAI key, or swap it for any other tool-calling model<br>- **Webz.io news search** needs a *Bearer Auth* credential holding your Webz.io API token — leave *Tools to Include* set to **All**<br>- **Enforce the round format** guarantees every round arrives with company, round_type, amount_usd, lead_investors, and url |

---

### 4. Reproducing the Workflow from Scratch

1. **Configure Workflow Timezone:** Open workflow settings in n8n and set the target timezone (e.g., your local timezone) to ensure the schedule trigger fires accurately.
2. **Create Trigger:** Add a **Schedule Trigger** node named `Every morning at 08:00`. Configure its trigger rule to fire daily at hour 8.
3. **Create Settings Node:** Add a **Set** node named `Funding tracker settings`. Configure manual assignments for:
   - `sectors` (string): `AI, fintech, climate tech, cybersecurity`
   - `stages` (string): `seed, series A, series B, series C`
   - `minAmountUsd` (number): `1000000`
   - `lookbackDays` (number): `1`
   - `articleCount` (number): `15`
   - `slackChannel` (string): `#funding-alerts`
   - *Connection:* Connect `Every morning at 08:00` output to this node.
4. **Create Sector Fan-Out Node:** Add a **Code** node named `One item per sector`. Paste JavaScript logic to split the comma-separated sector string into individual items.
   - *Connection:* Connect `Funding tracker settings` output to this node.
5. **Create AI Agent & Sub-nodes:**
   - Add an **AI Agent** node named `Find funding rounds`. Set options to enable error output on failure (`onError: continueErrorOutput`), max tries to 2, and prompt type to define. Set the prompt text to instruct the agent to query the Webz.io tool based on input variables (`sector`, `stages`, `lookbackDays`, `articleCount`) and output structured JSON.
   - Add an **OpenAI Chat Model** sub-node, select model `gpt-4.1-mini`, and link it to the agent's `ai_languageModel` input. Configure valid OpenAI API credentials.
   - Add an **MCP Client Tool** node named `Webz.io news search`, set endpoint URL to `https://news-search-mcp.webz.io/mcp` with HTTP Streamable transport, and link it to the agent's `ai_tool` input. Configure Bearer Auth credentials with your Webz.io API token.
   - Add a **Structured Output Parser** sub-node named `Enforce the round format`. Configure its manual input schema to enforce an object containing `sector` (string) and `rounds` (array of objects with required properties: `company`, `round_type`, `amount_usd`, `lead_investors`, `company_description`, `headline`, `publisher`, `announced`, `url`). Link it to the agent's `ai_outputParser` input.
   - *Connection:* Connect `One item per sector` output to `Find funding rounds`.
6. **Create Error Handling Node:** Add a **Stop and Error** node named `Report the failure`.
   - *Connection:* Connect the error output (second output) of `Find funding rounds` to this node.
7. **Create Deduplication Node:** Add a **Code** node named `Drop rounds already seen`. Paste JavaScript logic to check global static data (`seenRounds`), filter rounds below `minAmountUsd`, deduplicate entries, format amounts and labels, and assemble a consolidated alert text.
   - *Connection:* Connect the main output (first output) of `Find funding rounds` to this node.
8. **Create Conditional Branching Node:** Add an **If** node named `Any new rounds?`. Configure a condition checking `={{ $json.hasNew }}` equals `true`.
   - *Connection:* Connect `Drop rounds already seen` output to this node.
9. **Create Slack Notification Node:** Add a **Slack** node named `Send the funding alert to Slack`. Set text to `={{ $json.alertText }}` and channel to `={{ $json.slackChannel }}`. Configure Slack credentials with a bot token having `chat:write` and `channels:read` scopes.
   - *Connection:* Connect the true branch of `Any new rounds?` to this node.
10. **Create Sheet Row Splitter Node:** Add a **Code** node named `Split into sheet rows`. Paste JavaScript logic to map the array of rounds into individual items with an ISO timestamp (`found_at`).
    - *Connection:* Connect the true branch of `Any new rounds?` to this node.
11. **Create Google Sheets Appender Node:** Add a **Google Sheets** node named `Add rounds to the lead list`. Set operation to `append`, configure Google OAuth2 or Service Account credentials, select the target spreadsheet and tab, and map columns (`found_at`, `company`, `round_type`, `amount_usd`, `lead_investors`, `sector`, `company_description`, `headline`, `publisher`, `announced`, `url`) to incoming values.
    - *Connection:* Connect `Split into sheet rows` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Webz.io API Token Registration | [webz.io](https://webz.io) |
| Setup notes and full filter reference | [GitHub Repository README](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |