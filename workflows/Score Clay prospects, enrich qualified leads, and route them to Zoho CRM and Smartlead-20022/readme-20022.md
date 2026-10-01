Score Clay prospects, enrich qualified leads, and route them to Zoho CRM and Smartlead

https://n8nworkflows.xyz/workflows/score-clay-prospects--enrich-qualified-leads--and-route-them-to-zoho-crm-and-smartlead-20022


# Score Clay prospects, enrich qualified leads, and route them to Zoho CRM and Smartlead

### 1. Workflow Overview

This workflow is an outbound scoring, enrichment, and routing engine. Its primary purpose is to ingest prospect lists originating from Clay (or evaluated via built-in demo records), score and tier them against configurable criteria before spending credits, enrich only qualified prospects meeting specific thresholds, verify email deliverability, and synchronize the resulting lead states to Zoho CRM and Smartlead campaigns.

The execution logic is structured into the following functional blocks:
- **1.1 Input Reception & Configuration Initialization:** Captures incoming prospect data via HTTP webhooks or manual demo data loaders, merging them with centralized scoring, verification, CRM, and campaign parameters.
- **1.2 Prospect Parsing, Scoring, & Tiering:** Normalizes input schemas, extracts seniority and buyer personas, calculates lead scores, flags duplicates, and filters prospects against scoring rules and disqualification lists.
- **1.3 Credit-Gated Enrichment:** Determines which prospects qualify for paid contact data enrichment based on tier assignments and budget caps, executing Apollo API calls or falling back to mock datasets.
- **1.4 Email Verification & Deliverability Filtering:** Validates email addresses using MillionVerifier (or fallback rules) to ensure strict filtering before outreach sequences begin.
- **1.5 CRM Upsert & Lead Routing:** Routes evaluated records to Zoho CRM via batched upsert operations using duplicate-checking keys, handling authentication token exchanges and preview state branches.
- **1.6 Outbound Campaign Synchronization & Team Reporting:** Dispatches qualified leads into targeted Smartlead campaigns, generates comprehensive operational summaries, and routes telemetry alerts to team messaging channels (Slack, Teams, or Google Chat).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration Initialization
- **Overview:** Initializes the global configuration schema and ingests raw prospect payloads from either external Clay API requests or manual test runs.
- **Nodes Involved:** `A Clay row arrives`, `Run the demo prospects`, `load the demo prospects`, `config`.
- **Node Details:**
  - **A Clay row arrives**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook trigger)
    - *Configuration Choices:* Listens for incoming HTTP POST requests on path `clay-prospect` with response mode set to `onReceived`.
    - *Key Expressions/Variables:* Uses webhook ID `clay-prospect`.
    - *Input/Output:* Output connects to `config`.
    - *Edge Cases/Failures:* Incorrect payload formatting or network timeout from Clay.
  - **Run the demo prospects**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual trigger)
    - *Configuration Choices:* Executes workflow on demand without external webhook payloads.
    - *Input/Output:* Output connects to `load the demo prospects`.
    - *Edge Cases/Failures:* None.
  - **load the demo prospects**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript processing)
    - *Configuration Choices:* Generates 12 invented B2B SaaS prospect records simulating Clay HTTP API payloads, including demo enrichment and verification metadata (`_demo`).
    - *Key Expressions/Variables:* Iterates over predefined array and wraps outputs in `{ json: { body: row, _source: 'demo prospects (invented)' } }`.
    - *Input/Output:* Input from manual trigger; output connects to `config`.
    - *Edge Cases/Failures:* Array parsing errors if modified incorrectly.
  - **config**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Configuration repository)
    - *Configuration Choices:* Houses all operational variables, safety switches (`TEST_RUN`), mapping profiles (`CLAY_FIELDS`), scoring matrices (`SCORING`, `DISQUALIFY`, `TIERS`), enrichment budgets, Zoho CRM credentials, Smartlead campaign IDs, and notification endpoints.
    - *Key Expressions/Variables:* Evaluates dynamic flags like `MODE`, `ZOHO_AUTH_READY`, and `SETTINGS`.
    - *Input/Output:* Input from webhooks or demo loaders; output connects to `read the prospects`.
    - *Edge Cases/Failures:* Missing required keys or misconfigured base URLs.

#### 2.2 Prospect Parsing, Scoring, & Tiering
- **Overview:** Normalizes raw attribute names, derives seniorities and buyer personas, scores prospects against rule tables, and detects run-level duplicates.
- **Nodes Involved:** `read the prospects`, `score and tier`, `plan the enrichment`.
- **Node Details:**
  - **read the prospects**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data normalization)
    - *Configuration Choices:* Maps dynamic field aliases (`CLAY_FIELDS`), cleans domains, validates email patterns, and flags unusable rows lacking names or company details.
    - *Key Expressions/Variables:* Uses helper functions `pick()`, `cleanDomain()`, `cleanEmail()`, `toList()`, `toNumber()`, and `seniorityFrom()`.
    - *Input/Output:* Input from `config`; output connects to `score and tier`.
    - *Edge Cases/Failures:* Malformed email structures or unexpected null values in source payloads.
  - **score and tier**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Scoring engine)
    - *Configuration Choices:* Evaluates scoring rules (`SCORING`), checks disqualifiers (`DISQUALIFY`), assigns tiers (`TIERS`), determines personas, and detects duplicate entries based on email, LinkedIn URL, or name-domain combinations.
    - *Key Expressions/Variables:* Evaluates rule criteria using `matches()`, `personaFor()`, and `actionFor()`.
    - *Input/Output:* Input from `read the prospects`; output connects to `plan the enrichment`.
    - *Edge Cases/Failures:* Division by zero or type coercion errors during score clamping.
  - **plan the enrichment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Budget allocation)
    - *Configuration Choices:* Filters prospects to those in enrichment tiers (`ENRICH_TIERS`) missing an email, sorts them by score descending, and enforces the `ENRICH_BUDGET` cap.
    - *Key Expressions/Variables:* Calculates `spent` credits and constructs `_enrich_body` payloads.
    - *Input/Output:* Input from `score and tier`; output connects to `worth a credit?`.
    - *Edge Cases/Failures:* Exceeding budgetary credit limits.

#### 2.3 Credit-Gated Enrichment
- **Overview:** Executes paid person-matching API calls for funded prospects or substitutes mock data when API keys are absent.
- **Nodes Involved:** `worth a credit?`, `enrichment configured?`, `not worth a credit`, `Apollo - enrich the contact`, `use the demo enrichment`, `read the enrichment`, `one list of prospects`.
- **Node Details:**
  - **worth a credit?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $json._needs_enrich }}`.
    - *Input/Output:* Input from `plan the enrichment`; outputs connect to `enrichment configured?` (true) and `not worth a credit` (false).
  - **enrichment configured?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $('config').first().json.enrich_enabled }}`.
    - *Input/Output:* Input from `worth a credit?`; outputs connect to `Apollo - enrich the contact` (true) and `use the demo enrichment` (false).
  - **not worth a credit**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Pass-through processor)
    - *Configuration Choices:* Appends `_enriched: false` to unfunded records.
    - *Input/Output:* Input from `worth a credit?` (false branch); output connects to `one list of prospects` (input 1).
  - **Apollo - enrich the contact**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* Sends POST requests to `ENRICH_URL` with `x-api-key` headers and JSON request bodies. Error handling set to `continueRegularOutput`.
    - *Key Expressions/Variables:* Uses `{{ $('config').first().json.enrich_url }}`, `{{ $('config').first().json.enrich_api_key }}`, and `{{ JSON.stringify($json._enrich_body) }}`.
    - *Input/Output:* Input from `enrichment configured?` (true branch); output connects to `read the enrichment`.
    - *Edge Cases/Failures:* HTTP rate limits, invalid API keys, network timeouts (30s).
  - **use the demo enrichment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Mock enrichment)
    - *Configuration Choices:* Reads embedded `_demo` test properties to populate emails and phone numbers.
    - *Input/Output:* Input from `enrichment configured?` (false branch); output connects to `one list of prospects` (input 0).
  - **read the enrichment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Response parser)
    - *Configuration Choices:* Maps Apollo API match responses back onto prospect objects, validating locked or missing email statuses.
    - *Input/Output:* Input from `Apollo - enrich the contact`; output connects to `one list of prospects` (input 0).
  - **one list of prospects**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data merger)
    - *Configuration Choices:* Appends enriched and skipped prospect streams into a unified list.
    - *Input/Output:* Inputs from `read the enrichment` and `not worth a credit`; output connects to `who needs verifying`.

#### 2.4 Email Verification & Deliverability Filtering
- **Overview:** Validates email deliverability status via external APIs or fallback logic, ensuring invalid addresses fail closed.
- **Nodes Involved:** `who needs verifying`, `verify with a key?`, `Verifier - check the address`, `verification without a key`, `apply the verification`, `one list of verdicts`.
- **Node Details:**
  - **who needs verifying**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Verifier eligibility checker)
    - *Configuration Choices:* Flags A/B tier prospects containing emails that are not duplicates (`_needs_verify`).
    - *Input/Output:* Input from `one list of prospects`; output connects to `verify with a key?`.
  - **verify with a key?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $json._needs_verify && $('config').first().json.verifier_enabled }}`.
    - *Input/Output:* Input from `who needs verifying`; outputs connect to `Verifier - check the address` (true) and `verification without a key` (false).
  - **Verifier - check the address**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* Sends GET requests to MillionVerifier API endpoint with query parameters for API key, email, and timeout (10s). Error handling set to `continueRegularOutput`.
    - *Key Expressions/Variables:* Uses `{{ $('config').first().json.verifier_url }}` and `{{ $json.email }}`.
    - *Input/Output:* Input from `verify with a key?` (true branch); output connects to `apply the verification`.
    - *Edge Cases/Failures:* API timeouts, invalid keys, credit exhaustion.
  - **verification without a key**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Mock verification)
    - *Configuration Choices:* Assigns demo verification statuses or marks real unverified rows as `unverified` (failing closed).
    - *Input/Output:* Input from `verify with a key?` (false branch); output connects to `one list of verdicts` (input 1).
  - **apply the verification**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Response parser)
    - *Configuration Choices:* Maps verifier results against `accept_statuses` array to determine `_verified` boolean flags.
    - *Input/Output:* Input from `Verifier - check the address`; output connects to `one list of verdicts` (input 0).
  - **one list of verdicts**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data merger)
    - *Configuration Choices:* Combines verified and unverified prospect streams.
    - *Input/Output:* Inputs from `apply the verification` and `verification without a key`; output connects to `decide what goes where`.

#### 2.5 CRM Upsert & Lead Routing
- **Overview:** Routes prospects into destination buckets, authenticates against Zoho CRM via OAuth refresh tokens, builds upsert payloads, and handles error states.
- **Nodes Involved:** `decide what goes where`, `build the Zoho upsert`, `writing to Zoho?`, `Zoho - get an access token`, `zoho-preview`, `read the access token`, `authenticated?`, `Zoho CRM - upsert the leads`, `zoho-noauth`, `zoho-answers`, `zoho-rejected`.
- **Node Details:**
  - **decide what goes where**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Routing engine)
    - *Configuration Choices:* Categorizes prospects into Zoho CRM targets, Smartlead campaign buckets, or held lists with structured reasons. Outputs a single aggregated summary payload.
    - *Input/Output:* Input from `one list of verdicts`; output connects to `build the Zoho upsert`.
  - **build the Zoho upsert**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Payload builder)
    - *Configuration Choices:* Constructs batch upsert payload structs mapping standard and custom Zoho Lead fields, configuration criteria, and duplicate check parameters.
    - *Input/Output:* Input from `decide what goes where`; output connects to `writing to Zoho?`.
  - **writing to Zoho?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $('config').first().json.zoho_enabled && $json._count > 0 }}`.
    - *Input/Output:* Input from `build the Zoho upsert`; outputs connect to `Zoho - get an access token` (true) and `zoho-preview` (false).
  - **Zoho - get an access token**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (OAuth exchange)
    - *Configuration Choices:* POSTs grant requests to Zoho accounts base exchanging refresh tokens for temporary access tokens. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `writing to Zoho?` (true branch); output connects to `read the access token`.
  - **zoho-preview** (STOP: preview - Zoho payload shown)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Preview handler)
    - *Configuration Choices:* Outputs structured preview payload explaining why Zoho was not written (test mode or blank credentials).
    - *Input/Output:* Input from `writing to Zoho?` (false branch); output connects to `build the Smartlead pushes`.
  - **read the access token**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Token validator)
    - *Configuration Choices:* Parses OAuth responses and validates presence of `access_token`, generating diagnostic hints if refusals occur.
    - *Input/Output:* Input from `Zoho - get an access token`; output connects to `authenticated?`.
  - **authenticated?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $json._authed }}`.
    - *Input/Output:* Input from `read the access token`; outputs connect to `Zoho CRM - upsert the leads` (true) and `zoho-noauth` (false).
  - **Zoho CRM - upsert the leads**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* Posts batch lead upserts to Zoho CRM v8 API using Bearer token authorization. Error handling set to `continueErrorOutput`.
    - *Input/Output:* Input from `authenticated?` (true branch); outputs connect to `zoho-answers` (success) and `zoho-rejected` (error).
  - **zoho-noauth** (STOP: could not authenticate with Zoho)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Error handler)
    - *Configuration Choices:* Returns authentication failure diagnostics.
    - *Input/Output:* Input from `authenticated?` (false branch); output connects to `build the Smartlead pushes`.
  - **zoho-answers**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Response parser)
    - *Configuration Choices:* Evaluates individual record upsert codes (`SUCCESS`, insert vs. update actions, refusals).
    - *Input/Output:* Input from `Zoho CRM - upsert the leads` (success branch); output connects to `build the Smartlead pushes`.
  - **zoho-rejected** (STOP: Zoho rejected the write)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Error handler)
    - *Configuration Choices:* Formats Zoho API error responses.
    - *Input/Output:* Input from `Zoho CRM - upsert the leads` (error branch); output connects to `build the Smartlead pushes`.

#### 2.6 Outbound Campaign Synchronization & Team Reporting
- **Overview:** Pushes verified leads into Smartlead campaigns with custom metadata, generates run summaries, and posts telemetry notifications.
- **Nodes Involved:** `build the Smartlead pushes`, `pushing to Smartlead?`, `Smartlead - add leads to the campaign`, `sl-preview`, `read Smartlead's answer`, `after the campaign step`, `build the run summary`, `posting the summary?`, `Notify - post the run summary`, `summary-none`, `summary-done`.
- **Node Details:**
  - **build the Smartlead pushes**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Payload builder)
    - *Configuration Choices:* Batches leads into chunked arrays per tier campaign, respecting test redirection caps (`TEST_CAMPAIGN_ID`, `TEST_CAP`) and global block/unsubscribe rules.
    - *Input/Output:* Inputs from `read Zoho's answers`, `zoho-preview`, `zoho-noauth`, and `zoho-rejected`; output connects to `pushing to Smartlead?`.
  - **pushing to Smartlead?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $('config').first().json.smartlead_enabled && !$json._none && !$json._placeholder }}`.
    - *Input/Output:* Input from `build the Smartlead pushes`; outputs connect to `Smartlead - add leads to the campaign` (true) and `sl-preview` (false).
  - **Smartlead - add leads to the campaign**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration Choices:* POSTs lead batches to Smartlead API endpoint with query parameter API keys. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `pushing to Smartlead?` (true branch); output connects to `read Smartlead's answer`.
  - **sl-preview** (STOP: preview - Smartlead payload shown)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Preview handler)
    - *Configuration Choices:* Outputs structured preview payload detailing payloads that would be pushed.
    - *Input/Output:* Input from `pushing to Smartlead?` (false branch); output connects to `after the campaign step` (input 1).
  - **read Smartlead's answer**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Response parser)
    - *Configuration Choices:* Parses API responses for added counts, skipped duplicates, and error messages.
    - *Input/Output:* Input from `Smartlead - add leads to the campaign`; output connects to `after the campaign step` (input 0).
  - **after the campaign step**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data merger)
    - *Configuration Choices:* Merges live Smartlead response streams with preview streams.
    - *Input/Output:* Inputs from `read Smartlead's answer` and `sl-preview`; output connects to `build the run summary`.
  - **build the run summary**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Summary generator)
    - *Configuration Choices:* Compiles aggregate metrics (tiers, credit usage, credit savings, CRM insert/update counts, Smartlead additions, and held prospect reasons) into human-readable text.
    - *Input/Output:* Input from `after the campaign step`; output connects to `posting the summary?`.
  - **posting the summary?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional router)
    - *Configuration Choices:* Evaluates `{{ $('config').first().json.notify_enabled }}`.
    - *Input/Output:* Input from `build the run summary`; outputs connect to `Notify - post the run summary` (true) and `summary-none` (false).
  - **Notify - post the run summary**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Webhook integration)
    - *Configuration Choices:* POSTs JSON notification payloads to `NOTIFY_WEBHOOK_URL`. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `posting the summary?` (true branch); output connects to `summary-done`.
  - **summary-none** (STOP: summary not posted)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Preview handler)
    - *Configuration Choices:* Returns summary payload locally when webhook URLs are unconfigured or mode is preview.
    - *Input/Output:* Input from `posting the summary?` (false branch); output terminal.
  - **summary-done** (STOP: summary posted)
    - *Type and Technical Role:* `n8n-nodes-base.code` (Final status handler)
    - *Configuration Choices:* Verifies team channel webhook delivery status.
    - *Input/Output:* Input from `Notify - post the run summary`; output terminal.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A Clay row arrives | webhook | Ingests prospect rows via HTTP POST | None | config | Score Clay prospects, enrich only the qualified and route them to Zoho CRM and Smartlead |
| Run the demo prospects | manualTrigger | Initiates manual workflow execution | None | load the demo prospects | Score Clay prospects, enrich only the qualified and route them to Zoho CRM and Smartlead |
| load the demo prospects | code | Generates invented test prospect records | Run the demo prospects | config | Score Clay prospects, enrich only the qualified and route them to Zoho CRM and Smartlead |
| config | code | Stores global configuration parameters | A Clay row arrives, load the demo prospects | read the prospects | Score Clay prospects, enrich only the qualified and route them to Zoho CRM and Smartlead |
| read the prospects | code | Normalizes and validates incoming rows | config | score and tier | 1. Score first, on what you already have |
| score and tier | code | Scores leads, assigns tiers, detects duplicates | read the prospects | plan the enrichment | 1. Score first, on what you already have |
| plan the enrichment | code | Allocates enrichment budget based on tier and score | score and tier | worth a credit? | 1. Score first, on what you already have |
| worth a credit? | if | Determines if a prospect requires enrichment | plan the enrichment | enrichment configured?, not worth a credit | 2. Enrich only what was funded |
| enrichment configured? | if | Checks if Apollo API key is configured | worth a credit? | Apollo - enrich the contact, use the demo enrichment | 2. Enrich only what was funded |
| not worth a credit | code | Marks unfunded records as unenriched | worth a credit? | one list of prospects | 2. Enrich only what was funded |
| Apollo - enrich the contact | httpRequest | Executes paid person-match enrichment call | enrichment configured? | read the enrichment | 2. Enrich only what was funded |
| use the demo enrichment | code | Returns mock enrichment metadata for tests | enrichment configured? | one list of prospects | 2. Enrich only what was funded |
| read the enrichment | code | Parses Apollo API match responses | Apollo - enrich the contact | one list of prospects | 2. Enrich only what was funded |
| one list of prospects | merge | Merges enriched and skipped prospect streams | read the enrichment, not worth a credit | who needs verifying | 2. Enrich only what was funded |
| who needs verifying | code | Identifies prospects requiring email verification | one list of prospects | verify with a key? | 3. Verify before anything is sequenced |
| verify with a key? | if | Checks if MillionVerifier API key is configured | who needs verifying | Verifier - check the address, verification without a key | 3. Verify before anything is sequenced |
| Verifier - check the address | httpRequest | Queries MillionVerifier API for deliverability | verify with a key? | apply the verification | 3. Verify before anything is sequenced |
| verification without a key | code | Assigns mock or unverified fallback statuses | verify with a key? | one list of verdicts | 3. Verify before anything is sequenced |
| apply the verification | code | Evaluates deliverability verdicts | Verifier - check the address | one list of verdicts | 3. Verify before anything is sequenced |
| one list of verdicts | merge | Merges verified and unverified prospect lists | apply the verification, verification without a key | decide what goes where | 3. Verify before anything is sequenced |
| decide what goes where | code | Routes prospects to CRM, Smartlead, or holds | one list of verdicts | build the Zoho upsert | 4. One write to Zoho CRM |
| build the Zoho upsert | code | Constructs batch Zoho Lead upsert payload | decide what goes where | writing to Zoho? | 4. One write to Zoho CRM |
| writing to Zoho? | if | Checks if Zoho integration is enabled and active | build the Zoho upsert | Zoho - get an access token, zoho-preview | 4. One write to Zoho CRM, Nothing is written until you say so |
| Zoho - get an access token | httpRequest | Exchanges OAuth refresh token for access token | writing to Zoho? | read the access token | 4. One write to Zoho CRM |
| zoho-preview | code | Outputs preview payload when Zoho is bypassed | writing to Zoho? | build the Smartlead pushes | 4. One write to Zoho CRM, Nothing is written until you say so |
| read the access token | code | Validates OAuth access token response | Zoho - get an access token | authenticated? | 4. One write to Zoho CRM |
| authenticated? | if | Confirms successful OAuth authentication | read the access token | Zoho CRM - upsert the leads, zoho-noauth | 4. One write to Zoho CRM |
| Zoho CRM - upsert the leads | httpRequest | Upserts lead batches into Zoho CRM | authenticated? | zoho-answers, zoho-rejected | 4. One write to Zoho CRM |
| zoho-noauth | code | Handles Zoho authentication failure diagnostics | authenticated? | build the Smartlead pushes | 4. One write to Zoho CRM |
| zoho-answers | code | Parses individual Zoho upsert result codes | Zoho CRM - upsert the leads | build the Smartlead pushes | 4. One write to Zoho CRM |
| zoho-rejected | code | Formats Zoho API batch refusal responses | Zoho CRM - upsert the leads | build the Smartlead pushes | 4. One write to Zoho CRM |
| build the Smartlead pushes | code | Batches qualified leads into campaign payloads | zoho-answers, zoho-rejected, zoho-preview, zoho-noauth | pushing to Smartlead? | 5. Sequence, then report |
| pushing to Smartlead? | if | Checks Smartlead integration enablement | build the Smartlead pushes | Smartlead - add leads to the campaign, sl-preview | 5. Sequence, then report, Nothing is written until you say so |
| Smartlead - add leads to the campaign | httpRequest | Pushes lead batches to Smartlead campaigns | pushing to Smartlead? | read Smartlead's answer | 5. Sequence, then report |
| sl-preview | code | Outputs preview payload when Smartlead is bypassed | pushing to Smartlead? | after the campaign step | 5. Sequence, then report, Nothing is written until you say so |
| read Smartlead's answer | code | Parses Smartlead campaign addition results | Smartlead - add leads to the campaign | after the campaign step | 5. Sequence, then report |
| after the campaign step | merge | Merges live Smartlead responses with previews | read Smartlead's answer, sl-preview | build the run summary | 5. Sequence, then report |
| build the run summary | code | Compiles aggregate execution metrics and text | after the campaign step | posting the summary? | 5. Sequence, then report |
| posting the summary? | if | Checks team notification webhook configuration | build the run summary | Notify - post the run summary, summary-none | 5. Sequence, then report |
| Notify - post the run summary | httpRequest | Posts run summary to team incoming webhook | posting the summary? | summary-done | 5. Sequence, then report |
| summary-none | code | Outputs summary locally in preview mode | posting the summary? | None | 5. Sequence, then report |
| summary-done | code | Verifies team webhook post delivery status | Notify - post the run summary | None | 5. Sequence, then report |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Trigger Setup:**
   - Create a **Webhook** node (`n8n-nodes-base.webhook`). Name it `A Clay row arrives`. Set HTTP Method to `POST`, Path to `clay-prospect`, and Response Mode to `On Received`.
   - Create a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run the demo prospects`.

2. **Demo Load & Global Configuration:**
   - Connect `Run the demo prospects` to a **Code** node (`n8n-nodes-base.code`) named `load the demo prospects`. Paste the 12-item demo prospect array logic mapping into the JavaScript parameter.
   - Create a central **Code** node named `config`. Connect both `A Clay row arrives` and `load the demo prospects` into it. Populate the configuration script containing `TEST_RUN = true`, `CLAY_FIELDS`, `SCORING`, `DISQUALIFY`, `TIERS`, `SENIORITY_FROM_TITLE`, `PERSONAS`, `RECOMMENDED_ACTION`, enrichment parameters, Zoho credentials, Smartlead campaign mappings, and derivation logic.

3. **Parsing & Scoring Pipeline:**
   - Add a **Code** node named `read the prospects`, connected from `config`. Implement the column aliasing, domain cleaning, and usability flagging logic.
   - Add a **Code** node named `score and tier`, connected from `read the prospects`. Implement rule evaluation, score clamping, persona assignment, and run-level duplicate detection.
   - Add a **Code** node named `plan the enrichment`, connected from `score and tier`. Implement tier filtering, scoring sort order, and budget enforcement (`ENRICH_BUDGET`).

4. **Enrichment Branching:**
   - Add an **If** node named `worth a credit?`, connected from `plan the enrichment`. Set condition to evaluate `{{ $json._needs_enrich }} === true`.
   - On the `false` branch, add a **Code** node named `not worth a credit` to mark records as unenriched.
   - On the `true` branch, add an **If** node named `enrichment configured?`. Evaluate `{{ $('config').first().json.enrich_enabled }} === true`.
   - On the `false` branch of `enrichment configured?`, add a **Code** node named `use the demo enrichment`.
   - On the `true` branch, add an **HTTP Request** node named `Apollo - enrich the contact`. Set Method to `POST`, URL to `{{ $('config').first().json.enrich_url }}`, header `x-api-key` to `{{ $('config').first().json.enrich_api_key }}`, and body to JSON stringified `_enrich_body`. Enable **Continue On Fail** (`continueRegularOutput`).
   - Connect `Apollo - enrich the contact` to a **Code** node named `read the enrichment`.
   - Merge `use the demo enrichment`, `read the enrichment`, and `not worth a credit` using a **Merge** node (`n8n-nodes-base.merge`, mode: `Append`) named `one list of prospects`.

5. **Verification Pipeline:**
   - Connect `one list of prospects` to a **Code** node named `who needs verifying`.
   - Connect to an **If** node named `verify with a key?`. Evaluate `{{ $json._needs_verify && $('config').first().json.verifier_enabled }} === true`.
   - On the `false` branch, add a **Code** node named `verification without a key`.
   - On the `true` branch, add an **HTTP Request** node named `Verifier - check the address`. Set Method to `GET`, URL to `{{ $('config').first().json.verifier_url }}`, and query parameters for `api` and `email`. Enable **Continue On Fail** (`continueRegularOutput`).
   - Connect `Verifier - check the address` to a **Code** node named `apply the verification`.
   - Merge `apply the verification` and `verification without a key` using a **Merge** node named `one list of verdicts` (mode: `Append`).

6. **CRM Routing & Upsert:**
   - Connect `one list of verdicts` to a **Code** node named `decide what goes where`.
   - Connect to a **Code** node named `build the Zoho upsert`.
   - Connect to an **If** node named `writing to Zoho?`. Evaluate `{{ $('config').first().json.zoho_enabled && $json._count > 0 }} === true`.
   - On the `false` branch, add a **Code** node named `zoho-preview` (`STOP: preview - Zoho payload shown`).
   - On the `true` branch, add an **HTTP Request** node named `Zoho - get an access token`. POST to `{{ $('config').first().json.zoho_accounts_base + '/oauth/v2/token' }}`, setting query parameters for `refresh_token`, `client_id`, `client_secret`, and `grant_type=refresh_token`. Enable **Continue On Fail** (`continueRegularOutput`).
   - Connect to a **Code** node named `read the access token`.
   - Connect to an **If** node named `authenticated?`. Evaluate `{{ $json._authed }} === true`.
   - On the `false` branch, add a **Code** node named `zoho-noauth` (`STOP: could not authenticate with Zoho`).
   - On the `true` branch, add an **HTTP Request** node named `Zoho CRM - upsert the leads`. POST to `{{ $json._zoho.url }}`, setting Authorization header `Zoho-oauthtoken <access_token>` and JSON body. Enable **Continue On Fail** (`continueErrorOutput`).
   - Connect success output to a **Code** node named `read Zoho's answers`. Connect error output to a **Code** node named `zoho-rejected` (`STOP: Zoho rejected the write`).

7. **Smartlead Synchronization:**
   - Connect `zoho-answers`, `zoho-rejected`, `zoho-preview`, and `zoho-noauth` into a **Code** node named `build the Smartlead pushes`.
   - Connect to an **If** node named `pushing to Smartlead?`. Evaluate `{{ $('config').first().json.smartlead_enabled && !$json._none && !$json._placeholder }} === true`.
   - On the `false` branch, add a **Code** node named `sl-preview` (`STOP: preview - Smartlead payload shown`).
   - On the `true` branch, add an **HTTP Request** node named `Smartlead - add leads to the campaign`. POST to `{{ $json._url }}`, setting query parameter `api_key` and JSON body. Enable **Continue On Fail** (`continueRegularOutput`).
   - Connect to a **Code** node named `read Smartlead's answer`.
   - Merge `read Smartlead's answer` and `sl-preview` using a **Merge** node named `after the campaign step` (mode: `Append`).

8. **Run Summary & Notifications:**
   - Connect `after the campaign step` to a **Code** node named `build the run summary`. Set node execution options to `Execute Once`.
   - Connect to an **If** node named `posting the summary?`. Evaluate `{{ $('config').first().json.notify_enabled }} === true`.
   - On the `false` branch, add a **Code** node named `summary-none` (`STOP: summary not posted`).
   - On the `true` branch, add an **HTTP Request** node named `Notify - post the run summary`. POST to `{{ $('config').first().json.notify_webhook_url }}`, sending JSON body `_notify_body`. Enable **Continue On Fail** (`continueRegularOutput`).
   - Connect to a **Code** node named `summary-done` (`STOP: summary posted`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Companion Workflow Integration** | Import the companion replies workflow for inbound classification and daily call list generation. |
| **Safety Switch Architecture** | Ships with `TEST_RUN = true`. All Zoho and Smartlead payloads are built in full and routed to preview STOP nodes until credentials and live settings are applied. |
| **Credit Optimization Strategy** | Pre-enrichment scoring and tiering prevents wasting paid credits on C-tier or rejected prospects. |