Audit websites from Telegram with GPT-4o-mini and log results to Google Sheets

https://n8nworkflows.xyz/workflows/audit-websites-from-telegram-with-gpt-4o-mini-and-log-results-to-google-sheets-19698


# Audit websites from Telegram with GPT-4o-mini and log results to Google Sheets

### 1. Workflow Overview

This workflow automates website audits via a Telegram bot. When a user sends a message containing a URL, the bot scrapes the target website, uses OpenAI (gpt-4o-mini) to generate a structured marketing and SEO audit, logs the record in a Google Sheet, and responds directly inside the Telegram chat. If a user sends a message without a URL, the bot provides a helpful usage guide instead.

The workflow logic is divided into three functional blocks:
- **1.1 Input Reception & Validation:** Captures incoming messages from Telegram, sets global configuration parameters, extracts URLs using regular expressions, and branches execution depending on whether a valid URL is present.
- **1.2 Website Scraping & AI Processing:** Fetches the website HTML over HTTP, strips out unnecessary markup (navigation, footers, scripts, styles) to minimize token usage, and queries the OpenAI model using a specialized system prompt coupled with a structured output parser.
- **1.3 Data Logging & Response Delivery:** Formats the AI-generated audit results into a readable text message, logs the data to a Google Sheets document, and sends both the confirmation reply to the user and fallback help messages when applicable.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Input Reception & Validation

#### Overview
This block listens for incoming Telegram messages, injects global configuration attributes (such as the target Google Sheet and sender identity), extracts and normalizes URLs from the message body, and determines whether the execution should proceed with an audit or return a help message.

#### Nodes Involved
- Receive Telegram message (`n8n-nodes-base.telegramTrigger`)
- Set config (`n8n-nodes-base.set`)
- Find a URL in the message (`n8n-nodes-base.code`)
- Was a URL sent? (`n8n-nodes-base.if`)
- Reply with help (`n8n-nodes-base.telegram`)

#### Node Details

##### 1. Receive Telegram message
- **Type and Technical Role:** `n8n-nodes-base.telegramTrigger` (v1.1) — Listens for new update events from Telegram via webhook.
- **Configuration Choices:** Configured to watch for message updates (`"updates": ["message"]`).
- **Key Expressions or Variables:** None (trigger node).
- **Input and Output Connections:** 
  - Input: None (Workflow entry point)
  - Output: Connects to `Set config`.
- **Version-Specific Requirements:** Requires a registered Telegram bot token.
- **Edge Cases / Potential Failures:** Webhook registration failures or token invalidation will halt trigger ingestion.
- **Sub-workflow Reference:** None.

##### 2. Set config
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.4) — Injects static and dynamic configuration metadata into the data stream.
- **Configuration Choices:** Assigns spreadsheet identifiers, sheet name, sender name, and computes the current date.
- **Key Expressions or Variables:** 
  - `today`: `={{ $now.toFormat('dd MMM yyyy') }}`
- **Input and Output Connections:**
  - Input: `Receive Telegram message`
  - Output: Connects to `Find a URL in the message`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** Hardcoded placeholders (`YOUR_GOOGLE_SHEET_ID`, `YOUR_NAME`) must be updated prior to production use.
- **Sub-workflow Reference:** None.

##### 3. Find a URL in the message
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Custom JavaScript processor that extracts URLs, normalizes trailing punctuation, identifies user metadata, and defines help text templates.
- **Configuration Choices:** Executes custom ES6 logic to parse incoming message properties.
- **Key Expressions or Variables:** Accesses upstream node data via `$('Receive Telegram message').item.json` and `$('Set config').item.json`.
- **Input and Output Connections:**
  - Input: `Set config`
  - Output: Connects to `Was a URL sent?`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** Malformed payloads missing text or chat properties default gracefully to empty strings.
- **Sub-workflow Reference:** None.

##### 4. Was a URL sent?
- **Type and Technical Role:** `n8n-nodes-base.if` (v2.2) — Conditional router that splits the workflow based on URL validity.
- **Configuration Choices:** Evaluates boolean condition on `$json.hasUrl`.
- **Key Expressions or Variables:** `={{ $json.hasUrl }}`
- **Input and Output Connections:**
  - Input: `Find a URL in the message`
  - Output (TRUE): Connects to `Fetch the website`.
  - Output (FALSE): Connects to `Reply with help`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** None.
- **Sub-workflow Reference:** None.

##### 5. Reply with help
- **Type and Technical Role:** `n8n-nodes-base.telegram` (v1.2) — Telegram action node sending message content back to the chat.
- **Configuration Choices:** Sends text response with attribution disabled (`"appendAttribution": false`).
- **Key Expressions or Variables:** 
  - Text: `={{ $('Find a URL in the message').item.json.helpMessage }}`
  - Chat ID: `={{ $('Find a URL in the message').item.json.chatId }}`
- **Input and Output Connections:**
  - Input: `Was a URL sent?` (FALSE branch)
  - Output: None (Terminal node for non-URL queries).
- **Version-Specific Requirements:** Requires active Telegram API credentials.
- **Edge Cases / Potential Failures:** Chat ID mismatches or blocked bots can cause API errors.
- **Sub-workflow Reference:** None.

---

### Block 1.2: Website Scraping & AI Processing

#### Overview
This block retrieves the target website's raw HTML, strips away non-essential elements (like scripts, styles, and navigation tags), and invokes an OpenAI LLM agent restricted by a structured output parser to generate a comprehensive audit.

#### Nodes Involved
- Fetch the website (`n8n-nodes-base.httpRequest`)
- Strip the HTML down to text (`n8n-nodes-base.code`)
- Write the audit (`@n8n/n8n-nodes-langchain.agent`)
- Chat model (`@n8n/n8n-nodes-lmChatOpenAi`)
- Output Parser - Audit fields (`@n8n/n8n-nodes-langchain.outputParserStructured`)

#### Node Details

##### 1. Fetch the website
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Retrieves remote web page content over HTTP/HTTPS.
- **Configuration Choices:** 
  - URL: `={{ $json.websiteUrl }}`
  - Timeout: 15,000 ms
  - Response Format: Text
  - Error Handling: Configured with `"onError": "continueRegularOutput"` to prevent workflow termination on scraping failures.
- **Key Expressions or Variables:** `={{ $json.websiteUrl }}`
- **Input and Output Connections:**
  - Input: `Was a URL sent?` (TRUE branch)
  - Output: Connects to `Strip the HTML down to text`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** Target servers blocking automated scrapers, DNS resolution failures, or slow response times exceeding the 15-second timeout. Handled gracefully downstream.
- **Sub-workflow Reference:** None.

##### 2. Strip the HTML down to text
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Custom JavaScript processor that cleans raw HTML strings.
- **Configuration Choices:** Removes scripts, styles, comments, navigation blocks, and headers using Regular Expressions, decodes HTML entities, and truncates the resulting string to 4000 characters.
- **Key Expressions or Variables:** Reads HTML payload from `$input.first().json` and configuration from `Find a URL in the message`.
- **Input and Output Connections:**
  - Input: `Fetch the website`
  - Output: Connects to `Write the audit`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** If scraping fails or yields insufficient text, a fallback error message is generated in the payload (`scrapeFailed: true`).
- **Sub-workflow Reference:** None.

##### 3. Write the audit
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.agent` (v1.7) — LangChain AI Agent that processes the cleaned web content using system instructions and requested output schemas.
- **Configuration Choices:** Uses defined prompt input type, links to a chat model and a structured output parser.
- **Key Expressions or Variables:** 
  - Website URL: `{{ $json.websiteUrl }}`
  - Page Content: `{{ $json.pageContent }}`
- **Input and Output Connections:**
  - Input: `Strip the HTML down to text`
  - AI Model Input: Connected from `Chat model`
  - AI Output Parser Input: Connected from `Output Parser - Audit fields`
  - Output: Connects to `Format the reply`.
- **Version-Specific Requirements:** Requires LangChain node dependencies.
- **Edge Cases / Potential Failures:** API rate limits, upstream OpenAI outages, or malformed model responses if schema strictness fails.
- **Sub-workflow Reference:** None.

##### 4. Chat model
- **Type and Technical Role:** `@n8n/n8n-nodes-lmChatOpenAi` (v1.2) — Language model provider configuration node.
- **Configuration Choices:** 
  - Model: `gpt-4o-mini`
  - Temperature: `0.3`
  - Max Tokens: `800`
- **Key Expressions or Variables:** None.
- **Input and Output Connections:**
  - Input: None (Sub-node configuration linked via AI connection)
  - Output: Connects to `Write the audit`.
- **Version-Specific Requirements:** Requires OpenAI API credentials.
- **Edge Cases / Potential Failures:** Authentication failures or account quota depletion.
- **Sub-workflow Reference:** None.

##### 5. Output Parser - Audit fields
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — Enforces strict JSON formatting rules matching the required schema.
- **Configuration Choices:** Manual schema validation enforcing 7 required fields: `businessType`, `overallScore`, `criticalIssues`, `quickWins`, `keyRecommendation`, `estimatedOpportunity`, and `talkingPoints`.
- **Key Expressions or Variables:** None (JSON Schema definition).
- **Input and Output Connections:**
  - Input: None (Sub-node configuration linked via AI connection)
  - Output: Connects to `Write the audit`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** LLM output drift failing schema validation.
- **Sub-workflow Reference:** None.

---

### Block 1.3: Data Logging & Response Delivery

#### Overview
This block compiles the structured AI output and page metadata into a clear plain-text telegram message, writes a permanent audit entry to Google Sheets, and sends the final response back to the requesting Telegram chat.

#### Nodes Involved
- Format the reply (`n8n-nodes-base.code`)
- Log the audit (`n8n-nodes-base.googleSheets`)
- Reply with the audit (`n8n-nodes-base.telegram`)

#### Node Details

##### 1. Format the reply
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript data-mapping node that structures the final response text.
- **Configuration Choices:** Parses OpenAI output properties, handles missing analysis fallbacks, builds multi-section plain text strings, and formats timestamps.
- **Key Expressions or Variables:** Reads AI output from `$input.first().json.output` and page configuration from `Strip the HTML down to text`.
- **Input and Output Connections:**
  - Input: `Write the audit`
  - Output: Connects to `Log the audit`.
- **Version-Specific Requirements:** None.
- **Edge Cases / Potential Failures:** Missing AI attributes default safely to placeholders to prevent downstream execution crashes.
- **Sub-workflow Reference:** None.

##### 2. Log the audit
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (v4.5) — Appends a new row of data to a target Google Sheets document.
- **Configuration Choices:** 
  - Operation: `append`
  - Cell Format: `USER_ENTERED`
  - Mapping Mode: `defineBelow`
- **Key Expressions or Variables:** 
  - `Date`: `={{ $json.today }}`
  - `Score`: `={{ $json.overallScore }}`
  - `Logged At`: `={{ $json.loggedAt }}`
  - `Quick Wins`: `={{ $json.topWins }}`
  - `Top Issues`: `={{ $json.topIssues }}`
  - `Website URL`: `={{ $json.websiteUrl }}`
  - `Requested By`: `={{ $json.fromUsername }}`
  - `Business Type`: `={{ $json.businessType }}`
  - `Key Recommendation`: `={{ $json.keyRecommendation }}`
  - Sheet ID: `={{ $json.sheetId }}`
  - Sheet Name: `={{ $json.sheetName }}`
- **Input and Output Connections:**
  - Input: `Format the reply`
  - Output: Connects to `Reply with the audit`.
- **Version-Specific Requirements:** Requires Google Sheets OAuth2 credentials.
- **Edge Cases / Potential Failures:** Missing spreadsheet tabs, permission errors, or invalid Document IDs will trigger execution failures.
- **Sub-workflow Reference:** None.

##### 3. Reply with the audit
- **Type and Technical Role:** `n8n-nodes-base.telegram` (v1.2) — Telegram action node sending final results back to the chat.
- **Configuration Choices:** Sends text response with attribution disabled (`"appendAttribution": false`).
- **Key Expressions or Variables:** 
  - Text: `={{ $('Format the reply').item.json.auditMessage }}`
  - Chat ID: `={{ $('Format the reply').item.json.chatId }}`
- **Input and Output Connections:**
  - Input: `Log the audit`
  - Output: None (Terminal node).
- **Version-Specific Requirements:** Requires active Telegram API credentials.
- **Edge Cases / Potential Failures:** Invalid chat identifiers or revoked API tokens.
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Telegram message | `n8n-nodes-base.telegramTrigger` | Listens for any message sent to your Telegram bot. | None | Set config | ## 1. Take the message<br>The bot receives any Telegram message and a Code node looks for a URL in it. Anything that is not a URL gets a short help reply explaining how to use the bot.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Set config | `n8n-nodes-base.set` | Sets configuration variables including Sheet ID, sheet name, sender name, and current date. | Receive Telegram message | Find a URL in the message | ## 1. Take the message<br>The bot receives any Telegram message and a Code node looks for a URL in the message. Anything that is not a URL gets a short help reply explaining how to use the bot.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Find a URL in the message | `n8n-nodes-base.code` | Reads message text, extracts URL, handles trailing punctuation, and sets up help message payloads. | Set config | Was a URL sent? | ## 1. Take the message<br>The bot receives any Telegram message and a Code node looks for a URL in the message. Anything that is not a URL gets a short help reply explaining how to use the bot.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Was a URL sent? | `n8n-nodes-base.if` | Evaluates if a valid URL was detected in the incoming message. | Find a URL in the message | Fetch the website, Reply with help | ## 1. Take the message<br>The bot receives any Telegram message and a Code node looks for a URL in the message. Anything that is not a URL gets a short help reply explaining how to use the bot.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Reply with help | `n8n-nodes-base.telegram` | Sends a help message when no URL is provided. | Was a URL sent? | None | ## 1. Take the message<br>The bot receives any Telegram message and a Code node looks for a URL in the message. Anything that is not a URL gets a short help reply explaining how to use the bot.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Fetch the website | `n8n-nodes-base.httpRequest` | Fetches target web page HTML with a 15-second timeout. | Was a URL sent? | Strip the HTML down to text | ## 2. Scrape and audit<br>The page is fetched with a fifteen second timeout and stripped to plain text, then GPT returns a structured audit: score, business type, top issues, quick wins and a recommendation.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Strip the HTML down to text | `n8n-nodes-base.code` | Cleans raw HTML, removes non-essential sections, and formats readable text for LLM consumption. | Fetch the website | Write the audit | ## 2. Scrape and audit<br>The page is fetched with a fifteen second timeout and stripped to plain text, then GPT returns a structured audit: score, business type, top issues, quick wins and a recommendation.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Write the audit | `@n8n/n8n-nodes-langchain.agent` | AI agent analyzing website content using system prompt instructions. | Strip the HTML down to text | Format the reply | ## 2. Scrape and audit<br>The page is fetched with a fifteen second timeout and stripped to plain text, then GPT returns a structured audit: score, business type, top issues, quick wins and a recommendation.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Chat model | `@n8n/n8n-nodes-lmChatOpenAi` | Connects OpenAI gpt-4o-mini chat model to the AI agent. | None (AI sub-node) | Write the audit | ## 2. Scrape and audit<br>The page is fetched with a fifteen second timeout and stripped to plain text, then GPT returns a structured audit: score, business type, top issues, quick wins and a recommendation. |
| Output Parser - Audit fields | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured JSON output schema containing 7 required audit fields. | None (AI sub-node) | Write the audit | ## 2. Scrape and audit<br>The page is fetched with a fifteen second timeout and stripped to plain text, then GPT returns a structured audit: score, business type, top issues, quick wins and a recommendation. |
| Format the reply | `n8n-nodes-base.code` | Formats AI output into a clear plain-text Telegram message and maps data attributes. | Write the audit | Log the audit | ## 3. Log and reply<br>The audit is formatted for Telegram, written to your sheet first so nothing is lost, then sent back to whoever asked.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Log the audit | `n8n-nodes-base.googleSheets` | Appends audit details as a new row in Google Sheets. | Format the reply | Reply with the audit | ## 3. Log and reply<br>The audit is formatted for Telegram, written to your sheet first so nothing is lost, then sent back to whoever asked.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>3. If there is no URL, the sender gets a short help message and nothing else runs.<br>4. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>5. The HTML is stripped to readable text so only the content reaches the model.<br>6. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>7. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives you.<br>2. Add that token as a Telegram credential and connect it to all three Telegram nodes.<br>3. Add your OpenAI API key and connect your Google Sheets account.<br>4. Open Set config and replace the sheet ID and your name.<br>5. Create a tab called Website Audits with headers for Date, Website URL, Business Type, Score, Top Issues, Quick Wins, Key Recommendation and Requested By.<br><br>### Customization<br>Edit the agent prompt to audit for a specific angle, such as accessibility or ecommerce conversion. |
| Reply with the audit | `n8n-nodes-base.telegram` | Sends the completed audit result back to the Telegram chat. | Log the audit | None | ## 3. Log and reply<br>The audit is formatted for Telegram, written to your sheet first so nothing is lost, then sent back to whoever asked.<br><br># Telegram bot that audits any website with GPT<br><br>Paste a website URL into your Telegram bot and get a sales-ready audit back in seconds: an overall score, what kind of business it is, the biggest problems, the quickest wins, and a recommendation you can use as a talking point. Every audit is logged to a Google Sheet, so it doubles as a prospect research log.<br><br>### How it works<br>1. The bot receives any message sent to it. One config node holds your sheet and your name.<br>2. A Code nodeль holds your sheet and your name.<br>3. A Code node looks for a URL in the message and tidies up trailing punctuation.<br>4. If there is no URL, the sender gets a short help message and nothing else runs.<br>5. If there is one, the page is fetched with a fifteen second timeout. A blocked or slow site does not crash the run.<br>6. The HTML is stripped to readable text so only the content reaches the model.<br>7. GPT returns a structured audit: score, business type, top issues, quick wins and a key recommendation.<br>8. The audit is formatted for Telegram, written to your sheet, then sent back to whoever asked.<br><br>### Setup<br>1. Message @BotFather on Telegram, send /newbot, and copy the bot token it gives user.<br>*(Content continues in sticky note definition)* |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps in sequence:

1. **Create Trigger Node:**
   - Add a **Receive Telegram message** (`n8n-nodes-base.telegramTrigger`) node.
   - Configure updates to listen for `message`.
   - Configure credentials to connect to your Telegram API token.

2. **Add Configuration Node:**
   - Add a **Set config** (`n8n-nodes-base.set`) node.
   - Connect **Receive Telegram message** to **Set config**.
   - Create assignments:
     - `sheetId`: `YOUR_GOOGLE_SHEET_ID` (string)
     - `sheetName`: `Website Audits` (string)
     - `senderName`: `YOUR_NAME` (string)
     - `today`: `={{ $now.toFormat('dd MMM yyyy') }}` (string)

3. **Add URL Extraction Node:**
   - Add a **Find a URL in the message** (`n8n-nodes-base.code`) node.
   - Connect **Set config** to this node.
   - Paste the custom JavaScript extraction code that parses incoming messages and identifies URLs via regular expressions (`/(https?:\/\/[^\s]+)/i`).

4. **Add Conditional Router:**
   - Add a **Was a URL sent?** (`n8n-nodes-base.if`) node.
   - Connect **Find a URL in the message** to this node.
   - Set condition to evaluate `={{ $json.hasUrl }}` equal to `true`.

5. **Configure Help Message Branch (FALSE Path):**
   - Add a **Reply with help** (`n8n-nodes-base.telegram`) node.
   - Connect the FALSE output of **Was a URL sent?** to this node.
   - Set text expression to `={{ $('Find a URL in the message').item.json.helpMessage }}` and chat ID to `={{ $('Find a URL in the message').item.json.chatId }}` using the same Telegram API credentials.

6. **Configure Web Scraping (TRUE Path):**
   - Add a **Fetch the website** (`n8n-nodes-base.httpRequest`) node.
   - Connect the TRUE output of **Was a URL sent?** to this node.
   - Set URL to `={{ $json.websiteUrl }}`, timeout to `15000`, response format to `text`, and configure error handling to **Continue Regular Output**.

7. **Add HTML Cleaning Node:**
   - Add a **Strip the HTML down to text** (`n8n-nodes-base.code`) node.
   - Connect **Fetch the website** to this node.
   - Paste the code block that cleans script tags, styles, navigation, headers, and footers while preserving body text and capping length at 4,000 characters.

8. **Configure LangChain AI Agent:**
   - Add a **Write the audit** (`@n8n/n8n-nodes-langchain.agent`) node.
   - Connect **Strip the HTML down to text** to this node.
   - Prompt input type: `Define`. Populate the system prompt template requesting strict JSON output containing `businessType`, `overallScore`, `criticalIssues`, `quickWins`, `keyRecommendation`, `estimatedOpportunity`, and `talkingPoints`.
   - Attach a **Chat model** (`@n8n/n8n-nodes-lmChatOpenAi`) sub-node configured with model `gpt-4o-mini`, temperature `0.3`, max tokens `800`, and active OpenAI credentials.
   - Attach an **Output Parser - Audit fields** (`@n8n/n8n-nodes-langchain.outputParserStructured`) sub-node configured with the manual JSON schema defining the 7 expected audit parameters.

9. **Format AI Response:**
   - Add a **Format the reply** (`n8n-nodes-base.code`) node.
   - Connect **Write the audit** to this node.
   - Paste the JavaScript code that extracts output arrays, structures a plain-text Telegram report, and sets fallback values for failed parses.

10. **Log Audit to Google Sheets:**
    - Add a **Log the audit** (`n8n-nodes-base.googleSheets`) node.
    - Connect **Format the reply** to this node.
    - Configure operation to `append`, mapping mode to `define below`, cell format to `USER_ENTERED`, Document ID to `={{ $json.sheetId }}`, Sheet Name to `={{ $json.sheetName }}`, and map columns (`Date`, `Score`, `Logged At`, `Quick Wins`, `Top Issues`, `Website URL`, `Requested By`, `Business Type`, `Key Recommendation`) using corresponding expressions.
    - Authenticate with Google Sheets OAuth2 credentials.

11. **Send Final Audit Response:**
    - Add a **Reply with the audit** (`n8n-nodes-base.telegram`) node.
    - Connect **Log the audit** to this node.
    - Set text expression to `={{ $('Format the reply').item.json.auditMessage }}` and chat ID to `={{ $('Format the reply').item.json.chatId }}` using the Telegram API credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a Telegram bot via BotFather | [Telegram BotFather](https://t.me/BotFather) |
| Obtain OpenAI API credentials | [OpenAI API Keys Platform](https://platform.openai.com/api-keys) |
| Enable Google Sheets API in Google Cloud | [Google Cloud Console](https://console.cloud.google.com) |