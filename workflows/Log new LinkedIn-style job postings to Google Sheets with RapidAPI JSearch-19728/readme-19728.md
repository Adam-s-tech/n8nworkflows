Log new LinkedIn-style job postings to Google Sheets with RapidAPI JSearch

https://n8nworkflows.xyz/workflows/log-new-linkedin-style-job-postings-to-google-sheets-with-rapidapi-jsearch-19728


# Log new LinkedIn-style job postings to Google Sheets with RapidAPI JSearch

### 1. Workflow Overview

This workflow automates the discovery, filtering, deduplication, and logging of LinkedIn-style job postings. It runs on a fixed 12-hour interval, queries the RapidAPI JSearch service, filters out senior management or advanced-level titles, cross-references existing records in a Google Sheets tracking log, and appends only new opportunities for human review.

The processing logic splits into two parallel branches following parameter initialization: one branch queries the external job API and normalizes/filters the results, while the other branch retrieves historical entries from Google Software storage. Both data streams converge at a comparison engine, and newly isolated entries write sequentially back to the destination sheet before terminating at a neutral operational node.

---

### 1.1 Input Reception & Configuration
- **Schedule Trigger:** Initiates the automation cycle every 12 hours.
- **Search Parameters:** Centralizes query variables (job title, geographical location, remote constraints, time windows, and exclusion terms).

### 1.2 External Retrieval & Normalization
- **Search Jobs (RapidAPI):** Executes an HTTP GET request against the JSearch `/search-v2` endpoint using query parameters derived from the configuration node.
- **Split Job Results:** Iterates through the response payload array (`data.jobs`) to flatten nested job listings into discrete, independent operational items.
- **Exclude Unwanted Titles:** Evaluates each job title against a regular expression filter to drop senior positions (Senior, Lead, Staff, Principal).

### 1.3 State Retrieval & Deduplication
- **Get Existing Jobs:** Pulls all rows asynchronously from the designated Google Sheets document to establish a baseline of previously logged records.
- **Dedupe New Jobs:** Compares the filtered incoming stream against the existing sheet records using the unique `job_id` property, isolating items present only in the incoming data stream.

### 1.4 Persistence & Finalization
- **Log New Job:** Appends validated, unique job records to the Google Sheets document, leaving status tracking fields blank for manual operations.
- **Finish:** Acts as the terminal node (No-Op) signaling the end of the execution flow.

---

### 2. Block-by-Block Analysis

---

### 2.1 Block 1: Trigger & Configuration

#### Overview
This block establishes the execution cadence and provides a centralized configuration node to declare job search criteria, variables, and filters.

#### Nodes Involved
- Schedule Trigger
- Search Parameters

#### Node Details

##### Schedule Trigger
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (v1.2) - Time-based event initiator.
- **Configuration Choices:** Configured to fire at an interval of every 12 hours (`hoursInterval: 12`).
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** 
  - Input: None (Trigger node).
  - Output: Connects to `Search Parameters`.
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure types:** Missed executions if the n8n instance is offline during the trigger window; platform concurrency limitations.
- **Sub-workflow Reference:** None.

##### Search Parameters
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.4) - Data transformation and variable assignment node.
- **Configuration Choices:** Assigns static variables for target roles, locations, remote preferences, date windows, and exclusion keywords.
- **Key Expressions or Variables:** 
  - `job_title`: `Automation Engineer`
  - `location`: `India`
  - `remote_only`: `false`
  - `date_posted`: `month`
  - `exclude_keywords`: `Senior,Lead,Staff,Principal`
- **Input and Output Connections:** 
  - Input: Receives execution from `Schedule Trigger`.
  - Output: Fan-out connection to both `Search Jobs (RapidAPI)` and `Get Existing Jobs`.
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure types:** Variable type mismatches (e.g., boolean vs. string) if modified incorrectly. Note that `exclude_keywords` is declared here for documentation purposes but evaluated via regex in later nodes.
- **Sub-workflow Reference:** None.

---

### 2.2 Block 2: Fetch & Filter Jobs

#### Overview
This block queries the external JSearch API via RapidAPI using the configured search parameters, explodes the resulting array into individual elements, and drops unwanted senior-level titles.

#### Nodes Involved
- Search Jobs (RapidAPI)
- Split Job Results
- Exclude Unwanted Titles

#### Node Details

##### Search Jobs (RapidAPI)
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) - External API integration node.
- **Configuration Choices:** Sends a GET request to `https://jsearch.p.rapidapi.com/search-v2` using generic header authentication. Passes query parameters (`query`, `page`, `num_pages`, `date_posted`, `remote_jobs_only`, `country`) mapped dynamically from the preceding Set node.
- **Key Expressions or Variables:** 
  - Query: `={{ $json.job_title + ' in ' + $json.location }}`
  - Date Posted: `={{ $json.date_posted }}`
  - Remote Only: `={{ $json.remote_only }}`
- **Input and Output Connections:** 
  - Input: Receives execution from `Search Parameters`.
  - Output: Connects to `Split Job Results`.
- **Version-specific Requirements:** Requires an active RapidAPI Key credential configured with the `X-RapidAPI-Host` header.
- **Edge Cases or Potential Failure types:** Rate-limiting errors (HTTP 429), authentication failures (HTTP 401/403) due to invalid RapidAPI keys, network timeouts, or schema changes in the JSearch API response.
- **Sub-workflow Reference:** None.

##### Split Job Results
- **Type and Technical Role:** `n8n-nodes-base.splitOut` (v1) - Array flattener/item list processor.
- **Configuration Choices:** Splits the nested JSON array path `data.jobs` into distinct, individual items for downstream evaluation.
- **Key Expressions or Variables:** 
  - Field to Split Out: `=data.jobs`
- **Input and Output Connections:** 
  - Input: Receives execution from `Search Jobs (RapidAPI)`.
  - Output: Connects to `Exclude Unwanted Titles`.
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure types:** Execution failure if the API returns an empty dataset, missing `data.jobs` property, or a null payload.
- **Sub-workflow Reference:** None.

##### Exclude Unwanted Titles
- **Type and Technical Role:** `n8n-nodes-base.filter` (v2.2) - Conditional filtering node.
- **Configuration Choices:** Filters incoming items based on the job title property using a negative regular expression match.
- **Key Expressions or Variables:** 
  - Left Value: `={{ $json.job_title }}`
  - Operator: `notRegex`
  - Right Value: `=/(Senior|Lead|Staff|Principal)/i`
- **Input and Output Connections:** 
  - Input: Receives execution from `Split Job Results`.
  - Output: Connects to `Dedupe New Jobs` (Input A).
- **Version-specific Requirements:** Uses rule version 2 syntax.
- **Edge Cases or Potential Failure types:** Failure if `$json.job_title` is undefined or null; overly aggressive filtering if titles contain compound words.
- **Sub-workflow Reference:** None.

---

### 2.3 Block 3: Load Logged Jobs

#### Overview
This block runs asynchronously alongside the API fetch branch to retrieve all existing entries from the tracking database, establishing a historical baseline.

#### Nodes Involved
- Get Existing Jobs

#### Node Details

##### Get Existing Jobs
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (v4.5) - Spreadsheet data retrieval node.
- **Configuration Choices:** Reads rows from a specified Google Sheets document ID (`1SUdsZ6m_PGHTYnZmQETYESSiwv4_Kz4p9aA329feQ0A`) targeting the `Jobs` tab (`gid=0`).
- **Key Expressions or Variables:** None (uses static document and sheet identifiers).
- **Input and Output Connections:** 
  - Input: Receives execution from `Search Parameters`.
  - Output: Connects to `Dedupe New Jobs` (Input B).
- **Version-specific Requirements:** Requires an active Google Sheets OAuth2 credential with read permissions.
- **Edge Cases or Potential Failure types:** API quota limits, authorization token expiration, or incorrect sheet/tab names resulting in sheet-not-found errors.
- **Sub-workflow Reference:** None.

---

### 2.4 Block 4: Dedupe

#### Overview
This block cross-references newly fetched listings against historical records to isolate unseen job opportunities.

#### Nodes Involved
- Dedupe New Jobs

#### Node Details

##### Dedupe New Jobs
- **Type and Technical Role:** `n8n-nodes-base.compareDatasets` (v2.3) - Data comparison and merging node.
- **Configuration Choices:** Compares Input A (filtered API jobs) and Input B (Google Sheet historical rows) using `job_id` as the matching key field. Configured to output only items unique to Input A (`In A only`).
- **Key Expressions or Variables:** 
  - Merge By Fields: `job_id` (Field 1 from Input A, Field 2 from Input B).
- **Input and Output Connections:** 
  - Input 1 (A): Receives data from `Exclude Unwanted Titles`.
  - Input 2 (B): Receives data from `Get Existing Jobs`.
  - Output: Connects to `Log New Job`.
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure types:** Data type mismatches between string-based IDs and numeric IDs causing false-positive duplicates; missing `job_id` fields in incoming data payloads.
- **Sub-workflow Reference:** None.

---

### 2.5 Block 5: Log New Jobs

#### Overview
This block appends validated, unseen job listings to the tracking spreadsheet and completes the execution cycle.

#### Nodes Involved
- Log New Job
- Finish

#### Node Details

##### Log New Job
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (v4.5) - Spreadsheet data insertion node.
- **Configuration Choices:** Appends new rows to the target spreadsheet (`1SUdsZ6m_PGHTYnZmQETYESSiwv4_Kz4p9aA329feQ0A`) on tab `gid=0`. Explicitly maps API payload properties to columns (`job_id`, `title`, `company`, `location`, `url`, `date_posted`), leaving `status` blank.
- **Key Expressions or Variables:** 
  - `url`: `={{ $json.job_apply_link }}`
  - `title`: `={{ $json.job_title }}`
  - `job_id`: `={{ $json.job_id }}`
  - `company`: `={{ $json.employer_name }}`
  - `location`: `={{ $json.job_location }}`
  - `date_posted`: `={{ $json.job_posted_at_datetime_utc ? new Date($json.job_posted_at_datetime_utc).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }) : '' }}`
- **Input and Output Connections:** 
  - Input: Receives data from `Dedupe New Jobs`.
  - Output: Connects to `Finish`.
- **Version-specific Requirements:** Requires an active Google Sheets OAuth2 credential with write/append permissions.
- **Edge Cases or Potential Failure types:** Malformed date strings causing JavaScript parsing errors; missing target columns in the Google Sheet; API rate limits imposed by Google.
- **Sub-workflow Reference:** None.

##### Finish
- **Type and Technical Role:** `n8n-nodes-base.noOp` (v1) - Neutral operational end-cap node.
- **Configuration Choices:** Default no-operation settings.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** 
  - Input: Receives execution from `Log New Job`.
  - Output: None (Terminal node).
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure types:** None.
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Triggers execution every 12 hours | None | Search Parameters | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**1. Trigger & Config**<br><br>Runs every 12 hours and defines the search (title, location, remote flag, date window, excluded keywords) in one Set node. |
| **Search Parameters** | `n8n-nodes-base.set` | Assigns search variables and parameters | Schedule Trigger | Search Jobs (RapidAPI), Get Existing Jobs | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**1. Trigger & Config**<br><br>Runs every 12 hours and defines the search (title, location, remote flag, date window, excluded keywords) in one Set node.<br><br>Edit your search criteria here.<br><br>Note: `exclude_keywords` isn't wired to the title filter yet. |
| **Search Jobs (RapidAPI)** | `n8n-nodes-base.httpRequest` | Queries JSearch API via RapidAPI | Search Parameters | Split Job Results | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**2. Fetch & Filter Jobs**<br><br>Calls the JSearch API on RapidAPI, splits `data.jobs` into one item per job, then drops Senior / Lead / Staff / Principal titles.<br><br>Add your RapidAPI Header Auth credential here (`X-RapidAPI-Key`). |
| **Split Job Results** | `n8n-nodes-base.splitOut` | Flattens job array items | Search Jobs (RapidAPI) | Exclude Unwanted Titles | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**2. Fetch & Filter Jobs**<br><br>Calls the JSearch API on RapidAPI, splits `data.jobs` into one item per job, then drops Senior / Lead / Staff / Principal titles. |
| **Exclude Unwanted Titles** | `n8n-nodes-base.filter` | Filters out senior-level job titles | Split Job Results | Dedupe New Jobs | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**2. Fetch & Filter Jobs**<br><br>Calls the JSearch API on RapidAPI, splits `data.jobs` into one item per job, then drops Senior / Lead / Staff / Principal titles. |
| **Get Existing Jobs** | `n8n-nodes-base.googleSheets` | Reads current sheet records | Search Parameters | Dedupe New Jobs | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**3. Load Logged Jobs**<br><br>Runs in parallel with section 2. Reads every row from the `Jobs` sheet so already-seen jobs can be skipped.<br><br>Add your Google Sheets OAuth2 credential and Sheet ID here. |
| **Dedupe New Jobs** | `n8n-nodes-base.compareDatasets` | Compares incoming jobs against existing logs | Exclude Unwanted Titles, Get Existing Jobs | Log New Job | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**4. Dedupe**<br><br>Compares fetched jobs (Input A) with sheet rows (Input B) on `job_id`. Only the 'In A only' output continues. |
| **Log New Job** | `n8n-nodes-base.googleSheets` | Appends unique records to sheet | Dedupe New Jobs | Finish | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready<br><br>**5. Log New Jobs**<br><br>Appends each new job to the `Jobs` sheet. `status` stays blank so you can track it manually.<br><br>Add your Google Sheets credential here. Use the same Sheet ID as Get Existing Jobs. |
| **Finish** | `n8n-nodes-base.noOp` | Terminal end-cap node | Log New Job | None | **Process Implementation Guide:** A fully automated workflow that checks for new job postings every 12 hours and logs the ones it hasn't seen before to a Google Sheet.<br><br>**Setup (credentials needed)**<br><br>1. **RapidAPI (JSearch)** – Header Auth credential (`X-RapidAPI-Key`), used to search for jobs<br>2. **Google Sheets OAuth2** – used to read existing jobs and log new ones<br>3. Create the `Jobs` tab using the header row in the sheet note above<br>4. Set your Sheet ID in **Get Existing Jobs** and **Log New Job**<br>5. Edit **Search Parameters** to match your target role<br>6. Activate the workflow once ready |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Target Google Sheet Document:**
   - Create a new Google Sheet and rename the first tab to `Jobs`.
   - In cell `A1`, insert the header row: `job_id`, `title`, `company`, `location`, `url`, `date_posted`, `status`.
   - Note the spreadsheet ID from the URL.

2. **Configure Credentials:**
   - Set up an HTTP Header Auth credential in n8n for RapidAPI (`X-RapidAPI-Key`).
   - Set up a Google Sheets OAuth2 credential in n8n with read and write permissions.

3. **Build Node 1: Schedule Trigger**
   - Create a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to fire every `12` hours.

4. **Build Node 2: Search Parameters**
   - Create a **Set** node (`n8n-nodes-base.set`).
   - Add string assignments: `job_title` ("Automation Engineer"), `location` ("India"), `date_posted` ("month"), `exclude_keywords` ("Senior,Lead,Staff,Principal").
   - Add boolean assignment: `remote_only` (`false`).
   - Connect `Schedule Trigger` output to `Search Parameters`.

5. **Build Node 3: Search Jobs (RapidAPI)**
   - Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Set Method to `GET`, URL to `https://jsearch.p.rapidapi.com/search-v2`.
   - Set Authentication to `Generic Credential Type` -> `HTTP Header Auth`.
   - Add Header Parameter: `X-RapidAPI-Host` = `jsearch.p.rapidapi.com`.
   - Add Query Parameters:
     - `query` = `={{ $json.job_title + ' in ' + $json.location }}`
     - `page` = `1`
     - `num_pages` = `1`
     - `date_posted` = `={{ $json.date_posted }}`
     - `remote_jobs_only` = `={{ $json.remote_only }}`
     - `country` = `in`
   - Connect execution output from `Search Parameters` to this node.

6. **Build Node 4: Split Job Results**
   - Create a **Split Out** node (`n8n-nodes-base.splitOut`).
   - Set field to split out to `=data.jobs`.
   - Connect output of `Search Jobs (RapidAPI)` to this node.

7. **Build Node 5: Exclude Unwanted Titles**
   - Create a **Filter** node (`n8n-nodes-base.filter`).
   - Configure condition: Left Value `={{ $json.job_title }}`, operator `notRegex`, Right Value `=/(Senior|Lead|Staff|Principal)/i`.
   - Connect output of `Split Job Results` to this node.

8. **Build Node 6: Get Existing Jobs**
   - Create a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Set Operation to `Get` (or default read rows behavior), resource `Sheet`, Document ID to your target Google Sheet ID, and Sheet Name to `Jobs`.
   - Connect second execution output from `Search Parameters` to this node.

9. **Build Node 7: Dedupe New Jobs**
   - Create a **Compare Datasets** node (`n8n-nodes-base.compareDatasets`).
   - Connect Input A to `Exclude Unwanted Titles` and Input B to `Get Existing Jobs`.
   - Set merge criteria to match on field `job_id` (`field1: job_id`, `field2: job_id`).
   - Configure output to return items in `In A only`.

10. **Build Node 8: Log New Job**
    - Create a second **Google Sheets** node (`n8n-nodes-base.googleSheets`).
    - Set Operation to `Append`, Document ID to your target Google Sheet ID, and Sheet Name to `Jobs`.
    - Map columns explicitly using definitions:
      - `job_id` = `={{ $json.job_id }}`
      - `title` = `={{ $json.job_title }}`
      - `company` = `={{ $json.employer_name }}`
      - `location` = `={{ $json.job_location }}`
      - `url` = `={{ $json.job_apply_link }}`
      - `date_posted` = `={{ $json.job_posted_at_datetime_utc ? new Date($json.job_posted_at_datetime_utc).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' }) : '' }}`
      - Leave `status` unmapped.
    - Connect output from `Dedupe New Jobs` to this node.

11. **Build Node 9: Finish**
    - Create a **No-Op** node (`n8n-nodes-base.noOp`).
    - Connect output of `Log New Job` to this node to complete the flow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheet Template Schema | `job_id`, `title`, `company`, `location`, `url`, `date_posted`, `status` |
| RapidAPI JSearch Documentation Endpoint | Target base URL: `https://jsearch.p.rapidapi.com/search-v2` |