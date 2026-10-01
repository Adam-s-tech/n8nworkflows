Detect NSE options anomalies with Supabase, OpenAI, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/detect-nse-options-anomalies-with-supabase--openai--google-sheets-and-gmail-20003


# Detect NSE options anomalies with Supabase, OpenAI, Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the detection, analysis, and reporting of unusual activity within NSE (National Stock Exchange) option chains. Operating on a 30-minute schedule, it scans a predefined watchlist, compares live market metrics against historical baselines stored in a database, uses artificial intelligence to interpret market signals, logs the results for record-keeping, and dispatches a consolidated executive digest via email.

The workflow logic is categorized into six functional blocks:
- **1.1 Market Scan Scheduling:** Triggers the automation execution on a routine timetable and extracts active symbols from a cloud spreadsheet watchlist.
- **1.2 Option Data Collection:** Queries an external market data endpoint for live option chain statistics and normalizes the payload attributes into uniform data structures.
- **1.3 Historical Data Analysis:** Retrieves baseline metrics from a Supabase database, computes volume ratios, open interest (OI) ratios, and statistical z-scores, and applies threshold rules to filter anomalous instruments.
- **1.4 Snapshot Persistence & AI Processing:** Stores historical snapshots of flagged contracts in Supabase and prompts an OpenAI model to generate a risk-aware qualitative interpretation.
- **1.5 Anomaly Reporting & Logging:** Persists AI-enriched findings to a database table and writes audit entries to a tracking sheet.
- **1.6 Email Notification:** Aggregates all flagged anomalies from the execution run into a structured text digest and sends the summary report through Gmail.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Market Scan Scheduling
- **Overview:** Initiates the scanning routine on a fixed interval and extracts the targeted symbols and configurations from an external spreadsheet.
- **Nodes Involved:** 
  - `Market Scanner Schedule`
  - `Read NSE Watchlist`
  - `Prepare Watchlist Symbols`

- **Node Details:**
  - **Market Scanner Schedule**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire automatically every 30 minutes.
    - *Key Expressions or Variables:* None.
    - *Input / Output Connections:* Output connects to `Read NSE Watchlist`.
    - *Version-specific Requirements:* Version 1.2.
    - *Edge Cases / Potential Failure Types:* Workflow inactivity or scheduler misconfiguration.
    - *Sub-workflow Reference:* None.

  - **Read NSE Watchlist**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads the `Watchlist` worksheet from the document ID `1moSuSzyIdsd3c3CupkOA9DxjsIA0w1gwEW3Z1OVIyiM`.
    - *Key Expressions or Variables:* None.
    - *Input / Output Connections:* Input from `Market Scanner Schedule`; output connects to `Prepare Watchlist Symbols`.
    - *Version-specific Requirements:* Version 4.5.
    - *Edge Cases / Potential Failure Types:* Google Sheets credential expiry, incorrect sheet title, or network timeout.
    - *Sub-workflow Reference:* None.

  - **Prepare Watchlist Symbols**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Executes JavaScript code to filter rows where the `enabled` attribute equals `true` (case-insensitive), mapping each item into a clean schema (`symbol`, `exchange`, `scan_mode`, `strike_count`).
    - *Key Expressions or Variables:* Uses `$input.all()`.
    - *Input / Output Connections:* Input from `Read NSE Watchlist`; output connects to `Fetch NSE Option Chain`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Malformed row properties or missing boolean flags causing all rows to drop.
    - *Sub-workflow Reference:* None.

---

#### Block 1.2: Option Data Collection
- **Overview:** Fetches option chain data for the validated instruments via an HTTP webhook endpoint and normalizes the resulting arrays into standardized individual contract items.
- **Nodes Involved:**
  - `Fetch NSE Option Chain`
  - `Normalize Option Contracts`

- **Node Details:**
  - **Fetch NSE Option Chain**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Network Request)
    - *Configuration Choices:* Issues a GET request to `http://192.168.101.63:5678/webhook-test/mock/nse-option-chain`.
    - *Key Expressions or Variables:* None.
    - *Input / Output Connections:* Input from `Prepare Watchlist Symbols`; output connects to `Normalize Option Contracts`.
    - *Version-specific Requirements:* Version 4.2.
    - *Edge Cases / Potential Failure Types:* Endpoint unavailability, schema drift in the response payload, or timeout errors.
    - *Sub-workflow Reference:* None.

  - **Normalize Option Contracts**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Iterates over the `data` array returned by the HTTP request, casting numerical string fields (strike, ltp, volume, open_interest, change_in_oi, iv, underlying_price) into proper numeric types.
    - *Key Expressions or Variables:* Uses `$input.first().json`.
    - *Input / Output Connections:* Input from `Fetch NSE Option Chain`; output connects to `Fetch Historical Option Data`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Missing root `data` attribute or null/undefined fields throwing type errors during casting.
    - *Sub-workflow Reference:* None.

---

#### Block 1.3: Historical Data Analysis
- **Overview:** Pulls historical baseline metrics from Supabase, calculates volume and open interest ratios along with statistical z-scores, and isolates contracts exceeding defined thresholds.
- **Nodes Involved:**
  - `Fetch Historical Option Data`
  - `Calculate Option Anomaly`
  - `Filter Anomalous Contracts`

- **Node Details:**
  - **Fetch Historical Option Data**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Operation)
    - *Configuration Choices:* Executes an `getAll` operation on the `option_snapshots` table filtered by `symbol` equals the current item's symbol.
    - *Key Expressions or Variables:* `={{ $json.symbol }}`
    - *Input / Output Connections:* Input from `Normalize Option Contracts`; output connects to `Calculate Option Anomaly`.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases / Potential Failure Types:* Empty historical datasets, Supabase authentication errors, or missing table schemas.
    - *Sub-workflow Reference:* None.

  - **Calculate Option Anomaly**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Computes moving averages, standard deviations, volume ratios, open interest ratios, and z-scores. Flags anomalies based on specified thresholds (volume ratio $\ge 3$, OI ratio $\ge 2$, or z-score $\ge 3$).
    - *Key Expressions or Variables:* Uses `$input.all()`.
    - *Input / Output Connections:* Input from `Fetch Historical Option Data`; output connects to `Filter Anomalous Contracts`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Division by zero when historical volume or standard deviation equals zero.
    - *Sub-workflow Reference:* None.

  - **Filter Anomalous Contracts**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether the boolean property `anomaly` is strictly true.
    - *Key Expressions or Variables:* `={{ $json.anomaly }}`
    - *Input / Output Connections:* Input from `Calculate Option Anomaly`; true branch outputs to `Save Option Snapshot`.
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failure Types:* Boolean casting issues if upstream data returns unexpected types.
    - *Sub-workflow Reference:* None.

---

#### Block 1.4: Snapshot Persistence & AI Processing
- **Overview:** Records the anomaly snapshot data back into Supabase and invokes an OpenAI large language model to generate a structured, qualitative assessment.
- **Nodes Involved:**
  - `Save Option Snapshot`
  - `Generate AI Anomaly Interpretation`

- **Node Details:**
  - **Save Option Snapshot**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Operation)
    - *Configuration Choices:* Inserts the evaluated option contract record into the `option_snapshots` table with mapped parameters (`captured_at`, `trading_date`, `symbol`, `strike`, `volume_ratio`, `z_score`, etc.).
    - *Key Expressions or Variables:* `={{ $json.captured_at }}`, `={{ $json.symbol }}`, etc.
    - *Input / Output Connections:* Input from `Filter Anomalous Contracts` (true branch); output connects to `Generate AI Anomaly Interpretation`.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases / Potential Failure Types:* Schema constraint violations or DB write timeouts.
    - *Sub-workflow Reference:* None.

  - **Generate AI Anomaly Interpretation**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI / LLM Node)
    - *Configuration Choices:* Uses model `gpt-4.1-nano` with a prompt template instructing the model to analyze option statistics and return plain text with three sections: *Observation*, *Possible interpretation*, and *Risk note*.
    - *Key Expressions or Variables:* `={{ $json.symbol }}`, `={{ $json.volume_ratio }}`, `={{ $json.z_score }}`, etc.
    - *Input / Output Connections:* Input from `Save Option Snapshot`; output connects to `Save Anomaly Analysis`.
    - *Version-specific Requirements:* Version 1.8.
    - *Edge Cases / Potential Failure Types:* API rate limits, billing errors, or model response quota exhaustion.
    - *Sub-workflow Reference:* None.

---

#### Block 1.5: Anomaly Reporting & Logging
- **Overview:** Commits the AI-enriched anomaly record to a dedicated database table and appends a row to a Google Sheets tracking log.
- **Nodes Involved:**
  - `Save Anomaly Analysis`
  - `Log Anomaly to Google Sheets`

- **Node Details:**
  - **Save Anomaly Analysis**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Operation)
    - *Configuration Choices:* Inserts rows into the `option_anomalies` table, mapping contract parameters and extracting the AI text output from various fallback paths.
    - *Key Expressions or Variables:* `={{ $('Calculate Option Anomaly').item.json.symbol }}`, `={{ $json.text || $json.output || $json.message.content || '' }}`
    - *Input / Output Connections:* Input from `Generate AI Anomaly Interpretation`; output connects to `Log Anomaly to Google Sheets`.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases / Potential Failure Types:* Unhandled JSON path resolutions for the AI output string.
    - *Sub-workflow Reference:* None.

  - **Log Anomaly to Google Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Logging)
    - *Configuration Choices:* Appends a new row to the `Anomaly Log sheet` worksheet inside document `1moSuSzyIdsd3c3CupkOA9DxjsIA0w1gwEW3Z1OVIyiM`.
    - *Key Expressions or Variables:* `={{ $json.ltp }}`, `={{ $json.ai_interpretation }}`, etc.
    - *Input / Output Connections:* Input from `Save Anomaly Analysis`; output connects to `Build Email Alert Digest`.
    - *Version-specific Requirements:* Version 4.5.
    - *Edge Cases / Potential Failure Types:* Sheet permission errors or column header mismatches.
    - *Sub-workflow Reference:* None.

---

#### Block 1.6: Email Notification
- **Overview:** Compiles all detected anomalies into a single, cohesive textual digest and sends the alert email through Gmail.
- **Nodes Involved:**
  - `Build Email Alert Digest`
  - `Send Gmail Anomaly Digest`

- **Node Details:**
  - **Build Email Alert Digest**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Collects all items in the execution stream, formats numbers and ratios, builds section templates for every detected anomaly, aggregates anomaly type counts, and returns a single summary item containing `subject`, `body`, and `anomaly_count`.
    - *Key Expressions or Variables:* Uses `$input.all()`.
    - *Input / Output Connections:* Input from `Log Anomaly to Google Sheets`; output connects to `Send Gmail Anomaly Digest`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Empty item inputs handled by returning a fallback "No anomalies detected" message.
    - *Sub-workflow Reference:* None.

  - **Send Gmail Anomaly Digest**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Communication)
    - *Configuration Choices:* Sends a plain-text email message using Gmail OAuth credentials to the specified recipient address.
    - *Key Expressions or Variables:* `={{ $json.body }}`, `={{ $json.subject }}`
    - *Input / Output Connections:* Input from `Build Email Alert Digest`; terminal node.
    - *Version-specific Requirements:* Version 2.1.
    - *Edge Cases / Potential Failure Types:* Gmail token revocation, sending limits exceeded, or incorrect recipient formatting.
    - *Sub-workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Market Scanner Schedule** | `n8n-nodes-base.scheduleTrigger` | Trigger Node | None | Read NSE Watchlist | ## Market Scan Scheduling<br><br>This section starts the NSE options monitoring workflow on the configured schedule. It reads the symbols and scanning configuration from Google Sheets and filters the watchlist to include only enabled instruments before preparing clean symbol data for the options data request. |
| **Read NSE Watchlist** | `n8n-nodes-base.googleSheets` | Data Retrieval | Market Scanner Schedule | Prepare Watchlist Symbols | ## Market Scan Scheduling<br><br>This section starts the NSE options monitoring workflow on the configured schedule. It reads the symbols and scanning configuration from Google Sheets and filters the watchlist to include only enabled instruments before preparing clean symbol data for the options data request. |
| **Prepare Watchlist Symbols** | `n8n-nodes-base.code` | Data Transformation | Read NSE Watchlist | Fetch NSE Option Chain | ## Market Scan Scheduling<br><br>This section starts the NSE options monitoring workflow on the configured schedule. It reads the symbols and scanning configuration from Google Sheets and filters the watchlist to include only enabled instruments before preparing clean symbol data for the options data request. |
| **Fetch NSE Option Chain** | `n8n-nodes-base.httpRequest` | Network Request | Prepare Watchlist Symbols | Normalize Option Contracts | ## Option Data Collection<br>This section retrieves option chain data from the configured NSE data endpoint. The returned contracts are then converted into a consistent structure containing expiry, strike, option type, price, volume, open interest, implied volatility, and underlying price for further analysis. |
| **Normalize Option Contracts** | `n8n-nodes-base.code` | Data Transformation | Fetch NSE Option Chain | Fetch Historical Option Data | ## Option Data Collection<br>This section retrieves option chain data from the configured NSE data endpoint. The returned contracts are then converted into a consistent structure containing expiry, strike, option type, price, volume, open interest, implied volatility, and underlying price for further analysis. |
| **Fetch Historical Option Data** | `n8n-nodes-base.supabase` | Database Operation | Normalize Option Contracts | Calculate Option Anomaly | ## Historical Data Analysis<br><br>This section compares the latest option activity against historical information stored in Supabase. Volume ratios, open interest ratios, and statistical z scores are calculated for each contract. Contracts meeting the configured anomaly conditions are then separated for deeper processing |
| **Calculate Option Anomaly** | `n8n-nodes-base.code` | Data Transformation | Fetch Historical Option Data | Filter Anomalous Contracts | ## Historical Data Analysis<br><br>This section compares the latest option activity against historical information stored in Supabase. Volume ratios, open interest ratios, and statistical z scores are calculated for each contract. Contracts meeting the configured anomaly conditions are then separated for deeper processing |
| **Filter Anomalous Contracts** | `n8n-nodes-base.if` | Flow Control | Calculate Option Anomaly | Save Option Snapshot | ## Historical Data Analysis<br><br>This section compares the latest option activity against historical information stored in Supabase. Volume ratios, open interest ratios, and statistical z scores are calculated for each contract. Contracts meeting the configured anomaly conditions are then separated for deeper processing |
| **Save Option Snapshot** | `n8n-nodes-base.supabase` | Database Operation | Filter Anomalous Contracts | Generate AI Anomaly Interpretation | ## Snapshot Persistence<br><br>This section preserves the detected option snapshot in Supabase and sends anomalous contracts to OpenAI for interpretation. The AI explains the unusual activity and discusses possible bullish, bearish, hedging, or event driven explanations without treating the anomaly as confirmed smart money activity. |
| **Generate AI Anomaly Interpretation** | `@n8n/n8n-nodes-langchain.openAi` | AI / LLM Node | Save Option Snapshot | Save Anomaly Analysis | ## Snapshot Persistence<br><br>This section preserves the detected option snapshot in Supabase and sends anomalous contracts to OpenAI for interpretation. The AI explains the unusual activity and discusses possible bullish, bearish, hedging, or event driven explanations without treating the anomaly as confirmed smart money activity. |
| **Save Anomaly Analysis** | `n8n-nodes-base.supabase` | Database Operation | Generate AI Anomaly Interpretation | Log Anomaly to Google Sheets | ## Anomaly Reporting and Email Notification<br>This section stores AI enhanced anomaly results in Supabase, logs all detected contracts in Google Sheets, and combines them into a single email digest. The final Gmail alert includes complete anomaly details, statistical metrics, AI interpretation, and risk information. |
| **Log Anomaly to Google Sheets** | `n8n-nodes-base.googleSheets` | Data Logging | Save Anomaly Analysis | Build Email Alert Digest | ## Anomaly Reporting and Email Notification<br>This section stores AI enhanced anomaly results in Supabase, logs all detected contracts in Google Sheets, and combines them into a single email digest. The final Gmail alert includes complete anomaly details, statistical metrics, AI interpretation, and risk information. |
| **Build Email Alert Digest** | `n8n-nodes-base.code` | Data Transformation | Log Anomaly to Google Sheets | Send Gmail Anomaly Digest | ## Anomaly Reporting and Email Notification<br>This section stores AI enhanced anomaly results in Supabase, logs all detected contracts in Google Sheets, and combines them into a single email digest. The final Gmail alert includes complete anomaly details, statistical metrics, AI interpretation, and risk information. |
| **Send Gmail Anomaly Digest** | `n8n-nodes-base.gmail` | Communication | Build Email Alert Digest | None | ## Anomaly Reporting and Email Notification<br>This section stores AI enhanced anomaly results in Supabase, logs all detected contracts in Google Sheets, and combines them into a single email digest. The final Gmail alert includes complete anomaly details, statistical metrics, AI interpretation, and risk information. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) node. Set the interval rule to trigger every 30 minutes. Name it `Market Scanner Schedule`.

2. **Add Watchlist Retrieval:**
   - Create a **Google Sheets** (`n8n-nodes-base.googleSheets`) node. Connect Google Sheets credentials.
   - Set Operation to `Get` or use custom parameters pointing to document ID `1moSuSzyIdsd3c3CupkOA9DxjsIA0w1gwEW3Z1OVIyiM` and worksheet `Watchlist`. Name it `Read NSE Watchlist`.
   - Connect `Market Scanner Schedule` output to `Read NSE Watchlist`.

3. **Filter Watchlist Symbols:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Prepare Watchlist Symbols`.
   - Insert JavaScript to filter items where `item.json.enabled` equals `'true'` (case-insensitive) and map them to `{ symbol, exchange, scan_mode, strike_count }`.
   - Connect `Read NSE Watchlist` output to `Prepare Watchlist Symbols`.

4. **Fetch Option Chain Data:**
   - Add an **HTTP Request** (`n8n-nodes-base.httpRequest`) node named `Fetch NSE Option Chain`.
   - Set method to `GET` and URL to `http://192.168.101.63:5678/webhook-test/mock/nse-option-chain`.
   - Connect `Prepare Watchlist Symbols` output to `Fetch NSE Option Chain`.

5. **Normalize Contracts:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Normalize Option Contracts`.
   - Insert JavaScript to extract contracts from `input.data` and normalize properties (`strike`, `ltp`, `volume`, `open_interest`, `change_in_oi`, `iv`, `underlying_price`) into numeric values.
   - Connect `Fetch NSE Option Chain` output to `Normalize Option Contracts`.

6. **Query Historical Data:**
   - Add a Supabase (`n8n-nodes-base.supabase`) node named `Fetch Historical Option Data`.
   - Connect Supabase credentials. Select table `option_snapshots`, operation `getAll`, and add a filter condition where `symbol` equals `={{ $json.symbol }}` with return all enabled.
   - Connect `Normalize Option Contracts` output to `Fetch Historical Option Data`.

7. **Calculate Anomalies:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Calculate Option Anomaly`.
   - Insert JavaScript to calculate volume ratio, OI ratio, and z-score, setting `anomaly` to true if `volume_ratio >= 3`, `oi_ratio >= 2`, or `z_score >= 3`.
   - Connect `Fetch Historical Option Data` output to `Calculate Option Anomaly`.

8. **Filter Anomalous Contracts:**
   - Add an **If** (`n8n-nodes-base.if`) node named `Filter Anomalous Contracts`.
   - Set condition to evaluate boolean `={{ $json.anomaly }}` equals `true`.
   - Connect `Calculate Option Anomaly` output to `Filter Anomalous Contracts`.

9. **Persist Option Snapshot:**
   - Add a Supabase (`n8n-nodes-base.supabase`) node named `Save Option Snapshot`.
   - Select table `option_snapshots` and map fields (`captured_at`, `trading_date`, `symbol`, `expiry`, `strike`, `option_type`, `ltp`, `volume`, `open_interest`, `change_in_oi`, `iv`, `underlying_price`, `source`, `avg_volume_20d`, `avg_oi_20d`, `volume_ratio`, `oi_ratio`, `z_score`, `anomaly_type`, `anomaly`) to corresponding expression inputs.
   - Connect the `true` output branch of `Filter Anomalous Contracts` to `Save Option Snapshot`.

10. **Generate AI Interpretation:**
    - Add an OpenAI (`@n8n/n8n-nodes-langchain.openAi`) node named `Generate AI Anomaly Interpretation`.
    - Configure credentials for OpenAI and select model `gpt-4.1-nano`.
    - Provide a prompt message supplying anomaly metrics and instructing the model to return plain text containing *Observation*, *Possible interpretation*, and *Risk note* sections.
    - Connect `Save Option Snapshot` output to `Generate AI Anomaly Interpretation`.

11. **Save Anomaly Analysis:**
    - Add a Supabase (`n8n-nodes-base.supabase`) node named `Save Anomaly Analysis`.
    - Select table `option_anomalies` and map fields, including `ai_interpretation` mapped to `={{ $json.text || $json.output || $json.message.content || '' }}`.
    - Connect `Generate AI Anomaly Interpretation` output to `Save Anomaly Analysis`.

12. **Log to Google Sheets:**
    - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Log Anomaly to Google Sheets`.
    - Select document ID `1moSuSzyIdsd3c3CupkOA9DxjsIA0w1gwEW3Z1OVIyiM` and sheet tab `Anomaly Log sheet`. Set operation to `append` and map columns to corresponding item properties.
    - Connect `Save Anomaly Analysis` output to `Log Anomaly to Google Sheets`.

13. **Build Email Digest:**
    - Add a **Code** (`n8n-nodes-base.code`) node named `Build Email Alert Digest`.
    - Insert JavaScript to gather all items, format numbers/ratios, construct text blocks for each anomaly, summarize anomaly type counts, and output an object containing `subject`, `body`, and `anomaly_count`.
    - Connect `Log Anomaly to Google Sheets` output to `Build Email Alert Digest`.

14. **Send Email Notification:**
    - Add a **Gmail** (`n8n-nodes-base.gmail`) node named `Send Gmail Anomaly Digest`.
    - Connect Gmail OAuth2 credentials. Set `sendTo` to the destination email address, `subject` to `={{ $json.subject }}`, `message` to `={{ $json.body }}`, and email type to text.
    - Connect `Build Email Alert Digest` output to `Send Gmail Anomaly Digest`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| NSE Options Anomaly Detection and Alert Workflow Source Details | Original workflow created using n8n with integrations for Supabase, Google Sheets, OpenAI, and Gmail. |
| WeblineIndia Support & Customization Services | For custom modifications, market data endpoint integrations, or dashboard additions, assistance can be requested via WeblineIndia. |