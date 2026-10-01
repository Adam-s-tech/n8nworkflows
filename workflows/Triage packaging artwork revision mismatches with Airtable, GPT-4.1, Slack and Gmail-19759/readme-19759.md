Triage packaging artwork revision mismatches with Airtable, GPT-4.1, Slack and Gmail

https://n8nworkflows.xyz/workflows/triage-packaging-artwork-revision-mismatches-with-airtable--gpt-4-1--slack-and-gmail-19759


# Triage packaging artwork revision mismatches with Airtable, GPT-4.1, Slack and Gmail

### 1. Workflow Overview

This workflow automates the monitoring, comparison, and triage of packaging artwork revision mismatches. Its primary purpose is to poll an Airtable register of print jobs every 30 minutes, compare current approved versus released artwork revisions against a historical state snapshot stored in Google Sheets, filter out unstable updates via a debounce window, use OpenAI to generate concise review briefs, route critical mismatches (plated or printing states) to Slack or non-critical ones to Gmail drafts, and safely update the state snapshot only after delivery is verified.

The logic is grouped into five functional blocks:
- **1.1 Input Reception & Configuration:** Scheduled triggering and initialization of centralized configuration parameters.
- **1.2 Data Retrieval & State Comparison:** Fetching active print jobs from Airtable, pulling historical snapshots from Google Sheets, and validating/diffing revisions.
- **1.3 Debouncing & Stability Verification:** Holding execution to allow concurrent edits to settle, re-fetching the register, and confirming stable deltas.
- **1.4 AI Brief Generation & Risk Routing:** Summarizing discrepancies via an LLM model and routing alerts based on press state severity (Slack vs. Gmail).
- **1.5 Delivery Verification & State Commit:** Confirming external notification delivery before writing the updated state back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the execution cycle on a scheduled basis and injects centralized global variables (such as Airtable base IDs, table names, Google Sheet IDs, Slack channels, and recipient emails) into the data stream.
- **Nodes Involved:** 
  - `Poll Print Release Register`
  - `Normalise Print Configuration`
- **Node Details:**
  - **Poll Print Release Register**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates workflow execution every 30 minutes.
    - *Configuration:* Interval set to every 30 minutes.
    - *Inputs / Outputs:* Inputs: None (Trigger). Outputs: Connects to `Normalise Print Configuration`.
    - *Edge Cases / Failure Types:* None significant; relies on n8n internal scheduler.
  - **Normalise Print Configuration**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Establishes environment and routing variables.
    - *Configuration:* Assigns string values: `baseId` ("REPLACE_AIRTABLE_BASE"), `jobsTable` ("PrintJobs"), `sheetId` ("REPLACE_GOOGLE_SHEET"), `slackChannel` ("REPLACE_SLACK_CHANNEL"), and `recipient` ("REPLACE_INTERNAL_REVIEW_EMAIL").
    - *Inputs / Outputs:* Inputs: `Poll Print Release Register`. Outputs: `Fetch Approved Print Jobs`.
    - *Edge Cases / Failure Types:* Placeholder values (`REPLACE_...`) must be replaced prior to execution; otherwise, downstream API requests will fail.

#### 2.2 Data Retrieval & State Comparison
- **Overview:** Pulls active records from Airtable and historical snapshots from Google Sheets, cross-referencing them to calculate discrepancies, validate formatting, and identify revision deltas.
- **Nodes Involved:**
  - `Fetch Approved Print Jobs`
  - `Read Revision Snapshot`
  - `Diff Released Revisions`
  - `Check Print Source Integrity`
  - `Stop Invalid Print Input`
- **Node Details:**
  - **Fetch Approved Print Jobs**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Fetches up to 100 print job records from Airtable.
    - *Configuration:* HTTP GET request using Generic HTTP Header Authentication (Bearer token). URL constructed dynamically from configuration variables with `pageSize=100`. Configured with a 30-second timeout, 3 max tries, and 2000ms wait between retries.
    - *Inputs / Outputs:* Inputs: `Normalise Print Configuration`. Outputs: `Read Revision Snapshot`.
    - *Edge Cases / Failure Types:* Airtable authentication errors, rate limiting (HTTP 429), or pagination limits (fails closed if an offset is returned).
  - **Read Revision Snapshot**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Storage). Loads the last reported "release-register" snapshot from the "Snapshot" tab.
    - *Configuration:* Reads from document ID linked via configuration, sheet name set to "Snapshot". Always outputs data, with 3 max retries.
    - *Inputs / Outputs:* Inputs: `Fetch Approved Print Jobs`. Outputs: `Diff Released Revisions`.
    - *Edge Cases / Failure Types:* Missing sheet tabs, invalid document IDs, or OAuth authorization failures.
  - **Diff Released Revisions**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing / JavaScript). Validates structural integrity of records and compares signatures against prior snapshot state.
    - *Configuration:* Custom JavaScript verifying schema compliance (`JobId`, `ApprovedRevision`, `ReleasedRevision`, `PressState`, `ReprintExposure`), checking for pagination offsets, and filtering changed items into a delta array.
    - *Inputs / Outputs:* Inputs: `Read Revision Snapshot`. Outputs: `Check Print Source Integrity`.
    - *Edge Cases / Failure Types:* Throws validation errors if Airtable returns malformed fields or if the snapshot cell exceeds safe character limits (>45,000 characters).
  - **Check Print Source Integrity**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Validates whether the source data and snapshot passed safety checks.
    - *Configuration:* Evaluates condition `{{ $json.valid }}` equals `true`.
    - *Inputs / Outputs:* Inputs: `Diff Released Revisions`. Outputs: True branch goes to `Detect Revision Delta`; False branch goes to `Stop Invalid Print Input`.
    - *Edge Cases / Failure Types:* Halts the processing pipeline cleanly if structural inconsistencies are found.
  - **Stop Invalid Print Input**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Error Handling / Termination). Throws an explicit error message detailing structural validation failures.
    - *Configuration:* JavaScript execution: `throw Error(...)`.
    - *Inputs / Outputs:* Inputs: `Check Print Source Integrity` (False branch). Outputs: None (Terminal node).
    - *Edge Cases / Failure Types:* Intentional termination of workflow execution when data is corrupted.

#### 2.3 Debouncing & Stability Verification
- **Overview:** Implements a delay window to allow multi-user updates or unstable records to settle before re-verifying the Airtable register.
- **Nodes Involved:**
  - `Detect Revision Delta`
  - `Debounce Artwork Changes`
  - `Recheck Print Release Register`
  - `Confirm Stable Revision Delta`
- **Node Details:**
  - **Detect Revision Delta**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Determines if any artwork revisions have actually changed.
    - *Configuration:* Evaluates condition `{{ $json.changed }}` equals `true`.
    - *Inputs / Outputs:* Inputs: `Check Print Source Integrity` (True branch). Outputs: True branch connects to `Debounce Artwork Changes`. (If false, workflow ends quietly without action).
    - *Edge Cases / Failure Types:* No failures; normal bypass when no revisions have changed.
  - **Debounce Artwork Changes**
    - *Type & Technical Role:* `n8n-nodes-base.wait` (Flow Control / Delay). Pauses execution for stability settlement.
    - *Configuration:* Amount set to 60 seconds. Uses a webhook mechanism for resuming.
    - *Inputs / Outputs:* Inputs: `Detect Revision Delta`. Outputs: `Recheck Print Release Register`.
    - *Edge Cases / Failure Types:* Webhook timeout or server restart during the wait duration.
  - **Recheck Print Release Register**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Refetches print job records from Airtable post-settlement.
    - *Configuration:* Identical configuration to `Fetch Approved Print Jobs` (HTTP Header Auth, 30-second timeout, 3 retries).
    - *Inputs / Outputs:* Inputs: `Debounce Artwork Changes`. Outputs: `Confirm Stable Revision Delta`.
    - *Edge Cases / Failure Types:* Network drops or API timeouts during refetch.
  - **Confirm Stable Revision Delta**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing / JavaScript). Compares the refetched records against the initial delta check to discard transient/unstable edits.
    - *Configuration:* JavaScript validation ensuring current fetched state matches pre-wait snapshots; appends a `critical` boolean flag (`true` if press state is `plated` or `printing`).
    - *Inputs / Outputs:* Inputs: `Recheck Print Release Register`. Outputs: `Explain Reprint Exposure`.
    - *Edge Cases / Failure Types:* Throws an error if records changed mid-wait or if pagination is introduced.

#### 2.4 AI Brief Generation & Risk Routing
- **Overview:** Leverages OpenAI to draft a brief explaining the revision mismatch and routes the output to Slack or Gmail depending on press exposure severity.
- **Nodes Involved:**
  - `Supply Print Briefing Model`
  - `Explain Reprint Exposure`
  - `Shape Revision Brief`
  - `Route Press Exposure`
  - `Post Press Review Alert`
  - `Draft Queued Revision Review`
- **Node Details:**
  - **Supply Print Briefing Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Sub-node / Language Model). Provides the LLM backend configuration.
    - *Configuration:* Model set to `gpt-4.1-mini`, temperature set to `0`, 30-second timeout, 0 max retries.
    - *Inputs / Outputs:* Inputs: None (Model provider). Outputs: Connects via `ai_languageModel` to `Explain Reprint Exposure`.
    - *Edge Cases / Failure Types:* OpenAI API outages, rate limits, or quota exhaustion.
  - **Explain Reprint Exposure**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Chain Node). Generates an automated JSON summary of the revision delta.
    - *Configuration:* System prompt instructs the model to act as a packaging print coordinator, parsing notes strictly as data and returning a JSON string field named `summary`. Configured with 3 max retries and fallback handling (`continueRegularOutput`).
    - *Inputs / Outputs:* Inputs: `Confirm Stable Revision Delta` (data) and `Supply Print Briefing Model` (LLM). Outputs: `Shape Revision Brief`.
    - *Edge Cases / Failure Types:* AI timeout or malformed JSON output (handled gracefully by downstream parsing code).
  - **Shape Revision Brief**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing / JavaScript). Normalizes AI outputs, formats fallback text if the AI fails, and builds payloads for notification channels.
    - *Configuration:* JavaScript parsing `summary` from AI text, concatenating deterministic row data, and preparing subject lines, Slack text snippets, and snapshot strings.
    - *Inputs / Outputs:* Inputs: `Explain Reprint Exposure`. Outputs: `Route Press Exposure`.
    - *Edge Cases / Failure Types:* JSON parse exceptions on AI response are caught and routed to a deterministic text fallback.
  - **Route Press Exposure**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Separates critical press states from queued changes.
    - *Configuration:* Evaluates `{{ $('Shape Revision Brief').item.json.critical }}` equals `true`.
    - *Inputs / Outputs:* Inputs: `Shape Revision Brief`. Outputs: True branch connects to `Post Press Review Alert`; False branch connects to `Draft Queued Revision Review`.
    - *Edge Cases / Failure Types:* None; binary evaluation.
  - **Post Press Review Alert**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Integration / Notification). Posts operational review alerts directly to Slack.
    - *Configuration:* POST request to `https://slack.com/api/chat.postMessage` using predefined Slack API credentials. Sends JSON body containing channel, text, and disabled unfurling options. 3 max retries.
    - *Inputs / Outputs:* Inputs: `Route Press Exposure` (True branch). Outputs: `Verify Revision Brief Delivery`.
    - *Edge Cases / Failure Types:* Slack API errors, invalid channel names, or token authorization failures.
  - **Draft Queued Revision Review**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Integration). Creates an internal review email draft in Gmail for non-critical changes.
    - *Configuration:* Resource set to `draft`. Uses dynamic expressions for recipient, subject, and message body supplied by `Shape Revision Brief`. 3 max retries.
    - *Inputs / Outputs:* Inputs: `Route Press Exposure` (False branch). Outputs: `Verify Revision Brief Delivery`.
    - *Edge Cases / Failure Types:* Gmail OAuth token expiration or quota limits.

#### 2.5 Delivery Verification & State Commit
- **Overview:** Validates that the review artifact (Slack message or Gmail draft) was successfully delivered before committing the updated state snapshot back to Google Sheets.
- **Nodes Involved:**
  - `Verify Revision Brief Delivery`
  - `Save Reported Revision Snapshot`
  - `Verify Snapshot Commit`
- **Node Details:**
  - **Verify Revision Brief Delivery**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing / Guard Node). Ensures downstream notification success before updating state.
    - *Configuration:* JavaScript checking that either Slack response `ok === true` or Gmail `id` is a valid string. Throws an error otherwise.
    - *Inputs / Outputs:* Inputs: `Post Press Review Alert` or `Draft Queued Revision Review`. Outputs: `Save Reported Revision Snapshot`.
    - *Edge Cases / Failure Types:* Halts state progression if notification fails, preventing state-commit race conditions or silent failures.
  - **Save Reported Revision Snapshot**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Storage). Updates or appends the current job signature snapshot in Google Sheets.
    - *Configuration:* Operation set to `appendOrUpdate`, matching on column `Key` ("release-register") and updating the `Snapshot` column with the serialized state string. 3 max retries.
    - *Inputs / Outputs:* Inputs: `Verify Revision Brief Delivery`. Outputs: `Verify Snapshot Commit`.
    - *Edge Cases / Failure Types:* Google Sheets write API throttling, concurrency conflicts, or permission errors.
  - **Verify Snapshot Commit**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing / Guard Node). Confirms that the Google Sheets cell write was successful and matches expected payloads.
    - *Configuration:* JavaScript verifying returned row Key equals `release-register` and Snapshot value matches the dispatched payload.
    - *Inputs / Outputs:* Inputs: `Save Reported Revision Snapshot`. Outputs: Terminal execution complete (`review_filed_snapshot_saved`).
    - *Edge Cases / Failure Types:* Write verification mismatch halts execution to signal state reconciliation failure.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Poll Print Release Register | n8n-nodes-base.scheduleTrigger | Check the approved packaging production register every 30 minutes. | None | Normalise Print Configuration | **Poll Print Release Register**<br>Check the approved packaging production register every 30 minutes.<br>Configure timezone and production polling cadence. |
| Normalise Print Configuration | n8n-nodes-base.set | Normalise input and centralise buyer configuration. | Poll Print Release Register | Fetch Approved Print Jobs | **Normalise Print Configuration**<br>Normalise input and centralise buyer configuration.<br>Replace REPLACE values; inputs use explicit defaults. |
| Fetch Approved Print Jobs | n8n-nodes-base.httpRequest | Read revision, press status and reprint exposure for active jobs. | Normalise Print Configuration | Read Revision Snapshot | **Fetch Approved Print Jobs**<br>Read revision, press status and reprint exposure for active jobs.<br>Fields: JobId, ApprovedRevision, ReleasedRevision, PressState, ReprintExposure. |
| Read Revision Snapshot | n8n-nodes-base.googleSheets | Load the last successfully reported revision snapshot. | Fetch Approved Print Jobs | Diff Released Revisions | **Read Revision Snapshot**<br>Load the last successfully reported revision snapshot.<br>Snapshot columns Key, Snapshot; row release-register, {}. |
| Diff Released Revisions | n8n-nodes-base.code | Validate sources and compare revision tuples with prior state. | Read Revision Snapshot | Check Print Source Integrity | **Diff Released Revisions**<br>Validate sources and compare revision tuples with prior state.<br>Only changed revision mismatches create a delta. |
| Check Print Source Integrity | n8n-nodes-base.if | Route malformed or truncated data to a visible failure. | Diff Released Revisions | Detect Revision Delta, Stop Invalid Print Input | **Check Print Source Integrity**<br>Route malformed or truncated data to a visible failure.<br>False: stop; do not update the snapshot. |
| Detect Revision Delta | n8n-nodes-base.if | Suppress unchanged jobs and matching revisions. | Check Print Source Integrity | Debounce Artwork Changes | **Detect Revision Delta**<br>Suppress unchanged jobs and matching revisions.<br>False output ends quietly with no external writes. |
| Debounce Artwork Changes | n8n-nodes-base.wait | Allow upstream records to settle before rereading. | Detect Revision Delta | Recheck Print Release Register | **Debounce Artwork Changes**<br>Allow upstream records to settle before rereading.<br>Wait 60 seconds; no backward connection. |
| Recheck Print Release Register | n8n-nodes-base.httpRequest | Refetch revisions after the settlement window. | Debounce Artwork Changes | Confirm Stable Revision Delta | **Recheck Print Release Register**<br>Refetch revisions after the settlement window.<br>Must exactly match the first read before filing. |
| Confirm Stable Revision Delta | n8n-nodes-base.code | Drop unstable observations and mark plate or press exposure. | Recheck Print Release Register | Explain Reprint Exposure | **Confirm Stable Revision Delta**<br>Drop unstable observations and mark plate or press exposure.<br>Plated/printing mismatches are critical; next poll retries unstable rows. |
| Explain Reprint Exposure | @n8n/n8n-nodes-langchain.chainLlm | Draft a grounded explanation of the computed business decision. | Confirm Stable Revision Delta, Supply Print Briefing Model | Shape Revision Brief | **Explain Reprint Exposure**<br>Draft a grounded explanation of the computed business decision.<br>Return JSON only; never change deterministic numbers. |
| Shape Revision Brief | n8n-nodes-base.code | Create the immutable brief and next snapshot for downstream actions. | Explain Reprint Exposure | Route Press Exposure | **Shape Revision Brief**<br>Create the immutable brief and next snapshot for downstream actions.<br>AI failure uses deterministic rows; no cost or status comes from AI. |
| Route Press Exposure | n8n-nodes-base.if | Separate plated/printing risks from queued revision changes. | Shape Revision Brief | Post Press Review Alert, Draft Queued Revision Review | **Route Press Exposure**<br>Separate plated/printing risks from queued revision changes.<br>True: Slack review alert. False: internal Gmail draft. |
| Post Press Review Alert | n8n-nodes-base.httpRequest | Post a concise operational hold-review alert to the configured team. | Route Press Exposure | Verify Revision Brief Delivery | **Post Press Review Alert**<br>Post a concise operational hold-review alert to the configured team.<br>Slack API must return ok=true before state advances. |
| Draft Queued Revision Review | n8n-nodes-base.gmail | Save an internal review email as a Gmail draft. | Route Press Exposure | Verify Revision Brief Delivery | **Draft Queued Revision Review**<br>Save an internal review email as a Gmail draft.<br>Draft only; recipient, subject and body come from Shape. |
| Verify Revision Brief Delivery | n8n-nodes-base.code | Block snapshot advancement when the review artifact was not saved. | Post Press Review Alert, Draft Queued Revision Review | Save Reported Revision Snapshot | **Verify Revision Brief Delivery**<br>Block snapshot advancement when the review artifact was not saved.<br>Require Slack ok=true or a Gmail draft ID. |
| Save Reported Revision Snapshot | n8n-nodes-base.googleSheets | Replace the snapshot only after the brief is successfully filed. | Verify Revision Brief Delivery | Verify Snapshot Commit | **Save Reported Revision Snapshot**<br>Replace the snapshot only after the brief is successfully filed.<br>Keyed snapshot row; configure single-writer execution. |
| Verify Snapshot Commit | n8n-nodes-base.code | Require confirmation that the state cell was written. | Save Reported Revision Snapshot | None | **Verify Snapshot Commit**<br>Require confirmation that the state cell was written.<br>Require the returned Key and Snapshot to match. |
| Stop Invalid Print Input | n8n-nodes-base.code | Fail clearly on missing fields, pagination or corrupt state. | Check Print Source Integrity | None | **Stop Invalid Print Input**<br>Fail clearly on missing fields, pagination or corrupt state.<br>Correct the source data before retrying. |
| Supply Print Briefing Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide a low-temperature OpenAI model to the attached AI node. | None | Explain Reprint Exposure | **Supply Print Briefing Model**<br>Provide a low-temperature OpenAI model to the attached AI node.<br>Choose an available model; temperature 0. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Schedule Trigger (`Poll Print Release Register`):**
   - Add a **Schedule Trigger** node. Set interval to every 30 minutes.
2. **Configure Environment Variables (`Normalise Print Configuration`):**
   - Add a **Set** (`Edit Fields`) node.
   - Define string assignments: `baseId` (your Airtable Base ID), `jobsTable` (`PrintJobs`), `sheetId` (your Google Sheet ID), `slackChannel` (your Slack channel ID/name), and `recipient` (internal review email).
3. **Fetch Airtable Records (`Fetch Approved Print Jobs`):**
   - Add an **HTTP Request** node.
   - Set Method to `GET`, URL to `={{ 'https://api.airtable.com/v0/'+$('Normalise Print Configuration').first().json.baseId+'/'+encodeURIComponent($('Normalise Print Configuration').first().json.jobsTable)+'?pageSize=100' }}`.
   - Configure Authentication using **HTTP Header Auth** (Bearer Token credential). Enable retry on fail (3 tries, 2000ms wait). Set `Continue On Fail` to regular output.
4. **Read State Snapshot (`Read Revision Snapshot`):**
   - Add a **Google Sheets** node. Set operation to `Get Many` or `Read`.
   - Set Document ID expression: `={{ $('Normalise Print Configuration').first().json.sheetId }}` and Sheet Name to `Snapshot`. Enable `Always Output Data` and retry settings.
5. **Compute Revisions Diff (`Diff Released Revisions`):**
   - Add a **Code** node. Insert JavaScript logic to parse current Airtable records, extract snapshot values from Google Sheets where `Key === 'release-register'`, validate record schemas (`JobId`, `ApprovedRevision`, `ReleasedRevision`, `PressState`, `ReprintExposure`), and return `valid`, `problems`, `jobs`, `snapshot`, `delta`, and `changed` flags.
6. **Validate Source Integrity (`Check Print Source Integrity`):**
   - Add an **If** node checking `{{ $json.valid }}` equals `true`.
   - *True branch* connects to Step 7. *False branch* connects to Step 18 (`Stop Invalid Print Input`).
7. **Detect Revision Changes (`Detect Revision Delta`):**
   - Add an **If** node checking `{{ $json.changed }}` equals `true`.
   - *True branch* connects to Step 8.
8. **Debounce Changes (`Debounce Artwork Changes`):**
   - Add a **Wait** node. Set amount to `60` seconds.
9. **Recheck Airtable (`Recheck Print Release Register`):**
   - Add an **HTTP Request** node configured identically to step 3 (`Fetch Approved Print Jobs`).
10. **Confirm Stable Delta (`Confirm Stable Revision Delta`):**
    - Add a **Code** node comparing rechecked records against the initial diff to ensure stability, appending the `critical` boolean flag (`true` if `plated` or `printing`).
11. **Provide LLM Model (`Supply Print Briefing Model`):**
    - Add an **OpenAI Chat Model** sub-node. Select model `gpt-4.1-mini`, set temperature to `0`, timeout to 30000ms.
12. **Generate AI Explanation (`Explain Reprint Exposure`):**
    - Add a **Basic LLM Chain** node. Link its language model input to `Supply Print Briefing Model`.
    - Set Text expression to `={{ JSON.stringify($('Confirm Stable Revision Delta').item.json) }}` and configure system message prompting JSON summary output. Set error handling to continue regular output.
13. **Shape Review Artifacts (`Shape Revision Brief`):**
    - Add a **Code** node parsing AI output (with fallback to deterministic row text) and building `subject`, `reportText`, `slackText`, `recipient`, `slackChannel`, and `snapshotText`.
14. **Route Press Risk (`Route Press Exposure`):**
    - Add an **If** node evaluating `{{ $('Shape Revision Brief').item.json.critical }}` equals `true`.
    - *True branch* connects to Step 15 (`Post Press Review Alert`). *False branch* connects to Step 16 (`Draft Queued Revision Review`).
15. **Send Slack Alert (`Post Press Review Alert`):**
    - Add an **HTTP Request** node. Set Method to `POST`, URL to `https://slack.com/api/chat.postMessage`.
    - Configure Slack API credentials and JSON body mapping `channel` and `text`. Set retries on fail.
16. **Create Gmail Draft (`Draft Queued Revision Review`):**
    - Add a **Gmail** node. Resource: `Draft`, Operation: `Create`.
    - Map recipient, subject, and message body expressions from `Shape Revision Brief`. Configure Gmail credentials.
17. **Verify Delivery (`Verify Revision Brief Delivery`):**
    - Add a **Code** node verifying that Slack response `ok === true` or Gmail draft `id` exists; throw an error otherwise. Connect outputs of both Step 15 and Step 16 here.
18. **Save Updated Snapshot (`Save Reported Revision Snapshot`):**
    - Add a **Google Sheets** node. Operation: `Append or Update`, Matching Column: `Key`.
    - Set Document ID expression, Sheet Name `Snapshot`, and map columns: `Key` = `release-register`, `Snapshot` = `={{ $('Shape Revision Brief').item.json.snapshotText }}`.
19. **Verify Snapshot Commit (`Verify Snapshot Commit`):**
    - Add a **Code** node verifying that the returned Google Sheets row matches `release-register` and expected snapshot string.
20. **Handle Invalid Inputs (`Stop Invalid Print Input`):**
    - Add a **Code** node connected from the false branch of Step 6, throwing an explicit error: `throw Error('Print source validation failed: ...')`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator & Credits | Swapnil AI Labs — [swapnil.mandloi7@gmail.com](mailto:swapnil.mandloi7@gmail.com) — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Workflow Purpose & Scope | Packaging Artwork Release Watchdog designed for print production environments to track approved vs. released revision mismatches. |
| Concurrency & Execution Notice | Requires a single-writer execution configuration to prevent state-commit race conditions between scheduled runs. |