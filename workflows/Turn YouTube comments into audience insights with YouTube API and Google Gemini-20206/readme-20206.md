Turn YouTube comments into audience insights with YouTube API and Google Gemini

https://n8nworkflows.xyz/workflows/turn-youtube-comments-into-audience-insights-with-youtube-api-and-google-gemini-20206


# Turn YouTube comments into audience insights with YouTube API and Google Gemini

### 1. Workflow Overview

This workflow is an AI-powered audience research pipeline that mines public YouTube video comments for recurring questions, pain points, content ideas, and product opportunities. It captures user inputs via a form interface, fetches metadata and comment threads directly from the YouTube Data API v3 without requiring a paid scraper, filters out low-value content and spam, structures the data into an evidence-backed format, uses Google Gemini to analyze the demand, and renders a fully styled HTML report back to the user within n8n.

The workflow logic is divided into five functional blocks:
- **1.1 Input Reception & Normalization:** Captures user inputs from the research form and normalizes the target video URL or ID.
- **1.2 YouTube Data Extraction:** Queries the YouTube Data API v3 to collect video statistics and comment threads (including replies).
- **1.3 Data Cleansing & Preparation:** Filters out junk and spam, removes duplicates, calculates engagement weights, assigns stable evidence IDs (`C001`), and prepares a compact payload for AI processing.
- **1.4 AI Demand Analysis:** Passes the cleaned comment dataset to an AI Agent powered by Google Gemini using an evidence-first system prompt.
- **1.5 Report Generation & Delivery:** Validates AI outputs against collected evidence, generates a responsive HTML report with deep-links back to individual YouTube comments, and presents the output via a form completion step.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization

##### Overview
This block acts as the entry point of the workflow. It presents a form to the user to collect a public YouTube URL, the maximum number of comments to analyze, and the comment sorting strategy, then sanitizes and normalizes the URL into a valid video identifier.

##### Nodes Involved
- `When Form Submitted`
- `Prepare Video Data`

##### Node Details

###### When Form Submitted
- **Type and Technical Role:** `n8n-nodes-base.formTrigger` (Trigger Node). Acts as a built-in web form to capture research requests.
- **Configuration Choices:** Configured with a button label (`Analyze audience`), form title (`YouTube Audience Intelligence`), and a description. Includes three form fields:
  1. *YouTube video URL* (Text input, required)
  2. *Comments to analyze* (Number input, default value `100`, required)
  3. *Comment selection* (Dropdown with options `relevance` and `time`, default value `relevance`, required)
- **Key Expressions or Variables:** Sets `responseMode` to `lastNode`.
- **Input and Output Connections:** No input connections (Trigger). Outputs to `Prepare Video Data`.
- **Version-specific Requirements:** TypeVersion 2.3.
- **Edge Cases or Potential Failure Types:** Users submitting invalid URLs, malformed identifiers, or empty strings. Handled downstream by the normalization code node.
- **Sub-workflow Reference:** None.

###### Prepare Video Data
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation). Parses the submitted URL using regular expressions to extract an 11-character YouTube video ID (supporting standard watch URLs, short links, shorts, embeds, and live URLs). Clamps the comment count between 25 and 100, enforces sorting rules, and sets a minimum character length (`12`) for valid comment filtering.
- **Configuration Choices:** JavaScript execution environment (`v2`).
- **Key Expressions or Variables:** Reads `$input.first().json["YouTube video URL"]`, `$input.first().json["Comments to analyze"]`, and `$input.first().json["Comment selection"]`.
- **Input and Output Connections:** Input received from `When Form Submitted`. Output connects to `Fetch Video Details`.
- **Version-specific Requirements:** TypeVersion 2.
- **Edge Cases or Potential Failure Types:** Throws an explicit error (`Invalid YouTube URL or video ID`) if no valid 11-character video ID can be matched.
- **Sub-workflow Reference:** None.

---

#### Block 1.2: YouTube Data Extraction

##### Overview
This block interacts with the external YouTube Data API v3 to retrieve comprehensive video metadata (title, channel, view count, like count, comment count) and public comment threads with associated replies.

##### Nodes Involved
- `Fetch Video Details`
- `Fetch Comment Threads`

##### Node Details

###### Fetch Video Details
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Calls the YouTube Data `videos` endpoint to retrieve video metadata and statistics.
- **Configuration Choices:** Uses GET method against `https://www.googleapis.com/youtube/v3/videos`. Configured with `genericQueryAuth` authentication. Query parameters include `part` set to `snippet,statistics` and `id` bound to the prepared video ID.
- **Key Expressions or Variables:** `={{ $('Prepare Video Data').first().json.videoId }}`
- **Input and Output Connections:** Input received from `Prepare Video Data`. Output connects to `Fetch Comment Threads`.
- **Version-specific Requirements:** TypeVersion 4.4.
- **Edge Cases or Potential Failure Types:** API authentication failures, revoked keys, or queries for private/deleted videos.
- **Sub-workflow Reference:** None.

###### Fetch Comment Threads
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Calls the YouTube Data `commentThreads` endpoint to fetch top-level comments and replies.
- **Configuration Choices:** Uses GET method against `https://www.googleapis.com/youtube/v3/commentThreads`. Query parameters include `part` (`snippet,replies`), `videoId`, `maxResults`, `order`, and `textFormat` set to `plainText`.
- **Key Expressions or Variables:** 
  - `videoId`: `={{ $('Prepare Video Data').first().json.videoId }}`
  - `maxResults`: `={{ $('Prepare Video Data').first().json.maxComments }}`
  - `order`: `={{ $('Prepare Video Data').first().json.order }}`
- **Input and Output Connections:** Input received from `Fetch Video Details`. Output connects to `Prepare Comments Data`.
- **Version-specific Requirements:** TypeVersion 4.4.
- **Edge Cases or Potential Failure Types:** Comments disabled on the target video or rate-limiting by Google Cloud APIs.
- **Sub-workflow Reference:** None.

---

#### Block 1.3: Data Cleansing & Preparation

##### Overview
This block processes the raw API responses, cleans HTML tags and emojis, strips out low-value spam (e.g., "first", "nice video"), deduplicates comments, calculates an engagement score based on likes and replies, assigns deterministic evidence IDs (`C001`), and prepares a compact dataset for AI consumption.

##### Nodes Involved
- `Prepare Comments Data`

##### Node Details

###### Prepare Comments Data
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation). Iterates through top-level comments and replies, applies sanitization and junk-filtering regex arrays, ranks comments by engagement or publication date, and slices the result to the requested maximum length.
- **Configuration Choices:** JavaScript execution environment (`v2`).
- **Key Expressions or Variables:** Reads data from `Prepare Video Data`, `Fetch Video Details`, and `Fetch Comment Threads`.
- **Input and Output Connections:** Input received from `Fetch Comment Threads`. Output connects to `Content Demand Analysis Agent`.
- **Version-specific Requirements:** TypeVersion 2.
- **Edge Cases or Potential Failure Types:** Handles missing error flags or empty comment threads gracefully by throwing descriptive errors if video lookups fail or comments are inaccessible.
- **Sub-workflow Reference:** None.

---

#### Block 1.4: AI Demand Analysis

##### Overview
This block leverages Google Gemini via an advanced LangChain AI Agent. The agent operates under strict evidence-first constraints to extract audience signals, content opportunities, and product ideas from the sanitized comments without hallucinating demand.

##### Nodes Involved
- `Google Gemini Model`
- `Content Demand Analysis Agent`

##### Node Details

###### Google Gemini Model
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Sub-node / Language Model). Provides the underlying LLM chat model configuration for the agent.
- **Configuration Choices:** Uses model name `models/gemini-2.5-flash`.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Connects as a language model provider to `Content Demand Analysis Agent`.
- **Version-specific Requirements:** TypeVersion 1.1.
- **Edge Cases or Potential Failure Types:** API key quotas or network timeouts during model inference.
- **Sub-workflow Reference:** None.

###### Content Demand Analysis Agent
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent Node). Analyzes the prepared comment dataset and structures the output into an executive summary, audience signals, content opportunities, product/service opportunities, and caveats.
- **Configuration Choices:** 
  - **Prompt Type:** `define`
  - **System Message:** Instructs the agent to act as an evidence-first YouTube audience researcher, utilizing *only* supplied comments, citing exact `evidence_id` references, and avoiding fabricated demand or quotes.
  - **Text:** JSON stringified input of the prepared comment dataset.
- **Key Expressions or Variables:** `={{JSON.stringify($('Prepare Comments Data').first().json)}}`
- **Input and Output Connections:** Receives input data from `Prepare Comments Data` and connects to `Google Gemini Model` for its language model. Output connects to `Generate Research Report`.
- **Version-specific Requirements:** TypeVersion 3.1.
- **Edge Cases or Potential Failure Types:** Malformed JSON output from the LLM or failure to adhere to evidence citation requirements.
- **Sub-workflow Reference:** None.

---

#### Block 1.5: Report Generation & Delivery

##### Overview
This block validates the AI-generated findings against actual collected evidence IDs, builds a fully responsive HTML report complete with inline styles and direct jump-links to individual YouTube comments, and presents the output to the user via an n8n form completion response.

##### Nodes Involved
- `Generate Research Report`
- `Display Report Form`

##### Node Details

###### Generate Research Report
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation). Parses the AI agent response (supporting standard JSON and markdown-fenced blocks), validates cited evidence IDs, maps metrics (such as unique commenters and total likes), and generates an executive HTML email-safe layout.
- **Configuration Choices:** JavaScript execution environment (`v2`).
- **Key Expressions or Variables:** Reads AI output from `Content Demand Analysis Agent` and comment metadata from `Prepare Comments Data`.
- **Input and Output Connections:** Input received from `Content Demand Analysis Agent`. Output connects to `Display Report Form`.
- **Version-specific Requirements:** TypeVersion 2.
- **Edge Cases or Potential Failure Types:** Unparseable AI responses or missing evidence identifiers are caught and handled by extraction helpers and fallback defaults.
- **Sub-workflow Reference:** None.

###### Display Report Form
- **Type and Technical Role:** `n8n-nodes-base.form` (Form Response Node). Displays the final HTML analysis report immediately upon completion.
- **Configuration Choices:** Operation set to `completion`, responding with `showText`, and rendering the HTML generated in the preceding code node.
- **Key Expressions or Variables:** `={{ $('Generate Research Report').item.json.reportHtml }}`
- **Input and Output Connections:** Input received from `Generate Research Report`. No outgoing connections (Terminal Node).
- **Version-specific Requirements:** TypeVersion 2.5.
- **Edge Cases or Potential Failure Types:** Form session expiration if left idle for extended periods.
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & Overview | None | None | ## Turn YouTube Comments Into Audience Demand & Content Ideas with AI<br><br>### How it works<br><br>This workflow starts from a research form, normalizes the submitted YouTube video input, and retrieves video metadata plus comments from the YouTube Data API. It packages the video and comment data for an AI agent, which uses Gemini to infer audience demand, pain points, questions, and content opportunities. The workflow then builds an evidence-backed report and displays it back to the user in a form response.<br><br>### Setup steps<br><br>- Configure the YouTube Data API access used by the HTTP Request nodes, including a valid API key or credential with permission to call video and commentThreads endpoints.<br>- Configure the Google Gemini credential/model used by the AI Agent's Google Gemini Chat Model sub-node.<br>- Review the form fields in "YouTube Research Form" so they collect the expected YouTube URL or video ID and any research options required by the normalization code.<br>- Verify the code nodes' assumptions for input field names, API response structure, comment limits, and report formatting before running with production data.<br><br>### Customization<br><br>You can adjust the number of comments fetched, the fields included in the AI research packet, the Gemini model or prompt, and the final report structure to match different content strategy frameworks. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Block Documentation | None | None | ## Collect research input<br><br>Captures the user's YouTube research request and normalizes/configures the submitted video information for downstream API calls. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Block Documentation | None | None | ## Fetch YouTube data<br><br>Retrieves the target video's details and comment thread data, then prepares a consolidated research packet for AI analysis. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Block Documentation | None | None | ## Analyze audience demand<br><br>Uses the AI Agent with the connected Google Gemini chat model to analyze the prepared comments and video context for audience demand signals and content ideas. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Block Documentation | None | None | ## Build and show report<br><br>Transforms the AI analysis into an evidence-backed report and presents the finished output to the user. |
| `When Form Submitted` | `n8n-nodes-base.formTrigger` | Triggers the workflow via web form | None | `Prepare Video Data` | |
| `Prepare Video Data` | `n8n-nodes-base.code` | Normalizes video URL and settings | `When Form Submitted` | `Fetch Video Details` | |
| `Fetch Video Details` | `n8n-nodes-base.httpRequest` | Retrieves video metadata | `Prepare Video Data` | `Fetch Comment Threads` | |
| `Fetch Comment Threads` | `n8n-nodes-base.httpRequest` | Retrieves comment threads & replies | `Fetch Video Details` | `Prepare Comments Data` | |
| `Prepare Comments Data` | `n8n-nodes-base.code` | Cleans, filters, and formats comments | `Fetch Comment Threads` | `Content Demand Analysis Agent` | |
| `Google Gemini Model` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Language model provider for AI Agent | None | `Content Demand Analysis Agent` | |
| `Content Demand Analysis Agent` | `@n8n/n8n-nodes-langchain.agent` | Analyzes comments for demand signals | `Prepare Comments Data`, `Google Gemini Model` | `Generate Research Report` | |
| `Generate Research Report` | `n8n-nodes-base.code` | Builds HTML report & validates IDs | `Content Demand Analysis Agent` | `Display Report Form` | |
| `Display Report Form` | `n8n-nodes-base.form` | Displays final report to user | `Generate Research Report` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in an n8n instance:

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node named `When Form Submitted`.
   - Set the form title to `YouTube Audience Intelligence` and the button label to `Analyze audience`.
   - Add three fields:
     1. Text field labeled `YouTube video URL` (Placeholder: `https://www.youtube.com/watch?v=...`, Required: true).
     2. Number field labeled `Comments to analyze` (Default: `100`, Required: true).
     3. Dropdown field labeled `Comment selection` with options `relevance` and `time` (Default: `relevance`, Required: true).

2. **Add URL Normalization:**
   - Add a **Code** node named `Prepare Video Data`.
   - Connect `When Form Submitted` to `Prepare Video Data`.
   - Paste the normalization JavaScript code to extract the 11-character video ID, clamp comment counts between 25 and 100, and define minimum character filters.

3. **Fetch Video Metadata:**
   - Add an **HTTP Request** node named `Fetch Video Details`.
   - Connect `Prepare Video Data` to `Fetch Video Details`.
   - Set method to `GET`, URL to `https://www.googleapis.com/youtube/v3/videos`.
   - Configure Authentication to `Generic Credential Type` -> `HTTP Query Auth` (Credential Name: `key`, Value: your YouTube Data API v3 key).
   - Add query parameters: `part` = `snippet,statistics`, `id` = `={{ $('Prepare Video Data').first().json.videoId }}`.

4. **Fetch Comment Threads:**
   - Add an **HTTP Request** node named `Fetch Comment Threads`.
   - Connect `Fetch Video Details` to `Fetch Comment Threads`.
   - Set method to `GET`, URL to `https://www.googleapis.com/youtube/v3/commentThreads`.
   - Reuse the same YouTube API Query Auth credential.
   - Add query parameters: `part` = `snippet,replies`, `videoId` = `={{ $('Prepare Video Data').first().json.videoId }}`r, `maxResults` = `={{ $('Prepare Video Data').first().json.maxComments }}`, `order` = `={{ $('Prepare Video Data').first().json.order }}`, `textFormat` = `plainText`.

5. **Clean and Structure Comments:**
   - Add a **Code** node named `Prepare Comments Data`.
   - Connect `Fetch Comment Threads` to `Prepare Comments Data`.
   - Insert the data-cleansing JavaScript logic that strips HTML, filters spam/junk, calculates engagement scores, assigns `C001`-style evidence IDs, and outputs a structured JSON dataset.

6. **Configure the AI Model and Agent:**
   - Add a **Google Gemini Chat Model** node named `Google Gemini Model`. Configure the model name to `models/gemini-2.5-flash` and attach your Google Gemini API credentials.
   - Add an **AI Agent** node named `Content Demand Analysis Agent`.
   - Connect `Prepare Comments Data` to the primary input of the AI Agent.
   - Connect `Google Gemini Model` to the `AI Language Model` input of the AI Agent.
   - Set Prompt Type to `Define`. Provide the evidence-first system prompt instructing the agent to extract signals, content opportunities, and product ideas citing supplied `evidence_id` references.
   - Set the agent text parameter to `={{JSON.stringify($('Prepare Comments Data').first().json)}}`.

7. **Generate the Report:**
   - Add a **Code** node named `Generate Research Report`.
   - Connect `Content Demand Analysis Agent` to `Generate Research Report`.
   - Insert the report-generation JavaScript code that parses the AI JSON, validates evidence references, calculates mention statistics, and builds the responsive HTML layout.

8. **Display the Output Form:**
   - Add a **Form** node named `Display Report Form`.
   - Connect `Generate Research Report` to `Display Report Form`.
   - Set Operation to `Completion`, Respond With `Show Text`, and Response Text to `={{ $('Generate Research Report').item.json.reportHtml }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official YouTube Data API v3 Documentation | [Google Cloud Console / YouTube API Reference](https://developers.google.com/youtube/v3) |
| n8n LangChain & AI Agent Integration Guide | [n8n Documentation on AI Nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agent/) |