Sync WordPress and WooCommerce content with Pinecone using OpenAI embeddings

https://n8nworkflows.xyz/workflows/sync-wordpress-and-woocommerce-content-with-pinecone-using-openai-embeddings-20047


# Sync WordPress and WooCommerce content with Pinecone using OpenAI embeddings

### 1. Workflow Overview

This workflow automates the synchronization of published WordPress and WooCommerce pages and blog posts into a Pinecone vector index. Its primary purpose is to maintain an up-to-date knowledge base for AI assistants by ingesting new or modified content, removing deleted articles, and tracking synchronization state via an n8n Data Table. 

The execution logic is structured into the following functional blocks:

- **1.1 Input Reception & Configuration:** Manually or schedule-triggered execution paths that pass control to a central configuration node.
- **1.2 Content Inventory Retrieval:** Fetches lightweight summaries (ID and modification date) of up to 100 published pages and 100 published posts from the WordPress REST API.
- **1.3 Change Detection & Orphan Identification:** Compares live WordPress inventory with the `indexed_wc_blog_v1` n8n Data Table to discover modified items and identify orphaned records deleted from WordPress.
- **1.4 Orphan Cleanup:** Iterates through deleted entries to remove their vectors from Pinecone and clear their records from the Data Table.
- **1.5 Batch Content Retrieval:** Splits changed item IDs by type, fetches full content in batches from WordPress, and combines them into an actionable processing array.
- **1.6 Vectorization & Storage Iteration:** Loops through each item, formats the text by stripping HTML, purges existing Pinecone chunks, generates embeddings using OpenAI, and upserts the new vectors and sync metadata.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Acts as the entry point for the workflow, offering both manual testing capabilities and a daily scheduled cron execution, feeding parameters into a shared variable node.
- **Nodes Involved:** `Manual Trigger for Content Sync`, `Scheduled Static Content Sync`, `Set Static Content Settings`.
- **Node Details:**
  - **Manual Trigger for Content Sync**
    - Type: `n8n-nodes-base.manualTrigger`
    - Technical Role: Initiates execution manually on demand.
    - Configuration: Default settings.
    - Input Connections: None (Entry Point).
    - Output Connections: `Set Static Content Settings`.
    - Edge Cases: None.
  - **Scheduled Static Content Sync**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Technical Role: Triggers execution on a daily cron schedule (`0 4 * * *`).
    - Configuration: Cron expression set to run daily at 4:00 AM.
    - Input Connections: None (Entry Point).
    - Output Connections: `Set Static Content Settings`.
    - Edge Cases: Timezone discrepancies if n8n server timezone differs from expectations.
  - **Set Static Content Settings**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Establishes global environment variables for WooCommerce, Pinecone, and batch sizing.
    - Configuration: Assigns `wooBaseUrl`, `wooCurrency`, `pineconeIndexName`, `pineconeIndexHost`, `pineconeNamespace`, and `staticBatchSize`.
    - Expressions Used: Static string and number assignments.
    - Input Connections: `Manual Trigger for Content Sync`, `Scheduled Static Content Sync`.
    - Output Connections: `Fetch Static Pages`.
    - Edge Cases: Invalid URL structures or incorrect Pinecone host settings will cause downstream API request failures.

#### 2.2 Content Inventory Retrieval
- **Overview:** Queries the WordPress REST API to retrieve lightweight metadata (IDs and modification dates) for all published pages and posts.
- **Nodes Involved:** `Fetch Static Pages`, `Fetch Blog Posts`, `Combine Pages and Posts`.
- **Node Details:**
  - **Fetch Static Pages**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches up to 100 published WordPress pages.
    - Configuration: GET request to `/wp-json/wp/v2/pages` with query parameters `per_page=100`, `status=publish`, and `_fields=id,modified`. Uses WooCommerce API credentials.
    - Input Connections: `Set Static Content Settings`.
    - Output Connections: `Fetch Blog Posts`.
    - Edge Cases: Authentication failures or network timeouts (set to 30,000ms).
  - **Fetch Blog Posts**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches up to 100 published WordPress blog posts.
    - Configuration: GET request to `/wp-json/wp/v2/posts` with query parameters `per_page=100`, `status=publish`, and `_fields=id,modified`. Uses WooCommerce API credentials.
    - Input Connections: `Fetch Static Pages`.
    - Output Connections: `Combine Pages and Posts`.
    - Edge Cases: Authentication errors or timeouts.
  - **Combine Pages and Posts**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Normalizes and merges page and post arrays into a single inventory list.
    - Configuration: Custom JavaScript flattening arrays and assigning document kinds (`page` or `post`).
    - Input Connections: `Fetch Blog Posts`.
    - Output Connections: `Get Deduped Content Entries`.
    - Edge Cases: Unexpected payload formats from WordPress.

#### 2.3 Change Detection & Orphan Identification
- **Overview:** Compares live WordPress items against records stored in the n8n Data Table to isolate items requiring updates and identify orphaned content removed from the source site.
- **Nodes Involved:** `Get Deduped Content Entries`, `Filter Content for Processing`, `Identify Deleted Entries`.
- **Node Details:**
  - **Get Deduped Content Entries**
    - Type: `n8n-nodes-base.dataTable`
    - Technical Role: Retrieves all existing records from the synchronization tracking table.
    - Configuration: Operation set to `get`, returns all rows from Data Table `indexed_wc_blog_v1`. Always outputs data.
    - Input Connections: `Combine Pages and Posts`.
    - Output Connections: `Filter Content for Processing`, `Identify Deleted Entries`.
    - Edge Cases: Missing Data Table will cause errors if it was not created prior to execution.
  - **Filter Content for Processing**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Identifies new or modified pages and posts based on modification timestamps, limited by `staticBatchSize`.
    - Configuration: Custom JavaScript comparing live modified dates against table records.
    - Input Connections: `Get Deduped Content Entries`.
    - Output Connections: `Separate Content IDs by Type`.
    - Edge Cases: Empty input lists or sync discrepancies.
  - **Identify Deleted Entries**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Compares table records against live items to detect deleted content with safety guards against empty payloads or pagination limits.
    - Configuration: Custom JavaScript checking for orphan IDs while skipping deletion if list length reaches the 100-item safety limit.
    - Input Connections: `Get Deduped Content Entries`.
    - Output Connections: `Loop Over Deleted Entries`.
    - Edge Cases: Safety triggers prevent accidental mass deletion during network outages.

#### 2.4 Orphan Cleanup
- **Overview:** Iterates through deleted content identifiers, purges their vector embeddings from Pinecone, and removes their corresponding tracking records from the Data Table.
- **Nodes Involved:** `Loop Over Deleted Entries`, `Remove Deleted Vectors`, `Delete Entry from Table`.
- **Node Details:**
  - **Loop Over Deleted Entries**
    - Type: `n8n-nodes-base.splitInBatches`
    - Technical Role: Iterates sequentially through orphaned entries for cleanup.
    - Input Connections: `Identify Deleted Entries`, `Delete Entry from Table`.
    - Output Connections: `Content Sync Finished Marker` (when finished), `Remove Deleted Vectors` (loop item).
    - Edge Cases: Unhandled iteration failures.
  - **Remove Deleted Vectors**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Deletes vectors matching a specific `fileId` from the Pinecone vector index.
    - Configuration: POST request to Pinecone vector delete endpoint with `fileId` filter and namespace parameters. Uses Pinecone API credentials. `onError` set to continue regular output.
    - Input Connections: `Loop Over Deleted Entries`.
    - Output Connections: `Delete Entry from Table`.
    - Edge Cases: API version header mismatches (`X-Pinecone-Api-Version: 2025-10`).
  - **Delete Entry from Table**
    - Type: `n8n-nodes-base.dataTable`
    - Technical Role: Removes tracking records for deleted items from the Data Table.
    - Configuration: Operation set to `deleteRows` on `indexed_wc_blog_v1` where `productId` matches the current loop item.
    - Input Connections: `Remove Deleted Vectors`.
    - Output Connections: `Loop Over Deleted Entries`.
    - Edge Cases: Row mismatch or table locking.

#### 2.5 Batch Content Retrieval
- **Overview:** Groups changed item IDs by content type, fetches their full content payload from WordPress, and unifies them into a single processing batch.
- **Nodes Involved:** `Separate Content IDs by Type`, `Fetch Full Batch Pages`, `Fetch Full Batch Posts`, `Combine Full Batch Data`.
- **Node Details:**
  - **Separate Content IDs by Type**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Splits filtered item objects into comma-separated strings of page IDs and post IDs.
    - Input Connections: `Filter Content for Processing`.
    - Output Connections: `Fetch Full Batch Pages`.
    - Edge Cases: Empty ID arrays fallback to `'0'`.
  - **Fetch Full Batch Pages**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves full content bodies for the specified batch of page IDs.
    - Configuration: GET request to `/wp-json/wp/v2/pages` using `include` query parameter. `onError` set to continue regular output.
    - Input Connections: `Separate Content IDs by Type`.
    - Output Connections: `Fetch Full Batch Posts`.
    - Edge Cases: Timeouts or missing IDs.
  - **Fetch Full Batch Posts**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves full content bodies for the specified batch of post IDs.
    - Configuration: GET request to `/wp-json/wp/v2/posts` using `include` query parameter. `onError` set to continue regular output.
    - Input Connections: `Fetch Full Batch Pages`.
    - Output Connections: `Combine Full Batch Data`.
    - Edge Cases: Timeouts or missing IDs.
  - **Combine Full Batch Data**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Normalizes and merges full page and post payloads, decoding HTML entities.
    - Input Connections: `Fetch Full Batch Posts`.
    - Output Connections: `Loop Over Static Content`.
    - Edge Cases: Malformed HTML content.

#### 2.6 Vectorization & Storage Iteration
- **Overview:** Iterates through each fetched item, strips HTML, deletes legacy vectors in Pinecone, chunks the text, generates OpenAI embeddings, stores the new vectors in Pinecone, and records sync metadata.
- **Nodes Involved:** `Loop Over Static Content`, `Format Static Content`, `Remove Old Content Chunks`, `Load Content Documents`, `Split Content Text`, `Generate OpenAI Embeddings`, `Store Content in Pinecone`, `Save Content Sync Data`, `Static Content Processing Complete`, `Content Sync Finished Marker`.
- **Node Details:**
  - **Loop Over Static Content**
    - Type: `n8n-nodes-base.splitInBatches`
    - Technical Role: Iterates through the batch of changed content items one by one.
    - Input Connections: `Combine Full Batch Data`, `Save Content Sync Data`.
    - Output Connections: `Static Content Processing Complete` & `Content Sync Finished Marker` (when finished), `Format Static Content` (loop item).
    - Edge Cases: Empty batches.
  - **Format Static Content**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Strips HTML tags, decodes entities, builds document text, and assigns metadata variables.
    - Configuration: Custom JavaScript HTML stripper and metadata constructor.
    - Input Connections: `Loop Over Static Content`.
    - Output Connections: `Remove Old Content Chunks`.
    - Edge Cases: Unclosed HTML tags.
  - **Remove Old Content Chunks**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Deletes existing Pinecone vector chunks associated with the current `fileId`.
    - Configuration: POST request to Pinecone vector delete endpoint with `fileId` filter. `onError` set to continue regular output.
    - Input Connections: `Format Static Content`.
    - Output Connections: `Store Content in Pinecone`.
    - Edge Cases: First-time indexing where no prior vectors exist (handled by `continueRegularOutput`).
  - **Load Content Documents**
    - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`
    - Technical Role: Prepares raw text and metadata attributes for vectorization.
    - Configuration: Maps JSON text and applies metadata values (`title`, `source`, `fileId`, `entity`, `doc_type`, `object_id`, `link`).
    - Input Connections: `Split Content Text` (AI text splitter), `Format Static Content` (via expressions).
    - Output Connections: `Store Content in Pinecone` (AI document).
    - Edge Cases: Missing metadata properties.
  - **Split Content Text**
    - Type: `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`
    - Technical Role: Splits document text into manageable chunks for embedding generation.
    - Configuration: Chunk size set to `1500` characters with a chunk overlap of `300` characters.
    - Input Connections: None directly (connected via LangChain AI input).
    - Output Connections: `Load Content Documents`.
    - Edge Cases: Extremely short text or oversize paragraphs.
  - **Generate OpenAI Embeddings**
    - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`
    - Technical Role: Generates vector embeddings for text chunks using OpenAI.
    - Configuration: Uses OpenAI API credentials. Default embedding model settings.
    - Input Connections: None directly (connected via LangChain AI input).
    - Output Connections: `Store Content in Pinecone`.
    - Edge Cases: OpenAI rate limits or API key invalidity.
  - **Store Content in Pinecone**
    - Type: `@n8n/n8n-nodes-langchain.vectorStorePinecone`
    - Technical Role: Inserts vectorized text chunks and metadata into the configured Pinecone vector store.
    - Configuration: Mode set to `insert`, namespace configured via expression, and Pinecone index referenced dynamically. Uses Pinecone API credentials.
    - Input Connections: `Load Content Documents`, `Remove Old Content Chunks`, `Generate OpenAI Embeddings`.
    - Output Connections: `Save Content Sync Data`.
    - Edge Cases: Dimension mismatches between embedding model (1536) and Pinecone index configuration.
  - **Save Content Sync Data**
    - Type: `n8n-nodes-base.dataTable`
    - Technical Role: Upserts synchronization tracking records into the n8n Data Table.
    - Configuration: Operation set to `upsert` on Data Table `indexed_wc_blog_v1`. Maps `productId` (`fileId`), `indexedAt` (`$now.toISO()`), and `dateModified`. Filter matches on `productId`.
    - Input Connections: `Store Content in Pinecone`.
    - Output Connections: `Loop Over Static Content`.
    - Edge Cases: Data Table constraints or missing columns.
  - **Static Content Processing Complete**
    - Type: `n8n-nodes-base.noOp`
    - Technical Role: Marker node indicating static content batch processing has concluded.
    - Input Connections: `Loop Over Static Content`.
    - Output Connections: None.
    - Edge Cases: None.
  - **Content Sync Finished Marker**
    - Type: `n8n-nodes-base.noOp`
    - Technical Role: Marker node indicating the overall content synchronization run has finished.
    - Input Connections: `Loop Over Static Content`, `Loop Over Deleted Entries`.
    - Output Connections: None.
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & Setup Guide | None | None | ## WooCommerce Blog-Pinecone... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Start and configure sync... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Fetch content lists... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Detect content changes... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Remove deleted content... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Retrieve changed content... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Iterate content batch... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Prepare item vectors... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Embed and store chunks... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Section Documentation | None | None | ## Record sync entry... |
| Fetch Static Pages | n8n-nodes-base.httpRequest | Fetch published WordPress pages | Set Static Content Settings | Fetch Blog Posts | ## Fetch content lists |
| Fetch Blog Posts | n8n-nodes-base.httpRequest | Fetch published WordPress posts | Fetch Static Pages | Combine Pages and Posts | ## Fetch content lists |
| Combine Pages and Posts | n8n-nodes-base.code | Normalize and merge page and post lists | Fetch Blog Posts | Get Deduped Content Entries | ## Fetch content lists |
| Static Content Processing Complete | n8n-nodes-base.noOp | Marks completion of static content batch | Loop Over Static Content | None | ## Iterate content batch |
| Content Sync Finished Marker | n8n-nodes-base.noOp | Marks completion of entire sync run | Loop Over Static Content, Loop Over Deleted Entries | None | ## Iterate content batch |
| Loop Over Static Content | n8n-nodes-base.splitInBatches | Iterates through changed content items | Combine Full Batch Data, Save Content Sync Data | Static Content Processing Complete, Content Sync Finished Marker, Format Static Content | ## Iterate content batch |
| Format Static Content | n8n-nodes-base.code | Strip HTML and prepare item metadata | Loop Over Static Content | Remove Old Content Chunks | ## Prepare item vectors |
| Remove Old Content Chunks | n8n-nodes-base.httpRequest | Delete old Pinecone vectors for item | Format Static Content | Store Content in Pinecone | ## Prepare item vectors |
| Manual Trigger for Content Sync | n8n-nodes-base.manualTrigger | Manual execution entry point | None | Set Static Content Settings | ## Start and configure sync |
| Set Static Content Settings | n8n-nodes-base.set | Assign configuration variables | Manual Trigger for Content Sync, Scheduled Static Content Sync | Fetch Static Pages | ## Start and configure sync |
| Get Deduped Content Entries | n8n-nodes-base.dataTable | Retrieve synchronization tracking table records | Combine Pages and Posts | Filter Content for Processing, Identify Deleted Entries | ## Detect content changes |
| Filter Content for Processing | n8n-nodes-base.code | Select items requiring update based on timestamps | Get Deduped Content Entries | Separate Content IDs by Type | ## Detect content changes |
| Generate OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generate vector embeddings via OpenAI | None | Store Content in Pinecone | ## Embed and store chunks |
| Split Content Text | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split document text into chunks | None | Load Content Documents | ## Embed and store chunks |
| Load Content Documents | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Load documents with metadata for vector store | Split Content Text, Format Static Content | Store Content in Pinecone | ## Embed and store chunks |
| Store Content in Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone | Store text chunks in Pinecone vector store | Load Content Documents, Remove Old Content Chunks, Generate OpenAI Embeddings | Save Content Sync Data | ## Embed and store chunks |
| Save Content Sync Data | n8n-nodes-base.dataTable | Upsert sync metadata into Data Table | Store Content in Pinecone | Loop Over Static Content | ## Record sync entry |
| Separate Content IDs by Type | n8n-nodes-base.code | Group changed content IDs by type | Filter Content for Processing | Fetch Full Batch Pages | ## Retrieve changed content |
| Fetch Full Batch Pages | n8n-nodes-base.httpRequest | Fetch full content for changed pages | Separate Content IDs by Type | Fetch Full Batch Posts | ## Retrieve changed content |
| Fetch Full Batch Posts | n8n-nodes-base.httpRequest | Fetch full content for changed posts | Fetch Full Batch Pages | Combine Full Batch Data | ## Retrieve changed content |
| Combine Full Batch Data | n8n-nodes-base.code | Normalize and combine full batch content | Fetch Full Batch Posts | Loop Over Static Content | ## Retrieve changed content |
| Scheduled Static Content Sync | n8n-nodes-base.scheduleTrigger | Scheduled execution trigger (daily 4:00 AM) | None | Set Static Content Settings | ## Start and configure sync |
| Identify Deleted Entries | n8n-nodes-base.code | Detect orphaned items deleted from WordPress | Get Deduped Content Entries | Loop Over Deleted Entries | ## Detect content changes |
| Loop Over Deleted Entries | n8n-nodes-base.splitInBatches | Iterate through deleted items for cleanup | Identify Deleted Entries, Delete Entry from Table | Content Sync Finished Marker, Remove Deleted Vectors | ## Remove deleted content |
| Remove Deleted Vectors | n8n-nodes-base.httpRequest | Remove deleted item vectors from Pinecone | Loop Over Deleted Entries | Delete Entry from Table | ## Remove deleted content |
| Delete Entry from Table | n8n-nodes-base.dataTable | Delete sync table row for removed item | Remove Deleted Vectors | Loop Over Deleted Entries | ## Remove deleted content |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Data Table:**
   - Create a new n8n Data Table named `indexed_wc_blog_v1` with three text columns: `productId`, `indexedAt`, and `dateModified`.
2. **Create Triggers and Settings:**
   - Add a **Manual Trigger for Content Sync** (`n8n-nodes-base.manualTrigger`).
   - Add a **Scheduled Static Content Sync** (`n8n-nodes-base.scheduleTrigger`) set to run daily via cron expression `0 4 * * *`.
   - Add a **Set Static Content Settings** node (`n8n-nodes-base.set`). Assign variables: `wooBaseUrl` (string), `wooCurrency` (string), `pineconeIndexName` (string), `pineconeIndexHost` (string), `pineconeNamespace` (string, e.g., `"default"`), and `staticBatchSize` (number, e.g., `5`). Connect both triggers to this node.
3. **Fetch Content Lists:**
   - Add **Fetch Static Pages** (`n8n-nodes-base.httpRequest`). Configure GET request to `={{ $('Set Static Content Settings').first().json.wooBaseUrl.replace(/\/+$/, '') + '/wp-json/wp/v2/pages' }}` with query parameters `per_page=100`, `status=publish`, and `_fields=id,modified`. Set authentication to WooCommerce API credentials. Connect `Set Static Content Settings` to this node.
   - Add **Fetch Blog Posts** (`n8n-nodes-base.httpRequest`). Configure similarly to the pages node but targeting `/wp-json/wp/v2/posts`. Connect `Fetch Static Pages` to this node.
   - Add **Combine Pages and Posts** (`n8n-nodes-base.code`) using the JavaScript provided in the JSON specification to normalize and merge pages and posts. Connect `Fetch Blog Posts` to this node.
4. **Detect Changes and Orphans:**
   - Add **Get Deduped Content Entries** (`n8n-nodes-base.dataTable`) with operation `get` on Data Table `indexed_wc_blog_v1`. Connect `Combine Pages and Posts` to this node.
   - Add **Filter Content for Processing** (`n8n-nodes-base.code`) to select new or modified items based on modification dates and `staticBatchSize`. Connect `Get Deduped Content Entries` to this node.
   - Add **Identify Deleted Entries** (`n8n-nodes-base.code`) to detect orphaned records with safety checks against empty or 100-item limit lists. Connect `Get Deduped Content Entries` to this node.
5. **Handle Deleted Entries (Orphan Cleanup):**
   - Add **Loop Over Deleted Entries** (`n8n-nodes-base.splitInBatches`). Connect `Identify Deleted Entries` to its input.
   - Add **Remove Deleted Vectors** (`n8n-nodes-base.httpRequest`). Configure POST request to `={{ 'https://' + $('Set Static Content Settings').first().json.pineconeIndexHost.replace(/^https?:\/\//, '') + '/vectors/delete' }}` with Pinecone API credentials and header `X-Pinecone-Api-Version: 2025-10`. Set `onError` to continue regular output. Connect `Loop Over Deleted Entries` (item output) to this node.
   - Add **Delete Entry from Table** (`n8n-nodes-base.dataTable`) with operation `deleteRows` on `indexed_wc_blog_v1` filtering by `productId`. Connect `Remove Deleted Vectors` to this node and loop back to `Loop Over Deleted Entries`.
6. **Retrieve Changed Content Batches:**
   - Add **Separate Content IDs by Type** (`n8n-nodes-base.code`) to split changed item IDs. Connect `Filter Content for Processing` to this node.
   - Add **Fetch Full Batch Pages** (`n8n-nodes-base.httpRequest`) targeting `/wp-json/wp/v2/pages` with query parameters `include` and `per_page=100` using WooCommerce API credentials. Set `onError` to continue regular output. Connect `Separate Content IDs by Type` to this node.
   - Add **Fetch Full Batch Posts** (`n8n-nodes-base.httpRequest`) targeting `/wp-json/wp/v2/posts` with query parameters `include` and `per_page=100` using WooCommerce API credentials. Set `onError` to continue regular output. Connect `Fetch Full Batch Pages` to this node.
   - Add **Combine Full Batch Data** (`n8n-nodes-base.code`) to normalize full batch payloads. Connect `Fetch Full Batch Posts` to this node.
7. **Iterate, Vectorize, and Store Chunks:**
   - Add **Loop Over Static Content** (`n8n-nodes-base.splitInBatches`). Connect `Combine Full Batch Data` to its input.
   - Add **Format Static Content** (`n8n-nodes-base.code`) to strip HTML, construct text, and assign metadata. Connect `Loop Over Static Content` (item output) to this node.
   - Add **Remove Old Content Chunks** (`n8n-nodes-base.httpRequest`) configured identically to the delete vector request in step 5, filtering by `fileId`. Set `onError` to continue regular output. Connect `Format Static Content` to this node.
   - Add LangChain nodes:
     - **Split Content Text** (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`) with chunk size `1500` and chunk overlap `300`. Connect its `ai_textSplitter` output to **Load Content Documents**.
     - **Load Content Documents** (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) configured with custom expression metadata mapping. Connect its `ai_document` output to **Store Content in Pinecone**.
     - **Generate OpenAI Embeddings** (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`) using OpenAI credentials. Connect its `ai_embedding` output to **Store Content in Pinecone**.
     - **Store Content in Pinecone** (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) in `insert` mode using Pinecone credentials and index settings. Connect `Remove Old Content Chunks` to its main input.
   - Add **Save Content Sync Data** (`n8n-nodes-base.dataTable`) with operation `upsert` on `indexed_wc_blog_v1`, mapping `productId`, `indexedAt`, and `dateModified`. Connect `Store Content in Pinecone` to this node, and loop back to `Loop Over Static Content`.
   - Add **Static Content Processing Complete** (`n8n-nodes-base.noOp`) and **Content Sync Finished Marker** (`n8n-nodes-base.noOp`) connected to the completion output of the batch loops.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Syncs WordPress/WooCommerce static pages and blog posts into a Pinecone vector index for AI assistant search. | Workflow Purpose |
| Requires a Pinecone index with a dimension of 1536 (default OpenAI embedding size). | Technical Requirement |
| Safety guard prevents deletion of content categories if WordPress returns empty lists or hits the 100-item pagination limit. | Safety Mechanism |