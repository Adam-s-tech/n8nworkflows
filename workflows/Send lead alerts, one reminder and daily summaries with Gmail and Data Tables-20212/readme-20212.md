Send lead alerts, one reminder and daily summaries with Gmail and Data Tables

https://n8nworkflows.xyz/workflows/send-lead-alerts--one-reminder-and-daily-summaries-with-gmail-and-data-tables-20212


# Send lead alerts, one reminder and daily summaries with Gmail and Data Tables

### 1. Workflow Overview

This workflow is designed for small businesses and service teams to capture website leads via webhook, deduplicate and store them in an n8n Data Table, send a real-time email alert to the owner with an interactive “mark as answered” link, deliver a single conditional reminder if the lead remains unaddressed, and compile an automated evening summary report. 

The logic branches from two distinct triggers into two parallel execution paths:
- **1.1 Lead Ingestion & Processing:** Receives form submissions, normalizes and hashes contact information, checks for duplicates within a specified time window, stores new leads, alerts the owner via Gmail, and pauses execution to wait for human interaction or a timeout.
- **1.2 Evening Reporting:** Runs on a daily schedule at 20:00 to query the Data Table, calculate daily metrics (received, answered in time, open, reminded), and email a summary report.

---

### 2. Block-by-Block Analysis

---

#### Block 1: Input Reception & Settings Initialization
**Overview:** Captures incoming lead data via HTTP POST or initiates a scheduled daily report, while initializing global workflow settings such as owner email, reminder intervals, repeat windows, and timezones.

**Nodes Involved:**
- `A lead arrives`
- `Every evening at 20:00`
- `Your settings`
- `A new lead? (otherwise the evening summary)`

**Node Details:**
- **A lead arrives**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger)
  - *Configuration:* Listens for POST requests on path `new-lead`. Includes a pre-flight validation filter (`onlyRunIf`) ensuring payload contains a valid non-empty email or phone number.
  - *Key Expressions:* `={{ !!(String($json.body?.email ?? '').trim() || String($json.body?.phone ?? '').replace(/[^0-9]/g, '')) }}`
  - *Input/Output:* Output connects to `Your settings`.
  - *Edge Cases:* Rejects empty or malformed payloads lacking contact identifiers.

- **Every evening at 20:00**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
  - *Configuration:* Configured to trigger daily at hour 20:00 (8:00 PM).
  - *Input/Output:* Output connects to `Your settings`.
  - *Edge Cases:* Requires correct workflow-level timezone configuration.

- **Your settings**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Parameter Initialization)
  - *Configuration:* Defines global variables: `ownerEmail` (`you@example.com`), `reminderMinutes` (`30`), `repeatWindowDays` (`30`), and `timezone` (`UTC`).
  - *Key Expressions:* Manual assignment values.
  - *Input/Output:* Inputs from `A lead arrives` and `Every evening at 20:00`. Output connects to `A new lead? (otherwise the evening summary)`.
  - *Edge Cases:* Defaults must be updated by the user to valid production values.

- **A new lead? (otherwise the evening summary)**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router)
  - *Configuration:* Routes execution based on whether the workflow was invoked by the webhook trigger.
  - *Key Expressions:* `={{ $('A lead arrives').isExecuted }}`
  - *Input/Output:* Input from `Your settings`. True branch connects to `Clean up the lead`; False branch connects to `Get today's leads`.

---

#### Block 2: Lead Deduplication & Storage
**Overview:** Cleans raw incoming lead data, generates a SHA-256 cryptographic lead key based on email or phone, checks the n8n Data Table for existing entries within the repeat window, and stores new unique leads.

**Nodes Involved:**
- `Clean up the lead`
- `Look for the same lead`
- `Seen within the repeat window?`
- `Skip the repeat`
- `Store the lead`

**Node Details:**
- **Clean up the lead**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Data Normalization)
  - *Configuration:* Trims and formats lead fields (`name`, `email`, `phone`, `message`, `source`, `received_at`). Generates a deterministic SHA-256 hash as a `lead_key`.
  - *Key Expressions:* `={{ (String($json.body?.email ?? '').trim().toLowerCase() || String($json.body?.phone ?? '').replace(/[^0-9]/g, '')).hash('sha256') }}`
  - *Input/Output:* Input from `A new lead?`. Output connects to `Look for the same lead`.
  - *Edge Cases:* Truncates string lengths to prevent storage overflow.

- **Look for the same lead**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Data Table Lookup)
  - *Configuration:* Queries the Data Table for existing rows matching the generated `lead_key`, ordering by creation date descending.
  - *Key Expressions:* `={{ $json.lead_key }}`
  - *Input/Output:* Input from `Clean up the lead`. Output connects to `Seen within the repeat window?`.
  - *Edge Cases:* Always outputs data (`alwaysOutputData: true`) to prevent halting on empty query results.

- **Seen within the repeat window?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router)
  - *Configuration:* Evaluates if a matching lead record exists and was received within the timeframe defined by `repeatWindowDays`.
  - *Key Expressions:* `={{ !!$json.id && new Date($json.received_at || $json.createdAt).getTime() > Date.now() - Number($('Your settings').first().json.repeatWindowDays) * 86400000 }}`
  - *Input/Output:* Input from `Look for the same lead`. True branch connects to `Skip the repeat`; False branch connects to `Store the lead`.

- **Skip the repeat**
  - *Type & Technical Role:* `n8n-nodes-base.noOp` (No Operation / Terminal)
  - *Configuration:* Terminates the duplicate lead branch cleanly without performing actions.
  - *Input/Output:* Input from `Seen within the repeat window?` (True). No outputs.

- **Store the lead**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Data Table Insert)
  - *Configuration:* Inserts a new row containing lead details and receipt timestamp into the Data Table.
  - *Key Expressions:* Maps properties from `Clean up the lead`.
  - *Input/Output:* Input from `Seen within the repeat window?` (False). Output connects to `Alert the owner`.

---

#### Block 3: Notification, Waiting, & Response Handling
**Overview:** Alerts the workflow owner via Gmail with an interactive execution-resume link, pauses workflow execution for a configured duration, and evaluates whether the owner marked the lead as answered.

**Nodes Involved:**
- `Alert the owner`
- `Wait for \"answered\" or the time limit`
- `Marked as answered?`
- `Record the answer time`
- `Send the one reminder`
- `Record the reminder`

**Node Details:**
- **Alert the owner**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
  - *Configuration:* Sends a plain text email to the owner containing lead details and a unique webhook resume URL with query parameter `?answered=1`.
  - *Credentials:* `Gmail account` (OAuth2).
  - *Key Expressions:* `={{ $execution.resumeUrl }}?answered=1`
  - *Input/Output:* Input from `Store the lead`. Output connects to `Wait for "answered" or the time limit`.
  - *Edge Cases:* Requires active Gmail OAuth2 credentials.

- **Wait for "answered" or the time limit**
  - *Type & Technical Role:* `n8n-nodes-base.wait` (Execution Pause / Webhook Resume)
  - *Configuration:* Pauses workflow execution using a webhook GET listener, with a fallback timeout defined by `reminderMinutes`.
  - *Key Expressions:* `={{ $('Your settings').first().json.reminderMinutes }}`
  - *Input/Output:* Input from `Alert the owner`. Output connects to `Marked as answered?`.
  - *Edge Cases:* Execution times out and resumes automatically if the link is not clicked within the time limit.

- **Marked as answered?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router)
  - *Configuration:* Checks if the incoming query parameter from the resume webhook equals `1`.
  - *Key Expressions:* `={{ $json.query?.answered ?? "" }}`
  - *Input/Output:* Input from `Wait`. True branch connects to `Record the answer time`; False branch connects to `Send the one reminder`.

- **Record the answer time**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Data Table Update)
  - *Configuration:* Updates the corresponding lead record with the current UTC timestamp (`answered_at`).
  - *Key Expressions:* `={{ $now.toUTC().toISO() }}` (filtered by `id` from `Store the lead`).
  - *Input/Output:* Input from `Marked as answered?` (True). No subsequent nodes.

- **Send the one reminder**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
  - *Configuration:* Sends a single reminder email to the owner if the time limit expires without human intervention.
  - *Credentials:* `Gmail account` (OAuth2).
  - *Input/Output:* Input from `Marked as answered?` (False). Output connects to `Record the reminder`.

- **Record the reminder**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Data Table Update)
  - *Configuration:* Updates the lead record with the reminder timestamp (`reminded_at`).
  - *Key Expressions:* `={{ $now.toUTC().toISO() }}` (filtered by `id` from `Store the lead`).
  - *Input/Output:* Input from `Send the one reminder`. No subsequent nodes.

---

#### Block 4: Evening Summary Reporting
**Overview:** Queries today's leads from the Data Table, aggregates performance metrics via custom JavaScript, and emails a daily summary report to the owner.

**Nodes Involved:**
- `Get today's leads`
- `Count the day`
- `Email the daily summary`

**Node Details:**
- **Get today's leads**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Data Table Lookup)
  - *Configuration:* Retrieves all Data Table rows where `received_at` is greater than or equal to the start of the current day in the configured timezone.
  - *Key Expressions:* `={{ $now.setZone($json.timezone || 'UTC').startOf('day').toUTC().toISO() }}`
  - *Input/Output:* Input from `A new lead?` (False). Output connects to `Count the day`.
  - *Edge Cases:* Configured with `alwaysOutputData: true` to ensure reporting proceeds even on days with zero leads.

- **Count the day**
  - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Data Aggregation)
  - *Configuration:* Processes raw table rows to calculate total received leads, leads answered within the time window, open leads, and total reminders sent.
  - *Key Expressions:* Custom JavaScript processing `$input.all()`.
  - *Input/Output:* Input from `Get today's leads`. Output connects to `Email the daily summary`.

- **Email the daily summary**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
  - *Configuration:* Sends the aggregated daily summary report email to the owner.
  - *Credentials:* `Gmail account` (OAuth2).
  - *Key Expressions:* `={{ $json.text }}` and `={{ $json.subject }}`.
  - *Input/Output:* Input from `Count the day`. No subsequent nodes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template description | n8n-nodes-base.stickyNote | Documentation & Setup Guide | None | None | ## Send new-lead alerts, a 30-minute reminder and a daily summary with Gmail and Data Tables<br><br>### Who's it for<br>Small businesses and teams whose website leads sometimes wait hours for a first answer, and who want one clear alert per lead instead of a busy inbox.<br><br>### How it works<br>1. Your form posts each lead (name, email, phone, message, source) to the webhook.<br>2. The lead is cleaned up and given a key: a hash of the email or phone. If the same person already arrived within the repeat window, nothing happens.<br>3. A new lead is stored in a data table and the owner gets one Gmail alert with a "mark as answered" link.<br>4. If nobody opens that link within the reminder time, one reminder is sent. There is never a second one.<br>5. Every evening a short email counts the day: received, answered in time, still open.<br><br>### How to set up<br>1. Create a data table with the columns **lead_key, name, email, phone, source** (text) and **received_at, answered_at, reminded_at** (date). Select it in the five Data table nodes.<br>2. Connect your Gmail account in the three Gmail nodes.<br>3. Fill in **Your settings**: owner email, reminder minutes, repeat window in days, timezone.<br>4. Point your form at the webhook's production URL, then activate the workflow.<br><br>### Requirements<br>- n8n with Data tables<br>- A Gmail account<br><br>### How to customize the workflow<br>- Swap the Gmail nodes for Slack, Telegram or Outlook.<br>- Change the summary time in **Every evening at 20:00** and set the same timezone in the workflow settings.<br>- Add Header Auth to the webhook if your form can send a header.<br><br>Made by Betterlane — step-by-step guide: https://betterlaneagency.com/learn/n8n-workflow-runs-twice |
| Step 1 note | n8n-nodes-base.stickyNote | Architecture Documentation (Lead Ingestion) | None | None | ## 1. A lead arrives and is stored once<br>The webhook answers the form at once. The lead key is a hash, so a repeat within the window is recognised without using the contact as the key. |
| Step 2 note | n8n-nodes-base.stickyNote | Architecture Documentation (Alerts & Reminders) | None | None | ## 2. One alert, one reminder<br>The alert carries a private link. Opening it resumes this run and records the answer time. If the time limit passes first, one reminder goes out. |
| Step 3 note | n8n-nodes-base.stickyNote | Architecture Documentation (Daily Summary) | None | None | ## 3. The daily summary<br>Every evening the day's rows are counted from the data table and emailed to the owner, even on a day with no leads. |
| A lead arrives | n8n-nodes-base.webhook | Webhook trigger for incoming form submissions | None | Your settings | |
| Your settings | n8n-nodes-base.set | Sets global configuration variables | A lead arrives, Every evening at 20:00 | A new lead? (otherwise the evening summary) | |
| Every evening at 20:00 | n8n-nodes-base.scheduleTrigger | Daily schedule trigger for evening reports | None | Your settings | |
| A new lead? (otherwise the evening summary) | n8n-nodes-base.if | Routes execution based on invocation source | Your settings | Clean up the lead, Get today's leads | |
| Clean up the lead | n8n-nodes-base.set | Normalizes fields and generates a SHA-256 lead hash | A new lead? (otherwise the evening summary) | Look for the same lead | |
| Look for the same lead | n8n-nodes-base.dataTable | Queries Data Table for existing lead matches | Clean up the lead | Seen within the repeat window? | |
| Seen within the repeat window? | n8n-nodes-base.if | Evaluates if lead arrived within repeat window | Look for the same lead | Skip the repeat, Store the lead | |
| Skip the repeat | n8n-nodes-base.noOp | Terminates duplicate lead execution path | Seen within the repeat window? | None | |
| Store the lead | n8n-nodes-base.store | Inserts new lead record into Data Table | Seen within the repeat window? | Alert the owner | |
| Alert the owner | n8n-nodes-base.gmail | Sends initial lead notification email with resume link | Store the lead | Wait for "answered" or the time limit | |
| Wait for "answered" or the time limit | n8n-nodes-base.wait | Pauses execution awaiting webhook resume or timeout | Alert the owner | Marked as answered? | |
| Marked as answered? | n8n-nodes-base.if | Checks if owner clicked the "answered" link | Wait for "answered" or the time limit | Record the answer time, Send the one reminder | |
| Record the answer time | n8n-nodes-base.dataTable | Updates Data Table row with answered timestamp | Marked as answered? | None | |
| Send the one reminder | n8n-nodes-base.gmail | Sends single reminder email if time limit expires | Marked as answered? | Record the reminder | |
| Record the reminder | n8n-nodes-base.dataTable | Updates Data Table row with reminder timestamp | Send the one reminder | None | |
| Get today's leads | n8n-nodes-base.dataTable | Retrieves today's records from Data Table | A new lead? (otherwise the evening summary) | Count the day | |
| Count the day | n8n-nodes-base.code | Aggregates daily lead metrics via JavaScript | Get today's leads | Email the daily summary | |
| Email the daily summary | n8n-nodes-base.gmail | Sends evening performance summary report via Gmail | Count the day | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create an n8n Data Table:**
   - Create a new Data Table with the following columns:
     - `lead_key` (Type: Text)
     - `name` (Type: Text)
     - `email` (Type: Text)
     - `phone` (Type: Text)
     - `source` (Type: Text)
     - `received_at` (Type: Date/Time)
     - `answered_at` (Type: Date/Time)
     - `reminded_at` (Type: Date/Time)

2. **Set up Triggers & Configuration:**
   - **Node 1:** Create a **Webhook** node named `A lead arrives`. Set HTTP Method to `POST`, path to `new-lead`, and response mode to `onReceived`. Add an expression under options `onlyRunIf`: `={{ !!(String($json.body?.email ?? '').trim() || String($json.body?.phone ?? '').replace(/[^0-9]/g, '')) }}`.
   - **Node 2:** Create a **Schedule Trigger** node named `Every evening at 20:00`. Configure interval to trigger every 1 day at hour `20`, minute `0`.
   - **Node 3:** Create a **Set** node named `Your settings`. Add string assignments for `ownerEmail` (`you@example.com`) and `timezone` (`UTC`), and number assignments for `reminderMinutes` (`30`) and `repeatWindowDays` (`30`). Enable `Include Other Fields`. Connect both `A lead arrives` and `Every evening at 20:00` to this node.
   - **Node 4:** Create an **If** node named `A new lead? (otherwise the evening summary)`. Set condition left value to `={{ $('A lead arrives').isExecuted }}` with operation `Boolean: true`. Connect `Your settings` to this node.

3. **Build the Lead Ingestion & Deduplication Path:**
   - **Node 5:** Create a **Set** node named `Clean up the lead`. Connect the `true` output of `A new lead?` here. Configure manual assignments for:
     - `name`: `={{ String($json.body?.name ?? '').trim().slice(0, 120) }}`
     - `email`: `={{ String($json.body?.email ?? '').trim().toLowerCase().slice(0, 200) }}`
     - `phone`: `={{ String($json.body?.phone ?? '').replace(/[^0-9+]/g, '').slice(0, 30) }}`
     - `message`: `={{ String($json.body?.message ?? '').trim().slice(0, 2000) }}`
     - `source`: `={{ String($json.body?.source ?? 'website').trim().slice(0, 80) || 'website' }}`
     - `received_at`: `={{ $now.toUTC().toISO() }}`
     - `lead_key`: `={{ (String($json.body?.email ?? '').trim().toLowerCase() || String($json.body?.phone ?? '').replace(/[^0-9]/g, '')).hash('sha256') }}`
   - **Node 6:** Create a **Data Table** node named `Look for the same lead`. Set resource to `Row`, operation to `Get`, select your Data Table, set limit to `1`, order by `createdAt` (DESC), and add a filter condition where `lead_key` equals `={{ $json.lead_key }}`. Enable `Always Output Data`. Connect `Clean up the lead` here.
   - **Node 7:** Create an **If** node named `Seen within the repeat window?`. Set condition to evaluate: `={{ !!$json.id && new Date($json.received_at || $json.createdAt).getTime() > Date.now() - Number($('Your settings').first().json.repeatWindowDays) * 86400000 }}` (Operation: Boolean `true`). Connect `Look for the same lead` here.
   - **Node 8:** Create a **No Operation** node named `Skip the repeat`. Connect the `true` output of `Seen within the repeat window?` here.
   - **Node 9:** Create a **Data Table** node named `Store the lead`. Connect the `false` output of `Seen within the repeat window?` here. Set resource to `Row`, operation to `Insert`, select your Data Table, and map table columns to corresponding fields from `Clean up the lead`.

4. **Build the Notification & Response Handling Path:**
   - **Node 10:** Create a **Gmail** node named `Alert the owner`. Configure with credential `Gmail OAuth2`. Set resource to `Message`, operation to `Send`, recipient to `={{ $('Your settings').first().json.ownerEmail }}`, subject to `=New lead: {{ $('Clean up the lead').first().json.name || $('Clean up the lead').first().json.email || $('Clean up the lead').first().json.phone }}`, and message body containing `={{ $execution.resumeUrl }}?answered=1`. Connect `Store the lead` here.
   - **Node 11:** Create a **Wait** node named `Wait for "answered" or the time limit`. Set resume to `Webhook`, limit type to `After Time Interval`, resume amount to `={{ $('Your settings').first().json.resumeMinutes }}` (or `reminderMinutes`), and response data to indicate successful acknowledgment. Connect `Alert the owner` here.
   - **Node 12:** Create an **If** node named `Marked as answered?`. Set condition checking if `={{ $json.query?.answered ?? "" }}` equals `1`. Connect `Wait` here.
   - **Node 13:** Create a **Data Table** node named `Record the answer time` (connected to `true` output of `Marked as answered?`). Set resource `Row`, operation `Update`, filter where `id` equals `={{ $('Store the lead').first().json.id }}`, and update `answered_at` to `={{ $now.toUTC().toISO() }}`.
   - **Node 14:** Create a **Gmail** node named `Send the one reminder` (connected to `false` output of `Marked as answered?`). Configure Gmail OAuth2 credentials, recipient `={{ $('Your settings').first().json.ownerEmail }}`, subject `=Reminder: lead not marked as answered...`, and body detailing the unaddressed lead.
   - **Node 15:** Create a **Data Table** node named `Record the reminder`. Connect `Send the one reminder` here. Set resource `Row`, operation `Update`, filter where `id` equals `={{ $('Store the lead').first().json.id }}`, and update `reminded_at` to `={{ $now.toUTC().toISO() }}`.

5. **Build the Evening Summary Reporting Path:**
   - **Node 16:** Create a **Data Table** node named `Get today's leads`. Connect the `false` output of `A new lead?` here. Set resource `Row`, operation `Get (All)`, select your Data Table, filter where `received_at` is greater than or equal (`gte`) to `={{ $now.setZone($json.timezone || 'UTC').startOf('day').toUTC().toISO() }}`. Enable `Always Output Data`.
   - **Node 17:** Create a **Code** node named `Count the day`. Connect `Get today's leads` here. Paste the JavaScript snippet provided in the workflow JSON to aggregate metrics.
   - **Node 18:** Create a **Gmail** node named `Email the daily summary`. Connect `Count the day` here. Configure Gmail OAuth2 credentials, recipient `={{ $('Your settings').first().json.ownerEmail }}`, subject `={{ $json.subject }}`, and message `={{ $json.text }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Made by Betterlane — step-by-step guide | [Betterlane Guide](https://betterlaneagency.com/learn/n8n-workflow-runs-twice) |