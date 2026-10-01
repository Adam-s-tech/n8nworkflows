Answer WhatsApp gold chain enquiries with Google Sheets, Twilio, and OpenAI

https://n8nworkflows.xyz/workflows/answer-whatsapp-gold-chain-enquiries-with-google-sheets--twilio--and-openai-20021


# Answer WhatsApp gold chain enquiries with Google Sheets, Twilio, and OpenAI

### 1. Workflow Overview

This workflow automates the handling of incoming WhatsApp enquiries for a gold chain wholesale business. It receives messages via Twilio webhook (or manual demo triggers), loads stock data from Google Sheets (or built-in fallback data), extracts customer intent using keyword logic and optionally an OpenAI-compatible model, generates verified responses based on approved price lists, handles human handoffs/complaints, and records opt-outs (e.g., “STOP”) back to the sheet.

The workflow logic is grouped into five functional blocks:
- **1.1 Input Reception & Configuration:** Receives incoming messages or demo triggers, normalizes global settings, and validates whether a live Google Sheets stock list is configured.
- **1.2 Stock Loading & Initial Parsing:** Reads stock entries from Google Sheets or uses fallback demo inventory, then applies initial keyword extraction to determine requested chain styles, purities, lengths, quantities, and opt-out/handoff intents.
- **1.3 Optional AI Intent Extraction:** Conditionally invokes an OpenAI-compatible model to refine message parsing without exposing pricing/stock logic to the AI.
- **1.4 Decision Engine & Messaging:** Evaluates the parsed data against inventory, determines whether to reply, escalate to an owner, or confirm an opt-out, and routes messages for live dispatch or preview.
- **1.5 Opt-Out Recording & Finalization:** Updates customer records in Google Sheets when an opt-out is detected and terminates execution with outcome logs.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
**Overview:**  
Captures incoming webhook payloads from Twilio or manual test triggers, centralizes all business rules and configurations into a single JavaScript object, and evaluates whether a live Google Sheets connection is enabled.

**Nodes Involved:**
- `A WhatsApp message arrives`
- `Run the demo messages`
- `config`
- `stock sheet configured?`

**Node Details:**
- **A WhatsApp message arrives**
  - *Type & Role:* Webhook trigger (`n8n-nodes-base.webhook`). Listens for incoming HTTP POST requests from Twilio.
  - *Configuration:* Path: `gold-whatsapp`, HTTP Method: `POST`, Response Mode: `onReceived`.
  - *Connections:* Input: None (Trigger). Output: `config`.
  - *Failure Modes:* Webhook URL misconfigured in Twilio; HTTP timeout if downstream execution hangs.

- **Run the demo messages**
  - *Type & Role:* Manual Trigger (`n8n-nodes-base.manualTrigger`). Allows manual execution of the workflow using predefined test messages.
  - *Configuration:* Default execution parameters.
  - *Connections:* Input: None. Output: `config`.

- **config**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Central configuration store defining business constants, pricing rules, styles, gold API keys, sheet IDs, Twilio parameters, and fallback demo data.
  - *Configuration:* Executes custom JavaScript to inject a unified `SETTINGS` object into every incoming payload.
  - *Connections:* Input: `A WhatsApp message arrives` or `Run the demo messages`. Output: `stock sheet configured?`.
  - *Failure Modes:* Syntax errors in custom JavaScript object definitions.

- **stock sheet configured?**
  - *Type & Role:* If node (`n8n-nodes-base.if`). Checks whether `sheet_enabled` is true based on the configuration node.
  - *Configuration:* Evaluates `{{ $('config').first().json.sheet_enabled }}` equals `true`.
  - *Connections:* Input: `config`. Output True branch: `[cred] Sheets - read the stock`. Output False branch: `use the demo stock`.

---

#### Block 1.2: Stock Loading & Initial Parsing
**Overview:**  
Fetches the current inventory list from Google Sheets or fallback demo data, then parses incoming customer messages using keyword matching to identify style, karat, length, quantity, and specific intents.

**Nodes Involved:**
- `[cred] Sheets - read the stock`
- `collect the stock`
- `use the demo stock`
- `read the message`

**Node Details:**
- **[cred] Sheets - read the stock**
  - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Reads inventory rows from the designated stock tab.
  - *Configuration:* Uses Google Sheets credentials. Document ID and Sheet Name dynamically mapped from `config`.
  - *Connections:* Input: `stock sheet configured?` (True). Output: `collect the stock`.
  - *Failure Modes:* Invalid Google Sheets credentials, missing sheet ID, or incorrect tab names resulting in API errors.

- **collect the stock**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Transforms sheet rows into a standardized array format.
  - *Configuration:* Filters out empty SKUs and wraps rows into `{ source: 'stock sheet', rows }`.
  - *Connections:* Input: `[cred] Sheets - read the stock`. Output: `read the message`.

- **use the demo stock**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Provides built-in mock inventory when no Google Sheet ID is supplied.
  - *Configuration:* Returns invented stock items defined in `config`.
  - *Connections:* Input: `stock sheet configured?` (False). Output: `read the message`.

- **read the message**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Normalizes incoming WhatsApp message payloads and executes keyword parsing.
  - *Configuration:* Parses text for styles, karat (e.g., "22k"), length (e.g., "20 inch"), quantity, opt-out keywords, and handoff triggers.
  - *Connections:* Input: `collect the stock` or `use the demo stock`. Output: `AI configured?`.
  - *Failure Modes:* Malformed text payloads causing regex parsing failures (handled gracefully via fallbacks).

---

#### Block 1.3: Optional AI Intent Extraction
**Overview:**  
Checks if an OpenAI API key is configured; if enabled, sends message text to an AI endpoint to parse intent into structured parameters while strictly restricting the model from generating prices or replies.

**Nodes Involved:**
- `AI configured?`
- `build the AI request`
- `AI - read what they want`
- `apply the AI reading`
- `keyword reading only`

**Node Details:**
- **AI configured?**
  - *Type & Role:* If node (`n8n-nodes-base.if`). Evaluates whether `ai_enabled` is true in the configuration.
  - *Configuration:* Evaluates `{{ $('config').first().json.ai_enabled }}` equals `true`.
  - *Connections:* Input: `read the message`. Output True branch: `build the AI request`. Output False branch: `keyword reading only`.

- **build the AI request**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Prepares the system and user prompts for the AI chat completion API.
  - *Configuration:* Enforces strict JSON output containing style, purity, length, quantity, and human handoff flags.
  - *Connections:* Input: `AI configured?` (True). Output: `AI - read what they want`.

- **AI - read what they want**
  - *Type & Role:* HTTP Request node (`n8n-nodes-base.httpRequest`). Calls an OpenAI-compatible chat completion endpoint.
  - *Configuration:* POST request to `{{ $('config').first().json.ai_base_url + '/chat/completions' }}` with Bearer token authentication and a 30-second timeout. Error handling set to `continueRegularOutput`.
  - *Connections:* Input: `build the AI request`. Output: `apply the AI reading`.
  - *Failure Modes:* API rate limits, authentication failures, invalid API keys, or timeout errors.

- **apply the AI reading**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Merges AI-extracted parameters with baseline keyword data.
  - *Configuration:* Validates JSON responses; falls back to keyword extraction if parsing fails or returned styles are invalid.
  - *Connections:* Input: `AI - read what they want`. Output: `decide the reply`.

- **keyword reading only**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Passes through keyword-based extraction results when AI is disabled or fails.
  - *Configuration:* Sets `read_by` attribute to indicate keyword fallback.
  - *Connections:* Input: `AI configured?` (False). Output: `decide the reply`.

---

#### Block 1.4: Decision Engine & Messaging
**Overview:**  
Executes core business logic to determine customer replies or owner handoff alerts using verified stock sheet pricing, then routes messages based on environment settings (live, test, or preview).

**Nodes Involved:**
- `decide the reply`
- `address the messages`
- `sending on?`
- `[cred] Twilio - send on WhatsApp`
- `replied`
- `reply-preview`

**Node Details:**
- **decide the reply**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Implements pricing calculations, stock lookups, opt-out handling, and handoff logic.
  - *Configuration:* Evaluates opt-out status, quantity thresholds (`HANDOFF_QTY`), and exact/alternative stock matches. Generates customer responses and owner alert payloads.
  - *Connections:* Input: `apply the AI reading` or `keyword reading only`. Output: `address the messages` (and `rows for the customers tab`).
  - *Failure Modes:* Missing stock attributes resulting in NaN pricing calculations (guarded by row validation functions).

- **address the messages**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Determines message routing based on execution mode (`live`, `test`, or `preview`).
  - *Configuration:* In test mode, redirects messages to `TEST_WHATSAPP` and caps broadcasts. Strips `whatsapp:` prefixes.
  - *Connections:* Input: `decide the reply`. Output: `sending on?`.

- **sending on?**
  - *Type & Role:* If node (`n8n-nodes-base.if`). Checks whether message sending is enabled.
  - *Configuration:* Evaluates `{{ $('config').first().json.send_enabled }}` equals `true`.
  - *Connections:* Input: `address the messages`. Output True branch: `[cred] Twilio - send on WhatsApp`. Output False branch: `reply-preview`.

- **[cred] Twilio - send on WhatsApp**
  - *Type & Role:* Twilio node (`n8n-nodes-base.twilio`). Dispatches WhatsApp messages to customers or owners.
  - *Configuration:* Uses Twilio credentials. Maps `to`, `from` (`TWILIO_WHATSAPP_FROM`), message body, and enables WhatsApp messaging options.
  - *Connections:* Input: `sending on?` (True). Output: `replied`.
  - *Failure Modes:* Invalid Twilio credentials, unverified WhatsApp sender numbers, or API rejection.

- **replied**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Returns final execution confirmation for sent messages.
  - *Configuration:* Executes once to output success state and message counts.
  - *Connections:* Input: `[cred] Twilio - send on WhatsApp`. Output: None (Terminal node).

- **reply-preview**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Displays simulated responses when sending is disabled.
  - *Configuration:* Outputs preview metadata including incoming text, extraction method, and drafted reply body.
  - *Connections:* Input: `sending on?` (False). Output: None (Terminal node).

---

#### Block 1.5: Opt-Out Recording & Finalization
**Overview:**  
Filters opt-out events, updates customer records in Google Sheets during live runs, or outputs preview logs when writing is disabled.

**Nodes Involved:**
- `rows for the customers tab`
- `writing on?`
- `[cred] Sheets - record the opt-outs`
- `optout-done`
- `optout-preview`

**Node Details:**
- **rows for the customers tab**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Filters out non-opt-out items to isolate records requiring database updates.
  - *Configuration:* Extracts `opt_out_record` objects from the decision engine output.
  - *Connections:* Input: `decide the reply`. Output: `writing on?`.

- **writing on?**
  - *Type & Role:* If node (`n8n-nodes-base.if`). Checks whether database write operations are permitted.
  - *Configuration:* Evaluates `{{ $('config').first().json.write_enabled }}` equals `true`.
  - *Connections:* Input: `rows for the customers tab`. Output True branch: `[cred] Sheets - record the opt-outs`. Output False branch: `optout-preview`.

- **[cred] Sheets - record the opt-outs**
  - *Type & Role:* Google Sheets node (`n8n-nodes-base.googleSheets`). Updates the Customers tab to mark numbers as opted out.
  - *Configuration:* Operation: `update`, Sheet Name mapped from configuration, matching column set to `whatsapp`, cell format `RAW`.
  - *Connections:* Input: `writing on?` (True). Output: `optout-done`.
  - *Failure Modes:* Google Sheets API errors, permission issues, or missing customer columns.

- **optout-done**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Returns confirmation that opt-outs were successfully recorded.
  - *Configuration:* Executes once to return outcome status.
  - *Connections:* Input: `[cred] Sheets - record the opt-outs`. Output: None (Terminal node).

- **optout-preview**
  - *Type & Role:* Code node (`n8n-nodes-base.code`). Generates preview logs for unrecorded opt-outs during test or preview runs.
  - *Configuration:* Outputs intended write actions without modifying external data.
  - *Connections:* Input: `writing on?` (False). Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A WhatsApp message arrives | webhook | Twilio webhook receiver | None | config | ## Answer WhatsApp gold chain enquiries from a Google Sheets price list<br><br>**Who's it for**<br>Jewellery wholesalers who get "22k rope 20 inch, how much?" on WhatsApp all day and want every answer to quote the same approved price list.<br><br>**How it works**<br>Twilio posts each incoming WhatsApp message to the webhook. The workflow reads the Stock tab, then works out what the customer wants: style, karat, length and quantity. Keyword rules always run, and an optional AI model can fill gaps, but only into those fields; it never sees a price and never writes the reply. The reply is decided in order: an opt-out is confirmed and recorded, an order, bulk quantity, custom piece or complaint goes to the owner, a stock question is answered from the sheet at the last approved price, and a vague message is asked for style and karat.<br><br>**How to set up**<br>1. Run **Run the demo messages** first: five invented messages, nothing sent.<br>2. In `config`, set `SHEET_ID`, `TWILIO_WHATSAPP_FROM` and `OWNER_WHATSAPP`.<br>3. Connect Google Sheets and Twilio on the `[cred]` nodes and point the sender's incoming webhook here.<br>4. Set `TEST_WHATSAPP`, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs and a Twilio WhatsApp sender. An AI key is optional.<br><br>**How to customize**<br>`HANDOFF_QTY` sets when a quantity goes to a person, the keyword lists in `read the message` decide what is recognised, and `STYLES` limits which styles are ever offered. Pair it with the companion repricing workflow, which writes the prices this one quotes. |
| Run the demo messages | manualTrigger | Manual execution trigger | None | config | ## Answer WhatsApp gold chain enquiries from a Google Sheets price list<br><br>**Who's it for**<br>Jewellery wholesalers who get "22k rope 20 inch, how much?" on WhatsApp all day and want every answer to quote the same approved price list.<br><br>**How it works**<br>Twilio posts each incoming WhatsApp message to the webhook. The workflow reads the Stock tab, then works out what the customer wants: style, karat, length and quantity. Keyword rules always run, and an optional AI model can fill gaps, but only into those fields; it never sees a price and never writes the reply. The reply is decided in order: an opt-out is confirmed and recorded, an order, bulk quantity, custom piece or complaint goes to the owner, a stock question is answered from the sheet at the last approved price, and a vague message is asked for style and karat.<br><br>**How to set up**<br>1. Run **Run the demo messages** first: five invented messages, nothing sent.<br>2. In `config`, set `SHEET_ID`, `TWILIO_WHATSAPP_FROM` and `OWNER_WHATSAPP`.<br>3. Connect Google Sheets and Twilio on the `[cred]` nodes and point the sender's incoming webhook here.<br>4. Set `TEST_WHATSAPP`, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs and a Twilio WhatsApp sender. An AI key is optional.<br><br>**How to customize**<br>`HANDOFF_QTY` sets when a quantity goes to a person, the keyword lists in `read the message` decide what is recognised, and `STYLES` limits which styles are ever offered. Pair it with the companion repricing workflow, which writes the prices this one quotes. |
| config | code | Central configuration store | A WhatsApp message arrives, Run the demo messages | stock sheet configured? | ## Answer WhatsApp gold chain enquiries from a Google Sheets price list<br><br>**Who's it for**<br>Jewellery wholesalers who get "22k rope 20 inch, how much?" on WhatsApp all day and want every answer to quote the same approved price list.<br><br>**How it works**<br>Twilio posts each incoming WhatsApp message to the webhook. The workflow reads the Stock tab, then works out what the customer wants: style, karat, length and quantity. Keyword rules always run, and an optional AI model can fill gaps, but only into those fields; it never sees a price and never writes the reply. The reply is decided in order: an opt-out is confirmed and recorded, an order, bulk quantity, custom piece or complaint goes to the owner, a stock question is answered from the sheet at the last approved price, and a vague message is asked for style and karat.<br><br>**How to set up**<br>1. Run **Run the demo messages** first: five invented messages, nothing sent.<br>2. In `config`, set `SHEET_ID`, `TWILIO_WHATSAPP_FROM` and `OWNER_WHATSAPP`.<br>3. Connect Google Sheets and Twilio on the `[cred]` nodes and point the sender's incoming webhook here.<br>4. Set `TEST_WHATSAPP`, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs and a Twilio WhatsApp sender. An AI key is optional.<br><br>**How to customize**<br>`HANDOFF_QTY` sets when a quantity goes to a person, the keyword lists in `read the message` decide what is recognised, and `STYLES` limits which styles are ever offered. Pair it with the companion repricing workflow, which writes the prices this one quotes. |
| stock sheet configured? | if | Evaluates sheet configuration status | config | [cred] Sheets - read the stock, use the demo stock | ### 1. Take the message in, with the stock beside it<br><br>`A WhatsApp message arrives` is the Twilio webhook; `Run the demo messages` feeds five invented messages instead. The stock comes from `[cred] Sheets - read the stock`, or `use the demo stock` when no sheet is set.<br><br>`read the message` applies the keyword rules: style, karat, length, quantity, and whether a person is needed. |
| [cred] Sheets - read the stock | googleSheets | Reads stock inventory from Google Sheets | stock sheet configured? | collect the stock | ### 1. Take the message in, with the stock beside it<br><br>`A WhatsApp message arrives` is the Twilio webhook; `Run the demo messages` feeds five invented messages instead. The stock comes from `[cred] Sheets - read the stock`, or `use the demo stock` when no sheet is set.<br><br>`read the message` applies the keyword rules: style, karat, length, quantity, and whether a person is needed. |
| collect the stock | code | Formats sheet stock rows | [cred] Sheets - read the stock | read the message | ### 1. Take the message in, with the stock beside it<br><br>`A WhatsApp message arrives` is the Twilio webhook; `Run the demo messages` feeds five invented messages instead. The stock comes from `[cred] Sheets - read the stock`, or `use the demo stock` when no sheet is set.<br><br>`read the message` applies the keyword rules: style, karat, length, quantity, and whether a person is needed. |
| use the demo stock | code | Fallback inventory data provider | stock sheet configured? | read the message | ### 1. Take the message in, with the stock beside it<br><br>`A WhatsApp message arrives` is the Twilio webhook; `Run the demo messages` feeds five invented messages instead. The stock comes from `[cred] Sheets - read the stock`, or `use the demo stock` when no sheet is set.<br><br>`read the message` applies the keyword rules: style, karat, length, quantity, and whether a person is needed. |
| read the message | code | Normalizes and parses incoming messages | collect the stock, use the demo stock | AI configured? | ### 1. Take the message in, with the stock beside it<br><br>`A WhatsApp message arrives` is the Twilio webhook; `Run the demo messages` feeds five invented messages instead. The stock comes from `[cred] Sheets - read the stock`, or `use the demo stock` when no sheet is set.<br><br>`read the message` applies the keyword rules: style, karat, length, quantity, and whether a person is needed. |
| AI configured? | if | Checks if AI API key is provided | read the message | build the AI request, keyword reading only | ### 2. Let a model help read, never answer<br><br>With `AI_API_KEY` set, `AI - read what they want` reads the message into the same fields. It is never shown a price and never writes the reply.<br><br>`apply the AI reading` ignores a style that is not in `STYLES`, and an answer that does not parse keeps the keyword read. Without a key, `keyword reading only` carries on alone. |
| build the AI request | code | Formats AI chat prompt | AI configured? | AI - read what they want | ### 2. Let a model help read, never answer<br><br>With `AI_API_KEY` set, `AI - read what they want` reads the message into the same fields. It is never shown a price and never writes the reply.<br><br>`apply the AI reading` ignores a style that is not in `STYLES`, and an answer that does not parse keeps the keyword read. Without a key, `keyword reading only` carries on alone. |
| AI - read what they want | httpRequest | Queries OpenAI-compatible AI model | build the AI request | apply the AI reading | ### 2. Let a model help read, never answer<br><br>With `AI_API_KEY` set, `AI - read what they want` reads the message into the same fields. It is never shown a price and never writes the reply.<br><br>`apply the AI reading` ignores a style that is not in `STYLES`, and an answer that does not parse keeps the keyword read. Without a key, `keyword reading only` carries on alone. |
| apply the AI reading | code | Merges AI response with baseline parsing | AI - read what they want | decide the reply | ### 2. Let a model help read, never answer<br><br>With `AI_API_KEY` set, `AI - read what they want` reads the message into the same fields. It is never shown a price and never writes the reply.<br><br>`apply the AI reading` ignores a style that is not in `STYLES`, and an answer that does not parse keeps the keyword read. Without a key, `keyword reading only` carries on alone. |
| keyword reading only | code | Fallback keyword-only parser | AI configured? | decide the reply | ### 2. Let a model help read, never answer<br><br>With `AI_API_KEY` set, `AI - read what they want` reads the message into the same fields. It is never shown a price and never writes the reply.<br><br>`apply the AI reading` ignores a style that is not in `STYLES`, and an answer that does not parse keeps the keyword read. Without a key, `keyword reading only` carries on alone. |
| decide the reply | code | Evaluates stock and builds responses/alerts | apply the AI reading, keyword reading only | address the messages, rows for the customers tab | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| address the messages | code | Routes messages based on execution mode | decide the reply | sending on? | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| sending on? | if | Determines if message dispatch is enabled | address the messages | [cred] Twilio - send on WhatsApp, reply-preview | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| [cred] Twilio - send on WhatsApp | twilio | Sends WhatsApp messages via Twilio | sending on? | replied | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| replied | code | Terminates successful send execution | [cred] Twilio - send on WhatsApp | None | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| reply-preview | code | Outputs simulated reply logs | sending on? | None | ### 3. Answer from the sheet, or hand it to a person<br><br>`decide the reply` quotes every price and stock count from the Stock tab at the price last approved, with the list date, so the bot cannot quote a number the owner never saw. Orders, bulk quantities, custom pieces, payment questions and complaints are acknowledged and sent to the owner with who, what and why. The bot does not negotiate.<br><br>`sending on?` sends through `[cred] Twilio - send on WhatsApp`, or shows every reply on `STOP: preview - replies shown`. |
| rows for the customers tab | code | Isolates opt-out records | decide the reply | writing on? | ### 4. Record opt-outs<br><br>A "STOP" is confirmed to the customer, and `[cred] Sheets - record the opt-outs` marks them in the Customers tab, so the daily announcement never messages that number again. In preview, `STOP: preview - opt-outs not recorded` shows the rows it would write. |
| writing on? | if | Determines if sheet writing is enabled | rows for the customers tab | [cred] Sheets - record the opt-outs, optout-preview | ### 4. Record opt-outs<br><br>A "STOP" is confirmed to the customer, and `[cred] Sheets - record the opt-outs` marks them in the Customers tab, so the daily announcement never messages that number again. In preview, `STOP: preview - opt-outs not recorded` shows the rows it would write. |
| [cred] Sheets - record the opt-outs | googleSheets | Updates opt-out status in Google Sheets | writing on? | optout-done | ### 4. Record opt-outs<br><br>A "STOP" is confirmed to the customer, and `[cred] Sheets - record the opt-outs` marks them in the Customers tab, so the daily announcement never messages that number again. In preview, `STOP: preview - opt-outs not recorded` shows the rows it would write. |
| optout-done | code | Terminates successful opt-out recording | [cred] Sheets - record the opt-outs | None | ### 4. Record opt-outs<br><br>A "STOP" is confirmed to the customer, and `[cred] Sheets - record the opt-outs` marks them in the Customers tab, so the daily announcement never messages that number again. In preview, `STOP: preview - opt-outs not recorded` shows the rows it would write. |
| optout-preview | code | Outputs simulated opt-out write logs | writing on? | None | ### 4. Record opt-outs<br><br>A "STOP" is confirmed to the customer, and `[cred] Sheets - record the opt-outs` marks them in the Customers tab, so the daily announcement never messages that number again. In preview, `STOP: preview - opt-outs not recorded` shows the rows it would write. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Triggers:**
   - Create a **Webhook** node named `A WhatsApp message arrives` with path `gold-whatsapp` and method `POST`.
   - Create a **Manual Trigger** node named `Run the demo messages`.
2. **Add Configuration Node:**
   - Create a **Code** node named `config`. Paste the configuration JavaScript logic containing business rules, styles, purity mappings, and demo datasets.
   - Connect both `A WhatsApp message arrives` and `Run the demo messages` to `config`.
3. **Configure Stock Source Branching:**
   - Create an **If** node named `stock sheet configured?`. Set condition to check `{{ $('config').first().json.sheet_enabled === true }}`.
   - Connect `config` to `stock sheet configured?`.
   - Create a **Google Sheets** node named `[cred] Sheets - read the stock`. Set operation to read, document ID and sheet name from `config`. Connect `stock sheet configured?` (True branch) here.
   - Create a **Code** node named `collect the stock` to process sheet rows. Connect `[cred] Sheets - read the stock` here.
   - Create a **Code** node named `use the demo stock` for fallback inventory. Connect `stock sheet configured?` (False branch) here.
4. **Implement Message Reading:**
   - Create a **Code** node named `read the message`. Connect both `collect the stock` and `use the demo stock` to this node. This node normalizes payloads and applies keyword checks.
5. **Set Up Optional AI Processing:**
   - Create an **If** node named `AI configured?` evaluating `{{ $('config').first().json.ai_enabled === true }}`. Connect `read the message` here.
   - Create a **Code** node named `build the AI request` (connected to True branch).
   - Create an **HTTP Request** node named `AI - read what they want` pointing to `{{ $('config').first().json.ai_base_url + '/chat/completions' }}` with POST method, JSON body, Bearer auth header, and error handling set to `continueRegularOutput`. Connect `build the AI request` here.
   - Create a **Code** node named `apply the AI reading` to parse AI responses. Connect `AI - read what they want` here.
   - Create a **Code** node named `keyword reading only` for keyword fallback (connected to False branch of `AI configured?`).
6. **Implement Decision Engine & Routing:**
   - Create a **Code** node named `decide the reply`. Connect both `apply the AI reading` and `keyword reading only` to this node.
   - Create a **Code** node named `address the messages` to handle test/live routing. Connect `decide the reply` here.
   - Create an **If** node named `sending on?` evaluating `{{ $('config').first().json.send_enabled === true }}`. Connect `address the messages` here.
   - Create a **Twilio** node named `[cred] Twilio - send on WhatsApp` (connected to True branch) with credentials configured, mapping `to`, `from`, and message body.
   - Create a **Code** node named `replied` connected from `[cred] Twilio - send on WhatsApp`.
   - Create a **Code** node named `reply-preview` (connected to False branch of `sending on?`).
7. **Implement Opt-Out Tracking:**
   - Create a **Code** node named `rows for the customers tab` connected from `decide the reply`.
   - Create an **If** node named `writing on?` evaluating `{{ $('config').first().json.write_enabled === true }}`. Connect `rows for the customers tab` here.
   - Create a **Google Sheets** node named `[cred] Sheets - record the opt-outs` (connected to True branch) configured for update operations on the customers tab matching on `whatsapp`.
   - Create a **Code** node named `optout-done` connected from `[cred] Sheets - record the opt-outs`.
   - Create a **Code** node named `optout-preview` connected from the False branch of `writing on?`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Part 1 of 2 parts Gold Chain Catalogue Engine | Workflow architecture context |
| Companion Repricing Workflow | Pair with the repricing and announcement workflow which writes the prices this workflow quotes |