Route AV rental kit readiness with Airtable, OpenAI, Google Sheets, and Trello

https://n8nworkflows.xyz/workflows/route-av-rental-kit-readiness-with-airtable--openai--google-sheets--and-trello-19758


# Route AV rental kit readiness with Airtable, OpenAI, Google Sheets, and Trello

### 1. Workflow Overview

The **Route AV Rental Kit Readiness** workflow automates the operational assessment and routing of live-event audio-visual (AV) equipment reservations. Its primary purpose is to cross-reference incoming reservation changes against real-time warehouse inventory, evaluate equipment shortages, approve validated substitute items within specific compatibility groups, use AI to parse technician notes for risks, and direct the operational request to the appropriate channel (Airtable, Google Sheets, Gmail, or Trello).

The execution logic is grouped into the following functional blocks:

- **1.1 Input Reception & Normalization:** Captures the authenticated reservation webhook, parses the unique event parameters, injects configuration constants, and introduces a stabilization delay.
- **1.2 Data Retrieval & Stock Allocation:** Fetches reservation details and warehouse inventory from Airtable, then executes core allocation mathematics to calculate exact SKU shortages, allowable substitutions, and dispatch timelines.
- **1.3 Data Integrity & AI Classification:** Validates data structures against corruption or pagination errors, utilizes an OpenAI language model to screen technician notes for unresolved safety or compliance constraints, and checks routing integrity.
- **1.4 Policy Routing & Execution:** Directs valid requests down specific conditional pathways based on inventory status and AI evaluation—updating Airtable for ready kits, logging substitutions in Google Sheets, drafting cross-hire briefs in Gmail, or creating blocked task cards in Trello.
- **1.5 Artifact Verification:** Confirms that the target integration successfully acknowledged and processed the final assessment output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** This block receives the external HTTP POST request triggered by a reservation change, maps the primary identifiers, sets up operational environment variables, and pauses briefly to let concurrent stock updates settle.
- **Nodes Involved:**
  - `Receive Reservation Change`
  - `Normalise Kit Event`
  - `Settle Stock Reservation Updates`

- **Node Details:**
  - **Receive Reservation Change**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger node). Listens for incoming webhook payloads via HTTP POST.
    - *Configuration:* Path set to `sal-av-kit-readiness-v1`, authentication set to Header Auth.
    - *Input/Output:* No input connections; outputs to `Normalise Kit Event`.
    - *Edge Cases:* Unauthenticated requests or invalid webhook paths will return a 401/404 error from the n8n webhook runner.
  - **Normalise Kit Event**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Transform node). Normalizes incoming payloads and establishes baseline configuration constants.
    - *Configuration:* Assigns `recordId` and `eventId` from the webhook body. Configures environment placeholders (`baseId`, table names, `sheetId`, `trelloList`, `recipient`, and `crossHireLeadHours` set to 12).
    - *Input/Output:* Input from `Receive Reservation Change`; outputs to `Settle Stock Reservation Updates`.
    - *Edge Cases:* Missing keys in the incoming JSON fallback to empty strings using nullish coalescing.
  - **Settle Stock Reservation Updates**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Utility node). Pauses execution to prevent race conditions with upstream inventory changes.
    - *Configuration:* Wait duration set to 15 seconds.
    - *Input/Output:* Input from `Normalise Kit Event`; outputs to `Fetch Rental Reservation`.
    - *Edge Cases:* Timeouts or long-running executions if system load is extremely high.

---

#### 2.2 Data Retrieval & Stock Allocation
- **Overview:** This block queries Airtable to fetch reservation details and warehouse kit stock, then runs deterministic allocation logic to determine inventory availability, shortages, and substitution feasibility.
- **Nodes Involved:**
  - `Fetch Rental Reservation`
  - `Fetch Kit Stock`
  - `Shape Kit Readiness`

- **Node Details:**
  - **Fetch Rental Reservation**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Fetches a single reservation record from Airtable.
    - *Configuration:* Uses dynamic URL construction via Airtable REST API. Configured with Generic HTTP Header Authentication (PAT). Includes `continueRegularOutput` error handling with 3 retries and a 2,000ms delay.
    - *Input/Output:* Input from `Settle Stock Reservation Updates`; outputs to `Fetch Kit Stock`.
    - *Edge Cases:* HTTP 404 if the record ID does not exist; API rate-limiting (handled via built-in retries).
  - **Fetch Kit Stock**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Retrieves warehouse inventory and free quantities from Airtable.
    - *Configuration:* Requests the `KitStock` table with a maximum page size of 100 (`?pageSize=100`). Uses HTTP Header Auth. Configured with up to 3 retries.
    - *Input/Output:* Input from `Fetch Rental Reservation`; outputs to `Shape Kit Readiness`.
    - *Edge Cases:* Unhandled pagination offsets (`stock.offset`) are caught downstream in the validation logic.
  - **Shape Kit Readiness**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transform node). Allocates exact SKUs, processes approved substitutes within matching compatibility groups, calculates shortages, and establishes the operational route.
    - *Configuration:* Custom JavaScript parsing `RequirementsJson` (max 50 lines) against stock quantities. Computes time-to-dispatch against `crossHireLeadHours` to determine routes (`ready`, `substitute`, `crosshire`, or `blocked`).
    - *Input/Output:* Input from `Fetch Kit Stock`; outputs to `Check Kit Input Integrity`.
    - *Edge Cases:* Malformed JSON in `RequirementsJson` or missing `DispatchAt` parameters are caught and pushed to an `errors` array.

---

#### 2.3 Data Integrity & AI Classification
- **Overview:** This block validates input integrity to block corrupted requests, initializes the OpenAI language model, and classifies unstructured technician notes to detect hidden safety or compliance concerns.
- **Nodes Involved:**
  - `Check Kit Input Integrity`
  - `Stop Invalid Kit Input`
  - `Supply Kit Classification Model`
  - `Classify Technician Constraints`

- **Node Details:**
  - **Check Kit Input Integrity**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Branching node). Verifies whether the upstream shaping step found validation errors.
    - *Configuration:* Evaluates `{{ $('Shape Kit Readiness').item.json.valid }}` equals `true`.
    - *Input/Output:* Input from `Shape Kit Readiness`; outputs `true` branch to `Classify Technician Constraints` and `false` branch to `Stop Invalid Kit Input`.
    - *Edge Cases:* Halts processing immediately if source records are incomplete or paginated.
  - **Stop Invalid Kit Input**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Error handling node). Terminates the workflow execution with a descriptive failure message if data is invalid.
    - *Configuration:* Throws an explicit error containing validation failure arrays.
    - *Input/Output:* Input from `Check Kit Input Integrity` (false branch); terminal node.
  - **Supply Kit Classification Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Sub-node / AI Model provider). Configures the LLM backend for text classification.
    - *Configuration:* Uses model `gpt-4.1-mini` with temperature set to `0` for deterministic outputs. Includes retry logic.
    - *Input/Output:* Connected via AI language model connection to `Classify Technician Constraints`.
  - **Classify Technician Constraints**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.textClassifier` (AI processing node). Classifies technician notes into operational categories.
    - *Configuration:* System prompt instructs the model to treat text as untrusted evidence and classify notes into `clear` (no issues) or `review` (uncertainty, safety conditions, or missing client sign-off).
    - *Input/Output:* Input from `Check Kit Input Integrity`; outputs evaluated classification results to `Route Kit Policy` and blocked task handling.
    - *Edge Cases:* API timeouts or malformed model outputs default to fallback categories.

---

#### 2.4 Policy Routing & Execution
- **Overview:** Evaluates the combined outcomes of inventory math and AI classification to route the reservation assessment to the correct destination system (Airtable, Google Sheets, Gmail, or Trello).
- **Nodes Involved:**
  - `Route Kit Policy`
  - `File Readiness Assessment`
  - `Queue Substitution Review`
  - `Draft Cross Hire Brief`
  - `Create Blocked Kit Task`

- **Node Details:**
  - **Route Kit Policy**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Branching node). Directs flow based on inventory route (`ready`, `substitute`, `crosshire`) combined with classification state.
    - *Configuration:* Evaluates assessment status and routes to corresponding output connectors. Includes a fallback output.
    - *Input/Output:* Input from `Classify Technician Constraints`; outputs to respective integration nodes.
  - **File Readiness Assessment**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Updates the original Airtable reservation record with readiness assessment fields.
    - *Configuration:* Executes an HTTP PATCH request to the Airtable API, sending `ReadinessAssessment`, `AssessmentJson`, and `AssessmentEventId`.
    - *Input/Output:* Input from `Route Kit Policy` (ready branch); outputs to `Verify Kit Artifact`.
  - **Queue Substitution Review**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Integration node). Appends a new review row to a specified Google Sheet.
    - *Configuration:* Operates on sheet name `Substitutions`, mapping columns: `eventId`, `recordId`, `status`, and `assessmentJson`.
    - *Input/Output:* Input from `Route Kit Policy` (substitute branch); outputs to `Verify Kit Artifact`.
  - **Draft Cross Hire Brief**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Integration node). Creates an internal draft email for cross-hire procurement.
    - *Configuration:* Sets resource to `draft`, populates recipient, subject, and message body from shape data.
    - *Input/Output:* Input from `Route Kit Policy` (crosshire branch); outputs to `Verify Kit Artifact`.
  - **Create Blocked Kit Task**
    - *Type and Technical Role:* `n8n-nodes-base.trello` (Integration node). Creates a task card in a pre-configured Trello list for blocked or review-required kits.
    - *Configuration:* Sets card name to assessment subject, maps description to report text and technician notes, using target `trelloList` ID.
    - *Input/Output:* Input from policy routing or AI review branches; outputs to `Verify Kit Artifact`.

---

#### 2.5 Artifact Verification
- **Overview:** Validates that the downstream integration successfully created or updated the target artifact before concluding the execution.
- **Nodes Involved:**
  - `Verify Kit Artifact`

- **Node Details:**
  - **Verify Kit Artifact**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript validation node). Confirms the response returned by the destination system matches expected schema attributes.
    - *Configuration:* Validates response payload IDs, event IDs, and status codes. Throws an error if confirmation fails.
    - *Input/Output:* Inputs from all integration destination nodes; outputs workflow completion status `{ status: 'assessment_filed' }`.
    - *Edge Cases:* Network failures or unacknowledged API responses trigger execution failures for traceability.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Reservation Change | n8n-nodes-base.webhook | Receive an authenticated Airtable reservation-change event. | None | Normalise Kit Event | Receive an authenticated Airtable reservation-change event.<br>POST body: recordId and eventId; select Header Auth credential. |
| Normalise Kit Event | n8n-nodes-base.set | Normalise input and centralise buyer configuration. | Receive Reservation Change | Settle Stock Reservation Updates | Normalise input and centralise buyer configuration.<br>Replace REPLACE values; inputs use explicit defaults. |
| Settle Stock Reservation Updates | n8n-nodes-base.wait | Allow upstream records to settle before rereading. | Normalise Kit Event | Fetch Rental Reservation | Allow upstream records to settle before rereading.<br>Wait 15 seconds; no backward connection. |
| Fetch Rental Reservation | n8n-nodes-base.httpRequest | Retrieve the current kit lines and dispatch deadline. | Settle Stock Reservation Updates | Fetch Kit Stock | Retrieve the current kit lines and dispatch deadline.<br>RequirementsJson and DispatchAt must be current at source. |
| Fetch Kit Stock | n8n-nodes-base.httpRequest | Read free quantities and approved compatibility groups. | Fetch Rental Reservation | Shape Kit Readiness | Read free quantities and approved compatibility groups.<br>FreeQty must already exclude reservations for other jobs. |
| Shape Kit Readiness | n8n-nodes-base.code | Allocate exact SKUs first, deplete substitutes and compute one policy route. | Fetch Kit Stock | Check Kit Input Integrity | Allocate exact SKUs first, deplete substitutes and compute one policy route.<br>Only approvedSubstitutes with matching CompatibilityGroup qualify. |
| Check Kit Input Integrity | n8n-nodes-base.if | Reject malformed stock, duplicated SKUs or missing dispatch times. | Shape Kit Readiness | Classify Technician Constraints, Stop Invalid Kit Input | Reject malformed stock, duplicated SKUs or missing dispatch times.<br>False output stops before drafting any action. |
| Stop Invalid Kit Input | n8n-nodes-base.code | Expose source data problems as an execution failure. | Check Kit Input Integrity | None | Expose source data problems as an execution failure.<br>No automatic inventory reservation or dispatch takes place. |
| Supply Kit Classification Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide a low-temperature OpenAI model to the attached AI node. | None (AI Model connection) | Classify Technician Constraints | Provide a low-temperature OpenAI model to the attached AI node.<br>Choose an available model; temperature 0. |
| Classify Technician Constraints | @n8n/n8n-nodes-langchain.textClassifier | Route unresolved technician constraints to a manual task. | Check Kit Input Integrity | Route Kit Policy, Create Blocked Kit Task | Route unresolved technician constraints to a manual task.<br>Clear reaches deterministic policy routing; review/other goes to Trello. |
| Route Kit Policy | n8n-nodes-base.switch | Choose readiness filing, substitution review, cross-hire draft or a blocked task. | Classify Technician Constraints | File Readiness Assessment, Queue Substitution Review, Draft Cross Hire Brief, Create Blocked Kit Task | Choose readiness filing, substitution review, cross-hire draft or a blocked task.<br>Routes derive from stock maths; AI cannot alter quantities. |
| File Readiness Assessment | n8n-nodes-base.httpRequest | Update the existing reservation with a readiness assessment. | Route Kit Policy | Verify Kit Artifact | Update the existing reservation with a readiness assessment.<br>Does not reserve inventory or authorise dispatch. |
| Queue Substitution Review | n8n-nodes-base.googleSheets | Append the proposed kit substitutions for an operator to review. | Route Kit Policy | Verify Kit Artifact | Append the proposed kit substitutions for an operator to review.<br>Columns: eventId, recordId, status, assessmentJson. |
| Draft Cross Hire Brief | n8n-nodes-base.gmail | Save an internal review email as a Gmail draft. | Route Kit Policy | Verify Kit Artifact | Save an internal review email as a Gmail draft.<br>Draft only; recipient, subject and body come from Shape. |
| Create Blocked Kit Task | n8n-nodes-base.trello | Create an internal task for urgent shortages or uncertain technician notes. | Route Kit Policy, Classify Technician Constraints | Verify Kit Artifact | Create an internal task for urgent shortages or uncertain technician notes.<br>Configured Trello list; human resolves readiness. |
| Verify Kit Artifact | n8n-nodes-base.code | Check the selected destination acknowledged the assessment. | File Readiness Assessment, Queue Substitution Review, Draft Cross Hire Brief, Create Blocked Kit Task | None | Check the selected destination acknowledged the assessment.<br>Require a saved record/card/draft ID or one appended row. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the Webhook Trigger
1. Add a **Webhook** node named `Receive Reservation Change`.
2. Set **HTTP Method** to `POST`, **Path** to `sal-av-kit-readiness-v1`, and **Authentication** to `Header Auth`.

#### Step 2: Configure Event Normalization
1. Add a **Set** (Edit Fields) node named `Normalise Kit Event`.
2. Create string assignments for `recordId` (`={{ String($json.body?.recordId ?? '') }}`), `eventId` (`={{ String($json.body?.eventId ?? '') }}`), `baseId`, `reservationsTable` (`Reservations`), `stockTable` (`KitStock`), `sheetId`, `trelloList`, `recipient`, and a number assignment for `crossHireLeadHours` (`12`).
3. Connect `Receive Reservation Change` to `Normalise Kit Event`.

#### Step 3: Add Settlement Wait Period
1. Add a **Wait** node named `Settle Stock Reservation Updates`.
2. Set the amount to `15` seconds.
3. Connect `Normalise Kit Event` to `Settle Stock Reservation Updates`.

#### Step 4: Fetch Reservation and Inventory Data from Airtable
1. Add an **HTTP Request** node named `Fetch Rental Reservation`. Configure authentication using generic HTTP Header Auth (Airtable PAT). Set URL to construct dynamically using expressions from the Normalise node. Enable `continueRegularOutput`, set retries to `3`, and wait between tries to `2000` ms.
2. Connect `Settle Stock Reservation Updates` to `Fetch Rental Reservation`.
3. Add a second **HTTP Request** node named `Fetch Kit Stock` with identical Airtable authentication and retry settings. Set the URL endpoint to query `KitStock?pageSize=100`.
4. Connect `Fetch Rental Reservation` to `Fetch Kit Stock`.

#### Step 5: Implement Core Allocation Logic
1. Add a **Code** node named `Shape Kit Readiness`.
2. Paste JavaScript code that parses `RequirementsJson`, verifies stock records, computes exact SKU matches and approved compatibility-group substitutes, determines shortages, and calculates hours to dispatch against `crossHireLeadHours`.
3. Connect `Fetch Kit Stock` to `Shape Kit Readiness`.

#### Step 6: Validate Input Integrity
1. Add an **If** node named `Check Kit Input Integrity`. Configure condition to check if `{{ $('Shape Kit Readiness').item.json.valid }}` equals `true`.
2. Add a **Code** node named `Stop Invalid Kit Input` that throws an Error with validation failure messages. Connect the `false` output of `Check Kit Input Integrity` here.
3. Connect `Shape Kit Readiness` to `Check Kit Input Integrity`.

#### Step 7: Configure AI Text Classification
1. Add an **Advanced AI -> Chat Model** node named `Supply Kit Classification Model`. Configure with OpenAI credential, model `gpt-4.1-mini`, and temperature `0`.
2. Add an **Advanced AI -> Text Classifier** node named `Classify Technician Constraints`. Set input text to parse technician notes from the shaping node. Configure categories (`clear` and `review`) with descriptive fallback behavior.
3. Connect the AI model node to the Text Classifier via an AI language model connection.
4. Connect the `true` output of `Check Kit Input Integrity` to `Classify Technician Constraints`.

#### Step 8: Set Up Policy Routing & Destination Nodes
1. Add a **Switch** node named `Route Kit Policy`. Set up output rules for `ready`, `substitute`, and `crosshire`. Connect `Classify Technician Constraints` to this switch.
2. Add an **HTTP Request** node named `File Readiness Assessment` (Airtable PATCH request). Connect the `ready` output of `Route Kit Policy` here.
3. Add a **Google Sheets** node named `Queue Substitution Review`. Select the `Substitutions` sheet and map columns (`eventId`, `recordId`, `status`, `assessmentJson`). Connect the `substitute` output of `Route Kit Policy` here.
4. Add a **Gmail** node named `Draft Cross Hire Brief`. Set resource to `draft` and map recipient, subject, and body. Connect the `crosshire` output of `Route Kit Policy` here.
5. Add a **Trello** node named `Create Blocked Kit Task`. Configure card creation using target `trelloList` ID and descriptive text fields. Connect the `crosshire`/`review` fallback paths and classification review outputs to this node.

#### Step 9: Verify Final Artifacts
1. Add a **Code** node named `Verify Kit Artifact`. Implement verification checks to ensure downstream integrations acknowledged the task, row, or draft.
2. Connect all destination nodes (`File Readiness Assessment`, `Queue Substitution Review`, `Draft Cross Hire Brief`, `Create Blocked Kit Task`) to `Verify Kit Artifact`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator & Contact | Swapnil AI Labs — `swapnil.mandloi7@gmail.com` — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Workflow Branding & Purpose | SAL \| AV Rental Kit Readiness Router \| Shared stock depletion and substitute allocation for live-event AV rental. |