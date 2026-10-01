Collect student absence reasons via WhatsApp, Google Sheets, Gemini, and Redis

https://n8nworkflows.xyz/workflows/collect-student-absence-reasons-via-whatsapp--google-sheets--gemini--and-redis-19812


# Collect student absence reasons via WhatsApp, Google Sheets, Gemini, and Redis

### 1. Workflow Overview

This workflow automates the collection and logging of student absence reasons by integrating Google Sheets, WhatsApp Business API, Redis, and Google Gemini AI. It operates across two primary functional pathways: a daily outbound notification trigger and an inbound message processing pipeline featuring robust deduplication, message buffering, concurrency locking, and generative AI classification.

The logic is grouped into the following functional blocks:
- **1.1 Outbound Attendance Check & Notification:** Scheduled daily execution to identify absent students lacking a recorded reason and dispatch initial WhatsApp outreach to parents.
- **1.2 Inbound WhatsApp Reception & Validation:** Captures incoming webhooks from WhatsApp, isolates message payloads, and validates that the message format is text.
- **1.3 Redis-Based Deduplication & Buffering:** Employs Redis keys to drop duplicate webhooks, append rapid incoming replies into a temporary buffer, and debounce incoming traffic.
- **1.4 Concurrency Locking & Student Record Lookup:** Acquires a Redis lock per phone number to prevent race conditions, aggregates the buffered messages, and queries Google Sheets to fetch the corresponding student details.
- **1.5 AI Classification & Google Sheets Update:** Passes the context and conversation history to a Google Gemini LangChain agent structured with an output parser, then updates Google Sheets and dispatches confirmation or clarification responses via WhatsApp.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Outbound Attendance Check & Notification
- **Overview:** Triggers every day at a set time, reads the attendance tracking sheet, identifies absent students without a documented reason, sends an automated inquiry via WhatsApp, and logs the notice in Redis.
- **Nodes Involved:**
  - `When Every Day at 10:30 AM`
  - `Retrieve Absent Students from Sheets`
  - `Check If Reason is Missing`
  - `Send Absence Notice on WhatsApp`
  - `Store Notice in Redis`

- **Node Details:**
  - **When Every Day at 10:30 AM**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Initiates the daily workflow schedule.
    - *Configuration Choices:* Configured for daily execution at 10:30 AM.
    - *Inputs / Outputs:* Inputs: None. Outputs: Triggers `Retrieve Absent Students from Sheets`.
  - **Retrieve Absent Students from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Action). Reads rows from the attendance tracker spreadsheet.
    - *Configuration Choices:* Operation set to "Get Many" or "Lookup" targeting the attendance sheet range.
    - *Inputs / Outputs:* Inputs: `When Every Day at 10:30 AM`. Outputs: `Check If Reason is Missing`.
  - **Check If Reason is Missing**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates whether the absence reason field is empty.
    - *Configuration Choices:* Evaluates `{{ $json.Reason }}` or equivalent empty-check expression.
    - *Inputs / Outputs:* Inputs: `Retrieve Absent Students from Sheets`. Outputs: True branch goes to `Send Absence Notice on WhatsApp`.
  - **Send Absence Notice on WhatsApp**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Communication Action). Sends outbound template or text messages to parents.
    - *Configuration Choices:* Uses WhatsApp Business Cloud API to transmit the absence inquiry using parent contact parameters.
    - *Inputs / Outputs:* Inputs: `Check If Reason is Missing` (True). Outputs: `Store Notice in Redis`.
    - *Failure Types:* Invalid phone number format, expired template, or WhatsApp API authorization failure.
  - **Store Notice in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Database/Cache). Records sent notification state in Redis.
    - *Configuration Choices:* Executes command to set key-value pairs regarding the outbound notification.
    - *Inputs / Outputs:* Inputs: `Send Absence Notice on WhatsApp`. Outputs: None (terminal for this branch).

---

#### 2.2 Inbound WhatsApp Reception & Validation
- **Overview:** Listens for inbound parent replies via WhatsApp webhook, formats the payload data, and ensures the received message is text before proceeding.
- **Nodes Involved:**
  - `On WhatsApp Message Reception`
  - `Set Message Details`
  - `Verify Message Contains Text`
  - `Confirm Message is Text Type`
  - `Warn Non-Text Message on WhatsApp`

- **Node Details:**
  - **On WhatsApp Message Reception**
    - *Type and Technical Role:* `n8n-nodes-base.whatsAppTrigger` (Webhook Trigger). Captures incoming webhook events from the WhatsApp Cloud API.
    - *Configuration Choices:* Listens for incoming message events on the configured webhook ID.
    - *Inputs / Outputs:* Inputs: External WhatsApp webhook. Outputs: `Set Message Details`.
  - **Set Message Details**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Normalizes incoming webhook fields into structured variables (sender phone number, message body, message ID).
    - *Configuration Choices:* Maps JSON paths from the WhatsApp payload to clean property names.
    - *Inputs / Outputs:* Inputs: `On WhatsApp Message Reception`. Outputs: `Verify Message Contains Text`.
  - **Verify Message Contains Text**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Validates existence of text data.
    - *Configuration Choices:* Checks if the message type field equals "text".
    - *Inputs / Outputs:* Inputs: `Set Message Details`. Outputs: `Confirm Message is Text Type`.
  - **Confirm Message is Text Type**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Directs execution depending on whether the payload contains valid text.
    - *Configuration Choices:* Conditional split based on text presence.
    - *Inputs / Outputs:* Inputs: `Verify Message Contains Text`. Outputs: True branch goes to `Verify Dedup Key in Redis`; False branch goes to `Warn Non-Text Message on WhatsApp`.
  - **Warn Non-Text Message on WhatsApp**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Communication Action). Replies to the user when an unsupported media type (images, audio, video) is received.
    - *Configuration Choices:* Sends a pre-configured text notification instructing the parent to reply using text only.
    - *Inputs / Outputs:* Inputs: `Confirm Message is Text Type` (False). Outputs: None (terminal branch).

---

#### 2.3 Redis-Based Deduplication & Buffering
- **Overview:** Protects the workflow against webhook retries using deduplication keys, and buffers rapid sequential messages from the same sender using Redis lists and timestamps.
- **Nodes Involved:**
  - `Verify Dedup Key in Redis`
  - `Determine Message Duplication`
  - `Ignore Duplicate Message Entry`
  - `Log Dedup Key in Redis`
  - `Retrieve Message Buffer from Redis`
  - `Save Message to Buffer`
  - `Store Message Buffer in Redis`
  - `Log Message Timestamp in Redis`
  - `Pause 1s for Debounce`
  - `Retrieve Latest Timestamp from Redis`
  - `Verify Latest Message Status`
  - `Abort Stale Message Process`

- **Node Details:**
  - **Verify Dedup Key in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Check). Checks if a unique message ID key already exists in Redis.
    - *Configuration Choices:* Redis `EXISTS` or `GET` command using the WhatsApp message ID.
    - *Inputs / Outputs:* Inputs: `Confirm Message is Text Type` (True). Outputs: `Determine Message Duplication`.
  - **Determine Message Duplication**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Splits execution based on whether the dedup key was found.
    - *Configuration Choices:* Evaluates existence check result.
    - *Inputs / Outputs:* Inputs: `Verify Dedup Key in Redis`. Outputs: True branch (is duplicate) goes to `Ignore Duplicate Message Entry`; False branch goes to `Log Dedup Key in Redis`.
  - **Ignore Duplicate Message Entry**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control). Terminates processing for duplicate webhook deliveries.
    - *Inputs / Outputs:* Inputs: `Determine Message Duplication` (True). Outputs: None.
  - **Log Dedup Key in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Set). Writes the message ID key to Redis with an expiration TTL to prevent duplicate processing.
    - *Inputs / Outputs:* Inputs: `Determine Message Duplication` (False). Outputs: `Retrieve Message Buffer from Redis`.
  - **Retrieve Message Buffer from Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Read). Fetches existing buffered messages for the sender.
    - *Inputs / Outputs:* Inputs: `Log Dedup Key in Redis`. Outputs: `Save Message to Buffer`.
  - **Save Message to Buffer**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Appends the new message text to the retrieved message array/buffer.
    - *Inputs / Outputs:* Inputs: `Retrieve Message Buffer from Redis`. Outputs: `Store Message Buffer in Redis`.
  - **Store Message Buffer in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Set). Saves the updated message buffer back to Redis.
    - *Inputs / Outputs:* Inputs: `Save Message to Buffer`. Outputs: `Log Message Timestamp in Redis`.
  - **Log Message Timestamp in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Set). Records the timestamp of the latest message arrival.
    - *Inputs / Outputs:* Inputs: `Store Message Buffer in Redis`. Outputs: `Pause 1s for Debounce`.
  - **Pause 1s for Debounce**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control). Pauses execution briefly to allow rapid multi-message grouping (debouncing).
    - *Configuration Choices:* 1-second delay timer.
    - *Inputs / Outputs:* Inputs: `Log Message Timestamp in Redis`. Outputs: `Retrieve Latest Timestamp from Redis`.
  - **Retrieve Latest Timestamp from Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Read). Fetches the latest timestamp stored for this sender.
    - *Inputs / Outputs:* Inputs: `Pause 1s for Debounce`. Outputs: `Verify Latest Message Status`.
  - **Verify Latest Message Status**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Compares timestamps to verify if this execution stream represents the most recent message batch (discarding stale overlapping runs).
    - *Inputs / Outputs:* Inputs: `Retrieve Latest Timestamp from Redis`. Outputs: True branch goes to `Check Processing Lock in Redis`; False branch goes to `Abort Stale Message Process`.
  - **Abort Stale Message Process**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control). Terminates older, overlapping execution instances.
    - *Inputs / Outputs:* Inputs: `Verify Latest Message Status` (False). Outputs: None.

---

#### 2.4 Concurrency Locking & Student Record Lookup
- **Overview:** Acquires an execution lock in Redis to ensure serial processing per phone number, extracts the combined message buffer, clears the temporary storage, and queries Google Sheets for the student profile.
- **Nodes Involved:**
  - `Check Processing Lock in Redis`
  - `Detect Active Processing Lock`
  - `Cancel Locked Processing`
  - `Gain Processing Lock in Redis`
  - `Acquire Final Buffer from Redis`
  - `Create Combined Message Buffer`
  - `Remove Buffer from Redis`
  - `Read Student Data from Sheets`
  - `Check Student Reason Presence`

- **Node Details:**
  - **Check Processing Lock in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Check). Checks if a processing lock key exists for the sender's phone number.
    - *Inputs / Outputs:* Inputs: `Verify Latest Message Status` (True). Outputs: `Detect Active Processing Lock`.
  - **Detect Active Processing Lock**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates lock status.
    - *Inputs / Outputs:* Inputs: `Check Processing Lock in Redis`. Outputs: True branch goes to `Cancel Locked Processing`; False branch goes to `Gain Processing Lock in Redis`.
  - **Cancel Locked Processing**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control). Exits execution if another thread is currently processing messages for this user.
    - *Inputs / Outputs:* Inputs: `Detect Active Processing Lock` (True). Outputs: None.
  - **Gain Processing Lock in Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Set). Sets a short-lived concurrency lock key in Redis.
    - *Inputs / Outputs:* Inputs: `Detect Active Processing Lock` (False). Outputs: `Acquire Final Buffer from Redis`.
  - **Acquire Final Buffer from Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Read). Retrieves the consolidated message buffer contents.
    - *Inputs / Outputs:* Inputs: `Gain Processing Lock in Redis`. Outputs: `Create Combined Message Buffer`.
  - **Create Combined Message Buffer**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Merges individual message entries into a single cohesive string for AI evaluation.
    - *Inputs / Outputs:* Inputs: `Acquire Final Buffer from Redis`. Outputs: `Remove Buffer from Redis`.
  - **Remove Buffer from Redis**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Delete). Clears the message buffer key from Redis now that it has been read.
    - *Inputs / Outputs:* Inputs: `Create Combined Message Buffer`. Outputs: `Read Student Data from Sheets`.
  - **Read Student Data from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Action). Queries the attendance tracker sheet using the parent's phone number as the lookup key.
    - *Inputs / Outputs:* Inputs: `Remove Buffer from Redis`. Outputs: `Check Student Reason Presence`.
  - **Check Student Reason Presence**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Validates that a matching student record was found and retrieved.
    - *Inputs / Outputs:* Inputs: `Read Student Data from Sheets`. Outputs: True branch proceeds to `Classify Absence Reason`.

---

#### 2.5 AI Classification & Google Sheets Update
- **Overview:** Uses Google Gemini via a LangChain agent and Redis chat memory to analyze whether the parent's message constitutes a valid absence reason, updates the spreadsheet accordingly, and responds via WhatsApp.
- **Nodes Involved:**
  - `Classify Absence Reason`
  - `Google Gemini Classifier`
  - `Parse AI Response`
  - `Maintain Chat Memory in Redis`
  - `Assess If Reason Is Given`
  - `Update Student Data in Sheets`
  - `Dispatch Confirmation Message`
  - `Release Lock Post Confirmation`
  - `Request Reason Clarification`
  - `Free Lock After Clarification`

- **Node Details:**
  - **Classify Absence Reason**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Orchestrates the LLM, output parser, and chat memory to evaluate the message.
    - *Inputs / Outputs:* Inputs: `Check Student Reason Presence`, `Google Gemini Classifier`, `Parse AI Response`, `Maintain Chat Memory in Redis`. Outputs: `Assess If Reason Is Given`.
  - **Google Gemini Classifier**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model). Provides the underlying Google Gemini LLM capabilities.
    - *Inputs / Outputs:* Connected to `Classify Absence Reason`.
  - **Parse AI Response**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser). Enforces structured JSON output from the AI model (e.g., determining validity and extracting the reason text).
    - *Inputs / Outputs:* Connected to `Classify Absence Reason`.
  - **Maintain Chat Memory in Redis**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.memoryRedisChat` (AI Memory). Persists conversation context in Redis per user session.
    - *Inputs / Outputs:* Connected to `Classify Absence Reason`.
  - **Assess If Reason Is Given**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates the boolean result returned by the structured parser.
    - *Inputs / Outputs:* Inputs: `Classify Absence Reason`. Outputs: True branch goes to `Update Student Data in Sheets`; False branch goes to `Request Reason Clarification`.
  - **Update Student Data in Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Action). Updates the student row in Google Sheets with the validated absence reason and marks status flags (e.g., `got_reason`).
    - *Inputs / Outputs:* Inputs: `Assess If Reason Is Given` (True). Outputs: `Dispatch Confirmation Message`.
  - **Dispatch Confirmation Message**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Communication Action). Sends a confirmation message to the parent confirming receipt of the absence reason.
    - *Inputs / Outputs:* Inputs: `Update Student Data in Sheets`. Outputs: `Release Lock Post Confirmation`.
  - **Release Lock Post Confirmation**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Delete). Removes the processing lock key for the phone number.
    - *Inputs / Outputs:* Inputs: `Dispatch Confirmation Message`. Outputs: None (terminal).
  - **Request Reason Clarification**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Communication Action). Sends a follow-up message asking the parent to provide a valid, clearer reason for the absence.
    - *Inputs / Outputs:* Inputs: `Assess If Reason Is Given` (False). Outputs: `Free Lock After Clarification`.
  - **Free Lock After Clarification**
    - *Type and Technical Role:* `n8n-nodes-base.redis` (Cache Delete). Releases the processing lock key so subsequent replies can be processed.
    - *Inputs / Outputs:* Inputs: `Request Reason Clarification`. Outputs: None (terminal).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Every Day at 10:30 AM` | `scheduleTrigger` | Trigger daily check | None | `Retrieve Absent Students from Sheets` | |
| `Retrieve Absent Students from Sheets` | `googleSheets` | Read attendance sheet | `When Every Day at 10:30 AM` | `Check If Reason is Missing` | |
| `Check If Reason is Missing` | `if` | Check if reason is empty | `Retrieve Absent Students from Sheets` | `Send Absence Notice on WhatsApp` | |
| `Send Absence Notice on WhatsApp` | `whatsApp` | Send absence alert | `Check If Reason is Missing` | `Store Notice in Redis` | |
| `Store Notice in Redis` | `redis` | Log notice state | `Send Absence Notice on WhatsApp` | None | |
| `On WhatsApp Message Reception` | `whatsAppTrigger` | Inbound WhatsApp webhook | None | `Set Message Details` | |
| `Set Message Details` | `set` | Normalize payload | `On WhatsApp Message Reception` | `Verify Message Contains Text` | |
| `Verify Message Contains Text` | `if` | Check if message is text | `Set Message Details` | `Confirm Message is Text Type` | |
| `Confirm Message is Text Type` | `if` | Route text vs media | `Verify Message Contains Text` | `Verify Dedup Key in Redis`, `Warn Non-Text Message on WhatsApp` | |
| `Warn Non-Text Message on WhatsApp` | `whatsApp` | Send media warning | `Confirm Message is Text Type` | None | |
| `Verify Dedup Key in Redis` | `redis` | Check dedup key | `Confirm Message is Text Type` | `Determine Message Duplication` | |
| `Determine Message Duplication` | `if` | Branch on duplication | `Verify Dedup Key in Redis` | `Ignore Duplicate Message Entry`, `Log Dedup Key in Redis` | |
| `Ignore Duplicate Message Entry` | `noOp` | Drop duplicate | `Determine Message Duplication` | None | |
| `Log Dedup Key in Redis` | `redis` | Store dedup key | `Determine Message Duplication` | `Retrieve Message Buffer from Redis` | |
| `Retrieve Message Buffer from Redis` | `redis` | Fetch message buffer | `Log Dedup Key in Redis` | `Save Message to Buffer` | |
| `Save Message to Buffer` | `set` | Append message | `Retrieve Message Buffer from Redis` | `Store Message Buffer in Redis` | |
| `Store Message Buffer in Redis` | `redis` | Save message buffer | `Save Message to Buffer` | `Log Message Timestamp in Redis` | |
| `Log Message Timestamp in Redis` | `redis` | Store timestamp | `Store Message Buffer in Redis` | `Pause 1s for Debounce` | |
| `Pause 1s for Debounce` | `wait` | Debounce pause | `Log Message Timestamp in Redis` | `Retrieve Latest Timestamp from Redis` | |
| `Retrieve Latest Timestamp from Redis` | `redis` | Get latest timestamp | `Pause 1s for Debounce` | `Verify Latest Message Status` | |
| `Verify Latest Message Status` | `if` | Check execution order | `Retrieve Latest Timestamp from Redis` | `Check Processing Lock in Redis`, `Abort Stale Message Process` | |
| `Abort Stale Message Process` | `noOp` | Drop stale run | `Verify Latest Message Status` | None | |
| `Check Processing Lock in Redis` | `redis` | Check concurrency lock | `Verify Latest Message Status` | `Detect Active Processing Lock` | |
| `Detect Active Processing Lock` | `if` | Branch on lock state | `Check Processing Lock in Redis` | `Cancel Locked Processing`, `Gain Processing Lock in Redis` | |
| `Cancel Locked Processing` | `noOp` | Stop locked run | `Detect Active Processing Lock` | None | |
| `Gain Processing Lock in Redis` | `redis` | Set processing lock | `Detect Active Processing Lock` | `Acquire Final Buffer from Redis` | |
| `Acquire Final Buffer from Redis` | `redis` | Get final buffer | `Gain Processing Lock in Redis` | `Create Combined Message Buffer` | |
| `Create Combined Message Buffer` | `set` | Merge messages | `Acquire Final Buffer from Redis` | `Remove Buffer from Redis` | |
| `Remove Buffer from Redis` | `redis` | Clear buffer key | `Create Combined Message Buffer` | `Read Student Data from Sheets` | |
| `Read Student Data from Sheets` | `googleSheets` | Lookup student record | `Remove Buffer from Redis` | `Check Student Reason Presence` | |
| `Check Student Reason Presence` | `if` | Validate record found | `Read Student Data from Sheets` | `Classify Absence Reason` | |
| `Classify Absence Reason` | `agent` | AI Classification agent | `Check Student Reason Presence`, `Google Gemini Classifier`, `Parse AI Response`, `Maintain Chat Memory in Redis` | `Assess If Reason Is Given` | |
| `Google Gemini Classifier` | `lmChatGoogleGemini` | Gemini LLM provider | None | `Classify Absence Reason` | |
| `Parse AI Response` | `outputParserStructured` | Structured JSON parser | None | `Classify Absence Reason` | |
| `Maintain Chat Memory in Redis` | `memoryRedisChat` | Redis chat memory | None | `Classify Absence Reason` | |
| `Assess If Reason Is Given` | `if` | Check if reason valid | `Classify Absence Reason` | `Update Student Data in Sheets`, `Request Reason Clarification` | |
| `Update Student Data in Sheets` | `googleSheets` | Write reason to sheet | `Assess If Reason Is Given` | `Dispatch Confirmation Message` | |
| `Dispatch Confirmation Message` | `whatsApp` | Send success reply | `Update Student Data in Sheets` | `Release Lock Post Confirmation` | |
| `Release Lock Post Confirmation` | `redis` | Release user lock | `Dispatch Confirmation Message` | None | |
| `Request Reason Clarification` | `whatsApp` | Send clarification request | `Assess If Reason Is Given` | `Free Lock After Clarification` | |
| `Free Lock After Clarification` | `redis` | Release user lock | `Request Reason Clarification` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in an n8n instance:

1. **Prerequisites & Credentials Setup:**
   - Configure **Google Service Account** credentials with access to Google Sheets.
   - Configure **WhatsApp Business Cloud API** credentials (Phone Number ID and Access Token).
   - Configure **Redis** connection parameters (Host, Port, Password if applicable).
   - Configure **Google Gemini (Google PaLM)** API credentials.

2. **Build Block 1: Daily Outbound Notification**
   - Create a **Schedule Trigger** node (`When Every Day at 10:30 AM`) set to run daily at 10:30 AM.
   - Add a **Google Sheets** node (`Retrieve Absent Students from Sheets`) set to *Get Many*, connecting to your attendance spreadsheet.
   - Add an **If** node (`Check If Reason is Missing`) to check if the absence reason column is empty.
   - Connect the True output to a **WhatsApp** node (`Send Absence Notice on WhatsApp`) configured to send the notification template to `{{ $json.ParentPhoneNumber }}`.
   - Connect the WhatsApp node to a **Redis** node (`Store Notice in Redis`) to log the notice state.

3. **Build Block 2: Inbound Message Reception**
   - Create a **WhatsApp Trigger** node (`On WhatsApp Message Reception`) configured with your webhook ID.
   - Add a **Set** node (`Set Message Details`) to extract `from` (phone number), `text.body`, and `id` (message ID).
   - Add an **If** node (`Verify Message Contains Text`) and subsequent **If** node (`Confirm Message is Text Type`) to ensure `type === 'text'`.
   - On the False branch, add a **WhatsApp** node (`Warn Non-Text Message on WhatsApp`) to reply asking for text input.

4. **Build Block 3: Deduplication & Buffering**
   - On the True branch of text verification, add a **Redis** node (`Verify Dedup Key in Redis`) to check if the message ID exists.
   - Add an **If** node (`Determine Message Duplication`). If duplicate, route to a **NoOp** node (`Ignore Duplicate Message Entry`).
   - If not duplicate, route to a **Redis** node (`Log Dedup Key in Redis`) to save the message ID with a TTL.
   - Chain subsequent **Redis** and **Set** nodes to retrieve the message list, append the new message, store the buffer back in Redis, and log the timestamp.
   - Add a **Wait** node (`Pause 1s for Debounce`) configured for a 1-second delay.
   - Add a **Redis** node (`Retrieve Latest Timestamp from Redis`) and an **If** node (`Verify Latest Message Status`) to drop stale execution runs (route False to an **Abort Stale Message Process** NoOp node).

5. **Build Block 4: Concurrency Locking & Sheet Lookup**
   - Route the valid status branch to a **Redis** node (`Check Processing Lock in Redis`) and an **If** node (`Detect Active Processing Lock`).
   - Route the active lock condition to a **NoOp** node (`Cancel Locked Processing`).
   - Route the inactive lock condition to a **Redis** node (`Gain Processing Lock in Redis`), followed by acquiring the final buffer (`Acquire Final Buffer from Redis`), combining messages (`Create Combined Message Buffer`), and removing the buffer key (`Remove Buffer from Redis`).
   - Add a **Google Sheets** node (`Read Student Data from Sheets`) configured to lookup the student record matching the parent's phone number.
   - Add an **If** node (`Check Student Reason Presence`) to verify a student record was found.

6. **Build Block 5: AI Classification & Update**
   - Create an **Advanced AI Agent** node (`Classify Absence Reason`).
   - Connect a **Google Gemini Chat Model** (`Google Gemini Classifier`), a **Structured Output Parser** (`Parse AI Response`), and a **Redis Chat Memory** (`Maintain Chat Memory in Redis`) to the AI Agent node.
   - Configure the AI agent prompt to analyze the student context and parent messages, returning whether a valid absence reason is present in structured JSON format.
   - Add an **If** node (`Assess If Reason Is Given`) to evaluate the AI's output.
   - **If True (Valid Reason):**
     - Add a **Google Sheets** node (`Update Student Data in Sheets`) to write the reason and update status flags.
     - Add a **WhatsApp** node (`Dispatch Confirmation Message`) to send a success reply.
     - Add a **Redis** node (`Release Lock Post Confirmation`) to delete the user's processing lock key.
   - **If False (Incomplete/Invalid Reason):**
     - Add a **WhatsApp** node (`Request Reason Clarification`) to ask for clarification.
     - Add a **Redis** node (`Free Lock After Clarification`) to delete the user's processing lock key.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| YouTube Video Walkthrough | https://youtu.be/Bn1u3pc0Lpo |
| Required Spreadsheet Fields | Must include columns: `ParentPhoneNumber`, `ParentName`, `StudentName`, `Reason`, and `got_reason`. |