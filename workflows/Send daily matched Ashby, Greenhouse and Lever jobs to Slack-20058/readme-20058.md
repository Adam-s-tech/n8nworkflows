Send daily matched Ashby, Greenhouse and Lever jobs to Slack

https://n8nworkflows.xyz/workflows/send-daily-matched-ashby--greenhouse-and-lever-jobs-to-slack-20058


# Send daily matched Ashby, Greenhouse and Lever jobs to Slack

### 1. Workflow Overview

This workflow automates the daily monitoring of job boards (Ashby, Greenhouse, and Lever) for specific target roles and locations. It runs every morning, fetches open positions through public company APIs, filters them using custom keyword and geography lists, deduplicates results across historical executions, and consolidates new findings into a single notification posted to Slack.

The logical flow is organized into four distinct functional blocks:

- **1.1 Input Reception & Configuration:** Initiates the workflow via a manual trigger or daily schedule, defining target companies and filtering criteria.
- **1.2 Data Acquisition:** Transforms configuration data into API requests and queries each job board endpoint.
- **1.3 Data Processing & Filtering:** Normalizes heterogeneous API payloads into a uniform schema, applies inclusion/exclusion rules, and filters out previously processed jobs.
- **1.4 Notification Dispatch:** Aggregates newly discovered roles into a formatted summary and sends the message to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Sets up the execution triggers (manual and scheduled) and stores the primary configuration data including target boards, job title keywords, and location constraints.
- **Nodes Involved:** `Run it once now`, `Every morning at 8`, `Boards and filters`
- **Node Details:**
  - **Run it once now**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` — Manual execution entry point.
    - *Configuration:* Default settings.
    - *Outputs:* Triggers `Boards and filters`.
    - *Edge Cases:* None.
  - **Every morning at 8**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` — Time-based trigger.
    - *Configuration:* Executes daily at hour 08:00 (`days` interval).
    - *Outputs:* Triggers `Boards and filters`.
    - *Edge Cases:* Relies on host system timezone.
  - **Boards and filters**
    - *Type & Technical Role:* `n8n-nodes-base.set` — Static data provider.
    - *Configuration:* Raw JSON configuration output containing the `boards` array, `title_any`, `title_none`, `location_any`, and `location_none` arrays.
    - *Outputs:* Feeds configuration into `Build board requests`.
    - *Edge Cases:* JSON syntax errors in configuration data will break downstream parsing.

#### 2.2 Data Acquisition
- **Overview:** Dynamically builds valid API requests for each configured Applicant Tracking System (ATS) and executes HTTP requests to retrieve public job board data.
- **Nodes Involved:** `Build board requests`, `Fetch the board`
- **Node Details:**
  - **Build board requests**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node.
    - *Configuration:* Maps ATS slugs to public API endpoint patterns (Ashby, Greenhouse, Lever, Recruitee). Filters out unsupported ATS entries.
    - *Key Expressions/Variables:* Reads `$input.first().json` to access configuration properties.
    - *Inputs / Outputs:* Input from `Boards and filters` / Output to `Fetch the board`.
    - *Edge Cases:* Unrecognized ATS keys are logged to console and skipped.
  - **Fetch the board**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` — Web request executor.
    - *Configuration:* Method GET, URL set dynamically via expression. Configured to return response as `text` to prevent n8n from splitting JSON array payloads across multiple items prematurely.
    - *Key Expressions/Variables:* `={{ $json.url }}`
    - *Error Handling:* `onError` set to `continueRegularOutput` to gracefully capture and skip failed board requests.
    - *Inputs / Outputs:* Input from `Build board requests` / Output to `Normalise and match`.
    - *Edge Cases:* HTTP 404/500 errors or network timeouts; handled by continuing regular output so downstream logic can process error states.

#### 2.3 Data Processing & Filtering
- **Overview:** Parses raw string responses from various ATS formats into uniform job records, applies strict inclusion/exclusion filters for titles and locations, and filters out listings identified in past executions.
- **Nodes Involved:** `Normalise and match`, `Only roles not seen before`
- **Node Details:**
  - **Normalise and match**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node.
    - *Configuration:* Custom parsing logic handling Ashby, Greenhouse, Lever, and Recruitee payload schemas. Implements word-boundary regex checks for short keywords (e.g., "ai", "eu") and substring matching for longer terms. Applies geographical and remote-work validation logic.
    - *Key Expressions/Variables:* References upstream nodes `$('Boards and filters')` and `$('Build board requests')`.
    - *Inputs / Outputs:* Input from `Fetch the board` / Output to `Only roles not seen before`.
    - *Edge Cases:* Malformed JSON bodies are caught and return empty arrays. Board failures indicated by error properties are safely skipped.
  - **Only roles not seen before**
    - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` — State deduplication node.
    - *Configuration:* Operation configured to remove items seen in previous executions using a custom deduplication value key.
    - *Key Expressions/Variables:* `={{ $json.id }}` (combines ATS, slug, and job reference ID).
    - *Inputs / Outputs:* Input from `Normalise and match` / Output to `Compose one message`.
    - *Edge Cases:* Requires state persistence enabled in n8n instance to maintain deduplication memory across distinct workflow runs.

#### 2.4 Notification Dispatch
- **Overview:** Aggregates newly found roles into a single structured summary message and delivers it to the target Slack channel.
- **Nodes Involved:** `Compose one message`, `Post new roles to Slack`
- **Node Details:**
  - **Compose one message**
    - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node.
    - *Configuration:* Limits output to a maximum of 20 items per message, formatting titles, companies, locations, posting dates, and URLs into a Markdown list. If no items are present, returns an empty array to halt execution before the Slack node.
    - *Inputs / Outputs:* Input from `Only roles not seen before` / Output to `Post new roles to Slack`.
    - *Edge Cases:* Empty input arrays terminate the branch, preventing empty notifications.
  - **Post new roles to Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` — Chat notification node.
    - *Configuration:* Sends text message via API to a selected channel.
    - *Key Expressions/Variables:* `={{ $json.text }}`
    - *Credentials Required:* Slack OAuth2 / Bot API credentials.
    - *Inputs / Outputs:* Input from `Compose one message` / Terminal node.
    - *Edge Cases:* Invalid channel IDs or expired Slack tokens will cause execution failure.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template description | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Scan Ashby, Greenhouse and Lever job boards daily and send new matches to Slack<br><br>### Who's it for<br>... |
| Configure here | `n8n-nodes-base.stickyNote` | Configuration visual grouping | None | None | ## 1 · The only node you edit<br>Boards to watch and the words that decide a match. Everything downstream reads from here. |
| Fetch | `n8n-nodes-base.stickyNote` | Data acquisition visual grouping | None | None | ## 2 · One request per board<br>Public posting APIs, no keys. The HTTP node is set to return text so that a board answering with a bare array does not get split into separate items — that keeps each response paired with the board that produced it. A board that fails is logged and skipped. |
| Filter and remember | `n8n-nodes-base.stickyNote` | Processing and deduplication grouping | None | None | ## 3 · Match, then forget what you have seen<br>Four response shapes become one row, then title and location filters run. Remove Duplicates remembers the keys it has already let through, across executions, so a posting is announced once. |
| Tell someone | `n8n-nodes-base.stickyNote` | Notification visual grouping | None | None | ## 4 · One message, not twenty<br>The roles are folded into a single Slack message. Swap Slack for Gmail, Telegram or a spreadsheet — the rows are plain JSON. |
| Run it once now | `n8n-nodes-base.manualTrigger` | Manual trigger | None | Boards and filters | |
| Every morning at 8 | `n8n-nodes-base.scheduleTrigger` | Daily schedule trigger | None | Boards and filters | |
| Boards and filters | `n8n-nodes-base.set` | Define target boards and filter keywords | Run it once now, Every morning at 8 | Build board requests | |
| Build board requests | `n8n-nodes-base.code` | Generate ATS-specific API URLs | Boards and filters | Fetch the board | |
| Fetch the board | `n8n-nodes-base.httpRequest` | Fetch public job postings via HTTP | Build board requests | Normalise and match | |
| Normalise and match | `n8n-nodes-base.code` | Parse schemas, filter titles/locations | Fetch the board | Only roles not seen before | |
| Only roles not seen before | `n8n-nodes-base.removeDuplicates` | Filter out previously alerted jobs | Normalise and match | Compose one message | |
| Compose one message | `n8n-nodes-base.code` | Aggregate roles into a single message | Only roles not seen before | Post new roles to Slack | |
| Post new roles to Slack | `n8n-nodes-base.slack` | Post consolidated summary to Slack | Compose one message | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Run it once now`.
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) named `Every morning at 8`. Configure its rule interval to trigger daily at hour `8`.

2. **Create Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Boards and filters`.
   - Set mode to `Raw` and populate the JSON output with your target `boards` array (specifying `ats` and `slug`) along with `title_any`, `title_none`, `location_any`, and `location_none` keyword arrays.
   - Connect both `Run it once now` and `Every morning at 8` to `Boards and filters`.

3. **Create Request Builder Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build board requests`.
   - Insert JavaScript to map ATS configurations (`ashby`, `greenhouse`, `lever`, `recruitee`) to their respective public API endpoints.
   - Connect `Boards and filters` to `Build board requests`.

4. **Create HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Fetch the board`.
   - Set the URL parameter to expression `={{ $json.url }}`.
   - Under options, set response format to `Text`.
   - Set error handling parameter `onError` to `continueRegularOutput`.
   - Connect `Build board requests` to `Fetch the board`.

5. **Create Normalization & Filtering Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Normalise and match`.
   - Insert parsing logic to normalize payloads from Ashby, Greenhouse, Lever, and Recruitee into unified objects (`ref`, `title`, `company`, `location`, `remote`, `url`, `posted`).
   - Implement keyword validation logic utilizing `title_any`, `title_none`, `location_any`, and `location_none` from the configuration node.
   - Connect `Fetch the board` to `Normalise and match`.

6. **Create Deduplication Node:**
   - Add a **Remove Duplicates** node (`n8n-nodes-base.removeDuplicates`) named `Only roles not seen before`.
   - Set logic to `Remove items with already seen key values` and operation to `Remove items seen in previous executions`.
   - Set the deduplication value expression to `={{ $json.id }}`.
   - Connect `Normalise and match` to `Only roles not seen before`.

7. **Create Message Composition Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Compose one message`.
   - Insert JavaScript to slice incoming rows (up to 20) and format them into a single Markdown-styled summary text object containing a `count` and `text` property.
   - Connect `Only roles not seen before` to `Compose one message`.

8. **Create Notification Node:**
   - Add a **Slack** node (`n8n-nodes-base.slack`) named `Post new roles to Slack`.
   - Configure credentials (requires Slack API/OAuth2 authentication).
   - Set action parameters to post a message to a channel (`select: channel`), defining the channel name (e.g., `#jobs`).
   - Set the message text parameter to expression `={{ $json.text }}`.
   - Connect `Compose one message` to `Post new roles to Slack`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Companion template 1 | https://n8n.io/workflows/19797 |
| Companion template 2 | https://n8n.io/workflows/19895 |