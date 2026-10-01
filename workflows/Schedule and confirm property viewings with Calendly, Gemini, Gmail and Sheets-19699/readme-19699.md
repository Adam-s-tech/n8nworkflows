Schedule and confirm property viewings with Calendly, Gemini, Gmail and Sheets

https://n8nworkflows.xyz/workflows/schedule-and-confirm-property-viewings-with-calendly--gemini--gmail-and-sheets-19699


# Schedule and confirm property viewings with Calendly, Gemini, Gmail and Sheets

### 1. Workflow Overview

This workflow automates the end-to-end lifecycle of real estate property viewings, operating across two independent execution phases:
- **Phase 1 (Viewing Proposal):** Receives incoming leads via webhook, normalizes the request data, calculates available viewing slots based on customizable weekly office hours, uses an AI engine (Gemini, ChatGPT, or Claude) to draft a personalized email proposal with a Calendly booking link, dispatches the email via Gmail, and logs the request as "Pending" in Google Sheets.
- **Phase 2 (Booking Confirmation & Reminders):** Listens for Calendly booking webhooks, creates a Google Calendar event with both the lead and agent as attendees, triggers confirmation emails to both parties via Gmail, updates the Google Sheet row status to "Booked", pauses via a Wait node until 24 hours prior to the viewing, and finally sends automated 24-hour reminder emails to both the lead and the agent while updating the reminder log.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Phase 1 Input Reception & Normalization
- **Overview:** Captures incoming property viewing inquiries from external webhooks (or optional Google Sheets triggers) and standardizes lead and property fields into a uniform schema.
- **Nodes Involved:** `Webhook: Viewing Request Received`, `Google Sheets Trigger`, `Normalize Request Details`.
- **Node Details:**
  - `Webhook: Viewing Request Received`
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Version 2.1) — Entry point for Phase 1.
    - *Configuration:* HTTP Method set to `POST`, path configured as `viewing-request`.
    - *Input/Output:* No inputs; outputs raw HTTP request body.
    - *Edge Cases:* Unauthenticated endpoints; ensure payloads match expected keys or let the normalizer handle missing parameters.
  - `Google Sheets Trigger`
    - *Type and Technical Role:* `n8n-nodes-base.googleSheetsTrigger` (Version 1) — Alternative entry point triggered when a row is added. (Currently disabled by default).
    - *Configuration:* Polls every minute on event `rowAdded`.
    - *Input/Output:* No inputs; outputs newly added row data.
    - *Edge Cases:* API rate limits or polling delays.
  - `Normalize Request Details`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Data transformation node.
    - *Configuration:* JavaScript execution to extract and normalize fields from either Google Form responses or standard webhook formats, generating a unique `recordId` and ISO `timestamp`.
    - *Input/Output:* Input from either webhook or sheets trigger; outputs normalized lead object.
    - *Edge Cases:* Malformed incoming JSON structures throwing extraction exceptions.

#### 2.2 Block 2: Agent Configuration & Slot Generation
- **Overview:** Defines agency settings, available time windows, and automatically computes up to three upcoming available time slots based on days-ahead and slot duration rules.
- **Nodes Involved:** `Set: Agent Configuration`, `Generate Available Slots`.
- **Node Details:**
  - `Set: Agent Configuration`
    - *Type and Technical Role:* `n8n-nodes-base.set` (Version 3.4) — Sets global configuration variables for Phase 1.
    - *Configuration:* Assigns string variables including `AGENT_NAME`, `AGENT_EMAIL`, `AGENT_PHONE`, `CALENDAR_ID`, `CALENDLY_USERNAME`, `CALENDLY_EVENT_SLUG`, `AI_ENGINE`, `ANTHROPIC_API_KEY`, property details, daily working hours (`MON_HOURS` through `SUN_HOURS`), `VIEWING_DURATION`, and `SLOT_DAYS_AHEAD`.
    - *Input/Output:* Input from normalization node; outputs configuration assignments alongside incoming data.
    - *Edge Cases:* Empty hours string means the agent is unavailable on that day.
  - `Generate Available Slots`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Slot generation algorithm.
    - *Configuration:* JavaScript snippet parsing configuration limits and iterating through upcoming days to output up to 3 valid future time slots and a dynamic Calendly URL.
    - *Input/Output:* Input from configuration node; outputs enriched dataset containing `slots`, `slot1` through `slot3`, and `calendlyUrl`.
    - *Edge Cases:* If all configured weekly hours are blank, no slots are generated, leading to fallback outputs in subsequent steps.

#### 2.3 Block 3: AI Engine Routing & Generation
- **Overview:** Dynamically switches execution flow to the selected AI provider (Gemini, OpenAI, or Anthropic Claude) to draft a personalized email proposal and SMS snippet.
- **Nodes Involved:** `Switch: Select AI Engine`, `Gemini: Draft Viewing Proposal`, `OpenAI: Draft Viewing Proposal`, `Build Anthropic Request`, `Anthropic: Draft Viewing Proposal`, `Parse AI Output`.
- **Node Details:**
  - `Switch: Select AI Engine`
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Version 3.4) — Conditional router.
    - *Configuration:* Evaluates `{{ $json.config.AI_ENGINE }}` against rules for "Gemini", "ChatGPT", and "Claude".
    - *Input/Output:* Input from slot generator; branches into three exclusive routes.
  - `Gemini: Draft Viewing Proposal`
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (Version 1.2) — AI generation node using Google Gemini.
    - *Configuration:* Uses model `models/gemini-3-flash-preview` with a real estate assistant system message.
    - *Input/Output:* Input from switch rule 1; outputs Gemini text completion.
    - *Credentials:* Google Gemini API.
  - `OpenAI: Draft Viewing Proposal`
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (Version 2.3) — AI generation node using OpenAI (disabled by default).
    - *Configuration:* System prompt matching the assistant specifications.
    - *Input/Output:* Input from switch rule 2; outputs OpenAI text completion.
    - *Credentials:* OpenAI API.
  - `Build Anthropic Request`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Payload constructor for Claude.
    - *Configuration:* JavaScript snippet formatting the system and user prompts into a JSON string body for Anthropic.
    - *Input/Output:* Input from switch rule 3; outputs structured `anthropicBody`.
  - `Anthropic: Draft Viewing Proposal`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.4) — External API call to Anthropic.
    - *Configuration:* POST request to `https://api.anthropic.com/v1/messages` using header parameters (`x-api-key`, `anthropic-version`).
    - *Input/Output:* Input from request builder; outputs Claude JSON response.
  - `Parse AI Output`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Response parser and HTML builder.
    - *Configuration:* Normalizes outputs across Gemini, OpenAI, and Claude formats, cleans markdown blocks, parses JSON with a robust fallback mechanism, and constructs formatted HTML proposal emails.
    - *Input/Output:* Input from any of the three AI provider nodes; outputs unified lead, slot, and formatted email payload.

#### 2.4 Block 4: Proposal Delivery & Initial Logging
- **Overview:** Sends the AI-generated proposal email to the prospective lead and records the inquiry details in Google Sheets with a "Pending" status.
- **Nodes Involved:** `Gmail: Send Viewing Proposal`, `Sheets: Log Viewing Request`.
- **Node Details:**
  - `Gmail: Send Viewing Proposal`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2) — Email dispatch node.
    - *Configuration:* Sends email to `leadEmail` with `proposalEmailHtml` message body and generated subject line. Attribution appended is disabled.
    - *Input/Output:* Input from output parser; outputs delivery metadata.
    - *Credentials:* Gmail OAuth2.
    - *Edge Cases:* Invalid recipient email formats or authentication expiry.
  - `Sheets: Log Viewing Request`
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Version 4.7) — Spreadsheet logging node.
    - *Configuration:* Appends row data (`Record ID`, `Timestamp`, `Lead Name`, `Email`, `Phone`, `Property`, slots, `AI Engine Used`, `Email Sent` = true, `Status` = "Pending") to the selected Google Sheet.
    - *Input/Output:* Input from Gmail dispatch; outputs append confirmation.
    - *Credentials:* Google Sheets OAuth2.

#### 2.5 Block 5: Phase 2 Booking Reception & Configuration
- **Overview:** Entry point for Phase 2, triggered when a lead books an appointment via Calendly, setting independent agent configuration parameters and extracting event payload details.
- **Nodes Involved:** `Webhook: Calendly Booking Received`, `Set: Agent Configuration (Phase 2)`, `Extract Booking Details`.
- **Node Details:**
  - `Webhook: Calendly Booking Received`
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Version 2.1) — Phase 2 entry point webhook.
    - *Configuration:* HTTP Method set to `POST`, path configured as `calendly-booking-received`.
    - *Input/Output:* No inputs; outputs raw Calendly webhook body.
  - `Set: Agent Configuration (Phase 2)`
    - *Type and Technical Role:* `n8n-nodes-base.set` (Version 3.4) — Configuration variables mirror for Phase 2 execution context.
    - *Configuration:* Assigns `AGENT_NAME`, `AGENT_EMAIL`, `AGENT_PHONE`, `CALENDAR_ID`, `CALENDLY_USERNAME`, `CALENDLY_EVENT_SLUG`, `PROPERTY_ADDRESS`, `PROPERTY_URL`, and `PROPERTY_DESCRIPTION`.
    - *Input/Output:* Input from Calendly webhook; outputs agent configuration values.
  - `Extract Booking Details`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Payload parser.
    - *Configuration:* JavaScript extraction of event URIs, cancel/reschedule URLs, invitee details, start/end times, and computation of `reminderTime` (24 hours prior to viewing start).
    - *Input/Output:* Input from configuration node; outputs extracted booking properties.

#### 2.6 Block 6: Calendar Scheduling & Notifications
- **Overview:** Creates a Google Calendar event with the lead and agent as attendees and dispatches immediate confirmation emails to both parties while updating the Google Sheet status to "Booked".
- **Nodes Involved:** `Build Calendar Event Body`, `Create Viewing Event on Google Calendar`, `Gmail: Send Confirmation to Lead`, `Gmail: Notify Agent — Viewing Booked`, `Sheets: Update Status to Booked`.
- **Node Details:**
  - `Build Calendar Event Body`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Version 2) — Payload formatter.
    - *Configuration:* Cleans microsecond timestamp formatting and constructs Google Calendar API JSON body containing summary, location, description, time zone (UTC), and attendees list (`leadEmail`, `AGENT_EMAIL`).
    - *Input/Output:* Input from booking extraction; outputs `eventBodyJson`.
  - `Create Viewing Event on Google Calendar`
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.4) — REST API call to Google Calendar.
    - *Configuration:* POST request to `https://www.googleapis.com/calendar/v3/calendars/primary/events` using predefined Google Calendar OAuth2 credentials.
    - *Input/Output:* Input from event builder; outputs created calendar event details.
    - *Credentials:* Google Calendar OAuth2.
  - `Gmail: Send Confirmation to Lead`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2) — Lead confirmation email node.
    - *Configuration:* Sends HTML confirmation email containing property address, date/time, agent info, and cancel/reschedule links.
    - *Input/Output:* Input from calendar event creation; outputs email metadata.
    - *Credentials:* Gmail OAuth2.
  - `Gmail: Notify Agent — Viewing Booked`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2) — Agent notification email node.
    - *Configuration:* Sends HTML notification email to `AGENT_EMAIL` detailing lead particulars and booking time.
    - *Input/Output:* Input from calendar event creation; outputs email metadata.
    - *Credentials:* Gmail OAuth2.
  - `Sheets: Update Status to Booked`
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Version 4.7) — Spreadsheet update node.
    - *Configuration:* Updates spreadsheet row matching on `Email`, changing `Status` to "Booked", updating viewing start/end times and Calendly Event URL.
    - *Input/Output:* Input from lead confirmation email; outputs update confirmation.
    - *Credentials:* Google Sheets OAuth2.

#### 2.7 Block 7: Automated 24-Hour Reminders
- **Overview:** Pauses execution until 24 hours prior to the scheduled viewing, then simultaneously sends reminder emails to the lead and agent before updating the log to indicate reminders have been sent.
- **Nodes Involved:** `Wait: 24 Hours Before Viewing`, `Gmail: Reminder to Lead`, `Gmail: Reminder to Agent`, `Update Sheet After Reminder`.
- **Node Details:**
  - `Wait: 24 Hours Before Viewing`
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Version 1.1) — Execution pause node.
    - *Configuration:* Resume set to `specificTime`, evaluated at `{{ $('Extract Booking Details').first().json.reminderTime }}`.
    - *Input/Output:* Input from sheet update; outputs execution stream upon reaching target timestamp.
    - *Edge Cases:* If `reminderTime` has already passed (same-day bookings), the node resumes execution immediately.
  - `Gmail: Reminder to Lead`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2) — Lead reminder email node.
    - *Configuration:* Sends reminder email to `leadEmail` with property details and scheduling modification links.
    - *Input/Output:* Input from wait node; outputs email delivery receipt.
    - *Credentials:* Gmail OAuth2.
  - `Gmail: Reminder to Agent`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Version 2.2) — Agent reminder email node.
    - *Configuration:* Sends reminder email to `AGENT_EMAIL` summarizing lead contact details and viewing schedule.
    - *Input/Output:* Input from wait node; outputs email delivery receipt.
    - *Credentials:* Gmail OAuth2.
  - `Update Sheet After Reminder`
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Version 4.7) — Spreadsheet update node.
    - *Configuration:* Updates spreadsheet row matching on `Email`, setting `Reminder Sent` to `true`.
    - *Input/Output:* Input from lead reminder email; outputs final update confirmation.
    - *Credentials:* Google Sheets OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook: Viewing Request Received` | `n8n-nodes-base.webhook` | Phase 1 webhook entry point | None | `Normalize Request Details` | Phase 1 — Viewing Request Entry<br>Webhook is the default. It accepts a POST from any website form, CRM, or previous QualMatic template. Google Sheet trigger is optional and disabled. Both produce identical output into the Normalize node. |
| `Google Sheets Trigger` | `n8n-nodes-base.googleSheetsTrigger` | Alternative row-added trigger | None | `Normalize Request Details` | Phase 1 — Viewing Request Entry<br>Webhook is the default. It accepts a POST from any website form, CRM, or previous QualMatic template. Google Sheet trigger is optional and disabled. Both produce identical output into the Normalize node. |
| `Normalize Request Details` | `n8n-nodes-base.code` | Standardizes incoming lead schema | `Webhook: Viewing Request Received`, `Google Sheets Trigger` | `Set: Agent Configuration` | Request Normalization & Slot Generation<br>Incoming request is normalized from webhook or Google Sheet format into clean named fields. All agent settings — availability hours, AI engine, Calendly slug, and property details — are configured in the Agent Configuration node. Three viewing slots are calculated automatically from the availability windows. Empty hours = that day unavailable. |
| `Set: Agent Configuration` | `n8n-nodes-base.set` | Assigns Phase 1 global agent variables | `Normalize Request Details` | `Generate Available Slots` | Request Normalization & Slot Generation<br>Incoming request is normalized from webhook or Google Sheet format into clean named fields. All agent settings — availability hours, AI engine, Calendly slug, and property details — are configured in the Agent Configuration node. Three viewing slots are calculated automatically from the availability windows. Empty hours = that day unavailable.<br>Update Phase 1 Configuration<br>Set these values:<br>AGENT_NAME, AGENT_EMAIL, AGENT_PHONE<br>CALENDAR_ID CALENDLY_USERNAME and CALENDLY_EVENT_SLUG<br>PROPERTY_ADDRESS and PROPERTY_URL<br>AI_ENGINE — Gemini / ChatGPT / Claude<br>Availability hours — empty string means unavailable. |
| `Generate Available Slots` | `n8n-nodes-base.code` | Computes 3 valid viewing slots | `Set: Agent Configuration` | `Switch: Select AI Engine` | Request Normalization & Slot Generation<br>Incoming request is normalized from webhook or Google Sheet format into clean named fields. All agent settings — availability hours, AI engine, Calendly slug, and property details — are configured in the Agent Configuration node. Three viewing slots are calculated automatically from the availability windows. Empty hours = that day unavailable. |
| `Switch: Select AI Engine` | `n8n-nodes-base.switch` | Routes execution to chosen AI model | `Generate Available Slots` | `Gemini: Draft Viewing Proposal`, `OpenAI: Draft Viewing Proposal`, `Build Anthropic Request` | Choose Your AI Engine<br>Switch node routes to Gemini, ChatGPT, or Claude based on AI_ENGINE in the Configuration node. Only one fires per execution and the others are skipped. All three produce identical JSON output read by the same Parse AI Output node. Add credentials for whichever engine you prefer. All three can be connected simultaneously. |
| `Gemini: Draft Viewing Proposal` | `@n8n/n8n-nodes-langchain.googleGemini` | Generates proposal text via Gemini | `Switch: Select AI Engine` | `Parse AI Output` | Choose Your AI Engine<br>Switch node routes to Gemini, ChatGPT, or Claude based on AI_ENGINE in the Configuration node. Only one fires per execution and the others are skipped. All three produce identical JSON output read by the same Parse AI Output node. Add credentials for whichever engine you prefer. All three can be connected simultaneously. |
| `OpenAI: Draft Viewing Proposal` | `@n8n/n8n-nodes-langchain.openAi` | Generates proposal text via OpenAI | `Switch: Select AI Engine` | `Parse AI Output` | Choose Your AI Engine<br>Switch node routes to Gemini, ChatGPT, or Claude based on AI_ENGINE in the Configuration node. Only one fires per execution and the others are skipped. All three produce identical JSON output read by the same Parse AI Output node. Add credentials for whichever engine you prefer. All three can be connected simultaneously. |
| `Build Anthropic Request` | `n8n-nodes-base.code` | Prepares Anthropic API payload | `Switch: Select AI Engine` | `Anthropic: Draft Viewing Proposal` | Choose Your AI Engine<br>Switch node routes to Gemini, ChatGPT, or Claude based on AI_ENGINE in the Configuration node. Only one fires per execution and the others are skipped. All three produce identical JSON output read by the same Parse AI Output node. Add credentials for whichever engine you prefer. All three can be connected simultaneously. |
| `Anthropic: Draft Viewing Proposal` | `n8n-nodes-base.httpRequest` | Calls Anthropic API for Claude text | `Build Anthropic Request` | `Parse AI Output` | Choose Your AI Engine<br>Switch node routes to Gemini, ChatGPT, or Claude based on AI_ENGINE in the Configuration node. Only one fires per execution and the others are skipped. All three produce identical JSON output read by the same Parse AI Output node. Add credentials for whichever engine you prefer. All three can be connected simultaneously. |
| `Parse AI Output` | `n8n-nodes-base.code` | Parses AI response & formats email HTML | `Gemini: Draft Viewing Proposal`, `OpenAI: Draft Viewing Proposal`, `Anthropic: Draft Viewing Proposal` | `Gmail: Send Viewing Proposal` | Viewing Proposal — Phase 1 Output<br>Lead receives a personalized email with three available viewing slots and a Calendly booking link. Slots are calculated from the availability windows in the Configuration node — no manual diary checking. Every request logs to Google Sheets with Status = Pending. |
| `Gmail: Send Viewing Proposal` | `n8n-nodes-base.gmail` | Sends proposal email to lead | `Parse AI Output` | `Sheets: Log Viewing Request` | Viewing Proposal — Phase 1 Output<br>Lead receives a personalized email with three available viewing slots and a Calendly booking link. Slots are calculated from the availability windows in the Configuration node — no manual diary checking. Every request logs to Google Sheets with Status = Pending. |
| `Sheets: Log Viewing Request` | `n8n-nodes-base.googleSheets` | Logs inquiry to Google Sheets as Pending | `Gmail: Send Viewing Proposal` | None | Viewing Proposal — Phase 1 Output<br>Lead receives a personalized email with three available viewing slots and a Calendly booking link. Slots are calculated from the availability windows in the Configuration node — no manual diary checking. Every request logs to Google Sheets with Status = Pending. |
| `Webhook: Calendly Booking Received` | `n8n-nodes-base.webhook` | Phase 2 webhook entry point (Calendly) | None | `Set: Agent Configuration (Phase 2)` | Phase 2 — Booking Confirmation & Reminder<br><br>Fires independently when the lead books via Calendly. No connection to Phase 1 — shares only Google Sheets. Phase 2 has its own Configuration node — update both when changing agent details.<br>Register This URL in Calendly<br>Copy this webhook's Production URL and paste it into Calendly → Integrations → Webhooks.<br>Event to subscribe: invitee.created<br>Requires a permanent public n8n URL — not localhost. For local testing use the Test URL with a tunnel. |
| `Set: Agent Configuration (Phase 2)` | `n8n-nodes-base.set` | Sets Phase 2 global agent variables | `Webhook: Calendly Booking Received` | `Extract Booking Details` | Phase 2 Configuration & Booking Details<br>Phase 2 runs as a separate execution and cannot <br>access Phase 1 nodes. Update agent details in <br>this Configuration node independently.<br>Booking details — lead name, email, viewing time, <br>cancel and reschedule links — are extracted from <br>the Calendly webhook payload here.<br>Update Phase 2 Configuration<br>Phase 2 runs as a separate execution and cannot <br>read Phase 1's Configuration node.<br>Update AGENT_NAME, AGENT_EMAIL, AGENT_PHONE, <br>CALENDAR_ID, and PROPERTY_ADDRESS here too for keeping both Config nodes in sync. |
| `Extract Booking Details` | `n8n-nodes-base.code` | Parses Calendly booking payload | `Set: Agent Configuration (Phase 2)` | `Build Calendar Event Body` | Phase 2 Configuration & Booking Details<br>Phase 2 runs as a separate execution and cannot <br>access Phase 1 nodes. Update agent details in <br>this Configuration node independently.<br>Booking details — lead name, email, viewing time, <br>cancel and reschedule links — are extracted from <br>the Calendly webhook payload here. |
| `Build Calendar Event Body` | `n8n-nodes-base.code` | Constructs Google Calendar API JSON payload | `Extract Booking Details` | `Create Viewing Event on Google Calendar` | Confirmation & Calendar Event<br>When Calendly fires the booking webhook, a Google Calendar event is created with both lead and agent as attendees — both receive a calendar invite. Confirmation emails send to lead and agent in parallel. Google Sheets updates Status from Pending to Booked. |
| `Create Viewing Event on Google Calendar` | `n8n-nodes-base.httpRequest` | Creates event in Google Calendar | `Build Calendar Event Body` | `Gmail: Send Confirmation to Lead`, `Gmail: Notify Agent — Viewing Booked` | Confirmation & Calendar Event<br>When Calendly fires the booking webhook, a Google Calendar event is created with both lead and agent as attendees — both receive a calendar invite. Confirmation emails send to lead and agent in parallel. Google Sheets updates Status from Pending to Booked. |
| `Gmail: Send Confirmation to Lead` | `n8n-nodes-base.gmail` | Sends booking confirmation to lead | `Create Viewing Event on Google Calendar` | `Sheets: Update Status to Booked` | Confirmation & Calendar Event<br>When Calendly fires the booking webhook, a Google Calendar event is created with both lead and agent as attendees — both receive a calendar invite. Confirmation emails send to lead and agent in parallel. Google Sheets updates Status from Pending to Booked. |
| `Gmail: Notify Agent — Viewing Booked` | `n8n-nodes-base.gmail` | Sends booking notification to agent | `Create Viewing Event on Google Calendar` | None | Confirmation & Calendar Event<br>When Calendly fires the booking webhook, a Google Calendar event is created with both lead and agent as attendees — both receive a calendar invite. Confirmation emails send to lead and agent in parallel. Google Sheets updates Status from Pending to Booked. |
| `Sheets: Update Status to Booked` | `n8n-nodes-base.googleSheets` | Updates Google Sheet status to Booked | `Gmail: Send Confirmation to Lead` | `Wait: 24 Hours Before Viewing` | Confirmation & Calendar Event<br>When Calendly fires the booking webhook, a Google Calendar event is created with both lead and agent as attendees — both receive a calendar invite. Confirmation emails send to lead and agent in parallel. Google Sheets updates Status from Pending to Booked. |
| `Wait: 24 Hours Before Viewing` | `n8n-nodes-base.wait` | Pauses execution until 24h before viewing | `Sheets: Update Status to Booked` | `Gmail: Reminder to Lead`, `Gmail: Reminder to Agent` | 24-Hour Reminder<br>The Wait node pauses execution until exactly 24 hours before the viewing start time — then fires reminder emails to both lead and agent simultaneously. Cancel and reschedule links are included in both reminder emails. |
| `Gmail: Reminder to Lead` | `n8n-nodes-base.gmail` | Sends 24h reminder email to lead | `Wait: 24 Hours Before Viewing` | `Update Sheet After Reminder` | 24-Hour Reminder<br>The Wait node pauses execution until exactly 24 hours before the viewing start time — then fires reminder emails to both lead and agent simultaneously. Cancel and reschedule links are included in both reminder emails. |
| `Gmail: Reminder to Agent` | `n8n-nodes-base.gmail` | Sends 24h reminder email to agent | `Wait: 24 Hours Before Viewing` | None | 24-Hour Reminder<br>The Wait node pauses execution until exactly 24 hours before the viewing start time — then fires reminder emails to both lead and agent simultaneously. Cancel and reschedule links are included in both reminder emails. |
| `Update Sheet After Reminder` | `n8n-nodes-base.googleSheets` | Updates Google Sheet reminder sent flag | `Gmail: Reminder to Lead` | None | 24-Hour Reminder<br>The Wait node pauses execution until exactly 24 hours before the viewing start time — then fires reminder emails to both lead and agent simultaneously. Cancel and reschedule links are included in both reminder emails. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create Phase 1 Entry Point and Normalizer
1. Create a **Webhook** node named `Webhook: Viewing Request Received`. Set HTTP Method to `POST` and path to `viewing-request`.
2. (Optional) Create a **Google Sheets Trigger** node configured with `rowAdded`. (Leave disabled if using webhooks exclusively).
3. Create a **Code** node named `Normalize Request Details`. Connect the webhook output to this node. Add the JavaScript snippet from the source workflow to extract lead data and generate a unique `recordId` and `timestamp`.

#### Step 2: Set Up Agent Configuration and Slot Generator
1. Create a **Set** node named `Set: Agent Configuration`. Connect `Normalize Request Details` to it.
2. Add string assignments for `AGENT_NAME`, `AGENT_EMAIL`, `AGENT_PHONE`, `CALENDAR_ID`, `CALENDLY_USERNAME`, `CALENDLY_EVENT_SLUG`, `AI_ENGINE` (set to `Gemini`), `ANTHROPIC_API_KEY`, `PROPERTY_ADDRESS`, `PROPERTY_URL`, `PROPERTY_DESCRIPTION`, daily hours (`MON_HOURS` through `SUN_HOURS`), `VIEWING_DURATION` (`30`), and `SLOT_DAYS_AHEAD` (`7`).
3. Create a **Code** node named `Generate Available Slots`. Connect `Set: Agent Configuration` to it. Insert the slot calculation logic script to produce up to three valid future slots and the Calendly booking URL.

#### Step 3: Configure AI Routing and Generation Nodes
1. Create a **Switch** node named `Switch: Select AI Engine`. Connect `Generate Available Slots` to it. Define 3 string rules evaluating `{{ $json.config.AI_ENGINE }}` for equals `Gemini`, `ChatGPT`, and `Claude`.
2. **Gemini Route:** Create a Google Gemini node named `Gemini: Draft Viewing Proposal` using model `models/gemini-3-flash-preview`. Connect rule 1 of the switch node to it. Connect **Google Gemini API** credentials.
3. **OpenAI Route:** Create an OpenAI node named `OpenAI: Draft Viewing Proposal`. Connect rule 2 of the switch node to it. (Keep disabled if using Gemini).
4. **Anthropic Route:** 
   - Create a **Code** node named `Build Anthropic Request`. Connect rule 3 of the switch node to it, using the script to format the Claude JSON body.
   - Create an **HTTP Request** node named `Anthropic: Draft Viewing Proposal`. Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`, body content type to raw JSON, and add header parameters for `x-api-key`, `anthropic-version`, and `content-type`. Connect the request builder to this node.

#### Step 4: Parse AI Output and Dispatch Proposal
1. Create a **Code** node named `Parse AI Output`. Connect outputs from `Gemini: Draft Viewing Proposal`, `OpenAI: Draft Viewing Proposal`, and `Anthropic: Draft Viewing Proposal` into this single node. Use the parsing and HTML formatting script.
2. Create a **Gmail** node named `Gmail: Send Viewing Proposal`. Connect `Parse AI Output` to it. Configure `sendTo` (`{{ $json.leadEmail }}`), subject (`{{ $json.email_subject }}`), and message (`{{ $json.proposalEmailHtml }}`). Connect **Gmail OAuth2** credentials.
3. Create a **Google Sheets** node named `Sheets: Log Viewing Request`. Connect Gmail node to it. Set operation to `append`, document ID, sheet name, and map columns (`Email`, `Phone`, `Slot 1`, `Slot 2`, `Slot 3`, `Status` = "Pending", `Property`, `Lead Name`, `Record ID`, `Timestamp`, `Email Sent` = true, `Reminder Sent` = false, `AI Engine Used`). Connect **Google Sheets OAuth2** credentials.

#### Step 5: Create Phase 2 Booking Receiver and Configuration
1. Create a **Webhook** node named `Webhook: Calendly Booking Received`. Set HTTP Method to `POST` and path to `calendly-booking-received`.
2. Create a **Set** node named `Set: Agent Configuration (Phase 2)`. Connect the webhook to it. Assign agent configuration variables (`AGENT_NAME`, `AGENT_EMAIL`, `AGENT_PHONE`, `CALENDAR_ID`, `CALENDLY_USERNAME`, `CALENDLY_EVENT_SLUG`, `PROPERTY_ADDRESS`, `PROPERTY_URL`, `PROPERTY_DESCRIPTION`).
3. Create a **Code** node named `Extract Booking Details`. Connect the Phase 2 configuration node to it. Use the parsing script to extract invitee details, viewing times, cancel/reschedule links, and calculate `reminderTime`.

#### Step 6: Calendar Event Creation and Notifications
1. Create a **Code** node named `Build Calendar Event Body`. Connect `Extract Booking Details` to it to format `eventBodyJson`.
2. Create an **HTTP Request** node named `Create Viewing Event on Google Calendar`. Set method to `POST`, URL to `https://www.googleapis.com/calendar/v3/calendars/primary/events`, content type to raw JSON, and authenticate using **Google Calendar OAuth2**. Connect the event body builder to this node.
3. Create two **Gmail** nodes connected in parallel from the calendar event node:
   - `Gmail: Send Confirmation to Lead`: Sends booking confirmation HTML to `leadEmail`. Connect **Gmail OAuth2**.
   - `Gmail: Notify Agent — Viewing Booked`: Sends booking notification HTML to `AGENT_EMAIL`. Connect **Gmail OAuth2**.
4. Create a **Google Sheets** node named `Sheets: Update Status to Booked`. Connect `Gmail: Send Confirmation to Lead` to it. Set operation to `update`, matching columns to `Email`, and map updated fields (`Status` = "Booked", `Viewing Start`, `Viewing End`, `Calendly Event URL`). Connect **Google Sheets OAuth2**.

#### Step 7: Automated 24-Hour Reminders
1. Create a **Wait** node named `Wait: 24 Hours Before Viewing`. Connect `Sheets: Update Status to Booked` to it. Set resume to `specificTime` with dateTime `{{ $('Extract Booking Details').first().json.reminderTime }}`.
2. Create two **Gmail** nodes connected in parallel from the Wait node:
   - `Gmail: Reminder to Lead`: Sends reminder email to `leadEmail`. Connect **Gmail OAuth2**.
   - `Gmail: Reminder to Agent`: Sends reminder email to `AGENT_EMAIL`. Connect **Gmail OAuth2**.
3. Create a **Google Sheets** node named `Update Sheet After Reminder`. Connect `Gmail: Reminder to Lead` to it. Set operation to `update`, matching columns to `Email`, and map `Reminder Sent` to `true`. Connect **Google Sheets OAuth2**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Template #5: Qualify Facebook lead ads and send follow-ups with Gemini, Gmail, and Sheets | https://n8n.io/workflows/17258-qualify-facebook-lead-ads-and-send-follow-ups-with-gemini-gmail-and-sheets/ |
| Template #6: Qualify inbound real estate leads from Gmail with Gemini and HubSpot | https://n8n.io/workflows/17409-qualify-inbound-real-estate-leads-from-gmail-with-gemini-and-hubspot/ |
| Template #10: Match new property listings to buyer leads with Gemini, Gmail, and Sheets | https://n8n.io/workflows/18329-match-new-property-listings-to-buyer-leads-with-gemini-gmail-and-sheets/ |