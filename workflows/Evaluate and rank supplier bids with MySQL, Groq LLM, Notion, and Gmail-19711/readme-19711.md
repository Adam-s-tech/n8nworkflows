Evaluate and rank supplier bids with MySQL, Groq LLM, Notion, and Gmail

https://n8nworkflows.xyz/workflows/evaluate-and-rank-supplier-bids-with-mysql--groq-llm--notion--and-gmail-19711


# Evaluate and rank supplier bids with MySQL, Groq LLM, Notion, and Gmail

### 1. Workflow Overview

This workflow acts as an automated procurement intelligence pipeline. Its primary purpose is to pull pending vendor bids from a database, normalize financial and delivery metrics across different currencies and units, use a Groq-hosted Large Language Model (LLM) to evaluate qualitative text and compliance certifications anonymously, rank suppliers using weighted scoring scenarios, log the top results to a Notion database, and compile an executive summary report delivered via Gmail.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception & Data Normalization:** Initiates the workflow manually, pulls live currency rates via an HTTP request, queries submitted proposals from a MySQL database, and standardizes prices and delivery durations while applying blind aliases to eliminate vendor bias.
- **1.2 AI Qualitative Evaluation Loop:** Iterates through each vendor proposal using a batching loop, passes qualitative data and compliance data to an AI model with a structured output parser, and checks for security risks.
- **1.3 Ranking, Archiving & Summary Generation:** Merges quantitative and qualitative metrics, runs a math and ranking engine across multiple priority scenarios, filters the top 3 suppliers, archives details to Notion, and generates a structured executive summary via an LLM.
- **1.4 Executive Notifications & Delivery:** Formats the generated report, converts the markdown output to HTML, and securely dispatches the final email summary to stakeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Normalization
**Overview:** This block establishes the entry point for the pipeline, gathers live conversion metrics, queries structured procurement data from a MySQL database, and standardizes values into a uniform format while masking supplier identities.

**Nodes Involved:**
- Start Workflow
- Fetch Live Exchange Rates
- Fetch Submitted Bids (MySQL)
- Normalize & Anonymize Data

**Node Details:**
- **Start Workflow**
  - Type: `n8n-nodes-base.manualTrigger`
  - Technical Role: Manually triggers the execution of the evaluation pipeline.
  - Configuration Choices: Default parameters (no configuration required).
  - Key Expressions/Variables: None.
  - Input/Output: Outputs a single trigger signal to the HTTP Request node.
  - Edge Cases/Failures: User error if triggered prematurely.
- **Fetch Live Exchange Rates**
  - Type: `n8n-nodes-base.httpRequest`
  - Technical Role: Queries external currency APIs to obtain live conversion rates based against USD.
  - Configuration Choices: GET request to `https://api.freecurrencyapi.com/v1/latest` with query parameters enabled (`apikey`). Executed once.
  - Key Expressions/Variables: Uses the configured FreeCurrencyAPI key.
  - Input/Output: Input from Start Workflow; output connects to the MySQL fetch node.
  - Edge Cases/Failures: Invalid or expired API key causes HTTP request failure.
- **Fetch Submitted Bids (MySQL)**
  - Type: `n8n-nodes-base.mySql`
  - Technical Role: Retrieves pending vendor proposal records matching a specific RFP ID and status from a database.
  - Configuration Choices: Select operation on table `submitted_bids` filtered where `rfp_id` equals `REQ-2026-IT` and `status` equals `submitted`.
  - Key Expressions/Variables: Uses MySQL credential configuration.
  - Input/Output: Input from Fetch Live Exchange Rates; output connects to Normalize & Anonymize Data.
  - Edge Cases/Failures: Database connection failure or missing table/filters returning zero records.
- **Normalize & Anonymize Data**
  - Type: `n8n-nodes-base.code`
  - Technical Role: Standardizes multi-currency pricing to USD equivalents, normalizes delivery units (weeks/months) into absolute days, and assigns blind vendor aliases (`Supplier_1`, etc.) to mitigate evaluation bias.
  - Configuration Choices: JavaScript execution block processing all input rows.
  - Key Expressions/Variables: References live rates via `$('Fetch Live Exchange Rates').first().json.data`.
  - Input/Output: Input from Fetch Submitted Bids; output connects to Loop Through Bids.
  - Edge Cases/Failures: Unexpected currency or delivery time formats falling back to defaults.

---

#### 2.2 AI Qualitative Evaluation Loop
**Overview:** This block iterates through the normalized vendor proposals one by one, uses a Groq-hosted LLM with an enforced schema parser to evaluate qualitative proposals, extracts structured metrics, and evaluates security risk criteria.

**Nodes Involved:**
- Loop Through Bids
- AI Proposal Auditor
- Groq LLM (Auditor)
- Enforce Scoring Schema
- Wait for Next Item
- Extract Evaluated Scores
- Check Security Risk

**Node Details:**
- **Loop Through Bids**
  - Type: `n8n-nodes-base.splitInBatches`
  - Technical Role: Handles iterative processing of the supplier array.
  - Configuration Choices: Standard batching loop configuration.
  - Key Expressions/Variables: None.
  - Input/Output: Input from Normalize & Anonymize Data; outputs branch to Extract Evaluated Scores / AI Proposal Auditor.
  - Edge Cases/Failures: Infinite loops if batching configurations are modified improperly.
- **AI Proposal Auditor**
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`
  - Technical Role: Core reasoning node evaluating qualitative texts, SLAs, and compliance certifications.
  - Configuration Choices: Uses a defined prompt type requiring a strict JSON output matching metrics like `sla_score`, `support_score`, `compliance_score`, `risk_flag`, and `ai_justification`.
  - Key Expressions/Variables: `=Input: Alias: {{ $json.blind_alias }} Certifications: {{ $json.compliance_flags }} Proposal: {{ $json.qualitative_proposal }}`
  - Input/Output: Connected to Groq LLM and Enforce Scoring Schema via AI parameters; output connects to Wait for Next Item.
  - Edge Cases/Failures: LLM hallucination or schema parsing errors if model output deviates from guidelines.
- **Groq LLM (Auditor)**
  - Type: `@n8n/n8n-nodes-langchain.lmChatGroq`
  - Technical Role: Language model provider for the AI Proposal Auditor node.
  - Configuration Choices: Model set to `openai/gpt-oss-120b`. Uses Groq API credentials.
  - Key Expressions/Variables: None.
  - Input/Output: Linked as language model dependency to AI Proposal Auditor.
  - Edge Cases/Failures: API quota exhaustion or model unavailability.
- **Enforce Scoring Schema**
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
  - Technical Role: Forces the LLM output into an explicit JSON schema structure.
  - Configuration Choices: Manual schema input defining integer scores, boolean risk flags, and string justifications.
  - Key Expressions/Variables: None.
  - Input/Output: Linked as output parser to AI Proposal Auditor.
  - Edge Cases/Failures: Type mismatch exceptions if the LLM output fails validation.
- **Wait for Next Item**
  - Type: `n8n-nodes-base.wait`
  - Technical Role: Pauses execution briefly between batch iterations.
  - Configuration Choices: Webhook-driven wait configuration.
  - Key Expressions/Variables: None.
  - Input/Output: Input from AI Proposal Auditor; output loops back to Loop Through Bids.
  - Edge Cases/Failures: Timeout or webhook transmission drops.
- **Extract Evaluated Scores**
  - Type: `n8n-nodes-base.set`
  - Technical Role: Maps output fields from the structured AI evaluation parser into clean property assignments.
  - Configuration Choices: Assignments for `sla_score`, `support_score`, `compliance_score`, `risk_flag`, `ai_justification`, and `Supplier_alias`.
  - Key Expressions/Variables: `={{ $json.output.sla_score }}`, etc.
  - Input/Output: Input from Loop Through Bids; output connects to Check Security Risk.
  - Edge Cases/Failures: Missing fields in the preceding parse step.
- **Check Security Risk**
  - Type: `n8n-nodes-base.if`
  - TechnicalRole: Inspects whether the `risk_flag` property is set to true/false to route anomalies.
  - Configuration Choices: Condition checking `={{ $json.risk_flag }}` equals `false`.
  - Key Expressions/Variables: `={{ $json.risk_flag }}`
  - Input/Output: Input from Extract Evaluated Scores; true/clear path proceeds to the Math & Ranking Engine.
  - Edge Cases/Failures: Boolean evaluation errors if data types are altered.

---

#### 2.3 Ranking, Archiving & Summary Generation
**Overview:** This block aggregates qualitative AI metrics with normalized quantitative parameters, executes a weighted math engine to calculate multi-scenario rankings, writes records to a Notion database, and synthesizes an executive summary report via an LLM.

**Nodes Involved:**
- Math & Ranking Engine
- Log Supplier to Notion
- Generate Executive Summary
- Groq LLM (Summarizer)
- Enforce Report Schema

**Node Details:**
- **Math & Ranking Engine**
  - Type: `n8n-nodes-base.code`
  - Technical Role: Computes quality scores with risk penalties, scales pricing and speed factors, calculates alternative weighted scenarios (standard, cost-optimized, risk-averse, speed-to-market), sorts candidates, and limits output to the top 3 suppliers.
  - Configuration Choices: JavaScript processing block merging `aiResults` with `Normalize & Anonymize Data`.
  - Key Expressions/Variables: References node data arrays via `$('Normalize & Anonymize Data').all()`.
  - Input/Output: Input from Check Security Risk; outputs connect to Log Supplier to Notion and Generate Executive Summary.
  - Edge Cases/Failures: Array index mismatches if input nodes get out of sync.
- **Log Supplier to Notion**
  - Type: `n8n-nodes-base.notion`
  - Technical Role: Creates Notion database entries for each shortlisted vendor detailing metrics, delivery times, scores, and justifications.
  - Configuration Choices: Resource set to database page using data source ID linked to the target Notion database.
  - Key Expressions/Variables: Uses expressions such as `={{ $json.alias }}`, `={{ $json.supplier_id }}`, `={{ $json.price }}`.
  - Input/Output: Input from Math & Ranking Engine.
  - Edge Cases/Failures: Insufficient integration permissions on the target Notion page or database schema mismatches.
- **Generate Executive Summary**
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`
  - Technical Role: Synthesizes ranked multi-scenario leadership data into a cohesive executive procurement summary.
  - Configuration Choices: Defined prompt providing JSON stringified input data to a senior procurement intelligence advisor persona.
  - Key Expressions/Variables: `=Here is the ranked supplier evaluation data with scenario simulations: {{ JSON.stringify($input.all().map(item => item.json), null, 2) }}`
  - Input/Output: Linked to Groq LLM (Summarizer) and Enforce Report Schema; output connects to Format Final Output.
  - Edge Cases/Failures: Token-limit overflow if too many supplier items are passed.
- **Groq LLM (Summarizer)**
  - Type: `@n8n/n8n-nodes-langchain.lmChatGroq`
  - Technical Role: Language model provider backing the executive summary generation.
  - Configuration Choices: Model configured to `openai/gpt-oss-20b`. Uses Groq API credentials.
  - Key Expressions/Variables: None.
  - Input/Output: Linked as language model dependency to Generate Executive Summary.
  - Edge Cases/Failures: Rate limiting or API timeout errors.
- **Enforce Report Schema**
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
  - Technical Role: Enforces structured formatting requirements on the executive summary JSON output.
  - Configuration Choices: Manual schema input requiring `executive_decision`, `scenario_analysis`, `risk_alert`, `next_steps` (array), and `full_markdown_report`.
  - Key Expressions/Variables: None.
  - Input/Output: Linked as output parser to Generate Executive Summary.
  - Edge Cases/Failures: JSON parsing failure if markdown content contains unescaped characters.

---

#### 2.4 Executive Notifications & Delivery
**Overview:** This block structures the final report data, converts the synthesized markdown text into HTML format, and dispatches the procurement brief via Gmail to stakeholders.

**Nodes Involved:**
- Format Final Output
- Convert Markdown to HTML
- Send Email Report

**Node Details:**
- **Format Final Output**
  - Type: `n8n-nodes-base.set`
  - Technical Role: Maps output parameters from the executive summary schema into a consolidated dataset.
  - Configuration Choices: Field assignments for summary keys.
  - Key Expressions/Variables: `={{ $json.output.executive_decision }}`, `={{ $json.output.full_markdown_report }}`, etc.
  - Input/Output: Input from Generate Executive Summary; output connects to Convert Markdown to HTML.
  - Edge Cases/Failures: Schema mapping discrepancies.
- **Convert Markdown to HTML**
  - Type: `n8n-nodes-base.markdown`
  - Technical Role: Transforms the markdown-formatted executive report into standard HTML markup suitable for email rendering.
  - Configuration Choices: Mode set to `markdownToHtml`.
  - Key Expressions/Variables: `={{ $json.full_markdown_report }}`
  - Input/Output: Input from Format Final Output; output connects to Send Email Report.
  - Edge Cases/Failures: Formatting anomalies with complex nested markdown blocks.
- **Send Email Report**
  - Type: `n8n-nodes-base.gmail`
  - Technical Role: Dispatches the converted HTML executive brief via Gmail to designated stakeholders.
  - Configuration Choices: Subject configured as `New Vendor Evaluation: IT Procurement`.
  - Key Expressions/Variables: Message payload set to `={{ $json.data }}`.
  - Input/Output: Input from Convert Markdown to HTML; terminal node.
  - Edge Cases/Failures: Expired OAuth tokens or invalid recipient configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Workflow | `n8n-nodes-base.manualTrigger` | Manually triggers the workflow execution. | None | Fetch Live Exchange Rates | ## Workflow Overview: AI-Powered Supplier Bid Evaluation<br><br>This workflow acts as an automated **Procurement Intelligence Engine** for evaluating vendor proposals. It extracts submitted bids from a MySQL database, fetches live currency exchange rates, and standardizes financial and delivery metrics while anonymizing vendor names to prevent bias. |
| Fetch Live Exchange Rates | `n8n-nodes-base.httpRequest` | Fetches live currency exchange rates from an external API. | Start Workflow | Fetch Submitted Bids (MySQL) | ## Data Ingestion & Normalization <br>Fetches live exchange rates and pending supplier bids from the database, standardizing currencies and delivery times while anonymizing vendor names. |
| Fetch Submitted Bids (MySQL) | `n8n-nodes-base.mySql` | Queries pending bid records from a MySQL database. | Fetch Live Exchange Rates | Normalize & Anonymize Data | ## Data Ingestion & Normalization <br>Fetches live exchange rates and pending supplier bids from the database, standardizing currencies and delivery times while anonymizing vendor names. |
| Normalize & Anonymize Data | `n8n-nodes-base.code` | Normalizes currency values and delivery times while masking supplier identities. | Fetch Submitted Bids (MySQL) | Loop Through Bids | ## Data Ingestion & Normalization <br>Fetches live exchange rates and pending supplier bids from the database, standardizing currencies and delivery times while anonymizing vendor names. |
| Loop Through Bids | `n8n-nodes-base.splitInBatches` | Iterates through supplier bids one by one. | Normalize & Anonymize Data | AI Proposal Auditor, Extract Evaluated Scores | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| AI Proposal Auditor | `@n8n/n8n-nodes-langchain.chainLlm` | Evaluates qualitative text and compliance certifications using an LLM. | Loop Through Bids | Wait for Next Item | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| Groq LLM (Auditor) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides the language model for proposal auditing. | None | AI Proposal Auditor | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| Enforce Scoring Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured output formatting for bid evaluations. | None | AI Proposal Auditor | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| Wait for Next Item | `n8n-nodes-base.wait` | Pauses execution between loop iterations. | AI Proposal Auditor | Loop Through Bids | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| Extract Evaluated Scores | `n8n-nodes-base.set` | Maps evaluated score attributes into clear variables. | Loop Through Bids | Check Security Risk | ## AI Qualitative Evaluation Loop<br>Processes each supplier bid individually, using an AI model to evaluate qualitative proposal text and extract structured compliance and SLA scores. |
| Check Security Risk | `n8n-nodes-base.if` | Inspects whether the risk flag is triggered. | Extract Evaluated Scores | Math & Ranking Engine | ## Security Anomaly Exception Routing<br>Intercepts high-risk supplier bids flagged for security or compliance gaps using an IF node and routes immediate warning alerts to a dedicated Slack channel. |
| Math & Ranking Engine | `n8n-nodes-base.code` | Computes weighted scenario scores and sorts top suppliers. | Check Security Risk | Log Supplier to Notion, Generate Executive Summary | ## Ranking, Archiving & Summary Generation<br>Merges qualitative AI scores with quantitative metrics, calculates scenario rankings, logs individual metrics to Notion, and generates the final executive summary. |
| Log Supplier to Notion | `n8n-nodes-base.notion` | Saves individual supplier metrics to a Notion database. | Math & Ranking Engine | None | ## Ranking, Archiving & Summary Generation<br>Merges qualitative AI scores with quantitative metrics, calculates scenario rankings, logs individual metrics to Notion, and generates the final executive summary. |
| Generate Executive Summary | `@n8n/n8n-nodes-langchain.chainLlm` | Synthesizes ranked leaderboard data into an executive report. | Math & Ranking Engine | Format Final Output | ## Ranking, Archiving & Summary Generation<br>Merges qualitative AI scores with quantitative metrics, calculates scenario rankings, logs individual metrics to Notion, and generates the final executive summary. |
| Groq LLM (Summarizer) | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides the language model for executive summary generation. | None | Generate Executive Summary | ## Ranking, Archiving & Summary Generation<br>Merges qualitative AI scores with quantitative metrics, calculates scenario rankings, logs individual metrics to Notion, and generates the final executive summary. |
| Enforce Report Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured formatting requirements on the executive report. | None | Generate Executive Summary | ## Ranking, Archiving & Summary Generation<br>Merges qualitative AI scores with quantitative metrics, calculates scenario rankings, logs individual metrics to Notion, and generates the final executive summary. |
| Format Final Output | `n8n-nodes-base.set` | Consolidates executive report output attributes. | Generate Executive Summary | Convert Markdown to HTML | ## Executive Notifications & Delivery<br>Formats the generated executive report and securely dispatches formatted alerts directly to team Slack channels and stakeholder email inboxes. |
| Convert Markdown to HTML | `n8n-nodes-base.markdown` | Transforms markdown executive summaries into HTML markup. | Format Final Output | Send Email Report | ## Executive Notifications & Delivery<br>Formats the generated executive report and securely dispatches formatted alerts directly to team Slack channels and stakeholder email inboxes. |
| Send Email Report | `n8n-nodes-base.gmail` | Emails the finalized HTML procurement report to stakeholders. | Convert Markdown to HTML | None | ## Executive Notifications & Delivery<br>Formats the generated executive report and securely dispatches formatted alerts directly to team Slack channels and stakeholder email inboxes. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) as the starting entry point.
2. **Fetch Exchange Rates:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Set the method to `GET`, URL to `https://api.freecurrencyapi.com/v1/latest`, enable query parameters, add a parameter named `apikey`, and supply your FreeCurrencyAPI key. Connect the Manual Trigger output to this node.
3. **Fetch Database Bids:** Add a **MySQL** node (`n8n-nodes-base.mySql`). Configure credentials, select operation `Select`, table `submitted_bids`, and add where conditions matching column `rfp_id` equal to `REQ-2026-IT` and column `status` equal to `submitted`. Connect the HTTP Request output to this node.
4. **Normalize & Anonymize Data:** Add a **Code** node (`n8n-nodes-base.code`). Insert JavaScript logic to extract live currency rates from the HTTP Request node, convert quoted pricing to USD equivalents, normalize delivery units (weeks/months to days), and assign blind aliases (`Supplier_1`, etc.). Connect the MySQL node output to this node.
5. **Set up Batch Loop:** Add a **Split In Batches** node (`n8n-nodes-base.splitInBatches`) to process each supplier item iteratively. Connect the Code node output to this node.
6. **Add AI Proposal Auditor Chain:**
   - Add an **Advanced AI -> Chain: LLM** node (`@n8n/n8n-nodes-langchain.chainLlm`). Set the prompt type to define and supply the auditor prompt evaluating `sla_score`, `support_score`, `compliance_score`, `risk_flag`, `ai_justification`, and `Supplier_alias`.
   - Add a **Groq Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatGroq`), select the model (e.g., `openai/gpt-oss-120b`), configure Groq API credentials, and connect it to the LLM Chain's model input.
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`), define the required JSON schema with integer scores, boolean risk flags, and string descriptions, and connect it to the LLM Chain's parser input.
   - Connect the batch loop output to the Chain LLM node.
7. **Handle Iteration & Wait:**
   - Add a **Wait** node (`n8n-nodes-base.wait`) configured via webhooks, and connect the Chain LLM output to it. Connect the Wait node output back to the **Split In Batches** node to complete the loop.
8. **Extract Evaluated Scores:** Add a **Set** node (`n8n-nodes-base.set`) to map output attributes from the AI evaluation parser (`sla_score`, `support_score`, `compliance_score`, `risk_flag`, `ai_justification`, `Supplier_alias`). Connect the batch loop output branch to this node.
9. **Check Security Risk:** Add an **IF** node (`n8n-nodes-base.if`) with a condition verifying that `={{ $json.risk_flag }}` equals `false`. Connect the Set node output to this node.
10. **Build the Ranking Engine:** Add a **Code** node (`n8n-nodes-base.code`) to merge AI evaluations with normalized pricing and delivery days, calculate quality scores with risk penalties, compute multi-scenario weighted rankings (standard, cost-optimized, risk-averse, speed-to-market), sort results, and restrict output to the top 3 items via `.slice(0, 3)`. Connect the clear path output of the IF node to this node.
11. **Log to Notion:** Add a **Notion** node (`n8n-nodes-base.notion`). Set resource to `databasePage`, select your target database ID, and map properties to fields such as `Name`, `Supplier ID`, `Quoted Price`, `Delivery Days`, `Standard Weight Score`, `AI Justification`, and `Quality Score`. Connect the Ranking Engine output to this node.
12. **Generate Executive Summary:**
   - Add a second **Chain: LLM** node (`@n8n/n8n-nodes-langchain.chainLlm`). Set the prompt type to define with instructions for a senior procurement advisor analyzing ranked JSON data.
   - Add a **Groq Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatGroq`) with model `openai/gpt-oss-20b` and Groq API credentials, connected to the summarizer chain.
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) defining the report schema (`executive_decision`, `scenario_analysis`, `risk_alert`, `next_steps`, `full_markdown_report`), connected to the summarizer chain.
   - Connect the Ranking Engine output to this summarizer chain.
13. **Format & Deliver Output:**
   - Add a **Set** node (`n8n-nodes-base.set`) to map output parameters from the executive summary parser. Connect the summarizer chain output to this node.
   - Add a **Markdown** node (`n8n-nodes-base.markdown`) set to `markdownToHtml` mode, taking `={{ $json.full_markdown_report }}` as input. Connect the Set node output to this node.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) configured with your Gmail credentials, subject `New Vendor Evaluation: IT Procurement`, and message body set to `={{ $json.data }}`. Connect the Markdown node output to this node to complete the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Need help with setup, custom database connections, or AI prompt tuning? | Contact WeblineIndia: [WeblineIndia Contact Us](https://www.weblineindia.com/contact-us.html) |