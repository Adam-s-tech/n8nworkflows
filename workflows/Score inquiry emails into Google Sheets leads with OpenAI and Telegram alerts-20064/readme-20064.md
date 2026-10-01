Score inquiry emails into Google Sheets leads with OpenAI and Telegram alerts

https://n8nworkflows.xyz/workflows/score-inquiry-emails-into-google-sheets-leads-with-openai-and-telegram-alerts-20064


# Score inquiry emails into Google Sheets leads with OpenAI and Telegram alerts

### 1. Workflow Overview

This workflow automates the capture and qualification of sales leads originating from inbound email messages. Designed for freelancers, agencies, and small service businesses, it polls an inbox, filters out irrelevant noise, leverages an AI language model to extract structured lead data, synchronizes the record to a CRM-like Google Sheet, and triggers instant alerts via Telegram for high-priority opportunities.

The workflow logic is grouped into four distinct functional blocks:

- **1.1 Input Reception & Configuration:** Polls the mailbox periodically and establishes core global variables (chat IDs, budget thresholds, ignored sender keywords).
- **1.2 Filtering & Noise Reduction:** Sanitizes raw email content by stripping HTML tags, removing quoted reply threads, and discarding newsletters, auto-replies, bounces, and blacklisted senders before incurring AI processing costs.
- **1.3 AI Processing & Data Normalization:** Uses an OpenAI language model to extract structured attributes (name, company, budget, deadline, service requested) and rigorously validates and cleans the model's output to prevent formula injection and data corruption.
- **1.4 Persistence & Alerting:** Manages records in Google Sheets using the sender's email address as a unique key (inserting new records or updating existing ones) and dispatches formatted, HTML-escaped notification alerts via Telegram for hot leads.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the operational parameters and periodically checks the connected email inbox for incoming messages matching specified criteria.
- **Nodes Involved:** 
  - `Check inbox for new inquiries`
  - `Set lead settings`

- **Node Details:**
  - **Check inbox for new inquiries**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger). Polls a Gmail account every 5 minutes.
    - *Configuration Choices:* Query filter applied: `in:inbox -from:me -category:promotions -category:social`. Simplify mode set to `false` to capture complete headers and body structures.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input: None (Trigger). Output: Connects to `Set lead settings`.
    - *Edge Cases / Potential Failures:* OAuth2 token expiration or revoked Gmail permissions; rate-limiting if polling intervals are set too aggressively.
  - **Set lead settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Transformer). Establishes static environmental parameters and configuration flags for downstream code nodes.
    - *Configuration Choices:* Assigns configuration properties: `telegramChatId`, `hotBudgetMin` (default: 1000), `defaultCurrency` (default: USD), `ignoreSenders` (comma-separated string), and `maxEmailChars` (default: 6000).
    - *Key Expressions/Variables:* Static hardcoded strings and numbers representing user settings.
    - *Input/Output Connections:* Input: `Check inbox for new inquiries`. Output: Connects to `Skip newsletters and auto-replies`.
    - *Edge Cases / Potential Failures:* Missing or invalid target identifiers (e.g., placeholder chat IDs).

---

#### 2.2 Filtering & Noise Reduction
- **Overview:** Evaluates incoming email metadata and textual content, dropping non-inquiries (newsletters, bounces, auto-replies) and formatting clean text for the AI module.
- **Nodes Involved:**
  - `Skip newsletters and auto-replies`
  - `Is it a real inquiry?`

- **Node Details:**
  - **Skip newsletters and auto-replies**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Transformer). Parses raw MIME headers, extracts sender addresses, strips HTML tags, cuts away historical reply threads (`On ... wrote:`), and flags spam or automated messages.
    - *Configuration Choices:* Custom ES6 JavaScript execution parsing upstream configuration arrays.
    - *Key Expressions/Variables:* `$('Set lead settings').first().json` to access global parameters.
    - *Input/Output Connections:* Input: `Set lead settings`. Output: Connects to `Is it a real inquiry?`.
    - *Edge Cases / Potential Failures:* Malformed email headers lacking standard structures; heavily obfuscated HTML bodies that bypass standard tag-stripping regex.
  - **Is it a real inquiry?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Gates the flow based on whether the previous node marked the email as valid.
    - *Configuration Choices:* Evaluates `{{ $json.skip === false }}`.
    - *Key Expressions/Variables:* `{{ $json.skip === false }}`.
    - *Input/Output Connections:* Input: `Skip newsletters and auto-replies`. Output (True branch): Connects to `Extract lead details with AI`. Output (False branch): Terminated/dropped.
    - *Edge Cases / Potential Failures:* Edge-case emails incorrectly flagged as `true` (passed to AI) or valid client inquiries incorrectly caught by the skip filter.

---

#### 2.3 AI Processing & Data Normalization
- **Overview:** Invokes an OpenAI model using an information-extraction prompt to pull structured lead attributes, followed by comprehensive data validation and priority scoring.
- **Nodes Involved:**
  - `Extract lead details with AI`
  - `OpenAI chat model`
  - `Check AI answer and score lead`
  - `Is it a lead?`

- **Node Details:**
  - **Extract lead details with AI**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.informationExtractor` (AI Agent / Structured Output Node). Forces the language model to extract strict schema fields.
    - *Configuration Choices:* Defines data schema attributes (`is_inquiry`, `name`, `company`, `phone`, `service`, `budget`, `currency`, `deadline`, `summary`). System prompt enforces strict anti-hallucination constraints.
    - *Key Expressions/Variables:* `={{ 'From: ' + $json.fromName + ' <' + $json.fromEmail + '>\\nSubject: ' + $json.subject + '\\n\\n' + $json.body }}`.
    - *Input/Output Connections:* Input: `Is it a real inquiry?` and `OpenAI chat model` (AI connection). Output: Connects to `Check AI answer and score lead`.
    - *Edge Cases / Potential Failures:* OpenAI API rate limits, billing errors, or malformed JSON responses failing schema compliance.
  - **OpenAI chat model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Model Provider Sub-node). Configures the underlying LLM engine.
    - *Configuration Choices:* Model set to `gpt-4o-mini`, temperature set to `0` for deterministic outputs.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Connected strictly to `Extract lead details with AI`.
    - *Edge Cases / Potential Failures:* Invalid API keys or account quota exhaustion.
  - **Check AI answer and score lead**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Transformer). Validates untrusted AI outputs, normalizes budgets and phone numbers, neutralizes spreadsheet formula injections, and computes priority levels (`hot` vs `normal`).
    - *Configuration Choices:* Custom JavaScript sanitizing input data strings and calculating lead scores against threshold parameters.
    - *Key Expressions/Variables:* `$('Set lead settings').first().json` and `$('Is it a real inquiry?').all()`.
    - *Input/Output Connections:* Input: `Extract lead details with AI`. Output: Connects to `Is it a lead?`.
    - *Edge Cases / Potential Failures:* Non-standard budget string formats that defy regex matching, resulting in empty budget values.
  - **Is it a lead?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Verifies that the processed record represents a confirmed lead.
    - *Configuration Choices:* Evaluates `{{ $json.isLead }}`.
    - *Key Expressions/Variables:* `{{ $json.isLead }}`.
    - *Input/Output Connections:* Input: `Check AI answer and score lead`. Output (True branch): Connects to `Keep only the lead row`. Output (False branch): Terminated.
    - *Edge Cases / Potential Failures:* Boolean type coercion mismatches.

---

#### 2.4 Persistence & Alerting
- **Overview:** Writes validated lead records to Google Sheets on an upsert basis, and pushes instant notifications to a Telegram chat if the lead is classified as hot.
- **Nodes Involved:**
  - `Keep only the lead row`
  - `Add or update lead row`
  - `Is it a hot lead?`
  - `Format hot lead alert`
  - `Send hot lead alert to Telegram`

- **Node Details:**
  - **Keep only the lead row**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Formatter). Extracts and flattens the nested row object payload.
    - *Configuration Choices:* Mode set to raw JSON stringification of `$json.row`.
    - *Key Expressions/Variables:* `={{ JSON.stringify($json.row) }}`.
    - *Input/Output Connections:* Input: `Is it a lead?`. Output: Connects to `Add or update lead row`.
    - *Edge Cases / Potential Failures:* Serialization errors if payload objects contain circular references.
  - **Add or update lead row**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Database / Spreadsheet Action). Appends a new row or updates an existing row based on a matching key column.
    - *Configuration Choices:* Operation: `appendOrUpdate`. Sheet Name: `Leads`. Matching Column: `key`. Retry on fail enabled (max 3 tries, 2-second wait).
    - *Key Expressions/Variables:* Spreadsheet ID reference (`YOUR_SPREADSHEET_ID`).
    - *Input/Output Connections:* Input: `Keep only the lead row`. Output: Connects to `Is it a hot lead?`.
    - *Edge Cases / Potential Failures:* Google API rate limits, missing sheet headers, or mismatched key columns causing duplicate row insertions.
  - **Is it a hot lead?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Evaluates whether the lead's priority is marked as hot.
    - *Configuration Choices:* Evaluates `={{ $json.priority === 'hot' }}`.
    - *Key Expressions/Variables:* `={{ $json.priority === 'hot' }}`.
    - *Input/Output Connections:* Input: `Add or update lead row`. Output (True branch): Connects to `Format hot lead alert`. Output (False branch): Terminated.
    - *Edge Cases / Potential Failures:* String matching failures due to whitespace or unexpected casing.
  - **Format hot lead alert**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Transformer). Constructs an HTML-formatted notification string and safely escapes special characters to prevent Telegram API formatting rejections.
    - *Configuration Choices:* Custom JavaScript processing lead attributes and applying HTML entity escaping.
    - *Key Expressions/Variables:* `$('Set lead settings').first().json`.
    - *Input/Output Connections:* Input: `Is it a hot lead?`. Output: Connects to `Send hot lead alert to Telegram`.
    - *Edge Cases / Potential Failures:* Message length exceeding Telegram's 4,000-character limit (mitigated by `.slice(0, 4000)`).
  - **Send hot lead alert to Telegram**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Notification Action). Dispatches instant messages via a Telegram Bot.
    - *Configuration Choices:* Parse Mode set to `HTML`. Append Attribution set to `false`. Retry on fail enabled.
    - *Key Expressions/Variables:* `={{ $json.text }}` and `={{ $json.chatId }}`.
    - *Input/Output Connections:* Input: `Format hot lead alert`. Output: None (Terminal node).
    - *Edge Cases / Potential Failures:* Invalid Telegram Bot tokens, revoked chat access, or unescaped HTML characters causing parsing exceptions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Documentation placeholder outlining the workflow description, setup instructions, and requirements. | None | None | ## Turn inquiry emails into leads in Google Sheets with OpenAI and Telegram alerts<br><br>**Who's it for:** freelancers, agencies and small service businesses whose leads arrive as plain emails and end up forgotten in the inbox.<br><br>### How it works<br>1. **Check inbox for new inquiries** polls Gmail every 5 minutes.<br>2. **Skip newsletters and auto-replies** drops mailing lists, out-of-office replies, bounces and ignored senders before any AI call, so you don't pay for noise.<br>3. **Extract lead details with AI** fills fixed fields: is it an inquiry, name, company, phone, service, budget, deadline, a one-line summary. It is told not to guess.<br>4. **Check AI answer and score lead** double-checks what the AI returned: the email address is taken from the sender, the budget has to be a real number, and any text that would turn into a spreadsheet formula is neutralized. A lead is **hot** when the budget reaches your threshold or the deadline says urgent.<br>5. **Add or update lead row** writes to Google Sheets matched on the sender's email: a second email from the same person updates one row.<br>6. Hot leads also go to Telegram as an HTML-escaped alert with a link to the email.<br><br>### Setup<br>1. Create a sheet named **Leads** with columns: `key, received_at, name, email, company, phone, service, budget, currency, deadline, summary, subject, priority, gmail_link`.<br>2. Add Gmail, OpenAI, Google Sheets and Telegram credentials; put your spreadsheet ID in **Add or update lead row**.<br>3. In **Set lead settings**, set your Telegram chat ID, the hot budget threshold and senders to ignore.<br><br>### Requirements<br>Gmail, an OpenAI API key, Google Sheets, a Telegram bot.<br><br>### Customization tips<br>Add fields in **Extract lead details with AI**, or replace the sheet node with your CRM.<br> |
| **Section: filter** | `n8n-nodes-base.stickyNote` | Visual grouping note for the input filtering phase. | None | None | ## 1. New email, skip the noise<br>Newsletters, auto-replies and ignored senders stop here, before any AI call. |
| **Section: extract** | `n8n-nodes-base.stickyNote` | Visual grouping note for the AI extraction phase. | None | None | ## 2. AI reads the email<br>Fixed fields only; the answer is checked before it is saved. |
| **Section: save** | `n8n-nodes-base.stickyNote` | Visual grouping note for the persistence and alerting phase. | None | None | ## 3. Save and alert<br>One row per sender; hot leads go to Telegram. |
| **Check inbox for new inquiries** | `n8n-nodes-base.gmailTrigger` | Polls Gmail inbox every 5 minutes for new matching messages. | None | Set lead settings | |
| **Set lead settings** | `n8n-nodes-base.set` | Establishes configuration variables (chat ID, budget threshold, ignore list). | Check inbox for new inquiries | Skip newsletters and auto-replies | |
| **Skip newsletters and auto-replies** | `n8n-nodes-base.code` | Cleans message body, strips HTML, and filters out noise and automated emails. | Set lead settings | Is it a real inquiry? | |
| **Is it a real inquiry?** | `n8n-nodes-base.if` | Routes valid non-skipped emails forward. | Skip newsletters and auto-replies | Extract lead details with AI | |
| **Extract lead details with AI** | `@n8n/n8n-nodes-langchain.informationExtractor` | Extracts structured lead attributes from the cleaned email text using OpenAI. | Is it a real inquiry?, OpenAI chat model | Check AI answer and score lead | |
| **OpenAI chat model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides the GPT-4o-mini language model backend for extraction. | None | Extract lead details with AI | |
| **Check AI answer and score lead** | `n8n-nodes-base.code` | Validates AI outputs, neutralizes formula injections, and computes hot/normal priority. | Extract lead details with AI | Is it a lead? | |
| **Is it a lead?** | `n8n-nodes-base.if` | Verifies that the processed payload is a confirmed lead. | Check AI answer and score lead | Keep only the lead row | |
| **Keep only the lead row** | `n8n-nodes-base.set` | Flattens the structured row object payload. | Is it a lead? | Add or update lead row | |
| **Add or update lead row** | `n8n-nodes-base.googleSheets` | Upserts lead data into Google Sheets matched on the sender's email key. | Keep only the lead row | Is it a hot lead? | |
| **Is it a hot lead?** | `n8n-nodes-base.if` | Routes hot-priority leads to the alert branch. | Add or update lead row | Format hot lead alert | |
| **Format hot lead alert** | `n8n-nodes-base.code` | Prepares and HTML-escapes notification strings for Telegram. | Is it a hot lead? | Send hot lead alert to Telegram | |
| **Send hot lead alert to Telegram** | `n8n-nodes-base.telegram` | Dispatches formatted alerts to the configured Telegram chat. | Format hot lead alert | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet:**
   - Create a new Google Spreadsheet and name the primary tab `Leads`.
   - Add the following exact column headers in row 1: `key`, `received_at`, `name`, `email`, `company`, `phone`, `service`, `budget`, `currency`, `deadline`, `summary`, `subject`, `priority`, `gmail_link`.

2. **Add Trigger Node (`Check inbox for new inquiries`):**
   - Create a **Gmail Trigger** node.
   - Set polling interval to every `5` minutes.
   - Configure search query (`q`): `in:inbox -from:me -category:promotions -category:social`.
   - Set **Simplify** option to `false`.

3. **Add Configuration Node (`Set lead settings`):**
   - Create a **Set (Edit Fields)** node connected downstream of the Gmail Trigger.
   - Add string and number assignments:
     - `telegramChatId`: `"YOUR_CHAT_ID"` (String)
     - `hotBudgetMin`: `1000` (Number)
     - `defaultCurrency`: `"USD"` (String)
     - `ignoreSenders`: `"noreply,no-reply,donotreply,mailer-daemon,newsletter,notifications,billing"` (String)
     - `maxEmailChars`: `6000` (Number)
   - Enable **Include Other Fields**.

4. **Add Noise Filter Code Node (`Skip newsletters and auto-replies`):**
   - Create a **Code** node (JavaScript mode).
   - Paste the logic to parse headers, strip HTML tags via regex, cut quoted reply lines, and check against the ignored senders list. Ensure it outputs `messageId`, `fromEmail`, `fromName`, `subject`, `receivedAt`, `body`, `skip`, and `skipReason`.

5. **Add Condition Router (`Is it a real inquiry?`):**
   - Create an **If** node.
   - Set condition: `{{ $json.skip === false }}` (Boolean operation: true).

6. **Add AI Extraction Node and Language Model:**
   - Create an **Information Extractor** node (`@n8n/n8n-nodes-langchain.informationExtractor`).
   - Configure the system prompt instructing the model to act as an email reader that extracts structured attributes without guessing.
   - Add schema attributes: `is_inquiry` (Boolean), `name` (String), `company` (String), `phone` (String), `service` (String), `budget` (String), `currency` (String), `deadline` (String), `summary` (String).
   - Set input text expression: `={{ 'From: ' + $json.fromName + ' <' + $json.fromEmail + '>\\nSubject: ' + $json.subject + '\\n\\n' + $json.body }}`.
   - Create an **OpenAI Chat Model** sub-node, set model to `gpt-4o-mini`, temperature to `0`, and connect it to the AI language model input of the Information Extractor.

7. **Add AI Validation Code Node (`Check AI answer and score lead`):**
   - Create a **Code** node (JavaScript mode).
   - Implement validation logic to override AI-generated email addresses with verified sender emails, sanitize formula-injection attempts (`=`, `+`, `-`, `@`), parse budgets, and evaluate priority rules (`hot` if budget $\ge$ threshold or deadline contains urgent keywords).

8. **Add Lead Verification Router (`Is it a lead?`):**
   - Create an **If** node.
   - Set condition: `{{ $json.isLead }}` (Boolean operation: true).

9. **Add Row Formatting Node (`Keep only the lead row`):**
   - Create a **Set** node.
   - Set mode to **Raw** and JSON Output to: `={{ JSON.stringify($json.row) }}`.

10. **Add Google Sheets Persistence Node (`Add or update lead row`):**
    - Create a **Google Sheets** node.
    - Set operation to `Append or Update`.
    - Select Document by ID (`YOUR_SPREADSHEET_ID`) and Sheet Name (`Leads`).
    - Set matching columns to `key`. Enable retry on fail (3 attempts, 2000ms wait). Configure credentials.

11. **Add Hot Lead Router (`Is it a hot lead?`):**
    - Create an **If** node.
    - Set condition: `={{ $json.priority === 'hot' }}` (Boolean operation: true).

12. **Add Alert Formatting Node (`Format hot lead alert`):**
    - Create a **Code** node (JavaScript mode).
    - Implement HTML-escaping helper functions and construct formatted notification strings containing lead details and Gmail deep links, capped at 4000 characters.

13. **Add Telegram Notification Node (`Send hot lead alert to Telegram`):**
    - Create a **Telegram** node.
    - Set Action to send message.
    - Set Text expression: `={{ $json.text }}` and Chat ID expression: `={{ $json.chatId }}`.
    - Configure additional fields: Parse Mode = `HTML`, Append Attribution = `false`. Enable retry on fail. Configure Telegram Bot credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Audience | Freelancers, agencies, and small service businesses managing client leads via email. |
| Core Integrations Required | Gmail OAuth2, OpenAI API Key, Google Sheets OAuth2 / Service Account, Telegram Bot Token. |