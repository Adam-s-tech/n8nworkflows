Reprice gold chain stock daily and announce it on WhatsApp with Google Sheets, goldapi.io and Twilio

https://n8nworkflows.xyz/workflows/reprice-gold-chain-stock-daily-and-announce-it-on-whatsapp-with-google-sheets--goldapi-io-and-twilio-20020


# Reprice gold chain stock daily and announce it on WhatsApp with Google Sheets, goldapi.io and Twilio

### 1. Workflow Overview

This workflow is a comprehensive, automated catalogue repricing and broadcasting engine designed for jewellery wholesalers and gold traders. Its core purpose is to automate the daily calculation of gold chain prices using live market rates, format customer-facing updates, filter and validate recipient lists, and obtain mandatory human approval before broadcasting announcements via WhatsApp.

The workflow logic is categorized into six functional blocks:
- **1.1 Initialization and Configuration:** Triggers the workflow execution via schedule or manual input and centralizes all operational settings, constants, and fallback demo data.
- **1.2 Stock Ingestion and Validation:** Connects to Google Sheets to ingest inventory or falls back to demo stock, rigorously filtering out rows missing mandatory attributes (e.g., weight, karat purity, or style).
- **1.3 Gold Rate Acquisition and Pricing Engine:** Fetches live 24k gold rates from goldapi.io (or uses a demo rate), validates rate plausibility against historical moves, and calculates precise item-level wholesale prices based on weight, purity, making charges, and margins.
- **1.4 Catalogue and Announcement Synthesis:** Transforms repriced rows into structured customer catalogs, internal inventory lists, CSV outputs, and concise delta announcements highlighting inventory changes (new, back in stock, last few, sold out).
- **1.5 Customer List Ingestion and Broadcast Planning:** Ingests customer contact lists from Google Sheets or demo data, filters out opted-out, duplicate, or invalid numbers, and enforces strict broadcast caps.
- **1.6 Approval, Dispatch, and Audit Logging:** Requests owner approval via an interactive WhatsApp link, handles timeouts, dispatches approved messages via Twilio, and optionally writes updated prices and quantities back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Configuration
- **Overview:** This block serves as the single entry point for the workflow, establishing execution schedules and providing a unified configuration node that supplies parameters, business rules, and fallback data to all downstream nodes.
- **Nodes Involved:** `Every weekday, 08:00`, `Run it now`, `config`, `stock sheet configured?`
- **Node Details:**
  - `Every weekday, 08:00`
    - *Type and Technical Role:* Schedule Trigger (Cron expression `0 8 * * 1-5`).
    - *Configuration choices:* Fires automatically every Monday through Friday at 08:00.
    - *Input/Output:* No inputs; outputs execution trigger to `config`.
    - *Edge cases:* System downtime during scheduled hours; missed executions must be handled by n8n queue settings.
  - `Run it now`
    - *Type and Technical Role:* Manual Trigger.
    - *Configuration choices:* Allows on-demand execution for testing or manual runs.
    - *Input/Output:* No inputs; outputs execution trigger to `config`.
  - `config`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Central configuration repository holding business identity, purity fractions, style-specific making charges, wholesale margins, rounding rules, API keys, sheet IDs, Twilio parameters, safety switches, and comprehensive demo datasets (`DEMO_STOCK`, `DEMO_CUSTOMERS`, `DEMO_MESSAGES`). Evaluates runtime operational flags (`sheet_enabled`, `gold_api_enabled`, `ai_enabled`, `send_enabled`, `write_enabled`, `mode`).
    - *Key expressions:* Maps incoming payloads and appends global `SETTINGS` object to every item.
    - *Input/Output:* Inputs from triggers; outputs to `stock sheet configured?`.
    - *Edge cases:* Invalid JavaScript syntax or malformed configuration keys; invalid types will default gracefully using logical OR operators.
  - `stock sheet configured?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $('config').first().json.sheet_enabled }}` as boolean `true`.
    - *Input/Output:* Input from `config`; outputs True to `[cred] Sheets - read the stock`, False to `use the demo stock`.

---

#### 1.2 Stock Ingestion and Validation
- **Overview:** This block retrieves raw inventory data either from a connected Google Sheets document or via built-in fallback demo data, then performs strict data validation to ensure no item is priced on incomplete or erroneous attributes.
- **Nodes Involved:** `[cred] Sheets - read the stock`, `stock-collect`, `stock-demo`, `check the stock rows`
- **Node Details:**
  - `[cred] Sheets - read the stock`
    - *Type and Technical Role:* Google Sheets Node (Read operation).
    - *Configuration choices:* Reads document ID and sheet name dynamically from configuration variables (`sheet_id` and `stock_tab`). Configured with `alwaysOutputData: true`.
    - *Input/Output:* Input from `stock sheet configured?` (True branch); outputs raw sheet rows to `collect the stock`.
    - *Edge cases:* Google Sheets API authentication failure, invalid sheet ID, or missing tab name resulting in execution errors.
  - `stock-collect`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Maps sheet rows, filters out rows lacking a SKU, and aggregates them into a unified array object labeled `{ source: 'stock sheet', rows }`.
    - *Input/Output:* Input from Google Sheets read node; outputs aggregated stock object to `check the stock rows`.
  - `stock-demo`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Returns built-in demo inventory items from `cfg.demo_stock` when no production sheet is configured.
    - *Input/Output:* Input from `stock sheet configured?` (False branch); outputs demo stock object to `check the stock rows`.
  - `check the stock rows`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Iterates through raw rows, executing validation checks (`readRow`, `priceChain`, `money`, `chainLabel`). Evaluates SKUs, style whitelist, karat purity whitelist, positive weight, and non-negative quantity. Holds back invalid rows with descriptive reasons (`held` array) and forwards valid rows (`ok` array). Calculates median historical rate (`last_rate`).
    - *Input/Output:* Inputs from `stock-collect` or `stock-demo`; outputs validation summary object to `gold rate configured?`.

---

#### 1.3 Gold Rate Acquisition and Pricing Engine
- **Overview:** This block acquires the live spot price of 24k gold per gram from goldapi.io (or uses a demo rate), validates price movement against historical thresholds to prevent catastrophic pricing errors, and calculates precise retail/wholesale prices for each valid inventory item.
- **Nodes Involved:** `gold rate configured?`, `Gold rate - today's price`, `rate-demo`, `read the gold rate`, `rate-fail`, `price every chain`, `rate looks right?`, `rate-wrong`
- **Node Details:**
  - `gold rate configured?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $('config').first().json.gold_api_enabled }}` as boolean `true`.
    - *Input/Output:* Input from `check the stock rows`; outputs True to `Gold rate - today's price`, False to `use the demo gold rate`.
  - `Gold rate - today's price`
    - *Type and Technical Role:* HTTP Request Node.
    - *Configuration choices:* Requests URL dynamically combining `gold_api_base`, `/XAU/`, and `currency`. Sends custom headers `x-access-token` (using `gold_api_key`) and `Accept: application/json`. Timeout set to 20,000ms. Configured with `onError: continueErrorOutput`.
    - *Input/Output:* Input from `gold rate configured?` (True); outputs successful HTTP response to `read the gold rate` or error output to `rate-fail`.
    - *Edge cases:* HTTP 401/403 authentication errors, rate-limiting, network timeouts, or invalid JSON payloads.
  - `rate-fail`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Captures HTTP failure outputs and halts execution with a descriptive stop reason (`_outcome: 'stopped — nothing priced, nothing sent'`).
    - *Input/Output:* Input from error path of HTTP Request; terminates branch.
  - `use the demo gold rate`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Supplies fallback `demo_gold_rate` from configuration and tags source as a non-market demo rate.
    - *Input/Output:* Input from `gold rate configured?` (False); outputs demo rate object to `price every chain`.
  - `read the gold rate`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Parses goldapi.io responses supporting multiple legacy schemas and troy ounce conversions, normalizing output into rounded 24k per-gram spot values (`rate_24k`, `currency`, `rate_source`).
    - *Input/Output:* Input from HTTP Request; outputs normalized rate object to `price every chain`.
  - `price every chain`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Applies pricing formula: `metal = weight_g * purity * rate_24k`, `making = weight_g * making_per_gram`, `gross = (metal + making) * (1 + margin_pct / 100)`, rounded according to `round_to`. Calculates percentage movement against historical median rate (`last_rate`), evaluating against `MAX_RATE_MOVE_PCT`. Determines inventory status changes (`new`, `sold out`, `back in stock`, `last few`).
    - *Input/Output:* Inputs from rate nodes and stock check node; outputs priced chain array and rate validation flags to `rate looks right?`.
  - `rate looks right?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $json.rate_ok }}` as boolean `true`.
    - *Input/Output:* Input from `price every chain`; outputs True to `build the catalogue`, False to `rate-wrong`.
  - `rate-wrong`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Halts execution when rate movement exceeds safety limits or currency mismatches occur, preventing erroneous pricing broadcasts.
    - *Input/Output:* Input from `rate looks right?` (False); terminates branch.

---

#### 1.4 Catalogue and Announcement Synthesis
- **Overview:** This block takes the fully priced inventory list and generates multiple synchronized outputs: a clean customer-facing WhatsApp catalogue, an internal count sheet, a CSV export string, and a concise delta announcement focusing strictly on inventory changes.
- **Nodes Involved:** `build the catalogue`, `customer list configured?`
- **Node Details:**
  - `build the catalogue`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Generates 22k derived rates, formats structured WhatsApp catalog text grouped by style, formats plain-text internal inventory reports, generates CSV export data strings with CSV-compliant escaping, and builds delta announcements (`new`, `back in stock`, `last few`, `sold out`). Enforces WhatsApp message length truncation (< 1,500 characters).
    - *Input/Output:* Input from `rate looks right?` (True); outputs synthesized catalog, announcements, and statistics to `customer list configured?`.
    - *Edge cases:* Extremely large inventory catalogues exceeding WhatsApp character limits (handled by truncation logic).
  - `customer list configured?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $('config').first().json.sheet_enabled }}` as boolean `true`.
    - *Input/Output:* Input from `build the catalogue`; outputs True to `[cred] Sheets - read the customers`, False to `use the demo customers`.

---

#### 1.5 Customer List Ingestion and Broadcast Planning
- **Overview:** This block loads recipient contact data from Google Sheets or demo datasets, validates telephone formats to E.164 standards, filters out opted-out or duplicate contacts, enforces broadcast caps, and prepares the owner's verification summary message.
- **Nodes Involved:** `[cred] Sheets - read the customers`, `cust-collect`, `cust-demo`, `plan the broadcast`, `sending on?`
- **Node Details:**
  - `[cred] Sheets - read the customers`
    - *Type and Technical Role:* Google Sheets Node (Read operation).
    - *Configuration choices:* Reads customer sheet rows dynamically using `sheet_id` and `customers_tab`. Configured with `alwaysOutputData: true`.
    - *Input/Output:* Input from `customer list configured?` (True); outputs raw customer rows to `collect the customers`.
  - `cust-collect`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Filters out rows lacking WhatsApp numbers and aggregates customer records into `{ source: 'customers sheet', customers }`.
    - *Input/Output:* Input from Google Sheets read node; outputs aggregated customer object to `plan the broadcast`.
  - `cust-demo`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Loads built-in demo customer records from `cfg.demo_customers`.
    - *Input/Output:* Input from `customer list configured?` (False); outputs demo customer object to `plan the broadcast`.
  - `plan the broadcast`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Validates phone numbers against E.164 formatting standards (`e164`), checks opt-out flags (`opted_out`), filters duplicate numbers using a `Set`, and enforces `BROADCAST_CAP`. In `test` mode, redirects all messages to `test_whatsapp` up to `test_cap` (3 messages) with `[TEST]` tags. Constructs owner approval message payload.
    - *Input/Output:* Inputs from customer collection nodes and catalog builder; outputs planned broadcast list and owner approval payload to `sending on?`.
  - `sending on?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $('config').first().json.send_enabled }}` as boolean `true`.
    - *Input/Output:* Input from `plan the broadcast`; outputs True to `[cred] Twilio - ask the owner to approve`, False to `STOP: preview - draft shown`.

---

#### 1.6 Approval, Dispatch, and Audit Logging
- **Overview:** This block manages human-in-the-loop governance by sending an interactive approval request to the business owner via WhatsApp, holding execution until approval is received or times out, broadcasting approved messages to customers via Twilio, and updating Google Sheets inventory data upon successful execution.
- **Nodes Involved:** `[cred] Twilio - ask the owner to approve`, `wait for the owner's approval`, `approved?`, `not-approved`, `preview`, `per-customer`, `[cred] Twilio - send the announcement`, `sent`, `writing on?`, `rows for the stock sheet`, `[cred] Sheets - write today's prices`, `written`, `not-written`
- **Node Details:**
  - `[cred] Twilio - ask the owner to approve`
    - *Type and Technical Role:* Twilio Node (Send WhatsApp message).
    - *Configuration choices:* Sends WhatsApp message to `owner_to` from `twilio_whatsapp_from` with message body appending `$execution.resumeUrl + '?approve=yes'`. Parameter `toWhatsapp: true`.
    - *Input/Output:* Input from `sending on?` (True); outputs message dispatch confirmation to `wait for the owner's approval`.
    - *Edge cases:* Twilio API rate-limiting or invalid owner phone number formatting.
  - `wait for the owner's approval`
    - *Type and Technical Role:* Wait Node.
    - *Configuration choices:* Configured with webhook resume type (`webhook-id: gold-catalogue-approve`). Limit wait time enabled after time interval using `approval_wait_hours` from configuration.
    - *Input/Output:* Input from Twilio approval request node; outputs web resumption payload to `approved?`.
  - `approved?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ !!($json.query && $json.query.approve === 'yes') }}` as boolean `true`.
    - *Input/Output:* Input from `wait for the owner's approval`; outputs True branch to `per-customer` and `writing on?`, False branch to `not-approved`.
  - `not-approved`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Stops execution due to approval timeout (`_outcome: 'not sent'`).
    - *Input/Output:* Input from `approved?` (False); terminates branch.
  - `preview`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Outputs comprehensive preview details and un-sent draft messages when sending is disabled.
    - *Input/Output:* Input from `sending on?` (False); terminates branch.
  - `per-customer`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Maps approved broadcast array into individual items per recipient.
    - *Input/Output:* Input from `approved?` (True); outputs individual recipient items to `[cred] Twilio - send the announcement`.
  - `[cred] Twilio - send the announcement`
    - *Type and Technical Role:* Twilio Node (Send WhatsApp message).
    - *Configuration choices:* Sends individual WhatsApp messages to `$json.to` from `twilio_whatsapp_from` with message body `$json.body`. Parameter `toWhatsapp: true`.
    - *Input/Output:* Input from `per-customer`; outputs dispatch confirmation to `sent`.
    - *Edge cases:* Twilio errors, WhatsApp template rejections, or disconnected client numbers.
  - `sent`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Executes once (`executeOnce: true`), generating final success audit log (`_outcome: 'sent'`).
    - *Input/Output:* Input from Twilio send node; completes workflow branch.
  - `writing on?`
    - *Type and Technical Role:* If Node.
    - *Configuration choices:* Evaluates `={{ $('config').first().json.write_enabled }}` as boolean `true`.
    - *Input/Output:* Input from `approved?` (True); outputs True to `rows for the stock sheet`, False to `not-written`.
  - `rows for the stock sheet`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Maps repriced chains into sheet update objects (`sku`, `price`, `rate`, `priced_qty`, `priced_at`). Configured with `executeOnce: true`.
    - *Input/Output:* Input from `writing on?` (True); outputs update payload to `[cred] Sheets - write today's prices`.
  - `[cred] Sheets - write today's prices`
    - *Type and Technical Role:* Google Sheets Node (Update operation).
    - *Configuration choices:* Operation: `update`. Matching columns: `sku`. Options: `cellFormat: RAW`. Dynamic document ID and sheet name.
    - *Input/Output:* Input from `rows for the stock sheet`; outputs update confirmation to `written`.
    - *Edge cases:* Google Sheets write locks, API rate limits, or mismatched SKU rows.
  - `written`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Executes once (`executeOnce: true`), outputting completion audit log (`_outcome: 'written'`).
    - *Input/Output:* Input from Sheets update node; completes workflow branch.
  - `not-written`
    - *Type and Technical Role:* Code Node (JavaScript).
    - *Configuration choices:* Executes once (`executeOnce: true`), logging preview state when live sheet writing is disabled.
    - *Input/Output:* Input from `writing on?` (False); completes workflow branch.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run it now` | manualTrigger | On-demand manual execution trigger. | None | `config` | Reprice gold chain stock daily and announce it on WhatsApp |
| `Every weekday, 08:00` | scheduleTrigger | Automated weekday schedule trigger at 08:00. | None | `config` | Reprice gold chain stock daily and announce it on WhatsApp |
| `config` | code | Central configuration and parameters loader. | `Run it now`, `Every weekday, 08:00` | `stock sheet configured?` | Reprice gold chain stock daily and announce it on WhatsApp |
| `stock sheet configured?` | if | Evaluates whether Google Sheets stock tab is enabled. | `config` | `[cred] Sheets - read the stock`, `use the demo stock` | 1. Read the stock, and check it before pricing |
| `[cred] Sheets - read the stock` | googleSheets | Reads stock inventory rows from Google Sheets. | `stock sheet configured?` | `collect the stock` | 1. Read the stock, and check it before pricing |
| `stock-collect` | code | Aggregates sheet rows into a unified stock object. | `[cred] Sheets - read the stock` | `check the stock rows` | 1. Read the stock, and check it before pricing |
| `stock-demo` | code | Provides built-in demo stock inventory data. | `stock sheet configured?` | `check the stock rows` | 1. Read the stock, and check it before pricing |
| `check the stock rows` | code | Validates inventory rows and holds back invalid items. | `stock-collect`, `stock-demo` | `gold rate configured?` | 1. Read the stock, and check it before pricing |
| `gold rate configured?` | if | Evaluates whether goldapi.io API key is configured. | `check the stock rows` | `Gold rate - today's price`, `use the demo gold rate` | 2. Price every chain at today's rate |
| `Gold rate - today's price` | httpRequest | Fetches live 24k gold spot rate from goldapi.io. | `gold rate configured?` | `read the gold rate`, `rate-fail` | 2. Price every chain at today's rate |
| `rate-read` | code | Normalizes and parses gold rate API response. | `Gold rate - today's price` | `price every chain` | 2. Price every chain at today's rate |
| `rate-fail` | code | Halts execution when gold rate API fails. | `Gold rate - today's price` | None | 2. Price every chain at today's rate |
| `rate-demo` | code | Provides fallback demo gold spot rate. | `gold rate configured?` | `price every chain` | 2. Price every chain at today's rate |
| `price every chain` | code | Calculates item prices and validates rate movement. | `rate-read`, `rate-demo`, `check the stock rows` | `rate looks right?` | 2. Price every chain at today's rate |
| `rate-ok` | if | Evaluates rate movement against maximum percentage limits. | `price every chain` | `build the catalogue`, `rate-wrong` | 2. Price every chain at today's rate |
| `rate-wrong` | code | Stops execution when rate movement exceeds safety limits. | `rate-ok` | None | 2. Price every chain at today's rate |
| `catalogue` | code | Generates catalogs, stock lists, CSVs, and delta announcements. | `rate-ok` | `customer list configured?` | 3. Four views of the same priced rows |
| `cust-on` | if | Evaluates whether customer Google Sheet is enabled. | `catalogue` | `[cred] Sheets - read the customers`, `use the demo customers` | 3. Four views of the same priced rows |
| `cust-read` | googleSheets | Reads customer contact rows from Google Sheets. | `cust-on` | `collect the customers` | 3. Four views of the same priced rows |
| `cust-collect` | code | Aggregates customer contact rows. | `cust-read` | `plan the broadcast` | 3. Four views of the same priced rows |
| `cust-demo` | code | Provides built-in demo customer contact data. | `cust-on` | `plan the broadcast` | 3. Four views of the same priced rows |
| `plan the broadcast` | code | Validates phone numbers, filters opt-outs, and plans send. | `cust-collect`, `cust-demo`, `catalogue` | `sending on?` | 3. Four views of the same priced rows |
| `send-on` | if | Evaluates whether Twilio WhatsApp sending is enabled. | `plan the broadcast` | `[cred] Twilio - ask the owner to approve`, `preview` | 4. The owner approves, then it sends and writes back |
| `preview` | code | Displays preview messages and drafts when sending is disabled. | `send-on` | None | Nothing is sent until you say so |
| `ask` | twilio | Sends approval request message with resume link to owner. | `send-on` | `wait for the owner's approval` | 4. The owner approves, then it sends and writes back |
| `wait` | wait | Pauses execution awaiting owner approval via webhook. | `ask` | `approved?` | 4. The owner approves, then it sends and writes back |
| `approved` | if | Evaluates whether the owner approved the broadcast. | `wait` | `one message per customer`, `writing on?`, `not-approved` | 4. The owner approves, then it sends and writes back |
| `not-approved` | code | Handles approval timeout or rejection. | `approved` | None | 4. The owner approves, then it sends and writes back |
| `per-customer` | code | Splits broadcast list into individual recipient items. | `approved` | `[cred] Twilio - send the announcement` | 4. The owner approves, then it sends and writes back |
| `send` | twilio | Sends individual customer announcement via WhatsApp. | `per-customer` | `sent` | 4. The owner approves, then it sends and writes back |
| `sent` | code | Outputs final successful broadcast audit log. | `send` | None | 4. The owner approves, then it sends and writes back |
| `write-on` | if | Evaluates whether live Google Sheets writing is enabled. | `approved` | `rows for the stock sheet`, `not-written` | 4. The owner approves, then it sends and writes back |
| `rows` | code | Formats repriced stock rows for sheet update. | `write-on` | `[cred] Sheets - write today's prices` | 4. The owner approves, then it sends and writes back |
| `write` | googleSheets | Updates Google Sheets stock tab with today's prices. | `rows` | `written` | 4. The owner approves, then it sends and writes back |
| `written` | code | Outputs final sheet write audit log. | `write` | None | 4. The owner approves, then it sends and writes back |
| `not-written` | code | Outputs preview log when sheet writing is disabled. | `write-on` | None | Nothing is sent until you say so |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these exact steps:

1. **Create Triggers & Configuration:**
   - Add a **Schedule Trigger** node (`Every weekday, 08:00`), setting cron expression `0 8 * * 1-5`.
   - Add a **Manual Trigger** node (`Run it now`).
   - Add a **Code** node (`config`), connecting both triggers as inputs. Paste the configuration JavaScript code containing constants (`BUSINESS_NAME`, `PURITY`, `STYLES`, `WHOLESALE_MARGIN_PCT`, `ROUND_TO`, `DEMO_STOCK`, `DEMO_CUSTOMERS`, etc.) and return the mapped `SETTINGS` object.

2. **Setup Stock Ingestion Branch:**
   - Add an **If** node (`stock sheet configured?`), connecting input from `config`. Condition: `{{ $('config').first().json.sheet_enabled }}` equals `true`.
   - **True Branch:** Add a **Google Sheets** node (`[cred] Sheets - read the stock`). Set operation to `Read`, document ID to `={{ $('config').first().json.sheet_id }}`, sheet name to `={{ $('config').first().json.stock_tab }}`, and enable `alwaysOutputData`. Connect output to a **Code** node (`collect the stock`) to filter and format rows.
   - **False Branch:** Add a **Code** node (`use the demo stock`) returning `cfg.demo_stock`.
   - Connect both collection branches to a **Code** node (`check the stock rows`) to perform validation and hold-back checks.

3. **Setup Gold Rate & Pricing Engine:**
   - Add an **If** node (`gold rate configured?`), connecting input from `check the stock rows`. Condition: `{{ $('config').first().json.gold_api_enabled }}` equals `true`.
   - **True Branch:** Add an **HTTP Request** node (`Gold rate - today's price`). Method: `GET`, URL: `={{ $('config').first().json.gold_api_base + '/XAU/' + $('config').first().json.currency }}`, Header: `x-access-token` equal to `={{ $('config').first().json.gold_api_key }}` and `Accept: application/json`. Set `onError: continueErrorOutput`. Connect success output to a **Code** node (`read the gold rate`) and error output to a **Code** node (`rate-fail`).
   - **False Branch:** Add a **Code** node (`use the demo gold rate`) returning `cfg.demo_gold_rate`.
   - Connect rate nodes to a **Code** node (`price every chain`) to compute item prices and validate rate movement.
   - Add an **If** node (`rate looks right?`). Condition: `{{ $json.rate_ok }}` equals `true`. Connect True to catalog generation, False to a **Code** node (`rate-wrong`).

4. **Setup Catalogue and Announcement Builder:**
   - Add a **Code** node (`build the catalogue`) taking input from `rate looks right?`. Generates catalogue text, CSV exports, stock lists, and delta announcements.

5. **Setup Customer Ingestion & Broadcast Planning:**
   - Add an **If** node (`customer list configured?`), connecting input from `build the catalogue`. Condition: `{{ $('config').first().json.sheet_enabled }}` equals `true`.
   - **True Branch:** Add a **Google Sheets** node (`[cred] Sheets - read the customers`) reading from `customers_tab`. Connect to a **Code** node (`collect the customers`).
   - **False Branch:** Add a **Code** node (`use the demo customers`) returning `cfg.demo_customers`.
   - Connect customer inputs to a **Code** node (`plan the broadcast`) to filter opt-outs, validate E.164 numbers, enforce caps, and build owner messages.

6. **Setup Approval, Dispatch, and Writing Workflow:**
   - Add an **If** node (`sending on?`). Condition: `{{ $('config').first().json.send_enabled }}` equals `true`.
   - **True Branch:** Add a **Twilio** node (`[cred] Twilio - ask the owner to approve`). Set resource to `WhatsApp`, operation to `Send`, recipient to `={{ $json.owner_to }}`, sender to `={{ $('config').first().json.twilio_whatsapp_from }}`, and message body appending `?approve=yes`. Connect to a **Wait** node (`wait for the owner's approval`) resuming on webhook (`gold-catalogue-approve`) with limit time based on `approval_wait_hours`.
   - **False Branch:** Add a **Code** node (`STOP: preview - draft shown`).
   - Following the Wait node, add an **If** node (`approved?`). Condition: `={{ !!($json.query && $json.query.approve === 'yes') }}` equals `true`.
   - **True Branch (Messaging):** Connect to a **Code** node (`per-customer`), then a **Twilio** node (`[cred] Twilio - send the announcement`) sending WhatsApp messages per recipient, ending at a **Code** node (`STOP: announcement sent`).
   - **True Branch (Google Sheets Writing):** Connect to an **If** node (`writing on?`). Condition: `{{ $('config').first().json.write_enabled }}` equals `true`. If true, connect to a **Code** node (`rows for the stock sheet`), then a **Google Sheets** node (`[cred] Sheets - write today's prices`) updating rows matched on `sku`, ending at a **Code** node (`STOP: prices written`). If false, connect to `STOP: preview - prices not written`.
   - **False Branch:** Connect to a **Code** node (`STOP: not approved`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for jewellery wholesalers and gold traders automating daily WhatsApp price updates with human-in-the-loop approval. | Workflow Overview & Target Use Case |
| Companion workflow available for handling incoming customer WhatsApp enquiries and automated pricing responses. | Enquiries Workflow Integration (Workflow 02) |