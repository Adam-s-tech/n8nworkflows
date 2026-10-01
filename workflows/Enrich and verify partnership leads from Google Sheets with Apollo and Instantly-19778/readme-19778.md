Enrich and verify partnership leads from Google Sheets with Apollo and Instantly

https://n8nworkflows.xyz/workflows/enrich-and-verify-partnership-leads-from-google-sheets-with-apollo-and-instantly-19778


# Enrich and verify partnership leads from Google Sheets with Apollo and Instantly

### 1. Workflow Overview

This workflow is an automated lead enrichment, verification, and outbound sync engine. Designed for partnership, referral, and B2B outreach teams, it processes batches of raw organisation records from Google Sheets, normalizes and classifies them, discovers decision-maker contacts via Apollo (with fallback support), scores them against an Ideal Customer Profile (ICP), verifies email deliverability via MillionVerifier, and safely pushes qualified contacts into an Instantly outreach campaign.

The logic is grouped into the following functional blocks:
- **1.1 Initialization & Configuration:** Handles scheduled or manual triggers and establishes all runtime settings, limits, rules, and flags.
- **1.2 Ingestion & Batching:** Reads the raw organisation list from Google Sheets, filters out stale or duplicate entries, and extracts a controlled batch for processing.
- **1.3 Organisation Normalization & Classification:** Cleans organization domains and applies keyword rules or an optional Anthropic AI classification step.
- **1.4 Contact Discovery:** Queries the Apollo API for decision-maker contacts or falls back to pre-existing contact data stored in the Google Sheet.
- **1.5 ICP Scoring & Email Verification:** Scores contacts based on title weights and demographic bonuses, then verifies email addresses using MillionVerifier (or handles unverified paths securely).
- **1.6 Safety Gate & Data Persistence:** Evaluates final sending eligibility (failing closed on risk factors), writes organization and contact results back to Google Sheets, and pushes approved leads to Instantly.
- **1.7 Reporting & Alerting:** Compiles run metrics, evaluates system warnings, and dispatches a summary report via webhook.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Initialization & Configuration
**Overview:** This block triggers the workflow either via a schedule or manually and sets up the centralized configuration parameters, feature flags, and business rules.

**Nodes Involved:**
- `Run it now`
- `Every weekday, 07:00`
- `config`

**Node Details:**

- **Run it now**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Workflow manual execution entry point).
  - **Configuration:** Default parameters.
  - **Connections:** Output connects to `config`.
  - **Edge Cases:** None.

- **Every weekday, 07:00**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Cron-based automated trigger).
  - **Configuration:** Cron expression set to `0 7 * * 1-5` (Monday through Friday at 07:00).
  - **Connections:** Output connects to `config`.
  - **Edge Cases:** Execution timezone depends on n8n server settings.

- **config**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript initialization block).
  - **Configuration:** Defines global constants including Google Sheet IDs, tab names, column headers, batch sizes, daily caps, ICP rules, suppression lists, API keys, and safety mode logic (`preview`, `test`, or `live`).
  - **Key Expressions:** Evaluates runtime modes based on `TEST_RUN`, `TEST_CAMPAIGN_ID`, and API key presence.
  - **Connections:** Inputs from `Run it now` and `Every weekday, 07:00`; output connects to `[cred] Sheets - read the raw list`.
  - **Edge Cases:** Blank API keys correctly disable integrations rather than failing execution.

---

#### 1.2 Ingestion & Batching
**Overview:** This block reads the raw organisation database from Google Sheets, filters out completed or duplicate records, and slices out a defined batch size for the run.

**Nodes Involved:**
- `[cred] Sheets - read the raw list`
- `pick this run's batch`
- `anything to enrich?`
- `STOP: nothing to enrich`

**Node Details:**

- **[cred] Sheets - read the raw list**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets integration reader).
  - **Configuration:** Operation set to read rows; utilizes expressions to fetch spreadsheet ID and tab name from the `config` node. `alwaysOutputData` is enabled.
  - **Connections:** Input from `config`; output connects to `pick this run's batch`.
  - **Edge Cases:** If the tab contains only headers, `alwaysOutputData` ensures an empty item structure is passed downstream rather than halting silently.

- **pick this run's batch**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript data filtering and deduplication).
  - **Configuration:** Filters rows matching `READY_STATUSES`, checks enrichment timestamps against `REENRICH_AFTER_DAYS`, deduplicates by normalized website/name, and limits output to `BATCH_SIZE`.
  - **Connections:** Input from `[cred] Sheets - read the raw list`; output connects to `anything to enrich?`.
  - **Edge Cases:** Empty sheets return metadata tracking flags (`_sheet_empty: true`).

- **anything to enrich?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional branching router).
  - **Configuration:** Evaluates whether `_has_work` equals `true`.
  - **Connections:** Input from `pick this run's batch`; outputs connect to `normalise the organisation` (true path) and `STOP: nothing to enrich` (false path).
  - **Edge Cases:** Loose type validation enabled.

- **STOP: nothing to enrich**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Termination/logging block).
  - **Configuration:** Formats a clear diagnostic message indicating why no records were processed (e.g., empty tab or all items already enriched).
  - **Connections:** Input from `anything to enrich?`; output connects to `End`.
  - **Edge Cases:** Gracefully terminates the execution path with explanatory details.

---

#### 1.3 Organisation Normalization & Classification
**Overview:** This block normalizes organization names and domains, applies deterministic keyword rules, and conditionally triggers an Anthropic AI classification call for unclassified entities.

**Nodes Involved:**
- `normalise the organisation`
- `AI classification needed?`
- `AI - classify the organisation`
- `apply the AI classification`
- `STOP: keyword rules only`
- `one list of organisations`

**Node Details:**

- **normalise the organisation**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript text normalization and payload builder).
  - **Configuration:** Strips protocols and paths from website URLs to establish clean domains; checks category rules against a search haystack; constructs Anthropic and Apollo API payloads.
  - **Connections:** Input from `anything to enrich?`; output connects to `AI classification needed?`.
  - **Edge Cases:** Handles missing website strings or special characters gracefully.

- **AI classification needed?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `_needs_ai` equals `true`.
  - **Connections:** Input from `normalise the organisation`; outputs connect to `AI - classify the organisation` (true) and `STOP: keyword rules only` (false).
  - **Edge Cases:** Bypasses AI if keyword rules established confidence or if AI is disabled.

- **AI - classify the organisation**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Anthropic Messages API client).
  - **Configuration:** Sends a POST request to `ai_url` with API key headers and a JSON body containing system and user prompts. `onError` set to `continueRegularOutput`, with automatic retries.
  - **Connections:** Input from `AI classification needed?`; output connects to `apply the AI classification`.
  - **Edge Cases:** Handles API timeouts (30s) and transient HTTP failures by continuing regular output.

- **apply the AI classification**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript parser).
  - **Configuration:** Parses Anthropic response blocks, validates the returned category against allowed configuration values, and sets fallback values if parsing fails.
  - **Connections:** Input from `AI - classify the organisation`; output connects to `one list of organisations` (Input 0).
  - **Edge Cases:** Malformed JSON or unexpected model output defaults to `'Unclassified'` without breaking the pipeline.

- **STOP: keyword rules only**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Passthrough/logging block).
  - **Configuration:** Flags items that bypassed AI classification (`_ai_called: false`) for tracking.
  - **Connections:** Input from `AI classification needed?` (false branch); output connects to `one list of organisations` (Input 1).
  - **Edge Cases:** None.

- **one list of organisations**
  - **Type & Technical Role:** `n8n-nodes-base.merge` (Data merger).
  - **Configuration:** Mode set to `append` (combines AI-classified and keyword-classified items).
  - **Connections:** Inputs from `apply the AI classification` and `STOP: keyword rules only`; output connects to `Apollo configured?`.
  - **Edge Cases:** Preserves array order and metadata structures.

---

#### 1.4 Contact Discovery
**Overview:** This block queries the Apollo API to discover decision-maker contacts for each organization, or falls back to pre-configured contact columns in Google Sheets if Apollo is disabled.

**Nodes Involved:**
- `Apollo configured?`
- `Apollo - find decision makers`
- `read Apollo's people`
- `STOP: Apollo not configured`
- `one list of contacts`

**Node Details:**

- **Apollo configured?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `apollo_enabled` equals `true`.
  - **Connections:** Input from `one list of organisations`; outputs connect to `Apollo - find decision makers` (true) and `STOP: Apollo not configured` (false).
  - **Edge Cases:** Directs execution to fallback logic if the Apollo API key is blank.

- **Apollo - find decision makers**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Apollo API client).
  - **Configuration:** Sends a POST request to Apollo’s mixed people search endpoint using JSON bodies built during normalization. `onError` set to `continueRegularOutput` with retries.
  - **Connections:** Input from `Apollo configured?`; output connects to `read Apollo's people`.
  - **Edge Cases:** Handles rate limits or upstream timeouts.

- **read Apollo's people**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript contact extractor).
  - **Configuration:** Parses Apollo search results, filters out placeholder/unlocked email strings (e.g., `email_not_unlocked`), and outputs up to `MAX_CONTACTS_PER_ORG` individual contact items.
  - **Connections:** Input from `Apollo - find decision makers`; output connects to `one list of contacts` (Input 0).
  - **Edge Cases:** Marks organizations with zero valid contacts as `_no_contact: true`.

- **STOP: Apollo not configured**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Fallback contact reader).
  - **Configuration:** Reads contact name, title, and email columns directly from the raw Google Sheet row when Apollo is inactive.
  - **Connections:** Input from `Apollo configured?` (false branch); output connects to `one list of contacts` (Input 1).
  - **Edge Cases:** Marks records without email addresses as uncontactable.

- **one list of contacts**
  - **Type & Technical Role:** `n8n-nodes-base.merge` (Data merger).
  - **Configuration:** Mode set to `append`.
  - **Connections:** Inputs from `read Apollo's people` and `STOP: Apollo not configured`; output connects to `score against the ICP`.
  - **Edge Cases:** Combines API-sourced and sheet-sourced contact arrays into a unified stream.

---

#### 1.5 ICP Scoring & Email Verification
**Overview:** This block scores contacts against ICP title and demographic rules, checks verification prerequisites, and executes email deliverability checks via MillionVerifier.

**Nodes Involved:**
- `score against the ICP`
- `verification possible?`
- `Verifier - check the address`
- `apply the verification`
- `STOP: unverified - held back`
- `one list of verdicts`

**Node Details:**

- **score against the ICP**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript scoring engine).
  - **Configuration:** Evaluates job titles against weighted title rules and exclusion lists; applies category, phone, LinkedIn, and name bonuses to calculate a 0–100 ICP score and assign tiers (A, B, C, X).
  - **Connections:** Input from `one list of contacts`; output connects to `verification possible?`.
  - **Edge Cases:** Excluded titles are assigned tier `X` and given a score of 0.

- **verification possible?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `_verifiable` equals `true` (requires a valid email format and an active verifier API key).
  - **Connections:** Input from `score against the ICP`; outputs connect to `Verifier - check the address` (true) and `STOP: unverified - held back` (false).
  - **Edge Cases:** Routes unverified or uncheckable emails to the fallback path.

- **Verifier - check the address**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (MillionVerifier v3 API client).
  - **Configuration:** Sends a GET request to `verifier_url` with API key, email parameter, and a 10-second timeout. `onError` set to `continueRegularOutput` with retries.
  - **Connections:** Input from `verification possible?`; output connects to `apply the verification`.
  - **Edge Cases:** Timeouts or API errors are captured gracefully.

- **apply the verification**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript verdict parser).
  - **Configuration:** Parses MillionVerifier responses, maps status strings against `ACCEPT_STATUSES` and `RISKY_STATUSES`, and determines whether an address is `accepted`, `risky`, or `rejected`.
  - **Connections:** Input from `Verifier - check the address`; output connects to `one list of verdicts` (Input 0).
  - **Edge Cases:** Unrecognized statuses default to `unknown` and fail closed.

- **STOP: unverified - held back**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Fallback verification handler).
  - **Configuration:** Assigns fallback statuses to unverified contacts. Supports deterministic demo verdicts if `DEMO_VERIFIER` is enabled.
  - **Connections:** Input from `verification possible?` (false branch); output connects to `one list of verdicts` (Input 1).
  - **Edge Cases:** Prevents unverified records from being marked as verified.

- **one list of verdicts**
  - **Type & Technical Role:** `n8n-nodes-base.merge` (Data merger).
  - **Configuration:** Mode set to `append`.
  - **Connections:** Inputs from `apply the verification` and `STOP: unverified - held back`; output connects to `decide who is safe to email`.
  - **Edge Cases:** Unifies verified and unverified contact streams.

---

#### 1.6 Safety Gate & Data Persistence
**Overview:** This block executes the final sending gate (failing closed on all risk factors), writes updated organization and contact rows back to Google Sheets, and pushes approved leads to Instantly.

**Nodes Involved:**
- `decide who is safe to email`
- `org-rows`
- `[cred] Sheets - update the organisations`
- `any-contacts`
- `STOP: no contacts found`
- `contact-rows`
- `[cred] Sheets - write the contacts`
- `pushing to the campaign?`
- `payload for the campaign`
- `Instantly - add to the campaign`
- `pushed`
- `[cred] Sheets - mark them pushed`
- `STOP: not pushed`

**Node Details:**

- **decide who is safe to email**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript safety gate).
  - **Configuration:** Evaluates suppression lists, duplicate emails, shared mailbox prefixes, state requirements, verifier verdicts, minimum ICP scores, and daily sending caps. Assigns plain-text hold reasons.
  - **Connections:** Input from `one list of verdicts`; output connects to `org-rows`.
  - **Edge Cases:** Fails closed; any ambiguity results in the contact being held back (`_safe: false`).

- **org-rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript row aggregator).
  - **Configuration:** Folds contact results back into organization-level records, determining final organization statuses (`Enriched` vs. `No contact found`) and collating notes.
  - **Connections:** Input from `decide who is safe to email`; output connects to `[cred] Sheets - update the organisations`.
  - **Edge Cases:** Summarizes multi-contact findings into a single organization row update.

- **[cred] Sheets - update the organisations**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets updater).
  - **Configuration:** Operation set to `appendOrUpdate`, matching on `Org id` with cell format set to `RAW`.
  - **Connections:** Input from `org-rows`; output connects to `any-contacts`.
  - **Edge Cases:** Requires the `Org id` column header to match configuration specifications.

- **any-contacts**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `_contacts_to_write` equals `true`.
  - **Connections:** Input from `[cred] Sheets - update the organisations`; outputs connect to `contact-rows` (true) and `STOP: no contacts found` (false).
  - **Edge Cases:** Loose type validation enabled.

- **STOP: no contacts found**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Termination/logging block).
  - **Configuration:** Compiles hold reason distributions when no usable contacts were discovered.
  - **Connections:** Input from `any-contacts` (false branch); output connects to `build the run summary`.
  - **Edge Cases:** Routes diagnostic statistics to reporting.

- **contact-rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript contact row mapper).
  - **Configuration:** Maps approved and held contact attributes to the exact column schema required by the Contacts Google Sheet tab.
  - **Connections:** Input from `any-contacts` (true branch); output connects to `[cred] Sheets - write the contacts`.
  - **Edge Cases:** Omits push-specific timestamp fields during initial database writes to prevent overwriting historical push markers.

- **[cred] Sheets - write the contacts**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets updater).
  - **Configuration:** Operation set to `appendOrUpdate`, matching on `Contact id` with cell format set to `RAW`.
  - **Connections:** Input from `contact-rows`; output connects to `pushing to the campaign?`.
  - **Edge Cases:** Every contact (including held ones with reasons) is persisted in the database.

- **pushing to the campaign?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `_any_safe` equals `true` and push operations are enabled.
  - **Connections:** Input from `[cred] Sheets - write the contacts`; outputs connect to `payload for the campaign` (true) and `STOP: not pushed` (false).
  - **Edge Cases:** Defaults to preview mode (`STOP: not pushed`) when test switches or API keys restrict live sending.

- **payload for the campaign**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript API payload builder).
  - **Configuration:** Constructs Instantly v2 lead payloads containing contact identifiers, emails, names, and custom personalization variables.
  - **Connections:** Input from `pushing to the campaign?`; output connects to `Instantly - add to the campaign`.
  - **Edge Cases:** Filters out non-safe records.

- **Instantly - add to the campaign**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Instantly v2 API client).
  - **Configuration:** Sends a POST request to Instantly’s leads endpoint with Bearer token authentication and JSON bodies. `onError` set to `continueRegularOutput` with retries.
  - **Connections:** Input from `payload for the campaign`; output connects to `pushed`.
  - **Edge Cases:** Handles API rejections gracefully without halting the workflow.

- **pushed**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript response parser).
  - **Configuration:** Evaluates Instantly API response payloads for successful lead creation IDs, updating push timestamps and capturing rejection reasons if applicable.
  - **Connections:** Input from `Instantly - add to the campaign`; output connects to `[cred] Sheets - mark them pushed`.
  - **Edge Cases:** Rejections are diverted into hold reason fields rather than recording false successes.

- **[cred] Sheets - mark them pushed**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets updater).
  - **Configuration:** Operation set to `appendOrUpdate`, matching on `Contact id`.
  - **Connections:** Input from `pushed`; output connects to `build the run summary`.
  - **Edge Cases:** Updates only successfully accepted leads.

- **STOP: not pushed**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Review screen generator).
  - **Configuration:** Compiles a comprehensive review payload containing names, titles, scores, verifier verdicts, and hold reasons for all contacts processed during preview runs.
  - **Connections:** Input from `pushing to the campaign?` (false branch); output connects to `build the run summary`.
  - **Edge Cases:** Serves as the primary inspection screen when `TEST_RUN` is active.

---

#### 1.7 Reporting & Alerting
**Overview:** This block compiles run metrics, formats system warnings, and dispatches a run summary report to a Slack webhook endpoint.

**Nodes Involved:**
- `build the run summary`
- `alert configured?`
- `Post the run summary`
- `STOP: no alert webhook`
- `End`

**Node Details:**

- **build the run summary**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript metrics aggregator).
  - **Configuration:** Calculates organizations worked, contacts found, verified deliverable rates, bounce risk avoided, hold reason distributions, and system warnings. Formats text for Slack webhooks.
  - **Connections:** Inputs from `STOP: not pushed`, `[cred] Sheets - mark them pushed`, `STOP: no contacts found`, and `STOP: nothing to enrich`; output connects to `alert configured?`.
  - **Edge Cases:** Safely handles executions where push branches did not activate.

- **alert configured?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional router).
  - **Configuration:** Evaluates whether `alert_enabled` equals `true`.
  - **Connections:** Input from `build the run summary`; outputs connect to `Post the run summary` (true) and `STOP: no alert webhook` (false).
  - **Edge Cases:** Routes to silent termination if no webhook URL is configured.

- **Post the run summary**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Webhook HTTP client).
  - **Configuration:** Sends a POST request to `alert_webhook_url` with JSON-formatted summary text. `onError` set to `continueRegularOutput`.
  - **Connections:** Input from `alert configured?` (true branch); output connects to `End`.
  - **Edge Cases:** Network timeouts (15s) do not crash the workflow.

- **STOP: no alert webhook**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (No-operation pass-through).
  - **Configuration:** Acts as a quiet termination point when alerting is disabled.
  - **Connections:** Input from `alert configured?` (false branch); output connects to `End`.
  - **Edge Cases:** None.

- **End**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` (Workflow completion point).
  - **Configuration:** Terminal node for all execution paths.
  - **Connections:** Inputs from `STOP: nothing to enrich`, `Post the run summary`, and `STOP: no alert webhook`.
  - **Edge Cases:** Marks successful completion of the workflow execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run it now` | `n8n-nodes-base.manualTrigger` | Workflow manual execution entry point | None | `config` | Template overview |
| `Every weekday, 07:00` | `n8n-nodes-base.scheduleTrigger` | Cron-based automated trigger | None | `config` | Template overview |
| `config` | `n8n-nodes-base.code` | JavaScript initialization block | `Run it now`, `Every weekday, 07:00` | `[cred] Sheets - read the raw list` | Template overview <br> 1. Take a batch, not the whole list <br> 3. This is the under-2% bounce rate <br> 4. Every optional step degrades, none of them fail <br> 5. Nothing is pushed until you say so |
| `[cred] Sheets - read the raw list` | `n8n-nodes-base.googleSheets` | Google Sheets integration reader | `config` | `pick this run's batch` | Template overview |
| `pick this run's batch` | `n8n-nodes-base.code` | Data filtering and deduplication | `[cred] Sheets - read the raw list` | `anything to enrich?` | 1. Take a batch, not the whole list |
| `anything to enrich?` | `n8n-nodes-base.if` | Conditional branching router | `pick this run's batch` | `normalise the organisation`, `STOP: nothing to enrich` | |
| `STOP: nothing to enrich` | `n8n-nodes-base.code` | Termination/logging block | `anything to enrich?` | `End` | |
| `normalise the organisation` | `n8n-nodes-base.code` | Text normalization and payload builder | `anything to enrich?` | `AI classification needed?` | 1. Take a batch, not the whole list |
| `AI classification needed?` | `n8n-nodes-base.if` | Conditional router | `normalise the organisation` | `AI - classify the organisation`, `STOP: keyword rules only` | 2. Classify, then find the people |
| `AI - classify the organisation` | `n8n-nodes-base.httpRequest` | Anthropic Messages API client | `AI classification needed?` | `apply the AI classification` | 2. Classify, then find the people |
| `apply the AI classification` | `n8n-nodes-base.code` | JSON parser | `AI - classify the organisation` | `one list of organisations` | 2. Classify, then find the people |
| `STOP: keyword rules only` | `n8n-nodes-base.code` | Passthrough/logging block | `AI classification needed?` | `one list of organisations` | 2. Classify, then find the people |
| `one list of organisations` | `n8n-nodes-base.merge` | Data merger | `apply the AI classification`, `STOP: keyword rules only` | `Apollo configured?` | 2. Classify, then find the people |
| `Apollo configured?` | `n8n-nodes-base.if` | Conditional router | `one list of organisations` | `Apollo - find decision makers`, `STOP: Apollo not configured` | 2. Classify, then find the people <br> 4. Every optional step degrades, none of them fail |
| `Apollo - find decision makers` | `n8n-nodes-base.httpRequest` | Apollo API client | `Apollo configured?` | `read Apollo's people` | 2. Classify, then find the people |
| `read Apollo's people` | `n8n-nodes-base.code` | Contact extractor | `Apollo - find decision makers` | `one list of contacts` | 2. Classify, then find the people |
| `STOP: Apollo not configured` | `n8n-nodes-base.code` | Fallback contact reader | `Apollo configured?` | `one list of contacts` | 2. Classify, then find the people <br> 4. Every optional step degrades, none of them fail |
| `one list of contacts` | `n8n-nodes-base.merge` | Data merger | `read Apollo's people`, `STOP: Apollo not configured` | `score against the ICP` | |
| `score against the ICP` | `n8n-nodes-base.code` | Scoring engine | `one list of contacts` | `verification possible?` | Template overview |
| `verification possible?` | `n8n-nodes-base.if` | Conditional router | `score against the ICP` | `Verifier - check the address`, `STOP: unverified - held back` | 3. This is the under-2% bounce rate |
| `Verifier - check the address` | `n8n-nodes-base.httpRequest` | MillionVerifier v3 API client | `verification possible?` | `apply the verification` | 3. This is the under-2% bounce rate |
| `apply the verification` | `n8n-nodes-base.code` | Verdict parser | `Verifier - check the address` | `one list of verdicts` | 3. This is the under-2% bounce rate |
| `STOP: unverified - held back` | `n8n-nodes-base.code` | Fallback verification handler | `verification possible?` | `one list of verdicts` | 3. This is the under-2% bounce rate <br> 5. Nothing is pushed until you say so |
| `one list of verdicts` | `n8n-nodes-base.merge` | Data merger | `apply the verification`, `STOP: unverified - held back` | `decide who is safe to email` | |
| `decide who is safe to email` | `n8n-nodes-base.code` | Safety gate | `one list of verdicts` | `org-rows` | |
| `org-rows` | `n8n-nodes-base.code` | Row aggregator | `decide who is safe to email` | `[cred] Sheets - update the organisations` | |
| `[cred] Sheets - update the organisations` | `n8n-nodes-base.googleSheets` | Google Sheets updater | `org-rows` | `any-contacts` | |
| `any-contacts` | `n8n-nodes-base.if` | Conditional router | `[cred] Sheets - update the organisations` | `contact-rows`, `STOP: no contacts found` | |
| `STOP: no contacts found` | `n8n-nodes-base.code` | Termination/logging block | `any-contacts` | `build the run summary` | |
| `contact-rows` | `n8n-nodes-base.code` | Contact row mapper | `any-contacts` | `[cred] Sheets - write the contacts` | |
| `[cred] Sheets - write the contacts` | `n8n-nodes-base.googleSheets` | Google Sheets updater | `contact-rows` | `pushing to the campaign?` | |
| `pushing to the campaign?` | `n8n-nodes-base.if` | Conditional router | `[cred] Sheets - write the contacts` | `payload for the campaign`, `STOP: not pushed` | 5. Nothing is pushed until you say so |
| `payload for the campaign` | `n8n-nodes-base.code` | API payload builder | `pushing to the campaign?` | `Instantly - add to the campaign` | |
| `Instantly - add to the campaign` | `n8n-nodes-base.httpRequest` | Instantly v2 API client | `payload for the campaign` | `pushed` | |
| `pushed` | `n8n-nodes-base.code` | Response parser | `Instantly - add to the campaign` | `[cred] Sheets - mark them pushed` | |
| `[cred] Sheets - mark them pushed` | `n8n-nodes-base.googleSheets` | Google Sheets updater | `pushed` | `build the run summary` | |
| `STOP: not pushed` | `n8n-nodes-base.code` | Review screen generator | `pushing to the campaign?` | `build the run summary` | 5. Nothing is pushed until you say so |
| `build the run summary` | `n8n-nodes-base.code` | Metrics aggregator | `STOP: not pushed`, `[cred] Sheets - mark them pushed`, `STOP: no contacts found`, `STOP: nothing to enrich` | `alert configured?` | 6. Report what happened |
| `alert configured?` | `n8n-nodes-base.if` | Conditional router | `build the run summary` | `Post the run summary`, `STOP: no alert webhook` | 6. Report what happened |
| `Post the run summary` | `n8n-nodes-base.httpRequest` | Webhook HTTP client | `alert configured?` | `End` | 6. Report what happened |
| `STOP: no alert webhook` | `n8n-nodes-base.noOp` | No-operation pass-through | `alert configured?` | `End` | 6. Report what happened |
| `End` | `n8n-nodes-base.noOp` | Workflow completion point | `STOP: nothing to enrich`, `Post the run summary`, `STOP: no alert webhook` | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Trigger Nodes:**
   - Add a `Manual Trigger` (`n8n-nodes-base.manualTrigger`) named `Run it now`.
   - Add a `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) named `Every weekday, 07:00`. Set the cron expression to `0 7 * * 1-5`.

2. **Add Configuration Node:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `config`. Paste the configuration JavaScript code (defining `SHEET_ID`, column mappings, category rules, ICP scoring thresholds, API keys, and mode switches).
   - Connect both triggers to `config`.

3. **Ingestion & Batching Nodes:**
   - Add a `Google Sheets` node (`n8n-nodes-base.googleSheets`) named `[cred] Sheets - read the raw list`. Configure it to read from `ORGS_TAB` using spreadsheet ID from `config`. Enable `alwaysOutputData`. Connect `config` to this node.
   - Add a `Code` node named `pick this run's batch`. Connect `[cred] Sheets - read the raw list` here.
   - Add an `If` node named `anything to enrich?`. Condition: `{{ $json._has_work }} === true`. Connect `pick this run's batch` here.
   - Add a `Code` node named `STOP: nothing to enrich`. Connect the `false` output of `anything to enrich?` here, and connect its output to `End`.

4. **Normalization & Classification Nodes:**
   - Add a `Code` node named `normalise the organisation`. Connect the `true` output of `anything to enrich?` here.
   - Add an `If` node named `AI classification needed?`. Condition: `{{ $json._needs_ai }} === true`. Connect `normalise the organisation` here.
   - Add an HTTP Request node named `AI - classify the organisation`. Method: `POST`, URL: `={{ $('config').first().json.ai_url }}`, headers: `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json`. Set body to JSON. Connect `true` output of `AI classification needed?` here.
   - Add a `Code` node named `apply the AI classification`. Connect `AI - classify the organisation` here.
   - Add a `Code` node named `STOP: keyword rules only`. Connect `false` output of `AI classification needed?` here.
   - Add a `Merge` node (`n8n-nodes-base.merge`) named `one list of organisations` in `append` mode. Connect `apply the AI classification` (Input 0) and `STOP: keyword rules only` (Input 1) here.

5. **Contact Discovery Nodes:**
   - Add an `If` node named `Apollo configured?`. Condition: `={{ $('config').first().json.apollo_enabled }} === true`. Connect `one list of organisations` here.
   - Add an HTTP Request node named `Apollo - find decision makers`. Method: `POST`, URL: `={{ $('config').first().json.apollo_url }}`, headers: `x-api-key`, `Content-Type: application/json`. Connect `true` output of `Apollo configured?` here.
   - Add a `Code` node named `read Apollo's people`. Connect `Apollo - find decision makers` here.
   - Add a `Code` node named `STOP: Apollo not configured`. Connect `false` output of `Apollo configured?` here.
   - Add a `Merge` node named `one list of contacts` in `append` mode. Connect `read Apollo's people` (Input 0) and `STOP: Apollo not configured` (Input 1) here.

6. **ICP Scoring & Email Verification Nodes:**
   - Add a `Code` node named `score against the ICP`. Connect `one list of contacts` here.
   - Add an `If` node named `verification possible?`. Condition: `={{ $json._verifiable }} === true`. Connect `score against the ICP` here.
   - Add an HTTP Request node named `Verifier - check the address`. Method: `GET`, URL: `={{ $('config').first().json.verifier_url }}`, query parameters: `api`, `email`, `timeout`. Connect `true` output of `verification possible?` here.
   - Add a `Code` node named `apply the verification`. Connect `Verifier - check the address` here.
   - Add a `Code` node named `STOP: unverified - held back`. Connect `false` output of `verification possible?` here.
   - Add a `Merge` node named `one list of verdicts` in `append` mode. Connect `apply the verification` (Input 0) and `STOP: unverified - held back` (Input 1) here.

7. **Safety Gate & Data Persistence Nodes:**
   - Add a `Code` node named `decide who is safe to email`. Connect `one list of verdicts` here.
   - Add a `Code` node named `org-rows`. Connect `decide who is safe to email` here.
   - Add a `Google Sheets` node named `[cred] Sheets - update the organisations`. Operation: `appendOrUpdate`, matching columns: `Org id`. Connect `org-rows` here.
   - Add an `If` node named `any-contacts?`. Condition: `={{ $('rows for the organisations tab').first().json._contacts_to_write }} === true`. Connect `[cred] Sheets - update the organisations` here.
   - Add a `Code` node named `STOP: no contacts found`. Connect `false` output of `any-contacts?` here.
   - Add a `Code` node named `contact-rows`. Connect `true` output of `any-contacts?` here.
   - Add a `Google Sheets` node named `[cred] Sheets - write the contacts`. Operation: `appendOrUpdate`, matching columns: `Contact id`. Connect `contact-rows` here.
   - Add an `If` node named `pushing to the campaign?`. Condition: `={{ $('rows for the organisations tab').first().json._any_safe }} === true`. Connect `[cred] Sheets - write the contacts` here.
   - Add a `Code` node named `payload for the campaign`. Connect `true` output of `pushing to the campaign?` here.
   - Add an HTTP Request node named `Instantly - add to the campaign`. Method: `POST`, URL: `={{ $('config').first().json.instantly_url }}`, headers: `Authorization: Bearer <key>`, `Content-Type: application/json`. Connect `payload for the campaign` here.
   - Add a `Code` node named `pushed`. Connect `Instantly - add to the campaign` here.
   - Add a `Google Sheets` node named `[cred] Sheets - mark them pushed`. Operation: `appendOrUpdate`, matching columns: `Contact id`. Connect `pushed` here.
   - Add a `Code` node named `STOP: not pushed`. Connect `false` output of `pushing to the campaign?` here.

8. **Reporting & Alerting Nodes:**
   - Add a `Code` node named `build the run summary`. Connect outputs from `[cred] Sheets - mark them pushed`, `STOP: not pushed`, `STOP: no contacts found`, and `STOP: nothing to enrich` here.
   - Add an `If` node named `alert configured?`. Condition: `={{ $('config').first().json.alert_enabled }} === true`. Connect `build the run summary` here.
   - Add an HTTP Request node named `Post the run summary`. Method: `POST`, URL: `={{ $('config').first().json.alert_webhook_url }}`, headers: `Content-Type: application/json`. Connect `true` output of `alert configured?` here.
   - Add a `NoOp` node named `STOP: no alert webhook`. Connect `false` output of `alert configured?` here.
   - Add a `NoOp` node named `End`. Connect outputs from `Post the run summary`, `STOP: no alert webhook`, and `STOP: nothing to enrich` here.

9. **Credentials Setup:**
   - Configure Google Sheets OAuth2 credentials across all four Google Sheets nodes.
   - Ensure the Google Sheet tabs (`Organisations` and `Contacts`) contain row 1 headers matching the keys defined in `config`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Web Address Structure | `docs.google.com/spreadsheets/d/<THIS BIT>/edit` |
| Apollo Developer Settings | `app.apollo.io -> Settings -> Integrations -> API` |
| MillionVerifier API Documentation | `https://www.millionverifier.com/api-docs` |
| Instantly v2 API Settings | `Settings -> Integrations -> API keys` |