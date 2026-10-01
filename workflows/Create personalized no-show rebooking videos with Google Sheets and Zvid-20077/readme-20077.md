Create personalized no-show rebooking videos with Google Sheets and Zvid

https://n8nworkflows.xyz/workflows/create-personalized-no-show-rebooking-videos-with-google-sheets-and-zvid-20077


# Create personalized no-show rebooking videos with Google Sheets and Zvid

### 1. Workflow Overview

This n8n workflow automates the generation of personalized video re-engagement invitations for attendees who missed scheduled meetings or webinars (no-shows). It can be triggered manually, via a 15-minute schedule, or through real-time webhooks (e.g., from Calendly or a CRM). The system pulls data, inspects background music assets, builds a structured 3-scene vertical video payload using the Zvid engine, validates it, and branches based on a dry-run or live execution mode. In live mode, it renders the video, polls for completion, optionally sends an email notification with the video link via SMTP, updates Google Sheets, and downloads the resulting MP4 file.

The logical blocks are organized as follows:
- **1.1 Input Reception & Configuration:** Receives triggers (Manual, Schedule, Webhook), loads configuration parameters, and checks background music availability via HTTP.
- **1.2 Data Normalization & Queue Processing:** Determines whether to process a webhook payload or poll Google Sheets for a pending no-show row, validating input requirements.
- **1.3 Project Construction & Design Validation:** Compiles text and design configurations into a multi-scene Zvid project JSON object, validates its schema against the Zvid API, and evaluates whether to run a dry-run or live render.
- **1.4 Rendering & Polling Loop:** Submits the render job to Zvid, polls its completion status iteratively with a built-in timeout guard, and extracts the final video output URL.
- **1.5 Distribution, Recording & Review:** Delivers the email notification (if enabled), updates the tracking status in Google Sheets, returns responses to webhook callers, and downloads/previews the final MP4 file.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** This block initializes the workflow from multiple entry points, parses global configuration settings, and performs a pre-flight check on optional background music to prevent runtime failures.
- **Nodes Involved:** `Test manually`, `Every 15 minutes`, `No-show webhook`, `Config`, `Check music`, `Music guard`.
- **Node Details:**
  - `Test manually` (n8n-nodes-base.manualTrigger)
    - *Role:* Initiates workflow executions manually for testing.
    - *Config:* Default settings.
    - *Connections:* Input: None | Output: `Config`
    - *Failure Modes:* None.
  - `Every 15 minutes` (n8n-nodes-base.scheduleTrigger)
    - *Role:* Triggers automated sheet polling on a 15-minute timer.
    - *Config:* Interval rule set to 15 minutes.
    - *Connections:* Input: None | Output: `Config`
    - *Failure Modes:* None.
  - `No-show webhook` (n8n-nodes-base.webhook)
    - *Role:* Receives incoming POST requests containing no-show attendee data.
    - *Config:* Method set to POST, path `no-show`, response mode set to use response node.
    - *Connections:* Input: None | Output: `Config`
    - *Failure Modes:* Malformed incoming JSON payloads.
  - `Config` (n8n-nodes-base.set)
    - *Role:* Establishes global parameters including API URLs, branding tokens, presenter details, color palettes, and operational flags (`dryRun`, `sendEmail`, `source`).
    - *Config:* Raw JSON mode outputting comprehensive configuration settings.
    - *Connections:* Input: `Test manually`, `Every 15 minutes`, `No-show webhook` | Output: `Check music`
    - *Failure Modes:* Missing required configuration keys.
  - `Check music` (n8n-nodes-base.httpRequest)
    - *Role:* Performs an HTTP HEAD request to verify the availability and headers of the background music URL.
    - *Config:* Method HEAD, timeout 15,000ms, ignores standard errors to capture full response headers.
    - *Connections:* Input: `Config` | Output: `Music guard`
    - *Failure Modes:* Network timeouts, unreachable URLs, or HTTP 4xx/5xx responses (handled gracefully by downstream code).
  - `Music guard` (n8n-nodes-base.code)
    - *Role:* Evaluates the HTTP response from the music check, validating file size against limits (e.g., 5 MB cap) and stripping the track if invalid.
    - *Config:* Custom JavaScript logic evaluating content-length and status codes.
    - *Connections:* Input: `Check music` | Output: `Source?`
    - *Failure Modes:* Parsing errors on unexpected headers.

#### 1.2 Data Normalization & Queue Processing
- **Overview:** Inspects the operational source mode (`webhook` vs. `sheet`), normalizes incoming webhooks or reads the next unserviced Google Sheets row, and confirms whether a valid target item exists.
- **Nodes Involved:** `Source?`, `Webhook payload`, `Read no-shows sheet`, `Pick next no-show`, `No-show found?`, `Nothing to send`.
- **Node Details:**
  - `Source?` (n8n-nodes-base.if)
    - *Role:* Branches workflow execution based on whether the data source is configured as `webhook` or `sheet`.
    - *Config:* Evaluates `{{ $('Config').first().json.source === 'webhook' }}`.
    - *Connections:* Input: `Music guard` | Output: True (`Webhook payload`), False (`Read no-shows sheet`)
    - *Failure Modes:* None.
  - `Webhook payload` (n8n-nodes-base.code)
    - *Role:* Normalizes incoming webhook payloads or falls back to sample configuration data when run manually.
    - *Config:* Custom JavaScript matching field variants (Calendly vs. flat schemas).
    - *Connections:* Input: `Source?` (True branch) | Output: `No-show found?`
    - *Failure Modes:* Missing essential attendee names (`firstName`).
  - `Read no-shows sheet` (n8n-nodes-base.googleSheets)
    - *Role:* Reads records from the configured Google Sheets document.
    - *Config:* Resource `sheet`, operation `read`, with retry logic on failure.
    - *Connections:* Input: `Source?` (False branch) | Output: `Pick next no-show`
    - *Failure Modes:* Invalid Google Sheets credentials, missing document ID, or network errors.
  - `Pick next no-show` (n8n-nodes-base.code)
    - *Role:* Identifies the first row where the `Status` column is empty, skipping spacer rows and mismatched run modes.
    - *Config:* Custom JavaScript filtering items and extracting row numbers.
    - *Connections:* Input: `Read no-shows sheet` | Output: `No-show found?`
    - *Failure Modes:* Missing required columns (`FirstName`, `EventName`).
  - `No-show found?` (n8n-nodes-base.if)
    - *Role:* Verifies whether a valid no-show entity was extracted from the queue or webhook.
    - *Config:* Evaluates `{{ $json.found }}`.
    - *Connections:* Input: `Webhook payload`, `Pick next no-show` | Output: True (`Build project JSON`), False (`Nothing to send`)
    - *Failure Modes:* None.
  - `Nothing to send` (n8n-nodes-base.code)
    - *Role:* Generates a graceful termination payload when the queue is empty or operational modes mismatch.
    - *Config:* Custom JavaScript outputting reason and checking metrics.
    - *Connections:* Input: `No-show found?` (False branch) | Output: `Respond ok`
    - *Failure Modes:* None.

#### 1.3 Project Construction & Design Validation
- **Overview:** Compiles dynamic tokens, design coordinates, typography, and optional audio tracks into a complete Zvid project JSON schema, validating it against the Zvid API before execution.
- **Nodes Involved:** `Build project JSON`, `Validate project (free)`, `Check validation`, `Dry run?`, `Save draft to editor`, `Dry run summary`.
- **Node Details:**
  - `Build project JSON` (n8n-nodes-base.code)
    - *Role:* Assembles the 3-scene vertical video definition (hook, recap highlights, call-to-action) incorporating safe text wrapping, color blending, and optional presenter avatars.
    - *Config:* Custom JavaScript containing layout math and SVG backdrop generators.
    - *Connections:* Input: `No-show found?` (True branch) | Output: `Validate project (free)`
    - *Failure Modes:* Script evaluation errors due to malformed configuration values.
  - `Validate project (free)` (@zvid/n8n-nodes-zvid.zvid)
    - *Role:* Submits the generated project schema to Zvid's free validation endpoint to check for syntax or layout issues and calculate credit costs.
    - *Config:* Resource `render`, operation `validate`, using JSON source configuration.
    - *Connections:* Input: `Build project JSON` | Output: `Check validation`
    - *Failure Modes:* API authentication failures or network timeouts.
  - `Check validation` (n8n-nodes-base.code)
    - *Role:* Parses validation responses from Zvid, surfacing detailed field error messages if validation fails.
    - *Config:* Custom JavaScript validating HTTP status codes and extracting schema versions and warnings.
    - *Connections:* Input: `Validate project (free)` | Output: `Dry run?`
    - *Failure Modes:* Throws explicit runtime errors when validation fails (non-200 status).
  - `Dry run?` (n8n-nodes-base.if)
    - *Role:* Branches based on whether `dryRun` mode is enabled in the configuration.
    - *Config:* Evaluates `{{ $('Config').first().json.dryRun }}`.
    - *Connections:* Input: `Check validation` | Output: True (`Save draft to editor`), False (`Submit render`)
    - *Failure Modes:* None.
  - `Save draft to editor` (@zvid/n8n-nodes-zvid.zvid)
    - *Role:* Creates an editable draft project within the Zvid workspace during dry-run executions.
    - *Config:* Resource `project`, operation `create`, with error continuation enabled and retry settings.
    - *Connections:* Input: `Dry run?` (True branch) | Output: `Dry run summary`
    - *Failure Modes:* API rejection or permission issues (mitigated by `onError: continueRegularOutput`).
  - `Dry run summary` (n8n-nodes-base.code)
    - *Role:* Compiles a summary report including editor links and credit quotes without spending render credits or updating external sheets.
    - *Config:* Custom JavaScript formatting dry run metadata.
    - *Connections:* Input: `Save draft to editor` | Output: `Respond ok`
    - *Failure Modes:* None.

#### 1.4 Rendering & Polling Loop
- **Overview:** Submits paid render requests to Zvid and manages an asynchronous polling loop to check job progress until completion or timeout.
- **Nodes Involved:** `Submit render`, `Wait`, `Get render status`, `Render finished?`, `Still rendering?`, `Video URL`.
- **Node Details:**
  - `Submit render` (@zvid/n8n-nodes-zvid.zvid)
    - *Role:* Submits the video render job asynchronously to Zvid.
    - *Config:* Resource `render`, operation `create`, video render type, with `waitForCompletion` set to false.
    - *Connections:* Input: `Dry run?` (False branch) | Output: `Wait`
    - *Failure Modes:* Insufficient account credits or invalid project payloads.
  - `Wait` (n8n-nodes-base.wait)
    - *Role:* Pauses execution between polling intervals.
    - *Config:* Dynamic unit and amount based on configuration poll settings.
    - *Connections:* Input: `Submit render`, `Still rendering?` | Output: `Get render status`
    - *Failure Modes:* None.
  - `Get render status` (@zvid/n8n-nodes-zvid.zvid)
    - *Role:* Queries the current progress and state of the submitted render job from Zvid.
    - *Config:* Resource `render`, operation `get`, using job IDs from the submission step.
    - *Connections:* Input: `Wait` | Output: `Render finished?`
    - *Failure Modes:* API connectivity issues or invalid job IDs.
  - `Render finished?` (n8n-nodes-base.if)
    - *Role:* Checks whether the render job state has reached completion.
    - *Config:* Evaluates `{{ $json.state === 'completed' }}`.
    - *Connections:* Input: `Get render status` | Output: True (`Video URL`), False (`Still rendering?`)
    - *Failure Modes:* None.
  - `Still rendering?` (n8n-nodes-base.code)
    - *Role:* Validates render job health, throwing errors on failure or timeout limits, or looping back to wait.
    - *Config:* Custom JavaScript calculating maximum retry attempts against configured timeout minutes.
    - *Connections:* Input: `Render finished?` (False branch) | Output: `Wait` (loops back)
    - *Failure Modes:* Throws timeout errors or render failure reasons when jobs fail.
  - `Video URL` (n8n-nodes-base.code)
    - *Role:* Extracts the final video URL from the completed job payload and constructs HTML/text email templates.
    - *Config:* Custom JavaScript template engine escaping HTML entities and injecting video links.
    - *Connections:* Input: `Render finished?` (True branch) | Output: `Send email?`
    - *Failure Modes:* Missing output URLs in completed job results.

#### 1.5 Distribution, Recording & Review
- **Overview:** Handles optional email distribution via SMTP, updates Google Sheets tracking statuses, responds to webhook triggers, and downloads/previews the generated MP4 file.
- **Nodes Involved:** `Send email?`, `Send re-book email (SMTP)`, `Sheet row?`, `Mark row sent`, `Run summary`, `Respond ok`, `Video ready to watch?`, `▶ Watch video`.
- **Node Details:**
  - `Send email?` (n8n-nodes-base.if)
    - *Role:* Evaluates whether email sending is enabled and whether a valid recipient address is present.
    - *Config:* Evaluates `sendEmail` configuration flag and email format validity.
    - *Connections:* Input: `Video URL` | Output: True (`Send re-book email (SMTP)`), False (`Sheet row?`)
    - *Failure Modes:* None.
  - `Send re-book email (SMTP)` (n8n-nodes-base.emailSend)
    - *Role:* Dispatches the HTML email invitation containing the personalized video recap link.
    - *Config:* Uses HTML format, dynamic subject/body parameters, and retry policies with error continuation enabled.
    - *Connections:* Input: `Send email?` (True branch) | Output: `Sheet row?`
    - *Failure Modes:* SMTP authentication errors or connection timeouts (handled gracefully by error continuation).
  - `Sheet row?` (n8n-nodes-base.if)
    - *Role:* Checks whether the processed item originated from a Google Sheets row requiring write-back updates.
    - *Config:* Evaluates `{{ $('Video URL').first().json.rowNumber !== null }}`.
    - *Connections:* Input: `Send email?` (False branch), `Send re-book email (SMTP)` | Output: True (`Mark row sent`), False (`Run summary`)
    - *Failure Modes:* None.
  - `Mark row sent` (n8n-nodes-base.googleSheets)
    - *Role:* Updates the specific Google Sheets row with the completion status and generated video URL.
    - *Config:* Resource `sheet`, operation `update`, matching on `row_number`.
    - *Connections:* Input: `Sheet row?` (True branch) | Output: `Run summary`
    - *Failure Modes:* Sheet permission errors or missing row identifiers.
  - `Run summary` (n8n-nodes-base.code)
    - *Role:* Compiles a comprehensive final execution report detailing credit charges, delivery status, and tracking metrics.
    - *Config:* Custom JavaScript aggregating execution metrics and email delivery outcomes.
    - *Connections:* Input: `Sheet row?` (False branch), `Mark row sent` | Output: `Video ready to watch?`, `Respond ok`
    - *Failure Modes:* None.
  - `Respond ok` (n8n-nodes-base.respondToWebhook)
    - *Role:* Returns HTTP responses to webhook callers.
    - *Config:* Responds with the first incoming item.
    - *Connections:* Input: `Nothing to send`, `Dry run summary`, `Run summary` | Output: None
    - *Failure Modes:* Closed webhook connection streams.
  - `Video ready to watch?` (n8n-nodes-base.if)
    - *Role:* Determines whether a finished live video file is available for local binary downloading/previewing.
    - *Config:* Evaluates `{{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
    - *Connections:* Input: `Run summary` | Output: True (`▶ Watch video`), False (None)
    - *Failure Modes:* None.
  - `▶ Watch video` (n8n-nodes-base.httpRequest)
    - *Role:* Downloads the completed MP4 video file into binary data for inspection within n8n.
    - *Config:* Method GET, response format set to file, saving to binary property `data`.
    - *Connections:* Input: `Video ready to watch?` (True branch) | Output: None
    - *Failure Modes:* Network interruptions or broken video URLs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Test manually | n8n-nodes-base.manualTrigger | Initiates workflow executions manually for testing. | None | Config | 1. Choose settings and an entry point |
| Every 15 minutes | n8n-nodes-base.scheduleTrigger | Triggers automated sheet polling on a 15-minute timer. | None | Config | 1. Choose settings and an entry point |
| No-show webhook | n8n-nodes-base.webhook | Receives incoming POST requests containing no-show attendee data. | None | Config | 1. Choose settings and an entry point |
| Config | n8n-nodes-base.set | Establishes global parameters including API URLs, branding tokens, presenter details, color palettes, and operational flags. | Test manually, Every 15 minutes, No-show webhook | Check music | 1. Choose settings and an entry point |
| Check music | n8n-nodes-base.httpRequest | Performs an HTTP HEAD request to verify the availability and headers of the background music URL. | Config | Music guard | 4. Check the background music |
| Music guard | n8n-nodes-base.code | Evaluates the HTTP response from the music check, validating file size against limits and stripping the track if invalid. | Check music | Source? | 4. Check the background music |
| Source? | n8n-nodes-base.if | Branches workflow execution based on whether the data source is configured as webhook or sheet. | Music guard | Webhook payload, Read no-shows sheet | 2. Load recorded no-shows |
| Webhook payload | n8n-nodes-base.code | Normalizes incoming webhook payloads or falls back to sample configuration data when run manually. | Source? | No-show found? | 2. Load recorded no-shows |
| Read no-shows sheet | n8n-nodes-base.googleSheets | Reads records from the configured Google Sheets document. | Source? | Pick next no-show | 2. Load recorded no-shows |
| Pick next no-show | n8n-nodes-base.code | Identifies the first row where the Status column is empty, skipping spacer rows and mismatched run modes. | Read no-shows sheet | No-show found? | 2. Load recorded no-shows |
| No-show found? | n8n-nodes-base.if | Verifies whether a valid no-show entity was extracted from the queue or webhook. | Webhook payload, Pick next no-show | Build project JSON, Nothing to send | 3. Check for a pending reminder |
| Nothing to send | n8n-nodes-base.code | Generates a graceful termination payload when the queue is empty or operational modes mismatch. | No-show found? | Respond ok | 3. Check for a pending reminder |
| Build project JSON | n8n-nodes-base.code | Assembles the 3-scene vertical video definition incorporating safe text wrapping, color blending, and optional presenter avatars. | No-show found? | Validate project (free) | 5. Build and validate the design |
| Validate project (free) | @zvid/n8n-nodes-zvid.zvid | Submits the generated project schema to Zvid's free validation endpoint to check for syntax or layout issues and calculate credit costs. | Build project JSON | Check validation | 5. Build and validate the design |
| Check validation | n8n-nodes-base.code | Parses validation responses from Zvid, surfacing detailed field error messages if validation fails. | Validate project (free) | Dry run? | 5. Build and validate the design |
| Dry run? | n8n-nodes-base.if | Branches based on whether dryRun mode is enabled in the configuration. | Check validation | Save draft to editor, Submit render | 6. Choose an optional editor preview |
| Save draft to editor | @zvid/n8n-nodes-zvid.zvid | Creates an editable draft project within the Zvid workspace during dry-run executions. | Dry run? | Dry run summary | 6. Choose an optional editor preview |
| Dry run summary | n8n-nodes-base.code | Compiles a summary report including editor links and credit quotes without spending render credits or updating external sheets. | Save draft to editor | Respond ok | 6. Choose an optional editor preview |
| Submit render | @zvid/n8n-nodes-zvid.zvid | Submits the video render job asynchronously to Zvid. | Dry run? | Wait | 7. Render and wait for completion |
| Wait | n8n-nodes-base.wait | Pauses execution between polling intervals. | Submit render, Still rendering? | Get render status | 7. Render and wait for completion |
| Get render status | @zvid/n8n-nodes-zvid.zvid | Queries the current progress and state of the submitted render job from Zvid. | Wait | Render finished? | 7. Render and wait for completion |
| Render finished? | n8n-nodes-base.if | Checks whether the render job state has reached completion. | Get render status | Video URL, Still rendering? | 7. Render and wait for completion |
| Still rendering? | n8n-nodes-base.code | Validates render job health, throwing errors on failure or timeout limits, or looping back to wait. | Render finished? | Wait | 7. Render and wait for completion |
| Video URL | n8n-nodes-base.code | Extracts the final video URL from the completed job payload and constructs HTML/text email templates. | Render finished? | Send email? | 8. Optionally send the rebooking video |
| Send email? | n8n-nodes-base.if | Evaluates whether email sending is enabled and whether a valid recipient address is present. | Video URL | Send re-book email (SMTP), Sheet row? | 8. Optionally send the rebooking video |
| Send re-book email (SMTP) | n8n-nodes-base.emailSend | Dispatches the HTML email invitation containing the personalized video recap link. | Send email? | Sheet row? | 8. Optionally send the rebooking video |
| Sheet row? | n8n-nodes-base.if | Checks whether the processed item originated from a Google Sheets row requiring write-back updates. | Send email?, Send re-book email (SMTP) | Mark row sent, Run summary | 9. Record or return the result |
| Mark row sent | n8n-nodes-base.googleSheets | Updates the specific Google Sheets row with the completion status and generated video URL. | Sheet row? | Run summary | 9. Record or return the result |
| Run summary | n8n-nodes-base.code | Compiles a comprehensive final execution report detailing credit charges, delivery status, and tracking metrics. | Sheet row?, Mark row sent | Video ready to watch?, Respond ok | 9. Record or return the result |
| Respond ok | n8n-nodes-base.respondToWebhook | Returns HTTP responses to webhook callers. | Nothing to send, Dry run summary, Run summary | None | 9. Record or return the result |
| ▶ Watch video | n8n-nodes-base.httpRequest | Downloads the completed MP4 video file into binary data for inspection within n8n. | Video ready to watch? | None | 10. Review the finished media |
| Video ready to watch? | n8n-nodes-base.if | Determines whether a finished live video file is available for local binary downloading/previewing. | Run summary | ▶ Watch video | 10. Review the finished media |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Install Community Nodes and Credentials
1. Open your n8n instance and navigate to **Settings → Community nodes**.
2. Install the package: `@zvid/n8n-nodes-zvid`. (Note: Self-hosted instances may require a workspace administrator or owner to perform this installation).
3. Generate a Zvid API key at [Zvid API Keys](https://app.zvid.io/api-keys) and set up a new Zvid API credential in n8n using Base URL `https://api.zvid.io`.
4. Configure Google Sheets OAuth2/Service Account credentials and SMTP credentials (if email delivery will be used).

#### Step 2: Create Triggers and Configuration
1. **Manual Trigger:** Create a `Manual Trigger` node (`Test manually`).
2. **Schedule Trigger:** Create a `Schedule Trigger` node (`Every 15 minutes`) configured with an interval rule of 15 minutes.
3. **Webhook Trigger:** Create a `Webhook` node (`No-show webhook`) with HTTP method `POST`, path `no-show`, and response mode set to `Respond Using 'Respond to Webhook' Node`.
4. **Configuration Set Node:** Create a `Set` node (`Config`) in raw JSON mode. Populate it with global settings including `apiUrl`, `editorUrl`, `source` (`sheet` or `webhook`), branding strings, font selections, color hex codes, and the `samplePayload` object.
5. Connect `Test manually`, `Every 15 minutes`, and `No-show webhook` outputs to the `Config` node.

#### Step 3: Set up Music Verification
1. Create an `HTTP Request` node (`Check music`) connected after `Config`. Set method to `HEAD`, URL to `{{ $('Config').first().json.musicUrl }}`, timeout to `15000`, and enable full response output.
2. Create a `Code` node (`Music guard`) connected after `Check music`. Paste JavaScript logic to inspect content-length headers and enforce file size caps (e.g., 5MB limit). Connect its output to the source router.

#### Step 4: Configure Data Sourcing and Normalization
1. Create an `If` node (`Source?`) checking `{{ $('Config').first().json.source === 'webhook' }}`.
2. **Webhook Branch (True):**
   - Create a `Code` node (`Webhook payload`) to normalize incoming POST bodies or fall back to `samplePayload`. Connect output to `No-show found?`.
3. **Sheet Branch (False):**
   - Create a Google Sheets node (`Read no-shows sheet`) configured to read rows from your target spreadsheet and tab.
   - Create a `Code` node (`Pick next no-show`) to locate the first row with an empty `Status` column. Connect output to `No-show found?`.
4. Create an `If` node (`No-show found?`) evaluating `{{ $json.found }}`.
   - True branch connects to `Build project JSON`.
   - False branch connects to a `Code` node (`Nothing to send`), which connects to `Respond ok`.

#### Step 5: Build Project and Validate
1. Create a `Code` node (`Build project JSON`) to construct the 3-scene vertical video payload (hook, recap highlights, CTA) along with metadata.
2. Create a Zvid node (`Validate project (free)`) configured with resource `render`, operation `validate`, using JSON source configuration.
3. Create a `Code` node (`Check validation`) to parse validation responses and throw detailed errors if validation fails.
4. Create an `If` node (`Dry run?`) evaluating `{{ $('Config').first().json.dryRun }}`.
   - **True branch:** Connect to a Zvid node (`Save draft to editor`) with resource `project`, operation `create`, error continuation enabled. Connect output to a `Code` node (`Dry run summary`), which connects to `Respond ok`.
   - **False branch:** Connect to a Zvid node (`Submit render`) with resource `render`, operation `create`, render type `video`, and `waitForCompletion` set to false.

#### Step 6: Configure Render Polling and Completion
1. Connect `Submit render` to a `Wait` node configured with duration `{{ $('Config').first().json.pollSeconds }}` seconds.
2. Connect `Wait` to a Zvid node (`Get render status`) configured with resource `render`, operation `get`, using `{{ $('Submit render').first().json.jobId }}`.
3. Connect `Get render status` to an `If` node (`Render finished?`) evaluating `{{ $json.state === 'completed' }}`.
   - **False branch:** Connect to a `Code` node (`Still rendering?`) that validates timeout limits and loops back to the `Wait` node.
   - **True branch:** Connect to a `Code` node (`Video URL`) to extract output URLs and prepare email template strings.

#### Step 7: Configure Distribution, Write-Back, and Review
1. Connect `Video URL` to an `If` node (`Send email?`) evaluating `{{ $('Config').first().json.sendEmail }}` and recipient address validity.
   - **True branch:** Connect to an Email Send node (`Send re-book email (SMTP)`) with error continuation enabled.
   - **False branch / SMTP output:** Connect to an `If` node (`Sheet row?`) checking `{{ $('Video URL').first().json.rowNumber !== null }}`.
2. **Sheet Row Branch (True):**
   - Connect to a Google Sheets node (`Mark row sent`) configured to update row numbers with status values and video URLs.
3. Connect both sheet update and non-sheet branches to a `Code` node (`Run summary`).
4. Connect `Run summary` outputs to:
   - `Respond ok` (Respond to Webhook node).
   - An `If` node (`Video ready to watch?`) checking live render execution state.
   - True branch from `Video ready to watch?` connects to an `HTTP Request` node (`▶ Watch video`) configured with method `GET`, URL pointing to `videoUrl`, and response format set to file (`data`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Comprehensive Zvid n8n workflow setup guide and repository instructions | [Zvid GitHub Workflow Guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/no-show-reengagement.md) |
| Obtain Zvid API keys for authentication credentials | [Zvid API Keys Console](https://app.zvid.io/api-keys) |
| Contact support or submit inquiries regarding integration assistance | [Zvid Contact Support](https://zvid.io/contact) |