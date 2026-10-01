Send tiered overdue invoice reminders with Xero, Gmail, Slack, and Google Sheets

https://n8nworkflows.xyz/workflows/send-tiered-overdue-invoice-reminders-with-xero--gmail--slack--and-google-sheets-20015


# Send tiered overdue invoice reminders with Xero, Gmail, Slack, and Google Sheets

### 1. Workflow Overview

This workflow automates the accounts receivable process by identifying unpaid invoices in Xero, calculating how many days overdue they are, and routing them into tiered reminder sequences (7, 14, or 30 days). It incorporates a Google Sheets logging mechanism to prevent duplicate notifications, dispatches targeted emails via Gmail and alerts via Slack, and records successfully processed reminders back into the audit sheet.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule & Configuration:** Triggers daily and initializes global variables such as reminder thresholds and communication channels.
- **1.2 Data Retrieval & Tier Calculation:** Fetches unpaid invoices from Xero, parses due dates, computes days overdue, and assigns each invoice to a specific tier.
- **1.3 Deduplication & Filtering:** Cross-references Google Sheets to ensure reminders are not sent multiple times for the same invoice and tier.
- **1.4 Tiered Notification & Routing:** Branches eligible invoices to send appropriate emails and Slack messages depending on the assigned tier (7, 14, or 30 days).
- **1.5 Audit Logging:** Merges all notification paths and writes the processed records back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the daily execution cycle and sets global parameters for day thresholds and destination Slack channels.
- **Nodes Involved:** 
  - `When Every Day at 8am`
  - `Set Reminder Configurations`

- **Node Details:**
  - **When Every Day at 8am**
    - *Type and technical role:* `scheduleTrigger` — Triggers the workflow execution based on a time interval.
    - *Configuration choices:* Cron expression set to run daily at 08:00 (`0 8 * * *`).
    - *Input/Output:* Output connects to `Set Reminder Configurations`.
    - *Edge cases:* System downtime during the scheduled minute; n8n queue handling will catch missed executions if configured.
  - **Set Reminder Configurations**
    - *Type and technical role:* `set` — Defines static configuration variables for downstream nodes.
    - *Configuration choices:* Sets `tier1Days` (7), `tier2Days` (14), `tier3Days` (30), `accountOwnerSlackChannel` (`#ar-alerts`), and `escalationSlackChannel` (`#ar-escalations`).
    - *Input/Output:* Input from `When Every Day at 8am`; output to `Fetch Unpaid Invoices in Xero`.
    - *Edge cases:* Missing configuration keys will cause downstream reference errors.

---

#### 2.2 Data Retrieval & Tier Calculation
- **Overview:** Retrieves all unpaid invoices from Xero, parses raw date formats, calculates the exact aging, and filters/categorizes invoices meeting the minimum overdue threshold.
- **Nodes Involved:** 
  - `Fetch Unpaid Invoices in Xero`
  - `Calculate Overdue Days and Tier`

- **Node Details:**
  - **Fetch Unpaid Invoices in Xero**
    - *Type and technical role:* `xero` — Integrates with the Xero API to fetch financial records.
    - *Configuration choices:* Operation set to `getAll` for organization ID `2456a90a-71ac-4df1-a5fb-fb0234672663`. Requires valid Xero OAuth2 credentials.
    - *Input/Output:* Input from `Set Reminder Configurations`; output to `Calculate Overdue Days and Tier`.
    - *Edge cases:* 403 AuthenticationUnsuccessful errors if OAuth tokens expire or if organization permissions are misconfigured.
  - **Calculate Overdue Days and Tier**
    - *Type and technical role:* `code` — Executes custom JavaScript to process and transform raw invoice data.
    - *Configuration choices:* Custom JS handles regex matching for Xero ASP.NET JSON date formats (`/Date(...)//`), computes difference from current time in milliseconds, filters out invoices below `tier1Days`, and outputs a standardized object containing a unique composite key (`invoiceId_tier`).
    - *Key expressions:* References variables from `Set Reminder Configurations` using `$('Set Reminder Configurations').first().json`.
    - *Input/Output:* Input from `Fetch Unpaid Invoices in Xero`; output to `Check Sent Invoices in Sheets`.
    - *Edge cases:* Null `DueDateString` values are skipped; missing contact emails evaluate to `null`.

---

#### 2.3 Deduplication & Filtering
- **Overview:** Checks an external Google Sheets audit log to verify whether a specific invoice and reminder tier combination has already been processed.
- **Nodes Involved:** 
  - `Check Sent Invoices in Sheets`
  - `If Not Previously Sent`

- **Node Details:**
  - **Check Sent Invoices in Sheets**
    - *Type and technical role:* `googleSheets` — Queries a target spreadsheet tab to lookup existing tracking records.
    - *Configuration choices:* Document ID points to the tracking sheet, sheet ID points to tab `Overdue invoice Followup sequence`. `continueOnFail` is enabled to ensure execution doesn't halt if a lookup returns empty or encounters minor issues.
    - *Input/Output:* Input from `Calculate Overdue Days and Tier`; output to `If Not Previously Sent`.
    - *Edge cases:* API rate limits or invalid Google Sheets OAuth credentials.
  - **If Not Previously Sent**
    - *Type and technical role:* `if` — Conditional router based on evaluation parameters.
    - *Configuration choices:* Checks if the evaluated expression (`={{ $json.invoiceId_tier }}`) is empty, confirming no prior reminder has been logged.
    - *Input/Output:* Input from `Check Sent Invoices in Sheets`; output to `Route by Overdue Tier`.
    - *Edge cases:* Strict type validation issues if data structures mutate unexpectedly.

---

#### 2.4 Tiered Notification & Routing
- **Overview:** Evaluates the assigned tier value and routes the invoice down distinct communication paths: 7-day email, 14-day email and channel alert, or 30-day escalation and call task.
- **Nodes Involved:** 
  - `Route by Overdue Tier`
  - `Send Day 7 Reminder Email`
  - `Send Day 14 Reminder Email`
  - `Alert Account Owner on Slack`
  - `Escalate on Slack and Assign Call Task`
  - `Merge Sent Notifications`

- **Node Details:**
  - **Route by Overdue Tier**
    - *Type and technical role:* `switch` — Directs items based on multi-branch numerical conditions.
    - *Configuration choices:* Evaluates tier values equal to `7`, `14`, or `30` referencing `$('Calculate Overdue Days and Tier').item.json.tier`.
    - *Input/Output:* Input from `If Not Previously Sent`; outputs connect to respective notification nodes.
  - **Send Day 7 Reminder Email**
    - *Type and technical role:* `gmail` — Sends customer-facing collection notices via Gmail OAuth2.
    - *Configuration choices:* Recipient set to `contactEmail`, dynamic subject and message body referencing invoice details.
    - *Input/Output:* Input from `Route by Overdue Tier` (Branch 1); output to `Merge Sent Notifications`.
    - *Edge cases:* Fails if `contactEmail` is null or invalid.
  - **Send Day 14 Reminder Email**
    - *Type and technical role:* `gmail` — Sends a firmer second-notice email to the customer.
    - *Configuration choices:* Recipient set to `contactEmail` with second-notice copy.
    - *Input/Output:* Input from `Route by Overdue Tier` (Branch 2); output to `Merge Sent Notifications`.
  - **Alert Account Owner on Slack**
    - *Type and technical role:* `slack` — Posts internal notifications to a designated channel.
    - *Configuration choices:* Target channel ID set to `C0C0UEMPD5E`, posting a warning message detailing 14+ day status.
    - *Input/Output:* Input from `Route by Overdue Tier` (Branch 2); output to `Merge Sent Notifications`.
  - **Escalate on Slack and Assign Call Task**
    - *Type and technical role:* `slack` — Posts high-priority escalation messages with actionable tasks.
    - *Configuration choices:* Target channel ID set to `C0C0UEMPD5E`, includes options to append a link to the workflow execution.
    - *Input/Output:* Input from `Route by Overdue Tier` (Branch 3); output to `Merge Sent Notifications`.
  - **Merge Sent Notifications**
    - *Type and technical role:* `merge` — Combines parallel execution branches back into a single unified stream.
    - *Configuration choices:* Default append/combine mode to aggregate outputs from all notification handlers.
    - *Input/Output:* Inputs from all notification nodes; output to `Mark Invoice as Sent in Sheets`.

---

#### 2.5 Audit Logging
- **Overview:** Writes processed reminder records back to Google Sheets to establish persistence and prevent future duplicate runs.
- **Nodes Involved:** 
  - `Mark Invoice as Sent in Sheets`

- **Node Details:**
  - **Mark Invoice as Sent in Sheets**
    - *Type and technical role:* `googleSheets` — Appends or updates rows in the tracking spreadsheet.
    - *Configuration choices:* Operation set to `appendOrUpdate`, matching columns configured to use `invoiceId_tier` with auto-mapped input data.
    - *Input/Output:* Input from `Merge Sent Notifications`; no downstream output.
    - *Edge cases:* Mismatched column headers in Google Sheets will cause property mapping failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | Xero Overdue Invoice Reminders: This workflow runs on a daily schedule to find unpaid Xero invoices, calculate how many days overdue they are, and assign them to reminder tiers. It checks Google Sheets to avoid sending duplicate reminders, routes eligible invoices by tier, then sends the appropriate Gmail or Slack notification. Completed reminder paths are merged and written back to Google Sheets as sent records. Setup steps: Connect and authorize the Xero credential... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Start and configure: Runs the workflow on a daily cadence and defines reminder thresholds plus the Slack channel used later in the process. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Fetch and classify invoices: Retrieves unpaid invoices from Xero and uses custom code to calculate days overdue and assign each invoice to the correct reminder tier. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Check duplicate sends: Looks up the reminder tracking sheet and only allows invoices that have not already had the relevant reminder sent to continue. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Route by tier: Branches each eligible overdue invoice into the appropriate reminder or escalation path based on its calculated tier. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Send reminders and alerts: Sends the tier-specific customer emails and internal Slack alerts, including escalation messaging and a call task for the most serious overdue cases. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | Log sent reminders: Combines completed reminder paths and records sent reminders back into Google Sheets for future duplicate checks. |
| `When Every Day at 8am` | `n8n-nodes-base.scheduleTrigger` | Daily schedule trigger | None | `Set Reminder Configurations` | |
| `Set Reminder Configurations` | `n8n-nodes-base.set` | Define config parameters | `When Every Day at 8am` | `Fetch Unpaid Invoices in Xero` | |
| `Fetch Unpaid Invoices in Xero` | `n8n-nodes-base.xero` | Retrieve Xero invoices | `Set Reminder Configurations` | `Calculate Overdue Days and Tier` | |
| `Calculate Overdue Days and Tier` | `n8n-nodes-base.code` | Compute age and tiers via JS | `Fetch Unpaid Invoices in Xero` | `Check Sent Invoices in Sheets` | |
| `Check Sent Invoices in Sheets` | `n8n-nodes-base.googleSheets` | Query audit log | `Calculate Overdue Days and Tier` | `If Not Previously Sent` | |
| `If Not Previously Sent` | `n8n-nodes-base.if` | Filter out duplicate sends | `Check Sent Invoices in Sheets` | `Route by Overdue Tier` | |
| `Route by Overdue Tier` | `n8n-nodes-base.switch` | Route items by tier value | `If Not Previously Sent` | `Send Day 7 Reminder Email`, `Send Day 14 Reminder Email`, `Alert Account Owner on Slack`, `Escalate on Slack and Assign Call Task` | |
| `Send Day 7 Reminder Email` | `n8n-nodes-base.gmail` | Send tier 1 customer email | `Route by Overdue Tier` | `Merge Sent Notifications` | |
| `Send Day 14 Reminder Email` | `n8n-nodes-base.gmail` | Send tier 2 customer email | `Route by Overdue Tier` | `Merge Sent Notifications` | |
| `Alert Account Owner on Slack` | `n8n-nodes-base.slack` | Send internal Slack warning | `Route by Overdue Tier` | `Merge Sent Notifications` | |
| `Escalate on Slack and Assign Call Task` | `n8n-nodes-base.slack` | Send escalation & call task | `Route by Overdue Tier` | `Merge Sent Notifications` | |
| `Merge Sent Notifications` | `n8n-nodes-base.merge` | Combine notification branches | `Send Day 7 Reminder Email`, `Send Day 14 Reminder Email`, `Alert Account Owner on Slack`, `Escalate on Slack and Assign Call Task` | `Mark Invoice as Sent in Sheets` | |
| `Mark Invoice as Sent in Sheets` | `n8n-nodes-base.googleSheets` | Log completed reminders | `Merge Sent Notifications` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`), name it `When Every Day at 8am`, and set the interval rule to Cron expression `0 8 * * *`.
2. **Configure Global Variables:** Add a **Set** node (`n8n-nodes-base.set`), name it `Set Reminder Configurations`, and assign parameters: `tier1Days` (number: 7), `tier2Days` (number: 14), `tier3Days` (number: 30), `accountOwnerSlackChannel` (string: `#ar-alerts`), and `escalationSlackChannel` (string: `#ar-escalations`). Connect `When Every Day at 8am` to this node.
3. **Fetch Xero Invoices:** Add a **Xero** node (`n8n-nodes-base.xero`), name it `Fetch Unpaid Invoices in Xero`, set operation to `getAll`, specify your Xero organization ID, and authenticate with Xero OAuth2 credentials. Connect `Set Reminder Configurations` to this node.
4. **Calculate Overdue Metrics:** Add a **Code** node (`n8n-nodes-base.code`), name it `Calculate Overdue Days and Tier`, and insert the JavaScript snippet parsing Xero dates, calculating aging against configuration thresholds, and generating the `invoiceId_tier` string. Connect `Fetch Unpaid Invoices in Xero` to this node.
5. **Lookup Tracking Log:** Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`), name it `Check Sent Invoices in Sheets`, select your target spreadsheet and worksheet tab, and enable `Always Output Data` / `Continue on Fail`. Connect `Calculate Overdue Days and Tier` to this node.
6. **Filter Duplicates:** Add an **If** node (`n8n-nodes-base.if`), name it `If Not Previously Sent`, and set condition to check if `{{ $json.invoiceId_tier }}` is empty. Connect `Check Sent Invoices in Sheets` to this node.
7. **Route by Tier:** Add a **Switch** node (`n8n-nodes-base.switch`), name it `Route by Overdue Tier`, and create three rules checking if tier equals `7`, `14`, or `30` based on the code output. Connect `If Not Previously Sent` to this node.
8. **Configure 7-Day Email:** Add a **Gmail** node (`n8n-nodes-base.gmail`), name it `Send Day 7 Reminder Email`, configure Gmail OAuth2 credentials, set recipient to `contactEmail`, and provide a friendly reminder subject and body. Connect rule 1 of `Route by Overdue Tier` here.
9. **Configure 14-Day Communications:** 
   - Add a **Gmail** node, name it `Send Day 14 Reminder Email`, configure recipient and second-notice text.
   - Add a **Slack** node (`n8n-nodes-base.slack`), name it `Alert Account Owner on Slack`, set authentication credentials, choose target channel ID `C0C0UEMPD5E`, and add warning text. Connect rule 2 of `Route by Overdue Tier` to both nodes.
10. **Configure 30-Day Escalation:** Add a **Slack** node, name it `Escalate on Slack and Assign Call Task`, set target channel ID `C0C0UEMPD5E`, write escalation message text including telephone emoji and call task instructions, and enable workflow execution linking options. Connect rule 3 of `Route by Overdue Tier` to this node.
11. **Merge Execution Branches:** Add a **Merge** node (`n8n-nodes-base.merge`), name it `Merge Sent Notifications`, and connect the outputs of `Send Day 7 Reminder Email`, `Send Day 14 Reminder Email`, `Alert Account Owner on Slack`, and `Escalate on Slack and Assign Call Task` into its inputs.
12. **Audit Logging Step:** Add a **Google Sheets** node, name it `Mark Invoice as Sent in Sheets`, set operation to `appendOrUpdate`, select matching column `invoiceId_tier`, and map input data automatically using Google Sheets OAuth2 credentials. Connect `Merge Sent Notifications` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Website Template Link | https://www.intuz.com/n8n-workflow-automation-templates/ |
| Developer Contact Email | getstarted@intuz.com |
| LinkedIn Profile | https://www.linkedin.com/company/intuz |
| Partner Link / Getting Started | https://n8n.partnerlinks.io/intuz |
| Custom Workflow Automation Inquiry | https://www.intuz.com/get-started/ |