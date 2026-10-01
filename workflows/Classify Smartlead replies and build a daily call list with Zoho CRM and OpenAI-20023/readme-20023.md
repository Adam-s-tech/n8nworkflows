Classify Smartlead replies and build a daily call list with Zoho CRM and OpenAI

https://n8nworkflows.xyz/workflows/classify-smartlead-replies-and-build-a-daily-call-list-with-zoho-crm-and-openai-20023


# Classify Smartlead replies and build a daily call list with Zoho CRM and OpenAI

### 1. Workflow Overview

This workflow automates the handling of incoming email replies from Smartlead campaigns. Its primary purpose is to parse responses, clean up email threads, classify the intent of the reply (using OpenAI or keyword-fallback rules), update lead statuses and create follow-up tasks in Zoho CRM, handle opt-outs via the Smartlead API, alert the team via an incoming webhook (Slack/Teams style), and generate a prioritized daily call list every weekday morning.

The workflow logic is divided into two primary execution paths triggered from a central configuration hub:
1. **Reply Processing Path:** Triggered by live Smartlead webhooks or manual demo execution, it normalizes reply text, classifies intent (`interested`, `not_now`, `referral`, `unsubscribe`, `out_of_office`, `needs_human`), writes back changes to Zoho CRM, creates tasks linked to lead IDs, processes global unsubscribes in Smartlead, and dispatches team notifications.
2. **Morning Call List Path:** Triggered on weekdays at 07:30 or via a manual run, it connects to Zoho CRM via COQL, queries relevant lead statuses, ranks prospects based on engagement tier and score, and posts a formatted call list to the team channel.

---

### 2. Block-by-Block Analysis

---

### Block 2.1: Entry Points & Central Configuration
- **Overview:** Receives triggers from webhooks, schedules, or manual actions, and passes the payload through a centralized configuration node that defines API endpoints, behavior flags, and operational thresholds.
- **Nodes Involved:** 
  - `A Smartlead reply arrives`
  - `Run the demo replies`
  - `Every weekday, 07:30`
  - `Build the call list now`
  - `load the demo replies`
  - `config`

#### Node Details:
- **A Smartlead reply arrives**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Webhook trigger)
  - **Configuration:** Listens for HTTP POST requests on path `smartlead-reply` with response mode set to "On Received".
  - **Expressions/Variables:** Uses incoming webhook body properties.
  - **Connections:** Input: None; Output: `config`
  - **Edge Cases / Failures:** Network timeout, malformed JSON payloads from upstream webhook sources.

- **Run the demo replies**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Manual trigger)
  - **Configuration:** Executes on-demand for testing purposes.
  - **Connections:** Input: None; Output: `load the demo replies`

- **Every weekday, 07:30**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Cron schedule trigger)
  - **Configuration:** Executes using expression `30 7 * * 1-5` (Monday through Friday at 07:30).
  - **Connections:** Input: None; Output: `config`

- **Build the call list now**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Manual trigger)
  - **Configuration:** Executes on-demand to generate the morning call list immediately.
  - **Connections:** Input: None; Output: `config`

- **load the demo replies**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Generates six mock Smartlead reply payloads simulating various intents (`interested`, `not_now`, `referral`, `unsubscribe`, `out_of_office`, `needs_human`).
  - **Connections:** Input: `Run the demo replies`; Output: `config`

- **config**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Acts as the single source of truth for configuration variables (API keys, flags like `TEST_RUN`, field mappings, classification classes, and webhook URLs).
  - **Expressions/Variables:** Evaluates global settings objects and packages incoming payloads under `incoming`.
  - **Connections:** Input: `A Smartlead reply arrives`, `Every weekday, 07:30`, `Build the call list now`, `load the demo replies`; Output: `building the call list?`

---

### Block 2.2: Reply Ingestion & Classification Logic
- **Overview:** Evaluates the workflow route, extracts and sanitizes email reply content by stripping quoted reply histories, and determines classification via OpenAI (with strict JSON schema enforcement) or keyword fallback rules.
- **Nodes Involved:** 
  - `building the call list?`
  - `read the reply`
  - `AI configured?`
  - `build the AI request`
  - `classify by keywords`
  - `OpenAI - classify the reply`
  - `apply the AI reading`

#### Node Details:
- **building the call list?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Evaluates whether execution originated from call list triggers (`Every weekday, 07:30` or `Build the call list now`).
  - **Expressions/Variables:** `={{ $('Every weekday, 07:30').isExecuted || $('Build the call list now').isExecuted }}`
  - **Connections:** Input: `config`; Output (True): `Zoho connected for the call list?`; Output (False): `read the reply`

- **read the reply**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Normalizes email lead fields and runs a regex/string parser (`stripQuoted`) to remove email thread history.
  - **Connections:** Input: `building the call list?`; Output: `AI configured?`

- **AI configured?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks if an OpenAI API key is present in configuration (`ai_enabled`).
  - **Expressions/Variables:** `={{ $('config').first().json.ai_enabled }}`
  - **Connections:** Input: `read the reply`; Output (True): `build the AI request`; Output (False): `classify by keywords`

- **build the AI request**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Prepares structured chat completion payloads with strict JSON schema definitions for intent classification.
  - **Connections:** Input: `AI configured?`; Output: `OpenAI - classify the reply`

- **classify by keywords**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Applies regex-based fallback keyword matching rules to determine classification and confidence scores.
  - **Connections:** Input: `AI configured?`; Output: `decide the follow-up`

- **OpenAI - classify the reply**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Sends structured prompts to OpenAI endpoints (`openai_url`) using Bearer token authentication.
  - **Expressions/Variables:** URL from `config`, JSON body from `build the AI request`.
  - **Connections:** Input: `build the AI request`; Output: `apply the AI reading`
  - **Edge Cases / Failures:** API rate limits, invalid tokens, JSON parsing errors (handled via `continueRegularOutput`).

- **apply the AI reading**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Parses OpenAI responses, validates confidence thresholds against `min_ai_confidence`, discards unverified referral emails, and ensures unsubscribe rules take precedence.
  - **Connections:** Input: `OpenAI - classify the reply`; Output: `decide the follow-up`

---

### Block 2.3: Follow-Up Action & Zoho CRM Synchronization
- **Overview:** Maps classified email intent to corresponding Zoho CRM lead status updates, creates due-dated task payloads, exchanges OAuth tokens, upserts lead records, and generates corresponding CRM tasks linked by record IDs.
- **Nodes Involved:** 
  - `decide the follow-up`
  - `build the Zoho updates`
  - `writing to Zoho?`
  - `Zoho - get an access token`
  - `STOP: preview - Zoho updates shown`
  - `read the access token`
  - `authenticated?`
  - `Zoho CRM - update the leads`
  - `STOP: could not authenticate with Zoho`
  - `build the tasks`
  - `STOP: Zoho rejected the write`
  - `tasks to create?`
  - `Zoho CRM - create the tasks`
  - `STOP: Zoho updated`

#### Node Details:
- **decide the follow-up**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Maps intent classifications to Zoho status fields, opt-out flags, notification booleans, and task due dates.
  - **Connections:** Input: `classify by keywords`, `apply the AI reading`; Output: `build the Zoho updates`, `unsubscribes for Smartlead`, `compose the team alert`

- **build the Zoho updates**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Filters replies requiring Zoho modifications and builds batch upsert payloads matched on email addresses.
  - **Connections:** Input: `decide the follow-up`; Output: `writing to Zoho?`

- **writing to Zoho?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks if Zoho integration is enabled and if record counts exceed zero.
  - **Expressions/Variables:** `={{ $('config').first().json.zoho_enabled && $json._count > 0 }}`
  - **Connections:** Input: `build the Zoho updates`; Output (True): `Zoho - get an access token`; Output (False): `STOP: preview - Zoho updates shown`

- **Zoho - get an access token**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Exchanges Zoho refresh tokens for temporary access tokens using POST query parameters.
  - **Connections:** Input: `writing to Zoho?`; Output: `read the access token`
  - **Edge Cases / Failures:** Invalid refresh tokens, scope mismatches, region account base misconfigurations.

- **STOP: preview - Zoho updates shown**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Halts live execution when running in test/preview mode, outputting structured payloads instead.
  - **Connections:** Input: `writing to Zoho?`; Output: None

- **read the access token**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Validates Zoho OAuth token responses and formats error hints if authentication fails.
  - **Connections:** Input: `Zoho - get an access token`; Output: `authenticated?`

- **authenticated?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks `_authed` boolean flag.
  - **Expressions/Variables:** `={{ $json._authed }}`
  - **Connections:** Input: `read the access token`; Output (True): `Zoho CRM - update the leads`; Output (False): `STOP: could not authenticate with Zoho`

- **Zoho CRM - update the leads**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Executes upsert requests to Zoho CRM Leads API endpoints using OAuth headers.
  - **Connections:** Input: `authenticated?`; Output: `build the tasks`, `STOP: Zoho rejected the write`
  - **Edge Cases / Failures:** API rejections, mandatory field omissions, validation failures (handled via `continueErrorOutput`).

- **STOP: could not authenticate with Zoho**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Outputs diagnostic messages detailing OAuth failure causes and hints.
  - **Connections:** Input: `authenticated?`; Output: None

- **build the tasks**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Pairs successful lead record IDs returned from upserts with action tasks required for reps.
  - **Connections:** Input: `Zoho CRM - update the leads`; Output: `tasks to create?`

- **STOP: Zoho rejected the write**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Captures error responses when Zoho refuses update or create calls.
  - **Connections:** Input: `Zoho CRM - update the leads`, `Zoho CRM - create the tasks`; Output: None

- **tasks to create?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Verifies if task counts exceed zero.
  - **Expressions/Variables:** `={{ $json._task_count > 0 }}`
  - **Connections:** Input: `build the tasks`; Output (True): `Zoho CRM - create the tasks`; Output (False): `STOP: Zoho updated`

- **Zoho CRM - create the tasks**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Posts created tasks to Zoho CRM Tasks API linked via `What_Id`.
  - **Connections:** Input: `tasks to create?`; Output: `STOP: Zoho updated`, `STOP: Zoho rejected the write`
  - **Edge Cases / Failures:** Task validation failures, incorrect module associations.

- **STOP: Zoho updated**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Finalizes successful execution logs for Zoho CRM updates and task creations.
  - **Connections:** Input: `tasks to create?`, `Zoho CRM - create the tasks`; Output: None

---

### Block 2.4: Suppression & Team Notifications
- **Overview:** Handles global Smartlead unsubscribe requests across campaigns and formats rich-text alerts for hot replies (interested, referral, needs human) for delivery to team collaboration channels.
- **Nodes Involved:** 
  - `unsubscribes for Smartlead`
  - `unsub-on`
  - `Smartlead - unsubscribe the lead`
  - `unsub-preview`
  - `unsub-done`
  - `alert`
  - `alert-on`
  - `Notify - post the team alert`
  - `alert-preview`
  - `alert-done`

#### Node Details:
- **unsubscribes for Smartlead**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Filters unsubscribe intents and constructs Smartlead suppression request URLs.
  - **Connections:** Input: `decide the follow-up`; Output: `suppressing in Smartlead?`

- **suppressing in Smartlead?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks if Smartlead integration is enabled, items are present, and lead IDs exist.
  - **Expressions/Variables:** `={{ $('config').first().json.smartlead_enabled && !$json._none && !!$json.lead_id }}`
  - **Connections:** Input: `unsubscribes for Smartlead`; Output (True): `Smartlead - unsubscribe the lead`; Output (False): `STOP: preview - unsubscribe not sent`

- **Smartlead - unsubscribe the lead**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Calls Smartlead unsubscribe API endpoints using API key query parameters.
  - **Connections:** Input: `suppressing in Smartlead?`; Output: `STOP: unsubscribed in Smartlead`
  - **Edge Cases / Failures:** Missing lead IDs, invalid API keys.

- **STOP: preview - unsubscribe not sent**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Generates preview outputs when suppression calls are skipped.
  - **Connections:** Input: `suppressing in Smartlead?`; Output: None

- **STOP: unsubscribed in Smartlead**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Reports confirmation or failure status of Smartlead suppression requests.
  - **Connections:** Input: `Smartlead - unsubscribe the lead`; Output: None

- **compose the team alert**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Aggregates hot replies and formats quotation blocks for incoming webhooks.
  - **Connections:** Input: `decide the follow-up`; Output: `alerting the team?`

- **alerting the team?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Evaluates notification flags and hot reply counts.
  - **Expressions/Variables:** `={{ $('config').first().json.notify_enabled && $json._count > 0 }}`
  - **Connections:** Input: `compose the team alert`; Output (True): `Notify - post the team alert`; Output (False): `STOP: preview - alert not posted`

- **Notify - post the team alert**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Posts formatted markdown alert payloads to `NOTIFY_WEBHOOK_URL`.
  - **Connections:** Input: `alerting the team?`; Output: `STOP: alert posted`
  - **Edge Cases / Failures:** Invalid webhook URLs, HTTP payload size limits.

- **STOP: preview - alert not posted**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Displays team alert previews when notifications are disabled.
  - **Connections:** Input: `alerting the team?`; Output: None

- **STOP: alert posted**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Confirms successful webhook posting.
  - **Connections:** Input: `Notify - post the team alert`; Output: None

---

### Block 2.5: Morning Call List Generation
- **Overview:** Connects to Zoho CRM via OAuth token exchange, runs COQL queries to fetch target statuses, ranks prospects based on recency and tier scores, and posts the compiled call list to team collaboration webhooks.
- **Nodes Involved:** 
  - `Zoho connected for the call list?`
  - `Zoho - get a token for the call list`
  - `use the demo pipeline`
  - `build the call list query`
  - `authenticated for the call list?`
  - `Zoho CRM - read the call list`
  - `STOP: call list could not authenticate`
  - `read the leads`
  - `STOP: could not read Zoho`
  - `rank the call list`
  - `posting the call list?`
  - `Notify - post the call list`
  - `STOP: preview - call list shown`
  - `STOP: call list posted`

#### Node Details:
- **Zoho connected for the call list?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks if Zoho authentication parameters are ready (`zoho_auth_ready`).
  - **Expressions/Variables:** `={{ $('config').first().json.zoho_auth_ready }}`
  - **Connections:** Input: `building the call list?`; Output (True): `Zoho - get a token for the call list`; Output (False): `use the demo pipeline`

- **Zoho - get a token for the call list**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Requests access tokens using Zoho OAuth refresh credentials for call list generation.
  - **Connections:** Input: `Zoho connected for the call list?`; Output: `build the call list query`

- **use the demo pipeline**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Provides mock pipeline records when Zoho is disconnected.
  - **Connections:** Input: `Zoho connected for the call list?`; Output: `rank the call list`

- **build the call list query**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Constructs COQL queries targeting configured `CALL_LIST_STATUSES` with dynamic field selections.
  - **Connections:** Input: `Zoho - get a token for the call list`; Output: `authenticated for the call list?`

- **authenticated for the call list?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Verifies `_authed` status for call list queries.
  - **Expressions/Variables:** `={{ $json._authed }}`
  - **Connections:** Input: `build the call list query`; Output (True): `Zoho CRM - read the call list`; Output (False): `STOP: call list could not authenticate`

- **Zoho CRM - read the call list**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Executes POST requests to Zoho COQL endpoints (`/coql`).
  - **Connections:** Input: `authenticated for the call list?`; Output: `read the leads`, `STOP: could not read Zoho`
  - **Edge Cases / Failures:** Invalid query syntax, missing COQL scopes (`ZohoCRM.coql.READ`).

- **STOP: call list could not authenticate**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Reports authentication failures for call list generation.
  - **Connections:** Input: `authenticated for the call list?`; Output: None

- **read the leads**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Normalizes COQL response data into standard pipeline record arrays.
  - **Connections:** Input: `Zoho CRM - read the call list`; Output: `rank the call list`

- **STOP: could not read Zoho**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Captures and reports COQL query errors.
  - **Connections:** Input: `Zoho CRM - read the call list`; Output: None

- **rank the call list**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Ranks prospects by intent priority (`interested` first, then `referrals`, then A-tier by fit score) and caps results at `CALL_LIST_SIZE`.
  - **Connections:** Input: `read the leads`, `use the demo pipeline`; Output: `posting the call list?`

- **posting the call list?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router)
  - **Configuration:** Checks `notify_enabled` configuration flags.
  - **Expressions/Variables:** `={{ $('config').first().json.notify_enabled }}`
  - **Connections:** Input: `rank the call list`; Output (True): `Notify - post the call list`; Output (False): `STOP: preview - call list shown`

- **Notify - post the call list**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (HTTP API Request)
  - **Configuration:** Posts formatted morning call lists to `NOTIFY_WEBHOOK_URL`.
  - **Connections:** Input: `posting the call list?`; Output: `STOP: call list posted`
  - **Edge Cases / Failures:** Webhook timeouts or refusals.

- **STOP: preview - call list shown**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Outputs call list previews when notifications are disabled.
  - **Connections:** Input: `posting the call list?`; Output: None

- **STOP: call list posted**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution)
  - **Configuration:** Confirms successful posting of morning call lists.
  - **Connections:** Input: `Notify - post the call list`; Output: None

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A Smartlead reply arrives | webhook | Webhook trigger | None | config | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| Run the demo replies | manualTrigger | Manual trigger | None | load the demo replies | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| Every weekday, 07:30 | scheduleTrigger | Cron schedule trigger | None | config | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| Build the call list now | manualTrigger | Manual trigger | None | config | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| load the demo replies | code | Generate mock replies | Run the demo replies | config | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| config | code | Centralized configuration hub | A Smartlead reply arrives, Every weekday, 07:30, Build the call list now, load the demo replies | building the call list? | ## Classify Smartlead replies into Zoho CRM tasks and send a daily call list... |
| building the call list? | if | Route call list vs reply paths | config | Zoho connected for the call list?, read the reply | ### 1. Read and classify the reply... |
| read the reply | code | Sanitize & parse reply body | building the call list? | AI configured? | ### 1. Read and classify the reply... |
| AI configured? | if | Check OpenAI key presence | read the reply | build the AI request, classify by keywords | ### 1. Read and classify the reply... |
| build the AI request | code | Prepare AI prompt schema | AI configured? | OpenAI - classify the reply | ### 1. Read and classify the reply... |
| classify by keywords | code | Keyword rule classification fallback | AI configured? | decide the follow-up | ### 1. Read and classify the reply... |
| OpenAI - classify the reply | httpRequest | Query OpenAI classification | build the AI request | apply the AI reading | ### 1. Read and classify the reply... |
| apply the AI reading | code | Validate AI response | OpenAI - classify the reply | decide the follow-up | ### 1. Read and classify the reply... |
| decide the follow-up | code | Map class to Zoho status & task | classify by keywords, apply the AI reading | build the Zoho updates, unsubscribes for Smartlead, compose the team alert | ### 2. Update the lead, then give a person the task... |
| build the Zoho updates | code | Prepare Zoho upsert payload | decide the follow-up | writing to Zoho? | ### 2. Update the lead, then give a person the task... |
| writing to Zoho? | if | Check Zoho write conditions | build the Zoho updates | Zoho - get an access token, STOP: preview - Zoho updates shown | ### 2. Update the lead, then give a person the task... |
| Zoho - get an access token | httpRequest | Request Zoho OAuth token | writing to Zoho? | read the access token | ### 2. Update the lead, then give a person the task... |
| STOP: preview - Zoho updates shown | code | Output Zoho update preview | writing to Zoho? | None | ### 2. Update the lead, then give a person the task... |
| read the access token | code | Validate Zoho token response | Zoho - get an access token | authenticated? | ### 2. Update the lead, then give a person the task... |
| authenticated? | if | Check Zoho auth status | read the access token | Zoho CRM - update the leads, STOP: could not authenticate with Zoho | ### 2. Update the lead, then give a person the task... |
| Zoho CRM - update the leads | httpRequest | Upsert lead records in Zoho | authenticated? | build the tasks, STOP: Zoho rejected the write | ### 2. Update the lead, then give a person the task... |
| STOP: could not authenticate with Zoho | code | Output auth failure notice | authenticated? | None | ### 2. Update the lead, then give a person the task... |
| build the tasks | code | Pair record IDs with task payloads | Zoho CRM - update the leads | tasks to create? | ### 2. Update the lead, then give a person the task... |
| STOP: Zoho rejected the write | code | Handle Zoho API errors | Zoho CRM - update the leads, Zoho CRM - create the tasks | None | ### 2. Update the lead, then give a person the task... |
| tasks to create? | if | Check pending task count | build the tasks | Zoho CRM - create the tasks, STOP: Zoho updated | ### 2. Update the lead, then give a person the task... |
| Zoho CRM - create the tasks | httpRequest | Create CRM tasks in Zoho | tasks to create? | STOP: Zoho updated, STOP: Zoho rejected the write | ### 2. Update the lead, then give a person the task... |
| STOP: Zoho updated | code | Confirm Zoho write completion | tasks to create?, Zoho CRM - create the tasks | None | ### 2. Update the lead, then give a person the task... |
| unsubscribes for Smartlead | code | Prepare Smartlead suppression | decide the follow-up | suppressing in Smartlead? | ### 3. Suppress and alert... |
| suppressing in Smartlead? | if | Check Smartlead suppression conditions | unsubscribes for Smartlead | Smartlead - unsubscribe the lead, STOP: preview - unsubscribe not sent | ### 3. Suppress and alert... |
| Smartlead - unsubscribe the lead | httpRequest | Call Smartlead unsubscribe API | suppressing in Smartlead? | STOP: unsubscribed in Smartlead | ### 3. Suppress and alert... |
| STOP: preview - unsubscribe not sent | code | Output suppression preview | suppressing in Smartlead? | None | ### 3. Suppress and alert... |
| STOP: unsubscribed in Smartlead | code | Confirm Smartlead suppression | Smartlead - unsubscribe the lead | None | ### 3. Suppress and alert... |
| compose the team alert | code | Format team alert message | decide the follow-up | alerting the team? | ### 3. Suppress and alert... |
| alerting the team? | if | Check team notification flags | compose the team alert | Notify - post the team alert, STOP: preview - alert not posted | ### 3. Suppress and alert... |
| Notify - post the team alert | httpRequest | Post team notification webhook | alerting the team? | STOP: alert posted | ### 3. Suppress and alert... |
| STOP: preview - alert not posted | code | Output alert preview | alerting the team? | None | ### 3. Suppress and alert... |
| STOP: alert posted | code | Confirm team alert posting | Notify - post the team alert | None | ### 3. Suppress and alert... |
| Zoho connected for the call list? | if | Check Zoho auth for call list | building the call list? | Zoho - get a token for the call list, use the demo pipeline | ### 4. The morning call list... |
| Zoho - get a token for the call list | httpRequest | Request Zoho OAuth token for call list | Zoho connected for the call list? | build the call list query | ### 4. The morning call list... |
| use the demo pipeline | code | Provide mock call list pipeline | Zoho connected for the call list? | rank the call list | ### 4. The morning call list... |
| build the call list query | code | Construct Zoho COQL query | Zoho - get a token for the call list | authenticated for the call list? | ### 4. The morning call list... |
| authenticated for the call list? | if | Check call list auth status | build the call list query | Zoho CRM - read the call list, STOP: call list could not authenticate | ### 4. The morning call list... |
| Zoho CRM - read the call list | httpRequest | Execute COQL query | authenticated for the call list? | read the leads, STOP: could not read Zoho | ### 4. The morning call list... |
| STOP: call list could not authenticate | code | Output call list auth error | authenticated for the call list? | None | ### 4. The morning call list... |
| read the leads | code | Normalize COQL records | Zoho CRM - read the call list | rank the call list | ### 4. The morning call list... |
| STOP: could not read Zoho | code | Output COQL query error | Zoho CRM - read the call list | None | ### 4. The morning call list... |
| rank the call list | code | Rank and cap morning call list | read the leads, use the demo pipeline | posting the call list? | ### 4. The morning call list... |
| posting the call list? | if | Check notification enabled flag | rank the call list | Notify - post the call list, STOP: preview - call list shown | ### 4. The morning call list... |
| Notify - post the call list | httpRequest | Post morning call list webhook | posting the call list? | STOP: call list posted | ### 4. The morning call list... |
| STOP: preview - call list shown | code | Output call list preview | posting the call list? | None | ### 4. The morning call list... |
| STOP: call list posted | code | Confirm call list posting | Notify - post the call list | None | ### 4. The morning call list... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Triggers:**
   - Add a **Webhook** node named `A Smartlead reply arrives` with HTTP method `POST` and path `smartlead-reply`.
   - Add a **Schedule Trigger** node named `Every weekday, 07:30` configured with cron expression `30 7 * * 1-5`.
   - Add two **Manual Trigger** nodes named `Run the demo replies` and `Build the call list now`.

2. **Set Up Demo & Configuration Hub:**
   - Connect `Run the demo replies` to a **Code** node named `load the demo replies` returning sample JSON reply structures.
   - Connect `A Smartlead reply arrives`, `Every weekday, 07:30`, `Build the call list now`, and `load the demo replies` to a **Code** node named `config` containing global settings (API keys, Zoho credentials, thresholds, and flags).

3. **Build the Reply Routing & Parsing Branch:**
   - Connect `config` to an **If** node named `building the call list?` checking if call list triggers executed.
   - On the `false` output branch, add a **Code** node named `read the reply` to strip email quotes and normalize fields.
   - Connect `read the reply` to an **If** node named `AI configured?` checking `ai_enabled`.

4. **Implement Classification Logic:**
   - On the `true` branch of `AI configured?`, add a **Code** node named `build the AI request` and connect it to an **HTTP Request** node named `OpenAI - classify the reply` (`POST` to OpenAI chat completions with JSON schema enforcement). Connect it to a **Code** node named `apply the AI reading`.
   - On the `false` branch of `AI configured?`, add a **Code** node named `classify by keywords`.
   - Route both classification outputs into a **Code** node named `decide the follow-up`.

5. **Configure Zoho CRM Lead Synchronization & Tasks:**
   - Connect `decide the follow-up` to:
     - A **Code** node named `build the Zoho updates`, followed by an **If** node `writing to Zoho?`.
     - When true, call an **HTTP Request** node `Zoho - get an access token`, validate via `read the access token` and **If** node `authenticated?`, then execute an **HTTP Request** node `Zoho CRM - update the leads`.
     - Pass upsert outputs to a **Code** node `build the tasks`, check via **If** node `tasks to create?`, and execute an **HTTP Request** node `Zoho CRM - create the tasks`. Handle failures with appropriate `STOP` nodes.

6. **Configure Suppression & Team Notifications:**
   - Connect `decide the follow-up` to a **Code** node named `unsubscribes for Smartlead`, an **If** node `suppressing in Smartlead?`, and an **HTTP Request** node `Smartlead - unsubscribe the lead`.
   - Connect `decide the follow-up` to a **Code** node named `compose the team alert`, an **If** node `alerting the team?`, and an **HTTP Request** node `Notify - post the team alert`.

7. **Configure Morning Call List Generation:**
   - From the `true` output of `building the call list?`, add an **If** node `Zoho connected for the call list?`.
   - On the `true` branch, add an **HTTP Request** node `Zoho - get a token for the call list`, a **Code** node `build the call list query`, an **If** node `authenticated for the call list?`, an **HTTP Request** node `Zoho CRM - read the call list`, and a **Code** node `read the leads`.
   - On the `false` branch, add a **Code** node `use the demo pipeline`.
   - Route both into a **Code** node `rank the call list`, an **If** node `posting the call list?`, and an **HTTP Request** node `Notify - post the call list`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clay Outbound Scoring Engine (Part 2 of 2) | Integrates with downstream outbound scoring workflows and shared configuration structures. |
| Zoho CRM API Requirements | Requires Self Client credentials with scopes `ZohoCRM.modules.ALL` and `ZohoCRM.coql.READ`. |