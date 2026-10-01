Queue rights-verified video watermark removal jobs with HTTP API polling

https://n8nworkflows.xyz/workflows/queue-rights-verified-video-watermark-removal-jobs-with-http-api-polling-19783


# Queue rights-verified video watermark removal jobs with HTTP API polling

### 1. Workflow Overview

This workflow automates the validation, submission, and status tracking of rights-verified video watermark-removal jobs. Its primary purpose is to safely interface with an authorized watermark-cleanup API (or run locally using offline mock data) while ensuring strict compliance checks, idempotency, and cost safeguards before initiating paid processing.

The logical execution follows these functional blocks:
- **1.1 Initialization and Configuration:** Manually triggers the execution and injects the base operational payload, defining execution mode (`demo` or `live`), provider URLs, safety flags, and the batch of video jobs to process.
- **1.2 Validation and Payload Preparation:** Validates job configurations (such as rights verification, HTTPS boundaries, and coordinate ratios) and normalizes payloads with idempotency keys.
- **1.3 Execution Branching (Demo vs. Live):** Determines whether to bypass external network calls via offline mock fixtures or dispatch live requests to an external processing provider.
- **1.4 Submission and Status Polling:** Submits jobs to the provider via POST requests, handles response classification, and executes a bounded polling loop (with delays) to track asynchronous job completion.
- **1.5 Audit and Reporting:** Consolidates final job states, attempts, provider IDs, and result URLs into a structured audit manifest.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration
- **Overview:** Initializes the workflow execution manually and supplies the base configuration dataset containing job parameters and operational modes.
- **Nodes Involved:** `Start`, `Configuration`

##### Node Details: `Start`
- **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Acts as the manual entry point for workflow execution.
- **Configuration Choices:** Uses default parameters.
- **Expressions/Variables:** None.
- **Connections:** Output connects to `Configuration`.
- **Version Requirements:** v1.
- **Edge Cases / Potential Failures:** None.

##### Node Details: `Configuration`
- **Type & Technical Role:** `n8n-nodes-base.set` — Injects static JSON data containing the execution mode, provider base URL, safety flags, and an array of target jobs.
- **Configuration Choices:** Configured with raw JSON mode containing parameters (`mode`, `providerBaseUrl`, `allowPaidProcessing`, and `jobs`).
- **Expressions/Variables:** None.
- **Connections:** Input from `Start`; output connects to `Validate rights and removal regions`.
- **Version Requirements:** v3.4.
- **Edge Cases / Potential Failures:** Malformed JSON payload structure will cause downstream validation failures.

---

#### 2.2 Validation and Payload Preparation
- **Overview:** Enforces strict validation rules on the configuration and job definitions, ensuring lawful operation parameters and correct geographic coordinate ratios, then transforms the data into normalized provider requests.
- **Nodes Involved:** `Validate rights and removal regions`, `Build provider requests`

##### Node Details: `Validate rights and removal regions`
- **Type & Technical Role:** `n8n-nodes-base.code` — Executes JavaScript validation logic to inspect configuration parameters and individual job attributes.
- **Configuration Choices:** Custom inline JavaScript checking for mode validity, paid processing safety flags, HTTPS URL schemes, job array limits (1–5 jobs), alphanumeric unique job IDs, rights confirmation flags, and boundary checks for rectangular removal regions (0–1 coordinate ratios, maximum 600-second duration).
- **Expressions/Variables:** Accesses `$input.first().json`.
- **Connections:** Input from `Configuration`; output connects to `Build provider requests`.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** Throws explicit JavaScript errors if validation rules fail (e.g., missing rights references, invalid region coordinates exceeding frame bounds, or unapproved paid processing).

##### Node Details: `Build provider requests`
- **Type & Technical Role:** `n8n-nodes-base.code` — Maps validated job items into structured provider-compatible request payloads.
- **Configuration Choices:** Custom JavaScript mapping function generating an idempotency key (`watermark-<jobId>`), metadata references, and request options.
- **Expressions/Variables:** Accesses `$input.first().json`.
- **Connections:** Input from `Validate rights and removal regions`; output connects to `Use offline demo fixtures`.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** Failure if upstream data arrays are missing.

---

#### 2.3 Execution Branching
- **Overview:** Evaluates the execution mode (`demo` vs. `live`) to route jobs either through offline mock simulation or live API calls.
- **Nodes Involved:** `Use offline demo fixtures`, `Simulate accepted and completed job`

##### Node Details: `Use offline demo fixtures`
- **Type & Technical Role:** `n8n-nodes-base.if` — Conditional router checking whether the execution mode is set to demo.
- **Configuration Choices:** Evaluates condition `{{ $json.mode === 'demo' }}` returning true or false paths.
- **Expressions/Variables:** `{{ $json.mode === 'demo' }}`
- **Connections:** Input from `Build provider requests`; True output connects to `Simulate accepted and completed job`, False output connects to `Submit each approved job once`.
- **Version Requirements:** v2.2.
- **Edge Cases / Potential Failures:** Unexpected mode string values default to the live processing path.

##### Node Details: `Simulate accepted and completed job`
- **Type & Technical Role:** `n8n-nodes-base.code` — Simulates a successful API response and completed processing state for offline testing.
- **Configuration Choices:** Custom JavaScript generating mock successful provider attributes (`providerJobId`, `state: 'completed'`, mock `resultUrl`).
- **Expressions/Variables:** Accesses `$input.all()`.
- **Connections:** Input from `Use offline demo fixtures` (True branch); output connects to `Create audit manifest`.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** None.

---

#### 2.4 Submission and Status Polling
- **Overview:** Submits jobs to the external provider via HTTP POST, classifies responses, and manages an iterative polling loop to check job completion status.
- **Nodes Involved:** `Submit each approved job once`, `Classify submit response`, `Queued for bounded polling`, `Wait before status check`, `Read job status`, `Merge poll result and cap attempts`

##### Node Details: `Submit each approved job once`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Dispatches job requests to the watermark removal API provider.
- **Configuration Choices:** Method: `POST`, URL constructed dynamically from configuration, includes HTTP Header Authentication, Idempotency-Key header, and JSON body payload. Configured with error handling (`onError: continueRegularOutput`, `neverError: true`) to capture non-2xx status codes.
- **Expressions/Variables:** `={{ $("Configuration").first().json.providerBaseUrl + "/jobs" }}`, `={{ JSON.stringify($json.request) }}`, `={{ $json.idempotencyKey }}`
- **Connections:** Input from `Use offline demo fixtures` (False branch); output connects to `Classify submit response`.
- **Version Requirements:** v4.2.
- **Edge Cases / Potential Failures:** Authentication failures, network timeouts (60s limit), or HTTP 4xx/5xx responses. Managed by response classification downstream.
- **Credentials:** Uses Generic Credential Type (`httpHeaderAuth`).

##### Node Details: `Classify submit response`
- **Type & Technical Role:** `n8n-nodes-base.code` — Analyzes HTTP submission status codes and body content to classify the job state.
- **Configuration Choices:** Custom JavaScript mapping HTTP status codes (200, 201, 202, 4xx, 5xx) into operational states (`queued`, `invalid_response`, `rejected`, `provider_error`).
- **Expressions/Variables:** Accesses `$input.all()` and `$('Build provider requests').all()`.
- **Connections:** Input from `Submit each approved job once`; output connects to `Queued for bounded polling`.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** Unexpected API response schemas default to unknown states with warning flags.

##### Node Details: `Queued for bounded polling`
- **Type & Technical Role:** `n8n-nodes-base.if` — Checks if a submitted job has successfully entered the `queued` state and requires status polling.
- **Configuration Choices:** Evaluates condition `{{ $json.state === 'queued' }}`.
- **Expressions/Variables:** `={{ $json.state === 'queued' }}`
- **Connections:** Input from `Classify submit response`; True output connects to `Wait before status check`, False output connects to `Create audit manifest`.
- **Version Requirements:** v2.2.
- **Edge Cases / Potential Failures:** None.

##### Node Details: `Wait before status check`
- **Type & Technical Role:** `n8n-nodes-base.wait` — Introduces a pause between polling attempts to prevent rate-limiting.
- **Configuration Choices:** Pauses execution for 15 seconds (`amount: 15`, `unit: "seconds"`).
- **Expressions/Variables:** None.
- **Connections:** Input from `Queued for bounded polling` (True branch); output connects to `Read job status`.
- **Version Requirements:** v1.1.
- **Edge Cases / Potential Failures:** None.

##### Node Details: `Read job status`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Polls the provider endpoint to retrieve current processing status.
- **Configuration Choices:** Method: `GET`, URL constructed dynamically using provider base URL and provider job ID. Configured with error tolerance (`neverError: true`) and HTTP Header Authentication.
- **Expressions/Variables:** `={{ $("Configuration").first().json.providerBaseUrl + "/jobs/" + $json.providerJobId }}`
- **Connections:** Input from `Wait before status check`; output connects to `Merge poll result and cap attempts`.
- **Version Requirements:** v4.2.
- **Edge Cases / Potential Failures:** API timeouts (30s limit) or network interruptions during polling.
- **Credentials:** Uses Generic Credential Type (`httpHeaderAuth`).

##### Node Details: `Merge poll result and cap attempts`
- **Type & Technical Role:** `n8n-nodes-base.code` — Evaluates polling results, updates attempt counters, caps polling attempts at a maximum threshold, and determines terminal statuses.
- **Configuration Choices:** Custom JavaScript tracking retry counts (forcing `manual_followup` if attempts reach 4 while still processing), parsing completion/failure states, and extracting result URLs.
- **Expressions/Variables:** Accesses `$input.all()` and `$('Classify submit response').all()`.
- **Connections:** Input from `Read job status`; output connects to `Create audit manifest`.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** Infinite polling loops prevented via hard attempt caps.

---

#### 2.5 Audit and Reporting
- **Overview:** Aggregates processed job metrics into a final structured audit manifest record.
- **Nodes Involved:** `Create audit manifest`

##### Node Details: `Create audit manifest`
- **Type & Technical Role:** `n8n-nodes-base.code` — Formats final job outcomes into an audit log payload.
- **Configuration Choices:** Custom JavaScript mapping job identifiers, provider IDs, terminal states, attempt counts, result URLs, timestamps, and compliance review notices.
- **Expressions/Variables:** Accesses `$input.all()`.
- **Connections:** Inputs from `Simulate accepted and completed job`, `Queued for bounded polling` (False branch), and `Merge poll result and cap attempts`; no downstream outputs.
- **Version Requirements:** v2.
- **Edge Cases / Potential Failures:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start | n8n-nodes-base.manualTrigger | Manual workflow entry point | None | Configuration | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Configuration | n8n-nodes-base.set | Defines operational configuration and job payloads | Start | Validate rights and removal regions | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Validate rights and removal regions | n8n-nodes-base.code | Validates configuration, permissions, and region coordinates | Configuration | Build provider requests | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Build provider requests | n8n-nodes-base.code | Normalizes payloads and generates idempotency keys | Validate rights and removal regions | Use offline demo fixtures | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Use offline demo fixtures | n8n-nodes-base.if | Routes execution based on demo vs. live mode | Build provider requests | Simulate accepted and completed job, Submit each approved job once | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Simulate accepted and completed job | n8n-nodes-base.code | Generates mock completion response in demo mode | Use offline demo fixtures | Create audit manifest | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Submit each approved job once | n8n-nodes-base.httpRequest | Submits jobs to the external API via HTTP POST | Use offline demo fixtures | Classify submit response | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Classify submit response | n8n-nodes-base.code | Classifies HTTP submission outcomes | Submit each approved job once | Queued for bounded polling | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Queued for bounded polling | n8n-nodes-base.if | Filters queued jobs for status polling | Classify submit response | Wait before status check, Create audit manifest | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Wait before status check | n8n-nodes-base.wait | Delays execution between polling requests | Queued for bounded polling | Read job status | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Read job status | n8n-nodes-base.httpRequest | Polls job processing status via HTTP GET | Wait before status check | Merge poll result and cap attempts | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Merge poll result and cap attempts | n8n-nodes-base.code | Evaluates poll responses and manages retry limits | Read job status | Create audit manifest | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials. The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |
| Create audit manifest | n8n-nodes-base.code | Compiles final audit report records | Queued for bounded polling, Simulate accepted and completed job, Merge poll result and cap attempts | None | ## Purpose<br>Queue up to five watermark-cleanup jobs only after ownership or editing permission has been recorded. Every job carries normalized frame regions, an idempotency key, a rights reference, bounded status polling and a final audit manifest. It is designed for lawful cleanup of the user's own or licensed footage, not removal of third-party ownership marks.<br><br>## Setup<br>The default demo is offline and produces a completed fixture without credentials or network calls. For live use, replace providerBaseUrl with your authorized processing API, configure n8n Header Auth on both HTTP nodes, review its request/response contract and set allowPaidProcessing to true. Keep secrets in n8n credentials.The expected API accepts POST /jobs and returns jobId, then GET /jobs/{jobId} with processing, completed or failed plus an optional resultUrl.<br><br>## Safety and recovery<br>Each job needs rightsConfirmed=true and a meaningful rightsReference. One to eight time-bounded rectangular regions use 0-1 frame ratios and are rejected if they cross frame boundaries. POST retries are disabled to avoid duplicate charges. Polling is capped; unknown or provider-error outcomes require checking the provider dashboard before retrying. The workflow never downloads, republishes or deletes media automatically.<br><br>## Output<br>The final node emits one audit record per job with provider ID, terminal state, attempts, result URL and rights evidence reference. Review cleaned footage frame-by-frame and confirm that licensed logos, credits or attribution required data required by contract remain intact before publishing.<br><br>## Attribution<br>Created by Shisan Hua, who works on [Video Watermark Remover](https://videowatermarkremover.org/). The template uses a generic authorized provider contract and does not claim a public Video Watermark Remover API. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node 1 (`Start`)**:
   - Type: `n8n-nodes-base.manualTrigger`
   - Configuration: Leave parameters empty.

2. **Create Node 2 (`Configuration`)**:
   - Type: `n8n-nodes-base.set`
   - Configuration: Set mode to `raw`, and input the configuration JSON structure defining `mode` (`demo`), `providerBaseUrl`, `allowPaidProcessing` (`false`), and the `jobs` array containing sample job properties (`jobId`, `videoUrl`, `rightsConfirmed`, `rightsReference`, `keepAudio`, and `regions`).

3. **Create Node 3 (`Validate rights and removal regions`)**:
   - Type: `n8n-nodes-base.code`
   - Configuration: Paste the JavaScript code snippet that enforces validation constraints on mode, payment flags, HTTPS URLs, job count limits (1–5), unique job ID patterns, rights evidence, and 0–1 frame coordinate ratios for removal regions.

4. **Create Node 4 (`Build provider requests`)**:
   - Type: `n8n-nodes-base.code`
   - Configuration: Paste the JavaScript mapping code to generate idempotency keys (`watermark-<jobId>`) and structured request objects per job.

5. **Create Node 5 (`Use offline demo fixtures`)**:
   - Type: `n8n-nodes-base.if`
   - Configuration: Add a condition where Left Value is `{{ $json.mode === 'demo' }}` and Operation is boolean `true`.

6. **Create Node 6 (`Simulate accepted and completed job`)**:
   - Type: `n8n-nodes-base.code`
   - Configuration: Paste the JavaScript code to generate mock completed job attributes (`providerJobId`, `state: 'completed'`, mock `resultUrl`).

7. **Create Node 7 (`Submit each approved job once`)**:
   - Type: `n8n-nodes-base.httpRequest`
   - Configuration:
     - Method: `POST`
     - URL: `={{ $("Configuration").first().json.providerBaseUrl + "/jobs" }}`
     - Specify Body: `json`
     - JSON Body: `={{ JSON.stringify($json.request) }}`
     - Send Headers: Enabled
     - Header Parameters: Add `Idempotency-Key` = `={{ $json.idempotencyKey }}` and `Content-Type` = `application/json`.
     - Options: Set Response Format to `json`, enable `Never Error`, and set Timeout to `60000`ms.
     - Authentication: Configure Generic Credential Type (`httpHeaderAuth`).

8. **Create Node 8 (`Classify submit response`)**:
   - Type: `n8n-nodes-base.code`
   - Configuration: Paste the JavaScript code to classify HTTP response codes into job states (`queued`, `rejected`, `provider_error`, etc.).

9. **Create Node 9 (`Queued for bounded polling`)**:
   - Type: `n8n-nodes-base.if`
   - Configuration: Add a condition where Left Value is `{{ $json.state === 'queued' }}` and Operation is boolean `true`.

10. **Create Node 10 (`Wait before status check`)**:
    - Type: `n8n-nodes-base.wait`
    - Configuration: Set Amount to `15` and Unit to `seconds`.

11. **Create Node 11 (`Read job status`)**:
    - Type: `n8n-nodes-base.httpRequest`
    - Configuration:
      - Method: `GET`
      - URL: `={{ $("Configuration").first().json.providerBaseUrl + "/jobs/" + $json.providerJobId }}`
      - Options: Set Response Format to `json`, enable `Never Error`, and set Timeout to `30000`ms.
      - Authentication: Configure Generic Credential Type (`httpHeaderAuth`).

12. **Create Node 12 (`Merge poll result and cap attempts`)**:
    - Type: `n8n-nodes-base.code`
    - Configuration: Paste the JavaScript code to track poll attempts (capping at 4) and parse terminal statuses.

13. **Create Node 13 (`Create audit manifest`)**:
    - Type: `n8n-nodes-base.code`
    - Configuration: Paste the JavaScript code to compile the final audit record containing tracking attributes and review notices.

14. **Establish Connections**:
    - `Start` $\rightarrow$ `Configuration`
    - `Configuration` $\rightarrow$ `Validate rights and removal regions`
    - `Validate rights and removal regions` $\rightarrow$ `Build provider requests`
    - `Build provider requests` $\rightarrow$ `Use offline demo fixtures`
    - `Use offline demo fixtures` (True Output) $\rightarrow$ `Simulate accepted and completed job`
    - `Use offline demo fixtures` (False Output) $\rightarrow$ `Submit each approved job once`
    - `Simulate accepted and completed job` $\rightarrow$ `Create audit manifest`
    - `Submit each approved job once` $\rightarrow$ `Classify submit response`
    - `Classify submit response` $\rightarrow$ `Queued for bounded polling`
    - `Queued for bounded polling` (True Output) $\rightarrow$ `Wait before status check`
    - `Queued for bounded polling` (False Output) $\rightarrow$ `Create audit manifest`
    - `Wait before status check` $\rightarrow$ `Read job status`
    - `Read job status` $\rightarrow$ `Merge poll result and cap attempts`
    - `Merge poll result and cap attempts` $\rightarrow$ `Create audit manifest`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by Shisan Hua, contributor to Video Watermark Remover. | [Video Watermark Remover Website](https://videowatermarkremover.org/) |