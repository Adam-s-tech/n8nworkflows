Validate procurement records and learn error patterns with Gemini and Data Tables

https://n8nworkflows.xyz/workflows/validate-procurement-records-and-learn-error-patterns-with-gemini-and-data-tables-20038


# Validate procurement records and learn error patterns with Gemini and Data Tables

### 1. Workflow Overview

The **AI-Based Procurement Data Error Detection** workflow provides automated validation, normalization, and semantic error-checking for incoming procurement line-item records. Target use cases include automated accounts payable (AP) pre-screening, purchase order (PO) line validation, and continuous learning of recurring invoicing anomalies. 

The processing logic is divided into four functional blocks:
- **1.1 Input Reception & Deterministic Validation:** Ingests raw HTTP POST payloads, sets configuration variables, applies deterministic rules (required fields, math checking, unit alias normalization), and creates a unique signature key.
- **1.2 Historical Pattern Matching & AI Review:** Queries an n8n Data Table (`procurement_error_patterns`) using the pattern key. Trusted historical patterns bypass AI evaluation; otherwise, payloads are sent to Google Gemini for contextual and semantic anomaly analysis.
- **1.3 Unified Decision & Error Learning:** Aggregates deterministic errors, historical matches, and LLM findings into a single decision. If errors or anomalies are found, execution updates occurrence counts and confidence metrics in the Data Table via upsert operations.
- **1.4 Response Generation:** Formats the final accept/reject JSON payload containing validation status, confidence scores, detected errors, and correction guidance, then responds directly to the webhook caller.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Deterministic Validation
This block receives raw procurement payloads via webhook, injects validation thresholds, normalizes text and unit values, and performs rigorous programmatic math and schema validations.

- **Receive Procurement Record**
  - **Type & Role:** `n8n-nodes-base.webhook` (Trigger). Listens for HTTP POST requests at `/procurement-data-quality-check` to initiate the workflow.
  - **Configuration:** Method: `POST`, Response Mode: `responseNode`.
  - **Inputs / Outputs:** Input: None (Trigger) | Output: Sends raw HTTP body to `Configure Validation Rules`.
  - **Failure Modes:** Timeout or network issues from the calling API client; handled by standard webhook error responses.

- **Configure Validation Rules**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Defines runtime constants and rule configurations.
  - **Configuration:** Assigns operational thresholds and rules (`cfg_default_currency`, `cfg_tax_tolerance`, `cfg_subtotal_tolerance`, `cfg_pattern_confidence_threshold`, `cfg_recurring_error_threshold`, `cfg_ai_confidence_threshold`, `cfg_enable_pattern_learning`, `cfg_required_fields`).
  - **Inputs / Outputs:** Input: `Receive Procurement Record` | Output: Sends parameters and original body to `Normalize and Validate Procurement Record`.

- **Normalize and Validate Procurement Record**
  - **Type & Role:** `n8n-nodes-base.code` (Python execution). Normalizes raw procurement fields, resolves unit aliases (e.g., `PCS` to `EA`), evaluates mandatory fields, checks numerical bounds, recalculates subtotals and tax amounts, and generates a composite `pattern_key`.
  - **Key Expressions / Variables:** Uses Python helper functions (`text`, `num`, `round2`, `to_bool`) and outputs `record`, `config`, `validation` metadata, and `pattern_key`.
  - **Inputs / Outputs:** Input: `Configure Validation Rules` | Output: Branches to `Find Known Error Pattern` and `Merge Record With Pattern`.
  - **Failure Modes:** Python exceptions due to unparseable non-string values or malformed JSON payloads.

---

#### Block 1.2: Historical Pattern Matching & AI Review
This block queries the database for known error patterns matching the signature key, assesses trust levels, and invokes Google Gemini for advanced contextual evaluation when historical data is absent or unverified.

- **Find Known Error Pattern**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database lookup). Queries the `procurement_error_patterns` Data Table for matching error signatures.
  - **Configuration:** Operation: `get`, Filter: `pattern_key` equals `{{ $json.pattern_key }}`, Limit: 1, `alwaysOutputData: true`.
  - **Inputs / Outputs:** Input: `Normalize and Validate Procurement Record` | Output: Passes matching table records to `Merge Record With Pattern`.
  - **Failure Modes:** Database connection errors or missing Data Table definitions.

- **Merge Record With Pattern**
  - **Type & Role:** `n8n-nodes-base.merge` (Data combination). Merges the initial validation output with historical pattern search results.
  - **Configuration:** Mode: `combine`, Combine By: `combineByPosition`.
  - **Inputs / Outputs:** Input: `Normalize and Validate Procurement Record` and `Find Known Error Pattern` | Output: Sends combined dataset to `Is Trusted Pattern Known`.

- **Is Trusted Pattern Known**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional logic). Determines whether a retrieved database record meets frequency and confidence criteria to be treated as a trusted recurring error.
  - **Key Expressions:** Evaluates occurrence counts against `cfg_recurring_error_threshold` and confidence against `cfg_pattern_confidence_threshold`.
  - **Inputs / Outputs:** Input: `Merge Record With Pattern` | Output (True): `Reuse Trusted Pattern Correction`. Output (False): `Analyze Procurement Anomalies` and `Merge Record With AI Review`.

- **Reuse Trusted Pattern Correction**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Flags the record as processed via historical pattern matching and injects stored correction attributes.
  - **Inputs / Outputs:** Input: `Is Trusted Pattern Known` (True) | Output: `Build Unified Error Decision`.

- **Analyze Procurement Anomalies**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.chainLlm` (AI processing node). Executes a structured prompt against Google Gemini to detect semantic, contextual, or coding inconsistencies.
  - **Configuration:** Prompt type: `define`, expects strict JSON output matching predefined anomaly structures.
  - **Inputs / Outputs:** Input: `Is Trusted Pattern Known` (False) | Output: Sends AI string output to `Merge Record With AI Review`.
  - **Failure Modes:** Rate limiting, invalid API credentials, or non-JSON model outputs.

- **Google Gemini Chat Model - procurement**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (LLM Provider). Configures the underlying Google Gemini model (`models/gemini-3.5-flash-lite`).
  - **Credentials:** Uses Google Gemini (AIStudio) credentials.
  - **Inputs / Outputs:** Input: Connected to `Analyze Procurement Anomalies` via AI language model connection.

- **Merge Record With AI Review**
  - **Type & Role:** `n8n-nodes-base.merge` (Data combination). Synchronizes the original transaction payload with the AI analysis response.
  - **Configuration:** Mode: `combine`, Combine By: `combineByPosition`.
  - **Inputs / Outputs:** Input: `Is Trusted Pattern Known` and `Analyze Procurement Anomalies` | Output: `Build Unified Error Decision`.

---

#### Block 1.3: Unified Decision & Error Learning
This block consolidates validation flags, historical match data, and AI insights into a single final decision, writes newly encountered error signatures back to the data table, and prepares rejection or acceptance payloads.

- **Build Unified Error Decision**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Evaluates deterministic errors, historical matches, and AI confidence thresholds to establish unified flags (`has_detected_error`, `review_source`, `decision_confidence`, `detected_issues`).
  - **Inputs / Outputs:** Input: `Merge Record With AI Review` or `Reuse Trusted Pattern Correction` | Output: `Are Errors Detected`.

- **Are Errors Detected**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional logic). Routes valid records to acceptance handling and invalid records to the learning and correction path.
  - **Key Expressions:** Evaluates `{{ $json.has_detected_error }}` equals `true`.
  - **Inputs / Outputs:** Input: `Build Unified Error Decision` | Output (True): `Prepare Learned Error Pattern`. Output (False): `Build Valid Record Response`.

- **Prepare Learned Error Pattern**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Formats payload properties for database upsert, calculating updated occurrence counts, confidence metrics, and serialized issue summaries.
  - **Inputs / Outputs:** Input: `Are Errors Detected` (True) | Output: `Learn Recurring Error Pattern`.

- **Learn Recurring Error Pattern**
  - **Type & Role:** `n8n-nodes-base.dataTable` (Database upsert). Upserts the error pattern record into the `procurement_error_patterns` Data Table based on `pattern_key`.
  - **Configuration:** Operation: `upsert`, matching on `pattern_key`.
  - **Inputs / Outputs:** Input: `Prepare Learned Error Pattern` | Output: `Notify User With Correction`.
  - **Failure Modes:** Database write failures, schema mismatches.

- **Notify User With Correction**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Builds the final rejection response payload containing error summaries, review sources, and correction advice.
  - **Inputs / Outputs:** Input: `Learn Recurring Error Pattern` | Output: `Return Validation Result`.

- **Build Valid Record Response**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation). Formats the success response payload confirming that the procurement record passed all verification checks.
  - **Inputs / Outputs:** Input: `Are Errors Detected` (False) | Output: `Return Validation Result`.

---

#### Block 1.4: Response Generation
This block finalizes HTTP communications back to the requesting client system.

- **Return Validation Result**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` (Webhook response). Sends the final JSON payload back to the HTTP client.
  - **Configuration:** Respond with: `json`, Response body: `{{ $json }}`.
  - **Inputs / Outputs:** Input: `Notify User With Correction` or `Build Valid Record Response` | Output: None (Terminal node).
  - **Failure Modes:** Disconnected client connection before response transmission.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Workflow Overview | `n8n-nodes-base.stickyNote` | Documentation note | None | None | # AI-Based Procurement Data Error Detection<br><br>## How it works<br>This workflow receives procurement records through a POST webhook, normalizes and validates required fields, quantities, pricing, subtotal, tax, currency, and unit values, then checks an n8n Data Table for a previously learned error signature. Trusted recurring patterns bypass AI. New or untrusted combinations are reviewed by a Basic LLM Chain using Google Gemini for semantic anomalies such as category, coding, supplier-item, or unit inconsistencies. Detected issues are returned immediately to the calling system and saved as reusable patterns for future prevention.<br><br>## Setup steps<br>1. Create an n8n Data Table named `procurement_error_patterns` with the columns listed in the Learning section.<br>2. Select your Google Gemini credential in the Gemini Chat Model node.<br>3. Review the workflow configuration in n8n Variables, including AI confidence, recurring-error, pattern-confidence, tax-tolerance, subtotal-tolerance, default currency, default tax rate, required fields, and pattern-learning settings.<br>4. Send procurement records as JSON to the webhook. Recommended fields: supplier, item_code, description, category, quantity, unit, unit_price, subtotal, tax_rate, tax_amount, and currency.<br>5. Test valid, mathematical-error, semantic-error, and recurring-pattern cases before activation. |
| Learning Section | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Learning & Validation Response<br><br>Upserts detected error patterns into the n8n Data Table for future reuse. Invalid records return detected issues and correction guidance, while valid records return an accepted response. Both branches finish through the same webhook response node. |
| Receive Procurement Record | `n8n-nodes-base.webhook` | Trigger webhook | None | Configure Validation Rules | ## Input & Deterministic Validation<br><br>Receives procurement data, applies configurable validation rules, normalizes fields, checks pricing and tax consistency, and generates a reusable pattern signature. The resulting signature is checked against previously learned procurement error patterns. |
| Configure Validation Rules | `n8n-nodes-base.set` | Define validation variables | Receive Procurement Record | Normalize and Validate Procurement Record | ## Input & Deterministic Validation<br><br>Receives procurement data, applies configurable validation rules, normalizes fields, checks pricing and tax consistency, and generates a reusable pattern signature. The resulting signature is checked against previously learned procurement error patterns. |
| Normalize and Validate Procurement Record | `n8n-nodes-base.code` | Python data validation & normalization | Configure Validation Rules | Find Known Error Pattern, Merge Record With Pattern | ## Input & Deterministic Validation<br><br>Receives procurement data, applies configurable validation rules, normalizes fields, checks pricing and tax consistency, and generates a reusable pattern signature. The resulting signature is checked against previously learned procurement error patterns. |
| Find Known Error Pattern | `n8n-nodes-base.dataTable` | Query error patterns database | Normalize and Validate Procurement Record | Merge Record With Pattern | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Merge Record With Pattern | `n8n-nodes-base.merge` | Combine validation and DB lookup | Normalize and Validate Procurement Record, Find Known Error Pattern | Is Trusted Pattern Known | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Is Trusted Pattern Known | `n8n-nodes-base.if` | Evaluate pattern trust level | Merge Record With Pattern | Reuse Trusted Pattern Correction, Analyze Procurement Anomalies, Merge Record With AI Review | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Reuse Trusted Pattern Correction | `n8n-nodes-base.set` | Assign historical correction metadata | Is Trusted Pattern Known | Build Unified Error Decision | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Analyze Procurement Anomalies | `@n8n/n8n-nodes-langchain.chainLlm` | LLM anomaly detection chain | Is Trusted Pattern Known | Merge Record With AI Review | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Merge Record With AI Review | `n8n-nodes-base.merge` | Combine payload with AI review | Analyze Procurement Anomalies, Is Trusted Pattern Known | Build Unified Error Decision | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Build Unified Error Decision | `n8n-nodes-base.set` | Consolidate decision metrics | Merge Record With AI Review, Reuse Trusted Pattern Correction | Are Errors Detected | ## Unified Decision & Learning<br><br>Combines deterministic validation, historical matches, and Gemini findings into one decision. Detected issues are normalized into a common structure. Records requiring correction are prepared with pattern details, occurrence count, confidence, source, and correction information. |
| Are Errors Detected | `n8n-nodes-base.if` | Route valid vs. invalid records | Build Unified Error Decision | Prepare Learned Error Pattern, Build Valid Record Response | ## Unified Decision & Learning<br><br>Combines deterministic validation, historical matches, and Gemini findings into one decision. Detected issues are normalized into a common structure. Records requiring correction are prepared with pattern details, occurrence count, confidence, source, and correction information. |
| Prepare Learned Error Pattern | `n8n-nodes-base.set` | Prepare upsert payload | Are Errors Detected | Learn Recurring Error Pattern | ## Unified Decision & Learning<br><br>Combines deterministic validation, historical matches, and Gemini findings into one decision. Detected issues are normalized into a common structure. Records requiring correction are prepared with pattern details, occurrence count, confidence, source, and correction information. |
| Learn Recurring Error Pattern | `n8n-nodes-base.dataTable` | Upsert error pattern in Data Table | Prepare Learned Error Pattern | Notify User With Correction | ## Learning & Validation Response<br><br>Upserts detected error patterns into the n8n Data Table for future reuse. Invalid records return detected issues and correction guidance, while valid records return an accepted response. Both branches finish through the same webhook response node. |
| Notify User With Correction | `n8n-nodes-base.set` | Format error notification response | Learn Recurring Error Pattern | Return Validation Result | ## Learning & Validation Response<br><br>Upserts detected error patterns into the n8n Data Table for future reuse. Invalid records return detected issues and correction guidance, while valid records return an accepted response. Both branches finish through the same webhook response node. |
| Build Valid Record Response | `n8n-nodes-base.set` | Format success response | Are Errors Detected | Return Validation Result | ## Learning & Validation Response<br><br>Upserts detected error patterns into the n8n Data Table for future reuse. Invalid records return detected issues and correction guidance, while valid records return an accepted response. Both branches finish through the same webhook response node. |
| Return Validation Result | `n8n-nodes-base.respondToWebhook` | Send HTTP JSON response | Notify User With Correction, Build Valid Record Response | None | ## Learning & Validation Response<br><br>Upserts detected error patterns into the n8n Data Table for future reuse. Invalid records return detected issues and correction guidance, while valid records return an accepted response. Both branches finish through the same webhook response node. |
| Google Gemini Chat Model - procurement | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Google Gemini LLM configuration | None | Analyze Procurement Anomalies | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Validation Section | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Input & Deterministic Validation<br><br>Receives procurement data, applies configurable validation rules, normalizes fields, checks pricing and tax consistency, and generates a reusable pattern signature. The resulting signature is checked against previously learned procurement error patterns. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Pattern Decision & AI Review<br><br>Combines historical pattern data with the current record. Trusted recurring patterns reuse previous corrections and skip AI. Unseen or insufficiently trusted records are reviewed by Gemini for semantic procurement anomalies such as category, coding, and unit mismatches. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Unified Decision & Learning<br><br>Combines deterministic validation, historical matches, and Gemini findings into one decision. Detected issues are normalized into a common structure. Records requiring correction are prepared with pattern details, occurrence count, confidence, source, and correction information. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the n8n Data Table
1. In your n8n instance, create a new **Data Table** named `procurement_error_patterns`.
2. Add the following columns with their respective data types:
   - `pattern_key` (String)
   - `supplier` (String)
   - `item_code` (String)
   - `error_type` (String)
   - `incorrect_value` (String)
   - `suggested_value` (String)
   - `reason` (String)
   - `occurrence_count` (Number)
   - `confidence` (Number)
   - `first_seen` (DateTime)
   - `last_seen` (DateTime)
   - `source` (String)
   - `issue_summary` (String)

#### Step 2: Set Up Credentials
1. Configure a valid **Google Gemini (Google Palm API)** credential with your API key.

#### Step 3: Build the Nodes and Connections
1. **Receive Procurement Record (`n8n-nodes-base.webhook`):**
   - Path: `procurement-data-quality-check`
   - Method: `POST`
   - Response Mode: `responseNode`
2. **Configure Validation Rules (`n8n-nodes-base.set`):**
   - Connect input from `Receive Procurement Record`.
   - Add string assignments (e.g., `cfg_default_currency` = `USD`, `cfg_required_fields` = `supplier,item_code,description,category,quantity,unit,unit_price,currency`) and numeric assignments (`cfg_tax_tolerance` = `0.01`, `cfg_subtotal_tolerance` = `0.01`, `cfg_pattern_confidence_threshold` = `0.9`, `cfg_recurring_error_threshold` = `3`, `cfg_ai_confidence_threshold` = `0.7`, `cfg_default_tax_rate` = `0`). Add boolean assignment `cfg_enable_pattern_learning` = `true`.
3. **Normalize and Validate Procurement Record (`n8n-nodes-base.code`):**
   - Set language to `pythonNative`.
   - Insert the Python parsing, unit alias translation (`EACH`/`PCS` to `EA`), math verification, and `pattern_key` generation logic provided in the JSON.
4. **Find Known Error Pattern (`n8n-nodes-base.dataTable`):**
   - Operation: `get`
   - Data Table: `procurement_error_patterns`
   - Filters: Condition where `pattern_key` equals `{{ $json.pattern_key }}`. Set `Always Output Data` to true.
5. **Merge Record With Pattern (`n8n-nodes-base.merge`):**
   - Mode: `combine`, Combine By: `combineByPosition`. Connect inputs from `Normalize and Validate Procurement Record` (Input 1) and `Find Known Error Pattern` (Input 2).
6. **Is Trusted Pattern Known (`n8n-nodes-base.if`):**
   - Condition: `{{ Boolean($json.error_type) && Number($json.occurrence_count || 0) >= Number($vars.cfg_recurring_error_threshold) && Number($json.confidence || 0) >= Number($vars.cfg_pattern_confidence_threshold) }}` equals `true`.
7. **Reuse Trusted Pattern Correction (`n8n-nodes-base.set`):**
   - Connected to the `true` output of `Is Trusted Pattern Known`. Assigns historical metadata fields (`review_source` = `historical_pattern`, `known_pattern_matched` = `true`, etc.).
8. **Google Gemini Chat Model - procurement (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`):**
   - Select the Google Gemini credential. Set model name to `models/gemini-3.5-flash-lite`.
9. **Analyze Procurement Anomalies (`@n8n/n8n-nodes-langchain.chainLlm`):**
   - Connected to the `false` output of `Is Trusted Pattern Known`. Configure prompt type as `define` with strict JSON output instructions for detecting contextual procurement anomalies. Connect the AI Language Model port to `Google Gemini Chat Model - procurement`.
10. **Merge Record With AI Review (`n8n-nodes-base.merge`):**
    - Mode: `combine`, Combine By: `combineByPosition`. Combine outputs from `Is Trusted Pattern Known` (Input 1) and `Analyze Procurement Anomalies` (Input 2).
11. **Build Unified Error Decision (`n8n-nodes-base.set`):**
    - Consolidates results from either `Reuse Trusted Pattern Correction` or `Merge Record With AI Review`. Computes unified decision flags (`has_detected_error`, `review_source`, `decision_confidence`, `detected_issues`).
12. **Are Errors Detected (`n8n-nodes-base.if`):**
    - Condition: `{{ $json.has_detected_error }}` equals `true`.
13. **Prepare Learned Error Pattern (`n8n-nodes-base.set`):**
    - Connected to the `true` branch of `Are Errors Detected`. Maps fields for database upsert (`occurrence_count` increment, timestamps, confidence).
14. **Learn Recurring Error Pattern (`n8n-nodes-base.dataTable`):**
    - Operation: `upsert`, Data Table: `procurement_error_patterns`, matching on `pattern_key`.
15. **Notify User With Correction (`n8n-nodes-base.set`):**
    - Sets failure response payload (`accepted` = `false`, `status` = `validation_failed`, maps `errors` array).
16. **Build Valid Record Response (`n8n-nodes-base.set`):**
    - Connected to the `false` branch of `Are Errors Detected`. Sets success response payload (`accepted` = `true`, `status` = `valid`).
17. **Return Validation Result (`n8n-nodes-base.respondToWebhook`):**
    - Respond with `json`, body `{{ $json }}`. Connect both `Notify User With Correction` and `Build Valid Record Response` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Data Table Setup | Requires a Data Table named `procurement_error_patterns` with 13 specific schema columns for error pattern tracking. |
| AI Model Configuration | Uses Google Gemini (`models/gemini-3.5-flash-lite`) via Google Palm API credentials for semantic anomaly detection. |