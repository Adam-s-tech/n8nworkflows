Send AI-written multi-channel ad performance PDFs with GA4, Google Ads, Meta and GoHighLevel

https://n8nworkflows.xyz/workflows/send-ai-written-multi-channel-ad-performance-pdfs-with-ga4--google-ads--meta-and-gohighlevel-20271


# Send AI-written multi-channel ad performance PDFs with GA4, Google Ads, Meta and GoHighLevel

### 1. Workflow Overview

This workflow automates the generation and delivery of multi-channel client performance reports on a monthly schedule. It extracts performance metrics from Google Analytics 4 (GA4), Google Ads, and Meta Ads, processes the data through OpenAI to generate executive summaries, converts custom HTML reports into PDFs via PDFShift, uploads them to Google Drive, and sends the deliverables directly to clients via GoHighLevel (LeadConnector).

The workflow execution is divided into the following logical blocks:
- **1.1 Schedule & Client Configuration:** Triggers automatically on the first day of every month, loads agency configuration settings, and splits the client roster into individual processing items.
- **1.2 Multi-Channel Data Fetching:** Sequentially queries GA4, Google Ads, and Meta Ads APIs to gather period-over-period performance data for each client.
- **1.3 KPI Aggregation & AI Processing:** Normalizes and blends platform metrics into a unified data model, then leverages OpenAI (`gpt-4o-mini`) to generate structured executive commentary, wins, and recommendations.
- **1.4 Report Generation & Cloud Storage:** Converts the synthesized data and summaries into a branded HTML document, transforms it into an A4 PDF using PDFShift, uploads it to Google Drive, and generates a public shareable link.
- **1.5 CRM Integration & Email Delivery:** Upserts the client contact within GoHighLevel and dispatches an automated email containing both the inline summary, the direct Google Drive link, and the PDF attached.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule & Client Configuration
- **Overview:** Initializes the automated pipeline on a monthly schedule, defines global agency parameters, and unpacks the client roster for downstream iteration.
- **Nodes Involved:** `Monthly – 1st, 9 AM IST1`, `Config1`, `Split Clients1`
- **Node Details:**
  - **Monthly – 1st, 9 AM IST1**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration:* Cron expression set to fire on the 1st of every month at 09:00 AM (`0 9 1 * *`).
    - *Input/Output:* Output connected to `Config1`.
    - *Edge Cases:* Ensure server timezone settings align with expected execution times.
  - **Config1**
    - *Type & Role:* `n8n-nodes-base.set` (Parameter Initialization)
    - *Configuration:* Stores global agency metadata (`agency_name`, `brand_color`, `currency_symbol`), API versions, developer tokens, GHL routing parameters, and a JSON array of `clients`.
    - *Input/Output:* Input from `Monthly – 1st, 9 AM IST1`; output to `Split Clients1`.
    - *Edge Cases:* JSON syntax errors in the `clients` array will halt execution in the subsequent node.
  - **Split Clients1**
    - *Type & Role:* `n8n-nodes-base.code` (Data Transformation & Looping)
    - *Configuration:* Parses the client roster JSON string, calculates dynamic reporting date windows (current vs. previous month) using the `Asia/Kolkata` timezone, and outputs each client as a distinct item.
    - *Input/Output:* Input from `Config1`; output to `GA4 Run Reports1`.
    - *Edge Cases:* Throws a custom error if the client roster is malformed JSON.

#### Block 1.2: Multi-Channel Data Fetching
- **Overview:** Executes sequential HTTP API requests to gather web analytics and paid advertising performance metrics across three major platforms.
- **Nodes Involved:** `GA4 Run Reports1`, `Google Ads Campaign Report1`, `Meta Campaign Insights1`
- **Node Details:**
  - **GA4 Run Reports1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Sends a `POST` request to the Google Analytics Data API (`batchRunReports`) for the target property ID, requesting metrics for sessions, users, conversions, and channel groupings across current and previous date ranges. Error handling set to `continueRegularOutput`.
    - *Key Expressions:* Uses `{{ $json.ga4_property_id }}` and dynamic start/end dates from `Split Clients1`.
    - *Input/Output:* Input from `Split Clients1`; output to `Google Ads Campaign Report1`.
    - *Credentials:* `googleAnalyticsOAuth2`
    - *Edge Cases:* Authentication failures or invalid GA4 property IDs return structured error objects handled gracefully downstream.
  - **Google Ads Campaign Report1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Queries the Google Ads API (`googleAds:search`) using GAQL to retrieve campaign-level metrics (spend, impressions, clicks, conversions) for the specified customer ID. Error handling set to `continueRegularOutput`.
    - *Key Expressions:* Header parameters include developer tokens and login customer IDs.
    - *Input/Output:* Input from `GA4 Run Reports1`; output to `Meta Campaign Insights1`.
    - *Credentials:* `googleAdsOAuth2Api`
    - *Edge Cases:* Missing developer tokens or incorrect MCC routing headers cause HTTP 401/403 responses.
  - **Meta Campaign Insights1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* Fetches campaign insights from the Meta Graph API using GET parameters for fields, levels, and JSON-encoded time ranges. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `Google Ads Campaign Report1`; output to `Aggregate KPIs1`.
    - *Credentials:* Generic HTTP Header Auth (Meta Access Token)
    - *Edge Cases:* Expired access tokens or inactive ad account IDs.

#### Block 1.3: KPI Aggregation & AI Processing
- **Overview:** Consolidates multi-platform metrics into a unified data structure, computes period-over-period percentage changes, and prompts OpenAI to synthesize professional business insights.
- **Nodes Involved:** `Aggregate KPIs1`, `AI Executive Summary1`
- **Node Details:**
  - **Aggregate KPIs1**
    - *Type & Role:* `n8n-nodes-base.code` (Data Normalization)
    - *Configuration:* Executes once per item to parse raw responses from GA4, Google Ads, and Meta, calculate derived metrics (CTR, CPC, CPA, ROAS, period deltas), and output a single blended performance model.
    - *Input/Output:* Input from `Meta Campaign Insights1`; output to `AI Executive Summary1`.
    - *Edge Cases:* Handles missing or disconnected ad accounts without breaking the pipeline.
  - **AI Executive Summary1**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Generation)
    - *Configuration:* Uses model `gpt-4o-mini` with a temperature of `0.4`. System instructions enforce strict adherence to provided numbers, Indian English formatting, and JSON-only output constraints containing headlines, executive summaries, wins, concerns, and recommendations.
    - *Input/Output:* Input from `Aggregate KPIs1`; output to `Build HTML Report1`.
    - *Credentials:* OpenAI API
    - *Edge Cases:* Rate limits or token exhaustion; JSON parsing errors handled in subsequent nodes.

#### Block 1.4: Report Generation & Cloud Storage
- **Overview:** Transforms structured KPIs and AI narratives into a localized HTML document, converts it to PDF, uploads it to Google Drive, and establishes public read access.
- **Nodes Involved:** `Build HTML Report1`, `HTML → PDF (PDFShift1)`, `Upload PDF to Drive1`, `Share PDF (link)1`
- **Node Details:**
  - **Build HTML Report1**
    - *Type & Role:* `n8n-nodes-base.code` (HTML/CSS Templating)
    - *Configuration:* Generates a responsive, print-optimized A4 HTML string featuring agency branding, metric cards, data tables, and AI takeaways, alongside a matching email body template.
    - *Input/Output:* Input from `AI Executive Summary1`; output to `HTML → PDF (PDFShift)1`.
  - **HTML → PDF (PDFShift)1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (PDF Conversion API)
    - *Configuration:* Sends a POST request to the PDFShift API with the raw HTML string, requesting A4 sizing and print styling. Configured to return binary file output.
    - *Input/Output:* Input from `Build HTML Report1`; output to `Upload PDF to Drive1`.
    - *Credentials:* Generic HTTP Header Auth (PDFShift API Key)
  - **Upload PDF to Drive1**
    - *Type & Role:* `n8n-nodes-base.googleDrive` (Cloud Storage)
    - *Configuration:* Uploads the binary PDF payload to a designated Google Drive folder using a dynamically generated filename.
    - *Input/Output:* Input from `HTML → PDF (PDFShift)1`; output to `Share PDF (link)1`.
    - *Credentials:* `googleOAuth2`
  - **Share PDF (link)1**
    - *Type & Role:* `n8n-nodes-base.googleDrive` (Cloud Permissions)
    - *Configuration:* Updates file sharing permissions to grant read access to anyone with the link.
    - *Input/Output:* Input from `Upload PDF to Drive1`; output to `Prepare GHL Email1`.
    - *Credentials:* `googleOAuth2`

#### Block 1.5: CRM Integration & Email Delivery
- **Overview:** Prepares communication payloads, upserts client records in GoHighLevel, and dispatches performance reports via email with attachments.
- **Nodes Involved:** `Prepare GHL Email1`, `GHL Upsert Client Contact1`, `GHL Send Report Email1`
- **Node Details:**
  - **Prepare GHL Email1**
    - *Type & Role:* `n8n-nodes-base.code` (Payload Formatting)
    - *Configuration:* Combines Drive file IDs, view URLs, contact details, and HTML email templates into structured JSON payloads for GoHighLevel API consumption.
    - *Input/Output:* Input from `Share PDF (link)1`; output to `GHL Upsert Client Contact1`.
  - **GHL Upsert Client Contact1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (CRM Integration)
    - *Configuration:* Sends a POST request to LeadConnector API (`/contacts/upsert`) to create or update the client contact record, appending the tag `client-report-recipient`.
    - *Input/Output:* Input from `Prepare GHL Email1`; output to `GHL Send Report Email1`.
    - *Credentials:* Generic HTTP Header Auth (GHL Private Integration Token)
  - **GHL Send Report Email1**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (Email Dispatch)
    - *Configuration:* Sends an outbound email via LeadConnector API (`/conversations/messages`) containing the generated HTML body, subject line, and the Google Drive PDF download link as an attachment. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `GHL Upsert Client Contact1`; output terminates the branch.
    - *Credentials:* Generic HTTP Header Auth (GHL Private Integration Token)

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `📌 Overview – How This Workflow Works` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📊 Multi-Channel Agency Client Report – GA4 + Google Ads + Meta + AI + PDF + GHL<br><br>### How it works<br>On the 1st of every month at 9 AM, this workflow generates and delivers a full performance report for every client in your agency roster. The `Config` node holds the client list with their GA4 property IDs, Google Ads customer IDs, Meta ad account IDs, and GHL contact emails. `Split Clients` loops through each one. For each client, three parallel data fetches run: GA4 (sessions, conversions, revenue), Google Ads (spend, clicks, ROAS), and Meta (spend, reach, purchases). The raw numbers are aggregated into a single KPI object, then GPT-4o-mini writes an executive summary with month-over-month commentary. A code node builds a branded HTML report, PDFShift converts it to a PDF, the file is uploaded to Google Drive and a shareable link is generated. GHL upserts the contact and sends the report link by email.<br><br>### Setup steps<br>1. In `Config`, replace the `clients` array with your real clients — GA4 property ID, Google Ads customer ID, Meta ad account, email.<br>2. Add **Google OAuth2** to `GA4 Run Reports`.<br>3. Add your **Google Ads developer token + login customer ID** to `Google Ads Campaign Report`.<br>4. Add your **Meta access token** to `Meta Campaign Insights`.<br>5. Connect **OpenAI API** to `AI Executive Summary`.<br>6. Add your **PDFShift API key** to `HTML → PDF (PDFShift)`.<br>7. Connect **Google Drive OAuth2** to both Drive nodes.<br>8. Add your **GHL Private Integration Token** to both GHL nodes.<br>9. Activate the monthly schedule. |
| `Section – Schedule & Client Config` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## ⚙️ Schedule & Client Config<br>Fires on the 1st of every month. The `Config` node holds the full client roster as a JSON array — GA4, Google Ads, and Meta IDs per client. `Split Clients` calculates the reporting window (current month vs prior month) and loops through each client independently. |
| `Section – Multi-Channel Data Fetch` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📈 Multi-Channel Data Fetch<br>Three sequential API calls pull performance data for the current and prior month: GA4 (sessions, users, conversions, revenue), Google Ads (impressions, clicks, spend, conversions), and Meta (reach, impressions, spend, purchases). All three aggregate into a single KPI object. |
| `Section – AI Summary & PDF Report` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `Section – GHL Email Delivery` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📧 GHL Email Delivery<br>A code node formats the email body with the Drive PDF link. GHL upserts the client contact (creates if new, updates if existing), then sends the report email from your GHL sub-account. The client receives their branded monthly report directly in their inbox. |
| `🔑 Credentials & Configuration` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🔑 Credentials Required<br>- **Google OAuth2** — GA4 data + Drive upload<br>- **Google Ads** — developer token + login customer ID in headers<br>- **Meta access token** — campaign insights HTTP header<br>- **OpenAI API** — executive summary<br>- **PDFShift API key** — HTML to PDF conversion<br>- **GHL Private Integration Token** — contact upsert + email send |
| `Monthly – 1st, 9 AM IST1` | `n8n-nodes-base.scheduleTrigger` | Trigger Execution | None | `Config1` | |
| `Config1` | `n8n-nodes-base.set` | Set Global Agency Parameters | `Monthly – 1st, 9 AM IST1` | `Split Clients1` | ## ⚙️ Schedule & Client Config<br>Fires on the 1st of every month. The `Config` node holds the full client roster as a JSON array — GA4, Google Ads, and Meta IDs per client. `Split Clients` calculates the reporting window (current month vs prior month) and loops through each client independently. |
| `Split Clients1` | `n8n-nodes-base.code` | Date Window Calculation & Iteration | `Config1` | `GA4 Run Reports1` | ## ⚙️ Schedule & Client Config<br>Fires on the 1st of every month. The `Config` node holds the full client roster as a JSON array — GA4, Google Ads, and Meta IDs per client. `Split Clients` calculates the reporting window (current month vs prior month) and loops through each client independently. |
| `GA4 Run Reports1` | `n8n-nodes-base.httpRequest` | Fetch GA4 Analytics Data | `Split Clients1` | `Google Ads Campaign Report1` | ## 📈 Multi-Channel Data Fetch<br>Three sequential API calls pull performance data for the current and prior month: GA4 (sessions, users, conversions, revenue), Google Ads (impressions, clicks, spend, conversions), and Meta (reach, impressions, spend, purchases). All three aggregate into a single KPI object. |
| `Google Ads Campaign Report1` | `n8n-nodes-base.httpRequest` | Fetch Google Ads Campaign Data | `GA4 Run Reports1` | `Meta Campaign Insights1` | ## 📈 Multi-Channel Data Fetch<br>Three sequential API calls pull performance data for the current and prior month: GA4 (sessions, users, conversions, revenue), Google Ads (impressions, clicks, spend, conversions), and Meta (reach, impressions, spend, purchases). All three aggregate into a single KPI object. |
| `Meta Campaign Insights1` | `n8n-nodes-base.httpRequest` | Fetch Meta Ads Insights | `Google Ads Campaign Report1` | `Aggregate KPIs1` | ## 📈 Multi-Channel Data Fetch<br>Three sequential API calls pull performance data for the current and prior month: GA4 (sessions, users, conversions, revenue), Google Ads (impressions, clicks, spend, conversions), and Meta (reach, impressions, spend, purchases). All three aggregate into a single KPI object. |
| `Aggregate KPIs1` | `n8n-nodes-base.code` | Aggregate Metrics & Calculate Deltas | `Meta Campaign Insights1` | `AI Executive Summary1` | |
| `AI Executive Summary1` | `@n8n/n8n-nodes-langchain.openAi` | Generate AI Executive Summary | `Aggregate KPIs1` | `Build HTML Report1` | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `Build HTML Report1` | `n8n-nodes-base.code` | Generate Branded HTML & Email Body | `AI Executive Summary1` | `HTML → PDF (PDFShift)1` | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `HTML → PDF (PDFShift)1` | `n8n-nodes-base.httpRequest` | Convert HTML to PDF Binary | `Build HTML Report1` | `Upload PDF to Drive1` | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `Upload PDF to Drive1` | `n8n-nodes-base.googleDrive` | Upload PDF to Google Drive | `HTML → PDF (PDFShift)1` | `Share PDF (link)1` | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `Share PDF (link)1` | `n8n-nodes-base.googleDrive` | Make Drive File Publicly Readable | `Upload PDF to Drive1` | `Prepare GHL Email1` | ## 🤖 AI Executive Summary & PDF Report<br>GPT-4o-mini receives the aggregated KPIs and writes a plain-English executive summary with month-over-month commentary. A code node builds a branded HTML report from the KPIs + summary. PDFShift converts it to a PDF. The file is uploaded to Google Drive and a shareable link is returned. |
| `Prepare GHL Email1` | `n8n-nodes-base.code` | Format CRM Contact & Email Payloads | `Share PDF (link)1` | `GHL Upsert Client Contact1` | ## 📧 GHL Email Delivery<br>A code node formats the email body with the Drive PDF link. GHL upserts the client contact (creates if new, updates if existing), then sends the report email from your GHL sub-account. The client receives their branded monthly report directly in their inbox. |
| `GHL Upsert Client Contact1` | `n8n-nodes-base.httpRequest` | Upsert Contact in GoHighLevel | `Prepare GHL Email1` | `GHL Send Report Email1` | ## 📧 GHL Email Delivery<br>A code node formats the email body with the Drive PDF link. GHL upserts the client contact (creates if new, updates if existing), then sends the report email from your GHL sub-account. The client receives their branded monthly report directly in their inbox. |
| `GHL Send Report Email1` | `n8n-nodes-base.httpRequest` | Send Report Email via LeadConnector | `GHL Upsert Client Contact1` | None | ## 📧 GHL Email Delivery<br>A code node formats the email body with the Drive PDF link. GHL upserts the client contact (creates if new, updates if existing), then sends the report email from your GHL sub-account. The client receives their branded monthly report directly in their inbox. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Trigger Node:** Create a **Schedule Trigger** node (`Monthly – 1st, 9 AM IST1`). Set the interval rule to Cron Expression `0 9 1 * *`.
2. **Configuration Node:** Create an **Edit Fields (Set)** node (`Config1`). Connect it to the trigger. Add string assignments for agency metadata (`agency_name`, `brand_color`, `currency_symbol`), API configuration strings (`google_ads_api_version`, `google_ads_developer_token`, `google_ads_login_customer_id`, `meta_graph_version`, `meta_result_action_types`), GHL settings (`ghl_location_id`, `ghl_email_from`, `drive_folder_id`), and a JSON array representing `clients`.
3. **Client Splitting Node:** Create a **Code** node (`Split Clients1`). Connect it to `Config1`. Paste JavaScript logic to parse the `clients` JSON, compute dynamic current/previous reporting date intervals based on the `Asia/Kolkata` timezone, and output individual client items.
4. **GA4 Data Node:** Create an **HTTP Request** node (`GA4 Run Reports1`). Connect to `Split Clients1`. Configure a `POST` request to `https://analyticsdata.googleapis.com/v1beta/properties/{{ $json.ga4_property_id }}:batchRunReports`. Set authentication to **Google Analytics OAuth2**. Enable `On Error -> Continue Regular Output`.
5. **Google Ads Node:** Create an **HTTP Request** node (`Google Ads Campaign Report1`). Connect to `GA4 Run Reports1`. Configure a `POST` request to `https://googleads.googleapis.com/{{ $('Split Clients1').item.json.google_ads_api_version }}/customers/{{ $('Split Clients1').item.json.google_ads_customer_id }}/googleAds:search`. Add header parameters for `developer-token` and `login-customer-id`. Set authentication to **Google Ads OAuth2 API**. Enable `On Error -> Continue Regular Output`.
6. **Meta Ads Node:** Create an **HTTP Request** node (`Meta Campaign Insights1`). Connect to `Google Ads Campaign Report1`. Configure a `GET` request to `https://graph.facebook.com/{{ $('Split Clients1').item.json.meta_graph_version }}/{{ $('Split Clients1').item.json.meta_ad_account_id }}/insights`. Add query parameters for fields, levels, and JSON time ranges. Set authentication to **Generic Credential Type (HTTP Header Auth)** using your Meta access token. Enable `On Error -> Continue Regular Output`.
7. **KPI Aggregation Node:** Create a **Code** node (`Aggregate KPIs1`). Connect to `Meta Campaign Insights1`. Paste normalization code to process GA4, Google Ads, and Meta responses into a single blended KPI object with period-over-period deltas.
8. **AI Summary Node:** Create an **OpenAI** node (`AI Executive Summary1`). Connect to `Aggregate KPIs1`. Select model `gpt-4o-mini`, set temperature to `0.4`, configure system instructions enforcing strict JSON output, and pass the aggregated KPI JSON into the user prompt. Enable JSON output mode.
9. **HTML Generation Node:** Create a **Code** node (`Build HTML Report1`). Connect to `AI Executive Summary1`. Implement styling templates that build a branded responsive A4 HTML report string and email markup using the AI responses and metrics.
10. **PDF Conversion Node:** Create an **HTTP Request** node (`HTML → PDF (PDFShift1)`). Connect to `Build HTML Report1`. Configure a `POST` request to `https://api.pdfshift.io/v3/convert/pdf`. Set body payload to JSON containing source HTML, format `A4`, and print settings. Configure response format to `File`. Set authentication to **Generic HTTP Header Auth** with your PDFShift API key.
11. **Drive Upload Node:** Create a **Google Drive** node (`Upload PDF to Drive1`). Connect to `HTML → PDF (PDFShift)1`. Select operation `Upload`, set binary file input, specify target folder ID dynamically from configuration, and connect **Google Drive OAuth2** credentials.
12. **Drive Share Node:** Create a **Google Drive** node (`Share PDF (link)1`). Connect to `Upload PDF to Drive1`. Select operation `Share`, set file ID from the upload node, and configure permissions with role `reader` and type `anyone`.
13. **GHL Email Preparation Node:** Create a **Code** node (`Prepare GHL Email1`). Connect to `Share PDF (link)1`. Format contact details, direct download URLs, view URLs, and email HTML content into structured GHL payloads.
14. **GHL Contact Upsert Node:** Create an **HTTP Request** node (`GHL Upsert Client Contact1`). Connect to `Prepare GHL Email1`. Configure a `POST` request to `https://services.leadconnectorhq.com/contacts/upsert` with JSON body payloads containing location ID, email, names, and tags. Set headers for API version and JSON acceptance. Authenticate via **Generic HTTP Header Auth** using your GHL Private Integration Token.
15. **GHL Email Dispatch Node:** Create an **HTTP Request** node (`GHL Send Report Email1`). Connect to `GHL Upsert Client Contact1`. Configure a `POST` request to `https://services.leadconnectorhq.com/conversations/messages` passing message type `Email`, contact ID references, subject, HTML content, sender email, and PDF download attachments. Authenticate via **Generic HTTP Header Auth** using your GHL Private Integration Token. Enable `On Error -> Continue Regular Output`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Platform-reported conversion overlaps | Platform-reported conversions (Google Ads vs Meta Ads) may overlap across channels; consider attribution modeling adjustments. |
| Timezone consistency | Execution timestamps rely on server-side zone enforcement (`Asia/Kolkata` configured in client splitter). |