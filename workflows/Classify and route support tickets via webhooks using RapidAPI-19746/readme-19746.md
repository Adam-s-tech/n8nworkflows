Classify and route support tickets via webhooks using RapidAPI

https://n8nworkflows.xyz/workflows/classify-and-route-support-tickets-via-webhooks-using-rapidapi-19746


# Classify and route support tickets via webhooks using RapidAPI

### 1. Workflow Overview

This workflow is designed to automate the initial intake, AI-powered classification, and intelligent routing of customer support tickets. Its primary purpose is to eliminate manual triage by analyzing incoming support requests via an external AI classifier API, evaluating their urgency, and dividing them into either an immediate escalation path for high-priority incidents or a standard update path for regular queries.

The workflow logic is grouped into four functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests from any ticketing source or web form and standardizes the ticket body into a unified format.
- **1.2 AI Analysis & Triage:** Sends the normalized text to a RapidAPI classification endpoint to extract structured metadata (priority, category, sentiment, SLA risk, and suggested tags).
- **1.3 Escalation Routing:** Evaluates whether a ticket is marked as `high` or `critical`, constructs an urgent alert message, and routes it to placeholder incident notification services.
- **1.4 Standard Routing:** Processes non-urgent tickets by formatting a structured helpdesk update payload for standard ticketing platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** This block acts as the entry point, capturing raw webhook payloads from external support channels and normalizing varying property names into a consistent internal data structure.
- **Nodes Involved:** 
  - `When Ticket Created`
  - `Normalize Ticket Content`
- **Node Details:**
  - **When Ticket Created**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (v2). Acts as a webhook trigger listening for incoming POST requests.
    - *Configuration Choices:* Configured to listen on path `new-support-ticket` with method `POST` and response mode set to `onReceived`.
    - *Key Expressions:* None (triggers on payload arrival).
    - *Input/Output Connections:* Output connects to `Normalize Ticket Content`.
    - *Edge Cases / Potential Failures:* Network dropouts, invalid JSON payloads, or missing authentication tokens if secured externally.
  - **Normalize Ticket Content**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Transforms and normalizes incoming data properties.
    - *Configuration Choices:* Manual assignment mode.
    - *Key Expressions:* Assigns a string value to the `ticket` property using fallback properties: `={{ $json.body?.ticket || $json.body?.description || $json.body?.comment || $json.ticket || '' }}`.
    - *Input/Output Connections:* Input from `When Ticket Created`; output connects to `Post to RapidAPI for Analysis`.
    - *Edge Cases / Potential Failures:* Expression failure if data structure diverges entirely; results in an empty string if all fallback properties evaluate to undefined.

#### 2.2 AI Analysis & Triage
- **Overview:** This block sends the normalized support ticket to a third-party AI classifier via HTTP, retrieving critical metadata such as priority, category, sentiment, and SLA risks.
- **Nodes Involved:**
  - `Post to RapidAPI for Analysis`
  - `Check Ticket Priority`
- **Node Details:**
  - **Post to RapidAPI for Analysis**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Makes an external API call to an HTTP endpoint.
    - *Configuration Choices:* Sends a `POST` request to `https://support-ticket-triage.p.rapidapi.com/analyze-ticket`. Uses Generic Credential Type with HTTP Header Auth. Includes headers `X-RapidAPI-Host` and `Content-Type: application/json`.
    - *Key Expressions:* JSON body payload: `={{ { "ticket": $json.ticket } }}`.
    - *Input/Output Connections:* Input from `Normalize Ticket Content`; output connects to `Check Ticket Priority`.
    - *Edge Cases / Potential Failures:* API rate limiting (e.g., exceeding the free plan limits), invalid API keys, network timeouts, or upstream API downtime.
  - **Check Ticket Priority**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2). Conditional router that splits the workflow based on data attributes.
    - *Configuration Choices:* Evaluates conditions using a logical `OR` combinator with strict type validation.
    - *Key Expressions:* Checks if `{{ $json.priority }}` equals `critical` or `high`.
    - *Input/Output Connections:* Input from `Post to RapidAPI for Analysis`. True output branch connects to `Prepare Escalation Alert`; false output branch connects to `Prepare Standard Ticket Update`.
    - *Edge Cases / Potential Failures:* Missing `priority` field in the API response will cause the condition to evaluate to false.

#### 2.3 Escalation Routing
- **Overview:** Formats high-priority or critical incidents into actionable alert text and routes them to alerting infrastructure placeholders.
- **Nodes Involved:**
  - `Prepare Escalation Alert`
  - `Send Notification to Apps`
- **Node Details:**
  - **Prepare Escalation Alert**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Prepares structured notification data.
    - *Configuration Choices:* Manual assignment mode.
    - *Key Expressions:* Constructs an `alert_message` string using template interpolation: `=🚨 [{{ $json.priority.toUpperCase() }}] {{ $json.one_line_summary }}\nTeam: {{ $json.suggested_team }} | SLA risk: {{ $json.sla_breach_risk }}\nSignals: {{ $json.urgency_signals.join(', ') }}`.
    - *Input/Output Connections:* Input from the True branch of `Check Ticket Priority`; output connects to `Send Notification to Apps`.
    - *Edge Cases / Potential Failures:* JavaScript runtime errors if expected array properties (like `urgency_signals`) are null or undefined when `.join()` is called.
  - **Send Notification to Apps**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (v1). Placeholder node representing notification dispatches.
    - *Configuration Choices:* None (no-operation).
    - *Input/Output Connections:* Input from `Prepare Escalation Alert`; no outgoing connections.
    - *Edge Cases / Potential Failures:* None (placeholder).

#### 2.4 Standard Routing
- **Overview:** Consolidates non-urgent ticket data into a structured payload for standard helpdesk updates.
- **Nodes Involved:**
  - `Prepare Standard Ticket Update`
  - `Update Helpdesk Systems`
- **Node Details:**
  - **Prepare Standard Ticket Update**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Formats data object for downstream systems.
    - *Configuration Choices:* Manual assignment mode.
    - *Key Expressions:* Creates a structured `helpdesk_update` object: `={{ { priority: $json.priority, category: $json.category, tags: $json.suggested_tags, team: $json.suggested_team, summary: $json.one_line_summary } }}`.
    - *Input/Output Connections:* Input from the False branch of `Check Ticket Priority`; output connects to `Update Helpdesk Systems`.
    - *Edge Cases / Potential Failures:* Property mapping errors if the upstream API response schema changes unexpectedly.
  - **Update Helpdesk Systems**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (v1). Placeholder node representing ticketing system updates.
    - *Configuration Choices:* None (no-operation).
    - *Input/Output Connections:* Input from `Prepare Standard Ticket Update`; no outgoing connections.
    - *Edge Cases / Potential Failures:* None (placeholder).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When Ticket Created` | `n8n-nodes-base.webhook` | Receives incoming ticket webhooks via HTTP POST | None | `Normalize Ticket Content` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works<br><br>This workflow receives newly created support tickets through a webhook, normalizes the ticket text, and sends it to a RapidAPI-based triage service for classification. It then checks whether the returned priority is high or critical and routes the ticket into either an escalation path or a standard helpdesk update path. The final nodes are placeholders for notifying incident channels or updating a ticketing system.<br><br>### Setup steps<br><br>- Configure the webhook URL in the source support system so new tickets trigger the workflow.<br>- Set the normalization fields to match the incoming ticket payload, such as subject, description, requester, and ticket ID.<br>- Add the required RapidAPI credentials and confirm the triage API endpoint, headers, and request body mapping are correct.<br>- Replace the no-op destination nodes with real Slack, PagerDuty, Teams, Zendesk, Freshdesk, or Jira integrations as needed.<br><br>### Customization<br><br>Adjust the IF condition to match the exact priority labels returned by the classifier, and customize the escalation alert or helpdesk update fields to fit your support process. |
| `Normalize Ticket Content` | `n8n-nodes-base.set` | Normalizes incoming payload to a single ticket text field | `When Ticket Created` | `Post to RapidAPI for Analysis` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Receive and normalize ticket<br><br>Captures a newly created ticket from the webhook and prepares a normalized ticket text payload for downstream analysis. |
| `Normalize Ticket Content` | `n8n-nodes-base.set` | Normalizes incoming payload to a single ticket text field | `When Ticket Created` | `Post to RapidAPI for Analysis` | ## Receive and normalize ticket<br><br>Captures a newly created ticket from the webhook and prepares a normalized ticket text payload for downstream analysis. |
| `Post to RapidAPI for Analysis` | `n8n-nodes-base.httpRequest` | Calls RapidAPI endpoint for ticket classification | `Normalize Ticket Content` | `Check Ticket Priority` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Analyze and route priority<br><br>Sends the normalized ticket to the RapidAPI triage classifier, then branches based on whether the resulting priority is high or critical. |
| `Post to RapidAPI for Analysis` | `n8n-nodes-base.httpRequest` | Calls RapidAPI endpoint for ticket classification | `Normalize Ticket Content` | `Check Ticket Priority` | ## Analyze and route priority<br><br>Sends the normalized ticket to the RapidAPI triage classifier, then branches based on whether the resulting priority is high or critical. |
| `Check Ticket Priority` | `n8n-nodes-base.if` | Branches execution based on ticket priority levels | `Post to RapidAPI for Analysis` | `Prepare Escalation Alert`, `Prepare Standard Ticket Update` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Analyze and route priority<br><br>Sends the normalized ticket to the RapidAPI triage classifier, then branches based on whether the resulting priority is high or critical. |
| `Check Ticket Priority` | `n8n-nodes-base.if` | Branches execution based on ticket priority levels | `Post to RapidAPI for Analysis` | `Prepare Escalation Alert`, `Prepare Standard Ticket Update` | ## Analyze and route priority<br><br>Sends the normalized ticket to the RapidAPI triage classifier, then branches based on whether the resulting priority is high or critical. |
| `Prepare Escalation Alert` | `n8n-nodes-base.set` | Builds the structured alert string for urgent incidents | `Check Ticket Priority` (True) | `Send Notification to Apps` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Escalate urgent tickets<br><br>Builds an escalation alert for high-priority tickets and passes it to the placeholder notification step for Slack, PagerDuty, or Teams. |
| `Prepare Escalation Alert` | `n8n-nodes-base.set` | Builds the structured alert string for urgent incidents | `Check Ticket Priority` (True) | `Send Notification to Apps` | ## Escalate urgent tickets<br><br>Builds an escalation alert for high-priority tickets and passes it to the placeholder notification step for Slack, PagerDuty, or Teams. |
| `Send Notification to Apps` | `n8n-nodes-base.noOp` | Placeholder for notification delivery integrations | `Prepare Escalation Alert` | None | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Escalate urgent tickets<br><br>Builds an escalation alert for high-priority tickets and passes it to the placeholder notification step for Slack, PagerDuty, or Teams. |
| `Send Notification to Apps` | `n8n-nodes-base.noOp` | Placeholder for notification delivery integrations | `Prepare Escalation Alert` | None | ## Escalate urgent tickets<br><br>Builds an escalation alert for high-priority tickets and passes it to the placeholder notification step for Slack, PagerDuty, or Teams. |
| `Prepare Standard Ticket Update` | `n8n-nodes-base.set` | Prepares structured metadata object for helpdesk updates | `Check Ticket Priority` (False) | `Update Helpdesk Systems` | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Update standard tickets<br><br>Builds a standard helpdesk update for non-urgent tickets and passes it to the placeholder ticketing-system update step. |
| `Prepare Standard Ticket Update` | `n8n-nodes-base.set` | Prepares structured metadata object for helpdesk updates | `Check Ticket Priority` (False) | `Update Helpdesk Systems` | ## Update standard tickets<br><br>Builds a standard helpdesk update for non-urgent tickets and passes it to the placeholder ticketing-system update step. |
| `Update Helpdesk Systems` | `n8n-nodes-base.noOp` | Placeholder for helpdesk system update integrations | `Prepare Standard Ticket Update` | None | ## Support Ticket Triage & Classifier — Auto-Route New Tickets<br><br>### How it works...<br>*(Duplicate content due to parent sticky note)*<br><br>## Update standard tickets<br><br>Builds a standard helpdesk update for non-urgent tickets and passes it to the placeholder ticketing-system update step. |
| `Update Helpdesk Systems` | `n8n-nodes-base.noOp` | Placeholder for helpdesk system update integrations | `Prepare Standard Ticket Update` | None | ## Update standard tickets<br><br>Builds a standard helpdesk update for non-urgent tickets and passes it to the placeholder ticketing-system update step. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node:**
   - Add a **Webhook** node named `When Ticket Created`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `new-support-ticket`.
   - Set **Response Mode** to `On Received`.

2. **Create the Normalization Set Node:**
   - Add a **Set** node named `Normalize Ticket Content`.
   - Connect the output of `When Ticket Created` to this node.
   - Configure a string assignment named `ticket` with the expression: `={{ $json.body?.ticket || $json.body?.description || $json.body?.comment || $json.ticket || '' }}`.

3. **Create the HTTP Request Node for API Analysis:**
   - Add an **HTTP Request** node named `Post to RapidAPI for Analysis`.
   - Connect the output of `Normalize Ticket Content` to this node.
   - Set **Method** to `POST`.
   - Set **URL** to `https://support-ticket-triage.p.rapidapi.com/analyze-ticket`.
   - Set **Body Content Type** to `JSON`.
   - Set **JSON Body** expression to: `={{ { "ticket": $json.ticket } }}`.
   - Add Header Parameters:
     - `X-RapidAPI-Host`: `support-ticket-triage.p.rapidapi.com`
     - `Content-Type`: `application/json`
   - Configure **Authentication** using Generic Credential Type with **HTTP Header Auth** (`X-RapidAPI-Key`).

4. **Create the Priority Router IF Node:**
   - Add an **IF** node named `Check Ticket Priority`.
   - Connect the output of `Post to RapidAPI for Analysis` to this node.
   - Configure conditions using a combinator of `OR` with strict type validation:
     - Condition 1: Left Value `={{ $json.priority }}` equals `critical`
     - Condition 2: Left Value `={{ $json.priority }}` equals `high`

5. **Create the Escalation Path Nodes:**
   - Add a **Set** node named `Prepare Escalation Alert`. Connect the **True** output branch of `Check Ticket Priority` to this node.
     - Configure a string assignment named `alert_message` with the expression: `=🚨 [{{ $json.priority.toUpperCase() }}] {{ $json.one_line_summary }}\nTeam: {{ $json.suggested_team }} | SLA risk: {{ $json.sla_breach_risk }}\nSignals: {{ $json.urgency_signals.join(', ') }}`.
   - Add a **No Operation (NoOp)** node named `Send Notification to Apps`. Connect the output of `Prepare Escalation Alert` to this node.

6. **Create the Standard Routing Path Nodes:**
   - Add a **Set** node named `Prepare Standard Ticket Update`. Connect the **False** output branch of `Check Ticket Priority` to this node.
     - Configure an object assignment named `helpdesk_update` with the expression: `={{ { priority: $json.priority, category: $json.category, tags: $json.suggested_tags, team: $json.suggested_team, summary: $json.one_line_summary } }}`.
   - Add a **No Operation (NoOp)** node named `Update Helpdesk Systems`. Connect the output of `Prepare Standard Ticket Update` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full API reference (endpoint docs, response schema, error responses, and field mappings for Zendesk/Jira/Freshdesk/Salesforce) | [RapidAPI Listing](https://rapidapi.com/faisalakhtar12/api/support-ticket-triage) |