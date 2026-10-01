Detect search intent drift in Google SERPs with Claude and WhatsApp alerts

https://n8nworkflows.xyz/workflows/detect-search-intent-drift-in-google-serps-with-claude-and-whatsapp-alerts-20007


# Detect search intent drift in Google SERPs with Claude and WhatsApp alerts

### 1. Workflow Overview

This workflow monitors Google SERP changes for tracked keywords by storing daily SERP snapshots, classifying dominant intent over time with Anthropic Claude, detecting meaningful intent drift, and sending WhatsApp alerts and a run digest.

The logic is grouped into the following functional blocks:

- **1.1 Input Reception & Normalization:** Receives execution triggers from a daily schedule, ad-hoc webhook, or manual test, normalizes the inputs, and assigns global configuration parameters.
- **1.2 Queue Building & Control:** Determines execution mode, builds the keyword processing queue (either from an incoming payload or by fetching tracked keywords), and iterates through the queue item by item.
- **1.3 SERP Capture & Historical Retrieval:** Collects the current search engine results page (SERP) snapshot for the active keyword, normalizes it, saves it to the historical database, and retrieves historical SERP series data.
- **1.4 AI-Driven Intent Analysis & Drift Detection:** Computes periodic composition metrics, sends historical timelines to Anthropic Claude to classify search intent per period, and runs a secondary Claude analysis to detect meaningful intent drift.
- **1.5 Alerting, Digest & Response:** Evaluates drift significance, triggers immediate WhatsApp alerts for anomalous shifts, compiles a comprehensive run digest, sends a summary via WhatsApp, and returns the response to webhook callers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
**Overview:** This block handles workflow initiation from multiple sources (schedule, webhook, or manual test), normalizes the payload structure, and injects configuration variables for API endpoints, thresholds, and authentication.

- **Nodes Involved:** 
  - `When Daily Drift Check Initiates`
  - `When Keyword Registered for Drift`
  - `When Manual Test Run Initiates`
  - `Normalize Trigger Input Data`
  - `Set Configuration Parameters`
  - `Determine Trigger Mode`

- **Node Details:**
  - **When Daily Drift Check Initiates**
    - *Type & Role:* `scheduleTrigger` — Triggers execution on a fixed 24-hour interval.
    - *Configuration:* Interval set to every 24 hours.
    - *Expressions:* None.
    - *Connections:* Input: None | Output: `Normalize Trigger Input Data`
    - *Edge Cases:* Server downtime during scheduled runs; missed triggers must be caught by manual intervention or subsequent runs.
  - **When Keyword Registered for Drift**
    - *Type & Role:* `webhook` — Receives external HTTP POST requests to register or check a specific keyword ad-hoc.
    - *Configuration:* Path `intent-drift-register-keyword`, method `POST`, response mode using `responseNode`.
    - *Expressions:* None.
    - *Connections:* Input: None | Output: `Normalize Trigger Input Data`
    - *Edge Cases:* Malformed JSON payloads; missing body properties handled upstream.
  - **When Manual Test Run Initiates**
    - *Type & Role:* `manualTrigger` — Initiates test runs manually inside the n8n interface.
    - *Configuration:* Default empty execution.
    - *Expressions:* None.
    - *Connections:* Input: None | Output: `Normalize Trigger Input Data`
    - *Edge Cases:* None.
  - **Normalize Trigger Input Data**
    - *Type & Role:* `code` — Extracts and normalizes input payloads across different trigger types, defaulting to a sample query (`best crm software`) for manual runs.
    - *Configuration:* JavaScript data parsing logic.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: Schedule, Webhook, and Manual triggers | Output: `Set Configuration Parameters`
    - *Edge Cases:* Empty or null body objects resolved via fallback logic.
  - **Set Configuration Parameters**
    - *Type & Role:* `set` — Establishes global environment variables such as historical window size, API URLs, Anthropic model names, and WhatsApp phone credentials.
    - *Configuration:* Static assignments for numbers, strings, and thresholds (`historicalWindowSize: 6`, `driftConfidenceThreshold: 65`, etc.).
    - *Expressions:* None.
    - *Connections:* Input: `Normalize Trigger Input Data` | Output: `Determine Trigger Mode`
    - *Edge Cases:* Incorrect endpoint URLs or placeholder credentials failing downstream requests.
  - **Determine Trigger Mode**
    - *Type & Role:* `code` — Evaluates whether a specific trigger keyword is present to set the `isAdHocKeyword` boolean flag.
    - *Configuration:* JavaScript evaluation logic.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `Set Configuration Parameters` | Output: `If Ad Hoc Keyword Exists`
    - *Edge Cases:* Undefined properties safely coerced using boolean operators (`!!`).

---

#### 2.2 Queue Building & Control
**Overview:** Evaluates execution mode and sets up the processing queue by either extracting the single ad-hoc keyword or querying the database for all tracked keywords, then iteratively pops items for processing.

- **Nodes Involved:**
  - `If Ad Hoc Keyword Exists`
  - `Build Queue from Ad Hoc Keyword`
  - `Fetch Tracked Keywords`
  - `Build Queue from Tracked Keywords`
  - `Retrieve Next Keyword from Queue`
  - `If Keyword Queue is Empty`

- **Node Details:**
  - **If Ad Hoc Keyword Exists**
    - *Type & Role:* `if` — Branches the workflow based on whether an ad-hoc keyword was submitted via webhook or test.
    - *Configuration:* Evaluates `{{ $json.isAdHocKeyword }}` as boolean `true`.
    - *Expressions:* `{{ $json.isAdHocKeyword }}`
    - *Connections:* Input: `Determine Trigger Mode` | Output (True): `Build Queue from Ad Hoc Keyword` | Output (False): `Fetch Tracked Keywords`
    - *Edge Cases:* None.
  - **Build Queue from Ad Hoc Keyword**
    - *Type & Role:* `code` — Wraps the single ad-hoc keyword into a processing queue array.
    - *Configuration:* JavaScript array construction.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `If Ad Hoc Keyword Exists` (True) | Output: `Retrieve Next Keyword from Queue`
    - *Edge Cases:* None.
  - **Fetch Tracked Keywords**
    - *Type & Role:* `httpRequest` — Fetches the list of all tracked keywords from the historical database API.
    - *Configuration:* GET request to `{{ $json.trackedKeywordsUrl }}` using `httpHeaderAuth`. `continueOnFail` enabled.
    - *Expressions:* `={{ $json.trackedKeywordsUrl }}`
    - *Connections:* Input: `If Ad Hoc Keyword Exists` (False) | Output: `Build Queue from Tracked Keywords`
    - *Edge Cases:* API authentication failure, rate limits, or network timeouts returning empty results.
  - **Build Queue from Tracked Keywords**
    - *Type & Role:* `code` — Parses the database response into an array of keyword processing objects.
    - *Configuration:* JavaScript mapping logic handling arrays and wrapped objects.
    - *Expressions:* Uses upstream state and HTTP response data.
    - *Connections:* Input: `Fetch Tracked Keywords` | Output: `Retrieve Next Keyword from Queue`
    - *Edge Cases:* Unexpected JSON response structure resulting in an empty queue array.
  - **Retrieve Next Keyword from Queue**
    - *Type & Role:* `code` — Implements queue shifting, popping the next keyword to process and tracking whether more items remain.
    - *Configuration:* JavaScript array mutation (`shift()`).
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `Build Queue from Ad Hoc Keyword`, `Build Queue from Tracked Keywords`, and `If Significant Drift Found`/`Send Drift Alert via WhatsApp` | Output: `If Keyword Queue is Empty`
    - *Edge Cases:* Empty queue handling (`hasNext: false`).
  - **If Keyword Queue is Empty**
    - *Type & Role:* `if` — Checks if the queue is exhausted or if the maximum processed keyword count limit has been reached.
    - *Configuration:* Evaluates `hasNext` equals false or `processedCount >= maxKeywordsPerRun`.
    - *Expressions:* `{{ $json.hasNext }}` and `{{ $json.processedCount }}`
    - *Connections:* Input: `Retrieve Next Keyword from Queue` | Output (True): `Build Final Drift Digest` | Output (False): `Fetch Current SERP Snapshot`
    - *Edge Cases:* Infinite loops prevented by strict bounds checking on `maxKeywordsPerRun`.

---

#### 2.3 SERP Capture & Historical Retrieval
**Overview:** Requests live SERP data for the active keyword from an external provider, normalizes results, records the snapshot to the database, and fetches past historical periods for comparison.

- **Nodes Involved:**
  - `Fetch Current SERP Snapshot`
  - `Normalize SERP Snapshot Data`
  - `Save Snapshot to Historical DB`
  - `Fetch Historical SERP Data`
  - `Build Historical Snapshot Timeline`

- **Node Details:**
  - **Fetch Current SERP Snapshot**
    - *Type & Role:* `httpRequest` — Requests live search engine results for the current keyword.
    - *Configuration:* GET request to `{{ $json.serpApiUrl }}` with query parameters `q` and `num=20`. Uses `httpHeaderAuth`. `continueOnFail` enabled.
    - *Expressions:* `={{ $json.serpApiUrl }}`, `={{ $json.currentKeyword }}`
    - *Connections:* Input: `If Keyword Queue is Empty` (False) | Output: `Normalize SERP Snapshot Data`
    - *Edge Cases:* SERP provider errors, invalid API keys, or rate limits returning error payloads.
  - **Normalize SERP Snapshot Data**
    - *Type & Role:* `code` — Extracts organic results, extracts domain hosts from URLs, and identifies SERP feature flags (featured snippets, local packs, video results, etc.).
    - *Configuration:* JavaScript parsing logic.
    - *Expressions:* Uses upstream state and HTTP response.
    - *Connections:* Input: `Fetch Current SERP Snapshot` | Output: `Save Snapshot to Historical DB`
    - *Edge Cases:* Malformed URLs throwing exceptions handled via `try...catch` blocks.
  - **Save Snapshot to Historical DB**
    - *Type & Role:* `httpRequest` — Saves the normalized SERP snapshot and detected features to the historical database.
    - *Configuration:* POST request to `{{ $json.historicalDbSaveUrl }}` with JSON body containing keyword, timestamp, results, and features. Uses generic HTTP Header Auth. `continueOnFail` enabled.
    - *Expressions:* `={{ $json.historicalDbSaveUrl }}`, `={{ JSON.stringify(...) }}`
    - *Connections:* Input: `Normalize SERP Snapshot Data` | Output: `Fetch Historical SERP Data`
    - *Edge Cases:* Database write failures or schema validation errors.
  - **Fetch Historical SERP Data**
    - *Type & Role:* `httpRequest` — Retrieves historical snapshots for the keyword up to the configured window size.
    - *Configuration:* GET request to `{{ $json.historicalDbFetchUrl }}` with query parameters `keyword` and `limit`. Uses `httpHeaderAuth`. `continueOnFail` enabled.
    - *Expressions:* `={{ $json.historicalDbFetchUrl }}`, `={{ $json.currentKeyword }}`, `={{ $json.historicalWindowSize }}`
    - *Connections:* Input: `Save Snapshot to Historical DB` | Output: `Build Historical Snapshot Timeline`
    - *Edge Cases:* Database connectivity issues or missing historical records.
  - **Build Historical Snapshot Timeline**
    - *Type & Role:* `code` — Assembles historical snapshots into a sorted timeline, ensuring the newly captured snapshot is included, and checks if sufficient history exists.
    - *Configuration:* JavaScript timeline sorting and slicing logic.
    - *Expressions:* Uses upstream state and database response.
    - *Connections:* Input: `Fetch Historical SERP Data` | Output: `Compute Periodic Composition`
    - *Edge Cases:* Empty historical response gracefully handled by falling back to the current snapshot.

---

#### 2.4 AI-Driven Intent Analysis & Drift Detection
**Overview:** Computes content composition categories across time periods using heuristics, compares earliest vs. latest compositions, sends timelines to Anthropic Claude for intent classification, and performs a secondary analysis to detect meaningful drift.

- **Nodes Involved:**
  - `Compute Periodic Composition`
  - `Compare Periodic Compositions`
  - `Classify Intent with Claude`
  - `Parse Intent Classification Results`
  - `Detect Drift with Claude`
  - `Parse Drift Detection Report`

- **Node Details:**
  - **Compute Periodic Composition**
    - *Type & Role:* `code` — Categorizes SERP result titles and domains into content types (informational, commercial, UGC/forum, video, other) and calculates percentage distributions.
    - *Configuration:* JavaScript heuristic matching logic using Regular Expressions.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `Build Historical Snapshot Timeline` | Output: `Compare Periodic Compositions`
    - *Edge Cases:* Unmatched titles defaulting to 'other'.
  - **Compare Periodic Compositions**
    - *Type & Role:* `code` — Computes deltas in content composition percentages and SERP features between the earliest and latest periods in the timeline.
    - *Configuration:* JavaScript delta calculation logic.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `Compute Periodic Composition` | Output: `Classify Intent with Claude`
    - *Edge Cases:* Missing periods handled via null checks.
  - **Classify Intent with Claude**
    - *Type & Role:* `httpRequest` — Sends the composition timeline to Anthropic Claude to classify the dominant search intent for each period.
    - *Configuration:* POST request to `https://api.anthropic.com/v1/messages` using `claude-sonnet-4-6`. Max tokens 1000, max tries 2, with error continuation enabled.
    - *Expressions:* `={{ JSON.stringify({ model: $json.anthropicModel, ... }) }}`
    - *Connections:* Input: `Compare Periodic Compositions` | Output: `Parse Intent Classification Results`
    - *Edge Cases:* Rate limits, API downtime, or malformed model responses caught via error continuation and retry logic.
  - **Parse Intent Classification Results**
    - *Type & Role:* `code` — Parses Claude’s JSON response, cleaning markdown code blocks and extracting intent timelines.
    - *Configuration:* JavaScript JSON parsing with fallback objects.
    - *Expressions:* Uses upstream HTTP response and state.
    - *Connections:* Input: `Classify Intent with Claude` | Output: `Detect Drift with Claude`
    - *Edge Cases:* Invalid JSON syntax in LLM output returning empty intent arrays.
  - **Detect Drift with Claude**
    - *Type & Role:* `httpRequest` — Sends intent classifications, composition deltas, and feature changes to Anthropic Claude to determine if meaningful intent drift occurred.
    - *Configuration:* POST request to `https://api.anthropic.com/v1/messages`. Max tokens 900, max tries 2, error continuation enabled.
    - *Expressions:* `={{ JSON.stringify({ model: $json.anthropicModel, ... }) }}`
    - *Connections:* Input: `Parse Intent Classification Results` | Output: `Parse Drift Detection Report`
    - *Edge Cases:* API timeouts or invalid model responses.
  - **Parse Drift Detection Report**
    - *Type & Role:* `code` — Parses the drift detection report, evaluates confidence thresholds, appends the keyword report to cumulative execution results, and increments the processed count.
    - *Configuration:* JavaScript JSON parsing and report aggregation logic.
    - *Expressions:* Uses upstream HTTP response and state.
    - *Connections:* Input: `Detect Drift with Claude` | Output: `If Significant Drift Found`
    - *Edge Cases:* Unparsable AI responses defaulting to `driftDetected: false`.

---

#### 2.5 Alerting, Digest & Response
**Overview:** Evaluates whether detected drift is statistically significant, triggers WhatsApp alerts for anomalous shifts, builds a run summary digest, sends a WhatsApp digest, and returns the response to webhook callers.

- **Nodes Involved:**
  - `If Significant Drift Found`
  - `Send Drift Alert via WhatsApp`
  - `Build Final Drift Digest`
  - `Send Drift Digest via WhatsApp`
  - `Respond with Drift Digest via Webhook`

- **Node Details:**
  - **If Significant Drift Found**
    - *Type & Role:* `if` — Checks if the keyword experienced significant drift meeting confidence and history criteria.
    - *Configuration:* Evaluates `{{ $json.significantDrift }}` as boolean `true`.
    - *Expressions:* `{{ $json.significantDrift }}`
    - *Connections:* Input: `Parse Drift Detection Report` | Output (True): `Send Drift Alert via WhatsApp` | Output (False): `Retrieve Next Keyword from Queue`
    - *Edge Cases:* None.
  - **Send Drift Alert via WhatsApp**
    - *Type & Role:* `whatsApp` — Sends an instant WhatsApp notification detailing the intent drift, confidence level, narrative, and recommended actions.
    - *Configuration:* Operation `send`, using WhatsApp Business phone ID and recipient phone number.
    - *Expressions:* `={{ $json.currentKeyword }}`, `={{ $json.drift.fromIntent }}`, `={{ $json.whatsappBusinessPhoneId }}`, etc.
    - *Connections:* Input: `If Significant Drift Found` (True) | Output: `Retrieve Next Keyword from Queue`
    - *Edge Cases:* WhatsApp API rate limits, invalid phone ID, or recipient formatting errors.
  - **Build Final Drift Digest**
    - *Type & Role:* `code` — Aggregates all keyword reports into a final run digest, calculating total checked, keywords with drift, and top drift patterns.
    - *Configuration:* JavaScript aggregation and sorting logic.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `If Keyword Queue is Empty` (True) | Output: `Send Drift Digest via WhatsApp`
    - *Edge Cases:* Empty results array handled safely.
  - **Send Drift Digest via WhatsApp**
    - *Type & Role:* `whatsApp` — Sends a summary digest of the entire workflow execution over WhatsApp.
    - *Configuration:* Operation `send`, message body summarizing checked keywords and drift patterns.
    - *Expressions:* `={{ $json.keywordsChecked }}`, `={{ $json.keywordsWithDrift }}`, `={{ $json.whatsappBusinessPhoneId }}`
    - *Connections:* Input: `Build Final Drift Digest` | Output: `Respond with Drift Digest via Webhook`
    - *Edge Cases:* WhatsApp API failures.
  - **Respond with Drift Digest via Webhook**
    - *Type & Role:* `respondToWebhook` — Returns the final drift digest JSON payload to any HTTP caller that initiated an ad-hoc webhook run.
    - *Configuration:* Response type JSON, responding with `{{ $json }}`. `continueOnFail` enabled.
    - *Expressions:* `={{ $json }}`
    - *Connections:* Input: `Send Drift Digest via WhatsApp` | Output: None
    - *Edge Cases:* Connection closed prematurely by the HTTP client.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow overview and setup instructions | None | None | ## Search Intent Drift Detector... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Trigger and normalization documentation | None | None | ## Trigger and normalize input... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Run mode configuration documentation | None | None | ## Configure run mode... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Keyword queue building documentation | None | None | ## Build keyword queues... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Keyword loop control documentation | None | None | ## Control keyword loop... |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Final digest documentation | None | None | ## Send final digest... |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | SERP capture documentation | None | None | ## Capture current SERP... |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Historical metrics documentation | None | None | ## Build historical metrics... |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Intent classification documentation | None | None | ## Classify search intent... |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Keyword drift detection documentation | None | None | ## Detect keyword drift... |
| Sticky Note10 | `n8n-nodes-base.stickyNote` | Drift alerts documentation | None | None | ## Send drift alerts... |
| When Daily Drift Check Initiates | `n8n-nodes-base.scheduleTrigger` | Triggers execution daily | None | Normalize Trigger Input Data | Trigger and normalize input |
| When Keyword Registered for Drift | `n8n-nodes-base.webhook` | Receives ad-hoc keyword requests | None | Normalize Trigger Input Data | Trigger and normalize input |
| When Manual Test Run Initiates | `n8n-nodes-base.manualTrigger` | Triggers manual test runs | None | Normalize Trigger Input Data | Trigger and normalize input |
| Normalize Trigger Input Data | `n8n-nodes-base.code` | Normalizes input payloads across trigger types | When Daily Drift Check Initiates, When Keyword Registered for Drift, When Manual Test Run Initiates | Set Configuration Parameters | Trigger and normalize input |
| Set Configuration Parameters | `n8n-nodes-base.set` | Assigns global configuration parameters | Normalize Trigger Input Data | Determine Trigger Mode | Configure run mode |
| Determine Trigger Mode | `n8n-nodes-base.code` | Determines if run is ad-hoc | Set Configuration Parameters | If Ad Hoc Keyword Exists | Configure run mode |
| If Ad Hoc Keyword Exists | `n8n-nodes-base.if` | Branches based on trigger keyword presence | Determine Trigger Mode | Build Queue from Ad Hoc Keyword, Fetch Tracked Keywords | Configure run mode |
| Build Queue from Ad Hoc Keyword | `n8n-nodes-base.code` | Builds queue from single ad-hoc keyword | If Ad Hoc Keyword Exists | Retrieve Next Keyword from Queue | Build keyword queues |
| Fetch Tracked Keywords | `n8n-nodes-base.httpRequest` | Fetches tracked keywords from database API | If Ad Hoc Keyword Exists | Build Queue from Tracked Keywords | Build keyword queues |
| Build Queue from Tracked Keywords | `n8n-nodes-base.code` | Builds queue from fetched tracked keywords | Fetch Tracked Keywords | Retrieve Next Keyword from Queue | Build keyword queues |
| Retrieve Next Keyword from Queue | `n8n-nodes-base.code` | Shifts next keyword from queue | Build Queue from Ad Hoc Keyword, Build Queue from Tracked Keywords, If Significant Drift Found, Send Drift Alert via WhatsApp | If Keyword Queue is Empty | Control keyword loop |
| If Keyword Queue is Empty | `n8n-nodes-base.if` | Checks if queue is exhausted or limit reached | Retrieve Next Keyword from Queue | Build Final Drift Digest, Fetch Current SERP Snapshot | Control keyword loop |
| Fetch Current SERP Snapshot | `n8n-nodes-base.httpRequest` | Fetches live SERP data for active keyword | If Keyword Queue is Empty | Normalize SERP Snapshot Data | Capture current SERP |
| Normalize SERP Snapshot Data | `n8n-nodes-base.code` | Normalizes SERP results and feature flags | Fetch Current SERP Snapshot | Save Snapshot to Historical DB | Capture current SERP |
| Save Snapshot to Historical DB | `n8n-nodes-base.httpRequest` | Saves SERP snapshot to historical database | Normalize SERP Snapshot Data | Fetch Historical SERP Data | Capture current SERP |
| Fetch Historical SERP Data | `n8n-nodes-base.httpRequest` | Fetches historical SERP series | Save Snapshot to Historical DB | Build Historical Snapshot Timeline | Build historical metrics |
| Build Historical Snapshot Timeline | `n8n-nodes-base.code` | Assembles sorted historical timeline | Fetch Historical SERP Data | Compute Periodic Composition | Build historical metrics |
| Compute Periodic Composition | `n8n-nodes-base.code` | Computes content composition per period | Build Historical Snapshot Timeline | Compare Periodic Compositions | Build historical metrics |
| Compare Periodic Compositions | `n8n-nodes-base.code` | Compares composition and feature deltas | Compute Periodic Composition | Classify Intent with Claude | Build historical metrics |
| Classify Intent with Claude | `n8n-nodes-base.httpRequest` | Sends timeline to Claude for intent classification | Compare Periodic Compositions | Parse Intent Classification Results | Classify search intent |
| Parse Intent Classification Results | `n8n-nodes-base.code` | Parses Claude intent classification response | Classify Intent with Claude | Detect Drift with Claude | Classify search intent |
| Detect Drift with Claude | `n8n-nodes-base.httpRequest` | Evaluates intent drift with Claude | Parse Intent Classification Results | Parse Drift Detection Report | Detect keyword drift |
| Parse Drift Detection Report | `n8n-nodes-base.code` | Parses drift report and checks significance | Detect Drift with Claude | If Significant Drift Found | Detect keyword drift |
| If Significant Drift Found | `n8n-nodes-base.if` | Checks if drift is significant enough to alert | Parse Drift Detection Report | Send Drift Alert via WhatsApp, Retrieve Next Keyword from Queue | Send drift alerts |
| Send Drift Alert via WhatsApp | `n8n-nodes-base.whatsApp` | Sends WhatsApp alert for significant drift | If Significant Drift Found | Retrieve Next Keyword from Queue | Send drift alerts |
| Build Final Drift Digest | `n8n-nodes-base.code` | Builds final execution summary digest | If Keyword Queue is Empty | Send Drift Digest via WhatsApp | Send final digest |
| Send Drift Digest via WhatsApp | `n8n-nodes-base.whatsApp` | Sends run summary digest over WhatsApp | Build Final Drift Digest | Respond with Drift Digest via Webhook | Send final digest |
| Respond with Drift Digest via Webhook | `n8n-nodes-base.respondToWebhook` | Returns digest to webhook callers | Send Drift Digest via WhatsApp | None | Send final digest |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** (`scheduleTrigger`), name it `When Daily Drift Check Initiates`, and set the interval to 24 hours.
   - Add a **Webhook** (`webhook`), name it `When Keyword Registered for Drift`, set method to `POST`, path to `intent-drift-register-keyword`, and response mode to `responseNode`.
   - Add a **Manual Trigger** (`manualTrigger`) and name it `When Manual Test Run Initiates`.

2. **Add Normalization & Configuration Nodes:**
   - Add a **Code** node named `Normalize Trigger Input Data`. Paste JavaScript to extract `payload.keyword` or default to `'best crm software'` when empty.
   - Add a **Set** node named `Set Configuration Parameters`. Assign parameters: `historicalWindowSize` (6), `minPeriodsForDrift` (3), `driftConfidenceThreshold` (65), `maxKeywordsPerRun` (25), `serpApiUrl`, `historicalDbFetchUrl`, `historicalDbSaveUrl`, `trackedKeywordsUrl`, `anthropicModel` (`claude-sonnet-4-6`), `whatsappBusinessPhoneId`, and `whatsappRecipientPhone`.
   - Add a **Code** node named `Determine Trigger Mode` to set `isAdHocKeyword` based on `triggerKeyword`.

3. **Add Queue Control Nodes:**
   - Add an **If** node named `If Ad Hoc Keyword Exists` evaluating `{{ $json.isAdHocKeyword }}`.
   - Add a **Code** node named `Build Queue from Ad Hoc Keyword` (connected to True branch).
   - Add an **HTTP Request** node named `Fetch Tracked Keywords` (connected to False branch) using `httpHeaderAuth` targeting `{{ $json.trackedKeywordsUrl }}`.
   - Add a **Code** node named `Build Queue from Tracked Keywords` to parse the fetched keyword array.
   - Add a **Code** node named `Retrieve Next Keyword from Queue` to implement array shifting (`shift()`).
   - Add an **If** node named `If Keyword Queue is Empty` evaluating `hasNext` and `processedCount`.

4. **Add SERP Capture & Historical Nodes:**
   - Add an **HTTP Request** node named `Fetch Current SERP Snapshot` making a GET request to `serpApiUrl` with query parameters `q` and `num=20`. Configure `httpHeaderAuth` and `continueOnFail`.
   - Add a **Code** node named `Normalize SERP Snapshot Data` to parse organic results and feature flags.
   - Add an **HTTP Request** node named `Save Snapshot to Historical DB` performing a POST request to `historicalDbSaveUrl` with JSON body containing keyword, timestamp, results, and features.
   - Add an **HTTP Request** node named `Fetch Historical SERP Data` performing a GET request to `historicalDbFetchUrl` with query parameters `keyword` and `limit`.
   - Add a **Code** node named `Build Historical Snapshot Timeline` to sort snapshots and verify history length.

5. **Add AI Analysis Nodes:**
   - Add a **Code** node named `Compute Periodic Composition` to classify result titles/domains into content types.
   - Add a **Code** node named `Compare Periodic Compositions` to calculate earliest-vs-latest deltas.
   - Add an **HTTP Request** node named `Classify Intent with Claude` performing a POST request to `https://api.anthropic.com/v1/messages` using `anthropicModel`. Set max tokens to 1000, max tries to 2, and configure generic HTTP Header Auth for Anthropic.
   - Add a **Code** node named `Parse Intent Classification Results` to clean and parse Claude's JSON response.
   - Add an **HTTP Request** node named `Detect Drift with Claude` performing a POST request to `https://api.anthropic.com/v1/messages` with max tokens 900.
   - Add a **Code** node named `Parse Drift Detection Report` to evaluate confidence thresholds and update counters.

6. **Add Alerting & Response Nodes:**
   - Add an **If** node named `If Significant Drift Found` evaluating `{{ $json.significantDrift }}`.
   - Connect the True branch to a **WhatsApp** node named `Send Drift Alert via WhatsApp` (Operation: `send`, using `whatsappBusinessPhoneId` and `whatsappRecipientPhone`). Route its output back to `Retrieve Next Keyword from Queue`. Route the False branch of `If Significant Drift Found` back to `Retrieve Next Keyword from Queue` as well.
   - Connect the True branch of `If Keyword Queue is Empty` to a **Code** node named `Build Final Drift Digest`.
   - Connect `Build Final Drift Digest` to a **WhatsApp** node named `Send Drift Digest via WhatsApp`.
   - Connect `Send Drift Digest via WhatsApp` to a **Respond to Webhook** node named `Respond with Drift Digest via Webhook` configured to respond with JSON `{{ $json }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Search Intent Drift Detector Workflow Overview | Workflow documentation and configuration guide |