Send a daily tech and security briefing with Ollama, Gmail, and Discord

https://n8nworkflows.xyz/workflows/send-a-daily-tech-and-security-briefing-with-ollama--gmail--and-discord-20242


# Send a daily tech and security briefing with Ollama, Gmail, and Discord

### 1. Workflow Overview

This workflow automates the collection, deduplication, AI-powered ranking, summarization, and distribution of daily technology, security, and homelab briefings. It runs daily at 6:30 AM or can be triggered manually. The system fetches live data from configured RSS/Atom feeds and the CISA Known Exploited Vulnerabilities JSON catalog, filters out old or previously sent stories, executes local AI models via Ollama for semantic deduplication, classification, and summarization, and delivers a clean, multi-section briefing via SMTP email and Discord webhooks.

The logical processing is divided into the following functional blocks:
- **1.1 Triggers and Configuration:** Initiates the execution via schedule or manual input and supplies centralized variables (feeds, models, thresholds).
- **1.2 Feed and Vulnerability Collection:** Fetches and normalizes raw items from a list of RSS feeds and the CISA KEV catalog in parallel.
- **1.3 History and Semantic Deduplication:** Filters out previously processed stories using an n8n Data Table and removes near-duplicates via Ollama title embeddings and cosine similarity checks.
- **1.4 Classification and Ranking:** Uses a fast local language model to categorize headlines, assigns priority ratings, and applies keyword/freshness boosts to bucket stories into specific sections.
- **1.5 Summary Generation and Formatting:** Employs a writing model to draft contextual summaries for top stories and compiles the final layout into HTML, Markdown, and Discord-sized chunks.
- **1.6 Delivery and Archiving:** Sends the briefing via email, dispatches chunks to Discord, optionally pushes archives to an Obsidian vault, and saves processed story keys to the history table.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers and Configuration
- **Overview:** Initializes the workflow on a daily cron schedule or on-demand, immediately setting up global parameters, models, URLs, and target feeds for downstream nodes.
- **Nodes Involved:** 
  - `When 6:30 AM Daily`
  - `Manual Test Trigger`
  - `Build Config`

- **Node Details:**
  - **When 6:30 AM Daily**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - *Configuration Choices:* Configured with a cron expression (`30 6 * * *`) to execute daily at 06:30 AM.
    - *Input/Output Connections:* Output connects to `Build Config`.
    - *Edge Cases/Failures:* Missed triggers if the n8n instance is offline at the exact scheduled timestamp.
  - **Manual Test Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger)
    - *Configuration Choices:* Allows on-demand execution for testing and debugging.
    - *Input/Output Connections:* Output connects to `Build Config`.
  - **Build Config**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields)
    - *Configuration Choices:* Uses manual assignment with dot notation enabled (`dotNotation: true`). Stores global variables including Ollama base URL, model names (`qwen3:8b`, `gemma4:12b-it-qat`, `nomic-embed-text`), email addresses, Discord webhook URL, Vault base path, and an array of 18 RSS/Atom feed objects.
    - *Input/Output Connections:* Inputs from both triggers; outputs branch to `Split Feed List Into Items` and `Fetch CISA Exploited Vulnerabilities`.
    - *Edge Cases/Failures:* Missing or misconfigured URLs will cause downstream HTTP requests to fail.

---

#### 1.2 Feed and Vulnerability Collection
- **Overview:** Expands the feed configuration array, queries all RSS/Atom sources and the CISA KEV feed concurrently, merges the raw responses, and normalizes them into a unified story format limited to items published within the last 72 hours.
- **Nodes Involved:**
  - `Split Feed List Into Items`
  - `Read Each RSS Feed`
  - `Fetch CISA Exploited Vulnerabilities`
  - `Merge Feeds and CISA Data`
  - `Normalize Stories`

- **Node Details:**
  - **Split Feed List Into Items**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* JavaScript mapping function parsing `$input.first().json.feeds` into individual items with `feedUrl` and `sourceName`.
    - *Input/Output Connections:* Input from `Build Config`; output connects to `Read Each RSS Feed`.
  - **Read Each RSS Feed**
    - *Type and Technical Role:* `n8n-nodes-base.rssFeedRead` (RSS Feed Read)
    - *Configuration Choices:* Dynamic URL evaluation `={{ $json.feedUrl }}`. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Split Feed List Into Items`; output connects to `Merge Feeds and CISA Data` (Input 1).
    - *Edge Cases/Failures:* Network timeouts, invalid XML structures, or dead feed URLs.
  - **Fetch CISA Exploited Vulnerabilities**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* GET request to `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json` with a 30,000ms timeout. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Build Config`; output connects to `Merge Feeds and CISA Data` (Input 2).
    - *Edge Cases/Failures:* Remote server downtime or schema changes in the CISA JSON feed.
  - **Merge Feeds and CISA Data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Merge Node)
    - *Configuration Choices:* Combines data streams from the RSS reader and the CISA HTTP request.
    - *Input/Output Connections:* Inputs from `Read Each RSS Feed` and `Fetch CISA Exploited Vulnerabilities`; output connects to `Normalize Stories`.
  - **Normalize Stories**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* JavaScript logic calculating time differences against `cfg.maxHours` (72 hours), normalizing titles, extracting hostnames/sources, applying specific regex rules for GitHub releases and Reddit links, generating deterministic deduplication hashes (`dedupe_key`), and sorting stories by recency. Caps output at `cfg.maxStories` (120).
    - *Input/Output Connections:* Input from `Merge Feeds and CISA Data`; output connects to `Get History`.

---

#### 1.3 History and Semantic Deduplication
- **Overview:** Retrieves historical records from an n8n Data Table to drop exact duplicates, aggregates item titles, computes vector embeddings via Ollama, and executes cosine similarity checks to filter out semantic near-duplicates.
- **Nodes Involved:**
  - `Get History`
  - `Dedup Against History`
  - `Collect Titles for Embedding`
  - `Embed Titles with Ollama`
  - `Semantic Dedup`

- **Node Details:**
  - **Get History**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table)
    - *Configuration Choices:* Operation set to `get`, returns all records from data table `tech_briefing_history`. Executed once with `alwaysOutputData: true` and `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Normalize Stories`; output connects to `Dedup Against History`.
  - **Dedup Against History**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Filters incoming stories against a `Set` of `dedupe_key` values sourced from historical records.
    - *Input/Output Connections:* Input from `Get History`; output connects to `Collect Titles for Embedding`.
  - **Collect Titles for Embedding**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Maps all incoming story titles into an array payload.
    - *Input/Output Connections:* Input from `Dedup Against History`; output connects to `Embed Titles with Ollama`.
  - **Embed Titles with Ollama**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* POST request to `={{ $('Build Config').first().json.ollama + '/api/embed' }}` with a 120,000ms timeout. Body sends model (`nomic-embed-text`) and `input` array. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Collect Titles for Embedding`; output connects to `Semantic Dedup`.
    - *Edge Cases/Failures:* Ollama service unavailability or out-of-memory errors during embedding generation.
  - **Semantic Dedup**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Calculates cosine similarity between title embeddings. Drops items exceeding `cfg.simThreshold` (0.9).
    - *Input/Output Connections:* Input from `Embed Titles with Ollama`; output connects to `Build Classify Requests`.

---

#### 1.4 Classification and Ranking
- **Overview:** Prepares classification payloads, queries a fast local language model via Ollama to categorize and score each story against a JSON schema, and applies algorithmic boosts to select top stories and organize remaining items into categorized sections.
- **Nodes Involved:**
  - `Build Classify Requests`
  - `Classify Stories with Ollama`
  - `Select And Bucket`

- **Node Details:**
  - **Build Classify Requests**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Constructs chat prompt messages with a strict system prompt and JSON schema format defining categories (`ai_llm`, `banking_fintech`, `homelab_selfhosting`, `cybersecurity`, `tech_industry`, `excluded`) and priority levels (1–5). Uses model `qwen3:8b`.
    - *Input/Output Connections:* Input from `Semantic Dedup`; output connects to `Classify Stories with Ollama`.
  - **Classify Stories with Ollama**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* POST request to `={{ $('Build Config').first().json.ollama + '/api/chat' }}` with a 90,000ms timeout, `maxTries: 2`, and `waitBetweenTries: 1500`. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Build Classify Requests`; output connects to `Select And Bucket`.
    - *Edge Cases/Failures:* LLM returning invalid JSON or request timeouts.
  - **Select And Bucket**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Parses classification responses, filters excluded titles using regex, applies keyword and freshness score boosts, sorts by score, extracts top stories (`cfg.topCount`), and chunks the remaining items into category-specific sections (`radar`, `security`, `banking`, `worth`).
    - *Input/Output Connections:* Input from `Classify Stories with Ollama`; output connects to `Split Top Stories`.

---

#### 1.5 Summary Generation and Formatting
- **Overview:** Iterates through top stories to request detailed summaries from an Ollama writing model, then validates responses and formats the complete briefing into HTML, Markdown, and Discord-compatible message chunks.
- **Nodes Involved:**
  - `Split Top Stories`
  - `Build Write Requests`
  - `Write Summaries with Ollama`
  - `Validate And Format`

- **Node Details:**
  - **Split Top Stories**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Splits the top stories array into individual items for parallel summary generation.
    - *Input/Output Connections:* Input from `Select And Bucket`; output connects to `Build Write Requests`.
  - **Build Write Requests**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Builds chat payloads for model `gemma4:12b-it-qat` with a strict JSON schema requiring `what_happened`, `why_it_matters`, and `personal_relevance`. Enforces a professional tech/homelab reader profile constraint.
    - *Input/Output Connections:* Input from `Split Top Stories`; output connects to `Write Summaries with Ollama`.
  - **Write Summaries with Ollama**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* POST request to `={{ $('Build Config').first().json.ollama + '/api/chat' }}` with a 180,000ms timeout. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Build Write Requests`; output connects to `Validate And Format`.
    - *Edge Cases/Failures:* Long generation times leading to timeouts, or malformed JSON payloads from the LLM.
  - **Validate And Format**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Validates writing responses (falling back to raw summaries if needed), constructs responsive HTML email layouts, generates Markdown archive text, splits text into chunks under 1,900 characters for Discord, and compiles an array of history rows (`rows`) for future deduplication.
    - *Input/Output Connections:* Input from `Write Summaries with Ollama`; outputs connect to `Send Email Briefing`, `Split Discord Messages`, `Archive Markdown To Vault`, and `Split History Rows`.

---

#### 1.6 Delivery and Archiving
- **Overview:** Dispatches the finalized briefing via SMTP email and Discord webhook posts, optionally syncs archives to an Obsidian vault via REST API, and records processed story keys in the n8n Data Table.
- **Nodes Involved:**
  - `Send Email Briefing`
  - `Split Discord Messages`
  - `Post Briefing to Discord`
  - `Archive Markdown To Vault`
  - `Archive JSON To Vault`
  - `Split History Rows`
  - `Save Stories to History`

- **Node Details:**
  - **Send Email Briefing**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Email)
    - *Configuration Choices:* Sends both HTML and plain text email formats. Uses credentials for SMTP (`Gmail SMTP (Tech Briefing)`). Dynamic recipient and sender mappings from `Build Config`.
    - *Input/Output Connections:* Input from `Validate And Format`. Executed once.
    - *Edge Cases/Failures:* SMTP authentication failures or invalid recipient addresses.
  - **Split Discord Messages**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Splits text chunks into individual items for sequential Discord posting.
    - *Input/Output Connections:* Input from `Validate And Format`; output connects to `Post Briefing to Discord`.
  - **Post Briefing to Discord**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* POST request to `={{ $('Build Config').first().json.discordWebhookUrl }}` with a 30,000ms timeout and JSON body content. Error handling set to `continueRegularOutput`.
    - *Input/Output Connections:* Input from `Split Discord Messages`.
    - *Edge Cases/Failures:* Rate limiting or invalid webhook URLs.
  - **Archive Markdown To Vault**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* PUT request to Obsidian Local REST API endpoint (`http://host.docker.internal:27123/vault/...`). Raw content type `text/markdown`. Bearer token authentication via environment variable `OBSIDIAN_TOKEN`. Marked as `disabled: true` by default.
    - *Input/Output Connections:* Input from `Validate And Format`; output connects to `Archive JSON To Vault`.
  - **Archive JSON To Vault**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* PUT request to Obsidian Local REST API endpoint for JSON output. Raw content type `application/json`. Bearer token authentication via `OBSIDIAN_TOKEN`. Marked as `disabled: true` by default.
    - *Input/Output Connections:* Input from `Archive Markdown To Vault`.
  - **Split History Rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Maps array rows into individual items for bulk data table insertion.
    - *Input/Output Connections:* Input from `Validate And Format`; output connects to `Save Stories to History`.
  - **Save Stories to History**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table)
    - *Configuration Choices:* Operation set to save/insert rows into `tech_briefing_history` data table. Mapped columns: `dedupe_key`, `url`, `title`, `source`, `category`, `published_at`, and `first_seen`. Bulk optimization enabled (`optimizeBulk: true`).
    - *Input/Output Connections:* Input from `Split History Rows`.
    - *Edge Cases/Failures:* Data table schema mismatch or missing table definition.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When 6:30 AM Daily | scheduleTrigger | Triggers the workflow every morning at 6:30 AM. | None | Build Config | Triggers and configuration |
| Manual Test Trigger | manualTrigger | Allows manual execution for testing and debugging. | None | Build Config | Triggers and configuration |
| Build Config | set | Stores global configuration variables, models, and feed lists. | When 6:30 AM Daily, Manual Test Trigger | Split Feed List Into Items, Fetch CISA Exploited Vulnerabilities | Triggers and configuration |
| Split Feed List Into Items | code | Converts the feed array into individual workflow items. | Build Config | Read Each RSS Feed | Collect feeds and CISA KEV |
| Read Each RSS Feed | rssFeedRead | Fetches items from individual RSS and Atom feed URLs. | Split Feed List Into Items | Merge Feeds and CISA Data | Collect feeds and CISA KEV |
| Fetch CISA Exploited Vulnerabilities | httpRequest | Downloads the CISA Known Exploited Vulnerabilities JSON catalog. | Build Config | Merge Feeds and CISA Data | Collect feeds and CISA KEV |
| Merge Feeds and CISA Data | merge | Combines RSS feed items and CISA vulnerability data. | Read Each RSS Feed, Fetch CISA Exploited Vulnerabilities | Normalize Stories | Collect feeds and CISA KEV |
| Normalize Stories | code | Cleans, filters (72h window), and formats raw stories. | Merge Feeds and CISA Data | Get History | Collect feeds and CISA KEV |
| Get History | dataTable | Loads historical records from the history data table. | Normalize Stories | Dedup Against History | Skip stories already sent |
| Dedup Against History | code | Drops stories that already exist in the history table. | Get History | Collect Titles for Embedding | Skip stories already sent |
| Collect Titles for Embedding | code | Prepares an array of story titles for vector generation. | Dedup Against History | Embed Titles with Ollama | Remove near-duplicates |
| Embed Titles with Ollama | httpRequest | Generates vector embeddings for titles via Ollama. | Collect Titles for Embedding | Semantic Dedup | Remove near-duplicates |
| Semantic Dedup | code | Removes semantic near-duplicates using cosine similarity. | Embed Titles with Ollama | Build Classify Requests | Remove near-duplicates |
| Build Classify Requests | code | Prepares classification chat prompts and JSON schemas. | Semantic Dedup | Classify Stories with Ollama | Classify and rank |
| Classify Stories with Ollama | httpRequest | Sends classification payloads to Ollama for category/priority assignment. | Build Classify Requests | Select And Bucket | Classify and rank |
| Select And Bucket | code | Applies boosts, filters, and selects top stories and sections. | Classify Stories with Ollama | Split Top Stories | Classify and rank |
| Split Top Stories | code | Splits top stories into individual items for summarization. | Select And Bucket | Build Write Requests | Write the briefing |
| Build Write Requests | code | Prepares summary writing chat prompts and schemas for top stories. | Split Top Stories | Write Summaries with Ollama | Write the briefing |
| Write Summaries with Ollama | httpRequest | Generates structured summaries for each top story via Ollama. | Build Write Requests | Validate And Format | Write the briefing |
| Validate And Format | code | Validates outputs and formats the briefing into HTML, Markdown, and Discord chunks. | Write Summaries with Ollama | Send Email Briefing, Split Discord Messages, Archive Markdown To Vault, Split History Rows | Write the briefing |
| Send Email Briefing | emailSend | Sends the formatted HTML and text briefing via SMTP email. | Validate And Format | None | Deliver |
| Split Discord Messages | code | Splits message chunks for individual Discord webhook delivery. | Validate And Format | Post Briefing to Discord | Deliver |
| Post Briefing to Discord | httpRequest | Posts briefing chunks to a Discord channel via webhook. | Split Discord Messages | None | Deliver |
| Archive Markdown To Vault | httpRequest | Saves the markdown briefing to Obsidian via Local REST API. | Validate And Format | Archive JSON To Vault | Optional: archive to Obsidian |
| Archive JSON To Vault | httpRequest | Saves the JSON briefing to Obsidian via Local REST API. | Archive Markdown To Vault | None | Optional: archive to Obsidian |
| Split History Rows | code | Converts history entries into individual rows for data table insertion. | Validate And Format | Save Stories to History | Remember what was sent |
| Save Stories to History | dataTable | Saves processed story dedupe keys and metadata to the history data table. | Split History Rows | None | Remember what was sent |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Triggers and Configuration:**
   - Add a **Schedule Trigger** node (`When 6:30 AM Daily`) and set the cron expression to `30 6 * * *`.
   - Add a **Manual Trigger** node (`Manual Test Trigger`).
   - Add an **Edit Fields (Set)** node named `Build Config`. Enable dot notation (`dotNotation: true`) and configure manual string, number, and array assignments for `ollama` (`http://localhost:11434`), model names (`qwen3:8b`, `gemma4:12b-it-qat`, `nomic-embed-text`), email addresses, Discord webhook URL, Vault base path, thresholds (`maxHours`: 72, `freshHours`: 24, `simThreshold`: 0.9, `topCount`: 5, `sectionCap`: 5, `maxStories`: 120), and the `feeds` array containing your RSS/Atom feed URLs and names.
   - Connect both triggers to `Build Config`.

2. **Set Up Feed and Vulnerability Collection:**
   - Create a **Code** node named `Feed Feed List Into Items` connected from `Build Config`. Paste the JavaScript mapping logic to split the feeds array.
   - Create an **RSS Feed Read** node named `Read Each RSS Feed`. Set the URL expression to `={{ $json.feedUrl }}` and error handling to `continueRegularOutput`. Connect it from `Split Feed List Into Items`.
   - Create an HTTP Request node named `Fetch CISA Exploited Vulnerabilities`. Set method to GET, URL to `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`, timeout to 30000ms, and error handling to `continueRegularOutput`. Connect it from `Build Config`.
   - Create a **Merge** node named `Feeds and CISA Data`. Connect `Read Each RSS Feed` to Input 1 and `Fetch CISA Exploited Vulnerabilities` to Input 2.
   - Create a **Code** node named `Normalize Stories` connected from `Merge Feeds and CISA Data`. Insert the normalization script handling time windows, domain parsing, and deduplication hash generation.

3. **Set Up Deduplication Blocks:**
   - Ensure an n8n Data Table named `tech_briefing_history` exists with text columns: `dedupe_key`, `url`, `title`, `source`, `category`, `published_at`, `first_seen`.
   - Create a **Data Table** node named `Get History`. Set operation to `get`, return all rows from `tech_briefing_history`, execute once, and set `alwaysOutputData: true`. Connect from `Normalize Stories`.
   - Create a **Code** node named `Dedup Against History` connected from `Get History`.
   - Create a **Code** node named `Collect Titles for Embedding` connected from `Dedup Against History`.
   - Create an HTTP Request node named `Embed Titles with Ollama`. Set method to POST, URL to `={{ $('Build Config').first().json.ollama + '/api/embed' }}`, timeout to 120,000ms, specify body as JSON, and send body expression `={{ JSON.stringify({ model: $('Build Config').first().json.models.embed, input: $json.titles }) }}`. Connect from `Collect Titles for Embedding`.
   - Create a **Code** node named `Semantic Dedup` connected from `Embed Titles with Ollama` to filter cosine similarities.

4. **Set Up Classification and Ranking:**
   - Create a **Code** node named `Build Classify Requests` connected from `Semantic Dedup`.
   - Create an HTTP Request node named `Classify Stories with Ollama`. Set method to POST, URL to `={{ $('Build Config').first().json.ollama + '/api/chat' }}`, timeout 90000ms, max retries 2, wait 1500ms, and send JSON body `={{ JSON.stringify($json.body) }}`. Connect from `Build Classify Requests`.
   - Create a **Code** node named `Select And Bucket` connected from `Classify Stories with Ollama`.

5. **Set Up Summary Generation and Formatting:**
   - Create a **Code** node named `Split Top Stories` connected from `Select And Bucket`.
   - Create a **Code** node named `Build Write Requests` connected from `Split Top Stories`.
   - Create an HTTP Request node named `Write Summaries with Ollama`. Set method to POST, URL to `={{ $('Build Config').first().json.ollama + '/api/chat' }}`, timeout 180,000ms, and send JSON body `={{ JSON.stringify($json.body) }}`. Connect from `Build Write Requests`.
   - Create a **Code** node named `Validate And Format` connected from `Write Summaries with Ollama`.

6. **Set Up Delivery and Archiving:**
   - Create an **Email** node named `Send Email Briefing`. Configure SMTP credentials, format to `both`, subject `={{ $json.subject }}`, to address `={{ $('Build Config').first().json.emailTo }}`, from address `={{ $('Build Config').first().json.emailFrom }}`, HTML `={{ $json.email_html }}`, and text `={{ $json.email_text }}`. Connect from `Validate And Format`.
   - Create a **Code** node named `Split Discord Messages` connected from `Validate And Format`.
   - Create an HTTP Request node named `Post Briefing to Discord`. Set method to POST, URL to `={{ $('Build Config').first().json.discordWebhookUrl }}`, timeout 30000ms, and JSON body `={{ JSON.stringify({ content: $json.content }) }}`. Connect from `Split Discord Messages`.
   - *(Optional)* Create HTTP Request nodes `Archive Markdown To Vault` and `Archive JSON To Vault` connected from `Validate And Format` targeting Obsidian Local REST API (`http://host.docker.internal:27123/vault/...`) with Bearer token authentication (`OBSIDIAN_TOKEN`). Leave disabled by default.
   - Create a **Code** node named `Split History Rows` connected from `Validate And Format`.
   - Create a **Data Table** node named `Save Stories to History`. Configure operation to insert/save rows into `tech_briefing_history` data table, map columns (`dedupe_key`, `url`, `title`, `source`, `category`, `published_at`, `first_seen`), and enable bulk optimization. Connect from `Split History Rows`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Self-hosted n8n with network access to a local Ollama instance. | System architecture requirement |
| Recommended models: `qwen3:8b`, `gemma4:12b-it-qat`, and `nomic-embed-text` (approx. 16 GB RAM required). | Local LLM setup |
| Data Table name required: `tech_briefing_history`. | State management |
| Obsidian Local REST API plugin integration (optional). | Archival storage |