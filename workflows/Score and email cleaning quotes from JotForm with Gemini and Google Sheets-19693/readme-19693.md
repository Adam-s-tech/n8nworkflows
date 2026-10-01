Score and email cleaning quotes from JotForm with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/score-and-email-cleaning-quotes-from-jotform-with-gemini-and-google-sheets-19693


# Score and email cleaning quotes from JotForm with Gemini and Google Sheets

### 1. Workflow Overview

This workflow automates the ingestion, qualification, and estimation of commercial cleaning quote requests submitted via Jotform. It processes incoming data through AI analysis, applies business calculation logic for pricing and labor, filters low-quality leads, and routes qualified prospects into Google Sheets and Gmail.

The workflow logic is divided into five functional blocks:
- **1.1 Input Reception & Validation:** Captures submissions from Jotform, standardizes contact and property fields, and filters out entries missing critical information.
- **1.2 AI Requirement Analysis:** Employs Google Gemini with a structured output parser to interpret property notes and extract specific architectural variables.
- **1.3 Mathematical Estimation & Sizing:** Calculates estimated labor hours based on square footage, extracted rooms, and property modifiers, then computes a low/high internal quote range.
- **1.4 AI Summarization & Routing:** Uses a second Google Gemini agent to draft a client-ready proposal summary, rating its urgency and quality score via a conditional branch.
- **1.5 Data Persistence & Notification:** Appends or updates the qualified lead record in Google Sheets and dispatches an automated quotation email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
This block listens for new website quote submissions, maps raw form fields into a clean flat schema, and verifies that essential contact and property data exist before downstream processing.

- **Nodes Involved:** 
  - `When Quote Form Submitted`
  - `Set Submission Details`
  - `Filter Required Info`

- **Node Details:**
  - **When Quote Form Submitted**
    - *Type and Technical Role:* `n8n-nodes-base.jotFormTrigger` (Trigger node). Listens for webhook events when a form is submitted.
    - *Configuration Choices:* Configured for JotForm ID `262593642323054`.
    - *Key Expressions/Variables:* None (webhook trigger).
    - *Connections:* Input: None; Output: `Set Submission Details`.
    - *Version Requirements:* TypeVersion 1.
    - *Edge Cases/Failure Types:* Webhook delivery failure, invalid JotForm credentials, or deleted/changed form ID.

  - **Set Submission Details**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation node). Flattens complex object structures (such as nested name, phone, and address objects) into top-level string fields.
    - *Configuration Choices:* Maps fields: `Full Name`, `Email Address`, `Phone Number`, `Property Type`, `Address`, `Approximate Square Feet`, and `Other Information`.
    - *Key Expressions/Variables:* 
      - `Full Name`: `={{ $json['Full Name'].first }} {{ $json['Full Name'].last }}`
      - `Phone Number`: `={{ $json['Phone Number'].full }}`
      - `Address`: `={{ $json['Property Address'].addr_line1 }}, {{ $json['Property Address'].addr_line2 }}, {{ $json['Property Address'].city }}, {{ $json['Property Address'].state }},{{ $json['Property Address'].postal }} , {{ $json['Property Address'].country }}`
    - *Connections:* Input: `When Quote Form Submitted`; Output: `Filter Required Info`.
    - *Version Requirements:* TypeVersion 3.5.
    - *Edge Cases/Failure Types:* Missing sub-properties in JotForm JSON payloads throwing undefined property access errors.

  - **Filter Required Info**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Flow control node). Halts execution if mandatory data points are missing.
    - *Configuration Choices:* Uses an `AND` combinator requiring `Full Name`, `Email Address`, `Phone Number`, `Property Type`, and `Approximate Square Feet` to be non-empty.
    - *Key Expressions/Variables:* Evaluates `$json['Full Name']`, `$json['Email Address']`, etc., for non-empty string conditions.
    - *Connections:* Input: `Set Submission Details`; Output: `Quote Analysis Agent`.
    - *Version Requirements:* TypeVersion 2.2.
    - *Edge Cases/Failure Types:* Empty strings or whitespace-only inputs bypassing loose validation if payloads change unexpectedly.

---

#### 2.2 AI Requirement Analysis
This block passes raw customer notes and property parameters to a Google Gemini model, enforcing a structured output schema to extract quantitative cleaning requirements.

- **Nodes Involved:**
  - `Quote Analysis Agent`
  - `Gemini Quote Analysis`
  - `Parse Quote Analysis Output`
  - `Normalize Requirements Data`

- **Node Details:**
  - **Quote Analysis Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI agent orchestrator). Manages the language model interaction and structured parser schema.
    - *Configuration Choices:* Prompt defines extraction constraints using customer notes, property type, and square footage.
    - *Key Expressions/Variables:* `Property Type`, `Approximate Square Feet`, `Address`, `Other Information`.
    - *Connections:* Inputs: `Filter Required Info`, `Gemini Quote Analysis`, `Parse Quote Analysis Output`; Output: `Normalize Requirements Data`.
    - *Version Requirements:* TypeVersion 2.1.
    - *Edge Cases/Failure Types:* AI hallucination of room counts or invalid schema matching.

  - **Gemini Quote Analysis**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Sub-node model provider). Provides the LLM backend for the analysis agent.
    - *Configuration Choices:* Uses model `models/gemini-3.1-flash-lite`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Output: Linked to `Quote Analysis Agent`.
    - *Version Requirements:* TypeVersion 1.
    - *Edge Cases/Failure Types:* API rate limits, authentication token expiration, or model deprecation.

  - **Parse Quote Analysis Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node parser). Enforces JSON schema validation on the LLM response.
    - *Configuration Choices:* Manually defined schema expecting `extra_bathrooms` (number), `extra_kitchens` (number), `special_requirements` (array of strings), `complexity_multiplier` (number), `service_type` (string), and `notes_summary` (string).
    - *Connections:* Output: Linked to `Quote Analysis Agent`.
    - *Version Requirements:* TypeVersion 1.2.
    - *Edge Cases/Failure Types:* LLM failure to return valid JSON matching the exact schema keys.

  - **Normalize Requirements Data**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation node). Consolidates original submission data with parsed AI outputs into a clean unified payload.
    - *Configuration Choices:* Maps original submission fields alongside parsed JSON outputs (`Extra Bathrooms`, `Extra Kitchens`, `Special Requirements`, `Complexity Multiplier`, `Service Type`, `Notes Summary`).
    - *Key Expressions/Variables:* `={{ $json.output.extra_bathrooms }}`, `={{ ($json.output.special_requirements || []).join(', ') }}`.
    - *Connections:* Input: `Quote Analysis Agent`; Output: `Compute Labor Hours`.
    - *Version Requirements:* TypeVersion 3.5.
    - *Edge Cases/Failure Types:* Missing output properties from the parser causing type casting errors.

---

#### 2.3 Mathematical Estimation & Sizing
This block calculates estimated labor hours and establishes financial quote boundaries while filtering out projects that fall outside operational sizing limits.

- **Nodes Involved:**
  - `Compute Labor Hours`
  - `Determine Quote Range`
  - `Filter Lead Quality`

- **Node Details:**
  - **Compute Labor Hours**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation/calculation node). Executes a deterministic math formula to estimate labor duration.
    - *Configuration Choices:* Calculates hours based on square footage, extra bathrooms, kitchens, complexity multipliers, and property type modifiers.
    - *Key Expressions/Variables:* 
      - `Estimated Labor Hours`: `={{ (((Number($json['Approximate Square Feet']) / 500) + (Number($json['Extra Bathrooms']) * 0.5) + (Number($json['Extra Kitchens']) * 1)) * Number($json['Complexity Multiplier']) * ($json['Property Type'] === 'Office' ? 1.15 : 1)).toFixed(2) }}`
    - *Connections:* Input: `Normalize Requirements Data`; Output: `Determine Quote Range`.
    - *Version Requirements:* TypeVersion 3.5.
    - *Edge Cases/Failure Types:* Division by zero or NaN results if square footage is passed as a non-numeric string.

  - **Determine Quote Range**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation/calculation node). Derives monetary estimates from calculated labor hours.
    - *Configuration Choices:* Sets hourly low ($45) and high ($65) rates and calculates quote totals.
    - *Key Expressions/Variables:*
      - `Quote Low`: `={{ (Number($json['Estimated Labor Hours']) * 45).toFixed(2) }}`
      - `Quote High`: `={{ (Number($json['Estimated Labor Hours']) * 65 + 50).toFixed(2) }}`
    - *Connections:* Input: `Compute Labor Hours`; Output: `Filter Lead Quality`.
    - *Version Requirements:* TypeVersion 3.5.
    - *Edge Cases/Failure Types:* Propagation of invalid labor hour inputs.

  - **Filter Lead Quality**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Flow control node). Validates that project scope matches operational capabilities.
    - *Configuration Choices:* Ensures square footage is between 200 and 50,000 sq ft, and estimated labor hours are greater than 0.
    - *Key Expressions/Variables:* Evaluates `Approximate Square Feet` and `Estimated Labor Hours` against numerical thresholds.
    - *Connections:* Input: `Determine Quote Range`; Output: `Estimator Summary Agent`.
    - *Version Requirements:* TypeVersion 2.2.
    - *Edge Cases/Failure Types:* Strict type validation failures if numbers are stored as strings.

---

#### 2.4 AI Summarization & Routing
This block generates a client-facing proposal email body, subject line, and lead score using a secondary AI agent, routing the execution based on lead value.

- **Nodes Involved:**
  - `Estimator Summary Agent`
  - `Gemini Estimator Summary`
  - `Parse Estimator Summary Output`
  - `If Lead Quality High`

- **Node Details:**
  - **Estimator Summary Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI agent orchestrator). Generates customer-facing quote emails and operational metrics.
    - *Configuration Choices:* Prompt instructs the agent to write a warm, mobile-friendly quotation email incorporating previously calculated quote ranges.
    - *Key Expressions/Variables:* References data across upstream nodes (`Set Submission Details`, `Quote Analysis Agent`, `Normalize Requirements Data`, and current node quote values).
    - *Connections:* Inputs: `Filter Lead Quality`, `Gemini Estimator Summary`, `Parse Estimator Summary Output`; Output: `If Lead Quality High`.
    - *Version Requirements:* TypeVersion 2.1.
    - *Edge Cases/Failure Types:* Exceeding context size or generating invalid HTML tags in email bodies.

  - **Gemini Estimator Summary**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Sub-node model provider). LLM backend for the summary agent.
    - *Configuration Choices:* Uses model `models/gemini-3.1-flash-lite`.
    - *Connections:* Output: Linked to `Estimator Summary Agent`.
    - *Version Requirements:* TypeVersion 1.
    - *Edge Cases/Failure Types:* API connectivity issues.

  - **Parse Estimator Summary Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node parser). Enforces schema structure for summary outputs.
    - *Configuration Choices:* Expects `subject_line`, `summary` (or `summary_html`), `urgency`, `recommended_next_step` (or `next_step`), and `lead_quality_score`.
    - *Connections:* Output: Linked to `Estimator Summary Agent`.
    - *Version Requirements:* TypeVersion 1.2.
    - *Edge Cases/Failure Types:* Key mismatch between schema definitions and prompt expectations.

  - **If Lead Quality High**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Branches execution based on lead score evaluation.
    - *Configuration Choices:* Evaluates whether `lead_quality_score` is strictly greater than `75`.
    - *Key Expressions/Variables:* `={{ $json.output.lead_quality_score }}`.
    - *Connections:* Input: `Estimator Summary Agent`; Output: `Update Sheet with Quote Data`.
    - *Version Requirements:* TypeVersion 2.3.
    - *Edge Cases/Failure Types:* Missing score property causing conditional evaluation errors.

---

#### 2.5 Data Persistence & Notification
This block logs qualified leads into a Google Spreadsheet and triggers an automated notification email to the prospective client via Gmail.

- **Nodes Involved:**
  - `Update Sheet with Quote Data`
  - `Send Email via Gmail`

- **Node Details:**
  - **Update Sheet with Quote Data**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Integration node). Appends or updates lead records in Google Sheets.
    - *Configuration Choices:* Operation set to `appendOrUpdate`, matched on column `Email Address`. Target spreadsheet ID: `14hN1PBy0zz3uI_tV7qqADtQdgHGyQF2yyRQJj1iTNm4`, Sheet: `Sheet1`.
    - *Key Expressions/Variables:* Maps fields including `Address`, `Full Name`, `next_step`, `Phone Number`, `subject_line`, `summary_html`, `Email Address`, `Property Type`, `urgency_rating`, `Other Information`, `lead_quality_score`, and `Approximate Square Feet`.
    - *Connections:* Input: `If Lead Quality High`; Output: `Send Email via Gmail`.
    - *Version Requirements:* TypeVersion 4.7.
    - *Edge Cases/Failure Types:* Google API rate limits, permission errors on service accounts, or missing match columns during update operations.

  - **Send Email via Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Integration node). Sends the final quote email to the client.
    - *Configuration Choices:* Uses Gmail OAuth2 credentials. Sets recipient to client email, subject to generated subject line, and message body to generated HTML summary.
    - *Key Expressions/Variables:* 
      - `sendTo`: `={{ $('Set Submission Details').item.json['Email Address'] }}`
      - `message`: `={{ $('Estimator Summary Agent').item.json.output.summary_html }}`
      - `subject`: `={{ $('Estimator Summary Agent').item.json.output.subject_line }}`
    - *Connections:* Input: `Update Sheet with Quote Data`; Output: None (Terminal node).
    - *Version Requirements:* TypeVersion 2.2.
    - *Edge Cases/Failure Types:* Invalid OAuth2 tokens, sending limits exceeded, or rejected recipient addresses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | ## Commercial Property Cleaning Quote Request Pipeline... @[youtube](5QmlxcOuuSg) |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation block for lead capture | None | None | ## Capture and validate lead<br><br>Receives a quote request from the website form, maps the key contact and property fields, and filters out submissions missing required information. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation block for requirement analysis | None | None | ## Analyze cleaning requirements<br><br>Uses Gemini with a structured output parser to interpret the submitted quote details, then normalizes the extracted requirements for later calculations. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation block for quote calculations | None | None | ## Calculate quote fit<br><br>Estimates labor hours, calculates the internal low and high quote range, and filters leads based on quality or job-fit criteria. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation block for summarization and routing | None | None | ## Summarize and route lead<br><br>Generates an estimator summary with a second AI agent and structured parser, then evaluates the result through a conditional branch before saving the lead. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation block for recording and notification | None | None | ## Record and notify<br><br>Writes the qualified quote request and estimate details to Google Sheets, then sends a Gmail message to notify the appropriate recipient. |
| `When Quote Form Submitted` | `n8n-nodes-base.jotFormTrigger` | Trigger for new JotForm submissions | None | `Set Submission Details` | ## Capture and validate lead... |
| `Set Submission Details` | `n8n-nodes-base.set` | Flattens raw Jotform payload structure | `When Quote Form Submitted` | `Filter Required Info` | ## Capture and validate lead... |
| `Filter Required Info` | `n8n-nodes-base.filter` | Validates presence of core lead data | `Set Submission Details` | `Quote Analysis Agent` | ## Capture and validate lead... |
| `Quote Analysis Agent` | `@n8n/n8n-nodes-langchain.agent` | AI agent for extracting cleaning requirements | `Filter Required Info` | `Normalize Requirements Data` | ## Analyze cleaning requirements... |
| `Gemini Quote Analysis` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM model provider for requirement analysis | None | `Quote Analysis Agent` | ## Analyze cleaning requirements... |
| `Parse Quote Analysis Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Schema validator for requirement analysis | None | `Quote Analysis Agent` | ## Analyze cleaning requirements... |
| `Normalize Requirements Data` | `n8n-nodes-base.set` | Merges submission data with AI requirements | `Quote Analysis Agent` | `Compute Labor Hours` | ## Analyze cleaning requirements... |
| `Compute Labor Hours` | `n8n-nodes-base.set` | Calculates estimated labor hours | `Normalize Requirements Data` | `Determine Quote Range` | ## Calculate quote fit... |
| `Determine Quote Range` | `n8n-nodes-base.set` | Calculates internal low/high quote pricing | `Compute Labor Hours` | `Filter Lead Quality` | ## Calculate quote fit... |
| `Filter Lead Quality` | `n8n-nodes-base.filter` | Filters leads by sq ft and labor hour rules | `Determine Quote Range` | `Estimator Summary Agent` | ## Calculate quote fit... |
| `Estimator Summary Agent` | `@n8n/n8n-nodes-langchain.agent` | AI agent to generate client proposal email | `Filter Lead Quality` | `If Lead Quality High` | ## Summarize and route lead... |
| `Gemini Estimator Summary` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM model provider for estimator summary | None | `Estimator Summary Agent` | ## Summarize and route lead... |
| `Parse Estimator Summary Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Schema validator for estimator summary | None | `Estimator Summary Agent` | ## Summarize and route lead... |
| `If Lead Quality High` | `n8n-nodes-base.if` | Evaluates lead quality score threshold | `Estimator Summary Agent` | `Update Sheet with Quote Data` | ## Summarize and route lead... |
| `Update Sheet with Quote Data` | `n8n-nodes-base.googleSheets` | Logs/updates quote records in Google Sheets | `If Lead Quality High` | `Send Email via Gmail` | ## Record and notify... |
| `Send Email via Gmail` | `n8n-nodes-base.gmail` | Sends quotation email via Gmail | `Update Sheet with Quote Data` | None | ## Record and notify... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **JotForm Trigger** node named `When Quote Form Submitted`.
   - Set the `Form ID` to your target form (e.g., `262593642323054`).
   - Configure and select a valid **JotForm API** credential.

2. **Add Data Flattening:**
   - Add a **Set** node named `Set Submission Details` connected downstream from the JotForm trigger.
   - Configure string assignments to map nested object fields:
     - `Full Name`: `={{ $json['Full Name'].first }} {{ $json['Full Name'].last }}`
     - `Email Address`: `={{ $json['Email Address'] }}`
     - `Phone Number`: `={{ $json['Phone Number'].full }}`
     - `Property Type`: `={{ $json['Property Type'] }}`
     - `Address`: `={{ $json['Property Address'].addr_line1 }}, {{ $json['Property Address'].addr_line2 }}, {{ $json['Property Address'].city }}, {{ $json['Property Address'].state }},{{ $json['Property Address'].postal }} , {{ $json['Property Address'].country }}`
     - `Approximate Square Feet`: `={{ $json['Approximate Square Feet'] }}`
     - `Other Information`: `={{ $json['Other Information'] }}`

3. **Add Initial Validation Filter:**
   - Add a **Filter** node named `Filter Required Info`.
   - Set conditions to verify that `Full Name`, `Email Address`, `Phone Number`, `Property Type`, and `Approximate Square Feet` are all not empty.

4. **Set Up AI Requirement Analysis Agent:**
   - Add an **AI Agent** node named `Quote Analysis Agent`.
   - Connect a **Google Gemini Chat Model** sub-node (`Gemini Quote Analysis`) using model `models/gemini-3.1-flash-lite` and a valid Google Gemini/Palm API credential.
   - Connect a **Structured Output Parser** sub-node (`Parse Quote Analysis Output`) with a manual JSON schema defining: `extra_bathrooms` (number), `extra_kitchens` (number), `special_requirements` (array), `complexity_multiplier` (number), `service_type` (string), and `notes_summary` (string).
   - Configure the agent prompt to extract structured cleaning job requirements from the incoming property details and customer notes.

5. **Normalize Extracted Data:**
   - Add a **Set** node named `Normalize Requirements Data`.
   - Map original submission items using reference expressions (e.g., `{{ $('Set Submission Details').item.json['Full Name'] }}`) alongside AI parser output attributes (`{{ $json.output.extra_bathrooms }}`).

6. **Implement Labor and Pricing Logic:**
   - Add a **Set** node named `Compute Labor Hours` to calculate estimated labor hours using the square footage, bathroom/kitchen counts, and complexity multiplier.
   - Add a second **Set** node named `Determine Quote Range` to calculate `Quote Low` (`Estimated Labor Hours * 45`) and `Quote High` (`Estimated Labor Hours * 65 + 50`).

7. **Add Lead Qualification Filter:**
   - Add a **Filter** node named `Filter Lead Quality`.
   - Set conditions ensuring square footage is between `200` and `50000`, and estimated labor hours are greater than `0`.

8. **Set Up Estimator Summary Agent:**
   - Add an **AI Agent** node named `Estimator Summary Agent`.
   - Connect a **Google Gemini Chat Model** sub-node (`Gemini Estimator Summary`) and a **Structured Output Parser** sub-node (`Parse Estimator Summary Output`) with schema properties for `subject_line`, `summary` (or `summary_html`), `urgency`, `recommended_next_step` (or `next_step`), and `lead_quality_score`.
   - Configure the prompt to write a professional quotation email using client data and calculated quote ranges.

9. **Add Conditional Branching:**
   - Add an **If** node named `If Lead Quality High`.
   - Set a condition checking if `{{ $json.output.lead_quality_score }}` is greater than `75`.

10. **Configure Integrations (Google Sheets & Gmail):**
    - Add a **Google Sheets** node named `Update Sheet with Quote Data`:
      - Set operation to `appendOrUpdate`, authentication to Google Service Account, and configure matching columns to use `Email Address`.
      - Select the target Document ID and Sheet Name (`Sheet1`), then map fields for name, email, phone, address, summary, urgency, and score.
    - Add a **Gmail** node named `Send Email via Gmail`:
      - Connect Gmail OAuth2 credentials.
      - Set recipient (`sendTo`) to `={{ $('Set Submission Details').item.json['Email Address'] }}`, subject to `={{ $('Estimator Summary Agent').item.json.output.subject_line }}`, and message body to `={{ $('Estimator Summary Agent').item.json.output.summary_html }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| YouTube Video Walkthrough | [https://youtu.be/5QmlxcOuuSg](https://youtu.be/5QmlxcOuuSg) |