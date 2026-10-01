Track SERP winners and losers with a SERP API, Claude, and WhatsApp alerts

https://n8nworkflows.xyz/workflows/track-serp-winners-and-losers-with-a-serp-api--claude--and-whatsapp-alerts-20008


# Track SERP winners and losers with a SERP API, Claude, and WhatsApp alerts

### 1. Workflow Overview

This workflow automates the monitoring, historical comparison, AI-driven analysis, and alerting of Search Engine Results Pages (SERP) for designated keywords. Its primary purpose is to track visibility changes for a target domain and its competitors, identify ranking winners and losers, interpret broader SERP patterns using Anthropic Claude, dispatch immediate alerts for significant ranking events via WhatsApp, and aggregate run data into a final digest.

The logic is structured into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingests execution signals from a daily schedule, manual test run, or an on-demand webhook, normalizes the payload, and instantiates global configuration variables.
- **1.2 Queue Construction & Iteration Control:** Determines whether the run is ad-hoc or scheduled, builds a processing queue (either from a single webhook parameter or by querying a historical database for all tracked keywords), and manages the queue loop.
- **1.3 SERP Data Extraction & Historical Diffs:** Fetches current top-100 search results from an external SERP provider, normalizes them, retrieves the last historical snapshot from the database, calculates position deltas (winners, losers, new entrants, dropouts), and stores the new snapshot.
- **1.4 AI Analysis & Reporting:** Passes winner/loser shortlists to Anthropic Claude to determine commonalities and notable movements, runs a secondary Claude prompt to classify SERP behavior patterns, and compiles a per-keyword analysis report.
- **1.5 Alerting & Digest Dispatch:** Evaluates whether the target domain underwent significant movement and triggers immediate WhatsApp alerts if thresholds are breached, looping back until the queue is exhausted, whereupon a final run digest is compiled, transmitted via WhatsApp, and returned via webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block standardizes incoming execution triggers from multiple sources (schedule, webhook, or manual test) into a uniform data structure and loads environment configuration properties.
- **Nodes Involved:** 
  - `Daily SERP Check Trigger`
  - `Register Keyword Webhook`
  - `Test Run Manual Trigger`
  - `Normalize Trigger Input`
  - `Set Configuration Parameters`
  - `Determine Trigger Mode`

- **Node Details:**
  - **Daily SERP Check Trigger**
    - *Type & Role:* Schedule Trigger node executing every 24 hours.
    - *Configuration:* Interval set to 24 hours.
    - *Connections:* Output connects to `Normalize Trigger Input`.
    - *Failure Types:* None standard; relies on n8n scheduler uptime.
  - **Register Keyword Webhook**
    - *Type & Role:* Webhook node listening for incoming HTTP POST requests containing ad-hoc keywords.
    - *Configuration:* Path `serp-battlefield-register-keyword`, method `POST`, response mode handled by webhook response node.
    - *Connections:* Output connects to `Normalize Trigger Input`.
    - *Failure Types:* Network routing errors or malformed payloads.
  - **Test Run Manual Trigger**
    - *Type & Role:* Manual Trigger node for testing workflow logic interactively.
    - *Configuration:* Default manual trigger parameters.
    - *Connections:* Output connects to `Normalize Trigger Input`.
    - *Failure Types:* None.
  - **Normalize Trigger Input**
    - *Type & Role:* Code node (JavaScript) extracting and sanitizing trigger payloads.
    - *Configuration:* Parses `body` or root payload; defaults to placeholder keyword (`best project management software`) and domain (`example.com`) if payload is empty.
    - *Expressions:* Uses standard JavaScript property checks (`payload.keyword`, `payload.targetDomain`).
    - *Connections:* Input from all three triggers; output to `Set Configuration Parameters`.
    - *Failure Types:* Script execution errors if incoming JSON is unstructured.
  - **Set Configuration Parameters**
    - *Type & Role:* Set (Edit Fields) node defining runtime parameters.
    - *Configuration:* Assigns variables including movement threshold (`3`), top band size (`10`), max keywords per run (`25`), API endpoints, Anthropic model (`claude-sonnet-4-6`), and WhatsApp configuration properties.
    - *Connections:* Input from `Normalize Trigger Input`; output to `Determine Trigger Mode`.
    - *Failure Types:* Variable name mismatch in downstream nodes.
  - **Determine Trigger Mode**
    - *Type & Role:* Code node identifying execution mode.
    - *Configuration:* Evaluates whether `triggerKeyword` is present and sets boolean flag `isAdHocKeyword`.
    - *Connections:* Input from `Set Configuration Parameters`; output to `If Ad Hoc Keyword Provided`.
    - *Failure Types:* None.

---

#### 2.2 Queue Construction & Iteration Control
- **Overview:** Evaluates execution context to build a work queue containing either a single ad-hoc keyword or a full list of tracked keywords fetched from the historical database.
- **Nodes Involved:**
  - `If Ad Hoc Keyword Provided`
  - `Build Queue from Trigger Keyword`
  - `Fetch Tracked Keywords`
  - `Build Queue from Tracked Keywords`
  - `Get Next Keyword from Queue`
  - `If Keyword Queue Empty`

- **Node Details:**
  - **If Ad Hoc Keyword Provided**
    - *Type & Role:* If node branching logic based on trigger type.
    - *Configuration:* Evaluates `{{ $json.isAdHocKeyword }}` as true (ad-hoc) or false (scheduled/bulk).
    - *Connections:* Input from `Determine Trigger Mode`; True output to `Build Queue from Trigger Keyword`, False output to `Fetch Tracked Keywords`.
    - *Failure Types:* Evaluation syntax error.
  - **Build Queue from Trigger Keyword**
    - *Type & Role:* Code node initializing a single-item queue.
    - *Configuration:* Wraps `triggerKeyword` and `triggerTargetDomain` into an array named `queue`.
    - *Connections:* Input from `If Ad Hoc Keyword Provided` (True); output to `Get Next Keyword from Queue`.
    - *Failure Types:* None.
  - **Fetch Tracked Keywords**
    - *Type & Role:* HTTP Request node fetching the list of monitored SEO keywords from the historical database API.
    - *Configuration:* Uses HTTP Header Authentication, URL dynamically mapped from `{{ $json.trackedKeywordsUrl }}`. `continueOnFail` enabled.
    - *Connections:* Input from `If Ad Hoc Keyword Provided` (False); output to `Build Queue from Tracked Keywords`.
    - *Failure Types:* API timeouts, authentication failure, or invalid response schema.
  - **Build Queue from Tracked Keywords**
    - *Type & Role:* Code node mapping fetched API records into execution items.
    - *Configuration:* Normalizes response arrays into objects containing `keyword` and `targetDomain`.
    - *Connections:* Input from `Fetch Tracked Keywords`; output to `Get Next Keyword from Queue`.
    - *Failure Types:* Property mapping failure if API response structure changes.
  - **Get Next Keyword from Queue**
    - *Type & Role:* Code node operating as a queue consumer.
    - *Configuration:* Shifts the next keyword object from the `queue` array, updating state properties (`currentKeyword`, `currentTargetDomain`, `hasNext`).
    - *Connections:* Inputs from `Build Queue from Trigger Keyword`, `Build Queue from Tracked Keywords`, `If Target Domain Significant Move`, and `Send Battlefield Alert via WhatsApp`; output to `If Keyword Queue Empty`.
    - *Failure Types:* Null reference exceptions if queue is uninitialized.
  - **If Keyword Queue Empty**
    - *Type & Role:* If node controlling loop termination.
    - *Configuration:* Evaluates whether `hasNext` is false or if `processedCount` meets or exceeds `maxKeywordsPerRun`.
    - *Connections:* Input from `Get Next Keyword from Queue`; True output to `Build Final Battlefield Digest`, False output to `Fetch SERP Snapshot`.
    - *Failure Types:* Infinite loop if queue state fails to decrement.

---

#### 2.3 SERP Data Extraction & Historical Diffs
- **Overview:** Queries the SERP provider for ranking data, extracts current standings, compares them against previous database snapshots to calculate deltas, and saves the current snapshot.
- **Nodes Involved:**
  - `Fetch SERP Snapshot`
  - `Normalize SERP Snapshot`
  - `Fetch Previous Snapshot`
  - `Detect SERP Movement`
  - `Save Snapshot to Historical DB`

- **Node Details:**
  - **Fetch SERP Snapshot**
    - *Type & Role:* HTTP Request node calling the external SERP API.
    - *Configuration:* HTTP Header Auth, queries parameter `q` with `{{ $json.currentKeyword }}` and `num` set to `100`. `continueOnFail` enabled.
    - *Connections:* Input from `If Keyword Queue Empty` (False); output to `Normalize SERP Snapshot`.
    - *Failure Types:* API rate limiting, network failure, or invalid API credentials.
  - **Normalize SERP Snapshot**
    - *Type & Role:* Code node parsing provider responses.
    - *Configuration:* Extracts organic results (up to 100), isolates domains (using `URL` constructor fallback), and assigns positions.
    - *Connections:* Input from `Fetch SERP Snapshot`; output to `Fetch Previous Snapshot`.
    - *Failure Types:* Malformed SERP payload lacking organic result arrays.
  - **Fetch Previous Snapshot**
    - *Type & Role:* HTTP Request node querying the historical database for prior search results.
    - *Configuration:* HTTP Header Auth, requests endpoint `historicalDbFetchUrl` with query parameters `keyword` and `limit=1`. `continueOnFail` enabled.
    - *Connections:* Input from `Normalize SERP Snapshot`; output to `Detect SERP Movement`.
    - *Failure Types:* Database connection drops or missing records for new keywords.
  - **Detect SERP Movement**
    - *Type & Role:* Code node performing diff analysis between current and historical snapshots.
    - *Configuration:* Maps domains, identifies status changes (`tracked`, `new_entrant`, `dropped_out`), computes position deltas, and isolates big movers, top-band changes, and target domain movement.
    - *Connections:* Input from `Fetch Previous Snapshot`; output to `Save Snapshot to Historical DB`.
    - *Failure Types:* Type conversion issues on position numbers.
  - **Save Snapshot to Historical DB**
    - *Type & Role:* HTTP Request node persisting the current SERP state.
    - *Configuration:* POST request to `historicalDbSaveUrl` sending JSON body with keyword, domain, timestamp, and results array. Generic Header Auth. `continueOnFail` enabled.
    - *Connections:* Input from `Detect SERP Movement`; output to `Identify Winners and Losers`.
    - *Failure Types:* Database write rejections, payload size limitations, or auth failures.

---

#### 2.4 AI Analysis & Reporting
- **Overview:** Summarizes SERP shifts and submits structured movement data to Anthropic Claude to determine winner/loser characteristics and broader SERP behavior patterns.
- **Nodes Involved:**
  - `Identify Winners and Losers`
  - `Analyze Winners/Losers with Claude`
  - `Parse Winners/Losers Analysis`
  - `Determine Pattern with Claude`
  - `Parse Pattern & Build Report`

- **Node Details:**
  - **Identify Winners and Losers**
    - *Type & Role:* Code node shortlisting ranking movements.
    - *Configuration:* Sorts and slices top 10 winners and top 10 losers based on position delta or entry/dropout status.
    - *Connections:* Input from `Save Snapshot to Historical DB`; output to `Analyze Winners/Losers with Claude`.
    - *Failure Types:* Empty movement arrays.
  - **Analyze Winners/Losers with Claude**
    - *Type & Role:* HTTP Request node calling Anthropic's Messages API.
    - *Configuration:* POST to `https://api.anthropic.com/v1/messages` using generic HTTP Header Auth. Model configured via `anthropicModel`. Max tokens `800`. Retry on fail enabled (2 tries, 3s delay). Error output mapped via `continueErrorOutput`.
    - *Connections:* Input from `Identify Winners and Losers`; output to `Parse Winners/Losers Analysis`.
    - *Failure Types:* API rate limits, refusal errors, or malformed JSON generation by the model.
  - **Parse Winners/Losers Analysis**
    - *Type & Role:* Code node parsing AI output.
    - *Configuration:* Cleans markdown code fences (` ```json `) and parses the JSON payload containing characteristics and notable moves.
    - *Connections:* Input from `Analyze Winners/Losers with Claude`; output to `Determine Pattern with Claude`.
    - *Failure Types:* JSON parsing syntax failures if the LLM output contains stray text.
  - **Determine Pattern with Claude**
    - *Type & Role:* HTTP Request node invoking Anthropic API for strategic classification.
    - *Configuration:* POST to Anthropic Messages API, passing winner/loser insights, target domain movement, and big mover counts to classify SERP behavior patterns and recommend actions. Max tokens `800`. Retry enabled. Error output mapped via `continueErrorOutput`.
    - *Connections:* Input from `Parse Winners/Losers Analysis`; output to `Parse Pattern & Build Report`.
    - *Failure Types:* LLM timeout or invalid API token.
  - **Parse Pattern & Build Report**
    - *Type & Role:* Code node finalizing keyword-level analytics.
    - *Configuration:* Parses pattern classification JSON, evaluates whether target domain movement breaches significance thresholds, appends the report to results array, and increments `processedCount`.
    - *Connections:* Input from `Determine Pattern with Claude`; output to `If Target Domain Significant Move`.
    - *Failure Types:* Type validation errors on confidence scores.

---

#### 2.5 Alerting & Digest Dispatch
- **Overview:** Checks if the target domain experienced significant volatility, dispatches individual WhatsApp alerts when triggered, loops back to process remaining queue items, and compiles/returns the final run digest.
- **Nodes Involved:**
  - `If Target Domain Significant Move`
  - `Send Battlefield Alert via WhatsApp`
  - `Build Final Battlefield Digest`
  - `Send Battlefield Digest via WhatsApp`
  - `Return Digest via Webhook`

- **Node Details:**
  - **If Target Domain Significant Move**
    - *Type & Role:* If node evaluating alert criteria.
    - *Configuration:* Checks boolean flag `targetDomainSignificantMove`.
    - *Connections:* Input from `Parse Pattern & Build Report`; True output to `Send Battlefield Alert via WhatsApp`, False output to `Get Next Keyword from Queue`.
    - *Failure Types:* None.
  - **Send Battlefield Alert via WhatsApp**
    - *Type & Role:* WhatsApp Business Cloud node sending instant alerts.
    - *Configuration:* Operation `send`, uses dynamic expressions for `phoneNumberId`, `recipientPhoneNumber`, and formats a message body containing position changes and pattern summaries.
    - *Connections:* Input from `If Target Domain Significant Move` (True); output loops to `Get Next Keyword from Queue`.
    - *Failure Types:* Invalid WhatsApp credentials, expired token, or phone number formatting issues.
  - **Build Final Battlefield Digest**
    - *Type & Role:* Code node aggregating execution telemetry across all processed keywords.
    - *Configuration:* Computes total keywords tracked, significant movement counts, and top pattern frequencies.
    - *Connections:* Input from `If Keyword Queue Empty` (True); output to `Send Battlefield Digest via WhatsApp`.
    - *Failure Types:* Unhandled empty results arrays.
  - **Send Battlefield Digest via WhatsApp**
    - *Type & Role:* WhatsApp Business Cloud node dispatching run summaries.
    - *Configuration:* Operation `send`, transmits aggregated digest stats to recipient.
    - *Connections:* Input from `Build Final Battlefield Digest`; output to `Return Digest via Webhook`.
    - *Failure Types:* WhatsApp API failures.
  - **Return Digest via Webhook**
    - *Type & Role:* Respond to Webhook node returning execution output.
    - *Configuration:* Responds with JSON payload containing the complete digest. `continueOnFail` enabled.
    - *Connections:* Input from `Send Battlefield Digest via WhatsApp`; terminal node.
    - *Failure Types:* Webhook connection closed prematurely by caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation & setup guide | None | None | ## SERP Battlefield<br><br>### How it works<br><br>This workflow monitors search results for tracked or ad hoc keywords, compares each SERP snapshot with historical data, and detects ranking movement for a target domain and competitors. It loops through a keyword queue, uses Claude to analyze winners, losers, and broader SERP patterns, sends WhatsApp alerts for significant target-domain movement, and produces a final battlefield digest when the queue is complete.<br><br>### Setup steps<br><br>- Configure the schedule trigger frequency and enable the webhook URL if keywords will be registered or tested on demand.<br>- Set the tracked keywords endpoint, SERP API endpoint, historical database fetch/save URLs, target domain, movement threshold, top-band size, and maximum keywords per run in the configuration/code nodes.<br>- Add valid credentials or headers for the SERP provider, historical database API, Anthropic Claude API, and WhatsApp node.<br>- Test with the manual trigger and a small keyword set before enabling the daily schedule.<br><br>### Customization<br><br>Adjust significantMovementThreshold, topBandSize, maxKeywordsPerRun, target-domain logic, Claude prompts, and WhatsApp message templates to match the SEO monitoring strategy. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Trigger intake documentation | None | None | ## Trigger intake<br><br>Starts the workflow from a daily schedule, webhook keyword registration, or manual test run, then normalizes the incoming trigger payload into a common format. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Run mode configuration documentation | None | None | ## Configure run mode<br><br>Applies workflow settings, determines whether the run is ad hoc or scheduled, and branches based on whether a keyword was supplied directly. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Keyword queue builder documentation | None | None | ## Build keyword queue<br><br>Creates the processing queue either from the trigger keyword or by fetching the tracked keyword list and converting it into queued work items. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Queue advancement documentation | None | None | ## Advance keyword queue<br><br>Pulls the next keyword from the queue and decides whether to continue SERP analysis or finish the run because all keywords have been processed. |
| Sticky Note5 | n8n-nodes-base.stickyNote | SERP fetch documentation | None | None | ## Fetch current SERP<br><br>Requests the latest SERP snapshot for the active keyword and normalizes the provider response for downstream comparison. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Historical comparison documentation | None | None | ## Compare historical snapshots<br><br>Retrieves the previous snapshot, calculates ranking movement against the current SERP, and saves the new snapshot back to the historical database. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Winners/losers analysis documentation | None | None | ## Analyze winners losers<br><br>Identifies notable ranking winners and losers, sends the movement summary to Claude, and parses the returned competitive analysis. |
| Sticky Note8 | n8n-nodes-base.stickyNote | SERP pattern detection documentation | None | None | ## Detect SERP pattern<br><br>Uses a second Claude analysis to determine the broader SERP pattern and builds the per-keyword battlefield report. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Movement alert documentation | None | None | ## Send movement alert<br><br>Checks whether the target domain moved significantly and, when it did, sends a WhatsApp battlefield alert before looping back for the next keyword. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Final digest documentation | None | None | ## Send final digest<br><br>Builds the completed battlefield digest after the queue is empty, sends it via WhatsApp, and returns the digest to the webhook caller. |
| Daily SERP Check Trigger | n8n-nodes-base.scheduleTrigger | Executes workflow every 24 hours | None | Normalize Trigger Input | |
| Register Keyword Webhook | n8n-nodes-base.webhook | Ingests ad-hoc keywords via HTTP POST | None | Normalize Trigger Input | |
| Test Run Manual Trigger | n8n-nodes-base.manualTrigger | Initiates manual test runs | None | Normalize Trigger Input | |
| Normalize Trigger Input | n8n-nodes-base.code | Normalizes trigger payloads | Daily SERP Check Trigger, Register Keyword Webhook, Test Run Manual Trigger | Set Configuration Parameters | |
| Set Configuration Parameters | n8n-nodes-base.set | Defines global variables and API endpoints | Normalize Trigger Input | Determine Trigger Mode | |
| Determine Trigger Mode | n8n-nodes-base.code | Flags execution mode as ad-hoc or bulk | Set Configuration Parameters | If Ad Hoc Keyword Provided | |
| If Ad Hoc Keyword Provided | n8n-nodes-base.if | Branches execution based on trigger type | Determine Trigger Mode | Build Queue from Trigger Keyword, Fetch Tracked Keywords | |
| Build Queue from Trigger Keyword | n8n-nodes-base.code | Creates single-item queue from webhook/test | If Ad Hoc Keyword Provided | Get Next Keyword from Queue | |
| Fetch Tracked Keywords | n8n-nodes-base.httpRequest | Retrieves monitored keyword list from database | If Ad Hoc Keyword Provided | Build Queue from Tracked Keywords | |
| Build Queue from Tracked Keywords | n8n-nodes-base.code | Converts fetched keywords into queue items | Fetch Tracked Keywords | Get Next Keyword from Queue | |
| Get Next Keyword from Queue | n8n-nodes-base.code | Pulls next item from processing queue | Build Queue from Trigger Keyword, Build Queue from Tracked Keywords, If Target Domain Significant Move, Send Battlefield Alert via WhatsApp | If Keyword Queue Empty | |
| If Keyword Queue Empty | n8n-nodes-base.if | Controls loop termination based on queue size | Get Next Keyword from Queue | Build Final Battlefield Digest, Fetch SERP Snapshot | |
| Fetch SERP Snapshot | n8n-nodes-base.httpRequest | Fetches top-100 results from SERP provider API | If Keyword Queue Empty | Normalize SERP Snapshot | |
| Normalize SERP Snapshot | n8n-nodes-base.code | Structures raw SERP items into standard format | Fetch SERP Snapshot | Fetch Previous Snapshot | |
| Fetch Previous Snapshot | n8n-nodes-base.httpRequest | Retrieves latest prior snapshot from database | Normalize SERP Snapshot | Detect SERP Movement | |
| Detect SERP Movement | n8n-nodes-base.code | Computes position deltas and movement categories | Fetch Previous Snapshot | Save Snapshot to Historical DB | |
| Save Snapshot to Historical DB | n8n-nodes-base.httpRequest | Persists current SERP snapshot via POST | Detect SERP Movement | Identify Winners and Losers | |
| Identify Winners and Losers | n8n-nodes-base.code | Shortlists top 10 winners and losers | Save Snapshot to Historical DB | Analyze Winners/Losers with Claude | |
| Analyze Winners/Losers with Claude | n8n-nodes-base.httpRequest | Sends movement data to Anthropic Claude | Identify Winners and Losers | Parse Winners/Losers Analysis | |
| Parse Winners/Losers Analysis | n8n-nodes-base.code | Parses Claude's winner/loser response | Analyze Winners/Losers with Claude | Determine Pattern with Claude | |
| Determine Pattern with Claude | n8n-nodes-base.httpRequest | Requests SERP pattern classification from Claude | Parse Winners/Losers Analysis | Parse Pattern & Build Report | |
| Parse Pattern & Build Report | n8n-nodes-base.code | Compiles per-keyword report and updates counters | Determine Pattern with Claude | If Target Domain Significant Move | |
| If Target Domain Significant Move | n8n-nodes-base.if | Evaluates if target domain movement is significant | Parse Pattern & Build Report | Send Battlefield Alert via WhatsApp, Get Next Keyword from Queue | |
| Send Battlefield Alert via WhatsApp | n8n-nodes-base.whatsApp | Sends instant WhatsApp alert for significant moves | If Target Domain Significant Move | Get Next Keyword from Queue | |
| Build Final Battlefield Digest | n8n-nodes-base.code | Aggregates run telemetry across all keywords | If Keyword Queue Empty | Send Battlefield Digest via WhatsApp | |
| Send Battlefield Digest via WhatsApp | n8n-nodes-base.whatsApp | Dispatches summary digest via WhatsApp | Build Final Battlefield Digest | Return Digest via Webhook | |
| Return Digest via Webhook | n8n-nodes-base.respondToWebhook | Returns execution payload to webhook caller | Send Battlefield Digest via WhatsApp | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Entry Triggers:**
   - Add a **Schedule Trigger** node (`Daily SERP Check Trigger`), configure interval to 24 hours.
   - Add a **Webhook** node (`Register Keyword Webhook`), set HTTP method to `POST`, path to `serp-battlefield-register-keyword`, and response mode to "Response Node".
   - Add a **Manual Trigger** node (`Test Run Manual Trigger`).

2. **Add Normalization & Configuration:**
   - Create a **Code** node (`Normalize Trigger Input`). Connect all three triggers to its input. Insert JavaScript to parse incoming payloads and fallback to default test strings (`best project management software` / `example.com`).
   - Create a **Set** node (`Set Configuration Parameters`). Assign parameters: `significantMovementThreshold` (3), `topBandSize` (10), `maxKeywordsPerRun` (25), `serpApiUrl`, `historicalDbFetchUrl`, `historicalDbSaveUrl`, `trackedKeywordsUrl`, `anthropicModel` (`claude-sonnet-4-6`), `whatsappBusinessPhoneId`, and `whatsappRecipientPhone`.
   - Create a **Code** node (`Determine Trigger Mode`) to set `isAdHocKeyword` based on the presence of a trigger keyword.

3. **Build Queue Logic:**
   - Create an **If** node (`If Ad Hoc Keyword Provided`) evaluating `{{ $json.isAdHocKeyword }}`.
   - For the True branch, add a **Code** node (`Build Queue from Trigger Keyword`) to initialize an array with the single trigger keyword.
   - For the False branch, add an **HTTP Request** node (`Fetch Tracked Keywords`) using HTTP Header Auth to call `{{ $json.trackedKeywordsUrl }}`, followed by a **Code** node (`Build Queue from Tracked Keywords`) to map response items into queue objects.
   - Create a consumer **Code** node (`Get Next Keyword from Queue`) that shifts items from the queue.
   - Create an **If** node (`If Keyword Queue Empty`) checking if `hasNext` is false or `processedCount >= maxKeywordsPerRun`.

4. **Implement SERP Extraction & Diff Engine:**
   - If the queue is not empty, connect to an **HTTP Request** node (`Fetch SERP Snapshot`) using HTTP Header Auth, querying `serpApiUrl` with `q` (`{{ $json.currentKeyword }}`) and `num` (`100`). Enable `continueOnFail`.
   - Add a **Code** node (`Normalize SERP Snapshot`) to format organic results and extract domains.
   - Add an **HTTP Request** node (`Fetch Previous Snapshot`) to query historical data via `historicalDbFetchUrl` with query parameter `keyword`.
   - Add a **Code** node (`Detect SERP Movement`) to calculate position changes, new entrants, dropouts, and big movers.
   - Add an **HTTP Request** node (`Save Snapshot to Historical DB`), method `POST`, sending JSON body containing keyword, domain, timestamp, and results array via HTTP Header Auth.

5. **Integrate AI Analysis Nodes:**
   - Add a **Code** node (`Identify Winners and Losers`) to extract top 10 winners and top 10 losers.
   - Add an **HTTP Request** node (`Analyze Winners/Losers with Claude`) targeting `https://api.anthropic.com/v1/messages` (POST). Configure Generic HTTP Header Auth for Anthropic, header `anthropic-version: 2023-06-01`, and prompt model with max tokens `800`. Enable retry and error outputs.
   - Add a **Code** node (`Parse Winners/Losers Analysis`) to parse the JSON output from Claude.
   - Add an **HTTP Request** node (`Determine Pattern with Claude`) targeting the same Anthropic endpoint with a secondary prompt classifying SERP behavior patterns and recommended actions.
   - Add a **Code** node (`Parse Pattern & Build Report`) to parse the pattern response, evaluate target domain significance, and increment processed counters.

6. **Configure Alerting & Final Digest:**
   - Add an **If** node (`If Target Domain Significant Move`) evaluating `targetDomainSignificantMove`.
   - If True, connect to a **WhatsApp** node (`Send Battlefield Alert via WhatsApp`) configured with operation `send`, dynamic phone number ID, and formatted message body. Connect its output back to `Get Next Keyword from Queue` to continue the loop. If False, connect directly back to `Get Next Keyword from Queue`.
   - When the queue is empty, connect the True output of `If Keyword Queue Empty` to a **Code** node (`Build Final Battlefield Digest`) to aggregate run metrics.
   - Connect to a **WhatsApp** node (`Send Battlefield Digest via WhatsApp`) to dispatch the summary.
   - Conclude with a **Respond to Webhook** node (`Return Digest via Webhook`) returning the digest JSON.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Scope | Designed for automated SEO tracking, SERP volatility alerts, and competitive intelligence reporting via n8n, Anthropic Claude, and WhatsApp Business API. |