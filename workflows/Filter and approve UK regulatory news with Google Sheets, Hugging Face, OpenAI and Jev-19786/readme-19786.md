Filter and approve UK regulatory news with Google Sheets, Hugging Face, OpenAI and Jev

https://n8nworkflows.xyz/workflows/filter-and-approve-uk-regulatory-news-with-google-sheets--hugging-face--openai-and-jev-19786


# Filter and approve UK regulatory news with Google Sheets, Hugging Face, OpenAI and Jev

### 1. Workflow Overview

This workflow is an automated regulatory intelligence pipeline designed to ingest, filter, deduplicate, score, and approve UK financial services regulatory news. It ingests articles from Google Sheets and optional live RSS feeds (such as the FCA press releases), eliminates noise using deterministic keyword and source weights, clusters similar stories using semantic embeddings, and performs deep evaluation using Firecrawl and TypeSafe (JEV) structured relevance scoring. Finally, it records approved packages and updates an execution ledger back to Google Sheets.

The workflow logic is categorized into the following functional blocks:
- **1.1 Initialization and Configuration:** Sets execution triggers, context variables, and optional database cleanup gates.
- **1.2 Data Ingestion and Source Harmonization:** Pulls records from Google Sheets and optionally fetches live RSS feeds, merging them into a uniform schema.
- **1.3 Ledger Deduplication and Pending Tracking:** Checks incoming URLs against a historical ledger, filters duplicates, and marks new entries as pending.
- **1.4 Deterministic Rule-Based Filtering:** Scores articles based on keyword density and source authority, dropping irrelevant noise.
- **1.5 Semantic Embedding and Clustering:** Generates vector embeddings (via Hugging Face or OpenAI) and groups near-duplicate articles into clusters, selecting a primary representative per cluster.
- **1.6 Deep Content Scraping and Structured JEV Evaluation:** Scrapes primary articles via Firecrawl and passes them to TypeSafe JEV for multi-dimensional relevance scoring and quality classification.
- **1.7 Composite Scoring, Routing, and Ledger Finalization:** Computes a weighted final score, routes approved stories to output sheets, and logs execution outcomes and statuses back to the seen-article ledger.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Configuration
- **Overview:** Establishes manual or webhook entry points, initializes pipeline parameters, and optionally clears historical Google Sheets tabs for clean demo runs.
- **Nodes Involved:** `Run the pipeline`, `Test hook (temporary)`, `Configure context`, `Start fresh this run?`, `Clear previous packages`, `Clear seen ledger`, `Continue after clear gate`.
- **Node Details:**
  - **Run the pipeline** (`n8n-nodes-base.manualTrigger`):
    - *Role:* Manual workflow entry point.
    - *Configuration:* Default execution version 1.
    - *Connections:* Outputs to `Configure context`.
    - *Edge Cases:* Manual triggers do not pass external payload parameters; relies entirely on downstream default context.
  - **Test hook (temporary)** (`n8n-nodes-base.webhook`):
    - *Role:* Alternative webhook trigger path for incoming requests.
    - *Configuration:* Path set to `5d5cb8ed-aa6e-47f3-be90-94ce89f004d6`.
    - *Connections:* Outputs to `Configure context`.
    - *Edge Cases:* Potential unauthorized access if webhook endpoint is publicly exposed without signature verification.
  - **Configure context** (`n8n-nodes-base.set`):
    - *Role:* Central configuration store holding spreadsheet IDs, tab names, scoring weights, thresholds, and niche definitions.
    - *Configuration:* Assignments object storing parameters such as `sheetDocumentId`, `rulesKeywords`, `sourceWeights`, `rulesRejectThreshold`, `rulesHardWinThreshold`, and `embeddingProvider`.
    - *Connections:* Inputs from triggers; outputs to `Start fresh this run?` and `Fetch live articles too?`.
    - *Edge Cases:* Missing or incorrectly formatted JSON maps in configuration assignments can cause downstream parser failures.
  - **Start fresh this run?** (`n8n-nodes-base.if`):
    - *Role:* Evaluates whether to clear previous outputs and ledger records.
    - *Configuration:* Condition checks if `{{ $('Configure context').first().json.clearBeforeRun }}` equals `true`.
    - *Connections:* Input from `Configure context`; true branch outputs to `Clear previous packages`, false branch outputs to `Continue after clear gate`.
  - **Clear previous packages** (`n8n-nodes-base.googleSheets`):
    - *Role:* Clears the approved package sheet tab while retaining the header row.
    - *Configuration:* Operation `clear`, document ID and sheet name derived from context, `keepFirstRow: true`.
    - *Connections:* Input from `Start fresh this run?` (true); outputs to `Clear seen ledger`.
    - *Edge Cases:* API rate limits or invalid Google Sheets credentials will halt execution.
  - **Clear seen ledger** (`n8n-nodes-base.googleSheets`):
    - *Role:* Clears the historical seen articles ledger tab.
    - *Configuration:* Operation `clear`, keeping the first header row.
    - *Connections:* Input from `Clear previous packages`; outputs to `Continue after clear gate`.
  - **Continue after clear gate** (`n8n-nodes-base.merge`):
    - *Role:* Re-joins the execution flow following the conditional cleanup gate.
    - *Configuration:* Default merge settings.
    - *Connections:* Inputs from `Start fresh this run?` (false) and `Clear seen ledger`; outputs to `Read source articles` and `Read seen ledger`.

---

#### 1.2 Data Ingestion and Source Harmonization
- **Overview:** Pulls baseline source records from Google Sheets and optionally scrapes live RSS feeds, transforming all items into a unified schema.
- **Nodes Involved:** `Fetch live articles too?`, `Live fetch (FCA Press Release)`, `Map live articles to dataset shape`, `Read source articles`, `Merge sheet and live articles`.
- **Node Details:**
  - **Fetch live articles too?** (`n8n-nodes-base.if`):
    - *Role:* Gatekeeper to determine whether to execute live RSS fetching.
    - *Configuration:* Checks if `{{ $('Configure context').first().json.useLiveFetch }}` equals `true`.
    - *Connections:* Input from `Configure context`; true branch outputs to `Live fetch (FCA Press Release)`.
  - **Live fetch (FCA Press Release)** (`n8n-nodes-base.rssFeedRead`):
    - *Role:* Ingests live news items from the FCA RSS feed.
    - *Configuration:* URL set to `https://www.fca.org.uk/rss.xml`.
    - *Connections:* Input from `Fetch live articles too?`; outputs to `Map live articles to dataset shape`.
    - *Edge Cases:* RSS feed downtime, network timeouts, or malformed XML structures.
  - **Map live articles to dataset shape** (`n8n-nodes-base.set`):
    - *Role:* Normalizes raw RSS feed properties into the pipeline's standard article dataset shape.
    - *Configuration:* Maps `title`, `summary`, `source` (hardcoded to 'FCA'), `url` (from link), `published_at`, and `content_excerpt`.
    - *Connections:* Input from `Live fetch (FCA Press Release)`; outputs to `Merge sheet and live articles`.
  - **Read source articles** (`n8n-nodes-base.googleSheets`):
    - *Role:* Loads manually or externally provided source articles from the configured Google Sheets tab.
    - *Configuration:* Operation `get`, document ID and sheet name dynamically resolved from context.
    - *Connections:* Input from `Continue after clear gate`; outputs to `Merge sheet and live articles`.
  - **Merge sheet and live articles** (`n8n-nodes-base.merge`):
    - *Role:* Combines sheet-based articles and live RSS articles into a single dataset.
    - *Configuration:* Default append/combine mode.
    - *Connections:* Inputs from `Read source articles` and `Map live articles to dataset shape`; outputs to `Drop already-seen URLs`.

---

#### 1.3 Ledger Deduplication and Pending Tracking
- **Overview:** Compares incoming article URLs against the historical seen ledger, drops previously processed items, and records new URLs with a pending status.
- **Nodes Involved:** `Read seen ledger`, `Drop already-seen URLs`, `Stamp pending status`, `Record in seen ledger`.
- **Node Details:**
  - **Read seen ledger** (`n8n-nodes-base.googleSheets`):
    - *Role:* Fetches historical records from the seen-articles Google Sheets ledger.
    - *Configuration:* Operation `get`, `alwaysOutputData: true` enabled to prevent workflow failure if the sheet is empty.
    - *Connections:* Input from `Continue after clear gate`; outputs to `Drop already-seen URLs`.
  - **Drop already-seen URLs** (`n8n-nodes-base.merge`):
    - *Role:* Filters out incoming articles whose URLs already exist in the seen ledger.
    - *Configuration:* Mode `combine`, join mode `keepNonMatches`, fields to match set to `url`.
    - *Connections:* Inputs from `Merge sheet and live articles` (Input 1) and `Read seen ledger` (Input 2); outputs to `Stamp pending status`.
  - **Stamp pending status** (`n8n-nodes-base.set`):
    - *Role:* Assigns initial metadata (`status: pending`, execution ID as `run_id`, empty score/reason placeholders) to newly discovered articles.
    - *Configuration:* Assignments include `status`, `rules_score`, `reason`, `cluster_id`, and `run_id` (`{{ $execution.id }}`).
    - *Connections:* Input from `Drop already-seen URLs`; outputs to `Record in seen ledger`.
  - **Record in seen ledger** (`n8n-nodes-base.googleSheets`):
    - *Role:* Persists the newly discovered pending URLs into the Google Sheets ledger before processing.
    - *Configuration:* Operation `append`, mapping fields including `id`, `url`, `title`, `status`, and `run_id`.
    - *Connections:* Input from `Stamp pending status`; outputs to `Score with rules (Step 1)`.
    - *Edge Cases:* Write conflicts or sheet quota limitations.

---

#### 1.4 Deterministic Rule-Based Filtering
- **Overview:** Evaluates article text against keyword dictionaries and source authority weighting to calculate an initial rules score, instantly dropping obvious noise.
- **Nodes Involved:** `Score with rules (Step 1)`, `Drop rejected noise`.
- **Node Details:**
  - **Score with rules (Step 1)** (`n8n-nodes-base.code`):
    - *Role:* Executes deterministic keyword and source authority scoring using custom JavaScript.
    - *Configuration:* Iterates input items, matches regex word boundaries against titles (double weight) and summaries/excerpts, adds source weights, and assigns a rule band (`pass`, `reject`, or `hardWin`).
    - *Connections:* Input from `Record in seen ledger`; outputs to `Drop rejected noise`.
    - *Edge Cases:* Unhandled regex syntax characters in dynamic keywords; memory allocation issues with extremely large item batches.
  - **Drop rejected noise** (`n8n-nodes-base.filter`):
    - *Role:* Discards articles falling below the rejection score threshold.
    - *Configuration:* Condition verifies `{{ $json.rulesBand }}` does not equal `reject`.
    - *Connections:* Input from `Score with rules (Step 1)`; outputs to `Collect batch for embedding`.

---

#### 1.5 Semantic Embedding and Clustering
- **Overview:** Batches surviving articles, generates vector embeddings via Hugging Face or OpenAI, and clusters near-duplicate stories based on cosine similarity.
- **Nodes Involved:** `Collect batch for embedding`, `Choose embedding provider`, `Embed with Hugging Face`, `Embed with OpenAI`, `Normalize HF vectors`, `Normalize OpenAI vectors`, `Cluster stories (Step 2)`.
- **Node Details:**
  - **Collect batch for embedding** (`n8n-nodes-base.aggregate`):
    - *Role:* Aggregates filtered article items into a single array batch for vectorization.
    - *Configuration:* Aggregate mode set to `aggregateAllItemData`.
    - *Connections:* Input from `Drop rejected noise`; outputs to `Choose embedding provider`.
  - **Choose embedding provider** (`n8n-nodes-base.switch`):
    - *Role:* Routes batch execution depending on the configured embedding provider (`huggingface` or `openai`).
    - *Configuration:* Switch rules evaluating `{{ $('Configure context').first().json.embeddingProvider }}`.
    - *Connections:* Input from `Collect batch for embedding`; branch 1 outputs to `Embed with Hugging Face`, branch 2 outputs to `Embed with OpenAI`.
  - **Embed with Hugging Face** (`n8n-nodes-base.httpRequest`):
    - *Role:* Sends article texts to the Hugging Face Inference API for feature extraction.
    - *Configuration:* POST request to router endpoint using `huggingFaceApi` credentials.
    - *Connections:* Input from `Choose embedding provider` (Branch 1); outputs to `Normalize HF vectors`.
    - *Edge Cases:* API rate limits, model loading latency (`503 Service Unavailable`), or token expiration.
  - **Embed with OpenAI** (`n8n-nodes-base.httpRequest`):
    - *Role:* Sends article texts to the OpenAI Embeddings API.
    - *Configuration:* POST request to `https://api.openai.com/v1/embeddings` using `openAiApi` credentials.
    - *Connections:* Input from `Choose embedding provider` (Branch 2); outputs to `Normalize OpenAI vectors`.
    - *Edge Cases:* Rate limits (`429 Too Many Requests`), billing quota exhaustion, or payload size limits.
  - **Normalize HF vectors** (`n8n-nodes-base.set`):
    - *Role:* Standardizes Hugging Face response structure into a uniform array of vectors.
    - *Configuration:* Assigns `vectors` from `{{ $json.body }}`.
    - *Connections:* Input from `Embed with Hugging Face`; outputs to `Cluster stories (Step 2)`.
  - **Normalize OpenAI vectors** (`n8n-nodes-base.set`):
    - *Role:* Standardizes OpenAI response structure into a uniform array of vectors.
    - *Configuration:* Assigns `vectors` by mapping `{{ $json.body.data.map(entry => entry.embedding) }}`.
    - *Connections:* Input from `Embed with OpenAI`; outputs to `Cluster stories (Step 2)`.
  - **Cluster stories (Step 2)** (`n8n-nodes-base.code`):
    - *Role:* Calculates pairwise cosine similarities between article vectors to group duplicates into distinct story clusters.
    - *Configuration:* Custom JavaScript computes cosine matrix, groups items by `clusterSimilarityThreshold`, and selects a primary article per cluster (highest rule score, longest excerpt on ties).
    - *Connections:* Inputs from `Normalize HF vectors` or `Normalize OpenAI vectors` (plus reference data from `Drop rejected noise`); outputs to `Needs LLM judgment?`.
    - *Edge Cases:* Vector array length mismatch throws explicit runtime error.

---

#### 1.6 Deep Content Scraping and Structured JEV Evaluation
- **Overview:** Scrapes full text content for non-hard-win primary stories via Firecrawl and sends it to TypeSafe JEV for structured multi-dimensional evaluation.
- **Nodes Involved:** `Needs LLM judgment?`, `Get article with firecrawl`, `Set error`, `Judge relevance with Jev`.
- **Node Details:**
  - **Needs LLM judgment?** (`n8n-nodes-base.if`):
    - *Role:* Bypasses LLM judgment for articles meeting the `hardWin` threshold, forwarding only ambiguous primary stories to scraping and judgment.
    - *Configuration:* Evaluates `isPrimary === true` and `rulesBand !== hardWin`.
    - *Connections:* Input from `Cluster stories (Step 2)`; true branch outputs to `Get article with firecrawl`, false branch routes to `Merge`.
  - **Get article with firecrawl** (`n8n-nodes-base.httpRequest`):
    - *Role:* Scrapes full markdown article body from the target URL.
    - *Configuration:* POST request to `https://api.firecrawl.dev/v2/scrape` using `httpBearerAuth` credentials, with batching and retry logic enabled.
    - *Connections:* Input from `Needs LLM judgment?` (true); success outputs to `Judge relevance with Jev`, error output (`continueErrorOutput`) connects to `Set error`.
    - *Edge Cases:* Target site blocking web scrapers (`SCRAPE_ALL_ENGINES_FAILED`), HTTP 404/500 errors, or rate limits.
  - **Set error** (`n8n-nodes-base.set`):
    - *Role:* Captures scraping failure exceptions gracefully to prevent workflow crashes.
    - *Configuration:* Assigns `title`, `url`, and error message from `{{ $json.error }}`.
    - *Connections:* Input from error branch of `Get article with firecrawl`.
  - **Judge relevance with Jev** (`n8n-nodes-base.httpRequest`):
    - *Role:* Submits article markdown and metadata to TypeSafe JEV for structured multi-criteria scoring.
    - *Configuration:* POST request to `https://api.typesafe.ai/v1/systemone` using generic HTTP Bearer authentication, evaluating regulatory relevance, UK applicability, fintech applicability, firm actionability, regulatory significance, substantive depth, content quality, and client content value.
    - *Connections:* Input from `Get article with firecrawl`; outputs to `Merge`.
    - *Edge Cases:* API authentication errors (`401 Unauthorized`), request timeouts, or invalid JSON output schemas.

---

#### 1.7 Composite Scoring, Routing, and Ledger Finalization
- **Overview:** Merges evaluated stories, computes weighted final relevance scores, filters approved packages, and writes structured results and status ledgers back to Google Sheets.
- **Nodes Involved:** `Merge`, `Score and assemble ALL stories`, `If relevant or hard win`, `No Operation, do nothing`, `Write approved packages`, `Compile article outcomes`, `Update ledger outcomes`.
- **Node Details:**
  - **Merge** (`n8n-nodes-base.merge`):
    - *Role:* Recombines hard-win stories that bypassed the LLM with stories evaluated by JEV.
    - *Configuration:* Default combine/append mode.
    - *Connections:* Inputs from `Needs LLM judgment?` (false) and `Judge relevance with Jev`; outputs to `Score and assemble ALL stories`.
  - **Score and assemble ALL stories** (`n8n-nodes-base.code`):
    - *Role:* Converts JEV 0–4 scale scores into a weighted percentage score out of 100 and assembles comprehensive story metrics.
    - *Configuration:* Applies category weights (Regulatory Relevance 30%, UK Applicability 20%, Fintech Applicability 15%, Regulatory Significance 15%, Firm Actionability 10%, Client Content Value 10%), checks hard-reject classifications, and assigns score bands.
    - *Connections:* Input from `Merge`; outputs to `If relevant or hard win`.
  - **If relevant or hard win** (`n8n-nodes-base.if`):
    - *Role:* Final routing gate determining whether a story qualifies for the approved output package.
    - *Configuration:* Evaluates if `relevance_score > relevance_threshold` (60) OR `rules_band === hardWin`.
    - *Connections:* Input from `Score and assemble ALL stories`; true branch outputs to `Write approved packages` and `Compile article outcomes`, false branch outputs to `No Operation, do nothing`.
  - **No Operation, do nothing** (`n8n-nodes-base.noOp`):
    - *Role:* Terminal placeholder for unapproved stories.
    - *Configuration:* Default no-op behavior.
    - *Connections:* Input from `If relevant or hard win` (false).
  - **Write approved packages** (`n8n-nodes-base.googleSheets`):
    - *Role:* Appends approved story packages to the designated Google Sheets output tab.
    - *Configuration:* Operation `append`, mapping fields including `run_id`, `best_title`, `best_url`, `best_source`, `rules_score`, `verdict_reason`, and corroborating sources.
    - *Connections:* Input from `If relevant or hard win` (true).
    - *Edge Cases:* Column mapping mismatches or sheet permission constraints.
  - **Compile article outcomes** (`n8n-nodes-base.code`):
    - *Role:* Generates comprehensive status ledgers mapping every processed article (rejects, approved rules, approved JEV, rejected JEV, and duplicates) to its final fate.
    - *Configuration:* Custom JavaScript maps articles by URL and assigns specific status flags (`rejected_rules`, `approved_rules`, `approved_jev`, `rejected_jev`, `corroborating`).
    - *Connections:* Input from `If relevant or hard win` (true); outputs to `Update ledger outcomes`.
  - **Update ledger outcomes** (`n8n-nodes-base.googleSheets`):
    - *Role:* Updates the historical seen-articles ledger with final execution statuses, reasons, rules scores, and cluster IDs matched by URL.
    - *Configuration:* Operation `appendOrUpdate`, matching columns set to `url`.
    - *Connections:* Input from `Compile article outcomes`.
    - *Edge Cases:* Row locking or duplicate URL key collisions during update operations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run the pipeline | manualTrigger | Manual workflow entry point | None | Configure context | Triggers and context |
| Test hook (temporary) | webhook | Webhook execution entry point | None | Configure context | Triggers and context |
| Configure context | set | Initializes global configuration and thresholds | Run the pipeline, Test hook (temporary) | Start fresh this run?, Fetch live articles too? | Edit knobs here, nothing else |
| Start fresh this run? | if | Determines whether to clear historical ledger and output tabs | Configure context | Clear previous packages, Continue after clear gate | Fresh run cleanup |
| Clear previous packages | googleSheets | Clears the approved package sheet tab | Start fresh this run? | Clear seen ledger | Fresh run cleanup |
| Clear seen ledger | googleSheets | Clears the seen articles history ledger tab | Clear previous packages | Continue after clear gate | Fresh run cleanup |
| Continue after clear gate | merge | Re-joins workflow execution post-cleanup | Start fresh this run?, Clear seen ledger | Read source articles, Read seen ledger | Fresh run cleanup |
| Fetch live articles too? | if | Determines whether to pull live RSS feeds | Configure context | Live fetch (FCA Press Release) | Live RSS intake |
| Live fetch (FCA Press Release) | rssFeedRead | Fetches live articles from configured RSS feeds | Fetch live articles too? | Map live articles to dataset shape | Live RSS intake |
| Map live articles to dataset shape | set | Standardizes RSS items into uniform article schema | Live fetch (FCA Press Release) | Merge sheet and live articles | Live RSS intake |
| Read source articles | googleSheets | Loads source records from input Google Sheets tab | Continue after clear gate | Merge sheet and live articles | Read source records |
| Read seen ledger | googleSheets | Reads existing URL history for deduplication | Continue after clear gate | Drop already-seen URLs | Read source records |
| Merge sheet and live articles | merge | Combines sheet articles and live feed articles | Read source articles, Map live articles to dataset shape | Drop already-seen URLs | Read source records |
| Drop already-seen URLs | merge | Filters out URLs already existing in seen ledger | Merge sheet and live articles, Read seen ledger | Stamp pending status | Deduplicate new URLs |
| Stamp pending status | set | Assigns initial pending status and execution metadata | Drop already-seen URLs | Record in seen ledger | Deduplicate new URLs |
| Record in seen ledger | googleSheets | Appends newly discovered pending URLs to sheet ledger | Stamp pending status | Score with rules (Step 1) | Deduplicate new URLs |
| Score with rules (Step 1) | code | Calculates keyword and source weight scores | Record in seen ledger | Drop rejected noise | Rule score candidates |
| Drop rejected noise | filter | Discards items failing the minimum rules threshold | Score with rules (Step 1) | Collect batch for embedding | Rule score candidates |
| Collect batch for embedding | aggregate | Aggregates filtered items into a single batch array | Drop rejected noise | Choose embedding provider | Rule score candidates |
| Choose embedding provider | switch | Routes batch to Hugging Face or OpenAI embedding APIs | Collect batch for embedding | Embed with Hugging Face, Embed with OpenAI | Generate article embeddings |
| Embed with Hugging Face | httpRequest | Generates vector embeddings using Hugging Face API | Choose embedding provider | Normalize HF vectors | Generate article embeddings |
| Embed with OpenAI | httpRequest | Generates vector embeddings using OpenAI API | Choose embedding provider | Normalize OpenAI vectors | Generate article embeddings |
| Normalize HF vectors | set | Standardizes Hugging Face vector response | Embed with Hugging Face | Cluster stories (Step 2) | Generate article embeddings |
| Normalize OpenAI vectors | set | Standardizes OpenAI vector response | Embed with OpenAI | Cluster stories (Step 2) | Generate article embeddings |
| Cluster stories (Step 2) | code | Computes cosine similarity and clusters duplicates | Normalize HF vectors, Normalize OpenAI vectors | Needs LLM judgment? | Cluster and route |
| Needs LLM judgment? | if | Routes primary stories to LLM or bypasses hard wins | Cluster stories (Step 2) | Get article with firecrawl, Merge | Cluster and route |
| Get article with firecrawl | httpRequest | Scrapes full article markdown content | Needs LLM judgment? | Judge relevance with Jev, Set error | JEV relevance scoring |
| Set error | set | Captures web scraping failures gracefully | Get article with firecrawl | None | JEV relevance scoring |
| Judge relevance with Jev | httpRequest | Evaluates article against structured regulatory dimensions | Get article with firecrawl | Merge | JEV relevance scoring |
| Merge | merge | Recombines hard wins and LLM-evaluated stories | Needs LLM judgment?, Judge relevance with Jev | Score and assemble ALL stories | Weighted relevance scoring |
| Score and assemble ALL stories | code | Computes weighted percentage scores from JEV dimensions | Merge | If relevant or hard win | Weighted relevance scoring |
| If relevant or hard win | if | Routes approved stories to output paths | Score and assemble ALL stories | Write approved packages, Compile article outcomes, No Operation, do nothing | Weighted relevance scoring |
| No Operation, do nothing | noOp | Terminal node for unapproved stories | If relevant or hard win | None | Weighted relevance scoring |
| Write approved packages | googleSheets | Appends approved article packages to spreadsheet output | If relevant or hard win | None | Write approved packages |
| Compile article outcomes | code | Compiles final statuses and reasons for all articles | If relevant or hard win | Update ledger outcomes | Update outcome ledger |
| Update ledger outcomes | googleSheets | Updates historical seen ledger with final article outcomes | Compile article outcomes | None | Update outcome ledger |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n without importing the JSON, follow these exact steps:

1. **Create Triggers & Configuration:**
   - Add a **Manual Trigger** node (`Run the pipeline`).
   - Add a **Webhook** node (`Test hook (temporary)`) configured with method `POST` and path `5d5cb8ed-aa6e-47f3-be90-94ce89f004d6`.
   - Add a **Set** node (`Configure context`). Define string and object assignments matching the configuration JSON (e.g., `sheetDocumentId`, tab names `Source Articles`, `Articles`, `Approved Package`, rule keywords, source weights, and thresholds). Connect both triggers to this node.

2. **Set up Cleanup Gate:**
   - Add an **If** node (`Start fresh this run?`). Condition: `{{ $('Configure context').first().json.clearBeforeRun }}` equals `true`. Connect `Configure context` here.
   - Add a **Google Sheets** node (`Clear previous packages`). Operation: `Clear`, Sheet Name: `Approved Package`, Keep First Row: `true`. Connect true branch of the If node here.
   - Add a **Google Sheets** node (`Clear seen ledger`). Operation: `Clear`, Sheet Name: `Articles`, Keep First Row: `true`. Connect previous clear node here.
   - Add a **Merge** node (`Continue after clear gate`). Connect the false branch of `Start fresh this run?` and the output of `Clear seen ledger` into this merge node.

3. **Set up Live Ingestion (Optional Branch):**
   - Add an **If** node (`Fetch live articles too?`). Condition: `{{ $('Configure context').first().json.useLiveFetch }}` equals `true`. Connect `Configure context` here.
   - Add an **RSS Feed Read** node (`Live fetch (FCA Press Release)`). URL: `https://www.fca.org.uk/rss.xml`. Connect true branch.
   - Add a **Set** node (`Map live articles to dataset shape`). Map `title`, `summary`, `source` ('FCA'), `url` (`{{ $json.link }}`), and `content_excerpt`. Connect RSS node here.

4. **Read Sources & Seen Ledger:**
   - Add a **Google Sheets** node (`Read source articles`). Operation: `Get`, Sheet Name: `Source Articles`. Connect input from `Continue after clear gate`.
   - Add a **Google Sheets** node (`Read seen ledger`). Operation: `Get`, Sheet Name: `Articles`, set `Always Output Data` to true. Connect input from `Continue after clear gate`.
   - Add a **Merge** node (`Merge sheet and live articles`). Combine outputs from `Read source articles` and `Map live articles to dataset shape`.

5. **Deduplicate & Record Pending:**
   - Add a **Merge** node (`Drop already-seen URLs`). Mode: `Combine`, Join Mode: `Keep Non Matches`, Fields to Match: `url`. Input 1: `Merge sheet and live articles`, Input 2: `Read seen ledger`.
   - Add a **Set** node (`Stamp pending status`). Set fields: `status = 'pending'`, `run_id = {{ $execution.id }}`.
   - Add a **Google Sheets** node (`Record in seen ledger`). Operation: `Append`, mapping article fields to the `Articles` tab.

6. **Deterministic Rules & Filtering:**
   - Add a **Code** node (`Score with rules (Step 1)`). Insert JavaScript that iterates inputs, tests regex word boundaries for keywords in titles and bodies, adds source weights, and outputs `rulesScore` and `rulesBand`.
   - Add a **Filter** node (`Drop rejected noise`). Condition: `{{ $json.rulesBand }}` `notEquals` `reject`.

7. **Embeddings & Clustering:**
   - Add an **Aggregate** node (`Collect batch for embedding`). Aggregate All Item Data.
   - Add a **Switch** node (`Choose embedding provider`). Rule 1: `embeddingProvider equals huggingface`, Rule 2: `embeddingProvider equals openai`.
   - Add an **HTTP Request** node (`Embed with Hugging Face`). Method: `POST`, URL constructed using `hfEmbeddingModel`, authenticate with **Hugging Face API** credentials.
   - Add an **HTTP Request** node (`Embed with OpenAI`). Method: `POST`, URL: `https://api.openai.com/v1/embeddings`, authenticate with **OpenAI API** credentials.
   - Add two **Set** nodes (`Normalize HF vectors` and `Normalize OpenAI vectors`) to extract and standardize vector arrays into a common `vectors` property.
   - Add a **Code** node (`Cluster stories (Step 2)`). Insert JavaScript implementing cosine similarity calculation against the threshold to group clusters and select primary articles.

8. **Firecrawl Scraping & JEV Assessment:**
   - Add an **If** node (`Needs LLM judgment?`). Condition: `isPrimary === true` AND `rulesBand !== hardWin`.
   - Add an **HTTP Request** node (`Get article with firecrawl`). Method: `POST`, URL: `https://api.firecrawl.dev/v2/scrape`, authenticate using **HTTP Bearer Auth** (`Firecrawl Bearer`), set `onError` to continue execution on error output.
   - Add a **Set** node (`Set error`) connected to the error output of Firecrawl.
   - Add an **HTTP Request** node (`Judge relevance with Jev`). Method: `POST`, URL: `https://api.typesafe.ai/v1/systemone`, authenticate using **HTTP Bearer Auth** (`TypeSafe Bearer`), include state payload and structured questions object.
   - Add a **Merge** node to combine non-judged hard wins and JEV-evaluated primary stories.

9. **Composite Scoring & Final Output Routing:**
   - Add a **Code** node (`Score and assemble ALL stories`). Insert JavaScript calculating weighted percentage scores (0–100), checking hard rejections, and assembling metrics.
   - Add an **If** node (`If relevant or hard win`). Condition (OR): `relevance_score > relevance_threshold` OR `rules_band === hardWin`.
   - Add a **Google Sheets** node (`Write approved packages`). Operation: `Append`, Sheet Name: `Approved Package`, mapping all assembled package columns.
   - Add a **Code** node (`Compile article outcomes`) to map final statuses (`rejected_rules`, `approved_rules`, `approved_jev`, `rejected_jev`, `corroborating`) by URL.
   - Add a **Google Sheets** node (`Update ledger outcomes`). Operation: `Append Or Update`, Sheet Name: `Articles`, Matching Columns: `url`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Starter Google Sheet Template | [Google Sheets Starter Template](https://docs.google.com/spreadsheets/d/1Cmhh8FO9cljzfjni396o442rmLR31qJ1EtGcVFpAV6w/edit?usp=sharing) |
| Swift Standards Release Delay | [Swift Standards Migration Info](https://www.swift.com/swift-accepts-community-request-extend-structured-address-migration-iso-20022-payment-messages) |
| UK Accelerated Settlement Taskforce | [Accelerated Settlement UK](https://acceleratedsettlement.co.uk/) |
| FMSB Standard for Sharing SSIs | [FMSB SSIs Standard](https://fmsb.com/standard-for-sharing-of-standard-settlement-instructions/) |