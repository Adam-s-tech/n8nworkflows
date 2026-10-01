Submit new sitemap URLs to IndexNow with Telegram alerts

https://n8nworkflows.xyz/workflows/submit-new-sitemap-urls-to-indexnow-with-telegram-alerts-20067


# Submit new sitemap URLs to IndexNow with Telegram alerts

### 1. Workflow Overview

This workflow automates the detection, IndexNow submission, and alerting of newly added URLs from an XML sitemap. It runs on a daily schedule, fetches the target sitemap, extracts and filters URLs, compares them against a persistent history of previously tracked pages, and propagates new discoveries to search engines via the IndexNow API while notifying administrators via Telegram.

The logic is grouped into three distinct functional blocks:
- **1.1 Schedule & Configuration:** Triggers the workflow daily at a fixed time and initializes all required environmental variables, including sitemap URLs, host domains, filtering criteria, API authentication parameters, and notification endpoints.
- **1.2 Sitemap Retrieval & Change Detection:** Downloads the XML sitemap, parses its internal location elements, applies path-based filtering, and performs a differential comparison against workflow static data to isolate unregistered pages.
- **1.3 API Submission & Alerting:** Evaluates whether newly discovered URLs exist, processes HTTP POST submissions to the IndexNow protocol endpoint, and delivers formatted notifications to a specified Telegram chat identifier.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the execution timing and provides centralized configuration variables that govern the behavior of downstream HTTP requests, filtering rules, and alerting destinations.
- **Nodes Involved:** `When Every Morning at 8am`, `Set Sitemap Configurations`

##### Node Details:
- **When Every Morning at 8am**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
  - *Configuration Choices:* Configured with a cron expression to execute every day at 08:00 AM.
  - *Key Expressions/Variables:* Uses standard cron syntax: `0 8 * * *`.
  - *Input/Output Connections:* Output connects exclusively to `Set Sitemap Configurations`.
  - *Version-Specific Requirements:* Version 1.2.
  - *Edge Cases/Failure Types:* Failure to execute if n8n instance experiences downtime or host system time drifts.

- **Set Sitemap Configurations**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation / Setup Node)
  - *Configuration Choices:* Defines fixed strings for sitemap locations, path filters, hostnames, API keys, key file hosting locations, and Telegram chat identifiers.
  - *Key Expressions/Variables:* Assigns properties: `sitemapUrl`, `urlFilter`, `siteHost`, `indexNowKey`, `indexNowKeyLocation`, and `telegramChatId`.
  - *Input/Output Connections:* Input from `When Every Morning at 8am`; output to `Fetch Sitemap Data`.
  - *Version-Specific Requirements:* Version 3.4.
  - *Edge Cases/Failure Types:* Incorrect configuration formats or placeholder values (`YOUR_INDEXNOW_KEY`) will break downstream HTTP requests and API authentication.

---

#### 2.2 Sitemap Retrieval & Change Detection
- **Overview:** Fetches the raw XML sitemap via HTTP, extracts all URL tags using regular expressions, applies optional path filtering, and cross-references them against workflow static storage to identify new entries.
- **Nodes Involved:** `Fetch Sitemap Data`, `Identify New URLs`

##### Node Details:
- **Fetch Sitemap Data**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Network Request Node)
  - *Configuration Choices:* Performs an HTTP GET request to the dynamic sitemap URL with a custom User-Agent header, returning the response body strictly as text.
  - *Key Expressions/Variables:* URL evaluated via `={{ $json.sitemapUrl }}`. Header includes `User-Agent: Mozilla/5.0 (compatible; n8n-sitemap-watcher)`.
  - *Input/Output Connections:* Input from `Set Sitemap Configurations`; output to `Identify New URLs`.
  - *Version-Specific Requirements:* Version 4.2.
  - *Edge Cases/Failure Types:* HTTP errors (404, 500, timeouts) if the sitemap server is unreachable or responds with malformed data.

- **Identify New URLs**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node)
  - *Configuration Choices:* Parses XML string content using regex matching (`/<loc>\s*(.*?)\s*<\/loc>/g`), filters paths conditionally, and reads/writes persistent workflow static data (`$getWorkflowStaticData('global')`).
  - *Key Expressions/Variables:* References configuration values via `$('Set Sitemap Configurations').first().json`.
  - *Input/Output Connections:* Input from `Fetch Sitemap Data`; output to `If URLs Need Submission`.
  - *Version-Specific Requirements:* Version 2.
  - *Edge Cases/Failure Types:* Manual execution runs always treat the current run as the "first run" due to how n8n handles static data in manual tests; malformed XML structures may yield zero matches.

---

#### 2.3 API Submission & Alerting
- **Overview:** Evaluates whether new URLs were detected, conditionally dispatches them to the IndexNow protocol endpoint, and transmits the resulting alert message to Telegram.
- **Nodes Involved:** `If URLs Need Submission`, `Post URLs to IndexNow API`, `Send Telegram Notification`

##### Node Details:
- **If URLs Need Submission**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control Node)
  - *Configuration Choices:* Evaluates boolean evaluation conditions against the detection status.
  - *Key Expressions/Variables:* Left value: `={{ $json.hasNewUrls }}`, checked against boolean `true`.
  - *Input/Output Connections:* Input from `Identify New URLs`. True branch output connects to `Post URLs to IndexNow API`; false branch output connects to `Send Telegram Notification`.
  - *Version-Specific Requirements:* Version 2.2.
  - *Edge Cases/Failure Types:* Expression evaluation failures if previous node output structures deviate from expectations.

- **Post URLs to IndexNow API**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node)
  - *Configuration Choices:* Executes a POST request to the IndexNow endpoint with a constructed JSON body containing host details, keys, key file paths, and the array of new URLs. Configured to continue regular execution even if errors occur.
  - *Key Expressions/Variables:* JSON body compiled dynamically using `JSON.stringify` referencing configuration parameters and `$json.newUrls`.
  - *Input/Output Connections:* Input from `If URLs Need Submission` (True branch); output to `Send Telegram Notification`.
  - *Version-Specific Requirements:* Version 4.2.
  - *Edge Cases/Failure Types:* Rate limits, invalid API keys, or mismatched host values will cause API rejection (handled by `continueRegularOutput`).

- **Send Telegram Notification**
  - *Type & Technical Role:* `n8n-nodes-base.telegram` (Messaging Integration Node)
  - *Configuration Choices:* Sends a text message to a specified Telegram chat using bot credentials. Attribution appending is disabled.
  - *Key Expressions/Variables:* Text fetched via `={{ $('Identify New URLs').first().json.message }}`; chat ID fetched via `={{ $('Set Sitemap Configurations').first().json.telegramChatId }}`.
  - *Input/Output Connections:* Inputs accepted from both `If URLs Need Submission` (False branch) and `Post URLs to IndexNow API`. Has no downstream outputs.
  - *Version-Specific Requirements:* Version 1.2.
  - *Edge Cases/Failure Types:* Invalid Telegram credentials, revoked bot tokens, or invalid chat IDs will cause execution failure.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & Instructions | None | None | ## Submit new sitemap URLs to IndexNow and send Telegram alerts<br><br>### How it works<br><br>This workflow runs every morning, loads your configuration values, fetches your XML sitemap, and detects URLs that have not been seen before. When new URLs are found, it submits them to the IndexNow API so Bing and other IndexNow search engines crawl them quickly, then sends a Telegram alert listing the new pages.<br><br>The first run only saves a baseline: nothing is submitted and you receive a confirmation message.<br><br>### Setup steps<br><br>- Generate an IndexNow key and host the key file at the root of your site.<br>- Open the "Set Sitemap Configurations" node and fill in sitemapUrl, the optional urlFilter, siteHost, indexNowKey, indexNowKeyLocation and telegramChatId.<br>- Connect your Telegram credentials in the "Send Telegram Notification" node.<br>- Activate the workflow. Known URLs are stored in workflow static data, which is only saved when the workflow is active.<br><br>### Customization<br><br>- Set urlFilter to "/blog" to watch only blog posts, or leave it empty to watch the whole site.<br>- Change the schedule, the Telegram message text, or replace Telegram with Slack or email.<br>- The workflow reads one sitemap file. If you use a sitemap index, point it to the child sitemap you want to watch. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation & Instructions | None | None | ## Schedule and configure<br><br>Starts the workflow on a morning schedule and defines the sitemap, host, filtering, and IndexNow key values used downstream. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation & Instructions | None | None | ## Fetch and detect URLs<br><br>Retrieves the configured sitemap and compares its URLs against stored history to return only newly discovered URLs. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation & Instructions | None | None | ## Submit and alert<br><br>Checks whether there are new URLs to process, submits them to IndexNow when present, and sends a Telegram notification from either the decision path or after submission. |
| `When Every Morning at 8am` | `n8n-nodes-base.scheduleTrigger` | Daily schedule trigger | None | `Set Sitemap Configurations` | |
| `Set Sitemap Configurations` | `n8n-nodes-base.set` | Define configuration parameters | `When Every Morning at 8am` | `Fetch Sitemap Data` | ## Schedule and configure<br><br>Starts the workflow on a morning schedule and defines the sitemap, host, filtering, and IndexNow key values used downstream. |
| `Fetch Sitemap Data` | `n8n-nodes-base.httpRequest` | Fetch XML sitemap contents | `Set Sitemap Configurations` | `Identify New URLs` | ## Fetch and detect URLs<br><br>Retrieves the configured sitemap and compares its URLs against stored history to return only newly discovered URLs. |
| `Identify New URLs` | `n8n-nodes-base.code` | Extract URLs and filter new entries | `Fetch Sitemap Data` | `If URLs Need Submission` | ## Fetch and detect URLs<br><br>Retrieves the configured sitemap and compares its URLs against stored history to return only newly discovered URLs. |
| `If URLs Need Submission` | `n8n-nodes-base.if` | Check if new URLs require submission | `Identify New URLs` | `Post URLs to IndexNow API`, `Send Telegram Notification` | ## Submit and alert<br><br>Checks whether there are new URLs to process, submits them to IndexNow when present, and sends a Telegram notification from either the decision path or after submission. |
| `Post URLs to IndexNow API` | `n8n-nodes-base.httpRequest` | Submit URLs to IndexNow endpoint | `If URLs Need Submission` | `Send Telegram Notification` | ## Submit and alert<br><br>Checks whether there are new URLs to process, submits them to IndexNow when present, and sends a Telegram notification from either the decision path or after submission. |
| `Send Telegram Notification` | `n8n-nodes-base.telegram` | Send alerts via Telegram | `If URLs Need Submission`, `Post URLs to IndexNow API` | None | ## Submit and alert<br><br>Checks whether there are new URLs to process, submits them to IndexNow when present, and sends a Telegram notification from either the decision path or after submission. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `When Every Morning at 8am`.
   - Set parameters: Interval to `Cron Expression`, expression to `0 8 * * *`.

2. **Create the Configuration Node:**
   - Add a **Set** (`n8n-nodes-base.set`) node named `Set Sitemap Configurations`.
   - Connect `When Every Morning at 8am` to this node.
   - Configure string assignments:
     - `sitemapUrl`: `https://www.example.com/sitemap.xml`
     - `urlFilter`: `/blog`
     - `siteHost`: `www.example.com`
     - `indexNowKey`: `YOUR_INDEXNOW_KEY`
     - `indexNowKeyLocation`: `https://www.example.com/YOUR_INDEXNOW_KEY.txt`
     - `telegramChatId`: `YOUR_TELEGRAM_CHAT_ID`

3. **Create the HTTP Request Node for the Sitemap:**
   - Add an **HTTP Request** (`n8n-nodes-base.httpRequest`) node named `Fetch Sitemap Data`.
   - Connect `Set Sitemap Configurations` to this node.
   - Set method to `GET`, URL to `={{ $json.sitemapUrl }}`.
   - Under Options -> Response, set Response Format to `Text` and Output Property Name to `data`.
   - Add a header parameter: Name `User-Agent`, Value `Mozilla/5.0 (compatible; n8n-sitemap-watcher)`.

4. **Create the Code Node for URL Analysis:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Identify New URLs`.
   - Connect `Fetch Sitemap Data` to this node.
   - Insert the following JavaScript logic:
     ```javascript
     const config = $('Set Sitemap Configurations').first().json;
     const store = $getWorkflowStaticData('global');
     store.knownUrls = store.knownUrls || {};

     const xml = String($input.first().json.data || '');
     const allUrls = [...xml.matchAll(/<loc>\s*(.*?)\s*<\/loc>/g)].map((m) => m[1]);
     const filter = String(config.urlFilter || '').trim();
     const watched = filter ? allUrls.filter((u) => u.includes(filter)) : allUrls;

     const isFirstRun = Object.keys(store.knownUrls).length === 0;
     const today = new Date().toISOString().slice(0, 10);
     const newUrls = watched.filter((u) => !store.knownUrls[u]);
     for (const u of newUrls) store.knownUrls[u] = today;

     if (isFirstRun) {
       return [{ json: { hasNewUrls: false, newUrls: [],
         message: 'Baseline saved: ' + watched.length + ' URLs are now tracked. You will be alerted when a new one appears.' } }];
     }
     if (newUrls.length === 0) return [];

     return [{ json: { hasNewUrls: true, newUrls,
       message: 'New page(s) detected on your site:\n' + newUrls.join('\n') } }];
     ```

5. **Create the Flow Control Branching Node:**
   - Add an **If** (`n8n-nodes-base.if`) node named `If URLs Need Submission`.
   - Connect `Identify New URLs` to this node.
   - Set condition: Left Value `={{ $json.hasNewUrls }}`, Operator `true`, Type `Boolean`.

6. **Create the IndexNow API Request Node:**
   - Add an **HTTP Request** (`n8n-nodes-base.httpRequest`) node named `Post URLs to IndexNow API`.
   - Connect the **true output index (0)** of `If URLs Need Submission` to this node.
   - Set method to `POST`, URL to `https://api.indexnow.org/indexnow`.
   - Set Body Content Type to `JSON`.
   - Set JSON Body expression:
     `={{ JSON.stringify({ host: $('Set Sitemap Configurations').first().json.siteHost, key: $('Set Sitemap Configurations').first().json.indexNowKey, keyLocation: $('Set Sitemap Configurations').first().json.indexNowKeyLocation, urlList: $json.newUrls }) }}`
   - In node settings, enable **On Error** -> `Continue Regular Output`.

7. **Create the Telegram Notification Node:**
   - Add a **Telegram** (`n8n-nodes-base.telegram`) node named `Send Telegram Notification`.
   - Connect the **false output index (1)** of `If URLs Need Submission` AND the output of `Post URLs to IndexNow API` to this node.
   - Configure credentials: Add valid Telegram Bot credentials.
   - Set parameters:
     - Resource: `Message`
     - Operation: `Send Message`
     - Chat ID: `={{ $('Set Sitemap Configurations').first().json.telegramChatId }}`
     - Text: `={{ $('Identify New URLs').first().json.message }}`
     - Additional Fields -> Append Attribution: `false`.

8. **Activation:**
   - Save the workflow and activate it so persistent workflow static data functions correctly.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| IndexNow protocol specification and key hosting guidelines | [IndexNow Documentation](https://www.indexnow.org/) |
| n8n execution static data persistence behavior | [n8n Static Data Guide](https://docs.n8n.io/workflows/code/builtin-methods-variables/#getworkflowstaticdata) |