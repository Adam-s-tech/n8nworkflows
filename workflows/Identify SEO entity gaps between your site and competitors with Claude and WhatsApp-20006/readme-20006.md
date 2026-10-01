Identify SEO entity gaps between your site and competitors with Claude and WhatsApp

https://n8nworkflows.xyz/workflows/identify-seo-entity-gaps-between-your-site-and-competitors-with-claude-and-whatsapp-20006


# Identify SEO entity gaps between your site and competitors with Claude and WhatsApp

### 1. Workflow Overview

This workflow automates SEO entity gap analysis by comparing a website's pages against competitor pages for targeted topics. It identifies missing or weakly covered named entities using Anthropic Claude, prioritizes recommendations via an AI gap agent, issues immediate WhatsApp alerts for significant gaps, and compiles a comprehensive run digest.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Normalization:** Receives triggers from a weekly schedule, webhook payload, or manual run, normalizing raw inputs into a unified schema.
- **1.2 Configuration & Mode Routing:** Establishes global environment variables and runtime limits, determining whether to process an ad-hoc topic or fetch tracked topics from a database.
- **1.3 Topic Queue Construction:** Builds the list of topics to analyze either directly from the webhook payload or via an external API endpoint.
- **1.4 Topic Iteration & Control:** Manages the main processing loop over each topic in the queue, routing to final reporting when the queue is exhausted or limits are reached.
- **1.5 Own-Site Crawling & Entity Extraction:** Iterates through the target site's URLs, fetches and cleans HTML content, accumulates page text, and calls the Anthropic Claude API to extract named entities and prominence ratings.
- **1.6 Competitor Queue & Crawling:** Iterates through competitor URLs with rate limiting (crawl delays), fetches and parses competitor HTML pages, and extracts competitor entities using Anthropic Claude.
- **1.7 Gap Calculation & Strategic Analysis:** Aggregates competitor entity data, computes missing and weakly-covered gaps against your site, and uses Claude as a strategic agent to prioritize gaps into thematic clusters.
- **1.8 Alerting & Final Digest:** Evaluates gap severity thresholds to send immediate WhatsApp alerts and compiles a final run summary delivered via WhatsApp and returned to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** Captures execution signals from multiple entry points and normalizes the payload into a consistent working state.
- **Nodes Involved:** `When Weekly Gap Check Scheduled`, `When Topic Registered via Webhook`, `Start Manual Test Run`, `Normalize Trigger Data`, `Set Configuration Parameters`
- **Node Details:**
  - `When Weekly Gap Check Scheduled`: Trigger node (Schedule). Fires weekly. Inputs: None. Outputs: `Normalize Trigger Data`. Edge cases: None.
  - `When Topic Registered via Webhook`: Trigger node (Webhook). Receives external POST requests. Inputs: None. Outputs: `Normalize Trigger Data`. Configuration: Path `entity-gap-register-topic`, Method `POST`, Response Mode `responseNode`. Edge cases: Payload parsing errors or missing headers.
  - `Start Manual Test Run`: Trigger node (Manual). Inputs: None. Outputs: `Normalize Trigger Data`.
  - `Normalize Trigger Data`: Code node. Unifies inbound webhook or trigger payloads. Variables: Uses `$input.first().json`. Defaults to sample project management topics for manual runs. Inputs: Trigger nodes. Outputs: `Set Configuration Parameters`.
  - `Set Configuration Parameters`: Set node. Establishes runtime configuration constants. Configuration: Sets numeric limits (`maxUrlsPerSide: 10`, `maxTopicsPerRun: 15`, `minCompetitorMentionsForGap: 2`, `significantGapCountThreshold: 5`, `crawlDelaySeconds: 2`), API endpoints (`trackedTopicsUrl`), Anthropic model (`claude-sonnet-4-6`), User-Agent string, and WhatsApp parameters. Inputs: `Normalize Trigger Data`. Outputs: `Determine Mode from Input`.

#### 1.2 Configuration & Mode Routing
- **Overview:** Evaluates configuration parameters and flags whether the workflow execution is ad-hoc or automated.
- **Nodes Involved:** `Determine Mode from Input`, `Check for Ad Hoc Topic`
- **Node Details:**
  - `Determine Mode from Input`: Code node. Evaluates input data to set the `isAdHocTopic` boolean flag. Inputs: `Set Configuration Parameters`. Outputs: `Check for Ad Hoc Topic`.
  - `Check for Ad Hoc Topic`: If node. Branches execution based on `isAdHocTopic`. True branch goes to ad-hoc queue builder; False branch fetches tracked topics from a database API. Inputs: `Determine Mode from Input`. Outputs: `Build Queue from Trigger Topic` (True) or `Fetch Tracked Topics API` (False).

#### 1.3 Topic Queue Construction
- **Overview:** Constructs the array of topics and target URLs from either webhook inputs or a database endpoint.
- **Nodes Involved:** `Build Queue from Trigger Topic`, `Fetch Tracked Topics API`, `Queue Topics from Tracked Data`
- **Node Details:**
  - `Build Queue from Trigger Topic`: Code node. Wraps inbound ad-hoc topic details into a standardized topic queue array. Inputs: `Check for Ad Hoc Topic` (True). Outputs: `Retrieve Next Topic to Process`.
  - `Fetch Tracked Topics API`: HTTP Request node. Fetches monitored topics from the configured database endpoint. Configuration: Uses predefined `httpHeaderAuth` credentials, URL from `={{ $json.trackedTopicsUrl }}`, with `continueOnFail: true`. Inputs: `Check for Ad Hoc Topic` (False). Outputs: `Queue Topics from Tracked Data`. Edge cases: API authentication failures, network timeouts, or malformed JSON responses.
  - `Queue Topics from Tracked Data`: Code node. Maps API response items into the standard `topicQueue` format, applying URL limits. Inputs: `Fetch Tracked Topics API`. Outputs: `Retrieve Next Topic to Process`.

#### 1.4 Topic Iteration & Control
- **Overview:** Pulls the next topic from the queue and controls execution flow based on queue exhaustion or processing limits.
- **Nodes Involved:** `Retrieve Next Topic to Process`, `Check Topic Queue Status`
- **Node Details:**
  - `Retrieve Next Topic to Process`: Code node. Shifts the next topic out of `topicQueue`, initializing tracking arrays for own-site pages and competitor results. Inputs: Queue builders or alert/report nodes. Outputs: `Check Topic Queue Status`.
  - `Check Topic Queue Status`: If node. Determines if the queue is empty or if `processedTopicCount` has reached `maxTopicsPerRun`. Inputs: `Retrieve Next Topic to Process`. Outputs: `Get Next URL from Your Queue` (if more topics remain) or `Create Final Gap Digest Report` (if finished).

#### 1.5 Own-Site Crawling & Entity Extraction
- **Overview:** Iterates through your site's URLs, extracts clean text, and queries Anthropic Claude to extract named entities.
- **Nodes Involved:** `Get Next URL from Your Queue`, `Check Your URL Queue Status`, `Fetch Your Page HTML`, `Parse Your Page Content`, `Accumulate Your Site Data`, `Extract Your Site Entities API`, `Parse Extracted Entities`
- **Node Details:**
  - `Get Next URL from Your Queue`: Code node. Shifts the next URL from `yourUrlQueue`. Inputs: `Check Topic Queue Status` or `Accumulate Your Site Data`. Outputs: `Check Your URL Queue Status`.
  - `Check Your URL Queue Status`: If node. Checks whether `hasNextYourUrl` is false (all own URLs processed). Inputs: `Get Next URL from Your Queue`. Outputs: `Extract Your Site Entities API` (when done) or `Fetch Your Page HTML` (to continue crawling).
  - `Fetch Your Page HTML`: HTTP Request node. Fetches HTML content for the current URL. Configuration: Text response format, custom User-Agent header, `continueOnFail: true`. Inputs: `Check Your URL Queue Status`. Outputs: `Parse Your Page Content`. Edge cases: 404/500 errors, DNS lookup failures, or blocked user-agents.
  - `Parse Your Page Content`: Code node. Parses page title, strips scripts/styles/tags, and truncates text to 4,000 characters. Inputs: `Fetch Your Page HTML`. Outputs: `Accumulate Your Site Data`.
  - `Accumulate Your Site Data`: Code node. Appends parsed page data into the `yourSitePages` array and loops back to `Get Next URL from Your Queue`. Inputs: `Parse Your Page Content`. Outputs: `Get Next URL from Your Queue`.
  - `Extract Your Site Entities API`: HTTP Request node (Anthropic). Sends accumulated site content to Claude for entity extraction. Configuration: POST to `https://api.anthropic.com/v1/messages`, generic HTTP Header Auth, model `={{ $json.anthropicModel }}`, system prompt instructing strict JSON entity output with types and prominence ratings. Inputs: `Check Your URL Queue Status` (when own URLs are fully collected). Outputs: `Parse Extracted Entities`. Edge cases: Rate limits (429), token limits, or invalid JSON responses from the LLM.
  - `Parse Extracted Entities`: Code node. Cleans markdown wrappers from Claude's response, parses JSON, normalizes entity names, and prepares the data for competitor comparison. Inputs: `Extract Your Site Entities API`. Outputs: `Create Competitor URL Queue`.

#### 1.6 Competitor Queue & Crawling
- **Overview:** Manages competitor URL traversal, rate-limiting delays, HTML parsing, and entity extraction via Claude.
- **Nodes Involved:** `Create Competitor URL Queue`, `Retrieve next Competitor URL`, `Check Competitor Queue Status`, `Wait 2 Seconds Before Next Request`, `Fetch Competitor Page HTML`, `Parse Competitor Page Content`, `Extract Competitor Entities API`, `Accumulate Competitor Data`
- **Node Details:**
  - `Create Competitor URL Queue`: Code node. Normalizes competitor items into objects with names and URLs. Inputs: `Parse Extracted Entities`. Outputs: `Retrieve next Competitor URL`.
  - `Retrieve next Competitor URL`: Code node. Shifts the next competitor from `competitorQueue`. Inputs: `Create Competitor URL Queue` or `Accumulate Competitor Data`. Outputs: `Check Competitor Queue Status`.
  - `Check Competitor Queue Status`: If node. Checks if competitor queue is empty. Inputs: `Retrieve next Competitor URL`. Outputs: `Aggregate Competitors' Entity Data` (when queue is empty) or `Wait 2 Seconds Before Next Request` (to crawl next competitor).
  - `Wait 2 Seconds Before Next Request`: Wait node. Introduces a 2-second delay to respect rate limits. Inputs: `Check Competitor Queue Status`. Outputs: `Fetch Competitor Page HTML`.
  - `Fetch Competitor Page HTML`: HTTP Request node. Fetches competitor page HTML. Configuration: Text response, custom User-Agent, `continueOnFail: true`. Inputs: `Wait 2 Seconds Before Next Request`. Outputs: `Parse Competitor Page Content`. Edge cases: Timeout, SSL errors, or target site blocking.
  - `Parse Competitor Page Content`: Code node. Extracts page titles and strips tags for text analysis. Inputs: `Fetch Competitor Page HTML`. Outputs: `Extract Competitor Entities API`.
  - `Extract Competitor Entities API`: HTTP Request node (Anthropic). Sends competitor page text to Claude for entity extraction. Configuration: POST to `https://api.anthropic.com/v1/messages`, generic HTTP Header Auth, max tokens 1000. Inputs: `Parse Competitor Page Content`. Outputs: `Accumulate Competitor Data`. Edge cases: API timeout or refusal errors.
  - `Accumulate Competitor Data`: Code node. Parses Claude's response, normalizes competitor entities, appends them to `competitorEntityResults`, and loops back to `Retrieve next Competitor URL`. Inputs: `Extract Competitor Entities API`. Outputs: `Retrieve next Competitor URL`.

#### 1.7 Gap Calculation & Strategic Analysis
- **Overview:** Aggregates competitor entities, computes missing and weakly-covered gaps, and uses an AI agent to prioritize recommendations.
- **Nodes Involved:** `Aggregate Competitors' Entity Data`, `Calculate Entity Gap`, `Entity Gap Analysis API`, `Build Topic Report from Agent Data`, `Identify Significant Entity Gap`
- **Node Details:**
  - `Aggregate Competitors' Entity Data`: Code node. Aggregates entity occurrences across all analyzed competitors into a consensus table. Inputs: `Check Competitor Queue Status` (when competitor queue is exhausted). Outputs: `Calculate Entity Gap`.
  - `Calculate Entity Gap`: Code node. Compares competitor consensus entities with your site's entities, filtering by `minCompetitorMentionsForGap` to isolate missing entities and compute prominence gap severity for weakly covered entities. Inputs: `Aggregate Competitors' Entity Data`. Outputs: `Entity Gap Analysis API`.
  - `Entity Gap Analysis API`: HTTP Request node (Anthropic). Calls Claude as a strategic gap-analysis agent to prioritize gaps, form thematic clusters, and suggest content updates. Configuration: POST to `https://api.anthropic.com/v1/messages`, generic HTTP Header Auth, max tokens 1400. Inputs: `Calculate Entity Gap`. Outputs: `Build Topic Report from Agent Data`. Edge cases: LLM output truncation or invalid JSON structure.
  - `Build Topic Report from Agent Data`: Code node. Parses agent recommendations, determines if the topic exhibits a significant gap based on `significantGapCountThreshold`, appends the report to results, and increments the processed topic count. Inputs: `Entity Gap Analysis API`. Outputs: `Identify Significant Entity Gap`.
  - `Identify Significant Entity Gap`: If node. Checks whether `significantGap` is true. Inputs: `Build Topic Report from Agent Data`. Outputs: `Notify Gap Alert via WhatsApp` (True) or `Retrieve Next Topic to Process` (False, loops back for next topic).

#### 1.8 Alerting & Final Digest
- **Overview:** Dispatches real-time WhatsApp alerts for significant gaps and compiles/returns the final run digest.
- **Nodes Involved:** `Notify Gap Alert via WhatsApp`, `Create Final Gap Digest Report`, `Send Gap Digest via WhatsApp`, `Return Digest via Webhook Response`
- **Node Details:**
  - `Notify Gap Alert via WhatsApp`: WhatsApp node. Sends an instant alert message detailing missing and weakly covered entities. Configuration: Operation `send`, phone number ID and recipient from workflow state. Inputs: `Identify Significant Entity Gap` (True). Outputs: `Retrieve Next Topic to Process`. Edge cases: WhatsApp Cloud API errors, invalid phone number formatting, or rate limiting.
  - `Create Final Gap Digest Report`: Code node. Summarizes overall metrics across all processed topic reports once the topic queue is empty. Inputs: `Check Topic Queue Status` (True branch). Outputs: `Send Gap Digest via WhatsApp`.
  - `Send Gap Digest via WhatsApp`: WhatsApp node. Sends the final run digest message. Configuration: Operation `send`. Inputs: `Create Final Gap Digest Report`. Outputs: `Return Digest via Webhook Response`. Edge cases: API communication failures.
  - `Return Digest via Webhook Response`: Respond to Webhook node. Returns the final JSON digest to the original webhook caller if triggered via webhook. Configuration: Respond with `json`, body `={{ $json }}`, `continueOnFail: true`. Inputs: `Send Gap Digest via WhatsApp`. Outputs: None. Edge cases: Connection dropped by webhook caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Weekly Gap Check Scheduled` | `scheduleTrigger` | Starts the workflow on a weekly cadence. | None | `Normalize Trigger Data` | ## Trigger and normalize input<br><br>Starts the workflow from a schedule, webhook, or manual test run, then normalizes the incoming payload into a consistent shape for downstream processing. |
| `When Topic Registered via Webhook` | `webhook` | Receives ad-hoc topic registration POST requests. | None | `Normalize Trigger Data` | ## Trigger and normalize input<br><br>Starts the workflow from a schedule, webhook, or manual test run, then normalizes the incoming payload into a consistent shape for downstream processing. |
| `Start Manual Test Run` | `manualTrigger` | Initiates manual test executions. | None | `Normalize Trigger Data` | ## Trigger and normalize input<br><br>Starts the workflow from a schedule, webhook, or manual test run, then normalizes the incoming payload into a consistent shape for downstream processing. |
| `Normalize Trigger Data` | `code` | Normalizes trigger payloads into standard format. | `When Weekly Gap Check Scheduled`, `When Topic Registered via Webhook`, `Start Manual Test Run` | `Set Configuration Parameters` | ## Trigger and normalize input<br><br>Starts the workflow from a schedule, webhook, or manual test run, then normalizes the incoming payload into a consistent shape for downstream processing. |
| `Set Configuration Parameters` | `set` | Defines configuration constants and limits. | `Normalize Trigger Data` | `Determine Mode from Input` | ## Trigger and normalize input<br><br>Starts the workflow from a schedule, webhook, or manual test run, then normalizes the incoming payload into a consistent shape for downstream processing. |
| `Determine Mode from Input` | `code` | Sets ad-hoc detection flag. | `Set Configuration Parameters` | `Check for Ad Hoc Topic` | ## Configure trigger mode<br><br>Applies runtime configuration, determines whether the run is ad hoc or scheduled, and branches toward the appropriate topic source. |
| `Check for Ad Hoc Topic` | `if` | Branches execution between ad-hoc and tracked modes. | `Determine Mode from Input` | `Build Queue from Trigger Topic`, `Fetch Tracked Topics API` | ## Configure trigger mode<br><br>Applies runtime configuration, determines whether the run is ad hoc or scheduled, and branches toward the appropriate topic source. |
| `Build Queue from Trigger Topic` | `code` | Builds queue from webhook or manual trigger. | `Check for Ad Hoc Topic` | `Retrieve Next Topic to Process` | ## Build topic queue<br><br>Creates the topic queue either directly from an ad hoc trigger topic or by fetching tracked topics from the configured endpoint. |
| `Fetch Tracked Topics API` | `httpRequest` | Fetches monitored topics from database API. | `Check for Ad Hoc Topic` | `Queue Topics from Tracked Data` | ## Build topic queue<br><br>Creates the topic queue either directly from an ad hoc trigger topic or by fetching tracked topics from the configured endpoint. |
| `Queue Topics from Tracked Data` | `code` | Maps tracked API data into topic queue. | `Fetch Tracked Topics API` | `Retrieve Next Topic to Process` | ## Build topic queue<br><br>Creates the topic queue either directly from an ad hoc trigger topic or by fetching tracked topics from the configured endpoint. |
| `Retrieve Next Topic to Process` | `code` | Shifts next topic from queue and resets state. | `Build Queue from Trigger Topic`, `Queue Topics from Tracked Data`, `Notify Gap Alert via WhatsApp`, `Identify Significant Entity Gap` | `Check Topic Queue Status` | ## Control topic iteration<br><br>Pulls the next topic from the queue and decides whether to continue analysis or finish the run with a digest. |
| `Check Topic Queue Status` | `if` | Checks if topic queue is empty or limit reached. | `Retrieve Next Topic to Process` | `Create Final Gap Digest Report`, `Get Next URL from Your Queue` | ## Control topic iteration<br><br>Pulls the next topic from the queue and decides whether to continue analysis or finish the run with a digest. |
| `Get Next URL from Your Queue` | `code` | Shifts next URL from own-site URL queue. | `Check Topic Queue Status`, `Accumulate Your Site Data` | `Check Your URL Queue Status` | ## Loop own URLs<br><br>Iterates through the current topic’s own-site URL queue and routes either to page crawling or to entity extraction once all own URLs are collected. |
| `Check Your URL Queue Status` | `if` | Checks if own-site URL queue is exhausted. | `Get Next URL from Your Queue` | `Extract Your Site Entities API`, `Fetch Your Page HTML` | ## Loop own URLs<br><br>Iterates through the current topic’s own-site URL queue and routes either to page crawling or to entity extraction once all own URLs are collected. |
| `Fetch Your Page HTML` | `httpRequest` | Fetches own-site page HTML. | `Check Your URL Queue Status` | `Parse Your Page Content` | ## Collect own page text<br><br>Fetches each own-site page, parses usable page text, accumulates it, and loops back for the next own URL. |
| `Parse Your Page Content` | `code` | Cleans HTML and extracts main text. | `Fetch Your Page HTML` | `Accumulate Your Site Data` | ## Collect own page text<br><br>Fetches each own-site page, parses usable page text, accumulates it, and loops back for the next own URL. |
| `Accumulate Your Site Data` | `code` | Appends page text to own-site collection. | `Parse Your Page Content` | `Get Next URL from Your Queue` | ## Collect own page text<br><br>Fetches each own-site page, parses usable page text, accumulates it, and loops back for the next own URL. |
| `Extract Your Site Entities API` | `httpRequest` | Sends own-site content to Claude for entity extraction. | `Check Your URL Queue Status` | `Parse Extracted Entities` | ## Extract own entities<br><br>Sends accumulated own-site text to Claude and parses the returned entity set for the topic. |
| `Parse Extracted Entities` | `code` | Parses and normalizes own-site entities. | `Extract Your Site Entities API` | `Create Competitor URL Queue` | ## Extract own entities<br><br>Sends accumulated own-site text to Claude and parses the returned entity set for the topic. |
| `Create Competitor URL Queue` | `code` | Prepares competitor URL queue objects. | `Parse Extracted Entities` | `Retrieve next Competitor URL` | ## Prepare competitor queue<br><br>Builds and controls the competitor URL queue, deciding whether to keep crawling competitors or move on to aggregation. |
| `Retrieve next Competitor URL` | `code` | Shifts next competitor from queue. | `Create Competitor URL Queue`, `Accumulate Competitor Data` | `Check Competitor Queue Status` | ## Prepare competitor queue<br><br>Builds and controls the competitor URL queue, deciding whether to keep crawling competitors or move on to aggregation. |
| `Check Competitor Queue Status` | `if` | Checks if competitor queue is empty. | `Retrieve next Competitor URL` | `Aggregate Competitors' Entity Data`, `Wait 2 Seconds Before Next Request` | ## Prepare competitor queue<br><br>Builds and controls the competitor URL queue, deciding whether to keep crawling competitors or move on to aggregation. |
| `Wait 2 Seconds Before Next Request` | `wait` | Introduces crawl delay between competitor requests. | `Check Competitor Queue Status` | `Fetch Competitor Page HTML` | ## Fetch competitor pages<br><br>Adds a crawl delay, retrieves the current competitor page HTML, and parses it into clean text for entity extraction. |
| `Fetch Competitor Page HTML` | `httpRequest` | Fetches competitor page HTML. | `Wait 2 Seconds Before Next Request` | `Parse Competitor Page Content` | ## Fetch competitor pages<br><br>Adds a crawl delay, retrieves the current competitor page HTML, and parses it into clean text for entity extraction. |
| `Parse Competitor Page Content` | `code` | Cleans competitor HTML content. | `Fetch Competitor Page HTML` | `Extract Competitor Entities API` | ## Fetch competitor pages<br><br>Adds a crawl delay, retrieves the current competitor page HTML, and parses it into clean text for entity extraction. |
| `Extract Competitor Entities API` | `httpRequest` | Sends competitor page text to Claude for entities. | `Parse Competitor Page Content` | `Accumulate Competitor Data` | ## Extract competitor entities<br><br>Sends competitor page text to Claude, parses the returned entities, accumulates them, and returns to the competitor loop. |
| `Accumulate Competitor Data` | `code` | Accumulates competitor entity extraction results. | `Extract Competitor Entities API` | `Retrieve next Competitor URL` | ## Extract competitor entities<br><br>Sends competitor page text to Claude, parses the returned entities, accumulates them, and returns to the competitor loop. |
| `Aggregate Competitors' Entity Data` | `code` | Aggregates entity occurrences across competitors. | `Check Competitor Queue Status` | `Calculate Entity Gap` | ## Compute entity gaps<br><br>Aggregates entity data across all competitors and compares it with the site’s own entities to calculate raw gap signals. |
| `Calculate Entity Gap` | `code` | Computes missing and weakly-covered gaps. | `Aggregate Competitors' Entity Data` | `Entity Gap Analysis API` | ## Compute entity gaps<br><br>Aggregates entity data across all competitors and compares it with the site’s own entities to calculate raw gap signals. |
| `Entity Gap Analysis API` | `httpRequest` | Calls Claude gap-analysis agent for prioritization. | `Calculate Entity Gap` | `Build Topic Report from Agent Data` | ## Generate gap report<br><br>Uses Claude as a gap-analysis agent, parses the agent response, builds a topic report, and determines whether the gap is significant. |
| `Build Topic Report from Agent Data` | `code` | Parses agent response and builds topic report. | `Entity Gap Analysis API` | `Identify Significant Entity Gap` | ## Generate gap report<br><br>Uses Claude as a gap-analysis agent, parses the agent response, builds a topic report, and determines whether the gap is significant. |
| `Identify Significant Entity Gap` | `if` | Checks if topic gap count meets significance threshold. | `Build Topic Report from Agent Data` | `Notify Gap Alert via WhatsApp`, `Retrieve Next Topic to Process` | ## Generate gap report<br><br>Uses Claude as a gap-analysis agent, parses the agent response, builds a topic report, and determines whether the gap is significant. |
| `Notify Gap Alert via WhatsApp` | `whatsApp` | Sends immediate WhatsApp alert for significant gaps. | `Identify Significant Entity Gap` | `Retrieve Next Topic to Process` | ## Send gap alert<br><br>Sends an immediate WhatsApp alert when a topic has a significant entity gap, then loops back to the next topic. |
| `Create Final Gap Digest Report` | `code` | Compiles final run summary and digest. | `Check Topic Queue Status` | `Send Gap Digest via WhatsApp` | ## Send final digest<br><br>Builds the final entity gap digest after all topics are processed, sends it through WhatsApp, and returns it to the webhook caller. |
| `Send Gap Digest via WhatsApp` | `whatsApp` | Sends final digest report via WhatsApp. | `Create Final Gap Digest Report` | `Return Digest via Webhook Response` | ## Send final digest<br><br>Builds the final entity gap digest after all topics are processed, sends it through WhatsApp, and returns it to the webhook caller. |
| `Return Digest via Webhook Response` | `respondToWebhook` | Returns final digest JSON to webhook caller. | `Send Gap Digest via WhatsApp` | None | ## Send final digest<br><br>Builds the final entity gap digest after all topics are processed, sends it through WhatsApp, and returns it to the webhook caller. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Trigger and Normalization Nodes
1. Create a **Schedule Trigger** (`When Weekly Gap Check Scheduled`) set to weekly intervals.
2. Create a **Webhook** (`When Topic Registered via Webhook`) with Path `entity-gap-register-topic`, Method `POST`, and Response Mode `responseNode`.
3. Create a **Manual Trigger** (`Start Manual Test Run`).
4. Create a **Code** node (`Normalize Trigger Data`) connected to all three triggers. Set JavaScript to normalize incoming payloads, defaulting to sample URLs if empty.
5. Create a **Set** node (`Set Configuration Parameters`) to define configuration values (`maxUrlsPerSide`, `maxTopicsPerRun`, `minCompetitorMentionsForGap`, `significantGapCountThreshold`, `crawlDelaySeconds`, `trackedTopicsUrl`, `anthropicModel`, `userAgent`, `whatsappBusinessPhoneId`, `whatsappRecipientPhone`).

#### Step 2: Set Up Mode Routing and Topic Queues
1. Create a **Code** node (`Determine Mode from Input`) to add the `isAdHocTopic` boolean flag.
2. Create an **If** node (`Check for Ad Hoc Topic`) evaluating `={{ $json.isAdHocTopic }}`.
3. For the True branch, create a **Code** node (`Build Queue from Trigger Topic`).
4. For the False branch, create an **HTTP Request** node (`Fetch Tracked Topics API`) using predefined `httpHeaderAuth` credentials pointing to `trackedTopicsUrl`, configured with `continueOnFail: true`, followed by a **Code** node (`Queue Topics from Tracked Data`).

#### Step 3: Configure Topic Iteration Loop
1. Create a **Code** node (`Retrieve Next Topic to Process`) to shift topics from the queue and initialize tracking arrays.
2. Create an **If** node (`Check Topic Queue Status`) checking if `hasNextTopic` is false or `processedTopicCount >= maxTopicsPerRun`.

#### Step 4: Build Own-Site Crawling & Extraction Loop
1. Create a **Code** node (`Get Next URL from Your Queue`) to shift own URLs.
2. Create an **If** node (`Check Your URL Queue Status`) checking `hasNextYourUrl`.
3. If False (all own URLs collected), create an **HTTP Request** node (`Extract Your Site Entities API`) configured for POST to `https://api.anthropic.com/v1/messages` using generic HTTP Header Auth. Set the system prompt and user message to analyze site pages and return JSON entities. Connect to a **Code** node (`Parse Extracted Entities`).
4. If True (more URLs to crawl), create an **HTTP Request** node (`Fetch Your Page HTML`) with text response and custom User-Agent, connected to a **Code** node (`Parse Your Page Content`) and a **Code** node (`Accumulate Your Site Data`) that loops back to `Get Next URL from Your Queue`.

#### Step 5: Build Competitor Crawling & Extraction Loop
1. Create a **Code** node (`Create Competitor URL Queue`) followed by a **Code** node (`Retrieve next Competitor URL`).
2. Create an **If** node (`Check Competitor Queue Status`) checking `hasNextCompetitorUrl`.
3. If True, add a **Wait** node (`Wait 2 Seconds Before Next Request`) set to 2 seconds, then an **HTTP Request** node (`Fetch Competitor Page HTML`), a **Code** node (`Parse Competitor Page Content`), an **HTTP Request** node (`Extract Competitor Entities API`) calling Anthropic Claude, and a **Code** node (`Accumulate Competitor Data`) looping back to `Retrieve next Competitor URL`.
4. If False, create a **Code** node (`Aggregate Competitors' Entity Data`) to build the competitor consensus table.

#### Step 6: Configure Gap Analysis and Reporting
1. Create a **Code** node (`Calculate Entity Gap`) to compute missing and weakly-covered entities.
2. Create an **HTTP Request** node (`Entity Gap Analysis API`) calling Anthropic Claude for strategic gap prioritization and thematic clustering.
3. Create a **Code** node (`Build Topic Report from Agent Data`) to parse agent results and evaluate significance.
4. Create an **If** node (`Identify Significant Entity Gap`) checking `significantGap`.
5. If True, create a **WhatsApp** node (`Notify Gap Alert via WhatsApp`) configured with `send` operation, phone number ID, and recipient. Connect both branches back to `Retrieve Next Topic to Process`.

#### Step 7: Configure Final Digest and Webhook Response
1. When the topic queue is exhausted (from `Check Topic Queue Status`), connect to a **Code** node (`Create Final Gap Digest Report`).
2. Connect to a **WhatsApp** node (`Send Gap Digest via WhatsApp`) to send the final summary.
3. Connect to a **Respond to Webhook** node (`Return Digest via Webhook Response`) configured to return JSON response.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SEO Entity Gap Engine overview, setup steps, and customization options. | Main workflow reference documented in sticky notes. |