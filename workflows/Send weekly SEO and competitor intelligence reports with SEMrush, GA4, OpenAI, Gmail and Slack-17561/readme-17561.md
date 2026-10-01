Send weekly SEO and competitor intelligence reports with SEMrush, GA4, OpenAI, Gmail and Slack

https://n8nworkflows.xyz/workflows/send-weekly-seo-and-competitor-intelligence-reports-with-semrush--ga4--openai--gmail-and-slack-17561


# Send weekly SEO and competitor intelligence reports with SEMrush, GA4, OpenAI, Gmail and Slack

### 1. Workflow Overview

This workflow automates the weekly generation and distribution of SEO and competitive intelligence reports. Running on a scheduled basis every Monday morning, it concurrently fetches search ranking data from SEMrush, performance metrics from Google Analytics 4 (GA4), and raw HTML from a competitor's webpage. It then processes this data, tracks changes against historical hashes, uses an OpenAI language model to synthesize insights, and dispatches the final summary to both a client via email and an internal team via Slack.

The architecture is divided into four main functional blocks:
- **1.1 Trigger & Data Intake:** Schedules the pipeline run and concurrently pulls external data from SEMrush, GA4, and a target competitor website.
- **1.2 Analysis & AI Summary:** Consolidates multiple data streams into a single dataset, calculates hash-based change detection for competitor tracking, and delegates narrative generation to an AI agent powered by OpenAI.
- **1.3 Report Delivery:** Distributes the AI-generated intelligence report across channels (Gmail and Slack).
- **1.4 Error Handling:** Catches any runtime exceptions across the pipeline and alerts operations teams via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Intake
**Overview:** This block establishes the operational cadence of the pipeline by kicking off execution every Monday at 7 AM and executing three parallel HTTP requests to gather raw analytics and web scraping data.

**Nodes Involved:**
- `Every Monday 7am`
- `Fetch SEMrush Rankings`
- `Fetch GA4 Traffic`
- `Scrape Competitor Pages`

**Node Details:**
- **Every Monday 7am**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Acts as the primary time-based entry point for the workflow.
  - *Configuration:* Configured to trigger weekly on Mondays at 07:00.
  - *Key Expressions:* None.
  - *Connections:* Outputs to `Fetch SEMrush Rankings`, `Fetch GA4 Traffic`, and `Scrape Competitor Pages`.
  - *Edge Cases:* Server timezone configurations can impact the exact delivery hour if not synchronized with UTC or local time expectations.

- **Fetch SEMrush Rankings**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Queries the SEMrush API for organic keyword position data.
  - *Configuration:* Targets `https://api.semrush.com/` using query parameters for domain organic data (`type=domain_organic`), a specified domain placeholder, and targeted export columns (`Ph,Po,Pp,Nq,Ur`). Enabled with 3 retries on failure and a 15-second timeout.
  - *Key Expressions:* Query parameter fields use literal placeholder strings (`REPLACE_WITH_DOMAIN`).
  - *Connections:* Input from `Every Monday 7am`; output to `Combine Intelligence` (Input Index 0).
  - *Edge Cases:* Authentication failures if API keys are missing/invalid; rate-limiting or quota exhaustion on SEMrush.

- **Fetch GA4 Traffic**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Retrieves Google Analytics 4 report metrics via the Google Analytics Data API.
  - *Configuration:* POST request sent to `https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_PROPERTY_ID:runReport`. Body includes JSON payload requesting rolling 7-day data (`7daysAgo` to `today`), broken down by landing page dimensions and metrics for sessions and organic search clicks. Enabled with 3 retries on failure and a 15-second timeout.
  - *Key Expressions:* JSON body payload dynamically defined using n8n expressions with a date range object.
  - *Connections:* Input from `Every Monday 7am`; output to `Combine Intelligence` (Input Index 1).
  - *Edge Cases:* Invalid GA4 Property ID in URL; OAuth credential expiration or insufficient permissions.

- **Scrape Competitor Pages**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Downloads raw HTML from a competitor website to monitor structural or text-based changes.
  - *Configuration:* GET request to a dynamic URL placeholder. Configured with a text response format, a 15-second timeout, 2 retry attempts, and `continueOnFail` set to true to ensure that scraping blockages do not halt the entire reporting pipeline.
  - *Key Expressions:* `=REPLACE_WITH_COMPETITOR_URL`
  - *Connections:* Input from `Every Monday 7am`; output unlinked directly, but downstream node queries its payload via expression lookup.
  - *Edge Cases:* Target websites blocking bot scrapers (HTTP 403/429) or altering DOM structures. Handled gracefully via `continueOnFail`.

---

#### 2.2 Analysis & AI Summary
**Overview:** This block merges disparate JSON payloads, computes cryptographic hashes of competitor web pages to identify differential updates since the previous run, and utilizes an AI agent to format insights into plain-language summaries.

**Nodes Involved:**
- `Combine Intelligence`
- `Compute Movers & Anomalies`
- `Summarize Movers (AI Agent)`
- `OpenAI Chat Model`

**Node Details:**
- **Combine Intelligence**
  - *Type & Role:* `n8n-nodes-base.merge` (Merge) — Combines multiple upstream data streams into a single cohesive data object.
  - *Configuration:* Set to `combine` mode using the `combineAll` strategy.
  - *Key Expressions:* None.
  - *Connections:* Inputs from `Fetch SEMrush Rankings` (Input 0) and `Fetch GA4 Traffic` (Input 1); output to `Compute Movers & Anomalies`.
  - *Edge Cases:* Misalignment of item structures if upstream inputs yield variable array lengths.

- **Compute Movers & Anomalies**
  - *Type & Role:* `n8n-nodes-base.code` (Code) — Executes custom JavaScript to parse data inputs, calculate an MD5 hash of the competitor's HTML, and compare it against historical state.
  - *Configuration:* Utilizes Node.js crypto libraries (`require('crypto')`) and n8n workflow static data storage (`$getWorkflowStaticData('global')`) to retain state between executions.
  - *Key Expressions:* Retrieves data from preceding nodes using explicit node references: `$('Fetch SEMrush Rankings').first()`, `$('Fetch GA4 Traffic').first()`, and `$('Scrape Competitor Pages').first()`.
  - *Connections:* Input from `Combine Intelligence`; output to `Summarize Movers (AI Agent)`.
  - *Edge Cases:* Empty or null HTML payloads returning empty hashes; state loss if the workflow instance resets.

- **Summarize Movers (AI Agent)**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Core intelligence engine responsible for processing structured metrics into human-readable narratives.
  - *Configuration:* Prompt type defined with a strict system message instructing the model to act as an SEO analyst, synthesizing ranking movements, traffic anomalies, and competitor changes into 4–5 sentences with a key action item.
  - *Key Expressions:* `=SEO data this week: {{ JSON.stringify($json) }}`
  - *Connections:* Input from `Compute Movers & Anomalies` and `OpenAI Chat Model` (AI Language Model connection); outputs to `Email Report` and `Post to Slack`.
  - *Edge Cases:* Token limit excesses or model hallucinations if JSON input payloads are malformed or excessively large.

- **OpenAI Chat Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Chat Model) — Sub-node providing the underlying language model infrastructure for the AI Agent.
  - *Configuration:* Uses the `gpt-5-mini` model with a temperature setting of `0.4` for balanced creativity and determinism.
  - *Key Expressions:* None.
  - *Connections:* Output connected to `Summarize Movers (AI Agent)` via the `ai_languageModel` anchor.
  - *Edge Cases:* Invalid OpenAI API keys, insufficient account credits, or API downtime.

---

#### 2.3 Report Delivery
**Overview:** Formats and distributes the finalized weekly intelligence report to external clients via email and internal team members via messaging platforms.

**Nodes Involved:**
- `Email Report`
- `Post to Slack`

**Node Details:**
- **Email Report**
  - *Type & Role:* `n8n-nodes-base.gmail` (Gmail) — Dispatches HTML-formatted email reports to stakeholders.
  - *Configuration:* Configured with recipient address placeholder, custom subject line incorporating dynamic current dates, and message body conversion logic. Includes retry-on-fail parameters.
  - *Key Expressions:* 
    - Subject: `=Weekly SEO & Competitor Intelligence — {{ $now.toFormat('dd LLL yyyy') }}`
    - Message: `={{ '<p>' + $json.output.replace(/\n/g, '<br>') + '</p>' }}`
  - *Connections:* Input from `Summarize Movers (AI Agent)`.
  - *Edge Cases:* OAuth token expiration for the Gmail integration; recipient server bouncing emails due to formatting or spam filtering.

- **Post to Slack**
  - *Type & Role:* `n8n-nodes-base.slack` (Slack) — Publishes structured report text into a specified workspace channel.
  - *Configuration:* Target selection set to channel mode using a channel ID lookup parameter (`REPLACE_WITH_CHANNEL_ID`).
  - *Key Expressions:* `=📈 Weekly SEO Report:\n{{ $json.output }}`
  - *Connections:* Input from `Summarize Movers (AI Agent)`.
  - *Edge Cases:* Missing bot scopes or incorrect Slack channel IDs resulting in message delivery failure.

---

#### 2.4 Error Handling
**Overview:** Captures unhandled exceptions across the entire workflow runtime to prevent silent failures and ensure operations teams are immediately notified.

**Nodes Involved:**
- `Error Trigger`
- `Notify Ops`

**Node Details:**
- **Error Trigger**
  - *Type & Role:* `n8n-nodes-base.errorTrigger` (Error Trigger) — Global exception catcher activated whenever any node in the workflow fails.
  - *Configuration:* Default execution configurations.
  - *Key Expressions:* None.
  - *Connections:* Output to `Notify Ops`.
  - *Edge Cases:* None.

- **Notify Ops**
  - *Type & Role:* `n8n-nodes-base.slack` (Slack) — Sends operational alerts containing error metadata to an internal monitoring channel.
  - *Configuration:* Posts formatted failure diagnostics to an ops channel ID.
  - *Key Expressions:* `=🚨 SEO & Competitor Intelligence Pipeline failed: {{ $json.execution.error.message }}`
  - *Connections:* Input from `Error Trigger`.
  - *Edge Cases:* Slack API downtime preventing delivery of the alert.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Visual documentation and setup instructions. | None | None | 📈 SEO & Competitor Intelligence Reporting Pipeline<br><br>Every Monday morning, this workflow pulls competitor and search performance data from SEMrush, Google Analytics 4, and a tracked competitor's webpage, then uses AI to summarize the biggest ranking movements, traffic anomalies, and competitor changes into a report delivered by email and Slack.<br><br>**Perfect for:** SEO teams, marketing agencies, or in-house growth teams who need a recurring competitive intelligence briefing without manually pulling reports from multiple tools.<br><br>---<br><br>## How it works<br><br>1. **Every Monday 7am** — Triggers the pipeline on a weekly schedule.<br>2. **Fetch SEMrush Rankings** — Pulls organic keyword rankings and positions for the tracked domain.<br>3. **Fetch GA4 Traffic** — Retrieves the last 7 days of landing page sessions and organic search clicks from Google Analytics 4.<br>4. **Scrape Competitor Pages** — Downloads the raw HTML of a tracked competitor page. *(continues on failure so one bad scrape doesn't stop the report)*<br>5. **Combine Intelligence** — Merges the SEMrush, GA4, and competitor scrape outputs into a single item.<br>6. **Compute Movers & Anomalies** — Hashes the competitor page HTML and compares it to last week's stored hash to detect whether the competitor's page changed.<br>7. **Summarize Movers (AI Agent)** — Turns the combined data into a 4-5 sentence plain-language summary, calling out the single most important action item.<br>8. **OpenAI Chat Model** — Supplies the language model (GPT-5 mini) that powers the Summarize Movers agent.<br>9. **Email Report** — Sends the AI-written summary as an HTML email to the client.<br>10. **Post to Slack** — Posts the same summary to a Slack channel for the internal team.<br>11. **Error Trigger** — Catches any failure in the pipeline. *(error-handling path)*<br>12. **Notify Ops** — Posts a failure alert with the error message to an ops Slack channel.<br><br>---<br><br>## Setup (~10 minutes)<br><br>1. **SEMrush API** — Add your API key in the *Fetch SEMrush Rankings* node and set the domain to track.<br>2. **Google Analytics 4** — Add your GA4 credential in the *Fetch GA4 Traffic* node and replace `YOUR_GA4_PROPERTY_ID` in the URL.<br>3. **Competitor URL** — Replace the placeholder URL in the *Scrape Competitor Pages* node with the page you want to monitor.<br>4. **OpenAI** — Add your OpenAI API key in the *OpenAI Chat Model* node.<br>5. **Gmail** — Connect your Gmail account in the *Email Report* node and set the client's email address.<br>6. **Slack** — Connect your Slack account and set the channel IDs in *Post to Slack* and *Notify Ops*.<br>> Competitor scraping may break if the target site blocks bots or changes its HTML structure — check the *Scrape Competitor Pages* node if change detection stops firing. |
| **Section: Trigger & Data Intake** | `n8n-nodes-base.stickyNote` | Visual grouping for intake nodes. | None | None | ## 1️⃣ Trigger & Data Intake<br><br>Every Monday at 7am the **Every Monday 7am** schedule trigger kicks off three parallel data pulls: **Fetch SEMrush Rankings** for keyword position data, **Fetch GA4 Traffic** for the past week's organic sessions and clicks, and **Scrape Competitor Pages** for the raw HTML of a tracked competitor page. |
| **Section: Analysis & AI Summary** | `n8n-nodes-base.stickyNote` | Visual grouping for analysis nodes. | None | None | ## 2️⃣ Analysis & AI Summary<br><br>**Combine Intelligence** merges the three data sources into one item, then **Compute Movers & Anomalies** hashes the competitor HTML to flag whether it changed since last week. The **Summarize Movers (AI Agent)**, powered by the **OpenAI Chat Model**, turns the combined data into a plain-language summary highlighting the biggest movers and the top action item. |
| **Section: Report Delivery** | `n8n-nodes-base.stickyNote` | Visual grouping for delivery nodes. | None | None | ## 3️⃣ Report Delivery<br><br>The AI-generated summary is delivered two ways: **Email Report** sends it as an HTML email to the client, and **Post to Slack** posts the same summary into an internal Slack channel. |
| **Section: Error Handling** | `n8n-nodes-base.stickyNote` | Visual grouping for error handlers. | None | None | ## 4️⃣ Error Handling<br><br>If any step in the pipeline fails, the **Error Trigger** fires and **Notify Ops** posts an alert with the error message to an internal ops Slack channel, so failures are caught immediately instead of silently skipping the week's report. |
| **Every Monday 7am** | `n8n-nodes-base.scheduleTrigger` | Triggers pipeline weekly. | None | Fetch SEMrush Rankings, Fetch GA4 Traffic, Scrape Competitor Pages | ## 1️⃣ Trigger & Data Intake<br><br>Every Monday at 7am the **Every Monday 7am** schedule trigger kicks off three parallel data pulls: **Fetch SEMrush Rankings** for keyword position data, **Fetch GA4 Traffic** for the past week's organic sessions and clicks, and **Scrape Competitor Pages** for the raw HTML of a tracked competitor page. |
| **Fetch SEMrush Rankings** | `n8n-nodes-base.httpRequest` | Pulls organic search rankings from SEMrush. | Every Monday 7am | Combine Intelligence | ## 1️⃣ Trigger & Data Intake<br><br>Every Monday at 7am the **Every Monday 7am** schedule trigger kicks off three parallel data pulls: **Fetch SEMrush Rankings** for keyword position data, **Fetch GA4 Traffic** for the past week's organic sessions and clicks, and **Scrape Competitor Pages** for the raw HTML of a tracked competitor page. |
| **Fetch GA4 Traffic** | `n8n-nodes-base.httpRequest` | Pulls traffic and landing page metrics from GA4. | Every Monday 7am | Combine Intelligence | ## 1️⃣ Trigger & Data Intake<br><br>Every Monday at 7am the **Every Monday 7am** schedule trigger kicks off three parallel data pulls: **Fetch SEMrush Rankings** for keyword position data, **Fetch GA4 Traffic** for the past week's organic sessions and clicks, and **Scrape Competitor Pages** for the raw HTML of a tracked competitor page. |
| **Scrape Competitor Pages** | `n8n-nodes-base.httpRequest` | Downloads competitor web page HTML. | Every Monday 7am | None (Accessed via expression) | ## 1️⃣ Trigger & Data Intake<br><br>Every Monday at 7am the **Every Monday 7am** schedule trigger kicks off three parallel data pulls: **Fetch SEMrush Rankings** for keyword position data, **Fetch GA4 Traffic** for the past week's organic sessions and clicks, and **Scrape Competitor Pages** for the raw HTML of a tracked competitor page. |
| **Combine Intelligence** | `n8n-nodes-base.merge` | Merges SEMrush and GA4 data streams. | Fetch SEMrush Rankings, Fetch GA4 Traffic | Compute Movers & Anomalies | ## 2️⃣ Analysis & AI Summary<br><br>**Combine Intelligence** merges the three data sources into one item, then **Compute Movers & Anomalies** hashes the competitor HTML to flag whether it changed since last week. The **Summarize Movers (AI Agent)**, powered by the **OpenAI Chat Model**, turns the combined data into a plain-language summary highlighting the biggest movers and the top action item. |
| **Compute Movers & Anomalies** | `n8n-nodes-base.code` | Hashes competitor HTML and compiles dataset. | Combine Intelligence | Summarize Movers (AI Agent) | ## 2️⃣ Analysis & AI Summary<br><br>**Combine Intelligence** merges the three data sources into one item, then **Compute Movers & Anomalies** hashes the competitor HTML to flag whether it changed since last week. The **Summarize Movers (AI Agent)**, powered by the **OpenAI Chat Model**, turns the combined data into a plain-language summary highlighting the biggest movers and the top action item. |
| **Summarize Movers (AI Agent)** | `@n8n/n8n-nodes-langchain.agent` | Generates narrative report using AI model. | Compute Movers & Anomalies, OpenAI Chat Model | Email Report, Post to Slack | ## 2️⃣ Analysis & AI Summary<br><br>**Combine Intelligence** merges the three data sources into one item, then **Compute Movers & Anomalies** hashes the competitor HTML to flag whether it changed since last week. The **Summarize Movers (AI Agent)**, powered by the **OpenAI Chat Model**, turns the combined data into a plain-language summary highlighting the biggest movers and the top action item. |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides language model backend (GPT-5 mini). | None | Summarize Movers (AI Agent) | ## 2️⃣ Analysis & AI Summary<br><br>**Combine Intelligence** merges the three data sources into one item, then **Compute Movers & Anomalies** hashes the competitor HTML to flag whether it changed since last week. The **Summarize Movers (AI Agent)**, powered by the **OpenAI Chat Model**, turns the combined data into a plain-language summary highlighting the biggest movers and the top action item. |
| **Email Report** | `n8n-nodes-base.gmail` | Sends formatted HTML email report. | Summarize Movers (AI Agent) | None | ## 3️⃣ Report Delivery<br><br>The AI-generated summary is delivered two ways: **Email Report** sends it as an HTML email to the client, and **Post to Slack** posts the same summary into an internal Slack channel. |
| **Post to Slack** | `n8n-nodes-base.slack` | Publishes summary to internal Slack channel. | Summarize Movers (AI Agent) | None | ## 3️⃣ Report Delivery<br><br>The AI-generated summary is delivered two ways: **Email Report** sends it as an HTML email to the client, and **Post to Slack** posts the same summary into an internal Slack channel. |
| **Error Trigger** | `n8n-nodes-base.errorTrigger` | Captures workflow failures. | None (Global) | Notify Ops | ## 4️⃣ Error Handling<br><br>If any step in the pipeline fails, the **Error Trigger** fires and **Notify Ops** posts an alert with the error message to an ops Slack channel, so failures are caught immediately instead of silently skipping the week's report. |
| **Notify Ops** | `n8n-nodes-base.slack` | Sends failure alert to ops Slack channel. | Error Trigger | None | ## 4️⃣ Error Handling<br><br>If any step in the pipeline fails, the **Error Trigger** fires and **Notify Ops** posts an alert with the error message to an ops Slack channel, so failures are caught immediately instead of silently skipping the week's report. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node named `Every Monday 7am`.
   - Set interval configuration to trigger every week on Day 1 (Monday) at Hour 7.

2. **Create Data Intake Nodes:**
   - **Fetch SEMrush Rankings:**
     - Add an **HTTP Request** node.
     - Set URL to `https://api.semrush.com/`.
     - Add query parameters: `type` = `domain_organic`, `domain` = `REPLACE_WITH_DOMAIN`, `export_columns` = `Ph,Po,Pp,Nq,Ur`.
     - Configure 3 retries and 15s timeout.
   - **Fetch GA4 Traffic:**
     - Add an **HTTP Request** node.
     - Set method to `POST` and URL to `https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_PROPERTY_ID:runReport`.
     - Set body type to `JSON` and supply the payload:
       ```json
       {
         "dateRanges": [{"startDate": "7daysAgo", "endDate": "today"}],
         "dimensions": [{"name": "landingPagePlusQueryString"}],
         "metrics": [{"name": "sessions"}, {"name": "organicGoogleSearchClicks"}]
       }
       ```
     - Configure 3 retries and 15s timeout. Connect Google Cloud OAuth2 credentials.
   - **Scrape Competitor Pages:**
     - Add an **HTTP Request** node.
     - Set method to `GET` and URL to `REPLACE_WITH_COMPETITOR_URL`.
     - Under options, set response format to `Text` and timeout to `15000`. Enable **Continue On Fail**.
     - Connect output from `Every Monday 7am` to all three intake nodes.

3. **Create Analysis & AI Nodes:**
   - **Combine Intelligence:**
     - Add a **Merge** node. Set mode to `Combine` and Combine By to `Combine All`.
     - Connect `Fetch SEMrush Rankings` to Input 0 and `Fetch GA4 Traffic` to Input 1.
   - **Compute Movers & Anomalies:**
     - Add a **Code** node. Paste the JavaScript code snippet to read upstream nodes, compute MD5 hashes via Node's `crypto` module, compare with workflow static data, and return combined JSON.
     - Connect input from `Combine Intelligence`.
   - **OpenAI Chat Model:**
     - Add an **OpenAI Chat Model** sub-node. Select model `gpt-5-mini`, set temperature to `0.4`, and supply your OpenAI API credential.
   - **Summarize Movers (AI Agent):**
     - Add an **AI Agent** node. Set prompt type to define text input: `=SEO data this week: {{ JSON.stringify($json) }}`.
     - Configure System Message: `You are an SEO analyst. Summarize the biggest ranking movers, traffic anomalies, and whether a competitor page changed, in 4-5 plain-language sentences suitable for a Monday morning report. Call out the single most important thing to act on.`
     - Connect `OpenAI Chat Model` to the agent's AI language model input anchor. Connect input from `Compute Movers & Anomalies`.

4. **Create Delivery Nodes:**
   - **Email Report:**
     - Add a **Gmail** node. Set resource to `Message` and action to `Send`.
     - Set recipient email `client@REPLACE_WITH_CLIENT_DOMAIN.com`.
     - Set Subject: `=Weekly SEO & Competitor Intelligence — {{ $now.toFormat('dd LLL yyyy') }}`
     - Set Message Body: `={{ '<p>' + $json.output.replace(/\n/g, '<br>') + '</p>' }}`
     - Connect Gmail OAuth2 credentials.
   - **Post to Slack:**
     - Add a **Slack** node. Set resource to `Message` and action to `Post`.
     - Select channel mode and configure channel ID (`REPLACE_WITH_CHANNEL_ID`).
     - Set Text: `=📈 Weekly SEO Report:\n{{ $json.output }}`
     - Connect Slack OAuth2/Bot credentials.
     - Connect both `Email Report` and `Post to Slack` to the output of `Summarize Movers (AI Agent)`.

5. **Create Error Handling Nodes:**
   - **Error Trigger:**
     - Add an **Error Trigger** node as a detached entry point.
   - **Notify Ops:**
     - Add a **Slack** node configured to post messages to the ops monitoring channel (`REPLACE_WITH_CHANNEL_ID`).
     - Set Text: `=🚨 SEO & Competitor Intelligence Pipeline failed: {{ $json.execution.error.message }}`
     - Connect `Error Trigger` output to `Notify Ops`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Domain Setup** | Ensure placeholders such as `REPLACE_WITH_DOMAIN`, `YOUR_GA4_PROPERTY_ID`, and `REPLACE_WITH_COMPETITOR_URL` are updated before activating the workflow. |
| **Bot Defenses** | Competitor HTML scraping may occasionally return blocking pages (e.g., Cloudflare challenges) if the target site strengthens bot detection. The workflow uses `continueOnFail` on the scraper to ensure reporting continuity. |