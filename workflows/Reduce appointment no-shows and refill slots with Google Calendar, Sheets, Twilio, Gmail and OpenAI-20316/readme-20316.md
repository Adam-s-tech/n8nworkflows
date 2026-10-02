Reduce appointment no-shows and refill slots with Google Calendar, Sheets, Twilio, Gmail and OpenAI

https://n8nworkflows.xyz/workflows/reduce-appointment-no-shows-and-refill-slots-with-google-calendar--sheets--twilio--gmail-and-openai-20316


# Reduce appointment no-shows and refill slots with Google Calendar, Sheets, Twilio, Gmail and OpenAI

### 1. Workflow Overview

This workflow automates the full lifecycle of appointment management for clinics and service providers. Its primary goals are to reduce appointment no-shows, automatically refill cancelled slots from a waitlist, handle client replies via AI intent classification, manage check-up recalls, and provide operational reports to staff and management.

The workflow logic is categorized into the following functional blocks:

- **1.1 Entry Triggers & Settings:** Handles scheduled execution (every 15 minutes) and real-time inbound triggers from SMS (Twilio) or Email (Gmail), followed by global configuration initialization.
- **1.2 Data Ingestion & State Aggregation:** Loads and consolidates core operational data from Google Sheets (Clients, Services, Waitlist, Calendar Events, and Audit Log).
- **1.3 Strategic Planning & Logging:** Evaluates calendar state against historical records, plans confirmation requests, waitlist offers, recalls, and reports, and writes all decisions to the audit log before dispatching messages.
- **1.4 Outbound Messaging:** Routes planned communications (confirmations, offers, recalls, digests) to the correct channel (Twilio SMS or Gmail).
- **1.5 Inbound Reply Processing & AI Analysis:** Normalizes incoming replies, filters out automated/out-of-office responses, and uses OpenAI (`gpt-4o-mini`) to extract intent and exact evidentiary quotes.
- **1.6 Action Execution & Calendar Synchronization:** Processes client intents against active contexts, logs updates, updates Google Calendar (creating/deleting events), and sends direct responses.
- **1.7 Error Handling:** Listens for execution failures and emails administrative alerts.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Entry Triggers & Settings
- **Overview:** Initializes the workflow via a schedule or inbound webhooks/polls and establishes global clinic parameters and message templates.
- **Nodes Involved:** `Every 15 Minutes`, `When a Text Arrives`, `When an Email Arrives`, `Set Clinic Settings`, `Set Message Texts`.
- **Node Details:**
  - **Every 15 Minutes** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Periodic trigger running the planner routine.
    - *Config:* Interval set to every 15 minutes.
    - *Connections:* Outputs to `Set Clinic Settings`.
    - *Failure Modes:* Missed runs if n8n instance is offline.
  - **When a Text Arrives** (`n8n-nodes-base.twilioTrigger`)
    - *Role:* Webhook receiver for inbound Twilio SMS messages.
    - *Config:* Listens to inbound message events (`com.twilio.messaging.inbound-message.received`).
    - *Connections:* Outputs to `Set Clinic Settings`.
    - *Failure Modes:* Webhook URL misconfiguration in Twilio console.
  - **When an Email Arrives** (`n8n-nodes-base.gmailTrigger`)
    - *Role:* Polls Gmail inbox for incoming client replies.
    - *Config:* Query filter `in:inbox -from:me`, polled every 5 minutes.
    - *Connections:* Outputs to `Set Clinic Settings`.
    - *Failure Modes:* Gmail API authentication expiration.
  - **Set Clinic Settings** (`n8n-nodes-base.set`)
    - *Role:* Sets global variables (Sheet IDs, Calendar IDs, timezone, business rules).
    - *Config:* Defines static assignments including sheet identifiers, timezones, deposit thresholds, and a dynamic `trigger` property based on upstream execution source.
    - *Connections:* Input from all three triggers; output to `Set Message Texts`.
    - *Failure Modes:* Unreplaced placeholder values (`YOUR_SHEET_ID`).
  - **Set Message Texts** (`n8n-nodes-base.set`)
    - *Role:* Houses all user-facing localization and copy templates.
    - *Config:* Defines SMS and email body/subject templates (`confirm`, `nudge`, `offer`, `recall`, etc.).
    - *Connections:* Input from `Set Clinic Settings`; output to `Read Clients`.

#### Block 1.2: Data Ingestion & State Aggregation
- **Overview:** Reads data tabs from Google Sheets sequentially, aggregating multi-row results into single array payloads for the planner engine.
- **Nodes Involved:** `Read Clients`, `Aggregate Clients`, `Read Services`, `Aggregate Services`, `Read Waitlist`, `Aggregate Waitlist`, `Read Calendar Events`, `Aggregate Events`, `Read Log`, `Aggregate Log`.
- **Node Details:**
  - **Read Clients** / **Aggregate Clients** (`n8n-nodes-base.googleSheets` / `aggregate`)
    - *Role:* Fetches and rolls up client directory records.
    - *Config:* Reads sheet named `Clients` using dynamic sheet ID.
    - *Connections:* In: `Set Message Texts`; Out: `Read Services`.
    - *Failure Modes:* Missing sheet tab or API rate limits.
  - **Read Services** / **Aggregate Services** (`n8n-nodes-base.googleSheets` / `aggregate`)
    - *Role:* Fetches service catalogs (duration, pricing, recall intervals).
    - *Config:* Reads sheet named `Services`.
    - *Connections:* In: `Aggregate Clients`; Out: `Read Waitlist`.
    - *Failure Modes:* Schema mismatch on column headers.
  - **Read Waitlist** / **Aggregate Waitlist** (`n8n-nodes-base.googleSheets` / `aggregate`)
    - *Role:* Loads waiting list preferences and client requests.
    - *Config:* Reads sheet named `Waitlist`.
    - *Connections:* In: `Aggregate Services`; Out: `Read Calendar Events`.
    - *Failure Modes:* Empty tab structure causing parsing issues.
  - **Read Calendar Events** / **Aggregate Events** (`n8n-nodes-base.googleSheets` / `aggregate`)
    - *Role:* Fetches Google Calendar appointments within a dynamic lookahead/lookbehind window.
    - *Config:* `timeMin` set to $now - 2$ days; `timeMax` set to $now + \text{look\_ahead\_days}$.
    - *Connections:* In: `Aggregate Waitlist`; Out: `Read Log`.
    - *Failure Modes:* Calendar ID mismatch or insufficient OAuth permissions.
  - **Read Log** / **Aggregate Log** (`n8n-nodes-base.googleSheets` / `aggregate`)
    - *Role:* Reads historical audit logs to rebuild internal workflow state.
    - *Config:* Reads sheet named `Log`.
    - *Connections:* In: `Aggregate Events`; Out: `Route by Trigger`.
    - *Failure Modes:* Large sheet sizes causing performance degradation.

#### Block 1.3: Strategic Planning & Logging
- **Overview:** Routes execution flow, executes core decision logic in JavaScript, and appends actions to the audit log.
- **Nodes Involved:** `Route by Trigger`, `Plan the Next Actions`, `Append Planner Log`, `Pick Messages to Send`.
- **Node Details:**
  - **Route by Trigger** (`n8n-nodes-base.switch`)
    - *Role:* Branches execution based on whether the workflow was invoked by the scheduler or an inbound reply.
    - *Config:* Evaluates `{{ $('Set Clinic Settings').first().json.trigger }}` against `schedule` and `reply`.
    - *Connections:* In: `Aggregate Log`; Out 1: `Plan the Next Actions`, Out 2: `Read the Reply`.
  - **Plan the Next Actions** (`n8n-nodes-base.code`)
    - *Role:* Complex core JavaScript planner engine. Rebuilds state from logs, validates calendar integrity, identifies missed/cancelled slots, schedules confirmations (48h/24h), distributes waitlist batches, manages recalls, and generates operational digests and reports.
    - *Config:* Custom JS execution block handling date math, contact matching, and state machine transitions.
    - *Connections:* In: `Route by Trigger` (schedule branch); Out: `Append Planner Log`.
    - *Failure Modes:* JavaScript runtime errors due to malformed sheet columns or unexpected date formats.
  - **Append Planner Log** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends all planned actions to the Google Sheets audit log before message dispatch.
    - *Config:* Operation: `append`, Sheet: `Log`, raw cell formatting.
    - *Connections:* In: `Plan the Next Actions`; Out: `Pick Messages to Send`.
    - *Failure Modes:* Append write conflicts or sheet lockouts.
  - **Pick Messages to Send** (`n8n-nodes-base.code`)
    - *Role:* Filters logged output items that require outbound dispatch.
    - *Config:* Filters rows containing valid `send_to` and `send_text` properties.
    - *Connections:* In: `Append Planner Log`; Out: `Route by Channel`.

#### Block 1.4: Outbound Messaging
- **Overview:** Directs messages to Twilio SMS or Gmail according to recipient communication preference and opt-out status.
- **Nodes Involved:** `Route by Channel`, `Send Text Message`, `Send Email`.
- **Node Details:**
  - **Route by Channel** (`n8n-nodes-base.switch`)
    - *Role:* Splits traffic between SMS and Email channels based on message metadata.
    - *Config:* Compares `$json.channel` to `sms` and `email`.
    - *Connections:* In: `Pick Messages to Send`; Out 1: `Send Text Message`, Out 2: `Send Email`.
  - **Send Text Message** (`n8n-nodes-base.twilio`)
    - *Role:* Dispatches SMS notifications via Twilio.
    - *Config:* Maps `to`, `from` (Twilio number), and `message`.
    - *Connections:* In: `Route by Channel`; Out: None (Terminal node for outbound).
    - *Failure Modes:* Invalid phone number format, insufficient Twilio account funds, or carrier filtering.
  - **Send Email** (`n8n-nodes-base.gmail`)
    - *Role:* Dispatches email notifications via Gmail.
    - *Config:* Plain text email dispatch with configurable subject and recipient.
    - *Connections:* In: `Route by Channel`; Out: None (Terminal node for outbound).
    - *Failure Modes:* Gmail API quota exhaustion or invalid email addresses.

#### Block 1.5: Inbound Reply Processing & AI Analysis
- **Overview:** Standardizes inbound message structures, filters automated responses, and utilizes OpenAI to extract intent and evidentiary quotes.
- **Nodes Involved:** `Read the Reply`, `Classify the Reply`, `OpenAI Chat Model`.
- **Node Details:**
  - **Read the Reply** (`n8n-nodes-base.code`)
    - *Role:* Normalizes payloads from Twilio Trigger and Gmail Trigger, strips quoted historical email threads, detects automated out-of-office headers, and checks keyword shorthands.
    - *Config:* Custom JS parsing email and SMS payloads into a uniform structure.
    - *Connections:* In: `Route by Trigger` (reply branch); Out: `Classify the Reply`.
  - **Classify the Reply** (`@n8n/n8n-nodes-langchain.informationExtractor`)
    - *Role:* Structured information extraction using OpenAI to determine intent (`yes`, `no`, `reschedule`, `stop`, `question`, `other`) and mandatory evidentiary text quotes.
    - *Config:* Uses system prompt instructing the model to quote exact words from the message. `onError` set to `continueRegularOutput`.
    - *Connections:* In: `Read the Reply`, AI Model linked; Out: `Apply the Reply`.
    - *Failure Modes:* API rate limits, model hallucinations failing strict quote validation.
  - **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Provides the underlying language model for reply classification.
    - *Config:* Model: `gpt-4o-mini`, Temperature: `0`.
    - *Connections:* Linked to `Classify the Reply`.

#### Block 1.6: Action Execution & Calendar Synchronization
- **Overview:** Processes AI classification and keywords against active contexts, logs replies, updates Google Calendar events, and sends direct responses.
- **Nodes Involved:** `Apply the Reply`, `Append Reply Log`, `Pick Reply Actions`, `Route Reply Actions`, `Create Calendar Event`, `Delete Calendar Event`, `Send Reply Text`, `Send Reply Email`.
- **Node Details:**
  - **Apply the Reply** (`n8n-nodes-base.code`)
    - *Role:* Evaluates classified intents against active context (confirmations, offers, recalls), calculates cancellations or slot refills, and flags calendar sync actions.
    - *Config:* Extensive business logic matching incoming phone/email to active database rows.
    - *Connections:* In: `Classify the Reply`; Out: `Append Reply Log`.
  - **Append Reply Log** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends reply processing results and generated actions to the audit log.
    - *Config:* Operation: `append`, Sheet: `Log`.
    - *Connections:* In: `Apply the Reply`; Out: `Pick Reply Actions`.
  - **Pick Reply Actions** (`n8n-nodes-base.code`)
    - *Role:* Formats action payloads requiring calendar updates or client response dispatches.
    - *Config:* Filters items with defined `action` or outbound messaging parameters.
    - *Connections:* In: `Append Reply Log`; Out: `Route Reply Actions`.
  - **Route Reply Actions** (`n8n-nodes-base.switch`)
    - *Role:* Multi-output router handling simultaneous calendar mutations and response dispatches using `allMatchingOutputs: true`.
    - *Config:* Routes by `action` (`create`, `delete`) and `channel` (`sms`, `email`).
    - *Connections:* In: `Pick Reply Actions`; Out: `Create Calendar Event`, `Delete Calendar Event`, `Send Reply Text`, `Send Reply Email`.
  - **Create Calendar Event** (`n8n-nodes-base.googleCalendar`)
    - *Role:* Books refilled slots into Google Calendar.
    - *Config:* Operation: `create`, sets summary, start, end, and description containing refill metadata.
    - *Connections:* In: `Route Reply Actions`.
  - **Delete Calendar Event** (`n8n-nodes-base.googleCalendar`)
    - *Role:* Removes cancelled appointments from Google Calendar.
    - *Config:* Operation: `delete`, Event ID mapping. `onError` set to `continueRegularOutput`.
    - *Connections:* In: `Route Reply Actions`.
  - **Send Reply Text** (`n8n-nodes-base.twilio`)
    - *Role:* Sends direct SMS replies to clients.
    - *Config:* Maps `to`, `from`, and `message`.
    - *Connections:* In: `Route Reply Actions`.
  - **Send Reply Email** (`n8n-nodes-base.gmail`)
    - *Role:* Sends direct email replies to clients.
    - *Config:* Maps recipient, subject, and body.
    - *Connections:* In: `Route Reply Actions`.

#### Block 1.7: Error Handling
- **Overview:** Catastrophic error catching and alerting.
- **Nodes Involved:** `Workflow Failure Trigger`, `Alert on Workflow Failure`.
- **Node Details:**
  - **Workflow Failure Trigger** (`n8n-nodes-base.errorTrigger`)
    - *Role:* Captures unhandled workflow exceptions across any execution path.
    - *Config:* Default error trigger settings.
    - *Connections:* Out: `Alert on Workflow Failure`.
  - **Alert on Workflow Failure** (`n8n-nodes-base.gmail`)
    - *Role:* Sends detailed failure diagnostics to administrative staff.
    - *Config:* Dispatches email containing execution error message, last executed node name, and execution URL.
    - *Connections:* In: `Workflow Failure Trigger`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `stickyNote` | Documentation note | None | None | Reduce appointment no-shows and refill cancelled slots... |
| **Step 1** | `stickyNote` | Documentation note | None | None | Start on a timer or reply... |
| **Every 15 Minutes** | `scheduleTrigger` | Periodic timer trigger | None | Set Clinic Settings | |
| **When a Text Arrives** | `twilioTrigger` | Webhook SMS trigger | None | Set Clinic Settings | |
| **When an Email Arrives** | `gmailTrigger` | Polling Email trigger | None | Set Clinic Settings | |
| **Step 2** | `stickyNote` | Documentation note | None | None | Set your clinic and texts... |
| **Set Clinic Settings** | `set` | Configuration assignment | Every 15 Minutes, When a Text Arrives, When an Email Arrives | Set Message Texts | |
| **Set Message Texts** | `set` | Localization assignment | Set Clinic Settings | Read Clients | |
| **Step 3** | `stickyNote` | Documentation note | None | None | Load clients, services and waitlist... |
| **Read Clients** | `googleSheets` | Read Client data tab | Set Message Texts | Aggregate Clients | |
| **Aggregate Clients** | `aggregate` | Aggregate client rows | Read Clients | Read Services | |
| **Read Services** | `googleSheets` | Read Services tab | Aggregate Clients | Aggregate Services | |
| **Aggregate Services** | `aggregate` | Aggregate service rows | Read Services | Read Waitlist | |
| **Read Waitlist** | `googleSheets` | Read Waitlist tab | Aggregate Services | Aggregate Waitlist | |
| **Aggregate Waitlist** | `aggregate` | Aggregate waitlist rows | Read Waitlist | Read Calendar Events | |
| **Step 4** | `stickyNote` | Documentation note | None | None | Load the calendar and log... |
| **Read Calendar Events** | `googleCalendar` | Read calendar appointments | Aggregate Waitlist | Aggregate Events | |
| **Aggregate Events** | `aggregate` | Aggregate event rows | Read Calendar Events | Read Log | |
| **Read Log** | `googleSheets` | Read Audit Log tab | Aggregate Events | Aggregate Log | |
| **Aggregate Log** | `aggregate` | Aggregate log rows | Read Log | Route by Trigger | |
| **Step 5** | `stickyNote` | Documentation note | None | None | Plan what happens next... |
| **Route by Trigger** | `switch` | Branch by invocation source | Aggregate Log | Plan the Next Actions, Read the Reply | |
| **Plan the Next Actions** | `code` | Core planner engine | Route by Trigger | Append Planner Log | |
| **Step 6** | `stickyNote` | Documentation note | None | None | Write the log first... |
| **Append Planner Log** | `googleSheets` | Write audit log entries | Plan the Next Actions | Pick Messages to Send | |
| **Pick Messages to Send** | `code` | Filter outbound queue | Append Planner Log | Route by Channel | |
| **Step 7** | `stickyNote` | Documentation note | None | None | Send by text or email... |
| **Route by Channel** | `switch` | Branch by communication channel | Pick Messages to Send | Send Text Message, Send Email | |
| **Send Text Message** | `twilio` | Send outbound SMS | Route by Channel | None | |
| **Send Email** | `gmail` | Send outbound Email | Route by Channel | None | |
| **Step 8** | `stickyNote` | Documentation note | None | None | Read who replied and why... |
| **Read the Reply** | `code` | Parse and clean inbound reply | Route by Trigger | Classify the Reply | |
| **Classify the Reply** | `informationExtractor` | AI Intent Extraction | Read the Reply | Apply the Reply | |
| **OpenAI Chat Model** | `lmChatOpenAi` | LLM Provider configuration | None | Classify the Reply | |
| **Step 9** | `stickyNote` | Documentation note | None | None | Act on the reply... |
| **Apply the Reply** | `code` | Process reply against context | Classify the Reply | Append Reply Log | |
| **Append Reply Log** | `googleSheets` | Write reply audit log | Apply the Reply | Pick Reply Actions | |
| **Step 10** | `stickyNote` | Documentation note | None | None | Update the booking calendar... |
| **Pick Reply Actions** | `code` | Extract calendar/reply actions | Append Reply Log | Route Reply Actions | |
| **Route Reply Actions** | `switch` | Multi-output action dispatcher | Pick Reply Actions | Create Calendar Event, Delete Calendar Event, Send Reply Text, Send Reply Email | |
| **Create Calendar Event** | `googleCalendar` | Book refilled slot | Route Reply Actions | None | |
| **Delete Calendar Event** | `googleCalendar` | Remove cancelled appointment | Route Reply Actions | None | |
| **Step 11** | `stickyNote` | Documentation note | None | None | Answer the client... |
| **Send Reply Text** | `twilio` | Send reply SMS | Route Reply Actions | None | |
| **Send Reply Email** | `gmail` | Send reply Email | Route Reply Actions | None | |
| **Step 12** | `stickyNote` | Documentation note | None | None | Report a failure of the workflow... |
| **Workflow Failure Trigger** | `errorTrigger` | Global error listener | None | Alert on Workflow Failure | |
| **Alert on Workflow Failure** | `gmail` | Send error notification | Workflow Failure Trigger | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential setup instructions:

1. **Credential Setup:**
   - Configure credentials for **Google Calendar OAuth2**, **Google Sheets OAuth2**, **Gmail OAuth2**, **Twilio API**, and **OpenAI API**.

2. **Google Sheets Preparation:**
   - Create a Google Sheet containing four exact sheet tabs named: `Clients`, `Services`, `Waitlist`, and `Log`.

3. **Core Trigger Nodes:**
   - **Schedule Trigger:** Create `Every 15 Minutes` set to run every 15 minutes.
   - **Twilio Trigger:** Create `When a Text Arrives` listening to inbound message events. Note the generated webhook URL and configure it in your Twilio console.
   - **Gmail Trigger:** Create `When an Email Arrives` with filter query `in:inbox -from:me` polling every 5 minutes.

4. **Configuration Nodes:**
   - **Set Clinic Settings:** Add a Set node named `Set Clinic Settings`. Add string/number/boolean assignments matching your clinic configuration (e.g., `sheet_id`, `calendar_id`, `timezone: "Europe/London"`, `twilio_from`, etc.). Set `trigger` dynamically: `={{ $('Every 15 Minutes').isExecuted ? 'schedule' : 'reply' }}`.
   - **Set Message Texts:** Add a Set node named `Set Message Texts` containing all localized SMS and email templates (`confirm`, `nudge`, `offer`, `recall`, etc.).

5. **Data Ingestion Chain:**
   - Create a sequence of alternating Google Sheets Read nodes (`Read Clients`, `Read Services`, `Read Waitlist`, `Read Calendar Events`, `Read Log`) connected to Aggregate nodes (`Aggregate Clients`, `Aggregate Services`, `Aggregate Waitlist`, `Aggregate Events`, `Aggregate Log`).
   - Link `Set Message Texts` -> `Read Clients`. Chain each aggregate node sequentially into the next read node.

6. **Planner & Outbound Dispatch Chain:**
   - **Route by Trigger:** Add a Switch node routing based on `={{ $('Set Clinic Settings').first().json.trigger }}` (`schedule` vs `reply`).
   - **Plan the Next Actions:** Add a Code node containing the core planner JavaScript engine.
   - **Append Planner Log:** Add a Google Sheets node (Operation: `append`) pointing to the `Log` tab.
   - **Pick Messages to Send:** Add a Code node filtering items with valid outbound text and recipient properties.
   - **Route by Channel:** Add a Switch node splitting by `$json.channel` (`sms` vs `email`).
   - **Outbound Execution:** Connect to `Send Text Message` (Twilio node) and `Send Email` (Gmail node).

7. **Inbound Reply & AI Chain:**
   - **Read the Reply:** Add a Code node to parse and normalize inbound Twilio/Gmail payloads.
   - **Classify the Reply:** Add an Information Extractor AI node linked to an **OpenAI Chat Model** node configured with `gpt-4o-mini` (temperature `0`). Set node option `onError` to `continueRegularOutput`.
   - **Apply the Reply:** Add a Code node executing reply intent logic, updating context, and building actions.
   - **Append Reply Log:** Add a Google Sheets node (Operation: `append`) pointing to the `Log` tab.

8. **Action Execution & Synchronization Chain:**
   - **Pick Reply Actions:** Add a Code node formatting action payloads.
   - **Route Reply Actions:** Add a Switch node with option `allMatchingOutputs: true` routing to:
     - `Create Calendar Event` (Google Calendar `create` operation)
     - `Delete Calendar Event` (Google Calendar `delete` operation, `onError` set to `continueRegularOutput`)
     - `Send Reply Text` (Twilio node)
     - `Send Reply Email` (Gmail node)

9. **Error Handling:**
   - **Workflow Failure Trigger:** Add an Error Trigger node connected to an `Alert on Workflow Failure` Gmail node set to email your designated administrator address (`YOUR_ALERT_EMAIL`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Reduce appointment no-shows and refill slots workflow | Primary workflow documentation and architecture reference |
| Twilio Webhook Integration | Requires an active Twilio number capable of SMS messaging and webhook configuration |
| OpenAI GPT-4o-mini Integration | Used for structured intent extraction and evidence quoting from client replies |