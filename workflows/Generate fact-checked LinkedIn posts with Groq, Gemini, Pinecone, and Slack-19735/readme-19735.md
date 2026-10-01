Generate fact-checked LinkedIn posts with Groq, Gemini, Pinecone, and Slack

https://n8nworkflows.xyz/workflows/generate-fact-checked-linkedin-posts-with-groq--gemini--pinecone--and-slack-19735


# Generate fact-checked LinkedIn posts with Groq, Gemini, Pinecone, and Slack

### 1. Workflow Overview

This workflow automates the end-to-end generation, fact-checking, and publication of LinkedIn posts. It operates via a dual entry point: a daily scheduled cron run or a real-time Google Drive folder monitor. The system extracts source material (images, PDFs, URLs, or plain text), performs deep resource analysis, queries vector reference patterns via Pinecone to design a structured content strategy, drafts a post utilizing Groq and Google Gemini models, executes automated quality and fact-checking, and pauses for human approval via Slack (with Gmail timeout fallbacks) before finally publishing to LinkedIn.

The workflow logic is divided into the following functional blocks:
- **1.1 Trigger and Entry Routing:** Handles scheduled cron events or file uploads from Google Drive, tagging execution runs accordingly.
- **1.2 Topic Selection & Mapping:** Evaluates available Supabase database topics using an AI agent to select the best daily candidate or recommend skipping.
- **1.3 Source Routing, Extraction, and Normalization:** Dispatches execution paths based on source type, processing images (via OCR), PDFs, HTML URLs, or direct text strings into a unified normalized format.
- **1.4 Resource Analysis:** Truncates normalized text and extracts verified facts, technical details, angles, and privacy flags using an AI agent.
- **1.5 Content Strategy Development:** Combines resource analysis with reference-post patterns retrieved from a Pinecone vector database using a Gemini strategist model, outputting a strategy contract logged to Supabase.
- **1.6 Post Generation and Quality Control:** Drafts the post based on the strategy contract and strictly bounded verified facts, running an automated quality check for factual support, strategy compliance, readability, and privacy.
- **1.7 Human Review & Decision Handling:** Sends interactive approval forms to Slack, logs review metrics, monitors for timeouts via Gmail, and handles approval, rejection, or regeneration branches.
- **1.8 Publication and Archiving:** Publishes approved posts to LinkedIn and logs them, or archives rejected topics in Supabase.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Entry Routing
- **Overview:** Initiates the workflow either on a daily schedule or immediately when a new file is added to a specific Google Drive folder, assigning execution context variables.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `Google Drive Trigger`
  - `Assign Run Type`
  - `Assign Run TYpe`

- **Node Details:**
  - **Schedule Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Set to trigger daily at 10:00 AM.
    - *Input/Output:* No inputs; outputs execution start data to `Assign Run Type`.
  - **Google Drive Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.googleDriveTrigger` (Event Trigger)
    - *Configuration:* Polls every minute for `fileCreated` events inside the specific folder ID `1kWIykK2cIPVW-HbmuACYdsSYyFnfmkhC` (named "Post-content"). Requires Google Drive OAuth2 credentials.
    - *Input/Output:* No inputs; outputs file metadata to `Assign Run TYpe`.
  - **Assign Run Type**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns a string field `run_type` with the value `"scheduled"`.
    - *Input/Output:* Connects `Schedule Trigger` -> `Assign Run Type` -> `Get Topic From DB`.
  - **Assign Run TYpe**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets `run_type` to `"event"` and maps `id` from `={{ $json.id }}`.
    - *Input/Output:* Connects `Google Drive Trigger` -> `Assign Run TYpe` -> `Data Type`.
  - *Edge Cases / Potential Failures:* Google Drive token expiration or insufficient folder permissions will block event-driven triggers.

---

#### 1.2 Topic Selection & Mapping
- **Overview:** Queries eligible topics from Supabase during scheduled runs and leverages an AI agent to select the optimal post candidate or recommend a skip action.
- **Nodes Involved:**
  - `Get Topic From DB`
  - `Check for Eligible Topic`
  - `Topic Selection`
  - `Topic Selection Model`
  - `Structured Output Parser`
  - `Map Selected Topic`
  - `Topic Recommended ?`
  - `Nothing to Post`

- **Node Details:**
  - **Get Topic From DB**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Reader)
    - *Configuration:* Queries table/view `eligible_trending_topics`, orders by `added_at`, returns all rows. Requires Supabase credentials.
  - **Check for Eligible Topic**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Data Filter)
    - *Configuration:* Filters items where `={{ $json.id }}` is not empty.
  - **Topic Selection**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Evaluates the eligible topics list and executes selection logic. Uses `Structured Output Parser`.
    - *Key Expressions:* JSON stringification of eligible list items passed via prompt.
  - **Topic Selection Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Language Model)
    - *Configuration:* Uses model `openai/gpt-oss-120b`. Requires Groq API credentials.
  - **Structured Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser)
    - *Configuration:* Enforces a JSON schema containing `selected_topic_id` (string), `reasoning` (string), and `no_post_recommended` (boolean).
  - **Map Selected Topic**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Reader)
    - *Configuration:* Queries `trending_topics_inventory` matching `id` equal to `={{ $json.output.selected_topic_id }}`. Limit set to 1.
  - **Topic Recommended ?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration:* Evaluates whether `NO_POST_RECOMMENDED` equals `true`.
  - **Nothing to Post**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Termination / No-Operation)
    - *Configuration:* Terminal endpoint when no post is recommended.
  - *Edge Cases / Potential Failures:* Empty database responses or malformed agent JSON payloads handled via structured parsers.

---

#### 1.3 Source Routing, Extraction, and Normalization
- **Overview:** Routes content dynamically based on the source type (File, URL, Text) or event trigger, extracting and cleaning raw binary content or HTML before normalization.
- **Nodes Involved:**
  - `Data Type`
  - `Download file`
  - `File Type`
  - `Extract Image`
  - `Clean Data`
  - `Groq Chat Model`
  - `Normalise Image`
  - `Extract PDF`
  - `Normalise Pdf`
  - `Extract URl`
  - `Clean HTML File`
  - `Clean Extracted Data`
  - `Normalise Url`
  - `Get The Text`
  - `Normalise Text`
  - `Merge`

- **Node Details:**
  - **Data Type**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Router)
    - *Configuration:* Evaluates `run_type` and `source_type` into four outputs: `file`, `File`, `url`, and `text`.
  - **Download file**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (File Downloader)
    - *Configuration:* Downloads file using `={{ $('Google Drive Trigger').item.json.id }}`.
  - **File Type**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Router)
    - *Configuration:* Splits based on MIME type (`image/png` vs `application/pdf`).
  - **Extract Image**
    - *Type and Technical Role:* `n8n-nodes-tesseractjs.tesseractNode` (OCR Extraction)
    - *Configuration:* Extracts text out of binary image inputs.
  - **Clean Data**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (LLM Chain)
    - *Configuration:* Cleans OCR artifacts and broken line breaks using the attached Groq chat model.
  - **Groq Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Model)
    - *Configuration:* Model set to `openai/gpt-oss-20b`.
  - **Normalise Image / Pdf / Url / Text**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformer)
    - *Configuration:* Maps extracted content explicitly to standard schema parameters (e.g., `text` string).
  - **Extract URl**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Client)
    - *Configuration:* Fetches raw HTML content from `={{ $json.reference_source }}`.
  - **Clean HTML File & Clean Extracted Data**
    - *Type and Technical Role:* `n8n-nodes-base.html` / `n8n-nodes-base.code` (Data Cleaner)
    - *Configuration:* Strips script/style tags, handles block boundaries, removes boilerplate page noise, and caps raw output strings at 6,000 characters.
  - **Merge**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Combiner)
    - *Configuration:* Aggregates multi-source paths (4 inputs) into a single processing pipeline.
  - *Edge Cases / Potential Failures:* Network timeouts on HTTP requests, unrecognized MIME types, or heavily obfuscated OCR text.

---

#### 1.4 Resource Analysis
- **Overview:** Truncates and normalizes incoming source strings and executes an AI agent to extract verified facts, technical details, angles, and privacy flags.
- **Nodes Involved:**
  - `Normalize Input`
  - `Resource Analyzer`
  - `Resouce Analyzer Model`
  - `Structured Output Parser1`

- **Node Details:**
  - **Normalize Input**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Transformation)
    - *Configuration:* Enforces a maximum character limit of 6,000 characters to manage token allocation safely.
  - **Resource Analyzer**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Extracts structured resource components (verified facts, topic, angles, technical details, privacy flags) using a structured output parser.
  - **Resouce Analyzer Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Model)
    - *Configuration:* Uses model `openai/gpt-oss-120b`.
  - **Structured Output Parser1**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Output Parser)
    - *Configuration:* Defines array schemas for facts, angles, and privacy flags alongside string properties for topics and technical details.
  - *Edge Cases / Potential Failures:* Token limit overflow if input normalization fails; unparseable JSON schemas.

---

#### 1.5 Content Strategy Development
- **Overview:** Combines resource analysis with vector reference-post patterns via Pinecone to establish a formal strategy contract, which is then persisted to Supabase.
- **Nodes Involved:**
  - `Content Stratigist`
  - `Google Gemini Chat Model`
  - `Pinecone Vector Store`
  - `Embeddings Google Gemini`
  - `Structured Output Parser2`
  - `Log Strategy`

- **Node Details:**
  - **Content Stratigist**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Acts as the central strategist, invoking the Pinecone tool to retrieve reference-post patterns before determining the post angle, format, hook type, structure, CTA type, and target length.
  - **Google Gemini Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model)
    - *Configuration:* Model set to `models/gemini-3.1-flash-lite`. Requires Google Gemini credentials.
  - **Pinecone Vector Store**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.vectorStorePinecone` (Vector DB Integration)
    - *Configuration:* Configured as a tool (`retrieve-as-tool`) targeting the index `linkedin-reference-content`. Requires Pinecone credentials.
  - **Embeddings Google Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini` (Embeddings Generator)
    - *Configuration:* Generates vector embeddings for similarity queries.
  - **Structured Output Parser2**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Output Parser)
    - *Configuration:* Enforces string schemas for angle, format, hook type, structure, CTA type, target length, and reasoning.
  - **Log Strategy**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Writer)
    - *Configuration:* Inserts strategy parameters into the `strategy_contracts` Supabase table linked by `topic_id`.
  - *Edge Cases / Potential Failures:* Pinecone index connection failures or embedding mismatch errors.

---

#### 1.6 Post Generation and Quality Control
- **Overview:** Drafts the LinkedIn post strictly adhering to the strategy contract and verified facts, then runs an automated quality checker to evaluate factual support, strategy compliance, readability, and privacy.
- **Nodes Involved:**
  - `Post Generator`
  - `Post Generator Model`
  - `Quality Checker`
  - `Quality chcker model`
  - `Structured Output Parser3`
  - `Prepare Input`
  - `Redirect to Post Generator`
  - `Prepare For Regeneration`

- **Node Details:**
  - **Post Generator**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Generates draft post text based on strategy contract boundaries. Incorporates human feedback if run via regeneration loops.
  - **Post Generator Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Model)
    - *Configuration:* Uses model `openai/gpt-oss-120b`.
  - **Quality Checker**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Evaluates draft text across four vectors: factual support, strategy compliance, readability/repetition, and privacy/security. Configured with error handling (`onError: continueRegularOutput`).
  - **Quality chcker model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (AI Model)
    - *Configuration:* Uses model `openai/gpt-oss-120b`.
  - **Structured Output Parser3**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Output Parser)
    - *Configuration:* Enforces strict JSON return schema including `fact_check_pass`, `unsupported_claims`, `strategy_compliance_pass`, `strategy_deviations`, `readability_notes`, `privacy_security_flags`, and `overall_summary`.
  - **Prepare Input / Prepare For Regeneration**
    - *Type and Technical Role:* `n8n-nodes-base.set` / `n8n-nodes-base.code` (Data Transformers)
    - *Configuration:* Formats outputs into structured variables for Slack review inputs or maps reviewer feedback back to the generator.
  - *Edge Cases / Potential Failures:* Hallucinated facts caught by the quality checker; parser execution failures handled via continuation policies.

---

#### 1.7 Human Review & Decision Handling
- **Overview:** Dispatches interactive review requests to Slack, logs review decisions, monitors for timeouts via Gmail, and processes user choices (Approve, Reject, Regenerate).
- **Nodes Involved:**
  - `Send message and wait for response`
  - `Log Decision`
  - `Switch`
  - `Notify`
  - `wait for response`
  - `Verify Decision`

- **Node Details:**
  - **Send message and wait for response**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Interactive Notification / Webhook Wait)
    - *Configuration:* Sends a formatted message containing draft summaries, quality metrics, security flags, and unsupported claims to Slack channel ID `C0C25EDEAB0` with a custom form (`Approve`, `Reject`, `Regenerate`). Sets a limit wait time resume amount (2 hours). Requires Slack credentials.
  - **Log Decision**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Writer)
    - *Configuration:* Logs reviewer decisions, feedback timestamps, and truncated draft texts into the `post_review_log` table.
  - **Switch**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Decision Router)
    - *Configuration:* Routes execution based on `decision` values: `approved`, `rejected`, `regenerate`, or `TimedOut`.
  - **Notify**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Sender)
    - *Configuration:* Sends timeout notification emails containing resume webhook links (`user@example.com`). Requires Gmail OAuth2 credentials.
  - **wait for response**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Webhook Wait)
    - *Configuration:* Suspends workflow execution waiting for an external HTTP webhook resume call.
  - **Verify Decision**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Router)
    - *Configuration:* Evaluates decision parameters returned via the external email webhook resume query (`decision=approved` / `decision=rejected`).
  - *Edge Cases / Potential Failures:* Slack webhook timeouts, invalid token responses, or broken email resume links.

---

#### 1.8 Publication and Archiving
- **Overview:** Publishes approved posts to LinkedIn and updates internal audit tables, or archives rejected trending topics in Supabase.
- **Nodes Involved:**
  - `Create a post`
  - `Log Post`
  - `Update a row`
  - `Finish`

- **Node Details:**
  - **Create a post**
    - *Type and Technical Role:* `n8n-nodes-base.linkedIn` (Social Media Publisher)
    - *Configuration:* Publishes formatted post copy to LinkedIn. Requires LinkedIn OAuth2 credentials.
  - **Log Post**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Writer)
    - *Configuration:* Inserts audit metrics into the `published_posts` table (topic ID, format, hook type, structure, angle, CTA type, target length, post content).
  - **Update a row**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Writer)
    - *Configuration:* Updates table `trending_topics_inventory`, setting `status` to `archived` for rejected topics where condition `id` matches topic ID.
  - **Finish**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Terminal Node)
    - *Configuration:* Final convergence node for completed workflow executions.
  - *Edge Cases / Potential Failures:* LinkedIn API rate limits, invalid authorization scopes, or database constraint violations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Schedule Trigger` | `scheduleTrigger` | Triggers scheduled pipeline runs | None | `Assign Run Type` | **1A. Scheduled Trigger**<br>Starts a run every day at 10:00 and tags it `run_type = scheduled`. |
| `Google Drive Trigger` | `googleDriveTrigger` | Triggers pipeline on file creation | None | `Assign Run TYpe` | **1B. File Upload Trigger (event path)**<br>Watches the `Post-content` Drive folder every minute. A new file starts a run tagged `run_type = event` and goes straight to extraction in section 3, skipping topic selection. |
| `Structured Output Parser` | `outputParserStructured` | Parses structured topic selection output | `Topic Selection` | `Topic Selection` | |
| `Groq Chat Model` | `lmChatGroq` | Provides LLM backend for OCR cleaning | `Clean Data` | `Clean Data` | |
| `Download file` | `googleDrive` | Downloads binary files from Drive | `Data Type` | `File Type` | |
| `Merge` | `merge` | Aggregates all extraction paths | `Normalise Pdf`, `Normalise Url`, `Normalise Text`, `Normalise Image` | `Normalize Input` | |
| `Map Selected Topic` | `supabase` | Fetches full topic record from DB | `Topic Selection` | `Topic Recommended ?` | **Note - Map Topic**<br>Add your Supabase credentials here. |
| `Normalise Text` | `set` | Normalizes pasted text inputs | `Get The Text` | `Merge` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Get The Text` | `noOp` | Intermediate node for text extraction | `Data Type` | `Normalise Text` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Normalise Image` | `set` | Normalizes OCR extracted image text | `Clean Data` | `Merge` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Normalise Pdf` | `set` | Sets run type parameter for PDFs | `Extract PDF` | `Merge` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Clean HTML File` | `html` | Extracts HTML body content | `Extract URl` | `Clean Extracted Data` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Normalise Url` | `set` | Normalizes extracted URL text | `Clean Extracted Data` | `Merge` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Topic Selection` | `agent` | AI agent selects best topic | `Check for Eligible Topic` | `Map Selected Topic` | **2. Topic Selection (scheduled path)**<br>Reads the `eligible_trending_topics` view, drops empty results and asks an AI agent to pick the best topic to post today, or to recommend no post. The chosen id is looked up in `trending_topics_inventory`. If no post is recommended, the run ends. |
| `Resource Analyzer` | `agent` | AI agent extracts facts and flags | `Normalize Input` | `Content Stratigist` | **4. Truncate & Resource Analysis**<br>Trims the text to 6,000 characters, then an AI agent extracts verified facts, the core topic, content angles, technical details and privacy flags as structured JSON. A Gemini model is attached as fallback. |
| `Pinecone Vector Store` | `vectorStorePinecone` | Retrieves reference content patterns | `Content Stratigist` | `Content Stratigist` | **5. Content Strategy**<br>The Content Strategist turns the analysis into a strategy contract: angle, format, hook type, structure, CTA type and target length. It can query the Pinecone index of reference-post patterns as a tool. The contract is logged to `strategy_contracts`. |
| `Embeddings Google Gemini` | `embeddingsGoogleGemini` | Generates text embeddings for Pinecone | `Pinecone Vector Store` | `Pinecone Vector Store` | **5. Content Strategy**<br>The Content Strategist turns the analysis into a strategy contract: angle, format, hook type, structure, CTA type and target length. It can query the Pinecone index of reference-post patterns as a tool. The contract is logged to `strategy_contracts`. |
| `Structured Output Parser1` | `outputParserStructured` | Parses resource analysis outputs | `Resource Analyzer` | `Resource Analyzer` | |
| `Nothing to Post` | `noOp` | Terminates workflow on skip recommendation | `Topic Recommended ?` | None | **2B. No Post Today**<br>The run ends here when the agent recommends skipping today's post. |
| `Assign Run Type` | `set` | Assigns scheduled run context | `Schedule Trigger` | `Get Topic From DB` | **1A. Scheduled Trigger**<br>Starts a run every day at 10:00 and tags it `run_type = scheduled`. |
| `Get Topic From DB` | `supabase` | Queries eligible trending topics view | `Assign Run Type` | `Check for Eligible Topic` | **Note - Get Topic**<br>Add your Supabase credentials here. Reads the `eligible_trending_topics` view. |
| `Topic Recommended ?` | `if` | Evaluates skip post recommendation | `Map Selected Topic` | `Nothing to Post`, `Data Type` | |
| `Check for Eligible Topic` | `filter` | Filters non-empty topic records | `Get Topic From DB` | `Topic Selection` | |
| `Data Type` | `switch` | Routes execution by run or source type | `Topic Recommended ?`, `Assign Run TYpe` | `Download file`, `Extract URl`, `Get The Text` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `File Type` | `switch` | Routes files by MIME type | `Download file` | `Extract Image`, `Extract PDF` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Extract Image` | `tesseractNode` | Performs OCR on image binaries | `File Type` | `Clean Data` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Clean Data` | `chainLlm` | Cleans extracted OCR text | `Extract Image` | `Normalise Image` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Extract PDF` | `extractFromFile` | Extracts text from PDF files | `File Type` | `Normalise Pdf` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Extract URl` | `httpRequest` | Fetches target webpage HTML | `Data Type` | `Clean HTML File` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Clean Extracted Data` | `code` | Sanitizes extracted HTML and noise | `Clean HTML File` | `Normalise Url` | **3. Source Routing, Extraction & Normalisation**<br>Routes each topic by source type. Drive files are downloaded and split by type: PNG screenshots go through OCR and an LLM clean-up, PDFs through text extraction. URLs are fetched and stripped of page noise. Pasted text passes straight through. All four paths are normalised and merged into one `text` field. |
| `Assign Run TYpe` | `set` | Assigns event execution context | `Google Drive Trigger` | `Data Type` | **1B. File Upload Trigger (event path)**<br>Watches the `Post-content` Drive folder every minute. A new file starts a run tagged `run_type = event` and goes straight to extraction in section 3, skipping topic selection. |
| `Content Stratigist` | `agent` | AI agent builds strategy contract | `Resource Analyzer` | `Log Strategy` | **5. Content Strategy**<br>The Content Strategist turns the analysis into a strategy contract: angle, format, hook type, structure, CTA type and target length. It can query the Pinecone index of reference-post patterns as a tool. The contract is logged to `strategy_contracts`. |
| `Post Generator` | `agent` | AI agent drafts LinkedIn post | `Log Strategy`, `Redirect to Post Generator` | `Quality Checker` | **6. Post Generation & Quality Check**<br>The Post Generator writes one post that follows the contract and uses only verified facts. The Quality Checker then reviews it for factual support, strategy compliance, readability and privacy, without rewriting. On a regenerate decision, the reviewer's feedback is added to the prompt and the generator runs again. |
| `Quality Checker` | `agent` | Evaluates draft post compliance | `Post Generator` | `Prepare Input` | **6. Post Generation & Quality Check**<br>The Post Generator writes one post that follows the contract and uses only verified facts. The Quality Checker then reviews it for factual support, strategy compliance, readability and privacy, without rewriting. On a regenerate decision, the reviewer's feedback is added to the prompt and the generator runs again. |
| `Create a post` | `linkedIn` | Publishes approved post to LinkedIn | `Switch`, `Verify Decision` | `Log Post` | **Note - LinkedIn**<br>Add your LinkedIn credentials and set the author and post text here. |
| `Resouce Analyzer Model` | `lmChatGroq` | Provides LLM backend for Resource Analyzer | `Resource Analyzer` | `Resource Analyzer` | |
| `Topic Selection Model` | `lmChatGroq` | Provides LLM backend for Topic Selection | `Topic Selection` | `Topic Selection` | |
| `Structured Output Parser2` | `outputParserStructured` | Parses content strategy output | `Content Stratigist` | `Content Stratigist` | |
| `Log Strategy` | `supabase` | Logs strategy contract to DB | `Content Stratigist` | `Post Generator` | **Note - Log Strategy**<br>Add your Supabase credentials here. Logs to `strategy_contracts`. |
| `Structured Output Parser3` | `outputParserStructured` | Parses quality checker output | `Quality Checker` | `Quality Checker` | |
| `Send message and wait for response` | `slack` | Sends Slack review and waits for input | `Prepare Input` | `Log Decision` | **Note - Slack**<br>Add your Slack credentials and choose the review channel. |
| `Switch` | `switch` | Routes decision (approve/reject/regenerate/timeout) | `Log Decision` | `Create a post`, `Update a row`, `Prepare For Regeneration`, `Notify` | **7. Human Review (Slack)**<br>Sends the draft, quality summary, security flags and unsupported claims to Slack with an Approve / Reject / Regenerate form. The decision is logged to `post_review_log` and routed by the Switch. |
| `Log Decision` | `supabase` | Logs reviewer decision to DB | `Send message and wait for response` | `Switch` | **Note - Log Decision**<br>Add your Supabase credentials here. Logs to `post_review_log`. |
| `Update a row` | `supabase` | Archives rejected topics in DB | `Switch`, `Verify Decision` | `Finish` | **Note - Archive Topic**<br>Add your Supabase credentials here. Sets the topic's status to archived. |
| `Google Gemini Chat Model` | `lmChatGoogleGemini` | Provides LLM backend for Content Strategist | `Content Stratigist` | `Content Stratigist` | |
| `Post Generator Model` | `lmChatGroq` | Provides LLM backend for Post Generator | `Post Generator` | `Post Generator` | |
| `Quality chcker model` | `lmChatGroq` | Provides LLM backend for Quality Checker | `Quality Checker` | `Quality Checker` | |
| `Prepare Input` | `set` | Formats review payload for Slack | `Quality Checker` | `Send message and wait for response` | |
| `Finish` | `noOp` | Terminal workflow node | `Log Post`, `Update a row`, `Verify Decision` | None | |
| `Prepare For Regeneration` | `code` | Appends human feedback for regeneration | `Switch` | `Redirect to Post Generator` | |
| `Verify Decision` | `switch` | Routes email timeout decision webhooks | `wait for response` | `Create a post`, `Update a row`, `Finish` | |
| `Notify` | `gmail` | Sends timeout notification email | `Switch` | `wait for response` | |
| `wait for response` | `wait` | Suspends execution waiting for email webhook | `Notify` | `Verify Decision` | |
| `Log Post` | `supabase` | Logs published post metrics to DB | `Create a post` | `Finish` | **Note - Log Post**<br>Add your Supabase credentials here. Logs to `published_posts`. |
| `Normalize Input` | `code` | Truncates text to max length (6,000 chars) | `Merge` | `Resource Analyzer` | |
| `Redirect to Post Generator` | `noOp` | Routes regenerated content back to generator | `Prepare For Regeneration` | `Post Generator` | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Database Schema and Triggers
1. Set up a **Supabase** project and create the following five tables/views with exact column names:
   - `trending_topics_inventory`: `id`, `topic`, `source_type`, `reference_source`, `added_at`, `status`
   - `eligible_trending_topics` (View): Inventory topics currently eligible to post (requires `id`, `added_at`)
   - `strategy_contracts`: `topic_id`, `angle`, `format`, `hook_type`, `structure`, `cta_type`, `target_length`, `reasoning`
   - `post_review_log`: `draft_text`, `decision`, `feedback`, `reviewed_at`
   - `published_posts`: `topic_id`, `format`, `hook_type`, `structure`, `angle`, `cta_type`, `target_length`, `post_content`
2. Ensure `source_type` fields support values: `File`, `url`, or `text`.

#### Step 2: Build Entry Points (Block 1.1)
1. Create a `Schedule Trigger` node set to run daily at 10:00 AM. Connect it to a `Set` node (`Assign Run Type`) setting `run_type = "scheduled"`.
2. Create a `Google Drive Trigger` node watching specific folder ID `1kWIykK2cIPVW-HbmuACYdsSYyFnfmkhC` (polling every minute, event `fileCreated`). Connect it to a `Set` node (`Assign Run TYpe`) setting `run_type = "event"` and extracting `id`. Configure Google Drive OAuth2 credentials.

#### Step 3: Build Topic Selection Path (Block 1.2)
1. Add a Supabase node (`Get Topic From DB`) querying `eligible_trending_topics` ordered by `added_at`.
2. Add a `Filter` node (`Check for Eligible Topic`) ensuring `={{ $json.id }}` is not empty.
3. Add an AI Agent node (`Topic Selection`) connected to a Groq Chat Model (`Topic Selection Model`, model `openai/gpt-oss-120b`) and a Structured Output Parser with JSON schema containing `selected_topic_id`, `reasoning`, and `no_post_recommended`.
4. Add a Supabase node (`Map Selected Topic`) querying `trending_topics_inventory` where `id` equals selected topic ID.
5. Add an `If` node (`Topic Recommended ?`) checking if `NO_POST_RECOMMENDED` equals `true`. Route true cases to a `NoOp` node (`Nothing to Post`).

#### Step 4: Build Extraction & Normalization Pipeline (Block 1.3)
1. Add a `Switch` node (`Data Type`) routing by `run_type` and `source_type` (`file`, `File`, `url`, `text`).
2. **File Path:** Add a Google Drive node (`Download file`) downloading via file ID, followed by a `Switch` node (`File Type`) splitting MIME types `image/png` and `application/pdf`.
   - *Image Sub-path:* Add a Tesseract OCR node (`Extract Image`), then an AI Chain node (`Clean Data`) using Groq (`openai/gpt-oss-20b`), followed by a `Set` node (`Normalise Image`).
   - *PDF Sub-path:* Add an Extract From File node (`Extract PDF`), followed by a `Set` node (`Normalise Pdf`).
3. **URL Path:** Add an HTTP Request node (`Extract URl`), an HTML node (`Clean HTML File`) targeting `body`, a Code node (`Clean Extracted Data`) running custom sanitization JavaScript capping text at 6,000 characters, and a `Set` node (`Normalise Url`).
4. **Text Path:** Add a `NoOp` node (`Get The Text`) followed by a `Set` node (`Normalise Text`).
5. Combine all four normalization paths into a 4-input `Merge` node.

#### Step 5: Build Resource Analysis (Block 1.4)
1. Connect the `Merge` node output to a Code node (`Normalize Input`) ensuring text length does not exceed 6,000 characters.
2. Add an AI Agent node (`Resource Analyzer`) connected to a Groq Chat Model (`Resouce Analyzer Model`, model `openai/gpt-oss-120b`) and a Structured Output Parser (`Structured Output Parser1`) enforcing schemas for `facts`, `topic`, `angles`, `technical_details`, and `privacy_flags`.

#### Step 6: Build Content Strategy Engine (Block 1.5)
1. Add an AI Agent node (`Content Stratigist`) connected to a Google Gemini Chat Model (`Google Gemini Chat Model`, model `models/gemini-3.1-flash-lite`).
2. Attach a Pinecone Vector Store tool (`Pinecone Vector Store`) pointing to index `linkedin-reference-content` configured with Google Gemini Embeddings (`Embeddings Google Gemini`).
3. Attach a Structured Output Parser (`Structured Output Parser2`) enforcing strategy contract parameters.
4. Add a Supabase node (`Log Strategy`) writing records to `strategy_contracts`.

#### Step 7: Build Post Generation and Quality Control (Block 1.6)
1. Add an AI Agent node (`Post Generator`) connected to a Groq Chat Model (`Post Generator Model`, model `openai/gpt-oss-120b`).
2. Add an AI Agent node (`Quality Checker`) connected to a Groq Chat Model (`Quality chcker model`, model `openai/gpt-oss-120b`) and Structured Output Parser (`Structured Output Parser3`), with `onError` set to continue regular output.
3. Add a `Set` node (`Prepare Input`) formatting draft copy and evaluation summaries.

#### Step 8: Build Human Review and Decision Handling (Block 1.7)
1. Add a Slack node (`Send message and wait for response`) posting to channel `C0C25EDEAB0` with interactive form fields (`decision`, `feedback`) and wait settings configured. Requires Slack credentials.
2. Add a Supabase node (`Log Decision`) logging payloads to `post_review_log`.
3. Add a `Switch` node (`Switch`) routing decisions: `approved`, `rejected`, `regenerate`, or `TimedOut`.
4. **Regeneration Path:** Add a Code node (`Prepare For Regeneration`) appending feedback, linking back to a `NoOp` node (`Redirect to Post Generator`) which loops into `Post Generator`.
5. **Timeout Path:** Add a Gmail node (`Notify`) sending reminder emails with execution resume URLs, followed by a `Wait` node (`wait for response`) and a `Switch` node (`Verify Decision`).

#### Step 9: Build Publication and Archiving (Block 1.8)
1. Add a LinkedIn node (`Create a post`) configured with LinkedIn OAuth2 credentials.
2. Add a Supabase node (`Log Post`) saving metadata to `published_posts`.
3. Add a Supabase node (`Update a row`) updating `trending_topics_inventory` status to `archived` for rejected items.
4. Terminate all terminal branches at a `Finish` `NoOp` node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Process Implementation Guide:** Automated LinkedIn content pipeline selecting topics, analyzing resources, building strategy contracts via Pinecone, drafting posts, and publishing via Slack approval workflows. | [Implementation Guide Sticky Note](https://n8n.io) |
| **Supabase Tables & View Setup:** Required tables: `trending_topics_inventory`, `eligible_trending_topics`, `strategy_contracts`, `post_review_log`, `published_posts`. | [Supabase Schema Setup](https://supabase.com) |