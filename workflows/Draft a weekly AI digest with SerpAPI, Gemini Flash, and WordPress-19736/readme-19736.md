Draft a weekly AI digest with SerpAPI, Gemini Flash, and WordPress

https://n8nworkflows.xyz/workflows/draft-a-weekly-ai-digest-with-serpapi--gemini-flash--and-wordpress-19736


# Draft a weekly AI digest with SerpAPI, Gemini Flash, and WordPress

### 1. Workflow Overview

This workflow automates the creation of a weekly AI digest by combining scheduled execution, deterministic web searches via SerpAPI, structured AI synthesis using Google Gemini, safety checks, and content management integration with WordPress. It runs automatically every Friday, retrieves recent articles, verifies grounding against source URLs, and creates an unpublished draft post for editorial review.

The workflow logic is divided into five functional blocks:
- **1.1 Trigger and Configuration:** Handles the weekly schedule initiation and centralizes search queries, freshness limits, audience definitions, and character caps.
- **1.2 Search and Collation:** Executes sequential Google searches via SerpAPI, scopes results to the past week, flattens, and de-duplicates URLs into a clean citable findings list.
- **1.3 AI Processing and Structuring:** Supplies the compiled findings to Google Gemini Flash using a strict system prompt and an output parser to enforce a structured JSON response.
- **1.4 Transformation and CMS Ingestion:** Enforces output character bounds, validates the presence of body text, converts Markdown output into HTML, and creates a draft post in WordPress.
- **1.5 Annotations and Guidelines:** Provides architectural notes and operational guidance across multiple visual sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Configuration
- **Overview:** Initializes the schedule on a weekly basis and defines runtime variables used across all downstream nodes.
- **Nodes Involved:** `Every Friday 07:00`, `Config`
- **Node Details:**
  - **Every Friday 07:00** (`n8n-nodes-base.scheduleTrigger`)
    - *Technical Role:* Cron-based schedule trigger firing once a week.
    - *Configuration:* Triggers on day 5 (Friday) at hour 07:00.
    - *Input/Output:* No inputs; outputs an execution payload.
    - *Failure Handling:* None required.
  - **Config** (`n8n-nodes-base.code`)
    - *Technical Role:* Centralizes workflow parameters and search settings.
    - *Configuration:* Returns a JavaScript object (`CONFIG`) containing `queryOne`, `queryTwo`, `freshness` (`qdr:w`), `resultsPerSearch` (10), `audience`, `titleMaxChars` (60), and `excerptMaxChars` (155).
    - *Input/Output:* Input from `Every Friday 07:00`; output connects to `Search One`.
    - *Failure Handling:* Script execution error if syntax fails.

#### 1.2 Search and Collation
- **Overview:** Performs two distinct Google searches using SerpAPI within a weekly freshness window, then merges and de-duplicates the results.
- **Nodes Involved:** `Search One`, `Search Two`, `Collate Findings`
- **Node Details:**
  - **Search One** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Executes the primary SerpAPI Google Search query.
    - *Configuration:* GET request to `https://serpapi.com/search.json` using Predefined Credentials (`serpApi`). Query parameters evaluate `engine=google`, `q={{ $json.queryOne }}`, `tbs={{ $json.freshness }}`, and `num={{ $json.resultsPerSearch }}`. Retry configured for 3 attempts with 5-second intervals on failure.
    - *Input/Output:* Input from `Config`; output connects to `Search Two`.
    - *Failure Handling:* `retryOnFail` active with 3 attempts (`maxTries: 3`, `waitBetweenTries: 5000`). Fails on persistent API authentication or quota errors.
  - **Search Two** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Executes the secondary SerpAPI Google Search query.
    - *Configuration:* Identical endpoint and credential structure as Search One, but queries `queryTwo`. `executeOnce` is enabled to prevent duplicate runs per incoming item in chained execution.
    - *Input/Output:* Input from `Search One`; output connects to `Collate Findings`.
    - *Failure Handling:* `retryOnFail` active with 3 attempts (`maxTries: 3`, `waitBetweenTries: 5000`).
  - **Collate Findings** (`n8n-nodes-base.code`)
    - *Technical Role:* Flattens organic search results from both search nodes, filters out invalid entries, de-duplicates by URL, and numbers the items into a structured text block.
    - *Configuration:* JavaScript processing block referencing `Search One` and `Search Two` outputs. Limits results to top 8 items per search and outputs a combined string `findings` alongside a `count` integer.
    - *Input/Output:* Input from `Search Two`; output connects to `Write Digest`.
    - *Failure Handling:* Script throws errors if data mapping structures change unexpectedly.

#### 1.3 AI Processing and Structuring
- **Overview:** Sends the collated search results to Google Gemini Flash with strict system constraints and parses the response into structured fields.
- **Nodes Involved:** `Digest Model`, `Digest Shape`, `Write Digest`
- **Node Details:**
  - **Digest Model** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Technical Role:* Language model provider configuration for Gemini.
    - *Configuration:* Uses model `models/gemini-3.6-flash` with a low temperature of `0.5` for summarization accuracy.
    - *Input/Output:* Connects to `Write Digest` via `ai_languageModel`.
    - *Failure Handling:* Dependent on Google AI API availability and rate limits.
  - **Digest Shape** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Technical Role:* Enforces a rigid JSON schema output format on the language model response.
    - *Configuration:* Schema mandates keys: `title` (string), `excerpt` (string), `body` (string), and `tags` (array of strings).
    - *Input/Output:* Connects to `Write Digest` via `ai_outputParser`.
    - *Failure Handling:* Output validation errors if model output drifts from JSON schema.
  - **Write Digest** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Technical Role:* Core LLM chain executing the synthesis prompt.
    - *Configuration:* System prompt establishes audience constraints, citation requirements, formatting rules, and voice guidelines. User prompt incorporates current date (`$now`), search result count, and collated findings. Retry configured for 3 attempts.
    - *Input/Output:* Inputs from `Collate Findings` (main), `Digest Model` (ai_languageModel), and `Digest Shape` (ai_outputParser); output connects to `Enforce Output Limits`.
    - *Failure Handling:* `retryOnFail` enabled (`maxTries: 3`, `waitBetweenTries: 5000`).

#### 1.4 Transformation and CMS Ingestion
- **Overview:** Validates model output, trims titles and excerpts to character limits, converts Markdown to HTML, and generates a WordPress draft post.
- **Nodes Involved:** `Enforce Output Limits`, `Markdown To HTML`, `Create Draft`
- **Node Details:**
  - **Enforce Output Limits** (`n8n-nodes-base.code`)
    - *Technical Role:* Arithmetic validation and string length enforcement node.
    - *Configuration:* JavaScript code checks output parsing formats, clips `title` to `cfg.titleMaxChars` and `excerpt` to `cfg.excerptMaxChars` on word boundaries, normalizes `tags` to lowercase, and throws an explicit error if the model returns an empty body.
    - *Input/Output:* Input from `Write Digest`; output connects to `Markdown To HTML`.
    - *Failure Handling:* Explicitly halts execution by throwing an error if `body` is empty.
  - **Markdown To HTML** (`n8n-nodes-base.markdown`)
    - *Technical Role:* Converts Markdown content into HTML format compatible with CMS requirements.
    - *Configuration:* Translates `={{ $json.body }}` using options: tables enabled, GitHub-code blocks enabled, simplified auto-links enabled, and links set to open in new windows.
    - *Input/Output:* Input from `Enforce Output Limits`; output connects to `Create Draft`.
    - *Failure Handling:* Markdown parsing exceptions on malformed syntax.
  - **Create Draft** (`n8n-nodes-base.wordpress`)
    - *Technical Role:* Creates a draft post on a WordPress site via Basic Authentication.
    - *Configuration:* Uses Predefined Credentials (`wordpressBasicAuth`). Sets resource to `post`, operation to `create`, status to `draft`, title to `={{ $('Enforce Output Limits').first().json.title }}`, and content to `={{ $json.html }}`.
    - *Input/Output:* Input from `Markdown To HTML`; no downstream output.
    - *Failure Handling:* `retryOnFail` enabled (`maxTries: 3`, `waitBetweenTries: 5000`). Fails on invalid credentials or REST API blocks.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every Friday 07:00 | n8n-nodes-base.scheduleTrigger | Triggers workflow execution weekly on Fridays at 07:00. | None | Config | # Weekly AI digest, with sources it actually retrieved<br><br>Every Friday this searches the web for the past seven days, writes a digest that can **only** cite the links it was given, and files it as a **draft** for a person to review. Nothing is ever published automatically.<br><br>**Setup, about five minutes:** add a **SerpAPI** credential to both search nodes · a **chat model** credential to *Digest Model* · a **WordPress** credential to *Create Draft* · then edit the two queries in **Config**. That node is the only thing you need to change.<br><br>**What it costs to run:** 2 search calls and 1 model call per successful run — 8 searches a month, or 24 in the worst case where every call exhausts its 3 retries. Both sit inside SerpAPI’s free 100. The trigger fires 07:00 in **your instance’s timezone**; set that in Settings, or change the trigger.<br>## 1 · Trigger and settings<br><br>Fires once a week. Weekly rather than daily is deliberate: a daily digest of a field that moves weekly is a digest of noise.<br><br>**Config is the only node you edit.** It holds both search queries, the freshness window, how many results to pull, and one line describing who reads the digest — that line lands in the system prompt and is what keeps the output from reading like a press release.<br><br>Everything downstream reads its settings from here, so there is no second place to remember. |
| Config | n8n-nodes-base.code | Centralizes configuration parameters for search queries and limits. | Every Friday 07:00 | Search One | ## 1 · Trigger and settings<br><br>Fires once a week. Weekly rather than daily is deliberate: a daily digest of a field that moves weekly is a digest of noise.<br><br>**Config is the only node you edit.** It holds both search queries, the freshness window, how many results to pull, and one line describing who reads the digest — that line lands in the system prompt and is what keeps the output from reading like a press release.<br><br>Everything downstream reads its settings from here, so there is no second place to remember. |
| Search One | n8n-nodes-base.httpRequest | Executes first Google search query via SerpAPI. | Config | Search Two | ## 2 · Two searches, fixed<br><br>Two plain HTTP calls to SerpAPI. Keep the queries genuinely different from each other — two phrasings of the same question return the same links and waste a call.<br><br>`tbs=qdr:w` scopes results to the past week deterministically, so the model never has to reason about what "this week" means.<br><br>**`executeOnce` is on for Search Two.** Chained after Search One, it would otherwise run once per incoming item.<br><br>Both retry 3× on failure, so a transient 5xx does not cost you the week. |
| Search Two | n8n-nodes-base.httpRequest | Executes second Google search query via SerpAPI. | Search One | Collate Findings | ## 2 · Two searches, fixed<br><br>Two plain HTTP calls to SerpAPI. Keep the queries genuinely different from each other — two phrasings of the same question return the same links and waste a call.<br><br>`tbs=qdr:w` scopes results to the past week deterministically, so the model never has to reason about what "this week" means.<br><br>**`executeOnce` is on for Search Two.** Chained after Search One, it would otherwise run once per incoming item.<br><br>Both retry 3× on failure, so a transient 5xx does not cost you the week. |
| Collate Findings | n8n-nodes-base.code | Flattens, filters, and de-duplicates search results. | Search Two | Write Digest | ## 3 · Collate, then write once<br><br>Collate flattens both result sets, drops duplicate URLs and numbers them into one citable block. The same link listed twice reads to a model as two corroborating sources.<br><br>**Why this is a chain and not an agent.** The obvious build gives an agent a search tool and lets it decide what to look for. That cost one model call per iteration and took seven of them — impossible against a five-a-minute limit.<br><br>Grounding got **stronger**, not weaker: with findings pasted into the prompt, the model can only cite URLs in front of it. |
| Digest Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Configures Google Gemini Flash chat model provider. | None | Write Digest | ### Any chat model works<br><br>Gemini Flash is here because this pattern was built to survive a free tier. Swap in OpenAI, Anthropic or a local model and nothing else changes.<br><br>Keep the temperature low. This is a summarising job, not a creative one. |
| Digest Shape | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces a structured JSON output schema for the LLM. | None | Write Digest | |
| Write Digest | @n8n/n8n-nodes-langchain.chainLlm | Generates source-grounded AI digest via LLM chain. | Collate Findings, Digest Model, Digest Shape | Enforce Output Limits | |
| Enforce Output Limits | n8n-nodes-base.code | Trims title and excerpt to character boundaries and validates body content. | Write Digest | Markdown To HTML | ## 4 · Enforce the limits, then draft<br><br>Asking a model to count characters is asking the wrong tool, and it fails silently — you find out when the search result is already truncated. **Enforce Output Limits** trims the title and excerpt on a word boundary instead. It also refuses to draft an empty post: a failed execution is legible, an empty draft in the CMS looks like a decision somebody made.<br><br>Markdown To HTML exists because WordPress wants HTML, not markdown.<br><br>**Swapping the destination:** replace the last node only. Everything upstream is CMS-agnostic and hands on a clean `title`, `excerpt`, `body` and `tags` — use Notion, Ghost, or an HTTP Request to your own API.<br><br>**On tags:** the WordPress node wants term *IDs*, not names, so the model’s tags are not attached. Add them at review time. |
| Markdown To HTML | n8n-nodes-base.markdown | Converts Markdown body text to HTML. | Enforce Output Limits | Create Draft | ## 4 · Enforce the limits, then draft<br><br>Asking a model to count characters is asking the wrong tool, and it fails silently — you find out when the search result is already truncated. **Enforce Output Limits** trims the title and excerpt on a word boundary instead. It also refuses to draft an empty post: a failed execution is legible, an empty draft in the CMS looks like a decision somebody made.<br><br>Markdown To HTML exists because WordPress wants HTML, not markdown.<br><br>**Swapping the destination:** replace the last node only. Everything upstream is CMS-agnostic and hands on a clean `title`, `excerpt`, `body` and `tags` — use Notion, Ghost, or an HTTP Request to your own API.<br><br>**On tags:** the WordPress node wants term *IDs*, not names, so the model’s tags are not attached. Add them at review time. |
| Create Draft | n8n-nodes-base.wordpress | Creates a draft post in WordPress using basic authentication. | Markdown To HTML | None | ## 4 · Enforce the limits, then draft<br><br>Asking a model to count characters is asking the wrong tool, and it fails silently — you find out when the search result is already truncated. **Enforce Output Limits** trims the title and excerpt on a word boundary instead. It also refuses to draft an empty post: a failed execution is legible, an empty draft in the CMS looks like a decision somebody made.<br><br>Markdown To HTML exists because WordPress wants HTML, not markdown.<br><br>**Swapping the destination:** replace the last node only. Everything upstream is CMS-agnostic and hands on a clean `title`, `excerpt`, `body` and `tags` — use Notion, Ghost, or an HTTP Request to your own API.<br><br>**On tags:** the WordPress node wants term *IDs*, not names, so the model’s tags are not attached. Add them at review time. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to trigger weekly on day `5` (Friday) at hour `7`.
   - Name the node: `Every Friday 07:00`.

2. **Create Configuration Code Node:**
   - Add a Code node (`n8n-nodes-base.code`).
   - Name it: `Config`.
   - Connect `Every Friday 07:00` output to `Config`.
   - Insert the JavaScript code defining `CONFIG` with `queryOne`, `queryTwo`, `freshness` (`qdr:w`), `resultsPerSearch` (10), `audience`, `titleMaxChars` (60), and `excerptMaxChars` (155).

3. **Create Search One HTTP Request Node:**
   - Add an HTTP Request node (`n8n-nodes-base.httpRequest`).
   - Name it: `Search One`.
   - Connect `Config` output to `Search One`.
   - Configure method as `GET`, URL as `https://serpapi.com/search.json`.
   - Set Authentication to `Predefined Credential Type` using credential type `serpApi`.
   - Add query parameters: `engine` = `google`, `q` = `={{ $json.queryOne }}`, `tbs` = `={{ $json.freshness }}`, `num` = `={{ $json.resultsPerSearch }}`.
   - Enable retry on failure (`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000`).

4. **Create Search Two HTTP Request Node:**
   - Add an HTTP Request node (`n8n-nodes-base.httpRequest`).
   - Name it: `Search Two`.
   - Connect `Search One` output to `Search Two`.
   - Configure identically to `Search One`, except set `q` parameter to `={{ $('Config').first().json.queryTwo }}` and enable `executeOnce`.
   - Enable retry on failure (`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000`).

5. **Create Collate Findings Code Node:**
   - Add a Code node (`n8n-nodes-base.code`).
   - Name it: `Collate Findings`.
   - Connect `Search Two` output to `Collate Findings`.
   - Insert JavaScript to pull results from both search nodes, slice to 8 items, de-duplicate by URL, and format into a citable numbered string block.

6. **Create AI Model and Output Parser Nodes:**
   - Add a Google Gemini Chat Model sub-node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`). Set model name to `models/gemini-3.6-flash` and temperature to `0.5`. Assign valid Google AI credentials.
   - Add a Structured Output Parser sub-node (`@n8n/n8n-nodes-langchain.outputParserStructured`). Provide a JSON schema example with keys: `title`, `excerpt`, `body`, and `tags`.

7. **Create Basic LLM Chain Node:**
   - Add a Basic LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`).
   - Name it: `Write Digest`.
   - Connect `Collate Findings` to the main input. Connect `Digest Model` to `ai_languageModel` and `Digest Shape` to `ai_outputParser`.
   - Configure system message prompt with audience constraints and formatting instructions. Configure user prompt with current date (`$now`) and search findings payload. Enable retry on failure (`maxTries: 3`, `waitBetweenTries: 5000`).

8. **Create Limit Enforcement Code Node:**
   - Add a Code node (`n8n-nodes-base.code`).
   - Name it: `Enforce Output Limits`.
   - Connect `Write Digest` output to `Enforce Output Limits`.
   - Insert JavaScript to parse model response, clip `title` to 60 characters and `excerpt` to 155 characters on word boundaries, normalize tags, and throw an error if body content is empty.

9. **Create Markdown to HTML Conversion Node:**
   - Add a Markdown node (`n8n-nodes-base.markdown`).
   - Name it: `Markdown To HTML`.
   - Connect `Enforce Output Limits` output to `Markdown To HTML`.
   - Set mode to `markdownToHtml`, input markdown to `={{ $json.body }}`, destination key to `html`, and enable options: tables, GitHub code blocks, simplified auto-links, and open links in new window.

10. **Create WordPress Draft Creation Node:**
    - Add a WordPress node (`n8n-nodes-base.wordpress`).
    - Name it: `Create Draft`.
    - Connect `Markdown To HTML` output to `Create Draft`.
    - Set authentication type to `Basic Auth` using valid WordPress credentials.
    - Set resource to `post`, operation to `create`, status to `draft`, title to `={{ $('Enforce Output Limits').first().json.title }}`, and content to `={{ $json.html }}`.
    - Enable retry on failure (`maxTries: 3`, `waitBetweenTries: 5000`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Weekly AI digest with real sources to a WordPress draft | Workflow Title / Core Purpose |
| SerpAPI Google Search Endpoint Documentation | https://serpapi.com/search-api |
| Google Gemini API Documentation | https://ai.google.dev/ |
| WordPress REST API Posts Reference | https://developer.wordpress.org/rest-api/reference/posts/ |