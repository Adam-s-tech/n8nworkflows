Run a personal executive assistant with Gmail, Slack, Calendar, ClickUp, and OpenAI

https://n8nworkflows.xyz/workflows/run-a-personal-executive-assistant-with-gmail--slack--calendar--clickup--and-openai-19791


# Run a personal executive assistant with Gmail, Slack, Calendar, ClickUp, and OpenAI

### 1. Workflow Overview

This workflow acts as a personal executive assistant designed to automate daily operational tasks across Gmail, Google Calendar, Slack, ClickUp, and OpenAI. Running every 15 minutes, it simultaneously executes four distinct logical blocks to triage communications, manage schedule conflicts, and compile scheduled briefings.

- **1.1 Trigger and Parallel Pipeline Initialization:** A recurring schedule node triggers four parallel branches every 15 minutes for email triage, calendar conflict detection, Slack mention monitoring, and a daily time check.
- **1.2 Email Triage and Processing:** Fetches unread emails (excluding promotions and social categories), uses an AI agent with structured JSON output to evaluate priority and draft responses, conditionally requests Slack approval, dispatches or drafts the reply in Gmail, and logs a follow-up task in ClickUp.
- **1.3 Calendar Conflict Detection and Rescheduling:** Pulls upcoming Google Calendar events for the next 24 hours, programmatically checks for time overlaps using a custom JavaScript code node, routes conflicts to an AI agent for rescheduling recommendations, and updates the calendar event following Slack approval.
- **1.4 Slack Mention Triage and Response:** Retrieves recent Slack channel history (past 15 minutes), uses an AI agent to determine if a response is needed, and either sends the reply immediately or holds it for Slack approval.
- **1.5 Morning Briefing Generation:** Evaluates if the current hour is 7 AM, fetches tasks due today from ClickUp, compiles a comprehensive summary using an AI language model combined with unread email and calendar data, and delivers the briefing via Gmail and Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Parallel Pipeline Initialization
- **Overview:** Initializes the execution cycle every 15 minutes and fans out execution into four concurrent operational paths.
- **Nodes Involved:** 
  - `Every 15 Min Schedule`
- **Node Details:**
  - **Every 15 Min Schedule**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - *Configuration:* Rule-based interval set to fire every 15 minutes.
    - *Input/Output:* No inputs; outputs to `Gmail - Get Unread Emails`, `Google Calendar - Get Upcoming Events`, `Slack - Get Recent Mentions`, and `Is Briefing Time? (7AM)`.
    - *Edge Cases/Failures:* Missed executions if the n8n instance is offline.

#### 2.2 Email Triage Pipeline
- **Overview:** Fetches unread emails, analyzes them with GPT-4o-mini using strict JSON schema enforcement, branches based on urgency/sensitivity, handles sending or drafting in Gmail, and records tracking tasks in ClickUp.
- **Nodes Involved:** 
  - `Gmail - Get Unread Emails`
  - `AI Agent - Email Triage`
  - `OpenAI Model - Email Triage`
  - `Email Triage Output Schema`
  - `Needs Approval? (Email)`
  - `Slack - Request Email Reply Approval`
  - `Gmail - Send Approved Reply`
  - `Gmail - Save as Draft (No Approval)`
  - `ClickUp - Create Task from Email`
- **Node Details:**
  - **Gmail - Get Unread Emails**
    - *Type and Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration:* Fetches up to 20 messages matching the query filter `is:unread -category:promotions -category:social`. Uses Gmail OAuth2.
    - *Input/Output:* Input from schedule trigger; output to `AI Agent - Email Triage`.
    - *Edge Cases:* API rate limits, authentication token expiration.
  - **AI Agent - Email Triage**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
    - *Configuration:* Evaluates email metadata (sender, subject, body) and sets `requires_approval=true` for client-facing, financial, or ambiguous messages. Connected to an OpenAI model and structured output parser.
    - *Input/Output:* Inputs from Gmail; output to `Needs Approval? (Email)`.
  - **OpenAI Model - Email Triage**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Language Model)
    - *Configuration:* Uses model `gpt-4o-mini` with temperature set to `0.2`.
    - *Input/Output:* Linked to `AI Agent - Email Triage`.
  - **Email Triage Output Schema**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser)
    - *Configuration:* Enforces a JSON schema containing `priority`, `requires_approval`, `draft_reply`, `task_title`, and `task_description`.
    - *Input/Output:* Linked to `AI Agent - Email Triage`.
  - **Needs Approval? (Email)**
    - *Type and Role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration:* Checks if `{{ $json.output.requires_approval }}` equals `true`.
    - *Input/Output:* Input from `AI Agent - Email Triage`; true branch to Slack approval, false branch to save as draft.
  - **Slack - Request Email Reply Approval**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration with Wait capability)
    - *Configuration:* Sends an interactive approval card to channel `C0AN1UGL0RM` using OAuth2 and double approval settings.
    - *Input/Output:* Input from conditional router; output to `Gmail - Send Approved Reply`.
  - **Gmail - Send Approved Reply**
    - *Type and Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration:* Replies to the original message ID using the AI-generated draft response.
    - *Input/Output:* Input from Slack approval node; output to `ClickUp - Create Task from Email`.
  - **Gmail - Save as Draft (No Approval)**
    - *Type and Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration:* Creates a Gmail draft using the generated reply and thread ID when approval is bypassed.
    - *Input/Output:* Input from conditional router; output to `ClickUp - Create Task from Email`.
  - **ClickUp - Create Task from Email**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration:* Sends a POST request to ClickUp API (`https://api.clickup.com/api/v2/list/YOUR_CLICKUP_LIST_ID/task`) using header authentication. Payload extracts `task_title` and `task_description` from the AI output.
    - *Input/Output:* Inputs from `Gmail - Send Approved Reply` or `Gmail - Save as Draft (No Approval)`.

#### 2.3 Calendar Conflict Detection and Rescheduling
- **Overview:** Retrieves upcoming calendar events, detects overlapping times programmatically, requests AI rescheduling guidance, secures approval via Slack, and updates the calendar event.
- **Nodes Involved:** 
  - `Google Calendar - Get Upcoming Events`
  - `Detect Calendar Conflicts`
  - `Conflicts Found?`
  - `AI Agent - Reschedule Proposal`
  - `OpenAI Model - Reschedule Agent`
  - `Slack - Request Reschedule Approval`
  - `Google Calendar - Update Event`
- **Node Details:**
  - **Google Calendar - Get Upcoming Events**
    - *Type and Role:* `n8n-nodes-base.googleCalendar` (Google Calendar Integration)
    - *Configuration:* Retrieves events from the primary calendar between current time (`$now.iso()`) and 24 hours ahead.
    - *Input/Output:* Input from schedule trigger; output to `Detect Calendar Conflicts`.
  - **Detect Calendar Conflicts**
    - *Type and Role:* `n8n-nodes-base.code` (Custom JavaScript Code)
    - *Configuration:* Sorts events chronologically and iterates through them to identify if `start time of next event < end time of current event`. Returns conflicting pairs (`eventA` and `eventB`).
    - *Input/Output:* Input from calendar events; output to `Conflicts Found?`.
  - **Conflicts Found?**
    - *Type and Role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration:* Evaluates if total items in array > 0.
    - *Input/Output:* Input from code node; true branch to rescheduling AI agent.
  - **AI Agent - Reschedule Proposal**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
    - *Configuration:* Prompts the agent with conflict details to suggest which event to move and propose a non-conflicting time.
    - *Input/Output:* Input from conflict check; output to Slack approval.
  - **OpenAI Model - Reschedule Agent**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Language Model)
    - *Configuration:* Model `gpt-4o-mini`, temperature `0.2`.
    - *Input/Output:* Linked to `AI Agent - Reschedule Proposal`.
  - **Slack - Request Reschedule Approval**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration)
    - *Configuration:* Sends an interactive approval request to channel `C0AN1UGL0RM`.
    - *Input/Output:* Input from AI rescheduling agent; output to `Google Calendar - Update Event`.
  - **Google Calendar - Update Event**
    - *Type and Role:* `n8n-nodes-base.googleCalendar` (Google Calendar Integration)
    - *Configuration:* Updates the target event (`eventA`) in the primary calendar using OAuth2.
    - *Input/Output:* Input from Slack approval node.

#### 2.4 Slack Mention Triage and Response
- **Overview:** Monitors recent Slack channel history for mentions, evaluates urgency and response requirements via AI, and conditionally posts or requests approval.
- **Nodes Involved:** 
  - `Slack - Get Recent Mentions`
  - `AI Agent - Slack Triage`
  - `OpenAI Model - Slack Triage`
  - `Needs Approval? (Slack)`
  - `Slack - Request Response Approval`
  - `Slack - Send Reply`
- **Node Details:**
  - **Slack - Get Recent Mentions**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration)
    - *Configuration:* Fetches channel history from channel ID `C0AN1UGL0RM` with a time filter for the past 15 minutes (`oldest` calculated via expression).
    - *Input/Output:* Input from schedule trigger; output to `AI Agent - Slack Triage`.
  - **AI Agent - Slack Triage**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
    - *Configuration:* Triages incoming message text to determine priority, whether a reply is needed, and if a ClickUp task should be logged.
    - *Input/Output:* Input from Slack history; output to `Needs Approval? (Slack)`.
  - **OpenAI Model - Slack Triage**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Language Model)
    - *Configuration:* Model `gpt-4o-mini`, temperature `0.2`.
    - *Input/Output:* Linked to `AI Agent - Slack Triage`.
  - **Needs Approval? (Slack)**
    - *Type and Role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration:* Checks if `{{ $json.output.requires_approval }}` equals `true`.
    - *Input/Output:* Input from Slack triage agent; true branch requests approval, false branch sends directly.
  - **Slack - Request Response Approval**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration)
    - *Configuration:* Sends an interactive approval request to channel `C0AN1UGL0RM`.
    - *Input/Output:* Input from condition node; output to `Slack - Send Reply`.
  - **Slack - Send Reply**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration)
    - *Configuration:* Posts message text to channel `C0AN1UGL0RM`.
    - *Input/Output:* Inputs from approval node or direct condition bypass.

#### 2.5 Morning Briefing Generation
- **Overview:** Triggers exclusively at 7 AM, pulls daily ClickUp tasks, aggregates context from emails and calendar events, compiles a daily summary via AI, and distributes it via Gmail and Slack.
- **Nodes Involved:** 
  - `Is Briefing Time? (7AM)`
  - `ClickUp - Get Tasks Due Today`
  - `AI Agent - Compile Morning Briefing`
  - `OpenAI Model - Briefing Agent`
  - `Gmail - Send Morning Briefing`
  - `Slack - Post Morning Briefing`
- **Node Details:**
  - **Is Briefing Time? (7AM)**
    - *Type and Role:* `n8n-nodes-base.if` (Condition Router)
    - *Configuration:* Verifies if `{{ $now.hour }}` equals `7`.
    - *Input/Output:* Input from schedule trigger; true branch proceeds to ClickUp task retrieval.
  - **ClickUp - Get Tasks Due Today**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration:* Queries ClickUp tasks with date range filters (`due_date_gt` and `due_date_lt` for the current day) using header authentication.
    - *Input/Output:* Input from 7 AM check; output to briefing AI agent.
  - **AI Agent - Compile Morning Briefing**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (LangChain AI Agent)
    - *Configuration:* Aggregates unread email data, upcoming calendar events, and ClickUp tasks into a structured, skimmable briefing.
    - *Input/Output:* Input from ClickUp task fetch; outputs to Gmail and Slack.
  - **OpenAI Model - Briefing Agent**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Language Model)
    - *Configuration:* Model `gpt-4o-mini`, temperature `0.2`.
    - *Input/Output:* Linked to `AI Agent - Compile Morning Briefing`.
  - **Gmail - Send Morning Briefing**
    - *Type and Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration:* Sends an email to "me" with a dynamic subject line displaying the current date.
    - *Input/Output:* Input from briefing AI agent.
  - **Slack - Post Morning Briefing**
    - *Type and Role:* `n8n-nodes-base.slack` (Slack Integration)
    - *Configuration:* Posts briefing text to Slack channel `C0AN1UGL0RM`.
    - *Input/Output:* Input from briefing AI agent.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `📌 Overview – How This Workflow Works` | n8n-nodes-base.stickyNote | Documentation | None | None | ## 🧑‍💼 Personal Executive Assistant Agent – Email, Calendar, Slack & Briefing<br><br>### How it works<br>Every 15 minutes, a single schedule trigger fires four parallel pipelines that cover your core executive assistant tasks. **Email Triage:** fetches unread Gmail, triages each email with GPT-4o-mini (urgency, sentiment, draft reply), and asks for Slack approval before sending — or saves a draft if no approval is needed, then creates a ClickUp task. **Calendar Conflicts:** reads upcoming events, uses a code node to detect overlapping meetings, and routes to a rescheduling AI agent that proposes alternatives and seeks Slack approval before updating the event. **Slack Mentions:** fetches recent @mentions, triages them with GPT-4o-mini, and either sends a reply directly or requests approval first. **Morning Briefing:** checks if it's 7AM, fetches ClickUp tasks due today, and compiles a GPT-4o-mini briefing sent as both a Gmail email and a Slack message.<br><br>### Setup steps<br>1. Connect **Gmail OAuth2** to all Gmail nodes.<br>2. Connect **Slack OAuth2** (and Slack API) to all Slack nodes — replace `REPLACE_WITH_YOUR_SLACK_CHANNEL_ID` and your user ID in approval nodes.<br>3. Connect **Google Calendar OAuth2** to both Calendar nodes.<br>4. Connect **OpenAI API** to all four OpenAI Chat Model nodes.<br>5. Connect **ClickUp API** to `ClickUp - Create Task from Email` and `ClickUp - Get Tasks Due Today` — replace `REPLACE_WITH_YOUR_CLICKUP_LIST_ID`.<br>6. Activate the schedule trigger. |
| `Section – Email Triage Pipeline` | n8n-nodes-base.stickyNote | Section grouping | None | None | ## 📧 Email Triage Pipeline<br>Fetches unread Gmail messages, sends each to the Email Triage AI agent (urgency, sentiment, suggested reply). High-priority emails request Slack approval before sending; others are saved as drafts. Either way, a ClickUp task is created to track follow-up. |
| `Section – Calendar Conflict Detection & Rescheduling` | n8n-nodes-base.stickyNote | Section grouping | None | None | ## 📅 Calendar Conflict Detection & Rescheduling<br>Reads upcoming calendar events and checks for overlapping time slots in a code node. If conflicts are found, the Reschedule AI agent proposes alternatives. A Slack approval request fires before the calendar event is updated. |
| `Section – Slack Mention Triage` | n8n-nodes-base.stickyNote | Section grouping | None | None | ## 💬 Slack Mention Triage<br>Fetches recent Slack @mentions and passes them to the Slack Triage AI agent, which determines the right response. Low-stakes replies are sent automatically; responses requiring judgment are held for Slack approval before posting. |
| `Section – Morning Briefing` | n8n-nodes-base.stickyNote | Section grouping | None | None | ## ☀️ Morning Briefing (7AM)<br>Checks whether it's 7AM. If so, fetches all ClickUp tasks due today and sends them to the Briefing AI agent, which compiles a structured daily plan. The briefing is sent simultaneously as a Gmail email and a Slack message. |
| `🔑 Credentials & Configuration` | n8n-nodes-base.stickyNote | Credential reference | None | None | ## 🔑 Credentials Required<br>- **Gmail OAuth2** — email fetch, reply, draft, briefing send<br>- **Slack OAuth2 + API** — mention fetch, approvals, replies, briefing post<br>- **Google Calendar OAuth2** — event fetch + update<br>- **OpenAI API** — all four AI agents<br>- **ClickUp API** — task creation + today's task fetch |
| `Every 15 Min Schedule` | n8n-nodes-base.scheduleTrigger | Triggers pipeline every 15m | None | Gmail - Get Unread Emails, Google Calendar - Get Upcoming Events, Slack - Get Recent Mentions, Is Briefing Time? (7AM) | |
| `Gmail - Get Unread Emails` | n8n-nodes-base.gmail | Fetches unread Gmail | Every 15 Min Schedule | AI Agent - Email Triage | |
| `AI Agent - Email Triage` | @n8n/n8n-nodes-langchain.agent | Evaluates email urgency & draft | Gmail - Get Unread Emails | Needs Approval? (Email) | |
| `Needs Approval? (Email)` | n8n-nodes-base.if | Routes based on approval need | AI Agent - Email Triage | Slack - Request Email Reply Approval, Gmail - Save as Draft (No Approval) | |
| `Slack - Request Email Reply Approval` | n8n-nodes-base.slack | Requests human approval in Slack | Needs Approval? (Email) | Gmail - Send Approved Reply | |
| `Gmail - Send Approved Reply` | n8n-nodes-base.gmail | Sends approved email reply | Slack - Request Email Reply Approval | ClickUp - Create Task from Email | |
| `Gmail - Save as Draft (No Approval)` | n8n-nodes-base.gmail | Saves email draft | Needs Approval? (Email) | ClickUp - Create Task from Email | |
| `ClickUp - Create Task from Email` | n8n-nodes-base.httpRequest | Creates ClickUp follow-up task | Gmail - Send Approved Reply, Gmail - Save as Draft (No Approval) | None | |
| `Google Calendar - Get Upcoming Events` | n8n-nodes-base.googleCalendar | Fetches calendar events (24h) | Every 15 Min Schedule | Detect Calendar Conflicts | |
| `Detect Calendar Conflicts` | n8n-nodes-base.code | Detects schedule overlaps | Google Calendar - Get Upcoming Events | Conflicts Found? | |
| `Conflicts Found?` | n8n-nodes-base.if | Checks if conflicts exist | Detect Calendar Conflicts | AI Agent - Reschedule Proposal | |
| `AI Agent - Reschedule Proposal` | @n8n/n8n-nodes-langchain.agent | Proposes schedule changes | Conflicts Found? | Slack - Request Reschedule Approval | |
| `Slack - Request Reschedule Approval` | n8n-nodes-base.slack | Requests reschedule approval | AI Agent - Reschedule Proposal | Google Calendar - Update Event | |
| `Google Calendar - Update Event` | n8n-nodes-base.googleCalendar | Updates calendar event time | Slack - Request Reschedule Approval | None | |
| `Slack - Get Recent Mentions` | n8n-nodes-base.slack | Fetches recent Slack messages | Every 15 Min Schedule | AI Agent - Slack Triage | |
| `AI Agent - Slack Triage` | @n8n/n8n-nodes-langchain.agent | Triages Slack messages | Slack - Get Recent Mentions | Needs Approval? (Slack) | |
| `Needs Approval? (Slack)` | n8n-nodes-base.if | Routes Slack reply approval | AI Agent - Slack Triage | Slack - Request Response Approval, Slack - Send Reply | |
| `Slack - Request Response Approval` | n8n-nodes-base.slack | Requests Slack reply approval | Needs Approval? (Slack) | Slack - Send Reply | |
| `Slack - Send Reply` | n8n-nodes-base.slack | Sends reply to Slack channel | Needs Approval? (Slack), Slack - Request Response Approval | None | |
| `Is Briefing Time? (7AM)` | n8n-nodes-base.if | Checks if current hour is 7 AM | Every 15 Min Schedule | ClickUp - Get Tasks Due Today | |
| `ClickUp - Get Tasks Due Today` | n8n-nodes-base.httpRequest | Fetches tasks due today | Is Briefing Time? (7AM) | AI Agent - Compile Morning Briefing | |
| `AI Agent - Compile Morning Briefing` | @n8n/n8n-nodes-langchain.agent | Compiles daily summary | ClickUp - Get Tasks Due Today | Gmail - Send Morning Briefing, Slack - Post Morning Briefing | |
| `Gmail - Send Morning Briefing` | n8n-nodes-base.gmail | Sends briefing email | AI Agent - Compile Morning Briefing | None | |
| `Slack - Post Morning Briefing` | n8n-nodes-base.slack | Posts briefing to Slack | AI Agent - Compile Morning Briefing | None | |
| `OpenAI Model - Briefing Agent` | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Briefing Agent | None | None | |
| `OpenAI Model - Slack Triage` | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Slack Triage Agent | None | None | |
| `OpenAI Model - Email Triage` | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Email Triage Agent | None | None | |
| `Email Triage Output Schema` | @n8n/n8n-nodes-langchain.outputParserStructured | JSON Schema for Email Agent | None | None | |
| `OpenAI Model - Reschedule Agent` | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Reschedule Agent | None | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Trigger Node Creation:**
   - Create a `Schedule Trigger` node (`Every 15 Min Schedule`). Set interval rule to every `15` minutes.
2. **Email Triage Pipeline Setup:**
   - Create a `Gmail` node (`Gmail - Get Unread Emails`) with operation `Get Many`, limit `20`, and query filter `is:unread -category:promotions -category:social`.
   - Create an AI Agent node (`AI Agent - Email Triage`) connected to the Gmail node. Configure prompt to triage email inputs and determine approval requirements.
   - Create an OpenAI Chat Model node (`OpenAI Model - Email Triage`), select model `gpt-4o-mini`, temperature `0.2`, and connect it as the AI language model for the email agent.
   - Create a Structured Output Parser (`Email Triage Output Schema`) defining JSON properties: `priority` (enum: urgent, normal, low), `requires_approval` (boolean), `draft_reply` (string), `task_title` (string), `task_description` (string). Connect it to the email agent.
   - Create an IF node (`Needs Approval? (Email)`) evaluating `{{ $json.output.requires_approval === true }}`.
   - Connect the `true` branch to a `Slack` node (`Slack - Request Email Reply Approval`) configured with operation `Send and Wait` (double approval) targeting channel `C0AN1UGL0RM`.
   - Connect the Slack approval node to a `Gmail` node (`Gmail - Send Approved Reply`) with operation `Reply` using message ID from the original email and message body from the draft reply.
   - Connect the `false` branch of the IF node to a `Gmail` node (`Gmail - Save as Draft (No Approval)`) with resource `Draft` and thread ID configured.
   - Create an HTTP Request node (`ClickUp - Create Task from Email`) set to `POST` on `https://api.clickup.com/api/v2/list/YOUR_CLICKUP_LIST_ID/task`. Configure body to map `task_title` and `task_description` from the AI output. Connect both the send-reply and save-draft Gmail nodes to this HTTP request.
3. **Calendar Conflict Pipeline Setup:**
   - Create a `Google Calendar` node (`Google Calendar - Get Upcoming Events`) set to `Get Many`, pulling events from the `primary` calendar between `$now.iso()` and `$now.plus({hours: 24}).toISO()`.
   - Create a Code node (`Detect Calendar Conflicts`) with JavaScript code sorting events chronologically and pushing overlapping pairs (`eventA`, `eventB`) into an output array.
   - Create an IF node (`Conflicts Found?`) checking if `$items().length > 0`.
   - Create an AI Agent node (`AI Agent - Reschedule Proposal`) connected to the IF node's true branch.
   - Create an OpenAI Chat Model node (`OpenAI Model - Reschedule Agent`) using `gpt-4o-mini` (temperature `0.2`) linked to the reschedule agent.
   - Create a `Slack` node (`Slack - Request Reschedule Approval`) set to `Send and Wait` targeting channel `C0AN1UGL0RM`.
   - Create a `Google Calendar` node (`Google Calendar - Update Event`) set to `Update` using event ID from `eventA`.
4. **Slack Mention Triage Pipeline Setup:**
   - Create a `Slack` node (`Slack - Get Recent Mentions`) with resource `Channel` and operation `History`, filtering with `oldest` timestamp set to 15 minutes ago.
   - Create an AI Agent node (`AI Agent - Slack Triage`) linked to the Slack history node.
   - Create an OpenAI Chat Model node (`OpenAI Model - Slack Triage`) using `gpt-4o-mini` (temperature `0.2`) linked to the Slack triage agent.
   - Create an IF node (`Needs Approval? (Slack)`) evaluating `{{ $json.output.requires_approval === true }}`.
   - Connect the `true` branch to a `Slack` node (`Slack - Request Response Approval`) using `Send and Wait`.
   - Connect both the approval node and the `false` branch to a final `Slack` node (`Slack - Send Reply`) set to post text to channel `C0AN1UGL0RM`.
5. **Morning Briefing Pipeline Setup:**
   - Create an IF node (`Is Briefing Time? (7AM)`) evaluating `{{ $now.hour === 7 }}` connected from the schedule trigger.
   - Create an HTTP Request node (`ClickUp - Get Tasks Due Today`) set to `GET` on `https://api.clickup.com/api/v2/list/YOUR_CLICKUP_LIST_ID/task` with query parameters `due_date_gt` and `due_date_lt` for start and end of day.
   - Create an AI Agent node (`AI Agent - Compile Morning Briefing`) connected to the ClickUp task request.
   - Create an OpenAI Chat Model node (`OpenAI Model - Briefing Agent`) using `gpt-4o-mini` (temperature `0.2`) linked to the briefing agent.
   - Connect the briefing agent output in parallel to:
     - A `Gmail` node (`Gmail - Send Morning Briefing`) sending to `me`.
     - A `Slack` node (`Slack - Post Morning Briefing`) posting to channel `C0AN1UGL0RM`.
6. **Credential Assignment:**
   - Assign valid OAuth2 credentials for Gmail, Slack, and Google Calendar across all respective integration nodes.
   - Assign OpenAI API credentials to all four language model nodes.
   - Assign Generic HTTP Header Authentication credentials for ClickUp API requests.
   - Replace placeholder string `YOUR_CLICKUP_LIST_ID` and channel ID `C0AN1UGL0RM` with production values.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Gmail OAuth2 Configuration | Required across all email read, reply, draft, and briefing nodes. |
| Slack OAuth2 & API Setup | Required for channel history reading, interactive approvals, responses, and briefing posts. |
| Google Calendar OAuth2 Setup | Required for fetching upcoming events and updating event slots. |
| OpenAI API Key | Required for all four LangChain AI agent language models (`gpt-4o-mini`). |
| ClickUp API Authentication | Required for task creation and fetching tasks due today via HTTP Request nodes. Ensure list ID placeholders are updated. |