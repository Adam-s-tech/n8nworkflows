Turn Zoom coaching sessions into recap emails, homework, and nudges with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/turn-zoom-coaching-sessions-into-recap-emails--homework--and-nudges-with-openai-and-google-sheets-20088


# Turn Zoom coaching sessions into recap emails, homework, and nudges with OpenAI and Google Sheets

### 1. Workflow Overview

This workflow automates the post-coaching administrative process by converting completed Zoom coaching sessions into structured AI-generated action plans, logging sessions and homework assignments to Google Sheets, sending recap emails to clients via Gmail, and running a daily check-in routine to send motivational nudge emails for pending tasks.

The logic is partitioned into six functional blocks:
- **1.1 Input Reception & Zoom Verification:** Receives Zoom webhooks, handles security endpoint validation handshakes, acknowledges recording events immediately to prevent retries, and extracts core metadata.
- **1.2 Client Identification & Duplicate Prevention:** Queries Google Sheets for client matching (via Meeting ID or Topic) and filters out inactive clients or already processed sessions.
- **1.3 Transcript Acquisition:** Retrieves native Zoom transcripts when available or falls back to downloading session audio and transcribing it via OpenAI Whisper.
- **1.4 AI Action Plan Generation & Delivery:** Sends transcript text to OpenAI GPT-4o-mini to extract summaries, wins, insights, and structured homework items, generates an HTML email representation, dispatches it to the client, logs the session, and populates the Homework Tracker.
- **1.5 Nudge Rule Initialization & Selection:** Executes daily at 10:00 AM, sets configuration limits, parses the Homework Tracker, and identifies clients with tasks due soon or slightly overdue.
- **1.6 Nudge Dispatch & Log Update:** Uses OpenAI GPT-4o-mini to draft personalized check-in messages, sends the emails via Gmail, and increments nudge counters and timestamps in the tracking sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Zoom Verification
- **Overview:** Listens for incoming Zoom webhook events, responds to the endpoint verification challenge using HMAC SHA256 signing, immediately acknowledges recording events, and drops unsupported events.
- **Nodes Involved:** 
  - `Zoom Recording Webhook`
  - `Route Zoom Event`
  - `Sign Validation Token`
  - `Return Validation Response`
  - `Acknowledge Recording`
  - `Ignore Other Events`
- **Node Details:**
  - **Zoom Recording Webhook**
    - Type and role: Webhook trigger. Listens for POST requests at `/zoom-coaching-recording`.
    - Key expressions: None.
    - Input/Output: Inputs none; outputs to `Route Zoom Event`.
    - Edge cases: Webhook URL must be publicly accessible and correctly configured in Zoom Marketplace.
  - **Route Zoom Event**
    - Type and role: Switch node. Evaluates incoming event types (`endpoint.url_validation`, `recording.completed`, `recording.transcript_completed`).
    - Key expressions: `={{ $json.body.event }}`
    - Input/Output: Input from `Zoom Recording Webhook`; outputs to validation, recording, or fallback branches.
  - **Sign Validation Token**
    - Type and role: Crypto node. Generates an HMAC SHA256 hash using Zoom’s plain token.
    - Key expressions: Value = `={{ $json.body.payload.plainToken }}`
    - Input/Output: Input from `Route Zoom Event`; outputs to `Return Validation Response`.
    - Edge cases: Requires valid Zoom secret token configured in credentials/secret parameter.
  - **Return Validation Response**
    - Type and role: Respond to Webhook node. Returns JSON response containing plain and encrypted tokens.
    - Key expressions: `={{ JSON.stringify({ plainToken: $json.body.payload.plainToken, encryptedToken: $json.encryptedToken }) }}`
    - Input/Output: Input from `Sign Validation Token`; terminates execution branch.
  - **Acknowledge Recording**
    - Type and role: Respond to Webhook node. Responds with plain text `ok` to prevent Zoom webhook retries.
    - Input/Output: Input from `Route Zoom Event`; outputs to `Extract Recording Details`.
  - **Ignore Other Events**
    - Type and role: Respond to Webhook node. Terminates execution with no data for unhandled event types.
    - Input/Output: Input from `Route Zoom Event`; terminates execution branch.

#### 2.2 Client Identification & Duplicate Prevention
- **Overview:** Extracts recording file metadata, queries the Google Sheets "Clients" and "Session Log" tabs to match the meeting with an active client profile, and skips duplicate processing.
- **Nodes Involved:**
  - `Extract Recording Details`
  - `Read Clients`
  - `Read Session Log`
  - `Match Client to Meeting`
- **Node Details:**
  - **Extract Recording Details**
    - Type and role: Code node (JavaScript). Parses Zoom payload, identifies transcript and audio download links.
    - Key expressions: References `$('Zoom Recording Webhook')`.
    - Input/Output: Input from `Acknowledge Recording`; outputs to `Read Clients`.
  - **Read Clients**
    - Type and role: Google Sheets node. Retrieves all rows from the `Clients` sheet tab.
    - Input/Output: Input from `Extract Recording Details`; outputs to `Read Session Log`.
  - **Read Session Log**
    - Type and role: Google Sheets node. Retrieves all rows from the `Session Log` tab to check for duplicates.
    - Input/Output: Input from `Read Clients`; outputs to `Match Client to Meeting`.
  - **Match Client to Meeting**
    - Type and role: Code node (JavaScript). Matches meeting ID or topic name to client records and checks if session UUID exists in the session log. Filters out inactive clients.
    - Key expressions: References `$('Extract Recording Details')` and `$('Read Clients')`.
    - Input/Output: Input from `Read Session Log`; outputs to `Has Zoom Transcript?`.

#### 2.3 Transcript Acquisition
- **Overview:** Checks whether a native Zoom transcript is available; if so, downloads and cleans it. Otherwise, downloads the session audio file and transcribes it using OpenAI Whisper.
- **Nodes Involved:**
  - `Has Zoom Transcript?`
  - `Download Zoom Transcript`
  - `Clean Transcript Text`
  - `Download Session Audio`
  - `Transcribe Audio with Whisper`
  - `Use Whisper Transcript`
- **Node Details:**
  - **Has Zoom Transcript?**
    - Type and role: IF condition node. Evaluates `hasTranscript` boolean.
    - Key expressions: `={{ $json.hasTranscript }}`
    - Input/Output: Input from `Match Client to Meeting`; outputs true branch to `Download Zoom Transcript`, false branch to `Download Session Audio`.
  - **Download Zoom Transcript**
    - Type and role: HTTP Request node. Downloads WebVTT transcript file using access token query parameter.
    - Key expressions: URL = `={{ $json.transcriptUrl }}`, Query Param `access_token` = `={{ $json.downloadToken }}`
    - Input/Output: Input from `Has Zoom Transcript?`; outputs to `Clean Transcript Text`.
  - **Clean Transcript Text**
    - Type and role: Code node (JavaScript). Strips WebVTT metadata, timestamps, and duplicate sequential lines.
    - Input/Output: Input from `Download Zoom Transcript`; outputs to `Generate Action Plan`.
  - **Download Session Audio**
    - Type and role: HTTP Request node. Downloads M4A or MP4 audio file.
    - Key expressions: URL = `={{ $json.audioUrl }}`, Query Param `access_token` = `={{ $json.downloadToken }}`
    - Input/Output: Input from `Has Zoom Transcript?`; outputs to `Transcribe Audio with Whisper`.
  - **Transcribe Audio with Whisper**
    - Type and role: LangChain OpenAI node. Transcribes audio file binary data.
    - Input/Output: Input from `Download Session Audio`; outputs to `Use Whisper Transcript`.
    - Edge cases: OpenAI Whisper enforces a 25 MB file size limit (approx. 45 minutes of M4A audio).
  - **Use Whisper Transcript**
    - Type and role: Set node. Normalizes Whisper transcription output into the `transcript` property.
    - Key expressions: `={{ $json.text }}`
    - Input/Output: Input from `Transcribe Audio with Whisper`; outputs to `Generate Action Plan`.

#### 2.4 Action Plan Generation & Delivery
- **Overview:** Sends the conversation transcript to OpenAI GPT-4o-mini to generate structured JSON containing summaries, wins, insights, homework items, and scores. Formats an HTML email, delivers it via Gmail, logs the session, and creates individual homework tracking rows.
- **Nodes Involved:**
  - `Generate Action Plan`
  - `Build Recap & Homework Rows`
  - `Email Recap to Client`
  - `Log Session`
  - `Prepare Homework Rows`
  - `Add Homework to Tracker`
- **Node Details:**
  - **Generate Action Plan**
    - Type and role: LangChain OpenAI node (GPT-4o-mini). Processes transcript with structured JSON output configuration.
    - Key expressions: Injects client name, goals, session date, topic, and transcript variables into system/user prompts.
    - Input/Output: Input from `Clean Transcript Text` or `Use Whisper Transcript`; outputs to `Build Recap & Homework Rows`.
  - **Build Recap & Homework Rows**
    - Type and role: Code node (JavaScript). Parses JSON output from GPT, calculates homework due dates relative to session date, and compiles a styled HTML email template.
    - Input/Output: Input from `Generate Action Plan`; outputs to `Email Recap to Client`.
  - **Email Recap to Client**
    - Type and role: Gmail node. Sends HTML recap email to the client and CCs the coach.
    - Key expressions: Recipient = `={{ $('Match Client to Meeting').item.json.clientEmail }}`, Subject = `={{ $json.subject }}`, CC = `={{ $('Match Client to Meeting').item.json.coachEmail }}`
    - Input/Output: Input from `Build Recap & Homework Rows`; outputs to `Log Session`.
    - Edge cases: Requires valid Gmail OAuth2 credentials.
  - **Log Session**
    - Type and role: Google Sheets node. Appends session metadata and summary to the `Session Log` tab.
    - Input/Output: Input from `Email Recap to Client`; outputs to `Prepare Homework Rows`.
  - **Prepare Homework Rows**
    - Type and role: Code node (JavaScript). Unpacks array of homework tasks into individual records for sheet appending.
    - Input/Output: Input from `Log Session`; outputs to `Add Homework to Tracker`.
  - **Add Homework to Tracker**
    - Type and role: Google Sheets node. Appends rows to the `Homework Tracker` tab.
    - Input/Output: Input from `Prepare Homework Rows`; terminates primary branch.

#### 2.5 Nudge Rule Initialization & Selection
- **Overview:** Triggers daily at 10:00 AM, defines operational parameters for nudge frequency and windows, reads tracking data, and identifies clients requiring check-in emails.
- **Nodes Involved:**
  - `Every Day at 10 AM`
  - `Set Nudge Rules`
  - `Read Homework Tracker`
  - `Pick Clients to Nudge`
- **Node Details:**
  - **Every Day at 10 AM**
    - Type and role: Schedule Trigger node. Fires daily at cron `0 10 * * *`.
    - Input/Output: Outputs to `Set Nudge Rules`.
  - **Set Nudge Rules**
    - Type and role: Set node. Sets configuration constants (`coachName`, `coachReplyTo`, `nudgeWindowDays`, `stopAfterDaysOverdue`, `minDaysBetweenNudges`, `maxNudgesPerTask`).
    - Input/Output: Input from `Every Day at 10 AM`; outputs to `Read Homework Tracker`.
  - **Read Homework Tracker**
    - Type and role: Google Sheets node. Retrieves all records from the `Homework Tracker` tab.
    - Input/Output: Input from `Set Nudge Rules`; outputs to `Pick Clients to Nudge`.
  - **Pick Clients to Nudge**
    - Type and role: Code node (JavaScript). Filters open tasks within date criteria, enforces maximum nudge limits and minimum spacing, and groups tasks by client email.
    - Key expressions: References `$('Set Nudge Rules')`.
    - Input/Output: Input from `Read Homework Tracker`; outputs to `Draft Nudge Message`.

#### 2.6 Nudge Dispatch & Log Update
- **Overview:** Drafts individualized check-in emails using OpenAI GPT-4o-mini, sends them via Gmail, and updates the task tracker rows with new nudge counts and timestamps.
- **Nodes Involved:**
  - `Draft Nudge Message`
  - `Send Nudge Email`
  - `Expand Nudged Tasks`
  - `Update Nudge Log`
- **Node Details:**
  - **Draft Nudge Message**
    - Type and role: LangChain OpenAI node (GPT-4o-mini). Generates short, encouraging text-based check-in emails.
    - Key expressions: References client name, coach name, task completion stats, and task list.
    - Input/Output: Input from `Pick Clients to Nudge`; outputs to `Send Nudge Email`.
  - **Send Nudge Email**
    - Type and role: Gmail node. Sends plain-text nudge email to client with reply-to configured.
    - Key expressions: Recipient = `={{ $('Pick Clients to Nudge').item.json.clientEmail }}`, Subject = `={{ $json.message.content.subject }}`, Message = `={{ $json.message.content.body }}`
    - Input/Output: Input from `Draft Nudge Message`; outputs to `Expand Nudged Tasks`.
  - **Expand Nudged Tasks**
    - Type and role: Code node (JavaScript). Expands client-level email payload back into individual task updates.
    - Input/Output: Input from `Send Nudge Email`; outputs to `Update Nudge Log`.
  - **Update Nudge Log**
    - Type and role: Google Sheets node. Updates existing rows in the `Homework Tracker` tab matched by `Task ID`.
    - Input/Output: Input from `Expand Nudged Tasks`; terminates execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Workflow documentation and configuration overview. | None | None | Coaching Session to Action Plan... |
| Section - Zoom Intake & Verification | n8n-nodes-base.stickyNote | Visual grouping for intake and verification logic. | None | None | Zoom Intake & Verification... |
| Section - Identify Client | n8n-nodes-base.stickyNote | Visual grouping for client identification and deduplication. | None | None | Identify Client... |
| Section - Get the Transcript | n8n-nodes-base.stickyNote | Visual grouping for transcript download and transcription. | None | None | Get the Transcript... |
| Section - Action Plan & Delivery | n8n-nodes-base.stickyNote | Visual grouping for AI generation, emails, and logging. | None | None | Action Plan & Delivery... |
| Section - Pick Who to Nudge | n8n-nodes-base.stickyNote | Visual grouping for daily schedule and task filtering. | None | None | Pick Who to Nudge... |
| Section - Send & Log Nudges | n8n-nodes-base.stickyNote | Visual grouping for nudge generation, delivery, and sheet updates. | None | None | Send & Log Nudges... |
| Credentials & Security | n8n-nodes-base.stickyNote | Security and credential setup guidance. | None | None | Credentials & Security... |
| Zoom Recording Webhook | n8n-nodes-base.webhook | Receives incoming Zoom recording webhooks. | None | Route Zoom Event | Coaching Session to Action Plan... |
| Route Zoom Event | n8n-nodes-base.switch | Routes event based on event type. | Zoom Recording Webhook | Sign Validation Token, Acknowledge Recording, Ignore Other Events | Coaching Session to Action Plan... |
| Sign Validation Token | n8n-nodes-base.crypto | Generates HMAC SHA256 signature for URL validation. | Route Zoom Event | Return Validation Response | Coaching Session to Action Plan... |
| Return Validation Response | n8n-nodes-base.respondToWebhook | Returns Zoom endpoint validation payload. | Sign Validation Token | None | Coaching Session to Action Plan... |
| Acknowledge Recording | n8n-nodes-base.respondToWebhook | Acknowledges recording event with "ok". | Route Zoom Event | Extract Recording Details | Coaching Session to Action Plan... |
| Ignore Other Events | n8n-nodes-base.respondToWebhook | Terminates unhandled webhook events. | Route Zoom Event | None | Coaching Session to Action Plan... |
| Extract Recording Details | n8n-nodes-base.code | Extracts metadata and media URLs from Zoom payload. | Acknowledge Recording | Read Clients | Coaching Session to Action Plan... |
| Read Clients | n8n-nodes-base.googleSheets | Retrieves client definitions from sheet. | Extract Recording Details | Read Session Log | Identify Client... |
| Read Session Log | n8n-nodes-base.googleSheets | Retrieves existing session logs to prevent duplicates. | Read Clients | Match Client to Meeting | Identify Client... |
| Match Client to Meeting | n8n-nodes-base.code | Matches meeting to client and verifies active status. | Read Session Log | Has Zoom Transcript? | Identify Client... |
| Has Zoom Transcript? | n8n-nodes-base.if | Checks if native transcript file exists. | Match Client to Meeting | Download Zoom Transcript, Download Session Audio | Get the Transcript... |
| Download Zoom Transcript | n8n-nodes-base.httpRequest | Downloads WebVTT transcript file. | Has Zoom Transcript? | Clean Transcript Text | Get the Transcript... |
| Clean Transcript Text | n8n-nodes-base.code | Parses WebVTT into clean plain text lines. | Download Zoom Transcript | Generate Action Plan | Get the Transcript... |
| Download Session Audio | n8n-nodes-base.httpRequest | Downloads session audio file. | Has Zoom Transcript? | Transcribe Audio with Whisper | Get the Transcript... |
| Transcribe Audio with Whisper | @n8n/n8n-nodes-langchain.openAi | Transcribes audio file using Whisper. | Download Session Audio | Use Whisper Transcript | Get the Transcript... |
| Use Whisper Transcript | n8n-nodes-base.set | Formats Whisper transcription output. | Transcribe Audio with Whisper | Generate Action Plan | Get the Transcript... |
| Generate Action Plan | @n8n/n8n-nodes-langchain.openAi | Generates structured JSON action plan via GPT-4o-mini. | Clean Transcript Text, Use Whisper Transcript | Build Recap & Homework Rows | Action Plan & Delivery... |
| Build Recap & Homework Rows | n8n-nodes-base.code | Builds HTML email and structured homework items. | Generate Action Plan | Email Recap to Client | Action Plan & Delivery... |
| Email Recap to Client | n8n-nodes-base.gmail | Sends HTML recap email to client and CCs coach. | Build Recap & Homework Rows | Log Session | Action Plan & Delivery... |
| Log Session | n8n-nodes-base.googleSheets | Appends session summary to Session Log. | Email Recap to Client | Prepare Homework Rows | Action Plan & Delivery... |
| Prepare Homework Rows | n8n-nodes-base.code | Maps homework array into individual rows. | Log Session | Add Homework to Tracker | Action Plan & Delivery... |
| Add Homework to Tracker | n8n-nodes-base.googleSheets | Appends homework items to Homework Tracker. | Prepare Homework Rows | None | Action Plan & Delivery... |
| Every Day at 10 AM | n8n-nodes-base.scheduleTrigger | Triggers daily at 10:00 AM. | None | Set Nudge Rules | Pick Who to Nudge... |
| Set Nudge Rules | n8n-nodes-base.set | Defines global configuration rules for nudges. | Every Day at 10 AM | Read Homework Tracker | Pick Who to Nudge... |
| Read Homework Tracker | n8n-nodes-base.googleSheets | Reads tracking rows from Homework Tracker tab. | Set Nudge Rules | Pick Clients to Nudge | Pick Who to Nudge... |
| Pick Clients to Nudge | n8n-nodes-base.code | Filters pending tasks and groups them by client. | Read Homework Tracker | Draft Nudge Message | Pick Who to Nudge... |
| Draft Nudge Message | @n8n/n8n-nodes-langchain.openAi | Drafts friendly check-in email via GPT-4o-mini. | Pick Clients to Nudge | Send Nudge Email | Send & Log Nudges... |
| Send Nudge Email | n8n-nodes-base.gmail | Sends nudge email to client. | Draft Nudge Message | Expand Nudged Tasks | Send & Log Nudges... |
| Expand Nudged Tasks | n8n-nodes-base.code | Expands grouped client emails back into task updates. | Send Nudge Email | Update Nudge Log | Send & Log Nudges... |
| Update Nudge Log | n8n-nodes-base.googleSheets | Updates task rows with incremented nudge counts. | Expand Nudged Tasks | None | Send & Log Nudges... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet:** Create a new Google Sheet containing three tabs:
   - `Clients`: Columns: `Client ID`, `Client Name`, `Client Email`, `Zoom Meeting ID`, `Coach Email`, `Goals`, `Status`.
   - `Session Log`: Columns: `Session ID`, `Client ID`, `Client Name`, `Session Date`, `Duration`, `Summary`, `Homework Count`, `Progress Score`, `Next Session Focus`.
   - `Homework Tracker`: Columns: `Task ID`, `Client ID`, `Client Name`, `Client Email`, `Session Date`, `Task`, `Why`, `Due Date`, `Status`, `Nudge Count`, `Last Nudge Date`.
2. **Setup Credentials:** Configure credentials for Google Sheets OAuth2, Gmail OAuth2, and an OpenAI API Key.
3. **Build Intake Branch:**
   - Create a **Webhook** node (`Zoom Recording Webhook`) with path `zoom-coaching-recording` and HTTP Method `POST`.
   - Connect it to a **Switch** node (`Route Zoom Event`) with rules matching `endpoint.url_validation` and `recording.completed` / `recording.transcript_completed`.
   - Connect validation output to a **Crypto** node (`Sign Validation Token`) set to HMAC SHA256 using your Zoom Secret Token, followed by a **Respond to Webhook** node (`Return Validation Response`).
   - Connect recording output to a **Respond to Webhook** node (`Acknowledge Recording`) returning text `ok`.
   - Connect acknowledgement to a **Code** node (`Extract Recording Details`) using JavaScript to parse the incoming recording payload.
4. **Build Client Identification & Transcript Extraction:**
   - Chain **Google Sheets** (`Read Clients`) and **Google Sheets** (`Read Session Log`) to load records using your Google Sheet ID.
   - Add a **Code** node (`Match Client to Meeting`) to cross-reference Zoom IDs/topics and filter inactive clients or logged sessions.
   - Add an **If** node (`Has Zoom Transcript?`) evaluating `={{ $json.hasTranscript }}`.
   - **True Branch:** Add an **HTTP Request** node (`Download Zoom Transcript`) using the transcript URL and access token query parameter, followed by a **Code** node (`Clean Transcript Text`).
   - **False Branch:** Add an **HTTP Request** node (`Download Session Audio`), followed by an **OpenAI** node (`Transcribe Audio with Whisper`, resource: `audio`, operation: `transcribe`), followed by a **Set** node (`Use Whisper Transcript`).
5. **Build Action Plan & Delivery Branch:**
   - Connect both transcript paths to an **OpenAI** node (`Generate Action Plan`) using model `gpt-4o-mini`, temperature `0.3`, JSON output enabled, and prompt structures referencing client name, goals, session date, and transcript.
   - Add a **Code** node (`Build Recap & Homework Rows`) to generate HTML emails and task dates.
   - Add a **Gmail** node (`Email Recap to Client`) configured to send HTML content to the client email, CCing the coach.
   - Add a **Google Sheets** (`Log Session`) node to append data to the `Session Log` tab.
   - Add a **Code** node (`Prepare Homework Rows`) and a **Google Sheets** (`Add Homework to Tracker`) node to append tasks to the tracking tab.
6. **Build Daily Nudge Routine:**
   - Create a **Schedule Trigger** (`Every Day at 10 AM`) set to cron expression `0 10 * * *`.
   - Add a **Set** node (`Set Nudge Rules`) establishing parameters: `coachName`, `coachReplyTo`, `nudgeWindowDays` (e.g., `2`), `stopAfterDaysOverdue` (e.g., `7`), `minDaysBetweenNudges` (e.g., `2`), `maxNudgesPerTask` (e.g., `3`).
   - Add a **Google Sheets** node (`Read Homework Tracker`) to read all rows from the `Homework Tracker` tab.
   - Add a **Code** node (`Pick Clients to Nudge`) to filter and group open tasks.
   - Add an **OpenAI** node (`Draft Nudge Message`) using model `gpt-4o-mini`, temperature `0.6`, JSON output, to generate check-in drafts.
   - Add a **Gmail** node (`Send Nudge Email`) to send plain-text emails with reply-to headers.
   - Add a **Code** node (`Expand Nudged Tasks`) to flatten tasks and a **Google Sheets** (`Update Nudge Log`) node set to update rows by matching `Task ID`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zoom Marketplace Webhook Setup | Requires configuring a webhook app, subscribing to recording events, enabling download tokens, and pasting the secret token into the HMAC node. |
| Google Sheets Structure | Requires three tabs (`Clients`, `Session Log`, `Homework Tracker`) with specific column schemas referenced across multiple sheet nodes. |