Qualify event venue leads from Gmail with Gemini, HighLevel, Sheets and Slack

https://n8nworkflows.xyz/workflows/qualify-event-venue-leads-from-gmail-with-gemini--highlevel--sheets-and-slack-19724


# Qualify event venue leads from Gmail with Gemini, HighLevel, Sheets and Slack

### 1. Workflow Overview

This workflow automates the qualification, synchronization, and notification pipeline for event venue inquiries received via email. Its primary purpose is to filter inbound venue leads using Google Gemini AI, sync qualified contacts to GoHighLevel (GHL), track interaction states via Google Sheets, and prevent duplicate notifications on Slack. 

The execution logic is structured into four main operational blocks:
- **1.1 Input Reception & Preparation:** Monitors the Gmail inbox for unread messages, structures raw email payloads, and passes them to the AI processing engine.
- **1.2 AI Processing & Qualification:** Utilizes Google Gemini to extract lead details, calculate an Ideal Customer Profile (ICP) score, and filter leads based on qualification thresholds.
- **1.3 CRM & Spreadsheet Synchronization:** Checks GoHighLevel for existing contacts, performs upserts if necessary, queries Google Sheets for historical tracking, and manages record insertion pauses using a wait step.
- **1.4 Notification & State Management:** Validates notification flags to prevent duplicate alerts, dispatches formatted messages to Slack, and updates Google Sheets records to reflect current notification states.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Preparation
- **Overview:** Triggers on new incoming emails in a monitored Gmail inbox and formats the data for downstream AI consumption.
- **Nodes Involved:** `When Email Received`, `Set Enquiry Details`

##### Node Details:
- **When Email Received**
  - *Type & Role:* `n8n-nodes-base.gmailTrigger` — Triggers execution when an unread email arrives.
  - *Configuration:* Configured to monitor unread incoming messages using OAuth2 authentication.
  - *Input/Output:* Inputs: None (Trigger); Outputs: `Set Enquiry Details`.
  - *Edge Cases:* Authentication token expiration, rate limiting on Gmail API polling (runs every minute).

- **Set Enquiry Details**
  - *Type & Role:* `n8n-nodes-base.set` — Transforms and maps incoming email fields into standard variables.
  - *Configuration:* Extracts sender address, subject, and body text.
  - *Input/Output:* Inputs: `When Email Received`; Outputs: `Process with LLM Chain`.
  - *Edge Cases:* Missing email body parameters or empty subject lines.

---

#### Block 1.2: AI Processing & Qualification
- **Overview:** Feeds email contents into Google Gemini to extract structured event data and evaluate ICP scoring before filtering out unqualified leads.
- **Nodes Involved:** `Process with LLM Chain`, `Google Gemini Model`, `Set Output Field`, `If ICP Match`

##### Node Details:
- **Google Gemini Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Language model provider node.
  - *Configuration:* Configured with Google PaLM/Gemini API credentials and model version selection.
  - *Input/Output:* Inputs: None; Outputs: Connected to `Process with LLM Chain` via `ai_languageModel`.
  - *Edge Cases:* API rate limits, model quota exhaustion, or invalid API keys.

- **Process with LLM Chain**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` — Executes prompt logic combining the email details and Gemini.
  - *Configuration:* Evaluates email text against predefined prompting rules to output structured JSON data (event date, guest count, ICP score).
  - *Input/Output:* Inputs: `Set Enquiry Details`, `Google Gemini Model`; Outputs: `Set Output Field`.
  - *Edge Cases:* Malformed JSON responses from the LLM.

- **Set Output Field**
  - *Type & Role:* `n8n-nodes-base.set` — Normalizes the structured JSON output from the LLM chain into explicit expression paths.
  - *Configuration:* Maps extracted parameters (e.g., ICP score, event requirements).
  - *Input/Output:* Inputs: `Process with LLM Chain`; Outputs: `If ICP Match`.
  - *Edge Cases:* Missing keys in LLM JSON output causing undefined expression evaluations.

- **If ICP Match**
  - *Type & Role:* `n8n-nodes-base.if` — Conditional gatekeeper checking if the lead meets the ICP score threshold (>= 70).
  - *Configuration:* Evaluates `{{ $json.icpScore >= 70 }}`.
  - *Input/Output:* Inputs: `Set Output Field`; Outputs: `Fetch CRM Contact List` (on true).
  - *Edge Cases:* Type mismatch on the ICP score variable (string vs. number).

---

#### Block 1.3: CRM & Spreadsheet Synchronization
- **Overview:** Interacts with GoHighLevel to manage contact records and syncs state tracking with Google Sheets.
- **Nodes Involved:** `Fetch CRM Contact List`, `If Contact ID Exists`, `Post Lead to CRM`, `Read User Profile from Sheets`, `Wait for Event Trigger`

##### Node Details:
- **Fetch CRM Contact List**
  - *Type & Role:* `n8n-nodes-base.highLevel` — Queries GoHighLevel for existing contacts.
  - *Configuration:* Searches contacts by incoming sender email address using GHL OAuth2. `alwaysOutputData` is enabled.
  - *Input/Output:* Inputs: `If ICP Match`; Outputs: `If Contact ID Exists`.
  - *Edge Cases:* API connection failures, expired OAuth2 tokens.

- **If Contact ID Exists**
  - *Type & Role:* `n8n-nodes-base.if` — Checks whether a contact record was returned from GoHighLevel.
  - *Configuration:* Evaluates if a valid GHL Contact ID is present.
  - *Input/Output:* Inputs: `Fetch CRM Contact List`; Outputs: `Read User Profile from Sheets` (true branch) or `Post Lead to CRM` (false branch).
  - *Edge Cases:* Ambiguous response payloads if GHL returns empty lists.

- **Post Lead to CRM**
  - *Type & Role:* `n8n-nodes-base.httpRequest` — Creates or updates a contact in GoHighLevel if not found.
  - *Configuration:* Sends POST/PUT payloads to the GoHighLevel API with lead data and `locationId`.
  - *Input/Output:* Inputs: `If Contact ID Exists` (false branch); Outputs: `Read User Profile from Sheets`.
  - *Edge Cases:* Validation errors due to missing mandatory contact fields.

- **Read User Profile from Sheets**
  - *Type & Role:* `n8n-nodes-base.googleSheets` — Looks up the lead inside the tracking Google Sheet.
  - *Configuration:* Uses Service Account credentials to query rows by email. `alwaysOutputData` is enabled.
  - *Input/Output:* Inputs: `If Contact ID Exists` (true branch) or `Post Lead to CRM`; Outputs: `Wait for Event Trigger`.
  - *Edge Cases:* Sheet permission issues, missing column mappings.

- **Wait for Event Trigger**
  - *Type & Role:* `n8n-nodes-base.wait` — Pauses workflow execution briefly to allow external synchronization settling.
  - *Configuration:* Configured with a short pause duration using internal webhooks (`webhookId`).
  - *Input/Output:* Inputs: `Read User Profile from Sheets`; Outputs: `If User Logged in Sheets`.
  - *Edge Cases:* Webhook timeout or callback failure.

---

#### Block 1.4: Notification & State Management
- **Overview:** Evaluates Google Sheets logs and notification flags, dispatches Slack alerts for unnotified leads, and updates status flags to prevent duplicate messages.
- **Nodes Involved:** `If User Logged in Sheets`, `If User Already Notified`, `Append User to Sheets`, `Post to Slack Channel A`, `Post to Slack Channel B`, `Update Notified Status in Sheets A`, `Update Notified Status in Sheets B`

##### Node Details:
- **If User Logged in Sheets**
  - *Type & Role:* `n8n-nodes-base.if` — Evaluates if the user record already exists in Google Sheets.
  - *Configuration:* Checks lookup results from the spreadsheet.
  - *Input/Output:* Inputs: `Wait for Event Trigger`; Outputs: `If User Already Notified` (true) or `Append User to Sheets` (false).
  - *Edge Cases:* Null values returned from sheet search.

- **Append User to Sheets**
  - *Type & Role:* `n8n-nodes-base.googleSheets` — Adds a new row for unrecorded leads in Google Sheets.
  - *Configuration:* Inserts lead data with default notification status set to false.
  - *Input/Output:* Inputs: `If User Logged in Sheets` (false branch); Outputs: `Post to Slack Channel B`.
  - *Edge Cases:* Spreadsheet row write limits or schema mismatches.

- **If User Already Notified**
  - *Type & Role:* `n8n-nodes-base.if` — Verifies whether the notification flag in the sheet is already set to true.
  - *Configuration:* Evaluates the boolean status of the "notified" column.
  - *Input/Output:* Inputs: `If User Logged in Sheets` (true branch); Outputs: `Post to Slack Channel A` (false branch, meaning not yet notified).
  - *Edge Cases:* Type conversion issues with boolean sheet cell values.

- **Post to Slack Channel A & B**
  - *Type & Role:* `n8n-nodes-base.slack` — Sends lead alert notifications to a designated Slack channel.
  - *Configuration:* Uses Slack OAuth2/Bot credentials, injecting structured lead details into the message payload. Both nodes share webhook configuration.
  - *Input/Output:* Inputs: `If User Already Notified` or `Append User to Sheets`; Outputs: `Update Notified Status in Sheets A` / `Update Notified Status in Sheets B`.
  - *Edge Cases:* Slack API rate limits, invalid channel IDs, or missing bot scopes.

- **Update Notified Status in Sheets A & B**
  - *Type & Role:* `n8n-nodes-base.googleSheets` — Updates the specified row to mark the notification flag as true.
  - *Configuration:* Updates the row matching the lead's email identifier, setting the notified column to true.
  - *Input/Output:* Inputs: `Post to Slack Channel A` / `Post to Slack Channel B`; Outputs: None (Terminal nodes).
  - *Edge Cases:* Row index mismatch or update operation failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Visual workspace annotation | None | None | *(Blank)* |
| Set Enquiry Details | `n8n-nodes-base.set` | Normalizes incoming email payload fields | When Email Received | Process with LLM Chain | |
| Set Output Field | `n8n-nodes-base.set` | Maps LLM structured output variables | Process with LLM Chain | If ICP Match | |
| Fetch CRM Contact List | `n8n-nodes-base.highLevel` | Queries GoHighLevel for existing contact | If ICP Match | If Contact ID Exists | |
| If Contact ID Exists | `n8n-nodes-base.if` | Validates presence of GHL Contact ID | Fetch CRM Contact List | Read User Profile from Sheets, Post Lead to CRM | |
| Post Lead to CRM | `n8n-nodes-base.httpRequest` | Creates/updates lead in GoHighLevel | If Contact ID Exists | Read User Profile from Sheets | |
| If User Logged in Sheets | `n8n-nodes-base.if` | Checks if lead exists in Google Sheet | Wait for Event Trigger | If User Already Notified, Append User to Sheets | |
| If User Already Notified | `n8n-nodes-base.if` | Checks notification status flag | If User Logged in Sheets | Post to Slack Channel A | |
| Append User to Sheets | `n8n-nodes-base.googleSheets` | Inserts new lead row into Google Sheet | If User Logged in Sheets | Post to Slack Channel B | |
| Post to Slack Channel A | `n8n-nodes-base.slack` | Sends Slack notification for unnotified leads | If User Already Notified | Update Notified Status in Sheets A | |
| Post to Slack Channel B | `n8n-nodes-base.slack` | Sends Slack notification for newly appended leads | Append User to Sheets | Update Notified Status in Sheets B | |
| Update Notified Status in Sheets A | `n8n-nodes-base.googleSheets` | Sets notified flag to true in sheet | Post to Slack Channel A | None | |
| Update Notified Status in Sheets B | `n8n-nodes-base.googleSheets` | Sets notified flag to true in sheet | Post to Slack Channel B | None | |
| If ICP Match | `n8n-nodes-base.if` | Filters leads with ICP score >= 70 | Set Output Field | Fetch CRM Contact List | |
| Process with LLM Chain | `@n8n/n8n-nodes-langchain.chainLlm` | Executes prompt processing with Gemini | Set Enquiry Details, Google Gemini Model | Set Output Field | |
| When Email Received | `n8n-nodes-base.gmailTrigger` | Triggers on new unread emails | None | Set Enquiry Details | |
| Google Gemini Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides LLM model configuration | None | Process with LLM Chain | |
| Wait for Event Trigger | `n8n-nodes-base.wait` | Pauses execution briefly | Read User Profile from Sheets | If User Logged in Sheets | |
| Read User Profile from Sheets | `n8n-nodes-base.googleSheets` | Reads user profile row from Google Sheets | Post Lead to CRM, If Contact ID Exists | Wait for Event Trigger | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a `When Email Received` (`n8n-nodes-base.gmailTrigger`) node. Configure Gmail OAuth2 credentials and set it to monitor unread emails.
2. **Add Data Mapping (Enquiry):** Create a `Set` (`n8n-nodes-base.set`) node named `Set Enquiry Details`. Connect it as the output of `When Email Received`. Map parameters to extract the email subject, body, and sender.
3. **Configure AI Processing:**
   - Add a `Google Gemini Model` (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) node. Configure your Google PaLM/Gemini API credentials and choose your target model.
   - Add a `Process with LLM Chain` (`@n8n/n8n-nodes-langchain.chainLlm`) node. Connect the language model to its `ai_languageModel` input and `Set Enquiry Details` to its main input. Set up prompts to extract event details and calculate an ICP score.
4. **Normalize AI Output:** Add a second `Set` node named `Set Output Field`. Connect it to `Process with LLM Chain` to structure the LLM output variables.
5. **Add ICP Filter:** Add an `If` node named `If ICP Match`. Configure the condition to check if `{{ $json.icpScore >= 70 }}`. Connect `Set Output Field` to it.
6. **Integrate GoHighLevel:**
   - Add a `Fetch CRM Contact List` (`n8n-nodes-base.highLevel`) node. Connect it to the true branch of `If ICP Match`. Configure GHL OAuth2 credentials and search parameters for the sender email. Enable `alwaysOutputData`.
   - Add an `If` node named `If Contact ID Exists` to check if a valid GHL Contact ID was found.
   - Add an `HTTP Request` node named `Post Lead to CRM` (`n8n-nodes-base.httpRequest`). Connect it to the false branch of `If Contact ID Exists`. Configure the request payload with your GHL account's `locationId` and lead properties.
7. **Integrate Google Sheets (Lookup & Wait):**
   - Add a `Read User Profile from Sheets` (`n8n-nodes-base.googleSheets`) node. Connect outputs from both `If Contact ID Exists` (true branch) and `Post Lead to CRM` to it. Configure Google Sheets Service Account credentials, spreadsheet ID, sheet name, and email lookup parameters. Enable `alwaysOutputData`.
   - Add a `Wait for Event Trigger` (`n8n-nodes-base.wait`) node connected to the Sheets read output.
8. **Configure State Logic & Notifications:**
   - Add an `If` node named `If User Logged in Sheets` connected after the wait node.
   - **Branch 1 (Existing User Check):**
     - Add an `If` node named `If User Already Notified` connected to the true branch.
     - Add a `Slack` node named `Post to Slack Channel A` (`n8n-nodes-base.slack`) connected to the false branch of the notification check. Configure Slack credentials and destination channel.
     - Add a `Google Sheets` node named `Update Notified Status in Sheets A` connected to Slack Node A to set the notified flag to true.
   - **Branch 2 (New User Insertion):**
     - Add a `Google Sheets` node named `Append User to Sheets` connected to the false branch of `If User Logged in Sheets`. Configure it to insert a new row with the notified status set to false.
     - Add a `Slack` node named `Post to Slack Channel B` connected to `Append User to Sheets`.
     - Add a `Google Sheets` node named `Update Notified Status in Sheets B` connected to Slack Node B to update the notified flag to true.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| YouTube Video Walkthrough | [https://youtu.be/T1e7SoVQfbs](https://youtu.be/T1e7SoVQfbs) |
| Author LinkedIn Profile | [https://www.linkedin.com/in/iamvaar/](https://www.linkedin.com/in/iamvaar/) |