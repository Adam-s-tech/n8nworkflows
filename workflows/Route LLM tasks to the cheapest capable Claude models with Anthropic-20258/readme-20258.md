Route LLM tasks to the cheapest capable Claude models with Anthropic

https://n8nworkflows.xyz/workflows/route-llm-tasks-to-the-cheapest-capable-claude-models-with-anthropic-20258


# Route LLM tasks to the cheapest capable Claude models with Anthropic

### 1. Workflow Overview

This workflow implements an intelligent LLM task router utilizing Anthropic Claude models. Its primary purpose is to ingest incoming tasks via a webhook or manual trigger, classify their complexity, risk, and technical requirements using a lightweight model, and dynamically select the most cost-effective capable model from a configured catalog. If execution fails or a quality gate fails (judged by a secondary evaluation step), the workflow builds an escalation chain to retry with more powerful models. Finally, it returns the generated answer alongside complete token usage, cost breakdowns, savings metrics compared to a premium baseline model, and optional logging to an external execution ledger.

The operational logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Ingests the task payload, validates its existence, rejects invalid requests with a 400 error, and defines global routing variables, model catalog definitions, and limits.
- **1.2 Classification & Model Selection:** Evaluates the prompt via an AI classification request (with a built-in heuristic fallback), filters the model catalog by capability, tier, and budget constraints, and constructs an escalation chain.
- **1.3 Execution, Quality Judgment & Escalation Loop:** Executes the selected model, records token counts and costs, and routes the response either to finish, retry via escalation, or undergo quality scoring by a judge model.
- **1.4 Delivery & Ledger Logging:** Calculates total execution costs, calculates savings against a premium baseline model, responds to the webhook caller, and conditionally logs the execution trace to an external ledger.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block accepts incoming data from either a production webhook endpoint or a manual test runner, normalizes the structure, checks for valid task text, sets router configurations (such as model catalogs and pricing structures), and builds the initial classification prompt.
- **Nodes Involved:** 
  - `When Task Submitted`
  - `When Manual Test Run Initiates`
  - `Normalize Task Input`
  - `If Task Is Valid`
  - `Respond with Validation Error`
  - `Set Router Configuration`
  - `Build Classifier Request`

- **Node Details:**
  - **When Task Submitted**
    - *Type and Technical Role:* Webhook Trigger (`n8n-nodes-base.webhook`). Listens for incoming HTTP POST requests on path `llm-router`.
    - *Configuration Choices:* Method set to `POST`, response mode managed by downstream response nodes.
    - *Input/Output:* No inputs; outputs JSON payload containing request headers and body to `Normalize Task Input`.
    - *Edge Cases/Failures:* Invalid payload format or network connectivity issues to the webhook endpoint.
  - **When Manual Test Run Initiates**
    - *Type and Technical Role:* Manual Trigger (`n8n-nodes-base.manualTrigger`). Allows developers to run the workflow manually for debugging.
    - *Input/Output:* No inputs; outputs manual trigger execution context to `Normalize Task Input`.
  - **Normalize Task Input**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Parses raw webhook or manual payloads, extracts parameters (`task`, `context`, `maxCostUsd`, `minTier`, `forceModel`, `minQualityScore`, `maxAttempts`), and supplies default fallback values.
    - *Key Expressions:* Uses ternary operators and JavaScript checks against `$input.first().json` to clean and initialize the workflow state.
    - *Input/Output:* Inputs from both triggers; outputs normalized tracking object to `If Task Is Valid`.
    - *Edge Cases/Failures:* Malformed JSON payloads missing body attributes.
  - **If Task Is Valid**
    - *Type and Technical Role:* If Condition (`n8n-nodes-base.if`). Evaluates whether the task string is populated.
    - *Key Expressions:* `={{ $json.taskValid }}`
    - *Input/Output:* Input from `Normalize Task Input`. True branch goes to `Set Router Configuration`; False branch goes to `Respond with Validation Error`.
  - **Respond with Validation Error**
    - *Type and Technical Role:* Respond to Webhook (`n8n-nodes-base.respondToWebhook`). Returns a 400 Bad Request error when a task is empty.
    - *Configuration Choices:* HTTP Response Code set to `400`. Response body populated with JSON stringified validation error.
    - *Input/Output:* Input from `If Task Is Valid` (false branch); terminates execution path.
  - **Set Router Configuration**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`). Defines routing parameters, model catalog metadata, pricing per 1M tokens, context windows, quality thresholds, and API endpoints.
    - *Configuration Choices:* Sets fields including `classifierModel`, `judgeModel`, `premiumBaselineModel`, `modelCatalog` (JSON array of tier definitions), `judgeEnabled`, `minQualityScore`, `maxAttempts`, `anthropicApiUrl`, and `ledgerUrl`.
    - *Input/Output:* Input from `If Task Is Valid` (true branch); outputs extended configuration state to `Build Classifier Request`.
  - **Build Classifier Request**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Generates a local heuristic fallback analysis and builds the structured JSON request payload for the task classifier model.
    - *Key Expressions:* JavaScript regex checks for code keywords, reasoning indicators, word counts, and constructs the Anthropic system/user message payload.
    - *Input/Output:* Input from `Set Router Configuration`; outputs classifier payload to `Classify Task`.

---

#### 2.2 Classification & Model Selection
- **Overview:** Sends the task to a lightweight classifier model (Claude Haiku), parses the returned JSON or falls back to the local heuristic, calculates input complexity, and filters the model catalog to build an ordered escalation chain.
- **Nodes Involved:** 
  - `Classify Task`
  - `Select Cheapest Capable Model`

- **Node Details:**
  - **Classify Task**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Calls the Anthropic API to classify task complexity, risks, and functional needs.
    - *Configuration Choices:* POST method to `={{ $json.anthropicApiUrl }}`, timeout set to 120,000ms, generic HTTP Header Auth credential (`x-api-key`), custom headers (`anthropic-version: 2023-06-01`).
    - *Input/Output:* Input from `Build Classifier Request`; outputs API response to `Select Cheapest Capable Model`.
    - *Edge Cases/Failures:* API authentication failures, rate limiting (HTTP 429), or malformed response formats (handled safely by downstream fallback logic).
  - **Select Cheapest Capable Model**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Parses classifier responses, calculates classification costs, maps requirements to model tiers, filters the catalog by context window and budget limits, and constructs the sorted escalation chain.
    - *Key Expressions:* JavaScript JSON parsing with fallback to local heuristic, token-to-cost calculations based on per-1M pricing models.
    - *Input/Output:* Inputs from `Build Classifier Request` (via node reference) and `Classify Task`; outputs updated state containing `chain`, `chainDetail`, and classification metrics to `Build Execution Request`.

---

#### 2.3 Execution, Quality Judgment & Escalation Loop
- **Overview:** Executes the current model in the escalation chain, records token consumption and expenses, evaluates whether to judge quality, runs a strict grading model if enabled, and loops back via escalation if results fall below thresholds or errors occur.
- **Nodes Involved:** 
  - `Build Execution Request`
  - `Execute Model`
  - `Record Attempt`
  - `Route Next Action`
  - `Build Judge Request`
  - `Judge Quality`
  - `Evaluate Quality Score`
  - `If Should Escalate`

- **Node Details:**
  - **Build Execution Request**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Constructs the prompt payload for the currently selected model index in the chain, including tool definitions (such as web search) if required by the classification.
    - *Input/Output:* Inputs from `Select Cheapest Capable Model` or `If Should Escalate`; outputs execution request to `Execute Model`.
  - **Execute Model**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Sends the execution prompt to the Anthropic Messages API.
    - *Configuration Choices:* POST method to `={{ $json.anthropicApiUrl }}`, HTTP Header Auth, custom anthropic version headers, timeout 120,000ms.
    - *Input/Output:* Input from `Build Execution Request`; outputs raw model execution response to `Record Attempt`.
    - *Edge Cases/Failures:* Timeouts, upstream API errors, or token limit exhaustion.
  - **Record Attempt**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Evaluates the execution attempt success, calculates token usage and actual cost, logs metadata, and determines the next action (`finish`, `escalate`, or `judge`).
    - *Input/Output:* Inputs from `Build Execution Request` and `Execute Model`; outputs state update to `Route Next Action`.
  - **Route Next Action**
    - *Type and Technical Role:* Switch Node (`n8n-nodes-base.switch`). Branches workflow execution based on the `nextAction` string value (`judge`, `escalate`, or fallback `finish`).
    - *Key Expressions:* `={{ $json.nextAction }}`
    - *Input/Output:* Input from `Record Attempt`; routes to `Build Judge Request` (judge), `Build Execution Request` (escalate), or `Build Final Response` (finish).
  - **Build Judge Request**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Formats a strict grading prompt requesting a score between 1 and 10 and constructive feedback.
    - *Input/Output:* Input from `Route Next Action`; outputs judge request to `Judge Quality`.
  - **Judge Quality**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Calls the Anthropic API using the designated judge model.
    - *Configuration Choices:* POST method, HTTP Header Auth, timeout 120,000ms.
    - *Input/Output:* Input from `Build Judge Request`; outputs judge evaluation response to `Evaluate Quality Score`.
  - **Evaluate Quality Score**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Parses the judge's score and feedback, accumulates judge costs, and sets the `escalate` boolean flag if the score is below the threshold and remaining fallback models exist.
    - *Input/Output:* Inputs from `Build Judge Request` and `Judge Quality`; outputs evaluation results to `If Should Escalate`.
  - **If Should Escalate**
    - *Type and Technical Role:* If Condition (`n8n-nodes-base.if`). Checks whether the evaluation determined an escalation is necessary.
    - *Key Expressions:* `={{ $json.escalate }}`
    - *Input/Output:* Input from `Evaluate Quality Score`. True branch loops back to `Build Execution Request`; False branch proceeds to `Build Final Response`.

---

#### 2.4 Delivery & Ledger Logging
- **Overview:** Aggregates overall token metrics, computes total financial costs alongside savings comparisons against a premium baseline model, returns the structured answer to the webhook caller, and optionally posts execution events to an external ledger webhook.
- **Nodes Involved:** 
  - `Build Final Response`
  - `Respond with Answer via Webhook`
  - `If Ledger Logging Enabled`
  - `Log Calls to Ledger`

- **Node Details:**
  - **Build Final Response**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Summarizes all attempt traces, calculates total cost, computes savings against the premium baseline model, builds routing trace logs, and generates execution ledger payloads.
    - *Input/Output:* Inputs from `If Should Escalate` (false branch) or `Route Next Action` (finish branch); outputs delivery and ledger payloads to `Respond with Answer via Webhook` and `If Ledger Logging Enabled`.
  - **Respond with Answer via Webhook**
    - *Type and Technical Role:* Respond to Webhook (`n8n-nodes-base.respondToWebhook`). Returns the final JSON payload containing the answer, routing history, and cost summary to the initial caller.
    - *Input/Output:* Input from `Build Final Response`; terminates execution path.
  - **If Ledger Logging Enabled**
    - *Type and Technical Role:* If Condition (`n8n-nodes-base.if`). Checks whether a valid `ledgerUrl` was provided in configuration.
    - *Key Expressions:* `={{ $json.logToLedger }}`
    - *Input/Output:* Input from `Build Final Response`. True branch proceeds to `Log Calls to Ledger`.
  - **Log Calls to Ledger**
    - *Type and Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Posts execution events, routing decisions, and token consumption metrics to an external ledger endpoint.
    - *Configuration Choices:* POST method to `={{ $json.ledgerUrl }}`, timeout 20,000ms, JSON body transmission.
    - *Input/Output:* Input from `If Ledger Logging Enabled`; terminates execution path.
    - *Edge Cases/Failures:* External ledger endpoint unavailability (configured with `continueRegularOutput` error handling).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation & setup guide | None | None | ## Cheapest Capable LLM Router<br><br>### How it works<br><br>A task comes in by webhook. A cheap classifier model (Haiku) scores its complexity, risk and needs (reasoning, code, web search, long context). The router then picks the **cheapest model in the catalog that is capable enough**, builds an escalation chain of higher tiers, and runs the task. A cheap judge scores the answer; if it falls below the quality threshold the task escalates to the next tier with the judge's feedback. The response includes the answer, routing trace, actual cost and savings versus always using the premium model.<br><br>### Setup steps<br><br>- Create an HTTP Header Auth credential for Anthropic (header `x-api-key`) and select it on the three Claude HTTP nodes: Classify Task, Execute Model, Judge Quality.<br>- Edit **Set Router Configuration**: model catalog (tiers, prices per 1M tokens, context windows - verify against current pricing), quality threshold, max attempts, premium baseline model.<br>- Optional: set `ledgerUrl` to the Execution Ledger webhook (`/webhook/agent-ledger-event`) to log every call and its cost.<br>- POST to `/webhook/llm-router` with `{ \"task\": \"...\", \"context\": \"...\", \"maxCostUsd\": 0.05, \"minTier\": 1, \"forceModel\": \"\", \"minQualityScore\": 7, \"maxAttempts\": 3 }`. Only `task` is required. A manual run uses a sample task.<br><br>### Customization<br><br>- Add a model: add a row to the catalog JSON (id, tier 1-3, prices, contextWindow, webSearch).<br>- Change how complexity maps to tiers in the Select Cheapest Capable Model node.<br>- Set `judgeEnabled` to false to skip quality checks and never escalate on score. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Receive and configure documentation block | None | None | ## Receive and configure<br><br>Accepts a task from the webhook or a manual run, rejects requests without a task, then applies router configuration. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Classify and select documentation block | None | None | ## Classify and select<br><br>A cheap model classifies the task (with a heuristic fallback if it fails). The selector filters the catalog by capability, context window and budget, then builds the escalation chain. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Execute, judge and escalate documentation block | None | None | ## Execute, judge and escalate<br><br>Runs the cheapest capable model, then scores the answer with a cheap judge. Below the threshold, control loops back to execute with the next model in the chain and the judge's feedback. Failed calls also escalate. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Deliver documentation block | None | None | ## Deliver<br><br>Builds the final response with cost breakdown and savings, returns it to the caller, and optionally logs events to the execution ledger. |
| When Task Submitted | n8n-nodes-base.webhook | Ingest incoming task POST requests | None | Normalize Task Input | Receive and configure |
| When Manual Test Run Initiates | n8n-nodes-base.manualTrigger | Trigger manual workflow tests | None | Normalize Task Input | Receive and configure |
| Normalize Task Input | n8n-nodes-base.code | Normalize input parameters and defaults | When Task Submitted, When Manual Test Run Initiates | If Task Is Valid | Receive and configure |
| If Task Is Valid | n8n-nodes-base.if | Verify task string presence | Normalize Task Input | Set Router Configuration, Respond with Validation Error | Receive and configure |
| Respond with Validation Error | n8n-nodes-base.respondToWebhook | Return 400 error on empty task | If Task Is Valid | None | Receive and configure |
| Set Router Configuration | n8n-nodes-base.set | Define router catalogs and thresholds | If Task Is Valid | Build Classifier Request | Receive and configure |
| Build Classifier Request | n8n-nodes-base.code | Build classifier prompt & heuristic | Set Router Configuration | Classify Task | Classify and select |
| Classify Task | n8n-nodes-base.httpRequest | Call Anthropic API for classification | Build Classifier Request | Select Cheapest Capable Model | Classify and select |
| Select Cheapest Capable Model | n8n-nodes-base.code | Filter catalog & build escalation chain | Build Classifier Request, Classify Task | Build Execution Request | Classify and select |
| Build Execution Request | n8n-nodes-base.code | Build prompt payload for active model | Select Cheapest Capable Model, If Should Escalate, Route Next Action | Execute Model | Execute, judge and escalate |
| Execute Model | n8n-nodes-base.httpRequest | Execute task with selected model | Build Execution Request | Record Attempt | Execute, judge and escalate |
| Record Attempt | n8n-nodes-base.code | Record attempt metrics & route next action | Execute Model | Route Next Action | Execute, judge and escalate |
| Route Next Action | n8n-nodes-base.switch | Route between finish, judge, or escalate | Record Attempt | Build Judge Request, Build Execution Request, Build Final Response | Execute, judge and escalate |
| Build Judge Request | n8n-nodes-base.code | Construct quality evaluation prompt | Route Next Action | Judge Quality | Execute, judge and escalate |
| Judge Quality | n8n-nodes-base.httpRequest | Call Anthropic API for answer grading | Build Judge Request | Evaluate Quality Score | Execute, judge and escalate |
| Evaluate Quality Score | n8n-nodes-base.code | Parse judge score and set escalation flag | Judge Quality | If Should Escalate | Execute, judge and escalate |
| If Should Escalate | n8n-nodes-base.if | Check if escalation is required | Evaluate Quality Score | Build Execution Request, Build Final Response | Execute, judge and escalate |
| Build Final Response | n8n-nodes-base.code | Calculate costs, savings, and ledger events | If Should Escalate, Route Next Action | Respond with Answer via Webhook, If Ledger Logging Enabled | Deliver |
| Respond with Answer via Webhook | n8n-nodes-base.respondToWebhook | Return final JSON response to caller | Build Final Response | None | Deliver |
| If Ledger Logging Enabled | n8n-nodes-base.if | Check if external ledger logging is enabled | Build Final Response | Log Calls to Ledger | Deliver |
| Log Calls to Ledger | n8n-nodes-base.httpRequest | Send execution audit events to ledger | If Ledger Logging Enabled | None | Deliver |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Trigger Nodes:**
   - Create a **Webhook** node named `When Task Submitted`. Set `HTTP Method` to `POST` and `Path` to `llm-router`.
   - Create a **Manual Trigger** node named `When Manual Test Run Initiates`.

2. **Normalize Inputs:**
   - Create a **Code** node named `Normalize Task Input`. Connect both triggers to this node.
   - Paste JavaScript code to extract fields (`task`, `context`, `maxCostUsd`, `minTier`, `forceModel`, `minQualityScore`, `maxAttempts`) and assign safe defaults if values are missing.

3. **Add Validation Logic:**
   - Create an **If** node named `If Task Is Valid`. Connect `Normalize Task Input` here. Set condition to check `{{ $json.taskValid }}` equals `true`.
   - Create a **Respond to Webhook** node named `Respond with Validation Error`. Connect the `false` branch of `If Task Is Valid` to this node. Set response code to `400` and return JSON containing validation errors.

4. **Configure Global Router Settings:**
   - Create a **Set** node named `Set Router Configuration`. Connect the `true` branch of `If Task Is Valid` here.
   - Assign assignments for `classifierModel` (`claude-haiku-4-5-20251001`), `judgeModel` (`claude-haiku-4-5-20251001`), `premiumBaselineModel` (`claude-opus-4-5`), `modelCatalog` (JSON string array defining tiers 1 to 3 with input/output token pricing), `judgeEnabled` (`true`), `minQualityScore` (`7`), `maxAttempts` (`3`), `anthropicApiUrl` (`https://api.anthropic.com/v1/messages`), and `ledgerUrl` (empty string by default).

5. **Build Classification Step:**
   - Create a **Code** node named `Build Classifier Request`. Connect `Set Router Configuration` here. Add script to generate a heuristic fallback and construct the classification prompt object.
   - Create an **HTTP Request** node named `Classify Task`. Connect `Build Classifier Request` here. Set Method to `POST`, URL to `={{ $json.anthropicApiUrl }}`, configure generic HTTP Header Auth for Anthropic (`x-api-key`), add header `anthropic-version: 2023-06-01`, and enable error continuation and retry settings.

6. **Select Capable Model:**
   - Create a **Code** node named `Select Cheapest Capable Model`. Connect both `Build Classifier Request` (via pin data) and `Classify Task` here. Add script to parse classification results, calculate costs, determine required tiers, filter catalog models, and build the escalation chain.

7. **Build Execution & Retry Loop:**
   - Create a **Code** node named `Build Execution Request`. Connect `Select Cheapest Capable Model` here (as well as feedback loops from `If Should Escalate` and `Route Next Action`). Prepare prompt payloads for the active chain index.
   - Create an **HTTP Request** node named `Execute Model`. Connect `Build Execution Request` here. Configure identical Anthropic POST settings and authentication as the classification request.
   - Create a **Code** node named `Record Attempt` to parse execution outputs, calculate token costs, and determine next actions (`finish`, `escalate`, or `judge`). Connect `Execute Model` here.
   - Create a **Switch** node named `Route Next Action`. Connect `Record Attempt` here. Define rules for output keys `judge` and `escalate`, with a fallback output for `finish`.

8. **Implement Quality Judgment:**
   - Create a **Code** node named `Build Judge Request` (connected to the `judge` branch of `Route Next Action`).
   - Create an **HTTP Request** node named `Judge Quality` (connected to `Build Judge Request`), configured with Anthropic HTTP Header Auth.
   - Create a **Code** node named `Evaluate Quality Score` (connected to `Judge Quality`) to parse scores and decide whether escalation is needed.
   - Create an **If** node named `If Should Escalate` (connected to `Evaluate Quality Score`). Route its `true` branch back to `Build Execution Request`, and its `false` branch to `Build Final Response`.

9. **Build Final Delivery & Ledger Integration:**
   - Create a **Code** node named `Build Final Response`. Connect the `finish` output of `Route Next Action` and the `false` branch of `If Should Escalate` to this node. Calculate aggregate costs, baseline savings, and event ledgers.
   - Create a **Respond to Webhook** node named `Respond with Answer via Webhook`. Connect `Build Final Response` here to return the final JSON payload.
   - Create an **If** node named `If Ledger Logging Enabled` (connected to `Build Final Response`) checking `{{ $json.logToLedger }}`.
   - Create an **HTTP Request** node named `Log Calls to Ledger` (connected to `If Ledger Logging Enabled`) to send event payloads to `={{ $json.ledgerUrl }}` via POST.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Anthropic API Authentication | Requires an HTTP Header Auth credential named with header `x-api-key` pointing to the Anthropic API key. |
| Model Pricing & Catalog | Model catalog prices, tiers, and context limits in `Set Router Configuration` should be periodically verified and updated against current Anthropic pricing documentation. |
| Execution Ledger Integration | Optional ledger logging can transmit events to a separate n8n workflow or webhook endpoint accepting agentic event arrays. |