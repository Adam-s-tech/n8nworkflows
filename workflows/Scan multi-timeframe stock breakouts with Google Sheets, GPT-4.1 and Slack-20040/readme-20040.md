Scan multi-timeframe stock breakouts with Google Sheets, GPT-4.1 and Slack

https://n8nworkflows.xyz/workflows/scan-multi-timeframe-stock-breakouts-with-google-sheets--gpt-4-1-and-slack-20040


# Scan multi-timeframe stock breakouts with Google Sheets, GPT-4.1 and Slack

### 1. Workflow Overview

This workflow is a daily post-close automated scanner designed to detect multi-timeframe stock breakouts. It reads a watchlist of stock tickers from Google Sheets, retrieves historical daily and weekly OHLCV (Open, High, Low, Close, Volume) data via an HTTP endpoint, evaluates price action against support/resistance lookbacks and volume multipliers, validates timeframe alignment, processes confirmed signals through OpenAI to obtain qualitative narrative analysis, and finally logs the results to Google Sheets while dispatching a formatted alert to Slack.

The workflow logic is categorized into the following logical blocks:
- **1.1 Input Reception & Watchlist Preparation:** Triggers execution on a daily schedule and filters active ticker records from Google Sheets.
- **1.2 Market Data Retrieval & Daily Breakout Analysis:** Fetches daily OHLCV series, executes breakout math over a 20-period lookback, and stores the intermediate payload.
- **1.3 Weekly Market Data & Multi-Timeframe Signal Construction:** Fetches weekly OHLCV series, normalizes daily and weekly data structures, runs a 12-period weekly breakout calculation, and evaluates multi-timeframe signal confirmation.
- **1.4 AI Processing & Output Normalization:** Sends confirmed breakouts to OpenAI for technical evaluation and parses/sanitizes the JSON response.
- **1.5 Persistence & Notification Delivery:** Appends the final metrics to Google Sheets and formats/posts the summary message to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Watchlist Preparation
- **Overview:** Initiates the scanning process on a 24-hour interval, reads the user's watchlist spreadsheet, and extracts only actively marked symbols.
- **Nodes Involved:** `Daily Post-Close Scanner`, `Read Breakout Watchlist`, `Prepare Active Watchlist`.
- **Node Details:**
  - **Daily Post-Close Scanner**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Initiates workflow execution once every 24 hours.
    - *Configuration:* Interval set to `24` hours.
    - *Inputs:* None (Entry point).
    - *Outputs:* `Read Breakout Watchlist` (Main).
    - *Edge Cases:* Ensure workflow is activated; otherwise, schedule triggers will not fire automatically.
  - **Read Breakout Watchlist**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Input). Reads raw rows from the Watchlist tab.
    - *Configuration:* Operation set to read list from Document ID `16imV-VJTXA6Qen2ziUWVb3C0_-Re2B-BVbWR9GK7PIQ`, Sheet name `Watchlist` (`gid=0`).
    - *Inputs:* `Daily Post-Close Scanner`.
    - *Outputs:* `Prepare Active Watchlist` (Main).
    - *Edge Cases:* Google Sheets API rate limits, invalid OAuth credentials, or missing column headers (`symbol`, `active`).
  - **Prepare Active Watchlist**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Filters records where `active` equals `true`, `1`, or `yes`, and formats tickers to uppercase.
    - *Configuration:* JavaScript logic filtering incoming items array.
    - *Inputs:* `Read Breakout Watchlist`.
    - *Outputs:* `Fetch Daily OHLCV Data` (Main).
    - *Edge Cases:* Empty symbol fields or completely empty sheets resulting in zero downstream items.

#### 2.2 Market Data Retrieval & Daily Breakout Analysis
- **Overview:** Requests daily OHLCV series for each active ticker, normalizes the response payload, and calculates support, resistance, and volume ratio thresholds over a 20-period lookback.
- **Nodes Involved:** `Fetch Daily OHLCV Data`, `Store Daily Market Data`, `Detect Daily Breakout`.
- **Node Details:**
  - **Fetch Daily OHLCV Data**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request). Requests daily candle data for each ticker.
    - *Configuration:* GET request to `http://192.168.101.63:5678/webhook-test/dummy-market-data` with query parameters `symbol={{$json.symbol}}` and `timeframe=daily`.
    - *Inputs:* `Prepare Active Watchlist`.
    - *Outputs:* `Store Daily Market Data` (Main).
    - *Edge Cases:* Unreachable HTTP endpoints, timeouts, or changes in internal API schemas.
  - **Store Daily Market Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Extracts daily data arrays from the HTTP response payload and correlates them with metadata from the source watchlist item.
    - *Configuration:* JavaScript code handling response data mapping (`response.data` or `response.body.data`).
    - *Inputs:* `Fetch Daily OHLCV Data`.
    - *Outputs:* `Fetch Weekly OHLCV Data` (Main).
    - *Edge Cases:* Missing array properties resulting in empty daily data arrays.
  - **Detect Daily Breakout**
    - *Type & Role:* `n8n-nodes-base.code` (Algorithmic Processing). Computes a 20-period high/resistance, low/support, and average volume, then checks if the current close breaches these limits alongside a $\ge 1.5\times$ volume multiplier.
    - *Configuration:* Custom JS constants: `VOLUME_MULTIPLIER = 1.5`, `LOOKBACK = 20`.
    - *Inputs:* `Normalize Market Data` (Wait, logical sequence: output connects to Build Multi Timeframe Signal).
    - *Outputs:* `Build Multi Timeframe Signal` (Main).
    - *Edge Cases:* Insufficient daily historical data (fewer than 21 records), producing an `'Insufficient daily data'` reason flag.

#### 2.3 Weekly Market Data & Multi-Timeframe Signal Construction
- **Overview:** Fetches weekly price series, merges them with daily datasets, evaluates a 12-period weekly breakout, and routes aligned multi-timeframe signals.
- **Nodes Involved:** `Fetch Weekly OHLCV Data`, `Normalize Market Data`, `Build Multi Timeframe Signal`, `Check Multi Timeframe Confirmation`.
- **Node Details:**
  - **Fetch Weekly OHLCV Data**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request). Requests weekly candle data for each symbol.
    - *Configuration:* GET request to `http://192.168.101.63:5678/webhook-test/dummy-market-data` with query parameters `symbol={{$json.symbol}}` and `timeframe=weekly`.
    - *Inputs:* `Store Daily Market Data`.
    - *Outputs:* `Normalize Market Data` (Main).
    - *Edge Cases:* API service failure or invalid symbol queries.
  - **Normalize Market Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Combines stored daily datasets with incoming weekly HTTP responses into a unified JSON object containing both timeframes.
    - *Configuration:* JavaScript array mapping matching indices from upstream nodes.
    - *Inputs:* `Fetch Weekly OHLCV Data`.
    - *Outputs:* `Detect Daily Breakout` (Main).
    - *Edge Cases:* Index mismatch if item lengths differ between upstream storage and current execution items.
  - **Build Multi Timeframe Signal**
    - *Type & Role:* `n8n-nodes-base.code` (Algorithmic Processing). Evaluates weekly OHLCV series over a 12-period lookback with a $1.5\times$ volume threshold, returning structural signal blocks.
    - *Configuration:* Custom JS constants: `VOLUME_MULTIPLIER = 1.5`, `LOOKBACK = 12`.
    - *Inputs:* `Detect Daily Breakout`.
    - *Outputs:* `Check Multi Timeframe Confirmation` (Main).
    - *Edge Cases:* Fewer than 13 weekly records triggers an `'Insufficient weekly data'` warning.
  - **Check Multi Timeframe Confirmation**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional Router). Evaluates whether signals meet workflow parameters before advancing.
    - *Configuration:* Condition checks `{{ $json.signal.confirmed }}` equals `false` (Note: configuration evaluates unconfirmed statuses; ensure expression mapping matches expected boolean output).
    - *Inputs:* `Build Multi Timeframe Signal`.
    - *Outputs:* `Generate AI Breakout Analysis` (Main - True branch).
    - *Edge Cases:* Logical misconfigurations causing false-positive or false-negative filtering.

#### 2.4 AI Processing & Output Normalization
- **Overview:** Submits confirmed technical parameters to OpenAI to generate an analytical narrative, risk notes, and confidence scores, then parses raw model outputs.
- **Nodes Involved:** `Generate AI Breakout Analysis`, `Normalize AI Analysis`.
- **Node Details:**
  - **Generate AI Breakout Analysis**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Integration / Language Model). Prompts the LLM with breakout metrics.
    - *Configuration:* Model ID set to `gpt-4.1-nano`. Built-in tools disabled. Prompt template injects symbol, company, daily/weekly values, and confidence scores, enforcing JSON-only output with specific field requirements (`confidence_score`, `confidence_level`, `narrative`, `key_confirmation`, `risk_note`).
    - *Inputs:* `Check Multi Timeframe Confirmation`.
    - *Outputs:* `Normalize AI Analysis` (Main).
    - *Edge Cases:* OpenAI API rate limits, authentication failures, or token limit issues.
  - **Normalize AI Analysis**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Sanitizes LLM outputs by stripping markdown code fences (` ```json `), parsing JSON strings safely, and implementing fallback defaults if parsing fails.
    - *Configuration:* JavaScript robust string-trimming and `JSON.parse` wrapper blocks.
    - *Inputs:* `Generate AI Breakout Analysis`.
    - *Outputs:* `Log Confirmed Breakout` (Main).
    - *Edge Cases:* Malformed non-JSON strings returned by the LLM falling back to raw text narratives.

#### 2.5 Persistence & Notification Delivery
- **Overview:** Appends structured metrics and AI analysis to Google Sheets and formats a multi-line alert for Slack delivery.
- **Nodes Involved:** `Log Confirmed Breakout`, `Prepare Slack Breakout Alert`, `Send Slack Breakout Alert`.
- **Node Details:**
  - **Log Confirmed Breakout**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Data Output). Appends rows to the logging spreadsheet.
    - *Configuration:* Operation `append` on Document ID `16imV-VJTXA6Qen2ziUWVb3C0_-Re2B-BVbWR9GK7PIQ`, Sheet name `Breakout Signals` (`gid=561818559`). Maps fields like `symbol`, `timestamp` (`{{$now}}`), `daily_close`, `ai_narrative`, `confidence_level`, etc.
    - *Inputs:* `Normalize AI Analysis`.
    - *Outputs:* `Prepare Slack Breakout Alert` (Main).
    - *Edge Cases:* Missing column headers in the target sheet matching the property mapping keys.
  - **Prepare Slack Breakout Alert**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation). Compiles structured alert text containing symbol details, daily/weekly metrics, volume ratios, key confirmation notes, and AI narratives into a single string.
    - *Configuration:* Array map joining message strings with newline delimiters (`\n`).
    - *Inputs:* `Log Confirmed Breakout`.
    - *Outputs:* `Send Slack Breakout Alert` (Main).
    - *Edge Cases:* Undefined upstream variables rendering `N/A` text blocks.
  - **Send Slack Breakout Alert**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging Integration). Posts the generated message payload to a Slack channel.
    - *Configuration:* Text set to `={{$json.slack_message}}`, channel ID selected as `C0B1LNY15GW` (`n8n-workflow-testing`).
    - *Inputs:* `Prepare Slack Breakout Alert`.
    - *Outputs:* None (Terminal node).
    - *Edge Cases:* Slack authentication errors, missing bot permissions, or archived destination channels.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Daily Post-Close Scanner` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily | None | `Read Breakout Watchlist` | Scanner Trigger and Watchlist Preparation |
| `Read Breakout Watchlist` | `n8n-nodes-base.googleSheets` | Reads watchlist records from Google Sheets | `Daily Post-Close Scanner` | `Prepare Active Watchlist` | Scanner Trigger and Watchlist Preparation |
| `Prepare Active Watchlist` | `n8n-nodes-base.code` | Filters active ticker records | `Read Breakout Watchlist` | `Fetch Daily OHLCV Data` | Scanner Trigger and Watchlist Preparation |
| `Fetch Daily OHLCV Data` | `n8n-nodes-base.httpRequest` | Requests daily candle data | `Prepare Active Watchlist` | `Store Daily Market Data` | Daily Market Data Retrieval |
| `Store Daily Market Data` | `n8n-nodes-base.code` | Stores daily OHLCV series with metadata | `Fetch Daily OHLCV Data` | `Fetch Weekly OHLCV Data` | Daily Market Data Retrieval |
| `Fetch Weekly OHLCV Data` | `n8n-nodes-base.httpRequest` | Requests weekly candle data | `Store Daily Market Data` | `Normalize Market Data` | Weekly Market Data and Normalization |
| `Normalize Market Data` | `n8n-nodes-base.code` | Normalizes and merges daily/weekly payloads | `Fetch Weekly OHLCV Data` | `Detect Daily Breakout` | Weekly Market Data and Normalization |
| `Detect Daily Breakout` | `n8n-nodes-base.code` | Calculates daily price breakouts and volume ratios | `Normalize Market Data` | `Build Multi Timeframe Signal` | Weekly Market Data and Normalization |
| `Build Multi Timeframe Signal` | `n8n-nodes-base.code` | Calculates weekly breakout signals | `Detect Daily Breakout` | `Check Multi Timeframe Confirmation` | Multi Timeframe Confirmation |
| `Check Multi Timeframe Confirmation` | `n8n-nodes-base.if` | Routes confirmed multi-timeframe signals | `Build Multi Timeframe Signal` | `Generate AI Breakout Analysis` | Multi Timeframe Confirmation |
| `Generate AI Breakout Analysis` | `@n8n/n8n-nodes-langchain.openAi` | Generates AI analysis via OpenAI LLM | `Check Multi Timeframe Confirmation` | `Normalize AI Analysis` | AI Breakout Analysis |
| `Normalize AI Analysis` | `n8n-nodes-base.code` | Parses and sanitizes OpenAI JSON responses | `Generate AI Breakout Analysis` | `Log Confirmed Breakout` | AI Breakout Analysis |
| `Log Confirmed Breakout` | `n8n-nodes-base.googleSheets` | Logs confirmed breakout metrics to Google Sheets | `Normalize AI Analysis` | `Prepare Slack Breakout Alert` | Signal Logging and Alert Delivery |
| `Prepare Slack Breakout Alert` | `n8n-nodes-base.code` | Formats final Slack alert message | `Log Confirmed Breakout` | `Send Slack Breakout Alert` | Signal Logging and Alert Delivery |
| `Send Slack Breakout Alert` | `n8n-nodes-base.slack` | Posts breakout notifications to Slack | `Prepare Slack Breakout Alert` | None | Signal Logging and Alert Delivery |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Daily Post-Close Scanner`. Set the interval to every `24` hours.
2. **Add Watchlist Reader:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Read Breakout Watchlist`. Configure it with valid Google OAuth2 credentials, set the operation to read, specify your spreadsheet ID, and select the `Watchlist` sheet. Connect the trigger output to this node.
3. **Filter Active Symbols:**
   - Add a **Code** node named `Prepare Active Watchlist`. Insert JavaScript code to filter rows where `active` equals `true`, `1`, or `yes`, formatting the `symbol` string to uppercase. Connect `Read Breakout Watchlist` here.
4. **Fetch Daily Data:**
   - Add an **HTTP Request** node named `Fetch Daily OHLCV Data`. Configure the method as `GET`, URL as `http://192.168.101.63:5678/webhook-test/dummy-market-data`, and add query parameters `symbol` (`={{$json.symbol}}`) and `timeframe` (`daily`). Connect `Prepare Active Watchlist` here.
5. **Store Daily Data:**
   - Add a **Code** node named `Store Daily Market Data`. Insert code extracting `response.data` or `response.body.data` and binding it alongside source metadata. Connect `Fetch Daily OHLCV Data` here.
6. **Fetch Weekly Data:**
   - Add an **HTTP Request** node named `Fetch Weekly OHLCV Data`. Configure the method as `GET`, URL as `http://192.168.101.63:5678/webhook-test/dummy-market-data`, and add query parameters `symbol` (`={{$json.symbol}}`) and `timeframe` (`weekly`). Connect `Store Daily Market Data` here.
7. **Normalize Market Data:**
   - Add a **Code** node named `Normalize Market Data` to merge the stored daily structure with incoming weekly items. Connect `Fetch Weekly OHLCV Data` here.
8. **Detect Daily Breakout:**
   - Add a **Code** node named `Detect Daily Breakout` containing lookback algorithms (20 periods, $1.5\times$ volume multiplier) to evaluate bullish/bearish daily conditions. Connect `Normalize Market Data` here.
9. **Build Multi-Timeframe Signal:**
   - Add a **Code** node named `Build Multi Timeframe Signal` applying weekly lookback calculations (12 periods, $1.5\times$ volume multiplier). Connect `Detect Daily Breakout` here.
10. **Add Confirmation Router:**
    - Add an **If** node (`n8n-nodes-base.if`) named `Check Multi Timeframe Confirmation`. Configure condition rules matching your signal confirmation parameters. Connect `Build Multi Timeframe Signal` here.
11. **Configure OpenAI Integration:**
    - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.openAi`) named `Generate AI Breakout Analysis`. Configure OpenAI API credentials, set model ID to `gpt-4.1-nano`, and supply the analytical prompt template requesting strict JSON output containing `confidence_score`, `confidence_level`, `narrative`, `key_confirmation`, and `risk_note`. Connect the true branch of `Check Multi Timeframe Confirmation` here.
12. **Normalize AI Output:**
    - Add a **Code** node named `Normalize AI Analysis` to sanitize incoming markdown code blocks and parse JSON safely. Connect `Generate AI Breakout Analysis` here.
13. **Log Confirmed Signals:**
    - Add a **Google Sheets** node named `Log Confirmed Breakout`. Configure spreadsheet ID, select the `Breakout Signals` sheet tab, and map output expressions (`$json.symbol`, `$json.daily_signal.close`, `$now`, etc.) to corresponding column fields. Connect `Normalize AI Analysis` here.
14. **Prepare Slack Message:**
    - Add a **Code** node named `Prepare Slack Breakout Alert` to assemble the multi-line alert string. Connect `Log Confirmed Breakout` here.
15. **Send Slack Notification:**
    - Add a **Slack** node (`n8n-nodes-base.slack`) named `Send Slack Breakout Alert`. Configure Slack OAuth2 credentials, select resource channel posting, and assign the text expression `={{$json.slack_message}}`. Connect `Prepare Slack Breakout Alert` here as the final terminal node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Dummy Market Data Endpoint Warning** | The workflow currently targets an internal dummy market-data webhook (`http://192.168.101.63:5678/webhook-test/dummy-market-data`). Replace this endpoint with a live financial data provider API before production deployment. |
| **Trading Execution Disclaimer** | The workflow functions strictly as a breakout analysis and monitoring alert system; it does not execute automated trades. |
| **Automation & Customization Support** | Implementation, integration, and custom trading automation services can be requested via **WeblineIndia**. |