Queue consent-checked video background-removal jobs with an HTTP API QA manifest

https://n8nworkflows.xyz/workflows/queue-consent-checked-video-background-removal-jobs-with-an-http-api-qa-manifest-19784


# Queue consent-checked video background-removal jobs with an HTTP API QA manifest

### 1. Workflow Overview

This workflow automates the queueing, submission, bounded polling, and quality assurance (QA) evaluation of up to five video background-removal jobs. Its primary purpose is to safely interface with an authorized video processing provider API (or run entirely offline in a demo mode) while enforcing strict consent, rights verification, and parameter validation prior to execution. 

The logical blocks comprise:
- **1.1 Initialization and Configuration:** Manual triggering and loading of the environment configuration, job payloads, and processing modes.
- **1.2 Validation and Payload Construction:** Comprehensive programmatic validation of rights, URLs, durations, and output settings, followed by the generation of normalized provider requests with idempotency keys.
- **1.3 Routing and Execution:** Branching logic to either execute an offline demo simulation or transmit live HTTP POST requests with header-based authentication, followed by response classification.
- **1.4 Polling Loop and Status Tracking:** Bounded status checking via time-delayed GET requests and attempt-capping logic for jobs in a queued state.
- **1.5 QA Evaluation and Manifest Generation:** Metadata inspection to assess transparency/alpha readiness and audio preservation, culminating in an auditable delivery manifest.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Initialization and Configuration
**Overview:** This block initializes the workflow manually and defines the operational context, provider base URL, safety flags, and target video jobs.

**Nodes Involved:**
- `Start`
- `Configuration`

**Node Details:**
- **Start**
  - *Type & Role:* `n8n-nodes-base.manualTrigger` (Version 1). Initiates manual execution of the workflow.
  - *Configuration:* Default parameters (empty).
  - *Inputs / Outputs:* Inputs: None | Outputs: Triggers the Configuration node.
  - *Edge Cases:* None.
- **Configuration**
  - *Type & Role:* `n8n-nodes-base.set` (Version 3.4). Sets raw JSON workflow configuration parameters.
  - *Configuration:* Configured in `raw` mode with a JSON payload containing `mode` ('demo' or 'live'), `providerBaseUrl`, `allowPaidProcessing` (boolean), and an array of `jobs` (1 to 5 items with parameters such as `jobId`, `videoUrl`, `rightsConfirmed`, `rightsReference`, `durationSeconds`, `background`, `edgeMode`, and `keepAudio`).
  - *Inputs / Outputs:* Inputs: `Start` | Outputs: `Validate consent and output settings`.
  - *Edge Cases:* Incorrect JSON syntax or missing keys will cause downstream validation failures.

---

#### Block 1.2: Validation and Payload Construction
**Overview:** This block validates the configuration parameters and individual job rules before converting them into structured provider-ready requests.

**Nodes Involved:**
- `Validate consent and output settings`
- `Build provider requests`

**Node Details:**
- **Validate consent and output settings**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Programmatically validates constraints on mode, payment flags, URLs, job counts, unique IDs, rights confirmation, video duration (0-600s), edge modes, and background color formats (`#RRGGBB`).
  - *Configuration:* JavaScript execution block that throws descriptive errors if validation criteria fail.
  - *Inputs / Outputs:* Inputs: `Configuration` | Outputs: `Build provider requests`.
  - *Edge Cases:* Throws runtime errors if validation checks fail (e.g., unconfirmed rights, invalid duration, or malformed hex colors).
- **Build provider requests**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Transforms validated job data into normalized provider payload structures.
  - *Configuration:* JavaScript execution block mapping each job object to include a stable `idempotencyKey` (`background-{jobId}`), routing mode, and nested request parameters.
  - *Inputs / Outputs:* Inputs: `Validate consent and output settings` | Outputs: `Use offline demo fixtures`.

---

#### Block 1.3: Routing and Execution
**Overview:** This block routes execution based on the operational mode, either simulating a completed demo response or dispatching an authenticated HTTP POST request to the provider API, then classifying the response.

**Nodes Involved:**
- `Use offline demo fixtures`
- `Simulate alpha-ready result`
- `Submit each approved job once`
- `Classify submit response`

**Node Details:**
- **Use offline demo fixtures**
  - *Type & Role:* `n8n-nodes-base.if` (Version 2.2). Evaluates whether the workflow is running in demo mode.
  - *Configuration:* Checks expression `={{ $json.mode === 'demo' }}`.
  - *Inputs / Outputs:* Inputs: `Build provider requests` | Outputs: True branch goes to `Simulate alpha-ready result`; False branch goes to `Submit each approved job once`.
- **Simulate alpha-ready result**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Generates mock completed job data with alpha metadata for offline testing.
  - *Configuration:* JavaScript mapping block setting `state` to `completed` and HTTP status to `200`.
  - *Inputs / Outputs:* Inputs: `Use offline demo fixtures` (True) | Outputs: `Check alpha and delivery metadata`.
- **Submit each approved job once**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Submits the background removal job to the provider endpoint.
  - *Configuration:* Method: `POST`, URL: `={{ $("Configuration").first().json.providerBaseUrl + "/jobs" }}`. Authentication: Generic Credential Type (`httpHeaderAuth`). Headers include `Idempotency-Key` and `Content-Type: application/json`. Error handling configured to continue regular output (`neverError: true`). Timeout: 60,000ms.
  - *Inputs / Outputs:* Inputs: `Use offline demo fixtures` (False) | Outputs: `Classify submit response`.
  - *Edge Cases:* Network timeouts, authentication failures, or provider rejection (HTTP 4xx/5xx).
- **Classify submit response**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Inspects HTTP response statuses and categorizes jobs into states such as `queued`, `rejected`, `provider_error`, or `invalid_response`.
  - *Configuration:* JavaScript evaluation block.
  - *Inputs / Outputs:* Inputs: `Submit each approved job once` | Outputs: `Queued for bounded polling`.

---

#### Block 1.4: Polling Loop and Status Tracking
**Overview:** This block manages asynchronous job completion tracking by conditionally waiting, querying the provider status endpoint, and incrementing attempt counters to prevent infinite loops.

**Nodes Involved:**
- `Queued for bounded polling`
- `Wait before status check`
- `Read job status`
- `Merge poll result and cap attempts`

**Node Details:**
- **Queued for bounded polling**
  - *Type & Role:* `n8n-nodes-base.if` (Version 2.2). Checks if a job's current state is `queued`.
  - *Configuration:* Evaluates expression `={{ $json.state === 'queued' }}`.
  - *Inputs / Outputs:* Inputs: `Classify submit response` | Outputs: True branch goes to `Wait before status check`; False branch bypasses polling and routes to `Check alpha and delivery metadata`.
- **Wait before status check**
  - *Type & Role:* `n8n-nodes-base.wait` (Version 1.1). Pauses execution to respect polling intervals.
  - *Configuration:* Unit: `seconds`, Amount: `15`.
  - *Inputs / Outputs:* Inputs: `Queued for bounded polling` (True) | Outputs: `Read job status`.
- **Read job status**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (Version 4.2). Polls the provider API for job updates.
  - *Configuration:* Method: `GET`, URL: `={{ $("Configuration").first().json.providerBaseUrl + "/jobs/" + $json.providerJobId }}`. Authentication: Generic Credential Type (`httpHeaderAuth`). Response format set to JSON with error continuity (`neverError: true`). Timeout: 30,000ms.
  - *Inputs / Outputs:* Inputs: `Wait before status check` | Outputs: `Merge poll result and cap attempts`.
  - *Edge Cases:* Transient network drops or API rate limiting during polling.
- **Merge poll result and cap attempts**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Updates job states based on poll results and enforces a maximum retry limit (sets state to `manual_followup` if attempts reach 4 while still processing).
  - *Configuration:* JavaScript execution block.
  - *Inputs / Outputs:* Inputs: `Read job status` | Outputs: `Check alpha and delivery metadata`.

---

#### Block 1.5: QA Evaluation and Manifest Generation
**Overview:** This block evaluates output metadata for transparency and audio preservation, then compiles a final QA manifest summarizing execution metrics.

**Nodes Involved:**
- `Check alpha and delivery metadata`
- `Create background QA manifest`

**Node Details:**
- **Check alpha and delivery metadata**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Analyzes response metadata to assign quality assurance status codes (`alpha_metadata_ready`, `verify_alpha_export`, etc.) and audio status.
  - *Configuration:* JavaScript code block.
  - *Inputs / Outputs:* Inputs: `Simulate alpha-ready result`, `Queued for bounded polling` (False), and `Merge poll result and cap attempts` | Outputs: `Create background QA manifest`.
- **Create background QA manifest**
  - *Type & Role:* `n8n-nodes-base.code` (Version 2). Formats the final delivery checklist and manifest object.
  - *Configuration:* JavaScript mapping block generating output properties (`jobId`, `providerJobId`, `state`, `attempts`, `resultUrl`, `qaStatus`, `audioStatus`, `rightsReference`, `createdAt`, `notice`).
  - *Inputs / Outputs:* Inputs: `Check alpha and delivery metadata` | Outputs: End of workflow.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Start** | `n8n-nodes-base.manualTrigger` | Initiates manual execution | None | Configuration | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Configuration** | `n8n-nodes-base.set` | Loads workflow config and job list | Start | Validate consent and output settings | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Validate consent and output settings** | `n8n-nodes-base.code` | Validates rules, rights, and settings | Configuration | Build provider requests | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Build provider requests** | `n8n-nodes-base.code` | Normalizes job payloads and idempotency keys | Validate consent and output settings | Use offline demo fixtures | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Use offline demo fixtures** | `n8n-nodes-base.if` | Branches between demo and live mode | Build provider requests | Simulate alpha-ready result, Submit each approved job once | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Simulate alpha-ready result** | `n8n-nodes-base.code` | Generates offline mock completion data | Use offline demo fixtures (True) | Check alpha and delivery metadata | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Submit each approved job once** | `n8n-nodes-base.httpRequest` | Submits jobs to the live provider API | Use offline demo fixtures (False) | Classify submit response | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Classify submit response** | `n8n-nodes-base.code` | Categorizes HTTP submission status | Submit each approved job once | Queued for bounded polling | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Queued for bounded polling** | `n8n-nodes-base.if` | Checks if job requires status polling | Classify submit response | Wait before status check, Check alpha and delivery metadata | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Wait before status check** | `n8n-nodes-base.wait` | Pauses execution for polling interval | Queued for bounded polling (True) | Read job status | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Read job status** | `n8n-nodes-base.httpRequest` | Polls live provider status endpoint | Wait before status check | Merge poll result and cap attempts | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Merge poll result and cap attempts** | `n8n-nodes-base.code` | Updates poll state and enforces retry limits | Read job status | Check alpha and delivery metadata | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Check alpha and delivery metadata** | `n8n-nodes-base.code` | Evaluates alpha transparency and audio status | Simulate alpha-ready result, Queued for bounded polling (False), Merge poll result and cap attempts | Create background QA manifest | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |
| **Create background QA manifest** | `n8n-nodes-base.code` | Produces final auditable delivery checklist | Check alpha and delivery metadata | None | Purpose<br>Setup<br>Validation and execution<br>QA and recovery<br>Attribution |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in n8n:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Start`. Leave default settings.

2. **Add Configuration:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`). Name it `Configuration`.
   - Set **Mode** to `Raw`.
   - Insert the configuration JSON containing `mode`, `providerBaseUrl`, `allowPaidProcessing`, and the `jobs` array into the JSON output parameter.
   - *Connection:* Connect `Start` (main) to `Configuration` (main).

3. **Add Validation Code:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate consent and output settings`.
   - Paste the validation JavaScript code snippet that checks job limits, permissions, HTTPS URLs, and color codes.
   - *Connection:* Connect `Configuration` (main) to `Validate consent and output settings` (main).

4. **Build Provider Requests:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Build provider requests`.
   - Paste the JavaScript snippet that iterates through valid jobs to generate idempotency keys and structured request payloads.
   - *Connection:* Connect `Validate consent and output settings` (main) to `Build provider requests` (main).

5. **Configure Demo Branching (IF):**
   - Add an **If** node (`n8n-nodes-base.if`). Name it `Use offline demo fixtures`.
   - Set condition expression: `={{ $json.mode === 'demo' }}` (Boolean equals true).
   - *Connection:* Connect `Build provider requests` (main) to `Use offline demo fixtures` (main).

6. **Add Offline Simulation:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Simulate alpha-ready result`.
   - Paste the JavaScript fixture simulation code.
   - *Connection:* Connect True output of `Use offline demo fixtures` to `Simulate alpha-ready result` (main).

7. **Add Live Submission HTTP Request:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Submit each approved job once`.
   - Set **Method** to `POST`.
   - Set **URL** to `={{ $("Configuration").first().json.providerBaseUrl + "/jobs" }}`.
   - Configure **Authentication** as Generic Credential Type -> `httpHeaderAuth`.
   - Add Header parameters: `Idempotency-Key` (`={{ $json.idempotencyKey }}`) and `Content-Type` (`application/json`).
   - Set Body parameter to JSON stringified request (`={{ JSON.stringify($json.request) }}`).
   - Under Options, enable **Never Error** (`neverError: true`), full response, and response format JSON. Set timeout to `60000`.
   - *Connection:* Connect False output of `Use offline demo fixtures` to `Submit each approved job once` (main).

8. **Classify Submit Response:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Classify submit response`.
   - Paste the classification JavaScript snippet.
   - *Connection:* Connect `Submit each approved job once` (main) to `Classify submit response` (main).

9. **Add Polling Evaluation (IF):**
   - Add an **If** node (`n8n-nodes-base.if`). Name it `Queued for bounded polling`.
   - Set condition expression: `={{ $json.state === 'queued' }}` (Boolean equals true).
   - *Connection:* Connect `Classify submit response` (main) to `Queued for bounded polling` (main).

10. **Add Polling Wait & Request:**
    - Add a **Wait** node (`n8n-nodes-base.wait`). Name it `Wait before status check`. Set unit to `seconds` and amount to `15`.
    - Connect True output of `Queued for bounded polling` to `Wait before status check` (main).
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Read job status`.
    - Set **Method** to `GET`. Set **URL** to `={{ $("Configuration").first().json.providerBaseUrl + "/jobs/" + $json.providerJobId }}`.
    - Configure **Authentication** as Generic Credential Type -> `httpHeaderAuth`. Enable `neverError: true`. Set timeout to `30000`.
    - *Connection:* Connect `Wait before status check` (main) to `Read job status` (main).

11. **Merge Poll Results:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Merge poll result and cap attempts`.
    - Paste the poll state-merging and attempt-capping JavaScript snippet.
    - *Connection:* Connect `Read job status` (main) to `Merge poll result and cap attempts` (main).

12. **Evaluate Metadata and Generate QA Manifest:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Check alpha and delivery metadata`. Paste the evaluation script.
    - *Connections:* Connect `Simulate alpha-ready result` (main), False output of `Queued for bounded polling`, and `Merge poll result and cap attempts` (main) all to `Check alpha and delivery metadata` (main).
    - Add a final **Code** node (`n8n-nodes-base.code`). Name it `Create background QA manifest`. Paste the manifest generation script.
    - *Connection:* Connect `Check alpha and delivery metadata` (main) to `Create background QA manifest` (main).

13. **Credentials Setup:**
    - Create an n8n HTTP Header Auth credential containing your provider API key/token and select it on both `Submit each approved job once` and `Read job status` nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by Shisan Hua, who works on Video Background Remover | [Video Background Remover Website](https://www.videobgremover.org/) |