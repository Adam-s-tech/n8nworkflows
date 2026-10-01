Log Québec Law 25 confidentiality incidents with forms, Data Tables and email

https://n8nworkflows.xyz/workflows/log-qu-bec-law-25-confidentiality-incidents-with-forms--data-tables-and-email-19691


# Log Québec Law 25 confidentiality incidents with forms, Data Tables and email

### 1. Workflow Overview

This workflow automates the tracking, recording, updating, and reporting of confidentiality incidents required under Québec Law 25 (Act respecting the protection of personal information in the private sector). It serves as an internal compliance registry, ensuring that businesses accurately document security breaches, evaluate serious injury risks, notify authorities and affected individuals promptly, track mitigation measures, and enforce a mandatory 5-year retention schedule.

The workflow logic is divided into four primary functional blocks:

- **1.1 Incident Intake and Registration:** Captures new incident details via a secure n8n form, performs validation checks on dates and input values, generates a unique reference ID, calculates the 5-year data retention deadline, writes the record to an n8n Data Table, and presents confirmation details with legal next steps.
- **1.2 Incident Updates and Closure:** Processes subsequent modifications via a secondary form, searches the Data Table for the target record, merges new notice dates or extra mitigation measures, enforces closure requirements for serious incidents, and updates the register row.
- **1.3 Daily Compliance Reminders:** Automatically triggers every morning at 8:00 AM, retrieves all database records, filters incidents requiring immediate attention (pending notices, unassessed risks past threshold, or expired retention periods), and emails a summary to the privacy officer.
- **1.4 Register Export:** Allows manual generation and downloading of the complete incident registry as an Excel (`.xlsx`) file for regulatory audits upon request.

---

### 2. Block-by-Block Analysis

#### 2.1 Incident Intake and Registration
- **Overview:** Receives staff submissions regarding potential data breaches, validates logical constraints, structures data according to Québec Regulation respecting confidentiality incidents (s. 7), logs the entry into an n8n Data Table, and outputs compliance guidance.
- **Nodes Involved:** 
  - `Staff: record an incident`
  - `Build register entry`
  - `Create register table if missing`
  - `Keep register columns only`
  - `Add incident to register`
  - `Write next steps`
  - `Show the reference and next steps`
  - `Show why nothing was saved`
- **Node Details:**
  - **Staff: record an incident**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger)
    - *Configuration Choices:* Protected via `n8nUserAuth`. Collects dropdowns, dates, numbers, and textareas covering breach types, personal info categories, timestamps, number of impacted individuals, and risk assessments.
    - *Key Expressions or Variables:* Uses standard n8n Form fields (`$json`).
    - *Input/Output:* Trigger node; outputs data directly to `Build register entry`.
    - *Edge Cases/Failure Types:* Authentication failure if staff lack an n8n account; validation errors if required fields are omitted.
  - **Build register entry**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Validates chronological order of dates (e.g., end date cannot precede start date; awareness date cannot be in the future), checks for negative person counts, generates warnings for unconsulted privacy officers or mismatched risk evaluations, and calculates the 5-year retention deadline (`keep_until`).
    - *Key Expressions or Variables:* Uses `$input.first().json`, `$now`, and `DateTime.fromISO()`.
    - *Input/Output:* Input: Form data. Output: Structured JSON payload with internal helper attributes (`_warnings`, `_serious`).
    - *Edge Cases/Failure Types:* Throws a custom Error if validation checks fail, routing execution to the error path (`Show why nothing was saved`).
  - **Create register table if missing**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `table`, Operation: `create`, Table Name: `law25_incident_register`, Option `createIfNotExists: true`. Defines all 25 required schema columns matching Law 25 specifications.
    - *Input/Output:* Input: Structured entry; Output: Confirms table availability and passes items downstream.
    - *Edge Cases/Failure Types:* Database permission failures or schema mismatch.
  - **Keep register columns only**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Strips temporary metadata keys (`_warnings`, `_serious`) from the payload prior to database insertion.
    - *Input/Output:* Input: Object from `Build register entry`. Output: Clean object matching exact table schema.
    - *Edge Cases/Failure Types:* None expected unless data schema mismatch occurs.
  - **Add incident to register**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `row`, Operation: `insert`, Mapping Mode: `autoMapInputData`.
    - *Input/Output:* Input: Cleaned register entry. Output: Confirmation object.
    - *Edge Cases/Failure Types:* Insertion failure due to data type mismatch.
  - **Write next steps**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Formats an HTML confirmation page displaying the generated reference ID, retention expiry date, warnings, and regulatory guidelines for serious injury cases.
    - *Input/Output:* Input: Register payload. Output: HTML string payload.
    - *Edge Cases/Failure Types:* None.
  - **Show the reference and next steps**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Response)
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, Response text set to `={{ $json.html }}`.
    - *Input/Output:* Input: HTML string. Output: Final user-facing webpage.
    - *Edge Cases/Failure Types:* Session timeout.
  - **Show why nothing was saved**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Response)
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, parses error messages from upstream execution failures.
    - *Input/Output:* Input: Error object from `Build register entry`. Output: User-facing error page.
    - *Edge Cases/Failure Types:* None.

---

#### 2.2 Incident Updates and Closure
- **Overview:** Allows staff to update existing incident records by appending notification dates, public notice justifications, delay reasons, or additional mitigation steps, while preventing premature closure of non-compliant records.
- **Nodes Involved:**
  - `Staff: record notices and close`
  - `Create register table if missing (update)`
  - `Find the incident`
  - `Check and merge the update`
  - `Update the register`
  - `Show the update`
  - `Show why nothing was updated`
- **Node Details:**
  - **Staff: record notices and close**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger)
    - *Configuration Choices:* Protected via `n8nUserAuth`. Collects incident reference ID, conclusion adjustments, regulatory notification dates, public notice text, investigation delay reasons, additional measures, and a closure toggle.
    - *Input/Output:* Trigger node; outputs form parameters downstream.
    - *Edge Cases/Failure Types:* Authentication failure or missing mandatory reference ID.
  - **Create register table if missing (update)**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Ensures `law25_incident_register` table exists prior to lookup.
    - *Input/Output:* Input: Form trigger data. Output: Data table continuity.
  - **Find the incident**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `row`, Operation: `get`, Filter: `incident_id` equals form input reference (trimmed). Returns all matching rows (`returnAll: true`, `alwaysOutputData: true`).
    - *Input/Output:* Input: Form fields. Output: Existing database row data.
    - *Edge Cases/Failure Types:* Incident ID not found.
  - **Check and merge the update**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Validates whether the target incident exists, ensures notice dates do not precede the awareness date, appends new mitigation measures with timestamps without overwriting historical data, checks if mandatory notices are present when closing serious incidents, and computes updated status strings.
    - *Input/Output:* Input: Database row and update form data. Output: Merged update payload or thrown validation error.
    - *Edge Cases/Failure Types:* Throws error if required fields are missing during closure attempts.
  - **Update the register**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `row`, Operation: `update`, Mapping Mode: `defineBelow`. Maps status, notices, risk evaluation, and measures using filters matching `incident_id`.
    - *Input/Output:* Input: Merged update object. Output: Database modification confirmation.
    - *Edge Cases/Failure Types:* Database write errors.
  - **Show the update**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Response)
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, displays summary HTML confirming update status.
    - *Input/Output:* Input: HTML confirmation string. Output: User-facing success page.
  - **Show why nothing was updated**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Response)
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, displays parsed validation error details when updates or closures fail.
    - *Input/Output:* Input: Error object. Output: User-facing error page.

---

#### 2.3 Daily Compliance Reminders
- **Overview:** Executes a scheduled daily audit to detect overdue regulatory notifications, prolonged unassessed risk statuses, or expired data retention windows, and emails an automated summary report to the privacy officer.
- **Nodes Involved:**
  - `Every morning at 8`
  - `Settings`
  - `Create register table if missing (reminder)`
  - `Get all incidents`
  - `List incidents needing action`
  - `Email the person in charge`
- **Node Details:**
  - **Every morning at 8**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - *Configuration Choices:* Interval set to trigger daily at hour 8.
    - *Input/Output:* Trigger node; initiates the daily check flow.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields / Set)
    - *Configuration Choices:* Assigns global configuration variables: `privacy_officer_email`, `from_email`, and `assess_within_days` (default: 2).
    - *Input/Output:* Input: Trigger signal. Output: Configuration object.
  - **Create register table if missing (reminder)**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Ensures table availability prior to retrieval.
    - *Input/Output:* Input: Settings object. Output: Data table continuity.
  - **Get all incidents**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `row`, Operation: `get`, `returnAll: true`, `alwaysOutputData: true`. Retrieves the complete register.
    - *Input/Output:* Input: Settings/Table confirmation. Output: Array of all table records.
  - **List incidents needing action**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Filters register rows into three criteria: status `notices due`, status `to assess` exceeding configured threshold days, and `keep_until` dates earlier than current date. If no items match, returns an empty array to halt email dispatch.
    - *Input/Output:* Input: Register rows and settings. Output: Formatted email object containing recipient, sender, subject, and HTML body (or empty array).
    - *Edge Cases/Failure Types:* Empty input arrays cleanly short-circuit execution preventing unnecessary email generation.
  - **Email the person in charge**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Email Send)
    - *Configuration Choices:* Email format: `html`, maps fields from incoming JSON (`to`, `from`, `subject`, `html`). Requires SMTP or equivalent email credentials.
    - *Input/Output:* Input: Email object. Output: Delivery confirmation.
    - *Edge Cases/Failure Types:* Authentication failure, invalid recipient addresses, or SMTP server timeouts.

---

#### 2.4 Register Export
- **Overview:** Facilitates manual data extraction, converting the entire incident register database into an Excel spreadsheet for regulatory review.
- **Nodes Involved:**
  - `Export the register`
  - `Create register table if missing (export)`
  - `Get all incidents (export)`
  - `Convert register to Excel`
- **Node Details:**
  - **Export the register**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger)
    - *Configuration Choices:* User-initiated manual trigger.
    - *Input/Output:* Trigger node; starts export flow.
  - **Create register table if missing (export)**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Ensures table existence.
    - *Input/Output:* Input: Manual trigger signal. Output: Table confirmation.
  - **Get all incidents (export)**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (Data Table Operation)
    - *Configuration Choices:* Resource: `row`, Operation: `get`, `returnAll: true`.
    - *Input/Output:* Input: Table confirmation. Output: All database rows.
  - **Convert register to Excel**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Convert to File)
    - *Configuration Choices:* Operation: `xlsx`, Binary Property Name: `data`, Sheet Name: `Incidents`, Header Row: `true`, File Name: dynamic expression (`=law25-incident-register-{{ $now.toISODate() }}.xlsx`).
    - *Input/Output:* Input: Database rows. Output: Binary Excel file ready for download.
    - *Edge Cases/Failure Types:* Large dataset memory limits (though generally negligible for standard registry volumes).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Staff: record an incident | n8n-nodes-base.formTrigger | Collects new incident details from authorized staff | None | Build register entry | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Build register entry | n8n-nodes-base.code | Validates dates, calculates retention dates, and builds payload | Staff: record an incident | Create register table if missing, Show why nothing was saved | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Create register table if missing | n8n-nodes-base.dataTable | Ensures table schema exists in database | Build register entry | Keep register columns only | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Keep register columns only | n8n-nodes-base.code | Removes helper metadata before database write | Create register table if missing | Add incident to register | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Add incident to register | n8n-nodes-base.dataTable | Inserts validated incident into Data Table | Keep register columns only | Write next steps | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Write next steps | n8n-nodes-base.code | Formats HTML confirmation and legal instructions | Add incident to register | Show the reference and next steps | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Show the reference and next steps | n8n-nodes-base.form | Displays success page with reference and next steps | Write next steps | None | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Show why nothing was saved | n8n-nodes-base.form | Displays validation error feedback page | Build register entry | None | ### 1. Record an incident (staff only)<br>The fields follow Regulation respecting confidentiality incidents, s. 7. The risk factors are the ones s. 3.7 names: sensitivity of the information, anticipated consequences of its use, likelihood of injurious use. Record the incident the day you learn of it, even if the assessment is not finished: choose "Not yet assessed". |
| Staff: record notices and close | n8n-nodes-base.formTrigger | Receives incident updates and closure instructions | None | Create register table if missing (update) | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Create register table if missing (update) | n8n-nodes-base.dataTable | Ensures table schema exists for updates | Staff: record notices and close | Find the incident | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Find the incident | n8n-nodes-base.dataTable | Retrieves target incident row from database | Create register table if missing (update) | Check and merge the update | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Check and merge the update | n8n-nodes-base.code | Merges updates, validates closure rules | Find the incident | Update the register, Show why nothing was updated | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Update the register | n8n-nodes-base.dataTable | Applies updates to database row | Check and merge the update | Show the update | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Show the update | n8n-nodes-base.form | Displays update success completion page | Update the register | None | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Show why nothing was updated | n8n-nodes-base.form | Displays update error completion page | Check and merge the update | None | ### 2. Notices, measures, closing (staff only)<br>Empty fields keep what is in the register. Added measures are appended with the date, never overwritten. |
| Every morning at 8 | n8n-nodes-base.scheduleTrigger | Triggers daily compliance checks at 8:00 AM | None | Settings | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| Settings | n8n-nodes-base.set | Sets global email and timing variables | Every morning at 8 | Create register table if missing (reminder) | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| Create register table if missing (reminder) | n8n-nodes-base.dataTable | Ensures table schema exists for reminders | Settings | Get all incidents | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| Get all incidents | n8n-nodes-base.dataTable | Retrieves all register rows for audit | Create register table if missing (reminder) | List incidents needing action | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| List incidents needing action | n8n-nodes-base.code | Filters overdue notices, unassessed risks, expired items | Get all incidents | Email the person in charge | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| Email the person in charge | n8n-nodes-base.emailSend | Sends daily summary email to privacy officer | List incidents needing action | None | ### 3. Daily reminder<br>Lists what still needs action. Set the recipient in **Settings** and add email credentials. |
| Export the register | n8n-nodes-base.manualTrigger | Triggers manual Excel export | None | Create register table if missing (export) | ### 4. Export<br>Run manually to download the whole register as an Excel file. A copy must be sent to the Commission at its request (s. 3.8). |
| Create register table if missing (export) | n8n-nodes-base.dataTable | Ensures table schema exists for export | Export the register | Get all incidents (export) | ### 4. Export<br>Run manually to download the whole register as an Excel file. A copy must be sent to the Commission at its request (s. 3.8). |
| Get all incidents (export) | n8n-nodes-base.dataTable | Retrieves all rows for file generation | Create register table if missing (export) | Convert register to Excel | ### 4. Export<br>Run manually to download the whole register as an Excel file. A copy must be sent to the Commission at its request (s. 3.8). |
| Convert register to Excel | n8n-nodes-base.convertToFile | Converts rows to binary Excel (.xlsx) file | Get all incidents (export) | None | ### 4. Export<br>Run manually to download the whole register as an Excel file. A copy must be sent to the Commission at its request (s. 3.8). |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

1. **Create the Incident Recording Sub-Flow:**
   - **Node 1 (`Staff: record an incident`):** Add a **Form Trigger** node. Set title to `"Record a confidentiality incident"`. Configure authentication to `n8nUserAuth`. Add form fields matching Law 25 register specifications (Dropdown for `incident_type`, Multi-select dropdown for `info_categories`, textareas for `info_details` and `circumstances`, date fields for `incident_from`, `incident_to`, `became_aware_on`, dropdown for dates approximate, number field for `persons_concerned`, dropdowns for persons approximate, sensitivity, consequences, likelihood, textarea for `risk_reasoning`, dropdown for consulted, dropdown for decision, and textarea for measures).
   - **Node 2 (`Build register entry`):** Add a **Code** node connected to the success output of Node 1. Insert JavaScript logic to validate date logic (end date >= start date, awareness date >= start date, awareness date <= today, persons >= 0), compute retention expiry (`keep_until` = `became_aware_on` + 5 years), generate a unique ID (`INC-YYYYMMDD-XXXX`), and compile warnings.
   - **Node 3 (`Create register table if missing`):** Add a **Data Table** node (Resource: `table`, Operation: `create`, Table Name: `law25_incident_register`). Define all 25 schema columns (`incident_id`, `recorded_on`, `recorded_by`, `incident_type`, `information_concerned`, `circumstances`, `incident_from`, `incident_to`, `incident_dates_approximate`, `became_aware_on`, `persons_concerned`, `persons_approximate`, `sensitivity`, `anticipated_consequences`, `likelihood_of_injurious_use`, `risk_reasoning`, `person_in_charge_consulted`, `risk_of_serious_injury`, `measures_to_reduce_risk`, `cai_notified_on`, `persons_notified_on`, `public_notice`, `person_notice_delay_reason`, `status`, `keep_until`) with appropriate string/number types. Enable `createIfNotExists`.
   - **Node 4 (`Keep register columns only`):** Add a **Code** node to strip helper metadata keys (`_warnings`, `_serious`).
   - **Node 5 (`Add incident to register`):** Add a **Data Table** node (Resource: `row`, Operation: `insert`, Mapping Mode: `autoMapInputData`).
   - **Node 6 (`Write next steps`):** Add a **Code** node to build an HTML response string containing reference numbers, retention instructions, and regulatory notices.
   - **Node 7 (`Show the reference and next steps`):** Add a **Form** node (Operation: `completion`, Respond with: `showText`, Response text: `={{ $json.html }}`).
   - **Node 8 (`Show why nothing was saved`):** Add a **Form** node connected to the error output (`onError: continueErrorOutput`) of Node 2. Set operation to `completion`, Respond with: `showText`, displaying parsed error messages.

2. **Create the Incident Update and Closure Sub-Flow:**
   - **Node 9 (`Staff: record notices and close`):** Add a **Form Trigger** node. Set title to `"Update a confidentiality incident"`, auth to `n8nUserAuth`. Add fields for `incident_id` (text), `decision` (dropdown), `cai_notified_on` (date), `persons_notified_on` (date), `public_notice` (textarea), `delay_reason` (textarea), `more_measures` (textarea), and `close` (dropdown).
   - **Node 10 (`Create register table if missing (update)`):** Add a **Data Table** node ensuring table existence.
   - **Node 11 (`Find the incident`):** Add a **Data Table** node (Resource: `row`, Operation: `get`, `returnAll: true`). Filter by condition: `incident_id` equals `={{ $('Staff: record notices and close').first().json.incident_id.trim() }}`.
   - **Node 12 (`Check and merge the update`):** Add a **Code** node to verify incident existence, validate notice dates, append new measures with timestamps, enforce missing field checks for closure, and output updated status. Connect error output to Node 15.
   - **Node 13 (`Update the register`):** Add a **Data Table** node (Resource: `row`, Operation: `update`, Mapping Mode: `defineBelow`). Set filter matching `incident_id` to `={{ $('Check and merge the update').first().json._incident_id }}` and map updated fields.
   - **Node 14 (`Show the update`):** Add a **Form** node (Operation: `completion`, Respond with: `showText`, Response text: `={{ $('Check and merge the update').first().json._html }}`).
   - **Node 15 (`Show why nothing was updated`):** Add a **Form** node connected to the error output of Node 12 to display update errors.

3. **Create the Daily Reminder Sub-Flow:**
   - **Node 16 (`Every morning at 8`):** Add a **Schedule Trigger** node set to trigger at hour 8.
   - **Node 17 (`Settings`):** Add a **Set** node assigning assignments for `privacy_officer_email`, `from_email`, and `assess_within_days`.
   - **Node 18 (`Create register table if missing (reminder)`):** Add a **Data Table** node ensuring table existence.
   - **Node 19 (`Get all incidents`):** Add a **Data Table** node (Resource: `row`, Operation: `get`, `returnAll: true`).
   - **Node 20 (`List incidents needing action`):** Add a **Code** node filtering rows with due notices, unassessed risks past threshold, or expired retention dates. Returns empty array if no action items exist.
   - **Node 21 (`Email the person in charge`):** Add an **Email Send** node configured with SMTP credentials, mapping HTML body, subject, recipient, and sender from incoming JSON.

4. **Create the Register Export Sub-Flow:**
   - **Node 22 (`Export the register`):** Add a **Manual Trigger** node.
   - **Node 23 (`Create register table if missing (export)`):** Add a **Data Table** node ensuring table existence.
   - **Node 24 (`Get all incidents (export)`):** Add a **Data Table** node (Resource: `row`, Operation: `get`, `returnAll: true`).
   - **Node 25 (`Convert register to Excel`):** Add a **Convert to File** node (Operation: `xlsx`, Binary Property Name: `data`, Sheet Name: `Incidents`, Header Row: `true`, File Name: dynamic expression `=law25-incident-register-{{ $now.toISODate() }}.xlsx`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Québec Law 25 compliance requirements based on official consolidated texts on LegisQuebec (Act respecting the protection of personal information in the private sector, P-39.1, ss. 3.5 to 3.8; Regulation respecting confidentiality incidents, A-2.1, r. 3.1, ss. 3, 5, 7, 8). | Legal Reference Documentation |
| The Act sets no fixed number of days for notices, requiring them "promptly" (« avec diligence »). This workflow functions as a record-keeping and administrative tool, not formal legal advice. | Legal Context & Disclaimer |