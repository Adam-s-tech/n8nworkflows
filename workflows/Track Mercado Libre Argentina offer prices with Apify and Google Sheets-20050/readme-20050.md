Track Mercado Libre Argentina offer prices with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/track-mercado-libre-argentina-offer-prices-with-apify-and-google-sheets-20050


# Track Mercado Libre Argentina offer prices with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and tracking of Mercado Libre Argentina offer prices using Apify and logs the data into Google Sheets. It supports running new scrapes or resuming existing Apify runs, performing data validation, historical price comparisons, and conflict resolution before writing records.

The logical execution follows these functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution and validates user parameters (search keywords, limits, credentials, and spreadsheet IDs).
- **1.2 Destination Preparation & Locking:** Inspects the Google Sheets target, checks sheet capacities, provisions missing tabs, and applies a temporary execution lock sheet to prevent race conditions.
- **1.3 Execution Branching (New vs. Recovery):** Determines whether to initiate a new Apify scraping run or retrieve and validate an existing run using a provided `resumeRunId`.
- **1.4 Polling & Status Verification:** Periodically checks the status of the Apify scraping job until completion, updating a run checkpoint ledger in Google Sheets.
- **1.5 Data Processing & Comparison:** Retrieves the completed dataset from Apify, filters out invalid or malformed listings, and compares pricing data against existing baseline and history records.
- **1.6 Finalization & Lock Release:** Persists updated pricing data, observation history, and execution summaries to Google Sheets, then removes the temporary lock tab and outputs a final run summary.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Sets up initial search criteria and validates user configurations prior to accessing external services.
- **Nodes Involved:** `Run manually`, `Configure search`, `Validate configuration`
- **Node Details:**
  - **Run manually** (`n8n-nodes-base.manualTrigger`)
    - *Role:* Entry point for manual executions.
    - *Configuration:* Default manual trigger settings.
    - *Connections:* Output connects to `Configure search`.
  - **Configure search** (`n8n-nodes-base.set`)
    - *Role:* Assigns default parameters such as search terms (`iphone 15`), listing limits, and spreadsheet IDs.
    - *Configuration:* Sets payload fields: `search`, `maxItems` (20), `maxChargeUsd` (0.09), `spreadsheetId`, and `resumeRunId`.
    - *Connections:* Input from `Run manually`, output to `Validate configuration`.
  - **Validate configuration** (`n8n-nodes-base.code`)
    - *Role:* Executes core validation logic on configuration parameters, ensuring search strings, item limits, charge caps, and Google Sheets identifiers conform to format rules.
    - *Configuration:* JavaScript execution block containing the core profile rules and regex validators.
    - *Connections:* Input from `Configure search`, output to `Read destination`.
    - *Edge Cases:* Throws errors if search phrase length is invalid, if `spreadsheetId` does not match standard Google Sheets ID format, or if bounds are violated.

#### Block 1.2: Destination Preparation & Locking
- **Overview:** Connects to Google Sheets to verify workbook access, check capacity limits, create missing tabs, and set a temporary concurrency lock.
- **Nodes Involved:** `Read destination`, `Prepare tabs and lock`, `Acquire destination lock`, `Read existing rows`, `Validate existing tables`
- **Node Details:**
  - **Read destination** (`n8n-nodes-base.httpRequest`)
    - *Role:* Fetches spreadsheet metadata (sheet titles and properties) via the Google Sheets API.
    - *Configuration:* GET request to `https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}?fields=spreadsheetId,sheets(properties)`. Uses OAuth2 credentials (`googleSheetsOAuth2Api`). Max 3 retries.
    - *Connections:* Input from `Validate configuration`, output to `Prepare tabs and lock`.
  - **Prepare tabs and lock** (`n8n-nodes-base.code`)
    - *Role:* Generates batch update requests to create missing required tabs (`Mercado Current`, `Mercado History`, `Mercado Runs`) and a temporary lock tab (`_MERCADO_WORKFLOW_LOCK`).
    - *Configuration:* JavaScript function generating batch update payloads.
    - *Connections:* Input from `Read destination`, output to `Acquire destination lock`.
  - **Acquire destination lock** (`n8n-nodes-base.httpRequest`)
    - *Role:* Applies the lock sheet and tab additions to Google Sheets.
    - *Configuration:* POST request to `https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}:batchUpdate`. Uses OAuth2 credentials.
    - *Connections:* Input from `Prepare tabs and lock`, output to `Read existing rows`.
    - *Edge Cases:* Fails immediately if the lock tab already exists, indicating an active or interrupted prior execution.
  - **Read existing rows** (`n8n-nodes-base.httpRequest`)
    - *Role:* Retrieves existing data from the business, history, and run ledger tabs.
    - *Configuration:* GET request using `values:batchGet` for ranges `Mercado Current`, `Mercado Runs`, and `Mercado History`.
    - *Connections:* Input from `Acquire destination lock`, output to `Validate existing tables`.
  - **Validate existing tables** (`n8n-nodes-base.code`)
    - *Role:* Validates table headers, row counts, and duplicate keys.
    - *Configuration:* JavaScript validation block.
    - *Connections:* Input from `Read existing rows`, output to `Resume an existing run?`.
    - *Edge Cases:* Throws errors if headers differ, table capacity exceeds 5,000 rows, or duplicate offer keys are detected.

#### Block 1.3: Execution Branching (New vs. Recovery)
- **Overview:** Branches workflow execution based on whether a `resumeRunId` was provided to recover an existing Apify run or start a new scraping task.
- **Nodes Involved:** `Resume an existing run?`, `Load recovery run`, `Start bounded Mercado Libre run`, `Capture run`
- **Node Details:**
  - **Resume an existing run?** (`n8n-nodes-base.if`)
    - *Role:* Evaluates whether `resumeRunId` is set in configuration.
    - *Configuration:* Condition checks if `resumeRunId` is not an empty string.
    - *Connections:* Input from `Validate existing tables`. True branch outputs to `Load recovery run`; false branch outputs to `Start bounded Mercado Libre run`.
  - **Load recovery run** (`n8n-nodes-base.httpRequest`)
    - *Role:* Fetches metadata for an existing Apify run when recovering.
    - *Configuration:* GET request to `https://api.apify.com/v2/actor-runs/{resumeRunId}` using Apify credentials (`apifyApi`).
    - *Connections:* Input from `Resume an existing run?`, output to `Capture run`.
  - **Start bounded Mercado Libre run** (`n8n-nodes-base.httpRequest`)
    - *Role:* Initiates a new scraping run on Apify using the FalconScraper Mercado Libre Actor (`kBAOo15cdTngT2yya`).
    - *Configuration:* POST request to `https://api.apify.com/v2/acts/kBAOo15cdTngT2yya/runs` with JSON body specifying `siteId: 'MLA'`, search queries, item limits, and charge caps. Uses Apify credentials.
    - *Connections:* Input from `Resume an existing run?`, output to `Capture run`.
  - **Capture run** (`n8n-nodes-base.code`)
    - *Role:* Normalizes run metadata output from either the recovery or newly started run.
    - *Configuration:* JavaScript code parsing response payloads and verifying Actor ID match.
    - *Connections:* Input from `Load recovery run` or `Start bounded Mercado Libre run`, output to `Read original run input`.

#### Block 1.4: Polling & Status Verification
- **Overview:** Saves run checkpoints to Google Sheets and polls the Apify API until the scraping task succeeds or times out.
- **Nodes Involved:** `Read original run input`, `Prepare run checkpoint`, `Save run checkpoint`, `Wait for Actor status`, `Inspect Actor status`, `Actor succeeded?`
- **Node Details:**
  - **Read original run input** (`n8n-nodes-base.httpRequest`)
    - *Role:* Retrieves the input payload stored in the Apify key-value store for validation.
    - *Configuration:* GET request to Apify key-value store records endpoint using Apify credentials.
    - *Connections:* Input from `Capture run`, output to `Prepare run checkpoint`.
  - **Prepare run checkpoint** (`n8n-nodes-base.code`)
    - *Role:* Prepares initial ledger entry marking the run status as `PROCESSING`.
    - *Configuration:* JavaScript mapping context and building batch update payload for `Mercado Runs`.
    - *Connections:* Input from `Read original run input`, output to `Save run checkpoint`.
  - **Save run checkpoint** (`n8n-nodes-base.httpRequest`)
    - *Role:* Writes the `PROCESSING` run checkpoint to the Google Sheets run ledger.
    - *Configuration:* POST request to Google Sheets `batchUpdate` endpoint using OAuth2 credentials.
    - *Connections:* Input from `Prepare run checkpoint`, output to `Wait for Actor status`.
  - **Wait for Actor status** (`n8n-nodes-base.httpRequest`)
    - *Role:* Polls Apify for run status updates with a 30-second blocking wait parameter (`waitForFinish=30`).
    - *Configuration:* GET request to `https://api.apify.com/v2/actor-runs/{runId}?waitForFinish=30`. Uses Apify credentials.
    - *Connections:* Input from `Save run checkpoint` or `Actor succeeded?` (false branch), output to `Inspect Actor status`.
  - **Inspect Actor status** (`n8n-nodes-base.code`)
    - *Role:* Validates returned run status and enforces maximum polling iteration limits.
    - *Configuration:* JavaScript code validating run success, failure, abort, or timeout states.
    - *Connections:* Input from `Wait for Actor status`, output to `Actor succeeded?`.
  - **Actor succeeded?** (`n8n-nodes-base.if`)
    - *Role:* Checks whether the Apify actor run has completed successfully (`SUCCEEDED`).
    - *Configuration:* Evaluates `finished === true`.
    - *Connections:* Input from `Inspect Actor status`. True branch outputs to `Fetch complete bounded dataset`; false branch loops back to `Wait for Actor status`.

#### Block 1.5: Data Processing & Comparison
- **Overview:** Fetches scraped dataset items from Apify, normalizes offers, and plans price comparison writes against historical records.
- **Nodes Involved:** `Fetch complete bounded dataset`, `Plan price observations`
- **Node Details:**
  - **Fetch complete bounded dataset** (`n8n-nodes-base.httpRequest`)
    - *Role:* Downloads dataset items generated by the completed Apify run.
    - *Configuration:* GET request to `https://api.apify.com/v2/datasets/{datasetId}/items?format=json&limit=1001` using Apify credentials.
    - *Connections:* Input from `Actor succeeded?`, output to `Plan price observations`.
  - **Plan price observations** (`n8n-nodes-base.code`)
    - *Role:* Executes core pricing logic, normalizing listings, filtering out sponsored redirect links or mismatched currencies, calculating price deltas and percentages, and planning batch write ranges for Google Sheets.
    - *Configuration:* JavaScript processing block containing `normalize()`, `planWrites()`, and ledger generation.
    - *Connections:* Input from `Fetch complete bounded dataset`, output to `Write prices history and run log`.
    - *Edge Cases:* Throws errors if dataset retrieval is incomplete, prices are invalid, or observation tables exceed limits.

#### Block 1.6: Finalization & Lock Release
- **Overview:** Writes processed records to Google Sheets, releases the concurrency lock, and generates a structured run summary.
- **Nodes Involved:** `Write prices history and run log`, `Prepare lock release`, `Release destination lock`, `Run summary`
- **Node Details:**
  - **Write prices history and run log** (`n8n-nodes-base.httpRequest`)
    - *Role:* Writes current prices, history observations, and updated run logs to Google Sheets in batch.
    - *Configuration:* POST request to Google Sheets `batchUpdate` endpoint using OAuth2 credentials.
    - *Connections:* Input from `Plan price observations`, output to `Prepare lock release`.
  - **Prepare lock release** (`n8n-nodes-base.code`)
    - *Role:* Generates the delete sheet request payload for the temporary lock tab (`_MERCADO_WORKFLOW_LOCK`).
    - *Configuration:* JavaScript code parsing lock sheet properties and creating sheet deletion parameters.
    - *Connections:* Input from `Write prices history and run log`, output to `Release destination lock`.
  - **Release destination lock** (`n8n-nodes-base.httpRequest`)
    - *Role:* Deletes the temporary lock sheet from the Google Sheets spreadsheet.
    - *Configuration:* POST request to Google Sheets `batchUpdate` endpoint using OAuth2 credentials.
    - *Connections:* Input from `Prepare lock release`, output to `Run summary`.
  - **Run summary** (`n8n-nodes-base.code`)
    - *Role:* Compiles execution statistics, record counts, baselines, price changes, and spreadsheet direct URLs into a final output summary.
    - *Configuration:* JavaScript summary builder.
    - *Connections:* Input from `Release destination lock`. Terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | `manualTrigger` | Entry point for manual executions | None | Configure search | Track Mercado Libre Argentina offer prices... |
| Configure search | `set` | Assigns default search parameters and limits | Run manually | Validate configuration | Track Mercado Libre Argentina offer prices... |
| Validate configuration | `code` | Validates input configurations and parameters | Configure search | Read destination | 1. Set one Argentina search<br>Enter one Mercado Libre keyword and an empty spreadsheet ID. Start with 20 listings and a $0.09 Actor charge ceiling. These nodes validate input and Sheets access before paid work. |
| Read destination | `httpRequest` | Fetches spreadsheet metadata | Validate configuration | Prepare tabs and lock | 2. Prepare protected history tabs<br>The workflow creates **Mercado Current**, **Mercado History**, and **Mercado Runs**. A temporary lock prevents overlapping imports. Existing rows and capacity are checked before scraping. |
| Prepare tabs and lock | `code` | Generates tab creation and lock requests | Read destination | Acquire destination lock | 2. Prepare protected history tabs<br>The workflow creates **Mercado Current**, **Mercado History**, and **Mercado Runs**. A temporary lock prevents overlapping imports. Existing rows and capacity are checked before scraping. |
| Acquire destination lock | `httpRequest` | Applies lock sheet and missing tabs | Prepare tabs and lock | Read existing rows | 2. Prepare protected history tabs<br>The workflow creates **Mercado Current**, **Mercado History**, and **Mercado Runs**. A temporary lock prevents overlapping imports. Existing rows and capacity are checked before scraping. |
| Read existing rows | `httpRequest` | Retrieves existing sheet data values | Acquire destination lock | Validate existing tables | 2. Prepare protected history tabs<br>The workflow creates **Mercado Current**, **Mercado History**, and **Mercado Runs**. A temporary lock prevents overlapping imports. Existing rows and capacity are checked before scraping. |
| Validate existing tables | `code` | Validates headers, capacities, and keys | Read existing rows | Resume an existing run? | 2. Prepare protected history tabs<br>The workflow creates **Mercado Current**, **Mercado History**, and **Mercado Runs**. A temporary lock prevents overlapping imports. Existing rows and capacity are checked before scraping. |
| Resume an existing run? | `if` | Branches to new run or recovery | Validate existing tables | Load recovery run, Start bounded Mercado Libre run | 3. Start or recover a bounded run<br>Leave **resumeRunId** blank for a new listing-only Argentina run. A saved run ID resumes its original dataset without starting another scrape. The Actor, query, site, and settings must match. |
| Load recovery run | `httpRequest` | Fetches existing Apify run metadata | Resume an existing run? | Capture run | 3. Start or recover a bounded run<br>Leave **resumeRunId** blank for a new listing-only Argentina run. A saved run ID resumes its original dataset without starting another scrape. The Actor, query, site, and settings must match. |
| Start bounded Mercado Libre run | `httpRequest` | Initiates new Apify scraping job | Resume an existing run? | Capture run | 3. Start or recover a bounded run<br>Leave **resumeRunId** blank for a new listing-only Argentina run. A saved run ID resumes its original dataset without starting another scrape. The Actor, query, site, and settings must match. |
| Capture run | `code` | Normalizes run metadata | Load recovery run / Start bounded Mercado Libre run | Read original run input | 3. Start or recover a bounded run<br>Leave **resumeRunId** blank for a new listing-only Argentina run. A saved run ID resumes its original dataset without starting another scrape. The Actor, query, site, and settings must match. |
| Read original run input | `httpRequest` | Retrieves Apify key-value input payload | Capture run | Prepare run checkpoint | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Prepare run checkpoint | `code` | Prepares initial run ledger checkpoint entry | Read original run input | Save run checkpoint | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Save run checkpoint | `httpRequest` | Writes processing status to Google Sheets | Prepare run checkpoint | Wait for Actor status | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Wait for Actor status | `httpRequest` | Polls Apify for run status updates | Save run checkpoint / Actor succeeded? | Inspect Actor status | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Inspect Actor status | `httpRequest` / `code` | Validates Apify run completion status | Wait for Actor status | Actor succeeded? | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Actor succeeded? | `if` | Evaluates if Apify run succeeded | Inspect Actor status | Fetch complete bounded dataset, Wait for Actor status | 4. Save the ID and await success<br>Before polling, the run ID and original query are saved in **Mercado Runs**. A failed or unfinished Actor run does not import. Inspect Apify before any retry because n8n cancellation does not stop the Actor. |
| Fetch complete bounded dataset | `httpRequest` | Downloads scraped dataset items from Apify | Actor succeeded? | Plan price observations | 5. Compare identical offer IDs<br>Only MLA offers with valid ARS prices and direct product links are imported. Sponsored click-tracking redirects are rejected. The listing offer ID is the comparison key, never the catalogue ID or title. |
| Plan price observations | `code` | Normalizes items and plans delta calculations | Fetch complete bounded dataset | Write prices history and run log | 5. Compare identical offer IDs<br>Only MLA offers with valid ARS prices and direct product links are imported. Sponsored click-tracking redirects are rejected. The listing offer ID is the comparison key, never the catalogue ID or title. |
| Write prices history and run log | `httpRequest` | Writes observations and run log to Google Sheets | Plan price observations | Prepare lock release | 6. Release lock and review counts<br>After successful writes, the workflow releases its lock. **Run summary** reports baselines, price changes, rejected rows, and replay counts. Missing offers are not observed, not automatically sold out. |
| Prepare lock release | `code` | Prepares lock tab deletion request | Write prices history and run log | Release destination lock | 6. Release lock and review counts<br>After successful writes, the workflow releases its lock. **Run summary** reports baselines, price changes, rejected rows, and replay counts. Missing offers are not observed, not automatically sold out. |
| Release destination lock | `httpRequest` | Deletes temporary lock sheet | Prepare lock release | Run summary | 6. Release lock and review counts<br>After successful writes, the workflow releases its lock. **Run summary** reports baselines, price changes, rejected rows, and replay counts. Missing offers are not observed, not automatically sold out. |
| Run summary | `code` | Compiles final run metrics and summary | Release destination lock | None | 6. Release lock and review counts<br>After successful writes, the workflow releases its lock. **Run summary** reports baselines, price changes, rejected rows, and replay counts. Missing offers are not observed, not automatically sold out. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Add Search Configuration:**
   - Add a **Set** node (`n8n-nodes-base.set`). Name it `Configure search`.
   - Configure assignments with string and number fields: `search` (default `"iphone 15"`), `maxItems` (`20`), `maxChargeUsd` (`0.09`), `spreadsheetId` (`""`), and `resumeRunId` (`""`).
   - Connect `Run manually` to `Configure search`.
3. **Add Input Validation Code:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate configuration`.
   - Paste the core profile configuration and validation helper functions checking string lengths, sheet ID formats, and range bounds.
   - Connect `Configure search` to `Validate configuration`.
4. **Read Spreadsheet Metadata:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Read destination`.
   - Set method to `GET`, URL to `={{ 'https://sheets.googleapis.com/v4/spreadsheets/' + $('Validate configuration').first().json.spreadsheetId + '?fields=spreadsheetId,sheets(properties)' }}`.
   - Configure authentication using pre-defined Google Sheets OAuth2 API credentials.
   - Connect `Validate configuration` to `Read destination`.
5. **Prepare Sheets and Lock:**
   - Add a **Code** node. Name it `Prepare tabs and lock`.
   - Add an **HTTP Request** node named `Acquire destination lock` (POST to `:batchUpdate`, using Google Sheets OAuth2).
   - Connect `Read destination` $\rightarrow$ `Prepare tabs and lock` $\rightarrow$ `Acquire destination lock`.
6. **Read Existing Data and Validate:**
   - Add an **HTTP Request** node named `Read existing rows` (GET to `values:batchGet` for ranges `Mercado Current`, `Mercado Runs`, and `Mercado History`).
   - Add a **Code** node named `Validate existing tables` to verify table structures and capacities.
   - Connect `Acquire destination lock` $\rightarrow$ `Read existing rows` $\rightarrow$ `Validate existing tables`.
7. **Configure Branching (New vs. Recovery):**
   - Add an **If** node (`n8n-nodes-base.if`). Name it `Resume an existing run?`.
   - Set condition to evaluate if `resumeRunId` is not empty.
   - Add an **HTTP Request** node named `Load recovery run` (GET to Apify actor runs endpoint).
   - Add an **HTTP Request** node named `Start bounded Mercado Libre run` (POST to Apify acts endpoint with JSON payload `siteId: 'MLA'`, search queries, and limits).
   - Configure both to use Apify API credentials (`apifyApi`).
   - Connect `Validate existing tables` to `Resume an existing run?`. Route true to `Load recovery run`, false to `Start bounded Mercado Libre run`.
8. **Capture Run and Checkpoints:**
   - Add a **Code** node named `Capture run` connected from both run execution paths.
   - Add an **HTTP Request** node named `Read original run input` (GET to Apify key-value store records).
   - Add a **Code** node named `Prepare run checkpoint` and an **HTTP Request** node named `Save run checkpoint` (POST to Google Sheets `batchUpdate`).
   - Connect `Capture run` $\rightarrow$ `Read original run input` $\rightarrow$ `Prepare run checkpoint` $\rightarrow$ `Save run checkpoint`.
9. **Poll Actor Status:**
   - Add an **HTTP Request** node named `Wait for Actor status` (GET to Apify actor runs endpoint with `waitForFinish=30`).
   - Add a **Code** node named `Inspect Actor status`.
   - Add an **If** node named `Actor succeeded?`.
   - Connect `Save run checkpoint` to `Wait for Actor status` $\rightarrow$ `Inspect Actor status` $\rightarrow$ `Actor succeeded?`.
   - Route the false branch of `Actor succeeded?` back to `Wait for Actor status`.
10. **Fetch Dataset and Plan Writes:**
    - Add an **HTTP Request** node named `Fetch complete bounded dataset` (GET to Apify dataset items endpoint).
    - Add a **Code** node named `Plan price observations` to normalize items and calculate deltas.
    - Connect the true branch of `Actor succeeded?` $\rightarrow$ `Fetch complete bounded dataset` $\rightarrow$ `Plan price observations`.
11. **Write Data, Release Lock, and Summarize:**
    - Add an **HTTP Request** node named `Write prices history and run log` (POST to Google Sheets `batchUpdate`).
    - Add a **Code** node named `Prepare lock release`.
    - Add an **HTTP Request** node named `Release destination lock` (POST to Google Sheets `batchUpdate`).
    - Add a **Code** node named `Run summary` to compile execution results.
    - Connect in sequence: `Plan price observations` $\rightarrow$ `Write prices history and run log` $\rightarrow$ `Prepare lock release` $\rightarrow$ `Release destination lock` $\rightarrow$ `Run summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| FalconScrape Mercado Libre Listings Scraper Actor | [Apify Actor Link](https://apify.com/piotrv1001/mercado-libre-listings-scraper) |
| Target Market & Currency | Argentina (MLA), ARS currency. No automatic FX conversion. |
| Row Limit Constraint | Archive tables before reaching 5,000 rows per tab. |