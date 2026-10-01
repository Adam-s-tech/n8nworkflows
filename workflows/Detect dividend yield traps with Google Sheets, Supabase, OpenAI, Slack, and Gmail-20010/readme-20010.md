Detect dividend yield traps with Google Sheets, Supabase, OpenAI, Slack, and Gmail

https://n8nworkflows.xyz/workflows/detect-dividend-yield-traps-with-google-sheets--supabase--openai--slack--and-gmail-20010


# Detect dividend yield traps with Google Sheets, Supabase, OpenAI, Slack, and Gmail

### 1. Workflow Overview

The **Dividend Yield Trap Detector** workflow automates the weekly screening of a stock watchlist to identify high-risk dividend stocks ("yield traps"). It runs on a scheduled trigger, queries external APIs for dividend calendar events and financial fundamentals, evaluates risk metrics, leverages an AI model for qualitative insights on high-risk candidates, persists records to storage layers, and dispatches comprehensive digests via Slack and Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule & Watchlist Intake:** Initiates the weekly scan and reads/filters target stocks from Google Sheets.
- **1.2 Dividend Data Collection & Preparation:** Fetches upcoming ex-dividend dates and financial fundamentals via HTTP requests.
- **1.3 Data Normalization & Sustainability Calculation:** Merges datasets and calculates mathematical dividend sustainability and risk scores.
- **1.4 Risk Evaluation & AI Analysis:** Branches workflow execution based on a risk score threshold ($>=60$). Qualifying high-risk candidates undergo AI-powered analysis to generate investor guidance notes.
- **1.5 Storage & Notification (High-Risk Path):** Persists flagged records to Google Sheets and Supabase, builds a ranked weekly risk digest, and distributes it to Slack and Gmail.
- **1.6 No-Risk Handling (Alternative Path):** Generates and delivers a notification digest confirming zero qualifying risk candidates when no stocks exceed the threshold.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule & Watchlist Intake
- **Overview:** Triggers the weekly automation cycle, reads the core watchlist from Google Sheets, and filters out unselected or invalid rows.
- **Nodes Involved:**
  - `Weekly Dividend Trap Scan1`
  - `Read Dividend Watchlist`
  - `Prepare Watchlist Context`

- **Node Details:**
  - **Weekly Dividend Trap Scan1**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Configured to trigger execution weekly at hour 9.
    - *Input / Output:* None (Entry point) / Outputs trigger event to `Read Dividend Watchlist`.
    - *Failure Handling:* None.
  - **Read Dividend Watchlist**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Reader)
    - *Configuration:* Reads rows from Document ID `1FCGEXmbdS58yqfgqjhJCUn8IC0nZqDUDBi4l7FMWRmQ`, Sheet Name "Watchlist" (`gid=0`).
    - *Input / Output:* Connected from `Weekly Dividend Trap Scan1` / Outputs raw sheet rows to `Prepare Watchlist Context`.
    - *Edge Cases:* Authentication failure, missing sheet, or unmapped columns.
  - **Prepare Watchlist Context**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformer)
    - *Configuration:* JavaScript filter processing rows. Validates that `enabled` evaluates truthy (`true`, `'true'`, `'yes'`, `'1'`) and `symbol` is present. Extracts metadata (`company_name`, `sector`, `notes`).
    - *Key Expressions:* Accesses `$input.all()`. Throws explicit error if zero enabled symbols are found.
    - *Input / Output:* Connected from `Read Dividend Watchlist` / Outputs object containing `watchlist` array and `symbols` array to `Get Upcoming Ex-Dividend Data`.

---

#### Block 1.2: Dividend Data Collection & Preparation
- **Overview:** Queries an external calendar endpoint for upcoming ex-dividend dates, filters candidates falling within a 7-day window matching the watchlist, and fetches corresponding financial fundamentals.
- **Nodes Involved:**
  - `Get Upcoming Ex-Dividend Data`
  - `Prepare Fundamentals Request`
  - `Get Dividend Fundamentals`
  - `Normalize Fundamentals`

- **Node Details:**
  - **Get Upcoming Ex-Dividend Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Caller)
    - *Configuration:* GET request to `http://192.168.101.63:5678/webhook-test/mock/dividend-calendar` returning JSON.
    - *Input / Output:* Connected from `Prepare Watchlist Context` / Outputs calendar dataset to `Prepare Fundamentals Request`.
    - *Edge Cases:* Network timeouts, target endpoint downtime, or HTTP 4xx/5xx errors.
  - **Prepare Fundamentals Request**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Filter & Transformer)
    - *Configuration:* JavaScript filtering logic. Sets a date window from today ($T$) to $T + 7$ days. Retains calendar items whose symbols match enabled watchlist items and whose `ex_dividend_date` falls inside the window.
    - *Key Expressions:* Uses `$('Prepare Watchlist Context').first().json`.
    - *Input / Output:* Connected from `Get Upcoming Ex-Dividend Data` / Outputs filtered `dividend_candidates` and `symbols` array to `Get Dividend Fundamentals`.
  - **Get Dividend Fundamentals**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Caller)
    - *Configuration:* GET request to `http://192.168.101.63:5678/webhook-test/mock/dividend-fundamentals` returning JSON.
    - *Input / Output:* Connected from `Prepare Fundamentals Request` / Outputs raw fundamental indicators to `Normalize Fundamentals`.
    - *Edge Cases:* Missing stock symbols in fundamentals response payload.
  - **Normalize Fundamentals**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Formatter)
    - *Configuration:* Standardizes the incoming fundamentals array payload.
    - *Input / Output:* Connected from `Get Dividend Fundamentals` / Outputs `fundamentals` array to `Merge Calendar and Fundamentals`.

---

#### Block 1.3: Data Normalization & Sustainability Calculation
- **Overview:** Combines calendar candidates with financial fundamentals, computes a dividend sustainability score, calculates yield-trap risk scores and levels, and compiles specific warning red flags.
- **Nodes Involved:**
  - `Merge Calendar and Fundamentals`
  - `Calculate Dividend Sustainability`
  - `Calculate Yield Trap Risk`
  - `Identify Yield Trap Red Flags`

- **Node Details:**
  - **Merge Calendar and Fundamentals**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Merger & Cleaner)
    - *Configuration:* Maps fundamentals by symbol and merges them with matching dividend calendar items. Filters out records lacking a `share_price`.
    - *Key Expressions:* Uses `$('Prepare Fundamentals Request').first().json.dividend_candidates`.
    - *Input / Output:* Connected from `Normalize Fundamentals` / Outputs merged candidate records to `Calculate Dividend Sustainability`.
  - **Calculate Dividend Sustainability**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Scoring Engine)
    - *Configuration:* Deducts points from a base score of 100 based on payout ratios ($>60\%$, $>75\%$, $>90\%$), negative earnings/revenue growths, negative dividend growths, and historical dividend cuts over the last 5 years. Clamps scores between 0 and 100.
    - *Input / Output:* Connected from `Merge Calendar and Fundamentals` / Outputs records with added `sustainability_score` to `Calculate Yield Trap Risk`.
  - **Calculate Yield Trap Risk**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Risk Evaluator)
    - *Configuration:* Computes `risk_score` inverse to sustainability plus penalties for high yields ($>=6\%$, $>=8\%$), extreme payout ratios, negative growth, and cut history. Categorizes `risk_level` into Low, Medium, or High ($\ge 75$).
    - *Input / Output:* Connected from `Calculate Dividend Sustainability` / Outputs items containing `risk_score` and `risk_level` to `Identify Yield Trap Red Flags`.
  - **Identify Yield Trap Red Flags**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Condition Matcher)
    - *Configuration:* Evaluates financial metrics against thresholds and compiles an array of descriptive warning flags (`red_flags`) along with a `red_flag_count`.
    - *Input / Output:* Connected from `Calculate Yield Trap Risk` / Outputs enriched records to `Risk Above Threshold`.

---

#### Block 1.4: Risk Evaluation & AI Analysis
- **Overview:** Evaluates whether stock risk meets or exceeds the threshold of 60. Qualifying stocks are processed by an AI model to generate qualitative explanations, which are then parsed and normalized.
- **Nodes Involved:**
  - `Risk Above Threshold`
  - `Generate AI Yield Trap Explanation`
  - `Normalize AI Response`

- **Node Details:**
  - **Risk Above Threshold**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Branching Logic)
    - *Configuration:* Evaluates condition `{{ $json.risk_score }} >= 60`.
    - *Input / Output:* Connected from `Identify Yield Trap Red Flags` / Output index `0` (True) routes to `Generate AI Yield Trap Explanation`; Output index `1` (False) routes to `Build No-Risk Weekly Digest`.
  - **Generate AI Yield Trap Explanation**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (Generative AI)
    - *Configuration:* Uses model `gpt-4.1-nano` (credential `OpenAI (PAID) (AI-ML Team)`). Prompts the model to analyze metrics and return strict JSON containing `caution_note`, `key_concern`, and `investor_question`.
    - *Input / Output:* Connected from `Risk Above Threshold` (True branch) / Outputs model generation text to `Normalize AI Response`.
    - *Edge Cases:* API rate limits, schema validation failures, or token limits.
  - **Normalize AI Response**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Parser)
    - *Configuration:* Extracts text content from OpenAI response structure and parses JSON safely. Falls back to default caution text if parsing fails.
    - *Input / Output:* Connected from `Generate AI Yield Trap Explanation` / Outputs normalized AI fields to `Prepare Dividend Risk Record`.

---

#### Block 1.5: Storage & Notification (High-Risk Path)
- **Overview:** Prepares unified risk records, logs them to Google Sheets and Supabase, builds a ranked weekly digest, and sends notifications via Slack and Gmail.
- **Nodes Involved:**
  - `Prepare Dividend Risk Record`
  - `Log Flagged Risks to Google Sheets`
  - `Store Flagged Risks in Supabase`
  - `Build Weekly Risk Digest`
  - `Send Weekly Digest to Slack`
  - `Send Weekly Digest by Gmail`

- **Node Details:**
  - **Prepare Dividend Risk Record**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Consolidator)
    - *Configuration:* Combines AI outputs with original financial metrics using array index correlation against `Identify Yield Trap Red Flags`.
    - *Input / Output:* Connected from `Normalize AI Response` / Fans out to `Log Flagged Risks to Google Sheets` and `Store Flagged Risks in Supabase`.
  - **Log Flagged Risks to Google Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Writer)
    - *Configuration:* Appends rows to Spreadsheet ID `1FCGEXmbdS58yqfgqjhJCUn8IC0nZqDUDBi4l7FMWRmQ`, Sheet Name "Risk Results" (`gid=1650153831`). Maps fields: `symbol`, `dividend_yield`, `payout_ratio`, `risk_score`, `risk_level`, `risk_flags`.
    - *Input / Output:* Connected from `Prepare Dividend Risk Record` / Outputs appended rows to `Build Weekly Risk Digest`.
  - **Store Flagged Risks in Supabase**
    - *Type and Technical Role:* `n8n-nodes-base.supabase` (Database Writer)
    - *Configuration:* Inserts full structured record into table `dividend_risk_analysis`. Maps scan dates, financial indicators, risk attributes, and AI explanations.
    - *Input / Output:* Connected from `Prepare Dividend Risk Record` / Outputs inserted records to `Build Weekly Risk Digest`.
    - *Edge Cases:* Database constraint violations or missing table schema columns.
  - **Build Weekly Risk Digest**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Report Builder)
    - *Configuration:* Sorts records descending by `risk_score`. Compiles a text digest summarizing high-risk candidates, scores, red flags, and caution notes.
    - *Input / Output:* Connected from `Log Flagged Risks to Google Sheets` and `Store Flagged Risks in Supabase` / Fans out payload to Slack and Gmail nodes.
  - **Send Weekly Digest to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Notification Sender)
    - *Configuration:* Sends message content (`{{ $json.digest }}`) to channel ID `C0B1LNY15GW` ("n8n-workflow-testing").
    - *Input / Output:* Connected from `Build Weekly Risk Digest` / Terminal execution node.
    - *Edge Cases:* Invalid Slack token or missing channel membership permissions.
  - **Send Weekly Digest by Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Sender)
    - *Configuration:* Sends email to `user@example.com` with subject "Dividend Yield Trap Weekly Digest" and body `{{ $json.digest }}`.
    - *Input / Output:* Connected from `Build Weekly Risk Digest` / Terminal execution node.
    - *Edge Cases:* OAuth token expiration or quota limits.

---

#### Block 1.6: No-Risk Handling (Alternative Path)
- **Overview:** Handles execution when no scanned stocks reach the risk score threshold, generating an informational digest confirming normal status and dispatching alerts.
- **Nodes Involved:**
  - `Build No-Risk Weekly Digest`
  - `Send No-Risk Digest to Slack`
  - `Send No-Risk Digest by Gmail`

- **Node Details:**
  - **Build No-Risk Weekly Digest**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Message Builder)
    - *Configuration:* Generates a static payload stating that no candidates exceeded the threshold of 60 during the scan.
    - *Input / Output:* Connected from `Risk Above Threshold` (False branch) / Fans out to Slack and Gmail no-risk nodes.
  - **Send No-Risk Digest to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Notification Sender)
    - *Configuration:* Posts no-risk message string to channel `C0B1LNY15GW`.
    - *Input / Output:* Connected from `Build No-Risk Weekly Digest` / Terminal execution node.
  - **Send No-Risk Digest by Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Sender)
    - *Configuration:* Sends email to `user@example.com` with subject "Dividend Yield Trap Weekly Digest" and no-risk body content.
    - *Input / Output:* Connected from `Build No-Risk Weekly Digest` / Terminal execution node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Dividend Trap Scan1 | n8n-nodes-base.scheduleTrigger | Trigger workflow weekly | None | Read Dividend Watchlist | # Dividend Yield Trap Detector<br><br>## How it works:<br><br>This workflow runs a weekly dividend scan using an enabled Google Sheets watchlist. It retrieves upcoming ex dividend data and financial fundamentals, combines the information, calculates dividend sustainability and yield trap risk, and identifies financial red flags. Candidates with a risk score of sixty or higher are analyzed by AI, stored in Google Sheets and Supabase, and included in a weekly Slack and Gmail digest. If no candidates cross the threshold, a separate no-risk digest is sent.<br><br>## Setup Steps:<br>1. Configure the Google Sheets watchlist with enabled symbols, company names, sectors, and notes.<br>2. Configure the weekly schedule and verify the dividend calendar and fundamentals API endpoints.<br>3. Configure Google Sheets and Supabase credentials and verify the required result table and sheet.<br>4. Configure the OpenAI credential and model used for high-risk candidate explanations.<br>5. Configure Slack and Gmail credentials, destination channel, and recipient email address.<br>6. Test both branches by using data below and above the risk threshold of sixty, then activate the weekly schedule. |
| Read Dividend Watchlist | n8n-nodes-base.googleSheets | Read Google Sheet rows | Weekly Dividend Trap Scan1 | Prepare Watchlist Context | ## Watchlist Intake<br>This section starts the weekly dividend risk scan and retrieves the configured stock watchlist from Google Sheets. It filters the watchlist to include only enabled symbols and prepares the company, sector, notes, and symbol information required for the upcoming dividend analysis. |
| Prepare Watchlist Context | n8n-nodes-base.code | Filter and format watchlist | Read Dividend Watchlist | Get Upcoming Ex-Dividend Data | ## Watchlist Intake<br>This section starts the weekly dividend risk scan and retrieves the configured stock watchlist from Google Sheets. It filters the watchlist to include only enabled symbols and prepares the company, sector, notes, and symbol information required for the upcoming dividend analysis. |
| Get Upcoming Ex-Dividend Data | n8n-nodes-base.httpRequest | Fetch calendar API | Prepare Watchlist Context | Prepare Fundamentals Request | ## Dividend Data Collection<br>This section collects upcoming ex dividend information for the enabled watchlist and prepares the required symbols for fundamental analysis. The workflow then retrieves dividend and financial metrics such as yield, payout ratio, earnings growth, revenue growth, dividend growth, and historical dividend cuts. |
| Prepare Fundamentals Request | n8n-nodes-base.code | Filter calendar by date window | Get Upcoming Ex-Dividend Data | Get Dividend Fundamentals | ## Dividend Data Collection<br>This section collects upcoming ex dividend information for the enabled watchlist and prepares the required symbols for fundamental analysis. The workflow then retrieves dividend and financial metrics such as yield, payout ratio, earnings growth, revenue growth, dividend growth, and historical dividend cuts. |
| Get Dividend Fundamentals | n8n-nodes-base.httpRequest | Fetch fundamentals API | Prepare Fundamentals Request | Normalize Fundamentals | ## Dividend Data Collection<br>This section collects upcoming ex dividend information for the enabled watchlist and prepares the required symbols for fundamental analysis. The workflow then retrieves dividend and financial metrics such as yield, payout ratio, earnings growth, revenue growth, dividend growth, and historical dividend cuts. |
| Normalize Fundamentals | n8n-nodes-base.code | Normalize fundamentals payload | Get Dividend Fundamentals | Merge Calendar and Fundamentals | ## Data Normalization and Merge<br>This section converts the fundamentals response into a consistent structure and combines it with the upcoming dividend calendar and watchlist information. The workflow then calculates a sustainability score using payout ratio, earnings performance, revenue performance, dividend growth, and historical dividend cuts. |
| Merge Calendar and Fundamentals | n8n-nodes-base.code | Merge calendar and metrics | Normalize Fundamentals | Calculate Dividend Sustainability | ## Data Normalization and Merge<br>This section converts the fundamentals response into a consistent structure and combines it with the upcoming dividend calendar and watchlist information. The workflow then calculates a sustainability score using payout ratio, earnings performance, revenue performance, dividend growth, and historical dividend cuts. |
| Calculate Dividend Sustainability | n8n-nodes-base.code | Compute sustainability score | Merge Calendar and Fundamentals | Calculate Yield Trap Risk | ## Data Normalization and Merge<br>This section converts the fundamentals response into a consistent structure and combines it with the upcoming dividend calendar and watchlist information. The workflow then calculates a sustainability score using payout ratio, earnings performance, revenue performance, dividend growth, and historical dividend cuts. |
| Calculate Yield Trap Risk | n8n-nodes-base.code | Compute risk score and level | Calculate Dividend Sustainability | Identify Yield Trap Red Flags | ## Risk Calculation<br><br>This section converts dividend sustainability into an overall yield trap risk score between zero and one hundred. It assigns a risk level and identifies warning indicators such as elevated payout ratios, declining earnings, declining revenue, negative dividend growth, dividend cuts, and unusually high dividend yields. |
| Identify Yield Trap Red Flags | n8n-nodes-base.code | Compile warning flags | Calculate Yield Trap Risk | Risk Above Threshold | ## Risk Calculation<br><br>This section converts dividend sustainability into an overall yield trap risk score between zero and one hundred. It assigns a risk level and identifies warning indicators such as elevated payout ratios, declining earnings, declining revenue, negative dividend growth, dividend cuts, and unusually high dividend yields. |
| Risk Above Threshold | n8n-nodes-base.if | Branch based on risk score >= 60 | Identify Yield Trap Red Flags | Generate AI Yield Trap Explanation, Build No-Risk Weekly Digest | |
| Generate AI Yield Trap Explanation | @n8n/n8n-nodes-langchain.openAi | Generate AI explanation via OpenAI | Risk Above Threshold | Normalize AI Response | ## AI Risk Explanation<br><br>This section uses the AI model only for candidates that meet the configured risk threshold. The model creates a concise caution note, key concern, and investor question using the supplied financial data. The response is then parsed and combined with the original risk metrics into a structured record. |
| Normalize AI Response | n8n-nodes-base.code | Parse and normalize AI JSON | Generate AI Yield Trap Explanation | Prepare Dividend Risk Record | ## AI Risk Explanation<br><br>This section uses the AI model only for candidates that meet the configured risk threshold. The model creates a concise caution note, key concern, and investor question using the supplied financial data. The response is then parsed and combined with the original risk metrics into a structured record. |
| Prepare Dividend Risk Record | n8n-nodes-base.code | Assemble comprehensive risk record | Normalize AI Response | Log Flagged Risks to Google Sheets, Store Flagged Risks in Supabase | ## Risk Result Storage<br>This section records every flagged dividend risk candidate in both Google Sheets and Supabase. The stored record includes financial metrics, sustainability and risk scores, risk level, red flags, and AI-generated explanations. After storage, the flagged records are consolidated and sorted by risk score for the weekly digest. |
| Log Flagged Risks to Google Sheets | n8n-nodes-base.googleSheets | Append flagged risk to sheet | Prepare Dividend Risk Record | Build Weekly Risk Digest | ## Risk Result Storage<br>This section records every flagged dividend risk candidate in both Google Sheets and Supabase. The stored record includes financial metrics, sustainability and risk scores, risk level, red flags, and AI-generated explanations. After storage, the flagged records are consolidated and sorted by risk score for the weekly digest. |
| Store Flagged Risks in Supabase | n8n-nodes-base.supabase | Insert full record to Supabase | Prepare Dividend Risk Record | Build Weekly Risk Digest | ## Risk Result Storage<br>This section records every flagged dividend risk candidate in both Google Sheets and Supabase. The stored record includes financial metrics, sustainability and risk scores, risk level, red flags, and AI-generated explanations. After storage, the flagged records are consolidated and sorted by risk score for the weekly digest. |
| Build Weekly Risk Digest | n8n-nodes-base.code | Sort risks and build digest text | Log Flagged Risks to Google Sheets, Store Flagged Risks in Supabase | Send Weekly Digest to Slack, Send Weekly Digest by Gmail | ## Risk Notifications<br>This section distributes the weekly dividend risk report to stakeholders through Slack and Gmail. The digest contains the number of high-risk candidates and summarizes their risk scores, dividend information, sustainability scores, red flags, and AI-generated caution notes for review. |
| Send Weekly Digest to Slack | n8n-nodes-base.slack | Send high-risk digest to Slack | Build Weekly Risk Digest | None | ## Risk Notifications<br>This section distributes the weekly dividend risk report to stakeholders through Slack and Gmail. The digest contains the number of high-risk candidates and summarizes their risk scores, dividend information, sustainability scores, red flags, and AI-generated caution notes for review. |
| Send Weekly Digest by Gmail | n8n-nodes-base.gmail | Send high-risk digest via email | Build Weekly Risk Digest | None | ## Risk Notifications<br>This section distributes the weekly dividend risk report to stakeholders through Slack and Gmail. The digest contains the number of high-risk candidates and summarizes their risk scores, dividend information, sustainability scores, red flags, and AI-generated caution notes for review. |
| Build No-Risk Weekly Digest | n8n-nodes-base.code | Build no-risk notification payload | Risk Above Threshold | Send No-Risk Digest to Slack, Send No-Risk Digest by Gmail | ## No Risk Handling<br><br>This section handles scans where no stock reaches the configured risk threshold of sixty. Instead of producing an empty report, the workflow creates a clear no-risk weekly digest and sends it through both Slack and Gmail so stakeholders know that the scan completed without qualifying risk candidates. |
| Send No-Risk Digest to Slack | n8n-nodes-base.slack | Send no-risk digest to Slack | Build No-Risk Weekly Digest | None | ## No Risk Handling<br><br>This section handles scans where no stock reaches the configured risk threshold of sixty. Instead of producing an empty report, the workflow creates a clear no-risk weekly digest and sends it through both Slack and Gmail so stakeholders know that the scan completed without qualifying risk candidates. |
| Send No-Risk Digest by Gmail | n8n-nodes-base.gmail | Send no-risk digest via email | Build No-Risk Weekly Digest | None | ## No Risk Handling<br><br>This section handles scans where no stock reaches the configured risk threshold of sixty. Instead of producing an empty report, the workflow creates a clear no-risk weekly digest and sends it through both Slack and Gmail so stakeholders know that the scan completed without qualifying risk candidates. |

---

### 4. Reproducing the Workflow from Scratch

Follow this numbered list to rebuild the workflow manually in n8n:

1. **Create the Trigger:** Add a **Schedule Trigger** node (`Weekly Dividend Trap Scan1`). Set the interval to run weekly at hour 9.
2. **Read Watchlist:** Add a **Google Sheets** node (`Read Dividend Watchlist`). Configure credentials, set Document ID to `1FCGEXmbdS58yqfgqjhJCUn8IC0nZqDUDBi4l7FMWRmQ`, select the "Watchlist" sheet (`gid=0`), and connect it downstream from the trigger.
3. **Filter Watchlist:** Add a **Code** node (`Prepare Watchlist Context`). Insert JavaScript to filter enabled rows, validate symbols, and output `watchlist` and `symbols` arrays. Connect it from `Read Dividend Watchlist`.
4. **Fetch Calendar Data:** Add an **HTTP Request** node (`Get Upcoming Ex-Dividend Data`). Set method to GET and URL to `http://192.168.101.63:5678/webhook-test/mock/dividend-calendar` with JSON response format. Connect from `Prepare Watchlist Context`.
5. **Process Calendar Window:** Add a **Code** node (`Prepare Fundamentals Request`). Insert JavaScript to filter calendar events falling within 7 days from today that match enabled watchlist symbols. Connect from `Get Upcoming Ex-Dividend Data`.
6. **Fetch Fundamentals:** Add an **HTTP Request** node (`Get Dividend Fundamentals`). Set method to GET and URL to `http://192.168.101.63:5678/webhook-test/mock/dividend-fundamentals`. Connect from `Prepare Fundamentals Request`.
7. **Normalize Fundamentals:** Add a **Code** node (`Normalize Fundamentals`) to structure the incoming fundamentals array. Connect from `Get Dividend Fundamentals`.
8. **Merge Datasets:** Add a **Code** node (`Merge Calendar and Fundamentals`). Combine calendar candidates and fundamentals using a symbol map, filtering out items without a `share_price`. Connect from `Normalize Fundamentals`.
9. **Calculate Sustainability:** Add a **Code** node (`Calculate Dividend Sustainability`). Implement scoring penalties based on payout ratios, earnings/revenue growth, dividend growth, and cuts. Output `sustainability_score`. Connect from `Merge Dataset`.
10. **Calculate Risk:** Add a **Code** node (`Calculate Yield Trap Risk`). Compute `risk_score` ($100 - \text{sustainability}$ plus yield/payout/growth penalties) and assign `risk_level` (Low, Medium, High). Connect from `Calculate Dividend Sustainability`.
11. **Identify Red Flags:** Add a **Code** node (`Identify Yield Trap Red Flags`). Evaluate conditions and populate the `red_flags` array and count. Connect from `Calculate Yield Trap Risk`.
12. **Add Risk Threshold Branch:** Add an **If** node (`Risk Above Threshold`). Set condition to check if `{{ $json.risk_score }} >= 60`. Connect from `Identify Yield Trap Red Flags`.
13. **Configure AI Analysis (True Branch):**
    - Add an **OpenAI** node (`Generate AI Yield Trap Explanation`) connected to index `0` of the If node. Configure credentials (`OpenAI (PAID) (AI-ML Team)`), select model `gpt-4.1-nano`, and provide the structured JSON prompt.
    - Add a **Code** node (`Normalize AI Response`) to parse the JSON output safely. Connect from the OpenAI node.
    - Add a **Code** node (`Prepare Dividend Risk Record`) to assemble all metrics, flags, and AI notes into a unified record. Connect from `Normalize AI Response`.
14. **Configure Storage:**
    - Add a **Google Sheets** node (`Log Flagged Risks to Google Sheets`) configured with Document ID `1FCGEXmbdS58yqfgqjhJCUn8IC0nZqDUDBi4l7FMWRmQ`, Sheet Name "Risk Results" (`gid=1650153831`), and operation `append`. Map columns for symbol, yields, risk scores, levels, and flags. Connect from `Prepare Dividend Risk Record`.
    - Add a **Supabase** node (`Store Flagged Risks in Supabase`) configured with table `dividend_risk_analysis` and field mappings for all analytical indicators and AI notes. Connect from `Prepare Dividend Risk Record`.
15. **Configure High-Risk Notifications:**
    - Add a **Code** node (`Build Weekly Risk Digest`) taking inputs from both storage nodes to sort risks and format the summary text.
    - Add a **Slack** node (`Send Weekly Digest to Slack`) and a **Gmail** node (`Send Weekly Digest by Gmail`) connected downstream from `Build Weekly Risk Digest`. Set Slack channel ID `C0B1LNY15GW` and Gmail recipient address.
16. **Configure No-Risk Handling (False Branch):**
    - Add a **Code** node (`Build No-Risk Weekly Digest`) connected to index `1` of `Risk Above Threshold`.
    - Add **Slack** (`Send No-Risk Digest to Slack`) and **Gmail** (`Send No-Risk Digest by Gmail`) nodes connected downstream to send the no-risk notification payload.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| For help with workflow setup, integration, customization, troubleshooting or building similar n8n automations, contact WeblineIndia. | [WeblineIndia Contact Page](https://www.weblineindia.com/contact-us.html) |