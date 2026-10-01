Find and score event and speaker opportunities with Gumloop, Supabase, and OpenAI

https://n8nworkflows.xyz/workflows/find-and-score-event-and-speaker-opportunities-with-gumloop--supabase--and-openai-19748


# Find and score event and speaker opportunities with Gumloop, Supabase, and OpenAI

### 1. Workflow Overview

This workflow is an asynchronous opportunity finder system designed to ingest search requests from a front-end application (such as Lovable), trigger external scraping/search routines via Gumloop, process and score the returned results using Supabase (Postgres), and optionally deliver email notifications via Gmail. Additionally, it exposes auxiliary endpoints for polling search statuses, capturing subscriber emails, and drafting AI-powered outreach emails using OpenAI.

The system logic is organized into the following functional blocks:

- **1.1 Request Intake & Initialization:** Receives inbound POST requests, normalizes payloads, creates a tracking record in Supabase, returns an immediate acknowledgement with a `request_id`, and branches execution.
- **1.2 Search Trigger & Routing:** Evaluates whether the request pertains to event or speaker opportunities, optionally registers subscriber emails, and dispatches the payload to the appropriate Gumloop webhook.
- **1.3 Callback Reception & Scoring:** Ingests completed search data from Gumloop callbacks, matches it with stored Supabase context, evaluates scores based on operational criteria, and updates the database.
- **1.4 Result Delivery & Polling:** Sends optional HTML summary emails via Gmail to the requester, acknowledges Gumloop callbacks, and provides a GET polling endpoint to check search statuses.
- **1.5 AI Outreach Drafting:** Handles dedicated POST requests to draft personalized outreach emails using the OpenAI Chat Completions API.

---

### 2. Block-by-Block Analysis

#### 2.1 Request Intake & Initialization
**Overview:** This block handles the incoming HTTP POST request from the client front end, validates and sanitizes search configurations, initializes a tracking entry in the Supabase database, and responds immediately to prevent HTTP timeouts.

- **Nodes Involved:**
  - `Receive Opportunity Requests`
  - `Normalize Incoming Data`
  - `Insert Temp Search Record`
  - `Confirm Reception to Sender`
  - `Generate Search Parameters`

- **Node Details:**
  - **Receive Opportunity Requests**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`). Acts as the primary entry point for user search queries via HTTP POST.
    - *Configuration:* Path set to `opportunity-finder`, response mode set to handle responses via downstream nodes.
    - *Input/Output Connections:* Input: None (Trigger); Output: `Normalize Incoming Data`.
    - *Edge Cases:* Malformed JSON payloads or missing headers.
  - **Normalize Incoming Data**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Validates mode (`event` or `speaker`), constrains maximum results, and extracts mode-specific properties (industry/location vs. topic/context).
    - *Configuration:* JavaScript parsing block validating required fields (`industry` or `topic`).
    - *Input/Output Connections:* Input: `Receive Opportunity Requests`; Output: `Insert Temp Search Record`.
    - *Edge Cases:* Missing mandatory fields (`industry` or `topic`) triggers an intentional error.
  - **Insert Temp Search Record**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Executes an SQL query to insert a running task row into Supabase.
    - *Configuration:* Query runs `INSERT INTO temp_search_results (status, email, mode) VALUES ('running', $1, $2) RETURNING request_id;`.
    - *Input/Output Connections:* Input: `Normalize Incoming Data`; Output: `Confirm Reception to Sender`, `Generate Search Parameters`.
    - *Edge Cases:* Database connection failures or constraint violations.
  - **Confirm Reception to Sender**
    - *Type and Technical Role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns an immediate JSON response to the client containing the generated `request_id` and status `running`.
    - *Configuration:* Responds with JSON payload `={{ {request_id:$json.request_id,status:'running'} }}`.
    - *Input/Output Connections:* Input: `Insert Temp Search Record`; Output: None (Terminal node for this branch).
  - **Generate Search Parameters**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Merges normalized input parameters with the newly generated Supabase `request_id`.
    - *Configuration:* JavaScript combining upstream node outputs.
    - *Input/Output Connections:* Input: `Insert Temp Search Record`; Output: `Route by Search Mode`, `Check for Provided Email`.

---

#### 2.2 Search Trigger & Routing
**Overview:** This block branches execution according to the search mode, captures optional subscriber leads, and triggers the corresponding external Gumloop scraping sequence while monitoring for API success.

- **Nodes Involved:**
  - `Route by Search Mode`
  - `Check for Provided Email`
  - `Store Subscriber Email`
  - `Initiate Gumloop Event Search`
  - `Verify Event Trigger Success`
  - `Initiate Gumloop Speaker Search`
  - `Verify Speaker Trigger Success`
  - `Log Trigger Failure`

- **Node Details:**
  - **Route by Search Mode**
    - *Type and Technical Role:* Switch node (`n8n-nodes-base.switch`). Directs execution flow depending on whether `mode` is `event` or `speaker`.
    - *Input/Output Connections:* Input: `Generate Search Parameters`; Output: Branch 0 (`Initiate Gumloop Event Search`), Branch 1 (`Initiate Gumloop Speaker Search`).
  - **Check for Provided Email**
    - *Type and Technical Role:* If node (`n8n-nodes-base.if`). Evaluates if an email address was included in the search query.
    - *Configuration:* Checks expression `={{ !!$json.email }}`.
    - *Input/Output Connections:* Input: `Generate Search Parameters`; Output: True branch to `Store Subscriber Email`.
  - **Store Subscriber Email**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Persists subscriber information into the database.
    - *Configuration:* Query runs `INSERT INTO subscribers (email, source) VALUES ($1, $2);`.
    - *Input/Output Connections:* Input: `Check for Provided Email`; Output: None.
  - **Initiate Gumloop Event Search**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Calls the Gumloop webhook API to begin event crawling.
    - *Configuration:* POST request with a 15-second timeout, sending JSON payload containing event parameters and `request_id`.
    - *Input/Output Connections:* Input: Route by Search Mode; Output: `Verify Event Trigger Success` and error handling to `Log Trigger Failure`.
  - **Verify Event Trigger Success**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Asserts that Gumloop successfully accepted the trigger payload.
    - *Configuration:* Throws an error if `success !== true`.
    - *Input/Output Connections:* Input: `Initiate Gumloop Event Search`; Output: Error output mapped to `Log Trigger Failure`.
  - **Initiate Gumloop Speaker Search**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Triggers the Gumloop speaker scraping workflow.
    - *Configuration:* POST request with JSON payload containing speaker search parameters and `request_id`.
    - *Input/Output Connections:* Input: Route by Search Mode; Output: `Verify Speaker Trigger Success` and error handling to `Log Trigger Failure`.
  - **Verify Speaker Trigger Success**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Validates successful acceptance by the speaker search webhook.
    - *Input/Output Connections:* Input: `Initiate Gumloop Speaker Search`; Output: Error output mapped to `Log Trigger Failure`.
  - **Log Trigger Failure**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Updates the temporary search record status to `failed` if the Gumloop trigger fails.
    - *Configuration:* Query runs `UPDATE temp_search_results SET status='failed', error_message='Gumloop trigger failed' WHERE request_id=$1 AND status='running';`.
    - *Input/Output Connections:* Input: Error outputs from verification or HTTP request nodes; Output: None.

---

#### 2.3 Callback Reception & Scoring
**Overview:** This block processes the asynchronous callback from Gumloop containing raw search findings, links them back to the original request metadata, and applies scoring logic based on deadlines (events) or temporal recency (speakers).

- **Nodes Involved:**
  - `Receive Gumloop Callback`
  - `Validate Callback Data`
  - `Retrieve Temp Data Context`
  - `Combine Callback and Context`
  - `Route Scoring by Type`
  - `Evaluate Event Scoring`
  - `Evaluate Speaker Scoring`
  - `Update Scored Results`

- **Node Details:**
  - **Receive Gumloop Callback**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`). Receives the POST callback payload from Gumloop upon completion.
    - *Configuration:* Path set to `opportunity-finder-callback`.
    - *Input/Output Connections:* Input: External Gumloop service; Output: `Validate Callback Data`.
  - **Validate Callback Data**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Ensures that `request_id` and an array of `results` are present.
    - *Input/Output Connections:* Input: `Receive Gumloop Callback`; Output: `Retrieve Temp Data Context`.
  - **Retrieve Temp Data Context**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Queries Supabase to fetch the original search mode and email associated with the `request_id`.
    - *Configuration:* Query runs `SELECT mode, email FROM temp_search_results WHERE request_id = $1 AND status = 'running';`.
    - *Input/Output Connections:* Input: `Validate Callback Data`; Output: `Combine Callback and Context`.
  - **Combine Callback and Context**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Merges callback findings with database context parameters.
    - *Input/Output Connections:* Input: `Retrieve Temp Data Context`; Output: `Route Scoring by Type`.
  - **Route Scoring by Type**
    - *Type and Technical Role:* Switch node (`n8n-nodes-base.switch`). Routes the payload to event or speaker scoring algorithms based on the search mode.
    - *Input/Output Connections:* Input: `Combine Callback and Context`; Output: Branch 0 (`Evaluate Event Scoring`), Branch 1 (`Evaluate Speaker Scoring`).
  - **Evaluate Event Scoring**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Calculates opportunity scores for events based on remaining days until application deadlines.
    - *Input/Output Connections:* Input: `Route Scoring by Type`; Output: `Update Scored Results`.
  - **Evaluate Speaker Scoring**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Assigns scores to speaker opportunities based on publication or discussion recency.
    - *Input/Output Connections:* Input: `Route Scoring by Type`; Output: `Update Scored Results`.
  - **Update Scored Results**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Persists finalized scores into the database and marks the task as completed.
    - *Configuration:* Query runs `UPDATE temp_search_results SET status='completed', results=$1::jsonb WHERE request_id=$2 AND status='running' RETURNING email, results;`.
    - *Input/Output Connections:* Input: `Evaluate Event Scoring` or `Evaluate Speaker Scoring`; Output: `Verify Email Existence`, `Acknowledge Gumloop Callback`.

---

#### 2.4 Result Delivery & Polling
**Overview:** This block manages email notifications for completed searches, responds to Gumloop callbacks, and provides polling endpoints for clients to query progress or retrieve finished results.

- **Nodes Involved:**
  - `Verify Email Existence`
  - `Construct Email Content`
  - `Dispatch Results Email`
  - `Acknowledge Gumloop Callback`
  - `Fetch Result Requests`
  - `Extract Request ID`
  - `Retrieve Result Record`
  - `Create Result Response`
  - `Send Results Response`

- **Node Details:**
  - **Verify Email Existence**
    - *Type and Technical Role:* If node (`n8n-nodes-base.if`). Checks whether an email address was provided to determine if notification dispatch is necessary.
    - *Input/Output Connections:* Input: `Update Scored Results`; Output: True branch to `Construct Email Content`.
  - **Construct Email Content**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Generates an HTML-formatted summary table of the search results.
    - *Input/Output Connections:* Input: `Verify Email Existence`; Output: `Dispatch Results Email`.
  - **Dispatch Results Email**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`). Sends the formatted results email to the user via Gmail OAuth2.
    - *Credentials:* Uses Gmail OAuth2 credentials.
    - *Input/Output Connections:* Input: `Construct Email Content`; Output: None.
  - **Acknowledge Gumloop Callback**
    - *Type and Technical Role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns a confirmation status response to Gumloop.
    - *Configuration:* Responds with JSON payload `={{ {status:'received'} }}`.
    - *Input/Output Connections:* Input: `Update Scored Results`; Output: None.
  - **Fetch Result Requests**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`). Exposes a GET polling endpoint to check search progress.
    - *Configuration:* Path set to `opportunity-finder-results`, configured for GET requests.
    - *Input/Output Connections:* Input: External Client; Output: `Extract Request ID`.
  - **Extract Request ID**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Extracts the `request_id` parameter from query strings.
    - *Input/Output Connections:* Input: `Fetch Result Requests`; Output: `Retrieve Result Record`.
  - **Retrieve Result Record**
    - *Type and Technical Role:* Postgres node (`n8n-nodes-base.postgres`). Queries the database for task status, results, and error logs.
    - *Configuration:* Query runs `SELECT status, results, error_message FROM temp_search_results WHERE request_id=$1;`.
    - *Input/Output Connections:* Input: `Extract Request ID`; Output: `Create Result Response`.
  - **Create Result Response**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Formats the database record into a standardized API response structure.
    - *Input/Output Connections:* Input: `Retrieve Result Record`; Output: `Send Results Response`.
  - **Send Results Response**
    - *Type and Technical Role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns status and results data back to the polling client.
    - *Input/Output Connections:* Input: `Create Result Response`; Output: None.

---

#### 2.5 AI Outreach Drafting
**Overview:** This block exposes a dedicated webhook endpoint that accepts opportunity details and utilizes OpenAI to draft tailored, concise outreach emails.

- **Nodes Involved:**
  - `Receive Email Draft Request`
  - `Check Draft Request Validity`
  - `Draft Email via OpenAI`
  - `Extract Draft from OpenAI`
  - `Send Draft Response`

- **Node Details:**
  - **Receive Email Draft Request**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`). Listens for POST requests to generate outreach email drafts.
    - *Configuration:* Path set to `opportunity-finder-draft-email`.
    - *Input/Output Connections:* Input: External Client; Output: `Check Draft Request Validity`.
  - **Check Draft Request Validity**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Validates required fields (`title`, `contact_type`, `event_description`) and assigns default parameters (tone).
    - *Input/Output Connections:* Input: `Receive Email Draft Request`; Output: `Draft Email via OpenAI`.
  - **Draft Email via OpenAI**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Calls the OpenAI Chat Completions API using structured JSON schema output formatting.
    - *Configuration:* POST request to `https://api.openai.com/v1/chat/completions` using `gpt-4o-mini`, specifying HTTP Header Authentication and JSON object response formatting.
    - *Credentials:* Generic Credential Type (HTTP Header Auth for OpenAI API Key).
    - *Input/Output Connections:* Input: `Check Draft Request Validity`; Output: `Extract Draft from OpenAI`.
  - **Extract Draft from OpenAI**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Parses the JSON response generated by the LLM, validating the presence of subject and body properties.
    - *Input/Output Connections:* Input: `Draft Email via OpenAI`; Output: `Send Draft Response`.
  - **Send Draft Response**
    - *Type and Technical Role:* Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Returns the generated subject and body fields to the requesting client application.
    - *Input/Output Connections:* Input: `Extract Draft from OpenAI`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation and configuration overview | None | None | Find event and speaker opportunities with Gumloop, Supabase, and OpenAI<br><br>### How it works<br>This workflow accepts search requests from a Lovable front end, creates a temporary result record in Supabase/Postgres, immediately returns a request ID, and triggers the appropriate Gumloop search for event or speaker opportunities... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Lovable request intake documentation | None | None | ## Lovable request intake<br><br>Receives the initial Lovable webhook request, validates and normalizes the payload, and creates a temporary database row to track the async search. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Acknowledge and prepare search documentation | None | None | ## Acknowledge and prepare search<br><br>Immediately responds to Lovable with tracking information while also building normalized parameters for downstream Gumloop searches and optional email capture. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Route search mode documentation | None | None | ## Route search mode<br><br>Branches the request based on whether the user is searching for event opportunities or speaker opportunities. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Capture subscriber email documentation | None | None | ## Capture subscriber email<br><br>Checks whether an email address was provided and stores it in the subscribers table when present. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Trigger event search documentation | None | None | ## Trigger event search<br><br>Calls Gumloop for event-opportunity searches and validates that the event search trigger was accepted. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Trigger speaker search documentation | None | None | ## Trigger speaker search<br><br>Calls Gumloop for speaker-opportunity searches and validates that the speaker search trigger was accepted. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Record trigger failure documentation | None | None | ## Record trigger failure<br><br>Updates the database when a Gumloop trigger or acceptance check fails. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Load callback context documentation | None | None | ## Load callback context<br><br>Receives the Gumloop callback, validates its payload, fetches the original temporary row, and merges stored request context with callback results. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Score and store results documentation | None | None | ## Score and store results<br><br>Routes callback processing by search mode, scores event or speaker results, and writes the final scored results back to the temporary result row. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Email completed results documentation | None | None | ## Email completed results<br><br>If the original request included an email address, builds an HTML results email and sends it via Gmail. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Acknowledge Gumloop callback documentation | None | None | ## Acknowledge Gumloop callback<br><br>Returns a webhook response to Gumloop after the result row has been updated. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Results request intake documentation | None | None | ## Results request intake<br><br>Provides a polling endpoint that receives a request ID from the front end and extracts it for lookup. |
| Sticky Note13 | n8n-nodes-base.stickyNote | Return stored results documentation | None | None | ## Return stored results<br><br>Fetches the temporary result row, formats the current status or completed results, and returns them to the requester. |
| Sticky Note14 | n8n-nodes-base.stickyNote | Draft request intake documentation | None | None | ## Draft request intake<br><br>Receives and validates a request to draft an outreach email for a selected opportunity. |
| Sticky Note15 | n8n-nodes-base.stickyNote | Generate outreach draft documentation | None | None | ## Generate outreach draft<br><br>Calls OpenAI to generate the outreach email draft, parses the model response, and returns the draft to the front end. |
| Receive Opportunity Requests | n8n-nodes-base.webhook | Receives incoming search requests via POST | None | Normalize Incoming Data | ## Lovable request intake<br>Receives the initial Lovable webhook request, validates and normalizes the payload, and creates a temporary database row to track the async search. |
| Normalize Incoming Data | n8n-nodes-base.code | Normalizes payload parameters and validates search mode | Receive Opportunity Requests | Insert Temp Search Record | ## Lovable request intake<br>Receives the initial Lovable webhook request, validates and normalizes the payload, and creates a temporary database row to track the async search. |
| Insert Temp Search Record | n8n-nodes-base.postgres | Inserts running search record into Supabase | Normalize Incoming Data | Confirm Reception to Sender, Generate Search Parameters | ## Lovable request intake<br>Receives the initial Lovable webhook request, validates and normalizes the payload, and creates a temporary database row to track the async search. |
| Generate Search Parameters | n8n-nodes-base.code | Combines database request_id with normalized inputs | Insert Temp Search Record | Route by Search Mode, Check for Provided Email | ## Acknowledge and prepare search<br>Immediately responds to Lovable with tracking information while also building normalized parameters for downstream Gumloop searches and optional email capture. |
| Confirm Reception to Sender | n8n-nodes-base.respondToWebhook | Immediately responds to Lovable with task ID and status | Insert Temp Search Record | None | ## Acknowledge and prepare search<br>Immediately responds to Lovable with tracking information while also building normalized parameters for downstream Gumloop searches and optional email capture. |
| Route by Search Mode | n8n-nodes-base.switch | Branches execution path based on event or speaker mode | Generate Search Parameters | Initiate Gumloop Event Search, Initiate Gumloop Speaker Search | ## Route search mode<br>Branches the request based on whether the user is searching for event opportunities or speaker opportunities. |
| Check for Provided Email | n8n-nodes-base.if | Evaluates if user email is present | Generate Search Parameters | Store Subscriber Email | ## Capture subscriber email<br>Checks whether an email address was provided and stores it in the subscribers table when present. |
| Store Subscriber Email | n8n-nodes-base.postgres | Saves subscriber email into database | Check for Provided Email | None | ## Capture subscriber email<br>Checks whether an email address was provided and stores it in the subscribers table when present. |
| Initiate Gumloop Event Search | n8n-nodes-base.httpRequest | Calls Gumloop event webhook trigger | Route by Search Mode | Verify Event Trigger Success, Log Trigger Failure | ## Trigger event search<br>Calls Gumloop for event-opportunity searches and validates that the event search trigger was accepted. |
| Verify Event Trigger Success | n8n-nodes-base.code | Validates acceptance response from Gumloop event trigger | Initiate Gumloop Event Search | Log Trigger Failure | ## Trigger event search<br>Calls Gumloop for event-opportunity searches and validates that the event search trigger was accepted. |
| Initiate Gumloop Speaker Search | n8n-nodes-base.httpRequest | Calls Gumloop speaker webhook trigger | Route by Search Mode | Verify Speaker Trigger Success, Log Trigger Failure | ## Trigger speaker search<br>Calls Gumloop for speaker-opportunity searches and validates that the speaker search trigger was accepted. |
| Verify Speaker Trigger Success | n8n-nodes-base.code | Validates acceptance response from Gumloop speaker trigger | Initiate Gumloop Speaker Search | Log Trigger Failure | ## Trigger speaker search<br>Calls Gumloop for speaker-opportunity searches and validates that the speaker search trigger was accepted. |
| Log Trigger Failure | n8n-nodes-base.postgres | Updates Supabase status to failed upon error | Verify Event Trigger Success, Verify Speaker Trigger Success, Initiate Gumloop Event Search, Initiate Gumloop Speaker Search | None | ## Record trigger failure<br>Updates the database when a Gumloop trigger or acceptance check fails. |
| Receive Gumloop Callback | n8n-nodes-base.webhook | Receives finished search results from Gumloop | None | Validate Callback Data | ## Load callback context<br>Receives the Gumloop callback, validates its payload, fetches the original temporary row, and merges stored request context with callback results. |
| Validate Callback Data | n8n-nodes-base.code | Asserts presence of request_id and results array | Receive Gumloop Callback | Retrieve Temp Data Context | ## Load callback context<br>Receives the Gumloop callback, validates its payload, fetches the original temporary row, and merges stored request context with callback results. |
| Retrieve Temp Data Context | n8n-nodes-base.postgres | Fetches original task mode and email from Supabase | Validate Callback Data | Combine Callback and Context | ## Load callback context<br>Receives the Gumloop callback, validates its payload, fetches the original temporary row, and merges stored request context with callback results. |
| Combine Callback and Context | n8n-nodes-base.code | Merges callback results with database metadata | Retrieve Temp Data Context | Route Scoring by Type | ## Load callback context<br>Receives the Gumloop callback, validates its payload, fetches the original temporary row, and merges stored request context with callback results. |
| Route Scoring by Type | n8n-nodes-base.switch | Routes evaluation by event or speaker criteria | Combine Callback and Context | Evaluate Event Scoring, Evaluate Speaker Scoring | ## Score and store results<br>Routes callback processing by search mode, scores event or speaker results, and writes the final scored results back to the temporary result row. |
| Evaluate Event Scoring | n8n-nodes-base.code | Computes scores for event opportunities based on deadlines | Route Scoring by Type | Update Scored Results | ## Score and store results<br>Routes callback processing by search mode, scores event or speaker results, and writes the final scored results back to the temporary result row. |
| Evaluate Speaker Scoring | n8n-nodes-base.code | Computes scores for speaker opportunities based on recency | Route Scoring by Type | Update Scored Results | ## Score and store results<br>Routes callback processing by search mode, scores event or speaker results, and writes the final scored results back to the temporary result row. |
| Update Scored Results | n8n-nodes-base.postgres | Updates Supabase record to completed with scored JSON | Evaluate Event Scoring, Evaluate Speaker Scoring | Verify Email Existence, Acknowledge Gumloop Callback | ## Score and store results<br>Routes callback processing by search mode, scores event or speaker results, and writes the final scored results back to the temporary result row. |
| Verify Email Existence | n8n-nodes-base.if | Checks if user email exists for result dispatch | Update Scored Results | Construct Email Content | ## Email completed results<br>If the original request included an email address, builds an HTML results email and sends it via Gmail. |
| Construct Email Content | n8n-nodes-base.code | Generates HTML email table from scored results | Verify Email Existence | Dispatch Results Email | ## Email completed results<br>If the original request included an email address, builds an HTML results email and sends it via Gmail. |
| Dispatch Results Email | n8n-nodes-base.gmail | Sends HTML email via Gmail | Construct Email Content | None | ## Email completed results<br>If the original request included an email address, builds an HTML results email and sends it via Gmail. |
| Acknowledge Gumloop Callback | n8n-nodes-base.respondToWebhook | Responds to Gumloop callback request | Update Scored Results | None | ## Acknowledge Gumloop callback<br>Returns a webhook response to Gumloop after the result row has been updated. |
| Fetch Result Requests | n8n-nodes-base.webhook | Receives GET polling requests for task status | None | Extract Request ID | ## Results request intake<br>Provides a polling endpoint that receives a request ID from the front end and extracts it for lookup. |
| Extract Request ID | n8n-nodes-base.code | Extracts request_id query parameter | Fetch Result Requests | Retrieve Result Record | ## Results request intake<br>Provides a polling endpoint that receives a request ID from the front end and extracts it for lookup. |
| Retrieve Result Record | n8n-nodes-base.postgres | Queries Supabase for status and results | Extract Request ID | Create Result Response | ## Return stored results<br>Fetches the temporary result row, formats the current status or completed results, and returns them to the requester. |
| Create Result Response | n8n-nodes-base.code | Formats database row into clean API payload | Retrieve Result Record | Send Results Response | ## Return stored results<br>Fetches the temporary result row, formats the current status or completed results, and returns them to the requester. |
| Send Results Response | n8n-nodes-base.respondToWebhook | Returns status and results to polling client | Create Result Response | None | ## Return stored results<br>Fetches the temporary result row, formats the current status or completed results, and returns them to the requester. |
| Receive Email Draft Request | n8n-nodes-base.webhook | Ingests request to draft outreach email via POST | None | Check Draft Request Validity | ## Draft request intake<br>Receives and validates a request to draft an outreach email for a selected opportunity. |
| Check Draft Request Validity | n8n-nodes-base.code | Validates required draft fields and sets tone | Receive Email Draft Request | Draft Email via OpenAI | ## Draft request intake<br>Receives and validates a request to draft an outreach email for a selected opportunity. |
| Draft Email via OpenAI | n8n-nodes-base.httpRequest | Calls OpenAI Chat Completions API for email drafting | Check Draft Request Validity | Extract Draft from OpenAI | ## Generate outreach draft<br>Calls OpenAI to generate the outreach email draft, parses the model response, and returns the draft to the front end. |
| Extract Draft from OpenAI | n8n-nodes-base.code | Parses LLM response for subject and body | Draft Email via OpenAI | Send Draft Response | ## Generate outreach draft<br>Calls OpenAI to generate the outreach email draft, parses the model response, and returns the draft to the front end. |
| Send Draft Response | n8n-nodes-base.respondToWebhook | Returns email draft to the front end | Extract Draft from OpenAI | None | ## Generate outreach draft<br>Calls OpenAI to generate the outreach email draft, parses the model response, and returns the draft to the front end. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Database Setup (Supabase / Postgres):**
   - Create a table named `temp_search_results` with columns: `request_id` (UUID, Primary Key, Default `gen_random_uuid()`), `status` (text), `email` (text), `mode` (text), `results` (jsonb), and `error_message` (text).
   - Create a table named `subscribers` with columns: `email` (text) and `source` (text).

2. **Build Block 1.1 (Request Intake & Initialization):**
   - **Node 1 (`Receive Opportunity Requests`):** Create a Webhook node (`n8n-nodes-base.webhook`). Set HTTP Method to `POST` and Path to `opportunity-finder`. Set response mode to handle via downstream nodes.
   - **Node 2 (`Normalize Incoming Data`):** Create a Code node (`n8n-nodes-base.code`). Add JavaScript to validate `mode` (`event` or `speaker`), ensure required fields exist (`industry` or `topic`), and normalize the result limit. Connect from `Receive Opportunity Requests`.
   - **Node 3 (`Insert Temp Search Record`):** Create a Postgres node (`n8n-nodes-base.postgres`). Configure Postgres credentials. Run query: `INSERT INTO temp_search_results (status, email, mode) VALUES ('running', $1, $2) RETURNING request_id;` with query replacements `={{ [$json.email,$json.mode] }}`. Connect from `Normalize Incoming Data`.
   - **Node 4 (`Confirm Reception to Sender`):** Create a Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Respond with JSON: `={{ {request_id:$json.request_id,status:'running'} }}`. Connect from `Insert Temp Search Record`.
   - **Node 5 (`Generate Search Parameters`):** Create a Code node (`n8n-nodes-base.code`). Combine normalized data with `request_id`. Connect from `Insert Temp Search Record`.

3. **Build Block 1.2 (Search Trigger & Routing):**
   - **Node 6 (`Route by Search Mode`):** Create a Switch node (`n8n-nodes-base.switch`). Add rules where `={{ $json.mode }}` equals `event` (Branch 0) and `speaker` (Branch 1). Connect from `Generate Search Parameters`.
   - **Node 7 (`Check for Provided Email`):** Create an If node (`n8n-nodes-base.if`). Condition: `={{ !!$json.email }}` equals `true`. Connect from `Generate Search Parameters`.
   - **Node 8 (`Store Subscriber Email`):** Create a Postgres node (`n8n-nodes-base.postgres`). Run query: `INSERT INTO subscribers (email, source) VALUES ($1, $2);` with replacement `={{ [$json.email,'opportunity_finder_'+$json.mode] }}`. Connect from True output of `Check for Provided Email`.
   - **Node 9 (`Initiate Gumloop Event Search`):** Create an HTTP Request node (`n8n-nodes-base.httpRequest`). Method `POST`, URL: `https://api.gumloop.com/trigger_incoming_webhook/REPLACE_WITH_YOUR_EVENT_TRIGGER`. Set body to JSON containing search parameters and `request_id`. Enable `Continue On Fail`. Connect from Switch Branch 0.
   - **Node 10 (`Verify Event Trigger Success`):** Create a Code node (`n8n-nodes-base.code`). Check that `r.success === true`. Connect from `Initiate Gumloop Event Search`.
   - **Node 11 (`Initiate Gumloop Speaker Search`):** Create an HTTP Request node (`n8n-nodes-base.httpRequest`). Method `POST`, URL: `https://api.gumloop.com/trigger_incoming_webhook/REPLACE_WITH_YOUR_SPEAKER_TRIGGER`. Set body to JSON containing parameters and `request_id`. Enable `Continue On Fail`. Connect from Switch Branch 1.
   - **Node 12 (`Verify Speaker Trigger Success`):** Create a Code node (`n8n-nodes-base.code`). Check that `r.success === true`. Connect from `Initiate Gumloop Speaker Search`.
   - **Node 13 (`Log Trigger Failure`):** Create a Postgres node (`n8n-nodes-base.postgres`). Run query: `UPDATE temp_search_results SET status='failed', error_message='Gumloop trigger failed' WHERE request_id=$1 AND status='running';`. Connect error outputs from HTTP and verification nodes to this node.

4. **Build Block 1.3 (Callback Reception & Scoring):**
   - **Node 14 (`Receive Gumloop Callback`):** Create a Webhook node (`n8n-nodes-base.webhook`). Method `POST`, Path: `opportunity-finder-callback`.
   - **Node 15 (`Validate Callback Data`):** Create a Code node (`n8n-nodes-base.code`). Validate presence of `request_id` and `results`. Connect from `Receive Gumloop Callback`.
   - **Node 16 (`Retrieve Temp Data Context`):** Create a Postgres node (`n8n-nodes-base.postgres`). Query: `SELECT mode, email FROM temp_search_results WHERE request_id=$1 AND status='running';` with replacement `={{ [$json.request_id] }}`. Connect from `Validate Callback Data`.
   - **Node 17 (`Combine Callback and Context`):** Create a Code node (`n8n-nodes-base.code`). Merge callback and database context. Connect from `Retrieve Temp Data Context`.
   - **Node 18 (`Route Scoring by Type`):** Create a Switch node (`n8n-nodes-base.switch`). Route by `mode` (`event` vs `speaker`). Connect from `Combine Callback and Context`.
   - **Node 19 (`Evaluate Event Scoring`):** Create a Code node (`n8n-nodes-base.code`). Compute scores based on event deadlines. Connect from Switch Branch 0.
   - **Node 20 (`Evaluate Speaker Scoring`):** Create a Code node (`n8n-nodes-base.code`). Compute scores based on speaker recency. Connect from Switch Branch 1.
   - **Node 21 (`Update Scored Results`):** Create a Postgres node (`n8n-nodes-base.postgres`). Query: `UPDATE temp_search_results SET status='completed', results=$1::jsonb WHERE request_id=$2 AND status='running' RETURNING email, results;` with replacements `={{ [JSON.stringify($json.scored_results),$json.request_id] }}`. Connect from both scoring nodes.

5. **Build Block 1.4 (Result Delivery & Polling):**
   - **Node 22 (`Verify Email Existence`):** Create an If node (`n8n-nodes-base.if`). Condition: `={{ !!$json.email }}` equals `true`. Connect from `Update Scored Results`.
   - **Node 23 (`Construct Email Content`):** Create a Code node (`n8n-nodes-base.code`). Generate HTML table for email output. Connect from True output of `Verify Email Existence`.
   - **Node 24 (`Dispatch Results Email`):** Create a Gmail node (`n8n-nodes-base.gmail`). Configure Gmail OAuth2 credentials. Map sendTo, subject, and message. Connect from `Construct Email Content`.
   - **Node 25 (`Acknowledge Gumloop Callback`):** Create a Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Respond with JSON: `={{ {status:'received'} }}`. Connect from `Update Scored Results`.
   - **Node 26 (`Fetch Result Requests`):** Create a Webhook node (`n8n-nodes-base.webhook`). Method `GET`, Path: `opportunity-finder-results`.
   - **Node 27 (`Extract Request ID`):** Create a Code node (`n8n-nodes-base.code`). Extract `request_id` from query parameters. Connect from `Fetch Result Requests`.
   - **Node 28 (`Retrieve Result Record`):** Create a Postgres node (`n8n-nodes-base.postgres`). Query: `SELECT status, results, error_message FROM temp_search_results WHERE request_id=$1;` with replacement `={{ [$json.request_id] }}`. Connect from `Extract Request ID`.
   - **Node 29 (`Create Result Response`):** Create a Code node (`n8n-nodes-base.code`). Format response payload. Connect from `Retrieve Result Record`.
   - **Node 30 (`Send Results Response`):** Create a Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Respond with JSON containing task status and results. Connect from `Create Result Response`.

6. **Build Block 1.5 (AI Outreach Drafting):**
   - **Node 31 (`Receive Email Draft Request`):** Create a Webhook node (`n8n-nodes-base.webhook`). Method `POST`, Path: `opportunity-finder-draft-email`.
   - **Node 32 (`Check Draft Request Validity`):** Create a Code node (`n8n-nodes-base.code`). Validate required fields (`title`, `contact_type`, `event_description`). Connect from `Receive Email Draft Request`.
   - **Node 33 (`Draft Email via OpenAI`):** Create an HTTP Request node (`n8n-nodes-base.httpRequest`). Method `POST`, URL: `https://api.openai.com/v1/chat/completions`. Use generic HTTP Header Auth credential for OpenAI. Set model to `gpt-4o-mini`, response format to `json_object`, and include system/user prompts. Connect from `Check Draft Request Validity`.
   - **Node 34 (`Extract Draft from OpenAI`):** Create a Code node (`n8n-nodes-base.code`). Parse JSON string content from LLM response. Connect from `Draft Email via OpenAI`.
   - **Node 35 (`Send Draft Response`):** Create a Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Respond with JSON containing email subject and body. Connect from `Extract Draft from OpenAI`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Find and score event and speaker opportunities with Gumloop, Supabase, and OpenAI | Workflow title and integration overview |