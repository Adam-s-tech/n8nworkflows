Detect competitor SEO strategy shifts with Claude and WhatsApp alerts

https://n8nworkflows.xyz/workflows/detect-competitor-seo-strategy-shifts-with-claude-and-whatsapp-alerts-20056


# Detect competitor SEO strategy shifts with Claude and WhatsApp alerts

### 1. Workflow Overview

This workflow automates competitor SEO surveillance by crawling competitor websites, snapshotting site structures, calculating architectural and content differences against previous runs, and using Anthropic Claude to interpret strategic intent. It delivers real-time WhatsApp alerts for significant changes and aggregates a multi-competitor summary digest.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration**: Handles triggers (schedule, webhook, manual test), normalizes incoming payloads, and establishes execution limits and parameters.
- **1.2 Competitor Queue Management**: Determines whether the run targets a single ad-hoc competitor or performs a full batch sweep of all tracked competitors fetched from a historical database.
- **1.3 Site Crawling & Snapshot Engine**: Iterates through queued pages up to configured depth and page limits, respects a polite crawl delay, parses HTML structures (titles, H1s, schema, links, word counts), and generates content hashes.
- **1.4 Diff Engine & Storage**: Retrieves historical snapshots, computes structural shifts across seven distinct categories (new clusters, deleted pages, migrations, title updates, schema variations, link shifts, content edits), and persists current snapshots.
- **1.5 AI Strategy Analysis & Alerting**: Passes structured diff metrics to Anthropic Claude to generate a qualitative narrative, confidence rating, and counter-actions, subsequently firing WhatsApp alerts for major shifts and compiling a final execution digest.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
**Overview:** Captures entry events from multiple sources, standardizes the execution payload, and injects runtime configurations, limits, and API endpoints.

**Nodes Involved:**
- Schedule Trigger - Daily/Weekly Competitor Crawl
- Webhook - Register Competitor
- Manual Trigger - Test Run
- Code 1 - Normalize Trigger Input
- Set - Config
- Code 2 - Determine Trigger Mode
- IF - Ad Hoc Competitor Provided

**Node Details:**
- **Schedule Trigger - Daily/Weekly Competitor Crawl**
  - Type: `n8n-nodes-base.scheduleTrigger`
  - Role: Initiates scheduled execution every 24 hours.
  - Configuration: Interval set to 24 hours.
  - Connections: Output connects to `Code 1 - Normalize Trigger Input`.
  - Edge Cases: Missed runs if n8n instance is offline.
- **Webhook - Register Competitor**
  - Type: `n8n-nodes-base.webhook`
  - Role: Receives external POST requests to immediately track and crawl a specific competitor.
  - Configuration: Path set to `competitor-strategy-register`, method `POST`, response mode `responseNode`.
  - Connections: Output connects to `Code 1 - Normalize Trigger Input`.
  - Edge Cases: Invalid JSON payloads or unauthorized ingestion.
- **Manual Trigger - Test Run**
  - Type: `n8n-nodes-base.manualTrigger`
  - Role: Allows manual execution for testing.
  - Configuration: Default interactive run.
  - Connections: Output connects to `Code 1 - Normalize Trigger Input`.
- **Code 1 - Normalize Trigger Input**
  - Type: `n8n-nodes-base.code`
  - Role: Standardizes webhook body or empty manual/scheduled payloads into a unified object shape, applying default sample data for manual tests.
  - Key Expressions: Reads `$input.first().json.body || $input.first().json`.
  - Connections: Input from triggers; output connects to `Set - Config`.
- **Set - Config**
  - Type: `n8n-nodes-base.set`
  - Role: Establishes global configuration parameters (crawl delay, depth limits, database endpoints, Anthropic model, WhatsApp credentials).
  - Configuration: Defines integer thresholds (`crawlDelaySeconds: 3`, `significantChangeCountThreshold: 5`, etc.) and string endpoints.
  - Connections: Input from Code 1; output connects to `Code 2 - Determine Trigger Mode`.
- **Code 2 - Determine Trigger Mode**
  - Type: `n8n-nodes-base.code`
  - Role: Evaluates whether a specific competitor domain was passed in the trigger.
  - Key Expressions: Sets `isAdHocCompetitor` boolean flag.
  - Connections: Input from Set - Config; output connects to `IF - Ad Hoc Competitor Provided`.
- **IF - Ad Hoc Competitor Provided**
  - Type: `n8n-nodes-base.if`
  - Role: Branches execution between single-competitor ad-hoc processing and a full batch sweep of tracked competitors.
  - Key Expressions: Evaluates `{{ $json.isAdHocCompetitor }}`.
  - Connections: True branch routes to `Code 3`; False branch routes to `HTTP Request - Fetch Tracked Competitors`.

---

#### 1.2 Competitor Queue Management
**Overview:** Assembles the array of competitor domains to process, either from the single ad-hoc payload or via a database query fetching all active accounts.

**Nodes Involved:**
- Code 3 - Build Competitor Queue From Trigger
- HTTP Request - Fetch Tracked Competitors
- Code 4 - Build Competitor Queue From Tracked
- Code 5 - Get Next Competitor From Queue
- IF - Competitor Queue Empty
- Code 11 - Build Final Strategy Digest
- WhatsApp - Send Strategy Digest
- Respond to Webhook - Return Digest

**Node Details:**
- **Code 3 - Build Competitor Queue From Trigger**
  - Type: `n8n-nodes-base.code`
  - Role: Wraps the ad-hoc competitor parameters into a single-item queue array.
  - Connections: Input from IF True; output connects to `Code 5 - Get Next Competitor From Queue`.
- **HTTP Request - Fetch Tracked Competitors**
  - Type: `n8n-nodes-base.httpRequest`
  - Role: Retrieves all active monitored competitor domains from the historical database API.
  - Configuration: Uses HTTP Header Auth, endpoint URL from config.
  - Connections: Input from IF False; output connects to `Code 4 - Build Competitor Queue From Tracked`.
  - Edge Cases: API authentication failure or timeout; utilizes `continueOnFail: true`.
- **Code 4 - Build Competitor Queue From Tracked**
  - Type: `n8n-nodes-base.code`
  - Role: Transforms the API response into a standardized competitor queue array.
  - Connections: Input from fetch request; output connects to `Code 5 - Get Next Competitor From Queue`.
- **Code 5 - Get Next Competitor From Queue**
  - Type: `n8n-nodes-base.code`
  - Role: Pops the next competitor off the queue and initializes tracking variables (page queue, visited pages, site snapshot). Acts as the main loop recurrence point.
  - Connections: Input from queue builders or alert loops; output connects to `IF - Competitor Queue Empty`.
- **IF - Competitor Queue Empty**
  - Type: `n8n-nodes-base.if`
  - Role: Checks if all competitors have been processed or if the maximum run limit (`maxCompetitorsPerRun`) has been reached.
  - Key Expressions: Evaluates `hasNextCompetitor` and processed count against run limits.
  - Connections: True branch routes to `Code 11 - Build Final Strategy Digest`; False branch routes to `Code 6 - Get Next Page From Queue`.
- **Code 11 - Build Final Strategy Digest**
  - Type: `n8n-nodes-base.code`
  - Role: Aggregates metrics from all processed competitor reports into a summary object.
  - Connections: Input from IF (queue empty); output connects to `WhatsApp - Send Strategy Digest`.
- **WhatsApp - Send Strategy Digest**
  - Type: `n8n-nodes-base.whatsApp`
  - Role: Sends the execution summary digest via WhatsApp.
  - Configuration: Operation `send`, uses dynamic phone ID and recipient settings.
  - Connections: Input from Code 11; output connects to `Respond to Webhook - Return Digest`.
- **Respond to Webhook - Return Digest**
  - Type: `n8n-nodes-base.respondToWebhook`
  - Role: Returns the execution digest JSON payload to the original webhook caller.
  - Configuration: Responds with JSON.
  - Connections: Input from WhatsApp digest node.

---

#### 1.3 Site Crawling & Snapshot Engine
**Overview:** Recursively crawls a target competitor website page-by-page, respecting depth constraints, extraction delays, and page budgets.

**Nodes Involved:**
- Code 6 - Get Next Page From Queue
- IF - Page Queue Empty Or Page Budget Reached
- Wait - Crawl Delay
- HTTP Request - Fetch Page HTML
- Code 7 - Parse Page
- Code 8 - Update Page Queue & Site Snapshot

**Node Details:**
- **Code 6 - Get Next Page From Queue**
  - Type: `n8n-nodes-base.code`
  - Role: Pops the next URL off the current competitor's page queue. Acts as the inner crawling loop recurrence point.
  - Connections: Input from queue check; output connects to `IF - Page Queue Empty Or Page Budget Reached`.
- **IF - Page Queue Empty Or Page Budget Reached**
  - Type: `n8n-nodes-base.if`
  - Role: Determines if site crawling should continue or conclude based on remaining pages or budget limits.
  - Connections: True branch routes to `HTTP Request - Fetch Previous Site Snapshot From Historical DB`; False branch routes to `Wait - Crawl Delay`.
- **Wait - Crawl Delay**
  - Type: `n8n-nodes-base.wait`
  - Role: Implements politeness pause between HTTP requests.
  - Configuration: Amount set to 3 seconds.
  - Connections: Input from IF; output connects to `HTTP Request - Fetch Page HTML`.
- **HTTP Request - Fetch Page HTML**
  - Type: `n8n-nodes-base.httpRequest`
  - Role: Fetches raw HTML content from the target URL.
  - Configuration: Response format set to text, custom User-Agent header applied.
  - Connections: Input from Wait node; output connects to `Code 7 - Parse Page`.
  - Edge Cases: 404/500 errors, timeouts; uses `continueOnFail: true`.
- **Code 7 - Parse Page**
  - Type: `n8n-nodes-base.code`
  - Role: Extracts SEO metadata (title, canonical, H1 tags, JSON-LD schema types, internal links, word count, and rolling content hash).
  - Connections: Input from HTML fetch; output connects to `Code 8 - Update Page Queue & Site Snapshot`.
- **Code 8 - Update Page Queue & Site Snapshot**
  - Type: `n8n-nodes-base.code`
  - Role: Adds newly discovered internal links to the queue (respecting depth rules) and appends the current page record to the site snapshot array.
  - Connections: Input from Parse Page; output loops back to `Code 6 - Get Next Page From Queue`.

---

#### 1.4 Diff Engine & Storage
**Overview:** Compares the freshly captured site snapshot against the historical record retrieved from the database to isolate strategic shifts, then persists the current snapshot.

**Nodes Involved:**
- HTTP Request - Fetch Previous Site Snapshot From Historical DB
- Code 9 - Diff Engine: Compute Site Changes
- HTTP Request - Save Current Site Snapshot To Historical DB

**Node Details:**
- **HTTP Request - Fetch Previous Site Snapshot From Historical DB**
  - Type: `n8n-nodes-base.httpRequest`
  - Role: Retrieves the most recent historical snapshot for the competitor.
  - Configuration: Uses HTTP Header Auth, passes competitor domain as a query parameter.
  - Connections: Input from page crawl completion; output connects to `Code 9 - Diff Engine: Compute Site Changes`.
  - Edge Cases: Missing historical snapshot on first crawl; handled with `continueOnFail: true`.
- **Code 9 - Diff Engine: Compute Site Changes**
  - Type: `n8n-nodes-base.code`
  - Role: Computes site-wide structural deltas across seven categories: new content clusters, deleted pages, URL migrations (content hash matching), title changes, schema type alterations, internal link volume shifts, and content edits.
  - Connections: Input from historical fetch; output connects to `HTTP Request - Save Current Site Snapshot To Historical DB`.
- **HTTP Request - Save Current Site Snapshot To Historical DB**
  - Type: `n8n-nodes-base.httpRequest`
  - Role: Persists the current crawl's full site snapshot to the historical database.
  - Configuration: POST request, JSON body containing domain, timestamp, and page records.
  - Connections: Input from Diff Engine; output connects to `API 1 - Strategy Agent (Claude)`.
  - Edge Cases: Database write failure; uses `continueOnFail: true`.

---

#### 1.5 AI Strategy Analysis & Alerting
**Overview:** Feeds computed diff data to Anthropic Claude to determine qualitative strategic intent, evaluates significance thresholds, dispatches WhatsApp alerts, and loops to the next competitor.

**Nodes Involved:**
- API 1 - Strategy Agent (Claude)
- Code 10 - Parse Strategy Agent Response & Build Competitor Report
- IF - Significant Strategy Change Detected
- WhatsApp - Send Strategy Alert

**Node Details:**
- **API 1 - Strategy Agent (Claude)**
  - Type: `n8n-nodes-base.httpRequest`
  - Role: Queries Anthropic Claude API (`claude-sonnet-4-6`) to interpret strategic shifts and suggest counter-actions.
  - Configuration: POST request to `https://api.anthropic.com/v1/messages`, generic HTTP Header Auth, max tokens 1200, includes system prompt enforcing strict JSON output.
  - Connections: Input from DB save; output connects to `Code 10 - Parse Strategy Agent Response & Build Competitor Report`.
  - Edge Cases: API rate limits or malformed AI output; configured with `onError: continueErrorOutput`, retry logic, and fallback parsing.
- **Code 10 - Parse Strategy Agent Response & Build Competitor Report**
  - Type: `n8n-nodes-base.code`
  - Role: Safely parses Claude’s JSON response, calculates total change volume, evaluates significance against thresholds, and builds the competitor report object.
  - Connections: Input from Claude API; output connects to `IF - Significant Strategy Change Detected`.
- **IF - Significant Strategy Change Detected**
  - Type: `n8n-nodes-base.if`
  - Role: Evaluates whether the change volume and AI confidence clear the alerting thresholds.
  - Key Expressions: Evaluates `{{ $json.significantStrategyChange }}`.
  - Connections: True branch routes to `WhatsApp - Send Strategy Alert`; False branch loops back to `Code 5 - Get Next Competitor From Queue`.
- **WhatsApp - Send Strategy Alert**
  - Type: `n8n-nodes-base.whatsApp`
  - Role: Sends an instant WhatsApp notification detailing the competitor's strategic shift, confidence score, narrative, and recommended counter-actions.
  - Configuration: Operation `send`, uses dynamic phone ID and recipient settings.
  - Connections: Input from IF True; output loops back to `Code 5 - Get Next Competitor From Queue`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Workflow documentation and setup guide. | None | None | ## Competitor Strategy Change Detector<br>Don't just watch a competitor's rankings - watch what they're actually doing. Each cycle this crawls a competitor's whole site (up to a page budget), builds a full site snapshot (per-page title, canonical, headings, schema types, internal link count, word count, and a content hash), and diffs it against the last stored snapshot. The Diff Engine surfaces seven kinds of change at once - new content clusters, deleted pages, URL migrations, internal-link changes, title changes, schema changes, and content updates - and a Strategy Agent reads all of it together to answer "why did they do this?": the underlying strategic move, not just a list of diffs, plus recommended counter-actions. Significant strategic shifts trigger a WhatsApp alert; every run ends with a digest across all competitors checked.<br><br>### How to set up<br>1. Import this workflow into n8n<br>2. Add credentials: your historical database API (a simple store keyed by competitor domain + timestamp, holding a full page list per snapshot), Anthropic, and WhatsApp Business Cloud<br>3. Edit Set - Config: crawlDelaySeconds, defaultMaxPagesPerCompetitor, defaultMaxDepthPerCompetitor, maxCompetitorsPerRun, the change-detection thresholds, the database endpoints, the Anthropic model, and your WhatsApp sender + recipient<br>4. POST {competitorDomain, startUrl, maxPages, maxDepth} to Webhook - Register Competitor to start tracking a competitor immediately<br>5. Leave Schedule Trigger - Daily/Weekly Competitor Crawl active to re-crawl every tracked competitor on a cadence (edit the interval on the node itself)<br>6. Use the Manual Trigger to test end to end with a sample competitor<br><br>### How to customize<br>- Swap the URL-migration matching in Code 9 (currently exact content-hash match) for a fuzzy title/content-similarity match to catch pages that changed slightly during a move<br>- Add a schema.org type registry to Code 9 so schema changes are categorized (e.g. "added FAQPage", "dropped Product") rather than just flagged<br>- Feed the Strategy Agent a rolling history of past reports for this competitor for a longer-horizon read ("this is their third pillar-page push this quarter")<br>- Swap the WhatsApp alert for Slack/email, or add both |
| Sticky Note - Stage 1 | `n8n-nodes-base.stickyNote` | Stage 1 documentation covering ingestion, configuration, and queue management. | None | None | ## 1. Competitor<br><br>- **Schedule Trigger - Daily/Weekly Competitor Crawl**: fires on the configured interval to re-crawl every already-tracked competitor.<br>- **Webhook - Register Competitor**: receives {competitorDomain, startUrl, maxPages, maxDepth} to start tracking a competitor and crawl it immediately.<br>- **Manual Trigger - Test Run**: runs the workflow by hand for testing.<br>- **Code 1 - Normalize Trigger Input**: reads whichever payload came in (webhook body, or nothing for schedule/manual) and extracts the trigger competitor's fields; defaults to a sample competitor when the payload is empty so manual runs are testable end to end.<br>- **Set - Config**: defines crawlDelaySeconds, defaultMaxPagesPerCompetitor, defaultMaxDepthPerCompetitor, maxCompetitorsPerRun, internalLinkDeltaThreshold, wordCountDeltaThreshold, significantChangeCountThreshold, strategyConfidenceThreshold, the database endpoints, the Anthropic model, and the WhatsApp sender + recipient.<br>- **Code 2 - Determine Trigger Mode**: flags isAdHocCompetitor - true when a single competitor came in via webhook/manual, false when this is a full scheduled sweep.<br>- **IF - Ad Hoc Competitor Provided**: true (single competitor) routes to Code 3; false (full sweep) routes to the tracked-competitors fetch.<br>- **Code 3 - Build Competitor Queue From Trigger**: builds a one-item competitor queue from the ad-hoc payload.<br>- **HTTP Request - Fetch Tracked Competitors**: (false branch only) calls the historical database's tracked-competitors endpoint to get every competitor under active monitoring.<br>- **Code 4 - Build Competitor Queue From Tracked**: turns that list into the same competitor queue shape.<br>- **Code 5 - Get Next Competitor From Queue**: pops one competitor off the outer queue, and resets the inner per-site crawl state (page queue seeded with that competitor's start URL, visited pages cleared, site snapshot cleared) - the loop-back point every subsequent competitor returns to.<br>- **IF - Competitor Queue Empty**: true when the competitor queue is empty or maxCompetitorsPerRun has been hit, ending the run and building the digest; false continues into that competitor's site crawl.<br>- **Code 11 - Build Final Strategy Digest**: once the run ends (competitor queue empty or the run's competitor cap hit), rolls every competitor's report into one summary - competitors checked, how many showed significant strategic change, and total change counts per category across all of them.<br>- **WhatsApp - Send Strategy Digest**: sends that summary.<br>- **Respond to Webhook - Return Digest**: returns the digest as the webhook's JSON response (a harmless no-op on schedule-triggered runs, which have no webhook to respond to). |
| Sticky Note - Stage 2 | `n8n-nodes-base.stickyNote` | Stage 2 documentation covering the site crawl and diff engine. | None | None | ## 2. Daily/Weekly Crawl<br><br>- **Code 6 - Get Next Page From Queue**: pops one page URL off the current competitor's page queue each cycle - the loop-back point for every subsequent page.<br>- **IF - Page Queue Empty Or Page Budget Reached**: true when there are no more queued pages or this competitor's page budget is used up, moving on to the Diff Engine; false continues crawling.<br>- **Wait - Crawl Delay**: a short politeness pause before the next fetch.<br>- **HTTP Request - Fetch Page HTML**: pulls the raw page.<br>- **Code 7 - Parse Page**: extracts title, canonical, H1s, JSON-LD schema types, internal links, word count, and a lightweight content hash (a simple rolling hash of the visible text, used to spot moved or rewritten pages).<br>- **Code 8 - Update Page Queue & Site Snapshot**: marks the page visited, queues newly-discovered internal links (within maxDepth and not already visited/queued), and appends this page's record to the competitor's growing site snapshot.<br>- **HTTP Request - Fetch Previous Site Snapshot From Historical DB**: pulls the last full site snapshot stored for this competitor.<br>- **Code 9 - Diff Engine: Compute Site Changes**: compares the freshly crawled site against that previous snapshot and detects all seven change types at once - new pages are grouped into **newContentClusters** by URL path prefix once 2+ share one; removed URLs whose content hash matches a new URL are reclassified as **urlMigrations** rather than **deletedPages**; matching URLs get compared for **titleChanges**, **schemaChanges** (JSON-LD types added/removed), **internalLinkChanges** (delta beyond threshold), and **contentUpdates** (hash changed with the title unchanged).<br>- **HTTP Request - Save Current Site Snapshot To Historical DB**: persists this crawl's full site snapshot so the next run has something to diff against. |
| Sticky Note - Stage 4 | `n8n-nodes-base.stickyNote` | Stage 4 documentation covering AI strategy analysis and alerting. | None | None | ## 4. Strategy Agent - "Why did they do this?"<br><br>- **API 1 - Strategy Agent (Claude)**: given every category of change the Diff Engine found, reasons about what the competitor likely did strategically in each category, then gives one overall narrative answering "why did they do this?" - the underlying strategic shift behind the changes, not just a list of diffs - plus a confidence score and recommended counter-actions.<br>- **Code 10 - Parse Strategy Agent Response & Build Competitor Report**: parses that JSON response (with a safe fallback if parsing fails), tallies change counts per category, marks significantStrategyChange true only when the total change volume and the agent's confidence both clear their thresholds, and assembles the full per-competitor report.<br>- **IF - Significant Strategy Change Detected**: true routes to the WhatsApp alert first; false skips straight back to the loop.<br>- **WhatsApp - Send Strategy Alert**: sends the competitor, the overall narrative, confidence, and recommended counter-actions. Both branches of the IF then loop back to Code 5 for the next competitor. |
| Schedule Trigger - Daily/Weekly Competitor Crawl | `n8n-nodes-base.scheduleTrigger` | Fires scheduled daily competitor crawl runs. | None | Code 1 - Normalize Trigger Input | Sticky Note - Stage 1 |
| Webhook - Register Competitor | `n8n-nodes-base.webhook` | Receives external POST requests to register and track a competitor. | None | Code 1 - Normalize Trigger Input | Sticky Note - Stage 1 |
| Manual Trigger - Test Run | `n8n-nodes-base.manualTrigger` | Initiates manual test runs. | None | Code 1 - Normalize Trigger Input | Sticky Note - Stage 1 |
| Code 1 - Normalize Trigger Input | `n8n-nodes-base.code` | Normalizes trigger payloads and applies default test values. | Schedule Trigger, Webhook, Manual Trigger | Set - Config | Sticky Note - Stage 1 |
| Set - Config | `n8n-nodes-base.set` | Sets global configuration variables, thresholds, and endpoints. | Code 1 - Normalize Trigger Input | Code 2 - Determine Trigger Mode | Sticky Note - Stage 1 |
| Code 2 - Determine Trigger Mode | `n8n-nodes-base.code` | Flags whether the execution targets an ad-hoc competitor. | Set - Config | IF - Ad Hoc Competitor Provided | Sticky Note - Stage 1 |
| IF - Ad Hoc Competitor Provided | `n8n-nodes-base.if` | Branches between ad-hoc single competitor or full tracked sweep. | Code 2 - Determine Trigger Mode | Code 3, HTTP Request - Fetch Tracked Competitors | Sticky Note - Stage 1 |
| Code 3 - Build Competitor Queue From Trigger | `n8n-nodes-base.code` | Wraps ad-hoc competitor details into queue array format. | IF - Ad Hoc Competitor Provided (True) | Code 5 - Get Next Competitor From Queue | Sticky Note - Stage 1 |
| HTTP Request - Fetch Tracked Competitors | `n8n-nodes-base.httpRequest` | Fetches active monitored competitors list from historical DB. | IF - Ad Hoc Competitor Provided (False) | Code 4 - Build Competitor Queue From Tracked | Sticky Note - Stage 1 |
| Code 4 - Build Competitor Queue From Tracked | `n8n-nodes-base.code` | Transforms tracked competitor list into queue array format. | HTTP Request - Fetch Tracked Competitors | Code 5 - Get Next Competitor From Queue | Sticky Note - Stage 1 |
| Code 5 - Get Next Competitor From Queue | `n8n-nodes-base.code` | Pops next competitor from queue and initializes crawl state. | Code 3, Code 4, WhatsApp Alert, IF Significant Change (False) | IF - Competitor Queue Empty | Sticky Note - Stage 1 |
| IF - Competitor Queue Empty | `n8n-nodes-base.if` | Checks if competitor queue is exhausted or run limit reached. | Code 5 - Get Next Competitor From Queue | Code 11 - Build Final Strategy Digest, Code 6 - Get Next Page From Queue | Sticky Note - Stage 1 |
| Code 6 - Get Next Page From Queue | `n8n-nodes-base.code` | Pops next page URL from current competitor's crawl queue. | IF - Competitor Queue Empty (False), Code 8 | IF - Page Queue Empty Or Page Budget Reached | Sticky Note - Stage 2 |
| IF - Page Queue Empty Or Page Budget Reached | `n8n-nodes-base.if` | Checks if page crawl queue is empty or page budget is met. | Code 6 - Get Next Page From Queue | HTTP Request - Fetch Previous Snapshot, Wait - Crawl Delay | Sticky Note - Stage 2 |
| Wait - Crawl Delay | `n8n-nodes-base.wait` | Pauses execution between page requests for politeness. | IF - Page Queue Empty Or Page Budget Reached (False) | HTTP Request - Fetch Page HTML | Sticky Note - Stage 2 |
| HTTP Request - Fetch Page HTML | `n8n-nodes-base.httpRequest` | Fetches raw HTML content for the current page. | Wait - Crawl Delay | Code 7 - Parse Page | Sticky Note - Stage 2 |
| Code 7 - Parse Page | `n8n-nodes-base.code` | Extracts SEO tags, schema types, links, word counts, and hashes. | HTTP Request - Fetch Page HTML | Code 8 - Update Page Queue & Site Snapshot | Sticky Note - Stage 2 |
| Code 8 - Update Page Queue & Site Snapshot | `n8n-nodes-base.code` | Appends page record to snapshot and queues internal links. | Code 7 - Parse Page | Code 6 - Get Next Page From Queue | Sticky Note - Stage 2 |
| HTTP Request - Fetch Previous Site Snapshot From Historical DB | `n8n-nodes-base.httpRequest` | Retrieves previous site snapshot from historical database. | IF - Page Queue Empty Or Page Budget Reached (True) | Code 9 - Diff Engine: Compute Site Changes | Sticky Note - Stage 2 |
| Code 9 - Diff Engine: Compute Site Changes | `n8n-nodes-base.code` | Computes site-wide structural deltas across seven change types. | HTTP Request - Fetch Previous Site Snapshot From Historical DB | HTTP Request - Save Current Site Snapshot To Historical DB | Sticky Note - Stage 2 |
| HTTP Request - Save Current Site Snapshot To Historical DB | `n8n-nodes-base.httpRequest` | Saves current site snapshot to historical database. | Code 9 - Diff Engine: Compute Site Changes | API 1 - Strategy Agent (Claude) | Sticky Note - Stage 2 |
| API 1 - Strategy Agent (Claude) | `n8n-nodes-base.httpRequest` | Queries Anthropic Claude to analyze strategic intent. | HTTP Request - Save Current Site Snapshot To Historical DB | Code 10 - Parse Strategy Agent Response & Build Competitor Report | Sticky Note - Stage 4 |
| Code 10 - Parse Strategy Agent Response & Build Competitor Report | `n8n-nodes-base.code` | Parses AI JSON output and builds competitor strategy report. | API 1 - Strategy Agent (Claude) | IF - Significant Strategy Change Detected | Sticky Note - Stage 4 |
| IF - Significant Strategy Change Detected | `n8n-nodes-base.if` | Evaluates if change volume and AI confidence meet alerting thresholds. | Code 10 - Parse Strategy Agent Response & Build Competitor Report | WhatsApp - Send Strategy Alert, Code 5 - Get Next Competitor From Queue | Sticky Note - Stage 4 |
| WhatsApp - Send Strategy Alert | `n8n-nodes-base.whatsApp` | Sends WhatsApp alert for significant competitor strategy shifts. | IF - Significant Strategy Change Detected (True) | Code 5 - Get Next Competitor From Queue | Sticky Note - Stage 4 |
| Code 11 - Build Final Strategy Digest | `n8n-nodes-base.code` | Compiles multi-competitor summary digest. | IF - Competitor Queue Empty (True) | WhatsApp - Send Strategy Digest | Sticky Note - Stage 1 |
| WhatsApp - Send Strategy Digest | `n8n-nodes-base.whatsApp` | Sends execution summary digest via WhatsApp. | Code 11 - Build Final Strategy Digest | Respond to Webhook - Return Digest | Sticky Note - Stage 1 |
| Respond to Webhook - Return Digest | `n8n-nodes-base.respondToWebhook` | Returns execution digest JSON to webhook caller. | WhatsApp - Send Strategy Digest | None | Sticky Note - Stage 1 |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Triggers and Ingestion Nodes:**
   - Add a **Schedule Trigger - Daily/Weekly Competitor Crawl** node (Interval: 24 hours).
   - Add a **Webhook - Register Competitor** node (Path: `competitor-strategy-register`, Method: `POST`, Response Mode: `Response Node`).
   - Add a **Manual Trigger - Test Run** node.
   - Add a **Code - Normalize Trigger Input** node (JS code provided in JSON to parse incoming webhook body or fallback to test sample data). Connect all three triggers to this node.

2. **Configure Global Settings & Mode Selection:**
   - Add a **Set - Config** node to define parameters: `crawlDelaySeconds` (3), `defaultMaxPagesPerCompetitor` (30), `defaultMaxDepthPerCompetitor` (3), `maxCompetitorsPerRun` (10), `internalLinkDeltaThreshold` (5), `wordCountDeltaThreshold` (100), `significantChangeCountThreshold` (5), `strategyConfidenceThreshold` (60), API endpoints, Anthropic model, and WhatsApp credentials.
   - Add a **Code - Determine Trigger Mode** node to set `isAdHocCompetitor`. Connect Set - Config to this node.
   - Add an **IF - Ad Hoc Competitor Provided** node evaluating `{{ $json.isAdHocCompetitor }}`.

3. **Build Competitor Queue Logic:**
   - On the True branch of the IF node, add a **Code - Build Competitor Queue From Trigger** node.
   - On the False branch, add an **HTTP Request - Fetch Tracked Competitors** node (HTTP Header Auth, URL from config, `continueOnFail: true`), followed by a **Code - Build Competitor Queue From Tracked** node.
   - Both queue builder branches connect into a **Code - Get Next Competitor From Queue** node.
   - Add an **IF - Competitor Queue Empty** node to check if queues are empty or run limits are reached.

4. **Implement Crawling & Parsing Loop:**
   - From the False branch of the Queue Empty check, add a **Code - Get Next Page From Queue** node.
   - Add an **IF - Page Queue Empty Or Page Budget Reached** node.
   - On the False branch of the page budget check, add a **Wait - Crawl Delay** node (3 seconds).
   - Connect Wait to an **HTTP Request - Fetch Page HTML** node (Response Format: Text, custom User-Agent, `continueOnFail: true`).
   - Add a **Code - Parse Page** node to extract SEO tags, schema types, internal links, word counts, and content hashes.
   - Add a **Code - Update Page Queue & Site Snapshot** node. Connect its output back to **Code - Get Next Page From Queue**.

5. **Set Up Diff Engine & Database Storage:**
   - From the True branch of the page budget check, add an **HTTP Request - Fetch Previous Site Snapshot From Historical DB** node (HTTP Header Auth, query param `competitorDomain`, `continueOnFail: true`).
   - Add a **Code - Diff Engine: Compute Site Changes** node to process the 7 change categories.
   - Add an **HTTP Request - Save Current Site Snapshot To Historical DB** node (POST method, JSON body, Generic HTTP Header Auth, `continueOnFail: true`).

6. **Integrate AI Strategy Agent & Alerting:**
   - Connect the DB save node to an **HTTP Request - Strategy Agent (Claude)** node (POST to `https://api.anthropic.com/v1/messages`, HTTP Header Auth, system prompt with strict JSON formatting, max tokens 1200, retry logic, `onError: continueErrorOutput`).
   - Add a **Code - Parse Strategy Agent Response & Build Competitor Report** node to parse AI JSON and compute total changes and significance.
   - Add an **IF - Significant Strategy Change Detected** node evaluating `{{ $json.significantStrategyChange }}`.
   - On the True branch, add a **WhatsApp - Send Strategy Alert** node (Operation: Send, Phone Number ID and Recipient from config/state).
   - Connect both branches of the significance IF node back to **Code - Get Next Competitor From Queue** to process the next competitor.

7. **Finalize Digest & Webhook Response:**
   - From the True branch of the **IF - Competitor Queue Empty** node, add a **Code - Build Final Strategy Digest** node.
   - Add a **WhatsApp - Send Strategy Digest** node.
   - Add a **Respond to Webhook - Return Digest** node responding with JSON.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Competitor Strategy Change Detector Workflow | Built using n8n for automated SEO competitor intelligence and AI-driven strategic analysis. |
| Anthropic Claude API Integration | Relies on `claude-sonnet-4-6` with structured JSON output instructions (https://api.anthropic.com/v1/messages). |
| WhatsApp Business Cloud API Integration | Utilized for real-time strategic alerts and execution summary digests. |