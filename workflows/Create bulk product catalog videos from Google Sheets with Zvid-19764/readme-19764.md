Create bulk product catalog videos from Google Sheets with Zvid

https://n8nworkflows.xyz/workflows/create-bulk-product-catalog-videos-from-google-sheets-with-zvid-19764


# Create bulk product catalog videos from Google Sheets with Zvid

### 1. Workflow Overview

This workflow automates the process of reading product catalog data from Google Sheets, generating branded promotional videos in bulk using the Zvid API, writing the resulting video URLs back to the appropriate spreadsheet rows, and retrieving the finished MP4 files for quality assurance. 

The primary target use case is e-commerce catalog video generation, enabling marketing and content teams to produce branded social media videos (such as Instagram Reels) without manual video editing.

The workflow logic is grouped into seven functional blocks:
- **1.1 Input Reception & Configuration:** Manages workflow initialization (manual vs. scheduled triggers) and sets global brand and rendering parameters.
- **1.2 Product Queue Preparation:** Reads data from Google Sheets, filters unrendered items based on missing `VideoUrl` entries, validates schema requirements, checks for unique product names, and caps the batch size.
- **1.3 Project Construction & Validation:** Programmatically builds a multi-scene Zvid video template with per-product variable substitution, and validates the project against the Zvid API to calculate cost quotes.
- **1.4 Execution Mode Routing (Dry Run / Live Render):** Evaluates the `dryRun` flag to either generate an editable preview draft in the Zvid editor or submit a paid bulk rendering job.
- **1.5 Bulk Rendering & Polling:** Submits the batch render request, enters a polling loop with configurable delays and timeout limits, and tracks render status until completion.
- **1.6 Result Collection & Persistence:** Maps finished render jobs back to their corresponding source items and updates Google Sheets with the delivered video URLs.
- **1.7 Post-Processing & Media Review:** Evaluates completion metrics, handles error logging, and downloads the finalized MP4 binaries for visual inspection.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
**Overview:** This block initializes the workflow either on demand via a manual trigger or automatically on a recurring weekly schedule, loading global brand assets, design settings, and operational limits.
**Nodes Involved:**
- `Test manually`
- `Every Monday at 8am`
- `Config`

**Node Details:**
- **Test manually**
  - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` — Entry point for manual execution and debugging.
  - *Configuration:* Default parameters.
  - *Inputs / Outputs:* None (Input) → `Config` (Output).
  - *Edge Cases:* None.
- **Every Monday at 8am**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` — Cron-based entry point executing every Monday at 08:00.
  - *Configuration:* Interval configured for weeks (every 1 week, Mondays at 08:00).
  - *Inputs / Outputs:* None (Input) → `Config` (Output).
  - *Edge Cases:* Timezone dependencies rely on n8n instance settings.
- **Config**
  - *Type & Technical Role:* `n8n-nodes-base.set` — Emits a JSON payload containing global configuration variables (brand identity, design styling, music URLs, render constraints, and dry-run switches).
  - *Configuration:* Mode set to `raw`, outputting a JSON object with properties including `apiUrl`, `brandName`, `accentColor`, `maxProducts`, `dryRun`, `pollSeconds`, and `timeoutMinutes`.
  - *Inputs / Outputs:* `Test manually` or `Every Monday at 8am` (Inputs) → `Get catalog rows` (Output).
  - *Edge Cases:* Missing configuration keys will trigger downstream errors in code nodes or API requests.

---

#### Block 1.2: Product Queue Preparation
**Overview:** This block queries Google Sheets to read product rows, filters out items that already possess a `VideoUrl`, validates mandatory fields and name uniqueness, and enforces batch size limits.
**Nodes Involved:**
- `Get catalog rows`
- `Rows needing videos`
- `Products need videos?`
- `No pending products`

**Node Details:**
- **Get catalog rows**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` — Reads spreadsheet rows from a targeted document and sheet tab.
  - *Configuration:* Resource set to `sheet`, operation set to `read`. Spreadsheet ID and sheet name must be bound in the UI.
  - *Inputs / Outputs:* `Config` (Input) → `Rows needing videos` (Output).
  - *Edge Cases:* Authentication expiration, incorrect sheet naming, or empty sheets.
- **Rows needing videos**
  - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node that filters unrendered rows, validates URL formats, ensures product name uniqueness across the dataset, checks mandatory fields (`Price`, `ImageUrl1`, `Feature1-3`), and applies the `maxProducts` cap.
  - *Configuration:* Custom JS environment processing `$input.all()` against `Config` parameters. Throws an error if zero valid rows are found among pending entries.
  - *Inputs / Outputs:* `Get catalog rows` (Input) → `Products need videos?` (Output).
  - *Edge Cases:* Duplicate product names cause mapping ambiguities during write-back operations; invalid URLs cause exclusion.
- **Products need videos?**
  - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router checking whether the filtered row array contains items to process.
  - *Configuration:* Strict evaluation of `Array.isArray($json.rows) && $json.rows.length > 0`.
  - *Inputs / Outputs:* `Rows needing videos` (Input) → `Build project JSON` (True branch) or `No pending products` (False branch).
  - *Edge Cases:* Malformed row objects.
- **No pending products**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Terminal status node executed when all catalog rows are already processed.
  - *Configuration:* Outputs a structured JSON object indicating `status: 'nothing_to_render'`.
  - *Inputs / Outputs:* `Products need videos?` (False branch) (Input) → None (Terminal).
  - *Edge Cases:* None.

---

#### Block 1.3: Project Construction & Validation
**Overview:** This block compiles the design structure into a Zvid-compatible project payload with templated variable slots, builds item-specific variable bindings for each filtered product, and performs a free dry-run validation via the Zvid API.
**Nodes Involved:**
- `Build project JSON`
- `Validate project (free)`
- `Check validation`

**Node Details:**
- **Build project JSON**
  - *Type & Technical Role:* `n8n-nodes-base.code` — JavaScript execution node assembling the multi-scene Zvid project payload (Hero, Features, and CTA scenes with SVG backgrounds and typography rules) alongside per-row item variables and metadata.
  - *Configuration:* Custom script utilizing HTML escaping, money formatting helpers, font sizing algorithms, and schema builders.
  - *Inputs / Outputs:* `Products need videos?` (True branch) (Input) → `Validate project (free)` (Output).
  - *Edge Cases:* Overly long product names exceeding text-box constraints; invalid image URLs.
- **Validate project (free)**
  - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` — Calls the Zvid validation endpoint to test the compiled payload and first item variables without incurring credit costs.
  - *Configuration:* Source set to `json`, resource set to `render`, operation set to `validate`. Maps payload and validation variables via JSON stringification expressions.
  - *Credentials:* Requires a valid Zvid API credential.
  - *Inputs / Outputs:* `Build project JSON` (Input) → `Check validation` (Output).
  - *Edge Cases:* API authentication failures, invalid JSON schema structures, or insufficient plan permissions.
- **Check validation**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Parses API validation responses, calculates credit requirements (`creditsPerVideo * totalItems`), and extracts warnings or validation errors.
  - *Configuration:* JavaScript code block throwing detailed error messages if validation status codes differ from 200.
  - *Inputs / Outputs:* `Validate project (free)` (Input) → `Dry run?` (Output).
  - *Edge Cases:* API rate-limiting or unexpected response schema changes.

---

#### Block 1.4: Execution Mode Routing (Dry Run / Live Render)
**Overview:** This block checks the `dryRun` configuration flag to branch between generating an interactive editor draft preview or submitting a live bulk rendering job.
**Nodes Involved:**
- `Dry run?`
- `Save draft to editor`
- `Dry run summary`
- `Submit bulk render`

**Node Details:**
- **Dry run?**
  - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router evaluating the global `dryRun` setting.
  - *Configuration:* Loose type validation checking `={{ $('Config').first().json.dryRun }}` equals `true`.
  - *Inputs / Outputs:* `Check validation` (Input) → `Save draft to editor` (True branch) or `Submit bulk render` (False branch).
  - *Edge Cases:* Undefined configuration flags default based on n8n boolean evaluations.
- **Save draft to editor**
  - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` — Creates an editable draft project within the Zvid workspace for preview purposes.
  - *Configuration:* Resource set to `project`, operation set to `create`. Configured with retry logic (`maxTries: 3`, wait 5000ms) and `onError: continueRegularOutput` to ensure quotes are displayed even if draft creation fails.
  - *Credentials:* Zvid API credential.
  - *Inputs / Outputs:* `Dry run?` (True branch) (Input) → `Dry run summary` (Output).
  - *Edge Cases:* API timeouts or workspace project limits reached.
- **Dry run summary**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Generates an informational execution report containing estimated rendering costs, planned video counts, warnings, and editor preview links.
  - *Configuration:* JavaScript node safely extracting editor project identifiers and formatting status logs.
  - *Inputs / Outputs:* `Save draft to editor` (Input) → None (Terminal for dry run).
  - *Edge Cases:* Missing editor project IDs if draft creation failed gracefully.
- **Submit bulk render**
  - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` — Submits the complete multi-item payload to the Zvid bulk rendering queue.
  - *Configuration:* Resource set to `render`, operation set to `createBulk`, render type set to `video`. Configured with retry parameters (`maxTries: 3`, wait 5000ms).
  - *Credentials:* Zvid API credential.
  - *Inputs / Outputs:* `Dry run?` (False branch) (Input) → `Check batch accepted` (Output).
  - *Edge Cases:* API rejections due to account credit exhaustion or payload validation failures.

---

#### Block 1.5: Bulk Rendering & Polling
**Overview:** This block verifies batch acceptance, initiates a polling loop with configurable intervals and timeout limits, and checks whether background rendering tasks have completed.
**Nodes Involved:**
- `Check batch accepted`
- `Wait`
- `Get batch status`
- `Batch finished?`
- `Still rendering?`

**Node Details:**
- **Check batch accepted**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Parses the bulk submission response, extracts the `bulkId`, builds job index mapping lookups, and logs item-level submission errors.
  - *Configuration:* JavaScript execution node. Throws an error if `bulkId` is absent.
  - *Inputs / Outputs:* `Submit bulk render` (Input) → `Wait` (Output).
  - *Edge Cases:* Malformed API responses.
- **Wait**
  - *Type & Technical Role:* `n8n-nodes-base.wait` — Pauses workflow execution for a defined duration before polling status updates.
  - *Configuration:* Unit set to seconds, amount dynamically read from `={{ $('Config').first().json.pollSeconds }}`. Webhook ID configured.
  - *Inputs / Outputs:* `Check batch accepted` or `Still rendering?` (Inputs) → `Get batch status` (Output).
  - *Edge Cases:* Execution timeouts on long-running batches if n8n instance limits are exceeded.
- **Get batch status**
  - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` — Queries the Zvid API for current batch rendering progress and individual job states.
  - *Configuration:* Resource set to `render`, operation set to `getBulk`. Bulk ID bound via expression.
  - *Credentials:* Zvid API credential.
  - *Inputs / Outputs:* `Wait` (Input) → `Batch finished?` (Output).
  - *Edge Cases:* Network timeouts or transient API unavailability.
- **Batch finished?**
  - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router checking if the bulk render status is no longer processing.
  - *Configuration:* Evaluates `={{ $json.bulk.status !== 'processing' }}`.
  - *Inputs / Outputs:* `Get batch status` (Input) → `Collect video links` (True branch) or `Still rendering?` (False branch).
  - *Edge Cases:* Unexpected status strings from the API.
- **Still rendering?**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Evaluates failure states or enforces maximum timeout limits across polling cycles.
  - *Configuration:* JavaScript calculating maximum allowed poll iterations based on `timeoutMinutes` and `pollSeconds`. Throws an explicit timeout error if limits are breached.
  - *Inputs / Outputs:* `Batch finished?` (False branch) (Input) → `Wait` (Loop output).
  - *Edge Cases:* Premature timeouts configured on heavy catalog sizes.

---

#### Block 1.6: Result Collection & Persistence
**Overview:** This block maps completed render outputs back to their source catalog items, filters out failed jobs, and updates Google Sheets with the delivered video URLs using product name matching.
**Nodes Involved:**
- `Collect video links`
- `Write links to sheet`

**Node Details:**
- **Collect video links**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Maps finished job output URLs to original product keys and formats items for spreadsheet persistence.
  - *Configuration:* JavaScript node throwing errors if all jobs failed or if no output URLs were produced.
  - *Inputs / Outputs:* `Batch finished?` (True branch) (Input) → `Write links to sheet` and `Video ready to watch?` (Outputs).
  - *Edge Cases:* Partial batch failures where some items complete while others fail.
- **Write links to sheet**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` — Updates spreadsheet rows with the newly generated video URLs.
  - *Configuration:* Resource set to `sheet`, operation set to `update`. Matching column configured to `Product`. Mapping mode set to `defineBelow` mapping `Product` and `VideoUrl`. Configured with `alwaysOutputData: true`.
  - *Inputs / Outputs:* `Collect video links` (Input) → `Run summary` (Output).
  - *Edge Cases:* Spreadsheet column renaming breaking the `Product` lookup match; authentication expiry.

---

#### Block 1.7: Post-Processing & Media Review
**Overview:** This block compiles a comprehensive final execution summary, evaluates whether video assets are ready for review, and downloads completed MP4 files as binary attachments for inspection.
**Nodes Involved:**
- `Run summary`
- `Video ready to watch?`
- `▶ Watch video`

**Node Details:**
- **Run summary**
  - *Type & Technical Role:* `n8n-nodes-base.code` — Aggregates final metrics including delivered video counts, failure logs, credits spent, and spreadsheet update statuses.
  - *Configuration:* JavaScript execution node verifying that sheet writes successfully persisted data.
  - *Inputs / Outputs:* `Write links to sheet` (Input) → None (Terminal).
  - *Edge Cases:* Silent spreadsheet write failures caught and reported via validation logic.
- **Video ready to watch?**
  - *Type & Technical Role:* `n8n-nodes-base.if` — Conditional router checking if execution was live and generated a valid HTTP URL.
  - *Configuration:* Strict evaluation ensuring `!$json.dryRun && /^https?:\/\//i.test(String($json.VideoUrl))`.
  - *Inputs / Outputs:* `Collect video links` (Input) → `▶ Watch video` (True branch) or None (False branch).
  - *Edge Cases:* Malformed URL strings.
- **▶ Watch video**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` — HTTP GET request downloading the rendered MP4 video file into n8n's binary data stream for review.
  - *Configuration:* Method GET, response format set to `file`, outputting to binary property `data`. Configured with retry parameters (`maxTries: 3`, wait 5000ms) and `onError: continueRegularOutput`.
  - *Inputs / Outputs:* `Video ready to watch?` (True branch) (Input) → None (Terminal).
  - *Edge Cases:* Network timeouts when downloading large video files.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Test manually` | `n8n-nodes-base.manualTrigger` | Entry point for manual execution and testing | None | `Config` | 1. Choose settings and an entry point |
| `Every Monday at 8am` | `n8n-nodes-base.scheduleTrigger` | Recurring weekly trigger executing every Monday at 8 AM | None | `Config` | 1. Choose settings and an entry point |
| `Config` | `n8n-nodes-base.set` | Defines global branding, design parameters, and operational rules | `Test manually`, `Every Monday at 8am` | `Get catalog rows` | 1. Choose settings and an entry point |
| `Get catalog rows` | `n8n-nodes-base.googleSheets` | Reads product catalog rows from Google Sheets | `Config` | `Rows needing videos` | 2. Prepare the product queue |
| `Rows needing videos` | `n8n-nodes-base.code` | Filters unrendered rows, validates required fields, enforces uniqueness and batch limits | `Get catalog rows` | `Products need videos?` | 2. Prepare the product queue |
| `Products need videos?` | `n8n-nodes-base.if` | Routes execution based on whether pending rows require rendering | `Rows needing videos` | `Build project JSON`, `No pending products` | 2. Prepare the product queue |
| `No pending products` | `n8n-nodes-base.code` | Terminal node executed when all catalog rows already possess videos | `Products need videos?` | None | 2. Prepare the product queue |
| `Build project JSON` | `n8n-nodes-base.code` | Compiles the multi-scene Zvid project template and item variable sets | `Products need videos?` | `Validate project (free)` | 3. Build and validate the design |
| `Validate project (free)` | `@zvid/n8n-nodes-zvid.zvid` | Validates the Zvid project payload without incurring render costs | `Build project JSON` | `Check validation` | 3. Build and validate the design |
| `Check validation` | `n8n-nodes-base.code` | Parses validation results and calculates total credit quotes | `Validate project (free)` | `Dry run?` | 3. Build and validate the design |
| `Dry run?` | `n8n-nodes-base.if` | Routes execution between preview draft creation and live bulk rendering | `Check validation` | `Save draft to editor`, `Submit bulk render` | 4. Choose an optional editor preview |
| `Save draft to editor` | `@zvid/n8n-nodes-zvid.zvid` | Creates an editable project draft in the Zvid editor for review | `Dry run?` | `Dry run summary` | 4. Choose an optional editor preview |
| `Dry run summary` | `n8n-nodes-base.code` | Outputs dry-run statistics, credit estimates, and editor preview links | `Save draft to editor` | None | 4. Choose an optional editor preview |
| `Submit bulk render` | `@zvid/n8n-nodes-zvid.zvid` | Submits the batch rendering job to the Zvid API | `Dry run?` | `Check batch accepted` | 5. Render and wait for completion |
| `Check batch accepted` | `n8n-nodes-base.code` | Parses bulk submission response and maps job identifiers | `Submit bulk render` | `Wait` | 5. Render and wait for completion |
| `Wait` | `n8n-nodes-base.wait` | Pauses execution between polling intervals | `Check batch accepted`, `Still rendering?` | `Get batch status` | 5. Render and wait for completion |
| `Get batch status` | `@zvid/n8n-nodes-zvid.zvid` | Queries batch progress and individual render job statuses | `Wait` | `Batch finished?` | 5. Render and wait for completion |
| `Batch finished?` | `n8n-nodes-base.if` | Evaluates whether background bulk rendering has completed | `Get batch status` | `Collect video links`, `Still rendering?` | 5. Render and wait for completion |
| `Still rendering?` | `n8n-nodes-base.code` | Manages polling timeouts and error states for unfinished batches | `Batch finished?` | `Wait` | 5. Render and wait for completion |
| `Collect video links` | `n8n-nodes-base.code` | Maps completed render jobs back to source product rows | `Batch finished?` | `Write links to sheet`, `Video ready to watch?` | 6. Save the completed result |
| `Write links to sheet` | `n8n-nodes-base.googleSheets` | Updates Google Sheets product rows with the delivered video URLs | `Collect video links` | `Run summary` | 6. Save the completed result |
| `Run summary` | `n8n-nodes-base.code` | Generates a final execution report and error log | `Write links to sheet` | None | 6. Save the completed result |
| `Video ready to watch?` | `n8n-nodes-base.if` | Verifies whether output URLs are valid for media downloading | `Collect video links` | `▶ Watch video`, None | 7. Review the finished media |
| `▶ Watch video` | `n8n-nodes-base.httpRequest` | Downloads completed MP4 video files into binary data streams | `Video ready to watch?` | None | 7. Review the finished media |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Install Community Node:** Navigate to **Settings → Community nodes** in your n8n instance and install `@zvid/n8n-nodes-zvid` (workspace administrator permissions may be required).
2. **Create Credentials:** Set up a **Zvid API** credential using an API key generated from [Zvid API Keys](https://app.zvid.io/api-keys) and Base URL `https://api.zvid.io`. Set up a **Google Sheets OAuth2** credential.
3. **Create Triggers:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) configured to trigger every week on Mondays at 08:00.
4. **Configure Global Settings:**
   - Add a **Set** node named `Config` (`n8n-nodes-base.set`). Set mode to `raw` and provide the JSON configuration object containing brand details, styling variables, `maxProducts: 5`, `dryRun: false`, `pollSeconds: 10`, and `timeoutMinutes: 20`.
   - Connect both `Test manually` and `Every Monday at 8am` to `Config`.
5. **Set Up Google Sheets Reader:**
   - Add a **Google Sheets** node named `Get catalog rows` (`n8n-nodes-base.googleSheets`). Set resource to `sheet` and operation to `read`. Select your target document ID and sheet name.
   - Connect `Config` to `Get catalog rows`.
6. **Filter and Validate Pending Rows:**
   - Add a **Code** node named `Rows needing videos` (`n8n-nodes-base.code`). Populate it with the JavaScript logic that filters rows with empty `VideoUrl` values, validates required column fields (`Product`, `Price`, `ImageUrl1`, `Feature1-3`), checks product name uniqueness, and slices the array to `maxProducts`.
   - Connect `Get catalog rows` to `Rows needing videos`.
   - Add an **If** node named `Products need videos?` (`n8n-nodes-base.if`) checking if `rows.length > 0`. Connect `Rows needing videos` to it.
   - Add a **Code** node named `No pending products` (`n8n-nodes-base.code`) to handle the false branch when no renders are needed.
7. **Build Project Payload:**
   - Add a **Code** node named `Build project JSON` (`n8n-nodes-base.code`) connected to the true branch of `Products need videos?`. Populate it with the template compilation logic generating the 3-scene structure (Hero, Features, CTA) and item variables.
8. **Validate Project:**
   - Add a **Zvid** node named `Validate project (free)` (`@zvid/n8n-nodes-zvid.zvid`). Set resource to `render`, operation to `validate`, and configure JSON payload and variable stringification expressions. Select your Zvid credential.
   - Add a **Code** node named `Check validation` (`n8n-nodes-base.code`) to parse validation output and calculate credit estimates. Connect `Build project JSON` → `Validate project (free)` → `Check validation`.
9. **Implement Dry Run Branching:**
   - Add an **If** node named `Dry run?` (`n8n-nodes-base.if`) connected to `Check validation`, checking `={{ $('Config').first().json.dryRun }}`.
   - **True Branch:** Add a **Zvid** node named `Save draft to editor` (`@zvid/n8n-nodes-zvid.zvid`) (Resource: `project`, Operation: `create`, with error handling configured to continue execution). Connect it to a **Code** node named `Dry run summary`.
   - **False Branch:** Add a **Zvid** node named `Submit bulk render` (`@zvid/n8n-nodes-zvid.zvid`) (Resource: `render`, operation: `createBulk`, render type: `video`).
10. **Polling & Status Tracking:**
    - Add a **Code** node named `Check batch accepted` (`n8n-nodes-base.code`) connected to `Submit bulk render`.
    - Add a **Wait** node (`n8n-nodes-base.wait`) configured with unit `seconds` and amount set to `={{ $('Config').first().json.pollSeconds }}`.
    - Add a **Zvid** node named `Get batch status` (`@zvid/n8n-nodes-zvid.zvid`) (Resource: `render`, operation: `getBulk`).
    - Add an **If** node named `Batch finished?` (`n8n-nodes-base.if`) checking `={{ $json.bulk.status !== 'processing' }}`.
    - Add a **Code** node named `Still rendering?` (`n8n-nodes-base.code`) to manage timeout limits, connecting its output back to the `Wait` node to form a polling loop.
    - Connect: `Check batch accepted` → `Wait` → `Get batch status` → `Batch finished?`. The true branch connects to result collection; the false branch connects to `Still rendering?`.
11. **Result Collection & Sheet Update:**
    - Add a **Code** node named `Collect video links` (`n8n-nodes-base.code`) connected to the true branch of `Batch finished?`.
    - Add a **Google Sheets** node named `Write links to sheet` (`n8n-nodes-base.googleSheets`) (Resource: `sheet`, operation: `update`). Configure matching column to `Product` and map `Product` and `VideoUrl`. Enable `Always Output Data`.
    - Connect `Collect video links` to `Write links to sheet`.
12. **Summarization & Media Download:**
    - Add a **Code** node named `Run summary` (`n8n-nodes-base.code`) connected to `Write links to sheet`.
    - Add an **If** node named `Video ready to watch?` (`n8n-nodes-base.if`) connected to `Collect video links`, evaluating URL validity.
    - Add an **HTTP Request** node named `▶ Watch video` (`n8n-nodes-base.httpRequest`) configured to GET the `VideoUrl` with response format set to `file` (output property `data`). Connect `Video ready to watch?` true branch to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zvid Community Node Installation Guide | [Zvid Integration on n8n](https://n8n.io/integrations/zvid/) |
| Zvid API Key Generation | [Zvid API Keys Management](https://app.zvid.io/api-keys) |
| Sample Audio Asset | [Pixabay Audio CDN](https://cdn.pixabay.com/audio/2025/04/21/audio_ed6f0ed574.mp3) |
| Sample Imagery Assets | [Pexels Stock Photography](https://www.pexels.com/) |