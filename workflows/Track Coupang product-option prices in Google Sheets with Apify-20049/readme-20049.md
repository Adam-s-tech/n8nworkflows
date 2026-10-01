Track Coupang product-option prices in Google Sheets with Apify

https://n8nworkflows.xyz/workflows/track-coupang-product-option-prices-in-google-sheets-with-apify-20049


# Track Coupang product-option prices in Google Sheets with Apify

### 1. Workflow Overview

This workflow is a comprehensive price-tracking system designed to manually scrape Coupang product listings and option prices via the Apify platform (“FalconScrape Coupang Listings Scraper”) and persist structured price history, current statuses, and execution checkpoints inside Google Sheets. 

The primary target use cases are e-commerce monitoring, competitor price tracking, and product-option cost analysis in South Korean Won (KRW). The architecture ensures transactional safety using temporary locking mechanisms, strict schema validations, deduplication logic, and idempotency checks to prevent race conditions or duplicate pricing entries.

The workflow logic is categorized into six functional blocks:
- **1.1 Input Reception & Configuration Validation:** Captures manual execution parameters, validates search terms, charges, and Google Spreadsheet identifiers, and initializes runtime profiles.
- **1.2 Destination Initialization & Locking:** Inspects the target spreadsheet, sets up managed tabs (`Coupang Current`, `Coupang History`, `Coupang Runs`), and acquires a temporary synchronization lock.
- **1.3 Scraper Execution & Recovery Management:** Evaluates whether to resume an interrupted Apify run or initialize a new charge-capped scraping operation, recording run checkpoints.
- **1.4 Execution Polling & Verification:** Actively polls the Apify Actor status with timeout thresholds until completion and validates success metrics.
- **1.5 Data Normalization & Persistence:** Fetches complete datasets, normalizes and validates offers, computes pricing deltas against historical baselines, and writes transactional updates to Google Sheets.
- **1.6 Lock Release & Reporting:** Removes the temporary synchronization lock and aggregates a final run summary containing execution statistics and spreadsheet deep links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration Validation
- **Overview:** Initializes the manual run context, establishes user-defined parameters for the scrape (search keywords, budget ceilings, and sheet identifiers), and performs strict inline validation against security and format constraints.
- **Nodes Involved:** `Run manually`, `Configure search`, `Validate configuration`.
- **Node Details:**
  - **Run manually**
    - Type: `n8n-nodes-base.manualTrigger`
    - Technical Role: Entry point for manual workflow invocation.
    - Configuration: Default trigger parameters.
    - Input Connections: None (Source).
    - Output Connections: `Configure search`.
    - Edge Cases: Requires manual user interaction in the n8n interface.
  - **Configure search**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Sets up default string and numerical parameters for the run.
    - Configuration: Assigns `search` ("laptop"), `maxItems` (20), `maxChargeUsd` (0.05), `spreadsheetId` (""), and `resumeRunId` ("").
    - Input Connections: `Run manually`.
    - Output Connections: `Validate configuration`.
    - Edge Cases: Empty spreadsheet IDs or malformed inputs are caught in subsequent validation steps.
  - **Validate configuration**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Validates keyword length (2–100 chars), spreadsheet ID regex patterns (`/^[A-Za-z0-9_-]{20,100}$/`), item count bounds (1–100), and Apify run ID formats.
    - Configuration: Executes an embedded JavaScript validator matching configuration rules against predefined profiles.
    - Input Connections: `Configure search`.
    - Output Connections: `Read destination`.
    - Edge Cases: Throws errors on invalid string lengths, URL-formatted spreadsheet links instead of IDs, or out-of-bounds budgets.

#### 2.2 Destination Initialization & Locking
- **Overview:** Inspects the target Google Spreadsheet structure, creates missing tabs with predefined grid configurations, verifies table headers and row counts, and establishes a concurrency lock.
- **Nodes Involved:** `Read destination`, `Prepare tabs and lock`, `Acquire destination lock`, `Read existing rows`, `Validate existing tables`.
- **Node Details:**
  - **Read destination**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches spreadsheet metadata (sheet names and IDs) via Google Sheets REST API.
    - Configuration: GET request using predefined OAuth2 credentials (`googleSheetsOAuth2Api`), configured with up to 3 retries.
    - Input Connections: `Validate configuration`.
    - Output Connections: `Prepare tabs and lock`.
    - Edge Cases: Authentication failures or invalid spreadsheet IDs will cause API errors.
  - **Prepare tabs and lock**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Generates batch update requests to create the temporary lock sheet (`_COUPANG_WORKFLOW_LOCK`) and core data tabs if they do not exist.
    - Configuration: JavaScript logic analyzing spreadsheet properties.
    - Input Connections: `Read destination`.
    - Output Connections: `Acquire destination lock`.
    - Edge Cases: Fails if the spreadsheet metadata structure is unreadable.
  - **Acquire destination lock**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Executes the batch update payload to instantiate the lock sheet in Google Sheets.
    - Configuration: POST request to Google Sheets API (`:batchUpdate`). Retry on fail is disabled to prevent overlapping lock creation races.
    - Input Connections: `Prepare tabs and lock`.
    - Output Connections: `Read existing rows`.
    - Edge Cases: If a stale lock sheet already exists, execution halts to prevent race conditions.
  - **Read existing rows**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Bulk reads existing range values from current business data, execution logs, and historical archives.
    - Configuration: GET request using `batchGet` across ranges `'Coupang Current'!A1:O5002`, `'Coupang Runs'!A1:R5002`, and `'Coupang History'!A1:N5002`.
    - Input Connections: `Acquire destination lock`.
    - Output Connections: `Validate existing tables`.
    - Edge Cases: Network timeouts on large sheets; limited to a 5,000-row capacity constraint.
  - **Validate existing tables**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Validates existing table headers, checks for duplicate primary keys, and verifies row ceilings.
    - Configuration: JavaScript table validation logic.
    - Input Connections: `Read existing rows`.
    - Output Connections: `Resume an existing run?`.
    - Edge Cases: Throws an error if table headers have been altered or if tables exceed capacity limits.

#### 2.3 Scraper Execution & Recovery Management
- **Overview:** Checks whether an existing Apify run ID (`resumeRunId`) was specified for recovery or if a new charge-capped scraping actor run should be initialized.
- **Nodes Involved:** `Resume an existing run?`, `Load recovery run`, `Start bounded Coupang run`, `Capture run`, `Read original run input`, `Prepare run checkpoint`, `Save run checkpoint`.
- **Node Details:**
  - **Resume an existing run?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Routing switch determining whether to load a recovery run or start a fresh actor run.
    - Configuration: Evaluates if `resumeRunId` is not an empty string.
    - Input Connections: `Validate existing tables`.
    - Output Connections: `Loadrecovery run` (True branch), `Start bounded Coupang run` (False branch).
  - **Load recovery run**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches details of an existing Apify run for recovery purposes.
    - Configuration: GET request to Apify API (`/v2/actor-runs/{id}`) using Apify API credentials (`apifyApi`).
    - Input Connections: `Resume an existing run?` (True).
    - Output Connections: `Capture run`.
  - **Start bounded Coupang run**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Triggers a new execution of the Apify Coupang listings scraper actor with budget and item constraints.
    - Configuration: POST request to Apify Actor endpoint (`/v2/acts/WAsYoQvKU6uKzfJyo/runs`) with query parameters for timeout, build version, and maximum total charge ($USD).
    - Input Connections: `Resume an existing run?` (False).
    - Output Connections: `Capture run`.
  - **Capture run**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Validates that the fetched or initiated run belongs to the correct Actor ID.
    - Configuration: JavaScript runtime validation.
    - Input Connections: `Load recovery run`, `Start bounded Coupang run`.
    - Output Connections: `Read original run input`.
  - **Read original run input**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves the input key-value record associated with the Apify run to guarantee parameter integrity.
    - Configuration: GET request to Apify Key-Value Store endpoint (`/v2/key-value-stores/{id}/records/INPUT`).
    - Input Connections: `Capture run`.
    - Output Connections: `Prepare run checkpoint`.
  - **Prepare run checkpoint**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Formats the execution context and constructs a "PROCESSING" ledger row to log the run state in Google Sheets.
    - Configuration: JavaScript processing logic.
    - Input Connections: `Read original run input`.
    - Output Connections: `Save run checkpoint`.
  - **Save run checkpoint**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Writes the initial run execution log to the `Coupang Runs` sheet before polling begins.
    - Configuration: POST request to Google Sheets `:batchUpdate` API.
    - Input Connections: `Prepare run checkpoint`.
    - Output Connections: `Wait for Actor status`.

#### 2.4 Execution Polling & Verification
- **Overview:** Periodically polls the Apify Actor run status until execution succeeds, handling iteration limits and timeout protection.
- **Nodes Involved:** `Wait for Actor status`, `Inspect Actor status`, `Actor succeeded?`.
- **Node Details:**
  - **Wait for Actor status**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Polls Apify API with a blocking wait timeout (`waitForFinish=30`) to check run progress.
    - Configuration: GET request to Apify Actor Runs endpoint.
    - Input Connections: `Save run checkpoint`, `Actor succeeded?` (False branch).
    - Output Connections: `Inspect Actor status`.
  - **Inspect Actor status**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Validates the current run state, checking for failures, aborts, or timeouts, and enforces maximum loop iteration counts to prevent infinite polling.
    - Configuration: JavaScript validation script evaluating run status and execution index (`$runIndex`).
    - Input Connections: `Wait for Actor status`.
    - Output Connections: `Actor succeeded?`.
  - **Actor succeeded?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Conditional branch checking whether the Apify actor has successfully completed.
    - Configuration: Evaluates `$json.finished === true`.
    - Input Connections: `Inspect Actor status`.
    - Output Connections: `Fetch complete bounded dataset` (True branch), `Wait for Actor status` (False branch, looping back for polling).

#### 2.5 Data Normalization & Persistence
- **Overview:** Downloads the scraped dataset items, normalizes and deduplicates product-option records, computes price changes against previous observations, and writes updates to Google Sheets.
- **Nodes Involved:** `Fetch complete bounded dataset`, `Plan price observations`, `Write prices history and run log`.
- **Node Details:**
  - **Fetch complete bounded dataset**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Downloads the final JSON dataset items resulting from the completed Apify scraper run.
    - Configuration: GET request to Apify Datasets API (`/v2/datasets/{id}/items`).
    - Input Connections: `Actor succeeded?` (True).
    - Output Connections: `Plan price observations`.
  - **Plan price observations**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Normalizes raw items, filters out invalid prices or missing identifiers, maps product-and-option keys, computes baseline vs. price increase/decrease metrics, and builds structured batch update payloads for Google Sheets.
    - Configuration: Complex JavaScript planning script containing normalization algorithms and delta calculations.
    - Input Connections: `Fetch complete bounded dataset`.
    - Output Connections: `Write prices history and run log`.
    - Edge Cases: Rejects incomplete datasets, duplicate keys with conflicting prices, or rows exceeding table capacity limits.
  - **Write prices history and run log**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Writes the calculated business data, historical observations, and finalized run logs into Google Sheets in a single batch transaction.
    - Configuration: POST request to Google Sheets API (`:batchUpdate`).
    - Input Connections: `Plan price observations`.
    - Output Connections: `Prepare lock release`.

#### 2.6 Lock Release & Reporting
- **Overview:** Cleans up synchronization resources by deleting the temporary workflow lock tab and outputs a comprehensive execution summary.
- **Nodes Involved:** `Prepare lock release`, `Release destination lock`, `Run summary`.
- **Node Details:**
  - **Prepare lock release**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Extracts the sheet ID of the temporary lock tab from previous API responses and constructs a deletion payload.
    - Configuration: JavaScript extraction script.
    - Input Connections: `Write prices history and run log`.
    - Output Connections: `Release destination lock`.
  - **Release destination lock**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Executes the sheet deletion request to remove the concurrency lock from the spreadsheet.
    - Configuration: POST request to Google Sheets API (`:batchUpdate`).
    - Input Connections: `Prepare lock release`.
    - Output Connections: `Run summary`.
  - **Run summary**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Generates the final execution report containing added, updated, replayed, and rejected item counts alongside a direct hyperlink to the Google Spreadsheet.
    - Configuration: JavaScript formatting script.
    - Input Connections: `Release destination lock`.
    - Output Connections: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | `manualTrigger` | Entry point for manual workflow invocation | None | Configure search | Track Coupang product-option prices |
| Configure search | `set` | Sets up default search and configuration parameters | Run manually | Validate configuration | 1. Set a small Korean product search |
| Validate configuration | `code` | Validates search length, spreadsheet ID format, and numerical bounds | Configure search | Read destination | 1. Set a small Korean product search |
| Read destination | `httpRequest` | Fetches spreadsheet metadata and sheet tabs | Validate configuration | Prepare tabs and lock | 2. Create and protect three tabs |
| Prepare tabs and lock | `code` | Generates requests to create lock and core data tabs | Read destination | Acquire destination lock | 2. Create and protect three tabs |
| Acquire destination lock | `httpRequest` | Creates the temporary lock sheet in Google Sheets | Prepare tabs and lock | Read existing rows | 2. Create and protect three tabs |
| Read existing rows | `httpRequest` | Bulk reads current business data, runs, and history | Acquire destination lock | Validate existing tables | 2. Create and protect three tabs |
| Validate existing tables | `code` | Validates table headers and row capacity limits | Read existing rows | Resume an existing run? | 2. Create and protect three tabs |
| Resume an existing run? | `if` | Determines whether to resume a run or start a new one | Validate existing tables | Load recovery run, Start bounded Coupang run | 3. Start or resume one scrape |
| Load recovery run | `httpRequest` | Fetches details of an existing Apify run | Resume an existing run? | Capture run | 3. Start or resume one scrape |
| Start bounded Coupang run | `httpRequest` | Triggers a new charge-capped Apify actor run | Resume an existing run? | Capture run | 3. Start or resume one scrape |
| Capture run | `code` | Validates actor run ID and ownership | Load recovery run, Start bounded Coupang run | Read original run input | 3. Start or resume one scrape |
| Read original run input | `httpRequest` | Retrieves input key-value record from Apify | Capture run | Prepare run checkpoint | 4. Save the run before waiting |
| Prepare run checkpoint | `code` | Constructs initial run checkpoint ledger row | Read original run input | Save run checkpoint | 4. Save the run before waiting |
| Save run checkpoint | `httpRequest` | Writes initial run log to Google Sheets | Prepare run checkpoint | Wait for Actor status | 4. Save the run before waiting |
| Wait for Actor status | `httpRequest` | Polls Apify actor run status with timeout | Save run checkpoint, Actor succeeded? | Inspect Actor status | 4. Save the run before waiting |
| Inspect Actor status | `code` | Evaluates run status and prevents infinite loops | Wait for Actor status | Actor succeeded? | 4. Save the run before waiting |
| Actor succeeded? | `if` | Checks if Apify actor run completed successfully | Inspect Actor status | Fetch complete bounded dataset, Wait for Actor status | 4. Save the run before waiting |
| Fetch complete bounded dataset | `httpRequest` | Downloads final dataset items from Apify | Actor succeeded? | Plan price observations | 5. Compare the same product option |
| Plan price observations | `code` | Normalizes items, computes price deltas, plans sheet updates | Fetch complete bounded dataset | Write prices history and run log | 5. Compare the same product option |
| Write prices history and run log | `httpRequest` | Writes observations and run logs to Google Sheets | Plan price observations | Prepare lock release | 5. Compare the same product option |
| Prepare lock release | `code` | Prepares sheet deletion request for the lock tab | Write prices history and run log | Release destination lock | 6. Unlock and review the result |
| Release destination lock | `httpRequest` | Deletes the temporary lock sheet from Google Sheets | Prepare lock release | Run summary | 6. Unlock and review the result |
| Run summary | `code` | Aggregates execution statistics and spreadsheet URL | Release destination lock | None | 6. Unlock and review the result |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Configure Search Parameters:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Configure search`. Connect it to `Run manually`.
   - Add string and number assignments: `search` (string: "laptop"), `maxItems` (number: 20), `maxChargeUsd` (number: 0.05), `spreadsheetId` (string: ""), and `resumeRunId` (string: "").
3. **Add Configuration Validation:**
   - Add a **Code** node named `Validate configuration`. Connect it to `Configure search`.
   - Implement JavaScript validation logic to check keyword length, spreadsheet ID format, and numerical constraints.
4. **Read Google Sheets Metadata:**
   - Add an **HTTP Request** node named `Read destination`. Connect it to `Validate configuration`.
   - Set Method to `GET`, URL to `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Validate configuration').first().json.spreadsheetId }}?fields=spreadsheetId,sheets(properties)`.
   - Select predefined credentials for `googleSheetsOAuth2Api`. Enable retry on fail (up to 3 tries).
5. **Prepare Tabs and Lock:**
   - Add a **Code** node named `Prepare tabs and lock`. Connect it to `Read destination`.
   - Add JavaScript logic to build sheet creation requests for `_COUPANG_WORKFLOW_LOCK`, `Coupang Current`, `Coupang History`, and `Coupang Runs`.
6. **Acquire Destination Lock:**
   - Add an **HTTP Request** node named `Acquire destination lock`. Connect it to `Prepare tabs and lock`.
   - Set Method to `POST`, URL to `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Validate configuration').first().json.spreadsheetId }}:batchUpdate`. Send body as JSON.
   - Configure `googleSheetsOAuth2Api` credentials. **Important:** Disable retry on fail.
7. **Read Existing Rows:**
   - Add an **HTTP Request** node named `Read existing rows`. Connect it to `Acquire destination lock`.
   - Set Method to `GET`, URL to fetch ranges for `Coupang Current`, `Coupang Runs`, and `Coupang History` with `valueRenderOption=UNFORMATTED_VALUE`.
   - Configure `googleSheetsOAuth2Api` credentials with up to 3 retries.
8. **Validate Existing Tables:**
   - Add a **Code** node named `Validate existing tables`. Connect it to `Read existing rows`.
   - Implement table header and capacity validation logic.
9. **Branch for Run Recovery or New Execution:**
   - Add an **If** node named `Resume an existing run?`. Connect it to `Validate existing tables`.
   - Set condition to check if `{{ $('Validate configuration').first().json.resumeRunId !== '' }}`.
10. **Load Recovery Run or Start Bounded Run:**
    - Branch True: Add an **HTTP Request** node named `Load recovery run`. Set Method `GET`, URL `https://api.apify.com/v2/actor-runs/{{ $('Validate configuration').first().json.resumeRunId }}` using `apifyApi` credentials.
    - Branch False: Add an **HTTP Request** node named `Start bounded Coupang run`. Set Method `POST`, URL `https://api.apify.com/v2/acts/WAsYoQvKU6uKzfJyo/runs?timeout=240&build=0.0.10&maxTotalChargeUsd={{ $('Validate configuration').first().json.maxChargeUsd }}` with body containing `searchTerms`, `maxResults`, etc., using `apifyApi` credentials.
11. **Capture Run and Read Input:**
    - Connect both branches to a **Code** node named `Capture run`.
    - Connect `Capture run` to an **HTTP Request** node named `Read original run input` (GET `https://api.apify.com/v2/key-value-stores/{{ $json.defaultKeyValueStoreId }}/records/INPUT` using `apifyApi` credentials).
12. **Save Run Checkpoint:**
    - Connect `Read original run input` to a **Code** node named `Prepare run checkpoint`.
    - Connect to an **HTTP Request** node named `Save run checkpoint` (POST batchUpdate to Google Sheets using `googleSheetsOAuth2Api` credentials).
13. **Poll Actor Status Loop:**
    - Add an **HTTP Request** node named `Wait for Actor status`. Connect it to `Save run checkpoint` (and later to the False branch of the success check). Set URL `https://api.apify.com/v2/actor-runs/{{ $('Capture run').first().json.id }}+?waitForFinish=30` using `apifyApi` credentials.
    - Connect to a **Code** node named `Inspect Actor status`.
    - Connect to an **If** node named `Actor succeeded?` evaluating `{{ $json.finished === true }}`.
14. **Fetch Dataset & Plan Observations:**
    - On the True branch of `Actor succeeded?`, add an **HTTP Request** node named `Fetch complete bounded dataset` (GET `https://api.apify.com/v2/datasets/{{ $('Inspect Actor status').last().json.run.defaultDatasetId }}/items?format=json&limit=1001` using `apifyApi` credentials).
    - Connect to a **Code** node named `Plan price observations` to normalize items and calculate price deltas.
15. **Write Data and Release Lock:**
    - Connect `Plan price observations` to an **HTTP Request** node named `Write prices history and run log` (POST batchUpdate to Google Sheets using `googleSheetsOAuth2Api` credentials).
    - Connect to a **Code** node named `Prepare lock release`.
    - Connect to an **HTTP Request** node named `Release destination lock` (POST batchUpdate to Google Sheets to delete the lock sheet; disable retry on fail).
16. **Generate Final Summary:**
    - Connect `Release destination lock` to a **Code** node named `Run summary` to output execution metrics and the spreadsheet link.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Apify Actor Link | [FalconScrape Coupang Listings Scraper](https://apify.com/piotrv1001/coupang-listings-scraper) |
| Spreadsheet URL Template | `https://docs.google.com/spreadsheets/d/{spreadsheetId}` |
| Operational Disclaimer | Prices are observations of identical product-and-option IDs in KRW. Missing offers do not imply sell-outs, and sales prices may exclude shipping, coupons, or member-only conditions. |