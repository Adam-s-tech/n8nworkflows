Triage backflow compliance notices with Google Sheets, Gmail, Slack, OpenAI, and a state API

https://n8nworkflows.xyz/workflows/triage-backflow-compliance-notices-with-google-sheets--gmail--slack--openai--and-a-state-api-19760


# Triage backflow compliance notices with Google Sheets, Gmail, Slack, OpenAI, and a state API

### 1. Workflow Overview

The **Backflow Preventer Compliance Router** workflow automates the daily operational queue for municipal water utility cross-connection control programs. Its core purpose is to read a backflow device registry, calculate test overdue metrics and risk scores, classify each device using an AI model, and distribute enforcement notifications, state compliance reports, and audit logs automatically.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Preparation:** Triggers daily, fetches all device entries from Google Sheets, separates them into individual device records, computes temporal metrics (days since last test, days overdue, next due date) and hazard risk scores, and standardizes the data schema.
- **1.2 Data Completeness Verification:** Validates that mandatory operational identifiers (`device_id` and `owner_email`) exist before attempting AI classification. Incomplete rows trigger an immediate Slack data quality warning.
- **1.3 AI-Powered Compliance Classification:** Utilizes an OpenAI chat model configured with a municipal cross-connection control officer persona to categorize each device into strictly defined operational states: *Critical Overdue*, *Due Soon*, or *Compliant*.
- **1.4 Categorized Action Routing:** Evaluates the classification status via a conditional switch and executes branch-specific actions:
  - *Critical Overdue:* Posts a non-compliance report to the state e-reporting portal API, sends a 10-day shutoff warning via Gmail, and alerts the internal compliance team via Slack.
  - *Due Soon:* Sends a test reminder notice to the property owner via Gmail.
  - *Compliant:* Assigns a no-action audit timestamp.
- **1.5 Audit Trail Consolidation:** Merges all processing branches back into a single pipeline to append a consolidated historical audit row for every evaluated device into a Google Sheets audit log.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Preparation
- **Overview:** Automatically initiates the daily scan, retrieves the device registry database, breaks down bulk rows into discrete items, calculates risk metrics using custom code, and normalizes the output parameters.
- **Nodes Involved:** 
  - `Daily Compliance Scan`
  - `Fetch Backflow Device Registry`
  - `Split Out Device Records`
  - `Compute Days Since Last Test`
  - `Normalize Device Record`

- **Node Details:**
  - **Daily Compliance Scan**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2). Acts as the time-based workflow entry point.
    - *Configuration Choices:* Configured with a cron expression (`0 6 * * *`) to execute daily at 06:00 AM.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Output connected to `Fetch Backflow Device Registry`.
    - *Edge Cases/Failures:* Missed executions if the n8n instance is offline at the exact trigger minute.

  - **Fetch Backflow Device Registry**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5). Retrieves all rows from the specified spreadsheet tab.
    - *Configuration Choices:* Operation set to read, sheet name set to `Device Registry`, document ID set via placeholder `REPLACE_WITH_SHEET_ID`. Error handling enabled (`continueRegularOutput`) with up to 3 retries and a 2000ms delay.
    - *Key Expressions/Variables:* Document ID string placeholder.
    - *Input/Output:* Input from `Daily Compliance Scan`; output connected to `Split Out Device Records`.
    - *Edge Cases/Failures:* API authentication failures, rate-limiting quotas, or invalid Sheet IDs halting the initial data fetch.

  - **Split Out Device Records**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (v1). Flattens array elements into individual items for per-device processing.
    - *Configuration Choices:* Field to split out set to `data`.
    - *Key Expressions/Variables:* `data` array field from Google Sheets output.
    - *Input/Output:* Input from `Fetch Backflow Device Registry`; output connected to `Compute Days Since Last Test`.
    - *Edge Cases/Failures:* Malformed JSON arrays resulting in empty execution streams.

  - **Compute Days Since Last Test**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2). Executes custom JavaScript to evaluate operational risk metrics.
    - *Configuration Choices:* Uses native JavaScript `Date` operations to calculate days since last test, compute overdue windows against a standard 365-day test cycle, factor hazard levels (`high`, `medium`, `low`), and assign a cumulative `risk_score` (capped at 100).
    - *Key Expressions/Variables:* Reads `d.last_test_date`, `d.hazard_level`, and `d.device_type`. Outputs `days_since_last_test`, `days_overdue`, `next_due_date`, and `risk_score`.
    - *Input/Output:* Input from `Split Out Device Records`; output connected to `Normalize Device Record`.
    - *Edge Cases/Failures:* Date parsing exceptions if date strings deviate from standard ISO/datestamp formats.

  - **Normalize Device Record**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Shapes and restricts data fields passed downstream.
    - *Configuration Choices:* Explicitly maps and assigns properties: `device_id`, `address`, `hazard_level`, `days_overdue`, `risk_score`, and `owner_email`.
    - *Key Expressions/Variables:* `={{$json.device_id}}`, `={{$json.address}}`, `={{$json.hazard_level}}`, `={{$json.days_overdue}}`, `={{$json.risk_score}}`, `={{$json.owner_email}}`.
    - *Input/Output:* Input from `Compute Days Since Last Test`; output connected to `Check Device Data Completeness`.
    - *Edge Cases/Failures:* Missing property keys yielding empty string or null assignments.

---

#### Block 1.2: Data Completeness Verification
- **Overview:** Validates record integrity to ensure essential operational fields are populated before running AI classification or dispatching automated notices.
- **Nodes Involved:**
  - `Check Device Data Completeness`
  - `Flag Data Quality Issue`

- **Node Details:**
  - **Check Device Data Completeness**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2). Evaluates conditional expressions to split the execution path.
    - *Configuration Choices:* Strict type validation enabled with a combinator set to `and`. Conditions check that both `device_id` and `owner_email` are not empty.
    - *Key Expressions/Variables:* `={{$json.device_id}}` (not empty), `={{$json.owner_email}}` (not empty).
    - *Input/Output:* Input from `Normalize Device Record`. True output connected to `Classify Compliance Status`; false output connected to `Flag Data Quality Issue`.
    - *Edge Cases/Failures:* Whitespace-only strings passing validation if not trimmed.

  - **Flag Data Quality Issue**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (v2.2). Terminal notification node for data remediation alerts.
    - *Configuration Choices:* Sends a message to the `#cross-connection-data-quality` Slack channel. Error handling enabled (`continueRegularOutput`) with 3 retries.
    - *Key Expressions/Variables:* `=Registry row missing required fields: {{$json.device_id || 'UNKNOWN'}} / {{$json.address || 'no address'}}`.
    - *Input/Output:* Input from the false branch of `Check Device Data Completeness`; terminal branch (no downstream connections).
    - *Edge Cases/Failures:* Slack API token permission errors or missing channel access.

---

#### Block 1.3: AI-Powered Compliance Classification
- **Overview:** Leverages an OpenAI language model and a structured text classifier to assign a standardized compliance status to each fully populated device record.
- **Nodes Involved:**
  - `Compliance Classifier Model`
  - `Classify Compliance Status`

- **Node Details:**
  - **Compliance Classifier Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2). Provides the foundational LLM backend.
    - *Configuration Choices:* Model set to `gpt-4.1-mini` with temperature configured to `0.1` for deterministic, low-variance categorization. Error handling enabled with 3 retries.
    - *Key Expressions/Variables:* None (sub-node linked via AI language model parameter).
    - *Input/Output:* Connected to `Classify Compliance Status` via `ai_languageModel` connection.
    - *Edge Cases/Failures:* OpenAI API rate limits, billing limits, or upstream service outages.

  - **Classify Compliance Status**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.textClassifier` (v1). Evaluates incoming text against configured categories using the linked LLM.
    - *Configuration Choices:* System prompt sets a municipal compliance officer persona. Configured with three exact categories: *Critical Overdue*, *Due Soon*, and *Compliant*, with strict descriptions based on risk scores and test cycle windows.
    - *Key Expressions/Variables:* System prompt string template.
    - *Input/Output:* Input from `Check Device Data Completeness` (true branch); output connected to `Route By Compliance Status`.
    - *Edge Cases/Failures:* Unexpected model response drift if classification guidelines are interpreted loosely by the model.

---

#### Block 1.4: Categorized Action Routing
- **Overview:** Directs the classified device records down distinct operational pathways for state reporting, owner notifications, internal alerts, or standard logging.
- **Nodes Involved:**
  - `Route By Compliance Status`
  - `File State Non-Compliance Report`
  - `Send Test Reminder Notice`
  - `Mark Registry Updated`
  - `Send Shutoff Warning Letter`
  - `Alert Compliance Team Critical Device`

- **Node Details:**
  - **Route By Compliance Status**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (v3.2). Directs items down specific branches based on matching rule criteria.
    - *Configuration Choices:* Evaluates `={{$json.category}}` against three string-match rules: `Critical Overdue`, `Due Soon`, and `Compliant`. Fallback output set to none.
    - *Key Expressions/Variables:* `={{$json.category}}`.
    - *Input/Output:* Input from `Classify Compliance Status`. Output 0 connected to `File State Non-Compliance Report`; Output 1 connected to `Send Test Reminder Notice`; Output 2 connected to `Mark Registry Updated`.
    - *Edge Cases/Failures:* Unmatched category labels dropping out of the workflow entirely if fallback handling is absent.

  - **File State Non-Compliance Report**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Integrates with external REST APIs to report compliance violations.
    - *Configuration Choices:* HTTP method set to `POST`, target URL set to `https://REPLACE-state-ereporting-portal.example.gov/api/v1/noncompliance`. Sends a JSON body containing `device_id`, `address`, `days_overdue`, and `hazard_level`.
    - *Key Expressions/Variables:* `={{ { device_id: $json.device_id, address: $json.address, days_overdue: $json.days_overdue, hazard_level: $json.hazard_level } }}`.
    - *Input/Output:* Input from switch output 0; output connected to `Send Shutoff Warning Letter`.
    - *Edge Cases/Failures:* HTTP 4xx/5xx responses from the state portal API, network timeouts, or invalid authentication headers.

  - **Send Test Reminder Notice**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.1). Dispatches automated email messages to property owners.
    - *Configuration Choices:* Configured to send an email via Gmail OAuth2. Subject and message body dynamically populated. Error handling enabled with 3 retries.
    - *Key Expressions/Variables:* 
      - Send To: `={{$json.owner_email}}`
      - Subject: `=Backflow Preventer Test Reminder - {{$json.address}}`
      - Message: `=Your backflow preventer at {{$json.address}} is due for its annual test within 30 days. Please schedule with a certified tester.`
    - *Input/Output:* Input from switch output 1; output connected to `Combine Audit Trail` (index 1).
    - *Edge Cases/Failures:* Invalid email addresses, SMTP rejection, or OAuth token revocation.

  - **Mark Registry Updated**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Assigns static metadata values for compliant records requiring no outreach.
    - *Configuration Choices:* Assigns `audit_action` as `"Compliant - no action"` and `audit_timestamp` using the current ISO timestamp.
    - *Key Expressions/Variables:* `={{$now.toISO()}}`.
    - *Input/Output:* Input from switch output 2; output connected to `Combine Audit Trail` (via parallel merge streams).
    - *Edge Cases/Failures:* Expression evaluation failures on timestamp generation.

  - **Send Shutoff Warning Letter**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.1). Sends formal legal/ordinance warning notices to owners of critically overdue devices.
    - *Configuration Choices:* Configured via Gmail OAuth2 with dynamic recipient and body parameters. Error handling enabled with 3 retries.
    - *Key Expressions/Variables:*
      - Send To: `={{$json.owner_email}}`
      - Subject: `=URGENT: Backflow Preventer Non-Compliance - {{$json.address}}`
      - Message: `=Your device at {{$json.address}} is {{$json.days_overdue}} days overdue for testing. Water service may be discontinued under municipal cross-connection control ordinance if not remedied within 10 days.`
    - *Input/Output:* Input from `File State Non-Compliance Report`; output connected to `Alert Compliance Team Critical Device`.
    - *Edge Cases/Failures:* Mailbox sending limits or invalid recipient email formats.

  - **Alert Compliance Team Critical Device**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (v2.2). Sends operational alerts to internal teams regarding critical enforcement actions.
    - *Configuration Choices:* Posts to the `#cross-connection-alerts` Slack channel. Error handling enabled with 3 retries.
    - *Key Expressions/Variables:* `=CRITICAL: {{$json.device_id}} at {{$json.address}} is {{$json.days_overdue}} days overdue (risk {{$json.risk_score}}). Shutoff letter sent.`
    - *Input/Output:* Input from `Send Shutoff Warning Letter`; output connected to `Combine Audit Trail` (index 0).
    - *Edge Cases/Failures:* Slack webhook delivery errors or missing bot permissions in the target channel.

---

#### Block 1.5: Audit Trail Consolidation
- **Overview:** Re-converges all discrete processing branches into a single data stream and records a comprehensive audit entry for every processed device into Google Sheets.
- **Nodes Involved:**
  - `Combine Audit Trail`
  - `Append Compliance Audit Log`

- **Node Details:**
  - **Combine Audit Trail**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (v3.2). Consolidates multiple incoming execution branches into a unified output.
    - *Configuration Choices:* Mode set to `combine` with combination method set to `combineAll`.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs from `Send Test Reminder Notice`, `Alert Compliance Team Critical Device`, and `Mark Registry Updated`. Output connected to `Append Compliance Audit Log`.
    - *Edge Cases/Failures:* Data misalignment if branch execution sequences conflict unexpectedly.

  - **Append Compliance Audit Log**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5). Appends historical log data to a designated spreadsheet.
    - *Configuration Choices:* Operation set to `append`, sheet name set to `Audit Log`, document ID set via placeholder `REPLACE_WITH_SHEET_ID`. Error handling enabled with 3 retries.
    - *Key Expressions/Variables:* Document ID string placeholder.
    - *Input/Output:* Input from `Combine Audit Trail`; terminal workflow node.
    - *Edge Cases/Failures:* Google Sheets API rate limits or schema mismatch errors during row appending.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Compliance Scan | n8n-nodes-base.scheduleTrigger | Triggers the daily workflow execution | None | Fetch Backflow Device Registry | **Daily Compliance Scan**<br>Fires every day at 06:00 to scan the backflow device registry for due/overdue tests.<br>Schedule: daily 06:00 cron |
| Fetch Backflow Device Registry | n8n-nodes-base.googleSheets | Reads the device registry spreadsheet | Daily Compliance Scan | Split Out Device Records | **Fetch Backflow Device Registry**<br>Reads every tracked backflow preventer device row from the municipal registry sheet.<br>Sheet: 'Device Registry' tab |
| Split Out Device Records | n8n-nodes-base.splitOut | Flattens registry array into individual device items | Fetch Backflow Device Registry | Compute Days Since Last Test | **Split Out Device Records**<br>Splits the registry read into one item per physical backflow preventer device.<br>Splits on: registry rows |
| Compute Days Since Last Test | n8n-nodes-base.code | Calculates overdue days and risk scores | Split Out Device Records | Normalize Device Record | **Compute Days Since Last Test**<br>Real logic: computes days overdue vs the annual test cycle and a 0-100 risk score by hazard class.<br>Cycle: 365 days; risk = overdue-days + hazard weight |
| Normalize Device Record | n8n-nodes-base.set | Standardizes and structures device output properties | Compute Days Since Last Test | Check Device Data Completeness | **Normalize Device Record**<br>Shapes the enriched record down to the fields the classifier and downstream actions need.<br>Fields: device_id, hazard_level, days_overdue, risk_score |
| Check Device Data Completeness | n8n-nodes-base.if | Validates presence of required operational fields | Normalize Device Record | Classify Compliance Status, Flag Data Quality Issue | **Check Device Data Completeness**<br>Branch point 1: guards against incomplete registry rows before AI classification runs.<br>Requires: device_id AND owner_email present<br>Bad data would otherwise mis-route enforcement letters. |
| Flag Data Quality Issue | n8n-nodes-base.slack | Alerts team of incomplete data records via Slack | Check Device Data Completeness | None | **Flag Data Quality Issue**<br>Terminal path: alerts the data-entry team when a registry row is missing device_id or owner_email.<br>Channel: #cross-connection-data-quality |
| Compliance Classifier Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM backend for text classification | None | Classify Compliance Status | **Compliance Classifier Model**<br>Chat model backing the compliance classifier; low temperature for consistent categorisation.<br>Model: gpt-4.1-mini, temp 0.1 |
| Classify Compliance Status | @n8n/n8n-nodes-langchain.textClassifier | Classifies devices into compliance categories using AI | Check Device Data Completeness | Route By Compliance Status | **Classify Compliance Status**<br>AI step: classifies each device into Critical Overdue / Due Soon / Compliant per cross-connection policy.<br>3 categories; strict single-label output |
| Route By Compliance Status | n8n-nodes-base.switch | Routes execution path based on compliance category | Classify Compliance Status | File State Non-Compliance Report, Send Test Reminder Notice, Mark Registry Updated | **Route By Compliance Status**<br>Branch point 2: genuine 3-way Switch routing on the AI-assigned compliance category.<br>Outputs: Critical Overdue / Due Soon / Compliant |
| File State Non-Compliance Report | n8n-nodes-base.httpRequest | Submits compliance violation reports to state API | Route By Compliance Status | Send Shutoff Warning Letter | **File State Non-Compliance Report**<br>Files the critical non-compliance record with the state cross-connection e-reporting portal.<br>POST /api/v1/noncompliance |
| Send Test Reminder Notice | n8n-nodes-base.gmail | Sends annual test reminder emails to property owners | Route By Compliance Status | Combine Audit Trail | **Send Test Reminder Notice**<br>Sends a courtesy reminder to property owners whose device is due soon but not yet overdue.<br>Trigger: category = Due Soon |
| Mark Registry Updated | n8n-nodes-base.set | Assigns audit action metadata for compliant devices | Route By Compliance Status | Combine Audit Trail | **Mark Registry Updated**<br>Compliant devices need no outreach; this just stamps an audit action for the log.<br>audit_action = 'Compliant - no action' |
| Send Shutoff Warning Letter | n8n-nodes-base.gmail | Dispatches formal shutoff warning letters via email | File State Non-Compliance Report | Alert Compliance Team Critical Device | **Send Shutoff Warning Letter**<br>Sends the formal shutoff-warning letter required before enforcement on critically overdue devices.<br>10-day cure notice per ordinance |
| Alert Compliance Team Critical Device | n8n-nodes-base.slack | Notifies internal teams of critical enforcement actions | Send Shutoff Warning Letter | Combine Audit Trail | **Alert Compliance Team Critical Device**<br>Notifies the compliance team in real time whenever a critical overdue device is actioned.<br>Channel: #cross-connection-alerts |
| Combine Audit Trail | n8n-nodes-base.merge | Reconverges all processing branches into a single stream | Send Test Reminder Notice, Alert Compliance Team Critical Device, Mark Registry Updated | Append Compliance Audit Log | **Combine Audit Trail**<br>Reconverges all three Switch branches into a single stream for one unified audit log write.<br>3 inputs merged |
| Append Compliance Audit Log | n8n-nodes-base.googleSheets | Appends unified audit records to Google Sheets log | Combine Audit Trail | None | **Append Compliance Audit Log**<br>Append Compliance Audit Log<br>Appends one audit row per device processed today to the compliance audit log tab.<br>Sheet: 'Audit Log' tab |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to manually reconstruct the workflow in n8n:

1. **Create Schedule Trigger Node**
   - Node Type: `n8n-nodes-base.scheduleTrigger`
   - Name: `Daily Compliance Scan`
   - Configuration: Set trigger rule to cron expression `0 6 * * *`.

2. **Create Google Sheets Read Node**
   - Node Type: `n8n-nodes-base.googleSheets`
   - Name: `Fetch Backflow Device Registry`
   - Configuration: Set operation to read, Sheet Name to `Device Registry`, and Document ID to `REPLACE_WITH_SHEET_ID`. Enable error handling (`continueRegularOutput`, 3 retries, 2000ms delay).
   - Connection: Connect `Daily Compliance Scan` output to this node.

3. **Create Split Out Node**
   - Node Type: `n8n-nodes-base.splitOut`
   - Name: `Split Out Device Records`
   - Configuration: Field to split out set to `data`.
   - Connection: Connect `Fetch Backflow Device Registry` output to this node.

4. **Create Code Node for Calculations**
   - Node Type: `n8n-nodes-base.code`
   - Name: `Compute Days Since Last Test`
   - Configuration: Paste JavaScript code to calculate test cycle days, overdue status, and hazard risk scores.
   - Connection: Connect `Split Out Device Records` output to this node.

5. **Create Set Node for Normalization**
   - Node Type: `n8n-nodes-base.set`
   - Name: `Normalize Device Record`
   - Configuration: Add assignments for `device_id`, `address`, `hazard_level`, `days_overdue`, `risk_score`, and `owner_email` referencing incoming item properties.
   - Connection: Connect `Compute Days Since Last Test` output to this node.

6. **Create IF Validation Node**
   - Node Type: `n8n-nodes-base.if`
   - Name: `Check Device Data Completeness`
   - Configuration: Add strict conditions verifying `device_id` is not empty and `owner_email` is not empty.
   - Connection: Connect `Normalize Device Record` output to this node.

7. **Create Slack Data Quality Node (False Branch)**
   - Node Type: `n8n-nodes-base.slack`
   - Name: `Flag Data Quality Issue`
   - Configuration: Select channel `#cross-connection-data-quality`, set message text template for missing fields. Enable error handling.
   - Connection: Connect the `false` output of `Check Device Data Completeness` to this node.

8. **Create OpenAI Chat Model Sub-Node**
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Name: `Compliance Classifier Model`
   - Configuration: Set model to `gpt-4.1-mini` with temperature `0.1`. Configure OpenAI API credentials.
   - Connection: Link to `Classify Compliance Status` via AI language model connection.

9. **Create Text Classifier AI Node (True Branch)**
   - Node Type: `@n8n/n8n-nodes-langchain.textClassifier`
   - Name: `Classify Compliance Status`
   - Configuration: Configure system prompt template and define categories: *Critical Overdue*, *Due Soon*, and *Compliant*.
   - Connection: Connect the `true` output of `Check Device Data Completeness` to this node, and the language model sub-node.

10. **Create Switch Routing Node**
    - Node Type: `n8n-nodes-base.switch`
    - Name: `Route By Compliance Status`
    - Configuration: Create 3 rules matching `Critical Overdue`, `Due Soon`, and `Compliant`.
    - Connection: Connect `Classify Compliance Status` output to this node.

11. **Create HTTP Request Node (Critical Path)**
    - Node Type: `n8n-nodes-base.httpRequest`
    - Name: `File State Non-Compliance Report`
    - Configuration: Method `POST`, URL `https://REPLACE-state-ereporting-portal.example.gov/api/v1/noncompliance`, specify JSON body with device parameters. Configure HTTP Header Auth credentials.
    - Connection: Connect Switch output index 0 (`Critical Overdue`) to this node.

12. **Create Gmail Reminder Node (Due Soon Path)**
    - Node Type: `n8n-nodes-base.gmail`
    - Name: `Send Test Reminder Notice`
    - Configuration: Configure Gmail OAuth2 credentials, recipient `={{$json.owner_email}}`, subject, and reminder message.
    - Connection: Connect Switch output index 1 (`Due Soon`) to this node.

13. **Create Set Metadata Node (Compliant Path)**
    - Node Type: `n8n-nodes-base.set`
    - Name: `Mark Registry Updated`
    - Configuration: Assign `audit_action` = `"Compliant - no action"` and `audit_timestamp` = `={{$now.toISO()}}`.
    - Connection: Connect Switch output index 2 (`Compliant`) to this node.

14. **Create Gmail Warning Node (Critical Path continuation)**
    - Node Type: `n8n-nodes-base.gmail`
    - Name: `Send Shutoff Warning Letter`
    - Configuration: Configure Gmail OAuth2 credentials, recipient `={{$json.owner_email}}`, subject, and 10-day shutoff warning notice body.
    - Connection: Connect `File State Non-Compliance Report` output to this node.

15. **Create Slack Alert Node (Critical Path continuation)**
    - Node Type: `n8n-nodes-base.slack`
    - Name: `Alert Compliance Team Critical Device`
    - Configuration: Target channel `#cross-connection-alerts`, message text template reporting critical overdue status.
    - Connection: Connect `Send Shutoff Warning Letter` output to this node.

16. **Create Merge Node**
    - Node Type: `n8n-nodes-base.merge`
    - Name: `Combine Audit Trail`
    - Configuration: Mode set to `combine` with `combineAll`.
    - Connections: Connect outputs from `Send Test Reminder Notice`, `Alert Compliance Team Critical Device`, and `Mark Registry Updated` into this merge node.

17. **Create Google Sheets Append Node**
    - Node Type: `n8n-nodes-base.googleSheets`
    - Name: `Append Compliance Audit Log`
    - Configuration: Operation set to `append`, Sheet Name to `Audit Log`, Document ID to `REPLACE_WITH_SHEET_ID`. Configure Google Sheets OAuth2 credentials.
    - Connection: Connect `Combine Audit Trail` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Credits & Developer Contact | Swapnil AI Labs — [swapnil.mandloi7@gmail.com](mailto:swapnil.mandloi7@gmail.com) — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Workflow Purpose & Value Proposition | Automates municipal cross-connection control enforcement queues for water utility backflow devices. |
| Required External Credentials | Google Sheets OAuth2, Gmail OAuth2, Slack API, HTTP Header Auth (State Portal), and OpenAI API. |