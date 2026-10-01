Batch structured AI calls via OpenRouter, Data Tables, and webhooks

https://n8nworkflows.xyz/workflows/batch-structured-ai-calls-via-openrouter--data-tables--and-webhooks-20202


# Batch structured AI calls via OpenRouter, Data Tables, and webhooks

### 1. Workflow Overview

This workflow provides an authenticated, batch-processing endpoint designed for AI coding agents or external scripts. It accepts a POST request containing multiple structured generation tasks, validates the caller's API key against a dedicated Data Table, logs the activity, processes up to 50 items in parallel using OpenRouter models, enforces JSON schemas, and returns an ordered array of structured results.

The execution logic is partitioned into three functional blocks:
- **1.1 Input Reception and Authentication Gate:** Captures the incoming HTTP request, extracts the `X-API-Key` header and payload, delegates validation to a sub-workflow check against the `users` Data Table, logs authorized payloads to the `logs` Data Table, and issues an HTTP 401 response for invalid credentials.
- **1.2 Batch Preparation and Rate Limiting:** Restores the request body after authentication, splits the array of calls into individual items, and enforces a safety ceiling by retaining only the first 50 items.
- **1.3 AI Processing and Response Aggregation:** Dynamically routes items based on the requested model tier (`small` vs. `big`), executes the LangChain LLM chain in parallel batches, parses responses against user-supplied JSON schemas, aggregates the outcomes, and formats the final JSON response for the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authentication Gate

- **Overview:** Receives incoming HTTP POST calls, validates the authentication token against stored user records, logs valid requests, and rejects unauthorized traffic with an HTTP 401 error.
- **Nodes Involved:** Webhook, Check Auth, Valid?, Error, Stop and Error, Sub - Check auth, Get team member, Found?, Insert log, Success, Stop and Error1, Continue.
- **Node Details:**
  - **Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration Choices:* Listens for incoming `POST` requests on the path `externalized-ai-service` using the response mode `responseNode`.
    - *Key Expressions/Variables:* None (entry point).
    - *Input/Output:* Output connects to `Check Auth`.
    - *Version-specific Requirements:* Version 2.1.
    - *Edge Cases / Failure Types:* Public endpoint exposure; ensure rate limiting is handled externally if abused.
  - **Check Auth**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Action / Sub-workflow caller)
    - *Configuration Choices:* Invokes the workflow itself to process authentication, passing the request body calls and header API key. Configured to continue regular output on error.
    - *Key Expressions/Variables:* `ai_auth_key` = `={{ $json.headers['x-api-key'] }}`, `calls` = `={{ $json.body.calls }}`.
    - *Input/Output:* Input from `Webhook`; output connects to `Valid?`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases / Failure Types:* Sub-workflow recursion limits or missing headers resulting in empty keys.
  - **Valid?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether the sub-workflow authentication lookup returned a status of `success`.
    - *Key Expressions/Variables:* `={{ $('Check Auth').item.json.status }}` equals `success`.
    - *Input/Output:* Input from `Check Auth`; outputs connect to `Continue` (true) and `Error` (false).
    - *Version-specific Requirements:* Version 2.3.
  - **Error**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (Action)
    - *Configuration Choices:* Responds with HTTP code `401` and a plain text authentication failure message.
    - *Input/Output:* Input from `Valid?` (false branch); output connects to `Stop and Error`.
    - *Version-specific Requirements:* Version 1.5.
  - **Stop and Error**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (Action)
    - *Configuration Choices:* Terminates the execution path with a generic error message.
    - *Input/Output:* Input from `Error`.
  - **Continue**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control)
    - *Configuration Choices:* Passes execution through cleanly after successful validation.
    - *Input/Output:* Input from `Valid?` (true branch); output connects to `Prepare Calls`.
  - **Sub - Check auth**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflowTrigger` (Trigger)
    - *Configuration Choices:* Receives inputs `ai_auth_key` and `calls` from the parent workflow instance.
    - *Input/Output:* Output connects to `Get team member`.
  - **Get team member**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Database Action)
    - *Configuration Choices:* Queries the `Externalized AI Service - Users` Data Table using a filter where `auth_token` matches `ai_auth_key`. Always outputs data.
    - *Key Expressions/Variables:* `keyValue` = `={{ $json.ai_auth_key }}`.
    - *Input/Output:* Input from `Sub - Check auth`; output connects to `Found?`.
  - **Found?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Checks whether a record ID exists in the lookup result.
    - *Key Expressions/Variables:* `={{ $json.id }}` exists.
    - *Input/Output:* Input from `Get team member`; outputs connect to `Insert log` (true) and `Stop and Error1` (false).
  - **Insert log**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Database Action)
    - *Configuration Choices:* Inserts a row into the `Externalized AI Service - Logs` Data Table recording the user's email and the serialized calls payload.
    - *Key Expressions/Variables:* `user` = `={{ $json.email }}`, `log` = `={{ $('Sub - Check auth').item.json.calls.toJsonString() }}`.
    - *Input/Output:* Input from `Found?` (true); output connects to `Success`.
  - **Success**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Sets a static string assignment returning `status: success`.
    - *Input/Output:* Input from `Insert log`; returns data to the parent workflow's `Check Auth` node.
  - **Stop and Error1**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (Action)
    - *Configuration Choices:* Terminates execution with an auth error message containing the execution ID.
    - *Input/Output:* Input from `Found?` (false).

---

#### 2.2 Batch Preparation and Rate Limiting

- **Overview:** Restores the original payload structure post-authentication, splits the array of calls into distinct execution items, and caps the maximum items processed per request.
- **Nodes Involved:** Prepare Calls, Split Calls, Limit to 50.
- **Node Details:**
  - **Prepare Calls**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Restores the webhook body payload following the authentication sub-workflow boundary.
    - *Key Expressions/Variables:* `body` = `={{ $('Webhook').first().json.body ?? {} }}`.
    - *Input/Output:* Input from `Continue`; output connects to `Split Calls`.
    - *Version-specific Requirements:* Version 3.4.
  - **Split Calls**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Data Transformation)
    - *Configuration Choices:* Splits the array field `body.calls` into individual independent items.
    - *Input/Output:* Input from `Prepare Calls`; output connects to `Limit to 50`.
    - *Version-specific Requirements:* Version 1.0.
  - **Limit to 50**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Flow Control)
    - *Configuration Choices:* Restricts item flow to a maximum of 50 items, silently dropping extras.
    - *Input/Output:* Input from `Split Calls`; output connects to `Basic LLM Chain`.
    - *Version-specific Requirements:* Version 1.0.
    - *Edge Cases / Failure Types:* Silent data loss if clients submit arrays exceeding 50 items without pre-segmenting.

---

#### 2.3 AI Processing and Response Aggregation

- **Overview:** Routes requests to appropriate language models based on the specified tier, executes structured generation using JSON schemas, aggregates responses, and formats output.
- **Nodes Involved:** Basic LLM Chain, Structured Output Parser, Model Selector, GPT 6 Luna, 3.8 Flash, Aggregate Responses, Respond to Webhook.
- **Node Details:**
  - **Basic LLM Chain**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI / LangChain Core)
    - *Configuration Choices:* Processes items using defined prompts, with batching configured to a batch size of 50 and a 2000ms delay between batches. Retries on failure up to 2 times.
    - *Key Expressions/Variables:* Text = `={{ $json.userMessage }}`, Message = `={{ $json.systemPrompt }}`.
    - *Input/Output:* Inputs from `Limit to 50` and `Model Selector` (model & output parser); output connects to `Aggregate Responses`.
    - *Version-specific Requirements:* Version 1.9.
  - **Structured Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI / LangChain Output Parser)
    - *Configuration Choices:* Manual schema type utilizing an auto-fix mechanism.
    - *Key Expressions/Variables:* Input Schema = `={{ $json.jsonSchema }}`.
    - *Input/Output:* Input from `Model Selector`; output connects to `Basic LLM Chain`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases / Failure Types:* Malformed JSON schemas provided by callers can cause parser evaluation errors.
  - **Model Selector**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.modelSelector` (AI / LangChain Routing)
    - *Configuration Choices:* Routes execution based on rules: evaluates `aiMode` equals `small` to route to model index 0 (GPT 6 Luna), and `aiMode` equals `big` to route to model index 2 (3.8 Flash).
    - *Key Expressions/Variables:* `={{ $json.aiMode }}`.
    - *Input/Output:* Connects to `GPT 6 Luna`, `3.8 Flash`, `Basic LLM Chain`, and `Structured Output Parser`.
    - *Version-specific Requirements:* Version 1.0.
  - **GPT 6 Luna**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (AI / Language Model)
    - *Configuration Choices:* Uses model `openai/gpt-6-luna` with a 45-second timeout and JSON object response format.
    - *Credentials:* OpenRouter account.
    - *Input/Output:* Connects to `Model Selector`.
  - **3.8 Flash**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (AI / Language Model)
    - *Configuration Choices:* Uses model `google/gemini-3.8-flash` with an 80-second timeout and JSON object response format.
    - *Credentials:* OpenRouter account.
    - *Input/Output:* Connects to `Model Selector`.
  - **Aggregate Responses**
    - *Type and Technical Role:* `n8n-nodes-base.aggregate` (Data Transformation)
    - *Configuration Choices:* Aggregates all item data into a single consolidated item.
    - *Input/Output:* Input from `Basic LLM Chain`; output connects to `Respond to Webhook`.
    - *Version-specific Requirements:* Version 1.0.
  - **Respond to Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (Action)
    - *Configuration Choices:* Returns a JSON response mapping outputs alongside their original query inputs.
    - *Key Expressions/Variables:* `={{ $json.data.map((item, index) => ({ query: $('Split Calls').all()[index].json, output: item.output })) }}`.
    - *Input/Output:* Input from `Aggregate Responses`.
    - *Version-specific Requirements:* Version 1.5.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Webhook** | n8n-nodes-base.webhook | Receives incoming POST requests | None | Check Auth | Structured AI batches for agents / Use this for your AI Agent's skill |
| **Error** | n8n-nodes-base.respondToWebhook | Returns HTTP 401 on auth failure | Valid? | Stop and Error | Authentication gate |
| **Stop and Error** | n8n-nodes-base.stopAndError | Terminates failed authentication path | Error | None | Authentication gate |
| **Check Auth** | n8n-nodes-base.executeWorkflow | Invokes authentication lookup | Webhook | Valid? | Authentication gate |
| **Valid?** | n8n-nodes-base.if | Evaluates authentication success | Check Auth | Continue, Error | Authentication gate |
| **Respond to Webhook** | n8n-nodes-base.respondToWebhook | Returns structured JSON output | Aggregate Responses | None | Response shape |
| **Basic LLM Chain** | @n8n/n8n-nodes-langchain.chainLlm | Executes parallel LLM calls | Limit to 50, Model Selector, Structured Output Parser | Aggregate Responses | AI step |
| **Structured Output Parser** | @n8n/n8n-nodes-langchain.outputParserStructured | Applies JSON schema to model output | Model Selector | Basic LLM Chain | AI step |
| **Model Selector** | @n8n/n8n-nodes-langchain.modelSelector | Routes calls by `aiMode` tier | GPT 6 Luna, 3.8 Flash | Basic LLM Chain, Structured Output Parser | AI step |
| **3.8 Flash** | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Language model for `big` tier | None | Model Selector | AI step / Connect models and credentials |
| **Found?** | n8n-nodes-base.if | Checks team member table lookup result | Get team member | Insert log, Stop and Error1 | Auth and audit sub-workflow |
| **Success** | n8n-nodes-base.set | Returns success status object | Insert log | None | Auth and audit sub-workflow |
| **Get team member** | n8n-nodes-base.dataTable | Queries user credentials table | Sub - Check auth | Found? | Auth and audit sub-workflow / Connect the users table |
| **Stop and Error1** | n8n-nodes-base.stopAndError | Stops execution on invalid lookup | Found? | None | Auth and audit sub-workflow |
| **Insert log** | n8n-nodes-base.dataTable | Logs request payload to database | Found? | Success | Auth and audit sub-workflow / Connect the logs table |
| **Sub - Check auth** | n8n-nodes-base.executeWorkflowTrigger | Triggers auth sub-workflow execution | None | Get team member | Auth and audit sub-workflow |
| **GPT 6 Luna** | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Language model for `small` tier | None | Model Selector | AI step / Connect models and credentials |
| **Split Calls** | n8n-nodes-base.splitOut | Splits calls array into individual items | Prepare Calls | Limit to 50 | Batch preparation |
| **Aggregate Responses** | n8n-nodes-base.aggregate | Combines processed item responses | Basic LLM Chain | Respond to Webhook | Response shape |
| **Continue** | n8n-nodes-base.noOp | Passes verified execution flow | Valid? | Prepare Calls | Authentication gate |
| **Limit to 50** | n8n-nodes-base.limit | Enforces maximum 50-item threshold | Split Calls | Basic LLM Chain | Batch preparation |
| **Overview** | n8n-nodes-base.stickyNote | Workflow documentation & overview | None | None | Structured AI batches for agents |
| **Agent skill and onboarding** | n8n-nodes-base.stickyNote | Agent integration instructions | None | None | Use this for your AI Agent's skill |
| **Auth before AI** | n8n-nodes-base.stickyNote | Authentication gate notes | None | None | Authentication gate |
| **Batch and model routing** | n8n-nodes-base.stickyNote | Batch preparation notes | None | None | Batch preparation |
| **Results and logging** | n8n-nodes-base.stickyNote | Response shape notes | None | None | Response shape |
| **Credential lookup and audit** | n8n-nodes-base.stickyNote | Sub-workflow architecture notes | None | None | Auth and audit sub-workflow |
| **Prepare Calls** | n8n-nodes-base.set | Restores request body context | Continue | Split Calls | Batch preparation |
| **Batch and model routing1** | n8n-nodes-base.stickyNote | AI step configuration notes | None | None | AI step |
| **Sticky Note** | n8n-nodes-base.stickyNote | Instructions to connect users table | None | None | Connect the users table |
| **Sticky Note2** | n8n-nodes-base.stickyNote | Instructions to connect model nodes | None | None | Connect models and credentials |
| **Sticky Note3** | n8n-nodes-base.stickyNote | Instructions to connect logs table | None | None | Connect the logs table |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Project Resources (Data Tables)
1. Create a Data Table named `users` with string columns `email` and `auth_token`.
2. Create a Data Table named `logs` with string columns `user` and `log`.
3. Add at least one test user row with an alphanumeric `auth_token` and a valid `email`.

#### Step 2: Build the Sub-Workflow (Auth & Logging)
1. Create a new n8n workflow (or use the main workflow set to call itself).
2. Add an **Execute Workflow Trigger** node (`Sub - Check auth`) with inputs: `ai_auth_key` (string) and `calls` (array).
3. Connect it to a **Data Table** node (`Get team member`) configured to get records from the `users` table, filtering where `auth_token` equals `={{ $json.ai_auth_key }}`. Enable "Always Output Data".
4. Connect to an **If** node (`Found?`) checking if `={{ $json.id }}` exists.
5. On the *false* branch, connect a **Stop and Error** node (`Stop and Error1`) with message `=Auth error for our custom endpoint.\n\nExecution ID: {{ $execution.id }}`.
6. On the *true* branch, connect a **Data Table** node (`Insert log`) configured to insert rows into the `logs` table, setting `user` to `={{ $json.email }}` and `log` to `={{ $('Sub - Check auth').item.json.calls.toJsonString() }}`.
7. Connect `Insert log` to a **Set** node (`Success`) assigning `status` to `success`.

#### Step 3: Build the Main Workflow Trigger and Auth Gate
1. Add a **Webhook** node configured for method `POST`, path `externalized-ai-service`, with response mode set to `Response Node`.
2. Connect to an **Execute Workflow Function** node (`Check Auth`) configured to call the current workflow ID. Map workflow inputs: `calls` = `={{ $json.body.calls }}` and `ai_auth_key` = `={{ $json.headers['x-api-key'] }}`. Enable "Continue Regular Output".
3. Connect to an **If** node (`Valid?`) evaluating `={{ $('Check Auth').item.json.status }}` equals `success`.
4. On the *false* branch, connect a **Respond to Webhook** node (`Error`) with response code `401` and text response body. Connect this to a **Stop and Error** node.
5. On the *true* branch, connect a **NoOp** node (`Continue`).

#### Step 4: Configure Batch Preparation and Limits
1. Connect `Continue` to a **Set** node (`Prepare Calls`) assigning `body` = `={{ $('Webhook').first().json.body ?? {} }}`.
2. Connect to a **Split Out** node (`Split Calls`) setting the field to split out as `body.calls`.
3. Connect to a **Limit** node (`Limit to 50`) setting max items to `50`.

#### Step 5: Configure AI Components and Model Routing
1. Add two **OpenRouter Chat Model** nodes:
   - Name: `GPT 6 Luna`, Model: `openai/gpt-6-luna`, Timeout: `45000`, response format: `json_object`. Configure OpenRouter API credentials.
   - Name: `3.8 Flash`, Model: `google/gemini-3.8-flash`, Timeout: `80000`, response format: `json_object`. Configure OpenRouter API credentials.
2. Add a **Model Selector** node (`Model Selector`). Connect `GPT 6 Luna` to model index 0 and `3.8 Flash` to model index 2.
   - Set Rule 1 (`small`): Condition where `={{ $json.aiMode }}` equals `small`.
   - Set Rule 2 (`big`): Condition where `={{ $json.aiMode }}` equals `big`.
3. Add a **Structured Output Parser** node (`Structured Output Parser`) with schema type manual, input schema set to `={{ $json.jsonSchema }}`, and auto-fix enabled. Connect its AI output parser input from `Model Selector`.
4. Add a **Basic LLM Chain** node (`Basic LLM Chain`):
   - Set Prompt Type to **Define**.
   - Text: `={{ $json.userMessage }}`
   - Messages Message: `={{ $json.systemPrompt }}`
   - Batching: Batch size `50`, delay between batches `2000`ms.
   - Connect AI Language Model input from `Model Selector` and AI Output Parser input from `Structured Output Parser`.
   - Connect the main input from `Limit to 50`.

#### Step 6: Configure Output Aggregation and Webhook Response
1. Connect `Basic LLM Chain` to an **Aggregate** node (`Aggregate Responses`) aggregating all item data.
2. Connect to a **Respond to Webhook** node (`Respond to Webhook`) responding with JSON. Set response body expression:
   ```json
   ={{
   $json.data.map((item, index) => ({
     query: $('Split Calls').all()[index].json,
     output: item.output
   }))
   }}
   ```
3. Save and publish the workflow, taking note of the production webhook URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| OpenRouter Documentation | [OpenRouter Quickstart Guide](https://openrouter.ai/docs/quickstart) |
| Data Retention Policy | The `logs` table retains submitted prompts, source text, schemas, and modes. Restrict access and set retention policies appropriately. |
| Token Distribution | Deliver unique 12-character alphanumeric tokens to users exclusively through verified private channels. |
| Client Environment Variables | Callers should store credentials locally as `EXTERNALIZED_AI_URL` and `EXTERNALIZED_AI_API_TOKEN` in a `.env` file without quotes. |