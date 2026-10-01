Expose production tools to MCP clients with Slack approval gates

https://n8nworkflows.xyz/workflows/expose-production-tools-to-mcp-clients-with-slack-approval-gates-19742


# Expose production tools to MCP clients with Slack approval gates

### 1. Workflow Overview

This workflow exposes n8n as a Model Context Protocol (MCP) server, allowing connected MCP clients (such as Claude Desktop or Claude Code) to discover and execute a curated list of production tools. It provides a governance layer between autonomous AI agents and sensitive backend systems by enforcing allow-list validation, pre-execution audit logging, conditional human-approval gates for destructive actions, and narrow-scoped API executions.

The workflow logic is divided into four functional blocks:
- **1.1 Input Reception & Tool Provisioning:** Initializes the MCP Server Trigger, binds an optional helper Code Tool, and handles external webhook requests to accept tool-call payloads from MCP clients.
- **1.2 Validation & Pre-Execution Auditing:** Evaluates requested tool names against a rigid schema allow-list, validates required payload parameters, rejects malformed calls, and records valid execution attempts to an audit trail before system interaction.
- **1.3 Category Routing & Human Approval:** Distinguishes between read-only actions (executed immediately) and destructive operations (routed to Slack for human authorization, paused via a Wait node, and verified via conditional logic).
- **1.4 Narrow-Scoped Execution & Result Logging:** Dispatches calls to designated production endpoints (deployment, database querying, knowledge base searching, or refunds), merges outputs into a uniform schema, completes the audit log, and responds to the MCP client.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Tool Provisioning
- **Overview:** Listens for incoming MCP tool-call connections and HTTP payloads from clients, setting up the interaction interface for autonomous agents.
- **Nodes Involved:** `MCP Server Trigger`, `Webhook`, `Code Tool`
- **Node Details:**
  - **MCP Server Trigger** (`@n8n/n8n-nodes-langchain.mcpTrigger`)
    - *Technical Role:* Acts as the primary MCP server endpoint listening for tool execution requests.
    - *Configuration:* Path configured to `production-tools-mcp`.
    - *Input/Output:* No direct internal input; outputs tool execution requests to validation logic; connects downstream via `ai_tool` to the Code Tool.
    - *Edge Cases:* Client connection timeouts, dropped persistent SSE/transport layers.
  - **Webhook** (`n8n-nodes-base.webhook`)
    - *Technical Role:* Secondary entry point capturing HTTP webhook test/production events.
    - *Configuration:* Path configured using a unique hash identifier (`fe06438e-e71a-4bb5-94fd-df869cdad07b`).
    - *Input/Output:* Receives external HTTP calls; outputs payload to `JS - Validate Tool Call`.
    - *Edge Cases:* Invalid JSON payloads, missing authentication headers.
  - **Code Tool** (`@n8n/n8n-nodes-langchain.toolCode`)
    - *Technical Role:* Exposes custom script-defined tool declarations to the MCP Server instance.
    - *Configuration:* Default tool code bindings.
    - *Input/Output:* Connects exclusively to the `MCP Server Trigger` via an `ai_tool` connection.
    - *Edge Cases:* Malformed tool definitions causing schema registration failures.

#### 2.2 Validation & Pre-Execution Auditing
- **Overview:** Inspects incoming tool payloads against structural rules and security schemas, filters invalid parameters, and logs intent before executing side effects.
- **Nodes Involved:** `JS - Validate Tool Call`, `IF - Tool Call Valid?`, `Respond - Invalid Tool Call`, `JS - Log Call (Pre-Execution)`
- **Node Details:**
  - **JS - Validate Tool Call** (`n8n-nodes-base.code`)
    - *Technical Role:* JavaScript execution node checking tool names against an allow-list (`deploy_service`, `query_production_database`, `search_knowledge_base`, `issue_refund`) and verifying required parameters.
    - *Configuration:* Custom JavaScript snippet parsing `$input.first().json` and returning validation status flags.
    - *Input/Output:* Inputs from Webhook/Trigger; outputs data object containing `isValid`, `validationErrors`, and `isDestructive`.
    - *Edge Cases:* Unhandled property access on undefined objects if payload schemas deviate entirely from expectations.
  - **IF - Tool Call Valid?** (`n8n-nodes-base.if`)
    - *Technical Role:* Conditional router determining whether to proceed with execution or abort.
    - *Configuration:* Evaluates condition `{{ $json.isValid }}` equals `true`.
    - *Input/Output:* Input from `JS - Validate Tool Call`; outputs true branch to pre-execution logging, false branch to invalid response handler.
    - *Edge Cases:* Type mismatch if boolean validation flag is returned as a string.
  - **Respond - Invalid Tool Call** (`n8n-nodes-base.noOp`)
    - *Technical Role:* Placeholder terminal node for rejecting ill-formed tool calls without side effects.
    - *Configuration:* Standard No-Op configuration.
    - *Input/Output:* Input from `IF - Tool Call Valid?` (false branch); no outputs.
    - *Edge Cases:* Returning unstructured error payloads back to clients if not properly mapped.
  - **JS - Log Call (Pre-Execution)** (`n8n-nodes-base.code`)
    - *Technical Role:* Appends audit metadata (`auditStatus: 'call_logged_pending_execution'`) to valid payloads.
    - *Configuration:* JavaScript code appending ISO timestamp and audit status properties.
    - *Input/Output:* Input from `IF - Tool Call Valid?` (true branch); outputs to `Switch - Route By Tool Category`.
    - *Edge Cases:* Serialization errors with deep nested parameter structures.

#### 2.3 Category Routing & Human Approval
- **Overview:** Evaluates whether an approved tool action is read-only or destructive, routing destructive tasks through a Slack review cycle before execution.
- **Nodes Involved:** `Switch - Route By Tool Category`, `Slack - Request Approval (Destructive Tool)`, `Wait - For Approval Response`, `IF - Approved?`, `Respond - Approval Denied`
- **Node Details:**
  - **Switch - Route By Tool Category** (`n8n-nodes-base.switch`)
    - *Technical Role:* Routes workflow execution based on the `isDestructive` boolean flag.
    - *Configuration:* Rule 1 checks if `isDestructive` is `false` (routes to execution switch); Rule 2 checks if `isDestructive` is `true` (routes to Slack approval).
    - *Input/Output:* Input from pre-execution audit log; outputs to execution switch or Slack node.
    - *Edge Cases:* Unhandled boolean evaluation gaps.
  - **Slack - Request Approval (Destructive Tool)** (`n8n-nodes-base.slack`)
    - *Technical Role:* Sends interactive or notification alerts to a designated channel to solicit human sign-off.
    - *Configuration:* Posts formatted markdown text containing tool name and parameters to channel `production-tools-approvals`. Requires Slack credentials.
    - *Input/Output:* Input from Switch; outputs to `Wait - For Approval Response`.
    - *Edge Cases:* Slack API rate limits, invalid channel IDs, webhook callback failures.
  - **Wait - For Approval Response** (`n8n-nodes-base.wait`)
    - *Technical Role:* Pauses workflow execution until an external webhook resumes it with approval status.
    - *Configuration:* Webhook-resume wait configuration using path identifier (`2dba9873-3a7e-4285-9b5b-95a9c637d1e0`).
    - *Input/Output:* Input from Slack node; outputs to `IF - Approved?`.
    - *Edge Cases:* Workflow timeouts if approvers fail to respond within retention limits.
  - **IF - Approved?** (`n8n-nodes-base.if`)
    - *Technical Role:* Verifies the approval outcome provided upon resuming the wait node.
    - *Configuration:* Condition checks if `{{ $json.approved === true || $json.approved === 'true' }}`.
    - *Input/Output:* Input from Wait node; outputs true branch to execution routing, false branch to denial response.
    - *Edge Cases:* Payload formatting differences returning string values instead of booleans.
  - **Respond - Approval Denied** (`n8n-nodes-base.noOp`)
    - *Technical Role:* Terminal node handling rejected destructive tool requests.
    - *Configuration:* Standard No-Op setup.
    - *Input/Output:* Input from `IF - Approved?` (false branch); no outputs.
    - *Edge Cases:* Lack of explicit notification transmission back to the originating client.

#### 2.4 Narrow-Scoped Execution & Result Logging
- **Overview:** Dispatches validated and approved parameters to specific downstream HTTP APIs, aggregates performance metrics, updates audit logs, and returns final tool outcomes.
- **Nodes Involved:** `Switch - Route To Execution Tool`, `HTTP - Execute: Deploy Service`, `HTTP - Execute: Query Production Database`, `HTTP - Execute: Search Knowledge Base`, `HTTP - Execute: Issue Refund`, `JS - Merge Execution Result`, `JS - Log Call (Post-Execution)`, `Respond - Tool Result (Success)`
- **Node Details:**
  - **Switch - Route To Execution Tool** (`n8n-nodes-base.switch`)
    - *Technical Role:* Dispatches execution flow to the specific HTTP request node matching the `toolName`.
    - *Configuration:* Four routing rules matching exact strings: `deploy_service`, `query_production_database`, `search_knowledge_base`, `issue_refund`.
    - *Input/Output:* Inputs from direct read-only route or approved destructive route; outputs to corresponding HTTP nodes.
    - *Edge Cases:* Unmapped tool names reaching this switch.
  - **HTTP - Execute: Deploy Service** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Triggers a remote deployment pipeline via REST API.
    - *Configuration:* POST method to `https://api.deploy-platform.example.com/v1/services/{{ $json.params.serviceName }}/deploy`. Uses HTTP Header Auth. JSON body contains environment and version parameters.
    - *Input/Output:* Input from execution switch; outputs to `JS - Merge Execution Result`.
    - *Edge Cases:* Target environment downtime, network timeouts, authentication credential expiration.
  - **HTTP - Execute: Query Production Database** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Executes parameterized data lookups against internal systems.
    - *Configuration:* POST method to `https://api.internal-data.example.com/v1/queries/{{ $json.params.queryName }}`. Uses HTTP Header Auth. JSON body contains filters.
    - *Input/Output:* Input from execution switch; outputs to `JS - Merge Execution Result`.
    - *Edge Cases:* Query syntax errors, database locks.
  - **HTTP - Execute: Search Knowledge Base** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Queries internal documentation or vector indices.
    - *Configuration:* POST method to `https://api.knowledge-base.example.com/v1/search`. Uses HTTP Header Auth. JSON body contains query text.
    - *Input/Output:* Input from execution switch; outputs to `JS - Merge Execution Result`.
    - *Edge Cases:* Rate limiting on search endpoints.
  - **HTTP - Execute: Issue Refund** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Interacts with payment provider APIs to process financial refunds.
    - *Configuration:* POST method to `https://api.payments.example.com/v1/refunds`. Uses HTTP Header Auth. JSON body contains `orderId`, `amount`, and `reason`.
    - *Input/Output:* Input from execution switch; outputs to `JS - Merge Execution Result`.
    - *Edge Cases:* Insufficient funds, invalid order identifiers, financial gateway errors.
  - **JS - Merge Execution Result** (`n8n-nodes-base.code`)
    - *Technical Role:* Normalizes success and failure responses from divergent HTTP nodes into a unified schema format.
    - *Configuration:* JavaScript parsing `$input.first().json` to extract `executionSucceeded` and response bodies.
    - *Input/Output:* Inputs from any HTTP execution node; outputs to `JS - Log Call (Post-Execution)`.
    - *Edge Cases:* Unexpected response structure missing standard error properties.
  - **JS - Log Call (Post-Execution)** (`n8n-nodes-base.code`)
    - *Technical Role:* Finalizes audit tracking by recording execution outcomes and completion timestamps.
    - *Configuration:* JavaScript appending `auditStatus` (`call_executed_successfully` or `call_execution_failed`) and `auditCompletedAt`.
    - *Input/Output:* Input from merge node; outputs to success response node.
    - *Edge Cases:* Execution crash preventing final log write.
  - **Respond - Tool Result (Success)** (`n8n-nodes-base.noOp`)
    - *Technical Role:* Terminal delivery node passing the final tool execution payload back to the client interface.
    - *Configuration:* Standard No-Op configuration.
    - *Input/Output:* Input from post-execution audit log; no subsequent internal nodes.
    - *Edge Cases:* Transport errors transmitting payloads back across the MCP bridge.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Documentation placeholder outlining full workflow scope, operational patterns, and setup instructions. | None | None | MCP Server → n8n → Production Tools... [See workflow overview for full text] |
| Sticky Note - Stage 1 | `n8n-nodes-base.stickyNote` | Documentation block describing Stage 1 components (MCP trigger, validation, pre-audit logging). | None | None | Stage 1: MCP Trigger & Call Validation... [See workflow overview for full text] |
| Sticky Note - Stage 2 | `n8n-nodes-base.stickyNote` | Documentation block detailing Stage 2 routing rules and human approval gates via Slack. | None | None | Stage 2: Routing & Human Approval Gate... [See workflow overview for full text] |
| Sticky Note - Stage 3 | `n8n-nodes-base.stickyNote` | Documentation block summarizing narrow-scoped execution nodes, response normalization, and final logging. | None | None | Stage 3: Narrow-Scoped Execution & Audit Log... [See workflow overview for full text] |
| MCP Server Trigger | `@n8n/n8n-nodes-langchain.mcpTrigger` | Exposes the workflow as an MCP server endpoint to external AI clients. | None | `JS - Validate Tool Call` | |
| Webhook | `n8n-nodes-base.webhook` | Receives alternative incoming HTTP tool invocation payloads. | None | `JS - Validate Tool Call` | |
| Code Tool | `@n8n/n8n-nodes-langchain.toolCode` | Defines custom code tools available to the MCP trigger context. | None | `MCP Server Trigger` (AI Tool link) | |
| JS - Validate Tool Call | `n8n-nodes-base.code` | Evaluates tool name against allow-list schemas and validates required parameters. | `MCP Server Trigger`, `Webhook` | `IF - Tool Call Valid?` | |
| IF - Tool Call Valid? | `n8n-nodes-base.if` | Evaluates validation status to route valid calls to audit logging or reject invalid calls. | `JS - Validate Tool Call` | `JS - Log Call (Pre-Execution)`, `Respond - Invalid Tool Call` | |
| Respond - Invalid Tool Call | `n8n-nodes-base.noOp` | Terminal point for malformed or unauthorized tool requests. | `IF - Tool Call Valid?` | None | |
| JS - Log Call (Pre-Execution) | `n8n-nodes-base.code` | Writes pending execution requests to the audit trail before actions occur. | `IF - Tool Call Valid?` | `Switch - Route By Tool Category` | |
| Switch - Route By Tool Category | `n8n-nodes-base.switch` | Directs read-only actions straight to execution and destructive actions to approval gates. | `JS - Log Call (Pre-Execution)` | `Switch - Route To Execution Tool`, `Slack - Request Approval (Destructive Tool)` | |
| Slack - Request Approval (Destructive Tool) | `n8n-nodes-base.slack` | Sends approval requests for destructive actions to a designated Slack channel. | `Switch - Route By Tool Category` | `Wait - For Approval Response` | |
| Wait - For Approval Response | `n8n-nodes-base.wait` | Pauses workflow execution until an approver responds via webhook. | `Slack - Request Approval (Destructive Tool)` | `IF - Approved?` | |
| IF - Approved? | `n8n-nodes-base.if` | Checks whether human approver authorized the destructive action. | `Wait - For Approval Response` | `Switch - Route To Execution Tool`, `Respond - Approval Denied` | |
| Respond - Approval Denied | `n8n-nodes-base.noOp` | Terminal point for rejected destructive tool calls. | `IF - Approved?` | None | |
| Switch - Route To Execution Tool | `n8n-nodes-base.switch` | Routes requests to the appropriate target execution node based on tool name. | `Switch - Route By Tool Category`, `IF - Approved?` | `HTTP - Execute: Deploy Service`, `HTTP - Execute: Query Production Database`, `HTTP - Execute: Search Knowledge Base`, `HTTP - Execute: Issue Refund` | |
| HTTP - Execute: Deploy Service | `n8n-nodes-base.httpRequest` | Calls external deployment API endpoints. | `Switch - Route To Execution Tool` | `JS - Merge Execution Result` | |
| HTTP - Execute: Query Production Database | `n8n-nodes-base.httpRequest` | Executes safe, parameterized queries against internal data systems. | `Switch - Route To Execution Tool` | `JS - Merge Execution Result` | |
| HTTP - Execute: Search Knowledge Base | `n8n-nodes-base.httpRequest` | Searches internal knowledge bases or vector retrieval systems. | `Switch - Route To Execution Tool` | `JS - Merge Execution Result` | |
| HTTP - Execute: Issue Refund | `n8n-nodes-base.httpRequest` | Calls payments API to process transaction refunds. | `Switch - Route To Execution Tool` | `JS - Merge Execution Result` | |
| JS - Merge Execution Result | `n8n-nodes-base.code` | Normalizes outputs and error objects from divergent execution paths into a uniform format. | `HTTP - Execute: Deploy Service`, `HTTP - Execute: Query Production Database`, `HTTP - Execute: Search Knowledge Base`, `HTTP - Execute: Issue Refund` | `JS - Log Call (Post-Execution)` | |
| JS - Log Call (Post-Execution) | `n8n-nodes-base.code` | Updates audit log records with execution outcomes and completion timestamps. | `JS - Merge Execution Result` | `Respond - Tool Result (Success)` | |
| Respond - Tool Result (Success) | `n8n-nodes-base.noOp` | Finalizes workflow execution and passes results back to the client interface. | `JS - Log Call (Post-Execution)` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually rebuild the workflow in an n8n environment:

1. **Create Entry Points & Tools:**
   - Add an **MCP Server Trigger** node (`@n8n/n8n-nodes-langchain.mcpTrigger`). Set path parameter to `production-tools-mcp`.
   - Add a **Webhook** node (`n8n-nodes-base.webhook`). Set path parameter to `fe06438e-e71a-4bb5-94fd-df869cdad07b`.
   - Add a **Code Tool** node (`@n8n/n8n-nodes-langchain.toolCode`) and connect its `ai_tool` output anchor to the **MCP Server Trigger**.

2. **Setup Validation Block:**
   - Add a **Code** node named `JS - Validate Tool Call`. Populate its JavaScript parameters with allow-list definitions (`deploy_service`, `query_production_database`, `search_knowledge_base`, `issue_refund`) and validation checks. Connect both `MCP Server Trigger` and `Webhook` outputs to this node.
   - Add an **IF** node named `IF - Tool Call Valid?`. Set condition `{{ $json.isValid }}` equals `true` (strict type validation). Connect `JS - Validate Tool Call` output to this node.
   - Add a **No-Op** node named `Respond - Invalid Tool Call`. Connect the `false` output branch of `IF - Tool Call Valid?` to it.
   - Add a **Code** node named `JS - Log Call (Pre-Execution)` to append audit trail fields. Connect the `true` output branch of `IF - Tool Call Valid?` to this node.

3. **Setup Routing & Approval Block:**
   - Add a **Switch** node named `Switch - Route By Tool Category`. Configure Rule 1: `isDestructive` equals `false`; Rule 2: `isDestructive` equals `true`. Connect `JS - Log Call (Pre-Execution)` to this node.
   - Add a **Slack** node named `Slack - Request Approval (Destructive Tool)`. Set resource/action to post text message to channel `production-tools-approvals`. Configure Slack credentials. Connect Rule 2 output of the switch to this node.
   - Add a **Wait** node named `For Approval Response`. Configure resumption type as Webhook. Connect `Slack - Request Approval (Destructive Tool)` to this node.
   - Add an **IF** node named `IF - Approved?`. Set condition checking `{{ $json.approved === true || $json.approved === 'true' }}`. Connect `Wait - For Approval Response` to this node.
   - Add a **No-Op** node named `Respond - Approval Denied`. Connect the `false` branch of `IF - Approved?` to it.

4. **Setup Execution & Logging Block:**
   - Add a **Switch** node named `Switch - Route To Execution Tool`. Configure 4 string equality routing rules matching `toolName` against: `deploy_service`, `query_production_database`, `search_knowledge_base`, and `issue_refund`. Connect Rule 1 output of `Switch - Route By Tool Category` and the `true` branch of `IF - Approved?` to this switch.
   - Add four **HTTP Request** nodes configured with **HTTP Header Auth** credentials pointing to respective internal base URLs:
     - `HTTP - Execute: Deploy Service`: POST request to `https://api.deploy-platform.example.com/v1/services/{{ $json.params.serviceName }}/deploy` with JSON body payload containing environment and version. Connect to output branch 1 of the execution switch.
     - `HTTP - Execute: Query Production Database`: POST request to `https://api.internal-data.example.com/v1/queries/{{ $json.params.queryName }}` with JSON body filters. Connect to output branch 2 of the execution switch.
     - `HTTP - Execute: Search Knowledge Base`: POST request to `https://api.knowledge-base.example.com/v1/search` with JSON query body. Connect to output branch 3 of the execution switch.
     - `HTTP - Execute: Issue Refund`: POST request to `https://api.payments.example.com/v1/refunds` with JSON body containing order ID, amount, and reason. Connect to output branch 4 of the execution switch.
   - Add a **Code** node named `JS - Merge Execution Result` to normalize response formats. Connect all four HTTP Request nodes to this node.
   - Add a **Code** node named `JS - Log Call (Post-Execution)` to update final audit statuses. Connect `JS - Merge Execution Result` to this node.
   - Add a **No-Op** node named `Respond - Tool Result (Success)`. Connect `JS - Log Call (Post-Execution)` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| MCP server implementation workflow exposing restricted tool access via n8n. | Workflow architecture designed to sandbox autonomous model execution with human-in-the-loop validation gates. |