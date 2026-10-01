Post a daily multi-topic news digest to Slack and Google Sheets with Webz.io and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/post-a-daily-multi-topic-news-digest-to-slack-and-google-sheets-with-webz-io-and-gpt-4-1-mini-20199


# Post a daily multi-topic news digest to Slack and Google Sheets with Webz.io and GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the collection, curation, formatting, and publishing of a daily multi-topic news digest. It is designed for analysts, researchers, communication teams, and stakeholders who track multiple subjects and prefer a centralized morning briefing rather than manual searches. 

The process executes daily at 08:00, queries global news sources via Webz.io, processes and filters data using OpenAI GPT-4.1-mini with strict output structuring, posts the final digest to a designated Slack channel, and archives individual stories to Google Sheets.

The logical execution is grouped into five functional blocks:
- **1.1 Input Reception & Fan-Out:** Initializes execution via a schedule trigger, sets configurable parameters (topics, lookback window, article count, target channel), and splits the configuration into individual execution items per topic.
- **1.2 AI Curation & Web Search:** Uses an AI agent powered by OpenAI and integrated with the Webz.io Model Context Protocol (MCP) search tool to retrieve and curate relevant articles per topic, enforcing a strict JSON schema via an output parser.
- **1.3 Data Aggregation & Deduplication:** Merges all topic streams, removes duplicate URLs across different topics, constructs a formatted Markdown digest message for Slack, and packages an archive dataset.
- **1.4 Conditional Evaluation:** Evaluates whether any valid stories were found across all topics to decide if publishing and archiving should proceed or terminate gracefully.
- **1.5 Distribution & Archiving:** Sends the formatted digest message to Slack and expands the dataset to append individual story rows into Google Sheets.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Fan-Out
- **Overview:** Triggers the workflow on a daily schedule, establishes the central parameters for the news query, and transforms a comma-separated list of topics into independent items for parallel or sequential processing.
- **Nodes Involved:** `Every morning at 08:00`, `Digest settings`, `One item per topic`.
- **Node Details:**
  - **Every morning at 08:00**
    - *Type and technical role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Fires the workflow automatically based on a temporal rule.
    - *Configuration choices:* Configured to trigger daily at hour 8 (08:00).
    - *Input/Output:* No inputs; outputs execution context to `Digest settings`.
    - *Edge cases/Failures:* Relies heavily on the n8n instance timezone configuration (`Workflow Settings → Timezone`). If misconfigured, it defaults to UTC.
  - **Digest settings**
    - *Type and technical role:* `n8n-nodes-base.set` (Data Transformation). Acts as a centralized configuration hub containing variables for search topics, lookback window, article count, and the Slack channel.
    - *Configuration choices:* Manual assignment mode with custom fields: `searchTopics` (string), `lookbackDays` (number), `articleCount` (number), and `slackChannel` (string).
    - *Input/Output:* Input from `Every morning at 08:00`; output to `One item per topic`.
    - *Edge cases/Failures:* Empty or incorrectly formatted topic strings will cause downstream errors in the splitting node.
  - **One item per topic**
    - *Type and technical role:* `n8n-nodes-base.code` (JavaScript Execution). Parses the comma-separated `searchTopics` string, trims whitespace, filters out empty values, and generates an individual output item per topic containing its lookback window and article count.
    - *Configuration choices:* Custom JavaScript snippet reading `$input.first().json`.
    - *Key expressions or variables:* `settings.searchTopics`, `settings.lookbackDays`, `settings.articleCount`.
    - *Input/Output:* Input from `Digest settings`; output to `Curate the top stories per topic`.
    - *Edge cases/Failures:* Throws a hard JavaScript error (`Digest settings has no topics...`) if zero valid topics are provided.

#### Block 1.2: AI Curation & Web Search
- **Overview:** Iterates through each topic item, queries the Webz.io news database via an MCP client tool, utilizes OpenAI GPT-4.1-mini to select significant articles, and coerces the response into a strict JSON schema.
- **Nodes Involved:** `Curate the top stories per topic`, `OpenAI Chat Model`, `Webz.io news search`, `Enforce the story format`, `Report the failure`.
- **Node Details:**
  - **Curate the top stories per topic**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent Node). Orchestrates LLM calls, tool invocation, and output parsing for each topic item.
    - *Configuration choices:* Defined prompt structure instructing the agent to query Webz.io precisely once per topic, request language filters, and select up to 5 significant stories. Error handling is set to continue execution on error output (`onError: continueErrorOutput`) with a maximum of 2 retries.
    - *Key expressions or variables:* `{{ $json.topic }}`, `{{ $json.lookbackDays }}`, `{{ $json.articleCount }}`.
    - *Input/Output:* Inputs from `One item per topic`, `OpenAI Chat Model`, `Webz.io news search`, and `Enforce the story format`. Primary output connects to `Merge topics and build the digest`; error output connects to `Report the failure`.
    - *Edge cases/Failures:* If network timeouts occur or credentials fail, execution routes to the error branch.
  - **OpenAI Chat Model**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Sub-node / Language Model). Provides the underlying LLM intelligence for the agent.
    - *Configuration choices:* Model set to `gpt-4.1-mini`.
    - *Input/Output:* Connects via `ai_languageModel` to `Curate the top stories per topic`.
    - *Edge cases/Failures:* Requires valid OpenAI API credentials. Exceeding rate limits will trigger retries or failures.
  - **Webz.io news search**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.mcpClientTool` (Sub-node / Tool). Exposes the external Webz.io news search MCP server as a callable tool for the AI agent.
    - *Configuration choices:* Endpoint URL configured to `https://news-search-mcp.webz.io/mcp`, using `bearerAuth` authentication over an `httpStreamable` server transport.
    - *Input/Output:* Connects via `ai_tool` to `Curate the top stories per topic`.
    - *Edge cases/Failures:* Invalid or expired Webz.io Bearer tokens will cause tool execution failure.
  - **Enforce the story format**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node / Output Parser). Forces the LLM response into a strict JSON object schema containing a `topic` string and a `stories` array with required fields (`headline`, `publisher`, `why_it_matters`, `url`).
    - *Configuration choices:* Manual JSON schema input defining object and array types.
    - *Input/Output:* Connects via `ai_outputParser` to `Curate the top stories per topic`.
    - *Edge cases/Failures:* If the LLM generates output that violates the strict schema definition, validation fails.
  - **Report the failure**
    - *Type and technical role:* `n8n-nodes-base.stopAndError` (Error Handling Node). Halts workflow execution when an agent run fails to return a valid digest section.
    - *Configuration choices:* Error type set to custom error message referencing credential checks.
    - *Input/Output:* Input from the secondary (error) output of `Curate the top stories per topic`.

#### Block 1.3: Data Aggregation & Deduplication
- **Overview:** Consolidates all curated topic outputs, strips out duplicate story URLs across different topics, builds structured text sections, and packages an array of individual stories for archival.
- **Nodes Involved:** `Merge topics and build the digest`.
- **Node Details:**
  - **Merge topics and build the digest**
    - *Type and technical role:* `n8n-nodes-base.code` (JavaScript Execution). Iterates through all incoming items, cleans string characters, tracks seen URLs using a JavaScript `Set` to prevent duplication, formats Slack Markdown sections, and builds a consolidated dataset.
    - *Configuration choices:* Custom JavaScript snippet reading `$input.all()` and pulling initial settings from `$('Digest settings')`.
    - *Key expressions or variables:* `item.json.output`, `settings.slackChannel`.
    - *Input/Output:* Input from `Curate the top stories per topic`; output to `Is there anything to post?`.
    - *Edge cases/Failures:* Malformed JSON structures in individual agent outputs are handled safely via empty array defaults (`|| []`).

#### Block 1.4: Conditional Evaluation
- **Overview:** Checks whether any stories were compiled across all topics before deciding whether to proceed with publishing and archiving.
- **Nodes Involved:** `Is there anything to post?`.
- **Node Details:**
  - **Is there anything to post?**
    - *Type and technical role:* `n8n-nodes-base.if` (Flow Control). Evaluates boolean conditions to branch workflow execution.
    - *Configuration choices:* Evaluates whether `{{ $json.hasStories }}` is true.
    - *Input/Output:* Input from `Merge topics and build the digest`. True branch outputs to `Post the digest to Slack` and `One row per story`; false branch terminates execution.
    - *Edge cases/Failures:* If zero stories are found across all topics, the workflow stops quietly without errors, preventing empty broadcasts.

#### Block 1.5: Distribution & Archiving
- **Overview:** Sends the compiled news digest message to the configured Slack channel and loops through individual stories to append structured rows into Google Sheets.
- **Nodes Involved:** `Post the digest to Slack`, `One row per story`, `Archive stories to Google Sheets`.
- **Node Details:**
  - **Post the digest to Slack**
    - *Type and technical role:* `n8n-nodes-base.slack` (Action Node). Posts a message to a specific Slack channel.
    - *Configuration choices:* Operation set to send text, selecting channel by name (`={{ $json.slackChannel }}`).
    - *Key expressions or variables:* `={{ $json.digestText }}`, `={{ $json.slackChannel }}`.
    - *Input/Output:* Input from the true branch of `Is there anything to post?`.
    - *Edge cases/Failures:* Requires valid Slack bot credentials with `chat:write` and `channels:read` scopes, and the bot must be explicitly invited to the target channel.
  - **One row per story**
    - *Type and technical role:* `n8n-nodes-base.splitOut` (Data Transformation). Expands an array of story objects into individual items for downstream processing.
    - *Configuration choices:* Field to split out set to `stories`.
    - *Input/Output:* Input from the true branch of `Is there anything to post?`; output to `Archive stories to Google Sheets`.
  - **Archive stories to Google Sheets**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` (Action Node). Appends rows of data to a specified Google Sheets document.
    - *Configuration choices:* Operation set to `append`. Mapped columns: `date`, `topic`, `headline`, `publisher`, `why_it_matters`, and `url`.
    - *Key expressions or variables:* `={{ $json.url }}`, `={{ $json.date }}`, `={{ $json.topic }}`, `={{ $json.headline }}`, `={{ $json.publisher }}`, `={{ $json.why_it_matters }}`.
    - *Input/Output:* Input from `One row per story`.
    - *Edge cases/Failures:* Requires valid Google Sheets OAuth2 credentials and a target sheet pre-configured with the exact required header names.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template overview | `n8n-nodes-base.stickyNote` | Documentation & setup reference | None | None | ## Post a daily multi-topic news digest to Slack and Google Sheets with Webz.io and GPT-4.1-mini<br><br>### Who's it for<br>Analysts, comms and PR teams, founders, and researchers who follow more than one topic and want a single sourced briefing every morning instead of a tab-hopping session.<br><br>### How it works<br>1. A schedule trigger fires every morning at 08:00.<br>2. **Digest settings** holds a comma-separated list of topics, the lookback window, article count, and Slack channel, so it is the only node you edit.<br>3. **One item per topic** fans the list out, and **Curate the top stories per topic** runs once per topic: it searches Webz.io global news through the MCP Client Tool and returns structured stories — headline, publisher, why it matters, URL — enforced by the Structured Output Parser.<br>4. **Merge topics and build the digest** removes duplicate URLs that appear under several topics and assembles one Slack message with a section per topic.<br>5. If every topic is quiet, nothing is posted. If a topic run fails, the workflow stops with a readable error instead of posting something misleading.<br>6. Every posted story is also appended to a Google Sheets archive, one row per story, so you keep a searchable history of what the digest covered.<br><br>### How to set up<br>1. On **Webz.io news search**, create a **Bearer Auth** credential and paste your Webz.io API token as the bearer token.<br>2. On **OpenAI Chat Model**, add your OpenAI credential.<br>3. On **Post the digest to Slack**, add a Slack credential whose bot token has `chat:write` and `channels:read`, then invite that bot to your channel.<br>4. On **Archive stories to Google Sheets**, connect your Google account, pick a spreadsheet and tab, and give the tab the headers `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url`.<br>5. Set your topics and channel in **Digest settings**.<br>6. Set **Workflow Settings → Timezone**, or 08:00 means 08:00 UTC.<br><br>### Requirements<br>- A Webz.io API token from [webz.io](https://webz.io)<br>- An OpenAI API key, or any other tool-calling chat model<br>- A Slack workspace where you can install a bot<br>- A Google account for the archive sheet<br><br>### How to customize<br>Add or remove topics in **Digest settings** — one workflow covers your whole beat. Swap **OpenAI Chat Model** for Anthropic, Gemini, or Ollama, or replace Slack with email or Teams. Remove the archive branch if you only want the Slack post.<br><br>[Setup notes and full filter reference](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |
| Step 1 note | `n8n-nodes-base.stickyNote` | Documentation reference | None | None | ### 1. Runs every morning<br><br>Fires at 08:00 in the workflow timezone.<br><br>n8n defaults to UTC, so set **Workflow Settings → Timezone** before you trust the hour. |
| Step 2 note | `n8n-nodes-base.stickyNote` | Documentation reference | None | None | ### 2. Edit your settings here<br><br>Topics are a comma-separated list — every topic becomes its own section in the digest.<br><br>Lookback window, article count, and Slack channel live here too. You should not need to touch the prompt. |
| Step 3 note | `n8n-nodes-base.stickyNote` | Documentation reference | None | None | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| Step 4 note | `n8n-nodes-base.stickyNote` | Documentation reference | None | None | ### 4. Merge, dedupe, fail loudly<br><br>**Merge topics and build the digest** drops stories that already appeared under another topic and builds one Slack message with a section per topic.<br><br>A broken credential makes the agent answer as though there was no news, so a failed topic run is routed to **Report the failure** and stops the workflow with a readable error instead. |
| Step 5 note | `n8n-nodes-base.stickyNote` | Documentation reference | None | None | ### 5. Post it, archive it, or stay quiet<br><br>If every topic is quiet, nothing is posted and nothing is archived.<br><br>Otherwise the digest goes to Slack and each story is appended to Google Sheets — `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url` — so you keep a searchable history.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |
| Every morning at 08:00 | `n8n-nodes-base.scheduleTrigger` | Workflow trigger | None | Digest settings | ### 1. Runs every morning<br><br>Fires at 08:00 in the workflow timezone.<br><br>n8n defaults to UTC, so set **Workflow Settings → Timezone** before you trust the hour. |
| Digest settings | `n8n-nodes-base.set` | Configuration repository | Every morning at 08:00 | One item per topic | ### 2. Edit your settings here<br><br>Topics are a comma-separated list — every topic becomes its own section in the digest.<br><br>Lookback window, article count, and Slack channel live here too. You should not need to touch the prompt. |
| One item per topic | `n8n-nodes-base.code` | Topic list splitter | Digest settings | Curate the top stories per topic | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| Curate the top stories per topic | `@n8n/n8n-nodes-langchain.agent` | AI curation agent | One item per topic, OpenAI Chat Model, Webz.io news search, Enforce the story format | Merge topics and build the digest, Report the failure | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM provider sub-node | None | Curate the top stories per topic | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| Webz.io news search | `@n8n/n8n-nodes-langchain.mcpClientTool` | Search MCP tool sub-node | None | Curate the top stories per topic | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| Enforce the story format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Output schema validator sub-node | None | Curate the top stories per topic | ### 3. One search per topic, structured output<br><br>**One item per topic** splits the topic list, and the agent runs once per topic.<br><br>It calls Webz.io global news search, picks up to 5 significant stories, and the **Structured Output Parser** forces each into `headline`, `publisher`, `why_it_matters`, `url` — no free-form text to break the nodes downstream.<br><br>**Credentials to add below**<br><br>- **Webz.io news search**: a *Bearer Auth* credential holding your Webz.io API token<br>- **OpenAI Chat Model**: your OpenAI key, or swap the node for any other tool-calling model<br><br>Leave *Tools to Include* set to **All** — the server exposes a single tool. |
| Report the failure | `n8n-nodes-base.stopAndError` | Error reporting terminator | Curate the top stories per topic | None | ### 4. Merge, dedupe, fail loudly<br><br>**Merge topics and build the digest** drops stories that already appeared under another topic and builds one Slack message with a section per topic.<br><br>A broken credential makes the agent answer as though there was no news, so a failed topic run is routed to **Report the failure** and stops the workflow with a readable error instead. |
| Merge topics and build the digest | `n8n-nodes-base.code` | Aggregator & deduplicator | Curate the top stories per topic | Is there anything to post? | ### 4. Merge, dedupe, fail loudly<br><br>**Merge topics and build the digest** drops stories that already appeared under another topic and builds one Slack message with a section per topic.<br><br>A broken credential makes the agent answer as though there was no news, so a failed topic run is routed to **Report the failure** and stops the workflow with a readable error instead. |
| Is there anything to post? | `n8n-nodes-base.if` | Conditional router | Merge topics and build the digest | Post the digest to Slack, One row per story | ### 5. Post it, archive it, or stay quiet<br><br>If every topic is quiet, nothing is posted and nothing is archived.<br><br>Otherwise the digest goes to Slack and each story is appended to Google Sheets — `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url` — so you keep a searchable history.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |
| Post the digest to Slack | `n8n-nodes-base.slack` | Slack publisher | Is there anything to post? | None | ### 5. Post it, archive it, or stay quiet<br><br>If every topic is quiet, nothing is posted and nothing is archived.<br><br>Otherwise the digest goes to Slack and each story is appended to Google Sheets — `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url` — so you keep a searchable history.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |
| One row per story | `n8n-nodes-base.splitOut` | Story array expander | Is there anything to post? | Archive stories to Google Sheets | ### 5. Post it, archive it, or stay quiet<br><br>If every topic is quiet, nothing is posted and nothing is archived.<br><br>Otherwise the digest goes to Slack and each story is appended to Google Sheets — `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url` — so you keep a searchable history.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |
| Archive stories to Google Sheets | `n8n-nodes-base.googleSheets` | Spreadsheet archiver | One row per story | None | ### 5. Post it, archive it, or stay quiet<br><br>If every topic is quiet, nothing is posted and nothing is archived.<br><br>Otherwise the digest goes to Slack and each story is appended to Google Sheets — `date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url` — so you keep a searchable history.<br><br>Slack needs a bot token with `chat:write` and `channels:read`, and the bot has to be invited to the channel. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Configure Workflow Timezone:** Open Workflow Settings and set your local timezone (defaults to UTC).
2. **Create Schedule Trigger:** Add a **Schedule Trigger** node named `Every morning at 08:00`. Configure interval rules to trigger daily at hour `8`.
3. **Create Settings Node:** Add a **Set** node named `Digest settings`. In manual mode, add four assignments:
   - `searchTopics` (String): `"artificial intelligence policy and regulation, semiconductor supply chain"`
   - `lookbackDays` (Number): `1`
   - `articleCount` (Number): `10`
   - `slackChannel` (String): `"#news-digest"`
   Connect `Every morning at 08:00` to `Digest settings`.
4. **Create Topic Splitter:** Add a **Code** node named `One item per topic`. Paste JavaScript logic to split `searchTopics` by commas and output individual items. Connect `Digest settings` to `One item per topic`.
5. **Create AI Agent:** Add an **Advanced AI Agent** node named `Curate the top stories per topic`. 
   - Set Prompt Type to `Define`.
   - Set Text prompt to query Webz.io using `{{ $json.topic }}`, `{{ $json.lookbackDays }}`, and `{{ $json.articleCount }}`.
   - Configure error handling (`Continue On Fail` to error output). Set max retries to `2`.
   - Connect `One item per topic` to the main input of the agent.
6. **Configure AI Sub-Nodes:**
   - Add an **OpenAI Chat Model** node named `OpenAI Chat Model` configured with model `gpt-4.1-mini`. Connect its `ai_languageModel` output to the agent.
   - Add an **MCP Client Tool** node named `Webz.io news search`. Set Endpoint URL to `https://news-search-mcp.webz.io/mcp`, Authentication to `Bearer Auth`, and Transport to `HTTP Streamable`. Connect its `ai_tool` output to the agent.
   - Add a **Structured Output Parser** node named `Enforce the story format`. Define a manual JSON schema requiring `topic` (string) and `stories` array (containing objects with `headline`, `publisher`, `why_it_matters`, `url`). Connect its `ai_outputParser` output to the agent.
7. **Create Error Terminator:** Add a **Stop and Error** node named `Report the failure`. Connect the secondary error output of `Curate the top stories per topic` to this node.
8. **Create Aggregator & Deduplicator:** Add a **Code** node named `Merge topics and build the digest`. Add JavaScript to deduplicate URLs across topics, format Markdown sections, and compile `hasStories`, `digestText`, `stories`, and `slackChannel`. Connect the primary output of `Curate the top stories per topic` to this node.
9. **Create Conditional Branch:** Add an **If** node named `Is there anything to post?`. Set condition to evaluate `={{ $json.hasStories }}` as true. Connect `Merge topics and build the digest` to this node.
10. **Create Slack Publisher:** Add a **Slack** node named `Post the digest to Slack`. Set operation to send text, channel selection mode to name, and value to `={{ $json.slackChannel }}` with message text `={{ $json.digestText }}`. Connect the `true` output branch of `Is there anything to post?` here.
11. **Create Story Splitter:** Add a **Split Out** node named `One row per story`. Set field to split out to `stories`. Connect the `true` output branch of `Is there anything to post?` here as well.
12. **Create Google Sheets Archiver:** Add a **Google Sheets** node named `Archive stories to Google Sheets`. Set operation to `append`. Map columns (`date`, `topic`, `headline`, `publisher`, `why_it_matters`, `url`) to corresponding expression variables (`={{ $json.date }}`, etc.). Connect `One row per story` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Webz.io API Token Source | [webz.io](https://webz.io) |
| Setup notes and full filter reference | [GitHub Repository README](https://github.com/Webhose/webz-news-search/blob/main/n8n/README.md) |