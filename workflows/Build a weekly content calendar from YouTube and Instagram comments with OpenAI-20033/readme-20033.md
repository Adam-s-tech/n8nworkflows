Build a weekly content calendar from YouTube and Instagram comments with OpenAI

https://n8nworkflows.xyz/workflows/build-a-weekly-content-calendar-from-youtube-and-instagram-comments-with-openai-20033


# Build a weekly content calendar from YouTube and Instagram comments with OpenAI

### 1. Workflow Overview

This workflow automates weekly audience research and content planning by gathering recent comments from both YouTube and Instagram, extracting genuine questions and feature requests, grouping them into thematic clusters using OpenAI, and writing the results into Google Sheets while broadcasting a summary digest to Slack.

The workflow logic is categorized into the following functional blocks:
- **1.1 Initialization & Configuration:** Triggers automatically on a weekly schedule and supplies global environment variables (channel IDs, lookback windows, posting volumes, and target audience definitions).
- **1.2 YouTube Data Extraction:** Fetches recent uploads from a YouTube channel, isolates videos published within the lookback window, retrieves top-level comment threads, and normalizes them into a unified schema.
- **1.3 Instagram Data Extraction:** Fetches recent Instagram media posts, filters for posts with comments within the lookback window, pulls comments per post, and normalizes them into the same unified schema.
- **1.4 Comment Processing & AI Clustering:** Merges comments from both platforms, strips out spam, emojis, and duplicates, extracts high-intent questions/requests, and sends them to OpenAI to group recurring needs into weighted topics.
- **1.5 Data Persistence & Content Generation:** Scores the clusters based on engagement metrics, appends structured rows to a “Question Bank” tab in Google Sheets, generates a multi-platform content calendar for the upcoming week using OpenAI, records the calendar entries in a second spreadsheet tab, and notifies the team via Slack.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization & Configuration
- **Overview:** Initializes the weekly execution schedule and sets up a centralized configuration object containing target audience parameters, platform limits, and API version details.
- **Nodes Involved:** `Weekly Trigger`, `Config`
- **Node Details:**
  - **Weekly Trigger**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` — Initiates the workflow run.
    - *Configuration:* Set to trigger weekly at hour 9.
    - *Inputs/Outputs:* Input: None | Output: `Config`
    - *Edge Cases:* Ensure server timezones match expectations for Sunday runs.
  - **Config**
    - *Type and Role:* `n8n-nodes-base.set` — Holds metadata and scan settings (e.g., creator name, niche, audience profile, YouTube channel ID, Instagram user ID, lookback days, posting frequencies).
    - *Configuration:* Assignments mode supplying explicit string and number values.
    - *Inputs/Outputs:* Input: `Weekly Trigger` | Output: `YouTube: Get Recent Uploads`, `Instagram: Get Recent Media`
    - *Edge Cases:* Invalid ID strings will cause subsequent API requests to fail.

---

#### Block 1.2: YouTube Data Extraction
- **Overview:** Pulls recent videos from the configured YouTube channel's uploads playlist, filters out older entries, retrieves comment threads for valid videos, and flattens them into standard comment objects.
- **Nodes Involved:** `YouTube: Get Recent Uploads`, `Split YouTube Videos`, `YouTube: Get Comments`, `Flatten YouTube Comments`
- **Node Details:**
  - **YouTube: Get Recent Uploads**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Queries the YouTube Data API v3 playlistItems endpoint.
    - *Configuration:* Uses Generic Query Auth (`key`), dynamically formats the playlist ID using `UU` plus the channel ID suffix. Includes error continuation (`continueRegularOutput`).
    - *Inputs/Outputs:* Input: `Config` | Output: `Split YouTube Videos`
    - *Edge Cases:* API rate limits, revoked API keys, or invalid channel IDs. Videos with disabled comments bypass execution gracefully via error handling.
  - **Split YouTube Videos**
    - *Type and Role:* `n8n-nodes-base.code` — Iterates through API payloads, extracts individual video IDs, titles, and publication dates, and filters by the configured lookback window.
    - *Configuration:* JavaScript execution using `Date.now()` and configuration offsets.
    - *Inputs/Outputs:* Input: `YouTube: Get Recent Uploads` | Output: `YouTube: Get Comments`
  - **YouTube: Get Comments**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Fetches top-level comment threads per filtered video.
    - *Configuration:* Queries YouTube Data API `commentThreads` with `order=relevance` and `textFormat=plainText`.
    - *Inputs/Outputs:* Input: `Split YouTube Videos` | Output: `Flatten YouTube Comments`
  - **Flatten YouTube Comments**
    - *Type and Role:* `n8n-nodes-base.code` — Flattens nested YouTube comment structures into standard fields (`platform`, `comment_id`, `text`, `likes`, `replies`, `author`, `posted_at`, `source_title`, `source_url`).
    - *Configuration:* JavaScript mapping with `alwaysOutputData` enabled.
    - *Inputs/Outputs:* Input: `YouTube: Get Comments` | Output: `Merge Comments`

---

#### Block 1.3: Instagram Data Extraction
- **Overview:** Fetches recent media posts from the Instagram Graph API, filters for recent content containing comments, pulls those comments, and normalizes them to match the common comment schema.
- **Nodes Involved:** `Instagram: Get Recent Media`, `Split Instagram Posts`, `Instagram: Get Comments`, `Flatten Instagram Comments`
- **Node Details:**
  - **Instagram: Get Recent Media**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Queries the Meta Graph API for user media objects.
    - *Configuration:* Uses Generic Query Auth (`access_token`), dynamic URL construction using graph API version and user ID.
    - *Inputs/Outputs:* Input: `Config` | Output: `Split Instagram Posts`
    - *Edge Cases:* Token expiration or missing Instagram Business account permissions.
  - **Split Instagram Posts**
    - *Type and Role:* `n8n-nodes-base.code` — Filters media items to ensure they fall within the lookback window and possess non-zero comment counts.
    - *Configuration:* JavaScript code evaluating `comments_count` and `timestamp`.
    - *Inputs/Outputs:* Input: `Instagram: Get Recent Media` | Output: `Instagram: Get Comments`
  - **Instagram: Get Comments**
    - *Type and Role:* `n8n-nodes-base.httpRequest` — Fetches individual comments for each validated media ID.
    - *Configuration:* Meta Graph API `/{media-id}/comments` endpoint with field mappings for ID, text, like count, timestamp, and username.
    - *Inputs/Outputs:* Input: `Split Instagram Posts` | Output: `Flatten Instagram Comments`
  - **Flatten Instagram Comments**
    - *Type and Role:* `n8n-nodes-base.code` — Maps Meta Graph API comment responses to the standard flat comment format.
    - *Configuration:* JavaScript mapping aligned by index with split media items, with `alwaysOutputData` enabled.
    - *Inputs/Outputs:* Input: `Instagram: Get Comments` | Output: `Merge Comments`

---

#### Block 1.4: Comment Processing & AI Clustering
- **Overview:** Combines multi-platform comment streams, filters out noise, spam, and duplicates, ranks them by engagement, and uses OpenAI to group audience needs into structured topic clusters.
- **Nodes Involved:** `Merge Comments`, `Clean & Filter Comments`, `AI: Cluster Questions`, `Score Clusters`
- **Node Details:**
  - **Merge Comments**
    - *Type and Role:* `n8n-nodes-base.merge` — Combines the flattened YouTube and Instagram comment streams into a single dataset.
    - *Inputs/Outputs:* Inputs: `Flatten YouTube Comments`, `Flatten Instagram Comments` | Output: `Clean & Filter Comments`
  - **Clean & Filter Comments**
    - *Type and Role:* `n8n-nodes-base.code` — Applies regex heuristics to filter out spam, promotional URLs, emoji-only inputs, and short or duplicate remarks while isolating high-intent questions and requests.
    - *Configuration:* Sorts comments by engagement score (`likes + replies * 2`) and builds an indexed prompt block.
    - *Inputs/Outputs:* Input: `Merge Comments` | Output: `AI: Cluster Questions`
  - **AI: Cluster Questions**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` — Analyzes the curated prompt block to extract and group shared user pain points.
    - *Configuration:* Uses `gpt-4o-mini`, temperature `0.2`, max tokens `4000`, with structured JSON output enforcement.
    - *Inputs/Outputs:* Input: `Clean & Filter Comments` | Output: `Score Clusters`
    - *Edge Cases:* JSON parsing failures if the model outputs malformed strings.
  - **Score Clusters**
    - *Type and Role:* `n8n-nodes-base.code` — Parses AI responses, links cluster items back to raw source metadata, calculates quantitative demand scores, and computes the upcoming Monday’s date.
    - *Configuration:* Custom calculation: `demand_score = mention_count * 3 + total_likes * 0.5 + (cross_platform ? 5 : 0)`.
    - *Inputs/Outputs:* Input: `AI: Cluster Questions` | Output: `Clusters to Rows`, `AI: Build Content Calendar`

---

#### Block 1.5: Data Persistence & Content Generation
- **Overview:** Converts processed question clusters into spreadsheet-ready rows, appends them to Google Sheets, instructs OpenAI to generate a weekly content calendar, logs the calendar entries, and dispatches a notification digest to Slack.
- **Nodes Involved:** `Clusters to Rows`, `Sheets: Question Bank`, `AI: Build Content Calendar`, `Parse Calendar`, `Sheets: Content Calendar`, `Build Weekly Digest`, `Slack: Send Digest`
- **Node Details:**
  - **Clusters to Rows**
    - *Type and Role:* `n8n-nodes-base.code` — Transforms ranked cluster objects into flat row arrays mapped to table headers.
    - *Inputs/Outputs:* Input: `Score Clusters` | Output: `Sheets: Question Bank`
  - **Sheets: Question Bank**
    - *Type and Role:* `n8n-nodes-base.googleSheets` — Appends question bank records to Google Sheets.
    - *Configuration:* Operation: `append`, target sheet name: `Question Bank`.
    - *Inputs/Outputs:* Input: `Clusters to Rows` | Output: None
    - *Edge Cases:* Auth token expiration or missing target spreadsheet columns.
  - **AI: Build Content Calendar**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` — Generates a multi-day publishing schedule based on top question clusters and audience constraints.
    - *Configuration:* Uses `gpt-4o-mini`, temperature `0.6`, max tokens `4000`, with JSON output enforcement.
    - *Inputs/Outputs:* Input: `Score Clusters` | Output: `Parse Calendar`
  - **Parse Calendar**
    - *Type and Role:* `n8n-nodes-base.code` — Parses the AI-generated calendar JSON payload and normalizes entries for individual scheduling rows.
    - *Inputs/Outputs:* Input: `AI: Build Content Calendar` | Output: `Sheets: Content Calendar`
  - **Sheets: Content Calendar**
    - *Type and Role:* `n8n-nodes-base.googleSheets` — Appends generated calendar schedule entries to Google Sheets.
    - *Configuration:* Operation: `append`, target sheet name: `Content Calendar`.
    - *Inputs/Outputs:* Input: `Parse Calendar` | Output: `Build Weekly Digest`
  - **Build Weekly Digest**
    - *Type and Role:* `n8n-nodes-base.code` — Summarizes calendar items, stats, and top-ranking audience questions into a single Slack markdown message block.
    - *Configuration:* Executed once per batch (`executeOnce: true`).
    - *Inputs/Outputs:* Input: `Sheets: Content Calendar` | Output: `Slack: Send Digest`
  - **Slack: Send Digest**
    - *Type and Role:* `n8n-nodes-base.slack` — Posts the formatted content strategy digest message to a designated Slack channel.
    - *Configuration:* Target selection by channel name (`#content-planning`).
    - *Inputs/Outputs:* Input: `Build Weekly Digest` | Output: None
    - *Edge Cases:* Missing channel access or invalid bot scopes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Trigger | n8n-nodes-base.scheduleTrigger | Initiates the weekly workflow execution. | None | Config | 💬 Comment-to-Content Miner / 1️⃣ Schedule & Config |
| Config | n8n-nodes-base.set | Defines global configuration variables and settings. | Weekly Trigger | YouTube: Get Recent Uploads, Instagram: Get Recent Media | 💬 Comment-to-Content Miner / 1️⃣ Schedule & Config |
| YouTube: Get Recent Uploads | n8n-nodes-base.httpRequest | Fetches recent uploads from the YouTube channel uploads playlist. | Config | Split YouTube Videos | 💬 Comment-to-Content Miner / 2️⃣ YouTube Comments / 🔑 Credentials Required |
| Split YouTube Videos | n8n-nodes-base.code | Isolates individual video records within the lookback window. | YouTube: Get Recent Uploads | YouTube: Get Comments | 💬 Comment-to-Content Miner / 2️⃣ YouTube Comments |
| YouTube: Get Comments | n8n-nodes-base.httpRequest | Retrieves relevance-sorted comment threads per video. | Split YouTube Videos | Flatten YouTube Comments | 💬 Comment-to-Content Miner / 2️⃣ YouTube Comments |
| Flatten YouTube Comments | n8n-nodes-base.code | Normalizes YouTube comment structures into standard format. | YouTube: Get Comments | Merge Comments | 💬 Comment-to-Content Miner / 2️⃣ YouTube Comments |
| Instagram: Get Recent Media | n8n-nodes-base.httpRequest | Fetches recent media items from the Instagram user account. | Config | Split Instagram Posts | 💬 Comment-to-Content Miner / 3️⃣ Instagram Comments / 🔑 Credentials Required |
| Split Instagram Posts | n8n-nodes-base.code | Filters recent posts that contain comments within lookback criteria. | Instagram: Get Recent Media | Instagram: Get Comments | 💬 Comment-to-Content Miner / 3️⃣ Instagram Comments |
| Instagram: Get Comments | n8n-nodes-base.httpRequest | Retrieves comment lists for specific Instagram media IDs. | Split Instagram Posts | Flatten Instagram Comments | 💬 Comment-to-Content Miner / 3️⃣ Instagram Comments |
| Flatten Instagram Comments | n8n-nodes-base.code | Normalizes Instagram comment structures into standard format. | Instagram: Get Comments | Merge Comments | 💬 Comment-to-Content Miner / 3️⃣ Instagram Comments |
| Merge Comments | n8n-nodes-base.merge | Merges YouTube and Instagram comment streams. | Flatten YouTube Comments, Flatten Instagram Comments | Clean & Filter Comments | 💬 Comment-to-Content Miner |
| Clean & Filter Comments | n8n-nodes-base.code | Removes spam, duplicates, and formats high-intent questions. | Merge Comments | AI: Cluster Questions | 💬 Comment-to-Content Miner / 4️⃣ Clean, Cluster & Score |
| AI: Cluster Questions | @n8n/n8n-nodes-langchain.openAi | Groups recurring audience questions and requests into themes. | Clean & Filter Comments | Score Clusters | 💬 Comment-to-Content Miner / 4️⃣ Clean, Cluster & Score / 🔑 Credentials Required |
| Score Clusters | n8n-nodes-base.code | Calculates demand metrics and ranks question clusters. | AI: Cluster Questions | Clusters to Rows, AI: Build Content Calendar | 💬 Comment-to-Content Miner / 4️⃣ Clean, Cluster & Score |
| Clusters to Rows | n8n-nodes-base.code | Transforms ranked clusters into tabular spreadsheet rows. | Score Clusters | Sheets: Question Bank | 💬 Comment-to-Content Miner |
| Sheets: Question Bank | n8n-nodes-base.googleSheets | Appends question bank records to Google Sheets. | Clusters to Rows | None | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest / 🔑 Credentials Required |
| AI: Build Content Calendar | @n8n/n8n-nodes-langchain.openAi | Generates an upcoming content calendar from clustered topics. | Score Clusters | Parse Calendar | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest / 🔑 Credentials Required |
| Parse Calendar | n8n-nodes-base.code | Parses AI content calendar payloads into structured records. | AI: Build Content Calendar | Sheets: Content Calendar | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest |
| Sheets: Content Calendar | n8n-nodes-base.googleSheets | Appends content calendar schedule entries to Google Sheets. | Parse Calendar | Build Weekly Digest | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest / 🔑 Credentials Required |
| Build Weekly Digest | n8n-nodes-base.code | Compiles calendar entries and summaries into a Slack digest. | Sheets: Content Calendar | Slack: Send Digest | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest |
| Slack: Send Digest | n8n-nodes-base.slack | Posts the weekly summary report to a Slack channel. | Build Weekly Digest | None | 💬 Comment-to-Content Miner / 5️⃣ Content Calendar & Digest / 🔑 Credentials Required |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger & Config Nodes:**
   - Add a **Schedule Trigger** node set to trigger weekly at hour 9.
   - Add a **Set** node named `Config`. Assign string values for `creator_name`, `niche`, `audience`, `youtube_channel_id`, `instagram_user_id`, `platform_mix`, `graph_api_version`, and numeric values for `youtube_videos_to_scan`, `youtube_comments_per_video`, `instagram_posts_to_scan`, `lookback_days`, and `posts_per_week`. Connect the Trigger to this node.

2. **Build the YouTube Pipeline:**
   - Create an **HTTP Request** node named `YouTube: Get Recent Uploads`. Set authentication to Generic Query Auth, configure the URL to `https://www.googleapis.com/youtube/v3/playlistItems`, pass query parameters (`part`: `snippet,contentDetails`, `playlistId`: `={{"UU" + $json.youtube_channel_id.slice(2)}}`, `maxResults`: `={{$json.youtube_videos_to_scan}}`), and enable *Continue on Fail*. Connect `Config` to this node.
   - Create a **Code** node named `Split YouTube Videos` using JavaScript to filter items by lookback days. Connect `YouTube: Get Recent Uploads` to it.
   - Create an **HTTP Request** node named `YouTube: Get Comments`. Set the URL to `https://www.googleapis.com/youtube/v3/commentThreads`, configure query parameters (`part`: `snippet`, `videoId`: `={{$json.video_id}}`, `maxResults`: `={{$('Config').first().json.youtube_comments_per_video}}`, `order`: `relevance`, `textFormat`: `plainText`), and enable *Continue on Fail*. Connect `Split YouTube Videos` to this node.
   - Create a **Code** node named `Flatten YouTube Comments` to normalize comment thread attributes into flat objects. Connect `YouTube: Get Comments` to it.

3. **Build the Instagram Pipeline:**
   - Create an **HTTP Request** node named `Instagram: Get Recent Media`. Set the URL to `={{"https://graph.facebook.com/" + $json.graph_api_version + "/" + $json.instagram_user_id + "/media"}}`, add Generic Query Auth parameters, and enable *Continue on Fail*. Connect `Config` to this node.
   - Create a **Code** node named `Split Instagram Posts` to filter media items with comments inside the lookback window. Connect `Instagram: Get Recent Media` to it.
   - Create an **HTTP Request** node named `Instagram: Get Comments`. Set the URL to `={{"https://graph.facebook.com/" + $('Config').first().json.graph_api_version + "/" + $json.media_id + "/comments"}}`, query fields (`id,text,like_count,timestamp,username`), and enable *Continue on Fail*. Connect `Split Instagram Posts` to it.
   - Create a **Code** node named `Flatten Instagram Comments` to map Instagram payloads to standard comment fields, ensuring `alwaysOutputData` is active. Connect `Instagram: Get Comments` to it.

4. **Merge and Clean Comment Streams:**
   - Create a **Merge** node named `Merge Comments`. Connect `Flatten YouTube Comments` to input index 0 and `Flatten Instagram Comments` to input index 1.
   - Create a **Code** node named `Clean & Filter Comments` to strip spam, remove duplicates, filter high-intent questions, and format the prompt block. Connect `Merge Comments` to it.

5. **Clustering & Scoring:**
   - Create an **OpenAI** node named `AI: Cluster Questions`. Set model to `gpt-4o-mini`, temperature to `0.2`, enable JSON output, and provide a system prompt asking the model to group questions into weighted themes. Connect `Clean & Filter Comments` to it.
   - Create a **Code** node named `Score Clusters` to calculate quantitative demand scores and establish the upcoming Monday's date. Connect `AI: Cluster Questions` to it.

6. **Spreadsheet Persistence & Content Calendar Generation:**
   - Create a **Code** node named `Clusters to Rows` to format cluster items into row objects. Connect `Score Clusters` to it.
   - Create a **Google Sheets** node named `Sheets: Question Bank`. Configure operation to `append`, specify your Document ID, and select the `Question Bank` sheet. Connect `Clusters to Rows` to it.
   - Create an **OpenAI** node named `AI: Build Content Calendar`. Set model to `gpt-4o-mini`, temperature to `0.6`, enable JSON output, and supply a prompt instructing it to build a weekly calendar based on top clusters and platform mix. Connect `Score Clusters` to it.
   - Create a **Code** node named `Parse Calendar` to format AI calendar entries into individual schema rows. Connect `AI: Build Content Calendar` to it.
   - Create a **Google Sheets** node named `Sheets: Content Calendar`. Configure operation to `append`, specify your Document ID, and select the `Content Calendar` sheet. Connect `Parse Calendar` to it.

7. **Slack Notification:**
   - Create a **Code** node named `Build Weekly Digest` to aggregate calendar items and top questions into a Slack-compatible markdown message (`executeOnce: true`). Connect `Sheets: Content Calendar` to it.
   - Create a **Slack** node named `Slack: Send Digest`. Configure message output to use the generated text expression and select target channel `#content-planning`. Connect `Build Weekly Digest` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Single-platform usage warning | If operating on only one platform (YouTube or Instagram), leave the alternate branch intact; the respective flatten node guarantees continuous execution through output data emission. |
| API Authorization Requirements | Requires valid credentials for YouTube Data API v3 (`key`), Meta Graph API (`access_token`), OpenAI API, Google Sheets OAuth2, and Slack OAuth2. |