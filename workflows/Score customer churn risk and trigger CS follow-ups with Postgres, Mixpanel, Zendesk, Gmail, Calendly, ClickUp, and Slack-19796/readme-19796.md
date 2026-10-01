Score customer churn risk and trigger CS follow-ups with Postgres, Mixpanel, Zendesk, Gmail, Calendly, ClickUp, and Slack

https://n8nworkflows.xyz/workflows/score-customer-churn-risk-and-trigger-cs-follow-ups-with-postgres--mixpanel--zendesk--gmail--calendly--clickup--and-slack-19796


# Score customer churn risk and trigger CS follow-ups with Postgres, Mixpanel, Zendesk, Gmail, Calendly, ClickUp, and Slack

### 1. Workflow Overview

This workflow automates the detection and mitigation of customer churn risk. Operating on a 6-hour schedule, it aggregates signals from four distinct sources: database product usage snapshots, Mixpanel engagement metrics, database NPS survey feedback, and Zendesk support tickets. The aggregated data is unified per customer, processed through a composite risk-scoring algorithm, and routed through conditional gates. At-risk customers receive automated check-in communications via Gmail, while high-risk accounts trigger calendar booking integrations, task management items in ClickUp, and direct escalations to account managers via Slack. Every automated intervention is audit-logged back to a Postgres database.

The workflow logic is categorized into four sequential blocks:
- **1.1 Schedule Trigger & Parallel Signal Collection:** Initiates execution every 6 hours and queries Postgres, Mixpanel, and Zendesk simultaneously.
- **1.2 Data Unification & Risk Scoring:** Merges the parallel data streams by `customer_id` and computes a composite churn score and risk category.
- **1.3 Risk-Based Intervention Routing:** Evaluates customer risk scores via conditional logic to determine whether to bypass or route accounts to appropriate communication channels.
- **1.4 Multi-Channel Action & Audit Logging:** Dispatches emails, schedules meetings, creates project management tasks, posts internal escalations, and logs the execution output.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & Parallel Signal Collection
- **Overview:** Triggers the workflow execution on a fixed 6-hour interval and initiates four parallel data collection requests across internal databases and external APIs.
- **Nodes Involved:** 
  - `Every 6 Hours1`
  - `Get Product Usage (DB)1`
  - `Get Mixpanel Engagement1`
  - `Get Latest NPS Scores1`
  - `Get Recent Support Tickets1`

- **Node Details:**
  - **`Every 6 Hours1`**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire execution every 6 hours.
    - *Key Expressions/Variables:* None.
    - *Connections:* Output connects to all four data-fetching nodes simultaneously.
    - *Edge Cases/Failure Types:* Missed executions if the n8n instance is offline during the scheduled interval.
  - **`Get Product Usage (DB)1`**
    - *Type and Technical Role:* `n8n-nodes-base.postgres` (Database Operation)
    - *Configuration Choices:* Executes a custom SQL query using operation `executeQuery` to select customer usage records updated within the last 6 hours (`customer_usage_snapshot`).
    - *Key Expressions/Variables:* SQL interval filter `NOW() - INTERVAL '6 hours'`.
    - *Connections:* Input from `Every 6 Hours1`; output connects to `Merge Usage + Mixpanel1` (Input 0).
    - *Edge Cases/Failure Types:* Database connection timeout, invalid table/column names, or authentication failure.
  - **`Get Mixpanel Engagement1`**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Sends a POST request to the Mixpanel Engage API (`https://mixpanel.com/api/2.0/engage`) with a predefined credential type.
    - *Key Expressions/Variables:* JSON body containing `project_id` set to `"YOUR_MIXPANEL_PROJECT_ID"` and requested `output_properties`.
    - *Connections:* Input from `Every 6 Hours1`; output connects to `Merge Usage + Mixpanel1` (Input 1).
    - *Edge Cases/Failure Types:* Invalid Mixpanel API credentials, rate limiting, or project ID configuration errors.
  - **`Get Latest NPS Scores1`**
    - *Type and Technical Role:* `n8n-nodes-base.postgres` (Database Operation)
    - *Configuration Choices:* Executes a custom SQL query via `executeQuery` to retrieve NPS responses from the last 30 days (`nps_responses`).
    - *Key Expressions/Variables:* SQL interval filter `NOW() - INTERVAL '30 days'`.
    - *Connections:* Input from `Every 6 Hours1`; output connects to `Merge + NPS1` (Input 1).
    - *Edge Cases/Failure Types:* Database connectivity issues or schema mismatches.
  - **`Get Recent Support Tickets1`**
    - *Type and Technical Role:* `n8n-nodes-base.zendesk` (Service Integration)
    - *Configuration Choices:* Operation set to `getAll` with options to sort by `created_at` in descending order.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input from `Every 6 Hours1`; output connects to `Merge + Tickets1` (Input 1).
    - *Edge Cases/Failure Types:* Zendesk API authentication errors or subdomain misconfigurations.

---

#### 2.2 Data Unification & Risk Scoring
- **Overview:** Progressively merges the four disparate data streams into a single unified customer profile based on matching identifier keys, then calculates a weighted churn risk score.
- **Nodes Involved:**
  - `Merge Usage + Mixpanel1`
  - `Merge + NPS1`
  - `Merge + Tickets1`
  - `Calculate Churn Risk Score1`

- **Node Details:**
  - **`Merge Usage + Mixpanel1`**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, join mode set to `keepEverything`, matching on string `customer_id`.
    - *Key Expressions/Variables:* `customer_id`.
    - *Connections:* Inputs from `Get Product Usage (DB)1` and `Get Mixpanel Engagement1`; output connects to `Merge + NPS1` (Input 0).
    - *Edge Cases/Failure Types:* Unmatched identifiers resulting in fragmented data objects if keys do not align between systems.
  - **`Merge + NPS1`**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, join mode set to `keepEverything`, matching on string `customer_id`.
    - *Key Expressions/Variables:* `customer_id`.
    - *Connections:* Inputs from `Merge Usage + Mixpanel1` and `Get Latest NPS Scores1`; output connects to `Merge + Tickets1` (Input 0).
    - *Edge Cases/Failure Types:* Missing NPS telemetry for active customers.
  - **`Merge + Tickets1`**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, join mode set to `keepEverything`, matching on string `customer_id`.
    - *Key Expressions/Variables:* `customer_id`.
    - *Connections:* Inputs from `Merge + NPS1` and `Get Recent Support Tickets1`; output connects to `Calculate Churn Risk Score1`.
    - *Edge Cases/Failure Types:* Missing Zendesk metrics leaving ticket fields unpopulated.
  - **`Calculate Churn Risk Score1`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Logic Execution)
    - *Configuration Choices:* Executes JavaScript to iterate over input items, calculates metrics (days since login, feature usage drop percentage, seat utilization, NPS detractor status, and support ticket sentiment/volume), and assigns a composite score.
    - *Key Expressions/Variables:* Custom JavaScript utilizing item fields (`last_login_at`, `feature_usage_30d`, `seats_active`, `nps_score`, `open_ticket_count`, etc.) to output `churn_score`, `risk_level` ('high'/'medium'/'low'), `churn_reasons`, and `is_churn_risk`.
    - *Connections:* Input from `Merge + Tickets1`; output connects to `Is Churn Risk?1`.
    - *Edge Cases/Failure Types:* TypeError exceptions when evaluating null or undefined numerical properties.

---

#### 2.3 Risk-Based Intervention Routing
- **Overview:** Evaluates the computed churn risk and risk level to direct customers through targeted communication and escalation workflows.
- **Nodes Involved:**
  - `Is Churn Risk?1`
  - `Is High Risk?1`

- **Node Details:**
  - **`Is Churn Risk?1`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Validates whether the boolean property `is_churn_risk` equals `true`.
    - *Key Expressions/Variables:* `={{ $json.is_churn_risk }}`.
    - *Connections:* Input from `Calculate Churn Risk Score1`; outputs route to `Is High Risk?1`, `Send Check-in Email1`, `Book CS Call1`, and `Create CS Task (ClickUp)1` (note: branches conditionally based on risk profile evaluation).
    - *Edge Cases/Failure Types:* Evaluation failure if the risk scoring script returns an unexpected type.
  - **`Is High Risk?1`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Validates whether the string property `risk_level` equals `"high"`.
    - *Key Expressions/Variables:* `={{ $json.risk_level }}`.
    - *Connections:* Input from `Is Churn Risk?1`; output connects to `Escalate to Account Manager1`.
    - *Edge Cases/Failure Types:* Strict string comparison mismatches due to casing or unexpected data formats.

---

#### 2.4 Multi-Channel Action & Audit Logging
- **Overview:** Executes external notifications, task creations, and calendar bookings for at-risk accounts, then logs the intervention metadata back into the database.
- **Nodes Involved:**
  - `Send Check-in Email1`
  - `Book CS Call1`
  - `Create CS Task (ClickUp)1`
  - `Escalate to Account Manager1`
  - `Log Intervention1`

- **Node Details:**
  - **`Send Check-in Email1`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Communication Integration)
    - *Configuration Choices:* Sends an HTML-formatted check-in email using Gmail OAuth2 credentials.
    - *Key Expressions/Variables:* 
      - Recipient: `={{ $json.contact_email }}`
      - Subject: `={{ 'Checking in, ' + $json.account_name }}`
      - HTML body referencing `contact_first_name`, `account_name`, and `booking_link`.
    - *Connections:* Input from `Is Churn Risk?1`; output connects to `Log Intervention1`.
    - *Edge Cases/Failure Types:* Invalid Gmail OAuth2 tokens, missing email addresses, or SMTP rate limits.
  - **`Book CS Call1`**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Sends a POST request to Calendly's API endpoint (`https://api.calendly.com/scheduled_events`) using predefined credentials.
    - *Key Expressions/Variables:* JSON payload specifying `event_type` (`https://api.calendly.com/event_types/YOUR_CS_CALL_EVENT_TYPE_ID`), invitee details, and `customer_id`.
    - *Connections:* Input from `Is Churn Risk?1`.
    - *Edge Cases/Failure Types:* Invalid Calendly API credentials or expired event type IDs.
  - **`Create CS Task (ClickUp)1`**
    - *Type and Technical Role:* `n8n-nodes-base.clickUp` (Task Management)
    - *Configuration Choices:* Creates a task in a designated ClickUp list based on account risk severity.
    - *Key Expressions/Variables:* 
      - List ID: `YOUR_CS_LIST_ID`
      - Task Name: `=Churn risk follow-up: {{$json.account_name}} ({{$json.risk_level}})`
      - Priority: Evaluates `={{ $json.risk_level === 'high' ? 1 : 2 }}`.
    - *Connections:* Input from `Is Churn Risk?1`.
    - *Edge Cases/Failure Types:* Invalid ClickUp API tokens or unrecognised list IDs.
  - **`Escalate to Account Manager1`**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
    - *Configuration Choices:* Posts a formatted alert message to a Slack channel (`#account-escalations`) using Slack OAuth2 credentials.
    - *Key Expressions/Variables:* Text payload interpolating account metadata, churn score, reasons array mapped to bullet points, and user tag `<@ACCOUNT_MANAGER_SLACK_ID>`.
    - *Connections:* Input from `Is High Risk?1`.
    - *Edge Cases/Failure Types:* Expired Slack OAuth scopes or invalid channel names/IDs.
  - **`Log Intervention1`**
    - *Type and Technical Role:* `n8n-nodes-base.postgres` (Database Operation)
    - *Configuration Choices:* Executes an SQL insert statement via `executeQuery` into the `churn_intervention_log` table.
    - *Key Expressions/Variables:* SQL query string interpolating `customer_id`, `churn_score`, `risk_level`, `JSON.stringify($json.churn_reasons)`, and timestamp.
    - *Connections:* Input from `Send Check-in Email1`.
    - *Edge Cases/Failure Types:* Database constraint violations or syntax errors in the JSON stringification of reasons.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every 6 Hours1` | `scheduleTrigger` | Triggers workflow execution every 6 hours | None | `Get Product Usage (DB)1`, `Get Mixpanel Engagement1`, `Get Latest NPS Scores1`, `Get Recent Support Tickets1` | 📉 Customer Churn Risk Scoring Agent – Automated CS Intervention<br><br>### How it works<br>Every 6 hours, this workflow pulls signals from four data sources in parallel... |
| `Get Product Usage (DB)1` | `postgres` | Queries product usage and last-login data from Postgres | `Every 6 Hours1` | `Merge Usage + Mixpanel1` | 📉 Customer Churn Risk Scoring Agent – Automated CS Intervention<br><br>### How it works<br>Every 6 hours, this workflow pulls signals from four data sources in parallel... |
| `Get Mixpanel Engagement1` | `httpRequest` | Fetches user engagement telemetry from Mixpanel | `Every 6 Hours1` | `Merge Usage + Mixpanel1` | 📉 Customer Churn Risk Scoring Agent – Automated CS Intervention<br><br>### How it works<br>Every 6 hours, this workflow pulls signals from four data sources in parallel... |
| `Get Latest NPS Scores1` | `postgres` | Queries recent NPS survey responses from Postgres | `Every 6 Hours1` | `Merge + NPS1` | 📉 Customer Churn Risk Scoring Agent – Automated CS Intervention<br><br>### How it works<br>Every 6 hours, this workflow pulls signals from four data sources in parallel... |
| `Get Recent Support Tickets1` | `zendesk` | Retrieves support tickets from Zendesk | `Every 6 Hours1` | `Merge + Tickets1` | 📉 Customer Churn Risk Scoring Agent – Automated CS Intervention<br><br>### How it works<br>Every 6 hours, this workflow pulls signals from four data sources in parallel... |
| `Merge Usage + Mixpanel1` | `merge` | Combines database usage data with Mixpanel engagement data | `Get Product Usage (DB)1`, `Get Mixpanel Engagement1` | `Merge + NPS1` | 🧮 Signal Merge & Risk Score Calculation<br><br>Three Merge nodes progressively join the four data streams into a single per-customer object... |
| `Merge + NPS1` | `merge` | Merges NPS data into the unified customer record | `Merge Usage + Mixpanel1`, `Get Latest NPS Scores1` | `Merge + Tickets1` | 🧮 Signal Merge & Risk Score Calculation<br><br>Three Merge nodes progressively join the four data streams into a single per-customer object... |
| `Merge + Tickets1` | `merge` | Merges support ticket signals into the unified customer record | `Merge + NPS1`, `Get Recent Support Tickets1` | `Calculate Churn Risk Score1` | 🧮 Signal Merge & Risk Score Calculation<br><br>Three Merge nodes progressively join the four data streams into a single per-customer object... |
| `Calculate Churn Risk Score1` | `code` | Computes composite churn risk score, risk level, and reasons | `Merge + Tickets1` | `Is Churn Risk?1` | 🧮 Signal Merge & Risk Score Calculation<br><br>Three Merge nodes progressively join the four data streams into a single per-customer object... |
| `Is Churn Risk?1` | `if` | Conditional router checking if customer meets churn risk threshold | `Calculate Churn Risk Score1` | `Is High Risk?1`, `Send Check-in Email1`, `Book CS Call1`, `Create CS Task (ClickUp)1` | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Is High Risk?1` | `if` | Conditional router checking if customer risk level is high | `Is Churn Risk?1` | `Escalate to Account Manager1` | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Send Check-in Email1` | `gmail` | Sends a personalized check-in email via Gmail | `Is Churn Risk?1` | `Log Intervention1` | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Book CS Call1` | `httpRequest` | Schedules a Calendly event for at-risk customers | `Is Churn Risk?1` | None | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Create CS Task (ClickUp)1` | `clickUp` | Creates a CS task in ClickUp for account follow-up | `Is Churn Risk?1` | None | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Escalate to Account Manager1` | `slack` | Posts high-risk escalation messages to a Slack channel | `Is High Risk?1` | None | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |
| `Log Intervention1` | `postgres` | Inserts intervention audit details into Postgres database | `Send Check-in Email1` | None | 🚨 Risk-Based CS Intervention Routing<br><br>Two IF nodes gate the intervention level. Customers below the risk threshold are skipped... |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node. Set the interval to fire every 6 hours. Name it `Every 6 Hours1`.
2. **Create the Parallel Data Collection Nodes:**
   - Add a **Postgres** node (`Get Product Usage (DB)1`). Set operation to `Execute Query` and provide query: `SELECT customer_id, account_name, plan, last_login_at, feature_usage_7d, feature_usage_30d, seats_active, seats_purchased, mrr, renewal_date FROM customer_usage_snapshot WHERE updated_at >= NOW() - INTERVAL '6 hours';`. Configure Postgres credentials.
   - Add an **HTTP Request** node (`Get Mixpanel Engagement1`). Set Method to `POST`, URL to `https://mixpanel.com/api/2.0/engage`, body type to JSON, and supply payload: `{"project_id": "YOUR_MIXPANEL_PROJECT_ID", "output_properties": ["$last_seen", "total_events_30d", "key_feature_used_30d"]}`. Configure Mixpanel API credentials.
   - Add a **Postgres** node (`Get Latest NPS Scores1`). Set operation to `Execute Query` and provide query: `SELECT customer_id, nps_score, survey_date, verbatim_comment FROM nps_responses WHERE survey_date >= NOW() - INTERVAL '30 days';`. Use Postgres credentials.
   - Add a **Zendesk** node (`Get Recent Support Tickets1`). Set operation to `Get All` with sort options by `created_at` descending. Configure Zendesk credentials.
   - **Connections:** Connect `Every 6 Hours1` output to all four nodes.
3. **Create the Data Unification & Scoring Nodes:**
   - Add a **Merge** node (`Merge Usage + Mixpanel1`). Set mode to `Combine`, join mode to `Keep Everything`, and fields to match string: `customer_id`. Connect `Get Product Usage (DB)1` to Input 0 and `Get Mixpanel Engagement1` to Input 1.
   - Add a **Merge** node (`Merge + NPS1`). Set mode to `Combine`, join mode to `Keep Everything`, and fields to match string: `customer_id`. Connect `Merge Usage + Mixpanel1` to Input 0 and `Get Latest NPS Scores1` to Input 1.
   - Add a **Merge** node (`Merge + Tickets1`). Set mode to `Combine`, join mode to `Keep Everything`, and fields to match string: `customer_id`. Connect `Merge + NPS1` to Input 0 and `Get Recent Support Tickets1` to Input 1.
   - Add a **Code** node (`Calculate Churn Risk Score1`). Insert the JavaScript logic provided in the reference configuration to compute scores and output `churn_score`, `risk_level`, `churn_reasons`, and `is_churn_risk`. Connect `Merge + Tickets1` output to this node.
4. **Create the Conditional Routing Nodes:**
   - Add an **If** node (`Is Churn Risk?1`). Set condition to check if `{{ $json.is_churn_risk }}` is `true`. Connect `Calculate Churn Risk Score1` output to this node.
   - Add an **If** node (`Is High Risk?1`). Set condition to check if `{{ $json.risk_level }}` equals `high`. Connect the true branch of `Is Churn Risk?1` to this node.
5. **Create the Action & Logging Nodes:**
   - Add a **Gmail** node (`Send Check-in Email1`). Set recipient to `{{ $json.contact_email }}`, subject expression, and HTML body. Configure Gmail OAuth2 credentials. Connect the true output of `Is Churn Risk?1` here.
   - Add an **HTTP Request** node (`Book CS Call1`). Set Method to `POST`, URL to `https://api.calendly.com/scheduled_events`, body type to JSON with event payload referencing `YOUR_CS_CALL_EVENT_TYPE_ID`. Configure Calendly API credentials. Connect the true output of `Is Churn Risk?1` here.
   - Add a **ClickUp** node (`Create CS Task (ClickUp)1`). Configure list ID `YOUR_CS_LIST_ID`, task name expression, tags, content, and priority expression. Configure ClickUp API credentials. Connect the true output of `Is Churn Risk?1` here.
   - Add a **Slack** node (`Escalate to Account Manager1`). Configure message text expression, select `channel`, and set channel ID to `#account-escalations`. Configure Slack OAuth2 credentials. Connect the output of `Is High Risk?1` here.
   - Add a **Postgres** node (`Log Intervention1`). Set operation to `Execute Query` with the `INSERT INTO churn_intervention_log ...` SQL command. Connect the output of `Send Check-in Email1` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Required Credentials List | Postgres, Mixpanel API, Zendesk API, Gmail OAuth2, Calendly API, ClickUp API, Slack OAuth2 |
| Configuration Placeholders | Replace placeholders such as `YOUR_MIXPANEL_PROJECT_ID`, `YOUR_CS_CALL_EVENT_TYPE_ID`, `YOUR_CS_LIST_ID`, and `ACCOUNT_MANAGER_SLACK_ID` with production identifiers before activation. |