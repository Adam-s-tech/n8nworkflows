Monitor brand sentiment with Reddit, Hacker News, Google News and GPT-4o mini

https://n8nworkflows.xyz/workflows/monitor-brand-sentiment-with-reddit--hacker-news--google-news-and-gpt-4o-mini-19834


# Monitor brand sentiment with Reddit, Hacker News, Google News and GPT-4o mini

### 1. Workflow Overview

This workflow automates brand reputation management by monitoring online mentions across multiple platforms, analyzing sentiment and crisis levels using AI, logging all data to a centralized spreadsheet, and sending high-priority email alerts when urgent reputation risks arise.

The architecture is divided into the following logical blocks:
- **1.1 Schedule & Configuration:** Triggers execution automatically on a regular cadence and loads environment variables, brand names, competitors, and API keys.
- **1.2 Multi-Platform Search:** Executes parallel requests to fetch recent mentions from Reddit, Hacker News, and Google News.
- **1.3 Data Aggregation & Filtering:** Merges, parses, and normalizes mention payloads into a uniform structure, checking whether any results were found before proceeding.
- **1.4 AI Sentiment & Crisis Intelligence:** Sends collected mentions to an OpenAI chat model configured with a structured output parser to extract sentiment scores, concerns, opportunities, and crisis levels.
- **1.5 Persistence & Conditional Alerting:** Formulates alerts and log entries, appends records permanently to Google Sheets, and evaluates whether a crisis threshold has been met to trigger Gmail notifications.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Configuration

##### Overview
This block initiates the automation on a recurring schedule and injects centralized configuration variables (brand names, competitor names, credentials, and date formatting) into the processing pipeline.

##### Nodes Involved
- `Check every 4 hours`
- `Set config`

##### Node Details

- **Check every 4 hours**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger)
  - **Configuration:** Configured with a cron expression (`0 */4 * * *`) to fire automatically every four hours.
  - **Expressions/Variables:** Uses cron pattern definitions.
  - **Connections:** Input: None; Output: `Set config`.
  - **Edge Cases/Failures:** Missed executions during server downtime or n8n instance restarts if catching up is not enabled.

- **Set config**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation / Initialization)
  - **Configuration:** Assigns 10 static and dynamic key-value configuration variables, including `brandName`, `competitor1Name`, `competitor2Name`, `serperApiKey`, `recipientEmail`, `senderName`, `sheetId`, `sheetName`, `combinedQuery`, and `today`.
  - **Expressions/Variables:** Uses `{{ $json.brandName }}` and `{{ $now.toFormat('dd MMM yyyy HH:mm') }}`.
  - **Connections:** Input: `Check every 4 hours`; Output: `Search Reddit`, `Search Hacker News`, `Search the news`.
  - **Edge Cases/Failures:** Missing or placeholder configuration values will cause subsequent API requests to fail or query incorrect search terms.

---

#### Block 1.2: Multi-Platform Search

##### Overview
This block executes parallel HTTP requests to public APIs across three distinct channels (Reddit, Hacker News, and Google News via Serper) to gather brand and competitor mentions from the past week.

##### Nodes Involved
- `Search Reddit`
- `Search Hacker News`
- `Search the news`

##### Node Details

- **Search Reddit**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration)
  - **Configuration:** Sends a `GET` request to Reddit's public search endpoint (`/search.json`), sorting by new posts from the past week (`t=week`), retrieving up to 25 links. Includes a mandatory `User-Agent` header (`n8n-brand-monitor/1.0`). Configured with `onError: continueRegularOutput` to prevent node failure on API rate limits.
  - **Expressions/Variables:** Uses `{{ encodeURIComponent($json.brandName + ' OR ' + $json.competitor1Name) }}`.
  - **Connections:** Input: `Set config`; Output: `Combine the mentions`.
  - **Edge Cases/Failures:** HTTP 429 (Rate Limit) or network timeouts (configured to 15,000ms). Handled gracefully via `continueRegularOutput`.

- **Search Hacker News**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration)
  - **Configuration:** Sends a `GET` request to the Algolia Hacker News search API (`/api/v1/search`), querying stories and comments (`tags=comment,story`) with a limit of 20 hits. Timeout is set to 10,000ms.
  - **Expressions/Variables:** Uses `{{ $('Set config').item.json.brandName }}`.
  - **Connections:** Input: `Set config`; Output: `Combine the mentions`.
  - **Edge Cases/Failures:** API downtime or invalid brand strings resulting in empty hit arrays.

- **Search the news**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration)
  - **Configuration:** Sends a `POST` request to the Serper News API (`google.serper.dev/news`). Requires custom headers (`X-API-KEY`, `Content-Type`) and a JSON body targeting reviews, complaints, and feedback over the past 7 days (`tbs=qdr:w`).
  - **Expressions/Variables:** Uses `{{ $('Set config').item.json.brandName }}`, `{{ $('Set config').item.json.competitor1Name }}`, and `{{ $('Set config').item.json.serperApiKey }}`.
  - **Connections:** Input: `Set config`; Output: `Combine the mentions`.
  - **Edge Cases/Failures:** Invalid API keys, exhausted monthly quotas (Serper free tier limits), or malformed JSON payloads.

---

#### Block 1.3: Data Aggregation & Filtering

##### Overview
This block aggregates responses from all three search nodes, parses and normalizes the payload formats, filters out irrelevant results, and evaluates whether any content exists to process.

##### Nodes Involved
- `Combine the mentions`
- `Any mentions?`
- `Nothing this window`

##### Node Details

- **Combine the mentions**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
  - **Configuration:** Executes custom JavaScript to parse JSON payloads from Reddit, Hacker News, and Serper News. Normalizes fields (source, title, text snippet, score, URL, and formatted date) and slices the final array to a maximum of 20 items. Sets `hasContent` to `false` if total count equals zero.
  - **Expressions/Variables:** Reads upstream items via `$('Search Reddit').item.json`, `$('Search Hacker News').item.json`, `$('Search the news').item.json`, and `$('Set config').item.json`.
  - **Connections:** Input: `Search Reddit`, `Search Hacker News`, `Search the news`; Output: `Any mentions?`.
  - **Edge Cases/Failures:** Unstructured API responses throwing errors are caught via `try/catch` blocks, defaulting to empty arrays.

- **Any mentions?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Logical Branching)
  - **Configuration:** Evaluates whether `{{ $json.hasContent }}` evaluates to `true`.
  - **Expressions/Variables:** Evaluates boolean flag `{{ $json.hasContent }}`.
  - **Connections:** Input: `Combine the mentions`; True Output: `Analyse the mentions`; False Output: `Nothing this window`.
  - **Edge Cases/Failures:** Type validation issues if upstream data structures change.

- **Nothing this window**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Terminal State Assignment)
  - **Configuration:** Assigns an informational status string confirming zero mentions were found during the current monitoring window.
  - **Expressions/Variables:** None.
  - **Connections:** Input: `Any mentions?` (False branch); Output: None (Terminal node).
  - **Edge Cases/Failures:** None.

---

#### Block 1.4: AI Sentiment & Crisis Intelligence

##### Overview
This block utilizes an OpenAI language model and a structured output parser to analyze collected brand and competitor mentions, returning categorized intelligence including sentiment scores, concerns, opportunities, and crisis levels.

##### Nodes Involved
- `Analyse the mentions`
- `Chat model`
- `Output Parser - Intelligence`

##### Node Details

- **Analyse the mentions**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
  - **Configuration:** Configures a system prompt instructing the AI to act as a brand intelligence analyst. Enforces strict JSON compliance for 10 evaluation fields (`overallSentiment`, `crisisLevel`, `brandSentimentScore`, `competitorSentimentScore`, `topConcerns`, `topPraises`, `competitorWeaknesses`, `opportunitySignals`, `crisisIndicators`, `actionRecommendation`).
  - **Expressions/Variables:** Uses `{{ $json.brandName }}`, `{{ $json.competitor1Name }}`, `{{ $json.competitor2Name }}`, `{{ $json.mentionCount }}`, and `{{ $json.mentionsText }}`.
  - **Connections:** Input: `Any mentions?` (True branch), `Chat model` (AI Language Model), `Output Parser - Intelligence` (AI Output Parser); Output: `Build the alert`.
  - **Edge Cases/Failures:** LLM provider timeouts, rate limits, or output validation errors if the model fails to adhere to the schema.

- **Chat model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Provider)
  - **Configuration:** Configures OpenAI model `gpt-4o-mini` with `temperature: 0.2` and `maxTokens: 1000`.
  - **Credentials:** Uses OpenAI API credentials (`openAiApi`).
  - **Connections:** Output: Connected to `Analyse the mentions`.
  - **Edge Cases/Failures:** Invalid API keys, billing limits, or upstream API outages.

- **Output Parser - Intelligence**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (Schema Enforcement)
  - **Configuration:** Enforces a manual JSON schema defining properties and data types for the 10 required intelligence output fields.
  - **Connections:** Output: Connected to `Analyse the mentions`.
  - **Edge Cases/Failures:** Strict parser enforcement will reject non-compliant model responses.

---

#### Block 1.5: Persistence & Conditional Alerting

##### Overview
This block compiles analysis results and mention data, logs execution metrics and intelligence data permanently into Google Sheets, evaluates whether crisis thresholds are met, and conditionally dispatches email alerts via Gmail.

##### Nodes Involved
- `Build the alert`
- `Log to the sheet`
- `Crisis level?`
- `Email the alert`
- `All calm, log only`

##### Node Details

- **Build the alert**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation)
  - **Configuration:** Processes raw AI output and upstream mention variables. Constructs a formatted plain-text crisis alert email body and prepares normalized fields for Google Sheets row insertion. Evaluates whether `crisisLevel` is `Urgent` or `Critical` to set the `isCrisis` boolean flag.
  - **Expressions/Variables:** Reads `$input.first().json.output` and `$('Combine the mentions').item.json`.
  - **Connections:** Input: `Analyse the mentions`; Output: `Log to the sheet`.
  - **Edge Cases/Failures:** Throws an explicit error if the AI analysis object is missing or null.

- **Log to the sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Data Persistence)
  - **Configuration:** Appends a new row to the specified Google Sheet and tab (`Brand Intelligence`). Maps columns (`Date`, `Brand`, `Mentions Total`, `Reddit`, `HackerNews`, `News`, `Sentiment`, `Crisis Level`, `Brand Score`, `Competitor Score`, `Top Concern`, `Top Opportunity`, `Action Taken`, `Logged At`) using user-entered cell formatting (`USER_ENTERED`).
  - **Credentials:** Uses Google Sheets OAuth2 credentials.
  - **Expressions/Variables:** Uses dynamic column mappings sourced from `{{ $json.today }}`, `{{ $json.brandName }}`, `{{ $json.mentionCount }}`, etc.
  - **Connections:** Input: `Build the alert`; Output: `Crisis level?`.
  - **Edge Cases/Failures:** Google API permission errors, missing target spreadsheet IDs, or renamed spreadsheet tabs.

- **Crisis level?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Logical Branching)
  - **Configuration:** Evaluates whether `{{ $('Build the alert').item.json.isCrisis }}` is equal to `true`.
  - **Expressions/Variables:** Uses `{{ $('Build the alert').item.json.isCrisis }}`.
  - **Connections:** Input: `Log to the sheet`; True Output: `Email the alert`; False Output: `All calm, log only`.
  - **Edge Cases/Failures:** Boolean type mismatches.

- **Email the alert**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Email Integration)
  - **Configuration:** Sends an immediate email alert via Gmail containing the formatted crisis report. Sets recipient address, subject line, and message body. Disables attribution appending.
  - **Credentials:** Uses Gmail OAuth2 credentials.
  - **Expressions/Variables:** Uses `{{ $('Build the alert').item.json.recipientEmail }}`, `{{ $('Build the alert').item.json.alertEmailBody }}`, and `{{ $('Build the alert').item.json.alertEmailSubject }}`.
  - **Connections:** Input: `Crisis level?` (True branch); Output: None (Terminal node).
  - **Edge Cases/Failures:** Gmail API auth revocation, send quota limits, or invalid recipient email addresses.

- **All calm, log only**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Terminal State Assignment)
  - **Configuration:** Assigns a status string indicating monitoring is complete with no crisis detected, preventing inbox notifications.
  - **Expressions/Variables:** Uses `{{ $('Build the alert').item.json.crisisLevel }}` and `{{ $('Build the alert').item.json.mentionCount }}`.
  - **Connections:** Input: `Crisis level?` (False branch); Output: None (Terminal node).
  - **Edge Cases/Failures:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Check every 4 hours` | `n8n-nodes-base.scheduleTrigger` | Triggers execution every 4 hours | None | `Set config` | Runs every 4 hours automatically around the clock.<br><br>Change if needed:<br>- Every 2 hours for high-risk monitoring: 0 */2 * * *<br>- Every 6 hours for normal: 0 */6 * * *<br>- Every hour for crisis periods: 0 * * * *<br><br>No credentials needed. |
| `Set config` | `n8n-nodes-base.set` | Initializes configuration and variables | `Check every 4 hours` | `Search Reddit`<br>`Search Hacker News`<br>`Search the news` | REPLACE THESE 7 VALUES:<br><br>1. brandName<br>   Your brand or product name<br>   Example: Incrementors<br><br>2. competitor1Name<br>   First competitor to monitor<br>   Example: Backlinko<br><br>3. competitor2Name<br>   Second competitor (optional)<br>   Leave as BLANK if you only have one competitor to track<br>   Example: Ahrefs<br><br>4. serperApiKey<br>   Get free at serper.dev<br>   Free tier: 2500 searches per month<br>   This workflow uses 1 search per run<br><br>5. recipientEmail<br>   Who receives crisis alerts<br>   Example: you@example.com<br><br>6. senderName<br>   Your name for email sign-offs<br><br>7. sheetId<br>   Open your Google Sheet in browser<br>   URL: docs.google.com/spreadsheets/d/1ABC123/edit<br>   Copy the part between /d/ and /edit<br>   Create tab: Brand Intelligence<br>   Row 1 headers:<br>   Date \| Brand \| Mentions Total \| Reddit \| HackerNews \| News \| Sentiment \| Crisis Level \| Brand Score \| Competitor Score \| Top Concern \| Top Opportunity \| Action Taken \| Logged At |
| `Search Reddit` | `n8n-nodes-base.httpRequest` | Fetches recent Reddit mentions via public API | `Set config` | `Combine the mentions` | Searches Reddit for mentions of your brand AND competitor using the free public JSON API.<br><br>Endpoint: reddit.com/search.json<br>No API key or account required — fully public.<br>Returns 25 most recent posts from the past week.<br><br>User-Agent header is required by Reddit to prevent immediate rate limiting.<br><br>If Reddit rate limits the request (429 error), the Code node handles it gracefully<br>and continues with HackerNews and News data.<br><br>No changes needed. |
| `Search Hacker News` | `n8n-nodes-base.httpRequest` | Fetches Hacker News mentions via Algolia API | `Set config` | `Combine the mentions` | Searches HackerNews for brand mentions using the free Algolia API.<br><br>Endpoint: hn.algolia.com/api/v1/search<br>No API key required — fully public and free.<br>Returns up to 20 recent posts and comments.<br><br>HackerNews is especially important for tech products and B2B services.<br>Developer communities often discuss software tools, agencies, and services here.<br><br>No changes needed. |
| `Search the news` | `n8n-nodes-base.httpRequest` | Searches Google News via Serper API | `Set config` | `Combine the mentions` | Searches Google News for brand and competitor mentions using Serper API.<br><br>The query targets review, complaint, and feedback articles from the past week.<br>This surfaces news articles, blog posts, and review sites.<br><br>tbs=qdr:w restricts results to the past 7 days.<br><br>Serper API: 2500 free searches/month.<br>This workflow uses 1 news search per run = up to 180 uses/month on 4-hour schedule.<br><br>No changes needed. |
| `Combine the mentions` | `n8n-nodes-base.code` | Merges and parses platform payloads | `Search Reddit`<br>`Search Hacker News`<br>`Search the news` | `Any mentions?` | Merges mentions from Reddit, HackerNews, and Google News into one package.<br><br>For each platform:<br>- Reddit: extracts post title, body text, score, subreddit, date<br>- HackerNews: extracts story title, comment text, points, date<br>- Google News: extracts article title, snippet, publication domain, date<br><br>Handles failures gracefully — if one source fails, others still work.<br>Formats up to 20 mentions for AI analysis.<br><br>Returns hasContent = false if no mentions found in this window.<br>No changes needed. |
| `Any mentions?` | `n8n-nodes-base.if` | Checks whether any mentions were found | `Combine the mentions` | `Analyse the mentions`<br>`Nothing this window` | TRUE — mentions found this window — run AI analysis<br>FALSE — no mentions found — end silently<br><br>No email is sent when there are no mentions.<br>This prevents noisy notifications on quiet monitoring windows.<br>No changes needed. |
| `Analyse the mentions` | `@n8n/n8n-nodes-langchain.agent` | AI agent analyzing sentiment and intelligence | `Any mentions?`<br>`Chat model`<br>`Output Parser - Intelligence` | `Build the alert` | AI agent analyzes all brand and competitor mentions.<br><br>Outputs 10 intelligence fields:<br>1. Overall sentiment (Positive/Negative/Mixed/Neutral)<br>2. Crisis level (None to Critical scale)<br>3. Brand sentiment score (1-10 for YOUR brand)<br>4. Competitor sentiment score (1-10 for competitors)<br>5. Top concerns about YOUR brand<br>6. Top praises about YOUR brand<br>7. Competitor weaknesses (opportunities for you)<br>8. Opportunity signals (unmet needs found)<br>9. Crisis indicators (specific quotes if urgent)<br>10. One specific action recommendation<br><br>All fields enforced by Brand Intel Parser below.<br>Output is in $json.output |
| `Chat model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides OpenAI GPT-4o-mini model | None | `Analyse the mentions` | Connect your OpenAI API credential here.<br><br>1. Go to platform.openai.com/api-keys<br>2. Create a new key<br>3. In n8n: Settings → Credentials → Add New → OpenAI API<br>4. Connect to this node<br><br>temperature: 0.2 for precise, consistent analysis<br>Cost: about $0.002 per monitoring run |
| `Output Parser - Intelligence` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema for AI output | None | `Analyse the mentions` | Schema: 10 required fields for brand intelligence.<br>Ensures AI always returns clean consistent data.<br>No changes needed. |
| `Build the alert` | `n8n-nodes-base.code` | Formats email body and logging data | `Analyse the mentions` | `Log to the sheet` | Reads AI output from $json.output.<br>Reads mention data from Merge All Platform Mentions directly.<br><br>Sets routing flags:<br>- isCrisis = true when crisisLevel is Urgent or Critical<br>- hasOpportunities = true when competitor weaknesses were found<br><br>Builds the crisis alert email body with all intelligence sections.<br>Prepares clean fields for Google Sheets logging.<br><br>No changes needed. |
| `Log the sheet` | `n8n-nodes-base.googleSheets` | Appends monitoring data to Google Sheets | `Build the alert` | `Crisis level?` | Logs every monitoring run to Google Sheets permanently.<br><br>How to connect:<br>1. Go to console.cloud.google.com and enable Google Sheets API<br>2. In n8n: Settings → Credentials → Add New → Google Sheets OAuth2<br>3. Authenticate and connect to this node<br><br>Sheet tab: Brand Intelligence<br>Row 1 headers:<br>Date \| Brand \| Mentions Total \| Reddit \| HackerNews \| News \| Sentiment \| Crisis Level \| Brand Score \| Competitor Score \| Top Concern \| Top Opportunity \| Action Taken \| Logged At<br><br>Runs 6 times per day (every 4 hours) — up to 42 rows per week.<br>Filter by Crisis Level to see all past alerts.<br>Chart Brand Score over time to spot reputation trends.<br>Compare Brand Score vs Competitor Score weekly. |
| `Crisis level?` | `n8n-nodes-base.if` | Determines if crisis alert email is required | `Log to the sheet` | `Email the alert`<br>`All calm, log only` | TRUE — crisis level is Urgent or Critical — send immediate email alert<br>FALSE — no crisis — end silently<br><br>Reads isCrisis from Process Intelligence node directly.<br><br>This means you only get emailed when something actually needs attention.<br>Normal monitoring runs produce zero inbox noise.<br>No changes needed. |
| `Email the alert` | `n8n-nodes-base.gmail` | Sends crisis alert email via Gmail | `Crisis level?` | None | Sends immediate crisis alert when Urgent or Critical situation detected.<br><br>How to connect:<br>1. Go to console.cloud.google.com and enable the Gmail API<br>2. In n8n: Settings → Credentials → Add New → Gmail OAuth2<br>3. Authenticate and connect to this node<br><br>Alert email includes:<br>- Crisis level and detection time<br>- Mention breakdown by platform<br>- Brand score vs competitor score<br>- Specific crisis indicators (quotes from the mentions)<br>- Top concerns about your brand<br><br>- One specific recommended action<br>- Competitor weaknesses discovered<br><br>Typically arrives within 60-90 seconds of detection. |
| `All calm, log only` | `n8n-nodes-base.set` | Sets status when no crisis is detected | `Crisis level?` | None | FALSE branch — no crisis detected this monitoring window.<br>All mentions are logged to Google Sheets.<br>No email sent — your inbox stays clean.<br>Workflow ends silently.<br>No changes needed. |
| `Nothing this window` | `n8n-nodes-base.set` | Sets status when no mentions are found | `Any mentions?` | None | FALSE branch from Mentions Found? node.<br>No brand or competitor mentions were found across any platform.<br>Workflow ends silently — no logging, no email.<br>This is normal during quiet periods.<br>No changes needed. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to Cron Expression: `0 */4 * * *`.

2. **Configure Variables:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Configure assignments for: `brandName` (string), `competitor1Name` (string), `competitor2Name` (string), `serperApiKey` (string), `recipientEmail` (string), `senderName` (string), `sheetId` (string), `sheetName` (`Brand Intelligence`), `combinedQuery` (`={{ $json.brandName }}`), and `today` (`={{ $now.toFormat('dd MMM yyyy HH:mm') }}`).
   - Connect `Schedule Trigger` output to this node.

3. **Set Up Parallel Search Requests:**
   - **Reddit Search (`HTTP Request`):** Method `GET`, URL `=https://www.reddit.com/search.json?q={{ encodeURIComponent($json.brandName + ' OR ' + $json.competitor1Name) }}&sort=new&t=week&limit=25&type=link`. Set headers `User-Agent: n8n-brand-monitor/1.0`. Enable `onError: continueRegularOutput`.
   - **Hacker News Search (`HTTP Request`):** Method `GET`, URL `=https://hn.algolia.com/api/v1/search?query={{ $('Set config').item.json.brandName }}&tags=comment,story&hitsPerPage=20`. Enable `onError: continueRegularOutput`.
   - **Google News Search (`HTTP Request`):** Method `POST`, URL `https://google.serper.dev/news`. Add headers `X-API-KEY` (mapped to config) and `Content-Type: application/json`. Set JSON body targeting reviews/complaints. Enable `onError: continueRegularOutput`.
   - Connect `Set config` output to all three HTTP Request nodes.

4. **Aggregate and Filter Mentions:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Combine the mentions`. Paste JavaScript logic to parse Reddit (`data.children`), Hacker News (`hits`), and Serper News (`news`), combining them into an array of up to 20 formatted items and setting `hasContent`. Connect outputs from all three search nodes into this Code node.
   - Add an **If** node (`n8n-nodes-base.if`) named `Any mentions?`. Set condition to check if `{{ $json.hasContent }}` equals `true`. Connect `Combine the mentions` to this node.
   - Add a **Set** node named `Nothing this window` for the false branch of `Any mentions?`.

5. **Configure AI Intelligence Agent:**
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`) connected to the true branch of `Any mentions?`.
   - Configure prompt to evaluate sentiment, crisis level (`None`, `Low`, `Moderate`, `Urgent`, `Critical`), sentiment scores (1-10), concerns, praises, competitor weaknesses, opportunities, and action recommendations.
   - Attach a **Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with model `gpt-4o-mini`, temperature `0.2`, and OpenAI API credentials.
   - Attach a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) defining the JSON schema with all 10 required intelligence properties.

6. **Process and Log Results:**
   - Add a **Code** node named `Build the alert` to read AI output (`$input.first().json.output`), build email bodies, and establish the `isCrisis` boolean flag (`Urgent` or `Critical`). Connect `Analyse the mentions` to this node.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) configured to `append` operation. Map document ID, sheet name, and column mappings (`Date`, `Brand`, `Mentions Total`, `Reddit`, `HackerNews`, `News`, `Sentiment`, `Crisis Level`, `Brand Score`, `Competitor Score`, `Top Concern`, `Top Opportunity`, `Action Taken`, `Logged At`) using Google Sheets OAuth2 credentials. Connect `Build the alert` to this node.

7. **Conditional Alerting:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Crisis level?`. Set condition to check if `{{ $('Build the alert').item.json.isCrisis }}` equals `true`. Connect `Log to the sheet` to this node.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) on the true branch. Configure recipient (`={{ $('Build the alert').item.json.recipientEmail }}`, subject, and message body using Gmail OAuth2 credentials.
   - Add a **Set** node named `All calm, log only` on the false branch to conclude non-crisis runs silently.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Serper API Free Tier Limits | Get a free API key at [serper.dev](https://serper.dev) (provides 2,500 searches per month). |
| Reddit API Rate Limiting | Reddit public JSON API requires a distinct `User-Agent` header to prevent immediate HTTP 429 rate limit responses. |
| Google Cloud Console Setup | Requires Google Sheets API and Gmail API enabled in the [Google Cloud Console](https://console.cloud.google.com) for OAuth2 authentication. |