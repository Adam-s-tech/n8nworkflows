Send scheduled SMS and voice outreach from Supabase via Twilio and Retell

https://n8nworkflows.xyz/workflows/send-scheduled-sms-and-voice-outreach-from-supabase-via-twilio-and-retell-19776


# Send scheduled SMS and voice outreach from Supabase via Twilio and Retell

### 1. Workflow Overview

This workflow automates outbound recruitment outreach by running every 15 minutes to process due candidate communication tasks. It evaluates candidates against a multi-step sequence, respects local quiet hours and daily send caps, dispatches messages via Twilio (SMS) or Retell (AI voice calls), handles retries with backoff and jitter, writes audit logs and status updates to Supabase, and optionally notifies a Slack channel.

The logic is organized into the following functional blocks:
- **1.1 Initialization & Configuration:** Triggers the workflow on a schedule or manually, loads global campaign and runtime settings, and verifies Supabase connectivity.
- **1.2 Queue Retrieval & Decision Engine:** Claims due outreach enrollments from a database view (or falls back to a built-in sample queue in demo mode), then runs a state machine to determine the next outreach step, compute idempotency keys, and enforce quiet hours and weekend rules in the candidate's local time zone.
- **1.3 Channel Routing & Execution:** Evaluates whether the requested touch point requires SMS or voice, checks provider configurations, and executes requests against the Twilio or Retell APIs (or falls back to preview/simulation modes).
- **1.4 Result Processing & Persistence:** Classifies send outcomes (sent, transient, permanent, or carrier opt-out), calculates backoff schedules, writes records and enrollment updates back to Supabase, and compiles runner summaries.
- **1.5 Summary & Notification:** Determines whether there are notable execution events and optionally posts a runner summary payload to a Slack incoming webhook before terminating.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Configuration
- **Overview:** This block establishes the execution entry points, centralizes client configurations, credentials, sequence rules, and provider settings into a single data structure, and branches based on whether Supabase is enabled.
- **Nodes Involved:** `Every 15 minutes`, `Run it now`, `config`, `Supabase connected?`
- **Node Details:**
  - **Every 15 minutes**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Fires the workflow automatically every 15 minutes.
    - *Configuration:* Cron/interval rule set to 15-minute intervals (Version 1.2).
    - *Input/Output:* Output connects to `config`.
    - *Edge Cases:* Server clock drift or downtime may delay execution.
  - **Run it now**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger). Allows manual workflow testing.
    - *Configuration:* Default manual trigger (Version 1).
    - *Input/Output:* Output connects to `config`.
  - **config**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Central configuration repository defining client identifiers, Supabase parameters, ATS field mappings, AI settings, matching rules, outreach ladder (`SEQUENCE`), quiet hours, Twilio/Retell settings, safety switches (`DEMO_MODE`, `TEST_RUN`), and calculated execution flags (`send_enabled`, `writeback_enabled`).
    - *Key Expressions/Variables:* Returns a JSON array containing global parameters and evaluated mode flags (`mode`, `supabase_enabled`, `send_enabled`).
    - *Input/Output:* Inputs from `Every 15 minutes` or `Run it now`; output connects to `Supabase connected?`.
    - *Edge Cases:* Blank credential fields toggle respective services into simulation/preview mode instead of crashing execution.
  - **Supabase connected?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Evaluates whether Supabase is configured (`supabase_enabled === true`).
    - *Configuration:* Loose type validation checking `{{ $('config').first().json.supabase_enabled }}` equals `true`.
    - *Input/Output:* Input from `config`; true branch connects to `[cred] Supabase - claim the due queue`, false branch connects to `STOP: using the sample due queue`.

#### 2.2 Queue Retrieval & Decision Engine
- **Overview:** This block queries the database for due candidate enrollments or supplies a mock queue in demo mode, merges the data streams, executes state-machine rules to determine messaging content and timing, and verifies if work is pending.
- **Nodes Involved:** `[cred] Supabase - claim the due queue`, `STOP: using the sample due queue`, `one due queue`, `decide the next touch`, `anything to send?`, `STOP: nothing due`
- **Node Details:**
  - **[cred] Supabase - claim the due queue**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Calls a Supabase RPC endpoint (`claim_due_attempts`) using `SELECT ... FOR UPDATE SKIP LOCKED` logic to fetch a batch of due candidate enrollments.
    - *Configuration:* POST method to `{{ $('config').first().json.supabase_url }}/rest/v1/rpc/claim_due_attempts` with JSON body containing client ID, batch size, and lease duration. Includes Supabase API key headers. Retries up to 3 times on failure.
    - *Input/Output:* Input from `Supabase connected?` (True); output connects to `one due queue`.
    - *Edge Cases:* Network timeouts or database lock contention; handled by retry logic and `onError: continueRegularOutput`.
  - **STOP: using the sample due queue**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Fallback / Data Generation). Provides sample mock due rows simulating different ladder steps and timezones when Supabase is not connected and demo mode is active.
    - *Input/Output:* Input from `Supabase connected?` (False); output connects to `one due queue`.
  - **one due queue**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Data Consolidation). Combines database-claimed rows and sample queue rows into a single data stream.
    - *Configuration:* Two inputs merged (Version 3).
    - *Input/Output:* Inputs from `[cred] Supabase - claim the due queue` and `STOP: using the sample due queue`; output connects to `decide the next touch`.
  - **decide the next touch**
    - *Type & Technical Role:* `n8n-nodes-base.code` (State Machine Logic). Evaluates candidate records against the sequence ladder, checks local time zones for quiet hours and weekend restrictions, calculates segment counts, handles test phone redirection, and computes deterministic idempotency keys (`sha`-style FNV-1a hash of enrollment ID, channel, and step).
    - *Input/Output:* Input from `one due queue`; output connects to `anything to send?`.
    - *Edge Cases:* Unknown time zones fall back to UTC with an error flag; reaching the end of the ladder marks enrollments as `exhausted`.
  - **anything to send?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Determines whether any valid items are ready for outreach (`_has_work === true`).
    - *Input/Output:* Input from `decide the next touch`; true branch connects to `sending?`, false branch connects to `STOP: nothing due`.
  - **STOP: nothing due**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation / Reporting). Summarizes why no work was processed (e.g., deferrals due to quiet hours or empty queue).
    - *Input/Output:* Input from `anything to send?` (False); output connects to `record the attempt`.

#### 2.3 Channel Routing & Execution
- **Overview:** This block checks global sending permissions, determines whether communication should occur via SMS or voice, verifies provider credentials for each channel, and executes API requests against Twilio or Retell.
- **Nodes Involved:** `sending?`, `STOP: preview - nothing sent`, `SMS or voice?`, `Twilio connected?`, `[cred] Twilio - send the SMS`, `read the Twilio result`, `STOP: Twilio not configured`, `Retell connected?`, `[cred] Retell - start the call`, `read the Retell result`, `STOP: Retell not configured`
- **Node Details:**
  - **sending?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Checks if live sending is enabled (`send_enabled`).
    - *Input/Output:* Input from `anything to send?` (True); true branch connects to `SMS or voice?`, false branch connects to `STOP: preview - nothing sent`.
  - **STOP: preview - nothing sent**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Reporting / Simulation). Compiles preview metrics, potential segment counts, unicode warnings, and deferral lists without dispatching external API calls.
    - *Input/Output:* Input from `sending?` (False); output connects to `build the runner summary`.
  - **SMS or voice?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Branches items based on their communication channel (`channel === 'sms'`).
    - *Input/Output:* Input from `sending?` (True); true branch routes to `Twilio connected?`, false branch routes to `Retell connected?`.
  - **Twilio connected?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Verifies whether Twilio credentials and sender numbers are configured (`twilio_enabled === true`).
    - *Input/Output:* Input from `SMS or voice?` (True); true branch connects to `[cred] Twilio - send the SMS`, false branch connects to `STOP: Twilio not configured`.
  - **[cred] Twilio - send the SMS**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Dispatches an outbound SMS via the Twilio Messages API using URL-encoded form data and Basic Authentication.
    - *Configuration:* POST request to Twilio API endpoint. Sets Messaging Service SID or From number, recipient, body, and status callback URL. Uses Basic Auth with Twilio Account SID and Auth Token.
    - *Input/Output:* Input from `Twilio connected?` (True); output connects to `read the Twilio result`.
  - **read the Twilio result**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses Twilio API responses, identifies errors, and classifies outcomes as `sent`, `transient`, `permanent`, or `_carrier_opt_out` (Twilio error code 21610).
    - *Input/Output:* Input from `[cred] Twilio - send the SMS`; output connects to `one send result` (Index 0).
  - **STOP: Twilio not configured**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Fallback Transformation). Assigns transient status to items when Twilio credentials are missing so they back off safely instead of failing permanently.
    - *Input/Output:* Input from `Twilio connected?` (False); output connects to `one send result` (Index 0).
  - **Retell connected?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Verifies whether Retell API key and Agent ID are configured (`retell_enabled === true`).
    - *Input/Output:* Input from `SMS or voice?` (False); true branch connects to `[cred] Retell - start the call`, false branch connects to `STOP: Retell not configured`.
  - **[cred] Retell - start the call**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Initiates an outbound AI voice screening call via the Retell API.
    - *Configuration:* POST request to `https://api.retellai.com/v2/create-phone-call` with Bearer token authentication and JSON body containing agent overrides, dynamic LLM variables, and metadata (enrollment ID, candidate ID, campaign ID, step, idempotency key).
    - *Input/Output:* Input from `Retell connected?` (True); output connects to `read the Retell result`.
  - **read the Retell result**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses Retell API responses, extracts call IDs, checks for permanent configuration errors, and sets status flags (`_awaiting_webhook`).
    - *Input/Output:* Input from `[cred] Retell - start the call`; output connects to `one send result` (Index 1).
  - **STOP: Retell not configured**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Fallback Transformation). Assigns transient status when Retell credentials are missing.
    - *Input/Output:* Input from `Retell connected?` (False); output connects to `one send result` (Index 1).

#### 2.4 Result Processing & Persistence
- **Overview:** This block merges provider execution responses, classifies retry policies (exponential backoff with jitter for transient errors, termination for permanent errors, advancement for successful sends), and persists audit logs and state changes to Supabase.
- **Nodes Involved:** `one send result`, `record the attempt`, `persisting?`, `STOP: not persisted`, `[cred] Supabase - write the attempts`, `[cred] Supabase - advance the enrollments`, `[cred] Supabase - record carrier opt-outs`
- **Node Details:**
  - **one send result**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Data Consolidation). Recombines the Twilio and Retell communication branches into a single unified stream.
    - *Configuration:* Two inputs merged (Version 3).
    - *Input/Output:* Inputs from Twilio/Retell read and stop nodes; output connects to `record the attempt`.
  - **record the attempt**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Builds contact attempt audit records, determines enrollment state transitions (advancing ladder steps, setting backoff schedules with jitter for transient failures, or marking permanent failures), and aggregates carrier opt-out payloads.
    - *Input/Output:* Input from `one send result`; output connects to `persisting?`.
  - **persisting?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Checks if database writes are enabled (`_write === true`, i.e., Supabase enabled and not dry-run mode).
    - *Input/Output:* Input from `record the attempt`; true branch connects to `[cred] Supabase - write the attempts`, false branch connects to `STOP: not persisted`.
  - **STOP: not persisted**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Reporting / Simulation). Formats a summary of what would have been written to the database when persistence is disabled.
    - *Input/Output:* Input from `persisting?` (False); output connects to `build the runner summary`.
  - **[cred] Supabase - write the attempts**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Inserts contact attempt logs into Supabase with duplicate conflict resolution (`on_conflict=client_id,idempotency_key`).
    - *Configuration:* POST request to Supabase REST endpoint `/rest/v1/contact_attempts` with headers for API key, Authorization, and `Prefer: resolution=ignore-duplicates,return=minimal`.
    - *Input/Output:* Input from `persisting?` (True); output connects to `[cred] Supabase - advance the enrollments`.
  - **[cred] Supabase - advance the enrollments**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Updates candidate enrollment states (advancing step indexes, updating retry counts, setting next attempt timestamps or lease expirations) in Supabase.
    - *Configuration:* POST request to Supabase REST endpoint `/rest/v1/enrollments?on_conflict=id` with `Prefer: resolution=merge-duplicates,return=minimal`.
    - *Input/Output:* Input from `[cred] Supabase - write the attempts`; output connects to `[cred] Supabase - record carrier opt-outs`.
  - **[cred] Supabase - record carrier opt-outs**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Persists carrier-level opt-outs (e.g., Twilio error 21610) to the Supabase `opt_outs` table.
    - *Configuration:* POST request to Supabase REST endpoint `/rest/v1/opt_outs?on_conflict=client_id,phone_e164,channel` with `Prefer: resolution=ignore-duplicates,return=minimal`.
    - *Input/Output:* Input from `[cred] Supabase - advance the enrollments`; output connects to `build the runner summary`.

#### 2.5 Summary & Notification
- **Overview:** This block compiles execution metrics, determines whether notification thresholds are met, and posts runner summaries to Slack.
- **Nodes Involved:** `build the runner summary`, `worth posting?`, `Post the runner summary`, `End`
- **Node Details:**
  - **build the runner summary**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Generates human-readable summary text and Slack payload structures detailing sent messages, deferrals, retries, permanent failures, and carrier opt-outs.
    - *Input/Output:* Inputs from `STOP: preview - nothing sent`, `STOP: not persisted`, or `[cred] Supabase - record carrier opt-outs`; output connects to `worth posting?`.
  - **worth posting?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Flow Control). Determines whether execution metrics warrant a Slack notification (`_post === true`).
    - *Input/Output:* Input from `build the runner summary`; true branch connects to `Post the runner summary`, false branch connects to `End`.
  - **Post the runner summary**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request). Sends formatted runner summary payloads to a Slack incoming webhook.
    - *Configuration:* POST request to `slack_webhook_url` with JSON payload.
    - *Input/Output:* Input from `worth posting?` (True); output connects to `End`.
  - **End**
    - *Type & Technical Role:* `n8n-nodes-base.noOp` (Terminal). Marks the clean completion of the workflow execution path.
    - *Input/Output:* Inputs from `worth posting?` (False) and `Post the runner summary`; no outgoing connections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every 15 minutes | n8n-nodes-base.scheduleTrigger | Trigger workflow on interval | None | config | Template overview |
| Run it now | n8n-nodes-base.manualTrigger | Manual execution trigger | None | config | Template overview |
| config | n8n-nodes-base.code | Central configuration & flags | Every 15 minutes, Run it now | Supabase connected? | Template overview |
| Supabase connected? | n8n-nodes-base.if | Check if Supabase is configured | config | [cred] Supabase - claim the due queue, STOP: using the sample due queue | Template overview |
| [cred] Supabase - claim the due queue | n8n-nodes-base.httpRequest | Claim due queue via Supabase RPC | Supabase connected? | one due queue | Template overview<br>1. Claim the due queue |
| STOP: using the sample due queue | n8n-nodes-base.code | Provide mock sample queue | Supabase connected? | one due queue | Template overview |
| one due queue | n8n-nodes-base.merge | Merge database & sample queues | [cred] Supabase - claim the due queue, STOP: using the sample due queue | decide the next touch | Template overview |
| decide the next touch | n8n-nodes-base.code | State machine & template rendering | one due queue | anything to send? | Template overview<br>2. Never texted twice |
| anything to send? | n8n-nodes-base.if | Check if any items require sending | decide the next touch | sending?, STOP: nothing due | Template overview |
| STOP: nothing due | n8n-nodes-base.code | Summarize empty/deferred queue | anything to send? | record the attempt | Template overview |
| sending? | n8n-nodes-base.if | Check if live sending is enabled | anything to send? | SMS or voice?, STOP: preview - nothing sent | Template overview |
| STOP: preview - nothing sent | n8n-nodes-base.code | Generate preview output report | sending? | build the runner summary | Template overview |
| SMS or voice? | n8n-nodes-base.if | Route to Twilio or Retell branch | sending? | Twilio connected?, Retell connected? | Template overview<br>3. Send the touch |
| Twilio connected? | n8n-nodes-base.if | Check if Twilio is configured | SMS or voice? | [cred] Twilio - send the SMS, STOP: Twilio not configured | Template overview |
| [cred] Twilio - send the SMS | n8n-nodes-base.httpRequest | Send outbound SMS via Twilio | Twilio connected? | read the Twilio result | Template overview<br>3. Send the touch |
| read the Twilio result | n8n-nodes-base.code | Parse & classify Twilio response | [cred] Twilio - send the SMS | one send result | Template overview |
| STOP: Twilio not configured | n8n-nodes-base.code | Handle missing Twilio credentials | Twilio connected? | one send result | Template overview |
| Retell connected? | n8n-nodes-base.if | Check if Retell is configured | SMS or voice? | [cred] Retell - start the call, STOP: Retell not configured | Template overview |
| [cred] Retell - start the call | n8n-nodes-base.httpRequest | Initiate outbound AI call via Retell | Retell connected? | read the Retell result | Template overview<br>3. Send the touch |
| read the Retell result | n8n-nodes-base.code | Parse & classify Retell response | [cred] Retell - start the call | one send result | Template overview |
| STOP: Retell not configured | n8n-nodes-base.code | Handle missing Retell credentials | Retell connected? | one send result | Template overview |
| one send result | n8n-nodes-base.merge | Merge Twilio & Retell branch results | read the Twilio result, STOP: Twilio not configured, read the Retell result, STOP: Retell not configured | record the attempt | Template overview<br>3. Send the touch |
| record the attempt | n8n-nodes-base.code | Calculate retries, backoff & updates | one send result | persisting? | Template overview<br>4. Retries, in three rules |
| persisting? | n8n-nodes-base.if | Check if DB write is enabled | record the attempt | [cred] Supabase - write the attempts, STOP: not persisted | Template overview |
| STOP: not persisted | n8n-nodes-base.code | Handle unpersisted write summary | persisting? | build the runner summary | Template overview |
| [cred] Supabase - write the attempts | n8n-nodes-base.httpRequest | Write contact attempts to Supabase | persisting? | [cred] Supabase - advance the enrollments | Template overview<br>5. Record and report |
| [cred] Supabase - advance the enrollments | n8n-nodes-base.httpRequest | Update enrollment state in Supabase | [cred] Supabase - write the attempts | [cred] Supabase - record carrier opt-outs | Template overview<br>5. Record and report |
| [cred] Supabase - record carrier opt-outs | n8n-nodes-base.httpRequest | Record carrier opt-outs in Supabase | [cred] Supabase - advance the enrollments | build the runner summary | Template overview<br>5. Record and report |
| build the runner summary | n8n-nodes-base.code | Compile runner summary & Slack payload | STOP: preview - nothing sent, STOP: not persisted, [cred] Supabase - record carrier opt-outs | worth posting? | Template overview<br>5. Record and report |
| worth posting? | n8n-nodes-base.if | Check if Slack notification is warranted | build the runner summary | Post the runner summary, End | Template overview |
| Post the runner summary | n8n-nodes-base.httpRequest | Post runner summary to Slack webhook | worth posting? | End | Template overview |
| End | n8n-nodes-base.noOp | Terminate workflow execution | worth posting?, Post the runner summary | None | Template overview |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Triggers and Configuration Nodes:**
   - Create a `Schedule Trigger` node named `Every 15 minutes` configured with an interval of 15 minutes.
   - Create a `Manual Trigger` node named `Run it now`.
   - Create a `Code` node named `config`. Paste the global configuration JavaScript object (defining client slugs, Supabase connection strings, sequence ladder, quiet hours, Twilio/Retell settings, and safety flags). Connect both triggers to `config`.

2. **Set up Supabase Verification & Queue Retrieval:**
   - Create an `If` node named `Supabase connected?`. Set condition to evaluate `{{ $('config').first().json.supabase_enabled }} === true`. Connect `config` output to this node.
   - Create an `HTTP Request` node named `[cred] Supabase - claim the due queue`. Set method to `POST`, URL to `={{ $('config').first().json.supabase_url }}/rest/v1/rpc/claim_due_attempts`, and configure JSON body parameters (`p_client_id`, `p_limit`, `p_lease_seconds`). Add headers for `apikey`, `Authorization`, and `Content-Type: application/json`. Enable `onError: continueRegularOutput`, `retryOnFail: true`, and `alwaysOutputData: true`. Connect the True branch of `Supabase connected?` here.
   - Create a `Code` node named `STOP: using the sample due queue` to provide mock due data when Supabase is not configured. Connect the False branch of `Supabase connected?` here.
   - Create a `Merge` node named `one due queue` (number of inputs: 2). Connect both `[cred] Supabase - claim the due queue` and `STOP: using the sample due queue` into its inputs.

3. **Implement State Machine & Sending Checks:**
   - Create a `Code` node named `decide the next touch`. Implement logic to parse queue rows, calculate candidate-local time zones, evaluate quiet hours and weekend rules, render templates, compute FNV-1a idempotency keys, and output structured execution objects. Connect `one due queue` output here.
   - Create an `If` node named `anything to send?`. Condition: `{{ $json._has_work }} === true`. Connect `decide the next touch` to this node.
   - Create a `Code` node named `STOP: nothing due` for empty/deferred queues. Connect the False branch of `anything to send?` here, routing its output to `record the attempt`.
   - Create an `If` node named `sending?`. Condition: `{{ $('config').first().json.send_enabled }} === true`. Connect the True branch of `anything to send?` here.
   - Create a `Code` node named `STOP: preview - nothing sent` to format preview metrics. Connect the False branch of `sending?` here, routing its output to `build the runner summary`.

4. **Configure Channel Routing and Provider Integration:**
   - Create an `If` node named `SMS or voice?`. Condition: `{{ $json.channel === 'sms' }} === true`. Connect the True branch of `sending?` here.
   - **Twilio Branch:**
     - Create an `If` node named `Twilio connected?`. Condition: `{{ $('config').first().json.twilio_enabled }} === true`. Connect True branch of `SMS or voice?` here.
     - Create an `HTTP Request` node named `[cred] Twilio - send the SMS`. Method `POST`, URL `=https://api.twilio.com/2010-04-01/Accounts/{{ $('config').first().json.twilio_sid }}/Messages.json`, content type `raw` (`application/x-www-form-urlencoded`), body constructed with recipient, body, and optional Messaging Service SID or From number and status callback. Configure Basic Authentication using Twilio Account SID and Auth Token. Connect True branch of `Twilio connected?` here.
     - Create a `Code` node named `read the Twilio result` to parse response statuses, error codes, and carrier opt-outs (21610). Connect `[cred] Twilio - send the SMS` here.
     - Create a `Code` node named `STOP: Twilio not configured` to handle missing Twilio credentials safely as transient errors. Connect False branch of `Twilio connected?` here.
   - **Retell Branch:**
     - Create an `If` node named `Retell connected?`. Condition: `{{ $('config').first().json.retell_enabled }} === true`. Connect False branch of `SMS or voice?` here.
     - Create an `HTTP Request` node named `[cred] Retell - start the call`. Method `POST`, URL `https://api.retellai.com/v2/create-phone-call`, Bearer token authorization, JSON body containing `from_number`, `to_number`, `override_agent_id`, `retell_llm_dynamic_variables`, and `metadata`. Connect True branch of `Retell connected?` here.
     - Create a `Code` node named `read the Retell result` to parse call IDs and classify outcomes. Connect `[cred] Retell - start the call` here.
     - Create a `Code` node named `STOP: Retell not configured` for missing Retell credentials. Connect False branch of `Retell connected?` here.
   - Create a `Merge` node named `one send result` (number of inputs: 2). Connect outputs from `read the Twilio result` / `STOP: Twilio not configured` (Input 0) and `read the Retell result` / `STOP: Retell not configured` (Input 1) into this merge node.

5. **Build Persistence and Summary Logic:**
   - Create a `Code` node named `record the attempt` to process send outcomes, compute exponential backoff with jitter, and assemble attempt logs, enrollment updates, and opt-out records. Connect `one send result` output here.
   - Create an `If` node named `persisting?`. Condition: `{{ $json._write }} === true`. Connect `record the attempt` to this node.
   - Create a `Code` node named `STOP: not persisted` for dry-run/unconfigured DB states. Connect False branch of `persisting?` here, routing its output to `build the runner summary`.
   - **Supabase Write Operations:**
     - Create an `HTTP Request` node named `[cred] Supabase - write the attempts`. Method `POST`, URL `={{ $('config').first().json.supabase_url }}/rest/v1/contact_attempts?on_conflict=client_id,idempotency_key`, JSON body `$json.attempts`, headers for API key, Authorization, and `Prefer: resolution=ignore-duplicates,return=minimal`. Connect True branch of `persisting?` here.
     - Create an `HTTP Request` node named `[cred] Supabase - advance the enrollments`. Method `POST`, URL `={{ $('config').first().json.supabase_url }}/rest/v1/enrollments?on_conflict=id`, JSON body pointing to enrollment updates, with header `Prefer: resolution=merge-duplicates,return=minimal`. Connect previous node here.
     - Create an `HTTP Request` node named `[cred] Supabase - record carrier opt-outs`. Method `POST`, URL `={{ $('config').first().json.supabase_url }}/rest/v1/opt_outs?on_conflict=client_id,phone_e164,channel`, JSON body pointing to opt-outs, with header `Prefer: resolution=ignore-duplicates,return=minimal`. Connect previous node here, routing output to `build the runner summary`.
   - **Summary & Notification:**
     - Create a `Code` node named `build the runner summary` to compile execution results and format Slack payloads.
     - Create an `If` node named `worth posting?`. Condition: `{{ $json._post }} === true`. Connect `build the runner summary` here.
     - Create an `HTTP Request` node named `Post the runner summary`. Method `POST`, URL `={{ $('config').first().json.slack_webhook_url }}`, JSON body `$json.slack_payload`. Connect True branch of `worth posting?` here.
     - Create a `NoOp` node named `End`. Connect the False branch of `worth posting?` and the output of `Post the runner summary` to `End`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Part 1 of Candidate Engine Workflow | [n8n Workflow 19775](https://n8n.io/workflows/19775-score-ats-candidates-and-enroll-top-matches-into-supabase-outreach-campaigns/) |
| Part 3 of Candidate Engine Workflow | [n8n Workflow 19777](https://n8n.io/workflows/19777-qualify-inbound-candidate-replies-and-book-interviews-with-retell-and-google-calendar/) |