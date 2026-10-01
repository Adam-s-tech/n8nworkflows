Track Apple App Store keyword rankings with SerpApi and Google Sheets

https://n8nworkflows.xyz/workflows/track-apple-app-store-keyword-rankings-with-serpapi-and-google-sheets-20187


# Track Apple App Store keyword rankings with SerpApi and Google Sheets

### 1. Workflow Overview

This workflow is designed to automate App Store Optimization (ASO) tracking by periodically checking the search rankings of specified Apple App Store keywords using SerpApi. It targets mobile marketers and indie developers who need a reliable, automated way to track search visibility over time without manual checks.

The logical flow is structured into three main blocks:
- **1.1 Input Reception & Queue Initialization:** Handles manual or scheduled execution triggers and centralizes workflow configuration parameters, building an iterative processing queue for keywords.
- **1.2 API Processing & Rate-Limiting Loop:** Iterates through the keyword queue one by one, enforces a 1-second delay between requests to protect against rate limits, queries the SerpApi Apple App Store engine, and handles error or success outputs safely.
- **1.3 Data Parsing & Persistence:** Analyzes the search results to determine the exact organic rank or return a default threshold value (`>N`), formats the output payload, and appends a structured record into a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Queue Initialization
This block initiates the execution cycle either via a daily schedule or manual action, reading global configuration settings and compiling raw keyword strings into structured execution items.

- **Nodes Involved:** `Schedule daily`, `Manual trigger`, `Build queue from config`, `Sticky Note — Overview`, `Sticky Note — Setup`, `Sticky Note — Config`.

- **Node Details:**
  - **Schedule daily**
    - *Type and technical role:* Trigger node (`n8n-nodes-base.scheduleTrigger`). Automatically fires execution based on time rules.
    - *Configuration choices:* Set to trigger daily at hour 23 (11:00 PM).
    - *Input/Output connections:* No inputs; outputs to `Build queue from config`.
    - *Potential failure types:* None typical; dependent on n8n server uptime and timezone settings.
  - **Manual trigger**
    - *Type and technical role:* Trigger node (`n8n-nodes-base.manualTrigger`). Allows on-demand manual execution for testing and debugging.
    - *Configuration choices:* Default manual run settings.
    - *Input/Output connections:* No inputs; outputs to `Build queue from config`.
    - *Potential failure types:* None.
  - **Build queue from config**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Transforms raw configuration objects and comma/newline-separated keyword groups into flattened execution items containing API keys, settings, and individual keywords.
    - *Key expressions or variables used:* Uses custom JavaScript parsing to handle `SETTINGS` (including `spreadsheetId`, `appStoreAppId`, `country`, `lang`, `resultsNum`) and `GROUPS` arrays. Validates placeholder values.
    - *Input/Output connections:* Inputs from `Schedule daily` or `Manual trigger`; outputs to `Loop keywords`.
    - *Edge cases or potential failure types:* Throws explicit JavaScript errors if `spreadsheetId` contains placeholder text like `YOUR_GOOGLE` or if active group API keys remain set to placeholder values (`PASTE_...`).

---

#### 2.2 API Processing & Rate-Limiting Loop
This block manages iteration through the keyword queue, maintains throttling between outgoing network calls, and safely executes queries against the external search engine API.

- **Nodes Involved:** `Loop keywords`, `Wait 1s`, `SerpApi Apple App Store`, `Sticky Note — Loop`.

- **Node Details:**
  - **Loop keywords**
    - *Type and technical role:* Split in Batches node (`n8n-nodes-base.splitInBatches`). Controls iteration over the queue generated in the previous block.
    - *Configuration choices:* `batchSize` set to `1` to process keywords sequentially.
    - *Input/Output connections:* Inputs from `Build queue from config` and `Append position`. Output branch 1 (completion) loops back to nothing (terminating loop); output branch 2 (looping item) connects to `Wait 1s`.
    - *Potential failure types:* Infinite loops or memory overhead if input arrays are malformed (mitigated by batch sizing).
  - **Wait 1s**
    - *Type and technical role:* Wait node (`n8n-nodes-base.wait`). Pauses workflow execution temporarily.
    - *Configuration choices:* `amount` set to `1` second. Uses an internal webhook reference for resumption.
    - *Input/Output connections:* Inputs from `Loop keywords` (batch output); outputs to `SerpApi Apple App Store`.
    - *Edge cases or potential failure types:* Execution timeouts if set to excessively long periods without worker infrastructure support.
  - **SerpApi Apple App Store**
    - *Type and technical role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Queries the SerpApi endpoint for Apple App Store search results.
    - *Configuration choices:* Target URL `https://serpapi.com/search.json`. Query parameters dynamically populated: `engine=apple_app_store`, `term` (`={{ $json.keyword }}`), `country` (`={{ $json.country }}`), `lang` (`={{ $json.lang }}`), `num` (`={{ $json.resultsNum }}`), `api_key` (`={{ $json.serpApiKey }}`). Error handling configured to continue regular output (`neverError: true`).
    - *Input/Output connections:* Inputs from `Wait 1s`; outputs to `Parse position row`.
    - *Edge cases or potential failure types:* Network timeouts, invalid API keys, quota exhaustion, or malformed JSON responses from SerpApi. Mitigated by `neverError: true`.

---

#### 2.3 Data Parsing & Persistence
This block evaluates raw API payloads, maps organic app placement to quantitative positions, formats structural row dates, and appends the final tracking logs to Google Sheets.

- **Nodes Involved:** `Parse position row`, `Append position`.

- **Node Details:**
  - **Parse position row**
    - *Type and technical role:* Code node (`n8n-nodes-base.code`). Parses raw API search output to locate the target application ID within organic results.
    - *Key expressions or variables used:* Uses JavaScript to locate app matches (`Number(a.id) === appId`), calculate rank positions, format dates (`yyyy-MM-dd`), and isolate API error structures.
    - *Input/Output connections:* Inputs from `SerpApi Apple App Store`; outputs to `Append position`.
    - *Edge cases or potential failure types:* Returns position as `'ERROR'` if the API response contains an error message, or `'>maxResults'` if the target app does not appear within the top retrieved results.
  - **Append position**
    - *Type and technical role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Appends rows of tracking data into a specified spreadsheet tab.
    - *Configuration choices:* Operation set to `append`. Document ID mapped via expression (`={{ $json.spreadsheetId }}`). Sheet name mapped via expression (`={{ $json.positionSheetName }}`). Columns mapped explicitly: `apps`, `date`, `group`, `keyword`, `position`.
    - *Input/Output connections:* Inputs from `Parse position row`; outputs back to `Loop keywords` to process the next batch item.
    - *Edge cases or potential failure types:* Authentication/OAuth2 token expiration, missing target sheet tabs named `position`, or column header mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Schedule daily` | `n8n-nodes-base.scheduleTrigger` | Triggers execution automatically every day at 11:00 PM. | None | `Build queue from config` | ## Apple App Store ASO rank tracker<br><br>**Who it is for:** mobile marketers and indie developers tracking App Store Search visibility for a fixed keyword list.<br><br>**What it does:**<br>1. Builds a queue of keywords from the **Build queue from config** node (supports multiple SerpApi key groups for quota splitting).<br>2. For each keyword, calls [SerpApi Apple App Store engine](https://serpapi.com/apple-app-store-api) and finds your app’s organic rank.<br>3. Appends one row per keyword to Google Sheets: `keyword`, `group`, `position`, `apps`, `date`.<br>4. Runs on a daily schedule or via **Manual trigger** for testing.<br><br>**Requirements:**<br>- [SerpApi](https://serpapi.com/) account and API key(s)<br>- Google account + OAuth2 for Google Sheets<br>- Your app’s numeric **App Store ID** (same as `id` in SerpApi organic results)<br>- A spreadsheet with a sheet named `position` and header row matching the columns above<br><br>**Setup (short):** Edit **Build queue from config** → connect **Append position** → run **Manual trigger** once. |
| `Manual trigger` | `n8n-nodes-base.manualTrigger` | Allows manual workflow execution for testing and debugging. | None | `Build queue from config` | ## Apple App Store ASO rank tracker<br><br>**Who it is for:** mobile marketers and indie developers tracking App Store Search visibility for a fixed keyword list.<br><br>**What it does:**<br>1. Builds a queue of keywords from the **Build queue from config** node (supports multiple SerpApi key groups for quota splitting).<br>2. For each keyword, calls [SerpApi Apple App Store engine](https://serpapi.com/apple-app-store-api) and finds your app’s organic rank.<br>3. Appends one row per keyword to Google Sheets: `keyword`, `group`, `position`, `apps`, `date`.<br>4. Runs on a daily schedule or via **Manual trigger** for testing.<br><br>**Requirements:**<br>- [SerpApi](https://serpapi.com/) account and API key(s)<br>- Google account + OAuth2 for Google Sheets<br>- Your app’s numeric **App Store ID** (same as `id` in SerpApi organic results)<br>- A spreadsheet with a sheet named `position` and header row matching the columns above<br><br>**Setup (short):** Edit **Build queue from config** → connect **Append position** → run **Manual trigger** once. |
| `Build queue from config` | `n8n-nodes-base.code` | Parses configuration arrays and keywords to build a clean processing queue. | `Schedule daily`, `Manual trigger` | `Loop keywords` | ## Apple App Store ASO rank tracker<br><br>**Who it is for:** mobile marketers and indie developers tracking App Store Search visibility for a fixed keyword list.<br><br>**What it does:**<br>1. Builds a queue of keywords from the **Build queue from config** node (supports multiple SerpApi key groups for quota splitting).<br>2. For each keyword, calls [SerpApi Apple App Store engine](https://serpapi.com/apple-app-store-api) and finds your app’s organic rank.<br>3. Appends one row per keyword to Google Sheets: `keyword`, `group`, `position`, `apps`, `date`.<br>4. Runs on a daily schedule or via **Manual trigger** for testing.<br><br>**Requirements:**<br>- [SerpApi](https://serpapi.com/) account and API key(s)<br>- Google account + OAuth2 for Google Sheets<br>- Your app’s numeric **App Store ID** (same as `id` in SerpApi organic results)<br>- A spreadsheet with a sheet named `position` and header row matching the columns above<br><br>**Setup (short):** Edit **Build queue from config** → connect **Append position** → run **Manual trigger** once.<br><br>## Config<br><br>All keywords, SerpApi keys, spreadsheet ID, and App Store ID live in this Code node only. |
| `Loop keywords` | `n8n-nodes-base.splitInBatches` | Iterates through the keyword queue one item at a time. | `Build queue from config`, `Append position` | `Wait 1s` (loop), completion output (none) | ## Fetch & log<br><br>1s wait between keywords (rate limit). SerpApi returns up to `resultsNum` results; rank `>N` if your app is not in the list. |
| `Wait 1s` | `n8n-nodes-base.wait` | Pauses execution for 1 second between API calls to prevent rate limits. | `Loop keywords` | `SerpApi Apple App Store` | ## Fetch & log<br><br>1s wait between keywords (rate limit). SerpApi returns up to `resultsNum` results; rank `>N` if your app is not in the list. |
| `SerpApi Apple App Store` | `n8n-nodes-base.httpRequest` | Calls the SerpApi Apple App Store search endpoint with query parameters. | `Wait 1s` | `Parse position row` | ## Fetch & log<br><br>1s wait between keywords (rate limit). SerpApi returns up to `resultsNum` results; rank `>N` if your app is not in the list. |
| `Parse position row` | `n8n-nodes-base.code` | Parses search ranking data, finds app position, and formats the output row. | `SerpApi Apple App Store` | `Append position` | ## Fetch & log<br><br>1s wait between keywords (rate limit). SerpApi returns up to `resultsNum` results; rank `>N` if your app is not in the list. |
| `Append position` | `n8n-nodes-base.googleSheets` | Appends structured ranking results into the designated Google Sheets tab. | `Parse position row` | `Loop keywords` | ## Setup steps<br><br>1. **SerpApi:** Sign up at serpapi.com → Dashboard → API key. Paste into `apiKey` in **Build queue from config** (replace `PASTE_SERPAPI_KEY_*`). Never commit real keys to git.<br>2. **App ID:** Set `appStoreAppId` in SETTINGS to your app (e.g. from App Store URL or SerpApi `organic_results[].id`).<br>3. **Storefront:** Default is US (`country: us`, `lang: en-us`). Change in SETTINGS for other locales.<br>4. **Google Sheet:** Create sheet tab `position` with columns: `keyword`, `group`, `position`, `apps`, `date`. Set `spreadsheetId` in SETTINGS (from the sheet URL).<br>5. **Credentials:** Open **Append position** → connect **Google Sheets OAuth2**.<br>6. **Test:** Execute **Manual trigger**. Fix placeholders until rows append. Then enable **Schedule daily** if needed.<br><br>**Tip:** Use `enabled: false` on extra GROUPS entries instead of deleting them. Split keywords across groups if you hit SerpApi rate limits. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**
   - Add a **Schedule Trigger** node (`Schedule daily`). Configure the schedule rule interval to trigger daily at hour `23`.
   - Add a **Manual Trigger** node (`Manual trigger`) for on-demand test runs.
2. **Configure Queue Logic:**
   - Add a **Code** node (`Build queue from config`).
   - Insert the configuration script setting global parameters (`spreadsheetId`, `positionSheetName`, `appStoreAppId`, `country`, `lang`, `resultsNum`) and keyword group arrays (`GROUPS`). Ensure placeholders are replaced with valid configuration values.
3. **Setup Iteration and Throttling:**
   - Add a **Split In Batches** node (`Loop keywords`). Set `batchSize` to `1`.
   - Add a **Wait** node (`Wait 1s`). Set the amount parameter to `1` second.
4. **Configure Search API Call:**
   - Add an **HTTP Request** node (`SerpApi Apple App Store`).
   - Set Method to `GET` and URL to `https://serpapi.com/search.json`.
   - Configure query parameters:
     - `engine`: `apple_app_store`
     - `term`: `={{ $json.keyword }}`
     - `country`: `={{ $json.country }}`
     - `lang`: `={{ $json.lang }}`
     - `num`: `={{ $json.resultsNum }}`
     - `api_key`: `={{ $json.serpApiKey }}`
   - Under node options, enable `neverError` in response settings to ensure execution continues if the API returns an error structure.
5. **Add Parsing Logic:**
   - Add a **Code** node (`Parse position row`).
   - Insert the parsing script to extract search results, locate matching app IDs, format dates via luxon (`$now.toFormat('yyyy-MM-dd')`), and construct clean output records containing `keyword`, `group`, `position`, `apps`, `date`, `spreadsheetId`, and `positionSheetName`.
6. **Configure Google Sheets Destination:**
   - Add a **Google Sheets** node (`Append position`).
   - Configure credentials using **Google Sheets OAuth2**.
   - Set operation to `Append`.
   - Set Document ID mode to `ID` and map value to `={{ $json.spreadsheetId }}`.
   - Set Sheet Name mode to `Name` and map value to `={{ $json.positionSheetName }}`.
   - Map columns explicitly:
     - `keyword` → `={{ $json.keyword }}`
     - `group` → `={{ $json.group }}`
     - `position` → `={{ $json.position }}`
     - `apps` → `={{ $json.apps }}`
     - `date` → `={{ $json.date }}`
7. **Establish Connections:**
   - Connect both `Schedule daily` and `Manual trigger` to `Build queue from config`.
   - Connect `Build queue from config` to `Loop keywords`.
   - Connect the loop output (second output port) of `Loop keywords` to `Wait 1s`.
   - Connect `Wait 1s` to `SerpApi Apple App Store`.
   - Connect `SerpApi Apple App Store` to `Parse position row`.
   - Connect `Parse position row` to `Append position`.
   - Connect `Append position` back to `Loop keywords` to close the loop sequence.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SerpApi Apple App Store API documentation | [SerpApi Apple App Store API](https://serpapi.com/apple-app-store-api) |
| SerpApi main website and dashboard | [SerpApi](https://serpapi.com/) |