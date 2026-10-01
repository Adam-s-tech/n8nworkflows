Summarize YouTube keyword video topics for research with BeatAPI Social Data

https://n8nworkflows.xyz/workflows/summarize-youtube-keyword-video-topics-for-research-with-beatapi-social-data-20071


# Summarize YouTube keyword video topics for research with BeatAPI Social Data

### 1. Workflow Overview

This workflow automates the process of researching YouTube video topics using BeatAPI Social Data and generating a cited topic brief via a BeatAPI text model. It is designed for content creators, researchers, and marketers who want to quickly synthesize recurring themes, common questions, and specific video examples based on recent public search results for a given keyword.

The workflow operates in a linear sequence, grouped into the following logical blocks:
- **1.1 Input Reception & Configuration:** Manually triggers the workflow and sets the target search keyword and AI model parameters.
- **1.2 Social Data Retrieval & Filtering:** Queries the BeatAPI Social Data capability to fetch public YouTube video results, parses and normalizes the evidence, and validates the output.
- **1.3 AI Processing & Delivery:** Sends the structured evidence to a BeatAPI chat/completions model to generate a cited research brief and returns the final output alongside source metadata.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Initializes the execution manually and defines the core parameters (keyword and text model) required for the subsequent API requests.
- **Nodes Involved:** 
  - `Start keyword research`
  - `Configure Inputs`
- **Node Details:**
  - **Start keyword research**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Entry point for manual executions.
    - *Configuration Choices:* Standard manual trigger with no custom parameters.
    - *Input/Output:* No inputs; outputs execution signal to `Configure Inputs`.
    - *Version-specific requirements:* Version 1.
    - *Edge cases:* None (manual execution only).
  - **Configure Inputs**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Sets predefined workflow variables.
    - *Configuration Choices:* Assigns a string `keyword` (default: `"AI agents"`) and a string `model` (default: `"gpt-5.6-luna"`).
    - *Key Expressions:* `$json.keyword`, `$json.model`.
    - *Input/Output:* Input from `Start keyword research`; output connected to `Fetch Public Social Data`.
    - *Version-specific requirements:* Version 3.4.
    - *Edge cases:* Missing or misspelled model names will cause downstream failures in the AI model step.

#### Block 1.2: Social Data Retrieval & Filtering
- **Overview:** Fetches public YouTube search results via BeatAPI, extracts relevant metadata (titles, descriptions, URLs), filters out incomplete entries, and ensures valid data exists before proceeding.
- **Nodes Involved:**
  - `Fetch Public Social Data`
  - `Prepare Evidence`
- **Node Details:**
  - **Fetch Public Social Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Makes an authenticated POST request to the BeatAPI capability runner.
    - *Configuration Choices:* Uses HTTP Header Authentication (`httpHeaderAuth`). Sends a JSON payload requesting `data:youtube.web.search_video` in `preview` view mode, limited to 12 items, filtered by `this_month`, English (`en`), and US (`us`). Timeout set to 120,000ms.
    - *Key Expressions:* `={{ { reference: 'data:youtube.web.search_video', input: { search_query: $json.keyword, order_by: 'this_month', language_code: 'en', country_code: 'us' }, view: 'preview', max_items: 12 } }}`
    - *Input/Output:* Input from `Configure Inputs`; output connected to `Prepare Evidence`.
    - *Version-specific requirements:* Version 4.2.
    - *Edge cases:* Authentication errors (invalid Bearer token), API timeouts, rate limiting, or empty search responses.
  - **Prepare Evidence**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript code node for data sanitization and validation.
    - *Configuration Choices:* Validates that the API status is `succeeded`, extracts up to 12 video items, normalizes video IDs and URLs, truncates titles and descriptions to safe lengths, and filters out entries lacking both IDs/URLs and textual content. Throws an explicit error if no valid records are found.
    - *Key Expressions:* Accesses `$input.first().json` and references the upstream keyword via `$('Configure Inputs').first().json.keyword`.
    - *Input/Output:* Input from `Fetch Public Social Data`; output connected to `Summarize with BeatAPI Text Model`.
    - *Version-specific requirements:* Version 2.
    - *Edge cases:* Unexpected JSON schemas from the API or zero search results will throw explicit JavaScript errors and halt the workflow.

#### Block 1.3: AI Processing & Delivery
- **Overview:** Submits the compiled evidence to a BeatAPI language model to synthesize a structured research brief, then formats and returns the final output.
- **Nodes Involved:**
  - `Summarize with BeatAPI Text Model`
  - `Return Research Draft`
- **Node Details:**
  - **Summarize with BeatAPI Text Model**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Calls the BeatAPI chat/completions endpoint using HTTP Header Authentication.
    - *Configuration Choices:* Sends a POST request with system and user prompts instructing the model to generate a topic brief citing specific video IDs or URLs. Timeout set to 120,000ms.
    - *Key Expressions:* Uses `$('Configure Inputs').first().json.model` for the model name and `$json.prompt` for the user content.
    - *Input/Output:* Input from `Prepare Evidence`; output connected to `Return Research Draft`.
    - *Version-specific requirements:* Version 4.2.
    - *Edge cases:* Model availability issues, token limit overflows if descriptions are overly long, or API authentication failures.
  - **Return Research Draft**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript code node for final response formatting.
    - *Configuration Choices:* Extracts the AI-generated message content, merges it with source metadata (source count, total items, truncation status, request IDs), and returns a clean JSON output object.
    - *Key Expressions:* Accesses `$input.first().json.choices?.[0]?.message?.content` and `$('Prepare Evidence').first().json`.
    - *Input/Output:* Input from `Summarize with BeatAPI Text Model`; terminal node.
    - *Version-specific requirements:* Version 2.
    - *Edge cases:* Empty response choices from the language model will trigger a custom error.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| How to use this template | n8n-nodes-base.stickyNote | Workflow documentation and configuration guide | None | None | # Summarize YouTube keyword video results for topic research with BeatAPI Social Data<br><br>This workflow helps video creators turn a YouTube keyword search into a cited topic brief. Set a keyword in **Configure Inputs** and run it manually. BeatAPI Social Data searches public YouTube video results from this month with English and US defaults. The workflow keeps up to 12 identifiable results, including title, description, channel, video ID or URL and the BeatAPI request ID. A BeatAPI text model summarizes recurring topics and questions and cites specific videos.<br><br>To set it up, [create a BeatAPI key](https://beatapi.io/dashboard/apikeys) and add it in n8n as an **HTTP Header Auth** credential with header name `Authorization` and value `Bearer <your BeatAPI key>`. Select it in both BeatAPI HTTP Request nodes. Choose a text model available to your account. Each run uses BeatAPI credits for the Social Data and model calls. See the [YouTube API page](https://beatapi.io/youtube-api) and [Social Data action catalog](https://docs.beatapi.io/social-data-catalog).<br><br>This workflow reads search result metadata; it does not watch videos or fetch transcripts. Its output is a limited research draft, so check the linked videos before using the brief externally. It never uploads or publishes to YouTube. |
| Set the query | n8n-nodes-base.stickyNote | Documentation block for query setup | None | None | ## 1. Set the keyword<br><br>Replace `keyword` with your YouTube topic. The search defaults to this month, English, and US. Change these documented filters in Fetch Public Social Data to match your audience. Set an available BeatAPI text model in Configure Inputs. |
| Keep source evidence | n8n-nodes-base.stickyNote | Documentation block for evidence collection | None | None | ## 2. Keep source evidence<br><br>Calls `data:youtube.web.search_video` in preview mode. Keeps only identifiable public results and carries the BeatAPI request ID. An empty or unrecognizable preview stops with an explicit error. |
| Draft and verify | n8n-nodes-base.stickyNote | Documentation block for AI summarization | None | None | ## 3. Draft and verify<br><br>A cited YouTube video topic brief from keyword results. Check each cited source before sharing conclusions. This is a limited preview, not a representative sample. |
| Start keyword research | n8n-nodes-base.manualTrigger | Workflow entry point | None | Configure Inputs | |
| Configure Inputs | n8n-nodes-base.set | Defines search keyword and AI model parameters | Start keyword research | Fetch Public Social Data | |
| Fetch Public Social Data | n8n-nodes-base.httpRequest | Fetches public YouTube video search results via BeatAPI | Configure Inputs | Prepare Evidence | |
| Prepare Evidence | n8n-nodes-base.code | Normalizes, filters, and formats raw API search results | Fetch Public Social Data | Summarize with BeatAPI Text Model | |
| Summarize with BeatAPI Text Model | n8n-nodes-base.httpRequest | Generates a cited topic brief using a BeatAPI text model | Prepare Evidence | Return Research Draft | |
| Return Research Draft | n8n-nodes-base.code | Formats and outputs the final research brief and metadata | Summarize with BeatAPI Text Model | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Start keyword research`.
2. **Create a Set Node for Inputs:**
   - Add a **Set** node (`n8n-nodes-base.set`). Name it `Configure Inputs`.
   - Configure string assignments:
     - `keyword`: `"AI agents"`
     - `model`: `"gpt-5.6-luna"`
   - Connect `Start keyword research` to `Configure Inputs`.
3. **Configure BeatAPI Credentials:**
   - Create a new **HTTP Header Auth** credential in n8n.
   - Set the header name to `Authorization` and value to `Bearer <your BeatAPI key>`.
4. **Create the Social Data HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Fetch Public Social Data`.
   - Set Method to `POST`, URL to `https://api.beatapi.io/v1/capabilities/run`.
   - Set Authentication to `Generic Credential Type` -> `HTTP Header Auth` (select your BeatAPI credential).
   - Add Header parameters: `Content-Type: application/json` and `User-Agent: BeatAPI-n8n/0.2.0`.
   - Set Body to JSON and paste the expression:
     `={{ { reference: 'data:youtube.web.search_video', input: { search_query: $json.keyword, order_by: 'this_month', language_code: 'en', country_code: 'us' }, view: 'preview', max_items: 12 } }}`
   - Set timeout option to `120000`.
   - Connect `Configure Inputs` to `Fetch Public Social Data`.
5. **Create the Evidence Preparation Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare Evidence`.
   - Paste the following JavaScript:
     ```javascript
     const result = $input.first().json;
     if (result.status !== 'succeeded') throw new Error('BeatAPI Social Data search did not succeed.');
     const items = Array.isArray(result.items) ? result.items : (Array.isArray(result.data?.items) ? result.data.items : null);
     if (!items) throw new Error('No preview items array was returned. Inspect the Social Data response and action availability.');
     const records = items.slice(0, 12).map((p) => {
     const id = String(p.video_id ?? p.videoId ?? p.id?.videoId ?? p.id ?? '');
     const url = p.url ?? p.link ?? (id ? 'https://www.youtube.com/watch?v=' + id : '');
     return { id, url, title: String(p.title ?? p.video_title ?? '').slice(0, 350), description: String(p.description ?? p.text ?? '').slice(0, 1300), channel: p.channel_title ?? p.channel?.name ?? p.author ?? '', views: p.view_count ?? p.views ?? null, published_at: p.published_at ?? p.publish_time ?? null };
     }).filter((p) => (p.id || p.url) && (p.title || p.text || p.caption || p.description));
     if (!records.length) throw new Error('No identifiable public results were found for this keyword. Try another keyword or inspect the preview response.');
     return [{ json: { source: "YouTube public video search", source_count: records.length, items_total: result.items_total ?? null, preview_truncated: Boolean(result.truncated || (typeof result.items_total === 'number' && result.items_total > records.length)), request_ids: result.request_id ? [result.request_id] : [], prompt: JSON.stringify({ keyword: $('Configure Inputs').first().json.keyword, records }) } }];
     ```
   - Connect `Fetch Public Social Data` to `Prepare Evidence`.
6. **Create the AI Model HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Summarize with BeatAPI Text Model`.
   - Set Method to `POST`, URL to `https://api.beatapi.io/v1/chat/completions`.
   - Set Authentication to `Generic Credential Type` -> `HTTP Header Auth` (select your BeatAPI credential).
   - Add Header parameters: `Content-Type: application/json` and `User-Agent: BeatAPI-n8n/0.2.0`.
   - Set Body to JSON and paste the expression:
     `={{ { model: $('Configure Inputs').first().json.model, messages: [{ role: 'system', content: "Create a YouTube video topic brief from the sampled public search results. Identify recurring topics, common questions, and 3 specific videos with a video ID or URL citation for each example. Titles and descriptions are creator claims, not verified transcripts. Do not pretend you watched the videos or infer platform-wide trends from this limited preview." }, { role: 'user', content: $json.prompt }] } }}`
   - Set timeout option to `120000`.
   - Connect `Prepare Evidence` to `Summarize with BeatAPI Text Model`.
7. **Create the Output Formatting Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Return Research Draft`.
   - Paste the following JavaScript:
     ```javascript
     const answer = $input.first().json.choices?.[0]?.message?.content;
     if (!answer) throw new Error('The text model returned no message content.');
     const evidence = $('Prepare Evidence').first().json;
     return [{ json: { summary: answer, source: evidence.source, source_count: evidence.source_count, items_total: evidence.items_total, preview_truncated: evidence.preview_truncated, request_ids: evidence.request_ids } }];
     ```
   - Connect `Summarize with BeatAPI Text Model` to `Return Research Draft`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create and manage your BeatAPI API key | [https://beatapi.io/dashboard/apikeys](https://beatapi.io/dashboard/apikeys) |
| BeatAPI YouTube API and use cases | [https://beatapi.io/youtube-api](https://beatapi.io/youtube-api) |
| Social Data action catalog | [https://docs.beatapi.io/social-data-catalog](https://docs.beatapi.io/social-data-catalog) |