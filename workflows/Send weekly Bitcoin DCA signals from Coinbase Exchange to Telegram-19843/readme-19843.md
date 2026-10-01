Send weekly Bitcoin DCA signals from Coinbase Exchange to Telegram

https://n8nworkflows.xyz/workflows/send-weekly-bitcoin-dca-signals-from-coinbase-exchange-to-telegram-19843


# Send weekly Bitcoin DCA signals from Coinbase Exchange to Telegram

### 1. Workflow Overview

This workflow automates the generation and delivery of weekly Dollar-Cost Averaging (DCA) investment signals based on moving average market regimes. Designed for crypto investors monitoring assets on Coinbase, it evaluates market conditions weekly and delivers actionable insights directly to a Telegram chat without executing any live trades.

The logic is grouped into the following functional blocks:
- **1.1 Schedule & Configuration:** Triggers the workflow execution weekly and injects customizable parameters (such as target trading pairs, base investment amounts, moving-average windows, multipliers, and delivery destinations).
- **1.2 Data Acquisition:** Fetches historical daily candle data from the public Coinbase Exchange REST API.
- **1.3 Signal Computation & Formatting:** Processes the raw candle data to calculate the moving average, assesses whether the market is bullish or bearish relative to the threshold, applies asset multipliers, and compiles a structured HTML notification message.
- **1.4 Notification Delivery:** Sends the formatted DCA signal to the designated Telegram chat via the Telegram Bot API.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the workflow execution on a scheduled basis and defines core operational variables.
- **Nodes Involved:** 
  - `Every Monday 09:00`
  - `Configuration`

##### Node Details: Every Monday 09:00
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (v1.4) — Acts as the automated entry point for the workflow.
- **Configuration Choices:** Configured to trigger every 1 week(s) on Mondays at 09:00 AM (minute 0).
- **Key Expressions or Variables:** None.
- **Input / Output Connections:** Input: None (Trigger) | Output: Connected to `Configuration`.
- **Version-Specific Requirements:** Version 1.4 or higher.
- **Edge Cases / Potential Failures:** None typical, dependent on n8n instance uptime.
- **Sub-workflow Reference:** None.

##### Node Details: Configuration
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.5) — Sets hardcoded variables used downstream for API requests, financial calculations, and messaging routing.
- **Configuration Choices:** Manual assignment mode, suppressing other fields. 
  - `PRODUCT_ID`: `BTC-EUR`
  - `DCA_AMOUNT_EUR`: `20`
  - `MA_DAYS`: `200`
  - `MULTIPLIER_ABOVE_MA`: `1.5`
  - `MULTIPLIER_BELOW_MA`: `0.5`
  - `TELEGRAM_CHAT_ID`: `YOUR_CHAT_ID`
- **Key Expressions or Variables:** Static values defined within node parameters.
- **Input / Output Connections:** Input: `Every Monday 09:00` | Output: Connected to `Get daily candles (public API)`.
- **Version-Specific Requirements:** Version 3.5 or higher.
- **Edge Cases / Potential Failures:** Incorrect type assignments (e.g., passing strings instead of numbers for amounts or days) can disrupt downstream mathematical operations.
- **Sub-workflow Reference:** None.

---

#### 2.2 Data Acquisition
- **Overview:** Requests historical market data from an external public API endpoint using parameters established in the configuration block.
- **Nodes Involved:**
  - `Get daily candles (public API)`

##### Node Details: Get daily candles (public API)
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.5) — Performs an HTTP GET request to pull historical pricing data.
- **Configuration Choices:** 
  - URL constructed dynamically using `PRODUCT_ID`.
  - Timeout set to 20,000ms.
  - Response format set to text.
  - Query parameters include `granularity=86400` (daily candles).
  - Maximum retries set to 3 with a 3,000ms wait between tries.
- **Key expressions or variables:** `=https://api.exchange.coinbase.com/products/{{ $json.PRODUCT_ID }}/candles`
- **Input / Output Connections:** Input: `Configuration` | Output: Connected to `Compute MA200 signal`.
- **Version-Specific Requirements:** Version 4.5 or higher.
- **Edge Cases / Potential Failures:** Network timeouts, API rate limits, invalid product identifiers causing 404/400 errors from Coinbase.
- **Sub-workflow Reference:** None.

---

#### 2.3 Signal Computation & Formatting
- **Overview:** Parses raw candle data, computes the moving average over the specified window, evaluates the market regime, calculates the adjusted DCA amount, and formats the final notification payload.
- **Nodes Involved:**
  - `Compute MA200 signal`
  - `Build message`

##### Node Details: Compute MA200 signal
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Executes custom JavaScript to process financial arrays, calculate moving averages, and generate market signals.
- **Configuration Choices:** Mode set to "Run Once for All Items" using JavaScript.
- **Key expressions or variables:** Pulls configuration values via `$('Configuration').first().json` and raw candles from the preceding HTTP node. Dynamically calculates closing prices, moving averages, ratios, and final EUR amounts.
- **Input / Output Connections:** Input: `Get daily candles (public API)` | Output: Connected to `Build message`.
- **Version-Specific Requirements:** Version 2 or higher.
- **Edge Cases / Potential Failures:** Throws an explicit error if the returned candle array length is smaller than the configured `MA_DAYS` threshold. Malformed JSON payloads from the API will trigger parsing errors.
- **Sub-workflow Reference:** None.

##### Node Details: Build message
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Sanitizes and formats the calculation output into a structured HTML message suitable for Telegram.
- **Configuration Choices:** Mode set to "Run Once for All Items" using JavaScript. Includes helper functions for HTML entity escaping and number localization.
- **Key expressions or variables:** Reads JSON payload fields (`product_id`, `date`, `last_close`, `ma_days`, `ma_value`, `ratio`, `regime`, `multiplier`, `amount_eur`) and evaluates conditional logic for paused investments (`amount_eur < 1`).
- **Input / Output Connections:** Input: `Compute MA200 signal` | Output: Connected to `Telegram`.
- **Version-Specific Requirements:** Version 2 or higher.
- **Edge Cases / Potential Failures:** Expression failure if required input properties are missing.
- **Sub-workflow Reference:** None.

---

#### 2.4 Notification Delivery
- **Overview:** Transmits the generated HTML-formatted DCA signal message to the configured Telegram chat.
- **Nodes Involved:**
  - `Telegram`

##### Node Details: Telegram
- **Type and Technical Role:** `n8n-nodes-base.telegram` (v1.2) — Communicates with the Telegram Bot API to deliver messages to users.
- **Configuration Choices:** 
  - Resource: `message`
  - Operation: `sendMessage`
  - Additional fields: Parse mode set to `HTML`, attribution append set to `false`.
- **Key expressions or variables:** 
  - Text: `={{ $json.text }}`
  - Chat ID: `={{ $("Configuration").first().json.TELEGRAM_CHAT_ID }}`
- **Input / Output Connections:** Input: `Build message` | Output: None (Terminal node).
- **Version-Specific Requirements:** Version 1.2 or higher. Requires valid Telegram Bot API credentials.
- **Edge Cases / Potential Failures:** Invalid Telegram Bot Token, incorrect Chat ID, or sending messages to a chat where the bot has not been initiated/authorized. API rate limiting by Telegram if spammed.
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Monday 09:00` | `scheduleTrigger` | Triggers the workflow every Monday at 09:00 AM | None | `Configuration` | ## 📊 Weekly Bitcoin DCA signal (MA200) → Telegram<br><br>Every Monday at 9:00, this workflow reads 300 daily candles from the **public** Coinbase Exchange API (no key), computes the 200-day moving average and sends you the suggested DCA amount on Telegram:<br>- price **above** MA200 → base amount × 1.5<br>- price **below** MA200 → base amount × 0.5 (0 = pause)<br><br>### Setup<br>1. Create a Telegram bot with @BotFather and add a **Telegram API** credential to the Telegram node.<br>2. In **Configuration**, set `TELEGRAM_CHAT_ID`, `PRODUCT_ID` (BTC-EUR, ETH-EUR, BTC-USD…), `DCA_AMOUNT_EUR` (base amount, any quote currency), `MA_DAYS` and both multipliers.<br>3. Execute once to test, then activate.<br><br>No order is placed and no account access is needed. Not financial advice. |
| `Configuration` | `set` | Sets global configuration variables for asset pair, amounts, MA days, multipliers, and chat ID | `Every Monday 09:00` | `Get daily candles (public API)` | ## 📊 Weekly Bitcoin DCA signal (MA200) → Telegram<br><br>Every Monday at 9:00, this workflow reads 300 daily candles from the **public** Coinbase Exchange API (no key), computes the 200-day moving average and sends you the suggested DCA amount on Telegram:<br>- price **above** MA200 → base amount × 1.5<br>- price **below** MA200 → base amount × 0.5 (0 = pause)<br><br>### Setup<br>1. Create a Telegram bot with @BotFather and add a **Telegram API** credential to the Telegram node.<br>2. In **Configuration**, set `TELEGRAM_CHAT_ID`, `PRODUCT_ID` (BTC-EUR, ETH-EUR, BTC-USD…), `DCA_AMOUNT_EUR` (base amount, any quote currency), `MA_DAYS` and both multipliers.<br>3. Execute once to test, then activate.<br><br>No order is placed and no account access is needed. Not financial advice. |
| `Get daily candles (public API)` | `httpRequest` | Fetches historical daily candle data from the Coinbase Exchange API | `Configuration` | `Compute MA200 signal` | ## 📊 Weekly Bitcoin DCA signal (MA200) → Telegram<br><br>Every Monday at 9:00, this workflow reads 300 daily candles from the **public** Coinbase Exchange API (no key), computes the 200-day moving average and sends you the suggested DCA amount on Telegram:<br>- price **above** MA200 → base amount × 1.5<br>- price **below** MA200 → base amount × 0.5 (0 = pause)<br><br>### Setup<br>1. Create a Telegram bot with @BotFather and add a **Telegram API** credential to the Telegram node.<br>2. In **Configuration**, set `TELEGRAM_CHAT_ID`, `PRODUCT_ID` (BTC-EUR, ETH-EUR, BTC-USD…), `DCA_AMOUNT_EUR` (base amount, any quote currency), `MA_DAYS` and both multipliers.<br>3. Execute once to test, then activate.<br><br>No order is placed and no account access is needed. Not financial advice. |
| `Compute MA200 signal` | `code` | Computes moving average and applies market regime multipliers | `Get daily candles (public API)` | `Build message` | |
| `Build message` | `code` | Formats the calculated data into an HTML message string for Telegram | `Compute MA200 signal` | `Telegram` | |
| `Telegram` | `telegram` | Sends the notification message via the Telegram Bot API | `Build message` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Configure the rule interval to trigger every week on Day 1 (Monday) at hour 9, minute 0.

2. **Create the Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Set mode to **Manual Assignment**.
   - Add the following assignments:
     - `PRODUCT_ID` (String): `BTC-EUR`
     - `DCA_AMOUNT_EUR` (Number): `20`
     - `MA_DAYS` (Number): `200`
     - `MULTIPLIER_ABOVE_MA` (Number): `1.5`
     - `MULTIPLIER_BELOW_MA` (Number): `0.5`
     - `TELEGRAM_CHAT_ID` (String): `YOUR_CHAT_ID`
   - Uncheck "Include Other Fields".
   - Connect `Every Monday 09:00` to this node.

3. **Create the HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Set Method to `GET`.
   - Set URL to `=https://api.exchange.coinbase.com/products/{{ $json.PRODUCT_ID }}/candles`.
   - Add Query Parameter: Name `granularity`, Value `86400`.
   - Set Options: Timeout to `20000`, Response Format to `Text`.
   - Enable `Retry On Fail` (Max Tries: 3, Wait Between Tries: 3000ms).
   - Connect `Configuration` to this node.

4. **Create the Signal Computation Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Set Mode to **Run Once for All Items** and Language to **JavaScript**.
   - Insert the calculation logic parsing candles, checking data validity against `MA_DAYS`, calculating moving averages, and determining the regime multipliers.
   - Connect `Get daily candles (public API)` to this node.

5. **Create the Message Formatting Code Node:**
   - Add a second **Code** node (`n8n-nodes-base.code`).
   - Set Mode to **Run Once for All Items** and Language to **JavaScript**.
   - Insert the formatting logic to escape HTML special characters, format decimal numbers, and compile the final message body.
   - Connect `Compute MA200 signal` to this node.

6. **Create the Telegram Delivery Node:**
   - Add a **Telegram** node (`n8n-nodes-base.telegram`).
   - Select Resource: `Message`, Operation: `Send`.
   - Configure credentials: Add a new **Telegram API** credential using your Bot Token from `@BotFather`.
   - Set Text expression to `={{ $json.text }}`.
   - Set Chat ID expression to `={{ $("Configuration").first().json.TELEGRAM_CHAT_ID }}`.
   - Set Additional Fields -> Parse Mode to `HTML` and Append Attribution to `false`.
   - Connect `Build message` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full protocol with real numbers (17 live orders, 1.2% taker fees, a DCA that silently stopped) | [Utikcoin Protocol Article](https://utikcoin.com/article/dca-automatise-coinbase-protocole) |
| French README and source repository | [GitHub Repository HATEM6584](https://github.com/HATEM6584/n8n-dca-signal-mm200) |