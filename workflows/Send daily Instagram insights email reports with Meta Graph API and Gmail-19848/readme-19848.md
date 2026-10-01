Send daily Instagram insights email reports with Meta Graph API and Gmail

https://n8nworkflows.xyz/workflows/send-daily-instagram-insights-email-reports-with-meta-graph-api-and-gmail-19848


# Send daily Instagram insights email reports with Meta Graph API and Gmail

### 1. Workflow Overview

The **Send daily Instagram insights email reports with Meta Graph API and Gmail** workflow automates the retrieval, analysis, and reporting of Instagram Business account performance. Designed to execute daily, it fetches profile metrics and recent media posts via the Meta Graph API, evaluates engagement within a localized 24-hour window, compiles a mobile-friendly HTML summary report, and emails it via Gmail.

The workflow logic is grouped into five functional blocks:
- **1.1 Trigger & Initialization:** Establishes execution timing, manual configuration parameters, and date parsing.
- **1.2 Data Retrieval:** Connects to the Meta Graph API to fetch profile metadata and up to 100 recent media items.
- **1.3 Filtering & Splitting:** Converts the raw media array into individual items, normalizes timestamps based on a local UTC offset, and filters posts matching the target date.
- **1.4 Per-Post Insights & Aggregation:** Iterates over each post to fetch detailed analytics (likes, comments, shares, saves, reach, views), merges the results, and calculates overall totals and top performers.
- **1.5 Report Generation & Delivery:** Transforms aggregated data into an optimized HTML email layout and dispatches it through Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization
- **Overview:** Initiates the workflow on a daily schedule, establishes API credentials, and automatically computes the target reporting date.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `📝 Instagram Page Credentials`
  - `🗓️ Extract Date From Trigger`
  - `⚙️ Config: Page & Date`

- **Node Details:**
  - **Schedule Trigger**
    - *Type:* `n8n-nodes-base.scheduleTrigger` (v1.3)
    - *Technical Role:* Sets the automated execution interval to daily at 23:59.
    - *Configuration:* Interval configured for hour 23, minute 59.
    - *Connections:* Output connects to `📝 Instagram Page Credentials`.
    - *Edge Cases:* Missed executions due to n8n downtime will not backfill unless manually retriggered.
  - **📝 Instagram Page Credentials**
    - *Type:* `n8n-nodes-base.set` (v3.4)
    - *Technical Role:* Defines initial static parameters required for the Meta Graph API connection.
    - *Configuration:* Assigns `accessToken`, `pageId`, `apiVersion`, and `timezoneOffsetHours`.
    - *Connections:* Input from `Schedule Trigger`; output to `🗓️ Extract Date From Trigger`.
    - *Edge Cases:* Invalid access tokens or incorrect numeric page IDs will cause downstream HTTP requests to fail.
  - **🗓️ Extract Date From Trigger**
    - *Type:* `n8n-nodes-base.set` (v3.4)
    - *Technical Role:* Extracts calendar components from the execution date.
    - *Configuration:* Parses `Year`, `Month` (converting names to numbers), and `Day of month`.
    - *Connections:* Input from `📝 Instagram Page Credentials`; output to `⚙️ Config: Page & Date`.
  - **⚙️ Config: Page & Date**
    - *Type:* `n8n-nodes-base.set` (v3.4)
    - *Technical Role:* Consolidates authentication parameters and target date fields into a unified configuration payload.
    - *Configuration:* Maps input fields for access tokens, page IDs, target year/month/day, API versions, and timezone offsets.
    - *Connections:* Input from `🗓️ Extract Date From Trigger`; output to `Get Instagram Profile`.

---

#### 1.2 Data Retrieval
- **Overview:** Queries the Meta Graph API to retrieve account metadata and recent media items.
- **Nodes Involved:**
  - `Get Instagram Profile`
  - `Get Instagram Media`

- **Node Details:**
  - **Get Instagram Profile**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* Fetches core account details and follower counts.
    - *Configuration:* Sends a GET request to `https://graph.facebook.com/{{ $json.apiVersion }}/{{ $json.pageId }}` with query parameters for fields (`id,username,name,profile_picture_url,biography,followers_count`) and `access_token`. Timeout: 15,000ms. Error handling set to `continueRegularOutput` with 2 retries.
    - *Connections:* Input from `⚙️ Config: Page & Date`; output to `Get Instagram Media`.
    - *Edge Cases:* API rate limits or expired tokens return error objects handled gracefully by downstream nodes.
  - **Get Instagram Media**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* Retrieves a list of recent media posts from the Instagram account.
    - *Configuration:* Sends a GET request to the `/media` endpoint. Requests up to 100 items with fields (`id,caption,media_type,media_product_type,timestamp,permalink,thumbnail_url,media_url,like_count,comments_count,username`). Timeout: 20,000ms. Error handling set to `continueRegularOutput`.
    - *Connections:* Input from `Get Instagram Profile`; output to `📋 Filter & Split Posts`.

---

#### 1.3 Filtering & Splitting
- **Overview:** Normalizes raw post timestamps against the configured local timezone offset, isolates posts published within the target day, and splits them into individual execution items.
- **Nodes Involved:**
  - `📋 Filter & Split Posts`
  - `❓ Has Posts`

- **Node Details:**
  - **📋 Filter & Split Posts**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* Filters raw media arrays by local date ranges and transforms arrays into individual items.
    - *Configuration:* Executes custom JavaScript using Luxon (`DateTime`) to define local start/end boundaries. If zero posts match, it outputs a single sentinel item with `_noPosts: true`.
    - *Expressions:* Uses upstream configuration and response nodes to establish bounds.
    - *Connections:* Input from `Get Instagram Media`; output to `❓ Has Posts`.
    - *Edge Cases:* Incorrect timezone offsets may shift the 24-hour evaluation window, causing posts to be omitted or attributed to adjacent days.
  - **❓ Has Posts**
    - *Type:* `n8n-nodes-base.if` (v2.3)
    - *Technical Role:* Branches execution depending on whether posts were published during the target period.
    - *Configuration:* Evaluates condition where `{{ $json._noPosts }}` is `false`.
    - *Connections:* 
      - True branch output connects to `🌐 Get Post Insights`.
      - False branch output connects to input index 1 of `🔀 Combine Posts`.

---

#### 1.4 Per-Post Insights & Aggregation
- **Overview:** Fetches detailed analytics for each matching post, merges individual metrics, and aggregates them into statistical summaries and leaderboards.
- **Nodes Involved:**
  - `🌐 Get Post Insights`
  - `🔗 Merge Insights`
  - `🔀 Combine Posts`
  - `📊 Aggregate Instagram Report Data`

- **Node Details:**
  - **🌐 Get Post Insights**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* Requests media-specific metrics from the Meta Graph API for each post item.
    - *Configuration:* Sends a GET request to `.../{{ $json.id }}/insights`. Dynamically sets metric parameters depending on post type (`video` requests views, reach, likes, comments, shares, saved, total interactions; `photo` requests reach, likes, comments, saved, total interactions). Timeout: 15,000ms.
    - *Connections:* Input from `❓ Has Posts` (true branch); output to `🔗 Merge Insights`.
  - **🔗 Merge Insights**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* Parses API insight responses and maps them onto individual post objects.
    - *Configuration:* Executed in `runOnceForEachItem` mode. Extracts metric values and normalizes fallback totals.
    - *Connections:* Input from `🌐 Get Post Insights`; output to input index 0 of `🔀 Combine Posts`.
  - **🔀 Combine Posts**
    - *Type:* `n8n-nodes-base.merge` (v3.2)
    - *Technical Role:* Re-combines processed individual post items or zero-post sentinel items into a unified collection.
    - *Configuration:* Default merge settings.
    - *Connections:* 
      - Input 0 receives data from `🔗 Merge Insights`.
      - Input 1 receives data from `❓ Has Posts` (false branch).
      - Output connects to `📊 Aggregate Instagram Report Data`.
  - **📊 Aggregate Instagram Report Data**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* Computes account-level summary metrics, breakdown totals for photo and video formats, and top-performing post rankings.
    - *Configuration:* Executes custom JavaScript processing array items into aggregated summary objects (`summary`, `photoBreakdown`, `videoBreakdown`, `topPosts`).
    - *Connections:* Input from `🔀 Combine Posts`; output to `🎨 Build HTML Email Report`.

---

#### 1.5 Report Generation & Delivery
- **Overview:** Formats aggregated data into a responsive HTML email document and sends it to the designated recipient via Gmail.
- **Nodes Involved:**
  - `🎨 Build HTML Email Report`
  - `📧 Send Report Email`

- **Node Details:**
  - **🎨 Build HTML Email Report**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* Generates a self-contained, mobile-optimized HTML email layout containing summary tiles, top performer cards, and categorized post feeds.
    - *Configuration:* Executes JavaScript containing styling, formatting helper functions, and template strings. Requires editing the hardcoded `recipientEmail` constant inside the script.
    - *Connections:* Input from `📊 Aggregate Instagram Report Data`; output to `📧 Send Report Email`.
    - *Edge Cases:* Unhandled exceptions in template interpolation could fail the node; strict HTML formatting is used to maintain rendering compatibility across mobile email clients.
  - **📧 Send Report Email**
    - *Type:* `n8n-nodes-base.base.gmail` (v2.2)
    - *Technical Role:* Transmits the compiled report via Gmail API.
    - *Configuration:* 
      - Send To: `={{ $json.recipientEmail }}`
      - Subject: `={{ $json.emailSubject }}`
      - Message: `={{ $json.emailHtml }}`
      - Options: `appendAttribution` disabled.
    - *Credentials:* Requires a connected `gmailOAuth2` credential.
    - *Connections:* Input from `🎨 Build HTML Email Report`.
    - *Edge Cases:* OAuth2 token revocation or missing scopes will cause authentication errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Schedule Trigger** | `scheduleTrigger` | Triggers workflow daily at 23:59 | None | 📝 Instagram Page Credentials | 🚀 Instagram Daily Report — Single Page Setup Guide<br>📅 Schedule |
| **📝 Instagram Page Credentials** | `set` | Defines API access token, page ID, API version, and timezone | Schedule Trigger | 🗓️ Extract Date From Trigger | 🚀 Instagram Daily Report — Single Page Setup Guide<br>📝 STEP 1 — Instagram Credentials |
| **🗓️ Extract Date From Trigger** | `set` | Auto-extracts target year, month, and day from trigger | 📝 Instagram Page Credentials | ⚙️ Config: Page & Date | 🚀 Instagram Daily Report — Single Page Setup Guide<br>🗓️ STEP 2 — Date Auto-Extracted |
| **⚙️ Config: Page & Date** | `set` | Combines credentials and date parameters into config payload | 🗓️ Extract Date From Trigger | Get Instagram Profile | 🚀 Instagram Daily Report — Single Page Setup Guide |
| **Get Instagram Profile** | `httpRequest` | Fetches account profile metadata and follower counts | ⚙️ Config: Page & Date | Get Instagram Media | |
| **Get Instagram Media** | `httpRequest` | Retrieves recent posts list from Meta Graph API | Get Instagram Profile | 📋 Filter & Split Posts | |
| **📋 Filter & Split Posts** | `code` | Filters posts by local date window and splits into items | Get Instagram Media | ❓ Has Posts | |
| **❓ Has Posts** | `if` | Branches execution depending on post existence | 📋 Filter & Split Posts | 🌐 Get Post Insights, 🔀 Combine Posts | |
| **🌐 Get Post Insights** | `httpRequest` | Requests detailed per-post analytics from Meta Graph API | ❓ Has Posts | 🔗 Merge Insights | |
| **🔗 Merge Insights** | `code` | Parses and merges insight metrics into post objects | 🌐 Get Post Insights | 🔀 Combine Posts | |
| **🔀 Combine Posts** | `merge` | Re-combines post items or zero-post sentinel items | 🔗 Merge Insights, ❓ Has Posts | 📊 Aggregate Instagram Report Data | |
| **📊 Aggregate Instagram Report Data** | `code` | Aggregates summary statistics, breakdowns, and leaderboards | 🔀 Combine Posts | 🎨 Build HTML Email Report | |
| **🎨 Build HTML Email Report** | `code` | Compiles dynamic data into responsive HTML email format | 📊 Aggregate Instagram Report Data | 📧 Send Report Email | 🚀 Instagram Daily Report — Single Page Setup Guide<br>📧 STEP 3 — Email Recipient |
| **📧 Send Report Email** | `gmail` | Sends formatted report email via Gmail | 🎨 Build HTML Email Report | None | 🚀 Instagram Daily Report — Single Page Setup Guide |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: `Schedule Trigger`
   - Configuration: Set interval trigger to run at Hour: `23`, Minute: `59`.
2. **Create Instagram Page Credentials Node**
   - Type: `Edit Fields (Set)`
   - Configuration: Add string assignments for `accessToken`, `pageId`, `apiVersion` (e.g., `v19.0`), and number assignment for `timezoneOffsetHours` (e.g., `6`).
3. **Create Extract Date From Trigger Node**
   - Type: `Edit Fields (Set)`
   - Configuration: Map `targetYear` to `={{ $('Schedule Trigger').item.json.Year }}`, `targetMonth` to `={{ DateTime.fromFormat($('Schedule Trigger').item.json.Month, 'MMMM').month }}`, and `targetDay` to `={{ $('Schedule Trigger').item.json['Day of month'] }}` with "Include Other Fields" enabled. Connect input from previous node.
4. **Create Config: Page & Date Node**
   - Type: `Edit Fields (Set)`
   - Configuration: Consolidate configuration parameters referencing upstream item properties. Connect input from Extract Date node.
5. **Create Get Instagram Profile Node**
   - Type: `HTTP Request`
   - Configuration: Method: `GET`, URL: `={{ 'https://graph.facebook.com/' + $json.apiVersion + '/' + $json.pageId }}`, Query Parameters: `fields` (`id,username,name,profile_picture_url,biography,followers_count`), `access_token` (`={{ $json.accessToken }}`). Set error handling to continue regular output with retries. Connect input from Config node.
6. **Create Get Instagram Media Node**
   - Type: `HTTP Request`
   - Configuration: Method: `GET`, URL: `={{ 'https://graph.facebook.com/' + $('⚙️ Config: Page & Date').item.json.apiVersion + '/' + $('⚙️ Config: Page & Date').item.json.pageId + '/media' }}`, Query Parameters: `fields` (`id,caption,media_type,media_product_type,timestamp,permalink,thumbnail_url,media_url,like_count,comments_count,username`), `limit` (`100`), `access_token` (`={{ $('⚙️ Config: Page & Date').item.json.accessToken }}`). Connect input from Get Instagram Profile.
7. **Create Filter & Split Posts Node**
   - Type: `Code`
   - Configuration: Paste JavaScript implementation to evaluate timestamps against local timezone bounds and return individual post items or a `_noPosts` sentinel. Connect input from Get Instagram Media.
8. **Create Has Posts Node**
   - Type: `If`
   - Configuration: Set condition to check boolean value where `{{ $json._noPosts }}` equals `false`. Connect input from Filter & Split Posts.
9. **Create Get Post Insights Node**
   - Type: `HTTP Request`
   - Configuration: Method: `GET`, URL: `={{ 'https://graph.facebook.com/' + $json.apiVersion + '/' + $json.id + '/insights' }}`, Query Parameters: `metric` conditional expression based on `_type`, `access_token` (`={{ $json.accessToken }}`). Connect true output of Has Posts node.
10. **Create Merge Insights Node**
    - Type: `Code`
    - Configuration: Run mode set to `runOnceForEachItem`. Parses API metrics and injects them into the original post payload. Connect input from Get Post Insights.
11. **Create Combine Posts Node**
    - Type: `Merge`
    - Configuration: Default settings. Connect input 0 to Merge Insights and input 1 to the false output of the Has Posts node.
12. **Create Aggregate Instagram Report Data Node**
    - Type: `Code`
    - Configuration: JavaScript snippet aggregating total posts, reactions, category breakdowns, and top posts. Connect input from Combine Posts.
13. **Create Build HTML Email Report Node**
    - Type: `Code`
    - Configuration: JavaScript snippet returning `emailSubject`, `emailHtml`, and `recipientEmail`. **Action Required:** Update the `recipientEmail` constant string variable to the target destination email address. Connect input from Aggregate node.
14. **Create Send Report Email Node**
    - Type: `Gmail`
    - Configuration: Send To: `={{ $json.recipientEmail }}`, Subject: `={{ $json.emailSubject }}`, Message: `={{ $json.emailHtml }}`. Connect required `gmailOAuth2` credentials. Connect input from Build HTML Email Report node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Output Preview Screenshot** | [Visual Reference Link](https://raw.githubusercontent.com/mashunterbd/n8n-screenshot/4a2ba0161ae1da86dfed19224f2f25c9f947ee7a/ig-daily-report.png) |