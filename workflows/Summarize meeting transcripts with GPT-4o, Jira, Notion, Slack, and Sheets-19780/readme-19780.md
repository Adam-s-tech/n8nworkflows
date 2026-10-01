Summarize meeting transcripts with GPT-4o, Jira, Notion, Slack, and Sheets

https://n8nworkflows.xyz/workflows/summarize-meeting-transcripts-with-gpt-4o--jira--notion--slack--and-sheets-19780


# Summarize meeting transcripts with GPT-4o, Jira, Notion, Slack, and Sheets

### 1. Workflow Overview

This workflow automates meeting post-processing by ingesting transcripts from either a Fathom webhook or a Google Drive folder, extracting structured summaries using OpenAI, synchronizing action items to Jira and Google Sheets, archiving notes to Notion, and notifying stakeholders via Slack.

The architecture is divided into the following functional blocks:
- **1.1 Input Reception & File Ingestion:** Listens for incoming webhook payloads or polls Google Drive for new transcript files (TXT, PDF, DOCX).
- **1.2 Document Conversion & Normalization:** Converts DOCX files to Google Docs for text extraction, extracts raw text, and normalizes disparate payloads into a uniform meeting record schema.
- **1.3 AI Processing & Parsing:** Sends normalized text to OpenAI (GPT-4o) to extract summaries, decisions, action items, questions, and risks, then validates the JSON response.
- **1.4 Roster Enrichment & Date Resolution:** Fetches team roster data from Google Sheets, matches action item owners to system IDs, and resolves relative date phrases (e.g., "next Friday") into ISO dates.
- **1.5 Task Synchronization (Jira):** Splits action items, maps fields, creates corresponding Jira issues, and recombines tracking keys back into the master meeting payload.
- **1.6 Knowledge Base Publishing & Notifications (Notion & Slack):** Formats and publishes meeting records to a Notion database, posts recaps to Slack, and triggers alerts if owners cannot be matched.
- **1.7 Reminder Scheduler:** Appends 24-hour-before-due reminder rows to Google Sheets and executes an hourly background check to dispatch due reminders to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Ingestion
- **Overview:** Captures raw meeting data via a webhook endpoint or monitors a designated Google Drive directory for newly uploaded transcripts.
- **Nodes Involved:** `When Transcript Posted`, `When New Transcript File Added`, `Download Transcript from Google Drive`.
- **Node Details:**
  - **When Transcript Posted**
    - Type: `n8n-nodes-base.webhook`
    - Role: HTTP POST entry point for Fathom webhook payloads.
    - Config: Path `meeting-transcript`, HTTP method `POST`, response data set to "No Response Body".
    - Expressions: None.
    - Connections: Outputs to `Normalize Transcript Data`.
    - Edge cases: Fails if payload lacks `include_transcript=true` or network drops during transfer.
  - **When New Transcript File Added**
    - Type: `n8n-nodes-base.googleDriveTrigger`
    - Role: Polls Google Drive folder every minute for newly created files.
    - Config: Watches specific folder ID `1aZTOLwsRj1A6_acCoJLnAKxrvJR36mtT`, polls every 1 minute.
    - Connections: Outputs to `Download Transcript from Google Drive`.
    - Edge cases: API quota limits or permission revocation on the watched directory.
  - **Download Transcript from Google Drive**
    - Type: `n8n-nodes-base.googleDrive`
    - Role: Downloads binary content of the detected file.
    - Config: Operation `download`, file ID `={{$json.id}}`, converts Google Docs to `text/plain`.
    - Connections: Input from trigger; outputs to `Route by File Type`.
    - Edge cases: Large file timeouts or unsupported binary formats.

#### 2.2 Document Conversion & Normalization
- **Overview:** Inspects file extensions/MIME types, processes DOCX files via Google Docs temporary conversion, extracts text from PDFs or text files, and normalizes everything into a unified schema.
- **Nodes Involved:** `Route by File Type`, `Convert DOCX to Google Format`, `Fetch Converted Doc Text`, `Extract Text from Converted Doc`, `Remove Temp Doc from Drive`, `Extract Text from PDF`, `Extract Text from Transcript`, `Normalize Transcript Data`.
- **Node Details:**
  - **Route by File Type**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes file stream based on extension (`docx`, `pdf`, or default text).
    - Config: Evaluates MIME types and file extensions using case-insensitive checks.
    - Connections: Input from `Download Transcript from Google Drive`; outputs to conversion or extraction nodes.
  - **Convert DOCX to Google Format**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Copies a DOCX binary into Google Drive as a Google Doc for text export.
    - Config: POST to Google Drive API v3 files copy endpoint, sets MIME type to Google Docs.
    - Credentials: Google Drive OAuth2.
    - Connections: Input from `Route by File Type` (docx branch); outputs to `Fetch Converted Doc Text`.
  - **Fetch Converted Doc Text**
    - Type: `n8n-nodes-base.googleDrive`
    - Role: Downloads the temporary Google Doc as plain text.
    - Config: Operation `download`, exports to `text/plain`.
    - Connections: Input from `Convert DOCX to Google Format`; outputs to `Extract Text from Converted Doc`.
  - **Extract Text from Converted Doc**
    - Type: `n8n-nodes-base.extractFromFile`
    - Role: Extracts string content from the downloaded conversion stream.
    - Config: Operation `text`, assigns to `transcript_text`.
    - Connections: Output splits to `Remove Temp Doc from Drive` and `Normalize Transcript Data`.
  - **Remove Temp Doc from Drive**
    - Type: `n8n-nodes-base.googleDrive`
    - Role: Deletes the temporary conversion artifact from Google Drive.
    - Config: Operation `deleteFile`, targets file ID from `Convert DOCX to Google Format`. Error handling set to "continue regular output".
    - Connections: Input from `Extract Text from Converted Doc`.
  - **Extract Text from PDF**
    - Type: `n8n-nodes-base.extractFromFile`
    - Role: Extracts text from PDF files.
    - Config: Operation `pdf`, `joinPages: true`.
    - Connections: Input from `Route by File Type`; outputs to `Normalize Transcript Data`.
  - **Extract Text from Transcript**
    - Type: `n8n-nodes-base.extractFromFile`
    - Role: Extracts text from plain text files.
    - Config: Operation `text`, assigns to `transcript_text`.
    - Connections: Input from `Route by File Type`; outputs to `Normalize Transcript Data`.
  - **Normalize Transcript Data**
    - Type: `n8n-nodes-base.code`
    - Role: Consolidates webhook payloads or file extraction outputs into a unified data structure.
    - Config: Custom JavaScript parsing Fathom structures or file metadata.
    - Connections: Input from webhook or file extractors; outputs to `Extract Structured Notes`.
    - Edge cases: Throws explicit errors if transcript text is empty or contains binary garbage.

#### 2.3 AI Processing & Parsing
- **Overview:** Submits normalized transcripts to OpenAI GPT-4o to extract structured JSON data, then sanitizes and validates the output.
- **Nodes Involved:** `Extract Structured Notes`, `Parse Model Output`.
- **Node Details:**
  - **Extract Structured Notes**
    - Type: `n8n-nodes-base.openAi`
    - Role: Invokes OpenAI GPT-4o chat completion to extract summaries, decisions, action items, questions, and risks.
    - Config: Resource `chat`, model `gpt-4o`, temperature `0.2`, max tokens `2000`. System prompt enforces strict JSON output.
    - Credentials: OpenAI.
    - Expressions: `={{'Meeting title: ' + $json.meeting_title + '\nMeeting date: ' + $json.meeting_date + '\nAttendees: ' + JSON.stringify($json.attendees) + '\n\nTranscript:\n' + $json.transcript}}`
    - Connections: Input from `Normalize Transcript Data`; outputs to `Parse Model Output`.
    - Edge cases: Rate limits, token limits exceeded, or non-JSON model responses.
  - **Parse Model Output**
    - Type: `n8n-nodes-base.code`
    - Role: Cleans markdown formatting fences from LLM output, parses JSON, and guarantees default array structures.
    - Config: Custom JavaScript parsing response content.
    - Connections: Input from `Extract Structured Notes`; outputs to `Get Meeting Roster from Sheets`.
    - Edge cases: Throws syntax errors if model output fails JSON parsing.

#### 2.4 Roster Enrichment & Date Resolution
- **Overview:** Loads employee mappings from Google Sheets, resolves owners, and converts relative dates to standard ISO strings.
- **Nodes Involved:** `Get Meeting Roster from Sheets`, `Attach Roster to Record`, `Identify Action Owners`, `Compute Relative Dates`.
- **Node Details:**
  - **Get Meeting Roster from Sheets**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Fetches user directory rows from Google Sheets.
    - Config: Document ID and Sheet Name (`transcript of meeting`), operation `list`/`get`.
    - Credentials: Google Sheets OAuth2.
    - Connections: Input from `Parse Model Output`; outputs to `Attach Roster to Record`.
  - **Attach Roster to Record**
    - Type: `n8n-nodes-base.code`
    - Role: Merges the roster array into the active meeting record object.
    - Config: Custom JavaScript mapping sheet rows.
    - Connections: Input from Google Sheets node; outputs to `Identify Action Owners`.
  - **Identify Action Owners**
    - Type: `n8n-nodes-base.code`
    - Role: Matches action item owners against roster names and aliases, capturing unmatched users.
    - Config: Custom JavaScript fuzzy-matching names/aliases.
    - Connections: Input from `Attach Roster to Record`; outputs to `Compute Relative Dates`.
  - **Compute Relative Dates**
    - Type: `n8n-nodes-base.code`
    - Role: Resolves relative date expressions (e.g., "tomorrow", "next Friday", "in 3 days") relative to the meeting date.
    - Config: Custom JavaScript date math.
    - Connections: Input from `Identify Action Owners`; outputs to `Split Action Items`.

#### 2.5 Task Synchronization (Jira)
- **Overview:** Splits action items into individual items, formats Jira payloads, creates issues, and merges tracking keys back into the record.
- **Nodes Involved:** `Split Action Items`, `Build Jira Fields Data`, `Add Issue to Jira`, `Combine Jira and Action Data`, `Integrate Meeting Data`.
- **Node Details:**
  - **Split Action Items**
    - Type: `n8n-nodes-base.splitOut`
    - Role: Unrolls the `action_items` array into separate items for individual Jira processing.
    - Config: Field to split out `action_items`.
    - Connections: Input from `Compute Relative Dates`; outputs to `Build Jira Fields Data`.
  - **Build Jira Fields Data**
    - Type: `n8n-nodes-base.code`
    - Role: Maps action item properties to Jira summary, description, and priority fields.
    - Config: Custom JavaScript.
    - Connections: Input from `Split Action Items`; outputs to `Add Issue to Jira` and `Combine Jira and Action Data`.
  - **Add Issue to Jira**
    - Type: `n8n-nodes-base.jira`
    - Role: Creates a Jira issue for each action item.
    - Config: Project ID/Key, issue type ID (`10001`), maps summary and description expressions.
    - Credentials: Jira Software Cloud.
    - Expressions: Summary `={{$json.jira_summary}}`, Description `={{$json.jira_description}}`, Priority `={{$json.jira_priority_id}}`.
    - Connections: Input from `Build Jira Fields Data`; outputs to `Combine Jira and Action Data`.
  - **Combine Jira and Action Data**
    - Type: `n8n-nodes-base.merge`
    - Role: Merges Jira issue creation responses back with their corresponding action item data structures.
    - Config: Mode `combine`, combine by position, include unpaired.
    - Connections: Inputs from `Build Jira Fields Data` and `Add Issue to Jira`; outputs to `Integrate Meeting Data`.
  - **Integrate Meeting Data**
    - Type: `n8n-nodes-base.code`
    - Role: Reconstructs a unified meeting record containing the newly generated Jira issue keys.
    - Config: Custom JavaScript flattening items.
    - Connections: Input from `Combine Jira and Action Data`; outputs to `Construct Notion Data`.

#### 2.6 Knowledge Base Publishing & Notifications
- **Overview:** Builds Notion page structures, publishes documentation, sends Slack recaps, and alerts channels if owners are unmatched.
- **Nodes Involved:** `Construct Notion Data`, `Post to Notion API`, `Combine Notion with Record`, `Post Recap to Slack`, `Check for Unmatched Owners`, `Notify Unmatched Owners in Slack`.
- **Node Details:**
  - **Construct Notion Data**
    - Type: `n8n-nodes-base.code`
    - Role: Constructs the JSON block payload required by the Notion API.
    - Config: Custom JavaScript formatting headings, bullet lists, and transcript toggles.
    - Connections: Input from `Integrate Meeting Data`; outputs to `Post to Notion API` and `Combine Notion with Record`.
  - **Post to Notion API**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Calls the Notion API to create a new page in the target database.
    - Config: POST request to `https://api.notion.com/v1/pages`, headers set for Notion version `2022-06-28`. Error handling set to "continue regular output".
    - Credentials: Notion API.
    - Expressions: JSON Body `={{ JSON.stringify($json.notion_payload) }}`.
    - Connections: Input from `Construct Notion Data`; outputs to `Combine Notion with Record`.
  - **Combine Notion with Record**
    - Type: `n8n-nodes-base.merge`
    - Role: Merges the Notion API response (including the page URL) with the master meeting record.
    - Config: Mode `combine`, combine by position.
    - Connections: Inputs from `Construct Notion Data` and `Post to Notion API`; outputs to `Post Recap to Slack`, `Check for Unmatched Owners`, and `Check If Reminders Enabled`.
  - **Post Recap to Slack**
    - Type: `n8n-nodes-base.slack`
    - Role: Posts the formatted meeting summary and Notion link to Slack.
    - Config: Select `channel`, channel ID `C0C0UEMPD5E`.
    - Credentials: Slack.
    - Expressions: Formatted message template containing summary bullets, action items, user mentions, and Notion URL.
    - Connections: Input from `Combine Notion with Record`.
  - **Check for Unmatched Owners**
    - Type: `n8n-nodes-base.if`
    - Role: Checks whether any action item owners could not be matched against the roster.
    - Config: Condition `{{$json.unmatchedOwners.length}} > 0`.
    - Connections: Input from `Combine Notion with Record`; outputs to `Notify Unmatched Owners in Slack`.
  - **Notify Unmatched Owners in Slack**
    - Type: `n8n-nodes-base.slack`
    - Role: Sends a warning alert to Slack detailing unmatched action item owners.
    - Config: Select `channel`, channel ID `C0C0UEMPD5E`.
    - Credentials: Slack.
    - Expressions: Warning text mentioning unmatched names and fallback organizer assignment.
    - Connections: Input from `Check for Unmatched Owners`.

#### 2.7 Reminder Scheduler
- **Overview:** Appends 24-hour reminder rows to Google Sheets when enabled, and executes an hourly background check to send overdue reminders to Slack.
- **Nodes Involved:** `Check If Reminders Enabled`, `Generate Reminder Rows`, `Add Reminders to Sheets`, `Hourly Reminder Check`, `Fetch Reminders from Sheets`, `Filter Due Reminders`, `Post Reminder to Slack`, `Mark Reminders as Sent`.
- **Node Details:**
  - **Check If Reminders Enabled**
    - Type: `n8n-nodes-base.if`
    - Role: Verifies if the meeting configuration has reminders enabled.
    - Config: Condition `{{$json.reminders_enabled}} === true`.
    - Connections: Input from `Combine Notion with Record`; outputs to `Generate Reminder Rows`.
  - **Generate Reminder Rows**
    - Type: `n8n-nodes-base.code`
    - Role: Computes the 24-hour lead-time timestamp for each action item.
    - Config: Custom JavaScript subtracting 24 hours from due dates.
    - Connections: Input from `Check If Reminders Enabled`; outputs to `Add Reminders to Sheets`.
  - **Add Reminders to Sheets**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Appends reminder tracking rows to the Google Sheets Reminders tab.
    - Config: Operation `append`, sheet name `Reminders`, mapping mode `autoMapInputData`.
    - Credentials: Google Sheets OAuth2.
    - Connections: Input from `Generate Reminder Rows`.
  - **Hourly Reminder Check**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers execution every hour to process pending reminders.
    - Config: Interval rule set to `hours`.
    - Connections: Outputs to `Fetch Reminders from Sheets`.
  - **Fetch Reminders from Sheets**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Reads all rows from the Google Sheets Reminders tab.
    - Config: Operation `list`/`get`, sheet name `Reminders`.
    - Credentials: Google Sheets OAuth2.
    - Connections: Input from `Hourly Reminder Check`; outputs to `Filter Due Reminders`.
  - **Filter Due Reminders**
    - Type: `n8n-nodes-base.code`
    - Role: Filters reminder records that are due and have not yet been marked as sent.
    - Config: Custom JavaScript parsing ISO/serial dates and comparing against current time.
    - Connections: Input from `Fetch Reminders from Sheets`; outputs to `Post Reminder to Slack`.
  - **Post Reminder to Slack**
    - Type: `n8n-nodes-base.slack`
    - Role: Sends due reminder notifications to the designated Slack channel or user.
    - Config: Select `channel`, channel ID `C0C0UEMPD5E`.
    - Credentials: Slack.
    - Expressions: Formatted reminder text with task title, Jira key, and user mention.
    - Connections: Input from `Filter Due Reminders`; outputs to `Mark Reminders as Sent`.
  - **Mark Reminders as Sent**
    - Type: `n8n-nodes-base.googleSheets`
    - Role: Updates the Google Sheets reminder row to mark `sent` as true.
    - Config: Operation `update`, matching columns `row_number`.
    - Credentials: Google Sheets OAuth2.
    - Expressions: Sets `sent` to `={{ true }}` for row `={{ $('Filter Due Reminders').item.json.row_number }}`.
    - Connections: Input from `Post Reminder to Slack`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Transcript Posted | webhook | Webhook entry point for Fathom transcripts | None | Normalize Transcript Data | Meeting Recap - Reminder Scheduler (24h before due) |
| When New Transcript File Added | googleDriveTrigger | Polls Google Drive folder for new files | None | Download Transcript from Google Drive | Drive transcript intake |
| Download Transcript from Google Drive | googleDrive | Downloads binary file from Drive | When New Transcript File Added | Route by File Type | Drive transcript intake |
| Route by File Type | switch | Routes file stream by extension/MIME type | Download Transcript from Google Drive | Convert DOCX to Google Format, Extract Text from PDF, Extract Text from Transcript | Drive transcript intake |
| Convert DOCX to Google Format | httpRequest | Copies DOCX to Google Docs for conversion | Route by File Type | Fetch Converted Doc Text | Convert DOCX transcripts |
| Fetch Converted Doc Text | googleDrive | Downloads converted Google Doc as text | Convert DOCX to Google Format | Extract Text from Converted Doc | Convert DOCX transcripts |
| Extract Text from Converted Doc | extractFromFile | Extracts plain text from converted document stream | Fetch Converted Doc Text | Remove Temp Doc from Drive, Normalize Transcript Data | Convert DOCX transcripts |
| Remove Temp Doc from Drive | googleDrive | Cleans up temporary Google Doc file | Extract Text from Converted Doc | None | Convert DOCX transcripts |
| Extract Text from PDF | extractFromFile | Extracts text from PDF files | Route by File Type | Normalize Transcript Data | Normalize transcript input |
| Extract Text from Transcript | extractFromFile | Extracts text from plain text files | Route by File Type | Normalize Transcript Data | Normalize transcript input |
| Normalize Transcript Data | code | Normalizes disparate payloads into standard schema | When Transcript Posted, Extract Text from PDF, Extract Text from Converted Doc, Extract Text from Transcript | Extract Structured Notes | Normalize transcript input |
| Extract Structured Notes | openAi | Invokes GPT-4o to extract meeting structure | Normalize Transcript Data | Parse Model Output | Extract structured notes |
| Parse Model Output | code | Parses and validates LLM JSON response | Extract Structured Notes | Get Meeting Roster from Sheets | Extract structured notes |
| Get Meeting Roster from Sheets | googleSheets | Fetches user roster from Google Sheets | Parse Model Output | Attach Roster to Record | Resolve action details |
| Attach Roster to Record | code | Attaches roster data to meeting record | Get Meeting Roster from Sheets | Identify Action Owners | Resolve action details |
| Identify Action Owners | code | Matches action owners to roster IDs | Attach Roster to Record | Compute Relative Dates | Resolve action details |
| Compute Relative Dates | code | Resolves relative due date phrases to ISO dates | Identify Action Owners | Split Action Items | Resolve action details |
| Split Action Items | splitOut | Splits action items array into individual items | Compute Relative Dates | Build Jira Fields Data | Create Jira issues |
| Build Jira Fields Data | code | Prepares Jira field mapping payload | Split Action Items | Add Issue to Jira, Combine Jira and Action Data | Create Jira issues |
| Add Issue to Jira | jira | Creates Jira issue for each action item | Build Jira Fields Data | Combine Jira and Action Data | Create Jira issues |
| Combine Jira and Action Data | merge | Combines Jira responses with action item data | Build Jira Fields Data, Add Issue to Jira | Integrate Meeting Data | Create Jira issues |
| Integrate Meeting Data | code | Rebuilds unified meeting record with Jira keys | Combine Jira and Action Data | Construct Notion Data | Create Jira issues |
| Construct Notion Data | code | Builds Notion page block payload | Integrate Meeting Data | Post to Notion API, Combine Notion with Record | Create Notion recap |
| Post to Notion API | httpRequest | Publishes meeting record page to Notion | Construct Notion Data | Combine Notion with Record | Create Notion recap |
| Combine Notion with Record | merge | Merges Notion URL with master meeting record | Construct Notion Data, Post to Notion API | Post Recap to Slack, Check for Unmatched Owners, Check If Reminders Enabled | Create Notion recap |
| Post Recap to Slack | slack | Posts meeting recap message to Slack channel | Combine Notion with Record | None | Post Slack recap |
| Check for Unmatched Owners | if | Checks if any action owners are unmatched | Combine Notion with Record | Notify Unmatched Owners in Slack | Alert unmatched owners |
| Notify Unmatched Owners in Slack | slack | Alerts Slack channel of unmatched owners | Check for Unmatched Owners | None | Alert unmatched owners |
| Check If Reminders Enabled | if | Checks if reminder generation is enabled | Combine Notion with Record | Generate Reminder Rows | Store reminder rows |
| Generate Reminder Rows | code | Computes 24-hour pre-due reminder schedule | Check If Reminders Enabled | Add Reminders to Sheets | Store reminder rows |
| Add Reminders to Sheets | googleSheets | Appends reminder records to Google Sheets | Generate Reminder Rows | None | Store reminder rows |
| Hourly Reminder Check | scheduleTrigger | Triggers hourly execution for reminders | None | Fetch Reminders from Sheets | Read scheduled reminders |
| Fetch Reminders from Sheets | googleSheets | Reads reminder records from Google Sheets | Hourly Reminder Check | Filter Due Reminders | Read scheduled reminders |
| Filter Due Reminders | code | Filters due and unsent reminders | Fetch Reminders from Sheets | Post Reminder to Slack | Send due reminders |
| Post Reminder to Slack | slack | Posts due reminder notification to Slack | Filter Due Reminders | Mark Reminders as Sent | Send due reminders |
| Mark Reminders as Sent | googleSheets | Updates Google Sheets reminder row as sent | Post Reminder to Slack | None | Send due reminders |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the entire workflow manually in n8n:

1. **Setup Entry Triggers:**
   - Create a **Webhook** node named `When Transcript Posted` (Path: `meeting-transcript`, Method: `POST`).
   - Create a **Google Drive Trigger** node named `When New Transcript File Added` (Event: `fileCreated`, specific folder ID: `1aZTOLwsRj1A6_acCoJLnAKxrvJR36mtT`, poll every 1 minute).
   - Create a **Schedule Trigger** node named `Hourly Reminder Check` (Interval: `hours`).

2. **Build File Intake & Normalization Path:**
   - Connect `When New Transcript File Added` to a **Google Drive** node named `Download Transcript from Google Drive` (Operation: `download`, file ID: `={{$json.id}}`, export Google Docs to `text/plain`).
   - Connect `Download Transcript from Google Drive` to a **Switch** node named `Route by File Type` (Rules matching `.docx` / `wordprocessingml.document`, `.pdf` / `application/pdf`, and fallback text).
   - **DOCX Branch:** Connect switch to an **HTTP Request** node named `Convert DOCX to Google Format` (POST copy endpoint, Google Drive OAuth2 credential), then to a **Google Drive** node named `Fetch Converted Doc Text` (download export `text/plain`), then to an **Extract from File** node named `Extract Text from Converted Doc` (operation: `text`), and finally hook a cleanup **Google Drive** node named `Remove Temp Doc from Drive` (operation: `deleteFile`).
   - **PDF Branch:** Connect switch to an **Extract from File** node named `Extract Text from PDF` (operation: `pdf`).
   - **TXT Branch:** Connect switch to an **Extract from File** node named `Extract Text from Transcript` (operation: `text`).
   - Connect all extraction paths and `When Transcript Posted` to a **Code** node named `Normalize Transcript Data` to output a unified record structure.

3. **Configure AI Extraction:**
   - Connect `Normalize Transcript Data` to an **OpenAI** node named `Extract Structured Notes` (Resource: `chat`, model: `gpt-4o`, temperature `0.2`, max tokens `2000`, OpenAI credential). Configure prompt to request strict JSON for summary, decisions, action items, open questions, and risks.
   - Connect to a **Code** node named `Parse Model Output` to strip markdown fences and parse the JSON object.

4. **Setup Roster Enrichment & Date Resolution:**
   - Connect `Parse Model Output` to a **Google Sheets** node named `Get Meeting Roster from Sheets` (Google Sheets OAuth2 credential, document ID and sheet name).
   - Connect to a **Code** node named `Attach Roster to Record` to merge the roster.
   - Connect to a **Code** node named `Identify Action Owners` to match owners and log unmatched names.
   - Connect to a **Code** node named `Compute Relative Dates` to convert relative dates into ISO strings.

5. **Setup Jira Integration:**
   - Connect `Compute Relative Dates` to a **Split Out** node named `Split Action Items` (field: `action_items`).
   - Connect to a **Code** node named `Build Jira Fields Data` to prepare summaries and descriptions.
   - Connect to a **Jira** node named `Add Issue / Bug` (Jira Software Cloud credential, project key/ID, issue type `10001`).
   - Connect both `Build Jira Fields Data` and `Add Issue to Jira` to a **Merge** node named `Combine Jira and Action Data` (mode: `combine`, combine by position).
   - Connect to a **Code** node named `Integrate Meeting Data` to consolidate the record.

6. **Setup Notion & Slack Recaps:**
   - Connect `Integrate Meeting Data` to a **Code** node named `Construct Notion Data` to build the Notion block payload.
   - Connect to an **HTTP Request** node named `Post to Notion API` (POST `https://api.notion.com/v1/pages`, Notion API credential, version header `2022-06-28`).
   - Connect both `Construct Notion Data` and `Post to Notion API` to a **Merge** node named `Combine Notion with Record`.
   - Connect the merged output to:
     - A **Slack** node named `Post Recap to Slack` (Slack credential, channel ID `C0C0UEMPD5E`).
     - An **If** node named `Check for Unmatched Owners` (`{{$json.unmatchedOwners.length}} > 0`), connected to a **Slack** node named `Notify Unmatched Owners in Slack`.

7. **Setup Reminder Scheduler:**
     - From `Combine Notion with Record`, branch to an **If** node named `Check If Reminders Enabled` (`{{$json.reminders_enabled}} === true`).
     - Connect to a **Code** node named `Generate Reminder Rows` to compute 24-hour lead times.
     - Connect to a **Google Sheets** node named `Add Reminders to Sheets` (operation: `append`, sheet name `Reminders`).
     - From `Hourly Reminder Check`, connect to a **Google Sheets** node named `Fetch Reminders from Sheets` (operation: `list`, sheet name `Reminders`).
     - Connect to a **Code** node named `Filter Due Reminders` to evaluate due dates.
     - Connect to a **Slack** node named `Post Reminder to Slack` (channel ID `C0C0UEMPD5E`).
     - Connect to a **Google Sheets** node named `Mark Reminders as Sent` (operation: `update`, matching column `row_number`, sets `sent` to `true`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Website | https://www.intuz.com/n8n-workflow-automation-templates/ |
| Support Email | getstarted@intuz.com |
| LinkedIn Company Page | https://www.linkedin.com/company/intuz |
| Partner Links / Getting Started | https://n8n.partnerlinks.io/intuz |
| Custom Workflow Automation Inquiry | https://www.intuz.com/get-started/ |