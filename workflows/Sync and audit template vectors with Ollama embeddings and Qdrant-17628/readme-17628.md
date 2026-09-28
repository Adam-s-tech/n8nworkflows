Sync and audit template vectors with Ollama embeddings and Qdrant

https://n8nworkflows.xyz/workflows/sync-and-audit-template-vectors-with-ollama-embeddings-and-qdrant-17628


# Sync and audit template vectors with Ollama embeddings and Qdrant

### 1. Workflow Overview

This workflow automates the daily ingestion, security auditing, and vectorization of community workflow templates from the public n8n catalog API. Its primary objective is to maintain a synchronized, searchable local vector database within Qdrant, enabling Retrieval-Augmented Generation (RAG) applications over nearly 750,000 public automation templates. 

The execution logic is structured into four sequential functional blocks:
- **1.1 Trigger & Catalog Ingestion:** Initiates execution via a schedule trigger and paginates through the n8n public catalog API to retrieve recent template metadata, subsequently expanding array payloads into distinct individual workflow items.
- **1.2 Full Extraction & Security Audit:** Queries the individual template endpoint to fetch complete workflow JSON configurations, strips non-printable characters and emojis, and executes regular expression pattern matching to flag potential exposed API keys or secrets.
- **1.3 Vector Store Pruning:** Removes existing Qdrant vector points that match the incoming batch of template identifiers to prevent duplication before new vectors are inserted.
- **1.4 Embeddings & Vector Indexing:** Connects to a local Ollama instance utilizing the `nomic-embed-text:latest` model, extracts structured document content and metadata via the default data loader, and upserts the resulting embeddings into the `n8n_templates` Qdrant collection.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Catalog Ingestion
- **Overview:** Starts the daily synchronization run, queries the n8n template search endpoint across multiple pages, and flattens the resulting array into separate items for downstream processing.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `Fetch Template Catalog`
  - `Split Templates`
- **Node Details:**
  - **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Cron-based execution trigger.
    - *Configuration:* Configured to trigger daily at hour 2:00 AM.
    - *Connections:* Outputs to `Fetch Template Catalog`.
    - *Edge Cases:* Server downtime or timezone misconfigurations can cause missed executions; can be triggered manually.
  - **Fetch Template Catalog** (`n8n-nodes-base.httpRequest`)
    - *Role:* API client pulling templates from the n8n public catalog.
    - *Configuration:* GET request to `https://api.n8n.io/templates/search` with query parameters `rows=100` and `page=1`. Configured with iterative pagination pulling pages until `$pageCount >= 2`.
    - *Connections:* Input from `Schedule Trigger`, outputs to `Split Templates`.
    - *Edge Cases:* API rate limits, network timeouts, or schema changes in the n8n catalog response.
  - **Split Templates** (`n8n-nodes-base.itemLists`)
    - *Role:* Data transformation node separating workflow arrays.
    - *Configuration:* Splits out the `workflows` array field into individual items.
    - *Connections:* Input from `Fetch Template Catalog`, outputs to `Fetch Full Workflow JSON`.
    - *Edge Cases:* Empty responses or missing `workflows` property resulting in empty execution sets.

#### 2.2 Full Extraction & Security Audit
- **Overview:** Fetches detailed JSON for each individual workflow template, sanitizes invalid text and control characters, and performs security regex checks to detect exposed secrets or API keys.
- **Nodes Involved:**
  - `Fetch Full Workflow JSON`
  - `Security Audit & Text Sanitizer`
- **Node Details:**
  - **Fetch Full Workflow JSON** (`n8n-nodes-base.httpRequest`)
    - *Role:* Fetches detailed workflow definitions.
    - *Configuration:* GET request to `=https://api.n8n.io/workflows/templates/{{ $json.id }}`. Error handling set to `continueRegularOutput` to prevent single template failures from halting the entire batch.
    - *Connections:* Input from `Split Templates`, outputs to `Security Audit & Text Sanitizer`.
    - *Edge Cases:* 404 Not Found errors for deleted templates or downstream timeout errors.
  - **Security Audit & Text Sanitizer** (`n8n-nodes-base.code`)
    - *Role:* JavaScript execution block handling data sanitization, security checks, and document compilation.
    - *Configuration:* Custom JS parsing workflow nodes, sticky notes, system messages, and metadata. Applies regex patterns for secrets (such as AWS keys, GitHub tokens, Slack tokens, Bearer tokens, and database connection strings). Strips surrogate pairs and control characters.
    - *Key Expressions / Variables:** Uses `$('Split Templates').all()`, regex arrays (`secretRegexes`), and maps payloads to structured JSON.
    - *Connections:* Input from `Fetch Full Workflow JSON`, outputs to `Purge Existing Template Vectors`.
    - *Edge Cases:* Large template payloads causing memory spikes or regular expression catastrophic backtracking on deeply nested parameters.

#### 2.3 Vector Store Pruning
- **Overview:** Cleans the Qdrant database prior to insertion by deleting existing vector points associated with the current batch of templates.
- **Nodes Involved:**
  - `Purge Existing Template Vectors`
  - `Pass-Through Workflow Items`
- **Node Details:**
  - **Purge Existing Template Vectors** (`n8n-nodes-base.httpRequest`)
    - *Role:* API client executing bulk deletions against Qdrant.
    - *Configuration:* POST request to `http://qdrant:6333/collections/n8n_templates/points/delete`. Body contains a filter targeting `metadata.template_id` matching IDs from the sanitizer node. Executed once per batch (`executeOnce: true`).
    - *Connections:* Input from `Security Audit & Text Sanitizer`, outputs to `Pass-Through Workflow Items`.
    - *Edge Cases:* Qdrant container unreachability or invalid JSON payload filters if template IDs are missing.
  - **Pass-Through Workflow Items** (`n8n-nodes-base.code`)
    - *Role:* Re-emits sanitized workflow items after the purge request completes.
    - *Configuration:* JavaScript execution block returning `$("Security Audit & Text Sanitizer").all()`.
    - *Connections:* Input from `Purge Existing Template Vectors`, outputs to `Qdrant Vector Store`.

#### 2.4 Embeddings & Vector Indexing
- **Overview:** Generates vector embeddings using Ollama and upserts the documents, text chunks, and metadata fields into the Qdrant vector database.
- **Nodes Involved:**
  - `Qdrant Vector Store`
  - `Embeddings Ollama`
  - `Default Data Loader`
- **Node Details:**
  - **Qdrant Vector Store** (`@n8n/n8n-nodes-langchain.vectorStoreQdrant`)
    - *Role:* LangChain vector database integration node.
    - *Configuration:* Mode set to `insert`. Target collection `n8n_templates`. Embedding batch size set to `10`. Requires Qdrant API credentials.
    - *Connections:* Input main from `Pass-Through Workflow Items`, AI embedding input from `Embeddings Ollama`, AI document input from `Default Data Loader`.
    - *Edge Cases:* Connection failures to Qdrant or embedding size mismatches.
  - **Embeddings Ollama** (`@n8n/n8n-nodes-langchain.embeddingsOllama`)
    - *Role:* AI embedding generator.
    - *Configuration:* Model configured as `nomic-embed-text:latest`. Requires Ollama API credentials.
    - *Connections:* Outputs via `ai_embedding` to `Qdrant Vector Store`.
    - *Edge Cases:* Ollama model unavailable or out-of-memory errors on the host machine.
  - **Default Data Loader** (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`)
    - *Role:* Document processor mapping JSON properties to metadata fields.
    - *Configuration:* Maps 12 metadata fields (`template_id`, `name`, `has_security_warning`, `categories`, `nodeTypes`, `description`, `node_count`, `trigger_types`, `has_ai_nodes`, `views`, `created_at`, `template_url`) from `{{ $json.metadata[...] }}` expressions.
    - *Connections:* Outputs via `ai_document` to `Qdrant Vector Store`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Intro` | `n8n-nodes-base.stickyNote` | Workflow documentation and prerequisites. | None | None | ## 🛠️ n8n Template Catalog Vector Sync<br><br>**Purpose:** Daily incremental ingestion and security auditing of public community workflow templates into Qdrant for local RAG retrieval.<br><br>**👤 Who It's For:**<br>Automation architects wanting a searchable, locally hosted vector database of ~749k n8n community workflow patterns.<br><br>**🔗 Quick Links & Seed Database:**<br>• **GitHub Repository**: [jdm6457/n8n-rag-suite](https://github.com/jdm6457/n8n-rag-suite)<br>• **Instant Seed Download**: `wget https://huggingface.co/datasets/jdm6457/n8n-rag-suite-snapshot/resolve/main/n8n_templates_749k.snapshot.gz`<br><br>**⚙️ Prerequisites:**<br>• Host Ollama running `nomic-embed-text:latest`<br>• Local Qdrant instance on port 6333 (`n8n_templates` collection)<br><br>**🚀 How to Use:**<br>1. Run manually once to fetch recent templates, or let the Schedule Trigger execute daily at 2:00 AM.<br>2. Alternatively, use the Hugging Face snapshot link above to seed the Qdrant database instantly without scraping. |
| `Sticky Note - Ingestion` | `n8n-nodes-base.stickyNote` | Documents ingestion block. | None | None | ### 1. Ingestion & Incremental Fetch<br>Fetches top community templates from n8n public catalog API.<br>* Pages 1–2 only (~200 items) for daily updates.<br>* Splits collection array into individual workflow items. |
| `Sticky Note - Audit` | `n8n-nodes-base.stickyNote` | Documents extraction and audit block. | None | None | ### 2. Full Extraction & Security Audit<br>* Pulls full workflow metadata, sticky note text, and system prompts.<br>* Strips non-printable ASCII/hex control characters.<br>* Scans node configuration strings for exposed API keys. |
| `Sticky Note - Qdrant` | `n8n-nodes-base.stickyNote` | Documents vector indexing block. | None | None | ### 3. Purge Dupes, Embed & Vector Indexing<br>* Check for duplicate templates and delete them<br>* Embeds text content via Ollama (`nomic-embed-text`).<br>* Upserts vectors and metadata into Qdrant `n8n_templates` collection. |
| `Schedule Trigger` | `n8n-nodes-base.scheduleTrigger` | Daily schedule execution. | None | `Fetch Template Catalog` | |
| `Fetch Template Catalog` | `n8n-nodes-base.httpRequest` | Fetches template catalog via paginated API. | `Schedule Trigger` | `Split Templates` | |
| `Split Templates` | `n8n-nodes-base.itemLists` | Splits workflow array into individual items. | `Fetch Template Catalog` | `Fetch Full Workflow JSON` | |
| `Fetch Full Workflow JSON` | `n8n-nodes-base.httpRequest` | Retrieves full template payload by ID. | `Split Templates` | `Security Audit & Text Sanitizer` | |
| `Security Audit & Text Sanitizer` | `n8n-nodes-base.code` | Sanitizes text and scans for secrets. | `Fetch Full Workflow JSON` | `Purge Existing Template Vectors` | |
| `Purge Existing Template Vectors` | `n8n-nodes-base.httpRequest` | Deletes existing Qdrant points for templates. | `Security Audit & Text Sanitizer` | `Pass-Through Workflow Items` | |
| `Pass-Through Workflow Items` | `n8n-nodes-base.code` | Passes sanitized items to vector store. | `Purge Existing Template Vectors` | `Qdrant Vector Store` | |
| `Qdrant Vector Store` | `@n8n/n8n-nodes-langchain.vectorStoreQdrant` | Upserts documents and metadata into Qdrant. | `Pass-Through Workflow Items` | None | |
| `Embeddings Ollama` | `@n8n/n8n-nodes-langchain.embeddingsOllama` | Generates vector embeddings via Ollama. | None | `Qdrant Vector Store` (AI Embedding) | |
| `Default Data Loader` | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Maps workflow metadata fields to documents. | None | `Qdrant Vector Store` (AI Document) | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: `n8n-nodes-base.scheduleTrigger`
   - Configuration: Set trigger interval rule to run daily at hour `2`.
2. **Create Fetch Template Catalog Node**
   - Type: `n8n-nodes-base.httpRequest`
   - Configuration: Method GET, URL `https://api.n8n.io/templates/search`. Add query parameters `rows` = `100` and `page` = `1`. Configure pagination options: parameter name `page` with value `={{ $pageCount + 1 }}`, complete expression `={{ $pageCount >= 2 }}`.
   - Connect from: `Schedule Trigger`.
3. **Create Split Templates Node**
   - Type: `n8n-nodes-base.itemLists`
   - Configuration: Set **Field to Split Out** to `workflows`.
   - Connect from: `Fetch Template Catalog`.
4. **Create Fetch Full Workflow JSON Node**
   - Type: `n8n-nodes-base.httpRequest`
   - Configuration: Method GET, URL `=https://api.n8n.io/workflows/templates/{{ $json.id }}`. Under error handling options, enable `Continue On Fail` (`continueRegularOutput`).
   - Connect from: `Split Templates`.
5. **Create Security Audit & Text Sanitizer Node**
   - Type: `n8n-nodes-base.code`
   - Configuration: Insert JavaScript code to parse workflow elements, apply string sanitization for control characters, evaluate secret regex patterns, and format the output payload into `metadata` and `pageContent` structures.
   - Connect from: `Fetch Full Workflow JSON`.
6. **Create Purge Existing Template Vectors Node**
   - Type: `n8n-nodes-base.httpRequest`
   - Configuration: Method POST, URL `http://qdrant:6333/collections/n8n_templates/points/delete`. Set **Specify Body** to `JSON`. Use expression body filtering points where `metadata.template_id` matches the array of template IDs retrieved from the security audit node. Enable `Execute Once` in node settings.
   - Connect from: `Security Audit & Text Sanitizer`.
7. **Create Pass-Through Workflow Items Node**
   - Type: `n8n-nodes-base.code`
   - Configuration: Insert JavaScript returning `$("Security Audit & Text Sanitizer").all();`.
   - Connect from: `Purge Existing Template Vectors`.
8. **Create Qdrant Vector Store Node**
   - Type: `@n8n/n8n-nodes-langchain.vectorStoreQdrant`
   - Configuration: Mode `insert`. Collection selection: list value `n8n_templates`. Embedding batch size: `10`. Configure Qdrant credentials.
   - Connect from: `Pass-Through Workflow Items` (Main input).
9. **Create Embeddings Ollama Node**
   - Type: `@n8n/n8n-nodes-langchain.embeddingsOllama`
   - Configuration: Model name `nomic-embed-text:latest`. Configure Ollama credentials.
   - Connect to: `Qdrant Vector Store` via `ai_embedding` connection.
10. **Create Default Data Loader Node**
    - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`
    - Configuration: Add metadata values mapping fields (`template_id`, `name`, `has_security_warning`, `categories`, `nodeTypes`, `description`, `node_count`, `trigger_types`, `has_ai_nodes`, `views`, `created_at`, `template_url`) using respective `{{ $json.metadata[...] }}` expressions.
    - Connect to: `Qdrant Vector Store` via `ai_document` connection.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GitHub Repository | [jdm6457/n8n-rag-suite](https://github.com/jdm6457/n8n-rag-suite) |
| Instant Seed Download | `wget https://huggingface.co/datasets/jdm6457/n8n-rag-suite-snapshot/resolve/main/n8n_templates_749k.snapshot.gz` |