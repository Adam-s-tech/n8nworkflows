Manage dental appointments via ElevenLabs voice, OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/manage-dental-appointments-via-elevenlabs-voice--openai-and-google-sheets-20035


# Manage dental appointments via ElevenLabs voice, OpenAI and Google Sheets

### 1. Workflow Overview

This workflow functions as an AI-powered dental clinic voice receptionist that handles inbound requests from external voice platforms (such as ElevenLabs via webhook). It processes three core intents—booking, rescheduling, and cancelling appointments—utilizing OpenAI language models equipped with Google Sheets tools to query, update, or remove appointment records. Once the targeted database action completes successfully, a confirmation email can be dispatched via Gmail, and the final status is returned via webhook response to relay back to the user.

The logical execution paths are grouped into the following functional blocks:
- **1.1 Input Reception & Routing:** Receives external POST webhooks containing patient details and routes execution using condition-based switching.
- **1.2 Reschedule Processing:** Extracts reschedule parameters, invokes an OpenAI agent to orchestrate the update, and interacts with Google Sheets to locate and modify existing records.
- **1.3 Booking Processing:** Parses booking parameters, executes an OpenAI agent to validate or append new appointment slots to Google Sheets, sends a confirmation email via Gmail, and relays responses.
- **1.4 Cancellation Processing:** Captures cancellation payloads, uses an OpenAI agent to search and verify specific row constraints, and deletes target entries in Google Sheets.
- **1.5 Final Response:** Aggregates and returns all incoming items back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Routing
- **Overview:** Receives incoming HTTP POST requests containing conversation parameters (intent, names, timestamps, contact info) and directs the payload down the appropriate processing branch.
- **Nodes Involved:** `Webhook`, `Switch`

##### Node Details:
- **Webhook**
  - **Type & Role:** `n8n-nodes-base.webhook` (Trigger) — Accepts external inbound HTTP POST requests.
  - **Configuration:** HTTP method set to `POST` with a designated webhook path identifier.
  - **Expressions:** None (triggers execution).
  - **Connections:** Output connects to `Switch`.
  - **Version Requirements:** Type version 2.1.
  - **Failure Types:** Network timeouts, missing payload body, or unparsable JSON structures from the caller.

- **Switch**
  - **Type & Role:** `n8n-nodes-base.switch` (Flow Control) — Evaluates the incoming intent parameter and routes traffic.
  - **Configuration:** Evaluates `{{ $json.body.intent }}` against three rules: `book_appointment`, `reschedule_appointment`, and `cancel_appointment`.
  - **Expressions:** `={{ $json.body.intent }}`
  - **Connections:** Inputs from `Webhook`; outputs route respectively to `Get Data1` (Booking), `Get Data` (Rescheduling), and `Fetch Data` (Cancellation).
  - **Version Requirements:** Type version 3.4.
  - **Failure Types:** Unrecognized string values dropping through unmatched routing paths.

---

#### 1.2 Reschedule Processing
- **Overview:** Normalizes reschedule input variables, passes context into an OpenAI agent configured with scheduling system instructions, and uses Google Sheets tools to fetch and update calendar data.
- **Nodes Involved:** `Get Data`, `Rescheduling Appointment`, `Model`, `Get Appointment2`, `Reschedule Appointment`

##### Node Details:
- **Get Data**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Extracts and structures incoming parameters for the rescheduling agent.
  - **Configuration:** Maps properties (`Full Name`, `Phone Number`, `new_date`, `new_time`, `reason_for_rescheduling_appointment`, `intent`) from the webhook body.
  - **Expressions:** `={{ $json.body.full_name }}`, `={{ $json.body.phone_number }}`, `={{ $json.body.new_date }}`, `={{ $json.body.new_time }}`, `={{ $json.body.reason_for_rescheduling_appointment }}`, `={{ $json.body.intent }}`
  - **Connections:** Input from `Switch`; output connects to `Rescheduling Appointment`.
  - **Version Requirements:** Type version 3.4.
  - **Failure Types:** Missing properties in the payload returning empty values.

- **Rescheduling Appointment**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Manages the logic of collecting details and utilizing tools to update schedules.
  - **Configuration:** Uses a custom system message defining agent guidelines for rescheduling and constructs an execution prompt embedding extracted body variables. Connected to a language model and tool nodes.
  - **Expressions:** Dynamic template strings injecting fields like `{{ $json['Full Name'] }}` and `{{ $json.new_date }}`.
  - **Connections:** Inputs from `Get Data` and `Model` (AI model); tool inputs from `Get Appointment2` and `Reschedule Appointment`; output connects to `Respond to Webhook`.
  - **Version Requirements:** Type version 3.1.
  - **Failure Types:** Hallucinated parameters, missing tool arguments, or API rate limits from the language model provider.

- **Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider) — Supplies the underlying LLM engine for the rescheduling agent.
  - **Configuration:** Uses OpenAI model variant `gpt-4.1`.
  - **Expressions:** None.
  - **Connections:** Output links to the AI model input of `Rescheduling Appointment`.
  - **Version Requirements:** Type version 1.3.
  - **Failure Types:** OpenAI API authentication failures, quota exhaustion, or gateway timeouts.

- **Get Appointment2**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Searches Google Sheets records to match existing appointments prior to modification.
  - **Configuration:** Points to document ID `19Z3W8FQrkSM7ODCLL_SQDdxbh5OmmY6ZgciwIqrXOfQ`, sheet `Sheet1` (gid=0).
  - **Expressions:** None (parameters controlled dynamically via AI tool calling).
  - **Connections:** Connected as an `ai_tool` input to `Rescheduling Appointment`.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Spreadsheet permission revoked, missing headers, or API quota limits.

- **Reschedule Appointment**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Appends or updates the target spreadsheet record with updated scheduling data.
  - **Configuration:** Operation set to `appendOrUpdate` matching on column `Full Name`. Maps schema fields (`Email`, `Full Name`, `Phone Number`, `Preferred Date`, `Preferred Time`, `Reason For Booking Appointment`).
  - **Expressions:** `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI(...) }}` bindings for mapping tool inputs.
  - **Connections:** Connected as an `ai_tool` input to `Rescheduling Appointment`.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Schema mismatch, unique constraint conflicts, or write failures.

---

#### 1.3 Booking Processing
- **Overview:** Formats appointment booking payloads, coordinates an OpenAI booking assistant, validates existing appointments via Google Sheets, updates records, and triggers email notifications.
- **Nodes Involved:** `Get Data1`, `Booking Appointment `, `OpenAI`, `Get Appointments`, `Book Appointment`, `Send Email`

##### Node Details:
- **Get Data1**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Formats incoming JSON body properties for the booking flow.
  - **Configuration:** Sets explicit assignment mappings for `Full Name`, `Phone Number`, `Email`, `Reason For Booking Appointment`, `Preferred Day`, `Preferred Time`, and `intent`.
  - **Expressions:** `={{ $json.body.full_name }}`, `={{ $json.body.phone_number }}`, `={{ $json.body.email }}`, etc.
  - **Connections:** Input from `Switch`; output connects to `Booking Appointment `.
  - **Version Requirements:** Type version 3.4.
  - **Failure Types:** Property lookup errors if expected payload fields are omitted.

- **Booking Appointment **
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Acts as the ABZ Dental clinic booking assistant.
  - **Configuration:** Uses a system prompt instructing the model to collect required scheduling details, review availability using tools, book slots, and trigger email dispatches.
  - **Expressions:** Template strings mapping incoming parameters into the text prompt.
  - **Connections:** Inputs from `Get Data1` and `OpenAI`; tool inputs from `Get Appointments`, `Book Appointment`, and `Send Email`; output connects to `Respond to Webhook`.
  - **Version Requirements:** Type version 3.1.
  - **Failure Types:** Incomplete parameter extraction leading to tool execution validation failures.

- **OpenAI**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider) — Provides the LLM backbone for the booking agent.
  - **Configuration:** Uses model `gpt-4.1`.
  - **Expressions:** None.
  - **Connections:** Output links to the AI model input of `Booking Appointment `.
  - **Version Requirements:** Type version 1.3.
  - **Failure Types:** Authentication or token limit errors.

- **Get Appointments**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Queries existing spreadsheet records to check availability.
  - **Configuration:** Targets spreadsheet ID `19Z3W8FQrkSM7ODCLL_SQDdxbh5OmmY6ZgciwIqrXOfQ`, sheet `Sheet1`.
  - **Expressions:** None.
  - **Connections:** Connected as an `ai_tool` input to `Booking Appointment `.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Read permission errors or network timeouts communicating with Google API.

- **Book Appointment**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Appends or updates booked appointment records in Google Sheets.
  - **Configuration:** Operation set to `appendOrUpdate` matched on `Full Name`. Maps fields for Email, Full Name, Phone Number, Preferred Date, Preferred Time, and Reason.
  - **Expressions:** Automated tool extraction variables via `$fromAI`.
  - **Connections:** Connected as an `ai_tool` input to `Booking Appointment `.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Write errors or row-locking conflicts.

- **Send Email**
  - **Type & Role:** `n8n-nodes-base.gmailTool` (AI Tool / Integration) — Sends confirmation emails to users upon booking completion.
  - **Configuration:** Maps recipient (`sendTo`), subject, and message body parameters derived from AI agent calls. Attribution appending disabled.
  - **Expressions:** AI-injected parameter bindings using `$fromAI`.
  - **Connections:** Connected as an `ai_tool` input to `Booking Appointment `.
  - **Version Requirements:** Type version 2.2.
  - **Failure Types:** Invalid recipient email addresses, OAuth token expiration, or Gmail API sending limits.

---

#### 1.4 Cancellation Processing
- **Overview:** Extracts cancellation details from the payload, passes instructions to a cancellation agent, and utilizes Google Sheets tools to locate and delete matching row entries.
- **Nodes Involved:** `Fetch Data`, `Cancelling Appointment`, `Chat Model`, `Get Appointment`, `Cancel Appointment`

##### Node Details:
- **Fetch Data**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Standardizes incoming JSON properties for appointment cancellations.
  - **Configuration:** Maps `Full Name`, `Reason For Cancelling Appointment`, `Phone Number`, and `intent`.
  - **Expressions:** `={{ $json.body.fulll_name }}`, `={{ $json.body.reason_for_canceling_apppointment }}`, `={{ $json.body.phone_number }}`, `={{ $json.body.intent }}`
  - **Connections:** Input from `Switch`; output connects to `Cancelling Appointment`.
  - **Version Requirements:** Type version 3.4.
  - **Failure Types:** Typographical mismatches in property lookups (e.g., `fulll_name`).

- **Cancelling Appointment**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Coordinates appointment deletion workflows based on user input.
  - **Configuration:** Enforces system instructions requiring the agent to search for records first via tool lookup rather than guessing row indices.
  - **Expressions:** Template prompt interpolation for extracted variables.
  - **Connections:** Inputs from `Fetch Data` and `Chat Model`; tool inputs from `Get Appointment` and `Cancel Appointment`; output connects to `Respond to Webhook`.
  - **Version Requirements:** Type version 3.1.
  - **Failure Types:** Agent failing to retrieve matching row IDs before calling deletion tools.

- **Chat Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider) — Supplies the language model engine for the cancellation agent.
  - **Configuration:** Uses model variant `gpt-4o-mini`.
  - **Expressions:** None.
  - **Connections:** Output links to the AI model input of `Cancelling Appointment`.
  - **Version Requirements:** Type version 1.3.
  - **Failure Types:** OpenAI API failures or rate limits.

- **Get Appointment**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Searches Google Sheets to identify the specific row index associated with a patient's appointment.
  - **Configuration:** Targets spreadsheet ID `19Z3W8FQrkSM7ODCLL_SQDdxbh5OmmY6ZgciwIqrXOfQ`, sheet `Sheet1`.
  - **Expressions:** None.
  - **Connections:** Connected as an `ai_tool` input to `Cancelling Appointment`.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Missing search terms or unindexed data structures.

- **Cancel Appointment**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (AI Tool / Integration) — Deletes identified rows from the target spreadsheet.
  - **Configuration:** Operation set to `delete`. Requires start row index and number of rows to delete.
  - **Expressions:** `$fromAI('Start_Row_Number', ...)` and `$fromAI('Number_of_Rows_to_Delete', ...)`.
  - **Connections:** Connected as an `ai_tool` input to `Cancelling Appointment`.
  - **Version Requirements:** Type version 4.7.
  - **Failure Types:** Invalid row index numbers or missing permissions.

---

#### 1.5 Final Response
- **Overview:** Captures processed output data from all branching agent executions and returns the final items in the webhook HTTP response.
- **Nodes Involved:** `Respond to Webhook`

##### Node Details:
- **Respond to Webhook**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` (Response) — Sends HTTP responses back to the original caller (e.g., ElevenLabs).
  - **Configuration:** Respond option set to `allIncomingItems`.
  - **Expressions:** None.
  - **Connections:** Inputs from `Rescheduling Appointment`, `Booking Appointment `, and `Cancelling Appointment`.
  - **Version Requirements:** Type version 1.5.
  - **Failure Types:** Connection drops or timeouts if upstream agents take too long to respond.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | n8n-nodes-base.webhook | Trigger inbound HTTP requests | None | Switch | Receives the request from ElevenLabs. Checks the request and sends it to the correct part of the workflow. |
| Switch | n8n-nodes-base.switch | Route traffic based on intent | Webhook | Get Data1, Get Data, Fetch Data | Receives the request from ElevenLabs. Checks the request and sends it to the correct part of the workflow. |
| Get Data | n8n-nodes-base.set | Format reschedule variables | Switch | Rescheduling Appointment | This ai agent reschedules a dental appointment when the patient requests a new date and time. It uses the provided patient details to identify and update the appointment. |
| Rescheduling Appointment | @n8n/n8n-nodes-langchain.agent | Orchestrate reschedule flow | Get Data, Model | Respond to Webhook | This ai agent reschedules a dental appointment when the patient requests a new date and time. It uses the provided patient details to identify and update the appointment. |
| Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM Provider for rescheduling | None | Rescheduling Appointment | This ai agent reschedules a dental appointment when the patient requests a new date and time. It uses the provided patient details to identify and update the appointment. |
| Get Appointment2 | n8n-nodes-base.googleSheetsTool | Fetch sheet records for reschedule | None | Rescheduling Appointment | This ai agent reschedules a dental appointment when the patient requests a new date and time. It uses the provided patient details to identify and update the appointment. |
| Reschedule Appointment | n8n-nodes-base.googleSheetsTool | Update sheet record | None | Rescheduling Appointment | This ai agent reschedules a dental appointment when the patient requests a new date and time. It uses the provided patient details to identify and update the appointment. |
| Get Data1 | n8n-nodes-base.set | Format booking variables | Switch | Booking Appointment | This agent books a dental appointment using the patient’s details and preferred date and time. |
| Booking Appointment | @n8n/n8n-nodes-langchain.agent | Orchestrate booking flow | Get Data1, OpenAI | Respond to Webhook | This agent books a dental appointment using the patient’s details and preferred date and time. |
| OpenAI | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM Provider for booking | None | Booking Appointment | This agent books a dental appointment using the patient’s details and preferred date and time. |
| Get Appointments | n8n-nodes-base.googleSheetsTool | Fetch available appointments | None | Booking Appointment | This agent books a dental appointment using the patient’s details and preferred date and time. |
| Book Appointment | n8n-nodes-base.googleSheetsTool | Append/update new booking | None | Booking Appointment | This agent books a dental appointment using the patient’s details and preferred date and time. |
| Send Email | n8n-nodes-base.gmailTool | Send booking confirmation email | None | Booking Appointment | This agent books a dental appointment using the patient’s details and preferred date and time. |
| Fetch Data | n8n-nodes-base.set | Format cancellation variables | Switch | Cancelling Appointment | This agent cancels a dental appointment using the patient’s details and existing appointment information |
| Cancelling Appointment | @n8n/n8n-nodes-langchain.agent | Orchestrate cancellation flow | Fetch Data, Chat Model | Respond to Webhook | This agent cancels a dental appointment using the patient’s details and existing appointment information |
| Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM Provider for cancellation | None | Cancelling Appointment | This agent cancels a dental appointment using the patient’s details and existing appointment information |
| Get Appointment | n8n-nodes-base.googleSheetsTool | Search record for cancellation | None | Cancelling Appointment | This agent cancels a dental appointment using the patient’s details and existing appointment information |
| Cancel Appointment | n8n-nodes-base.googleSheetsTool | Delete spreadsheet row | None | Cancelling Appointment | This agent cancels a dental appointment using the patient’s details and existing appointment information |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return output to caller | Rescheduling Appointment, Booking Appointment , Cancelling Appointment | None | Send The Final Response Back To Elevenlabs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`). Set HTTP Method to `POST`. Note the generated webhook URL.
2. **Add Router Logic:**
   - Create a **Switch** node (`n8n-nodes-base.switch`). Connect `Webhook` output to `Switch`.
   - Configure three rules checking `={{ $json.body.intent }}` equal to:
     1. `book_appointment` (Output 0)
     2. `reschedule_appointment` (Output 1)
     3. `cancel_appointment` (Output 2)
3. **Build the Rescheduling Branch:**
   - Add a **Set** node named `Get Data` and assign variables (`Full Name`, `Phone Number`, `new_date`, `new_time`, `reason_for_rescheduling_appointment`, `intent`) mapped from `{{ $json.body.* }}`. Connect Switch Output 1 here.
   - Add an **AI Agent** named `Rescheduling Appointment` (`@n8n/n8n-nodes-langchain.agent`). Connect `Get Data` to its main input.
   - Add an OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with model `gpt-4.1` and connect it to the agent's model input. Configure OpenAI credentials.
   - Add two **Google Sheets Tool** nodes: `Get Appointment2` and `Reschedule Appointment` (Operation: `appendOrUpdate`, matching on `Full Name`, Document ID: `19Z3W8FQrkSM7ODCLL_SQDdxbh5OmmY6ZgciwIqrXOfQ`, Sheet: `Sheet1`). Connect both as AI tools to `Rescheduling Appointment`.
4. **Build the Booking Branch:**
   - Add a **Set** node named `Get Data1` and map booking properties (`Full Name`, `Phone Number`, `Email`, `Reason For Booking Appointment`, `Preferred Day`, `Preferred Time`, `intent`) from `{{ $json.body.* }}`. Connect Switch Output 0 here.
   - Add an **AI Agent** named `Booking Appointment ` (`@n8n/n8n-nodes-langchain.agent`). Connect `Get Data1` to its main input.
   - Add an OpenAI Chat Model node (`gpt-4.1`) connected to the agent's model input. Configure OpenAI credentials.
   - Add two **Google Sheets Tool** nodes: `Get Appointments` and `Book Appointment` (Operation: `appendOrUpdate`, matching on `Full Name`, Document ID: `19Z3W8FQrkSM7ODCLL_SQDdxbh5OmmY6ZgciwIqrXOfQ`, Sheet: `Sheet1`). Connect as AI tools.
   - Add a **Gmail Tool** node (`n8n-nodes-base.gmailTool`) configured with Gmail OAuth2 credentials, mapping recipient, subject, and message fields. Connect as an AI tool to `Booking Appointment `.
5. **Build the Cancellation Branch:**
   - Add a **Set** node named `Fetch Data` and map cancellation fields (`Full Name`, `Reason For Cancelling Appointment`, `Phone Number`, `intent`) from `{{ $json.body.* }}`. Connect Switch Output 2 here.
   - Add an **AI Agent** named `Cancelling Appointment` (`@n8n/n8n-nodes-langchain.agent`). Connect `Fetch Data` to its main input.
   - Add an OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with model `gpt-4o-mini` connected to the agent model input.
   - Add a **Google Sheets Tool** (`Get Appointment`) to search records, and another **Google Sheets Tool** (`Cancel Appointment`, Operation: `delete`) mapped to start row index and number of rows to delete via `$fromAI`. Connect both as AI tools to `Cancelling Appointment`.
6. **Configure Final Output:**
   - Add a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`) with response mode set to `allIncomingItems`.
   - Connect the main outputs of `Rescheduling Appointment`, `Booking Appointment `, and `Cancelling Appointment` to `Respond to Webhook`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video Tutorial Reference | [YouTube Video Link](https://www.youtube.com/watch?v=AlNnFHd96gw) |
| Dental AI Voice Receptionist Overview | Initial setup guide for integrating ElevenLabs voice agents with n8n, Google Sheets, and Google Calendar/Gmail. |