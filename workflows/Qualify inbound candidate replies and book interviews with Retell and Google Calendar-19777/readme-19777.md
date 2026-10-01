Qualify inbound candidate replies and book interviews with Retell and Google Calendar

https://n8nworkflows.xyz/workflows/qualify-inbound-candidate-replies-and-book-interviews-with-retell-and-google-calendar-19777


# Qualify inbound candidate replies and book interviews with Retell and Google Calendar

### 1. Workflow Overview

This workflow automates the processing of inbound candidate replies, delivery status callbacks from Twilio, and AI screening results from Retell (`call_analyzed` webhooks). It serves recruitment teams by immediately halting active messaging sequences upon receiving any reply, evaluating structured qualification metrics, booking interviews in Google Calendar for qualified candidates, writing standardized notes and status updates back to the ATS, and notifying recruiters via Slack and Gmail.

The workflow logic is grouped into six functional blocks:
- **1.1 Input Reception & Security:** Receives webhooks on a single public endpoint, loads global parameters, and cryptographically verifies request signatures using HMAC-SHA1 (Twilio) or HMAC-SHA256 (Retell).
- **1.2 Event Parsing & Opt-Out Handling:** Normalizes incoming payloads, determines the intent of SMS messages or AI screening outcomes, and executes transactional global opt-outs in Supabase if requested.
- **1.3 Candidate Qualification:** Evaluates structured custom analysis data against configurable `QUALIFY` rules to categorize candidates as qualified, needs review, or not qualified.
- **1.4 Interview Scheduling:** Selects the assigned recruiter and searches for the next available booking slot, creating a Google Calendar event and recording the appointment in Supabase when writeback is enabled.
- **1.5 ATS Write-Back:** Queues and posts structured interview notes and status updates back to the client's ATS (e.g., Bullhorn) using idempotency keys.
- **1.6 Notifications, Logging, & Response:** Dispatches optional alerts to Slack and Gmail, logs all inbound events to Supabase, and returns a fast, provider-friendly webhook response.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Security
- **Overview:** Receives raw HTTP POST payloads from Twilio and Retell on a unified webhook URL, instantiates global configuration variables, and validates security signatures to fail closed on unauthorized requests.
- **Nodes Involved:** `Inbound webhook`, `config`, `verify the signature`, `signature ok?`, `STOP: signature rejected`

##### Node Details:
- **Inbound webhook**
  - *Type:* `n8n-nodes-base.webhook`
  - *Technical Role:* Entry point for HTTP POST requests from Twilio and Retell.
  - *Configuration:* Path `stratton-inbound`, method `POST`, response mode set to `responseNode`.
  - *Connections:* Input: None; Output: `config`.
  - *Failure Modes:* Network timeout or gateway routing misconfigurations.
- **config**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Central configuration store for database URLs, ATS credentials, matching rules, outreach sequences, and safety switches (`DEMO_MODE`, `TEST_RUN`).
  - *Configuration:* JavaScript execution returning a comprehensive configuration object.
  - *Connections:* Input: `Inbound webhook`; Output: `verify the signature`.
- **verify the signature**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Cryptographically validates incoming requests using `X-Twilio-Signature` (HMAC-SHA1) or `X-Retell-Signature` (HMAC-SHA256).
  - *Configuration:* Uses Web Crypto API (`crypto.subtle`) to compute and compare HMAC signatures securely.
  - *Connections:* Input: `config`; Output: `signature ok?`.
  - *Failure Modes:* Mismatched proxy headers causing incorrect public URL reconstruction for Twilio.
- **signature ok?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Routes requests based on whether `_allow` is true (verified or preview mode).
  - *Configuration:* Evaluates `{{ $json._allow }}` equals `true`.
  - *Connections:* Input: `verify the signature`; Outputs: True -> `read the event`, False -> `STOP: signature rejected`.
- **STOP: signature rejected**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Prepares an HTTP 403 error response payload for unverified requests.
  - *Configuration:* Returns a descriptive error message indicating signature mismatch or missing secrets.
  - *Connections:* Input: `signature ok?` (False); Output: `the webhook response`.

---

#### 2.2 Event Parsing & Opt-Out Handling
- **Overview:** Normalizes heterogeneous payloads from Twilio SMS, Twilio status callbacks, and Retell voice calls into a unified schema, assesses message intent, and executes immediate opt-out sequences if requested.
- **Nodes Involved:** `read the event`, `actionable?`, `STOP: event ignored`, `opted out?`, `handle the opt-out`, `database connected?`, `STOP: opt-out not persisted`, `[cred] Supabase - stop all outreach`

##### Node Details:
- **read the event**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Parses request bodies to categorize events as `twilio_sms`, `twilio_status`, or `retell_call`, extracting phone numbers and intent keywords.
  - *Configuration:* JavaScript code parsing phone numbers to E.164 format and running boundary-word matching for opt-out, positive, and negative intent.
  - *Connections:* Input: `signature ok?` (True); Output: `actionable?`.
- **actionable?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Filters out non-actionable events (e.g., delivery receipts, call start events, unanswered calls).
  - *Configuration:* Evaluates `{{ $json._understood && $json._stop_sequence === true }}`.
  - *Connections:* Input: `read the event`; Outputs: True -> `opted out?`, False -> `STOP: event ignored`.
- **STOP: event ignored**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Formats a status 200 payload for ignored events so providers do not retry unnecessarily.
  - *Configuration:* Returns explanation metadata for bookkeeping events.
  - *Connections:* Input: `actionable?` (False); Output: `logging the event?`.
- **opted out?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Checks if the candidate intent is categorized as an opt-out.
  - *Configuration:* Evaluates `{{ $json.intent === 'opt_out' }}`.
  - *Connections:* Input: `actionable?` (True); Outputs: True -> `handle the opt-out`, False -> `qualify the candidate`.
- **handle the opt-out**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Structures the payload for stopping outreach and generating compliance notes.
  - *Configuration:* Prepares RPC parameters for Supabase and determines if a confirmation text is required.
  - *Connections:* Input: `opted out?` (True); Output: `database connected?`.
- **database connected?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Verifies whether Supabase integration is enabled in the configuration.
  - *Configuration:* Evaluates `{{ $('config').first().json.supabase_enabled }}`.
  - *Connections:* Input: `handle the opt-out`; Outputs: True -> `[cred] Supabase - stop all outreach`, False -> `STOP: opt-out not persisted`.
- **STOP: opt-out not persisted**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Logs a high-severity warning when an opt-out arrives without an active database configuration.
  - *Configuration:* Returns warning metadata explaining the missing persistence layer.
  - *Connections:* Input: `database connected?` (False); Output: `logging the event?`.
- **[cred] Supabase - stop all outreach**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Executes a single database transaction via RPC to add the phone number to the global opt-out list and nullify pending campaign enrollments.
  - *Configuration:* HTTP POST to `{{ supabase_url }}/rest/v1/rpc/stop_all_outreach` with Bearer token authentication.
  - *Credentials Required:* Supabase Service Key.
  - *Connections:* Input: `database connected?` (True); Output: `logging the event?`.

---

#### 2.3 Candidate Qualification
- **Overview:** Evaluates structured candidate answers from Retell AI screening calls or SMS interactions against deterministic configuration rules to establish qualification status.
- **Nodes Involved:** `qualify the candidate`, `qualified?`, `STOP: not qualified`

##### Node Details:
- **qualify the candidate**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Scores candidates against mandatory (`must`) criteria and point-based (`score`) thresholds defined in configuration.
  - *Configuration:* JavaScript execution enforcing strict evaluation where unestablished answers (`null`) never pass mandatory checks.
  - *Connections:* Input: `opted out?` (False); Output: `qualified?`.
- **qualified?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Routes candidates based on whether they met the qualification pass mark.
  - *Configuration:* Evaluates `{{ $json._qualified }}`.
  - *Connections:* Input: `qualify the candidate`; Outputs: True -> `pick the recruiter and slot`, False -> `STOP: not qualified`.
- **STOP: not qualified**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Summarizes reasons for disqualification or human review requirements before ATS writeback.
  - *Configuration:* Compiles missing answers and failed requirement details.
  - *Connections:* Input: `qualified?` (False); Output: `one outcome` (Input 2).

---

#### 2.4 Interview Scheduling
- **Overview:** Selects the responsible recruiter and identifies the next available interview time slot, attempting to book the meeting in Google Calendar and recording the appointment in Supabase.
- **Nodes Involved:** `pick the recruiter and slot`, `can we book?`, `[cred] Google Calendar - book the interview`, `[cred] Supabase - record the appointment`, `STOP: qualified - not booked`, `one outcome`

##### Node Details:
- **pick the recruiter and slot**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Determines the calendar ID based on job order ownership and iterates forward to find the first valid weekday booking slot within allowable hours.
  - *Configuration:* JavaScript timezone-aware calculation respecting lead hours and booking window constraints.
  - *Connections:* Input: `qualified?` (True); Output: `can we book?`.
- **can we book?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Checks if calendar integration is enabled, writebacks are active, and recruiter email information is present.
  - *Configuration:* Evaluates `{{ calendar_enabled && writeback_enabled && !$json._recruiter_missing }}`.
  - *Connections:* Input: `pick the recruiter and slot`; Outputs: True -> `[cred] Google Calendar - book the interview`, False -> `STOP: qualified - not booked`.
- **[cred] Google Calendar - book the interview**
  - *Type:* `n8n-nodes-base.googleCalendar`
  - *Technical Role:* Creates a calendar event for the qualified candidate interview.
  - *Configuration:* Uses start/end ISO timestamps, dynamic calendar ID, summary, attendee list, and description payload.
  - *Credentials Required:* Google Calendar OAuth2 API.
  - *Connections:* Input: `can we book?` (True); Output: `[cred] Supabase - record the appointment`.
- **[cred] Supabase - record the appointment**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Records the booked appointment details in Supabase with conflict resolution.
  - *Configuration:* HTTP POST to `{{ supabase_url }}/rest/v1/appointments?on_conflict=enrollment_id,starts_at` with upsert headers.
  - *Credentials Required:* Supabase Service Key.
  - *Connections:* Input: `[cred] Google Calendar - book the interview`; Output: `one outcome`.
- **STOP: qualified - not booked**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Handles scenarios where a candidate is qualified but calendar booking is skipped or missing recruiter details.
  - *Configuration:* Formats fallback metadata while preserving proposed slot data for manual booking.
  - *Connections:* Input: `can we book?` (False); Output: `one outcome` (Input 2).
- **one outcome**
  - *Type:* `n8n-nodes-base.merge`
  - *Technical Role:* Merges the successfully booked path and unbooked qualified paths back into a single processing stream.
  - *Configuration:* Standard two-input merge node.
  - *Connections:* Inputs: `[cred] Supabase - record the appointment` (Input 1), `STOP: qualified - not booked` (Input 2); Output: `the write-back payload`.

---

#### 2.5 ATS Write-Back
- **Overview:** Generates standardized note and status update payloads, queuing them into Supabase before posting them to the external ATS if live writebacks are enabled.
- **Nodes Involved:** `the write-back payload`, `writing back?`, `[cred] Supabase - queue the write-back`, `[cred] ATS - add the note`, `STOP: write-back skipped`

##### Node Details:
- **the write-back payload**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Constructs idempotent payloads for ATS notes and status updates, alongside Slack and email message templates.
  - *Configuration:* JavaScript building deterministic idempotency keys based on candidate ID, action type, and date.
  - *Connections:* Input: `one outcome`; Output: `writing back?`.
- **writing back?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Determines whether live writebacks are permitted based on operational mode (`MODE === 'live'`).
  - *Configuration:* Evaluates `{{ $json._write }}`.
  - *Connections:* Input: `the write-back payload`; Outputs: True -> `[cred] Supabase - queue the write-back`, False -> `STOP: write-back skipped`.
- **[cred] Supabase - queue the write-back**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Queues write-back payloads in Supabase with conflict ignore rules prior to ATS submission.
  - *Configuration:* HTTP POST to `{{ supabase_url }}/rest/v1/ats_writebacks?on_conflict=client_id,idempotency_key`.
  - *Credentials Required:* Supabase Service Key.
  - *Connections:* Input: `writing back?` (True); Output: `[cred] ATS - add the note`.
- **[cred] ATS - add the note**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Posts the evaluation note and status update to the external ATS REST API (e.g., Bullhorn).
  - *Configuration:* HTTP POST to `{{ ats_base_url }}/entity/Note` using bearer tokens and custom vendor headers (`BhRestToken`).
  - *Credentials Required:* ATS REST API Token.
  - *Connections:* Input: `[cred] Supabase - queue the write-back`; Output: `tell the recruiter?`.
- **STOP: write-back skipped**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Captures simulated write-back details when running in preview or test mode.
  - *Configuration:* Returns dry-run preview objects showing what would have been written to the ATS.
  - *Connections:* Input: `writing back?` (False); Output: `tell the recruiter?`.

---

#### 2.6 Notifications, Logging, & Response
- **Overview:** Dispatches recruiter alerts via Slack and Gmail, logs all inbound webhook events to Supabase, and returns a rapid response to the webhook provider.
- **Nodes Involved:** `tellslack?`, `Post the recruiter alert`, `tellmail?`, `[cred] Gmail - tell the recruiter`, `dolog?`, `[cred] Supabase - log the event`, `resp`, `Respond to the provider`

##### Node Details:
- **tellslack?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Checks if Slack webhook integration is configured.
  - *Configuration:* Evaluates `{{ $('config').first().json.slack_enabled }}`.
  - *Connections:* Input: `[cred] ATS - add the note` / `STOP: write-back skipped`; Outputs: True -> `Post the recruiter alert`, False -> `tellmail?`.
- **Post the recruiter alert**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Sends formatted markdown alerts to a Slack webhook channel.
  - *Configuration:* HTTP POST to `slack_webhook_url` with JSON text payloads.
  - *Connections:* Input: `tellslack?` (True); Output: `tellmail?`.
- **tellmail?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Checks if email notifications to recruiters are enabled, live writebacks are active, and a recruiter email exists.
  - *Configuration:* Evaluates `{{ notify_recruiter_email && writeback_enabled && !!booking.recruiter_email }}`.
  - *Connections:* Inputs: `tellslack?` (False), `Post the recruiter alert`; Outputs: True -> `[cred] Gmail - tell the recruiter`, False -> `dolog?`.
- **[cred] Gmail - tell the recruiter**
  - *Type:* `n8n-nodes-base.gmail`
  - *Technical Role:* Sends email notifications to recruiters regarding candidate status and interview bookings.
  - *Configuration:* Sets recipient, subject line, and plain text message body.
  - *Credentials Required:* Gmail OAuth2 API.
  - *Connections:* Input: `tellmail?` (True); Output: `dolog?`.
- **dolog?**
  - *Type:* `n8n-nodes-base.if`
  - *Technical Role:* Checks if Supabase database logging is enabled.
  - *Configuration:* Evaluates `{{ $('config').first().json.supabase_enabled }}`.
  - *Connections:* Inputs: `STOP: event ignored`, `tellmail?` (False), `[cred] Gmail - tell the recruiter`; Outputs: True -> `[cred] Supabase - log the event`, False -> `resp`.
- **[cred] Supabase - log the event**
  - *Type:* `n8n-nodes-base.httpRequest`
  - *Technical Role:* Records raw inbound events and normalized outcomes in Supabase with conflict resolution on event IDs.
  - *Configuration:* HTTP POST to `{{ supabase_url }}/rest/v1/inbound_events?on_conflict=client_id,source,provider_event_id`.
  - *Credentials Required:* Supabase Service Key.
  - *Connections:* Input: `dolog?` (True); Output: `resp`.
- **resp**
  - *Type:* `n8n-nodes-base.code`
  - *Technical Role:* Formats the HTTP status code and response body (TwiML XML for Twilio or JSON for Retell) to ensure rapid provider acknowledgment.
  - *Configuration:* JavaScript preparing content types and response wrappers.
  - *Connections:* Inputs: `STOP: signature rejected`, `dolog?` (False), `[cred] Supabase - log the event`; Output: `Respond to the provider`.
- **Respond to the provider**
  - *Type:* `n8n-nodes-base.respondToWebhook`
  - *Technical Role:* Sends the final HTTP response back to Twilio or Retell within required timeout limits.
  - *Configuration:* Responds with text/XML or application/json using dynamic status codes and headers.
  - *Connections:* Input: `resp`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Inbound webhook | `n8n-nodes-base.webhook` | Entry point for Twilio and Retell webhooks | None | config | Template overview |
| config | `n8n-nodes-base.code` | Loads global configuration and safety toggles | Inbound webhook | verify the signature | Template overview |
| verify the signature | `n8n-nodes-base.code` | Validates HMAC-SHA1 or HMAC-SHA256 signatures | config | signature ok? | 1. Receive and verify |
| signature ok? | `n8n-nodes-base.if` | Routes verified requests | verify the signature | read the event, STOP: signature rejected | 1. Receive and verify |
| STOP: signature rejected | `n8n-nodes-base.code` | Formats 403 error for unverified requests | signature ok? | the webhook response | 1. Receive and verify |
| read the event | `n8n-nodes-base.code` | Normalizes payloads and detects message intent | signature ok? | actionable? | 1. Receive and verify |
| actionable? | `n8n-nodes-base.if` | Filters out non-actionable events | read the event | opted out?, STOP: event ignored | 1. Receive and verify |
| STOP: event ignored | `n8n-nodes-base.code` | Formats 200 response for ignored events | actionable? | logging the event? | 1. Receive and verify |
| opted out? | `n8n-nodes-base.if` | Detects opt-out intent keywords | actionable? | handle the opt-out, qualify the candidate | 1. Receive and verify |
| handle the opt-out | `n8n-nodes-base.code` | Prepares database RPC call for opt-outs | opted out? | database connected? | 2. Opt-out is one database call |
| database connected? | `n8n-nodes-base.if` | Checks if Supabase is configured | handle the opt-out | [cred] Supabase - stop all outreach, STOP: opt-out not persisted | 2. Opt-out is one database call |
| STOP: opt-out not persisted | `n8n-nodes-base.code` | Logs warning if database missing for opt-out | database connected? | logging the event? | 2. Opt-out is one database call |
| [cred] Supabase - stop all outreach | `n8n-nodes-base.httpRequest` | Executes transactional opt-out RPC in Supabase | database connected? | logging the event? | 2. Opt-out is one database call |
| qualify the candidate | `n8n-nodes-base.code` | Evaluates answers against QUALIFY rules | opted out? | qualified? | 3. The AI fills in the blanks |
| qualified? | `n8n-nodes-base.if` | Routes based on qualification pass mark | qualify the candidate | pick the recruiter and slot, STOP: not qualified | 3. The AI fills in the blanks |
| STOP: not qualified | `n8n-nodes-base.code` | Prepares ATS payload for non-qualified candidates | qualified? | one outcome | 3. The AI fills in the blanks |
| pick the recruiter and slot | `n8n-nodes-base.code` | Chooses recruiter and next open calendar slot | qualified? | can we book? | 4. Book the interview |
| can we book? | `n8n-nodes-base.if` | Verifies calendar availability and live writeback mode | pick the recruiter and slot | [cred] Google Calendar - book the interview, STOP: qualified - not booked | 4. Book the interview |
| [cred] Google Calendar - book the interview | `n8n-nodes-base.googleCalendar` | Books interview event in Google Calendar | can we book? | [cred] Supabase - record the appointment | 4. Book the interview |
| [cred] Supabase - record the appointment | `n8n-nodes-base.httpRequest` | Records appointment in Supabase | [cred] Google Calendar - book the interview | one outcome | 4. Book the interview |
| STOP: qualified - not booked | `n8n-nodes-base.code` | Handles qualified candidate without calendar booking | can we book? | one outcome | 4. Book the interview |
| one outcome | `n8n-nodes-base.merge` | Merges booked and unbooked qualified paths | [cred] Supabase - record the appointment, STOP: qualified - not booked | the write-back payload | 4. Book the interview |
| the write-back payload | `n8n-nodes-base.code` | Constructs ATS note, status update, and notification payloads | one outcome | writing back? | 5. Write back to the ATS |
| writing back? | `n8n-nodes-base.if` | Checks if live writebacks are permitted | the write-back payload | [cred] Supabase - queue the write-back, STOP: write-back skipped | 5. Write back to the ATS |
| [cred] Supabase - queue the write-back | `n8n-nodes-base.httpRequest` | Queues ATS write-back row in Supabase | writing back? | [cred] ATS - add the note | 5. Write back to the ATS |
| [cred] ATS - add the note | `n8n-nodes-base.httpRequest` | Posts note to external ATS REST API | [cred] Supabase - queue the write-back | tell the recruiter? | 5. Write back to the ATS |
| STOP: write-back skipped | `n8n-nodes-base.code` | Simulates write-back payloads in non-live modes | writing back? | tell the recruiter? | 5. Write back to the ATS |
| tellslack? | `n8n-nodes-base.if` | Checks if Slack alerting is enabled | [cred] ATS - add the note, STOP: write-back skipped | Post the recruiter alert, tellmail? | 6. Alert, log and respond |
| Post the recruiter alert | `n8n-nodes-base.httpRequest` | Sends markdown alert to Slack webhook | tellslack? | tellmail? | 6. Alert, log and respond |
| tellmail? | `n8n-nodes-base.if` | Checks if email notifications are enabled | tellslack?, Post the recruiter alert | [cred] Gmail - tell the recruiter, dolog? | 6. Alert, log and respond |
| [cred] Gmail - tell the recruiter | `n8n-nodes-base.gmail` | Sends email notification to recruiter | tellmail? | dolog? | 6. Alert, log and respond |
| dolog? | `n8n-nodes-base.if` | Checks if event logging is enabled | STOP: event ignored, tellmail?, [cred] Gmail - tell the recruiter | [cred] Supabase - log the event, resp | 6. Alert, log and respond |
| [cred] Supabase - log the event | `n8n-nodes-base.httpRequest` | Logs inbound event to Supabase | dolog? | resp | 6. Alert, log and respond |
| resp | `n8n-nodes-base.code` | Formats final HTTP response and TwiML body | STOP: signature rejected, dolog?, [cred] Supabase - log the event | Respond to the provider | 6. Alert, log and respond |
| Respond to the provider | `n8n-nodes-base.respondToWebhook` | Returns HTTP response to Twilio or Retell | resp | None | 6. Alert, log and respond |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Create a **Webhook** node named `Inbound webhook`.
   - Set HTTP Method to `POST`, path to `stratton-inbound`, and Response Mode to `Response node`.
   - Connect it to a **Code** node named `config`. Paste the global configuration object containing client settings, Supabase URLs, ATS field mappings, qualification rules, and safety toggles (`DEMO_MODE`, `TEST_RUN`).

2. **Add Signature Verification:**
   - Create a **Code** node named `verify the signature`. Implement cryptographic HMAC validation for `X-Twilio-Signature` (SHA-1) and `X-Retell-Signature` (SHA-256).
   - Connect to an **If** node named `signature ok?` evaluating `{{ $json._allow }}` equals `true`.
   - On the `false` branch, add a **Code** node named `STOP: signature rejected` to return status 403. Route it to the webhook response node.

3. **Parse Events & Handle Opt-Outs:**
   - On the `true` branch of `signature ok?`, create a **Code** node named `read the event` to normalize Twilio SMS, Twilio status callbacks, and Retell call analysis payloads.
   - Connect to an **If** node named `actionable?` evaluating `{{ $json._understood && $json._stop_sequence === true }}`.
   - On the `false` branch, add `STOP: event ignored`.
   - On the `true` branch, add an **If** node named `opted out?` evaluating `{{ $json.intent === 'opt_out' }}`.
   - For opt-outs, route to a **Code** node (`handle the opt-out`), then an **If** node (`database connected?`), and finally an **HTTP Request** node (`[cred] Supabase - stop all outreach`) configured to call the `stop_all_outreach` RPC endpoint using Supabase service key credentials.

4. **Implement Candidate Qualification:**
   - On the non-opt-out branch of `opted out?`, add a **Code** node named `qualify the candidate` to score answers against `QUALIFY` rules.
   - Connect to an **If** node named `qualified?` evaluating `{{ $json._qualified }}`.
   - On the `false` branch, add `STOP: not qualified` and route it to input index 1 of a **Merge** node (`one outcome`).

5. **Configure Interview Scheduling:**
   - On the `true` branch of `qualified?`, add a **Code** node named `pick the recruiter and slot` to compute available interview times.
   - Connect to an **If** node named `can we book?` evaluating `{{ calendar_enabled && writeback_enabled && !$json._recruiter_missing }}`.
   - On the `true` branch, add a **Google Calendar** node (`[cred] Google Calendar - book the interview`) using Google OAuth2 credentials, followed by an **HTTP Request** node (`[cred] Supabase - record the appointment`) to log the appointment in Supabase. Route the output to input index 0 of `one outcome`.
   - On the `false` branch of `can we book?`, add `STOP: qualified - not booked` and route it to input index 1 of `one outcome`.

6. **Build ATS Write-Back:**
   - Connect `one outcome` to a **Code** node named `the write-back payload` to construct idempotent note and status update payloads.
   - Connect to an **If** node named `writing back?` evaluating `{{ $json._write }}`.
   - On the `true` branch, add an **HTTP Request** node (`[cred] Supabase - queue the write-back`), followed by an **HTTP Request** node (`[cred] ATS - add the note`) using ATS bearer tokens.
   - On the `false` branch, add `STOP: write-back skipped`. Both paths converge on the notification checks.

7. **Configure Notifications, Logging, & Response:**
   - Add an **If** node named `tellslack?` evaluating `{{ slack_enabled }}`. If true, post to Slack via an **HTTP Request** node (`Post the recruiter alert`).
   - Add an **If** node named `tellmail?` evaluating email notification rules. If true, send emails using a **Gmail** node (`[cred] Gmail - tell the recruiter`) with OAuth2 credentials.
   - Add an **If** node named `dolog?` evaluating `{{ supabase_enabled }}`. If true, log inbound events via an **HTTP Request** node (`[cred] Supabase - log the event`).
   - Route all paths to a **Code** node named `resp` to prepare provider-specific response bodies and status codes.
   - Conclude with a **Respond to Webhook** node (`Respond to the provider`) to complete the execution cycle.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Part 1 of 3 Candidate Engine Workflow | [Score ATS candidates and enroll top matches into Supabase outreach campaigns](https://n8n.io/workflows/19775-score-ats-candidates-and-enroll-top-matches-into-supabase-outreach-campaigns/) |
| Part 2 of 3 Candidate Engine Workflow | [Send scheduled SMS and voice outreach from Supabase via Twilio and Retell](https://n8n.io/workflows/19776-send-scheduled-sms-and-voice-outreach-from-supabase-via-twilio-and-retell/) |