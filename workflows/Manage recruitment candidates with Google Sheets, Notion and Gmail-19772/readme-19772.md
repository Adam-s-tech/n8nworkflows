Manage recruitment candidates with Google Sheets, Notion and Gmail

https://n8nworkflows.xyz/workflows/manage-recruitment-candidates-with-google-sheets--notion-and-gmail-19772


# Manage recruitment candidates with Google Sheets, Notion and Gmail

### 1. Workflow Overview

This workflow automates the candidate recruitment pipeline by synchronizing applications from Google Sheets into a Notion Applicant Tracking System (ATS), and automatically dispatching personalized notification emails via Gmail when a candidate's recruitment status is updated in Notion. 

The architecture divides into two independent operational paths:
- **1.1 Application Ingestion & ATS Sync:** Pulls candidate submissions from a Google Sheet, verifies against existing Notion records to prevent duplicates, inserts new candidates into the Notion Candidates Tracker database, and updates the Google Sheet row status to mark them as processed.
- **1.2 Status-Triggered Candidate Notifications:** Listens for updates in the Notion Candidates Tracker, compares the current state with historical records stored in an n8n data table to confirm actual status changes, fetches the corresponding email template from Notion, dynamically compiles personal placeholders, and delivers the message via Gmail.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Application Ingestion & ATS Sync
- **Overview:** Periodically or manually retrieves new job application rows from Google Sheets, verifies whether the candidate already exists in the Notion ATS, adds missing entries, and marks the spreadsheet row as processed.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
  - Schedule Trigger
  - Get new applicants
  - Get many database pages
  - Filter candidates not in lists
  - Check if need insert
  - Create a database page
  - Data processed
  - Update processed status

- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - *Type and technical role:* Manual trigger for testing and ad-hoc executions.
    - *Configuration choices:* Default parameters (no inputs).
    - *Key expressions or variables:* None.
    - *Input and output connections:* Output connects to `Get new applicants`.
    - *Version-specific requirements:* v1.
    - *Edge cases:* Manual runs will bypass schedule intervals.
  - **Schedule Trigger**
    - *Type and technical role:* Interval-based recurring trigger to automate application polling.
    - *Configuration choices:* Interval set to run every few seconds.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Output connects to `Get new applicants`.
    - *Version-specific requirements:* v1.3.
    - *Edge cases:* High frequency may hit rate limits on Google Sheets or Notion APIs.
  - **Get new applicants**
    - *Type and technical role:* Google Sheets integration node to read form response rows.
    - *Configuration choices:* Operation set to lookup/get, filtered by the "Processed" column. Configured with `executeOnce: true`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Inputs from triggers; output connects to `Get many database pages`.
    - *Credentials:* `Google Auth` (Google Sheets OAuth2 API).
    - *Version-specific requirements:* v4.7.
    - *Edge cases:* Spreadsheet structural changes or missing headers can cause read failures.
  - **Get many database pages**
    - *Type and technical role:* Notion integration node to fetch existing candidates.
    - *Configuration choices:* Resource set to database page, operation set to `getAll`, with manual filtering comparing email addresses. `alwaysOutputData` enabled.
    - *Key expressions or variables:* `={{ $json['Email Address'] }}`
    - *Input and output connections:* Input from `Get new applicants`; output connects to `Filter candidates not in lists`.
    - *Credentials:* `Notion account` (Notion OAuth2 API).
    - *Version-specific requirements:* v2.2.
    - *Edge cases:* Large databases may require pagination handling.
  - **Filter candidates not in lists**
    - *Type and technical role:* JavaScript code node to execute set operations and filter out already-tracked candidates.
    - *Configuration choices:* Custom JavaScript processing input streams. `alwaysOutputData` enabled.
    - *Key expressions or variables:* Evaluates `$input.all()` against `$('Get new applicants').all()`.
    - *Input and output connections:* Input from `Get many database pages`; output connects to `Check if need insert`.
    - *Version-specific requirements:* v2.
    - *Edge cases:* Mismatched email case-sensitivity.
  - **Check if need insert**
    - *Type and technical role:* Conditional branch node.
    - *Configuration choices:* Checks for the existence of the applicant email value.
    - *Key expressions or variables:* `={{ $input.item.json["Email Address"] }}`
    - *Input and output connections:* Input from `Filter candidates not in lists`; true branch connects to `Create a database page`, false branch connects to `Data processed`.
    - *Version-specific requirements:* v2.3.
    - *Edge cases:* Empty rows passing through filter logic.
  - **Create a database page**
    - *Type and technical role:* Notion integration node to create new candidate records.
    - *Configuration choices:* Resource set to database page, inserting candidate properties (Name, Email, Whatsapp, Role, CV/Resume file URL, and Status set to `INTAKE`). `alwaysOutputData` enabled.
    - *Key expressions or variables:* Uses dynamic JSON references like `={{ $json.Name }}` and `={{ $json['Email Address'] }}`.
    - *Input and output connections:* Input from `Check if need insert` (true branch); output connects to `Data processed`.
    - *Credentials:* `Notion account` (Notion OAuth2 API).
    - *Version-specific requirements:* v2.2.
    - *Edge cases:* Missing required Notion properties or invalid file URLs.
  - **Data processed**
    - *Type and technical role:* JavaScript code node to aggregate original applicant data for spreadsheet status updating. `executeOnce: true`.
    - *Configuration choices:* Returns all items from `Get new applicants`.
    - *Key expressions or variables:* `$('Get new applicants').all()`
    - *Input and output connections:* Inputs from `Check if need insert` (false branch) and `Create a database page`; output connects to `Update processed status`.
    - *Version-specific requirements:* v2.
    - *Edge cases:* Out-of-order index reference if data arrays shift.
  - **Update processed status**
    - *Type and technical role:* Google Sheets integration node to flag rows as processed.
    - *Configuration choices:* Operation set to update, matching on row number, mapping `Processed` to `"Yes"`.
    - *Key expressions or variables:* `={{ $json.row_number }}`
    - *Input and output connections:* Input from `Data processed`; no downstream outputs.
    - *Credentials:* `Google Auth` (Google Sheets OAuth2 API).
    - *Version-specific requirements:* v4.7.
    - *Edge cases:* Row index shifts if rows are deleted concurrently in Google Sheets.

---

#### Block 1.2: Status-Triggered Candidate Notifications
- **Overview:** Monitors Notion candidate records for modifications, checks historical states in a local n8n data table to confirm if the recruitment status actually changed, fetches matching email templates, compiles tokens, and sends automated emails via Gmail.
- **Nodes Involved:** 
  - Notion Trigger
  - Get old state
  - Save new state
  - Switch
  - Get email template
  - Formatting email from template
  - Send a message

- **Node Details:**
  - **Notion Trigger**
    - *Type and technical role:* Event-driven trigger listening to Notion database updates.
    - *Configuration choices:* Event set to page updated in database, polling interval set to every minute. Authentication via OAuth2.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Output connects to `Get old state`.
    - *Credentials:* `Notion account` (Notion OAuth2 API).
    - *Version-specific requirements:* v1.
    - *Edge cases:* Polling delays up to 1 minute.
  - **Get old state**
    - *Type and technical role:* n8n Data Table lookup node to retrieve historical row states.
    - *Configuration choices:* Operation set to `get`, matching historical records by `notion_id`. `alwaysOutputData` enabled.
    - *Key expressions or variables:* `={{ $json.id }}`
    - *Input and output connections:* Input from `Notion Trigger`; output connects to `Save new state`.
    - *Version-specific requirements:* v1.1.
    - *Edge cases:* First-time updates may return empty states, requiring safe null handling.
  - **Save new state**
    - *Type and technical role:* n8n Data Table upsert node to persist the latest trigger payload.
    - *Configuration choices:* Operation set to `upsert`, matching by `notion_id`, storing serialized JSON strings of the trigger data.
    - *Key expressions or variables:* `={{ JSON.stringify($('Notion Trigger').item.json) }}` and `={{ $('Notion Trigger').item.json.id }}`
    - *Input and output connections:* Input from `Get old state`; output connects to `Switch`.
    - *Version-specific requirements:* v1.1.
    - *Edge cases:* Write collisions on rapid consecutive status changes.
  - **Switch**
    - *Type and technical role:* Conditional expression routing node.
    - *Configuration choices:* Mode set to expression with 2 outputs. Compares old state status with new state status to suppress duplicate triggers if statuses match.
    - *Key expressions or variables:* Evaluates if `newState?.Status === oldState?.Status`.
    - *Input and output connections:* Input from `Save new state`; output connects to `Get email template`.
    - *Version-specific requirements:* v3.4.
    - *Edge cases:* Syntax or null property errors if historical JSON string parsing fails.
  - **Get email template**
    - *Type and technical role:* Notion integration node to fetch contextual email templates.
    - *Configuration choices:* Resource set to database page, operation set to `getAll`, matching the template's status filter to the candidate's current Notion status.
    - *Key expressions or variables:* `={{ $('Notion Trigger').item.json.Status }}`
    - *Input and output connections:* Input from `Switch`; output connects to `Formatting email from template`.
    - *Credentials:* `Notion account` (Notion OAuth2 API).
    - *Version-specific requirements:* v2.2.
    - *Edge cases:* Missing template for a specific status results in empty payload output.
  - **Formatting email from template**
    - *Type and technical role:* JavaScript code node to replace dynamic placeholders (e.g., `[Name]`) with candidate property values.
    - *Configuration choices:* Custom regex substitution script parsing template subject and body fields against the Notion trigger payload.
    - *Key expressions or variables:* `$input.first().json.property_subject`, `$input.first().json.property_body`, and `$("Notion Trigger").first().json`.
    - *Input and output connections:* Input from `Get email template`; output connects to `Send a message`.
    - *Version-specific requirements:* v2.
    - *Edge cases:* Unmatched placeholder keys fallback to empty strings.
  - **Send a message**
    - *Type and technical role:* Gmail integration node to dispatch the notification email.
    - *Configuration choices:* Email type set to text, attribution disabled.
    - *Key expressions or variables:* `={{ $('Notion Trigger').item.json.Email }}`, `={{ $json.email_body }}`, `={{ $json.email_subject }}`
    - *Input and output connections:* Input from `Formatting email from template`; no downstream outputs.
    - *Credentials:* `Gmail account` (Gmail OAuth2).
    - *Version-specific requirements:* v2.2.
    - *Edge cases:* Invalid email formatting, revoked Gmail scopes, or daily sending quotas exceeded.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual trigger for ad-hoc execution | None | Get new applicants | |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Interval-based trigger for polling applications | None | Get new applicants | |
| Get new applicants | n8n-nodes-base.googleSheets | Retrieves form responses from Google Sheets | When clicking ‘Execute workflow’, Schedule Trigger | Get many database pages | ## Get candidates and filtered it<br><br>Get the candidates from the google sheets, then we also get the candidates in our notion, and do filter the candidates that is not already in notion |
| Get many database pages | n8n-nodes-base.notion | Fetches existing candidates from Notion ATS | Get new applicants | Filter candidates not in lists | ## Get candidates and filtered it<br><br>Get the candidates from the google sheets, then we also get the candidates in our notion, and do filter the candidates that is not already in notion |
| Filter candidates not in lists | n8n-nodes-base.code | Filters out candidates already present in Notion | Get many database pages | Check if need insert | ## Get candidates and filtered it<br><br>Get the candidates from the google sheets, then we also get the candidates in our notion, and do filter the candidates that is not already in notion |
| Check if need insert | n8n-nodes-base.if | Conditional routing to check if email exists | Filter candidates not in lists | Create a database page, Data processed | ## Insert the candidates<br><br>Check if candidate email exists for validation, then insert the candidates data to the notion. You can also update the fields as you needed. |
| Create a database page | n8n-nodes-base.notion | Inserts new candidate data into Notion ATS | Check if need insert | Data processed | ## Insert the candidates<br><br>Check if candidate email exists for validation, then insert the candidates data to the notion. You can also update the fields as you needed. |
| Data processed | n8n-nodes-base.code | Aggregates processed applicant context | Check if need insert, Create a database page | Update processed status | ## Update our google sheet<br><br>After insert the candidate data to notion we update our google sheets status that the candidates already inserted |
| Update processed status | n8n-nodes-base.googleSheets | Updates Google Sheet row to mark candidate as processed | Data processed | None | ## Update our google sheet<br><br>After insert the candidate data to notion we update our google sheets status that the candidates already inserted |
| Notion Trigger | n8n-nodes-base.notionTrigger | Triggers when a Notion database page is updated | None | Get old state | ## Trigger when notion update<br><br>It triggers when the notion database page was updated |
| Get old state | n8n-nodes-base.dataTable | Retrieves historical state from n8n data table | Notion Trigger | Save new state | ## Get old state and update new state<br><br>Try to get the old state of our notion data in our data tables, we used it later for compare the status (because notion not giving the old state). And also we save the new state to update our data tables |
| Save new state | n8n-nodes-base.dataTable | Persists latest payload to n8n data table | Get old state | Switch | ## Get old state and update new state<br><br>Try to get the old state of our notion data in our data tables, we used it later for compare the status (because notion not giving the old state). And also we save the new state to update our data tables |
| Switch | n8n-nodes-base.switch | Evaluates whether candidate status changed | Save new state | Get email template | ## Check status<br><br>Check whether status was updated from the old state, then we continue to process |
| Get email template | n8n-nodes-base.notion | Fetches email template based on candidate status | Switch | Formatting email from template | ## Send email template<br><br>Get the email templates first from our notion based on the status, then we formatted it first to apply the candidates data into the email. After that we send that email using gmail |
| Formatting email from template | n8n-nodes-base.code | Replaces placeholders with candidate values | Get email template | Send a message | ## Send email template<br><br>Get the email templates first from our notion based on the status, then we formatted it first to apply the candidates data into the email. After that we send that email using gmail |
| Send a message | n8n-nodes-base.gmail | Sends the compiled notification email via Gmail | Formatting email from template | None | ## Send email template<br><br>Get the email templates first from our notion based on the status, then we formatted it first to apply the candidates data into the email. After that we send that email using gmail |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Tables**: Create an n8n data table named `candidates_trackers_histories` containing the following schema:
   - `id`: string
   - `notion_id`: string
   - `values`: string
   - `created_at`: datetime
   - `updated_at`: datetime

2. **Set Up Triggers**:
   - Create a **Manual Trigger** (`When clicking ‘Execute workflow'`).
   - Create a **Schedule Trigger** set to check at frequent intervals (e.g., every few seconds).
   - Create a **Notion Trigger** configured to monitor your `Candidates Tracker` database for `Page Updated in Database` events with a 1-minute polling interval. Authenticate using your Notion OAuth2 credential.

3. **Build the Ingestion Pipeline**:
   - Add a **Google Sheets** node (`Get new applicants`). Set operation to lookup/get, select your recruitment applications spreadsheet and sheet, and filter by the `Processed` column. Execute once per run. Configure your `Google Auth` credential.
   - Add a **Notion** node (`Get many database pages`). Resource: `Database Page`, Operation: `Get All`. Filter by email matching the Google Sheet's `Email Address`. Enable `alwaysOutputData`.
   - Add a **Code** node (`Filter candidates not in lists`). Add JavaScript logic to compare Google Sheet applicant emails against the items returned from Notion. Enable `alwaysOutputData`.
   - Add an **If** node (`Check if need insert`). Evaluate whether the applicant email property exists.
   - Add a **Notion** node (`Create a database page`). Map incoming sheet properties (`Name`, `Email Address`, `Whatsapp / Phone Number`, `Role`, `CV / Resume`) to corresponding Notion database properties. Set initial status to `INTAKE`.
   - Add a **Code** node (`Data processed`) to compile the applicant list.
   - Add a **Google Sheets** node (`Update processed status`). Operation: `Update`, matching on `row_number` and setting `Processed` to `"Yes"`.

4. **Build the Notification Pipeline**:
   - Connect the `Notion Trigger` to a **Data Table** node (`Get old state`). Operation: `Get`, filtering by `notion_id` matching `={{ $json.id }}`. Enable `alwaysOutputData`.
   - Connect to another **Data Table** node (`Save new state`). Operation: `Upsert`, matching by `notion_id`, storing serialized values using `={{ JSON.stringify($('Notion Trigger').item.json) }}`.
   - Connect to a **Switch** node (`Switch`). Set mode to Expression, comparing `newState?.Status === oldState?.Status` across 2 outputs.
   - Connect output to a **Notion** node (`Get email template`). Resource: `Database Page`, Operation: `Get All`, filtering by database templates where status equals `={{ $('Notion Trigger').item.json.Status }}`.
   - Connect to a **Code** node (`Formatting email from template`). Add the JavaScript token replacement script using regular expressions to swap bracketed placeholders like `[Name]` with actual candidate field values.
   - Connect to a **Gmail** node (`Send a message`). Configure recipient to `={{ $('Notion Trigger').item.json.Email }}`, subject to `={{ $json.email_subject }}`, and message body to `={{ $json.email_body }}`. Authenticate using your `Gmail account` credential.

5. **Establish Connections**: Connect nodes sequentially according to the mapping specified in Section 3.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Notion Recruitment Board Template Example | https://app.notion.com/p/Recruitment-Board-3d12566679c480fcb916d16c014c8197?source=copy_link |
| Google Sheets Applications Spreadsheet Example | https://docs.google.com/spreadsheets/d/1mjlkvPr2qX3GMaMLSAFuhJ63ptYhXk_Qtprk7y2YfPg/edit?usp=sharing |