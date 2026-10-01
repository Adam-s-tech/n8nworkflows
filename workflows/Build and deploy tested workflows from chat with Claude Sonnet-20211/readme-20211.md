Build and deploy tested workflows from chat with Claude Sonnet

https://n8nworkflows.xyz/workflows/build-and-deploy-tested-workflows-from-chat-with-claude-sonnet-20211


# Build and deploy tested workflows from chat with Claude Sonnet

### 1. Workflow Overview

This workflow functions as an intelligent, autonomous n8n workflow generator named **The Alchemist**. Its primary purpose is to take a natural language description of an automation goal provided via chat, convert it into a structured technical specification, generate, validate, and test/review the resulting workflow JSON, and finally deploy it directly to the active n8n instance. 

The application architecture relies on a main orchestration lane and a recursive sub-execution engine using `Execute Workflow` triggers (via the `Section Router` pattern) to manage different processing stages asynchronously and avoid execution timeouts.

#### Logical Blocks:
- **1.1 Input Reception & Initialization:** Handles chat inputs, enforces daily rate limits and usage quotas, checks instance connectivity, and initializes the state object.
- **1.2 Section Routing:** Central dispatcher that maps execution stages (`understand`, `build`, `check`, `review`, `test`, `deploy`, `catalog`) to their respective operational branches.
- **1.3 Section 1 - Understand:** Probes user endpoints, guards against prompt injections and private/internal IP ranges (SSRF protection), and converts the prompt into a typed JSON specification (`monitor` or `blueprint`).
- **1.4 Section 2 - Build:** Pulls pinned node catalogs, builds strict code or blueprint specifications via Claude Sonnet, parses raw model text into valid workflow JSON objects, and manages iterative repair loops.
- **1.5 Section 3 - Check:** Validates generated workflows against the official n8n node catalog schemas, ensuring parameter correctness, proper expression syntax, and blocking unauthorized code-execution nodes.
- **1.6 Section 4 - Review:** Performs a secondary LLM logic review on blueprints, aggregates warnings, and arranges node coordinates before generating final setup notes.
- **1.7 Section 5 - Test:** Builds synthetic test payloads for monitors (errors, timeouts, malformed bodies, DNS failures) and executes them against a local mock web server, followed by a live smoke test.
- **1.8 Section 6 - Deploy:** Creates workflows via the n8n REST API, selectively auto-activates fully configured monitors, or provisions blueprints in an inactive state with actionable configuration notes.
- **1.9 Support Sections (Catalog, Mock):** Fetches version-pinned n8n node metadata/schemas from jsDelivr and runs an internal testing webhook (`alchemist-mock`) for evaluating health monitor responses.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Initialization

##### Overview
This block receives user entries from the chat interface, validates configuration constants, applies throttling limits, checks accessibility of the internal test mock, and instantiates the execution state object.

##### Nodes Involved:
- Chat Trigger
- Settings
- Read Request
- Check Mock Webhook
- Initialize Case

##### Node Details:

- **Chat Trigger**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.chatTrigger` (v1.1) — Public chat interface entry point.
  - **Configuration:** Configured for public access (`public: true`), using the `responseNode` response mode. Sets initial welcome messages and input placeholders.
  - **Expressions:** None.
  - **Connections:** Input: None (Trigger). Output: `Settings`.
  - **Failure Types:** Authentication issues if Basic Auth is misconfigured.

- **Settings**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.5) — Configuration store.
  - **Configuration:** Declares global environment parameters including `PUBLIC_URL`, `CORE_NODES_VERSION` (`2.39.7`), `AI_NODES_VERSION` (`2.39.8`), `MAX_ATTEMPTS` (`3`), `TIME_BUDGET_MINUTES` (`12`), `PENDING_QUESTION_MINUTES` (`30`), and rate limits (`DAILY_RUNS_PER_CHAT`, `DAILY_RUNS_TOTAL`).
  - **Expressions:** None.
  - **Connections:** Input: `Chat Trigger`. Output: `Read Request`.
  - **Failure Types:** None.

- **Read Request**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript parser.
  - **Configuration:** Extracts goals, target URLs, alert webhooks, emails, and Slack channels from the incoming chat input string. Computes session identifiers and checks static execution limits.
  - **Expressions:** Reads `$json.chatInput`, `$json.PUBLIC_URL`, and `$json.PENDING_QUESTION_MINUTES`.
  - **Connections:** Input: `Settings`. Output: `Check Mock Webhook`.
  - **Failure Types:** Script execution halts if regex matching fails on malformed input.

- **Check Mock Webhook**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — HTTP connectivity test.
  - **Configuration:** Sends a GET probe to the local mock webhook endpoint to confirm readiness. Sets a 10-second timeout and ignores error codes (`neverError: true`).
  - **Expressions:** `={{ $json.base + '/webhook/alchemist-mock?status=200&type=json&body=%7B%22mock%22%3A1%7D' }}`
  - **Connections:** Input: `Read Request`. Output: `Initialize Case`.
  - **Failure Types:** Network timeouts or unreachable self-hosted webhook routes.

- **Initialize Case**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — State builder.
  - **Configuration:** Validates goal length, screens targets against SSRF/private network ranges (loopback, metadata IPs, internal subnets), and initializes the core application execution state object.
  - **Expressions:** Reads upstream data from `Read Request` and `Settings`.
  - **Connections:** Input: `Check Mock Webhook`. Output: `Case Finished?`.
  - **Failure Types:** Throws initialization errors if request structures are corrupted.

---

#### 2.2 Section Routing & Orchestration

##### Overview
Controls the recursive execution flow of the workflow by checking task completion status, routing execution stages through sub-workflow calls, and formatting final chat outputs.

##### Nodes Involved:
- Case Finished?
- Run Section
- Decide Next Step
- Format Chat Reply
- Send Reply
- Section Trigger
- Route to Section
- Unknown Stage

##### Node Details:

- **Case Finished?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.2) — Conditional gate.
  - **Configuration:** Evaluates whether the run has completed (`done: true`).
  - **Expressions:** `={{ $json.done }}`
  - **Connections:** Input: `Initialize Case`. Outputs: True -> `Format Chat Reply`, False -> `Run Section`.
  - **Failure Types:** Evaluation errors if boolean properties are missing.

- **Run / Decide Nodes (Run Section, Decide Next Step, Format Chat Reply, Send Reply)**
  - **Type & Technical Roles:** `executeWorkflow` (v1.2), `code` (v2), `code` (v2), `respondToWebhook` (v1.1).
  - **Configuration:** Manages asynchronous callback loops by re-invoking the active workflow ID with stage parameters, analyzing step results, updating retry counters, and compiling Markdown responses for the chat UI.
  - **Expressions:** Workflow IDs dynamically reference `={{ $workflow.id }}`. Response codes use `={{ $json.httpStatus }}`.
  - **Connections:** Interconnected in a cyclical loop between intake validation, worker sections, and reply handlers.
  - **Failure Types:** Sub-workflow execution failures, infinite recursion if time budgets fail.

- **Section Trigger & Route to Section**
  - **Type & Technical Roles:** `executeWorkflowTrigger` (v1.1), `switch` (v3.2).
  - **Configuration:** Serves as the entry gateway for sub-workflow calls originating from `Run Section`. Uses expression math to index stage names into eight discrete output ports.
  - **Expressions:** `={{ (['understand', 'build', 'test', 'deploy', 'review', 'catalog', 'check'].indexOf($json.stage) + 8) % 8 }}`
  - **Connections:** Inputs from sub-workflow execution calls. Outputs route to sections 1 through 8 or `Unknown Stage`.
  - **Failure Types:** Stage routing misses if unhandled string literals are passed.

---

#### 2.3 Section 1: Understand

##### Overview
Interprets user requests, tests target endpoints via live HTTP probes, prompts the Claude model to generate a strict JSON specification (`monitor` or `blueprint`), and sanitizes inputs against prompt injections and private IP ranges.

##### Nodes Involved:
- Start Understand
- Has Endpoint?
- Probe Endpoint
- Build Spec Prompt
- Anthropic Model (Spec)
- Write Spec (LLM)
- Validate Spec
- Spec Valid?
- Retry Spec?
- End Understand

##### Node Details:

- **Probe Endpoint**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Endpoint health pre-checker.
  - **Configuration:** Executes an unauthenticated GET request against the user-supplied target URL with redirects disabled (`followRedirects: false`) and a 10-second timeout.
  - **Expressions:** `={{ $json.state.userTargetUrl }}`
  - **Connections:** Input: `Has Endpoint?` (True). Output: `Build Spec Prompt`.
  - **Failure Types:** DNS resolution failures, connection timeouts, TLS handshake errors.

- **Write Spec (LLM) & Anthropic Model (Spec)**
  - **Type & Technical Roles:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.5) & `lmChatAnthropic` (v1.6) — AI generation components.
  - **Configuration:** Configured with `claude-sonnet-4-6`. Generates structured JSON specifications enforcing specific schema keys (`outOfScope`, `kind`, `service`, `targetUrl`, `checks`, `summary`).
  - **Expressions:** `={{ $json.userPrompt }}`
  - **Connections:** Model linked to chain via `ai_languageModel`. Chain outputs to `Validate Spec`.
  - **Failure Types:** Rate limiting (`429`), model overloading (`503`), or malformed JSON responses.

- **Validate Spec**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Spec validator.
  - **Configuration:** Sanitizes model outputs, intercepts prompt injection patterns, validates target URLs against local/private network ranges, and matches health checks against probe responses.
  - **Expressions:** Reads model text responses and execution state variables.
  - **Connections:** Input: `Write Spec (LLM)`. Output: `Spec Valid?`.
  - **Failure Types:** Parsing exceptions on invalid model text output.

---

#### 2.4 Section 2: Build

##### Overview
Generates the target n8n workflow JSON based on the specification. For blueprints, it queries the Node Catalog (Section 7) to inject exact parameter types and versions. For monitors, it enforces the rigid health monitor architectural contract.

##### Nodes Involved:
- Start Build
- Needs Node Catalog?
- Ask for Catalog
- Get Node Catalog
- Catalog Unavailable
- Build Workflow Prompt
- Anthropic Model (Build)
- Write Workflow (LLM)
- Parse Draft JSON
- End Build

##### Node Details:

- **Get Node Catalog**
  - **Type & Technical Role:** `n8n-nodes-base.executeWorkflow` (v1.2) — Sub-workflow catalog fetcher.
  - **Configuration:** Recursively calls the current workflow asking for catalog definitions (`catalogRequest: 'catalog'`).
  - **Expressions:** `={{ $workflow.id }}`
  - **Connections:** Input: `Ask for Catalog`. Output: `Build Workflow Prompt` (or `Catalog Unavailable` on error).
  - **Failure Types:** Execution timeouts or unpublished workflow states.

- **Write Workflow (LLM) & Anthropic Model (Build)**
  - **Type & Technical Roles:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.5) & `lmChatAnthropic` (v1.6) — Workflow builder.
  - **Configuration:** Uses `claude-sonnet-4-6` with defined prompt instructions to assemble complete, valid n8n workflow JSON payloads without hardcoded credentials or system-access nodes.
  - **Expressions:** `={{ $json.userPrompt }}`
  - **Connections:** Model linked via `ai_languageModel`. Output flows to `Parse Draft JSON`.
  - **Failure Types:** Context length limits, malformed JSON string blocks.

- **Parse Draft JSON**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — JSON parser & repairer.
  - **Configuration:** Extracts JSON blocks from markdown tags (`<workflow>...</workflow>`), repairs common syntax errors (trailing commas, unescaped newlines), and indexes catalog definitions.
  - **Expressions:** Reads raw text output from the LLM execution node.
  - **Connections:** Input: `Write Workflow (LLM)`. Output: `End Build`.
  - **Failure Types:** Unrecoverable parse failures if structural braces are mismatched.

---

#### 2.5 Section 3: Check

##### Overview
Performs rigorous structural and semantic validation on generated workflow drafts. Ensures adherence to n8n node catalog rules, blocks unauthorized execution nodes, verifies expression syntax, and collects output schemas.

##### Nodes Involved:
- Start Check
- Blueprint or Monitor?
- Validate Monitor Draft
- Draft Parsed?
- Ask for Output Schemas
- Get Output Schemas
- No Output Schemas
- Validate Blueprint Draft
- End Check

##### Node Details:

- **Validate Monitor Draft**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Monitor verifier.
  - **Configuration:** Enforces strict compliance with the monitor contract (presence of `Schedule`, `Test Entry`, `Config`, `Result OK`, `Result Alert`, correct HTTP GET/POST parameters, and exact expression structures).
  - **Expressions:** Validates state and draft configuration parameters.
  - **Connections:** Input: `Blueprint or Monitor?` (Monitor). Output: `End Check`.
  - **Failure Types:** Validation errors pushed to the repair loop feedback array.

- **Validate Blueprint Draft**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Blueprint schema validator.
  - **Configuration:** Validates node types, type versions, required parameters, resource/operation mappings, connection graph shapes, and expression syntax (`$().item.json` references). Blocks unsafe node types (`code`, `executeCommand`, `ssh`, `n8n`, etc.).
  - **Expressions:** Analyzes draft nodes and loaded output schemas.
  - **Connections:** Input: `Get Output Schemas` or `No Output Schemas`. Output: `End Check`.
  - **Failure Types:** Exception handling blocks script execution halts, collecting issues into warning/error arrays.

---

#### 2.6 Section 4: Review

##### Overview
Executes an advanced qualitative logic review on validated blueprint workflows using a secondary LLM pass, checking if the implementation matches the user's initial intent.

##### Nodes Involved:
- Start Review
- Build Review Prompt
- Anthropic Model (Review)
- Review Workflow (LLM)
- Finish Blueprint

##### Node Details:

- **Review Workflow (LLM) & Anthropic Model (Review)**
  - **Type & Technical Roles:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.5) & `lmChatAnthropic` (v1.6) — Logic auditor.
  - **Configuration:** Uses `claude-sonnet-4-6` to evaluate workflow functionality against verified schema facts, user intent, and potential edge-case failures.
  - **Expressions:** `={{ $json.reviewPrompt }}`
  - **Connections:** Model linked via `ai_languageModel`. Output flows to `Finish Blueprint`.
  - **Failure Types:** LLM timeouts or unparsable review defect arrays.

- **Finish Blueprint**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Blueprint assembler.
  - **Configuration:** Merges validator warnings with review defects, calculates automatic layout coordinates based on distance from trigger nodes, and appends the final Setup sticky note.
  - **Expressions:** Processes incoming execution state and validation objects.
  - **Connections:** Input: `Review Workflow (LLM)`. Output: `End Check` (via router loop).
  - **Failure Types:** Layout calculation errors on cyclic connections.

---

#### 2.7 Section 5: Test

##### Overview
Builds synthetic test harness payloads for health monitors, simulates various failure states (HTTP 500, timeouts, DNS errors, missing JSON fields, failed alert deliveries) against the built-in mock web server, and scores execution results.

##### Nodes Involved:
- Start Test
- Build Test Cases
- Run Tests
- Score Tests

##### Node Details:

- **Build Test Cases**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Test generator.
  - **Configuration:** Analyzes monitor specs to generate arrays of test scenarios targeting the local mock webhook (`/webhook/alchemist-mock`) with custom query parameters (`status`, `delay`, `body`, `type`).
  - **Expressions:** Reads execution state and spec configurations.
  - **Connections:** Input: `Start Test`. Output: `Run Tests`.
  - **Failure Types:** None.

- **Run Tests**
  - **Type & Technical Role:** `n8n-nodes-base.executeWorkflow` (v1.2) — Test runner.
  - **Configuration:** Executes the test workflow version of the draft iteratively for each generated test case using parameter mode with sub-workflow waiting enabled.
  - **Expressions:** `={{ JSON.stringify($('Start Test').first().json.testWorkflow) }}`
  - **Connections:** Input: `Build Test Cases`. Output: `Score Tests`.
  - **Failure Types:** Sub-workflow crash errors if workflow execution fails.

- **Score Tests**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Result analyzer.
  - **Configuration:** Pairs test run outputs with original test cases, evaluates alert expectations against boolean results, checks reason string completeness, and calculates overall test scores.
  - **Expressions:** Parses sub-workflow outputs via `$input.all()`.
  - **Connections:** Input: `Run Tests`. Output: `Decide Next Step` (via main lane router).
  - **Failure Types:** Array indexing mismatches during output pairing.

---

#### 2.8 Section 6: Deploy

##### Overview
Pushes finalized, validated workflows to the n8n instance via the REST API, manages auto-activation logic for fully configured monitors, and formats final delivery verdicts.

##### Nodes Involved:
- Start Deploy
- Create Workflow
- Can Activate?
- Activate Workflow
- Write Verdict
- Report Deploy Error

##### Node Details:

- **Create Workflow**
  - **Type & Technical Role:** `n8n-nodes-base.n8n` (v1) — n8n API client.
  - **Configuration:** Sends a POST request to the local n8n API (`/api/v1/workflows`) to persist the generated workflow JSON. Configured with n8n API credentials and error continuation (`continueErrorOutput`).
  - **Expressions:** `={{ JSON.stringify($('Start Deploy').first().json.workflow) }}`
  - **Connections:** Input: `Start Deploy`. Outputs: Success -> `Can Activate?`, Error -> `Report Deploy Error`.
  - **Failure Types:** Invalid API keys, base URL mismatches, or schema validation rejections by the core n8n API.

- **Activate Workflow**
  - **Type & Technical Role:** `n8n-nodes-base.n8n` (v1) — n8n API client.
  - **Configuration:** Sends an activation request to enable the newly created workflow ID.
  - **Expressions:** Workflow ID sourced from `={{ $('Create Workflow').first().json.id }}`.
  - **Connections:** Input: `Can Activate?` (True). Output: `Write Verdict`.
  - **Failure Types:** Activation errors if required credentials remain unconfigured in external nodes.

- **Write Verdict**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) — Reply formatter.
  - **Configuration:** Compiles deployment statistics, execution durations, dashboard links, test summaries, and next-step onboarding instructions into structured response objects.
  - **Expressions:** Reads execution state and API creation results.
  - **Connections:** Input: `Can Activate?` or `Activate Workflow`. Output: `Case Finished?` (via main lane).
  - **Failure Types:** None.

---

#### 2.9 Section 7: Node Catalog & Section 8: Test Mock

##### Overview
Support services providing external node metadata, parameter schemas, output sample structures (via jsDelivr CDN), and a simulated HTTP webhook testing server (`alchemist-mock`).

##### Nodes Involved:
- Start Catalog
- Schemas or Catalog?
- Fetch Core Node Catalog
- Fetch AI Node Catalog
- Prepare Node Catalog
- Pick Output Schemas
- Fetch Output Schemas
- Collect Output Schemas
- Mock Webhook
- Mock Has Delay?
- Mock Wait
- Mock Respond

##### Node Details:

- **Fetch Core & AI Node Catalogs**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — CDN downloader.
  - **Configuration:** Downloads official n8n node type definitions and schemas from `cdn.jsdelivr.net` pinned to versions `2.39.7` and `2.39.8`. Timeout set to 30 seconds.
  - **Expressions:** Version strings dynamically bound from execution configuration state.
  - **Connections:** Linked sequentially and terminating at `Prepare Node Catalog`.
  - **Failure Types:** CDN network outages or outbound internet access restrictions.

- **Mock Webhook & Mock Respond**
  - **Type & Technical Roles:** `n8n-nodes-base.webhook` (v2) & `respondToWebhook` (v1.1) — Simulated HTTP server.
  - **Configuration:** Listens on path `alchemist-mock`, supporting multiple methods. Parses query parameters (`status`, `type`, `body`, `delay`) to return simulated HTTP responses during monitor testing.
  - **Expressions:** Status codes, content types, and body payloads dynamically parsed from `query` objects.
  - **Connections:** Input: External HTTP calls. Flows through delay handlers to `Mock Respond`.
  - **Failure Types:** Port bindings or webhook route collisions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Workflow documentation & overview | None | None | ## The Alchemist<br>One sentence in. A checked n8n workflow out.<br>... |
| Setup | n8n-nodes-base.stickyNote | Configuration instructions | None | None | ## Setup<br>1. Connect a Claude model to Anthropic Model (Spec)... |
| Section Router | n8n-nodes-base.stickyNote | Routing documentation | None | None | ## Section router<br>Run Section calls this workflow with a stage field set... |
| Chat Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Public chat intake interface | None | Settings | Main lane |
| Settings | n8n-nodes-base.set | Global configuration variables | Chat Trigger | Read Request | Main lane |
| Read Request | n8n-nodes-base.code | Parse chat input, URLs, limits | Settings | Check Mock Webhook | Main lane |
| Check Mock Webhook | n8n-nodes-base.httpRequest | Test internal mock webhook readiness | Read Request | Initialize Case | Main lane |
| Initialize Case | n8n-nodes-base.code | Build execution state & SSRF checks | Check Mock Webhook | Case Finished? | Main lane |
| Case Finished? | n8n-nodes-base.if | Check if execution loop is complete | Initialize Case | Format Chat Reply, Run Section | Main lane |
| Run Section | n8n-nodes-base.executeWorkflow | Execute target workflow section | Case Finished? | Decide Next Step | Main lane |
| Decide Next Step | n8n-nodes-base.code | Manage repair loops & next steps | Run Section | Case Finished? | Main lane |
| Format Chat Reply | n8n-nodes-base.code | Format execution results into markdown | Case Finished? | Send Reply | Main lane |
| Send Reply | n8n-nodes-base.respondToWebhook | Return chat response to user | Format Chat Reply | None | Main lane |
| Section Trigger | n8n-nodes-base.executeWorkflowTrigger | Sub-workflow call entry point | None | Route to Section | Section router |
| Route to Section | n8n-nodes-base.switch | Route execution by stage index | Section Trigger | Section start nodes, Unknown Stage | Section router |
| Section 1: Understand | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 1: Understand<br>**Goal in, spec out.**<br>... |
| Start Understand | n8n-nodes-base.noOp | Section 1 entry point | Route to Section | Has Endpoint? | Section 1: Understand |
| Has Endpoint? | n8n-nodes-base.if | Check if target URL was provided | Start Understand | Probe Endpoint, Build Spec Prompt | Section 1: Understand |
| Probe Endpoint | n8n-nodes-base.httpRequest | HTTP GET probe of target URL | Has Endpoint? | Build Spec Prompt | Section 1: Understand |
| Build Spec Prompt | n8n-nodes-base.code | Construct spec generation prompt | Has Endpoint?, Probe Endpoint | Write Spec (LLM) | Section 1: Understand |
| Anthropic Model (Spec) | @n8n/n8n-nodes-langchain.lmChatAnthropic | AI model provider for specification | None | Write Spec (LLM) | Section 1: Understand |
| Write Spec (LLM) | @n8n/n8n-nodes-langchain.chainLlm | Generate workflow specification | Build Spec Prompt, Anthropic Model (Spec) | Validate Spec, End Understand | Section 1: Understand |
| Validate Spec | n8n-nodes-base.code | Validate spec JSON & security rules | Write Spec (LLM) | Spec Valid? | Section 1: Understand |
| Spec Valid? | n8n-nodes-base.if | Check if spec validation passed | Validate Spec | End Understand, Retry Spec? | Section 1: Understand |
| Retry Spec? | n8n-nodes-base.if | Evaluate spec retry attempts | Spec Valid? | Build Spec Prompt, End Understand | Section 1: Understand |
| End Understand | n8n-nodes-base.code | Finalize Section 1 execution state | Write Spec (LLM), Spec Valid?, Retry Spec? | Decide Next Step | Section 1: Understand |
| Section 2: Build | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 2: Build<br>**Spec in, workflow JSON out.**<br>... |
| Start Build | n8n-nodes-base.noOp | Section 2 entry point | Route to Section | Needs Node Catalog? | Section 2: Build |
| Needs Node Catalog? | n8n-nodes-base.if | Check if blueprint requires node catalog | Start Build | Ask for Catalog, Build Workflow Prompt | Section 2: Build |
| Ask for Catalog | n8n-nodes-base.set | Set parameters for catalog request | Needs Node Catalog? | Get Node Catalog | Section 2: Build |
| Get Node Catalog | n8n-nodes-base.executeWorkflow | Fetch node definitions via self-call | Ask for Catalog | Build Workflow Prompt, Catalog Unavailable | Section 2: Build |
| Catalog Unavailable | n8n-nodes-base.code | Handle missing catalog error | Get Node Catalog | Decide Next Step | Section 2: Build |
| Build Workflow Prompt | n8n-nodes-base.code | Construct workflow generation prompt | Needs Node Catalog?, Get Node Catalog | Write Workflow (LLM) | Section 2: Build |
| Anthropic Model (Build) | @n8n/n8n-nodes-langchain.lmChatAnthropic | AI model provider for workflow building | None | Write Workflow (LLM) | Section 2: Build |
| Write Workflow (LLM) | @n8n/n8n-nodes-langchain.chainLlm | Generate workflow JSON payload | Build Workflow Prompt, Anthropic Model (Build) | Parse Draft JSON, End Build | Section 2: Build |
| Parse Draft JSON | n8n-nodes-base.code | Parse and repair workflow JSON | Write Workflow (LLM) | End Build | Section 2: Build |
| End Build | n8n-nodes-base.code | Finalize Section 2 execution state | Write Workflow (LLM), Parse Draft JSON | Decide Next Step | Section 2: Build |
| Section 3: Check | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 3: Check<br>**Draft in, verified plan or errors out.**<br>... |
| Start Check | n8n-nodes-base.noOp | Section 3 entry point | Route to Section | Blueprint or Monitor? | Section 3: Check |
| Blueprint or Monitor? | n8n-nodes-base.if | Branch between blueprint and monitor | Start Check | Draft Parsed?, Validate Monitor Draft | Section 3: Check |
| Draft Parsed? | n8n-nodes-base.if | Verify draft JSON was successfully parsed | Blueprint or Monitor? | Ask for Output Schemas, Validate Blueprint Draft | Section 3: Check |
| Ask for Output Schemas | n8n-nodes-base.set | Set parameters for output schema fetch | Draft Parsed? | Get Output Schemas | Section 3: Check |
| Get Output Schemas | n8n-nodes-base.executeWorkflow | Fetch node output schemas via self-call | Ask for Output Schemas | Validate Blueprint Draft, No Output Schemas | Section 3: Check |
| No Output Schemas | n8n-nodes-base.code | Fallback when output schemas fail | Get Output Schemas | Validate Blueprint Draft | Section 3: Check |
| Validate Blueprint Draft | n8n-nodes-base.code | Validate blueprint against n8n node catalog | Draft Parsed?, Get Output Schemas, No Output Schemas | End Check | Section 3: Check |
| Validate Monitor Draft | n8n-nodes-base.code | Validate monitor against strict contract | Blueprint or Monitor? | End Check | Section 3: Check |
| End Check | n8n-nodes-base.noOp | Section 3 exit point | Validate Blueprint Draft, Validate Monitor Draft | Decide Next Step | Section 3: Check |
| Section 4: Review | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 4: Review (Blueprints)<br>**Validated draft in, finished blueprint out.**<br>... |
| Start Review | n8n-nodes-base.noOp | Section 4 entry point | Route to Section | Build Review Prompt | Section 4: Review |
| Build Review Prompt | n8n-nodes-base.code | Construct LLM review prompt | Start Review | Review Workflow (LLM) | Section 4: Review |
| Anthropic Model (Review) | @n8n/n8n-nodes-langchain.lmChatAnthropic | AI model provider for review | None | Review Workflow (LLM) | Section 4: Review |
| Review Workflow (LLM) | @n8n/n8n-nodes-langchain.chainLlm | Review logic against user request | Build Review Prompt, Anthropic Model (Review) | Finish Blueprint | Section 4: Review |
| Finish Blueprint | n8n-nodes-base.code | Merge findings, layout nodes, add notes | Review Workflow (LLM) | Decide Next Step | Section 4: Review |
| Section 5: Test | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 5: Test (Monitors)<br>**Draft workflow in, scored results out.**<br>... |
| Start Test | n8n-nodes-base.noOp | Section 5 entry point | Route to Section | Build Test Cases | Section 5: Test |
| Build Test Cases | n8n-nodes-base.code | Generate hidden test cases for monitor | Start Test | Run Tests | Section 5: Test |
| Run Tests | n8n-nodes-base.executeWorkflow | Execute monitor against mock test cases | Build Test Cases | Score Tests | Section 5: Test |
| Score Tests | n8n-nodes-base.code | Score test execution results | Run Tests | Decide Next Step | Section 5: Test |
| Section 6: Deploy | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 6: Deploy<br>**Workflow in, live n8n workflow out.**<br>... |
| Start Deploy | n8n-nodes-base.noOp | Section 6 entry point | Route to Section | Create Workflow | Section 6: Deploy |
| Create Workflow | n8n-nodes-base.n8n | Create workflow in n8n instance via API | Start Deploy | Can Activate?, Report Deploy Error | Section 6: Deploy |
| Can Activate? | n8n-nodes-base.if | Check if monitor is eligible for activation | Create Workflow | Activate Workflow, Write Verdict | Section 6: Deploy |
| Activate Workflow | n8n-nodes-base.n8n | Activate workflow via n8n API | Can Activate? | Write Verdict | Section 6: Deploy |
| Write Verdict | n8n-nodes-base.code | Compile final deployment response | Can Activate?, Activate Workflow | Decide Next Step | Section 6: Deploy |
| Report Deploy Error | n8n-nodes-base.code | Handle API creation failure | Create Workflow | Decide Next Step | Section 6: Deploy |
| Section 7: Node Catalog | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 7: Node Catalog<br>**Catalog in, node definitions out.**<br>... |
| Start Catalog | n8n-nodes-base.noOp | Section 7 entry point | Route to Section | Schemas or Catalog? | Section 7: Node Catalog |
| Schemas or Catalog? | n8n-nodes-base.if | Branch between catalog and schemas | Start Catalog | Pick Output Schemas, Fetch Core Node Catalog | Section 7: Node Catalog |
| Fetch Core Node Catalog | n8n-nodes-base.httpRequest | Download core node catalog from CDN | Schemas or Catalog? | Fetch AI Node Catalog | Section 7: Node Catalog |
| Fetch AI Node Catalog | n8n-nodes-base.httpRequest | Download AI node catalog from CDN | Fetch Core Node Catalog | Prepare Node Catalog | Section 7: Node Catalog |
| Prepare Node Catalog | n8n-nodes-base.code | Index node parameters, versions, credentials | Fetch AI Node Catalog | Decide Next Step | Section 7: Node Catalog |
| Pick Output Schemas | n8n-nodes-base.code | Identify output schema URLs for draft nodes | Schemas or Catalog? | Fetch Output Schemas | Section 7: Node Catalog |
| Fetch Output Schemas | n8n-nodes-base.httpRequest | Download node sample output schemas | Pick Output Schemas | Collect Output Schemas | Section 7: Node Catalog |
| Collect Output Schemas | n8n-nodes-base.code | Compile collected schemas into output map | Fetch Output Schemas | Decide Next Step | Section 7: Node Catalog |
| Section 8: Test Mock | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Section 8: Test Mock<br>**Config in, simulated HTTP response out.**<br>... |
| Mock Webhook | n8n-nodes-base.webhook | Webhook endpoint for test mock bench | None | Mock Has Delay? | Section 8: Test Mock |
| Mock Has Delay? | n8n-nodes-base.if | Check if simulated response requires delay | Mock Webhook | Mock Wait, Mock Respond | Section 8: Test Mock |
| Mock Wait | n8n-nodes-base.wait | Simulate network latency delay | Mock Has Delay? | Mock Respond | Section 8: Test Mock |
| Mock Respond | n8n-nodes-base.respondToWebhook | Return simulated HTTP response | Mock Has Delay?, Mock Wait | None | Section 8: Test Mock |
| Unknown Stage | n8n-nodes-base.code | Handle unrouted stage parameters | Route to Section | Decide Next Step | Section router |

---

### 4. Reproducing the Workflow from Scratch

To recreate this complete n8n workflow manually without importing the JSON, follow these sequential build steps:

1. **Create Base Trigger & Settings:**
   - Create a **Chat Trigger** node (`@n8n/n8n-nodes-langchain.chatTrigger`, v1.1). Set public access to `true`.
   - Create a **Set** node (`n8n-nodes-base.set`, v3.5) named `Settings`. Configure fields: `PUBLIC_URL` (string, `""`), `CORE_NODES_VERSION` (string, `"2.39.7"`), `AI_NODES_VERSION` (string, `"2.39.8"`), `MAX_ATTEMPTS` (number, `3`), `TIME_BUDGET_MINUTES` (number, `12`), `PENDING_QUESTION_MINUTES` (number, `30`), `DAILY_RUNS_PER_CHAT` (number, `10`), `DAILY_RUNS_TOTAL` (number, `50`). Connect `Chat Trigger` -> `Settings`.

2. **Build Intake & Initialization Logic:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Read Request` and paste the JavaScript parsing logic for chat inputs and rate limits. Connect `Settings` -> `Read Request`.
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, v4.2) named `Check Mock Webhook`. Set method to `GET`, URL to `={{ $json.base + '/webhook/alchemist-mock?status=200&type=json&body=%7B%22mock%22%3A1%7D' }}`, timeout to 10000ms, and enable `neverError: true`. Connect `Read Request` -> `Check Mock Webhook`.
   - Add a **Code** node named `Initialize Case` to build the execution state. Connect `Check Mock Webhook` -> `Initialize Case`.

3. **Build Main Orchestration Loop:**
   - Add an **If** node (`n8n-nodes-base.if`, v2.2) named `Case Finished?`. Condition: `={{ $json.done }}` is true. Connect `Initialize Case` -> `Case Finished?`.
   - Add an **Execute Workflow** node (`n8n-nodes-base.executeWorkflow`, v1.2) named `Run Section`. Set workflow ID to `={{ $workflow.id }}` and enable `waitForSubWorkflow`. Connect `Case Finished?` (false branch) -> `Run Section`.
   - Add a **Code** node named `Decide Next Step` to handle repair loops and state transitions. Connect `Run Section` -> `Decide Next Step`. Route output back to `Case Finished?`.
   - Add a **Code** node named `Format Chat Reply` and a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`, v1.1) named `Send Reply`. Connect `Case Finished?` (true branch) -> `Format Chat Reply` -> `Send Reply`.

4. **Build Section Router:**
   - Create an **Execute Workflow Trigger** (`n8n-nodes-base.executeWorkflowTrigger`, v1.1) named `Section Trigger`.
   - Add a **Switch** node (`n8n-nodes-base.switch`, v3.2) named `Route to Section`. Use expression mode with 8 outputs mapping stages (`understand`, `build`, `test`, `deploy`, `review`, `catalog`, `check`). Connect `Section Trigger` -> `Route to Section`.

5. **Build Section 1 (Understand):**
   - Add **NoOp** `Start Understand`, **If** `Has Endpoint?`, and **HTTP Request** `Probe Endpoint` (GET `={{ $json.state.userTargetUrl }}`, timeout 10000ms, redirects disabled).
   - Add **Code** `Build Spec Prompt`, **LLM Chain** `Write Spec (LLM)`, and **Anthropic Model (Spec)** (`lmChatAnthropic`, model: `claude-sonnet-4-6`). Link model via `ai_languageModel`.
   - Add **Code** `Validate Spec`, **If** `Spec Valid?`, **If** `Retry Spec?`, and **Code** `End Understand`. Connect sequentially to process specifications.

6. **Build Section 2 (Build):**
   - Add **NoOp** `Start Build`, **If** `Needs Node Catalog?`, **Set** `Ask for Catalog`, and **Execute Workflow** `Get Node Catalog` (self-call).
   - Add **Code** `Build Workflow Prompt`, **LLM Chain** `Write Workflow (LLM)`, **Anthropic Model (Build)** (`lmChatAnthropic`), **Code** `Parse Draft JSON`, and **Code** `End Build`.

7. **Build Section 3 (Check):**
   - Add **NoOp** `Start Check`, **If** `Blueprint or Monitor?`, **If** `Draft Parsed?`, **Set** `Ask for Output Schemas`, **Execute Workflow** `Get Output Schemas`, and **Code** `No Output Schemas`.
   - Add **Code** validator nodes `Validate Blueprint Draft` and `Validate Monitor Draft`, terminating at **NoOp** `End Check`.

8. **Build Section 4 (Review):**
   - Add **NoOp** `Start Review`, **Code** `Build Review Prompt`, **LLM Chain** `Review Workflow (LLM)`, **Anthropic Model (Review)** (`lmChatAnthropic`), and **Code** `Finish Blueprint`.

9. **Build Section 5 (Test):**
   - Add **NoOp** `Start Test`, **Code** `Build Test Cases`, **Execute Workflow** `Run Tests`, and **Code** `Score Tests`.

10. **Build Section 6 (Deploy):**
    - Add **NoOp** `Start Deploy`, **n8n API** `Create Workflow` (operation: `create`), **If** `Can Activate?`, **n8n API** `Activate Workflow` (operation: `activate`), **Code** `Write Verdict`, and **Code** `Report Deploy Error`. Configure both n8n API nodes with valid `n8nApi` credentials (Base URL: instance URL + `/api/v1`).

11. **Build Support Sections (Catalog & Mock):**
    - **Section 7 (Catalog):** Add **NoOp** `Start Catalog`, **If** `Schemas or Catalog?`, **HTTP Request** nodes `Fetch Core Node Catalog` and `Fetch AI Node Catalog` (URLs pointing to `cdn.jsdelivr.net` with pinned versions `2.39.7` and `2.39.8`), **Code** `Prepare Node Catalog`, **Code** `Pick Output Schemas`, **HTTP Request** `Fetch Output Schemas`, and **Code** `Collect Output Schemas`.
    - **Section 8 (Mock):** Add **Webhook** `Mock Webhook` (path: `alchemist-mock`), **If** `Mock Has Delay?`, **Wait** `Mock Wait`, and **Respond to Webhook** `Mock Respond`.

12. **Publish Workflow:**
    - Save and **Publish** the workflow so that internal `Execute Workflow` sub-workflow calls can successfully invoke published versions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Node Catalog CDN Dependency** | Outbound internet access to `cdn.jsdelivr.net` is required to fetch pinned node type definitions and output schemas. |
| **Credential Setup Requirements** | Requires an Anthropic API Key (or built-in AI credits) and an n8n API Key configured under Settings > n8n API. |
| **Execution Timeout Constraint** | Set the workflow-level timeout in n8n settings to at least 20 minutes to prevent premature termination of the build/test loop. |