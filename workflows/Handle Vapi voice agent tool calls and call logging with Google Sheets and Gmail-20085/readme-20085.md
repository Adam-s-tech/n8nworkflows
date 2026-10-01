Handle Vapi voice agent tool calls and call logging with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/handle-vapi-voice-agent-tool-calls-and-call-logging-with-google-sheets-and-gmail-20085


# Handle Vapi voice agent tool calls and call logging with Google Sheets and Gmail

### 1. Workflow Overview

This workflow acts as a comprehensive backend webhook for Vapi voice agents. It handles two primary operational categories: mid-call tool execution (fetching calendar availability, querying CRM records, and logging requests) and post-call analytics (saving conversation transcripts, scanning summaries for warning keywords, and triggering team alerts via email).

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Normalization:** Receives incoming HTTP POST payloads from the voice platform, establishes global environment configurations, and standardizes raw structures into clean, predictable variables.
- **1.2 Event & Function Routing:** Evaluates the standardized event type (`tool-call` vs. `end-of-call`) and routes live tool calls to specific business logic handlers based on their function name.
- **1.3 Tool Handlers & Execution:** Executes external platform calls (Google Calendar, Google Sheets) or handles unknown function definitions (such as call transfers), formatting outputs into Vapi's required JSON response schema.
- **1.4 Post-Call Processing & Alerting:** Processes end-of-call data by logging full call records to Google Sheets, evaluating text transcripts and summaries against flagged keywords, and conditionally dispatching team alert emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization

- **Overview:** This block acts as the entry point for all Vapi communications. It ingests webhook requests, injects user-defined configuration parameters, and normalizes disparate payload schemas into a unified object.
- **Nodes Involved:** 
  - `Voice Platform Webhook`
  - `Agent Config`
  - `Normalize Request`
  - `Route By Event`

- **Node Details:**
  - **Voice Platform Webhook**
    - *Type and technical role:* Webhook trigger (`n8n-nodes-base.webhook`). Acts as the HTTP POST endpoint listening for incoming voice agent events.
    - *Configuration choices:* Path set to `voice-agent`, HTTP method `POST`, response mode configured via downstream nodes.
    - *Input/Output connections:* Input: None (Trigger). Output: `Agent Config`.
    - *Edge cases/Failure types:* Missing incoming data or invalid JSON payloads will cause downstream parsing to fail gracefully if fields are undefined.
  - **Agent Config**
    - *Type and technical role:* Set node (`n8n-nodes-base.set`). Defines environment-wide variables such as Calendar ID, alert email recipient, slot duration, transfer messages, and follow-up keywords.
    - *Configuration choices:* Sets static string assignments for configuration variables.
    - *Input/Output connections:* Input: `Voice Platform Webhook`. Output: `Normalize Request`.
    - *Edge cases/Failure types:* Placeholder text values (`REPLACE_WITH_CALENDAR_ID_OR_EMAIL`, `user@example.com`) must be manually updated before production use.
  - **Normalize Request**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Flattens and parses incoming nested Vapi JSON structures.
    - *Key expressions or variables:* Extracts `msg.type`, `msg.toolCallList`, `msg.call`, `msg.artifact`, and `msg.analysis`. Converts stringified JSON arguments into JavaScript objects safely using try-catch blocks.
    - *Input/Output connections:* Input: `Agent Config`. Output: `Route By Event`.
    - *Edge cases/Failure types:* Malformed argument strings are caught and defaulted to empty objects to prevent execution halts.
  - **Route By Event**
    - *Type and technical role:* Switch node (`n8n-nodes-base.switch`). Branches workflow execution based on the normalized `eventType`.
    - *Configuration choices:* Routes `tool-call` to output 0, `end-of-call` to output 1, and any unmatched events to the fallback output.
    - *Input/Output connections:* Input: `Normalize Request`. Output: `Route By Function` (tool call), `Extract Call Record` (end of call), and `Unknown Function` (fallback).

---

#### 2.2 Event & Function Routing

- **Overview:** Inspects mid-call tool requests and directs them to the appropriate functional handler or routes unmatched functions safely.
- **Nodes Involved:** 
  - `Route By Function`

- **Node Details:**
  - **Route By Function**
    - *Type and technical role:* Switch node (`n8n-nodes-base.switch`). Directs tool calls by comparing `functionName`.
    - *Configuration choices:* Case-insensitive matching rules for `check_availability`, `lookup_customer`, and `log_request`. Fallback configured for unrecognized function names.
    - *Input/Output connections:* Input: `Route By Event`. Output: `Check Calendar Availability`, `Look Up Customer`, `Log Request`, or `Unknown Function`.

---

#### 2.3 Tool Handlers & Execution

- **Overview:** Processes individual mid-call actions (calendar lookups, customer CRM lookups, request logging, or unknown fallback handling) and standardizes all responses into Vapi's required format.
- **Nodes Involved:** 
  - `Check Calendar Availability`
  - `Format Availability`
  - `Look Up Customer`
  - `Format Customer`
  - `Log Request`
  - `Format Confirmation`
  - `Unknown Function`
  - `Respond To Tool Call`

- **Node Details:**
  - **Check Calendar Availability**
    - *Type and technical role:* Google Calendar node (`n8n-nodes-base.googleCalendar`). Fetches upcoming calendar events to calculate availability slots.
    - *Configuration choices:* Operation: `getAll`, ordered by `startTime`, with the calendar ID dynamically read from `{{ $('Agent Config').first().json.CalendarId }}`. Error handling set to continue regular output.
    - *Input/Output connections:* Input: `Route By Function`. Output: `Format Availability`.
    - *Edge cases/Failure types:* Authentication expiry or invalid Calendar IDs will output error objects, which are filtered out in the subsequent formatting code node.
  - **Format Availability**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Filters busy periods against operational hours (9 AM – 5 PM, next 5 days) to calculate open slots and generate spoken text strings.
    - *Input/Output connections:* Input: `Check Calendar Availability`. Output: `Respond To Tool Call`.
  - **Look Up Customer**
    - *Type and technical role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Retrieves rows from the "Customers" sheet tab.
    - *Configuration choices:* Document ID points to `YOUR_SPREADSHEET_ID`, sheet name set to `Customers`. Error handling set to continue regular output.
    - *Input/Output connections:* Input: `Route By Function`. Output: `Format Customer`.
  - **Format Customer**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Normalizes customer phone numbers, compares caller numbers against sheet records using suffix matching, and builds a response text string.
    - *Input/Output connections:* Input: `Look Up Customer`. Output: `Respond To Tool Call`.
  - **Log Request**
    - *Type and technical role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Appends new rows to the "Requests" sheet tab.
    - *Configuration choices:* Operation: `append`. Mapped columns include `Date`, `Caller`, `Call ID`, and `Request`. Error handling set to continue regular output.
    - *Input/Output connections:* Input: `Route By Function`. Output: `Format Confirmation`.
  - **Format Confirmation**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Evaluates whether the log insertion failed or succeeded, outputting a clear natural-language confirmation string for the agent.
    - *Input/Output connections:* Input: `Log Request`. Output: `Respond To Tool Call`.
  - **Unknown Function**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Handles unsupported or missing function names, including custom call transfers (`transfer_call`).
    - *Input/Output connections:* Input: `Route By Function`. Output: `Respond To Tool Call`.
  - **Respond To Tool Call**
    - *Type and technical role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns the tool execution result back to the voice platform.
    - *Key expressions or variables:* `={{ JSON.stringify({ results: [ { toolCallId: $json.toolCallId, result: $json.result } ] }) }}`
    - *Input/Output connections:* Input: `Format Availability`, `Format Customer`, `Format Confirmation`, or `Unknown Function`. Output: None (Terminal node).

---

#### 2.4 Post-Call Processing & Alerting

- **Overview:** Executes tasks after a voice call concludes by storing call metrics and transcripts in Google Sheets, scanning text for warning triggers, and alerting human teams via email if necessary.
- **Nodes Involved:** 
  - `Extract Call Record`
  - `Save Call Record`
  - `Needs Follow Up?`
  - `Alert Team`
  - `Acknowledge End Of Call`

- **Node Details:**
  - **Extract Call Record**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Parses call transcripts and summaries, performs case-insensitive keyword searches against configured follow-up triggers, and flags calls requiring intervention.
    - *Input/Output connections:* Input: `Route By Event`. Output: `Save Call Record`.
  - **Save Call Record**
    - *Type and technical role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Appends complete call metadata to the "Calls" sheet tab.
    - *Configuration choices:* Mapped columns: `Date`, `Flags`, `Caller`, `Call ID`, `Summary`, `Duration`, and `Ended Reason`. Error handling set to continue regular output.
    - *Input/Output connections:* Input: `Extract Call Record`. Output: `Needs Follow Up?`.
  - **Needs Follow Up?**
    - *Type and technical role:* If node (`n8n-nodes-base.if`). Evaluates whether `needsFollowUp` is true.
    - *Input/Output connections:* Input: `Save Call Record`. Output: True path (`Alert Team`), False path (`Acknowledge End Of Call`).
  - **Alert Team**
    - *Type and technical role:* Gmail node (`n8n-nodes-base.gmail`). Sends an alert email containing call details, flags, summary, and transcript to the designated support address.
    - *Configuration choices:* Recipient set to `={{ $('Agent Config').first().json.AlertEmail }}`, with subject and message bodies dynamically populated from extracted call metrics. Error handling set to continue regular output.
    - *Input/Output connections:* Input: `Needs Follow Up?` (True). Output: `Acknowledge End Of Call`.
  - **Acknowledge End Of Call**
    - *Type and technical role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Sends an acknowledgment confirmation back to Vapi indicating receipt of the end-of-call report.
    - *Key expressions or variables:* `={{ JSON.stringify({ received: true }) }}`
    - *Input/Output connections:* Input: `Needs Follow Up?` (False) or `Alert Team`. Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Voice Platform Webhook | n8n-nodes-base.webhook | HTTP endpoint receiving Vapi payloads | None | Agent Config | ## Voice AI Agent Backend for Vapi<br>The missing half of every voice agent build. Vapi handles the speech - this handles the business logic behind it: live tool calls during the call, and the record-keeping after it.<br><br>## How it works<br>1. Vapi POSTs to one webhook for everything<br>2. Normalize Request works out whether this is a mid-call tool call or an end-of-call report<br>3. **Tool calls** route by function name to a real handler - calendar availability, customer lookup, logging a request - and each returns Vapi's exact response shape so the caller hears an answer instead of silence<br>4. **End-of-call** saves the transcript and summary, scans for words that mean trouble, and emails you only when a call actually needs a human<br><br>## Setup<br>- [ ] Connect Google Calendar, Google Sheets and Gmail credentials<br>- [ ] Open Agent Config and set your calendar ID, alert email, slot length, and the keywords that should flag a call<br>- [ ] Create a spreadsheet with three tabs:<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Customers** - Name, Phone, Status, Last Contact, Notes<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Requests** - Date, Call ID, Caller, Request<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Calls** - Date, Call ID, Caller, Duration, Ended Reason, Summary, Flags<br>- [ ] Point all three Google Sheets nodes at that spreadsheet and the matching tab<br>- [ ] Activate, copy the production webhook URL, and paste it as the Server URL on your Vapi assistant<br>- [ ] In Vapi, define tools named check_availability, lookup_customer and log_request<br><br>## Customize<br>Add a tool by adding an output to Route By Function and a handler that returns { toolCallId, result }. Everything converges on Respond To Tool Call, so new tools need no rewiring beyond that. |
| Agent Config | n8n-nodes-base.set | Defines configuration parameters | Voice Platform Webhook | Normalize Request | ## Voice AI Agent Backend for Vapi<br>The missing half of every voice agent build. Vapi handles the speech - this handles the business logic behind it: live tool calls during the call, and the record-keeping after it.<br><br>## How it works<br>1. Vapi POSTs to one webhook for everything<br>2. Normalize Request works out whether this is a mid-call tool call or an end-of-call report<br>3. **Tool calls** route by function name to a real handler - calendar availability, customer lookup, logging a request - and each returns Vapi's exact response shape so the caller hears an answer instead of silence<br>4. **End-of-call** saves the transcript and summary, scans for words that mean trouble, and emails you only when a call actually needs a human<br><br>## Setup<br>- [ ] Connect Google Calendar, Google Sheets and Gmail credentials<br>- [ ] Open Agent Config and set your calendar ID, alert email, slot length, and the keywords that should flag a call<br>- [ ] Create a spreadsheet with three tabs:<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Customers** - Name, Phone, Status, Last Contact, Notes<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Requests** - Date, Call ID, Caller, Request<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Calls** - Date, Call ID, Caller, Duration, Ended Reason, Summary, Flags<br>- [ ] Point all three Google Sheets nodes at that spreadsheet and the matching tab<br>- [ ] Activate, copy the production webhook URL, and paste it as the Server URL on your Vapi assistant<br>- [ ] In Vapi, define tools named check_availability, lookup_customer and log_request<br><br>## Customize<br>Add a tool by adding an output to Route By Function and a handler that returns { toolCallId, result }. Everything converges on Respond To Tool Call, so new tools need no rewiring beyond that. |
| Normalize Request | n8n-nodes-base.code | Flattens and parses incoming webhook JSON payloads | Agent Config | Route By Event | ## 1. Intake<br>One webhook receives every event type. Normalize flattens the payload and Route By Event splits live tool calls from post-call reports. |
| Route By Event | n8n-nodes-base.switch | Routes execution based on event type | Normalize Request | Route By Function, Extract Call Record, Unknown Function | ## 1. Intake<br>One webhook receives every event type. Normalize flattens the payload and Route By Event splits live tool calls from post-call reports. |
| Route By Function | n8n-nodes-base.switch | Routes tool calls based on function name | Route By Event | Check Calendar Availability, Look Up Customer, Log Request, Unknown Function | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format. |
| Check Calendar Availability | n8n-nodes-base.googleCalendar | Fetches calendar events | Route By Function | Format Availability | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format.<br><br>**Latency is the whole game.** The caller is waiting in silence while this runs. Keep handlers under ~1 second - avoid slow lookups, big sheets, and extra AI calls inside a tool handler. Every handler here fails soft and still returns a sentence the agent can say out loud, because a timeout sounds like a dropped call. |
| Format Availability | n8n-nodes-base.code | Calculates open time slots and formats spoken text | Check Calendar Availability | Respond To Tool Call | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format. |
| Look Up Customer | n8n-nodes-base.googleSheets | Retrieves customer records from spreadsheet | Route By Function | Format Customer | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format.<br><br>**Latency is the whole game.** The caller is waiting in silence while this runs. Keep handlers under ~1 second - avoid slow lookups, big sheets, and extra AI calls inside a tool handler. Every handler here fails soft and still returns a sentence the agent can say out loud, because a timeout sounds like a dropped call. |
| Format Customer | n8n-nodes-base.code | Matches phone numbers against customer sheet rows | Look Up Customer | Respond To Tool Call | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format. |
| Log Request | n8n-nodes-base.googleSheets | Appends requests to spreadsheet | Route By Function | Format Confirmation | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format.<br><br>**Latency is the whole game.** The caller is waiting in silence while this runs. Keep handlers under ~1 second - avoid slow lookups, big sheets, and extra AI calls inside a tool handler. Every handler here fails soft and still returns a sentence the agent can say out loud, because a timeout sounds like a dropped call. |
| Format Confirmation | n8n-nodes-base.code | Formats confirmation text for tool execution | Log Request | Respond To Tool Call | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format. |
| Unknown Function | n8n-nodes-base.code | Handles unknown functions and call transfers | Route By Function | Respond To Tool Call | ## 2. Tool Handlers - answered live, mid-call<br>Each function gets its own handler. Unrecognised functions still get a usable answer instead of an error. All four paths converge on one response node in Vapi's required format. |
| Respond To Tool Call | n8n-nodes-base.respondToWebhook | Returns structured JSON response to Vapi | Format Availability, Format Customer, Format Confirmation, Unknown Function | None | ## Using Retell, Bland or another platform?<br>Two nodes are the adapter layer. **Normalize Request** reads Vapi's payload shape - add your platform's shape there. **Respond To Tool Call** emits Vapi's `{ results: [...] }` format - change it to match your platform's expected response. Everything between them is platform-agnostic. |
| Extract Call Record | n8n-nodes-base.code | Parses transcripts and evaluates follow-up keywords | Route By Event | Save Call Record | ## 3. After The Call<br>Every call is saved. Only flagged ones email you - the alert is for calls that need a human, not a notification per call. |
| Save Call Record | n8n-nodes-base.googleSheets | Appends call metrics and summaries to spreadsheet | Extract Call Record | Needs Follow Up? | ## 3. After The Call<br>Every call is saved. Only flagged ones email you - the alert is for calls that need a human, not a notification per call.<br><br>**Latency is the whole game.** The caller is waiting in silence while this runs. Keep handlers under ~1 second - avoid slow lookups, big sheets, and extra AI calls inside a tool handler. Every handler here fails soft and still returns a sentence the agent can say out loud, because a timeout sounds like a dropped call. |
| Needs Follow Up? | n8n-nodes-base.if | Evaluates whether call requires team intervention | Save Call Record | Alert Team, Acknowledge End Of Call | ## 3. After The Call<br>Every call is saved. Only flagged ones email you - the alert is for calls that need a human, not a notification per call. |
| Alert Team | n8n-nodes-base.gmail | Sends alert email for flagged calls | Needs Follow Up? | Acknowledge End Of Call | ## 3. After The Call<br>Every call is saved. Only flagged ones email you - the alert is for calls that need a human, not a notification per call.<br><br>**Latency is the whole game.** The caller is waiting in silence while this runs. Keep handlers under ~1 second - avoid slow lookups, big sheets, and extra AI calls inside a tool handler. Every handler here fails soft and still returns a sentence the agent can say out loud, because a timeout sounds like a dropped call. |
| Acknowledge End Of Call | n8n-nodes-base.respondToWebhook | Sends confirmation receipt for post-call reports | Alert Team, Needs Follow Up? | None | ## 3. After The Call<br>Every call is saved. Only flagged ones email you - the alert is for calls that need a human, not a notification per call. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger**
   - Add a **Webhook** node named `Voice Platform Webhook`.
   - Set **HTTP Method** to `POST` and **Path** to `voice-agent`. Set **Response Mode** to `Response Node`.

2. **Configure Environment Variables**
   - Add a **Set** node named `Agent Config` connected to the webhook output.
   - Add the following string assignments:
     - `CalendarId`: `REPLACE_WITH_CALENDAR_ID_OR_EMAIL`
     - `AlertEmail`: `user@example.com`
     - `SlotLengthMinutes`: `30`
     - `TransferMessage`: `Transferring you to a team member now, one moment.`
     - `FollowUpKeywords`: `complaint, refund, cancel, angry, legal, escalate`

3. **Normalize Incoming Payloads**
   - Add a **Code** node named `Normalize Request` connected to `Agent Config`.
   - Insert JavaScript to safely parse nested webhook bodies, extracting `eventType`, `toolCallId`, `functionName`, `args`, `callId`, `customerNumber`, `transcript`, `summary`, `endedReason`, and `durationSeconds`.

4. **Set Up Main Event Routing**
   - Add a **Switch** node named `Route By Event` connected to `Normalize Request`.
   - Configure rules to branch based on `{{ $json.eventType }}` matching `tool-call` (Output 0) and `end-of-call` (Output 1), with a fallback for extra types.

5. **Configure Function Routing**
   - Add a **Switch** node named `Route By Function` connected to output 0 of `Route By Event`.
   - Configure case-insensitive matching rules for `check_availability`, `lookup_customer`, and `log_request`, with a fallback output for unrecognized functions.

6. **Build Tool Handler: Check Calendar Availability**
   - Add a **Google Calendar** node named `Check Calendar Availability` connected to the first output of `Route By Function`.
   - Set operation to `getAll`, set calendar configuration to ID mode referencing `={{ $('Agent Config').first().json.CalendarId }}`, and configure error handling to `continueRegularOutput`.
   - Connect it to a **Code** node named `Format Availability` that parses open time slots and formats spoken responses.

7. **Build Tool Handler: Look Up Customer**
   - Add a **Google Sheets** node named `Look Up Customer` connected to the second output of `Route By Function`.
   - Set document ID to your spreadsheet (`YOUR_SPREADSHEET_ID`), select sheet name `Customers`, and set error handling to `continueRegularOutput`.
   - Connect it to a **Code** node named `Format Customer` that matches caller numbers against customer rows.

8. **Build Tool Handler: Log Request**
   - Add a **Google Sheets** node named `Log Request` connected to the third output of `Route By Function`.
   - Set operation to `append`, document ID to `YOUR_SPREADSHEET_ID`, select sheet name `Requests`, and map columns (`Date`, `Caller`, `Call ID`, `Request`). Set error handling to `continueRegularOutput`.
   - Connect it to a **Code** node named `Format Confirmation` that outputs a success or failure text response.

9. **Build Tool Handler: Unknown Function / Transfer**
   - Add a **Code** node named `Unknown Function` connected to the fallback output of `Route By Function`.
   - Handle custom fallback messages and check for a `transfer_call` function name mapping to `TransferMessage`.

10. **Aggregate Tool Responses**
    - Connect `Format Availability`, `Format Customer`, `Format Confirmation`, and `Unknown Function` outputs to a single **Respond to Webhook** node named `Respond To Tool Call`.
    - Set response body to return Vapi's required JSON shape: `={{ JSON.stringify({ results: [ { toolCallId: $json.toolCallId, result: $json.result } ] }) }}`.

11. **Build Post-Call Pipeline**
    - Add a **Code** node named `Extract Call Record` connected to the second output of `Route By Event`. It scans transcripts and summaries for follow-up keywords.
    - Connect it to a **Google Sheets** node named `Save Call Record`. Set operation to `append`, document ID to `YOUR_SPREADSHEET_ID`, select sheet name `Calls`, and map call metrics columns. Set error handling to `continueRegularOutput`.
    - Connect it to an **If** node named `Needs Follow Up?` checking if `needsFollowUp` is true.

12. **Configure Alerting & Acknowledgment**
    - On the true branch of `Needs Follow Up?`, add a **Gmail** node named `Alert Team`. Set recipient to `={{ $('Agent Config').first().json.AlertEmail }}`, configure subject and message bodies, and set error handling to `continueRegularOutput`.
    - Connect both the false branch of `Needs Follow Up?` and the output of `Alert Team` to a **Respond to Webhook** node named `Acknowledge End Of Call` responding with `={{ JSON.stringify({ received: true }) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Setup Requirements | Create a spreadsheet with three tabs containing exact headers:<br>- **Customers:** `Name`, `Phone`, `Status`, `Last Contact`, `Notes`<br>- **Requests:** `Date`, `Call ID`, `Caller`, `Request`<br>- **Calls:** `Date`, `Call ID`, `Caller`, `Duration`, `Ended Reason`, `Summary`, `Flags` |
| Vapi Assistant Configuration | Copy the production webhook URL from the `Voice Platform Webhook` node and paste it as the Server URL in your Vapi assistant settings. Define tools named `check_availability`, `lookup_customer`, `log_request`, and optionally `transfer_call` in Vapi. |
| Latency Considerations | Tool handlers execute while the caller waits in silence. Keep database lookups fast and avoid external AI calls inside handlers to prevent timeouts that sound like dropped calls. |