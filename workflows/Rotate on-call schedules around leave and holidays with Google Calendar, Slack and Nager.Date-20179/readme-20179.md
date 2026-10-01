Rotate on-call schedules around leave and holidays with Google Calendar, Slack and Nager.Date

https://n8nworkflows.xyz/workflows/rotate-on-call-schedules-around-leave-and-holidays-with-google-calendar--slack-and-nager-date-20179


# Rotate on-call schedules around leave and holidays with Google Calendar, Slack and Nager.Date

### 1. Workflow Overview

This workflow automates weekly on-call roster management by selecting the fairest available team member based on recent duty history, planned leave, and regional public holidays fetched from the Nager.Date API. It supports scheduled weekly rotations and real-time validation of manual calendar edits (swaps), handles interactive approval requests via Slack, and updates channel topics at handover times.

The execution logic is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Triggers via schedule (Thursdays at 10:00) or Google Calendar event updates, initializing local rotation parameters.
- **1.2 Data Aggregation:** Fetches past on-call events, upcoming leave schedules, and next public holidays.
- **1.3 Decision Engine:** Evaluates fairness, regional holidays, rest periods, and leave blockers to select a candidate or assess a calendar swap.
- **1.4 Action Routing & Escalation:** Directs the decision outcome to booking, escalation, rule violation reversion, or individual swap verification paths.
- **1.5 Swap Verification & Lifecycle:** Handles sequential batching, Slack interactive approval requests, and subsequent calendar/channel updates.
- **1.6 Handover & Announcement:** Posts assignments and updates Slack channel topics at handover time.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
**Overview:** Captures regular weekly execution timings or detects real-time modifications made to calendar-based on-call events, feeding parameter definitions downstream.

**Nodes Involved:**
- `Weekly Announcement Time`
- `On-Call Week Edited`
- `Rota Rules`

**Node Details:**
- **Weekly Announcement Time** (`n8n-nodes-base.scheduleTrigger`)
  - *Role:* Cron-based trigger running weekly on Thursdays at 10:00.
  - *Configuration:* Trigger interval set to weeks on day 4 at hour 10.
  - *Connections:* Outputs to `Rota Rules`.
  - *Edge Cases:* Timezone discrepancies if the host environment is not UTC-synchronized.
- **On-Call Week Edited** (`n8n-nodes-base.googleCalendarTrigger`)
  - *Role:* Polls the Google Calendar for updated events to catch manual shift swaps.
  - *Configuration:* Watches event updates every 5 minutes using Google Calendar OAuth2 credentials.
  - *Connections:* Outputs to `Rota Rules`.
  - *Edge Cases:* Missing calendar ID selection causes polling execution errors.
- **Rota Rules** (`n8n-nodes-base.set`)
  - *Role:* Establishes persistent baseline configuration variables (team roster, regions, Slack IDs, holiday country codes, rest/fairness constraints).
  - *Configuration:* Sets data fields including team member lines (`Name; region; Slack member ID`), `holiday_country`, `rest_weeks`, `fairness_weeks`, `handover_time`, `swap_answer_hours`, and `event_title_prefix`.
  - *Connections:* Input from triggers; outputs to data retrieval nodes (`Read Past On-Call Weeks`, `Read The Leave Calendar`, `Read Public Holidays`).
  - *Edge Cases:* Incorrectly formatted multi-line string syntax or missing Slack member IDs breaks downstream matching.

---

#### 2.2 Data Aggregation
**Overview:** Pulls historical scheduling data, team vacation calendars, and official holiday schedules concurrently to supply raw data to the evaluation engine.

**Nodes Involved:**
- `Read Past On-Call Weeks`
- `Read The Leave Calendar`
- `Read Public Holidays`
- `Combine History, Leave And Holidays`

**Node Details:**
- **Read Past On-Call Weeks** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Retrieves past and future on-call events to evaluate fairness and rest constraints.
  - *Configuration:* Retrieves all events from $now minus fairness weeks plus one up to $now plus 53 weeks. Uses Google Calendar OAuth2 credentials.
  - *Connections:* Input from `Rota Rules`; outputs to `Combine History, Leave And Holidays` (input index 0).
  - *Edge Cases:* Token expiry or unselected calendar IDs.
- **Read The Leave Calendar** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Retrieves team leave events from a dedicated vacation calendar.
  - *Configuration:* Retrieves events spanning from $now minus 1 week to $now plus 53 weeks using Google Calendar OAuth2 credentials.
  - *Connections:* Input from `Rota Rules`; outputs to `Combine History, Leave And Holidays` (input index 1).
  - *Edge Cases:* Mismatched naming formats in leave event summaries prevent string-matching.
- **Read Public Holidays** (`n8n-nodes-base.httpRequest`)
  - *Role:* Fetches upcoming public holidays using the Nager.Date API.
  - *Configuration:* GET request to `https://date.nager.at/api/v3/NextPublicHolidays/{{ $('Rota Rules').first().json.holiday_country }}` with a 20-second timeout, 3 retries, and error continuation.
  - *Connections:* Input from `Rota Rules`; outputs to `Combine History, Leave And Holidays` (input index 2).
  - *Edge Cases:* Network timeouts or invalid country codes cause fallback handling via downstream code.
- **Combine History, Leave And Holidays** (`n8n-nodes-base.merge`)
  - *Role:* Aggregates the three parallel data streams into a single dataset.
  - *Configuration:* Append mode with 3 inputs.
  - *Connections:* Inputs from the three retrieval nodes; output to `Decide The On-Call Week`.

---

#### 2.3 Decision Engine
**Overview:** Processes gathered calendars, leave requests, holidays, and team metadata to select the fairest available person or validate manual shift edits against policy rules.

**Nodes Involved:**
- `Decide The On-Call Week`
- `What Now For The Week?`

**Node Details:**
- **Decide The On-Call Week** (`n8n-nodes-base.code`)
  - *Role:* Complex JavaScript block executing selection algorithms, block checks (leave, public holidays, mandatory rest periods), and swap rule validations.
  - *Configuration:* Custom JavaScript snippet reading global variables and returning standardized state flags (`book`, `escalate`, `revert`, `ask`).
  - *Expressions:* Evaluates team roster strings, calculates Monday-anchored weeks, and verifies matching name strings in calendar descriptions.
  - *Connections:* Input from `Combine History, Leave And Holidays`; outputs to `What Now For The Week?`.
  - *Edge Cases:* Malformed team roster rows or empty API payload responses trigger escalation actions.
- **What Now For The Week?** (`n8n-nodes-base.switch`)
  - *Role:* Routes workflow execution paths based on the evaluated action output flag.
  - *Configuration:* Four routing rules matching string values: `book`, `escalate`, `revert`, `ask`, with an extra fallback output.
  - *Connections:* Input from `Decide The On-Call Week`; outputs respectively to `Book The On-Call Week`, `Escalate: Nobody Available`, `Put The Week Back`, and `Ask One Swap At A Time`.

---

#### 2.4 Action Routing & Escalation
**Overview:** Handles terminal or fallback execution paths when no team member can be assigned or when automated rules reject a manual schedule change.

**Nodes Involved:**
- `Escalate: Nobody Available`
- `Put The Week Back`
- `Explain Why ItWas Put Back`

**Node Details:**
- **Escalate: Nobody Available** (`n8n-nodes-base.slack`)
  - *Role:* Notifies team leadership in Slack when zero team members are available for scheduling.
  - *Configuration:* Posts message to a designated Slack channel via OAuth2.
  - *Connections:* Input from `What Now For The Week?`.
- **Put The Week Back** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Reverts an illegal manual calendar shift edit back to its original assignee.
  - *Configuration:* Updates calendar event summary and description using Google Calendar OAuth2.
  - *Connections:* Input from `What Now For The Week?`; outputs to `Explain Why It Was Put Back`.
- **Explain Why It Was Put Back** (`n8n-nodes-base.slack`)
  - *Role:* Announces rule violations and shift reversions in the main team Slack channel.
  - *Configuration:* Posts message via Slack OAuth2.
  - *Connections:* Input from `Put The Week Back`.

---

#### 2.5 Swap Verification & Lifecycle
**Overview:** Manages interactive swap requests sent directly to target team members in Slack, processes their approvals or denials, and updates calendars accordingly.

**Nodes Involved:**
- `Ask One Swap At A Time`
- `Ask The Other Person`
- `Their Answer?`
- `Confirm The Change`
- `Tell The Team About The Swap`
- `Undo The Declined Swap`
- `Say The Swap Was Declined`

**Node Details:**
- **Ask One Swap At A Time** (`n8n-nodes-base.splitInBatches`)
  - *Role:* Processes batch items sequentially to handle multiple swap verification requests safely.
  - *Configuration:* Batch size set to 1.
  - *Connections:* Input from `What Now For The Week?` (and loopback); outputs to `Ask The Other Person`.
- **Ask The Other Person** (`n8n-nodes-base.slack`)
  - *Role:* Sends an interactive approval message to the proposed replacement team member in Slack.
  - *Configuration:* Uses sendAndWait operation with a double approval layout ("I take it" / "No") and timeout bounds determined by `swap_answer_hours`. Slack OAuth2 authentication.
  - *Connections:* Input from batch node; outputs to `Their Answer?`.
  - *Edge Cases:* Member lacks a valid Slack member ID, causing delivery failures.
- **Their Answer?** (`n8n-nodes-base.switch`)
  - *Role:* Evaluates whether the requested user approved or declined/ignored the swap offer.
  - *Configuration:* Evaluates boolean evaluation of `$json.data.approved === true`.
  - *Connections:* Input from `Ask The Other Person`; true output links to `Confirm The Change`, false/fallback links to `Undo The Declined Swap`.
- **Confirm The Change** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Updates the calendar event description with the new verification marker upon approval.
  - *Configuration:* Updates event using Google Calendar OAuth2.
  - *Connections:* Input from `Their Answer?` (Confirmed); outputs to `Tell The Team About The Swap`.
- **Tell The Team About The Swap** (`n8n-nodes-base.slack`)
  - *Role:* Posts a successful shift-swap confirmation notice to the main team channel.
  - *Configuration:* Posts message via Slack OAuth2, looping back to process remaining batches.
  - *Connections:* Input from `Confirm The Change`; outputs to `Ask One Swap At A Time`.
- **Undo The Declined Swap** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Reverts the calendar event back to the original assignee when a swap is declined or times out.
  - *Configuration:* Updates event summary and description via Google Calendar OAuth2.
  - *Connections:* Input from `Their Answer?` (Declined); outputs to `Say The Swap Was Declined`.
- **Say The Swap Was Declined** (`n8n-nodes-base.slack`)
  - *Role:* Notifies the team channel that a requested swap was rejected or expired.
  - *Configuration:* Posts message via Slack OAuth2, looping back to process remaining batches.
  - *Connections:* Input from `Undo The Declined Swap`; outputs to `Ask One Swap At A Time`.

---

#### 2.6 Handover & Announcement
**Overview:** Books verified weekly assignments into Google Calendar, broadcasts public assignments, and schedules channel topic updates at handover time.

**Nodes Involved:**
- `Book The On-Call Week`
- `Announce Next On-Call`
- `Wait For The Handover`
- `Update The Channel Topic`

**Node Details:**
- **Book The On-Call Week** (`n8n-nodes-base.googleCalendar`)
  - *Role:* Creates a new all-day on-call calendar event for the selected week.
  - *Configuration:* Creates all-day event using start/end dates, dynamic summary title, and marker description via Google Calendar OAuth2.
  - *Connections:* Input from `What Now For The Week?` (Book it); outputs to `Announce Next On-Call`.
- **Announce Next On-Call** (`n8n-nodes-base.slack`)
  - *Role:* Posts the weekly on-call assignment announcement to the main team Slack channel.
  - *Configuration:* Posts message via Slack OAuth2.
  - *Connections:* Input from `Book The On-Call Week`; outputs to `Wait For The Handover`.
- **Wait For The Handover** (`n8n-nodes-base.wait`)
  - *Role:* Pauses execution until the configured handover timestamp.
  - *Configuration:* Resumes at specific time parsed from `handover_at`.
  - *Connections:* Input from `Announce Next On-Call`; outputs to `Update The Channel Topic`.
- **Update The Channel Topic** (`n8n-nodes-base.slack`)
  - *Role:* Updates the Slack channel topic to display the current on-call person at handover time.
  - *Configuration:* Sets channel topic via Slack OAuth2.
  - *Connections:* Input from `Wait For The Handover`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Announcement Time | n8n-nodes-base.scheduleTrigger | Cron trigger for weekly rotations | None | Rota Rules | ## 1. Weekly pick, or someone edited a week<br>Every setting lives in Rota Rules. |
| On-Call Week Edited | n8n-nodes-base.googleCalendarTrigger | Real-time trigger for manual swaps | None | Rota Rules | ## 1. Weekly pick, or someone edited a week<br>Every setting lives in Rota Rules. |
| Rota Rules | n8n-nodes-base.set | Defines team rules and roster variables | Weekly Announcement Time, On-Call Week Edited | Read Past On-Call Weeks, Read The Leave Calendar, Read Public Holidays | ## 1. Weekly pick, or someone edited a week<br>Every setting lives in Rota Rules. |
| Read Past On-Call Weeks | n8n-nodes-base.googleCalendar | Fetches historical on-call allocations | Rota Rules | Combine History, Leave And Holidays | ## 2. History, leave and holidays<br>The on-call calendar is the rota's own history. |
| Read The Leave Calendar | n8n-nodes-base.googleCalendar | Fetches planned team leave entries | Rota Rules | Combine History, Leave And Holidays | ## 2. History, leave and holidays<br>The on-call calendar is the rota's own history. |
| Read Public Holidays | n8n-nodes-base.httpRequest | Fetches public holiday API data | Rota Rules | Combine History, Leave And Holidays | ## 2. History, leave and holidays<br>The on-call calendar is the rota's own history. |
| Combine History, Leave And Holidays | n8n-nodes-base.merge | Aggregates data streams | Read Past On-Call Weeks, Read The Leave Calendar, Read Public Holidays | Decide The On-Call Week | ## 2. History, leave and holidays<br>The on-call calendar is the rota's own history. |
| Decide The On-Call Week | n8n-nodes-base.code | Selects assignee or validates edits | Combine History, Leave And Holidays | What Now For The Week? | ## 3. Decide the week<br>Fairest free person, or check an edit against the same rules. Nobody free? The lead is told. |
| What Now For The Week? | n8n-nodes-base.switch | Routes based on action outcomes | Decide The On-Call Week | Book The On-Call Week, Escalate: Nobody Available, Put The Week Back, Ask One Swap At A Time | ## 3. Decide the week<br>Fairest free person, or check an edit against the same rules. Nobody free? The lead is told. |
| Book The On-Call Week | n8n-nodes-base.googleCalendar | Books the new on-call week | What Now For The Week? | Announce Next On-Call | ## 6. Book and announce<br>The channel topic changes at handover time. |
| Announce Next On-Call | n8n-nodes-base.slack | Broadcasts assignment in Slack | Book The On-Call Week | Wait For The Handover | ## 6. Book and announce<br>The channel topic changes at handover time. |
| Wait For The Handover | n8n-nodes-base.wait | Pauses until handover time | Announce Next On-Call | Update The Channel Topic | ## 6. Book and announce<br>The channel topic changes at handover time. |
| Update The Channel Topic | n8n-nodes-base.slack | Updates Slack channel topic | Wait For The Handover | None | ## 6. Book and announce<br>The channel topic changes at handover time. |
| Escalate: Nobody Available | n8n-nodes-base.slack | Alerts lead on vacancy | What Now For The Week? | None | ## 3. Decide the week<br>Fairest free person, or check an edit against the same rules. Nobody free? The lead is told. |
| Put The Week Back | n8n-nodes-base.googleCalendar | Reverts illegal calendar edits | What Now For The Week? | Explain Why It Was Put Back | ## 5. A swap that breaks the rules<br>The week goes back to who had it, and the team hears why. |
| Explain Why It Was Put Back | n8n-nodes-base.slack | Explains reverted swap in channel | Put The Week Back | None | ## 5. A swap that breaks the rules<br>The week goes back to who had it, and the team hears why. |
| Ask One Swap At A Time | n8n-nodes-base.splitInBatches | Batches swap requests | What Now For The Week?, Tell The Team About The Swap, Say The Swap Was Declined | Ask The Other Person | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Ask The Other Person | n8n-nodes-base.slack | Requests confirmation from assignee | Ask One Swap At A Time | Their Answer? | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Their Answer? | n8n-nodes-base.switch | Evaluates approval status | Ask The Other Person | Confirm The Change, Undo The Declined Swap | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Confirm The Change | n8n-nodes-base.googleCalendar | Confirms swap in calendar | Their Answer? | Tell The Team About The Swap | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Tell The Team About The Swap | n8n-nodes-base.slack | Announces successful swap | Confirm The Change | Ask One Swap At A Time | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Undo The Declined Swap | n8n-nodes-base.googleCalendar | Reverts declined/timed-out swap | Their Answer? | Say The Swap Was Declined | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |
| Say The Swap Was Declined | n8n-nodes-base.slack | Announces declined swap | Undo The Declined Swap | Ask One Swap At A Time | ## 4. A swap to check<br>The new person confirms in Slack, one swap at a time. No answer means no. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers & Configuration Nodes:**
   - Add a `Schedule Trigger` named `Weekly Announcement Time`: configure interval for weeks, day 4 (Thursday), hour 10.
   - Add a `Google Calendar Trigger` named `On-Call Week Edited`: configure event monitoring poll interval to 5 minutes for `eventUpdated`. Link Google Calendar OAuth2 credentials.
   - Add a `Set` node named `Rota Rules`: configure fields:
     - `team` (string): `Alex; DE-BY; U+1234567890A\nSam; DE-BE; U+1234567890B\nRobin; DE-NW; U+1234567890C`
     - `holiday_country` (string): `DE`
     - `lead_slack_id` (string): *[Optional user ID]*
     - `rest_weeks` (number): `1`
     - `fairness_weeks` (number): `12`
     - `handover_time` (string): `09:00`
     - `swap_answer_hours` (number): `24`
     - `event_title_prefix` (string): `On call: `
   - Connect both triggers to `Rota Rules`.

2. **Create Data Retrieval Nodes:**
   - Add a `Google Calendar` node named `Read Past On-Call Weeks`: resource `event`, operation `getAll`, `singleEvents` enabled, time limits set relative to fairness parameters. Connect input from `Rota Rules`.
   - Add a `Google Calendar` node named `Read The Leave Calendar` named similarly: resource `event`, operation `getAll`, `singleEvents` enabled. Connect input from `Rota Rules`.
   - Add an `HTTP Request` node named `Read Public Holidays`: GET method, URL `https://date.nager.at/api/v3/NextPublicHolidays/{{ $('Rota Rules').first().json.holiday_country }}`, configure 3 retries with error continuation enabled. Connect input from `Rota Rules`.

3. **Merge and Decide Logic:**
   - Add a `Merge` node named `Combine History, Leave And Holidays`: mode `append`, 3 input ports. Connect outputs from the three data retrieval nodes to inputs 0, 1, and 2 respectively.
   - Add a `Code` node named `Decide The On-Call Week`: paste the JavaScript evaluation logic. Connect input from `Combine History, Leave And Holidays`.
   - Add a `Switch` node named `What Now For The Week?`: configure 4 rules matching `book`, `escalate`, `revert`, and `ask` based on `{{ $json.action }}`. Connect input from `Decide The On-Call Week`.

4. **Build Booking & Escalation Branches:**
   - **Book branch:** Connect output 1 (`Book it`) to a `Google Calendar` node named `Book The On-Call Week` (operation `create`, all-day enabled). Connect its output to a Slack node named `Announce Next On-Call`. Connect to a `Wait` node named `Wait For The Handover` (resume at specific time `{{ $('What Now For The Week?').first().json.handover_at }}`), finishing at a Slack node named `Update The Channel Topic`.
   - **Escalate branch:** Connect output 2 (`Nobody free`) to a Slack node named `Escalate: Nobody Available`.
   - **Revert branch:** Connect output 3 (`Swap breaks the rules`) to a `Google Calendar` node named `Put The Week Back` (operation `update`), connecting onward to a Slack node named `Explain Why It Was Put Back`.

5. **Build Swap Approval Workflow Branch:**
   - Connect output 4 (`Ask about the swap`) to a `Split In Batches` node named `Ask One Swap At A Time` (batch size 1).
   - Connect its loop output to a Slack node named `Ask The Other Person`: resource `message`, operation `sendAndWait`, response type `approval` (double approval), limit timeout using `swap_answer_hours`.
   - Add a `Switch` node named `Their Answer?` to evaluate `{{ !!($json.data && $json.data.approved === true) }}`.
   - **If Approved:** Connect to a `Google Calendar` node named `Confirm The Change`, then to a Slack node named `Tell The Team About The Swap`. Loop back to `Ask One Swap At A Time`.
   - **If Declined/Timeout:** Connect to a `Google Calendar` node named `Undo The Declined Swap`, then to a Slack node named `Say The Swap Was Declined`. Loop back to `Ask One Swap At A Time`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Rotate on-call fairly around leave and public holidays with Google Calendar, Slack and Nager.Date | Workflow purpose overview |
| Leave counts for a person when the leave event's title contains their name (e.g., "Robin holiday") | Event matching rule |