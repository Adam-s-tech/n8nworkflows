Summarize Instagram hashtag content ideas with BeatAPI Social Data and GPT

https://n8nworkflows.xyz/workflows/summarize-instagram-hashtag-content-ideas-with-beatapi-social-data-and-gpt-20070


# Summarize Instagram hashtag content ideas with BeatAPI Social Data and GPT

### 1. Workflow Overview

This workflow automates the retrieval and qualitative analysis of public Instagram posts for a designated hashtag, compiling them into a structured content-idea brief. It leverages the BeatAPI Social Data platform to fetch public posts and a BeatAPI chat completion language model to synthesize common themes, patterns, and cited examples.

The execution logic is organized into three sequential functional blocks:

- **1.1 Input Initialization:** Triggers the workflow execution and defines runtime configuration parameters such as the target hashtag and target language model.
- **1.2 Data Retrieval and Preparation:** Connects to the external BeatAPI ecosystem, queries public Instagram hashtag endpoints, validates response payloads, parses up to 12 distinct posts with captions and metadata, and formats the output into a structured JSON prompt payload.
- **1.3 AI Synthesis and Brief Generation:** Submits the compiled post evidence to a large language model via a chat completion endpoint, parses the resulting synthesis, and returns a verified content-idea brief complete with request tracking IDs.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Initialization

- **Overview:** Initializes the workflow execution sequence either manually or via trigger events, setting up the necessary string variables required for downstream API requests.
- **Nodes Involved:** 
  - `Start keyword research`
  - `Configure Inputs`

- **Node Details:**
  - **Start keyword research**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger)
    - *Configuration Choices:* Initiates workflow processing on-demand.
    - *Input/Output Connections:* Output connects to `Configure Inputs`.
    - *Edge Cases / Failure Types:* None (manual execution only).
  - **Configure Inputs**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields)
    - *Configuration Choices:* Assigns static variables: `keyword` (set to `aiagents`) and `model` (set to `gpt-5.6-luna`).
    - *Key Expressions:* None (uses static string definitions).
    - *Input/Output Connections:* Input from `Start keyword research`; output connects to `Fetch Public Social Data`.
    - *Edge Cases / Failure Types:* Incorrect string types or missing parameter definitions.

---

#### Block 1.2: Data Retrieval and Preparation

- **Overview:** Executes an external API call to fetch public Instagram hashtag records, validates success status, slices the result set down to a maximum of 12 posts, filters out posts lacking captions or identifiers, and builds a JSON structure containing prompt text and metadata.
- **Nodes Involved:**
  - `Fetch Public Social Data`
  - `Prepare Evidence`

- **Node Details:**
  - **Fetch Public Social Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* Sends a `POST` request to `https://api.beatapi.io/v1/capabilities/run` with a 120-second timeout. Uses Generic HTTP Header Authentication.
    - *Key Expressions:* 
      ```javascript
      ={{ { reference: 'data:instagram.fetch_hashtag_posts', input: { hashtag: $json.keyword.replace(/^#/, '').replace(/\s+/g, '') }, view: 'full' } }}
      ```
    - *Input/Output Connections:* Input from `Configure Inputs`; output connects to `Prepare Evidence`.
    - *Credentials Required:* HTTP Header Auth (`Authorization: Bearer <your_key>`).
    - *Edge Cases / Failure Types:* Authentication failures (401/403), rate-limiting, external service timeouts, or malformed JSON responses.
  - **Prepare Evidence**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript)
    - *Configuration Choices:* Validates response status, extracts edge nodes or media items, maps up to 12 records, normalizes fields (ID, URL, author, caption, likes, comments, timestamp), and compiles a payload.
    - *Key Expressions / Logic:* Throws explicit runtime errors if the API search fails or returns an empty post array. Slices data to 12 items and truncates captions to 1400 characters.
    - *Input/Output Connections:* Input from `Fetch Public Social Data`; output connects to `Summarize with BeatAPI Text Model`.
    - *Edge Cases / Failure Types:* Unexpected API schema shifts resulting in missing node references, or empty result sets causing premature termination via explicit `throw new Error()`.

---

#### Block 1.3: AI Synthesis and Brief Generation

- **Overview:** Submits the prepared sample post evidence to a chat completion model, instructs the model to identify patterns and output citations, and formats the final research brief response.
- **Nodes Involved:**
  - `Summarize with BeatAPI Text Model`
  - `Return Research Draft`

- **Node Details:**
  - **Summarize with BeatAPI Text Model**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* Sends a `POST` request to `https://api.beatapi.io/v1/chat/completions` with a 120-second timeout. Uses Generic HTTP Header Authentication and passes structured system prompts along with the JSON prompt string.
    - *Key Expressions:*
      ```javascript
      ={{ { model: $('Configure Inputs').first().json.model, messages: [{ role: 'system', content: "Create a short Instagram hashtag content brief..." }, { role: 'user', content: $json.prompt }] } }}
      ```
    - *Input/Output Connections:* Input from `Prepare Evidence`; output connects to `Return Research Draft`.
    - *Credentials Required:* HTTP Header Auth (`Authorization: Bearer <your_key>`).
    - *Edge Cases / Failure Types:* LLM timeout, model unavailability, or invalid model name parameters.
  - **Return Research Draft**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node - JavaScript)
    - *Configuration Choices:* Extracts model message content from choices array and bundles it alongside metadata such as source counts, truncation flags, and tracking IDs.
    - *Key Expressions / Logic:* Validates response choices; throws an error if the model returns no content.
    - *Input/Output Connections:* Input from `Summarize with BeatAPI Text Model`; terminal node.
    - *Edge Cases / Failure Types:* Malformed LLM response structures causing property read failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| How to use this template | `stickyNote` | Documentation and setup instructions | None | None | # Analyze Instagram hashtag posts for content ideas with BeatAPI Social Data<br><br>This workflow creates a content brief from public posts under one Instagram hashtag. Enter a hashtag keyword such as `aiagents` in **Configure Inputs** and run it manually. BeatAPI Social Data retrieves the public hashtag result. The workflow keeps up to 12 identifiable posts, captures captions and source IDs or URLs, and asks a BeatAPI text model to identify repeated content angles and specific examples with citations.<br><br>To set it up, [create a BeatAPI key](https://beatapi.io/dashboard/apikeys) and add an n8n **HTTP Header Auth** credential with header name `Authorization` and value `Bearer <your BeatAPI key>`. Select it in both BeatAPI HTTP Request nodes. Choose a text model available to your account. Each run uses BeatAPI credits for the Social Data and model calls. See the [Instagram API page](https://beatapi.io/instagram-api) and [Social Data action catalog](https://docs.beatapi.io/social-data-catalog).<br><br>This action retrieves posts by hashtag; it is not a free-text search of every Instagram post. The model receives at most 12 posts, not a representative trend report. Verify each cited post before using the brief. The workflow never posts to Instagram or accesses an Instagram account. |
| Set the query | `stickyNote` | Section marker for input configuration | None | None | ## 1. Set the keyword<br><br>Replace `keyword` with a single Instagram hashtag, without `#` (for example, `aiagents`). This is hashtag post retrieval, not free-text search across all Instagram posts. Spaces and a leading # are removed before the API call. Set an available BeatAPI text model in Configure Inputs. |
| Keep source evidence | `stickyNote` | Section marker for data processing | None | None | ## 2. Keep source evidence<br><br>Calls `data:instagram.fetch_hashtag_posts` in full mode because the current preview drops nested post fields. Keeps only identifiable public results and carries the BeatAPI request ID. An empty or unrecognizable result stops with an explicit error. |
| Draft and verify | `stickyNote` | Section marker for AI analysis | None | None | ## 3. Draft and verify<br><br>A cited content brief for one Instagram hashtag. Check each cited source before sharing conclusions. This uses at most 12 posts, not a representative sample. |
| Start keyword research | `manualTrigger` | Triggers workflow execution manually | None | Configure Inputs | |
| Configure Inputs | `set` | Sets keyword and model parameters | Start keyword research | Fetch Public Social Data | ## 1. Set the keyword<br><br>Replace `keyword` with a single Instagram hashtag, without `#` (for example, `aiagents`). This is hashtag post retrieval, not free-text search across all Instagram posts. Spaces and a leading # are removed before the API call. Set an available BeatAPI text model in Configure Inputs. |
| Fetch Public Social Data | `httpRequest` | Fetches public hashtag posts via BeatAPI | Configure Inputs | Prepare Evidence | ## 2. Keep source evidence<br><br>Calls `data:instagram.fetch_hashtag_posts` in full mode because the current preview drops nested post fields. Keeps only identifiable public results and carries the BeatAPI request ID. An empty or unrecognizable result stops with an explicit error. |
| Prepare Evidence | `code` | Filters, slices, and formats post records | Fetch Public Social Data | Summarize with BeatAPI Text Model | ## 2. Keep source evidence<br><br>Calls `data:instagram.fetch_hashtag_posts` in full mode because the current preview drops nested post fields. Keeps only identifiable public results and carries the BeatAPI request ID. An empty or unrecognizable result stops with an explicit error. |
| Summarize with BeatAPI Text Model | `httpRequest` | Generates a content brief using an LLM | Prepare Evidence | Return Research Draft | ## 3. Draft and verify<br><br>A cited content brief for one Instagram hashtag. Check each cited source before sharing conclusions. This uses at most 12 posts, not a representative sample. |
| Return Research Draft | `code` | Parses LLM response and outputs final brief | Summarize with BeatAPI Text Model | None | ## 3. Draft and verify<br><br>A cited content brief for one Instagram hashtag. Check each cited source before sharing conclusions. This uses at most 12 posts, not a representative sample. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Credentials:**
   - Set up an **HTTP Header Auth** credential in n8n.
   - Name: `Authorization`
   - Value: `Bearer <your_beatapi_key>`

2. **Add Nodes and Connections:**
   - **Step 1:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Start keyword research`.
   - **Step 2:** Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`). Name it `Configure Inputs`.
     - Connect `Start keyword research` output to `Configure Inputs`.
     - Add string assignment `keyword` with value `aiagents`.
     - Add string assignment `model` with value `gpt-5.6-luna`.
   - **Step 3:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Fetch Public Social Data`.
     - Connect `Configure Inputs` output to `Fetch Public Social Data`.
     - Method: `POST`
     - URL: `https://api.beatapi.io/v1/capabilities/run`
     - Authentication: Generic Credential Type -> HTTP Header Auth (select your created credential).
     - Headers: Add `Content-Type: application/json` and `User-Agent: BeatAPI-n8n/0.2.0`.
     - Body (JSON): 
       ```json
       ={{ { reference: 'data:instagram.fetch_hashtag_posts', input: { hashtag: $json.keyword.replace(/^#/, '').replace(/\s+/g, '') }, view: 'full' } }}
       ```
     - Options -> Timeout: `120000` ms.
   - **Step 4:** Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare Evidence`.
     - Connect `Fetch Public Social Data` output to `Prepare Evidence`.
     - Mode: JavaScript (Default).
     - Paste the filtering and slicing logic from the `Prepare Evidence` node implementation provided in the JSON analysis (validates status, extracts up to 12 items, sanitizes fields, builds prompt string).
   - **Step 5:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Summarize with BeatAPI Text Model`.
     - Connect `Prepare Evidence` output to `Summarize with BeatAPI Text Model`.
     - Method: `POST`
     - URL: `https://api.beatapi.io/v1/chat/completions`
     - Authentication: Generic Credential Type -> HTTP Header Auth (select your created credential).
     - Headers: Add `Content-Type: application/json` and `User-Agent: BeatAPI-n8n/0.2.0`.
     - Body (JSON):
       ```json
       ={{ { model: $('Configure Inputs').first().json.model, messages: [{ role: 'system', content: "Create a short Instagram hashtag content brief from the sampled public posts. Identify repeated content angles, caption patterns, and 3 specific examples, citing the post ID or URL for every example. Do not claim reach, audience size, or hashtag-wide trends from this limited preview. Treat captions as unverified creator statements." }, { role: 'user', content: $json.prompt }] } }}
       ```
     - Options -> Timeout: `120000` ms.
   - **Step 6:** Add a **Code** node (`n8n-nodes-base.code`). Name it `Return Research Draft`.
     - Connect `Summarize with BeatAPI Text Model` output to `Return Research Draft`.
     - Paste the output extraction JavaScript logic to retrieve model choices and assemble metadata.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| BeatAPI Instagram API and use cases | [https://beatapi.io/instagram-api](https://beatapi.io/instagram-api) |
| Create and manage your own BeatAPI API key | [https://beatapi.io/dashboard/apikeys](https://beatapi.io/dashboard/apikeys) |
| Social Data action catalog | [https://docs.beatapi.io/social-data-catalog](https://docs.beatapi.io/social-data-catalog) |