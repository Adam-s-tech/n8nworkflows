Track company career-page jobs with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/track-company-career-page-jobs-with-apify-and-google-sheets-20051


# Track company career-page jobs with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of job listings from a single company career page or Applicant Tracking System (ATS) board using the Apify Actor **FalconScrape Company Career Page Scraper** (`HiJYKx437rQqU6YMb`). It writes, updates, and tracks these listings in a dedicated Google Sheets spreadsheet. The workflow ensures data integrity through transactional locking, schema validation, and audit logs.

#### Functional Logical Blocks:
- **1.1 Input Reception & Validation:** Manually triggers the workflow, defines target parameters (`companyUrl`, `maxJobs`, `maxChargeUsd`, `spreadsheetId`, `resumeRunId`), and performs strict regex validation.
- **1.2 Destination Preparation & Locking:** Inspects the Google Sheets destination, creates required tabs (`Career Current`, `Career History`, `Career Runs`) if missing, and implements a temporary lock tab (`_CAREER_WORKFLOW_LOCK`) to prevent concurrent execution races.
- **1.3 Scraper Initialization / Recovery:** Evaluates whether to start a new Apify Actor run with budget caps or resume an existing run using a provided `resumeRunId`.
- **1.4 Execution Monitoring & Polling:** Checkpoints the run ID into the spreadsheet, polls Apify until the Actor successfully completes, and retrieves input parameters to guarantee configuration consistency.
- **1.5 Data Extraction & Normalization:** Fetches the complete dataset of scraped job items from Apify, sanitizes fields, normalizes domain/ID keys, and plans database write operations (classifying listings as baseline, new, updated, or replayed).
- **1.6 Persistence & Cleanup:** Batch-updates the Google Sheets destination with current jobs, historical observations, and run ledger entries, removes the temporary lock tab, and returns a final execution summary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Initializes execution parameters and enforces strict validation rules on URLs, budget caps, job limits, and spreadsheet identifiers.
- **Nodes Involved:** `Run manually`, `Configure search`, `Validate configuration`

##### Node Details:
- **Run manually**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Entry point for manual executions.
  - **Configuration Choices:** Default setup with no parameters.
  - **Input / Output:** Output connects to `Configure search`.
- **Configure search**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Injects parameters for target search and destination.
  - **Configuration Choices:** Assigns values for `companyUrl` (e.g., `https://www.figma.com`), `maxJobs` (20), `maxChargeUsd` (0.08), `spreadsheetId` (empty placeholder), and `resumeRunId` (empty placeholder).
  - **Input / Output:** Input from `Run manually`; output to `Validate configuration`.
- **Validate configuration**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Validates format and constraints using JavaScript.
  - **Configuration Choices:** Validates public HTTPS URLs (blocking localhost/IPs), verifies 20–100 character alphanumeric spreadsheet IDs, checks `maxJobs` (1–100) and `maxChargeUsd` ($0.01–$1.00).
  - **Edge Cases / Potential Failures:** Throws errors if URLs contain query strings, fragments, or lack proper public domains.

---

#### 2.2 Destination Preparation & Locking
- **Overview:** Connects to Google Sheets via API, checks existing spreadsheet tabs, provisions missing schema tabs, and inserts a lock sheet to prevent race conditions.
- **Nodes Involved:** `Read destination`, `Prepare tabs and lock`, `Acquire destination lock`, `Read existing rows`, `Validate existing tables`

##### Node Details:
- **Read destination**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Calls Google Sheets API to read spreadsheet metadata.
  - **Configuration Choices:** GET request to `https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}?fields=spreadsheetId,sheets(properties)`. Uses `googleSheetsOAuth2Api` credentials with retry configuration (up to 3 tries).
  - **Input / Output:** Input from `Validate configuration`; output to `Prepare tabs and lock`.
- **Prepare tabs and lock**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Evaluates tab metadata and constructs batch update payloads for schema creation and locking.
  - **Configuration Choices:** Checks if `_CAREER_WORKFLOW_LOCK` exists (fails if locked). Prepares requests to add the lock tab and missing business tabs (`Career Current`, `Career History`, `Career Runs`).
  - **Input / Output:** Input from `Read destination`; output to `Acquire destination lock`.
- **Acquire destination lock**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Executes batch updates to create the lock sheet in Google Sheets.
  - **Configuration Choices:** POST request to `:batchUpdate`. Authentication via `googleSheetsOAuth2Api`. `retryOnFail` is set to `false` to avoid unintended duplication of lock mutations.
  - **Input / Output:** Input from `Prepare tabs and lock`; output to `Read existing rows`.
- **Read existing rows**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Retrieves existing tabular data from managed sheets.
  - **Configuration Choices:** GET request using `values:batchGet` for ranges `'Career Current'!A1:R5002`, `'Career Runs'!A1:R5002`, and `'Career History'!A1:M5002` with `UNFORMATTED_VALUE`.
  - **Input / Output:** Input from `Acquire destination lock`; output to `Validate existing tables`.
- **Validate existing tables**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Validates sheet headers, row limits (max 5,000), and key uniqueness.
  - **Configuration Choices:** Ensures existing records match schema definitions and belong to the correct company URL.
  - **Input / Output:** Input from `Read existing rows`; output to `Resume an existing run?`.

---

#### 2.3 Scraper Initialization / Recovery
- **Overview:** Decides whether to resume an interrupted Apify run or dispatch a new bounded scraping job.
- **Nodes Involved:** `Resume an existing run?`, `Load recovery run`, `Start bounded career-page run`

##### Node Details:
- **Resume an existing run?**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Conditional router based on `resumeRunId`.
  - **Configuration Choices:** Checks if `resumeRunId` is non-empty.
  - **Input / Output:** Input from `Validate existing tables`; outputs true to `Load recovery run` and false to `Start bounded career-page run`.
- **Load recovery run**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches metadata for a previously initialized Apify run.
  - **Configuration Choices:** GET request to `https://api.apify.com/v2/actor-runs/{resumeRunId}` using `apifyApi` credentials.
  - **Input / Output:** Input from `Resume an existing run?` (true branch); output to `Capture run`.
- **Start bounded career-page run**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Triggers a new Apify Actor run with configuration limits.
  - **Configuration Choices:** POST request to `https://api.apify.com/v2/acts/HiJYKx437rQqU6YMb/runs?timeout=240&build=0.0.28&maxTotalChargeUsd={maxChargeUsd}` using `apifyApi` credentials. Sends JSON body with `startUrls` and `maxJobsPerCompany`.
  - **Input / Output:** Input from `Resume an existing run?` (false branch); output to `Capture run`.

---

#### 2.4 Execution Monitoring & Polling
- **Overview:** Captures run IDs, checks configuration integrity against original run payloads, logs processing checkpoints to Google Sheets, and polls Apify until completion.
- **Nodes Involved:** `Capture run`, `Read original run input`, `Prepare run checkpoint`, `Save run checkpoint`, `Wait for Actor status`, `Inspect Actor status`, `Actor succeeded?`

##### Node Details:
- **Capture run**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Extracts and validates the Apify Actor run object.
  - **Input / Output:** Input from `Load recovery run` or `Start bounded career-page run`; output to `Read original run input`.
- **Read original run input**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches the original configuration JSON from Apify key-value stores.
  - **Configuration Choices:** GET request to `https://api.apify.com/v2/key-value-stores/{defaultKeyValueStoreId}/records/INPUT` using `apifyApi` credentials.
  - **Input / Output:** Input from `Capture run`; output to `Prepare run checkpoint`.
- **Prepare run checkpoint**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Prepares spreadsheet row payloads to log the run status as `PROCESSING`.
  - **Input / Output:** Input from `Read original run input`; output to `Save run checkpoint`.
- **Save run checkpoint**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Writes the `PROCESSING` checkpoint to the `Career Runs` tab.
  - **Configuration Choices:** POST request to `:batchUpdate` using `googleSheetsOAuth2Api`.
  - **Input / Output:** Input from `Prepare run checkpoint`; output to `Wait for Actor status`.
- **Wait for Actor status**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Polls Apify for run status updates with a 30-second timeout buffer.
  - **Configuration Choices:** GET request to `https://api.apify.com/v2/actor-runs/{id}?waitForFinish=30` using `apifyApi` credentials.
  - **Input / Output:** Input from `Save run checkpoint` or `Actor succeeded?` (false branch); output to `Inspect Actor status`.
- **Inspect Actor status**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Inspects the Actor execution status and enforces a maximum loop limit (fails after 7 polling iterations without completion).
  - **Input / Output:** Input from `Wait for Actor status`; output to `Actor succeeded?`.
- **Actor succeeded?**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Evaluates whether the Apify Actor run finished with `SUCCEEDED` status.
  - **Input / Output:** Input from `Inspect Actor status`; outputs true to `Fetch complete bounded dataset` and false back to `Wait for Actor status`.

---

#### 2.5 Data Extraction & Normalization
- **Overview:** Retrieves scraped items from the Apify dataset, sanitizes records, normalizes job identities, and prepares tabular upserts.
- **Nodes Involved:** `Fetch complete bounded dataset`, `Plan hiring observations`

##### Node Details:
- **Fetch complete bounded dataset**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches all scraped job items from the Apify dataset storage.
  - **Configuration Choices:** GET request to `https://api.apify.com/v2/datasets/{defaultDatasetId}/items?format=json&limit=1001` using `apifyApi` credentials.
  - **Input / Output:** Input from `Actor succeeded?` (true branch); output to `Plan hiring observations`.
- **Plan hiring observations**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Normalizes items, identifies schema compliance, deduplicates keys, and plans database write structures (`BASELINE`, `NEW`, `REFRESHED`, `REPLAYED`).
  - **Input / Output:** Input from `Fetch complete bounded dataset`; output to `Write jobs history and run log`.

---

#### 2.6 Persistence & Cleanup
- **Overview:** Commits planned job records, history logs, and final run statuses to Google Sheets in a single batch operation, releases the temporary lock, and outputs execution metrics.
- **Nodes Involved:** `Write jobs history and run log`, `Prepare lock release`, `Release destination lock`, `Run summary`

##### Node Details:
- **Write jobs history and run log**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Batch updates `Career Current`, `Career History`, and `Career Runs` tabs.
  - **Configuration Choices:** POST request to `:batchUpdate` using `googleSheetsOAuth2Api`.
  - **Input / Output:** Input from `Plan hiring observations`; output to `Prepare lock release`.
- **Prepare lock release**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Extracts the sheet ID of the temporary lock tab for deletion.
  - **Input / Output:** Input from `Write jobs history and run log`; output to `Release destination lock`.
- **Release destination lock**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Deletes the temporary lock tab (`_CAREER_WORKFLOW_LOCK`) from the spreadsheet.
  - **Configuration Choices:** POST request to `:batchUpdate` with `{deleteSheet}` request body. `retryOnFail` is set to `false`.
  - **Input / Output:** Input from `Prepare lock release`; output to `Run summary`.
- **Run summary**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Compiles final metrics (counts for baseline, new, refreshed, rejected, and replayed jobs) alongside a direct link to the Google Sheet.
  - **Input / Output:** Input from `Release destination lock`; terminal workflow output.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | `n8n-nodes-base.manualTrigger` | Entry point for manual execution | None | Configure search | ## Track public company career-page jobs... |
| Configure search | `n8n-nodes-base.set` | Assigns configuration variables | Run manually | Validate configuration | ## 1. Choose one public career page... |
| Validate configuration | `n8n-nodes-base.code` | Validates search and destination inputs | Configure search | Read destination | ## 1. Choose one public career page... |
| Read destination | `n8n-nodes-base.httpRequest` | Reads spreadsheet metadata | Validate configuration | Prepare tabs and lock | ## 2. Prepare three managed tabs... |
| Prepare tabs and lock | `n8n-nodes-base.code` | Prepares tab creation and lock requests | Read destination | Acquire destination lock | ## 2. Prepare three managed tabs... |
| Acquire destination lock | `n8n-nodes-base.httpRequest` | Applies destination lock sheet | Prepare tabs and lock | Read existing rows | ## 2. Prepare three managed tabs... |
| Read existing rows | `n8n-nodes-base.httpRequest` | Reads managed tables data | Acquire destination lock | Validate existing tables | ## 2. Prepare three managed tabs... |
| Validate existing tables | `n8n-nodes-base.code` | Validates schema and row constraints | Read existing rows | Resume an existing run? | ## 2. Prepare three managed tabs... |
| Resume an existing run? | `n8n-nodes-base.if` | Routes to recovery or new run | Validate existing tables | Load recovery run, Start bounded career-page run | ## 3. Start or resume the same company... |
| Load recovery run | `n8n-nodes-base.httpRequest` | Fetches recovery run details | Resume an existing run? | Capture run | ## 3. Start or resume the same company... |
| Start bounded career-page run | `n8n-nodes-base.httpRequest` | Triggers new Apify Actor run | Resume an existing run? | Capture run | ## 3. Start or resume the same company... |
| Capture run | `n8n-nodes-base.code` | Captures run metadata | Load recovery run, Start bounded career-page run | Read original run input | ## 4. Checkpoint before polling... |
| Read original run input | `n8n-nodes-base.httpRequest` | Retrieves original run inputs | Capture run | Prepare run checkpoint | ## 4. Checkpoint before polling... |
| Prepare run checkpoint | `n8n-nodes-base.code` | Prepares run logging payload | Read original run input | Save run checkpoint | ## 4. Checkpoint before polling... |
| Save run checkpoint | `n8n-nodes-base.httpRequest` | Logs run start in spreadsheet | Prepare run checkpoint | Wait for Actor status | ## 4. Checkpoint before polling... |
| Wait for Actor status | `n8n-nodes-base.httpRequest` | Polls Apify Actor status | Save run checkpoint, Actor succeeded? | Inspect Actor status | ## 4. Checkpoint before polling... |
| Inspect Actor status | `n8n-nodes-base.code` | Evaluates Actor status and limits | Wait for Actor status | Actor succeeded? | ## 4. Checkpoint before polling... |
| Actor succeeded? | `n8n-nodes-base.if` | Checks if Actor run succeeded | Inspect Actor status | Fetch complete bounded dataset, Wait for Actor status | ## 4. Checkpoint before polling... |
| Fetch complete bounded dataset | `n8n-nodes-base.httpRequest` | Retrieves scraped job dataset items | Actor succeeded? | Plan hiring observations | ## 5. Record job observations... |
| Plan hiring observations | `n8n-nodes-base.code` | Normalizes and plans upserts | Fetch complete bounded dataset | Write jobs history and run log | ## 5. Record job observations... |
| Write jobs history and run log | `n8n-nodes-base.httpRequest` | Writes current jobs, history, and log | Plan hiring observations | Prepare lock release | ## 6. Review and release the lock... |
| Prepare lock release | `n8n-nodes-base.code` | Prepares lock sheet deletion | Write jobs history and run log | Release destination lock | ## 6. Review and release the lock... |
| Release destination lock | `n8n-nodes-base.httpRequest` | Deletes temporary lock sheet | Prepare lock release | Run summary | ## 6. Review and release the lock... |
| Run summary | `n8n-nodes-base.code` | Outputs final statistics and spreadsheet link | Release destination lock | None | ## 6. Review and release the lock... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Create a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Configure Search Parameters:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Configure search`.
   - Add string/number assignments: `companyUrl` (`https://www.figma.com`), `maxJobs` (`20`), `maxChargeUsd` (`0.08`), `spreadsheetId` (`""`), and `resumeRunId` (`""`).
3. **Add Validation Code Node:**
   - Add a **Code** node named `Validate configuration` with the validation JavaScript logic (checking URL format, spreadsheet ID, limits, etc.).
4. **Setup Google Sheets Metadata Fetch:**
   - Add an **HTTP Request** node named `Read destination`.
   - Configure method `GET`, URL `={{ 'https://sheets.googleapis.com/v4/spreadsheets/' + $('Validate configuration').first().json.spreadsheetId + '?fields=spreadsheetId,sheets(properties)' }}`, and select **Google Sheets OAuth2 API** credentials. Enable retry on fail (3 attempts).
5. **Prepare Lock and Tabs:**
   - Add a **Code** node named `Prepare tabs and lock` to generate the sheet creation and locking payload.
   - Add an **HTTP Request** node named `Acquire destination lock` (POST `:batchUpdate`, no retry on fail).
6. **Read Existing Tables:**
   - Add an **HTTP Request** node named `Read existing rows` (`GET` batch get for ranges `Career Current`, `Career Runs`, `Career History`).
   - Add a **Code** node named `Validate existing tables` to parse and validate tabular rows and uniqueness constraints.
7. **Configure Run Routing:**
   - Add an **If** node named `Resume an existing run?` evaluating `={{ $('Validate configuration').first().json.resumeRunId !== '' }}`.
   - Branch True: Add **HTTP Request** (`Load recovery run`) calling `https://api.apify.com/v2/actor-runs/{resumeRunId}` using **Apify API** credentials.
   - Branch False: Add **HTTP Request** (`Start bounded career-page run`) calling `https://api.apify.com/v2/acts/HiJYKx437rQqU6YMb/runs?timeout=240&build=0.0.28&maxTotalChargeUsd={maxChargeUsd}` via POST with Apify credentials.
8. **Checkpoint and Polling Loop:**
   - Add **Code** node (`Capture run`), **HTTP Request** (`Read original run input`), **Code** node (`Prepare run checkpoint`), and **HTTP Request** (`Save run checkpoint`).
   - Add **HTTP Request** (`Wait for Actor status`) calling `https://api.apify.com/v2/actor-runs/{id}?waitForFinish=30`.
   - Add **Code** node (`Inspect Actor status`) and **If** node (`Actor succeeded?`). Connect false loop back to `Wait for Actor status`.
9. **Data Processing and Persistence:**
   - From successful branch, add **HTTP Request** (`Fetch complete bounded dataset`) calling `https://api.apify.com/v2/datasets/{defaultDatasetId}/items?format=json&limit=1001`.
   - Add **Code** node (`Plan hiring observations`).
   - Add **HTTP Request** (`Write jobs history and run log`) to commit batch updates.
10. **Cleanup and Summary:**
    - Add **Code** node (`Prepare lock release`) and **HTTP Request** (`Release destination lock`) to delete the lock sheet (`_CAREER_WORKFLOW_LOCK`).
    - Add **Code** node (`Run summary`) to output execution metrics and the Google Sheets URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| FalconScrape Company Career Page Scraper | [Apify Actor Link](https://apify.com/piotrv1001/company-career-page-scraper) |
| Template Disclaimer | Free template; Apify Actor runs and n8n hosting may incur usage charges. |