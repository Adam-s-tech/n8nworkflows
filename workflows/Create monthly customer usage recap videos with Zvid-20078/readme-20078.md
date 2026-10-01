Create monthly customer usage recap videos with Zvid

https://n8nworkflows.xyz/workflows/create-monthly-customer-usage-recap-videos-with-zvid-20078


# Create monthly customer usage recap videos with Zvid

### 1. Workflow Overview

This workflow automates the generation of vertical, month-in-review video recaps for customers using usage metrics, animated statistics, and optional delta indicators. It leverages the Zvid API for project composition, validation, and rendering, and includes options for draft creation, email delivery, and direct video review.

The logic is organized into seven sequential functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution via schedule or manual trigger and loads global project configuration variables.
- **1.2 Customer Metrics Loading:** Conditionally fetches customer usage data from an external endpoint or falls back to a bundled demo dataset.
- **1.3 Project Construction & Validation:** Programmatically builds the multi-scene video project payload and validates it against the Zvid API.
- **1.4 Execution Mode Routing:** Splits processing based on a dry-run flag, routing to either an editor draft creator or the live render queue.
- **1.5 Render & Polling Loop:** Submits the render job, waits for processing intervals, and polls status until completion or timeout.
- **1.6 Delivery & Summary Generation:** Conditionally sends the final video link via email and compiles a run summary.
- **1.7 Media Preview:** Downloads and displays the finalized MP4 binary output for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes workflow execution on the first day of every month or through manual user intervention, defining global brand and rendering parameters.
- **Nodes Involved:** `On the 1st of each month`, `Test manually`, `Config`
- **Node Details:**
  - **On the 1st of each month**
    - Type and technical role: `n8n-nodes-base.scheduleTrigger` (v1.2). Triggers execution automatically at 09:00 AM on day 1 of every month.
    - Configuration choices: Interval set to months, trigger hour 9, day of month 1.
    - Output connections: Connects to `Config`.
    - Edge cases: Ensure workflow timezone matches the intended local execution time.
  - **Test manually**
    - Type and technical role: `n8n-nodes-base.manualTrigger` (v1). Allows manual execution of the workflow for testing.
    - Output connections: Connects to `Config`.
  - **Config**
    - Type and technical role: `n8n-nodes-base.set` (v3.4). Holds static configuration parameters including API URLs, brand colors, typography, timing intervals, and delivery options.
    - Configuration choices: Mode set to raw JSON output containing settings such as `apiUrl`, `brandName`, `accent`, `dryRun`, and `sendEmail`.
    - Key expressions or variables used: Raw configuration JSON object.
    - Input connections: `On the 1st of each month`, `Test manually`.
    - Output connections: `Has metrics endpoint?`.

#### 2.2 Customer Metrics Loading
- **Overview:** Evaluates whether a custom metrics endpoint is provided, fetching external customer data or defaulting to a built-in sample dataset before normalizing the data structure.
- **Nodes Involved:** `Has metrics endpoint?`, `Fetch metrics`, `Normalize metrics`
- **Node Details:**
  - **Has metrics endpoint?**
    - Type and technical role: `n8n-nodes-base.if` (v2.2). Branches logic based on whether `metricsUrl` is populated in the configuration.
    - Configuration choices: Loose type validation; evaluates if `metricsUrl` is not an empty string.
    - Input connections: `Config`.
    - Output connections: True branch to `Fetch metrics`, False branch to `Normalize metrics`.
  - **Fetch metrics**
    - Type and technical role: `n8n-nodes-base.httpRequest` (v4.2). Retrieves raw usage metrics from the configured external URL.
    - Configuration choices: URL dynamically parsed from `Config` node; 30-second timeout; up to 3 retry attempts with a 5000ms delay on failure.
    - Input connections: `Has metrics endpoint?`.
    - Output connections: `Normalize metrics`.
    - Edge cases: HTTP errors, invalid JSON responses, or timeouts from the third-party endpoint.
  - **Normalize metrics**
    - Type and technical role: `n8n-nodes-base.code` (v2). Cleans and standardizes incoming metric attributes, applying a built-in demo payload (`Maya`, July 2026 stats) if no endpoint is configured.
    - Configuration choices: Custom JavaScript parsing stat strings and validating required schema properties (`userName`, `monthLabel`, `stats`).
    - Input connections: `Fetch metrics`, `Has metrics endpoint?` (false branch).
    - Output connections: `Build project JSON`.
    - Edge cases: Throws an explicit error if a configured endpoint returns malformed data or empty statistics.

#### 2.3 Project Construction & Validation
- **Overview:** Generates the complete scene-by-scene Zvid project JSON payload and validates it against the Zvid API to verify structural integrity and credit quotes.
- **Nodes Involved:** `Build project JSON`, `Validate project (free)`, `Check validation`
- **Node Details:**
  - **Build project JSON**
    - Type and technical role: `n8n-nodes-base.code` (v2). Compiles configuration rules and normalized customer metrics into a structured Zvid render payload containing intro, data stat scenes, and an outro.
    - Configuration choices: JavaScript code generating dynamic SVG components, font styling, color manipulation functions (`hexToRgb`, `shade`), and scene durations.
    - Input connections: `Normalize metrics`.
    - Output connections: `Validate project (free)`.
  - **Validate project (free)**
    - Type and technical role: `@zvid/n8n-nodes-zvid.zvid` (v1). Calls the Zvid validation endpoint to check payload correctness and estimate required credits without triggering a billable render.
    - Configuration choices: Source set to json, resource set to render, operation set to validate. Project JSON populated via expression stringifying the payload.
    - Credentials: Requires Zvid API credential.
    - Input connections: `Build project JSON`.
    - Output connections: `Check validation`.
    - Edge cases: API authentication errors, schema validation failures.
  - **Check validation**
    - Type and technical role: `n8n-nodes-base.code` (v2). Inspects the validation response, throwing descriptive errors if validation fails or forwarding project metadata and credit requirements on success.
    - Configuration choices: JavaScript checking HTTP status and `valid` flags.
    - Input connections: `Validate project (free)`.
    - Output connections: `Dry run?`.

#### 2.4 Execution Mode Routing
- **Overview:** Inspects the `dryRun` configuration flag to branch execution between saving an editor draft or initiating a live video render.
- **Nodes Involved:** `Dry run?`, `Save draft to editor`, `Dry run summary`
- **Node Details:**
  - **Dry run?**
    - Type and technical role: `n8n-nodes-base.if` (v2.2). Evaluates whether `dryRun` is set to true.
    - Input connections: `Check validation`.
    - Output connections: True branch to `Save draft to editor`, False branch to `Submit render`.
  - **Save draft to editor**
    - Type and technical role: `@zvid/n8n-nodes-zvid.zvid` (v1). Creates an editable project draft in the Zvid workspace instead of rendering a final video file.
    - Configuration choices: Resource set to project, operation set to create. Configured with error handling (`onError: continueRegularOutput`). Up to 3 retry attempts.
    - Credentials: Requires Zvid API credential.
    - Input connections: `Dry run?` (true branch).
    - Output connections: `Dry run summary`.
  - **Dry run summary**
    - Type and technical role: `n8n-nodes-base.code` (v2). Generates a structured output report containing the editor draft link, required credits, and warnings.
    - Input connections: `Save draft to editor`.
    - Output connections: None (terminal branch for dry runs).

#### 2.5 Render & Polling Loop
- **Overview:** Submits the validated project payload to the Zvid rendering queue and polls job status iteratively until rendering completes or times out.
- **Nodes Involved:** `Submit render`, `Wait`, `Get render status`, `Render finished?`, `Still rendering?`
- **Node Details:**
  - **Submit render**
    - Type and technical role: `@zvid/n8n-nodes-zvid.zvid` (v1). Submits the video render job asynchronously to Zvid.
    - Configuration choices: Resource set to render, operation set to create, renderType set to video, `waitForCompletion` set to false. Max 3 retries.
    - Credentials: Requires Zvid API credential.
    - Input connections: `Dry run?` (false branch).
    - Output connections: `Wait`.
  - **Wait**
    - Type and technical role: `n8n-nodes-base.wait` (v1.1). Pauses execution for a configured number of seconds between status polls.
    - Configuration choices: Duration amount mapped from `Config` node (`pollSeconds`).
    - Input connections: `Submit render`, `Still rendering?`.
    - Output connections: `Get render status`.
  - **Get render status**
    - Type and technical role: `@zvid/n8n-nodes-zvid.zvid` (v1). Queries the current processing state of the submitted render job.
    - Configuration choices: Resource set to render, operation set to get, jobId mapped from `Submit render`.
    - Credentials: Requires Zvid API credential.
    - Input connections: `Wait`.
    - Output connections: `Render finished?`.
  - **Render finished?**
    - Type and technical role: `n8n-nodes-base.if` (v2.2). Evaluates whether the render job state equals `completed`.
    - Input connections: `Get render status`.
    - Output connections: True branch to `Send email?`, False branch to `Still rendering?`.
  - **Still rendering?**
    - Type and technical role: `n8n-nodes-base.code` (v2). Checks for render failure states or timeout limits based on elapsed poll iterations. Throws an error if the job failed or exceeded maximum timeout minutes.
    - Input connections: `Render finished?` (false branch).
    - Output connections: `Wait` (loops back to polling).

#### 2.6 Delivery & Summary Generation
- **Overview:** Assembles execution metrics and conditionally delivers the completed video link via email to the specified recipient.
- **Nodes Involved:** `Send email?`, `Email the video`, `Run summary`
- **Node Details:**
  - **Send email?**
    - Type and technical role: `n8n-nodes-base.if` (v2.2). Evaluates whether `sendEmail` is enabled in configuration.
    - Input connections: `Render finished?` (true branch).
    - Output connections: True branch to `Email the video`, False branch to `Run summary`.
  - **Email the video**
    - Type and technical role: `n8n-nodes-base.emailSend` (v2.1). Sends an HTML-formatted email containing the video watch link to the customer.
    - Configuration choices: Configured with SMTP credentials, dynamic subject line, recipient, sender, and HTML body expression. Error handling set to continue regular output (`onError: continueRegularOutput`).
    - Credentials: Requires SMTP account credentials.
    - Input connections: `Send email?` (true branch).
    - Output connections: `Run summary`.
  - **Run summary**
    - Type and technical role: `n8n-nodes-base.code` (v2). Compiles final execution metadata, including job ID, credits charged, video duration, and email delivery status.
    - Input connections: `Email the video`, `Send email?` (false branch).
    - Output connections: `Video ready to watch?`.

#### 2.7 Media Preview
- **Overview:** Verifies the completion status and downloads the final MP4 binary file for local inspection or downstream handling.
- **Nodes Involved:** `Video ready to watch?`, `▶ Watch video`
- **Node Details:**
  - **Video ready to watch?**
    - Type and technical role: `n8n-nodes-base.if` (v2.2). Checks that the run is not a dry run and that a valid video URL is present.
    - Input connections: `Run summary`.
    - Output connections: True branch to `▶ Watch video`, False branch (empty).
  - **▶ Watch video**
    - Type and technical role: `n8n-nodes-base.httpRequest` (v2). Performs an HTTP GET request to download the rendered video binary.
    - Configuration choices: Response format set to file, data property name set to `data`, 30-second timeout, error handling set to continue regular output (`onError: continueRegularOutput`). Max 3 retries.
    - Input connections: `Video ready to watch?` (true branch).
    - Output connections: None (terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On the 1st of each month | n8n-nodes-base.scheduleTrigger | Triggers workflow on day 1 of every month at 9 AM. | None | Config | 1. Choose settings and an / Create a month-in-review stats video |
| Test manually | n8n-nodes-base.manualTrigger | Triggers workflow manually for testing. | None | Config | 1. Choose settings and an / Create a month-in-review stats video |
| Config | n8n-nodes-base.set | Defines global configuration variables and visual styling. | On the 1st of each month, Test manually | Has metrics endpoint? | 1. Choose settings and an / Create a month-in-review stats video |
| Has metrics endpoint? | n8n-nodes-base.if | Checks if a custom metrics endpoint URL is configured. | Config | Fetch metrics, Normalize metrics | 2. Load the customer’s metrics / Create a month-in-review stats video |
| Fetch metrics | n8n-nodes-base.httpRequest | Retrieves customer metrics from an external endpoint. | Has metrics endpoint? | Normalize metrics | 2. Load the customer’s metrics / Create a month-in-review stats video |
| Normalize metrics | n8n-nodes-base.code | Normalizes incoming metrics or applies demo dataset. | Has metrics endpoint?, Fetch metrics | Build project JSON | 2. Load the customer’s metrics / Create a month-in-review stats video |
| Build project JSON | n8n-nodes-base.code | Builds the complete Zvid video scene and visual payload. | Normalize metrics | Validate project (free) | 3. Build and validate the design / Create a month-in-review stats video |
| Validate project (free) | @zvid/n8n-nodes-zvid.zvid | Validates the project payload and checks credit quotes. | Build project JSON | Check validation | 3. Build and validate the design / Create a month-in-review stats video |
| Check validation | n8n-nodes-base.code | Inspects validation results and handles payload errors. | Validate project (free) | Dry run? | 3. Build and validate the design / Create a month-in-review stats video |
| Submit render | @zvid/n8n-nodes-zvid.zvid | Submits the render job to the Zvid rendering queue. | Dry run? | Wait | 5. Render and wait for completion / Create a month-in-review stats video |
| Wait | n8n-nodes-base.wait | Pauses execution between status poll requests. | Submit render, Still rendering? | Get render status | 5. Render and wait for completion / Create a month-in-review stats video |
| Get render status | @zvid/n8n-nodes-zvid.zvid | Queries the processing status of the render job. | Wait | Render finished? | 5. Render and wait for completion / Create a month-in-review stats video |
| Render finished? | n8n-nodes-base.if | Evaluates if the render job has completed successfully. | Get render status | Send email?, Still rendering? | 5. Render and wait for completion / Create a month-in-review stats video |
| Still rendering? | n8n-nodes-base.code | Checks timeout limits and handles render failure states. | Render finished? | Wait | 5. Render and wait for completion / Create a month-in-review stats video |
| Send email? | n8n-nodes-base.if | Evaluates whether email delivery is enabled. | Render finished? | Email the video, Run summary | 6. Send the optional delivery / Create a month-in-review stats video |
| Email the video | n8n-nodes-base.emailSend | Sends the completed video watch link via email. | Send email? | Run summary | 6. Send the optional delivery / Create a month-in-review stats video |
| Run summary | n8n-nodes-base.code | Compiles the final execution report and metadata. | Email the video, Send email? | Video ready to watch? | 6. Send the optional delivery / Create a month-in-review stats video |
| ▶ Watch video | n8n-nodes-base.httpRequest | Downloads the rendered MP4 file binary for review. | Video ready to watch? | None | 7. Review the finished media / Create a month-in-review stats video |
| Dry run? | n8n-nodes-base.if | Branches execution based on the dry-run configuration. | Check validation | Save draft to editor, Submit render | 4. Choose an optional editor preview / Create a month-in-review stats video |
| Save draft to editor | @zvid/n8n-nodes-zvid.zvid | Saves an editable project draft in the Zvid workspace. | Dry run? | Dry run summary | 4. Choose an optional editor preview / Create a month-in-review stats video |
| Dry run summary | n8n-nodes-base.code | Generates a dry-run report with editor draft links. | Save draft to editor | None | 4. Choose an optional editor preview / Create a month-in-review stats video |
| Video ready to watch? | n8n-nodes-base.if | Verifies that a valid video URL exists for download. | Run summary | ▶ Watch video | 7. Review the finished media / Create a month-in-review stats video |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Community Node:** In your n8n instance, go to **Settings → Community nodes**, install `@zvid/n8n-nodes-zvid`, and restart if required.
2. **Configure Credentials:** Create a Zvid API credential using your API key from `https://app.zvid.io/api-keys` and Base URL `https://api.zvid.io`. Optionally configure SMTP credentials for email delivery.
3. **Create Entry Triggers:**
   - Add a **Schedule Trigger** (`On the 1st of each month`): set interval to months, trigger hour to 9, day of month to 1.
   - Add a **Manual Trigger** (`Test manually`).
4. **Create Configuration Node:**
   - Add a **Set** node named `Config`.
   - Set mode to raw JSON and provide configuration parameters (`apiUrl`, `metricsUrl`, `brandName`, `accent`, `dryRun`, `sendEmail`, etc.).
   - Connect both triggers to `Config`.
5. **Set Up Metrics Loading:**
   - Add an **If** node named `Has metrics endpoint?` checking `={{ String($json.metricsUrl || '').trim() !== '' }}`. Connect `Config` to it.
   - Add an **HTTP Request** node named `Fetch metrics`: connect the true branch of `Has metrics endpoint?`. Set URL to `={{ $('Config').first().json.metricsUrl }}`, timeout to 30000ms, retry enabled with 3 tries.
   - Add a **Code** node named `Normalize metrics`: connect `Fetch metrics` and the false branch of `Has metrics endpoint?`. Paste JavaScript code to parse stats and handle fallback demo data.
6. **Build and Validate Project:**
   - Add a **Code** node named `Build project JSON`: connect `Normalize metrics`. Paste JavaScript code to construct scene payloads and video assets.
   - Add a **Zvid** node (`Validate project (free)`): connect `Build project JSON`. Set resource to `render`, operation to `validate`, source to `json`, and Project JSON to `={{ JSON.stringify(({ payload: $json.payload }).payload) }}` using your Zvid credential.
   - Add a **Code** node named `Check validation`: connect `Validate project (free)` to inspect validation status and extract credit requirements.
7. **Configure Execution Mode Routing:**
   - Add an **If** node named `Dry run?`: connect `Check validation`. Condition: `={{ $('Config').first().json.dryRun }}`.
   - **Dry Run Branch:**
     - Add a **Zvid** node (`Save draft to editor`): connect true branch of `Dry run?`. Resource: `project`, operation: `create`.
     - Add a **Code** node named `Dry run summary`: connect `Save draft to editor`.
   - **Live Render Branch:**
     - Add a **Zvid** node (`Submit render`): connect false branch of `Dry run?`. Resource: `render`, operation: `create`, renderType `video`, `waitForCompletion` false.
     - Add a **Wait** node (`Wait`): connect `Submit render`. Unit: seconds, amount: `={{ $('Config').first().json.pollSeconds }}`.
     - Add a **Zvid** node (`Get render status`): connect `Wait`. Resource: `render`, operation: `get`, jobId: `={{ $('Submit render').first().json.jobId }}`.
     - Add an **If** node named `Render finished?`: connect `Get render status`. Condition: `={{ $json.state === 'completed' }}`.
     - Add a **Code** node named `Still rendering?`: connect false branch of `Render finished?`. Connect its output back to the `Wait` node.
8. **Configure Delivery & Summary:**
   - Add an **If** node named `Send email?`: connect true branch of `Render finished?`. Condition: `={{ $('Config').first().json.sendEmail }}`.
   - Add an **Email Send** node (`Email the video`): connect true branch of `Send email?`. Configure SMTP credentials, HTML body, subject, recipient, and sender properties. Set error handling to continue regular output.
   - Add a **Code** node named `Run summary`: connect `Email the video` and the false branch of `Send email?`.
9. **Configure Media Preview:**
   - Add an **If** node named `Video ready to watch?`: connect `Run summary`. Condition: `={{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
   - Add an **HTTP Request** node (`▶ Watch video`): connect true branch of `Video ready to watch?`. URL: `={{ $json.videoUrl }}`, response format: `file`, data property name: `data`. Set error handling to continue regular output.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Detailed setup guide | [GitHub Workflow Guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/month-in-review-stats.md) |
| Get your Zvid API key | [Zvid API Keys](https://app.zvid.io/api-keys) |
| Zvid Support & Contact | [Zvid Contact Page](https://zvid.io/contact) |