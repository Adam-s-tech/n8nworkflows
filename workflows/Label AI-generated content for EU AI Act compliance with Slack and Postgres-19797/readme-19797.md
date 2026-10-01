Label AI-generated content for EU AI Act compliance with Slack and Postgres

https://n8nworkflows.xyz/workflows/label-ai-generated-content-for-eu-ai-act-compliance-with-slack-and-postgres-19797


# Label AI-generated content for EU AI Act compliance with Slack and Postgres

### 1. Workflow Overview

This workflow automates the collection, compliance classification, labeling, human review, and auditing of AI-generated content (text, image, audio, and video) under Article 50 of the EU AI Act. It provides a standardized pipeline that evaluates assets for disclosure obligations, applies automated text modifications or media burn-in routines, manages a human-in-the-loop review gate, and records an immutable audit trail in a PostgreSQL database alongside an instance-wide AI systems registry snapshot.

The system is organized into eleven functional blocks:
- **1.1 Input Reception:** Captures incoming AI assets via an n8n Form trigger or an external sub-workflow call, generating a unique prompt hash for end-to-end traceability.
- **1.2 Classify Disclosure Route:** Evaluates asset parameters against Article 50 rules (such as deepfake thresholds and public dissemination criteria) and routes the item down the text or media processing branch.
- **1.3 Prepare Text Disclosure:** Appends the required statutory or house-policy disclosure sentence to text assets.
- **1.4 Request Media Labelling:** Saves media assets to disk and sends them to an external HTTP media-labeling service to burn in visibility flags.
- **1.5 Handle Burn-In Result:** Manages image-specific burn-in procedures via native image editing tools or handles fallback mechanics when external media labeling is unavailable.
- **1.6 Build and Store Manifest:** Compiles asset metadata, disclosure requirements, and execution URLs into a structured manifest and records the initial state in PostgreSQL.
- **1.7 Notify Approval Reviewer:** Posts review requests to a Slack channel and pauses execution until a human reviewer interacts with the approval gate.
- **1.8 Record Review Decision:** Processes human reviewer inputs (approvals, editorial exceptions, or returns) and logs the outcome in a secure PostgreSQL ledger.
- **1.9 Load Registry Inputs:** Queries stored asset records and fetches all n8n workflows from the instance via the n8n API.
- **1.10 Snapshot AI Registry:** Performs path analysis on workflow topologies to evaluate deployment compliance and saves an instance-wide AI systems registry snapshot to PostgreSQL.
- **1.11 Return Workflow Result:** Formats the final execution payload and returns asset-level compliance summaries to the caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Accepts raw AI-generated assets and associated metadata from users or downstream systems, generating a SHA-256 cryptographic hash of the prompt for tracking.
- **Nodes Involved:** `When Asset Produced`, `Trigger by External Workflow`, `Generate Prompt Hash`
- **Node Details:**
  - **When Asset Produced**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` — Entry point for manual form submissions.
    - *Configuration Choices:* Configured with an intake path (`ai-act-intake`) and fields capturing asset types, languages, models, prompts, operators, depiction details, origin, artistic status, and optional files.
    - *Input/Output Connections:* Output connects directly to `Generate Prompt Hash`.
    - *Edge Cases / Failures:* Missing required form fields will halt execution at intake.
  - **Trigger by External Workflow**
    - *Type & Technical Role:* `n8n-nodes-base.executeWorkflowTrigger` — Entry point for programmatic calls via sub-workflow execution.
    - *Configuration Choices:* Uses passthrough input source.
    - *Input/Output Connections:* Output connects directly to `Generate Prompt Hash`.
    - *Edge Cases / Failures:* Missing payload properties will cause downstream expression evaluation errors.
  - **Generate Prompt Hash**
    - *Type & Technical Role:* `n8n-nodes-base.crypto` — Generates a cryptographic hash.
    - *Configuration Choices:* Evaluates `={{ $json.Prompt || $json.prompt || '' }}` to produce a SHA-256 hash stored under `prompt_sha256`.
    - *Input/Output Connections:* Receives input from either intake trigger; outputs to `Classify Under Article 50`.
    - *Edge Cases / Failures:* Empty prompt values default to empty strings before hashing.

#### 2.2 Classify Disclosure Route
- **Overview:** Evaluates asset characteristics against EU AI Act Article 50 mandates, determines classification categories, assigns language-specific disclosure sentences, and routes the asset.
- **Nodes Involved:** `Classify Under Article 50`, `Check If Asset Is Text`
- **Node Details:**
  - **Classify Under Article 50**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript execution block implementing compliance business logic.
    - *Configuration Choices:* Normalizes incoming fields, determines deepfake and public information flags, sets multi-language disclosure text, and establishes storage path variables.
    - *Input/Output Connections:* Receives input from `Generate Prompt Hash`; outputs to `Check If Asset Is Text`.
    - *Edge Cases / Failures:* Unsupported language codes fall back to English; missing file binaries bypass media paths cleanly.
  - **Check If Asset Is Text**
    - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router.
    - *Configuration Choices:* Evaluates whether `={{ $json.asset_type }}` equals `text`.
    - *Input/Output Connections:* Receives input from `Classify Under Article 50`; outputs true branch to `Prepare Text Disclosure` and false branch to `Save Incoming File`.
    - *Edge Cases / Failures:* Unexpected asset type strings default to the media processing path.

#### 2.3 Prepare Text Disclosure
- **Overview:** Formats text assets by appending the statutory disclosure sentence as a footer when legally required.
- **Nodes Involved:** `Prepare Text Disclosure`
- **Node Details:**
  - **Prepare Text Disclosure**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript transformation block.
    - *Configuration Choices:* References the classification data from upstream nodes and appends disclosure formatting if `label_required` is true.
    - *Input/Output Connections:* Receives input from `Check If Asset Is Text` (true branch); outputs to `Build Asset Manifest`.
    - *Edge Cases / Failures:* Missing text body strings result in disclosure footers appended to empty values.

#### 2.4 Request Media Labelling
- **Overview:** Writes incoming binary media files to the local container filesystem and submits them to an external labeling microservice.
- **Nodes Involved:** `Save Incoming File`, `Label Media via API`
- **Node Details:**
  - **Save Incoming File**
    - *Type & Technical Role:* `n8n-nodes-base.readWriteFile` — Filesystem storage utility.
    - *Configuration Choices:* Writes binary files to `={{ $json.incoming_path }}` using the specified data property name.
    - *Input/Output Connections:* Receives input from `Check If Asset Is Text` (false branch); outputs to `Label Media via API`.
    - *Edge Cases / Failures:* Filesystem permission errors or disk exhaustion will halt execution.
  - **Label Media via API**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` — External service integration.
    - *Configuration Choices:* Sends a POST request to `http://kit-media-label:8881/label` with query parameters for type, extension, size constraints, and label text, setting a 300-second timeout and returning response data as a file.
    - *Input/Output Connections:* Receives input from `Save Incoming File`; outputs successful responses to `Write Labelled File` and error branches to `Check If Asset Is Image`.
    - *Edge Cases / Failures:* Service downtime triggers the configured `continueErrorOutput` path, routing execution to image inspection checks.

#### 2.5 Handle Burn-In Result
- **Overview:** Applies image-specific text overlays using native image editing tools or assigns fallback non-burn-in metadata when external labeling fails.
- **Nodes Involved:** `Check If Asset Is Image`, `Edit and Label Image`, `Write Labelled File`, `Set Burn-In Metadata`, `Set No Burn-In Metadata`
- **Node Details:**
  - **Check If Asset Is Image**
    - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router.
    - *Configuration Choices:* Evaluates if the asset type is `image`.
    - *Input/Output Connections:* Receives error flow from `Label Media via API`; outputs true to `Edit and Label Image` and false to `Set No Burn-In Metadata`.
    - *Edge Cases / Failures:* Non-image media that failed API labeling route directly to fallback metadata assignments.
  - **Edit and Label Image**
    - *Type & Technical Role:* `n8n-nodes-base.editImage` — Native image manipulation utility.
    - *Configuration Choices:* Applies a white text label at specific coordinates with dynamic font sizing based on compact flag settings.
    - *Input/Output Connections:* Receives input from `Check If Asset Is Image`; outputs success to `Write Labelled File` and error flows to `Set No Burn-In Metadata`.
    - *Edge Cases / Failures:* Malformed image binary buffers trigger error handling paths.
  - **Write Labelled File**
    - *Type & Technical Role:* `n8n-nodes-base.readWriteFile` — Filesystem storage utility.
    - *Configuration Choices:* Writes processed media binaries to `={{ $('Classify Under Article 50').first().json.labelled_path }}`.
    - *Input/Output Connections:* Receives input from `Label Media via API` and `Edit and Label Image`; outputs to `Set Burn-In Metadata`.
    - *Edge Cases / Failures:* Filesystem write failures will cause node execution errors.
  - **Set Burn-In Metadata**
    - *Type & Technical Role:* `n8n-nodes-base.set` — Data structuring node.
    - *Configuration Choices:* Assigns `burned_in` as boolean `true`.
    - *Input/Output Connections:* Receives input from `Write Labelled File`; outputs to `Build Asset Manifest`.
    - *Edge Cases / Failures:* None anticipated.
  - **Set No Burn-In Metadata**
    - *Type & Technical Role:* `n8n-nodes-base.set` — Data structuring node.
    - *Configuration Choices:* Assigns `burned_in` as boolean `false` and sets an explanatory `label_note`.
    - *Input/Output Connections:* Receives input from `Check If Asset Is Image` and `Edit and Label Image`; outputs to `Build Asset Manifest`.
    - *Edge Cases / Failures:* None anticipated.

#### 2.6 Build and Store Manifest
- **Overview:** Consolidates processing outputs into a unified JSON manifest containing review gate resume URLs and records the asset in PostgreSQL.
- **Nodes Involved:** `Build Asset Manifest`, `Store Asset in Database`
- **Node Details:**
  - **Build Asset Manifest**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript transformation block.
    - *Configuration Choices:* Merges upstream classification data with text or media branch flags, injecting execution resume URLs (`$execution.resumeFormUrl`) and execution IDs.
    - *Input/Output Connections:* Receives inputs from text and media branches; outputs to `Store Asset in Database`.
    - *Edge Cases / Failures:* Missing upstream execution context fields will result in undefined resume URLs.
  - **Store Asset in Database**
    - *Type & Technical Role:* `n8n-nodes-base.postgres` — Database persistence node.
    - *Configuration Choices:* Executes an `UPSERT` query on the `assets` table using query replacements mapped from JSON properties. Uses credential `kit-db`.
    - *Input/Output Connections:* Receives input from `Build Asset Manifest`; outputs to `Notify Reviewer via Slack`.
    - *Edge Cases / Failures:* Database connectivity loss or schema constraint violations will halt execution.

#### 2.7 Notify Approval Reviewer
- **Overview:** Sends notifications to a designated Slack channel and pauses workflow execution to await human approval.
- **Nodes Involved:** `Notify Reviewer via Slack`, `Wait for Reviewer Decision`
- **Node Details:**
  - **Notify Reviewer via Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` — External messaging integration.
    - *Configuration Choices:* Posts formatted compliance summaries and gate links to `#ai-act-gate`. Node is disabled by default pending credential configuration.
    - *Credentials Required:* Slack OAuth2 / Bot Token.
    - *Input/Output Connections:* Receives input from `Store Asset in Database`; outputs to `Wait for Reviewer Decision`.
    - *Edge Cases / Failures:* Expired credentials or invalid channel IDs trigger continue-regular-output flows, allowing the wait node to function independently.
  - **Wait for Reviewer Decision**
    - *Type & Technical Role:* `n8n-nodes-base.wait` — Execution pause utility.
    - *Configuration Choices:* Configured to resume via form interaction (`resume: "form"`), presenting an HTML review interface with asset metadata, legal obligations, and decision dropdowns.
    - *Input/Output Connections:* Receives input from `Notify Reviewer via Slack`; outputs to `Process Reviewer Decision`.
    - *Edge Cases / Failures:* Long-running pauses depend on n8n instance persistence settings and webhook availability.

#### 2.8 Record Review Decision
- **Overview:** Processes human reviewer selections, maps decisions to disclosure statuses, and logs the outcome in a PostgreSQL ledger.
- **Nodes Involved:** `Process Reviewer Decision`, `Record Decision in Database`
- **Node Details:**
  - **Process Reviewer Decision**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript transformation block.
    - *Configuration Choices:* Evaluates decision strings (approvals, editorial exceptions, returns), assigns responsible parties, and constructs an approval audit ledger object.
    - *Input/Output Connections:* Receives input from `Wait for Reviewer Decision`; outputs to `Record Decision in Database`.
    - *Edge Cases / Failures:* Unrecognized decision strings default to returned asset statuses.
  - **Record Decision in Database**
    - *Type & Technical Role:* `n8n-nodes-base.postgres` — Database persistence node.
    - *Configuration Choices:* Executes a CTE query inserting audit records into the `decisions` table while updating asset status fields in the `assets` table. Uses credential `kit-db`.
    - *Input/Output Connections:* Receives input from `Process Reviewer Decision`; outputs to `Fetch Stored Assets`.
    - *Edge Cases / Failures:* Database constraint errors on ledger foreign keys will halt execution.

#### 2.9 Load Registry Inputs
- **Overview:** Queries stored assets and retrieves instance-wide workflow metadata via the n8n API to prepare for registry generation.
- **Nodes Involved:** `Fetch Stored Assets`, `Read All Workflows`
- **Node Details:**
  - **Fetch Stored Assets**
    - *Type & Technical Role:* `n8n-nodes-base.postgres` — Database query node.
    - *Configuration Choices:* Selects `id`, `status`, `disclosure_status`, and `manifest` from the `assets` table. Uses credential `kit-db`.
    - *Input/Output Connections:* Receives input from `Record Decision in Database`; outputs to `Read All Workflows`.
    - *Edge Cases / Failures:* Empty result sets return empty arrays, which are handled downstream.
  - **Read All Workflows**
    - *Type & Technical Role:* `n8n-nodes-base.n8n` — Internal n8n API integration.
    - *Configuration Choices:* Retrieves all workflows on the instance (`resource: "workflow"`, `operation: "getAll"`) in a single execution. Uses credential `n8n-account`.
    - *Credentials Required:* n8n API Key.
    - *Input/Output Connections:* Receives input from `Fetch Stored Assets`; outputs to `Generate Registry Rows`.
    - *Edge Cases / Failures:* Invalid API credentials or unreachable internal APIs result in partial registry coverage flags.

#### 2.10 Snapshot AI Registry
- **Overview:** Analyzes workflow connection topologies and generator pathways to assess compliance statuses and records an instance-wide registry snapshot in PostgreSQL.
- **Nodes Involved:** `Generate Registry Rows`, `Snapshot Registry State`
- **Node Details:**
  - **Generate Registry Rows**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript analysis script.
    - *Configuration Choices:* Scans workflow node connections forward from AI generators to publishing exits, categorizes compliance statuses (disclosed, uncovered, likeness, etc.), and aggregates manifest-declared models.
    - *Input/Output Connections:* Receives input from `Read All Workflows`; outputs to `Snapshot Registry State`.
    - *Edge Cases / Failures:* Malformed workflow JSON structures are caught and skipped safely.
  - **Snapshot Registry State**
    - *Type & Technical Role:* `n8n-nodes-base.postgres` — Database persistence node.
    - *Configuration Choices:* Inserts registry summary payloads and partial flags into the `registry_snapshots` table. Uses credential `kit-db`.
    - *Input/Output Connections:* Receives input from `Generate Registry Rows`; outputs to `Return Asset Data to Caller`.
    - *Edge Cases / Failures:* Large JSON payloads exceeding database column size limits will cause transaction failures.

#### 2.11 Return Workflow Result
- **Overview:** Formats final execution summaries and returns compliance metadata to the caller.
- **Nodes Involved:** `Return Asset Data to Caller`
- **Node Details:**
  - **Return Asset Data to Caller**
    - *Type & Technical Role:* `n8n-nodes-base.code` — Custom JavaScript formatting block.
    - *Configuration Choices:* Extracts approval outcomes, disclosure statuses, responsible parties, and manifest paths into a structured return object.
    - *Input/Output Connections:* Receives input from `Snapshot Registry State`; final workflow termination node.
    - *Edge Cases / Failures:* Missing upstream review decisions will result in undefined property values.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Asset Produced | n8n-nodes-base.formTrigger | Entry point for manual form submissions | None | Generate Prompt Hash | Label AI-generated content under the EU AI Act with a Slack approval gate and Postgres ledger. Full source, the database schema with its append-only triggers, a docker compose install and a written case study: https://github.com/karusrus/transparency-kit — https://karusrus.github.io/transparency-kit/ |
| Trigger by External Workflow | n8n-nodes-base.executeWorkflowTrigger | Entry point for programmatic sub-workflow calls | None | Generate Prompt Hash | Label AI-generated content under the EU AI Act with a Slack approval gate and Postgres ledger. Full source, the database schema with its append-only triggers, a docker compose install and a written case study: https://github.com/karusrus/transparency-kit — https://karusrus.github.io/transparency-kit/ |
| Generate Prompt Hash | n8n-nodes-base.crypto | Generates SHA-256 prompt hash for traceability | When Asset Produced, Trigger by External Workflow | Classify Under Article 50 | Receive asset input: Accepts a newly produced asset either directly through a form or from another workflow, then creates a prompt hash for traceability. |
| Classify Under Article 50 | n8n-nodes-base.code | Classifies asset against Article 50 rules and assigns disclosure text | Generate Prompt Hash | Check If Asset Is Text | Classify disclosure route: Classifies the asset under the Article 50 disclosure logic and routes it into the text or media processing branch. |
| Check If Asset Is Text | n8n-nodes-base.if | Routes assets based on text versus media types | Classify Under Article 50 | Prepare Text Disclosure, Save Incoming File | Classify disclosure route: Classifies the asset under the Article 50 disclosure logic and routes it into the text or media processing branch. |
| Prepare Text Disclosure | n8n-nodes-base.code | Appends statutory disclosure footer to text assets | Check If Asset Is Text | Build Asset Manifest | Prepare text disclosure: Builds the disclosed version for text assets before the common manifest step. |
| Save Incoming File | n8n-nodes-base.readWriteFile | Writes incoming media files to container disk | Check If Asset Is Text | Label Media via API | Request media labelling: Stores the incoming media file, sends it to the external labeling service, and checks whether the result should follow the image burn-in path. |
| Label Media via API | n8n-nodes-base.httpRequest | Sends media to external labeling service via HTTP | Save Incoming File | Write Labelled File, Check If Asset Is Image | Request media labelling: Stores the incoming media file, sends it to the external labeling service, and checks whether the result should follow the image burn-in path. |
| Check If Asset Is Image | n8n-nodes-base.if | Routes failed API media checks for image handling | Label Media via API | Edit and Label Image, Set No Burn-In Metadata | Request media labelling: Stores the incoming media file, sends it to the external labeling service, and checks whether the result should follow the image burn-in path. |
| Edit and Label Image | n8n-nodes-base.editImage | Applies native text burn-in overlays to images | Check If Asset Is Image | Write Labelled File, Set No Burn-In Metadata | Handle burn-in result: Applies image burn-in when appropriate, writes labelled output, and marks whether the final media has a burned-in disclosure or only a note. |
| Write Labelled File | n8n-nodes-base.readWriteFile | Writes processed media files to output disk path | Label Media via API, Edit and Label Image | Set Burn-In Metadata | Handle burn-in result: Applies image burn-in when appropriate, writes labelled output, and marks whether the final media has a burned-in disclosure or only a note. |
| Set Burn-In Metadata | n8n-nodes-base.set | Sets burned-in flag to true | Write Labelled File | Build Asset Manifest | Handle burn-in result: Applies image burn-in when appropriate, writes labelled output, and marks whether the final media has a burned-in disclosure or only a note. |
| Set No Burn-In Metadata | n8n-nodes-base.set | Sets burned-in flag to false and configures notes | Check If Asset Is Image, Edit and Label Image | Build Asset Manifest | Handle burn-in result: Applies image burn-in when appropriate, writes labelled output, and marks whether the final media has a burned-in disclosure or only a note. |
| Build Asset Manifest | n8n-nodes-base.code | Consolidates asset metadata and review URLs into a manifest | Prepare Text Disclosure, Set Burn-In Metadata, Set No Burn-In Metadata | Store Asset in Database | Build and store manifest: Combines text or media branch outputs into a manifest with the review gate link, then inserts the asset record into Postgres. |
| Store Asset in Database | n8n-nodes-base.postgres | Inserts or updates asset records in PostgreSQL | Build Asset Manifest | Notify Reviewer via Slack | Build and store manifest: Combines text or media branch outputs into a manifest with the review gate link, then inserts the asset record into Postgres. |
| Notify Reviewer via Slack | n8n-nodes-base.slack | Posts approval requests to Slack channels | Store Asset in Database | Wait for Reviewer Decision | Notify approval reviewer: Sends the Slack review request and pauses execution until the reviewer responds through the approval gate. |
| Wait for Reviewer Decision | n8n-nodes-base.wait | Pauses workflow execution awaiting human form review | Notify Reviewer via Slack | Process Reviewer Decision | Notify approval reviewer: Sends the Slack review request and pauses execution until the reviewer responds through the approval gate. |
| Process Reviewer Decision | n8n-nodes-base.code | Processes human decision inputs and constructs ledger payloads | Wait for Reviewer Decision | Record Decision in Database | Record review decision: Converts the reviewer response into a disclosure status and records the approval decision in the Postgres ledger. |
| Record Decision in Database | n8n-nodes-base.postgres | Logs review decisions in audit ledger and updates asset status | Process Reviewer Decision | Fetch Stored Assets | Record review decision: Converts the reviewer response into a disclosure status and records the approval decision in the Postgres ledger. |
| Fetch Stored Assets | n8n-nodes-base.postgres | Queries stored assets from PostgreSQL | Record Decision in Database | Read All Workflows | Load registry inputs: Reads stored asset data and retrieves all n8n workflows to prepare an instance-wide AI-systems registry refresh. |
| Read All Workflows | n8n-nodes-base.n8n | Retrieves instance workflow metadata via n8n API | Fetch Stored Assets | Generate Registry Rows | Load registry inputs: Reads stored asset data and retrieves all n8n workflows to prepare an instance-wide AI-systems registry refresh. |
| Generate Registry Rows | n8n-nodes-base.code | Analyzes workflow paths and generates registry compliance data | Read All Workflows | Snapshot Registry State | Snapshot AI registry: Transforms workflow metadata into registry rows with generator path analysis and saves the registry snapshot to Postgres. |
| Snapshot Registry State | n8n-nodes-base.postgres | Saves instance-wide AI registry snapshots to PostgreSQL | Generate Registry Rows | Return Asset Data to Caller | Snapshot AI registry: Transforms workflow metadata into registry rows with generator path analysis and saves the registry snapshot to Postgres. |
| Return Asset Data to Caller | n8n-nodes-base.code | Formats final execution response payload | Snapshot Registry State | None | Return workflow result: Formats the final asset-level response returned to the calling workflow or final execution output. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in an n8n instance:

1. **Create Form Trigger Node:**
   - Add a **Form Trigger** node named `When Asset Produced`.
   - Set options path to `ai-act-intake`. Configure form fields for dropdowns (`Asset type`, `Language of the asset`, `What does it show?`, `Generated or manipulated?`, `Artistic, creative or satirical work?`, `Published to inform the public on a matter of public interest?`), text inputs (`Model`, `Model version`, `Operator`, `Consent reference`), textareas (`Prompt`, `Text content`), and file upload (`File`).
2. **Create External Workflow Trigger Node:**
   - Add an **Execute Workflow Trigger** node named `Trigger by External Workflow`. Set input source to `passthrough`.
3. **Create Crypto Hash Node:**
   - Add a **Crypto** node named `Generate Prompt Hash`.
   - Set action to `hash`, type to `SHA256`, value expression to `={{ $json.Prompt || $json.prompt || '' }}`, and data property name to `prompt_sha256`.
   - Connect both intake triggers to this node.
4. **Create Classification Code Node:**
   - Add a **Code** node named `Classify Under Article 50`.
   - Paste the Article 50 JavaScript classification logic establishing disclosure sentences and path variables.
   - Connect `Generate Prompt Hash` to this node.
5. **Create Text Routing IF Node:**
   - Add an **If** node named `Check If Asset Is Text`.
   - Set condition left value to `={{ $json.asset_type }}` and operation to `equals` `text`.
   - Connect `Classify Under Article 50` to this node.
6. **Create Text Disclosure Node:**
   - Add a **Code** node named `Prepare Text Disclosure`.
   - Implement the footer appending logic referencing upstream classification outputs.
   - Connect the true branch of `Check If Asset Is Text` to this node.
7. **Create Media Storage Node:**
   - Add a **Read/Write Files from Disk** node named `Save Incoming File`.
   - Set operation to `write`, file name to `={{ $json.incoming_path }}`, and data property name to `={{ $json.binary_key || 'File' }}`.
   - Connect the false branch of `Check If Asset Is Text` to this node.
8. **Create HTTP Media Labelling Node:**
   - Add an **HTTP Request** node named `Label Media via API`.
   - Set method to `POST`, URL to `={{ 'http://kit-media-label:8881/label?type=' + $('Classify Under Article 50').first().json.asset_type + '&ext=' + encodeURIComponent($('Classify Under Article 50').first().json.ext) + '&small=' + ($('Classify Under Article 50').first().json.label_small ? 1 : 0) + '&text=' + encodeURIComponent($('Classify Under Article 50').first().json.label_text) + '&comment=' + encodeURIComponent($('Classify Under Article 50').first().json.label_text + '; manifest ' + $('Classify Under Article 50').first().json.id) }}`, body content type to `binaryData`, input data field name to `={{ $json.binary_key || 'File' }}`, and configure timeout to 300,000ms. Set response format to file with output property `={{ $json.binary_key || 'File' }}`. Enable `Continue On Fail` (`continueErrorOutput`).
   - Connect `Save Incoming File` to this node.
9. **Create Image Inspection IF Node:**
   - Add an **If** node named `Check If Asset Is Image`.
   - Set condition left value to `={{ $('Classify Under Article 50').first().json.asset_type }}` equals `image`.
   - Connect the error output of `Label Media via API` to this node.
10. **Create Image Editing Node:**
    - Add an **Edit Image** node named `Edit and Label Image`.
    - Set operation to `text`, text to `={{ $('Classify Under Article 50').first().json.label_text }}`, font size to `={{ $('Classify Under Article 50').first().json.label_small ? 24 : 44 }}`, font color `#ffffff`, position X `24`, position Y `60`, line length `300`, and data property name to `={{ $('Classify Under Article 50').first().json.binary_key || 'File' }}`. Enable `Continue On Fail`.
    - Connect the true branch of `Check If Asset Is Image` to this node.
11. **Create Write Labelled File Node:**
    - Add a **Read/Write Files from Disk** node named `Write Labelled File`.
    - Set operation to `write`, file name to `={{ $('Classify Under Article 50').first().json.labelled_path }}`, and data property name to `={{ $json.binary_key || 'File' }}`.
    - Connect the success output of `Label Media via API` and the success output of `Edit and Label Image` to this node.
12. **Create Burn-In Metadata Nodes:**
    - Add a **Set** node named `Set Burn-In Metadata` assigning `burned_in` as boolean `true`. Connect `Write Labelled File` to this node.
    - Add a **Set** node named `Set No Burn-In Metadata` assigning `burned_in` as boolean `false` and `label_note`. Connect the false branch of `Check If Asset Is Image` and the error output of `Edit and Label Image` to this node.
13. **Create Manifest Construction Node:**
    - Add a **Code** node named `Build Asset Manifest`.
    - Paste the manifest merging logic that injects resume URLs (`$execution.resumeFormUrl`) and execution IDs.
    - Connect `Prepare Text Disclosure`, `Set Burn-In Metadata`, and `Set No Burn-In Metadata` to this node.
14. **Create Database Storage Node:**
    - Add a **Postgres** node named `Store Asset in Database`.
    - Configure operation to `Execute Query`, query to `insert into assets (...) values (...) on conflict (id) do update set ...`, and configure query replacement expressions mapping manifest fields. Configure credential `kit-db`.
    - Connect `Build Asset Manifest` to this node.
15. **Create Slack Notification Node:**
    - Add a **Slack** node named `Notify Reviewer via Slack`.
    - Set resource to `message`, operation to `post`, channel to `#ai-act-gate`, and message text expression incorporating asset manifest details and gate URLs. Configure Slack credentials. Disable the node by default if credentials are absent.
    - Connect `Store Asset in Database` to this node.
16. **Create Wait Node:**
    - Add a **Wait** node named `Wait for Reviewer Decision`.
    - Set resume to `form`, form title to `Human approval gate`, and configure the HTML field rendering asset metadata, obligations, and review dropdown fields.
    - Connect `Notify Reviewer via Slack` to this node.
17. **Create Review Processing Node:**
    - Add a **Code** node named `Process Reviewer Decision`.
    - Paste the JavaScript logic transforming reviewer selections into disclosure statuses and ledger audit payloads.
    - Connect `Wait for Reviewer Decision` to this node.
18. **Create Ledger Recording Node:**
    - Add a **Postgres** node named `Record Decision in Database`.
    - Configure operation to `Execute Query`, SQL query performing CTE inserts into `decisions` and updates on `assets`, using query replacements mapped from json properties. Configure credential `kit-db`.
    - Connect `Process Reviewer Decision` to this node.
19. **Create Asset Fetching Node:**
    - Add a **Postgres** node named `Fetch Stored Assets`.
    - Configure operation to `Execute Query`, query to `select id, status, disclosure_status, manifest from assets order by created_at`. Configure credential `kit-db`.
    - Connect `Record Decision in Database` to this node.
20. **Create n8n Workflow Reading Node:**
    - Add an **n8n** node named `Read All Workflows`.
    - Set resource to `workflow`, operation to `getAll`, return all to `true`, and execute once to `true`. Configure n8n API credentials (`n8n-account`). Enable `Continue On Fail`.
    - Connect `Fetch Stored Assets` to this node.
21. **Create Registry Analysis Code Node:**
    - Add a **Code** node named `Generate Registry Rows`.
    - Paste the registry analysis script performing graph topology evaluation and compliance classification.
    - Connect `Read All Workflows` to this node.
22. **Create Registry Snapshot Persistence Node:**
    - Add a **Postgres** node named `Snapshot Registry State`.
    - Configure operation to `Execute Query`, query to `insert into registry_snapshots (partial, systems, registry) values ($1, $2, $3::jsonb)` with query replacements. Configure credential `kit-db`.
    - Connect `Generate Registry Rows` to this node.
23. **Create Return Result Node:**
    - Add a **Code** node named `Return Asset Data to Caller`.
    - Paste the final response formatting code returning approval status and manifest pointers.
    - Connect `Snapshot Registry State` to this node as the final workflow termination point.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full source code, database schema with append-only triggers, Docker Compose setup, and written case study | https://github.com/karusrus/transparency-kit |
| Author and project context documentation | https://karusrus.github.io/transparency-kit/ |
| Project Credits | Built by Ruslan Karymov, AI Enablement & Automation Lead, Creative, Marketing and Business Operations |