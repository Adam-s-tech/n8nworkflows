Generate SEO site architecture from keyword lists with Claude and WhatsApp

https://n8nworkflows.xyz/workflows/generate-seo-site-architecture-from-keyword-lists-with-claude-and-whatsapp-20055


# Generate SEO site architecture from keyword lists with Claude and WhatsApp

### 1. Workflow Overview

This workflow ingests large volumes of SEO keywords (up to 10,000 entries), groups and categorizes them algorithmically and via Anthropic Claude AI, and constructs a structured pillar–cluster–supporting-page site architecture. It optionally saves the generated blueprint to a CMS endpoint and transmits a summary notification via WhatsApp Business Cloud before returning the complete site structure as a webhook response.

The workflow logic is categorized into the following functional blocks:

- **1.1 Input Reception & Configuration:** Handles incoming webhooks or manual test triggers, parses and de-duplicates raw keyword datasets, caps entries according to configuration limits, and loads execution parameters.
- **1.2 Algorithmic Coarse Clustering:** Removes stopwords and terms from keywords, maps them into raw thematic buckets using shared token overlaps, selects primary hub keywords, and prepares a batch queue.
- **1.3 AI Batch Labeling & Pillar Consolidation:** Iteratively processes batches through Anthropic Claude to assign human-readable topics and proposed parent pillars, subsequently consolidating near-duplicate labels into unified canonical topics.
- **1.4 Architecture Assembly & Review:** Applies the consolidated pillar mapping to construct a structured URL-friendly hierarchy tree, runs an AI review agent to evaluate balance, and compiles the final report payload.
- **1.5 Persistence, Notification, & Response:** Dispatches the final blueprint to a content-planning API, sends a completion notification via WhatsApp, and returns the full JSON report to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block establishes entry points for execution, ingests payloads (either JSON arrays or CSV text) or fallback test samples, normalizes data, and injects runtime configurations.

- **Webhook - Ingest Keyword List**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` — Acts as the production API entry point accepting POST requests.
  - **Configuration:** Path set to `site-architecture-ingest`, expecting POST method with JSON body payloads.
  - **Key Expressions:** None.
  - **Connections:** Input: None (Trigger); Output: `Code 1 - Ingest & Normalize Keywords`.
  - **Edge Cases / Failures:** Malformed JSON payloads or missing headers.

- **Manual Trigger - Test Run**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Provides a manual button click to initiate test executions.
  - **Configuration:** Default manual execution parameters.
  - **Key Expressions:** None.
  - **Connections:** Input: None (Trigger); Output: `Code 1 - Ingest & Normalize Keywords`.
  - **Edge Cases / Failures:** None.

- **Code 1 - Ingest & Normalize Keywords**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript execution node that sanitizes inputs, handles CSV splits, de-duplicates items case-insensitively, applies keyword caps, and falls back to a built-in sample list if empty.
  - **Configuration:** Custom parsing script for payload detection.
  - **Key Expressions:** Extracts parameters from `$input.first().json`.
  - **Connections:** Input: `Webhook - Ingest Keyword List`, `Manual Trigger - Test Run`; Output: `Set - Config`.
  - **Edge Cases / Failures:** Invalid CSV structures or empty inputs without fallback data.

- **Set - Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Defines global workflow variables.
  - **Configuration:** Assigns operational parameters including `batchSize` (50), `maxSampleKeywordsPerClusterForPrompt` (12), `anthropicModel` (`claude-sonnet-4-6`), CMS save endpoint, and WhatsApp identification values.
  - **Key Expressions:** Static assignment values.
  - **Connections:** Input: `Code 1 - Ingest & Normalize Keywords`; Output: `Code 2 - Coarse Cluster Keywords By Term Overlap`.
  - **Edge Cases / Failures:** Missing environment constants or misconfigured endpoint strings.

---

#### 2.2 Algorithmic Coarse Clustering
This block performs non-AI grouping of terms based on linguistic patterns, selects primary hub keywords, and organizes data into a sequential queue for downstream processing.

- **Code 2 - Coarse Cluster Keywords By Term Overlap**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Strips stop-words and numerical tokens, tokenizes keywords, and groups them by shared core terms.
  - **Configuration:** Script containing defined stop-word exclusion lists.
  - **Key Expressions:** Reads normalized keywords from state objects.
  - **Connections:** Input: `Set - Config`; Output: `Code 3 - Compute Primary & Supporting Keywords Per Cluster`.
  - **Edge Cases / Failures:** Over-aggressive stripping resulting in empty token signatures.

- **Code 3 - Compute Primary & Supporting Keywords Per Cluster**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Identifies the best hub keyword per cluster based on search volume or shortest phrasing length.
  - **Configuration:** Scoring algorithm implementation.
  - **Key Expressions:** Compares `searchVolume` and `keyword.length`.
  - **Connections:** Input: `Code 2 - Coarse Cluster Keywords By Term Overlap`; Output: `Code 4 - Build Cluster Batch Queue`.
  - **Edge Cases / Failures:** Missing metadata attributes resulting in fallback length sorting.

- **Code 4 - Build Cluster Batch Queue**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Splits the cluster array into manageable chunks for AI consumption.
  - **Configuration:** Array chunking based on `batchSize`.
  - **Key Expressions:** Uses `state.batchSize`.
  - **Connections:** Input: `Code 3 - Compute Primary & Supporting Keywords Per Cluster`; Output: `Code 5 - Get Next Cluster Batch From Queue`.
  - **Edge Cases / Failures:** Empty cluster arrays.

---

#### 2.3 AI Batch Labeling & Pillar Consolidation
This block manages the iterative AI labeling queue, queries Claude for topical categorization, aggregates disparate pillar names, and normalizes them into a canonical vocabulary.

- **Code 5 - Get Next Cluster Batch From Queue**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Pops the next batch from the queue array for loop iteration.
  - **Configuration:** Queue shift operation.
  - **Key Expressions:** Modifies `clusterBatchQueue`.
  - **Connections:** Input: `Code 4 - Build Cluster Batch Queue`, `Code 6 - Parse & Accumulate Batch Results`; Output: `IF - Cluster Batch Queue Empty`.
  - **Edge Cases / Failures:** Queue exhaustion out-of-bounds errors.

- **IF - Cluster Batch Queue Empty**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Evaluates whether batches remain in the processing queue.
  - **Configuration:** Boolean evaluation of `hasNextBatch`.
  - **Key Expressions:** `={{ $json.hasNextBatch }}`
  - **Connections:** Input: `Code 5 - Get Next Cluster Batch From Queue`; Output (True): `Code 7 - Aggregate Distinct Suggested Pillars`; Output (False): `API 1 - Assign Cluster Topics & Pillars (Claude)`.
  - **Edge Cases / Failures:** Infinite looping if state pointers fail to decrement.

- **API 1 - Assign Cluster Topics & Pillars (Claude)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Calls Anthropic's Messages API to generate topics and suggested pillars for current batches.
  - **Configuration:** POST request to Anthropic endpoint with system prompt and JSON schema constraints. Max tokens: 4000. Retries up to 2 times on failure.
  - **Key Expressions:** Dynamic payload mapping via JSON stringify of current batch data and model configurations.
  - **Connections:** Input: `IF - Cluster Batch Queue Empty` (False); Output: `Code 6 - Parse & Accumulate Batch Results`.
  - **Edge Cases / Failures:** API rate limits, authentication header errors, malformed JSON model responses.
  - **Authentication:** Generic Credential Type (HTTP Header Auth).

- **Code 6 - Parse & Accumulate Batch Results**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Parses API responses, extracts JSON data, handles syntax cleansing, and merges results into running labeled collections.
  - **Configuration:** JSON parse block with fallback handling.
  - **Key Expressions:** Accesses execution context variables across node references.
  - **Connections:** Input: `API 1 - Assign Cluster Topics & Pillars (Claude)`; Output: `Code 5 - Get Next Cluster Batch From Queue`.
  - **Edge Cases / Failures:** Unparseable AI markdown blocks or missing key fields.

- **Code 7 - Aggregate Distinct Suggested Pillars**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Gathers all raw pillar labels generated across batches and compiles sample cluster topics.
  - **Configuration:** Aggregation loop with counting maps.
  - **Key Expressions:** Reads `labeledClusters`.
  - **Connections:** Input: `IF - Cluster Batch Queue Empty` (True); Output: `API 2 - Consolidate Pillar Labels (Claude)`.
  - **Edge Cases / Failures:** Empty labeled cluster sets.

- **API 2 - Consolidate Pillar Labels (Claude)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Submits raw distinct pillar labels to Claude to merge near-duplicates into clean canonical pillars.
  - **Configuration:** POST request to Anthropic endpoint. Max tokens: 3000. Retries up to 2 times on failure.
  - **Key Expressions:** Dynamic request body construction embedding distinct pillar lists.
  - **Connections:** Input: `Code 7 - Aggregate Distinct Suggested Pillars`; Output: `Code 8 - Parse Pillar Consolidation & Build Pillar Mapping`.
  - **Edge Cases / Failures:** API timeouts or schema non-compliance by the model.
  - **Authentication:** Generic Credential Type (HTTP Header Auth).

- **Code 8 - Parse Pillar Consolidation & Build Pillar Mapping**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Parses canonical pillar groupings and builds lookup maps with safety fallbacks for unassigned labels.
  - **Configuration:** Dictionary mapping script.
  - **Key Expressions:** Reads AI response content texts.
  - **Connections:** Input: `API 2 - Consolidate Pillar Labels (Claude)`; Output: `Code 9 - Assemble Final Site Architecture`.
  - **Edge Cases / Failures:** Missing mapping keys.

---

#### 2.4 Architecture Assembly & Review
This block compiles the final architectural tree, generates URL-friendly slugs, evaluates structural balance using an AI reviewer, and formats the output payload.

- **Code 9 - Assemble Final Site Architecture**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Applies pillar mappings, nests clusters and supporting pages, computes structural volume metrics, and formats slugs.
  - **Configuration:** Tree-building script with slugification helper functions.
  - **Key Expressions:** Processes `pillarMapping` and `labeledClusters`.
  - **Connections:** Input: `Code 8 - Parse Pillar Consolidation & Build Pillar Mapping`; Output: `API 3 - Architecture Review Agent (Claude)`.
  - **Edge Cases / Failures:** Special characters breaking slug generators.

- **API 3 - Architecture Review Agent (Claude)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Requests a senior SEO architecture assessment evaluating structural balance, thin or broad pillars, and building priority.
  - **Configuration:** POST request to Anthropic endpoint. Max tokens: 1500. Retries up to 2 times.
  - **Key Expressions:** Passes total keyword volume and aggregated pillar counts.
  - **Connections:** Input: `Code 9 - Assemble Final Site Architecture`; Output: `Code 10 - Parse Architecture Review & Build Final Report`.
  - **Edge Cases / Failures:** Context length overflows or invalid model JSON responses.
  - **Authentication:** Generic Credential Type (HTTP Header Auth).

- **Code 10 - Parse Architecture Review & Build Final Report**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Parses review outputs, aggregates summary metrics, attaches timestamps, and constructs the final report object.
  - **Configuration:** Final report compilation script.
  - **Key Expressions:** Accesses review response text payloads.
  - **Connections:** Input: `API 3 - Architecture Review Agent (Claude)`; Output: `HTTP Request - Save Site Architecture To CMS/Database`.
  - **Edge Cases / Failures:** Parsing failures defaulting to fallback review structures.

---

#### 2.5 Persistence, Notification, & Response
This block handles external CMS integrations, delivers operational summaries via WhatsApp, and returns responses to the webhook caller.

- **HTTP Request - Save Site Architecture To CMS/Database**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Dispatches the complete site architecture JSON report to an external CMS or content planning database.
  - **Configuration:** POST method targeting configured CMS URL with `continueOnFail` enabled.
  - **Key Expressions:** `={{ $json.cmsSaveUrl }}` and `={{ JSON.stringify($json.finalReport) }}`.
  - **Connections:** Input: `Code 10 - Parse Architecture Review & Build Final Report`; Output: `WhatsApp - Send Completion Summary`.
  - **Edge Cases / Failures:** Network timeouts or remote server HTTP error responses (mitigated by continuation rules).
  - **Authentication:** Generic Credential Type (HTTP Header Auth).

- **WhatsApp - Send Completion Summary**
  - **Type & Technical Role:** `n8n-nodes-base.whatsApp` — Sends execution metrics and build priorities to a designated recipient via WhatsApp Business Cloud.
  - **Configuration:** Send text operation configured with template formatting.
  - **Key Expressions:** Evaluates summary counts and extracts top build priorities from previous node data.
  - **Connections:** Input: `HTTP Request - Save Site Architecture To CMS/Database`; Output: `Respond to Webhook - Return Site Architecture`.
  - **Edge Cases / Failures:** Invalid phone identifiers or expired WhatsApp API tokens.

- **Respond to Webhook - Return Site Architecture**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` — Returns the full site architecture report as the HTTP response payload to the original webhook caller.
  - **Configuration:** Responds with JSON format. Has `continueOnFail` enabled.
  - **Key Expressions:** `={{ $('Code 10 - Parse Architecture Review & Build Final Report').first().json.finalReport }}`.
  - **Connections:** Input: `WhatsApp - Send Completion Summary`; Output: None (Terminal Node).
  - **Edge Cases / Failures:** Responses triggered on manual test runs where no active webhook connection exists.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Documentation note describing overall workflow usage and customization options. | None | None | ## Automatic SEO Site Architecture Generator<br>Feed it a large keyword list - up to 10,000 - and it comes back with a Pillar → Cluster → Supporting-page site architecture... |
| Sticky Note - Stage 1 | `n8n-nodes-base.stickyNote` | Documentation note detailing the input ingestion and normalization stage. | None | None | ## 1. Keywords (10,000 in)<br><br>- **Webhook - Ingest Keyword List**: receives {keywords: [...]}... |
| Sticky Note - Stage 2 | `n8n-nodes-base.stickyNote` | Documentation note detailing coarse algorithmic clustering. | None | None | ## 2. Coarse Clustering (algorithmic, no AI)<br><br>- **Code 2 - Coarse Cluster Keywords By Term Overlap**... |
| Sticky Note - Stage 3 | `n8n-nodes-base.stickyNote` | Documentation note detailing AI batch labeling and pillar consolidation. | None | None | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| Sticky Note - Stage 5 | `n8n-nodes-base.stickyNote` | Documentation note detailing architecture assembly and review. | None | None | ## 5. Assemble Architecture & Review<br><br>- **Code 9 - Assemble Final Site Architecture**... |
| Sticky Note - Stage 6 | `n8n-nodes-base.stickyNote` | Documentation note detailing output persistence, notifications, and webhook responses. | None | None | ## 6. Output<br><br>- **HTTP Request - Save Site Architecture To CMS/Database**... |
| Webhook - Ingest Keyword List | `n8n-nodes-base.webhook` | Receives incoming POST keyword payloads. | None | Code 1 - Ingest & Normalize Keywords | ## 1. Keywords (10,000 in)<br><br>- **Webhook - Ingest Keyword List**: receives {keywords: [...]}... |
| Manual Trigger - Test Run | `n8n-nodes-base.manualTrigger` | Initiates manual test executions. | None | Code 1 - Ingest & Normalize Keywords | ## 1. Keywords (10,000 in)<br><br>- **Webhook - Ingest Keyword List**: receives {keywords: [...]}... |
| Code 1 - Ingest & Normalize Keywords | `n8n-nodes-base.code` | Normalizes, de-duplicates, caps, and falls back input keywords. | Webhook - Ingest Keyword List, Manual Trigger - Test Run | Set - Config | ## 1. Keywords (10,000 in)<br><br>- **Webhook - Ingest Keyword List**: receives {keywords: [...]}... |
| Set - Config | `n8n-nodes-base.set` | Assigns operational configuration parameters and thresholds. | Code 1 - Ingest & Normalize Keywords | Code 2 - Coarse Cluster Keywords By Term Overlap | ## 1. Keywords (10,000 in)<br><br>- **Webhook - Ingest Keyword List**: receives {keywords: [...]}... |
| Code 2 - Coarse Cluster Keywords By Term Overlap | `n8n-nodes-base.code` | Groups keywords into raw clusters using shared token overlap. | Set - Config | Code 3 - Compute Primary & Supporting Keywords Per Cluster | ## 2. Coarse Clustering (algorithmic, no AI)<br><br>- **Code 2 - Coarse Cluster Keywords By Term Overlap**... |
| Code 3 - Compute Primary & Supporting Keywords Per Cluster | `n8n-nodes-base.code` | Selects primary hub keywords and supporting keyword candidates. | Code 2 - Coarse Cluster Keywords By Term Overlap | Code 4 - Build Cluster Batch Queue | ## 2. Coarse Clustering (algorithmic, no AI)<br><br>- **Code 2 - Coarse Cluster Keywords By Term Overlap**... |
| Code 4 - Build Cluster Batch Queue | `n8n-nodes-base.code` | Splits clusters into batch arrays for processing queues. | Code 3 - Compute Primary & Supporting Keywords Per Cluster | Code 5 - Get Next Cluster Batch From Queue | ## 2. Coarse Clustering (algorithmic, no AI)<br><br>- **Code 2 - Coarse Cluster Keywords By Term Overlap**... |
| Code 5 - Get Next Cluster Batch From Queue | `n8n-nodes-base.code` | Retrieves the next batch from the processing queue. | Code 4 - Build Cluster Batch Queue, Code 6 - Parse & Accumulate Batch Results | IF - Cluster Batch Queue Empty | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| IF - Cluster Batch Queue Empty | `n8n-nodes-base.if` | Checks if all batch queues have been processed. | Code 5 - Get Next Cluster Batch From Queue | Code 7 - Aggregate Distinct Suggested Pillars, API 1 - Assign Cluster Topics & Pillars (Claude) | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| API 1 - Assign Cluster Topics & Pillars (Claude) | `n8n-nodes-base.httpRequest` | Calls Claude API to assign topics and suggested pillars to batches. | IF - Cluster Batch Queue Empty | Code 6 - Parse & Accumulate Batch Results | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| Code 6 - Parse & Accumulate Batch Results | `n8n-nodes-base.code` | Parses batch AI responses and appends results to accumulator lists. | API 1 - Assign Cluster Topics & Pillars (Claude) | Code 5 - Get Next Cluster Batch From Queue | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| Code 7 - Aggregate Distinct Suggested Pillars | `n8n-nodes-base.code` | Aggregates distinct raw pillar labels and sample cluster topics. | IF - Cluster Batch Queue Empty | API 2 - Consolidate Pillar Labels (Claude) | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| API 2 - Consolidate Pillar Labels (Claude) | `n8n-nodes-base.httpRequest` | Merges raw pillar labels into canonical top-level categories. | Code 7 - Aggregate Distinct Suggested Pillars | Code 8 - Parse Pillar Consolidation & Build Pillar Mapping | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| Code 8 - Parse Pillar Consolidation & Build Pillar Mapping | `n8n-nodes-base.code` | Parses consolidation mappings with safety fallback lookups. | API 2 - Consolidate Pillar Labels (Claude) | Code 9 - Assemble Final Site Architecture | ## 3. Cluster Labeling (AI, batched)<br><br>- **Code 5 - Get Next Cluster Batch From Queue**... |
| Code 9 - Assemble Final Site Architecture | `n8n-nodes-base.code` | Applies pillar mapping, structures tree hierarchies, and creates URL slugs. | Code 8 - Parse Pillar Consolidation & Build Pillar Mapping | API 3 - Architecture Review Agent (Claude) | ## 5. Assemble Architecture & Review<br><br>- **Code 9 - Assemble Final Site Architecture**... |
| API 3 - Architecture Review Agent (Claude) | `n8n-nodes-base.httpRequest` | Requests architectural balance assessment and build priorities from Claude. | Code 9 - Assemble Final Site Architecture | Code 10 - Parse Architecture Review & Build Final Report | ## 5. Assemble Architecture & Review<br><br>- **Code 9 - Assemble Final Site Architecture**... |
| Code 10 - Parse Architecture Review & Build Final Report | `n8n-nodes-base.code` | Assembles review outputs and summary metrics into a final report. | API 3 - Architecture Review Agent (Claude) | HTTP Request - Save Site Architecture To CMS/Database | ## 5. Assemble Architecture & Review<br><br>- **Code 9 - Assemble Financial Site Architecture**... |
| HTTP Request - Save Site Architecture To CMS/Database | `n8n-nodes-base.httpRequest` | Saves generated site architecture to a remote CMS or database API. | Code 10 - Parse Architecture Review & Build Final Report | WhatsApp - Send Completion Summary | ## 6. Output<br><br>- **HTTP Request - Save Site Architecture To CMS/Database**... |
| WhatsApp - Send Completion Summary | `n8n-nodes-base.whatsApp` | Sends execution metrics and build priorities via WhatsApp Business Cloud. | HTTP Request - Save Site Architecture To CMS/Database | Respond to Webhook - Return Site Architecture | ## 6. Output<br><br>- **HTTP Request - Save Site Architecture To CMS/Database**... |
| Respond to Webhook - Return Site Architecture | `n8n-nodes-base.respondToWebhook` | Returns the final JSON report back to the webhook caller. | WhatsApp - Send Completion Summary | None | ## 6. Output<br><br>- **HTTP Request - Save Site Architecture To CMS/Database**... |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, perform the following step-by-step procedure:

1. **Create Trigger Nodes:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`), set method to `POST`, and assign path `site-architecture-ingest`.
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) for testing.

2. **Add Ingestion Code Node:**
   - Create a **Code** node named `Code 1 - Ingest & Normalize Keywords`.
   - Connect both triggers to this node.
   - Insert JavaScript to parse payloads (`keywords` array or `csvText`), handle empty payloads with the bundled sample list, de-duplicate case-insensitively, and cap items at `maxKeywords`.

3. **Configure Settings Node:**
   - Add a **Set** node named `Set - Config`.
   - Configure assignments: `batchSize` (Number: 50), `maxSampleKeywordsPerClusterForPrompt` (Number: 12), `anthropicModel` (String: `claude-sonnet-4-6`), `cmsSaveUrl` (String), `whatsappBusinessPhoneId` (String), and `whatsappRecipientPhone` (String).
   - Connect `Code 1` output to this node.

4. **Build Algorithmic Clustering Nodes:**
   - Add a **Code** node named `Code 2 - Coarse Cluster Keywords By Term Overlap`. Tokenize keywords, strip stop-words/modifiers, and group by shared core terms. Connect `Set - Config` here.
   - Add a **Code** node named `Code 3 - Compute Primary & Supporting Keywords Per Cluster`. Score and select primary hub keywords vs. supporting candidates. Connect `Code 2` to this node.
   - Add a **Code** node named `Code 4 - Build Cluster Batch Queue`. Chunk clusters into arrays of size `batchSize`. Connect `Code 3` to this node.

5. **Establish AI Batch Processing Loop:**
   - Add a **Code** node named `Code 5 - Get Next Cluster Batch From Queue`. Pop batches sequentially. Connect `Code 4` (and later `Code 6`) to this node.
   - Add an **IF** node named `IF - Cluster Batch Queue Empty`. Set condition to evaluate `hasNextBatch` (false). Connect `Code 5` to this node.
   - Add an **HTTP Request** node named `API 1 - Assign Cluster Topics & Pillars (Claude)`. Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`, and configure Generic HTTP Header Auth (Anthropic API key). Set headers `anthropic-version: 2023-06-01` and `content-type: application/json`. Configure request body to send batch items. Enable `continueErrorOutput` and retry settings. Connect the `false` branch of the IF node here.
   - Add a **Code** node named `Code 6 - Parse & Accumulate Batch Results`. Parse Claude JSON responses, map signatures, and append to running lists. Connect `API 1` output here, and loop its output back into `Code 5`.

6. **Add Pillar Consolidation Nodes:**
   - From the `true` branch of `IF - Cluster Batch Queue Empty`, connect to a **Code** node named `Code 7 - Aggregate Distinct Suggested Pillars` to tally raw labels and sample topics.
   - Add an **HTTP Request** node named `API 2 - Consolidate Pillar Labels (Claude)`. Configure similar Anthropic headers and authentication, sending distinct pillars for consolidation. Connect `Code 7` here.
   - Add a **Code** node named `Code 8 - Parse Pillar Consolidation & Build Pillar Mapping` to process canonical mappings and apply fallback rules. Connect `API 2` here.

7. **Add Architecture Assembly and Review Nodes:**
   - Add a **Code** node named `Code 9 - Assemble Final Site Architecture`. Build tree objects, nest supporting pages, and generate URL slugs. Connect `Code 8` here.
   - Add an **HTTP Request** node named `API 3 - Architecture Review Agent (Claude)`. Configure Anthropic headers and authentication to review structural balance and build priorities. Connect `Code 9` here.
   - Add a **Code** node named `Code 10 - Parse Architecture Review & Build Final Report`. Parse review payloads and construct final report objects. Connect `API 3` here.

8. **Add Output, Persistence, and Response Nodes:**
   - Add an **HTTP Request** node named `HTTP Request - Save Site Architecture To CMS/Database`. Set method to `POST`, URL to `{{ $json.cmsSaveUrl }}`, and body to stringified final report data. Enable `continueOnFail`. Connect `Code 10` here.
   - Add a **WhatsApp** node named `WhatsApp - Send Completion Summary`. Set operation to `send`, configure credentials/connection, use `={{ $json.whatsappBusinessPhoneId }}` and `={{ $json.whatsappRecipientPhone }}`, and draft summary texts using metrics from `Code 10`. Connect the CMS request node here.
   - Add a **Respond to Webhook** node named `Respond to Webhook - Return Site Architecture`. Set response type to JSON, return `={{ $('Code 10 - Parse Architecture Review & Build Final Report').first().json.finalReport }}`, and enable `continueOnFail`. Connect the WhatsApp node here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automatic SEO Site Architecture Generator Implementation | Automated workflow designed to process keyword exports up to 10,000 rows into structured hierarchies. |