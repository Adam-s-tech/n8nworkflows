Monitor supplier diversity and ESG spend with Google Sheets, Groq and Notion

https://n8nworkflows.xyz/workflows/monitor-supplier-diversity-and-esg-spend-with-google-sheets--groq-and-notion-20001


# Monitor supplier diversity and ESG spend with Google Sheets, Groq and Notion

### 1. Workflow Overview

This workflow automates the procurement monitoring process by evaluating year-to-date (YTD) supplier spend against predefined diversity and ESG targets every week. It reads master datasets from Google Sheets, performs calculations in deterministic code nodes, uses a Groq large language model to suggest spend rebalancing shifts when targets are missed, records a formal scorecard in Notion, and emails a structured executive briefing to stakeholders.

The workflow logic is divided into four distinct functional blocks:

- **1.1 Schedule & Configuration Ingestion:** Triggers the pipeline weekly and initializes configuration constants (spreadsheet IDs, email targets, and corporate metadata).
- **1.2 Data Normalization & Metric Calculation:** Pulls supplier records, procurement transactions, and target rules in parallel from Google Sheets, joins them, and computes overall spend health and specific diversity/ESG metrics.
- **1.3 Conditional Evaluation & AI Rebalancing:** Evaluates whether targets are met. If targets are missed, it identifies category gaps and qualified candidate suppliers, prompts a Groq LLM to propose spend shifts, and rigorously validates and caps the AI's recommendations against risk and concentration rules.
- **1.4 Executive Reporting & Notification:** Generates an executive summary using a second Groq language model instance, structures the final payload, creates a new scorecard entry in Notion, and sends an HTML briefing email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration Ingestion
**Overview:**  
This block acts as the entry point for the automation, firing on a weekly schedule to set up essential environment variables and parameters before data extraction begins.

**Nodes Involved:**
- `Trigger Every Monday`
- `Set Config`

**Node Details:**
- **Trigger Every Monday**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger node) — Initiates workflow execution based on a cron-like schedule rule.
  - *Configuration Choices:* Configured to trigger weekly on Mondays at 09:00.
  - *Input / Output:* No input connections; outputs execution flow to the configuration node.
  - *Edge Cases / Failures:* Missed executions if the n8n instance is offline during the scheduled window.
- **Set Config**
  - *Type & Role:* `n8n-nodes-base.set` (Edit Fields node) — Establishes global static variables used across downstream nodes.
  - *Configuration Choices:* Sets key parameters such as company name, Google Spreadsheet ID, sheet tab names (`Suppliers`, `Transcations`, `Targets`), Notion database ID, report email recipient, and the evaluation period (e.g., `YTD 2026`).
  - *Input / Output:* Input from `Trigger Every Monday`; outputs in parallel to the three Google Sheets retrieval nodes.
  - *Edge Cases / Failures:* Missing or incorrect identifiers (such as a blank spreadsheet ID) will cause downstream spreadsheet requests to fail.

---

#### 2.2 Data Normalization & Metric Calculation
**Overview:**  
This block extracts master data and transaction records in parallel from Google Sheets, merges them into a single data stream, normalizes formatting, and computes aggregate spend metrics against configured targets.

**Nodes Involved:**
- `Get Suppliers`
- `Get Procurement Spend`
- `Get Targets`
- `Merge Data`
- `Normalize Data`
- `Calculate Spend Metrics`
- `Targets Met?`

**Node Details:**
- **Get Suppliers**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets node) — Retrieves supplier master records from the configured spreadsheet tab.
  - *Configuration Choices:* Dynamically references the spreadsheet ID and supplier sheet name from the `Set Config` node.
  - *Input / Output:* Input from `Set Config`; outputs raw supplier rows to `Merge Data`.
  - *Edge Cases / Failures:* Authentication expiry or API rate limits.
- **Get Procurement Spend**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets node) — Retrieves transaction-level procurement spend data.
  - *Configuration Choices:* References the spreadsheet ID and transaction spend sheet name (`Transcations`) from `Set Config`.
  - *Input / Output:* Input from `Set Config`; outputs transaction rows to `Merge Data`.
  - *Edge Cases / Failures:* Invalid sheet names or network timeouts.
- **Get Targets**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets node) — Retrieves target metrics and threshold definitions.
  - *Configuration Choices:* References the target sheet name (`Targets`) from `Set Config`.
  - *Input / Output:* Input from `Set Config`; outputs target rules to `Merge Data`.
  - *Edge Cases / Failures:* Missing required target rows will trigger fallback values in the calculation code.
- **Merge Data**
  - *Type & Role:* `n8n-nodes-base.merge` (Merge node) — Combines three separate input data streams into a unified execution context using a multi-input mode (`numberInputs: 3`).
  - *Configuration Choices:* Configured to wait for inputs from suppliers, spend, and targets.
  - *Input / Output:* Inputs from `Get Suppliers`, `Get Procurement Spend`, and `Get Targets`; outputs combined dataset to `Normalize Data`.
  - *Edge Cases / Failures:* Failure in any upstream sheet read node can stall or corrupt the merged output.
- **Normalize Data**
  - *Type & Role:* `n8n-nodes-base.code` (Code node) — Cleans, parses, and joins transaction items with supplier master data.
  - *Configuration Choices:* Uses custom JavaScript to convert currencies, parse boolean flags, standardize risk levels, and handle empty input datasets safely.
  - *Input / Output:* Input from `Merge Data`; outputs normalized rows to `Calculate Spend Metrics`.
  - *Edge Cases / Failures:* Malformed numeric strings or unexpected date formats.
- **Calculate Spend Metrics**
  - *Type & Role:* `n8n-nodes-base.code` (Code node) — Calculates actual versus target spend percentages for diverse, women-owned, minority-owned, and ESG-qualified spend categories.
  - *Configuration Choices:* Computes aggregate totals, gap percentages, category shares, and determines overall status (`ON_TRACK`, `NEEDS_ATTENTION`, `NO_DATA`).
  - *Input / Output:* Input from `Normalize Data`; outputs a consolidated metrics summary object to `Targets Met?`.
  - *Edge Cases / Failures:* Division by zero when total spend equals zero (handled via safety checks returning `NO_DATA`).
- **Targets Met?**
  - *Type & Role:* `n8n-nodes-base.if` (If node) — Branches the workflow path based on whether all targets are met or if no data is present.
  - *Configuration Choices:* Checks if `$json.all_targets_met` is true or if `$json.overall_status` equals `NO_DATA`.
  - *Input / Output:* Input from `Calculate Spend Metrics`; outputs to `AI - Executive Summary` (if targets are met) or `Find Rebalancing Opportunities` (if targets are missed).
  - *Edge Cases / Failures:* Boolean evaluation errors if properties are missing from the input schema.

---

#### 2.3 Conditional Evaluation & AI Rebalancing
**Overview:**  
Executed only when diversity or ESG targets are missed, this block isolates category gaps, filters low-risk diverse supplier candidates, prompts a Groq language model to suggest spend rebalancing actions, and validates the output against strict business constraints.

**Nodes Involved:**
- `Find Rebalancing Opportunities`
- `AI - Analyze Opportunities`
- `Groq Chat Model`
- `Validate Recommendations`

**Node Details:**
- **Find Rebalancing Opportunities**
  - *Type & Role:* `n8n-nodes-base.code` (Code node) — Identifies category spend gaps and compiles a list of eligible, underused, ESG-qualified, non-high-risk diverse supplier candidates.
  - *Configuration Choices:* Formats an AI payload containing constraints, candidate suppliers, and specific optimization rules.
  - *Input / Output:* Input from `Targets Met?` (false branch); outputs structured optimization payload to `AI - Analyze Opportunities`.
  - *Edge Cases / Failures:* Empty candidate arrays if no suppliers meet the qualification criteria.
- **AI - Analyze Opportunities**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI Chain node) — Interacts with an LLM to generate procurement spend shift recommendations.
  - *Configuration Choices:* Uses a strict system prompt demanding valid JSON without markdown wrapping, instructing the model to suggest 3 to 5 FROM → TO shifts.
  - *Input / Output:* Inputs from `Find Rebalancing Opportunities` and `Groq Chat Model`; outputs raw AI generation to `Validate Recommendations`.
  - *Edge Cases / Failures:* Malformed JSON outputs from the LLM.
- **Groq Chat Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Groq Chat Model sub-node) — Provides the underlying language model service for opportunity analysis.
  - *Configuration Choices:* Configured with model `openai/gpt-oss-20b` and a low temperature (`0.2`) for deterministic, structured output.
  - *Input / Output:* Connected via AI parameter link to `AI - Analyze Opportunities`.
  - *Edge Cases / Failures:* API authentication errors or model rate limits.
- **Validate Recommendations**
  - *Type & Role:* `n8n-nodes-base.code` (Code node) — Parses and rigorously validates the AI's recommendations.
  - *Configuration Choices:* Rejects high-risk suppliers, checks minimum ESG score thresholds, caps shifts against category potential and concentration limits, and builds fallback recommendations if the AI response is invalid.
  - *Input / Output:* Input from `AI - Analyze Opportunities`; outputs validated recommendations and review flags to `AI - Executive Summary`.
  - *Edge Cases / Failures:* Entirely unparseable model outputs trigger built-in programmatic fallback rules to ensure workflow continuity.

---

#### 2.4 Executive Reporting & Notification
**Overview:**  
This block compiles all metrics, validation notes, and rebalancing recommendations, generates a natural-language executive briefing via a secondary AI model instance, creates a structured Notion database scorecard page, and dispatches an HTML email briefing via Gmail.

**Nodes Involved:**
- `AI - Executive Summary`
- `Groq Chat Model Summary`
- `Prepare Results`
- `Create Notion Scorecard`
- `Send Weekly Briefing`

**Node Details:**
- **AI - Executive Summary**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI Chain node) — Generates a plain-text executive summary tailored for a Chief Procurement Officer.
  - *Configuration Choices:* Prompt instructs the model to use a specific plain-text template summarizing overall status, total spend, category gaps, and recommended actions.
  - *Input / Output:* Inputs from `Targets Met?`, `Validate Recommendations`, and `Groq Chat Model Summary`; outputs summary text to `Prepare Results`.
  - *Edge Cases / Failures:* Errors handled by `onError: continueRegularOutput` allowing execution to proceed with fallback summaries.
- **Groq Chat Model Summary**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Groq Chat Model sub-node) — Provides the language model service for the executive summary generation.
  - *Configuration Choices:* Configured with model `openai/gpt-oss-20b` and temperature `0.3`.
  - *Input / Output:* Connected via AI parameter link to `AI - Executive Summary`.
  - *Edge Cases / Failures:* API connectivity issues.
- **Prepare Results**
  - *Type & Role:* `n8n-nodes-base.code` (Code node) — Aggregates all data points into a unified schema, formats currency values, constructs HTML email bodies, and prepares property mappings for Notion.
  - *Configuration Choices:* Formats tables, lists of recommendations, subject lines, and truncates text strings to meet Notion's field length limitations.
  - *Input / Output:* Inputs from `Calculate Spend Metrics` and `AI - Executive Summary`; outputs structured payload to `Create Notion Scorecard`.
  - *Edge Cases / Failures:* Missing upstream metrics objects handled via safe try/catch fallbacks.
- **Create Notion Scorecard**
  - *Type & Role:* `n8n-nodes-base.notion` (Notion node) — Creates a new database page acting as a weekly scorecard.
  - *Configuration Choices:* Uses resource `databasePage`, references the database ID from `Set Config`, and maps dynamic properties (Title, Run Date, Period, Company, Status values, Spend numbers, and Executive Summary text).
  - *Input / Output:* Input from `Prepare Results`; outputs created page details (including URL) to `Send Weekly Briefing`.
  - *Edge Cases / Failures:* Notion select option mismatch (e.g., passing a status value not configured in the Notion database schema) handled with `onError: continueRegularOutput`.
- **Send Weekly Briefing**
  - *Type & Role:* `n8n-nodes-base.gmail` (Gmail node) — Sends the final HTML-formatted email briefing to the configured recipient.
  - *Configuration Choices:* Dynamically sets recipient email, subject line, and appends the Notion scorecard URL to the message body.
  - *Input / Output:* Input from `Create Notion Scorecard`; final termination node of the workflow.
  - *Edge Cases / Failures:* Gmail authentication failure or invalid recipient address.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger Every Monday | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow every Monday at 09:00 | None | Set Config | Supplier Diversity & ESG Spend Monitor / 1. Config & Data Ingestion |
| Set Config | `n8n-nodes-base.set` | Establishes global configuration parameters and spreadsheet identifiers | Trigger Every Monday | Get Suppliers, Get Procurement Spend, Get Targets | Supplier Diversity & ESG Spend Monitor / 1. Config & Data Ingestion |
| Get Suppliers | `n8n-nodes-base.googleSheets` | Retrieves supplier master records from Google Sheets | Set Config | Merge Data | 1. Config & Data Ingestion |
| Get Procurement Spend | `n8n-nodes-base.googleSheets` | Retrieves transaction spend data from Google Sheets | Set Config | Merge Data | 1. Config & Data Ingestion |
| Get Targets | `n8n-nodes-base.googleSheets` | Retrieves target metrics and thresholds from Google Sheets | Set Config | Merge Data | 1. Config & Data Ingestion |
| Merge Data | `n8n-nodes-base.merge` | Merges supplier, spend, and target streams into a single dataset | Get Suppliers, Get Procurement Spend, Get Targets | Normalize Data | 1. Config & Data Ingestion |
| Normalize Data | `n8n-nodes-base.code` | Normalizes transaction data and joins with supplier master records | Merge Data | Calculate Spend Metrics | 2. Normalize & Score vs Targets |
| Calculate Spend Metrics | `n8n-nodes-base.code` | Calculates actual vs. target spend percentages and overall status | Normalize Data | Targets Met? | 2. Normalize & Score vs Targets |
| Targets Met? | `n8n-nodes-base.if` | Evaluates if all targets are met or if no data exists | Calculate Spend Metrics | AI - Executive Summary, Find Rebalancing Opportunities | 2. Normalize & Score vs Targets |
| Find Rebalancing Opportunities | `n8n-nodes-base.code` | Identifies category gaps and eligible candidate suppliers for rebalancing | Targets Met? | AI - Analyze Opportunities | 3. AI Rebalancing with Guardrails |
| Groq Chat Model | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides language model capability for rebalancing analysis | None | AI - Analyze Opportunities | 3. AI Rebalancing with Guardrails |
| AI - Analyze Opportunities | `@n8n/n8n-nodes-langchain.chainLlm` | Prompts Groq LLM to propose procurement spend shift recommendations | Find Rebalancing Opportunities, Groq Chat Model | Validate Recommendations | 3. AI Rebalancing with Guardrails |
| Validate Recommendations | `n8n-nodes-base.code` | Validates, caps, and filters LLM recommendations against business rules | AI - Analyze Opportunities | AI - Executive Summary | 3. AI Rebalancing with Guardrails |
| Groq Chat Model Summary | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides language model capability for executive summary generation | None | AI - Executive Summary | 4. Scorecard & CPO Briefing |
| AI - Executive Summary | `@n8n/n8n-nodes-langchain.chainLlm` | Generates a plain-text executive briefing report | Targets Met?, Validate Recommendations, Groq Chat Model Summary | Prepare Results | 4. Scorecard & CPO Briefing |
| Prepare Results | `n8n-nodes-base.code` | Consolidates results, formats HTML email, and maps Notion properties | Calculate Spend Metrics, AI - Executive Summary | Create Notion Scorecard | 4. Scorecard & CPO Briefing |
| Create Notion Scorecard | `n8n-nodes-base.notion` | Creates a new database page for the weekly scorecard in Notion | Prepare Results | Send Weekly Briefing | 4. Scorecard & CPO Briefing |
| Send Weekly Briefing | `n8n-nodes-base.gmail` | Sends the formatted HTML briefing email to the CPO via Gmail | Create Notion Scorecard | None | 4. Scorecard & CPO Briefing |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to recreate the entire workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Trigger Every Monday`.
   - Configure the rule interval to trigger weekly on Mondays at `09:00`.

2. **Create the Configuration Node:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`) named `Set Config`.
   - Connect `Trigger Every Monday` to `Set Config`.
   - Add string assignments: `company_name` (default `""`), `spreadsheet_id` (Google Sheet ID), `suppliers_sheet` (`Suppliers`), `spend_sheet` (`Transcations`), `targets_sheet` (`Targets`), `notion_database_id` (Notion database ID), `report_email` (recipient email), and `period` (`YTD 2026`).

3. **Create the Data Ingestion Nodes:**
   - Add three **Google Sheets** nodes (`n8n-nodes-base.googleSheets`) named `Get Suppliers`, `Get Procurement Spend`, and `Get Targets`.
   - Connect the output of `Set Config` to each of these three nodes.
   - Configure each node to use the Spreadsheet ID from `={{ $('Set Config').first().json.spreadsheet_id }}` and respective sheet names (`suppliers_sheet`, `spend_sheet`, and `targets_sheet`).

4. **Merge Ingested Data:**
   - Add a **Merge** node (`n8n-nodes-base.merge`) named `Merge Data`.
   - Set the input count to `3`.
   - Connect `Get Suppliers`, `Get Procurement Spend`, and `Get Targets` into inputs `0`, `1`, and `2` respectively.

5. **Normalize and Calculate Metrics:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Normalize Data`. Connect `Merge Data` to it. Populate with the normalization JavaScript code (handling supplier joins, currency formatting, and boolean conversions).
   - Add a second **Code** node named `Calculate Spend Metrics`. Connect `Normalize Data` to it. Populate with the script that aggregates total spend, diverse spend, women-owned spend, minority-owned spend, and ESG-qualified spend against configured targets.

6. **Add Conditional Branching:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Targets Met?`. Connect `Calculate Spend Metrics` to it.
   - Configure conditions to check if `$json.all_targets_met` is true or if `$json.overall_status` equals `NO_DATA`.

7. **Build the AI Rebalancing Path (False Branch):**
   - Add a **Code** node named `Find Rebalancing Opportunities`. Connect the false branch of `Targets Met?` to it.
   - Add an **Advanced AI Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`) named `AI - Analyze Opportunities`. Connect `Find Rebalancing Opportunities` to it.
   - Add a **Groq Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatGroq`) named `Groq Chat Model` (configured with model `openai/gpt-oss-20b`, temperature `0.2`, and valid Groq API credentials). Link it to the `ai_languageModel` input of `AI - Analyze Opportunities`.
   - Add a **Code** node named `Validate Recommendations`. Connect `AI - Analyze Opportunities` to it to validate, cap, and filter spend shift recommendations.

8. **Build the Executive Summary Path:**
   - Add an **Advanced AI Chain** node named `AI - Executive Summary`. Connect the true branch of `Targets Met?` and the output of `Validate Recommendations` to it.
   - Add a **Groq Chat Model** sub-node named `Groq Chat Model Summary` (configured with model `openai/gpt-oss-20b`, temperature `0.3`, and valid Groq API credentials). Link it to the `ai_languageModel` input of `AI - Executive Summary`.
   - Set the error handling of `AI - Executive Summary` to `continueRegularOutput`.

9. **Prepare Results and Publish to Notion:**
   - Add a **Code** node named `Prepare Results`. Connect `Calculate Spend Metrics` and `AI - Executive Summary` to it. This node compiles enums, HTML email templates, and Notion property mappings.
   - Add a **Notion** node (`n8n-nodes-base.notion`) named `Create Notion Scorecard`. Connect `Prepare Results` to it.
   - Set resource to `Database Page`, database ID to `={{ $('Set Config').first().json.notion_database_id }}`, and map all required database property keys (Title, Run Date, Period, Company, Statuses, Numbers, Rich Text fields). Set error handling to `continueRegularOutput`.

10. **Send Briefing Email via Gmail:**
    - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Send Weekly Briefing`. Connect `Create Notion Scorecard` to it.
    - Configure the recipient (`={{ $('Prepare Results').first().json.report_email }}`), subject line, and HTML message body including the Notion page URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Primary objective is supplier diversity; ESG and risk are operational constraints. | Core workflow governance rule. |
| Notion database select options must strictly match: `ON_TRACK`, `NEEDS_ATTENTION`, `NO_DATA`, `MET`, `BELOW_TARGET`, `HIGH`, `MEDIUM`, `LOW`, `NONE`, `HUMAN_REVIEW`. | Notion Database Schema Setup |
| High-impact spend shifts automatically trigger a `HUMAN_REVIEW` flag and set priority to `HIGH`, but do not block automated reporting execution. | Workflow Safety & Governance |