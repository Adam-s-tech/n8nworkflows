Secure inbound webhooks with HMAC, rate limits, audit logs, and Slack alerts

https://n8nworkflows.xyz/workflows/secure-inbound-webhooks-with-hmac--rate-limits--audit-logs--and-slack-alerts-20060


# Secure inbound webhooks with HMAC, rate limits, audit logs, and Slack alerts

### 1. Workflow Overview

This workflow acts as an advanced, reusable security layer positioned directly in front of an inbound n8n webhook. Its primary purpose is to protect downstream business logic and systems from malicious payloads, abuse, unauthorized traffic, and replay attacks. 

The workflow is organized into three distinct logical blocks:

- **1.1 Ingress & Rate Limiting:** Receives incoming POST requests with raw body capture enabled, extracts metadata (such as source IP via `X-Forwarded-For` and timestamps), and enforces a per-IP sliding-window rate limit using static data. Over-limit traffic immediately triggers an HTTP 429 response to prevent volumetric exhaustion.
- **1.2 Signature, Replay & Schema Validation:** Performs timing-safe cryptographic verification of the inbound HMAC-SHA256 signature against the raw payload. It checks timestamps for allowed clock skew, prevents replay attacks by tracking seen nonces, validates structural schema requirements and payload size limits, and optionally verifies source IPs against a CIDR allow-list.
- **1.3 Decision, Audit Log & Alerting:** Consolidates all validation checks into a single pass/fail verdict and writes an audit log entry via an HTTP endpoint. Requests passing all gates proceed to downstream business logic and an HTTP 200 response. Rejected requests update a failure streak counter per IP; if failures exceed a configured threshold, an alert is posted to Slack before returning the corresponding error status code.

---

### 2. Block-by-Block Analysis

#### 2.1 Ingress & Rate Limiting
This block captures the raw HTTP request, normalizes identifying metadata, and filters out volumetric abuse before executing expensive cryptographic operations.

- **Nodes Involved:** `Webhook - Inbound`, `Set - Security Config`, `Code 1 - Extract Request Metadata`, `Code 2 - Rate Limiter (Sliding Window)`, `IF - Rate Limit Exceeded`, `Respond - Rate Limited (429)`.

##### Node Details:
- **Webhook - Inbound**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` — Receives incoming HTTP POST requests.
  - **Configuration Choices:** Configured with path `secure-webhook`, method `POST`, response mode set to use a response node, and `rawBody: true` enabled to capture exact byte streams for HMAC verification.
  - **Inputs / Outputs:** Input: None (Trigger). Output: Connected to `Set - Security Config`.
  - **Edge Cases / Failures:** Network timeouts, missing payload bodies, or malformed multipart data.
- **Set - Security Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Defines global security parameters, secrets, and threshold configurations.
  - **Configuration Choices:** Sets HMAC secrets, signature headers (`x-signature-256`), prefixes (`sha256=`), timestamp headers, clock skew limits (300s), rate limit windows (60s / 60 requests), max payload size (200KB), required fields (`event,data`), failure thresholds, Slack channels, and audit log URLs.
  - **Inputs / Outputs:** Input: `Webhook - Inbound`. Output: Connected to `Code 1 - Extract Request Metadata`.
  - **Edge Cases / Failures:** Plaintext secrets in production (should be replaced with environment variables or credentials).
- **Code 1 - Extract Request Metadata**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node extracting and normalizing request properties.
  - **Configuration Choices:** Resolves source IP using `X-Forwarded-For` or connection fallback, extracts headers, and captures ISO/epoch timestamps.
  - **Key Expressions:** References `$('Set - Security Config')` and `$('Webhook - Inbound')`.
  - **Inputs / Outputs:** Input: `Set - Security Config`. Output: Connected to `Code 2 - Rate Limiter (Sliding Window)`.
- **Code 2 - Rate Limiter (Sliding Window)**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node tracking request volume per source IP using workflow static data.
  - **Configuration Choices:** Maintains sliding window buckets per IP in workflow static data with automatic stale bucket pruning.
  - **Inputs / Outputs:** Input: `Code 1 - Extract Request Metadata`. Output: Connected to `IF - Rate Limit Exceeded`.
  - **Edge Cases / Failures:** Workflow static data is local to a single instance and is not distributed-safe across multiple n8n worker nodes at scale.
- **IF - Rate Limit Exceeded**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional router checking if `rateLimited` is true.
  - **Configuration Choices:** Boolean evaluation of `{{ $json.rateLimited }}`.
  - **Inputs / Outputs:** Input: `Code 2 - Rate Limiter (Sliding Window)`. Outputs: True branch connects to `Respond - Rate Limited (429)`, False branch connects to `Code 3 - Verify HMAC Signature`.
- **Respond - Rate Limited (429)**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` — Terminates execution for rate-limited requests.
  - **Configuration Choices:** Responds with HTTP status code `429` and JSON payload indicating `RATE_LIMITED`.
  - **Inputs / Outputs:** Input: `IF - Rate Limit Exceeded` (True branch). Output: None (Terminal).

---

#### 2.2 Signature, Replay & Schema Validation
This block validates the authenticity, integrity, chronological freshness, uniqueness, and structural compliance of the inbound request.

- **Nodes Involved:** `Code 3 - Verify HMAC Signature`, `Code 4 - Replay & Timestamp Guard`, `Code 5 - Validate Payload Schema`, `Code 6 - IP Allow-List Check`.

##### Node Details:
- **Code 3 - Verify HMAC Signature**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Performs cryptographic HMAC verification.
  - **Configuration Choices:** Uses Node.js `crypto` module to compute HMAC-SHA256 over raw body bytes, using `crypto.timingSafeEqual` for constant-time comparisons.
  - **Inputs / Outputs:** Input: `IF - Rate Limit Exceeded` (False branch). Output: Connected to `Code 4 - Replay & Timestamp Guard`.
  - **Edge Cases / Failures:** Verification failure when secrets mismatch or signature prefixes are improperly stripped.
- **Code 4 - Replay & Timestamp Guard**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Enforces timestamp skew checks and prevents replay attacks via nonce tracking.
  - **Configuration Choices:** Validates incoming epoch timestamps against `allowedClockSkewSeconds` and records hashed signature-timestamp nonces in static data.
  - **Inputs / Outputs:** Input: `Code 3 - Verify HMAC Signature`. Output: Connected to `Code 5 - Validate Payload Schema`.
- **Code 5 - Validate Payload Schema**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Checks structural integrity, required fields, and payload size bounds.
  - **Configuration Choices:** Evaluates parsed body elements against `requiredFields` and verifies buffer byte length against `maxPayloadBytes`.
  - **Inputs / Outputs:** Input: `Code 4 - Replay & Timestamp Guard`. Output: Connected to `Code 6 - IP Allow-List Check`.
- **Code 6 - IP Allow-List Check**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Optional CIDR-aware source IP validator.
  - **Configuration Choices:** Parses CIDR notations and verifies if the source IP falls within allowed ranges. Bypasses check if the configuration string is empty.
  - **Inputs / Outputs:** Input: `Code 5 - Validate Payload Schema`. Output: Connected to `Code 7 - Aggregate Security Verdict`.

---

#### 2.3 Decision, Audit Log & Alerting
This block aggregates all gate assessments, records telemetry, executes business logic for valid requests, and tracks security incidents for rejected requests.

- **Nodes Involved:** `Code 7 - Aggregate Security Verdict`, `HTTP Request - Write Audit Log`, `IF - Security Gate Passed`, `Business Logic - Process Payload`, `Respond - Accepted (200)`, `Code 8 - Track Failure Streak`, `IF - Alert Threshold Exceeded`, `Slack - Security Alert`, `Respond - Rejected`.

##### Node Details:
- **Code 7 - Aggregate Security Verdict**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Compiles evaluation results into a single verdict, reason code, and HTTP status.
  - **Configuration Choices:** Assigns specific status codes (200, 400, 401, 403, 413, 429) and reason codes (`OK`, `BAD_SIGNATURE`, `REPLAY_DETECTED`, etc.).
  - **Inputs / Outputs:** Input: `Code 6 - IP Allow-List Check`. Output: Connected to `HTTP Request - Write Audit Log`.
- **HTTP Request - Write Audit Log**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Transmits structured audit telemetry to an external endpoint.
  - **Configuration Choices:** Sends POST requests containing request metadata, verdicts, truncated body hashes, and user agents. Configured with error continuity (`continueErrorOutput`) and retry attempts.
  - **Inputs / Outputs:** Input: `Code 7 - Aggregate Security Verdict`. Output: Connected to `IF - Security Gate Passed`.
  - **Edge Cases / Failures:** Audit sink downtime or network timeout (mitigated by error output continuation).
- **IF - Security Gate Passed**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Routes traffic based on overall security success.
  - **Configuration Choices:** Evaluates `{{ $('Code 7 - Aggregate Security Verdict').first().json.securityPassed }}`.
  - **Inputs / Outputs:** Input: `HTTP Request - Write Audit Log`. Outputs: True branch connects to `Business Logic - Process Payload`, False branch connects to `Code 8 - Track Failure Streak`.
- **Business Logic - Process Payload**
  - **Type & Technical Role:** `n8n-nodes-base.noOp` — Placeholder node representing downstream application logic.
  - **Configuration Choices:** None (No-Op placeholder).
  - **Inputs / Outputs:** Input: `IF - Security Gate Passed` (True). Output: Connected to `Respond - Accepted (200)`.
- **Respond - Accepted (200)**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` — Responds to successfully processed webhook requests.
  - **Configuration Choices:** Returns HTTP status `200` with confirmation JSON payload.
  - **Inputs / Outputs:** Input: `Business Logic - Process Payload`. Output: None (Terminal).
- **Code 8 - Track Failure Streak**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Tracks rolling failure counts per source IP to identify enumeration or brute-force behavior.
  - **Configuration Choices:** Maintains sliding window failure counters in static data against `failureWindowSeconds` and `failureAlertThreshold`.
  - **Inputs / Outputs:** Input: `IF - Security Gate Passed` (False). Output: Connected to `IF - Alert Threshold Exceeded`.
- **IF - Alert Threshold Exceeded**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Determines whether failure counts surpass the alert threshold.
  - **Configuration Choices:** Evaluates boolean expression `{{ $json.alertThresholdExceeded }}`.
  - **Inputs / Outputs:** Input: `Code 8 - Track Failure Streak`. Outputs: True branch connects to `Slack - Security Alert`, False branch connects to `Respond - Rejected`.
- **Slack - Security Alert**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Broadcasts security incident notifications to a messaging channel.
  - **Configuration Choices:** Posts message payloads to configured security alert channels when attack thresholds are met.
  - **Inputs / Outputs:** Input: `IF - Alert Threshold Exceeded` (True). Output: Connected to `Respond - Rejected`.
  - **Credentials Required:** Slack OAuth2 / Bot Token.
- **Respond - Rejected**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` — Returns appropriate error status codes for rejected requests.
  - **Configuration Choices:** Dynamically pulls response codes from `{{ $('Code 7 - Aggregate Security Verdict').first().json.httpStatus }}` and formats rejection reasons.
  - **Inputs / Outputs:** Inputs: `IF - Alert Threshold Exceeded` (False) or `Slack - Security Alert`. Output: None (Terminal).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Inbound | n8n-nodes-base.webhook | Receives inbound POST requests with raw body capture. | None (Trigger) | Set - Security Config | ## 1. Ingress & Rate Limiting<br><br>Webhook - Inbound receives the raw POST (raw body capture enabled so the HMAC signature can be verified against the exact bytes sent). |
| Set - Security Config | n8n-nodes-base.set | Defines global security parameters, secrets, and limits. | Webhook - Inbound | Code 1 - Extract Request Metadata | ## Webhook Security Layer (Advanced)<br><br>A reusable, hardened front door for any inbound webhook. Every request passes through, in order:... |
| Code 1 - Extract Request Metadata | n8n-nodes-base.code | Normalizes source IP, headers, raw body, and timestamps. | Set - Security Config | Code 2 - Rate Limiter (Sliding Window) | ## 1. Ingress & Rate Limiting<br><br>...Code 1 - Extract Request Metadata pulls source IP (respecting X-Forwarded-For), headers, raw body, and a request timestamp into a clean shape used by every downstream gate. |
| Code 2 - Rate Limiter (Sliding Window) | n8n-nodes-base.code | Tracks request counts per source IP in sliding window. | Code 1 - Extract Request Metadata | IF - Rate Limit Exceeded | ## 1. Ingress & Rate Limiting<br><br>...Code 2 - Rate Limiter (Sliding Window) tracks request counts per source IP in workflow static data over a rolling window (default: 60 requests / 60s, configurable in Set - Security Config)... |
| IF - Rate Limit Exceeded | n8n-nodes-base.if | Routes traffic exceeding rate limits to HTTP 429 response. | Code 2 - Rate Limiter (Sliding Window) | Respond - Rate Limited (429), Code 3 - Verify HMAC Signature | ## 1. Ingress & Rate Limiting<br><br>...Requests over the limit are flagged rateLimited=true and routed straight to a 429 response — they never reach signature verification... |
| Code 3 - Verify HMAC Signature | n8n-nodes-base.code | Performs timing-safe HMAC-SHA256 signature validation. | IF - Rate Limit Exceeded | Code 4 - Replay & Timestamp Guard | ## 2. Signature, Replay & Schema Validation<br><br>Code 3 - Verify HMAC Signature recomputes an HMAC-SHA256 over the raw body using the shared secret and compares it to the incoming signature header using a timing-safe (constant-time) comparison... |
| Code 4 - Replay & Timestamp Guard | n8n-nodes-base.code | Rejects expired timestamps and duplicate nonce replays. | Code 3 - Verify HMAC Signature | Code 5 - Validate Payload Schema | ## 2. Signature, Replay & Schema Validation<br><br>...Code 4 - Replay & Timestamp Guard rejects requests whose timestamp is outside the allowed clock skew (default ±300s), and rejects any (signature, timestamp) pair already seen in the static-data nonce cache... |
| Code 5 - Validate Payload Schema | n8n-nodes-base.code | Validates required fields, types, and max payload size. | Code 4 - Replay & Timestamp Guard | Code 6 - IP Allow-List Check | ## 2. Signature, Replay & Schema Validation<br><br>...Code 5 - Validate Payload Schema checks required fields, types, and a max payload size, independent of business logic, so malformed bodies are rejected before anything downstream sees them. |
| Code 6 - IP Allow-List Check | n8n-nodes-base.code | Enforces optional CIDR-aware source IP allow-listing. | Code 5 - Validate Payload Schema | Code 7 - Aggregate Security Verdict | ## 2. Signature, Replay & Schema Validation<br><br>...Code 6 - IP Allow-List Check (optional gate, CIDR-aware) — only enforced if an allow-list is configured; otherwise passes through. |
| Code 7 - Aggregate Security Verdict | n8n-nodes-base.code | Consolidates all check outcomes into a final pass/fail verdict. | Code 6 - IP Allow-List Check | HTTP Request - Write Audit Log | ## 3. Decision, Audit Log & Alerting<br><br>Code 7 - Aggregate Security Verdict merges every gate's result into one pass/fail decision with a specific reason code... |
| HTTP Request - Write Audit Log | n8n-nodes-base.httpRequest | Logs request telemetry and truncated body hash to audit sink. | Code 7 - Aggregate Security Verdict | IF - Security Gate Passed | ## 3. Decision, Audit Log & Alerting<br><br>...Every request — accepted or rejected — is logged via HTTP Request - Write Audit Log (swap for Postgres/Airtable node) with IP, verdict, reason, timestamp, and a truncated body hash... |
| IF - Security Gate Passed | n8n-nodes-base.if | Branches execution based on overall security verdict. | HTTP Request - Write Audit Log | Business Logic - Process Payload, Code 8 - Track Failure Streak | ## 3. Decision, Audit Log & Alerting<br><br>...IF - Security Gate Passed branches:<br>- **Pass** → Business Logic - Process Payload (your real workflow) → Respond 200... |
| Business Logic - Process Payload | n8n-nodes-base.noOp | Placeholder for core downstream application logic. | IF - Security Gate Passed | Respond - Accepted (200) | ## 3. Decision, Audit Log & Alerting<br><br>...IF - Security Gate Passed branches:<br>- **Pass** → Business Logic - Process Payload (your real workflow) → Respond 200... |
| Respond - Accepted (200) | n8n-nodes-base.respondToWebhook | Returns HTTP 200 response for accepted requests. | Business Logic - Process Payload | None (Terminal) | ## 3. Decision, Audit Log & Alerting<br><br>...- **Pass** → Business Logic - Process Payload (your real workflow) → Respond 200... |
| Code 8 - Track Failure Streak | n8n-nodes-base.code | Increments rolling failure counts per source IP. | IF - Security Gate Passed | IF - Alert Threshold Exceeded | ## 3. Decision, Audit Log & Alerting<br><br>...- **Fail** → Code 8 - Track Failure Streak increments a per-IP failure counter; IF - Alert Threshold Exceeded fires Slack - Security Alert when an IP racks up repeated failures... |
| IF - Alert Threshold Exceeded | n8n-nodes-base.if | Evaluates whether IP failure count exceeds alert threshold. | Code 8 - Track Failure Streak | Slack - Security Alert, Respond - Rejected | ## 3. Decision, Audit Log & Alerting<br><br>...- **Fail** → Code 8 - Track Failure Streak increments a per-IP failure counter; IF - Alert Threshold Exceeded fires Slack - Security Alert when an IP racks up repeated failures (possible attack)... |
| Slack - Security Alert | n8n-nodes-base.slack | Posts security incident alerts to a designated Slack channel. | IF - Alert Threshold Exceeded | Respond - Rejected | ## 3. Decision, Audit Log & Alerting<br><br>...fires Slack - Security Alert when an IP racks up repeated failures (possible attack), then all failures return the correct error status via Respond - Rejected. |
| Respond - Rejected | n8n-nodes-base.respondToWebhook | Returns appropriate HTTP error status for rejected requests. | IF - Alert Threshold Exceeded, Slack - Security Alert | None (Terminal) | ## 3. Decision, Audit Log & Alerting<br><br>...then all failures return the correct error status via Respond - Rejected. |
| Respond - Rate Limited (429) | n8n-nodes-base.respondToWebhook | Returns HTTP 429 response for rate-limited traffic. | IF - Rate Limit Exceeded | None (Terminal) | ## 1. Ingress & Rate Limiting<br><br>...Requests over the limit are flagged rateLimited=true and routed straight to a 429 response... |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in an n8n environment:

1. **Create Webhook Trigger:**
   - Add a **Webhook** node named `Webhook - Inbound`. Set HTTP Method to `POST`, Path to `secure-webhook`, and Response Mode to `Response Node`. Under options, enable `Raw Body` and `Ignore Bots`.
2. **Configure Security Parameters:**
   - Add a **Set** node named `Set - Security Config`. Configure string assignments for `hmacSecret`, `signatureHeaderName` (`x-signature-256`), `signaturePrefix` (`sha256=`), `timestampHeaderName` (`x-request-timestamp`), `requiredFields` (`event,data`), `ipAllowList`, `slackSecurityChannel` (`#security-alerts`), and `auditLogUrl`. Configure numeric values for `allowedClockSkewSeconds` (300), `rateLimitWindowSeconds` (60), `rateLimitMaxRequests` (60), `maxPayloadBytes` (200000), `failureAlertThreshold` (5), and `failureWindowSeconds` (300). Connect `Webhook - Inbound` to this node.
3. **Extract Request Metadata:**
   - Add a **Code** node named `Code 1 - Extract Request Metadata`. Populate the JavaScript block to extract headers, normalize source IP (checking `X-Forwarded-For`), grab raw and parsed bodies, and record ISO/epoch timestamps. Connect `Set - Security Config` to this node.
4. **Implement Rate Limiting:**
   - Add a **Code** node named `Code 2 - Rate Limiter (Sliding Window)`. Implement sliding-window tracking utilizing workflow static data (`$getWorkflowStaticData('node')`). Connect `Code 1` to this node.
   - Add an **IF** node named `IF - Rate Limit Exceeded`. Set condition to evaluate `{{ $json.rateLimited }}` equals `true`. Connect `Code 2` to this node.
   - Add a **Respond to Webhook** node named `Respond - Rate Limited (429)`. Set Response Code to `429` and respond with JSON containing `RATE_LIMITED`. Connect the True output of `IF - Rate Limit Exceeded` to this node.
5. **Implement Cryptographic & Replay Checks:**
   - Add a **Code** node named `Code 3 - Verify HMAC Signature`. Implement HMAC-SHA256 signature verification over raw body bytes using `crypto.timingSafeEqual`. Connect the False output of `IF - Rate Limit Exceeded` to this node.
   - Add a **Code** node named `Code 4 - Replay & Timestamp Guard`. Implement clock skew validation and nonce duplicate checks using static data. Connect `Code 3` to this node.
   - Add a **Code** node named `Code 5 - Validate Payload Schema`. Validate required fields and payload size constraints against `maxPayloadBytes`. Connect `Code 4` to this node.
   - Add a **Code** node named `Code 6 - IP Allow-List Check`. Implement CIDR allow-list verification. Connect `Code 5` to this node.
6. **Aggregate Security Verdict & Audit Logging:**
   - Add a **Code** node named `Code 7 - Aggregate Security Verdict` to compile overall status, reason codes, and HTTP error mapping. Connect `Code 6` to this node.
   - Add an **HTTP Request** node named `HTTP Request - Write Audit Log`. Set method to `POST`, URL to `{{ $json.auditLogUrl }}`, and construct the JSON body containing telemetry data and a truncated SHA-256 body hash. Enable error continuation on fail and retries. Connect `Code 7` to this node.
7. **Branch Valid vs. Invalid Requests:**
   - Add an **IF** node named `IF - Security Gate Passed`. Evaluate `{{ $('Code 7 - Aggregate Security Verdict').first().json.securityPassed }}` equals `true`. Connect `HTTP Request - Write Audit Log` to this node.
8. **Configure Success Path:**
   - Add a **No-Op** node named `Business Logic - Process Payload` representing your downstream workflow logic. Connect the True output of `IF - Security Gate Passed` to this node.
   - Add a **Respond to Webhook** node named `Respond - Accepted (200)`. Set Response Code to `200` and return confirmation JSON. Connect `Business Logic - Process Payload` to this node.
9. **Configure Failure Tracking & Alerting:**
   - Add a **Code** node named `Code 8 - Track Failure Streak` to increment rolling failure counters per IP in static data. Connect the False output of `IF - Security Gate Passed` to this node.
   - Add an **IF** node named `IF - Alert Threshold Exceeded`. Evaluate `{{ $json.alertThresholdExceeded }}` equals `true`. Connect `Code 8` to this node.
   - Add a **Slack** node named `Slack - Security Alert`. Configure resource and operation to post messages to the target security channel using valid Slack credentials. Connect the True output of `IF - Alert Threshold Exceeded` to this node.
   - Add a **Respond to Webhook** node named `Respond - Rejected`. Set Response Code to expression `{{ $('Code 7 - Aggregate Security Verdict').first().json.httpStatus }}` and respond with rejection details. Connect both the False output of `IF - Alert Threshold Exceeded` and `Slack - Security Alert` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Static data is used for portability. For high-throughput production use, swap static data rate-limiting and nonce caches for Redis via REST proxy or native nodes. | Workflow architecture note |
| Replace placeholder secrets in `Set - Security Config` with secure n8n credentials or environment variables prior to production deployment. | Credential security practice |