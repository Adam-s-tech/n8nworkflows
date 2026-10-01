Track Trustpilot reviews with Scrapio, Google Sheets and Slack alerts

https://n8nworkflows.xyz/workflows/track-trustpilot-reviews-with-scrapio--google-sheets-and-slack-alerts-20016


# Track Trustpilot reviews with Scrapio, Google Sheets and Slack alerts

### 1. Workflow Overview

This workflow automates the collection, processing, and reporting of Trustpilot reviews for a defined list of business domains. It runs on dual schedules to provide both real-time alerts for incoming feedback and a comprehensive weekly digest. 

The logical architecture comprises two main execution paths:
- **Daily Review Processing:** Automatically pulls fresh Trustpilot reviews via Scrapio, cross-references them with Google Sheets to filter out duplicates, logs new entries, and dynamically routes alerts to dedicated Slack channels based on star ratings.
- **Weekly Summary Digest:** Periodically queries the historical review log, aggregates performance metrics from the past 7 days, and publishes a structured statistical recap to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & History Loading
- **Overview:** Triggers the daily check, initializes the target list of corporate domains, and fetches existing log data from Google Sheets to build a baseline for deduplication.
- **Nodes Involved:** `Every Day`, `Set Businesses to Track`, `Read Review History`, `Aggregate Review History`
- **Node Details:**
  - **Every Day** (`n8n-nodes-base.scheduleTrigger`)
    - *Type/Role:* Schedule trigger initiating execution on a daily interval.
    - *Configuration:* Interval configured to `daysInterval: 1`.
    - *Connections:* Outputs to `Set Businesses to Track` and `Read Review History`.
    - *Edge Cases:* Missed triggers due to system downtime depend on n8n instance configuration.
  - **Set Businesses to Track** (`n8n-nodes-base.set`)
    - *Type/Role:* Data initialization node defining domains to inspect.
    - *Configuration:* Raw mode with a static JSON payload containing an array of domains (e.g., `amazon.com`, `walmart.com`).
    - *Expressions:* None (static payload).
    - *Connections:* Input from `Every Day`, output to `Combine Businesses with History` (Input 0).
  - **Read Review History** (`n8n-nodes-base.googleSheets`)
    - *Type/Role:* Database read operation extracting historical reviews.
    - *Configuration:* Operation set to `read` using list selectors for spreadsheet and sheet.
    - *Connections:* Input from `Every Day`, output to `Aggregate Review History`.
    - *Edge Cases:* API rate limits, revoked Google OAuth credentials, or missing sheet headers.
  - **Aggregate Review History** (`n8n-nodes-base.aggregate`)
    - *Type/Role:* Data transformation node consolidating rows into a single array.
    - *Configuration:* `aggregateAllItemData` with destination field name set to `data`.
    - *Connections:* Input from `Read Review History`, output to `Combine Businesses with History` (Input 1).

#### 2.2 Business List Preparation & Scraping
- **Overview:** Merges the static business list with historical records, splits them into individual execution items, and queries Scrapio for recent Trustpilot reviews.
- **Nodes Involved:** `Combine Businesses with History`, `Split Out Businesses`, `Fetch Trustpilot Reviews`
- **Node Details:**
  - **Combine Businesses with History** (`n8n-nodes-base.merge`)
    - *Type/Role:* Data merging node joining domains with historical review data by position.
    - *Configuration:* Mode set to `combineByPosition`, 2 inputs.
    - *Connections:* Input 0 from `Set Businesses to Track`, Input 1 from `Aggregate Review History`, output to `Split Out Businesses`.
  - **Split Out Businesses** (`n8n-nodes-base.splitOut`)
    - *Type/Role:* Array expansion node converting the business array back into individual items.
    - *Configuration:* Field to split out set to `businesses`, retaining all other fields.
    - *Connections:* Input from `Combine Businesses with History`, output to `Fetch Trustpilot Reviews`.
  - **Fetch Trustpilot Reviews** (`n8n-nodes-scrapio.scrapio`)
    - *Type/Role:* External API integration fetching Trustpilot review data.
    - *Configuration:* Resource set to `trustpilot`, operation set to `reviews`, limit set to 20.
    - *Expressions:* Domain parameter uses `={{ $json.domain }}`.
    - *Connections:* Input from `Split Out Businesses`, output to `Find New Reviews`.
    - *Edge Cases:* Invalid API credentials, unreachable domains, or Scrapio service downtime.

#### 2.3 Review Filtering & Logging
- **Overview:** Compares scraped reviews against historical records to isolate unseen entries, formats the payloads, and persists them to Google Sheets.
- **Nodes Involved:** `Find New Reviews`, `Append Review to Sheets`
- **Node Details:**
  - **Find New Reviews** (`n8n-nodes-base.code`)
    - *Type/Role:* Custom JavaScript execution node for deduplication and payload structuring.
    - *Configuration:* Iterates through incoming reviews, filters out IDs present in the historical dataset, and structures new review objects.
    - *Expressions:* JavaScript environment utilizing `$('Split Out Businesses')` and `$input.all()`.
    - *Connections:* Input from `Fetch Trustpilot Reviews`, output to `Append Review to Sheets`.
  - **Append Review to Sheets** (`n8n-nodes-base.googleSheets`)
    - *Type/Role:* Database write operation adding new reviews to Google Sheets.
    - *Configuration:* Operation set to `append`, mapping mode set to define columns explicitly.
    - *Expressions:* Maps fields (`text`, `title`, `domain`, `rating`, `logged_at`, `review_id`, `is_verified`, `trust_score`, `business_name`, `consumer_name`, `published_date`, `business_website`, `consumer_country`) directly from `={{ $json.[field] }}`.
    - *Connections:* Input from `Find New Reviews`, output to `Route by Rating`.
    - *Edge Cases:* Schema mismatch or write permission errors.

#### 2.4 Rating-Based Slack Routing
- **Overview:** Evaluates each new review's star rating and routes the message to the corresponding Slack alert channel.
- **Nodes Involved:** `Route by Rating`, `Send Low Rating Alert to Slack`, `Post Great Review to Slack`, `Log Review to Slack`
- **Node Details:**
  - **Route by Rating** (`n8n-nodes-base.switch`)
    - *Type/Role:* Conditional routing node based on numeric values.
    - *Configuration:* Rule 1 (Low Rating): `rating` $\le$ 2. Rule 2 (Great Review): `rating` $\ge$ 4. Fallback output routed as `neutral`.
    - *Expressions:* Evaluates `={{ $json.rating }}`.
    - *Connections:* Input from `Append Review to Sheets`. Outputs connect to their respective Slack nodes.
  - **Send Low Rating Alert to Slack** (`n8n-nodes-base.slack`)
    - *Type/Role:* Messaging integration for critical negative feedback.
    - *Configuration:* Target set to channel, utilizing channel ID selector.
    - *Expressions:* Formatted markdown text including business name, rating stars, title, consumer details, and review text.
    - *Connections:* Input from `Route by Rating` (output 0).
  - **Post Great Review to Slack** (`n8n-nodes-base.slack`)
    - *Type/Role:* Messaging integration for positive feedback.
    - *Configuration:* Target set to channel, utilizing channel ID selector.
    - *Expressions:* Formatted markdown text with celebratory emojis for ratings $\ge$ 4.
    - *Connections:* Input from `Route by Rating` (output 1).
  - **Log Review to Slack** (`n8n-nodes-base.slack`)
    - *Type/Role:* Messaging integration for neutral feedback.
    - *Configuration:* Target set to channel, utilizing channel ID selector.
    - *Expressions:* Formatted markdown text for standard review logs.
    - *Connections:* Input from `Route by Rating` (fallback output).

#### 2.5 Weekly Summary & Digest
- **Overview:** Triggers weekly, isolates reviews logged within the past 7 days, aggregates performance metrics, and posts a statistical summary to Slack.
- **Nodes Involved:** `Every Week`, `Read Review Log`, `Filter Last 7 Days`, `Aggregate Weekly Reviews`, `Build Weekly Summary`, `Send Weekly Review Digest to Slack`
- **Node Details:**
  - **Every Week** (`n8n-nodes-base.scheduleTrigger`)
    - *Type/Role:* Schedule trigger initiating weekly execution.
    - *Configuration:* Interval configured to `weeksInterval: 1`.
    - *Connections:* Outputs to `Read Review Log`.
  - **Read Review Log** (`n8n-nodes-base.googleSheets`)
    - *Type/Role:* Database read operation for historical log data.
    - *Configuration:* Operation set to `read`.
    - *Connections:* Input from `Every Week`, output to `Filter Last 7 Days`.
  - **Filter Last 7 Days** (`n8n-nodes-base.filter`)
    - *Type/Role:* Data filtering node keeping only recent records.
    - *Configuration:* DateTime condition comparing `logged_at` to the date 7 days ago.
    - *Expressions:* `={{ new Date($json.logged_at) }}` after `={{ $today.minus({ days: 7 }) }}`.
    - *Connections:* Input from `Read Review Log`, output to `Aggregate Weekly Reviews`.
  - **Aggregate Weekly Reviews** (`n8n-nodes-base.aggregate`)
    - *Type/Role:* Data transformation node combining filtered items.
    - *Configuration:* `aggregateAllItemData` with destination field name `data`.
    - *Connections:* Input from `Filter Last 7 Days`, output to `Build Weekly Summary`.
  - **Build Weekly Summary** (`n8n-nodes-base.set`)
    - *Type/Role:* Data calculation node computing metrics (total count, average rating, low/high counts, worst review).
    - *Configuration:* Manual assignment mode with custom JavaScript expressions.
    - *Expressions:* Computes array length, averages rating values, filters counts, and sorts lowest-rated items.
    - *Connections:* Input from `Aggregate Weekly Reviews`, output to `Send Weekly Review Digest to Slack`.
  - **Send Weekly Review Digest to Slack** (`n8n-nodes-base.slack`)
    - *Type/Role:* Messaging integration delivering the weekly analytics report.
    - *Configuration:* Target set to channel via channel ID selector.
    - *Expressions:* Formatted markdown message incorporating total reviews, average rating, breakdown counts, and the worst review snippet.
    - *Connections:* Input from `Build Weekly Summary`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | Track Trustpilot reviews and alert on ratings in Slack<br><br>### How it works<br><br>1. Every day, Scrapio fetches the latest Trustpilot reviews for each business domain you're tracking.<br>2. New reviews are found by comparing against your Google Sheet history, so nothing is logged twice.<br>3. Each new review is logged to Google Sheets, then routed by star rating to the matching Slack channel.<br>4. A separate weekly branch reads the sheet and posts a rating digest to Slack.<br><br>### Setup steps<br><br>- [ ] Add your Scrapio API credential (get a key at app.scrapio.dev)<br>- [ ] Edit the business domains in Set Businesses to Track<br>- [ ] Connect your Google Sheet in both Read nodes and Append Review to Sheets (header row: domain, review_id, business_name, business_website, trust_score, rating, title, text, consumer_name, consumer_country, is_verified, published_date, logged_at)<br>- [ ] Add your Slack credential and channel IDs to the three alert nodes and Send Weekly Review Digest to Slack<br>- [ ] Activate the workflow<br><br>### Customization tips<br><br>Adjust the rating thresholds in Route by Rating, or change either schedule. |
| Every Day | n8n-nodes-base.scheduleTrigger | Daily execution trigger | None | Set Businesses to Track, Read Review History | Start daily check<br><br>Runs on a schedule and lists the business domains you're tracking. |
| Set Businesses to Track | n8n-nodes-base.set | Defines domains to monitor | Every Day | Combine Businesses with History | Start daily check<br><br>Runs on a schedule and lists the business domains you're tracking. |
| Read Review History | n8n-nodes-base.googleSheets | Reads historical log from Google Sheets | Every Day | Aggregate Review History | Load review history<br><br>Reads every past logged review into one item for lookup. |
| Aggregate Review History | n8n-nodes-base.aggregate | Consolidates review history into single item | Read Review History | Combine Businesses with History | Load review history<br><br>Reads every past logged review into one item for lookup. |
| Combine Businesses with History | n8n-nodes-base.merge | Joins domain list with historical review data | Set Businesses to Track, Aggregate Review History | Split Out Businesses | Prepare business list<br><br>Attaches the review history to each business before splitting. |
| Split Out Businesses | n8n-nodes-base.splitOut | Expands domain array into individual items | Combine Businesses with History | Fetch Trustpilot Reviews | Prepare business list<br><br>Attaches the review history to each business before splitting. |
| Fetch Trustpilot Reviews | n8n-nodes-scrapio.scrapio | Fetches Trustpilot reviews via Scrapio API | Split Out Businesses | Find New Reviews | Fetch and find new reviews<br><br>Pulls each business's latest reviews and keeps only the ones not seen before. |
| Find New Reviews | n8n-nodes-base.code | Deduplicates reviews against historical logs | Fetch Trustpilot Reviews | Append Review to Sheets | Fetch and find new reviews<br><br>Pulls each business's latest reviews and keeps only the ones not seen before. |
| Append Review to Sheets | n8n-nodes-base.googleSheets | Appends new review data to Google Sheets | Find New Reviews | Route by Rating | Log and route reviews<br><br>Logs every new review, then routes it by star rating. |
| Route by Rating | n8n-nodes-base.switch | Routes reviews based on star rating | Append Review to Sheets | Send Low Rating Alert to Slack, Post Great Review to Slack, Log Review to Slack | Log and route reviews<br><br>Logs every new review, then routes it by star rating. |
| Send Low Rating Alert to Slack | n8n-nodes-base.slack | Posts low rating alerts (1-2★) to Slack | Route by Rating | None | Send Slack alerts<br><br>Posts low ratings, great reviews, and everything else to separate channels. |
| Post Great Review to Slack | n8n-nodes-base.slack | Posts great review alerts (4-5★) to Slack | Route by Rating | None | Send Slack alerts<br><br>Posts low ratings, great reviews, and everything else to separate channels. |
| Log Review to Slack | n8n-nodes-base.slack | Posts neutral review logs to Slack | Route by Rating | None | Send Slack alerts<br><br>Posts low ratings, great reviews, and everything else to separate channels. |
| Every Week | n8n-nodes-base.scheduleTrigger | Weekly execution trigger | None | Read Review Log | Load last week's reviews<br><br>On a weekly schedule, reads the log and keeps the last 7 days. |
| Read Review Log | n8n-nodes-base.googleSheets | Reads complete review log from Google Sheets | Every Week | Filter Last 7 Days | Load last week's reviews<br><br>On a weekly schedule, reads the log and keeps the last 7 days. |
| Filter Last 7 Days | n8n-nodes-base.filter | Filters review items from the past 7 days | Read Review Log | Aggregate Weekly Reviews | Load last week's reviews<br><br>On a weekly schedule, reads the log and keeps the last 7 days. |
| Aggregate Weekly Reviews | n8n-nodes-base.aggregate | Consolidates filtered weekly reviews into single array | Filter Last 7 Days | Build Weekly Summary | Summarize and post digest<br><br>Averages the week's ratings and shares the lowest-rated review in Slack. |
| Build Weekly Summary | n8n-nodes-base.set | Calculates weekly metrics and worst review | Aggregate Weekly Reviews | Send Weekly Review Digest to Slack | Summarize and post digest<br><br>Averages the week's ratings and shares the lowest-rated review in Slack. |
| Send Weekly Review Digest to Slack | n8n-nodes-base.slack | Posts weekly analytics digest to Slack | Build Weekly Summary | None | Summarize and post digest<br><br>Averages the week's ratings and shares the lowest-rated review in Slack. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Daily Trigger:** Add a **Schedule Trigger** node (`Every Day`) and set its interval to daily (`daysInterval: 1`).
2. **Configure Business List:** Add a **Set** node (`Set Businesses to Track`), set mode to `raw`, and configure the JSON output with a `businesses` array containing domain objects (e.g., `amazon.com`).
3. **Read Historical Data (Daily Branch):** Add a **Google Sheets** node (`Read Review History`), set operation to `read`, and connect your Google account credential, spreadsheet, and sheet.
4. **Aggregate History:** Add an **Aggregate** node (`Aggregate Review History`), set aggregation to `aggregateAllItemData`, and assign the destination field to `data`.
5. **Merge Inputs:** Add a **Merge** node (`Combine Businesses with History`), set mode to `combineByPosition` with 2 inputs. Connect `Set Businesses to Track` to Input 0 and `Aggregate Review History` to Input 1.
6. **Split Domains:** Add a **Split Out** node (`Split Out Businesses`), set the field to split out to `businesses`. Connect input from the Merge node.
7. **Scrape Reviews:** Add a **Scrapio** node (`Fetch Trustpilot Reviews`), configure the resource to `trustpilot`, operation to `reviews`, limit to `20`, and set domain expression to `={{ $json.domain }}`. Connect input from `Split Out Businesses`.
8. **Deduplicate Reviews:** Add a **Code** node (`Find New Reviews`), paste the deduplication JavaScript snippet to compare incoming review IDs against stored history, and output structured review objects.
9. **Log to Sheets:** Add a **Google Sheets** node (`Append Review to Sheets`), set operation to `append`, and map all 13 required review attributes (`domain`, `review_id`, `business_name`, `business_website`, `trust_score`, `rating`, `title`, `text`, `consumer_name`, `consumer_country`, `is_verified`, `published_date`, `logged_at`) to their respective `={{ $json.[field] }}` expressions.
10. **Route by Rating:** Add a **Switch** node (`Route by Rating`). Define Rule 1 for `rating <= 2` (outputKey: `lowRating`) and Rule 2 for `rating >= 4` (outputKey: `greatReview`), with fallback output handling neutral reviews.
11. **Configure Slack Alerts:** Add three **Slack** nodes (`Send Low Rating Alert to Slack`, `Post Great Review to Slack`, `Log Review to Slack`), connect them to the respective outputs of the Switch node, configure Slack OAuth2 credentials, select target channels, and set up markdown templates.
12. **Create Weekly Trigger:** Add a **Schedule Trigger** node (`Every Week`) and set its interval to weekly (`weeksInterval: 1`).
13. **Read Log (Weekly Branch):** Add a **Google Sheets** node (`Read Review Log`), set operation to `read`, and link it to the weekly schedule.
14. **Filter Weekly Reviews:** Add a **Filter** node (`Filter Last 7 Days`), configuring a condition where `logged_at` is after `={{ $today.minus({ days: 7 }) }}`.
15. **Aggregate Weekly Data:** Add an **Aggregate** node (`Aggregate Weekly Reviews`) to bundle filtered records into an array under the `data` field.
16. **Build Summary:** Add a **Set** node (`Build Weekly Summary`) in manual assignment mode to compute `total`, `averageRating`, `lowCount`, `highCount`, and `worstReview` metrics using JavaScript expressions.
17. **Send Weekly Digest:** Add a **Slack** node (`Send Weekly Review Digest to Slack`), connect it to the summary node, select the target Slack channel, and configure the formatted summary text.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Scrapio API Credential | Obtain an API key from [App Scrapio](https://app.scrapio.dev) |
| Google Sheets Schema Requirements | Ensure sheet contains exact headers: `domain`, `review_id`, `business_name`, `business_website`, `trust_score`, `rating`, `title`, `text`, `consumer_name`, `consumer_country`, `is_verified`, `published_date`, `logged_at` |