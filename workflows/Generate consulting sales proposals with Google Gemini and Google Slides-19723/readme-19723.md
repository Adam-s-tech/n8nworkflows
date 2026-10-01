Generate consulting sales proposals with Google Gemini and Google Slides

https://n8nworkflows.xyz/workflows/generate-consulting-sales-proposals-with-google-gemini-and-google-slides-19723


# Generate consulting sales proposals with Google Gemini and Google Slides

### 1. Workflow Overview

This workflow automates the generation of professional consulting sales proposals for Optinov'ai. It accepts client intake data from multiple sources, extracts and normalizes a structured client dossier using Google Gemini, evaluates and prices matching consultant profiles via Google Sheets catalogs, validates the proposal quality, remediates rejected profiles, and finally generates customized Google Slides decks for each recommended profile.

The execution logic is grouped into the following functional blocks:
- **1.1 Input Reception & Dossier Extraction:** Captures incoming meeting transcripts via webhook or new audio/PDF files from Google Drive, then extracts a structured client dossier using Google Gemini.
- **1.2 Dossier Normalization & Qualification:** Cleans, parses, and validates the extracted raw JSON data, routing unreadable dossiers to a handling node.
- **1.3 Proposal Generation & AI Agents:** Employs AI agents integrated with Google Sheets lookup tools to select matching consultants, determine pricing (régie, forfait, or options), and draft a three-tier proposal.
- **1.4 Proposal Validation & Quality Control:** Performs automated quality checks on the drafted proposal, classifying results by verdict.
- **1.5 Profile Remediation:** Corrects and replaces rejected profiles using a dedicated remediation agent and filters out successful fixes.
- **1.6 Profile Loop & Google Slides Generation:** Iterates through validated profiles, backfills deterministic catalog data, duplicates a Google Slides presentation template, calculates financial totals, replaces text placeholders, and injects images via the Google Slides API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Dossier Extraction
- **Overview:** Receives raw client discovery data from an Upmeet webhook, Google Drive audio files, or Google Drive PDF uploads, and feeds them into specialized Google Gemini nodes to extract a standardized BEBEDC-formatted JSON dossier.
- **Nodes Involved:** 
  - `When Upmeet Webhook Received`
  - `When Audio File Uploaded`
  - `When PDF File Uploaded`
  - `Download Audio from Drive`
  - `Download PDF from Drive`
  - `Extract Transcript from Upmeet`
  - `Extract Data from Audio`
  - `Extract Data from PDF`
- **Node Details:**
  - **When Upmeet Webhook Received**
    - Type and technical role: Webhook trigger (`n8n-nodes-base.webhook`) capturing POST requests.
    - Configuration choices: Path `upmeet-optinovai`, HTTP method `POST`.
    - Input connections: None (Trigger). Output connections: `Extract Transcript from Upmeet`.
    - Edge cases/Failure types: Invalid payload structure or network timeout from Upmeet.
  - **When Audio File Uploaded**
    - Type and technical role: Google Drive Trigger (`n8n-nodes-base.googleDriveTrigger`) polling for new files.
    - Configuration choices: Event `fileCreated`, specific folder ID `REPLACE_WITH_DRIVE_FOLDER_ID_AUDIO`, polling interval every minute.
    - Input connections: None (Trigger). Output connections: `Download Audio from Drive`.
    - Sub-workflow reference: Uses Google Drive OAuth2 credentials (`xJn2ssNDGR8wzRwp`).
  - **When PDF File Uploaded**
    - Type and technical role: Google Drive Trigger (`n8n-nodes-base.googleDriveTrigger`) polling for new files.
    - Configuration choices: Event `fileCreated`, specific folder ID `REPLACE_WITH_DRIVE_FOLDER_ID_PDF`.
    - Input connections: None (Trigger). Output connections: `Download PDF from Drive`.
  - **Download Audio from Drive** / **Download PDF from Drive**
    - Type and technical role: Google Drive node (`n8n-nodes-base.googleDrive`) downloading file binaries.
    - Configuration choices: Operation `download`, file ID set dynamically via `={{ $json.id }}` with max 3 retries.
    - Input connections: respective drive triggers. Output connections: respective extraction nodes.
  - **Extract Transcript from Upmeet**
    - Type and technical role: Google Gemini node (`@n8n/n8n-nodes-langchain.googleGemini`) for text processing.
    - Configuration choices: Model `models/gemini-3.1-pro-preview`, max 3 retries.
    - Input connections: `When Upmeet Webhook Received`. Output connections: `Normalize Client Data`.
    - Credentials: Google Palm API (`I9mGYPSQmT2bOw6j`).
  - **Extract Data from Audio**
    - Type and technical role: Google Gemini node (`@n8n/n8n-nodes-langchain.googleGemini`) analyzing binary audio.
    - Configuration choices: Resource `audio`, input type `binary`, model `models/gemini-3.1-pro-preview`. Uses system prompt applying the BEBEDC methodology.
    - Input connections: `Download Audio from Drive`. Output connections: `Normalize Client Data`.
  - **Extract Data from PDF**
    - Type and technical role: Google Gemini node (`@n8n/n8n-nodes-langchain.googleGemini`) analyzing binary documents.
    - Configuration choices: Resource `document`, input type `binary`, model `models/gemini-2.5-flash`.
    - Input connections: `Download PDF from Drive`. Output connections: `Normalize Client Data`.

#### 1.2 Dossier Normalization & Qualification
- **Overview:** Consolidates outputs from all three intake channels, strips markdown formatting, parses the JSON payload safely, and evaluates if the dossier is complete enough to proceed.
- **Nodes Involved:**
  - `Normalize Client Data`
  - `Check if Dossier Usable`
  - `Handle Unreadable Dossier`
- **Node Details:**
  - **Normalize Client Data**
    - Type and technical role: Code node (`n8n-nodes-base.code`) running custom JavaScript.
    - Configuration choices: Sanitizes markdown code fences, fixes trailing commas, handles quotes, assigns default schema structures, and labels the source (`upmeet`, `audio`, or `pdf`).
    - Input connections: `Extract Transcript from Upmeet`, `Extract Data from Audio`, `Extract Data from PDF`.
    - Output connections: `Check if Dossier Usable`.
    - Edge cases: Throws parsing errors captured into an error JSON object if Gemini returns malformed strings.
  - **Check if Dossier Usable**
    - Type and technical role: If node (`n8n-nodes-base.if`).
    - Configuration choices: Evaluates `={{ $json.error === true ? 'ko' : 'ok' }}`.
    - Input connections: `Normalize Client Data`.
    - Output connections: True branch to `Proposal Drafting Agent`, False branch to `Handle Unreadable Dossier`.
  - **Handle Unreadable Dossier**
    - Type and technical role: No-Op node (`n8n-nodes-base.noOp`) acting as a termination point for corrupted/unusable client inputs.
    - Input connections: `Check if Dossier Usable`. Output connections: None.

#### 1.3 Proposal Generation & AI Agents
- **Overview:** Uses advanced AI agents backed by Google Sheets catalog tools to select matching consultants, retrieve pricing models, and draft the structured commercial proposal.
- **Nodes Involved:**
  - `Get Available Consultants Profiles`
  - `Get TJM Regie Rates`
  - `Get Forfait Rates`
  - `Consultant Selection Agent`
  - `Run Consultant Selection Model`
  - `Parse Consultant Output`
  - `Get One-time Options`
  - `Get Monthly Options`
  - `Pricing Options Agent`
  - `Run Pricing Model for Options`
  - `Parse Pricing Output`
  - `Proposal Drafting Agent`
  - `Run Proposal Build Model`
  - `Parse Proposal Data`
- **Node Details:**
  - **Google Sheets Tool Nodes** (`Get Available Consultants Profiles`, `Get TJM Regie Rates`, `Get Forfait Rates`, `Get One-time Options`, `Get Monthly Options`)
    - Type and technical role: Google Sheets Tool (`n8n-nodes-base.googleSheetsTool`) exposing spreadsheet tables as AI tools.
    - Configuration choices: Each points to specific spreadsheet IDs (`REPLACE_WITH_SHEET_ID_*`) with descriptive tool instructions.
    - Input/Output connections: Linked as AI tools (`ai_tool`) to `Consultant Selection Agent` or `Pricing Options Agent`.
    - Credentials: Google Sheets OAuth2 (`UJteTVcnnf1jHGcp`).
  - **Consultant Selection Agent** & **Pricing Options Agent**
    - Type and technical role: LangChain Agent Tools (`@n8n/n8n-nodes-langchain.agentTool` / `agent`).
    - Configuration choices: Configured with structured system prompts, binding specific LLM models and output parsers (`Parse Consultant Output`, `Parse Pricing Output`).
    - Input/Output connections: Connects language models via `ai_languageModel`, outputs to `Proposal Drafting Agent`.
  - **Proposal Drafting Agent**
    - Type and technical role: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`).
    - Configuration choices: Maximum 15 iterations, temperature 0 via `Run Proposal Build Model`, outputs structured JSON via `Parse Proposal Data`.
    - Input connections: `Check if Dossier Usable`. Output connections: `Validate Proposal Agent`.

#### 1.4 Proposal Validation & Quality Control
- **Overview:** Runs an independent AI quality controller that cross-checks the drafted proposal against the original client dossier for consistency, omissions, and compliance.
- **Nodes Involved:**
  - `Validate Proposal Agent`
  - `Run Proposal Validation Model`
  - `Parse Validation Data`
- **Node Details:**
  - **Validate Proposal Agent**
    - Type and technical role: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`).
    - Configuration choices: Uses `Run Proposal Validation Model` (`models/gemini-3.1-pro-preview`, temperature 0) and `Parse Validation Data` output parser.
    - Input connections: `Proposal Drafting Agent`. Output connections: `Flatten Consultant Data`.

#### 1.5 Profile Remediation
- **Overview:** Isolates non-compliant profiles, runs a remediation workflow to find substitute consultants, and validates successful fixes.
- **Nodes Involved:**
  - `Flatten Consultant Data`
  - `Route by Dossier Verdict`
  - `Proposal Correction Agent`
  - `Google Gemini Discussion Model`
  - `Parse Structured Output`
  - `Filter Successful Fixes`
  - `Flatten Replacement Data`
  - `Merge Validation and Replacement`
  - `Verdict inattendu - a traiter1`
- **Node Details:**
  - **Flatten Consultant Data**
    - Type and technical role: Code node (`n8n-nodes-base.code`) flattening validation and proposal results into individual profile items.
    - Input connections: `Validate Proposal Agent`. Output connections: `Route by Dossier Verdict`.
  - **Route by Dossier Verdict**
    - Type and technical role: Switch node (`n8n-nodes-base.switch`).
    - Configuration choices: Evaluates `={{ $json.verdict_machine }}` for rules `conforme` and `pas conforme`, with a fallback output.
    - Input connections: `Flatten Consultant Data`. Output connections: `Merge Validation and Replacement` (conforme), `Proposal Correction Agent` (pas conforme), and `Verdict inattendu - a traiter1` (fallback).
  - **Proposal Correction Agent**
    - Type and technical role: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) executing remediation logic for rejected profiles using `Google Gemini Discussion Model` and `Parse Structured Output`.
    - Input connections: `Route by Dossier Verdict`. Output connections: `Filter Successful Fixes`.
  - **Filter Successful Fixes** & **Flatten Replacement Data**
    - Type and technical role: Filter and Code nodes ensuring only valid remediation outputs proceed.
    - Output connections: `Merge Validation and Replacement` (index 1).
  - **Merge Validation and Replacement**
    - Type and technical role: Merge node (`n8n-nodes-base.merge`) combining accepted profiles and successful replacements.
    - Input connections: `Route by Dossier Verdict` and `Flatten Replacement Data`. Output connections: `Check at Least One Profile`.

#### 1.6 Profile Loop & Google Slides Generation
- **Overview:** Verifies that at least one usable profile exists, then loops through each consultant, backfills catalog details, duplicates the Google Slides template, calculates financial figures, updates text placeholders, and injects images.
- **Nodes Involved:**
  - `Check at Least One Profile`
  - `Handle No Useable Profiles`
  - `Loop Through Profiles`
  - `Search Profile Table`
  - `Fill Missing Catalog Data`
  - `Duplicate Drive File`
  - `Fetch Presentation Layout`
  - `Calculate Slide Totals`
  - `Update Presentation Text`
  - `Create Image Requests`
  - `Add Images to Slides`
  - `Wait for Next Profile`
- **Node Details:**
  - **Check at Least One Profile**
    - Type and technical role: If node (`n8n-nodes-base.if`).
    - Input connections: `Merge Validation and Replacement`. Output connections: True to `Loop Through Profiles`, False to `Handle No Useable Profiles`.
  - **Loop Through Profiles**
    - Type and technical role: Split In Batches node (`n8n-nodes-base.splitInBatches`) processing profiles iteratively.
    - Input connections: `Check at Least One Profile` (and `Add Images to Slides` via loopback). Output connections: `Search Profile Table`.
  - **Search Profile Table**
    - Type and technical role: Google Sheets node (`n8n-nodes-base.googleSheets`) querying catalog records.
    - Configuration choices: Document ID `REPLACE_WITH_SHEET_ID_PROFILS`, operation `get` / search.
    - Input connections: `Loop Through Profiles`. Output connections: `Fill Missing Catalog Data`.
  - **Fill Missing Catalog Data**
    - Type and technical role: Code node (`n8n-nodes-base.code`) deterministically backfilling missing consultant attributes (photos, pictos, seniorities) from the sheet row.
    - Input connections: `Search Profile Table`. Output connections: `Duplicate Drive File`.
  - **Duplicate Drive File**
    - Type and technical role: Google Drive node (`n8n-nodes-base.googleDrive`) copying the template deck.
    - Configuration choices: Operation `copy`, file ID `REPLACE_WITH_SLIDES_TEMPLATE_ID`, destination folder `REPLACE_WITH_DRIVE_FOLDER_ID_OUTPUT`, dynamic file naming: `Offre commerciale Optinov'ai - {{ $json.client }} - {{ $json.positionnement }}`.
    - Input connections: `Fill Missing Catalog Data`. Output connections: `Fetch Presentation Layout`.
  - **Fetch Presentation Layout**
    - Type and technical role: HTTP Request node (`n8n-nodes-base.httpRequest`) calling the Google Slides API.
    - Configuration choices: GET `https://slides.googleapis.com/v1/presentations/{{ $json.id }}` using Google Slides OAuth2 credentials (`a0p8owiqm3G5fNG3`).
    - Input connections: `Duplicate Drive File`. Output connections: `Calculate Slide Totals`.
  - **Calculate Slide Totals**
    - Type and technical role: Code node (`n8n-nodes-base.code`) calculating financial amounts (TJM × days + options).
    - Input connections: `Fetch Presentation Layout`. Output connections: `Update Presentation Text`.
  - **Update Presentation Text**
    - Type and technical role: Google Slides node (`n8n-nodes-base.googleSlides`) performing text placeholder replacements.
    - Configuration choices: Replaces tokens like `{{ENTREPRISE}}`, `{{PROFIL}}`, `{{TJM_HT}}`, `{{BESOINS_CLIENT}}`, etc.
    - Input connections: `Calculate Slide Totals`. Output connections: `Create Image Requests`.
  - **Create Image Requests**
    - Type and technical role: Code node (`n8n-nodes-base.code`) building batch image update payloads for photos and skill pictograms.
    - Input connections: `Update Presentation Text`. Output connections: `Add Images to Slides`.
  - **Add Images to Slides**
    - Type and technical role: HTTP Request node (`n8n-nodes-base.httpRequest`) sending batch update requests to the Google Slides API (`...:batchUpdate`).
    - Input connections: `Create Image Requests`. Output connections: `Wait for Next Profile`.
  - **Wait for Next Profile**
    - Type and technical role: Wait node (`n8n-nodes-base.wait`) pausing for 2 seconds to respect API rate limits before looping back to `Loop Through Profiles`.
    - Input connections: `Add Images to Slides`. Output connections: `Loop Through Profiles`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | stickyNote | Documentation | None | None | Optinov'ai - Génération Offre Commerciale Consulting<br>How it works<br>Setup steps<br>Customization |
| `Sticky Note1` | stickyNote | Documentation | None | None | Upmeet transcript intake |
| `Sticky Note2` | stickyNote | Documentation | None | None | Audio file intake |
| `Sticky Note3` | stickyNote | Documentation | None | None | PDF file intake |
| `Sticky Note4` | stickyNote | Documentation | None | None | Normalize and qualify dossier |
| `Sticky Note5` | stickyNote | Documentation | None | None | Build and validate proposal |
| `Sticky Note6` | stickyNote | Documentation | None | None | Consultant selector tool |
| `Sticky Note7` | stickyNote | Documentation | None | None | Options pricing tool |
| `Sticky Note8` | stickyNote | Documentation | None | None | Route validation verdicts |
| `Sticky Note9` | stickyNote | Documentation | None | None | Remediate rejected profiles |
| `Sticky Note10` | stickyNote | Documentation | None | None | Check profiles and loop |
| `Sticky Note11` | stickyNote | Documentation | None | None | Enrich profile template |
| `Sticky Note12` | stickyNote | Documentation | None | None | Prepare slide content |
| `Sticky Note13` | stickyNote | Documentation | None | None | Insert slide images |
| `When Upmeet Webhook Received` | webhook | Webhook Trigger | None | `Extract Transcript from Upmeet` | Upmeet transcript intake |
| `When Audio File Uploaded` | googleDriveTrigger | Drive Trigger | None | `Download Audio from Drive` | Audio file intake |
| `When PDF File Uploaded` | googleDriveTrigger | Drive Trigger | None | `Download PDF from Drive` | PDF file intake |
| `Download Audio from Drive` | googleDrive | File Download | `When Audio File Uploaded` | `Extract Data from Audio` | Audio file intake |
| `Download PDF from Drive` | googleDrive | File Download | `When PDF File Uploaded` | `Extract Data from PDF` | PDF file intake |
| `Extract Transcript from Upmeet` | googleGemini | AI Extraction | `When Upmeet Webhook Received` | `Normalize Client Data` | Upmeet transcript intake |
| `Extract Data from Audio` | googleGemini | AI Extraction | `Download Audio from Drive` | `Normalize Client Data` | Audio file intake |
| `Extract Data from PDF` | googleGemini | AI Extraction | `Download PDF from Drive` | `Normalize Client Data` | PDF file intake |
| `Normalize Client Data` | code | Data Normalization | Extract nodes | `Check if Dossier Usable` | Normalize and qualify dossier |
| `Check if Dossier Usable` | if | Validation Branch | `Normalize Client Data` | `Proposal Drafting Agent`<br>`Handle Unreadable Dossier` | Normalize and qualify dossier |
| `Handle Unreadable Dossier` | noOp | Termination | `Check if Dossier Usable` | None | Normalize and qualify dossier |
| `Get Available Consultants Profiles` | googleSheetsTool | AI Tool (Catalog) | None | `Consultant Selection Agent` | Build and validate proposal |
| `Get TJM Regie Rates` | googleSheetsTool | AI Tool (TJM) | None | `Consultant Selection Agent` | Build and validate proposal |
| `Get Forfait Rates` | googleSheetsTool | AI Tool (Forfait) | None | `Consultant Selection Agent` | Build and validate proposal |
| `Consultant Selection Agent` | agentTool | AI Agent Tool | Sheet tools | Drafting / Correction agents | Consultant selector tool |
| `Run Consultant Selection Model` | lmChatGoogleGemini | LLM Model | None | `Consultant Selection Agent` | Build and validate proposal |
| `Parse Consultant Output` | outputParserStructured | Output Parser | `Google Gemini Chat` | `Consultant Selection Agent` | Build and validate proposal |
| `Get One-time Options` | googleSheetsTool | AI Tool (Options) | None | `Pricing Options Agent` | Options pricing tool |
| `Get Monthly Options` | googleSheetsTool | AI Tool (Options) | None | `Pricing Options Agent` | Options pricing tool |
| `Pricing Options Agent` | agentTool | AI Agent Tool | Sheet tools | Drafting / Correction agents | Options pricing tool |
| `Run Pricing Model for Options` | lmChatGoogleGemini | LLM Model | None | `Pricing Options Agent` | Options pricing tool |
| `Parse Pricing Output` | outputParserStructured | Output Parser | None | `Pricing Options Agent` | Options pricing tool |
| `Proposal Drafting Agent` | agent | AI Proposal Builder | `Check if Dossier Usable` | `Validate Proposal Agent` | Build and validate proposal |
| `Run Proposal Build Model` | lmChatGoogleGemini | LLM Model | None | `Proposal Drafting Agent` | Build and validate proposal |
| `Parse Proposal Data` | outputParserStructured | Output Parser | None | `Proposal Drafting Agent` | Build and validate proposal |
| `Validate Proposal Agent` | agent | AI Quality Control | `Proposal Drafting Agent` | `Flatten Consultant Data` | Build and validate proposal |
| `Run Proposal Validation Model` | lmChatGoogleGemini | LLM Model | None | `Validate Proposal Agent` | Build and validate proposal |
| `Parse Validation Data` | outputParserStructured | Output Parser | None | `Validate Proposal Agent` | Build and validate proposal |
| `Flatten Consultant Data` | code | Data Transformation | `Validate Proposal Agent` | `Route by Dossier Verdict` | Route validation verdicts |
| `Route by Dossier Verdict` | switch | Verdict Routing | `Flatten Consultant Data` | Merge, Correction Agent, No-Op | Route validation verdicts |
| `Proposal Correction Agent` | agent | AI Remediation | `Route by Dossier Verdict` | `Filter Successful Fixes` | Remediate rejected profiles |
| `Google Gemini Discussion Model` | lmChatGoogleGemini | LLM Model | None | `Proposal Correction Agent` | Remediate rejected profiles |
| `Parse Structured Output` | outputParserStructured | Output Parser | None | `Proposal Correction Agent` | Remediate rejected profiles |
| `Filter Successful Fixes` | filter | Result Filtering | `Proposal Correction Agent` | `Flatten Replacement Data` | Remediate rejected profiles |
| `Flatten Replacement Data` | code | Data Transformation | `Filter Successful Fixes` | `Merge Validation and Replacement` | Remediate rejected profiles |
| `Merge Validation and Replacement` | merge | Data Merging | Verdict Router & Replacement | `Check at Least One Profile` | Route validation verdicts |
| `Check at Least One Profile` | if | Profile Check | `Merge Validation and Replacement` | Loop or No-Op | Check profiles and loop |
| `Handle No Useable Profiles` | noOp | Termination | `Check at Least One Profile` | None | Check profiles and loop |
| `Loop Through Profiles` | splitInBatches | Batch Looping | Check / Wait nodes | `Search Profile Table` | Check profiles and loop |
| `Search Profile Table` | googleSheets | Sheet Lookup | `Loop Through Profiles` | `Fill Missing Catalog Data` | Enrich profile template |
| `Fill Missing Catalog Data` | code | Data Backfill | `Search Profile Table` | `Duplicate Drive File` | Enrich profile template |
| `Duplicate Drive File` | googleDrive | Drive File Copy | `Fill Missing Catalog Data` | `Fetch Presentation Layout` | Enrich profile template |
| `Fetch Presentation Layout` | httpRequest | API Presentation Read | `Duplicate Drive File` | `Calculate Slide Totals` | Prepare slide content |
| `Calculate Slide Totals` | code | Financial Calculation | `Fetch Presentation Layout` | `Update Presentation Text` | Prepare slide content |
| `Update Presentation Text` | googleSlides | Slides Text Update | `Calculate Slide Totals` | `Create Image Requests` | Prepare slide content |
| `Create Image Requests` | code | Image Request Builder | `Update Presentation Text` | `Add Images to Slides` | Insert slide images |
| `Add Images to Slides` | httpRequest | API Slides Update | `Create Image Requests` | `Wait for Next Profile` | Insert slide images |
| `Wait for Next Profile` | wait | Rate-limit Pause | `Add Images to Slides` | `Loop Through Profiles` | Insert slide images |
| `Google Gemini Chat` | lmChatGoogleGemini | LLM Model | None | `Parse Consultant Output` | Build and validate proposal |
| `Verdict inattendu - a traiter1` | noOp | Fallback Termination | `Route by Dossier Verdict` | None | Route validation verdicts |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Trigger Nodes:**
   - Add a **Webhook** node (`When Upmeet Webhook Received`): Set HTTP Method to `POST`, path to `upmeet-optinovai`.
   - Add two **Google Drive Trigger** nodes (`When Audio File Uploaded`, `When PDF File Uploaded`): Configure event to `fileCreated`, polling `everyMinute`, and folder IDs (`REPLACE_WITH_DRIVE_FOLDER_ID_AUDIO`, `REPLACE_WITH_DRIVE_FOLDER_ID_PDF`). Connect Google Drive OAuth2 credentials.

2. **Add File Download & AI Extraction Nodes:**
   - Connect Google Drive download nodes (`Download Audio from Drive`, `Download PDF from Drive`) to respective triggers, operating on file IDs `={{ $json.id }}` with max 3 retries.
   - Add Google Gemini nodes (`Extract Transcript from Upmeet`, `Extract Data from Audio`, `Extract Data from PDF`) configured with Google Palm API credentials, selecting appropriate models (`models/gemini-3.1-pro-preview` or `models/gemini-2.5-flash`) and applying the BEBEDC extraction prompts.

3. **Normalize and Validate Dossier:**
   - Add a **Code** node (`Normalize Client Data`) to sanitize markdown fences, parse JSON, and handle source tagging. Connect all extraction outputs to this node.
   - Add an **If** node (`Check if Dossier Usable`) evaluating `={{ $json.error === true ? 'ko' : 'ok' }}`. Connect the true branch to the Proposal Drafting Agent and false to a **No-Op** node (`Handle Unreadable Dossier`).

4. **Setup AI Agents, Tools, and Parsers:**
   - Create Google Sheets Tool nodes (`Get Available Consultants Profiles`, `Get TJM Regie Rates`, `Get Forfait Rates`, `Get One-time Options`, `Get Monthly Options`) pointing to your respective spreadsheet IDs (`REPLACE_WITH_SHEET_ID_*`) and Google Sheets OAuth2 credentials.
   - Set up the **Consultant Selection Agent** and **Pricing Options Agent** using LangChain agent tools, paired with Google Gemini chat models (`models/gemini-3.1-pro-preview`) and structured output parsers.
   - Add the **Proposal Drafting Agent** and **Validate Proposal Agent** nodes, configuring their system prompts, Gemini chat models, and JSON schema output parsers.

5. **Configure Verdict Routing & Remediation:**
   - Add a **Code** node (`Flatten Consultant Data`) and a **Switch** node (`Route by Dossier Verdict`) to route based on `={{ $json.verdict_machine }}` (`conforme` vs `pas conforme`).
   - Configure the **Proposal Correction Agent** with its Gemini discussion model and structured parser to remediate rejected profiles, followed by a **Filter** node (`Filter Successful Fixes`) and **Code** node (`Flatten Replacement Data`).
   - Merge accepted and replacement profiles using a **Merge** node (`Merge Validation and Replacement`).

6. **Build the Slide Generation Loop:**
   - Add an **If** node (`Check at Least One Profile`) to verify usable profile existence.
   - Add a **Split In Batches** node (`Loop Through Profiles`) to loop over profiles.
   - Inside the loop:
     - Query the profile sheet using a **Google Sheets** node (`Search Profile Table`).
     - Backfill fields using a **Code** node (`Fill Missing Catalog Data`).
     - Duplicate the template deck using a **Google Drive** node (`Duplicate Drive File`), setting file ID `REPLACE_WITH_SLIDES_TEMPLATE_ID`, destination folder `REPLACE_WITH_DRIVE_FOLDER_ID_OUTPUT`, and dynamic naming.
     - Fetch layout via an **HTTP Request** node (`Fetch Presentation Layout`) using Google Slides OAuth2 credentials.
     - Calculate financial figures using a **Code** node (`Calculate Slide Totals`).
     - Replace text tokens using a **Google Slides** node (`Update Presentation Text`).
     - Build image requests in a **Code** node (`Create Image Requests`) and execute them via an **HTTP Request** node (`Add Images to Slides`) hitting the Slides `:batchUpdate` endpoint.
     - Pause execution with a **Wait** node (`Wait for Next Profile`) set to 2 seconds, then loop back to `Loop Through Profiles`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Optinov'ai Consulting Proposal Generation Solution | Automated workflow converting discovery audio/PDFs into custom Google Slides proposals. |
| Google Workspace API Requirements | Requires active Google Drive, Google Sheets, Google Slides APIs, and Google Gemini (PaLM) API credentials. |