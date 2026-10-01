Verify contract extraction and flag hallucinated fields with Judgment API

https://n8nworkflows.xyz/workflows/verify-contract-extraction-and-flag-hallucinated-fields-with-judgment-api-19722


# Verify contract extraction and flag hallucinated fields with Judgment API

### 1. Workflow Overview

This workflow is designed to automate the validation of extracted contract terms against an original source text using the Judgment API. It detects hallucinations, fabricated values, or discrepancies in data extraction pipelines, routing suspicious fields to a human review queue while passing verified data directly to confirmation endpoints.

The logic is categorized into three functional blocks:
- **1.1 Input Generation & Preparation:** Triggers manually and establishes two distinct dataset paths—one containing accurate contract extraction data and one containing deliberately fabricated/altered values—then combines them into a unified execution stream.
- **1.2 AI Evaluation & Normalization:** Sends the combined records to the Judgment API to execute structured validations (`not_in_source` and `wrong_value`) across multiple extracted fields in a single API call per record, subsequently flattening and reshaping the output for programmatic evaluation.
- **1.3 Conditional Routing & Triage:** Evaluates the risk score of each extracted field against a defined threshold, appending human-readable explanations to flagged flags and sorting outcomes into a review queue or a confirmed path.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Generation & Preparation
- **Overview:** Initializes the workflow execution, creates both valid and invalid test scenarios based on a reference contract text, and merges the data streams into a single list for side-by-side verification.
- **Nodes Involved:** 
  - `When Clicking Test Workflow`
  - `Source Document`
  - `Fabricated Extraction`
  - `Two Records`

- **Node Details:**
  - **When Clicking Test Workflow**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` (Manual Trigger Node). Initiates workflow execution on demand.
    - *Configuration choices:* Default settings.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Outputs to `Source Document` and `Fabricated Extraction`.
    - *Edge cases:* None.
  - **Source Document**
    - *Type and technical role:* `n8n-nodes-base.set` (Set / Edit Fields Node). Defines the base agreement text, unique record identifier, and baseline correct extractions.
    - *Configuration choices:* Manual mode, assigns string and numeric values to specific keys (`record`, `source_text`, `counterparty`, `term_months`, `annual_fee_usd`, `notice_days`, `governing_law`).
    - *Key expressions or variables:* Static values representing an accurate extraction.
    - *Input and output connections:* Input from `When Clicking Test Workflow`; output connects to input index 0 of `Two Records`.
    - *Edge cases:* Schema mismatches if downstream nodes expect different data types.
  - **Fabricated Extraction**
    - *Type and technical role:* `n8n-nodes-base.set` (Set / Edit Fields Node). Duplicates the base scenario while introducing intentional errors (`notice_days` = 90, `governing_law` = Norway) to simulate extraction hallucinations.
    - *Configuration choices:* Manual mode, copies `source_text` from upstream nodes while overriding specific JSON properties.
    - *Key expressions or variables:* `={{ $json.source_text }}`
    - *Input and output connections:* Input from `When Clicking Test Workflow`; output connects to input index 1 of `Two Records`.
    - *Edge cases:* Fails if execution order assumes sequential input rather than simultaneous triggering.
  - **Two Records**
    - *Type and technical role:* `n8n-nodes-base.merge` (Merge Node). Combines the accurate and fabricated extractions into a single output data stream using an append strategy.
    - *Configuration choices:* Mode set to `append`, configured for 2 inputs.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Inputs from `Source Document` (index 0) and `Fabricated Extraction` (index 1); outputs to `Verify Extract`.
    - *Edge cases:* Item pairing misalignment if stream lengths differ.

---

#### 2.2 AI Evaluation & Normalization
- **Overview:** Sends each record's extraction payload and source text to the Judgment API, executing ten verification checks across five fields simultaneously, then parses and normalizes the resulting risk scores and IDs.
- **Nodes Involved:** 
  - `Verify Extract`
  - `Reshape Checks`

- **Node Details:**
  - **Verify Extract**
    - *Type and technical role:* `n8n-nodes-judgment.judgment` (Judgment Community Node). Evaluates extracted JSON fields against the raw `source_text` utilizing custom natural-language questions.
    - *Configuration choices:* Resource: `evaluation`, Operation: `evaluate`, State: `fields`. Configures 10 distinct checks (`not_in_source` and `wrong_value` for `counterparty`, `term_months`, `annual_fee_usd`, `notice_days`, `governing_law`).
    - *Key expressions or variables:* 
      - Field Value 1: `={{ $json.source_text }}`
      - Field Value 2: `={{ { counterparty: $json.counterparty, term_months: $json.term_months, annual_fee_usd: $json.annual_fee_usd, notice_days: $json.notice_days, governing_law: $json.governing_law } }}`
    - *Input and output connections:* Input from `Two Records`; output to `Reshape Checks`.
    - *Version-specific requirements:* Requires the community node package `n8n-nodes-judgment` installed on a self-hosted instance.
    - *Edge cases / failure types:* API authentication errors, rate limiting, or invalid JSON structures inside the evaluated fields.
  - **Reshape Checks**
    - *Type and technical role:* `n8n-nodes-base.set` (Set / Edit Fields Node). Flattens compound question identifiers into discrete properties (field name, check type) and assigns dynamic risk verdicts.
    - *Configuration choices:* Manual mode, maps assignment expressions across incoming check items.
    - *Key expressions or variables:* 
      - Record reference: `={{ $('Two Records').item.json.record }}`
      - Field name: `={{ $json.id.split('::')[0] }}`
      - Check name: `={{ $json.id.split('::')[1] }}`
      - Extracted value: `={{ $('Two Records').item.json[$json.id.split('::')[0]] }}`
      - Risk score: `={{ $json.value }}`
      - Verdict determination: `={{ $json.value >= 0.5 ? 'needs review' : 'confirmed' }}`
    - *Input and output connections:* Input from `Verify Extract`; output to `Is the Field in Doubt?`.
    - *Edge cases:* Expression failures if the compound ID delimiter (`::`) is missing from the API response.

---

#### 2.3 Conditional Routing & Triage
- **Overview:** Evaluates the verdict of each processed field against a risk tolerance threshold, appending contextual explanations to flagged anomalies and routing items to either human review queues or automated confirmation workflows.
- **Nodes Involved:** 
  - `Is the Field in Doubt?`
  - `Explain the Flag`
  - `Review Queue`
  - `Confirmed`

- **Node Details:**
  - **Is the Field in Doubt?**
    - *Type and technical role:* `n8n-nodes-base.if` (If Node). Evaluates whether a given field's verdict requires human intervention.
    - *Configuration choices:* Condition type set to match string equality (`verdict` equals `needs review`).
    - *Key expressions or variables:* `={{ $json.verdict }}`
    - *Input and output connections:* Input from `Reshape Checks`; True output branch routes to `Explain the Flag`, False output branch routes to `Confirmed`.
    - *Edge cases:* Unhandled string casing or typos in verdict definitions.
  - **Explain the Flag**
    - *Type and technical role:* `n8n-nodes-base.set` (Set / Edit Fields Node). Enriches flagged items with descriptive human-readable failure explanations based on the check type.
    - *Configuration choices:* Manual mode, assigns `reviewReason` property.
    - *Key expressions or variables:* `={{ $json.check === 'not_in_source' ? 'Value not found in the source document' : 'Source states a different value' }}`
    - *Input and output connections:* Input from `Is the Field in Doubt?` (True branch); output to `Review Queue`.
    - *Edge cases:* None.
  - **Review Queue**
    - *Type and technical role:* `n8n-nodes-base.noOp` (No-Op Node). Terminal holding point for suspicious or hallucinated fields awaiting secondary model analysis or manual audits.
    - *Configuration choices:* Default settings.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input from `Explain the Flag`; no outbound connections.
    - *Edge cases:* None.
  - **Confirmed**
    - *Type and technical role:* `n8n-nodes-base.noOp` (No-Op Node). Terminal endpoint for verified fields that successfully passed extraction validation.
    - *Configuration choices:* Default settings.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input from `Is the Field in Doubt?` (False branch); no outbound connections.
    - *Edge cases:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| How This Works | `n8n-nodes-base.stickyNote` | Documentation and architecture overview | None | None | ## Verify extracted contract terms and flag the fabricated ones with Judgment<br><br>### How it works<br><br>1. Builds two worked examples of the same contract extraction: one Set node with values that match the document, and one with two fields changed.<br>2. Merges them into a single stream, so both records are verified side by side.<br>3. Checks every extracted value against the source in one judgment call, asking two questions per field: is this value absent from the document, and does the document state a different one.<br>4. Reshapes the answers into a field name, a check name, the extracted value, a probability, and a verdict.<br>5. Sends any field the verifier doubted through *Explain the Flag* to the *Review Queue*, and treats the rest as confirmed.<br><br>### Setup steps<br><br>- [ ] Add your **Judgment API** credential to the *Verify Extract* node, and set the Base URL if you are not using the default provider.<br>- [ ] Install `n8n-nodes-judgment` from **Settings → Community Nodes**. See the note beside the first node.<br>- [ ] Replace *Source Document* and *Fabricated Extraction* with your own extraction output, keeping the `source_text` and one field per extracted value.<br>- [ ] Add two questions to *Verify Extract* for each field you want checked, following the existing pattern.<br>- [ ] Review the 0.5 threshold in *Reshape Checks* against the cost of a wrong field reaching production.<br><br>### Customization<br><br>Field names are the keys of the `extraction` object, so adding a field means adding its two questions. The check name carries the field, which is why one call covers a whole record.<br><br>To build the full cascade, replace *Review Queue* with a stronger reasoning model that re-extracts just the flagged fields. |
| Install Note | `n8n-nodes-base.stickyNote` | Installation guidelines for community node dependency | None | None | ## Install the community node first<br><br>**Settings → Community Nodes → Install →** `n8n-nodes-judgment`, then add your API key on the **Judgment API** credential. Self-hosted only: community nodes cannot run on n8n Cloud. |
| When Clicking Test Workflow | `n8n-nodes-base.manualTrigger` | Initiates manual workflow execution | None | Source Document, Fabricated Extraction | Run this workflow to verify both sample records. |
| Verify Extract | `n8n-nodes-judgment.judgment` | Evaluates extracted contract fields against source text | Two Records | Reshape Checks | Two questions per field, ten in total, all answered in ONE API call per record. Each question names the field it checks. |
| Reshape Checks | `n8n-nodes-base.set` | Parses compound IDs into field names, check names, and verdicts | Verify Extract | Is the Field in Doubt? | Both halves of the question ID carry information, so the field name and the check name are split back out here. |
| Review Queue | `n8n-nodes-base.noOp` | Terminal node for flagged or suspicious fields | Explain the Flag | None | Fields the verifier doubted. Re-extract them with a stronger model, or send them to a person. |
| Confirmed | `n8n-nodes-base.noOp` | Terminal node for verified, accurate extractions | Is the Field in Doubt? | None | The cheap extraction stood up to verification, so write it to your system of record here. |
| Is the Field in Doubt? | `n8n-nodes-base.if` | Routes fields based on calculated risk verdicts | Reshape Checks | Explain the Flag, Confirmed | One word decides the route. Lower the threshold in Reshape Checks and more fields come down the review path. |
| Explain the Flag | `n8n-nodes-base.set` | Generates human-readable review reasons for failed validations | Is the Field in Doubt? | Review Queue | Turns the check name into a sentence a reviewer can act on. |
| Source Document | `n8n-nodes-base.set` | Establishes base reference text and accurate sample extractions | When Clicking Test Workflow | Two Records | The document to check against, and the first extraction. These values match the source. |
| Fabricated Extraction | `n8n-nodes-base.set` | Establishes sample extractions with intentional hallucinations | When Clicking Test Workflow | Two Records | The same document, but two fields are wrong: 90 days where the contract says 60, and Norway where it says Sweden. |
| Two Records | `n8n-nodes-base.merge` | Appends valid and fabricated records into a single execution stream | Source Document, Fabricated Extraction | Verify Extract | One item per extraction, so both are verified side by side. |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Community Node Requirement:**
   - Ensure you are running a self-hosted n8n instance. Navigate to **Settings → Community Nodes → Install** and install `n8n-nodes-judgment`.
2. **Create Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `When Clicking Test Workflow`.
3. **Configure Accurate Data Source Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Source Document`.
   - Configure assignments:
     - `record` (String): `agreement-2026-0147`
     - `source_text` (String): `Annual maintenance agreement between Northwind Labs and Cobalt Facilities.\n\nTerm: 24 months, commencing 1 March 2026.\nAnnual fee: 48,000 USD, invoiced quarterly.\nNotice period: 60 days written notice from either party.\nGoverning law: Sweden.`
     - `counterparty` (String): `Cobalt Facilities`
     - `term_months` (Number): `24`
     - `annual_fee_usd` (Number): `48000`
     - `notice_days` (Number): `60`
     - `governing_law` (String): `Sweden`
   - Set **Include Other Fields** to `false`.
4. **Configure Fabricated Data Source Node:**
   - Add a second **Set** node named `Fabricated Extraction`.
   - Configure assignments:
     - `record` (String): `agreement-2026-0148`
     - `source_text` (String): `={{ $json.source_text }}`
     - `counterparty` (String): `Cobalt Facilities`
     - `term_months` (Number): `24`
     - `annual_fee_usd` (Number): `48000`
     - `notice_days` (Number): `90`
     - `governing_law` (String): `Norway`
   - Set **Include Other Fields** to `false`.
5. **Connect Triggers to Data Sources:**
   - Connect the output of `When Clicking Test Workflow` to both `Source Document` and `Fabricated Extraction`.
6. **Configure Merge Node:**
   - Add a **Merge** node (`n8n-nodes-base.merge`) named `Two Records`.
   - Set **Mode** to `Append` with `2` inputs.
   - Connect `Source Document` output to input index 0, and `Fabricated Extraction` output to input index 1.
7. **Configure Judgment Evaluation Node:**
   - Add a **Judgment** node (`n8n-nodes-judgment.judgment`) named `Verify Extract`.
   - Set **Resource** to `evaluation`, **Operation** to `evaluate`, and **State** to `fields`.
   - Configure **State Fields**:
     - Field 1 Name: `source_text`, Value: `={{ $json.source_text }}`
     - Field 2 Name: `extraction`, Value: `={{ { counterparty: $json.counterparty, term_months: $json.term_months, annual_fee_usd: $json.annual_fee_usd, notice_days: $json.notice_days, governing_law: $json.governing_law } }}`
   - Define questions (add 10 total questions tracking `not_in_source` and `wrong_value` per field name pattern, e.g., `counterparty::not_in_source`, `counterparty::wrong_value`).
   - Configure credentials: Select or create a **Judgment API** credential.
   - Connect `Two Records` output to `Verify Extract`.
8. **Configure Reshape Checks Node:**
   - Add a **Set** node named `Reshape Checks`.
   - Configure manual assignments:
     - `record` (String): `={{ $('Two Records').item.json.record }}`
     - `field` (String): `={{ $json.id.split('::')[0] }}`
     - `check` (String): `={{ $json.id.split('::')[1] }}`
     - `extractedValue` (String): `={{ $('Two Records').item.json[$json.id.split('::')[0]] }}`
     - `risk` (Number): `={{ $json.value }}`
     - `verdict` (String): `={{ $json.value >= 0.5 ? 'needs review' : 'confirmed' }}`
   - Set **Include Other Fields** to `true`.
   - Connect `Verify Extract` output to `Reshape Checks`.
9. **Configure Conditional Routing Node:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Is the Field in Doubt?`.
   - Configure conditions: String equals comparison where left value is `={{ $json.verdict }}` and right value is `needs review`.
   - Connect `Reshape Checks` output to `Is the Field in Doubt?`.
10. **Configure Flag Explanation Node:**
    - Add a **Set** node named `Explain the Flag`.
    - Configure manual assignment:
      - `reviewReason` (String): `={{ $json.check === 'not_in_source' ? 'Value not found in the source document' : 'Source states a different value' }}`
    - Set **Include Other Fields** to `true`.
    - Connect the True branch of `Is the Field in Doubt?` to `Explain the Flag`.
11. **Configure Terminal Nodes:**
    - Add two **No-Op** nodes (`n8n-nodes-base.noOp`) named `Review Queue` and `Confirmed`.
    - Connect `Explain the Flag` output to `Review Queue`.
    - Connect the False branch of `Is the Field in Doubt?` to `Confirmed`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Self-hosted instances are required for community nodes. | n8n Cloud deployments do not support community node packages like `n8n-nodes-judgment`. |
| Risk score threshold tuning. | The default threshold of `0.5` in `Reshape Checks` should be adjusted based on the operational tolerance for false positives versus missed data hallucinations. |