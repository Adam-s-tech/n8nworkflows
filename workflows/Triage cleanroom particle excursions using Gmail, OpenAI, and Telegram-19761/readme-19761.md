Triage cleanroom particle excursions using Gmail, OpenAI, and Telegram

https://n8nworkflows.xyz/workflows/triage-cleanroom-particle-excursions-using-gmail--openai--and-telegram-19761


# Triage cleanroom particle excursions using Gmail, OpenAI, and Telegram

### 1. Workflow Overview

The **Cleanroom Contamination Excursion Orchestrator** is an automated monitoring and triage pipeline designed for semiconductor fabs, medical device manufacturers, and cleanroom environments operating under ISO 14644-1 programs. Its primary purpose is to ingest particle-counter excursion alerts from a Gmail inbox, parse unstructured notification texts into clean telemetry data, leverage AI to evaluate event severity and recommend mitigation, and dynamically route incidents based on cleanroom zone criticality. 

The workflow eliminates manual triage overhead by automatically logging minor excursions, delegating deep-dive root-cause investigations to a dedicated sub-workflow for major or critical events, executing automated hardware interventions (tool interlocks via HTTP APIs), and dispatching targeted real-time alerts to Facilities or EHS personnel via Telegram.

The logical execution blocks include:
- **1.1 Input Reception & Parsing:** Polling the designated email inbox every minute for raw particle counter alerts, extracting critical telemetry metrics, and normalizing data attributes.
- **1.2 AI Severity Classification:** Using an OpenAI language model constrained by a strict JSON output schema to categorize the event into Minor, Major, or Critical severity tiers and determine shutdown necessity.
- **1.3 Zone Routing & Protocol Tiering:** Segmenting incoming events by cleanroom zone criticality (Support, Process, or Litho/EUV) and tagging them with appropriate investigation protocol tiers before reconverging the stream.
- **1.4 Triage & Sub-Workflow Delegation:** Branching based on classified severity to either terminal logging (for Minor events) or payload preparation and delegation to a separate contamination investigation sub-workflow (for Major/Critical events).
- **1.5 Hardware Interlock & Notification Resolution:** Reconciling sub-workflow outcomes, evaluating tool shutdown prerequisites, executing API-driven tool interlocks, dispatching role-specific Telegram notifications, and persisting final audit logs to n8n Data Tables.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parsing
**Overview:** This block continuously monitors the facility's email inbox for automated particle-counter warnings, isolates relevant alert parameters using custom regex extraction, and standardizes the metrics into a uniform data schema.

- **Watch Particle Counter Alert Inbox**
  - *Type & Role:* `n8n-nodes-base.gmailTrigger` (Trigger) — Polls the configured Gmail account every minute for emails containing the subject query `EXCURSION` from a specific sender.
  - *Configuration:* Poll interval set to every minute; filter query `from:user@example.com subject:EXCURSION`.
  - *Expressions/Variables:* None.
  - *Connections:* Output connects to `Parse Particle Count Alert Email`.
  - *Version-specific requirements:* Version 1.2; requires valid Gmail OAuth2 credentials.
  - *Edge Cases/Failures:* Authentication token expiry, API rate limits, or missing emails due to overly restrictive query filters.

- **Parse Particle Count Alert Email**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation) — Executes custom JavaScript to parse the raw email text or snippet via regular expressions, extracting the zone identifier, tool ID, measured particle count, ISO class limit, and particle size, while calculating the exceedance ratio.
  - *Configuration:* Custom JavaScript block looping through input items and performing regex extractions (`/Zone:\s*([^\n]+)/i`, `/Tool:\s*([^\n]+)/i`, etc.).
  - *Expressions/Variables:* `$input.all()`, item JSON text/snippet properties.
  - *Connections:* Input from `Watch Particle Counter Alert Inbox`; output connects to `Normalize Excursion Record`.
  - *Version-specific requirements:* Version 2.
  - *Edge Cases/Failures:* Malformed email templates from particle counter vendors causing regex parsing failures or defaulting to `Unknown` / `0`.

- **Normalize Excursion Record**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) — Restructures and passes downstream only the essential parsed attributes needed for subsequent AI analysis.
  - *Configuration:* Maps `zone_id`, `tool_id`, `exceedance_ratio`, and `particle_size` explicitly.
  - *Expressions/Variables:* `={{$json.zone_id}}`, `={{$json.tool_id}}`, `={{$json.exceedance_ratio}}`, `={{$json.particle_size}}`.
  - *Connections:* Input from `Parse Particle Count Alert Email`; output connects to `Classify Excursion Severity`.
  - *Version-specific requirements:* Version 3.4.
  - *Edge Cases/Failures:* Expression evaluation errors if prior parsing nodes yield empty data sets.

---

#### 2.2 AI Severity Classification
**Overview:** This block feeds normalized excursion metrics into an OpenAI language model constrained by a structured schema to evaluate risk levels, provide rationales, and generate shutdown recommendations.

- **Severity Classification Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider) — Supplies the underlying Large Language Model configuration for the severity chain.
  - *Configuration:* Model set to `gpt-4.1-mini`; temperature configured to `0.1` for deterministic outputs. Error handling set to continue on regular output with up to 3 retries (2-second backoff).
  - *Expressions/Variables:* None.
  - *Connections:* Linked via LangChain AI language model connector to `Classify Excursion Severity`.
  - *Version-specific requirements:* Version 1.2; requires valid OpenAI API credentials.
  - *Edge Cases/Failures:* OpenAI API timeouts, rate-limiting, or upstream API service outages.

- **Severity Schema**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser) — Enforces strict JSON formatting on the LLM's response to ensure downstream nodes receive predictable data structures.
  - *Configuration:* Enforces a JSON schema containing `severity` (Minor, Major, Critical), `rationale` (string), and `requires_tool_shutdown` (boolean).
  - *Expressions/Variables:* None.
  - *Connections:* Linked to `Classify Excursion Severity`.
  - *Version-specific requirements:* Version 1.3.
  - *Edge Cases/Failures:* LLM output drifting outside defined enums or failing JSON validation constraints.

- **Classify Excursion Severity**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Chain Execution) — Evaluates the formatted excursion prompt against the OpenAI model using the defined schema.
  - *Configuration:* Prompt dynamically references zone, tool, particle size, and exceedance ratio. Retries on failure enabled (3 attempts, 2000ms wait).
  - *Expressions/Variables:* `=Cleanroom particle-count excursion. Zone: {{$json.zone_id}}. Tool: {{$json.tool_id}}. Particle size: {{$json.particle_size}}. Exceedance ratio (measured/limit): {{$json.exceedance_ratio}}x.`
  - *Connections:* Input from `Normalize Excursion Record`; output connects to `Route By Cleanroom Zone Criticality`.
  - *Version-specific requirements:* Version 1.5.
  - *Edge Cases/Failures:* Context window issues or schema validation errors.

---

#### 2.3 Zone Routing & Protocol Tiering
**Overview:** This block inspects the cleanroom zone identifier, branches execution based on area sensitivity, assigns appropriate protocol tiers, and reconverges the data streams.

- **Route By Cleanroom Zone Criticality**
  - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Evaluates the `zone_id` using regex rules to split events into appropriate cleanroom sensitivity tiers.
  - *Configuration:* 
    - Rule 0 (Support Area): Regex match `Gowning|Support`.
    - Rule 1 (Process Bay): Regex match `Wet-Etch|Process`.
    - Rule 2 (Litho/EUV Bay): Regex match `Litho|EUV`.
    - Fallback output routes to Rule 1.
  - *Expressions/Variables:* `={{$json.zone_id}}`.
  - *Connections:* Input from `Classify Excursion Severity`; outputs branch to the three respective `Tag Protocol Tier` nodes.
  - *Version-specific requirements:* Version 3.2.
  - *Edge Cases/Failures:* Unmatched nomenclature leading to incorrect fallback routing.

- **Tag Protocol Tier Support Area**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) — Appends the lowest protocol tier tag to support area excursions.
  - *Configuration:* Assigns string value `Tier 1 - Support` to `protocol_tier`.
  - *Expressions/Variables:* None.
  - *Connections:* Input from switch output 0; output connects to `Combine Zone Tagged Records`.
  - *Version-specific requirements:* Version 3.4.

- **Tag Protocol Tier Process Bay**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) — Appends the mid-level protocol tier tag to process bay excursions.
  - *Configuration:* Assigns string value `Tier 2 - Process` to `protocol_tier`.
  - *Expressions/Variables:* None.
  - *Connections:* Input from switch output 1; output connects to `Combine Zone Tagged Records`.
  - *Version-specific requirements:* Version 3.4.

- **Tag Protocol Tier Litho EUV Bay**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) — Appends the highest investigation protocol tier tag to advanced lithography/EUV excursions.
  - *Configuration:* Assigns string value `Tier 3 - Litho/EUV` to `protocol_tier`.
  - *Expressions/Variables:* None.
  - *Connections:* Input from switch output 2; output connects to `Combine Zone Tagged Records`.
  - *Version-specific requirements:* Version 3.4.

- **Combine Zone Tagged Records**
  - *Type & Role:* `n8n-nodes-base.merge` (Flow Control) — Reconverges the three segmented zone-tier paths back into a single unified data stream.
  - *Configuration:* Mode set to combine all inputs.
  - *Connections:* Inputs from the three protocol tagging nodes; output connects to `Route By Excursion Severity`.
  - *Version-specific requirements:* Version 3.2.

---

#### 2.4 Triage & Sub-Workflow Delegation
**Overview:** This block splits the workflow based on classified severity, logging minor events directly to audit tables while packaging and dispatching major or critical events to an external investigation sub-workflow.

- **Route By Excursion Severity**
  - *Type & Role:* `n8n-nodes-base.switch` (Flow Control) — Inspects the AI-determined severity to separate minor incidents from those requiring investigation.
  - *Configuration:*
    - Rule 0: `severity` equals `Minor`.
    - Rule 1: `severity` does not equal `Minor`.
  - *Expressions/Variables:* `={{$json.severity}}`.
  - *Connections:* Input from `Combine Zone Tagged Records`; output 0 connects to `Log Minor Excursion`, output 1 connects to `Prepare Sub-Workflow Payload`.
  - *Version-specific requirements:* Version 3.2.

- **Log Minor Excursion**
  - *Type & Role:* `n8n-nodes-base.dataTable` (Storage) — Persists minor excursions directly into an n8n Data Table without triggering operational blocks.
  - *Configuration:* Defines mapping for `tool_id`, `zone_id`, `severity`, `resolution` (`Logged - no investigation required`), and `protocol_tier`. Error handling set to continue regular output with 3 retries.
  - *Expressions/Variables:* Node field mappings pulling respective `$json` attributes.
  - *Connections:* Input from `Route By Excursion Severity` (Minor path).
  - *Version-specific requirements:* Version 1.

- **Prepare Sub-Workflow Payload**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation) — Encapsulates the entire excursion record inside an `investigation_request` object wrapper.
  - *Configuration:* Assigns object value containing full input context.
  - *Expressions/Variables:* `={{$json}}`.
  - *Connections:* Input from `Route By Excursion Severity` (Major/Critical path); output connects to `Run Contamination Investigation Sub-Workflow`.
  - *Version-specific requirements:* Version 3.4.

- **Run Contamination Investigation Sub-Workflow**
  - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-Workflow Execution) — Invokes the companion sub-workflow dedicated to contamination and CAPA investigations.
  - *Configuration:* Configured to wait for sub-workflow completion (`waitForSubWorkflow: true`). Workflow ID set via placeholder requiring manual replacement. Error handling configured with retries.
  - *Connections:* Input from `Prepare Sub-Workflow Payload`; output connects to `Merge Investigation Outcome`.
  - *Version-specific requirements:* Version 1.2.
  - *Sub-Workflow Reference:* Invokes companion sub-workflow file (`contamination-investigation-subworkflow.json`). Expects input payload under `investigation_request`.

---

#### 2.5 Hardware Interlock & Notification Resolution
**Overview:** This block processes the investigation results, checks shutdown prerequisites, executes hardware interlocks via HTTP when necessary, dispatches role-based Telegram alerts, and writes finalized audit logs.

- **Merge Investigation Outcome**
  - *Type & Role:* `n8n-nodes-base.code` (Data Transformation) — Reconciles the sub-workflow's CAPA response to establish explicit boolean shutdown requirements and CAPA identifiers.
  - *Configuration:* Custom JavaScript unpacking `capa_result` objects.
  - *Expressions/Variables:* `$input.all()`, `d.capa_result`.
  - *Connections:* Input from sub-workflow execution; output connects to `Check Tool Shutdown Required`.
  - *Version-specific requirements:* Version 2.

- **Check Tool Shutdown Required**
  - *Type & Role:* `n8n-nodes-base.if` (Flow Control) — Evaluates whether the resolved event mandates an immediate physical tool interlock.
  - *Configuration:* Loose type validation checking if `tool_shutdown_required` evaluates to `true`.
  - *Expressions/Variables:* `={{$json.tool_shutdown_required}}`.
  - *Connections:* Input from `Merge Investigation Outcome`; True branch connects to `Trigger Tool Interlock API`, False branch connects to `Notify EHS Excursion Resolved`.
  - *Version-specific requirements:* Version 2.2.

- **Trigger Tool Interlock API**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (Integration) — Issues an HTTP POST request to the internal fab tool interlock controller to halt operations.
  - *Configuration:* Method: POST; URL: internal interlock endpoint; Body formatting: JSON containing `tool_id`, action `shutdown`, and `capa_id`.
  - *Expressions/Variables:* `={{ { tool_id: $json.tool_id, action: 'shutdown', capa_id: $json.capa_id } }}`.
  - *Connections:* Input from `Check Tool Shutdown Required` (True path); output connects to `Alert Facilities Tool Shutdown Required`.
  - *Version-specific requirements:* Version 4.2; requires HTTP Header Authentication.
  - *Edge Cases/Failures:* API timeouts, network partitions, or authentication rejects from the SCADA/MES interlock gateway.

- **Alert Facilities Tool Shutdown Required**
  - *Type & Role:* `n8n-nodes-base.telegram` (Notification) — Sends an urgent alert to the Facilities on-call Telegram channel regarding the hardware shutdown.
  - *Configuration:* Message template noting tool interlock status, zone, and CAPA ID. Chat ID configured via placeholder.
  - *Expressions/Variables:* `=TOOL SHUTDOWN: {{$json.tool_id}} in {{$json.zone_id}} interlocked due to contamination excursion (CAPA {{$json.capa_id}}).`
  - *Connections:* Input from `Trigger Tool Interlock API`; output connects to `Combine Resolution Outcomes`.
  - *Version-specific requirements:* Version 1.2; requires Telegram Bot credentials.

- **Notify EHS Excursion Resolved**
  - *Type & Role:* `n8n-nodes-base.telegram` (Notification) — Dispatches an informational resolution notice to the EHS Telegram channel when no shutdown is required.
  - *Configuration:* Message template detailing investigation completion and logged CAPA ID. Chat ID configured via placeholder.
  - *Expressions/Variables:* `=Excursion in {{$json.zone_id}} ({{$json.tool_id}}) investigated, CAPA {{$json.capa_id}} logged, no shutdown required.`
  - *Connections:* Input from `Check Tool Shutdown Required` (False path); output connects to `Combine Resolution Outcomes`.
  - *Version-specific requirements:* Version 1.2; requires Telegram Bot credentials.

- **Combine Resolution Outcomes**
  - *Type & Role:* `n8n-nodes-base.merge` (Flow Control) — Reconverges the hardware shutdown path and the EHS notification path into a single execution stream.
  - *Configuration:* Mode set to combine all inputs.
  - *Connections:* Inputs from `Alert Facilities Tool Shutdown Required` and `Notify EHS Excursion Resolved`; output connects to `Log Excursion Resolution`.
  - *Version-specific requirements:* Version 3.2.

- **Log Excursion Resolution**
  - *Type & Role:* `n8n-nodes-base.dataTable` (Storage) — Persists the fully investigated and resolved excursion data, including CAPA reference IDs, into the audit log table.
  - *Configuration:* Defines mapping for `capa_id`, `tool_id`, `zone_id`, `resolution` (`Investigated`), and `tool_shutdown_required`. Error handling set to continue regular output with retries.
  - *Expressions/Variables:* Node field mappings pulling respective JSON properties.
  - *Connections:* Input from `Combine Resolution Outcomes`.
  - *Version-specific requirements:* Version 1.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Watch Particle Counter Alert Inbox | `gmailTrigger` | Polls inbox for excursion alert emails | None | Parse Particle Count Alert Email | **Watch Particle Counter Alert Inbox**<br>Polls the facilities inbox for automated excursion-alert emails from the ISO 14644-1 particle counting system.<br>Filter: from particle-monitor, subject EXCURSION |
| Parse Particle Count Alert Email | `code` | Extracts telemetry via regex and calculates exceedance ratio | Watch Particle Counter Alert Inbox | Normalize Excursion Record | **Parse Particle Count Alert Email**<br>Real logic: regex-parses the raw alert email body into structured zone/tool/particle-count fields and computes exceedance ratio.<br>exceedance_ratio = measured_count / iso_class_limit |
| Normalize Excursion Record | `set` | Restructures parsed fields for AI ingestion | Parse Particle Count Alert Email | Classify Excursion Severity | **Normalize Excursion Record**<br>Shapes the parsed excursion fields for the severity-classification chain.<br>Fields: zone_id, tool_id, exceedance_ratio, particle_size |
| Severity Classification Model | `lmChatOpenAi` | Provides gpt-4.1-mini chat model context | None | Classify Excursion Severity | **Severity Classification Model**<br>Chat model backing the excursion-severity Basic LLM Chain.<br>Model: gpt-4.1-mini, temp 0.1 |
| Severity Schema | `outputParserStructured` | Enforces strict JSON output schema | None | Classify Excursion Severity | **Severity Schema**<br>Forces the severity classification into a strict JSON schema for the downstream Switch.<br>Schema: severity, rationale, requires_tool_shutdown |
| Classify Excursion Severity | `chainLlm` | Classifies severity and shutdown recommendation via LLM | Normalize Excursion Record, Severity Classification Model, Severity Schema | Route By Cleanroom Zone Criticality | **Classify Excursion Severity**<br>AI step: classifies excursion severity and shutdown recommendation via a Basic LLM Chain with structured output.<br>Outputs: severity, rationale, requires_tool_shutdown |
| Route By Cleanroom Zone Criticality | `switch` | Branches workflow based on cleanroom zone regex | Classify Excursion Severity | Tag Protocol Tier Support Area, Tag Protocol Tier Process Bay, Tag Protocol Tier Litho EUV Bay | **Route By Cleanroom Zone Criticality**<br>Branch point 1 (Double-Switch, part 1 of 2): routes by cleanroom zone criticality tier.<br>Outputs: Support Area / Process Bay / Litho-EUV Bay |
| Tag Protocol Tier Support Area | `set` | Tags support area excursions with Tier 1 protocol | Route By Cleanroom Zone Criticality | Combine Zone Tagged Records | **Tag Protocol Tier Support Area**<br>Tags support/gowning-area excursions with the lowest investigation protocol tier.<br>protocol_tier = 'Tier 1 - Support' |
| Tag Protocol Tier Process Bay | `set` | Tags process bay excursions with Tier 2 protocol | Route By Cleanroom Zone Criticality | Combine Zone Tagged Records | **Tag Protocol Tier Process Bay**<br>Tags wet-etch/process-bay excursions with the mid investigation protocol tier.<br>protocol_tier = 'Tier 2 - Process' |
| Tag Protocol Tier Litho EUV Bay | `set` | Tags EUV/litho excursions with Tier 3 protocol | Route By Cleanroom Zone Criticality | Combine Zone Tagged Records | **Tag Protocol Tier Litho EUV Bay**<br>Tags lithography/EUV-bay excursions with the highest investigation protocol tier.<br>protocol_tier = 'Tier 3 - Litho/EUV' |
| Combine Zone Tagged Records | `merge` | Reconverges zone-tiered branches into a single stream | Tag Protocol Tier Support Area, Tag Protocol Tier Process Bay, Tag Protocol Tier Litho EUV Bay | Route By Excursion Severity | **Combine Zone Tagged Records**<br>Reconverges the three zone-tier branches so severity routing runs on a single stream.<br>3 inputs merged |
| Route By Excursion Severity | `switch` | Separates minor incidents from major/critical events | Combine Zone Tagged Records | Log Minor Excursion, Prepare Sub-Workflow Payload | **Route By Excursion Severity**<br>Branch point 2 (Double-Switch, part 2 of 2): Minor excursions log only; Major/Critical go to investigation.<br>Outputs: Minor / Major Or Critical |
| Log Minor Excursion | `dataTable` | Persists minor excursions directly to audit table | Route By Excursion Severity | None | **Log Minor Excursion**<br>Terminal path: Minor excursions are logged directly to the n8n Data Table without triggering investigation.<br>Table: cleanroom_excursion_log |
| Prepare Sub-Workflow Payload | `set` | Wraps excursion record for sub-workflow ingestion | Route By Excursion Severity | Run Contamination Investigation Sub-Workflow | **Prepare Sub-Workflow Payload**<br>Packages the excursion record as the payload handed to the investigation sub-workflow.<br>Payload: full excursion record |
| Run Contamination Investigation Sub-Workflow | `executeWorkflow` | Invokes companion contamination investigation sub-workflow | Prepare Sub-Workflow Payload | Merge Investigation Outcome | **Run Contamination Investigation Sub-Workflow**<br>Orchestrator step: hands Major/Critical excursions to the companion Contamination Investigation sub-workflow.<br>Calls sub-workflow ID (see companion JSON)<br>Workflow settings.errorWorkflow routes any sub-workflow failure to your error handler. |
| Merge Investigation Outcome | `code` | Reconciles sub-workflow CAPA response into shutdown flags | Run Contamination Investigation Sub-Workflow | Check Tool Shutdown Required | **Merge Investigation Outcome**<br>Real logic: reconciles the sub-workflow's CAPA response into a single tool-shutdown decision field.<br>tool_shutdown_required = capa.requires_tool_shutdown |
| Check Tool Shutdown Required | `if` | Evaluates whether physical tool interlock is needed | Merge Investigation Outcome | Trigger Tool Interlock API, Notify EHS Excursion Resolved | **Check Tool Shutdown Required**<br>Branch point 3: decides whether the affected tool must be interlocked/shut down immediately.<br>True = shutdown required |
| Trigger Tool Interlock API | `httpRequest` | Executes HTTP POST to halt manufacturing tool | Check Tool Shutdown Required | Alert Facilities Tool Shutdown Required | **Trigger Tool Interlock API**<br>Calls the tool interlock controller to physically halt the affected tool pending investigation.<br>POST /api/v1/interlock |
| Alert Facilities Tool Shutdown Required | `telegram` | Sends urgent shutdown notification to Facilities | Trigger Tool Interlock API | Combine Resolution Outcomes | **Alert Facilities Tool Shutdown Required**<br>Notifies the facilities on-call channel that a tool has been interlocked due to contamination.<br>Channel: Facilities on-call Telegram |
| Notify EHS Excursion Resolved | `telegram` | Sends resolution notice to EHS channel | Check Tool Shutdown Required | Combine Resolution Outcomes | **Notify EHS Excursion Resolved**<br>Notifies EHS that the investigated excursion did not require a tool shutdown.<br>Channel: EHS Telegram |
| Combine Resolution Outcomes | `merge` | Reconverges shutdown and non-shutdown pathways | Alert Facilities Tool Shutdown Required, Notify EHS Excursion Resolved | Log Excursion Resolution | **Combine Resolution Outcomes**<br>Reconverges the shutdown and no-shutdown resolution paths into one final log write.<br>2 inputs merged |
| Log Excursion Resolution | `dataTable` | Persists resolved excursion audit data with CAPA ID | Combine Resolution Outcomes | None | **Log Excursion Resolution**<br>Final storage: logs the fully investigated and resolved excursion, including CAPA ID, to the audit table.<br>Table: cleanroom_excursion_log |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in an n8n environment:

1. **Create the Base Canvas:** Initialize a new workflow in n8n and set execution order to `v1`. Configure the workflow-level error handler under Settings to point to your organization's designated error workflow.
2. **Add Gmail Trigger:** 
   - Create a node of type `n8n-nodes-base.gmailTrigger`.
   - Name it `Watch Particle Counter Alert Inbox`.
   - Configure parameters: set Poll Times to every minute (`everyMinute`). Set filter query to `from:user@example.com subject:EXCURSION`.
   - Connect Gmail OAuth2 credentials (`Fab Facilities - Gmail (Particle Counter Alerts Inbox)`).
3. **Add Parsing Code Node:**
   - Create a Code node (`n8n-nodes-base.code`) named `Parse Particle Count Alert Email`.
   - Insert JavaScript to extract `zone_id`, `tool_id`, `measured_count`, `iso_class_limit`, `particle_size`, and calculate `exceedance_ratio` (as defined in Section 2).
   - Connect input from `Watch Particle Counter Alert Inbox`.
4. **Add Normalization Node:**
   - Create a Set node (`n8n-nodes-base.set`) named `Normalize Excursion Record`.
   - Add string/number assignments for `zone_id`, `tool_id`, `exceedance_ratio`, and `particle_size`.
   - Connect input from `Parse Particle Count Alert Email`.
5. **Add LangChain AI Components:**
   - Create an OpenAI Chat Model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) named `Severity Classification Model`. Configure model to `gpt-4.1-mini`, temperature to `0.1`, and connect OpenAI API credentials (`Swapnil AI Labs - OpenAI API`).
   - Create a Structured Output Parser node (`@n8n/n8n-nodes-langchain.outputParserStructured`) named `Severity Schema`. Provide the JSON schema example for `severity`, `rationale`, and `requires_tool_shutdown`.
   - Create a Basic LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`) named `Classify Excursion Severity`. Link model and schema connections. Set prompt text referencing zone, tool, particle size, and exceedance ratio. Connect input from `Normalize Excursion Record`.
6. **Add Zone Criticality Switch:**
   - Create a Switch node (`n8n-nodes-base.switch`) named `Route By Cleanroom Zone Criticality`.
   - Configure three regex output rules matching `Gowning|Support`, `Wet-Etch|Process`, and `Litho|EUV`. Connect input from `Classify Excursion Severity`.
7. **Add Protocol Tier Set Nodes:**
   - Create three Set nodes (`n8n-nodes-base.set`) named:
     - `Tag Protocol Tier Support Area` (assign `protocol_tier` = `Tier 1 - Support`).
     - `Tag Protocol Tier Process Bay` (assign `protocol_tier` = `Tier 2 - Process`).
     - `Tag Protocol Tier Litho EUV Bay` (assign `protocol_tier` = `Tier 3 - Litho/EUV`).
   - Connect each respectively to the corresponding output of `Route By Cleanroom Zone Criticality`.
8. **Merge Zone-Tagged Records:**
   - Create a Merge node (`n8n-nodes-base.merge`) named `Combine Zone Tagged Records` configured to combine all inputs. Connect the three protocol tier nodes as inputs.
9. **Add Severity Routing Switch:**
   - Create a Switch node (`n8n-nodes-base.switch`) named `Route By Excursion Severity`.
   - Configure Rule 0 where `severity` equals `Minor`, and Rule 1 where `severity` does not equal `Minor`. Connect input from `Combine Zone Tagged Records`.
10. **Build Minor Excursion Terminal Path:**
    - Create a Data Table node (`n8n-nodes-base.dataTable`) named `Log Minor Excursion`.
    - Map fields for `tool_id`, `zone_id`, `severity`, `resolution` (`Logged - no investigation required`), and `protocol_tier`. Connect input from switch output 0.
11. **Build Major/Critical Investigation Path:**
    - Create a Set node (`n8n-nodes-base.set`) named `Prepare Sub-Workflow Payload` wrapping the input JSON into an `investigation_request` object. Connect input from switch output 1.
    - Create an Execute Workflow node (`n8n-nodes-base.executeWorkflow`) named `Run Contamination Investigation Sub-Workflow`. Set `waitForSubWorkflow` to true and populate the `workflowId` with your imported sub-workflow identifier. Connect input from `Prepare Sub-Workflow Payload`.
    - Create a Code node (`n8n-nodes-base.code`) named `Merge Investigation Outcome` to unpack CAPA results and establish boolean shutdown requirements. Connect input from sub-workflow execution.
12. **Add Shutdown Evaluation IF Node:**
    - Create an IF node (`n8n-nodes-base.if`) named `Check Tool Shutdown Required`. Set condition to evaluate if `tool_shutdown_required` equals `true`. Connect input from `Merge Investigation Outcome`.
13. **Build Hardware Interlock & Facilities Alert Path (True Branch):**
    - Create an HTTP Request node (`n8n-nodes-base.httpRequest`) named `Trigger Tool Interlock API`. Configure POST method to your endpoint URL (`https://REPLACE-tool-interlock-controller.example.internal/api/v1/interlock`) with JSON body payload (`tool_id`, action `shutdown`, `capa_id`). Configure HTTP Header Authentication credentials (`Tool Interlock Controller - API Key`).
    - Create a Telegram node (`n8n-nodes-base.telegram`) named `Alert Facilities Tool Shutdown Required`. Configure Telegram Bot credentials (`Fab EHS - Telegram Bot`), target chat ID (`REPLACE_WITH_FACILITIES_CHAT_ID`), and message text referencing tool shutdown and CAPA ID. Connect input from HTTP Request.
14. **Build EHS Notification Path (False Branch):**
    - Create a Telegram node (`n8n-nodes-base.telegram`) named `Notify EHS Excursion Resolved`. Configure Telegram Bot credentials (`Fab EHS - Telegram Bot`), target chat ID (`REPLACE_WITH_EHS_CHAT_ID`), and message text noting investigation completion without shutdown. Connect input from IF false output.
15. **Finalize Audit Logging:**
    - Create a Merge node (`n8n-nodes-base.merge`) named `Combine Resolution Outcomes` combining inputs from both Telegram notification nodes.
    - Create a Data Table node (`n8n-nodes-base.dataTable`) named `Log Excursion Resolution`. Map fields for `capa_id`, `tool_id`, `zone_id`, `resolution` (`Investigated`), and `tool_shutdown_required`. Connect input from `Combine Resolution Outcomes`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Credits & Author Contact | Swapnil AI Labs — [swapnil.mandloi7@gmail.com](mailto:swapnil.mandloi7@gmail.com) — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Companion Sub-Workflow Requirement | Requires prior import of `contamination-investigation-subworkflow.json` and subsequent wiring of its assigned workflow ID into the execution node. |