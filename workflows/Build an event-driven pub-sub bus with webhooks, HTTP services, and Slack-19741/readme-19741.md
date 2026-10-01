Build an event-driven pub/sub bus with webhooks, HTTP services, and Slack

https://n8nworkflows.xyz/workflows/build-an-event-driven-pub-sub-bus-with-webhooks--http-services--and-slack-19741


# Build an event-driven pub/sub bus with webhooks, HTTP services, and Slack

### 1. Workflow Overview

This workflow implements a robust, enterprise-grade event bus within n8n. It serves as a centralized pub/sub messaging backbone that ingests event envelopes from external webhooks or internal n8n workflows, validates and securely records them in an external event store, routes them to specific subscriber services via fan-out logic, and handles failures gracefully using exponential backoff retries, a dead-letter queue (DLQ), and automated Slack alerts.

The execution logic is organized into three distinct functional blocks:

- **1.1 Event Ingress, Validation & Durable Log:** Standardizes incoming requests from multiple entry points into a single envelope schema, validates mandatory fields, rejects invalid entries, and durably persists valid events to an event store before fan-out routing begins.
- **1.2 Routing & Fan-Out to Subscribers:** Inspects the event type to route payloads concurrently across independent subscriber endpoints (Order, Notification, and Analytics services), capturing unmapped events safely through a catch-all route.
- **1.3 Retry, Dead-Letter Queue & Acknowledgement:** Monitors subscriber outcomes to either acknowledge successful deliveries, execute exponential backoff delays for transient failures, or route exhausted attempts to a dead-letter queue while simultaneously alerting operations via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Ingress, Validation & Durable Log

##### Overview
This block unifies multiple ingress triggers into a single pipeline, loads environment and endpoint configurations, checks incoming event payloads against required schemas, rejects malformed requests immediately, and commits valid events to an immutable event store.

##### Nodes Involved
- `Event Ingress - Webhook`
- `Event Ingress - Execute Workflow Trigger`
- `Merge - Ingress Sources`
- `Set - Event Bus Config`
- `Code 1 - Validate & Enrich Envelope`
- `IF - Envelope Valid`
- `Respond - Invalid Envelope`
- `HTTP Request - Append to Event Store`
- `Respond - Event Accepted`

##### Node Details

- **Event Ingress - Webhook**
  - **Type and technical role:** `n8n-nodes-base.webhook` (Ingress Trigger). Accepts incoming HTTP POST requests containing event envelopes.
  - **Configuration choices:** Configured for HTTP method `POST` on path `event-bus/publish` with the response mode set to use a response node.
  - **Key expressions or variables:** None.
  - **Input and output connections:** Inputs: None (Entry point). Output: `Merge - Ingress Sources` (Index 0).
  - **Version-specific requirements:** Type Version 2.
  - **Edge cases or potential failure types:** Unauthenticated access or malformed JSON payloads resulting in HTTP 400 errors.

- **Event Ingress - Execute Workflow Trigger**
  - **Type and technical role:** `n8n-nodes-base.executeWorkflowTrigger` (Internal Trigger). Allows other local n8n workflows to push events directly into the bus without HTTP overhead.
  - **Configuration choices:** Configures workflow inputs for `eventType`, `eventId`, `source`, and `payload`.
  - **Key expressions or variables:** None.
  - **Input and output connections:** Inputs: None (Entry point). Output: `Merge - Ingress Sources` (Index 1).
  - **Version-specific requirements:** Type Version 1.1.
  - **Edge cases or potential failure types:** Missing required parameters passed from parent workflows.

- **Merge - Ingress Sources**
  - **Type and technical role:** `n8n-nodes-base.merge` (Flow Control). Consolidates the webhook and execute-workflow trigger streams into a single downstream execution path.
  - **Configuration choices:** Mode set to `chooseBranch`.
  - **Key expressions or variables:** None.
  - **Input and output connections:** Inputs: `Event Ingress - Webhook`, `Event Ingress - Execute Workflow Trigger`. Output: `Set - Event Bus Config`.
  - **Version-specific requirements:** Type Version 3.
  - **Edge cases or potential failure types:** Simultaneous trigger executions might queue items incorrectly if concurrency limits are exceeded.

- **Set - Event Bus Config**
  - **Type and technical role:** `n8n-nodes-base.set` (Data Transformation). Injects global infrastructure configurations, endpoint URLs, and operational parameters into the execution context.
  - **Configuration choices:** Assigns string values for `eventStoreUrl`, `eventStoreAckUrl`, `dlqUrl`, service URLs, and the Slack channel, alongside numeric values for `maxRetryAttempts` (3) and `baseBackoffSeconds` (5).
  - **Key expressions or variables:** None.
  - **Input and output connections:** Inputs: `Merge - Ingress Sources`. Output: `Code 1 - Validate & Enrich Envelope`.
  - **Version-specific requirements:** Type Version 3.4.
  - **Edge cases or potential failure types:** Outdated or unreachable endpoint URLs defined in assignments.

- **Code 1 - Validate & Enrich Envelope**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation / Validation). Normalizes raw input bodies, auto-generates UUIDs and ISO timestamps if missing, and validates required envelope parameters.
  - **Configuration choices:** Executes JavaScript to inspect the input object structure, compute missing fields, and construct a standardized enriched JSON object.
  - **Key expressions or variables:** Uses `$('Set - Event Bus Config').first().json` and `$input.first().json`.
  - **Input and output connections:** Inputs: `Set - Event Bus Config`. Output: `IF - Envelope Valid`.
  - **Version-specific requirements:** Type Version 2.
  - **Edge cases or potential failure types:** Script execution errors if incoming payloads lack expected structural properties.

- **IF - Envelope Valid**
  - **Type and technical role:** `n8n-nodes-base.if` (Flow Control). Branches the execution based on the validation status generated by the preceding code node.
  - **Configuration choices:** Evaluates whether `{{ $json.valid }}` evaluates to `true`.
  - **Key expressions or variables:** `={{ $json.valid }}`
  - **Input and output connections:** Inputs: `Code 1 - Validate & Enrich Envelope`. Outputs: True branch to `HTTP Request - Append to Event Store`; False branch to `Respond - Invalid Envelope`.
  - **Version-specific requirements:** Type Version 2.2.
  - **Edge cases or potential failure types:** Type mismatch in boolean evaluation.

- **Respond - Invalid Envelope**
  - **Type and technical role:** `n8n-nodes-base.respondToWebhook` (Webhook Response). Returns an immediate HTTP 400 rejection payload back to the webhook caller.
  - **Configuration choices:** Response code configured to `400`, responding with JSON containing failure reasons and missing fields.
  - **Key expressions or variables:** `={{ JSON.stringify({ status: 'rejected', reason: 'INVALID_ENVELOPE', missingFields: $json.missingFields }) }}`
  - **Input and output connections:** Inputs: `IF - Envelope Valid` (False branch). Output: None (Terminal node for invalid path).
  - **Version-specificrequirements:** Type Version 1.4.
  - **Edge cases or potential failure types:** Triggered only on webhook calls; fails silently if invoked from internal workflow triggers due to lack of an active HTTP response context.

- **HTTP Request - Append to Event Store**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Persists the validated event to an append-only external event store for auditing and replay capabilities.
  - **Configuration choices:** Sends a POST request with a JSON body containing event identifiers, source, type, payload, and status. Configured with built-in retry options (`retryOnFail` enabled, max 2 tries, 2000ms delay) and error output continuation.
  - **Key expressions or variables:** `={{ $json.eventStoreUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `IF - Envelope Valid` (True branch). Outputs: Main output connects to `Respond - Event Accepted` and `Switch - Route by Event Type`. Error output configured.
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** Network timeouts or authentication failures against the external event store endpoint.

- **Respond - Event Accepted**
  - **Type and technical role:** `n8n-nodes-base.respondToWebhook` (Webhook Response). Returns an HTTP 202 "Accepted" response to the event producer confirming durable storage.
  - **Configuration choices:** Response code set to `400` (Note: configured for asynchronous acceptance acknowledgment with response code `202`).
  - **Key expressions or variables:** `={{ JSON.stringify({ status: 'accepted', eventId: $json.eventId }) }}`
  - **Input and output connections:** Inputs: `HTTP Request - Append to Event Store`. Output: None (Terminal node for ingress response path).
  - **Version-specific requirements:** Type Version 1.4.
  - **Edge cases or potential failure types:** Delivery attempts on internal workflow triggers without an HTTP context.

---

#### 2.2 Routing & Fan-Out to Subscribers

##### Overview
This block inspects the normalized event type using conditional routing rules and fans out the event payload concurrently across appropriate downstream subscriber microservices, ensuring unmatched event types fall back gracefully to auditing endpoints.

##### Nodes Involved
- `Switch - Route by Event Type`
- `Subscriber - Order Service`
- `Subscriber - Notification Service`
- `Code - Unknown Event Type`
- `Subscriber - Analytics Service`
- `Merge - Subscriber Results`

##### Node Details

- **Switch - Route by Event Type**
  - **Type and technical role:** `n8n-nodes-base.switch` (Flow Control). Evaluates the event type to route execution down targeted subscriber branches simultaneously (fan-out pattern).
  - **Configuration choices:** Rule 1 matches `eventType` equals `order.created`; Rule 2 matches `eventType` equals `user.signed_up`. Fallback output enabled with all-matching outputs permitted.
  - **Key expressions or variables:** `={{ $json.eventType }}`
  - **Input and output connections:** Inputs: `HTTP Request - Append to Event Store`. Outputs: Output 0 to `Subscriber - Order Service`; Output 1 to `Subscriber - Notification Service`; Output 2 (Fallback) to `Code - Unknown Event Type`.
  - **Version-specific requirements:** Type Version 3.2.
  - **Edge cases or potential failure types:** Unmatched event types bypass primary subscribers and route exclusively to analytics.

- **Subscriber - Order Service**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Delivers order-related event payloads to the Order microservice endpoint.
  - **Configuration choices:** POST request with a 10-second timeout, JSON body containing event metadata, and error handling set to continue on failure.
  - **Key expressions or variables:** `={{ $json.orderServiceUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `Switch - Route by Event Type` (Output 0). Outputs: Main and error outputs connect to `Merge - Subscriber Results` (Index 0).
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** Downstream service downtime, HTTP 5xx errors, or timeout exceptions.

- **Subscriber - Notification Service**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Delivers notification-relevant events to the Notification microservice endpoint.
  - **Configuration choices:** POST request with a 10-second timeout, JSON body, and error handling set to continue on failure.
  - **Key expressions or variables:** `={{ $json.notificationServiceUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `Switch - Route by Event Type` (Output 1). Outputs: Main and error outputs connect to `Merge - Subscriber Results` (Index 1).
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** Network drops, service unavailability, or schema validation rejections at the subscriber end.

- **Code - Unknown Event Type**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation). Appends an informative note to unrouted event types before sending them to the audit/analytics layer.
  - **Configuration choices:** JavaScript assignment adding a descriptive note indicating fallback handling.
  - **Key expressions or variables:** Uses `$input.first().json`.
  - **Input and output connections:** Inputs: `Switch - Route by Event Type` (Fallback). Output: `Subscriber - Analytics Service`.
  - **Version-specific requirements:** Type Version 2.
  - **Edge cases or potential failure types:** None.

- **Subscriber - Analytics Service**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Acts as a catch-all subscriber delivering all event traffic to an analytics or audit log service.
  - **Configuration choices:** POST request with a 10-second timeout, JSON body including source metadata, and error handling set to continue on failure.
  - **Key expressions or variables:** `={{ $json.analyticsServiceUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `Code - Unknown Event Type` (and implicitly acts as a parallel branch for other flows). Outputs: Main and error outputs connect to `Merge - Subscriber Results`.
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** Analytics endpoint latency or ingestion failures.

- **Merge - Subscriber Results**
  - **Type and technical role:** `n8n-nodes-base.merge` (Flow Control). Reconciles the outputs from parallel subscriber execution branches back into a single processing stream.
  - **Configuration choices:** Mode set to `chooseBranch`.
  - **Key expressions or variables:** None.
  - **Input and output connections:** Inputs: Subscriber HTTP nodes. Output: `Code - Tag Subscriber & Outcome`.
  - **Version-specific requirements:** Type Version 3.
  - **Edge cases or potential failure types:** Misaligned branch arrivals if branch execution times vary drastically.

---

#### 2.3 Retry, Dead-Letter Queue & Acknowledgement

##### Overview
This block inspects the delivery outcome from each subscriber branch, records successful deliveries back to the event store, calculates exponential backoff intervals for failed transmissions, retries eligible attempts, and routes exhausted failures to a dead-letter queue accompanied by team Slack alerts.

##### Nodes Involved
- `Code - Tag Subscriber & Outcome`
- `IF - Delivery Failed`
- `HTTP Request - Ack Event Store`
- `Code - Compute Backoff & Retry Count`
- `IF - Attempts Exhausted`
- `Wait - Exponential Backoff`
- `HTTP Request - Write to Dead Letter Queue`
- `Slack - DLQ Alert`

##### Node Details

- **Code - Tag Subscriber & Outcome**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation). Evaluates execution output structures to determine if an HTTP call failed or succeeded.
  - **Configuration choices:** JavaScript checks for the presence of an `error` property injected by `continueOnFail` nodes and normalizes the outcome flags.
  - **Key expressions or variables:** Uses `$input.first().json`, `$execution.customData`.
  - **Input and output connections:** Inputs: `Merge - Subscriber Results`. Output: `IF - Delivery Failed`.
  - **Version-specific requirements:** Type Version 2.
  - **Edge cases or potential failure types:** Malformed error object structures from upstream HTTP responses.

- **IF - Delivery Failed**
  - **Type and technical role:** `n8n-nodes-base.if` (Flow Control). Separates successful deliveries from failed ones.
  - **Configuration choices:** Evaluates whether `{{ $json.deliveryFailed }}` is true.
  - **Key expressions or variables:** `={{ $json.deliveryFailed }}`
  - **Input and output connections:** Inputs: `Code - Tag Subscriber & Outcome`. Outputs: True branch to `Code - Compute Backoff & Retry Count`; False branch to `HTTP Request - Ack Event Store`.
  - **Version-specific requirements:** Type Version 2.2.
  - **Edge cases or potential failure types:** Incorrect boolean state resolution.

- **HTTP Request - Ack Event Store**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Reports successful delivery back to the event store to mark the event/subscriber pair as completed.
  - **Configuration choices:** POST request with retry configuration (max 2 tries, 2000ms delay) and error output continuation.
  - **Key expressions or variables:** `={{ $json.eventStoreAckUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `IF - Delivery Failed` (False branch). Output: None (Terminal success branch).
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** Acknowledgement endpoint unavailability.

- **Code - Compute Backoff & Retry Count**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation / State Management). Tracks delivery attempts using workflow static data keyed by event ID and computes exponential backoff delays.
  - **Configuration choices:** JavaScript accesses workflow static data (`$getWorkflowStaticData('node')`), increments attempt counters, computes backoff duration via exponential formulas, and flags exhaustion.
  - **Key expressions or variables:** Uses `$getWorkflowStaticData('node')`, `{{ $json.maxRetryAttempts }}`, `={{ $json.baseBackoffSeconds }}`.
  - **Input and output connections:** Inputs: `IF - Delivery Failed` (True branch). Output: `IF - Attempts Exhausted`.
  - **Version-specific requirements:** Type Version 2.
  - **Edge cases or potential failure types:** Static data memory limits across heavy execution loads.

- **IF - Attempts Exhausted**
  - **Type and technical role:** `n8n-nodes-base.if` (Flow Control). Determines whether to route a failed message to the dead-letter queue or queue it for a retry wait period.
  - **Configuration choices:** Evaluates whether `{{ $json.attemptsExhausted }}` is true.
  - **Key expressions or variables:** `={{ $json.attemptsExhausted }}`
  - **Input and output connections:** Inputs: `Code - Compute Backoff & Retry Count`. Outputs: True branch to `HTTP Request - Write to Dead Letter Queue`; False branch to `Wait - Exponential Backoff`.
  - **Version-specific requirements:** Type Version 2.2.
  - **Edge cases or potential failure types:** Miscalculated attempt limits leading to infinite retry loops.

- **Wait - Exponential Backoff**
  - **Type and technical role:** `n8n-nodes-base.wait` (Flow Control). Pauses execution for the dynamically calculated backoff duration before re-attempting subscriber delivery.
  - **Configuration choices:** Wait amount configured dynamically based on computed backoff seconds.
  - **Key expressions or variables:** `={{ $json.backoffSeconds }}`
  - **Input and output connections:** Inputs: `IF - Attempts Exhausted` (False branch). Output: `Merge - Subscriber Results` (Loops back into subscriber validation/delivery pipeline).
  - **Version-specific requirements:** Type Version 1.1.
  - **Edge cases or potential failure types:** Extended wait durations holding workflow execution slots open.

- **HTTP Request - Write to Dead Letter Queue**
  - **Type and technical role:** `n8n-nodes-base.httpRequest` (Integration). Persists terminally failed event payloads, error histories, and attempt metadata to an external DLQ endpoint.
  - **Configuration choices:** POST request with retry configuration (max 2 tries, 2000ms delay) and error output continuation.
  - **Key expressions or variables:** `={{ $json.dlqUrl }}`, `={{ JSON.stringify(...) }}`
  - **Input and output connections:** Inputs: `IF - Attempts Exhausted` (True branch). Output: `Slack - DLQ Alert`.
  - **Version-specific requirements:** Type Version 4.2.
  - **Edge cases or potential failure types:** DLQ endpoint unreachable, requiring secondary fallback logging.

- **Slack - DLQ Alert**
  - **Type and technical role:** `n8n-nodes-base.slack` (Notification). Sends an operational alert message to a designated Slack channel notifying the team of an unresolvable event delivery failure.
  - **Configuration choices:** Operation set to post a message. Requires Slack credentials.
  - **Key expressions or variables:** Relies on channel configuration defined in workflow state variables.
  - **Input and output connections:** Inputs: `HTTP Request - Write to Dead Letter Queue`. Output: None (Terminal alert path).
  - **Version-specific requirements:** Type Version 2.2.
  - **Edge cases or potential failure types:** Invalid Slack authentication credentials or unresolvably misconfigured channel names.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Event-Driven Architecture (Advanced)<br><br>A reusable, general-purpose event bus built in n8n. Any producer (webhook, cron, another workflow) publishes an *event envelope* — `{eventType, eventId, source, timestamp, payload}` — into a single **Event Ingress** entry point. From there:<br><br>1. **Validation & enrichment** — envelope is schema-checked and stamped with routing metadata<br>2. **Event Store (append-only log)** — every event is persisted before anything else happens, so nothing is ever routed without first being durably recorded (event sourcing pattern)<br>3. **Router (Switch)** — routes by `eventType` to one or more subscriber branches, fan-out style; unmatched types fall through to a catch-all<br>4. **Subscribers** — independent branches (order service, notification service, analytics service...) each with their own try/fail handling — one subscriber failing never blocks another<br>5. **Retry with backoff** — a failed subscriber delivery is retried a bounded number of times with increasing delay<br>6. **Dead-Letter Queue** — deliveries that exhaust retries are written to a DLQ table/queue and raise a Slack alert, instead of silently vanishing<br>7. **Ack** — successful delivery is recorded back on the event store row<br><br>This is the classic **pub/sub + event sourcing + DLQ** trio, implemented with n8n-native building blocks so it runs anywhere n8n runs, with no external broker required (swap the Postgres-style HTTP nodes for Kafka/RabbitMQ/SQS via their REST APIs if you outgrow this).<br><br>### Setup<br>1. Import into n8n<br>2. Set - Event Bus Config: event store endpoint, DLQ endpoint, retry policy, Slack channel, subscriber endpoints<br>3. Point each 'Subscriber -' HTTP node at your real service (or replace with a Function/DB node for in-n8n logic)<br>4. Wire your real producers (webhooks, other workflows via Execute Workflow, schedule triggers) into 'Event Ingress - Webhook' or trigger this as a sub-workflow with 'Event Ingress - Execute Workflow Trigger'<br>5. Activate<br><br>### Customize<br>- Add a subscriber: duplicate a 'Subscriber -' branch off the Switch node, add its eventType case<br>- Change fan-out to fan-in: replace Switch with a Merge if you want every event to hit every subscriber unconditionally<br>- Swap the event store / DLQ HTTP nodes for a native Postgres/Airtable node if you prefer |
| `Sticky Note - Stage 1` | `n8n-nodes-base.stickyNote` | Documentation for ingress and durable log stage | None | None | ## 1. Event Ingress, Validation & Durable Log<br><br>Two entry points feed the same pipeline:<br>- **Event Ingress - Webhook**: external producers POST an event envelope<br>- **Event Ingress - Execute Workflow Trigger**: other n8n workflows publish events by calling this workflow directly (in-process pub/sub, no HTTP hop)<br><br>Set - Event Bus Config centralizes the event store URL, DLQ URL, retry policy (max attempts, base delay), subscriber endpoints, and Slack channel.<br><br>Code 1 - Validate & Enrich Envelope checks the envelope has eventType, eventId (generates one if missing), source, timestamp, and payload; rejects malformed envelopes immediately (Respond - Invalid Envelope, webhook path only).<br><br>HTTP Request - Append to Event Store writes the raw event to an append-only log *before* any routing/delivery happens — this is what makes the bus replayable and auditable: even if every subscriber fails, the event itself is never lost. |
| `Sticky Note - Stage 2` | `n8n-nodes-base.stickyNote` | Documentation for routing and fan-out stage | None | None | ## 2. Routing & Fan-Out to Subscribers<br><br>Switch - Route by Event Type inspects `eventType` and sends the event down one or more of the matched branches, in parallel:<br>- `order.created` → Subscriber - Order Service<br>- `order.created` / `user.signed_up` → Subscriber - Notification Service (multiple types can share a subscriber)<br>- any type → Subscriber - Analytics Service (catch-all / audit trail, always fires)<br>- unmatched type → Code - Unknown Event Type (logged, not treated as an error)<br><br>Each Subscriber node is an independent HTTP call with `continueOnFail` + `retryOnFail` set at the node level for transient-fault tolerance, but the workflow-level retry/backoff/DLQ logic (next stage) is what handles a subscriber that's fully down, not just flaky. |
| `Sticky Note - Stage 3` | `n8n-nodes-base.stickyNote` | Documentation for retry, DLQ, and ack stage | None | None | ## 3. Retry, Dead-Letter Queue & Acknowledgement<br><br>Each subscriber branch converges into a shared IF - Delivery Failed gate.<br><br>- **Success** → HTTP Request - Ack Event Store (mark this event/subscriber pair as delivered) → done<br>- **Failure** → Code - Compute Backoff & Retry Count checks how many attempts have been made (tracked in workflow static data, keyed by eventId+subscriber) against the configured max attempts.<br>  - If attempts remain → Wait (exponential backoff) → loops back into the same subscriber call<br>  - If attempts are exhausted → HTTP Request - Write to Dead Letter Queue persists the event + failure reason + attempt history, then Slack - DLQ Alert notifies the team a message needed manual intervention<br><br>This means a subscriber outage never loses events (they sit safely in the DLQ, replayable later) and never silently fails without anyone knowing. |
| `Event Ingress - Webhook` | `n8n-nodes-base.webhook` | External HTTP webhook entry point | None | `Merge - Ingress Sources` | |
| `Event Ingress - Execute Workflow Trigger` | `n8n-nodes-base.executeWorkflowTrigger` | Internal sub-workflow entry point | None | `Merge - Ingress Sources` | |
| `Merge - Ingress Sources` | `n8n-nodes-base.merge` | Unifies ingress trigger sources | `Event Ingress - Webhook`, `Event Ingress - Execute Workflow Trigger` | `Set - Event Bus Config` | |
| `Set - Event Bus Config` | `n8n-nodes-base.set` | Injects global event bus infrastructure parameters | `Merge - Ingress Sources` | `Code 1 - Validate & Enrich Envelope` | |
| `Code 1 - Validate & Enrich Envelope` | `n8n-nodes-base.code` | Normalizes and validates the incoming event envelope | `Set - Event Bus Config` | `IF - Envelope Valid` | |
| `IF - Envelope Valid` | `n8n-nodes-base.if` | Gates execution based on envelope validity | `Code 1 - Validate & Enrich Envelope` | `HTTP Request - Append to Event Store`, `Respond - Invalid Envelope` | |
| `Respond - Invalid Envelope` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 for invalid webhooks | `IF - Envelope Valid` | None | |
| `HTTP Request - Append to Event Store` | `n8n-nodes-base.httpRequest` | Persists event to append-only log | `IF - Envelope Valid` | `Respond - Event Accepted`, `Switch - Route by Event Type` | |
| `Respond - Event Accepted` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 202 acceptance response | `HTTP Request - Append to Event Store` | None | |
| `Switch - Route by Event Type` | `n8n-nodes-base.switch` | Routes events concurrently by event type | `HTTP Request - Append to Event Store` | `Subscriber - Order Service`, `Subscriber - Notification Service`, `Code - Unknown Event Type` | |
| `Code - Unknown Event Type` | `n8n-nodes-base.code` | Logs unmapped event types for analytics fallback | `Switch - Route by Event Type` | `Subscriber - Analytics Service` | |
| `Subscriber - Order Service` | `n8n-nodes-base.httpRequest` | Delivers event to Order microservice | `Switch - Route by Event Type` | `Merge - Subscriber Results` | |
| `Subscriber - Notification Service` | `n8n-nodes-base.httpRequest` | Delivers event to Notification microservice | `Switch - Route by Event Type` | `Merge - Subscriber Results` | |
| `Subscriber - Analytics Service` | `n8n-nodes-base.httpRequest` | Delivers event to Analytics microservice | `Code - Unknown Event Type` | `Merge - Subscriber Results` | |
| `Merge - Subscriber Results` | `n8n-nodes-base.merge` | Consolidates parallel subscriber branch outputs | `Subscriber - Order Service`, `Subscriber - Notification Service`, `Wait - Exponential Backoff` | `Code - Tag Subscriber & Outcome` | |
| `Code - Tag Subscriber & Outcome` | `n8n-nodes-base.code` | Tags subscriber results with delivery success/failure flags | `Merge - Subscriber Results` | `IF - Delivery Failed` | |
| `IF - Delivery Failed` | `n8n-nodes-base.if` | Gates execution based on delivery failure status | `Code - Tag Subscriber & Outcome` | `HTTP Request - Ack Event Store`, `Code - Compute Backoff & Retry Count` | |
| `HTTP Request - Ack Event Store` | `n8n-nodes-base.httpRequest` | Acknowledges successful event delivery | `IF - Delivery Failed` | None | |
| `Code - Compute Backoff & Retry Count` | `n8n-nodes-base.code` | Manages retry state and exponential backoff calculations | `IF - Delivery Failed` | `IF - Attempts Exhausted` | |
| `IF - Attempts Exhausted` | `n8n-nodes-base.if` | Gates execution based on maximum retry attempts | `Code - Compute Backoff & Retry Count` | `HTTP Request - Write to Dead Letter Queue`, `Wait - Exponential Backoff` | |
| `Wait - Exponential Backoff` | `n8n-nodes-base.wait` | Pauses execution for exponential backoff delay | `IF - Attempts Exhausted` | `Merge - Subscriber Results` | |
| `HTTP Request - Write to Dead Letter Queue` | `n8n-nodes-base.httpRequest` | Writes terminally failed events to DLQ | `IF - Attempts Exhausted` | `Slack - DLQ Alert` | |
| `Slack - DLQ Alert` | `n8n-nodes-base.slack` | Posts operational failure alerts to Slack | `HTTP Request - Write to Dead Letter Queue` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually inside n8n:

1. **Create Entry Triggers:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`). Set HTTP Method to `POST`, Path to `event-bus/publish`, and Response Mode to `Response Node`. Name it `Event Ingress - Webhook`.
   - Add an **Execute Workflow Trigger** node (`n8n-nodes-base.executeWorkflowTrigger`). Configure workflow inputs with four string properties: `eventType`, `eventId`, `source`, and `payload`. Name it `Event Ingress - Execute Workflow Trigger`.

2. **Merge and Configure Ingress:**
   - Add a **Merge** node (`n8n-nodes-base.merge`). Set mode to `Choose Branch`. Connect both ingress triggers into this merge node. Name it `Merge - Ingress Sources`.
   - Add a **Set** node (`n8n-nodes-base.set`). Configure assignments for `eventStoreUrl`, `eventStoreAckUrl`, `dlqUrl`, `maxRetryAttempts` (number: 3), `baseBackoffSeconds` (number: 5), service URLs (`orderServiceUrl`, `notificationServiceUrl`, `analyticsServiceUrl`), and `slackDlqChannel`. Name it `Set - Event Bus Config`. Connect `Merge - Ingress Sources` to this node.

3. **Validation and Durable Logging:**
   - Add a **Code** node (`n8n-nodes-base.code`). Add JavaScript to normalize incoming bodies, validate mandatory fields (`eventType`, `payload`), auto-generate UUIDs if missing, and output enriched properties. Name it `Code 1 - Validate & Enrich Envelope`. Connect `Set - Event Bus Config` to it.
   - Add an **IF** node (`n8n-nodes-base.if`). Set condition to check if `{{ $json.valid }}` is true. Name it `IF - Envelope Valid`. Connect validation code to it.
   - Add a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`). Set response code to `400` and respond with JSON containing missing fields. Name it `Respond - Invalid Envelope`. Connect the false output of `IF - Envelope Valid` here.
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Set method to `POST`, URL to `{{ $json.eventStoreUrl }}`, specify body as JSON with event details, enable `retryOnFail` (max 2 tries, 2000ms delay), and set error output continuation. Name it `HTTP Request - Append to Event Store`. Connect the true output of `IF - Envelope Valid` here.
   - Add another **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`). Set response code to `202` and respond with JSON confirming acceptance. Name it `Respond - Event Accepted`. Connect success output of `HTTP Request - Append to Event Store` here.

4. **Routing and Fan-Out:**
   - Add a **Switch** node (`n8n-nodes-base.switch`). Create rules: Rule 0 matches `eventType` equals `order.created`; Rule 1 matches `eventType` equals `user.signed_up`. Enable fallback output. Name it `Switch - Route by Event Type`. Connect `HTTP Request - Append to Event Store` here.
   - Add three **HTTP Request** nodes for subscribers (`n8n-nodes-base.httpRequest`), each with `continueOnFail` enabled and a 10-second timeout:
     - Name the first `Subscriber - Order Service`, pointing to `{{ $json.orderServiceUrl }}`. Connect Switch Output 0 here.
     - Name the second `Subscriber - Notification Service`, pointing to `{{ $json.notificationServiceUrl }}`. Connect Switch Output 1 here.
     - Name the third `Subscriber - Analytics Service`, pointing to `{{ $json.analyticsServiceUrl }}`.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Unknown Event Type`. Connect the fallback output of `Switch - Route by Event Type` here, and connect its output to `Subscriber - Analytics Service`.
   - Add a **Merge** node (`n8n-nodes-base.merge`) set to `Choose Branch` named `Merge - Subscriber Results`. Connect outputs from `Subscriber - Order Service`, `Subscriber - Notification Service`, and `Subscriber - Analytics Service` (plus the backoff loop) into this merge node.

5. **Outcome Tagging and Error Handling:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Tag Subscriber & Outcome` to evaluate whether an `error` property exists on incoming items. Connect `Merge - Subscriber Results` here.
   - Add an **IF** node (`n8n-nodes-base.if`) named `IF - Delivery Failed` checking if `{{ $json.deliveryFailed }}` is true. Connect outcome tagging code here.
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `HTTP Request - Ack Event Store` pointing to `{{ $json.eventStoreAckUrl }}` with retry configuration. Connect the false output of `IF - Delivery Failed` here.

6. **Retry, Backoff, DLQ, and Alerting:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code - Compute Backoff & Retry Count`. Configure it to read workflow static data, increment attempt counters based on `eventId` and `eventType`, and calculate exponential backoff delay. Connect the true output of `IF - Delivery Failed` here.
   - Add an **IF** node (`n8n-nodes-base.if`) named `IF - Attempts Exhausted` checking if `{{ $json.attemptsExhausted }}` is true. Connect retry computation code here.
   - Add a **Wait** node (`n8n-nodes-base.wait`) configured for dynamic wait time using `{{ $json.backoffSeconds }}`. Name it `Wait - Exponential Backoff`. Connect the false output of `IF - Attempts Exhausted` here, and loop its output back into `Merge - Subscriber Results`.
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `HTTP Request - Write to Dead Letter Queue` pointing to `{{ $json.dlqUrl }}` with retry configuration. Connect the true output of `IF - Attempts Exhausted` here.
   - Add a **Slack** node (`n8n-nodes-base.slack`) configured with operation `postMessage` pointing to the configured DLQ channel. Name it `Slack - DLQ Alert`. Connect `HTTP Request - Write to Dead Letter Queue` here. Configure valid Slack credentials in n8n.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Event-Driven Architecture (Advanced) Pub/Sub Bus Pattern | Reusable n8n pattern implementing pub/sub, event sourcing, exponential backoff retries, and dead-letter queueing without external brokers. |
| Production Setup Requirement | Ensure external event store, subscriber services, and DLQ endpoints are reachable and configured with valid authentication before activating the workflow. |