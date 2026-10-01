Intake and route web form leads with Airtable and Slack

https://n8nworkflows.xyz/workflows/intake-and-route-web-form-leads-with-airtable-and-slack-19809


# Intake and route web form leads with Airtable and Slack

### 1. Workflow Overview

This workflow is designed to automate lead intake from a web form securely. It ingests incoming POST requests via a webhook, validates and normalizes the payload, prevents duplicate entries by checking Airtable, logs valid new leads, and routes notifications to specific Slack channels based on the submitted programme. It handles edge cases such as invalid data, duplicate submissions, and write failures by responding with appropriate HTTP status codes (200, 400, 500) and dispatching alerts to monitoring channels.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Validation:** Receives the HTTP POST request, normalizes data structures (supporting root-level or body-nested properties), validates required fields and email syntax, and routes invalid requests to a 400 rejection handler.
- **1.2 De-duplication & Existing Record Check:** Queries Airtable to check if a lead with the given email address already exists, returning a 200 duplicate response if found.
- **1.3 Storage & Programme Routing:** Creates a new record in Airtable for unique leads, inspects the programme to determine routing ("Priority" for Computer Science, "General" for others), notifies the corresponding Slack channel, and returns a 200 created response.
- **1.4 Error Handling & Resilience:** Captures runtime database write errors or unhandled global workflow exceptions, alerts a dedicated Slack failure channel, and returns an HTTP 500 response when necessary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
This block handles secure entry of lead data, flattens payload discrepancies, checks fields for presence and correctness, and prematurely halts invalid submissions.

- **Lead Intake Webhook**
  - **Type & Role:** `n8n-nodes-base.webhook` — Acts as the primary HTTP entry point for lead submissions.
  - **Configuration:** Configured for `POST` requests on path `lead-intake` using header authentication (`headerAuth`) and response management via the response node.
  - **Expressions/Variables:** None required.
  - **Connections:** Input: None (Trigger); Output: `Validate Submission`.
  - **Version Requirements:** Version 2.1.
  - **Edge Cases:** Unauthorized requests if header auth tokens mismatch; malformed JSON body payloads.

- **Validate Submission**
  - **Type & Role:** `n8n-nodes-base.set` — Normalizes incoming fields (checking both root and `.body` structures), evaluates whether all required fields are present, and validates email syntax via regex.
  - **Configuration:** Assigns calculated expressions to properties: `name`, `email`, `phone`, `country`, `programme`, `route` (evaluating if programme equals 'Computer Science'), `isValid` (boolean), and `validationErrors` (string).
  - **Expressions/Variables:** Uses complex JavaScript expression blocks parsing `$json.body` or `$json` fallbacks, trim methods, and regex validation (`/^[\s@]+@[^\s@]+\.[^\s@]+$/`).
  - **Connections:** Input: `Lead Intake Webhook`; Output: `Is Valid?`.
  - **Version Requirements:** Version 3.5.
  - **Edge Cases:** Runtime evaluation failures if payload structure deviates drastically from expected shapes.

- **Is Valid?**
  - **Type & Role:** `n8n-nodes-base.if` — Directs workflow branching depending on whether the `isValid` assignment evaluates to true.
  - **Configuration:** Evaluates condition where left value `{{ $('Validate Submission').item.json.isValid }}` equals boolean `true`.
  - **Expressions/Variables:** References node data via `$('Validate Submission').item.json.isValid`.
  - **Connections:** Input: `Validate Submission`; Outputs: True path to `Find Existing by Email`, False path to `Respond 400 Invalid`.
  - **Version Requirements:** Version 2.3.
  - **Edge Cases:** Boolean coercion issues if unhandled types pass through.

- **Respond 400 Invalid**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` — Terminates invalid request flows by sending an HTTP 400 response.
  - **Configuration:** Sets HTTP status code to `400`, responds with JSON indicating rejection status alongside concatenated validation error messages.
  - **Expressions/Variables:** `={{ { "status": "rejected", "message": "Submission rejected: " + $('Validate Submission').item.json.validationErrors } }}`
  - **Connections:** Input: `Is Valid?` (False branch); Output: None (Terminal).
  - **Version Requirements:** Version 1.5.
  - **Edge Cases:** None.

---

#### 2.2 De-duplication & Existing Record Check
This block verifies whether a lead already exists in the target database based on email uniqueness.

- **Find Existing by Email**
  - **Type & Role:** `n8n-nodes-base.airtable` — Queries the Airtable database to find records matching the submitted email.
  - **Configuration:** Set to `search` operation with up to 3 retries, continuing regular output on failure and always outputting data. Uses formula filtering.
  - **Expressions/Variables:** `={{ "LOWER({Email}) = "+ JSON.stringify($('Validate Submission').item.json.email) }}`
  - **Connections:** Input: `Is Valid?` (True branch); Output: `Errored?`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases:** Airtable API rate limits, temporary unavailability, or invalid token errors (handled via retry logic and error routing).

- **Errored?**
  - **Type & Role:** `n8n-nodes-base.if` — Evaluates if the Airtable search operation captured an error object.
  - **Configuration:** Strict object existence check on `$json.error`.
  - **Expressions/Variables:** `={{ $json.error }}`
  - **Connections:** Input: `Find Existing by Email`; Outputs: True to `Notify Failure Channel`, False to `Is Duplicate?`.
  - **Version Requirements:** Version 2.3.
  - **Edge Cases:** Uncaught schema changes.

- **Is Duplicate?**
  - **Type & Role:** `n8n-nodes-base.if` — Checks if an Airtable record ID was returned from the lookup step.
  - **Configuration:** Strict string existence check on the record ID.
  - **Expressions/Variables:** `={{ $('Find Existing by Email').item.json.id }}`
  - **Connections:** Input: `Errored?` (False branch); Outputs: True to `Respond 200 Duplicate`, False to `Create Lead Record`.
  - **Version Requirements:** Version 2.3.
  - **Edge Cases:** Empty search arrays returning undefined ID structures.

- **Respond 200 Duplicate**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` — Responds to webhook callers when a lead is identified as a duplicate.
  - **Configuration:** HTTP status `200`, responding with JSON status and informational message.
  - **Expressions/Variables:** Static JSON payload.
  - **Connections:** Input: `Is Duplicate?` (True branch); Output: None (Terminal).
  - **Version Requirements:** Version 1.5.
  - **Edge Cases:** None.

---

#### 2.3 Storage & Programme Routing
This block records new leads in Airtable, routes team notifications based on the programme value, and returns a success confirmation.

- **Create Lead Record**
  - **Type & Role:** `n8n-nodes-base.airtable` — Creates a new record in the Airtable leads table with mapped field attributes.
  - **Configuration:** Operation set to `create` with typecasting enabled, configured to continue on error output and retry up to 3 times.
  - **Expressions/Variables:** Maps name, email, phone, route, country, programme, current ISO timestamp (`$now.toISO()`), and source metadata from `Validate Submission`.
  - **Connections:** Input: `Is Duplicate?` (False branch); Outputs: Main branch to `Route by Programme`, Error branch to `Notify Failure Channel`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases:** Airtable field type mismatch errors, invalid options, or network timeouts.

- **Route by Programme**
  - **Type & Role:** `n8n-nodes-base.switch` — Directs flow according to the lead route value.
  - **Configuration:** Rule configured to match if route equals "Priority", with fallback configured as "General".
  - **Expressions/Variables:** `={{ $('Validate Submission').item.json.route }}`
  - **Connections:** Input: `Create Lead Record`; Outputs: Priority branch to `Notify Priority Channel`, General (fallback) branch to `Notify General Channel`.
  - **Version Requirements:** Version 3.4.
  - **Edge Cases:** Unexpected whitespace or spelling mismatches in route evaluation.

- **Notify Priority Channel**
  - **Type & Role:** `n8n-nodes-base.slack` — Posts priority lead alerts to a specific Slack channel.
  - **Configuration:** Posts text message to a designated channel ID (`C0BUSH624BY`) with retries enabled.
  - **Expressions/Variables:** Uses inline template strings reading form parameters and the new Airtable record ID.
  - **Connections:** Input: `Route by Programme` (Priority output); Output: `Respond 200 Created`.
  - **Version Requirements:** Version 2.7.
  - **Edge Cases:** Slack API downtime or invalid channel credentials.

- **Notify General Channel**
  - **Type & Role:** `n8n-nodes-base.slack` — Posts general lead alerts to a default Slack channel.
  - **Configuration:** Posts text message to a designated channel ID (`C0BUD4QJLDD`) with retries enabled.
  - **Expressions/Variables:** Uses inline template strings reading form parameters and the new Airtable record ID.
  - **Connections:** Input: `Route by Programme` (General output); Output: `Respond 200 Created`.
  - **Version Requirements:** Version 2.7.
  - **Edge Cases:** Slack API downtime or invalid channel credentials.

- **Respond 200 Created**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` — Finalizes successful creation flows with an HTTP 200 status.
  - **Configuration:** HTTP status `200`, responding with a success confirmation JSON body.
  - **Expressions/Variables:** Static JSON payload confirming storage and team notification.
  - **Connections:** Inputs: `Notify Priority Channel`, `Notify General Channel`; Output: None (Terminal).
  - **Version Requirements:** Version 1.5.
  - **Edge Cases:** None.

---

#### 2.4 Error Handling & Resilience
This block captures and responds to runtime storage faults and unhandled global exceptions.

- **Notify Failure Channel**
  - **Type & Role:** `n8n-nodes-base.slack` — Notifies an operations channel of database persistence issues.
  - **Configuration:** Posts failure notification to monitoring channel ID (`C0BUUF86ZU2`) including execution context, error details, and workflow link.
  - **Expressions/Variables:** Evaluates error message objects (`$json.error?.message ?? $json.message`) and execution IDs (`$execution.id`).
  - **Connections:** Inputs: `Create Lead Record` (Error output), `Errored?` (True branch); Output: `Respond 500 Failure`.
  - **Version Requirements:** Version 2.7.
  - **Edge Cases:** Slack connectivity issues during failure propagation.

- **Respond 500 Failure**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` — Returns an HTTP 500 error to the API caller when processing fails internally.
  - **Configuration:** HTTP status `500`, returning user-friendly failure messaging.
  - **Expressions/Variables:** Static JSON payload.
  - **Connections:** Input: `Notify Failure Channel`; Output: None (Terminal).
  - **Version Requirements:** Version 1.5.
  - **Edge Cases:** None.

- **On Workflow Error**
  - **Type & Role:** `n8n-nodes-base.errorTrigger` — Listens globally for any unhandled exceptions occurring anywhere in the workflow execution.
  - **Configuration:** Default trigger configuration.
  - **Connections:** Output: `Alert Error Channel`.
  - **Version Requirements:** Version 1.

- **Alert Error Channel**
  - **Type & Role:** `n8n-nodes-base.slack` — Broadcasts unhandled workflow crashes to the monitoring channel.
  - **Configuration:** Posts rich context including workflow name, failed node, execution error message, execution ID, and timestamp to channel ID (`C0BUUF86ZU2`).
  - **Expressions/Variables:** References global execution variables like `$json.workflow?.name`, `$json.execution?.lastNodeExecuted`, and `$now.toISO()`.
  - **Connections:** Input: `On Workflow Error`; Output: None (Terminal).
  - **Version Requirements:** Version 2.7.
  - **Edge Cases:** Slack API outages during global failure handling.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Lead Intake Webhook | n8n-nodes-base.webhook | Receives incoming POST web form data | None | Validate Submission | ## Lead Intake & Routing<br><br>Webhook API that captures student enquiries from a web form, validates them, de-duplicates against Airtable, stores new leads, and alerts the right Slack channel — returning a proper HTTP status to the caller every time.<br><br>### How it works<br>1. **Receive & validate** — a Set node normalises the payload (fields may arrive under `body` or at the root), checks name / email / phone / country / programme, and validates the email format. Invalid input returns **400** with the reasons.<br>2. **Dedupe** — looks the email up in Airtable. If it already exists, returns **200 duplicate** and creates nothing.<br>3. **Store & route** — creates the Airtable lead, routes *Computer Science* → priority Slack channel and everything else → general channel, then returns **200 created**.<br>4. **Resilience** — a write failure alerts a failure channel and returns **500**; an Error Trigger notifies a monitoring channel on any unhandled error.<br><br>### Setup<br>- Add credentials: **header-auth** (webhook `token`), **Airtable** PAT, **Slack**.<br>- Point the Airtable nodes at your own base / table (Name, Email, Phone, Country, Programme, Route, Submitted At, Source).<br>- Replace the demo Slack channel IDs with your own.<br><br>### Customization tips<br>- Edit the `route` rule in **Validate Submission** to match your programmes.<br>- Add form fields, then map them in **Create Lead Record**. <br> ⚠️ Before you go live<br>Set your own header-auth `token`, and swap the demo Airtable base & Slack channel IDs for your own. |
| Validate Submission | n8n-nodes-base.set | Normalizes fields and evaluates validation rules | Lead Intake Webhook | Is Valid? | ## 1 · Validate submission<br>Normalise the payload, then check required fields and email format. `isValid` gates the next branch. |
| Is Valid? | n8n-nodes-base.if | Gates flow based on submission validity | Validate Submission | Find Existing by Email, Respond 400 Invalid | ## 1 · Validate submission<br>Normalise the payload, then check required fields and email format. `isValid` gates the next branch. |
| Find Existing by Email | n8n-nodes-base.airtable | Queries Airtable to check for duplicate emails | Is Valid? | Errored? | ## 2 · Dedupe by email<br>Look the email up in Airtable. If it already exists, return **200 “duplicate”** and stop. |
| Is Duplicate? | n8n-nodes-base.if | Checks if an existing record was found | Errored? | Respond 200 Duplicate, Create Lead Record | ## 2 · Dedupe by email<br>Look the email up in Airtable. If it already exists, return **200 “duplicate”** and stop. |
| Respond 200 Duplicate | n8n-nodes-base.respondToWebhook | Returns 200 OK for duplicate leads | Is Duplicate? | None | ## 2 · Dedupe by email<br>Look the email up in Airtable. If it already exists, return **200 “duplicate”** and stop. |
| Create Lead Record | n8n-nodes-base.airtable | Creates a new record in Airtable | Is Duplicate? | Route by Programme, Notify Failure Channel | ## 3 · Store, route & notify<br>Create the Airtable lead, then route **Computer Science → priority** Slack and **others → general**, and respond **200 “created”**. |
| Route by Programme | n8n-nodes-base.switch | Routes leads by programme name | Create Lead Record | Notify Priority Channel, Notify General Channel | ## 3 · Store, route & notify<br>Create the Airtable lead, then route **Computer Science → priority** Slack and **others → general**, and respond **200 “created”**. |
| Notify Priority Channel | n8n-nodes-base.slack | Alerts priority Slack channel | Route by Programme | Respond 200 Created | ## 3 · Store, route & notify<br>Create the Airtable lead, then route **Computer Science → priority** Slack and **others → general**, and respond **200 “created”**. |
| Respond 200 Created | n8n-nodes-base.respondToWebhook | Returns 200 OK for successfully created leads | Notify Priority Channel, Notify General Channel | None | ## 3 · Store, route & notify<br>Create the Airtable lead, then route **Computer Science → priority** Slack and **others → general**, and respond **200 “created”**. |
| Notify General Channel | n8n-nodes-base.slack | Alerts general Slack channel | Route by Programme | Respond 200 Created | ## 3 · Store, route & notify<br>Create the Airtable lead, then route **Computer Science → priority** Slack and **others → general**, and respond **200 “created”**. |
| Notify Failure Channel | n8n-nodes-base.slack | Alerts monitoring Slack channel of write failures | Create Lead Record, Errored? | Respond 500 Failure | ## On write failure<br>Airtable write failed → alert the failure Slack channel and return **HTTP 500**. |
| Respond 400 Invalid | n8n-nodes-base.respondToWebhook | Returns 400 Bad Request for invalid validation | Is Valid? | None | ## Reject invalid<br>Malformed submissions return **HTTP 400** with the specific validation errors. |
| Respond 500 Failure | n8n-nodes-base.respondToWebhook | Returns 500 Error for internal storage failures | Notify Failure Channel | None | ## On write failure<br>Airtable write failed → alert the failure Slack channel and return **HTTP 500**. |
| Errored? | n8n-nodes-base.if | Checks if Airtable lookup failed | Find Existing by Email | Notify Failure Channel, Is Duplicate? | ## 2 · Dedupe by email<br>Look the email up in Airtable. If it already exists, return **200 “duplicate”** and stop. |
| On Workflow Error | n8n-nodes-base.errorTrigger | Triggers on global unhandled workflow errors | None | Alert Error Channel | ## Global error handler<br>The Error Trigger catches any unhandled failure and posts the details to the monitoring Slack channel. |
| Alert Error Channel | n8n-nodes-base.slack | Sends global crash notification to Slack | On Workflow Error | None | ## Global error handler<br>The Error Trigger catches any unhandled failure and posts the details to the monitoring Slack channel. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger**
   - Add a **Webhook** node named `Lead Intake Webhook`.
   - Set HTTP Method to `POST`, path to `lead-intake`, response mode to `Respond using 'Respond to Webhook' node`, and authentication to `Header Auth`.
   - Configure credentials for HTTP Header Auth (`local test` or your custom token).

2. **Add Payload Normalization and Validation**
   - Connect a **Set** node named `Validate Submission` to the webhook.
   - Configure assignments to parse values for `name`, `email`, `phone`, `country`, `programme`, `route` (evaluating equality to 'Computer Science'), `isValid` (boolean evaluation checking fields and email regex), and `validationErrors` (string aggregation).

3. **Configure the Validity Branching**
   - Connect an **If** node named `Is Valid?` to `Validate Submission`.
   - Add a condition checking that `{{ $('Validate Submission').item.json.isValid }}` equals `true`.
   - Connect the **false** output to a **Respond to Webhook** node named `Respond 400 Invalid` (HTTP Status code `400`, JSON body returning rejection status and validation errors).

4. **Implement Airtable Deduplication**
   - From the **true** output of `Is Valid?`, connect an **Airtable** node named `Find Existing by Email`.
   - Set operation to `search`, configure table to your Leads table, and set formula filtering: `={{ "LOWER({Email}) = "+ JSON.stringify($('Validate Submission').item.json.email) }}`. Enable retry logic and set error behavior to continue regular output.
   - Connect `Find Existing by Email` to an **If** node named `Errored?` checking if `{{ $json.error }}` exists.
   - Connect the **false** output of `Errored?` to an **If** node named `Is Duplicate?` checking if `{{ $('Find Existing by Email').item.json.id }}` exists.
   - Connect the **true** output of `Is Duplicate?` to a **Respond to Webhook** node named `Respond 200 Duplicate` (HTTP Status code `200`, JSON response confirming duplicate status).

5. **Build Storage and Programme Routing Logic**
   - Connect the **false** output of `Is Duplicate?` to an **Airtable** node named `Create Lead Record` (Operation: `create`, map fields: `Name`, `Email`, `Phone`, `Route`, `Source`, `Country`, `Programme`, `Submitted At`). Enable typecasting and error output routing.
   - Connect the main output of `Create Lead Record` to a **Switch** node named `Route by Programme`. Configure rules to match where route equals `Priority`, setting the fallback output name to `General`.
   - Connect the **Priority** output to a **Slack** node named `Notify Priority Channel` targeting your priority channel ID with custom message formatting.
   - Connect the **General** fallback output to a **Slack** node named `Notify General Channel` targeting your general channel ID with custom message formatting.
   - Both Slack notification nodes connect into a final **Respond to Webhook** node named `Respond 200 Created` (HTTP Status code `200`, JSON confirmation body).

6. **Establish Failure and Error Handlers**
   - Connect the error output of `Create Lead Record` and the **true** output of `Errored?` to a **Slack** node named `Notify Failure Channel` (targeting your failure monitoring channel ID, outputting error information and execution ID).
   - Connect `Notify Failure Channel` to a **Respond to Webhook** node named `Respond 500 Failure` (HTTP Status code `500`, JSON failure message).
   - Add an **Error Trigger** node named `On Workflow Error`.
   - Connect `On Workflow Error` to a **Slack** node named `Alert Error Channel` (targeting the monitoring channel ID to log global crash summaries).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Author Contact Email | salmanmehboob1947@gmail.com |
| Developer LinkedIn Profile | https://www.linkedin.com/in/salman-mehboob-pro/ |