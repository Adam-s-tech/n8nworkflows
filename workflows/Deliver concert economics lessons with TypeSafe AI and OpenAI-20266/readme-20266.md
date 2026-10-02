Deliver concert economics lessons with TypeSafe AI and OpenAI

https://n8nworkflows.xyz/workflows/deliver-concert-economics-lessons-with-typesafe-ai-and-openai-20266


# Deliver concert economics lessons with TypeSafe AI and OpenAI

### 1. Workflow Overview

This workflow powers an interactive, gamified concert economics lesson. It handles two primary entry points: processing user-defined scenario changes and questions about concert economics, and researching competing live or simulated events for specific cities and dates. 

The application logic is broken down into two main operational branches:
- **Lesson Request Processing:** Receives scenario data and user questions, validates inputs, performs deterministic economic calculations (revenue, costs, profit, ROI), classifies intent using TypeSafe AI, and generates explanatory text via OpenAI or deterministic fallbacks.
- **Event Research Request Processing:** Receives location and date parameters, validates the planning interval, and either returns clearly labeled simulated listings or queries OpenAI's web search tool to find, verify, and deduplicate real-world competing events.

---

### 2. Block-by-Block Analysis

#### 2.1 Lesson Input Reception and Validation
- **Overview:** Receives the initial HTTP POST request from the companion website, validates the payload structure, and computes baseline economic metrics using a deterministic JavaScript model.
- **Nodes Involved:** `Lesson request`, `Validate and calculate`, `Valid lesson request?`, `Explain how to correct input`.
- **Node Details:**
  - **Lesson request** (`n8n-nodes-base.webhook`)
    - *Role:* Entry point for lesson data and user queries.
    - *Configuration:* HTTP POST method, path `economics-concert-v1`, Header Authentication using credential `X-Lesson-Key`, response mode set to `responseNode`.
    - *Input/Output:* Trigger node; outputs to `Validate and calculate`.
    - *Edge Cases:* Unauthenticated requests or incorrect headers return 401 Unauthorized automatically handled by n8n.
  - **Validate and calculate** (`n8n-nodes-base.code`)
    - *Role:* Executes comprehensive validation routines, normalizes inputs, calculates scenario metrics (attendance, revenue, total cost, profit, return on cost), and prepares both fallback lessons and OpenAI request schemas.
    - *Configuration:* Custom JavaScript block containing all validation and economic formulas.
    - *Input/Output:* Input from `Lesson request`; outputs to `Valid lesson request?`.
    - *Edge Cases:* Throws errors if input parameters fall outside allowed numerical ranges or contain unexpected schema properties, which are caught and flagged via the `valid: false` property.
  - **Valid lesson request?** (`n8n-nodes-base.if`)
    - *Role:* Branching gate that routes successfully validated requests toward AI evaluation and invalid requests to error responses.
    - *Configuration:* Condition checks if `{{ $json.valid }}` equals `true`.
    - *Input/Output:* Input from `Validate and calculate`; outputs to `Identify concert question` (true) or `Explain how to correct input` (false).
  - **Explain how to correct input** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns a 400 Bad Request response containing the validation error message.
    - *Configuration:* HTTP response code 400, response body configured to return `{{ { error: $json.error } }}`.
    - *Input/Output:* Input from `Valid lesson request?` (false branch); terminal response node.

#### 2.2 AI Intent Classification and Confidence Routing
- **Overview:** Evaluates the user's natural language question against predefined intent categories using a type-safe AI node, assessing whether the intent is understood with high confidence.
- **Nodes Involved:** `Identify concert question`, `Read intent confidence`, `Confident supported intent?`, `Ask focused clarification`.
- **Node Details:**
  - **Identify concert question** (`@typesafe-ai/n8n-nodes-typesafe-ai.typeSafeAi`)
    - *Role:* Uses structured type-safe AI evaluation to classify the learner's question intent.
    - *Configuration:* Uses model `jev-latest`, operation `evaluate`, with strict question schemas defining categories such as `ticket_price`, `costs`, `profit_roi`, and `unclear_or_other`.
    - *Input/Output:* Input from `Valid lesson request?`; outputs to `Read intent confidence`.
    - *Version Requirements:* Requires the `@typesafe-ai/n8n-nodes-typesafe-ai` community node installed on the instance.
  - **Read intent confidence** (`n8n-nodes-base.code`)
    - *Role:* Parses the intent classification results and checks if confidence meets the required threshold ($\ge 0.8$).
    - *Configuration:* Custom JavaScript evaluating intent confidence scores and injecting the decision into the state object.
    - *Input/Output:* Input from `Identify concert question`; outputs to `Confident supported intent?`.
  - **Confident supported intent?** (`n8n-nodes-base.if`)
    - *Role:* Determines whether to proceed with dynamic OpenAI text assembly or request clarification from the user.
    - *Configuration:* Condition checks if `{{ $json.decision.explain }}` equals `true`.
    - *Input/Output:* Input from `Read intent confidence`; outputs to `Select lesson explanation` (true) or `Ask focused clarification` (false).
  - **Ask focused clarification** (`n8n-nodes-base.code`)
    - *Role:* Prepares a clarification response when intent confidence is low or the question is off-topic.
    - *Configuration:* Custom JavaScript generating a fallback clarification payload.
    - *Input/Output:* Input from `Confident supported intent?` (false branch); outputs to `Deliver interactive lesson`.

#### 2.3 AI Lesson Generation and Delivery
- **Overview:** Sends structured context and approved paragraph choices to OpenAI for explanation selection, validates the model output, and returns the final interactive lesson JSON to the caller.
- **Nodes Involved:** `Select lesson explanation`, `Check explanation or use prepared lesson`, `Deliver interactive lesson`.
- **Node Details:**
  - **Select lesson explanation** (`n8n-nodes-base.httpRequest`)
    - *Role:* Calls the OpenAI Responses API to select 1–3 approved paragraph IDs and a visual scene focus based on the classified intent.
    - *Configuration:* POST request to `https://api.openai.com/v1/responses`, 10-second timeout, JSON body containing model instructions and structured schemas, authenticated via predefined OpenAI credentials. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `Confident supported intent?` (true branch); outputs to `Check explanation or use prepared lesson`.
    - *Edge Cases:* API timeouts or malformed responses are captured by the downstream code node, which falls back to pre-authored lesson text.
  - **Check explanation or use prepared lesson** (`n8n-nodes-base.code`)
    - *Role:* Validates the OpenAI JSON response against strict structural rules and falls back to a deterministic explanation if validation fails.
    - *Configuration:* Custom JavaScript parsing response tokens and ensuring all selected paragraph IDs exist within the approved set.
    - *Input/Output:* Input from `Select lesson explanation`; outputs to `Deliver interactive lesson`.
  - **Deliver interactive lesson** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns the final lesson payload (calculations, explanations, scene actions, and metadata) to the webhook caller.
    - *Configuration:* HTTP 200 OK response with `Cache-Control: no-store` header, responding with JSON body `{{ $json }}`.
    - *Input/Output:* Input from either `Ask focused clarification` or `Check explanation or use prepared lesson`; terminal response node.

#### 2.4 Event Research and Simulation
- **Overview:** Receives event research requests, validates date/city constraints, and either generates fictional simulation data or queries OpenAI with live web search tools to gather and verify competing event listings.
- **Nodes Involved:** `Event research request`, `Validate planning interval`, `Valid planning request?`, `Return invalid request`, `Use illustrative listings?`, `Build clearly labeled simulation`, `Retrieve dated sources`, `Verify geography dates and deduplicate`, `Record research correlation`, `Return one research response`.
- **Node Details:**
  - **Event research request** (`n8n-nodes-base.webhook`)
    - *Role:* Entry point for event research and competitor analysis requests.
    - *Configuration:* HTTP POST method, path `concert-events-v1`, Header Authentication using credential `X-Lesson-Key`, response mode set to `responseNode`.
    - *Input/Output:* Trigger node; outputs to `Validate planning interval`.
  - **Validate planning interval** (`n8n-nodes-base.code`)
    - *Role:* Validates city IDs, date windows, time zones, and constructs query keys and OpenAI tool request payloads.
    - *Configuration:* Custom JavaScript containing city reference data, timezone calculation logic, and schema validators.
    - *Input/Output:* Input from `Event research request`; outputs to `Valid planning request?`.
  - **Valid planning request?** (`n8n-nodes-base.if`)
    - *Role:* Branching node validating whether the planning interval request is structurally sound.
    - *Configuration:* Condition checks if `{{ $json.valid }}` equals `true`.
    - *Input/Output:* Input from `Validate planning interval`; outputs to `Use illustrative listings?` (true) or `Return invalid request` (false).
  - **Return invalid request** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns a 400 Bad Request error response when planning parameters are invalid.
    - *Configuration:* HTTP response code 400, responding with JSON body `{{ $json.response }}`.
    - *Input/Output:* Input from `Valid planning request?` (false branch); terminal response node.
  - **Use illustrative listings?** (`n8n-nodes-base.if`)
    - *Role:* Routes execution based on whether simulation mode or live web search mode is selected.
    - *Configuration:* Condition checks if `{{ $json.plan.dataMode === 'simulation' }}`.
    - *Input/Output:* Input from `Valid planning request?` (true branch); outputs to `Build clearly labeled simulation` (true) or `Retrieve dated sources` (false).
  - **Build clearly labeled simulation** (`n8n-nodes-base.code`)
    - *Role:* Generates fictional, clearly labeled event listings for supported cities (Atlanta, Berlin, Kingston).
    - *Configuration:* Custom JavaScript outputting structured simulation event objects.
    - *Input/Output:* Input from `Use illustrative listings?` (true branch); outputs to `Record research correlation`.
  - **Retrieve dated sources** (`n8n-nodes-base.httpRequest`)
    - *Role:* Queries the OpenAI Responses API with web search capabilities enabled to find public event listings.
    - *Configuration:* POST request to `https://api.openai.com/v1/responses`, 50-second timeout, JSON body containing search tools and schemas, authenticated via predefined OpenAI credentials. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `Use illustrative listings?` (false branch); outputs to `Verify geography dates and deduplicate`.
    - *Edge Cases:* Web search failures or empty search results are safely handled downstream.
  - **Verify geography dates and deduplicate** (`n8n-nodes-base.code`)
    - *Role:* Parses OpenAI search output, validates source URLs against search annotations, verifies geographic and date constraints, and removes duplicate entries.
    - *Configuration:* Custom JavaScript filtering up to 8 verified events.
    - *Input/Output:* Input from `Retrieve dated sources`; outputs to `Record research correlation`.
  - **Record research correlation** (`n8n-nodes-base.code`)
    - *Role:* Logs execution metadata and correlation details for debugging and tracking.
    - *Configuration:* Custom JavaScript executing `console.log` and passing input items through.
    - *Input/Output:* Input from either `Build clearly labeled simulation` or `Verify geography dates and deduplicate`; outputs to `Return one research response`.
  - **Return one research response** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns the final consolidated event research response to the webhook caller.
    - *Configuration:* HTTP 200 OK response with `Cache-Control: no-store` header, responding with JSON body `{{ $json.response }}`.
    - *Input/Output:* Input from `Record research correlation`; terminal response node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Lesson request | `n8n-nodes-base.webhook` | Entry point for lesson data and questions | None | Validate and calculate | Set up your Concert System |
| Validate and calculate | `n8n-nodes-base.code` | Validates scenario and computes financial metrics | Lesson request | Valid lesson request? | Validate lesson request |
| Valid lesson request? | `n8n-nodes-base.if` | Routes valid lesson requests to AI evaluation | Validate and calculate | Identify concert question, Explain how to correct input | Validate lesson request |
| Explain how to correct input | `n8n-nodes-base.respondToWebhook` | Returns 400 error for invalid lesson input | Valid lesson request? | None | Explain input correction |
| Identify concert question | `@typesafe-ai/n8n-nodes-typesafe-ai.typeSafeAi` | Classifies learner question intent using TypeSafe AI | Valid lesson request? | Read intent confidence | Classify lesson intent |
| Read intent confidence | `n8n-nodes-base.code` | Checks intent classification confidence threshold | Identify concert question | Confident supported intent? | Classify lesson intent |
| Confident supported intent? | `n8n-nodes-base.if` | Routes based on intent classification confidence | Read intent confidence | Select lesson explanation, Ask focused clarification | Classify lesson intent |
| Ask focused clarification | `n8n-nodes-base.code` | Prepares clarification response for low confidence | Confident supported intent? | Deliver interactive lesson | Assemble lesson response |
| Select lesson explanation | `n8n-nodes-base.httpRequest` | Calls OpenAI to select approved explanation paragraphs | Confident supported intent? | Check explanation or use prepared lesson | Assemble lesson response |
| Check explanation or use prepared lesson | `n8n-nodes-base.code` | Validates OpenAI output or falls back to prepared text | Select lesson explanation | Deliver interactive lesson | Assemble lesson response |
| Deliver interactive lesson | `n8n-nodes-base.respondToWebhook` | Returns final lesson JSON to client | Ask focused clarification, Check explanation or use prepared lesson | None | Assemble lesson response |
| Event research request | `n8n-nodes-base.webhook` | Entry point for event research and planning | None | Validate planning interval | Set up your Concert System |
| Validate planning interval | `n8n-nodes-base.code` | Validates city, dates, and builds research queries | Event research request | Valid planning request? | Validate research request |
| Valid planning request? | `n8n-nodes-base.if` | Routes valid planning requests to sourcing | Validate planning interval | Use illustrative listings?, Return invalid request | Validate research request |
| Return invalid request | `n8n-nodes-base.respondToWebhook` | Returns 400 error for invalid planning requests | Valid planning request? | None | Return planning error |
| Use illustrative listings? | `n8n-nodes-base.if` | Routes between simulation and live web search | Valid planning request? | Build clearly labeled simulation, Retrieve dated sources | Prepare listing sources |
| Build clearly labeled simulation | `n8n-nodes-base.code` | Generates fictional simulation event listings | Use illustrative listings? | Record research correlation | Prepare listing sources |
| Retrieve dated sources | `n8n-nodes-base.httpRequest` | Queries OpenAI web search for live event listings | Use illustrative listings? | Verify geography dates and deduplicate | Prepare listing sources |
| Verify geography dates and deduplicate | `n8n-nodes-base.code` | Verifies and deduplicates live search event results | Retrieve dated sources | Record research correlation | Prepare listing sources |
| Record research correlation | `n8n-nodes-base.code` | Logs correlation details for research execution | Build clearly labeled simulation, Verify geography dates and deduplicate | Return one research response | Correlate research response |
| Return one research response | `n8n-nodes-base.respondToWebhook` | Returns final event research payload to client | Record research correlation | None | Correlate research response |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Header Auth Credentials:**
   - In n8n, create a new Header Auth credential. Set the header name to `X-Lesson-Key` and provide a secure random secret value.
2. **Create OpenAI Credentials:**
   - Create a standard OpenAI API credential with your active API key.
3. **Install Community Node:**
   - Ensure the `@typesafe-ai/n8n-nodes-typesafe-ai` community node is installed in your n8n environment and configure your TypeSafe AI credentials.
4. **Build the Lesson Request Branch:**
   - **Node 1 (`Lesson request`):** Add a **Webhook** node. Set HTTP Method to `POST`, path to `economics-concert-v1`, Authentication to `Header Auth` (select your `X-Lesson-Key` credential), and Response Mode to `Response Node`.
   - **Node 2 (`Validate and calculate`):** Add a **Code** node connected downstream. Insert the JavaScript scenario validation and calculation logic provided in the workflow source.
   - **Node 3 (`Valid lesson request?`):** Add an **If** node. Configure a condition where `{{ $json.valid }}` is boolean `true`.
   - **Node 4 (`Explain how to correct input`):** Add a **Respond to Webhook** node connected to the false output of `Valid lesson request?`. Set Response Code to `400` and body expression to `{{ { error: $json.error } }}`.
5. **Build the Intent Classification Branch:**
   - **Node 5 (`Identify concert question`):** Add a **TypeSafe AI** node connected to the true output of `Valid lesson request?`. Set Model to `jev-latest`, operation to `evaluate`, state expression to `={{ JSON.stringify({question:$json.request.question, ...}) }}`, and configure intent choice schemas. Select your TypeSafe AI credential.
   - **Node 6 (`Read intent confidence`):** Add a **Code** node to parse classification confidence against an 0.8 threshold.
   - **Node 7 (`Confident supported intent?`):** Add an **If** node checking if `{{ $json.decision.explain }}` is boolean `true`.
   - **Node 8 (`Ask focused clarification`):** Add a **Code** node connected to the false output to generate clarification text.
6. **Build the OpenAI Lesson Assembly Branch:**
   - **Node 9 (`Select lesson explanation`):** Add an **HTTP Request** node connected to the true output of `Confident supported intent?`. Set Method to `POST`, URL to `https://api.openai.com/v1/responses`, authentication to predefined OpenAI credential, and timeout to `10000`ms. Set error handling to `Continue Regular Output`.
   - **Node 10 (`Check explanation or use prepared lesson`):** Add a **Code** node to validate OpenAI's paragraph selection against approved content.
   - **Node 11 (`Deliver interactive lesson`):** Add a **Respond to Webhook** node connected to both `Ask focused clarification` and `Check explanation or use prepared lesson`. Set Response Code to `200`, response headers with `Cache-Control: no-store`, and body expression to `={{ $json }}`.
7. **Build the Event Research Branch:**
   - **Node 12 (`Event research request`):** Add a **Webhook** node. Set HTTP Method to `POST`, path to `concert-events-v1`, Authentication to `Header Auth` (select your `X-Lesson-Key` credential), and Response Mode to `Response Node`.
   - **Node 13 (`Validate planning interval`):** Add a **Code** node to validate city IDs, dates, and build research envelopes.
   - **Node 14 (`Valid planning request?`):** Add an **If** node checking if `{{ $json.valid }}` is boolean `true`.
   - **Node 15 (`Return invalid request`):** Add a **Respond to Webhook** node on the false output. Set Response Code to `400` and body to `{{ $json.response }}`.
   - **Node 16 (`Use illustrative listings?`):** Add an **If** node on the true output checking if `{{ $json.plan.dataMode === 'simulation' }}`.
   - **Node 17 (`Build clearly labeled simulation`):** Add a **Code** node on the true branch to generate simulated event datasets.
   - **Node 18 (`Retrieve dated sources`):** Add an **HTTP Request** node on the false branch of `Use illustrative listings?`. Set Method to `POST`, URL to `https://api.openai.com/v1/responses`, select your OpenAI credential, and set timeout to `50000`ms. Set error handling to `Continue Regular Output`.
   - **Node 19 (`Verify geography dates and deduplicate`):** Add a **Code** node to process, filter, and verify live search results.
   - **Node 20 (`Record research correlation`):** Add a **Code** node connected to both simulation and verification outputs to log execution metrics.
   - **Node 21 (`Return one research response`):** Add a **Respond to Webhook** node. Set Response Code to `200`, headers to `Cache-Control: no-store`, and body expression to `={{ $json.response }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Source code repository | https://github.com/Empower21/economics-of-everything-flow4gold |
| Complete setup instructions | https://github.com/Empower21/economics-of-everything-flow4gold/blob/main/submission/TEMPLATE-SETUP.txt |
| Live demonstration | https://web-production-af12a.up.railway.app |
| Walkthrough video | https://youtu.be/JP_wxRAzGkc |