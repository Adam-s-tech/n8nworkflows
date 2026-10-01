Qualify Google Sheets leads by website domain age using RDAP lookups

https://n8nworkflows.xyz/workflows/qualify-google-sheets-leads-by-website-domain-age-using-rdap-lookups-20042


# Qualify Google Sheets leads by website domain age using RDAP lookups

### 1. Workflow Overview

This workflow is designed to automate the enrichment of business leads stored in Google Sheets by calculating the registration age of each lead's website domain. Utilizing the public, keyless `rdap.org` (Registration Data Access Protocol) service, it evaluates the exact domain creation date and writes back qualification metrics without modifying existing row data.

The system execution is organized into three sequential functional blocks:
- **1.1 Input Reception & Configuration:** Initializes global execution parameters, loads the raw dataset from Google Sheets, and standardizes website URLs into clean, registrable domain formats while filtering out unprocessable entries (e.g., social media profiles, IP addresses, missing websites).
- **1.2 Iterative RDAP Processing:** Manages a controlled, rate-limited processing loop that queries domain registration data, implements automatic exponential backoff retries for transient HTTP errors (such as `429` or `5xx`), and computes operational metrics (domain creation date, years registered, data source, and validation timestamps).
- **1.3 Aggregation & Persistence:** Consolidates processed items, compiles structured result sets matched explicitly by source row numbers, and updates the target Google Sheets document.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

##### Overview
This block initializes the global settings, pulls all records from the target Google Sheets tab, and performs data cleaning and domain extraction to prepare individual lead items for subsequent lookup operations.

##### Nodes Involved
- `Start` (`n8n-nodes-base.manualTrigger`)
- `Settings` (`n8n-nodes-base.set`)
- `Read Leads` (`n8n-nodes-base.googleSheets`)
- `Prepare Domains` (`n8n-nodes-base.code`)

##### Node Details

- **Start**
  - **Type and Technical Role:** `n8n-nodes-base.manualTrigger` — Initiates manual test executions of the workflow.
  - **Configuration Choices:** Uses default parameters with no input fields.
  - **Key Expressions or Variables:** None.
  - **Input/Output Connections:** Input: None; Output: Connects to `Settings`.
  - **Version-Specific Requirements:** TypeVersion 1.
  - **Edge Cases / Failure Types:** None.

- **Settings**
  - **Type and Technical Role:** `n8n-nodes-base.set` — Defines global configuration parameters (Sheet ID, tab name, target website column, rate-limiting pacing, and maximum retry attempts).
  - **Configuration Choices:** Assigns string and numeric constants (`sheet_id`, `sheet_tab`, `website_column`, `rdap_pace_seconds`, `max_retries`).
  - **Key Expressions or Variables:** None (static assignment values).
  - **Input/Output Connections:** Input: `Start`; Output: `Read Leads`.
  - **Version-Specific Requirements:** TypeVersion 3.4.
  - **Edge Cases / Failure Types:** Invalid placeholders (e.g., leaving `REPLACE_ME_SHEET_ID` unconfigured) will cause subsequent Google Sheets API calls to fail.

- **Read Leads**
  - **Type and Technical Role:** `n8n-nodes-base.googleSheets` — Reads all rows from the specified Google Sheets worksheet.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `read`, using OAuth2 authentication.
  - **Key Expressions or Variables:**
    - Document ID: `={{ $('Settings').first().json.sheet_id }}`
    - Sheet Name: `={{ $('Settings').first().json.sheet_tab }}`
  - **Input/Output Connections:** Input: `Settings`; Output: `Prepare Domains`.
  - **Version-Specific Requirements:** TypeVersion 4.5.
  - **Edge Cases / Failure Types:** Authentication token expiration, incorrect sheet IDs, or network timeouts when contacting the Google Sheets API.
  - **Sub-Workflow Reference:** None.

- **Prepare Domains**
  - **Type and Technical Role:** `n8n-nodes-base.code` — JavaScript-based code node that cleans website URLs into registrable domains, strips `www.` prefixes, and shortens multi-level country code domains (e.g., `co.uk`). It filters out shared hosting platforms, social networks, and IP addresses.
  - **Configuration Choices:** Executes once for all items (`runOnceForAllItems`). Contains a static array of common third-party platform domains (`PLATFORMS`) and second-level suffixes (`SECOND_LEVEL`).
  - **Key Expressions or Variables:** Evaluates `$input.all()` and references settings from the `Settings` node.
  - **Input/Output Connections:** Input: `Read Leads`; Output: `Loop Over Rows`.
  - **Version-Specific Requirements:** TypeVersion 2.
  - **Edge Cases / Failure Types:** Throws a runtime error if configuration limits (`rdap_pace_seconds` or `max_retries`) are non-numeric or negative. Items missing a website or row number are skipped.

---

#### Block 1.2: Iterative RDAP Processing

##### Overview
This block loops through each prepared lead item one by one, enforces rate limits, queries the `rdap.org` registry endpoint, handles transient server errors via exponential backoff retries, and computes domain age metrics.

##### Nodes Involved
- `Loop Over Rows` (`n8n-nodes-base.splitInBatches`)
- `Needs Lookup?` (`n8n-nodes-base.if`)
- `Wait Before RDAP` (`n8n-nodes-base.wait`)
- `RDAP Lookup` (`n8n-nodes-base.httpRequest`)
- `Check RDAP Response` (`n8n-nodes-base.code`)
- `RDAP Again?` (`n8n-nodes-base.if`)

##### Node Details

- **Loop Over Rows**
  - **Type and Technical Role:** `n8n-nodes-base.splitInBatches` — Processes items sequentially (batch size of 1) to comply with external rate limits.
  - **Configuration Choices:** Default batch options.
  - **Input/Output Connections:** Input: `Prepare Domains` (and loopback from `Needs Lookup?` / `RDAP Again?`); Output: Index 0 connects to `Collect Results` (when loop completes), Index 1 connects to `Needs Lookup?`.
  - **Version-Specific Requirements:** TypeVersion 3.1.
  - **Edge Cases / Failure Types:** Infinite loops if internal state variables are corrupted.

- **Needs Lookup?**
  - **Type and Technical Role:** `n8n-nodes-base.if` — Evaluates whether a valid registrable domain was extracted during the preparation stage.
  - **Configuration Choices:** Condition checks if `$json.lookup` equals `yes`.
  - **Key Expressions or Variables:** `={{ $json.lookup }}`
  - **Input/Output Connections:** Input: `Loop Over Rows`; Output: True branch connects to `Wait Before RDAP`, False branch loops back to `Loop Over Rows`.
  - **Version-Specific Requirements:** TypeVersion 2.2.
  - **Edge Cases / Failure Types:** None.

- **Wait Before RDAP**
  - **Type and Technical Role:** `n8n-nodes-base.wait` — Pauses execution between API requests to respect the rate limits of `rdap.org` (10 requests per 10 seconds).
  - **Configuration Choices:** Resume mechanism set to `timeInterval`.
  - **Key Expressions or Variables:**
    - Wait Amount: `={{ $json.wait_s }}`
  - **Input/Output Connections:** Input: `Needs Lookup?` or `RDAP Again?`; Output: `RDAP Lookup`.
  - **Version-Specific Requirements:** TypeVersion 1.1.
  - **Edge Cases / Failure Types:** Long execution times for large datasets due to pacing intervals.

- **RDAP Lookup**
  - **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Executes an HTTP GET request against the `rdap.org` endpoint to fetch registration data.
  - **Configuration Choices:** Method: `GET`, Response Format: `JSON`, Full Response enabled, Error handling set to continue regular output (`onError: continueRegularOutput`), timeout set to 30,000ms. Sends custom header `Accept: application/rdap+json`.
  - **Key Expressions or Variables:**
    - Request URL: `=https://rdap.org/domain/{{ $json.domain }}`
  - **Input/Output Connections:** Input: `Wait Before RDAP`; Output: `Check RDAP Response`.
  - **Version-Specific Requirements:** TypeVersion 4.2.
  - **Edge Cases / Failure Types:** Network timeouts, DNS resolution failures, or HTTP status codes (`429`, `5xx`, `404`). The node catches errors and passes them down-chain instead of halting execution.

- **Check RDAP Response**
  - **Type and Technical Role:** `n8n-nodes-base.code` — JavaScript code node that parses RFC 9083 JSON responses from RDAP. It extracts registration dates, computes full years registered, and determines whether a retry or an error assignment (`unknown`) is required.
  - **Configuration Choices:** Runs once for all items (`runOnceForAllItems`).
  - **Key Expressions or Variables:** Accesses state from `Wait Before RDAP` and response payloads from `RDAP Lookup`.
  - **Input/Output Connections:** Input: `RDAP Lookup`; Output: `RDAP Again?`.
  - **Version-Specific Requirements:** TypeVersion 2.
  - **Edge Cases / Failure Types:** Unparseable dates, missing registration events in the JSON body, or exceeding `max_retries`.

- **RDAP Again?**
  - **Type and Technical Role:** `n8n-nodes-base.if` — Determines whether the current item needs to be retried due to a rate limit or server error.
  - **Configuration Choices:** Condition checks if `$json.next` equals `rdap`.
  - **Key Expressions or Variables:** `={{ $json.next }}`
  - **Input/Output Connections:** Input: `Check RDAP Response`; Output: True branch routes to `Wait Before RDAP` (retry), False branch loops back to `Loop Over Rows` (completion).
  - **Version-Specific Requirements:** TypeVersion 2.2.
  - **Edge Cases / Failure Types:** None.

---

#### Block 1.3: Aggregation & Persistence

##### Overview
This block collects all processed row data from the loop, filters it down to target update fields, and writes the results back to the source Google Sheets document.

##### Nodes Involved
- `Collect Results` (`n8n-nodes-base.code`)
- `Write Back Ages` (`n8n-nodes-base.googleSheets`)

##### Node Details

- **Collect Results**
  - **Type and Technical Role:** `n8n-nodes-base.code` — Aggregates processed items from the loop and strips out temporary metadata, retaining only the fields required for the sheet update.
  - **Configuration Choices:** Executes once for all items. Defines target columns: `row_number`, `domain_created`, `years_registered`, `age_source`, `checked_at`.
  - **Key Expressions or Variables:** Iterates over `$input.all()`.
  - **Input/Output Connections:** Input: `Loop Over Rows` (exit branch); Output: `Write Back Ages`.
  - **Version-Specific Requirements:** TypeVersion 2.
  - **Edge Cases / Failure Types:** Empty dataset handling.

- **Write Back Ages**
  - **Type and Technical Role:** `n8n-nodes-base.googleSheets` — Updates specific columns on the original Google Sheets rows based on row matching.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `update`, matching columns set to `row_number`, using OAuth2 authentication. Auto-maps input data schema.
  - **Key Expressions or Variables:**
    - Document ID: `={{ $('Settings').first().json.sheet_id }}`
    - Sheet Name: `={{ $('Settings').first().json.sheet_tab }}`
  - **Input/Output Connections:** Input: `Collect Results`; Output: None (Terminal node).
  - **Version-Specific Requirements:** TypeVersion 4.5.
  - **Edge Cases / Failure Types:** API rate limits enforced by Google, mismatched row identifiers, or expired OAuth2 tokens.
  - **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start | n8n-nodes-base.manualTrigger | Initiates manual workflow execution. | None | Settings | ## Qualify leads in Google Sheets by website domain age with RDAP<br><br>### Who's it for<br>Anyone with leads in Google Sheets who wants to know how long each business has existed, using the website domain's registration date from **rdap.org**: free, no key.<br><br>### How it works<br>1. **Settings** names the sheet, tab and website column. **Read Leads** loads every row.<br>2. **Prepare Domains** skips rows without a website and reads the registrable domain (`shop.example.co.uk` becomes `example.co.uk`). Shared hosts such as facebook.com get "unknown" without a lookup.<br>3. **Loop Over Rows**, **Wait Before RDAP** and **RDAP Lookup** check one domain every 1.1 s (rdap.org allows 10 requests per 10 s).<br>4. **Check RDAP Response** reads the registration date and retries a 429 or 5xx. A date it cannot get becomes "unknown" with the reason, never a guess.<br>5. **Collect Results** and **Write Back Ages** fill four columns on the same rows.<br><br>### How to set up<br>1. Create a Google Sheets credential named `Google Sheets account`.<br>2. Add four headers to your leads tab: `domain_created`, `years_registered`, `age_source`, `checked_at`.<br>3. Fill **Settings** (`sheet_id` is the long part of the sheet URL) and click Execute workflow.<br><br>### Requirements<br>n8n, a Google account, and a leads tab with a website column. 100 rows take about two minutes.<br><br>### How to customize the workflow<br>- Raise `rdap_pace_seconds` if you see 429s.<br>- The paid Local Lead List Builder finds local businesses and adds this domain-age check in one run: https://edgef4.gumroad.com/l/local-lead-list-builder |
| Settings | n8n-nodes-base.set | Defines configuration parameters (Sheet ID, tab, column, pacing, retries). | Start | Read Leads | [Duplicate of Start Sticky Note] |
| Read Leads | n8n-nodes-base.googleSheets | Reads every row of the leads tab; each row carries its row_number. | Settings | Prepare Domains | ### 1. Read the sheet<br>**Read Leads** loads every row. **Prepare Domains** keeps rows with a website and reads the registrable domain; shared hosts get unknown without a lookup. |
| Prepare Domains | n8n-nodes-base.code | Skips rows without a website; registrable domain; shared hosts and IPs get unknown without a lookup. | Read Leads | Loop Over Rows | [Duplicate of Read Leads Sticky Note] |
| Loop Over Rows | n8n-nodes-base.splitInBatches | Processes rows one at a time. | Prepare Domains, RDAP Again?, Needs Lookup? | Collect Results, Needs Lookup? | ### 2. Look each domain up<br>One row at a time, 1.1 s apart. rdap.org redirects to the registry. 429 or 5xx: wait 1, 2, 4, 8, 16 s and retry, then unknown. |
| Collect Results | n8n-nodes-base.code | Reduces rows to the five columns the sheet node updates, matched on row_number. | Loop Over Rows | Write Back Ages | ### 3. Write back<br>Four columns updated on the same rows, matched on row_number. Nothing else on the row changes. |
| Write Back Ages | n8n-nodes-base.googleSheets | Updates domain_created, years_registered, age_source, checked_at on the same rows (row_number). | Collect Results | None | [Duplicate of Collect Results Sticky Note] |
| Needs Lookup? | n8n-nodes-base.if | Filters out rows with no usable domain (shared host, IP, garbage); marks them unknown. | Loop Over Rows | Wait Before RDAP, Loop Over Rows | [Duplicate of Loop Over Rows Sticky Note] |
| Wait Before RDAP | n8n-nodes-base.wait | Enforces rate limits (rdap.org allows 10 requests per 10 seconds). | Needs Lookup?, RDAP Again? | RDAP Lookup | [Duplicate of Loop Over Rows Sticky Note] |
| RDAP Lookup | n8n-nodes-base.httpRequest | Calls rdap.org to fetch domain registration records, following 302 redirects. | Wait Before RDAP | Check RDAP Response | [Duplicate of Loop Over Rows Sticky Note] |
| Check RDAP Response | n8n-nodes-base.code | Parses registration date, manages retry logic, or assigns unknown with a reason. | RDAP Lookup | RDAP Again? | [Duplicate of Loop Over Rows Sticky Note] |
| RDAP Again? | n8n-nodes-base.if | Determines whether to wait and retry the RDAP lookup or exit the loop. | Check RDAP Response | Wait Before RDAP, Loop Over Rows | [Duplicate of Loop Over Rows Sticky Note] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Add a `Start` (`n8n-nodes-base.manualTrigger`) node.
2. **Create the Settings Node**
   - Add a `Set` (`n8n-nodes-base.set`) node named `Settings`.
   - Configure assignments under parameters with the following string and number fields:
     - `sheet_id` (String): `REPLACE_ME_SHEET_ID` (Update with your target Google Sheet ID)
     - `sheet_tab` (String): `Leads`
     - `website_column` (String): `website`
     - `rdap_pace_seconds` (Number): `1.1`
     - `max_retries` (Number): `5`
   - Connect `Start` to `Settings`.
3. **Create the Read Leads Node**
   - Add a Google Sheets (`n8n-nodes-base.googleSheets`) node named `Read Leads`.
   - Set Resource to `Sheet` and Operation to `Read`.
   - Configure credentials for `Google Sheets OAuth2 API`.
   - Set Document ID expression: `={{ $('Settings').first().json.sheet_id }}`
   - Set Sheet Name expression: `={{ $('Settings').first().json.sheet_tab }}`
   - Connect `Settings` to `Read Leads`.
4. **Create the Prepare Domains Node**
   - Add a Code (`n8n-nodes-base.code`) node named `Prepare Domains`.
   - Set mode to `Run Once for All Items` and language to `JavaScript`.
   - Paste the domain-cleaning script logic to extract registrable domains, filter out common platform URLs/IPs, and initialize tracking attributes (`attempt`, `requests`, `retries`, `wait_s`, `checked_at`).
   - Connect `Read Leads` to `Prepare Domains`.
5. **Create the Loop Control Node**
   - Add a Split In Batches (`n8n-nodes-base.splitInBatches`) node named `Loop Over Rows` with default options.
   - Connect `Prepare Domains` to `Loop Over Rows`.
6. **Create the Validation IF Node**
   - Add an If (`n8n-nodes-base.if`) node named `Needs Lookup?`.
   - Configure condition: String check where `={{ $json.lookup }}` equals `yes`.
   - Connect output index 1 (Loop branch) of `Loop Over Rows` to `Needs Lookup?`.
7. **Create the Wait Node**
   - Add a Wait (`n8n-nodes-base.wait`) node named `Wait Before RDAP`.
   - Set Unit to `Seconds` and Amount expression to `={{ $json.wait_s }}` with resume set to `Time Interval`.
   - Connect the True output of `Needs Lookup?` to `Wait Before RDAP`.
8. **Create the HTTP Request Node**
   - Add an HTTP Request (`n8n-nodes-base.httpRequest`) node named `RDAP Lookup`.
   - Set Method to `GET`, URL to `=https://rdap.org/domain/{{ $json.domain }}`, and Response Format to `JSON`.
   - Enable Full Response, set Request Timeout to `30000ms`, and configure error handling to continue regular output (`onError: continueRegularOutput`).
   - Add header parameter: Name: `Accept`, Value: `application/rdap+json`.
   - Connect `Wait Before RDAP` to `RDAP Lookup`.
9. **Create the RDAP Response Processor Node**
   - Add a Code (`n8n-nodes-base.code`) node named `Check RDAP Response`.
   - Set mode to `Run Once for All Items` and language to `JavaScript`.
   - Implement logic to parse status codes, check for retryable states (`429`, `5xx`, connection errors) up to `max_retries`, and extract registration dates to compute full years registered.
   - Connect `RDAP Lookup` to `Check RDAP Response`.
10. **Create the Retry Evaluation IF Node**
    - Add an If (`n8n-nodes-base.if`) node named `RDAP Again?`.
    - Configure condition: String check where `={{ $json.next }}` equals `rdap`.
    - Connect `Check RDAP Response` to `RDAP Again?`.
    - Connect the True output of `RDAP Again?` back to `Wait Before RDAP`.
    - Connect the False output of `RDAP Again?` back to `Loop Over Rows`.
    - Connect the False output of `Needs Lookup?` back to `Loop Over Rows`.
11. **Create the Results Aggregation Node**
    - Add a Code (`n8n-nodes-base.code`) node named `Collect Results`.
    - Set mode to `Run Once for All Items` and language to `JavaScript`.
    - Filter items down to columns: `row_number`, `domain_created`, `years_registered`, `age_source`, `checked_at`.
    - Connect output index 0 (Done branch) of `Loop Over Rows` to `Collect Results`.
12. **Create the Write Back Google Sheets Node**
    - Add a Google Sheets (`n8n-nodes-base.googleSheets`) node named `Write Back Ages`.
    - Set Resource to `Sheet`, Operation to `Update`, and configure matching columns to use `row_number`.
    - Configure credentials for `Google Sheets OAuth2 API`.
    - Set Document ID and Sheet Name expressions matching the configuration used in `Read Leads`.
    - Connect `Collect Results` to `Write Back Ages`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Paid Local Lead List Builder resource link | [Local Lead List Builder on Gumroad](https://edgef4.gumroad.com/l/local-lead-list-builder) |