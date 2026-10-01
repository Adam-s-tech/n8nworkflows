Reply to Facebook comments with OpenRouter, Supabase, Postgres and Google Drive

https://n8nworkflows.xyz/workflows/reply-to-facebook-comments-with-openrouter--supabase--postgres-and-google-drive-20192


# Reply to Facebook comments with OpenRouter, Supabase, Postgres and Google Drive

### 1. Workflow Overview

This workflow automates replies to Facebook Page comments using an AI Agent powered by OpenRouter. It maintains contextual chat memory via PostgreSQL and retrieves factual information from a Supabase vector knowledge base. Simultaneously, it maintains the knowledge base by monitoring Google Drive for PDF file additions, updates, and removals, ensuring that the RAG (Retrieval-Augmented Generation) corpus stays fully synchronized.

The system is organized into four main logical blocks:
- **1.1 Webhook Reception and Verification:** Intercepts incoming Meta requests, verifies webhook subscription challenges, and routes valid comment data.
- **1.2 AI Facebook Comment Auto-Reply Engine:** Filters out unwanted events (such as self-comments), enriches data with original post context, queries chat memory and vector stores via an AI Agent, and posts automated replies back to Facebook.
- **1.3 Automated Google Drive Ingestion (New & Updated Files):** Monitors Google Drive folders for new or modified PDFs, extracts text, generates embeddings via OpenAI, and indexes or re-indexes them in Supabase.
- **1.4 Knowledge Base Maintenance (Deleted Files):** Detects removed or trashed files in Google Drive and purges corresponding vector embeddings from Supabase to prevent outdated answers.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Reception and Verification
- **Overview:** Receives raw HTTP webhook payloads from Meta, separates verification challenges from incoming event data, and responds accordingly.
- **Nodes Involved:** 
  - `Webhook`
  - `Set Challenge Variable`
  - `Respond to Webhook`
- **Node Details:**
  - **Webhook**
    - Type and technical role: `n8n-nodes-base.webhook` — Core entry point for external HTTP POST/GET requests.
    - Configuration: Configured with custom path, multiple methods enabled, response mode set to use a response node.
    - Key expressions or variables: Uses webhook ID `ee30223b-1fdd-409d-9302-7d784d42ac1b`.
    - Input and output connections: Receives external requests; outputs to `Set Challenge Variable` and `Extract Comment Data`.
    - Edge cases / Failure types: Invalid payload structures or missing parameters can break downstream evaluation.
  - **Set Challenge Variable**
    - Type and technical role: `n8n-nodes-base.set` — Formats and assigns challenge tokens for verification.
    - Configuration: Assigns string value to `query['hub.challenge']`.
    - Key expressions or variables: `={{ $json.query['hub.challenge'] }}`
    - Input and output connections: Input from `Webhook`; output to `Respond to Webhook`.
  - **Respond to Webhook**
    - Type and technical role: `n8n-nodes-base.respondToWebhook` — Sends HTTP responses back to the webhook caller.
    - Configuration: Responds with text.
    - Key expressions or variables: `={{ $json.query['hub.challenge'] }}`
    - Input and output connections: Input from `Set Challenge Variable`.

#### 2.2 AI Facebook Comment Auto-Reply Engine
- **Overview:** Validates incoming comments, ignores page self-replies, gathers post context via Facebook Graph API, processes requests through an OpenRouter AI Agent backed by Postgres memory and Supabase search tools, and publishes replies.
- **Nodes Involved:**
  - `Extract Comment Data`
  - `Filter: Is Comment Event`
  - `End / Not a Comment`
  - `Filter: Ignore Page Self-Comments`
  - `End / Ignore Self-Comment`
  - `Get Post Content`
  - `AI Agent`
  - `OpenRouter Chat Model`
  - `Postgres Chat Memory`
  - `Search knowledge base`
  - `Embed the query`
  - `Send Reply Comment`
- **Node Details:**
  - **Extract Comment Data**
    - Type and technical role: `n8n-nodes-base.set` — Normalizes incoming JSON payloads into dedicated variables.
    - Configuration: Extracts `COMMENT_ID`, `MESSAGE`, `POST_ID`, and `FROM_ID`.
    - Key expressions or variables: Uses `$json.body.entry[0].changes[0].value` paths.
    - Input and output connections: Input from `Webhook`; output to `Filter: Is Comment Event`.
  - **Filter: Is Comment Event**
    - Type and technical role: `n8n-nodes-base.if` — Validates whether the extracted message field is non-empty.
    - Configuration: Checks if `MESSAGE` is not empty.
    - Input and output connections: Input from `Extract Comment Data`; outputs to `Filter: Ignore Page Self-Comments` (true) and `End / Not a Comment` (false).
  - **End / Not a Comment**
    - Type and technical role: `n8n-nodes-base.noOp` — Termination point for non-comment webhook events.
    - Input and output connections: Input from `Filter: Is Comment Event`.
  - **Filter: Ignore Page Self-Comments**
    - Type and technical role: `n8n-nodes-base.if` — Prevents infinite reply loops by checking if the commenter ID matches the page ID.
    - Configuration: `notEquals` condition between sender ID and page ID.
    - Key expressions or variables: Compares `={{ $('Webhook').item.json.body.entry[0].changes[0].value.from.id }}` with `={{ $('Webhook').item.json.body.entry[0].id }}`.
    - Input and output connections: Input from `Filter: Is Comment Event`; outputs to `Get Post Content` (true) and `End / Ignore Self-Comment` (false).
  - **End / Ignore Self-Comment**
    - Type and technical role: `n8n-nodes-base.noOp` — Termination point for page self-comments.
    - Input and output connections: Input from `Filter: Ignore Page Self-Comments`.
  - **Get Post Content**
    - Type and technical role: `n8n-nodes-base.facebookGraphApi` — Fetches original post text and metadata from Meta Graph API.
    - Configuration: Graph API version `v23.0`, requesting fields `message` and `created_time`.
    - Key expressions or variables: Node target `={{ $json.POST_ID }}`.
    - Credentials: Uses Facebook Graph account.
    - Input and output connections: Input from `Filter: Ignore Page Self-Comments`; output to `AI Agent`.
  - **AI Agent**
    - Type and technical role: `@n8n/n8n-nodes-langchain.agent` — Core reasoning engine for generating customer support replies.
    - Configuration: System prompt defines a polite social media customer support persona constrained to concise, relevant responses without hallucination.
    - Key expressions or variables: Prompt templates combine post content and user comment data.
    - Input and output connections: Connected to Language Model, Memory, and Tool inputs; output connects to `Send Reply Comment`.
  - **OpenRouter Chat Model**
    - Type and technical role: `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — Language model provider.
    - Configuration: Uses model `inclusionai/ling-3.0-flash-sante:free`.
    - Credentials: Uses OpenRouter account.
    - Input and output connections: Connects to AI Agent (`ai_languageModel`).
  - **Postgres Chat Memory**
    - Type and technical role: `@n8n/n8n-nodes-langchain.memoryPostgresChat` — Maintains conversation history across threads.
    - Configuration: Uses custom session key structure (`Post ID + User ID`).
    - Key expressions or variables: `={{ $('Extract Comment Data').item.json.POST_ID }}_{{ $('Extract Comment Data').item.json.FROM_ID }}`.
    - Credentials: Uses Postgres account.
    - Input and output connections: Connects to AI Agent (`ai_memory`).
  - **Search knowledge base**
    - Type and technical role: `@n8n/n8n-nodes-langchain.vectorStoreSupabase` — RAG tool allowing the agent to search internal documents.
    - Configuration: Mode set to `retrieve-as-tool`, querying Supabase table `documents` with `topK` set to 10.
    - Credentials: Uses Supabase account.
    - Input and output connections: Connects to AI Agent (`ai_tool`) and Embeddings model (`ai_embedding`).
  - **Embed the query**
    - Type and technical role: `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — Generates vector embeddings for user search queries.
    - Credentials: Uses OpenAI API account.
    - Input and output connections: Connects to `Search knowledge base` (`ai_embedding`).
  - **Send Reply Comment**
    - Type and technical role: `n8n-nodes-base.facebookGraphApi` — Publishes the AI-generated response as a threaded reply to the comment.
    - Configuration: HTTP POST request to edge `comments`, Graph API version `v23.0`.
    - Key expressions or variables: Target node `={{ $('Extract Comment Data').item.json.COMMENT_ID }}`, message body `={{ $json.output }}`.
    - Credentials: Uses Facebook Graph account.
    - Input and output connections: Input from `AI Agent`.

#### 1.3 Automated Google Drive Ingestion (New & Updated Files)
- **Overview:** Monitors Google Drive folders for new or modified PDF files, extracts content, embeds it via OpenAI, and upserts vectors into Supabase.
- **Nodes Involved:**
  - `New PDF added to Drive folder`
  - `Download the PDF`
  - `Extract text from PDF`
  - `Tag with file ID and name 01`
  - `Index into vector database`
  - `Split into document chunks 01`
  - `Embeddings OpenAI 01`
  - `PDF updated in Drive folder`
  - `Remove outdated vectors`
  - `Download updated PDF`
  - `Extract text from updated PDF`
  - `Tag with file ID and name 02`
  - `Re-index into vector database`
  - `Split into document chunks 02`
  - `Embeddings OpenAI 02`
- **Node Details:**
  - **New PDF added to Drive folder**
    - Type and technical role: `n8n-nodes-base.googleDriveTrigger` — Triggers when a new file is created in a specific folder.
    - Configuration: Watches folder ID `1fGPtZ2kyeDvNGti4_IQ-IRHEq4TaOxhF`, polling every minute.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Outputs to `Download the PDF`.
  - **Download the PDF**
    - Type and technical role: `n8n-nodes-base.googleDrive` — Downloads binary file content.
    - Configuration: Operation set to download with PDF conversion options.
    - Key expressions or variables: File ID `={{ $json.id }}`.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Input from trigger; output to `Extract text from PDF`.
  - **Extract text from PDF**
    - Type and technical role: `n8n-nodes-base.extractFromFile` — Extracts raw text strings from binary PDF data.
    - Configuration: PDF extraction operation.
    - Input and output connections: Input from `Download the PDF`; output to `Tag with file ID and name 01`.
  - **Tag with file ID and name 01**
    - Type and technical role: `n8n-nodes-base.set` — Formats extracted text fields.
    - Configuration: Assigns `text` property.
    - Key expressions or variables: `={{ $json.text }}`.
    - Input and output connections: Input from text extraction; output to `Index into vector database`.
  - **Index into vector database**
    - Type and technical role: `@n8n/n8n-nodes-langchain.vectorStoreSupabase` — Upserts document chunks into Supabase vector store.
    - Configuration: Mode set to `insert`, targeting table `documents`.
    - Credentials: Supabase account.
    - Input and output connections: Connects to text tagger, embeddings model (`ai_embedding`), and document loader (`ai_document`).
  - **Split into document chunks 01**
    - Type and technical role: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` — Splits raw text into semantic chunks and assigns metadata.
    - Configuration: Adds metadata `fileName` (from downloaded file) and `date` (`$now`).
    - Input and output connections: Connects to `Index into vector database` (`ai_document`).
  - **Embeddings OpenAI 01**
    - Type and technical role: `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — Generates embeddings for document chunking and indexing.
    - Credentials: OpenAI API account.
    - Input and output connections: Connects to `Index into vector database` (`ai_embedding`).
  - **PDF updated in Drive folder**
    - Type and technical role: `n8n-nodes-base.googleDriveTrigger` — Triggers when an existing file is updated.
    - Configuration: Watches folder ID `17ku3bQvHuRgIH1nJt_mvhoKeNnR6JM8d`, polling every minute.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Outputs to `Remove outdated vectors`.
  - **Remove outdated vectors**
    - Type and technical role: `n8n-nodes-base.supabase` — Deletes old vector records corresponding to the updated file name.
    - Configuration: Deletes from table `documents` using filter string.
    - Key expressions or variables: `=metadata->>fileName=like.*{{ $json.name }}`.
    - Credentials: Supabase account.
    - Input and output connections: Input from update trigger; output to `Download updated PDF`.
  - **Download updated PDF**
    - Type and technical role: `n8n-nodes-base.googleDrive` — Downloads the modified PDF file.
    - Configuration: Execute once enabled, downloading file by ID.
    - Key expressions or variables: `={{ $('PDF updated in Drive folder').item.json.id }}`.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Input from vector removal; output to `Extract text from updated PDF`.
  - **Extract text from updated PDF**
    - Type and technical role: `n8n-nodes-base.extractFromFile` — Extracts text from the new file version.
    - Input and output connections: Input from download node; output to `Tag with file ID and name 02`.
  - **Tag with file ID and name 02**
    - Type and technical role: `n8n-nodes-base.set` — Formats text assignment.
    - Key expressions or variables: `={{ $json.text }}`.
    - Input and output connections: Input from text extraction; output to `Re-index into vector database`.
  - **Re-index into vector database**
    - Type and technical role: `@n8n/n8n-nodes-langchain.vectorStoreSupabase` — Inserts re-processed chunks into Supabase.
    - Configuration: Mode set to `insert`, targeting table `documents`.
    - Credentials: Supabase account.
    - Input and output connections: Connects to text tagger, document loader, and embeddings model.
  - **Split into document chunks 02**
    - Type and technical role: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` — Chunks text and sets metadata for updated files.
    - Configuration: Metadata assigns current file name and `$now` timestamp.
    - Input and output connections: Connects to `Re-index into vector database` (`ai_document`).
  - **Embeddings OpenAI 02**
    - Type and technical role: `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — Generates vector embeddings for updated document chunks.
    - Credentials: OpenAI API account.
    - Input and output connections: Connects to `Re-index into vector database` (`ai_embedding`).

#### 1.4 Knowledge Base Maintenance (Deleted Files)
- **Overview:** Monitors trash or deletion events in Google Drive and purges matching vector records from Supabase.
- **Nodes Involved:**
  - `PDF moved to trash`
  - `Remove vectors from database`
  - `Confirm file removed from Drive`
- **Node Details:**
  - **PDF moved to trash**
    - Type and technical role: `n8n-nodes-base.googleDriveTrigger` — Triggers upon file creation/placement in the trash or designated folder.
    - Configuration: Watches folder ID `1V_i5KEmm2ocKib3YSC9jUmph6KGmwOnP`, polling every minute.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Outputs to `Remove vectors from database`.
  - **Remove vectors from database**
    - Type and technical role: `n8n-nodes-base.supabase` — Deletes vector embeddings from Supabase matching the deleted file name.
    - Configuration: Deletes from table `documents` with string filter.
    - Key expressions or variables: `=metadata->>fileName=like.*{{ $json.name }}`.
    - Credentials: Supabase account.
    - Input and output connections: Input from trash trigger; output to `Confirm file removed from Drive`.
  - **Confirm file removed from Drive**
    - Type and technical role: `n8n-nodes-base.googleDrive` — Permanently deletes or confirms file removal action.
    - Configuration: Operation set to `deleteFile`.
    - Key expressions or variables: File ID `={{ $('PDF moved to trash').item.json.id }}`.
    - Credentials: Google Drive OAuth2 API.
    - Input and output connections: Input from vector removal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | `n8n-nodes-base.webhook` | Receives Meta webhook events | None | Set Challenge Variable, Extract Comment Data | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Set Challenge Variable | `n8n-nodes-base.set` | Assigns challenge verification variable | Webhook | Respond to Webhook | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Respond to Webhook | `n8n-nodes-base.respondToWebhook` | Returns challenge token to Meta | Set Challenge Variable | None | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Extract Comment Data | `n8n-nodes-base.set` | Normalizes incoming webhook comment payloads | Webhook | Filter: Is Comment Event | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Filter: Is Comment Event | `n8n-nodes-base.if` | Checks if message payload is present | Extract Comment Data | Filter: Ignore Page Self-Comments, End / Not a Comment | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| End / Not a Comment | `n8n-nodes-base.noOp` | Terminates non-comment workflows | Filter: Is Comment Event | None | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Filter: Ignore Page Self-Comments | `n8n-nodes-base.if` | Filters out page self-comments to prevent loops | Filter: Is Comment Event | Get Post Content, End / Ignore Self-Comment | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| End / Ignore Self-Comment | `n8n-nodes-base.noOp` | Terminates self-comment workflows | Filter: Ignore Page Self-Comments | None | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Get Post Content | `n8n-nodes-base.facebookGraphApi` | Retrieves original Facebook post message | Filter: Ignore Page Self-Comments | AI Agent | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | Core AI reasoning and response generator | Get Post Content | Send Reply Comment | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| OpenRouter Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Provides OpenRouter LLM backend | None | AI Agent | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Postgres Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Maintains conversation history per user per post | None | AI Agent | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Search knowledge base | `@n8n/n8n-nodes-langchain.vectorStoreSupabase` | RAG tool for searching internal documentation | Embed the query | AI Agent | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Embed the query | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | Generates vector embeddings for search queries | None | Search knowledge base | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| Send Reply Comment | `n8n-nodes-base.facebookGraphApi` | Posts AI reply as a threaded comment on Facebook | AI Agent | None | Part 1 — AI Facebook Comment Auto-Reply / ENTERPRISE RAG ECOSYSTEM... |
| New PDF added to Drive folder | `n8n-nodes-base.googleDriveTrigger` | Monitors folder for new PDF files | None | Download the PDF | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Download the PDF | `n8n-nodes-base.googleDrive` | Downloads new PDF binary content | New PDF added to Drive folder | Extract text from PDF | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Extract text from PDF | `n8n-nodes-base.extractFromFile` | Extracts text from downloaded PDF | Download the PDF | Tag with file ID and name 01 | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Tag with file ID and name 01 | `n8n-nodes-base.set` | Formats extracted text property | Extract text from PDF | Index into vector database | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Index into vector database | `@n8n/n8n-nodes-langchain.vectorStoreSupabase` | Upserts new vectors into Supabase | Tag with file ID and name 01 | None | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Split into document chunks 01 | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Splits text into chunks and assigns metadata | None | Index into vector database | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| Embeddings OpenAI 01 | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | Generates OpenAI embeddings for new files | None | Index into vector database | Part 2 — Ingest new file / ENTERPRISE RAG ECOSYSTEM... |
| PDF updated in Drive folder | `n8n-nodes-base.googleDriveTrigger` | Monitors folder for updated files | None | Remove outdated vectors | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Remove outdated vectors | `n8n-nodes-base.supabase` | Purges old vectors for modified files | PDF updated in Drive folder | Download updated PDF | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Download updated PDF | `n8n-nodes-base.googleDrive` | Downloads updated PDF content | Remove outdated vectors | Extract text from updated PDF | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Extract text from updated PDF | `n8n-nodes-base.extractFromFile` | Extracts text from updated PDF | Download updated PDF | Tag with file ID and name 02 | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Tag with file ID and name 02 | `n8n-nodes-base.set` | Formats text property for updated files | Extract text from updated PDF | Re-index into vector database | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Re-index into vector database | `@n8n/n8n-nodes-langchain.vectorStoreSupabase` | Upserts re-indexed vectors into Supabase | Tag with file ID and name 02 | None | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Split into document chunks 02 | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Splits text and assigns metadata for updated files | None | Re-index into vector database | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| Embeddings OpenAI 02 | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | Generates OpenAI embeddings for updated files | None | Re-index into vector database | Part 3 — Update existing file / ENTERPRISE RAG ECOSYSTEM... |
| PDF moved to trash | `n8n-nodes-base.googleDriveTrigger` | Monitors folder for deleted/trashed files | None | Remove vectors from database | Part 4 — Delete a file / ENTERPRISE RAG ECOSYSTEM... |
| Remove vectors from database | `n8n-nodes-base.supabase` | Deletes vector embeddings for removed files | PDF moved to trash | Confirm file removed from Drive | Part 4 — Delete a file / ENTERPRISE RAG ECOSYSTEM... |
| Confirm file removed from Drive | `n8n-nodes-base.googleDrive` | Confirms file deletion action | Remove vectors from database | None | Part 4 — Delete a file / ENTERPRISE RAG ECOSYSTEM... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Set Up Credentials
Configure the following credentials in your n8n instance:
1. **OpenRouter API Account:** Required for the AI Agent language model.
2. **OpenAI API:** Required for `Embeddings OpenAI` and `Embed the query` nodes.
3. **Supabase API:** Required for vector store insert and retrieval nodes, as well as database delete nodes. Ensure the `pgvector` extension is enabled and the `documents` table is created following standard LangChain-Supabase schemas.
4. **Postgres:** Required for `Postgres Chat Memory`.
5. **Google Drive OAuth2 API:** Required for all Google Drive triggers and file operations.
6. **Facebook Graph API:** Required with appropriate page management permissions (`pages_manage_posts`, `pages_read_engagement`, `pages_messaging`).

#### Step 2: Build Part 1 (Webhook & AI Auto-Reply Engine)
1. **Webhook Node:** Create a Webhook node with path `ee30223b-1fdd-409d-9302-7d784d42ac1b`, multiple methods enabled, and response mode set to response node.
2. **Challenge Verification Branch:**
   - Create a Set node named `Set Challenge Variable` connected to output index 0 of the Webhook. Assign `query['hub.challenge']` to `={{ $json.query['hub.challenge'] }}`.
   - Connect it to a `Respond to Webhook` node responding with text using `={{ $json.query['hub.challenge'] }}`.
3. **Comment Extraction Branch:**
   - Create a Set node named `Extract Comment Data` connected to output index 1 of the Webhook. Assign `COMMENT_ID`, `MESSAGE`, `POST_ID`, and `FROM_ID` from the webhook body JSON path.
   - Connect to an If node named `Filter: Is Comment Event` checking that `MESSAGE` is not empty.
   - Connect the true branch to an If node named `Filter: Ignore Page Self-Comments` checking that the commenter ID (`from.id`) does not equal the page ID (`entry[0].id`). Connect the false branch to a NoOp node (`End / Not a Comment`).
   - Connect the false branch of the self-comment filter to a NoOp node (`End / Ignore Self-Comment`).
4. **Post Context & AI Processing:**
   - Connect the true branch of `Filter: Ignore Page Self-Comments` to a Facebook Graph API node named `Get Post Content` configured for Graph API `v23.0`, requesting fields `message` and `created_time`, using `={{ $json.POST_ID }}` as the node parameter.
   - Connect `Get Post Content` to an AI Agent node (`@n8n/n8n-nodes-langchain.agent`). Configure its system prompt to act as a polite social media customer support specialist.
   - Attach an OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) to the AI Agent (`ai_languageModel`), selecting model `inclusionai/ling-3.0-flash-sante:free`.
   - Attach a Postgres Chat Memory node to the AI Agent (`ai_memory`) with session key set to `={{ $('Extract Comment Data').item.json.POST_ID }}_{{ $('Extract Comment Data').item.json.FROM_ID }}`.
   - Attach a Supabase vector store tool named `Search knowledge base` (`@n8n/n8n-nodes-langchain.vectorStoreSupabase`) to the AI Agent (`ai_tool`) configured to retrieve as a tool (`topK`: 10) on table `documents`.
   - Attach an OpenAI Embeddings node (`Embed the query`) to `Search knowledge base` (`ai_embedding`).
5. **Response Execution:**
   - Connect the output of the AI Agent to a Facebook Graph API node named `Send Reply Comment` configured with HTTP method `POST`, edge `comments`, node ID `={{ $('Extract Comment Data').item.json.COMMENT_ID }}` and query parameter `message` set to `={{ $json.output }}`.

#### Step 3: Build Part 2 (Automated File Ingestion Pipeline)
1. **Google Drive Trigger:** Create a Google Drive Trigger node named `New PDF added to Drive folder` set to `fileCreated`, polling every minute, watching a specific folder ID (e.g., `1fGPtZ2kyeDvNGti4_IQ-IRHEq4TaOxhF`).
2. **Download & Extract:**
   - Connect to a Google Drive node (`Download the PDF`) set to download mode with ID `={{ $json.id }}` and PDF conversion options.
   - Connect to an Extract From File node (`Extract text from PDF`) set to `pdf` operation.
   - Connect to a Set node (`Tag with file ID and name 01`) assigning `text` to `={{ $json.text }}`.
3. **Vector Indexing:**
   - Connect to a Supabase vector store node (`Index into vector database`) set to `insert` on table `documents`.
   - Attach a Document Default Data Loader node (`Split into document chunks 01`) to its `ai_document` input, configuring metadata `fileName` as `={{ $('Download the PDF').item.json.name }}` and `date` as `={{ $now }}`.
   - Attach an OpenAI Embeddings node (`Embeddings OpenAI 01`) to its `ai_embedding` input.

#### Step 4: Build Part 3 (Knowledge Base Version Control)
1. **Drive Update Trigger:** Create a Google Drive Trigger node named `PDF updated in Drive folder` set to `fileUpdated`, polling every minute, watching folder ID `17ku3bQvHuRgIH1nJt_mvhoKeNnR6JM8d`.
2. **Vector Purging & Re-Indexing:**
   - Connect to a Supabase node (`Remove outdated vectors`) configured to delete records from table `documents` where filter string is `=metadata->>fileName=like.*{{ $json.name }}`.
   - Connect to a Google Drive node (`Download updated PDF`) configured with `executeOnce: true` and file ID `={{ $('PDF updated in Drive folder').item.json.id }}` with PDF conversion options.
   - Connect to an Extract From File node (`Extract text from updated PDF`).
   - Connect to a Set node (`Tag with file ID and name 02`) assigning `text` to `={{ $json.text }}`.
   - Connect to a Supabase vector store node (`Re-index into vector database`) set to `insert` on table `documents`.
   - Attach a Document Default Data Loader node (`Split into document chunks 02`) to its `ai_document` input with metadata `fileName` as `={{ $('PDF updated in Drive folder').item.json.name }}` and `date` as `={{ $now }}`.
   - Attach an OpenAI Embeddings node (`Embeddings OpenAI 02`) to its `ai_embedding` input.

#### Step 5: Build Part 4 (Automated Cleanup & Sync)
1. **Drive Deletion Trigger:** Create a Google Drive Trigger node named `PDF moved to trash` set to `fileCreated`, polling every minute, watching trash folder ID `1V_i5KEmm2ocKib3YSC9jUmph6KGmwOnP`.
2. **Database Scrubbing:**
   - Connect to a Supabase node (`Remove vectors from database`) set to delete records from table `documents` where filter string is `=metadata->>fileName=like.*{{ $json.name }}`.
   - Connect to a Google Drive node (`Confirm file removed from Drive`) set to `deleteFile` operation with file ID `={{ $('PDF moved to trash').item.json.id }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Supabase LangChain AI Integration Guide | [Supabase LangChain Guide](https://supabase.com/docs/guides/ai/langchain?database-method=sql) |
| n8n Webhook Node Documentation | [n8n Webhook Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook) |