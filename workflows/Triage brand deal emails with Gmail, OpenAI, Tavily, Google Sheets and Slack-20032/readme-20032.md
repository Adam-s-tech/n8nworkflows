Triage brand deal emails with Gmail, OpenAI, Tavily, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/triage-brand-deal-emails-with-gmail--openai--tavily--google-sheets-and-slack-20032


# Triage brand deal emails with Gmail, OpenAI, Tavily, Google Sheets and Slack

### 1. Workflow Overview

This workflow automates the detection, research, evaluation, and initial response preparation for brand deal and sponsorship inquiries arriving in a Gmail inbox. Designed for content creators, it filters incoming mail, evaluates sponsorship viability using OpenAI, conducts automated company and campaign research via Tavily, calculates data-driven counter-offer rates using a custom JavaScript pricing engine, generates a professional draft reply in Gmail, and logs the pipeline metrics to Google Sheets and Slack.

The workflow logic is grouped into five functional blocks:
- **1.1 Input Reception & Normalization:** Polls the Gmail inbox, captures unread emails, and normalizes raw payloads into clean structured objects.
- **1.2 Intake & Classification:** Uses OpenAI to classify emails as sponsorship opportunities, extract deal terms, and filter out irrelevant messages.
- **1.3 Sort & Brand Research:** Applies a Gmail tracking label to confirmed deals and executes parallelized web searches via Tavily to analyze brand funding, market tier, and past creator campaigns.
- **1.4 Pricing Engine & Response Drafting:** Computes an optimized rate based on audience metrics, engagement, and contract multipliers, then generates a tailored counter-offer draft via OpenAI.
- **1.5 Pipeline Logging & Notifications:** Saves the generated draft within the original Gmail thread, records structured deal telemetry in Google Sheets, and sends a summary notification to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block monitors the inbox for unread messages on a recurring schedule and standardizes email metadata for down-stream processing.

- **Gmail Trigger**
  - **Type & Role:** `n8n-nodes-base.gmailTrigger` (Trigger Node). Polls the Gmail inbox every 5 minutes for unread messages.
  - **Configuration:** Simple mode disabled, filtered by `labelIds: ["INBOX"]` and `readStatus: "unread"`. Polling schedule set to execute every 5 minutes.
  - **Key Expressions:** None.
  - **Connections:** Input: None (Trigger). Output: Connects to *Normalize Email*.
  - **Requirements:** Requires valid Gmail OAuth2 credentials.
  - **Edge Cases & Failure Types:** Authentication token expiration, rate limits during high email volume.

- **Normalize Email**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Flattens raw Gmail payload structures into consistent fields.
  - **Configuration:** Executed once for each incoming item using custom JavaScript. Extracts sender address, display name, sender domain, cleaned body text (truncated to 6,000 characters), and timestamps.
  - **Key Expressions:** Evaluates `$json` payload fields.
  - **Connections:** Input: *Gmail Trigger*. Output: Connects to *AI: Classify Email*.
  - **Requirements:** None.
  - **Edge Cases & Failure Types:** Missing sender objects or malformed email bodies handled through fallback empty strings.

---

#### 2.2 Intake & Classification
This block uses an AI model to evaluate whether an email is a legitimate sponsorship opportunity and extracts structured deal parameters.

- **AI: Classify Email**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (AI Processing). Evaluates email context and extracts JSON-formatted sponsorship metadata.
  - **Configuration:** Uses model `gpt-4o-mini` with temperature set to `0.1` for deterministic classification. Enforces strict JSON output rules.
  - **Key Expressions:** Uses HTML/Handlebars expressions to inject sender name, email address, domain, subject, and body into the prompt.
  - **Connections:** Input: *Normalize Email*. Output: Connects to *Parse Classification*.
  - **Requirements:** Requires OpenAI API credentials.
  - **Edge Cases & Failure Types:** API timeouts, rate limits, or non-JSON model outputs.

- **Parse Classification**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Cleans and parses the AI classification response.
  - **Configuration:** JavaScript execution handling markdown code-block strippping (` ```json `), JSON parsing, corporate domain validation against known free-mail providers, and scam signal accumulation.
  - **Key Expressions:** Parses `$json.message?.content` and accesses `$('Normalize Email').item.json`.
  - **Connections:** Input: *AI: Classify Email*. Output: Connects to *Is Sponsorship?*.
  - **Requirements:** None.
  - **Edge Cases & Failure Types:** Malformed JSON outputs default to an internal `__parse_error` object.

- **Is Sponsorship?**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional Router). Routes valid sponsorship leads forward while discarding non-deals.
  - **Configuration:** Evaluates conditions with loose type validation. Requires `is_sponsorship` to be `true` AND `confidence` greater than or equal to `0.6`.
  - **Key Expressions:** `={{ $json.is_sponsorship }}` and `={{ $json.confidence }}`.
  - **Connections:** Input: *Parse Classification*. Output True branch: Connects to *Label: Brand Deals*. Output False branch: Connects to *Not a Deal — Skip*.
  - **Requirements:** None.
  - **Edge Cases & Failure Types:** Type mismatches in confidence scores.

- **Not a Deal — Skip**
  - **Type & Role:** `n8n-nodes-base.noOp` (Termination / Placeholder). End-point for emails classified as non-sponsorships.
  - **Configuration:** Default passthrough node.
  - **Connections:** Input: *Is Sponsorship?* (False branch). Output: None.

---

#### 2.3 Sort & Brand Research
This block organizes confirmed opportunities within Gmail and gathers external market intelligence regarding brand funding and prior influencer campaigns.

- **Label: Brand Deals**
  - **Type & Role:** `n8n-nodes-base.gmail` (Integration). Applies a specific label to the email thread in Gmail.
  - **Configuration:** Operation set to `addLabels` using target message ID and label ID.
  - **Key Expressions:** `={{ $json.message_id }}`.
  - **Connections:** Input: *Is Sponsorship?* (True branch). Output: Connects to *Research: Funding*.
  - **Requirements:** Gmail OAuth2 credentials and a predefined Gmail label ID.
  - **Edge Cases & Failure Types:** Invalid label ID or missing thread reference.

- **Research: Funding**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Request). Queries the Tavily search engine for corporate funding data.
  - **Configuration:** POST request to `https://api.tavily.com/search` with a 30-second timeout. Uses generic Header Auth (`Authorization: Bearer <key>`).
  - **Key Expressions:** Dynamically constructs search queries using brand names (`={{ JSON.stringify({ query: $('Parse Classification').item.json.brand_name + ' company funding round investors valuation revenue', ... }) }}`).
  - **Connections:** Input: *Label: Brand Deals*. Output: Connects to *Research: Past Campaigns*.
  - **Requirements:** Tavily API key configured via Header Auth.
  - **Edge Cases & Failure Types:** Node error handling configured to *continue regular output* on failure, preventing workflow halts if search limits are hit.

- **Research: Past Campaigns**
  - **Type & Role:** `n8n-nodes-base.httpRequest` (API Request). Queries Tavily for past creator marketing campaigns associated with the brand.
  - **Configuration:** POST request to `https://api.tavily.com/search` with a 30-second timeout and Header Auth.
  - **Key Expressions:** Constructs search queries targeting influencer marketing history.
  - **Connections:** Input: *Research: Funding*. Output: Connects to *Compile Research*.
  - **Requirements:** Tavily API key.
  - **Edge Cases & Failure Types:** Configured to *continue regular output* on error.

- **Compile Research**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Consolidates and formats Tavily search outputs into structured text blocks.
  - **Configuration:** JavaScript execution safely extracting results and search summaries from both upstream HTTP request nodes.
  - **Key Expressions:** References upstream nodes via `$('Research: Funding')` and `$('Research: Past Campaigns')`.
  - **Connections:** Input: *Research: Past Campaigns*. Output: Connects to *AI: Brand Profile*.
  - **Requirements:** None.

- **AI: Brand Profile**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (AI Processing). Compiles research data into a standardized brand legitimacy profile.
  - **Configuration:** Uses model `gpt-4o-mini` with temperature `0.2` to generate structured JSON parameters (funding stage, tier, legitimacy score, red flags).
  - **Key Expressions:** Injects brand name, website, summary, and compiled research blocks into the prompt.
  - **Connections:** Input: *Compile Research*. Output: Connects to *Parse Brand Profile*.
  - **Requirements:** OpenAI API credentials.

- **Parse Brand Profile**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Parses the AI-generated brand profile JSON response.
  - **Configuration:** JavaScript execution handling markdown cleanup and fallback values.
  - **Connections:** Input: *AI: Brand Profile*. Output: Connects to *Creator Rate Config*.
  - **Requirements:** None.

---

#### 2.4 Pricing Engine & Response Drafting
This block applies quantitative pricing models based on audience reach and brand multipliers, then drafts an appropriate negotiation response.

- **Creator Rate Config**
  - **Type & Role:** `n8n-nodes-base.set` (Data Assignment). Defines creator channel metrics, CPM baselines, minimum rates, and media kit URLs.
  - **Configuration:** Static variable assignments for subscribers, average views, engagement rates, and financial constraints.
  - **Connections:** Input: *Parse Brand Profile*. Output: Connects to *Rate Calculator*.
  - **Requirements:** Manual user configuration of creator channel stats.

- **Rate Calculator**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation / Pricing Engine). Calculates deterministic pricing models without AI inference.
  - **Configuration:** Pure JavaScript executing reach calculations, format weighting multipliers, engagement adjustments, tier boosts, usage rights scaling, exclusivity premiums, and rush fees. Determines strategy pathways (`quote_rate`, `accept_with_upsell`, `counter_meet_near`, `counter_anchor`, `convert_to_paid`, `verify_first`).
  - **Connections:** Input: *Creator Rate Config*. Output: Connects to *AI: Draft Counter-Offer*.
  - **Requirements:** None.

- **AI: Draft Counter-Offer**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (AI Processing). Generates the body and subject of the reply email based on calculated negotiation strategies.
  - **Configuration:** Uses model `gpt-4o-mini` with temperature `0.5` to maintain professional tone variation.
  - **Key Expressions:** Injects creator identity, negotiation strategy, counter figures, upsell concepts, and original email body.
  - **Connections:** Input: *Rate Calculator*. Output: Connects to *Parse Draft*.
  - **Requirements:** OpenAI API credentials.

- **Parse Draft**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Parses the AI-generated draft response and normalizes subject line prefixes.
  - **Configuration:** JavaScript execution handling JSON cleanup and subject line formatting (`Re: ...`).
  - **Connections:** Input: *AI: Draft Counter-Offer*. Output: Connects to *Gmail: Create Draft Reply*.
  - **Requirements:** None.

- **Gmail: Create Draft Reply**
  - **Type & Role:** `n8n-nodes-base.gmail` (Integration). Creates a draft reply inside the original email thread.
  - **Configuration:** Resource set to `draft`, operation set to create using dynamic draft bodies, destination recipient addresses, and thread IDs.
  - **Key Expressions:** `={{ $json.draft_body }}`, `={{ $json.from_email }}`, `={{ $json.thread_id }}`, `={{ $json.draft_subject }}`.
  - **Connections:** Input: *Parse Draft*. Output: Connects to *Build Log Row*.
  - **Requirements:** Gmail OAuth2 credentials.
  - **Edge Cases & Failure Types:** Thread ID mismatches or invalid recipient addresses.

---

#### 2.5 Pipeline Logging & Notifications
This block records deal telemetry into a tracking spreadsheet and dispatches review alerts to communication channels.

- **Build Log Row**
  - **Type & Role:** `n8n-nodes-base.code` (Data Transformation). Formats deal telemetry into named columns matching Google Sheets headers.
  - **Configuration:** JavaScript object mapping mapping received dates, brand names, offers, rates, strategies, scores, and Gmail web links.
  - **Key Expressions:** Accesses `$json.id` from the created draft and upstream rate calculators.
  - **Connections:** Input: *Gmail: Create Draft Reply*. Output: Connects to *Log to Google Sheets*.
  - **Requirements:** None.

- **Log to Google Sheets**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Integration). Appends structured deal log rows to a tracking spreadsheet.
  - **Configuration:** Operation set to `append`, mapping input data automatically to the selected document ID and target sheet name (`Brand Deals`).
  - **Connections:** Input: *Build Log Row*. Output: Connects to *Slack: Notify*.
  - **Requirements:** Google Sheets OAuth2 credentials and a valid Document ID.
  - **Edge Cases & Failure Types:** Spreadsheet schema mismatches or permission errors.

- **Slack: Notify**
  - **Type & Role:** `n8n-nodes-base.slack` (Integration). Posts a rich summary notification to a designated Slack channel.
  - **Configuration:** Selects channel by name (`#brand-deals`) and formats markdown notification text including deal type, offered amounts, suggested rates, negotiation strategies, legitimacy scores, and direct Gmail thread links.
  - **Key Expressions:** Dynamically accesses properties from upstream parsing nodes using expression syntax (`{{ $('Parse Draft').item.json.brand_name }}`).
  - **Connections:** Input: *Log to Google Sheets*. Output: None (Terminal Node).
  - **Requirements:** Slack OAuth2 credentials and channel access permissions.
  - **Edge Cases & Failure Types:** Invalid channel names or missing bot scopes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail Trigger | `n8n-nodes-base.gmailTrigger` | Polls unread inbox messages | None | Normalize Email | 🤝 Brand Deal Inbox Manager<br>1️⃣ Intake & Classification |
| Normalize Email | `n8n-nodes-base.code` | Flattens raw Gmail payload | Gmail Trigger | AI: Classify Email | 🤝 Brand Deal Inbox Manager<br>1️⃣ Intake & Classification |
| AI: Classify Email | `@n8n/n8n-nodes-langchain.openAi` | Extracts deal terms via LLM | Normalize Email | Parse Classification | 🤝 Brand Deal Inbox Manager<br>1️⃣ Intake & Classification |
| Parse Classification | `n8n-nodes-base.code` | Cleans AI classification JSON | AI: Classify Email | Is Sponsorship? | 🤝 Brand Deal Inbox Manager<br>1️⃣ Intake & Classification |
| Is Sponsorship? | `n8n-nodes-base.if` | Filters valid sponsorship leads | Parse Classification | Label: Brand Deals, Not a Deal — Skip | 🤝 Brand Deal Inbox Manager<br>1️⃣ Intake & Classification |
| Not a Deal — Skip | `n8n-nodes-base.noOp` | End point for non-deals | Is Sponsorship? | None | 🤝 Brand Deal Inbox Manager<br>⏭️ Skipped |
| Label: Brand Deals | `n8n-nodes-base.gmail` | Adds tracking label to thread | Is Sponsorship? | Research: Funding | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| Research: Funding | `n8n-nodes-base.httpRequest` | Queries Tavily for funding info | Label: Brand Deals | Research: Past Campaigns | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| Research: Past Campaigns | `n8n-nodes-base.httpRequest` | Queries Tavily for past campaigns | Research: Funding | Compile Research | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| Compile Research | `n8n-nodes-base.code` | Formats Tavily search outputs | Research: Past Campaigns | AI: Brand Profile | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| AI: Brand Profile | `@n8n/n8n-nodes-langchain.openAi` | Builds brand legitimacy profile | Compile Research | Parse Brand Profile | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| Parse Brand Profile | `n8n-nodes-base.code` | Parses brand profile JSON | AI: Brand Profile | Creator Rate Config | 🤝 Brand Deal Inbox Manager<br>2️⃣ Sort & Brand Research |
| Creator Rate Config | `n8n-nodes-base.set` | Sets creator channel metrics | Parse Brand Profile | Rate Calculator | 🤝 Brand Deal Inbox Manager<br>3️⃣ Pricing Engine |
| Rate Calculator | `n8n-nodes-base.code` | Calculates deterministic pricing | Creator Rate Config | AI: Draft Counter-Offer | 🤝 Brand Deal Inbox Manager<br>3️⃣ Pricing Engine |
| AI: Draft Counter-Offer | `@n8n/n8n-nodes-langchain.openAi` | Writes strategy-based reply draft | Rate Calculator | Parse Draft | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |
| Parse Draft | `n8n-nodes-base.code` | Parses generated email draft | AI: Draft Counter-Offer | Gmail: Create Draft Reply | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |
| Gmail: Create Draft Reply | `n8n-nodes-base.gmail` | Creates draft in Gmail thread | Parse Draft | Build Log Row | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |
| Build Log Row | `n8n-nodes-base.code` | Formats deal telemetry for sheets | Gmail: Create Draft Reply | Log to Google Sheets | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |
| Log to Google Sheets | `n8n-nodes-base.googleSheets` | Appends row to deal log sheet | Build Log Row | Slack: Notify | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |
| Slack: Notify | `n8n-nodes-base.slack` | Sends notification to Slack | Log to Google Sheets | None | 🤝 Brand Deal Inbox Manager<br>4️⃣ Draft, Log & Notify |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential configuration steps:

1. **Create Gmail Trigger Node:**
   - Add a **Gmail Trigger** node. Set polling to run every 5 minutes, filter by `INBOX` label IDs, and set read status to `unread`. Configure Gmail OAuth2 credentials.
2. **Add Normalization Code Node:**
   - Connect a **Code** node named `Normalize Email`. Set mode to *Run Once for Each Item* and paste JavaScript code to flatten the payload (`message_id`, `thread_id`, `from_email`, `from_name`, `sender_domain`, `subject`, `body`, `received_at`).
3. **Configure AI Classification Node:**
   - Add an **OpenAI** node named `AI: Classify Email`. Select model `gpt-4o-mini`, set temperature to `0.1`, enable JSON output, and configure OpenAI API credentials. Supply the system prompt requiring strict JSON output for sponsorship detection.
4. **Parse Classification Node:**
   - Add a **Code** node named `Parse Classification` to clean JSON outputs, validate corporate sender domains, and accumulate scam signals.
5. **Add Is Sponsorship Conditional Router:**
   - Add an **If** node named `Is Sponsorship?`. Create two conditions: `is_sponsorship` equals `true` (boolean) and `confidence` is greater than or equal to `0.6` (number).
6. **Configure Gmail Label Node (True Branch):**
   - Connect the True output of the If node to a **Gmail** node named `Label: Brand Deals`. Set operation to `addLabels`, supply your target Gmail Label ID, and use message ID `={{ $json.message_id }}`.
7. **Configure Tavily Research Nodes:**
   - Add an **HTTP Request** node named `Research: Funding`. Set method to POST, URL to `https://api.tavily.com/search`, timeout to `30000ms`, and specify Header Auth (`Authorization: Bearer <key>`). Set error handling to *Continue Regular Output on Error*.
   - Add a second **HTTP Request** node named `Research: Past Campaigns` configured identically with query parameters targeting influencer campaigns.
8. **Compile & Profile Brand Nodes:**
   - Add a **Code** node named `Compile Research` to format search responses.
   - Add an **OpenAI** node named `AI: Brand Profile` (`gpt-4o-mini`, temperature `0.2`, JSON output mode) to generate a legitimacy assessment.
   - Add a **Code** node named `Parse Brand Profile` to clean the profile response.
9. **Set Creator Rate Configuration:**
   - Add a **Set (Edit Fields)** node named `Creator Rate Config`. Define custom values for creator name, niche, subscriber counts, average views, engagement rate percentage, base CPM, minimum deal rate, currency, and media kit URL.
10. **Implement Rate Calculator:**
    - Add a **Code** node named `Rate Calculator` executing deterministic JS pricing logic based on reach, CPM, format weights, and contract multipliers.
11. **Generate Counter-Offer Draft:**
    - Add an **OpenAI** node named `AI: Draft Counter-Offer` (`gpt-4o-mini`, temperature `0.5`, JSON output) using prompt instructions mapped to the selected negotiation strategy.
    - Add a **Code** node named `Parse Draft` to format the subject and body strings.
12. **Create Gmail Draft Reply:**
    - Add a **Gmail** node named `Gmail: Create Draft Reply`. Set resource to `draft`, configure send-to recipient email, thread ID, subject, and message body expressions.
13. **Log to Google Sheets:**
    - Add a **Code** node named `Build Log Row` to map deal data into table columns.
    - Add a **Google Sheets** node named `Log to Google Sheets`. Set operation to `append`, configure Google Sheets OAuth2 credentials, document ID, and target sheet tab (`Brand Deals`).
14. **Send Slack Notification:**
    - Add a **Slack** node named `Slack: Notify`. Configure Slack OAuth2 credentials, select channel type by name (`#brand-deals`), and paste the notification message template containing summary details and Gmail thread deep-links.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Free API Key for Tavily research searches | [Tavily AI](https://tavily.com) |
| Workflow safety design: Human-in-the-loop review via Gmail drafts | Prevents automated outbound messages; all drafts require manual review before sending. |
| Automatic error resilience on web research | Tavily HTTP request nodes are configured to continue execution on error, preventing API downtime from breaking the workflow. |