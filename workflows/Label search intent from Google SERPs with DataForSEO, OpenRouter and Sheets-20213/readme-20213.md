Label search intent from Google SERPs with DataForSEO, OpenRouter and Sheets

https://n8nworkflows.xyz/workflows/label-search-intent-from-google-serps-with-dataforseo--openrouter-and-sheets-20213


# Label search intent from Google SERPs with DataForSEO, OpenRouter and Sheets

### 1. Workflow Overview

This workflow automates the analysis of search engine result pages (SERPs) to evaluate user search intent, content format, and competitiveness for a given list of keywords. Designed for SEO specialists and content strategists, the automation eliminates manual SERP research by programmatically retrieving top organic results, evaluating competing page content, fetching search volume history, and leveraging a large language model to categorize search intents.

The execution logic is structured into four primary functional blocks:

- **1.1 Input Reception & Configuration:** Manually triggers the execution, sets global operational parameters (such as the OpenRouter model choice, spreadsheet URL, processing thresholds, and the master system prompt), and retrieves pending keywords from a Google Sheets document.
- **1.2 Data Collection (DataForSEO Integration):** Iterates through keywords to extract the live Google top 10 organic results, People Also Ask items, and AI Overviews. Depending on active configuration flags, it additionally parses competing pages into markdown, gathers multi-year search volume and seasonality history, and fetches estimated organic traffic per URL.
- **1.3 AI Processing & Statistical Computation:** Sends clean page data, titles, and snippets (excluding numeric metrics) to an OpenRouter LLM via a structured Basic LLM Chain to classify results into intent clusters. Subsequent JavaScript code nodes validate the model output and compute mathematical metrics such as coverage ratios, split traffic shares, reference word lengths, and SEO warnings.
- **1.4 Reporting & State Management:** Formats the computed analytical outputs and appends them across four structured Google Sheets tabs (`Summary`, `Intents`, `Results`, and updates the processing status within `Keywords`).

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes workflow parameters, establishes the execution configuration, and reads unprocessed keywords from the source Google Sheet.
- **Nodes Involved:** 
  - `Run on pending keywords`
  - `Config`
  - `Read Keywords tab`
  - `Pending keywords`

- **Node Details:**
  - **Run on pending keywords**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger) — Acts as the entry point for manual workflow execution.
    - *Configuration Choices:* Standard default configuration (no parameters required).
    - *Input/Output:* Output connects to the `Config` node.
    - *Edge Cases/Failures:* None (manual trigger).
  - **Config**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Set / Edit Fields) — Defines global configuration variables including the Google Sheets URL, OpenRouter model name (`openai/gpt-6-luna`), feature switches (`read_pages`, `search_volume`, `traffic`), maximum keyword limit, and the detailed `system_prompt`.
    - *Configuration Choices:* String, boolean, and numeric assignments.
    - *Input/Output:* Input from `Run on pending keywords`; output connects to `Read Keywords tab`.
    - *Edge Cases/Failures:* Ensure the spreadsheet URL placeholder is replaced with a valid document URL.
  - **Read Keywords tab**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Reads rows from the `Keywords` worksheet of the configured spreadsheet.
    - *Configuration Choices:* Operation set to `read`, document ID mapped dynamically via `{{ $('Config').first().json.spreadsheet_url }}`, and sheet name set to `Keywords`.
    - *Credentials:* Google Sheets OAuth2 API (`gSheets roman@ibb.media`).
    - *Input/Output:* Input from `Config`; output connects to `Pending keywords`.
    - *Edge Cases/Failures:* Authentication token expiry or missing sheet tabs will trigger standard API errors.
  - **Pending keywords**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code / JavaScript) — Filters rows where the `status` column is empty, translates numerical country codes and language tags into human-readable locations/languages, and caps the batch size using `max_keywords`.
    - *Configuration Choices:* Custom JavaScript using mapping and filtering arrays.
    - *Input/Output:* Input from `Read Keywords tab`; output connects to `Loop over keywords`.
    - *Edge Cases/Failures:* Malformed sheet data or missing columns may cause undefined property errors; handled via safe optional chaining (`??`).

---

#### 2.2 Data Collection (DataForSEO Integration)
- **Overview:** Iterates over the queue of pending keywords to query DataForSEO for live organic SERP items, optional web page markdown parsing, keyword search volume trends, and URL-level traffic estimations.
- **Nodes Involved:**
  - `Loop over keywords`
  - `All keywords processed`
  - `DataForSEO: Google top 10`
  - `Parse SERP`
  - `Read pages?`
  - `Split pages`
  - `DataForSEO: read page`
  - `Attach pages`
  - `Search volume?`
  - `DataForSEO: search volume`
  - `Attach volume`
  - `Traffic?`
  - `DataForSEO: traffic per URL`
  - `Attach traffic`

- **Node Details:**
  - **Loop over keywords**
    - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` (Split In Batches) — Loops through filtered keywords one by one.
    - *Configuration Choices:* Default batch settings.
    - *Input/Output:* Input from `Pending keywords`; outputs connect to `All keywords processed` (when complete) and `DataForSEO: Google top 10`.
  - **All keywords processed**
    - *Type & Technical Role:* `n8n-nodes-base.noOp` (No Operation) — Terminal node for the loop when all keywords are evaluated.
  - **DataForSEO: Google top 10**
    - *Type & Technical Role:* `n8n-nodes-dataforseo.dataForSeoSerpApi` (DataForSEO SERP API) — Fetches live Google organic search results (depth 10) along with People Also Ask, related searches, and AI Overviews.
    - *Configuration Choices:* Operation: `get-live-google-organic-serp-advanced`; parameters mapped to keyword, location name, and language name.
    - *Credentials:* DataForSEO API credential (`Czc1BI4E8eGKmkQJ`).
    - *Input/Output:* Input from `Loop over keywords`; output connects to `Parse SERP`.
    - *Edge Cases/Failures:* API rate limits, insufficient account balance, or unsupported location names.
  - **Parse SERP**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Extracts top 10 organic items, truncates descriptions, collects PAA/related queries, summarizes AI Overviews, and tracks DataForSEO costs.
    - *Input/Output:* Input from `DataForSEO: Google top 10`; output connects to `Read pages?`.
  - **Read pages?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (If) — Evaluates whether page content fetching is enabled (`read_pages === true`) and if valid search results exist.
    - *Input/Output:* Input from `Parse SERP`; true branch connects to `Split pages`, false branch bypasses to `Search volume?`.
  - **Split pages**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Splits result URLs into individual items to allow parallel page parsing calls.
    - *Input/Output:* Input from `Read pages?` (true branch); output connects to `DataForSEO: read page`.
  - **DataForSEO: read page**
    - *Type & Technical Role:* `n8n-nodes-dataforseo.dataForSeoOnPageApi` (DataForSEO OnPage API) — Parses target ranking pages into clean markdown view.
    - *Configuration Choices:* Operation: `get-live-parsed-content`; `markdown_view: true`. Error handling set to `continueRegularOutput`.
    - *Credentials:* DataForSEO API credential (`Czc1BI4E8eGKmkQJ`).
    - *Input/Output:* Input from `Split pages`; output connects to `Attach pages`.
    - *Edge Cases/Failures:* Target sites blocking scraping or returning 404/500 errors; handled by regular output continuation.
  - **Attach pages**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Analyzes markdown content (word counts, tables, lists, images, videos, headings, and paragraphs) to build a 400-character content digest. Flags pages under 150 words as thin.
    - *Input/Output:* Input from `DataForSEO: read page`; output connects to `Search volume?`.
  - **Search volume?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (If) — Checks if search volume gathering is enabled (`search_volume === true`).
    - *Input/Output:* Input from `Read pages?` (false branch) or `Attach pages`; true branch connects to `DataForSEO: search volume`, false branch bypasses to `Traffic?`.
  - **DataForSEO: search volume**
    - *Type & Technical Role:* `n8n-nodes-dataforseo.dataForSeoLabsApi` (DataForSEO Labs API) — Retrieves search volume, CPC, keyword difficulty, and historical monthly search data.
    - *Configuration Choices:* Operation: `get-keyword-overview`. Error handling set to `continueRegularOutput`.
    - *Credentials:* DataForSEO API credential (`Czc1BI4E8eGKmkQJ`).
    - *Input/Output:* Input from `Search volume?`; output connects to `Attach volume`.
  - **Attach volume**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Calculates arithmetic seasonality indices (peak vs. low month ratios) and year-over-year trends from multi-year monthly search data.
    - *Input/Output:* Input from `DataForSEO: search volume`; output connects to `Traffic?`.
  - **Traffic?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (If) — Checks if traffic estimation is enabled (`traffic === true`).
    - *Input/Output:* Input from `Search volume?` (false branch) or `Attach volume`; true branch connects to `DataForSEO: traffic per URL`, false branch bypasses to `Has results?`.
  - **DataForSEO: traffic per URL**
    - *Type & Technical Role:* `n8n-nodes-dataforseo.dataForSeoLabsApi` (DataForSEO Labs API) — Fetches bulk organic traffic estimations (ETV) for ranking target URLs.
    - *Configuration Choices:* Operation: `get-bulk-traffic-estimation`. Error handling set to `continueRegularOutput`.
    - *Credentials:* DataForSEO API credential (`Czc1BI4E8eGKmkQJ`).
    - *Input/Output:* Input from `Traffic?`; output connects to `Attach traffic`.
  - **Attach traffic**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Maps retrieved organic traffic metrics to respective result IDs, defaulting unindexed URLs to `null` rather than zero.
    - *Input/Output:* Input from `DataForSEO: traffic per URL`; output connects to `Has results?`.

---

#### 2.3 AI Processing & Statistical Computation
- **Overview:** Prepares the prompt payloads, groups search results into intents using an OpenRouter LLM, validates the structural integrity of the output, and calculates aggregate metrics, shares, and SEO warning flags.
- **Nodes Involved:**
  - `Has results?`
  - `Build prompt`
  - `Labeler: group into intents`
  - `OpenRouter Chat Model`
  - `Validate and count`

- **Node Details:**
  - **Has results?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (If) — Verifies that organic results exist and no SERP errors occurred.
    - *Input/Output:* Input from `Traffic?` (false branch) or `Attach traffic`; true branch connects to `Build prompt`, false branch connects to `Build sheet rows`.
  - **Build prompt**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Constructs the JSON payload containing SERP features, titles, snippets, and page digests (omitting numeric data) to be consumed by the LLM.
    - *Input/Output:* Input from `Has results?` (true branch); output connects to `Labeler: group into intents`.
  - **Labeler: group into intents**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Basic LLM Chain) — Sends the structured prompt using the system instructions to group results into intent clusters.
    - *Configuration Choices:* Prompt type: Define; Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `Build prompt`; AI Model input connected to `OpenRouter Chat Model`; output connects to `Validate and count`.
  - **OpenRouter Chat Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (OpenRouter Chat Model) — Configures the LLM provider and model endpoint.
    - *Configuration Choices:* Model: `openai/gpt-6-luna` (dynamic via Config); Max tokens: 12,000; Response format: `json_object`.
    - *Credentials:* OpenRouter API credential (`openrouter nimblio`).
    - *Input/Output:* Connects to `Labeler: group into intents` via AI language model connection.
  - **Validate and count**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Validates JSON response syntax and intent structures, handles unassigned fallback results, computes coverage ratios, split shares, traffic shares, median word counts for dominant intent pages, and flags potential SEO risks (e.g., mixed SERPs, non-article intent).
    - *Input/Output:* Input from `Labeler: group into intents`; output connects to `Build sheet rows`.

---

#### 2.4 Reporting & State Management
- **Overview:** Transforms processed data objects into flat tabular rows and appends them across multiple Google Sheets worksheets while updating the keyword processing status.
- **Nodes Involved:**
  - `Build sheet rows`
  - `Summary row`
  - `Write Summary`
  - `Intent rows`
  - `Write Intents`
  - `Result rows`
  - `Write Results`
  - `Keyword status`
  - `Mark keyword as done`

- **Node Details:**
  - **Build sheet rows**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Consolidates keyword execution state, decision metrics, costs, and token usage into structured rows mapped to Summary, Intents, Results, and keyword update schemas.
    - *Input/Output:* Input from `Validate and count` or `Has results?` (false branch); output connects to `Summary row`.
  - **Summary row**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Extracts the summary row object.
    - *Configuration Choices:* `executeOnce: true`.
    - *Input/Output:* Input from `Build sheet rows`; output connects to `Write Summary`.
  - **Write Summary**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Appends the summary record to the `Summary` sheet tab.
    - *Configuration Choices:* Operation: `append`; Sheet: `Summary`.
    - *Credentials:* Google Sheets OAuth2 API (`gSheets roman@ibb.media`).
    - *Input/Output:* Input from `Summary row`; output connects to `Intent rows`.
  - **Intent rows**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Maps individual intent clusters into separate row items.
    - *Configuration Choices:* `executeOnce: true`.
    - *Input/Output:* Input from `Write Summary`; output connects to `Write Intents`.
  - **Write Intents**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Appends intent records to the `Intents` sheet tab.
    - *Configuration Choices:* Operation: `append`; Sheet: `Intents`.
    - *Credentials:* Google Sheets OAuth2 API (`gSheets roman@ibb.media`).
    - *Input/Output:* Input from `Intent rows`; output connects to `Result rows`.
  - **Result rows**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Maps individual SERP result entries into individual row items.
    - *Configuration Choices:* `executeOnce: true`.
    - *Input/Output:* Input from `Write Intents`; output connects to `Write Results`.
  - **Write Results**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Appends result records to the `Results` sheet tab.
    - *Configuration Choices:* Operation: `append`; Sheet: `Results`.
    - *Credentials:* Google Sheets OAuth2 API (`gSheets roman@ibb.media`).
    - *Input/Output:* Input from `Result rows`; output connects to `Keyword status`.
  - **Keyword status**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Code) — Prepares the final status update object for the processed keyword row (`done` or `error`, timestamp, total cost).
    - *Configuration Choices:* `executeOnce: true`.
    - *Input/Output:* Input from `Write Results`; output connects to `Mark keyword as done`.
  - **Mark keyword as done**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Updates the specific keyword row in the `Keywords` tab using `row_number` as the matching column.
    - *Configuration Choices:* Operation: `update`; Matching columns: `row_number`; Sheet: `Keywords`.
    - *Credentials:* Google Sheets OAuth2 API (`gSheets roman@ibb.media`).
    - *Input/Output:* Input from `Keyword status`; output loops back to `Loop over keywords` to process the next item.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run on pending keywords` | `manualTrigger` | Triggers workflow execution manually | None | `Config` | Try it out! / Setup (5 minutes) |
| `Config` | `set` | Defines configuration variables and prompts | `Run on pending keywords` | `Read Keywords tab` | Try it out! / Setup (5 minutes) |
| `Read Keywords tab` | `googleSheets` | Reads rows from the Keywords tab | `Config` | `Pending keywords` | 1. Read pending keywords from Google Sheets |
| `Pending keywords` | `code` | Filters pending keywords and translates codes | `Read Keywords tab` | `Loop over keywords` | 1. Read pending keywords from Google Sheets |
| `Loop over keywords` | `splitInBatches` | Iterates over keywords sequentially | `Pending keywords` | `All keywords processed`, `DataForSEO: Google top 10` | 1. Read pending keywords from Google Sheets |
| `All keywords processed` | `noOp` | Terminal node when queue completes | `Loop over keywords` | None | 1. Read pending keywords from Google Sheets |
| `DataForSEO: Google top 10` | `dataForSeoSerpApi` | Fetches live Google SERP top 10 data | `Loop over keywords` | `Parse SERP` | 2. Collect SERP, page and demand data from DataForSEO |
| `Parse SERP` | `code` | Parses organic results, PAA, and AI Overview | `DataForSEO: Google top 10` | `Read pages?` | 2. Collect SERP, page and demand data from DataForSEO |
| `Read pages?` | `if` | Checks if page parsing is enabled | `Parse SERP` | `Split pages`, `Search volume?` | 2. Collect SERP, page and demand data from DataForSEO |
| `Split pages` | `code` | Splits result URLs for parallel parsing | `Read pages?` | `DataForSEO: read page` | 2. Collect SERP, page and demand data from DataForSEO |
| `DataForSEO: read page` | `dataForSeoOnPageApi` | Parses ranking web pages into markdown | `Split pages` | `Attach pages` | 2. Collect SERP, page and demand data from DataForSEO |
| `Attach pages` | `code` | Analyzes page markdown and builds digests | `DataForSEO: read page` | `Search volume?` | 2. Collect SERP, page and demand data from DataForSEO |
| `Search volume?` | `if` | Checks if search volume gathering is enabled | `Read pages?`, `Attach pages` | `DataForSEO: search volume`, `Traffic?` | 2. Collect SERP, page and demand data from DataForSEO |
| `DataForSEO: search volume` | `dataForSeoLabsApi` | Fetches historical search volume & seasonality | `Search volume?` | `Attach volume` | 2. Collect SERP, page and demand data from DataForSEO |
| `Attach volume` | `code` | Calculates seasonality and trend metrics | `DataForSEO: search volume` | `Traffic?` | 2. Collect SERP, page and demand data from DataForSEO |
| `Traffic?` | `if` | Checks if traffic estimation is enabled | `Search volume?`, `Attach volume` | `DataForSEO: traffic per URL`, `Has results?` | 2. Collect SERP, page and demand data from DataForSEO |
| `DataForSEO: traffic per URL` | `dataForSeoLabsApi` | Fetches bulk organic traffic estimates per URL | `Traffic?` | `Attach traffic` | 2. Collect SERP, page and demand data from DataForSEO |
| `Attach traffic` | `code` | Maps traffic estimates to result objects | `DataForSEO: traffic per URL` | `Has results?` | 2. Collect SERP, page and demand data from DataForSEO |
| `Has results?` | `if` | Validates that results exist without errors | `Traffic?`, `Attach traffic` | `Build prompt`, `Build sheet rows` | 3. Group with the Labeler (Basic LLM Chain), count with code |
| `Build prompt` | `code` | Prepares prompt payload for the LLM | `Has results?` | `Labeler: group into intents` | 3. Group with the Labeler (Basic LLM Chain), count with code |
| `Labeler: group into intents` | `chainLlm` | Basic LLM chain for intent categorization | `Build prompt` | `Validate and count` | 3. Group with the Labeler (Basic LLM Chain), count with code |
| `OpenRouter Chat Model` | `lmChatOpenRouter` | Configures OpenRouter chat model endpoint | None | `Labeler: group into intents` | 3. Group with the Labeler (Basic LLM Chain), count with code |
| `Validate and count` | `code` | Validates model output and computes SEO metrics | `Labeler: group into intents` | `Build sheet rows` | 3. Group with the Labeler (Basic LLM Chain), count with code |
| `Build sheet rows` | `code` | Formats data into rows for spreadsheet tabs | `Has results?`, `Validate and count` | `Summary row` | 4. Write results back to Google Sheets |
| `Summary row` | `code` | Extracts summary table row data | `Build sheet rows` | `Write Summary` | 4. Write results back to Google Sheets |
| `Write Summary` | `googleSheets` | Appends record to Summary tab | `Summary row` | `Intent rows` | 4. Write results back to Google Sheets |
| `Intent rows` | `code` | Extracts intent table row items | `Write Summary` | `Write Intents` | 4. Write results back to Google Sheets |
| `Write Intents` | `googleSheets` | Appends records to Intents tab | `Intent rows` | `Result rows` | 4. Write results back to Google Sheets |
| `Result rows` | `code` | Extracts result table row items | `Write Intents` | `Write Results` | 4. Write results back to Google Sheets |
| `Write Results` | `googleSheets` | Appends records to Results tab | `Result rows` | `Keyword status` | 4. Write results back to Google Sheets |
| `Keyword status` | `code` | Prepares keyword completion status update | `Write Results` | `Mark keyword as done` | 4. Write results back to Google Sheets |
| `Mark keyword as done` | `googleSheets` | Updates keyword row in Keywords tab | `Keyword status` | `Loop over keywords` | 4. Write results back to Google Sheets |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Trigger & Configuration:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Set** (`n8n-nodes-base.set`) node named `Config`. Define assignments:
     - `spreadsheet_url` (String): Your target Google Sheet URL.
     - `model` (String): `openai/gpt-6-luna` (or preferred OpenRouter model).
     - `read_pages` (Boolean): `true`
     - `search_volume` (Boolean): `true`
     - `traffic` (Boolean): `true`
     - `max_keywords` (Number): `10`
     - `system_prompt` (String): Paste the comprehensive prompt instructing the LLM to act as a SERP analyst, group search intents, and return strict JSON objects.
   - Connect `Run on pending keywords` to `Config`.

2. **Set up Google Sheet Reading & Queue Logic:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Read Keywords tab`. Set operation to `read`, document ID to `{{ $('Config').first().json.spreadsheet_url }}`, and sheet name to `Keywords`. Configure Google Sheets OAuth2 credentials.
   - Add a **Code** node named `Pending keywords`. Implement filtering for empty status fields, translation dictionaries for country/language codes, and limit slices.
   - Add a **Split In Batches** node (`n8n-nodes-base.splitInBatches`) named `Loop over keywords`. Add a **No Operation** node named `All keywords processed`.
   - Connect: `Config` ➔ `Read Keywords tab` ➔ `Pending keywords` ➔ `Loop over keywords`. Connect loop item output to `All keywords processed` and data output to the next step.

3. **Build the DataCollection Pipeline (DataForSEO):**
   - Add a **DataForSEO SERP API** node (`n8n-nodes-dataforseo.dataForSeoSerpApi`) named `DataForSEO: Google top 10`. Set operation to `get-live-google-organic-serp-advanced`, depth to `10`, and map expressions for keyword, location name, and language name. Configure DataForSEO API credentials.
   - Add a **Code** node named `Parse SERP` to extract organic items, PAA, and AI Overviews.
   - Add an **If** node named `Read pages?` checking `{{ $('Config').first().json.read_pages === true && !$json.error && $json.results.length > 0 }}`.
   - For the true branch, add a **Code** node named `Split pages` and connect it to a **DataForSEO OnPage API** node (`n8n-nodes-dataforseo.dataForSeoOnPageApi`) named `DataForSEO: read page` (operation: `get-live-parsed-content`, markdown view enabled, error handling set to continue regular output). Follow with a **Code** node named `Attach pages`.
   - Add an **If** node named `Search volume?` checking `{{ $('Config').first().json.search_volume === true && !$json.error && $json.results.length > 0 }}`.
   - If true, connect to a **DataForSEO Labs API** node (`n8n-nodes-dataforseo.dataForSeoLabsApi`) named `DataForSEO: search volume` (operation: `get-keyword-overview`), followed by a **Code** node named `Attach volume`.
   - Add an **If** node named `Traffic?` checking `{{ $('Config').first().json.traffic === true && !$json.error && $json.results.length > 0 }}`.
   - If true, connect to a **DataForSEO Labs API** node named `DataForSEO: traffic per URL` (operation: `get-bulk-traffic-estimation`), followed by a **Code** node named `Attach traffic`.

4. **Configure AI Intent Grouping & Validation:**
   - Add an **If** node named `Has results?` checking `{{ !$json.error && $json.results.length > 0 }}`.
   - Add a **Code** node named `Build prompt` to prepare the user message payload.
   - Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) named `Labeler: group into intents`.
   - Add an **OpenRouter Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) named `OpenRouter Chat Model` (model: `openai/gpt-6-luna`, max tokens: 12,000, response format: `json_object`). Configure OpenRouter API credentials and connect it to the LLM chain.
   - Add a **Code** node named `Validate and count` to parse JSON, compute coverage, split shares, traffic shares, median word lengths, and generate SEO warnings.

5. **Configure Writing & State Management Nodes:**
   - Add a **Code** node named `Build sheet rows` to format data objects for Summary, Intents, Results, and keyword status updates.
   - Create a sequence of alternating **Code** and **Google Sheets** (append operation) nodes:
     - `Summary row` ➔ `Write Summary` (Sheet: `Summary`)
     - `Intent rows` ➔ `Write Intents` (Sheet: `Intents`)
     - `Result rows` ➔ `Write Results` (Sheet: `Results`)
     - `Keyword status` ➔ `Mark keyword as done` (Google Sheets update operation, matching column: `row_number`, Sheet: `Keywords`).
   - Connect the final `Mark keyword as done` node back to the input of `Loop over keywords` to process the next keyword in the batch.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **DataForSEO Platform Access** | [DataForSEO Dashboard & API Access](https://skq.pl/data4seo) |
| **OpenRouter Structured Outputs Guide** | [OpenRouter Documentation](https://openrouter.ai/docs/features/structured-outputs) |
| **Google Sheets Node Documentation** | [n8n Google Sheets Integration Guide](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/) |
| **Open-Source Intent Labeler Reference** | [GitHub Repository & Browser Playground](https://romek-rozen.github.io/intent-labeler/) |
| **Example Report ("seo agency london")** | [Live Report Example](https://romek-rozen.github.io/intent-labeler/examples/seo-agency-london.html) |