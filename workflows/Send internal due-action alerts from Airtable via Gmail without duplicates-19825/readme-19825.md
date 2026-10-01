Send internal due-action alerts from Airtable via Gmail without duplicates

https://n8nworkflows.xyz/workflows/send-internal-due-action-alerts-from-airtable-via-gmail-without-duplicates-19825


# Send internal due-action alerts from Airtable via Gmail without duplicates

### 1. Workflow Overview

This workflow automates the process of checking an Airtable database every 10 minutes for tasks or actions that have reached or passed their scheduled due times ("Next Action At"). When overdue items are found—excluding statuses such as Done, Closed, Rejected, or Suppressed—the workflow checks a stateful deduplication cache to ensure notifications are not sent repeatedly for the same action. If new due items exist, it compiles them into a clean, time-ordered summary message and sends an internal notification email via Gmail.

The workflow logic is categorized into three functional blocks:
- **1.1 Workflow Initiation:** Triggers the pipeline either automatically on a 10-minute interval or manually for testing.
- **1.2 Data Retrieval:** Queries Airtable using a formula filter to pull only relevant, active, and overdue records.
- **1.3 Deduplication & Notification Processing:** Executes custom JavaScript logic to filter out previously alerted items, formats the output into an ordered list, and sends an alert email through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Initiation
- **Overview:** This block acts as the entry point for the automation, allowing both routine polling and on-demand testing.
- **Nodes Involved:** 
  - `Every 10 Minutes Trigger`
  - `Manual Execution Trigger`
- **Node Details:**
  - **Every 10 Minutes Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.3) — Automatically fires the workflow at a regular interval.
    - *Configuration Choices:* Configured with an interval rule set to execute every 10 minutes.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None; Output: Connected to `Fetch Due Airtable Records`.
    - *Edge Cases / Failure Types:* Timezone misconfigurations or missed intervals during n8n downtime.
  - **Manual Execution Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) — Allows users to manually execute the workflow from the n8n UI for testing or debugging.
    - *Configuration Choices:* Default configuration.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: None; Output: Connected to `Fetch Due Airtable Records`.
    - *Edge Cases / Failure Types:* None.

#### 2.2 Data Retrieval
- **Overview:** Queries the target Airtable base and table to fetch records meeting the overdue criteria.
- **Nodes Involved:**
  - `Fetch Due Airtable Records`
- **Node Details:**
  - **Fetch Due Airtable Records**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (v2.1) — Connects to Airtable to search and retrieve specific database records.
    - *Configuration Choices:* Uses operation `search` with placeholders `YOUR_AIRTABLE_BASE_ID` and `YOUR_TABLE_ID`. Requests specific fields: `Company`, `Status`, `Next Action`, and `Next Action At`. Applies a formula filter: `AND({Next Action At} != BLANK(), {Next Action At} <= NOW(), NOT(OR({Status}='Done',{Status}='Closed',{Status}='Rejected',{Status}='Suppressed')))`
    - *Key Expressions or Variables:* None (relies on Airtable's formula parser).
    - *Input and Output Connections:* Input: Connected from both triggers; Output: Connected to `Filter Duplicate Alerts`.
    - *Edge Cases / Failure Types:* Authentication errors (invalid API key/token), invalid Base ID or Table ID, or formula syntax rejection by Airtable.

#### 2.3 Deduplication & Notification Processing
- **Overview:** Processes incoming rows using custom JavaScript to filter duplicate notifications, orders them chronologically, constructs an email payload, and dispatches the alert via Gmail.
- **Nodes Involved:**
  - `Filter Duplicate Alerts`
  - `Send Due Alert Email`
- **Node Details:**
  - **Filter Duplicate Alerts**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — Executes custom JavaScript to process, filter, and format dataset items.
    - *Configuration Choices:* Utilizes workflow static data (`$getWorkflowStaticData('global')`) to maintain a `seen` record cache. Generates a unique composite key (`recordId | nextAt | next`) for each item. Implements cache pruning if the tracked keys exceed 1,500 entries. Sorts remaining items chronologically and formats a numbered text summary. Returns an empty array if no new items are found.
    - *Key Expressions or Variables:* Uses `$input.all()` and `$getWorkflowStaticData('global')`.
    - *Input and Output Connections:* Input: Connected from `Fetch Due Airtable Records`; Output: Connected to `Send Due Alert Email`.
    - *Edge Cases / Failure Types:* JavaScript runtime exceptions if expected Airtable fields are entirely missing, or memory bloat if cache pruning logic fails (mitigated by the 1,500-item threshold check).
  - **Send Due Alert Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Sends an email using a connected Gmail account.
    - *Configuration Choices:* Configured to send to recipient `you@example.com`. Maps dynamic subject and message body parameters.
    - *Key Expressions or Variables:* 
      - Subject: `={{ $json.subject }}`
      - Message: `={{ $json.message }}`
    - *Input and Output Connections:* Input: Connected from `Filter Duplicate Alerts`; Output: None (Terminal node).
    - *Edge Cases / Failure Types:* Authentication expiry (OAuth2 token revocation), exceeding Gmail sending quotas, or missing input data if the Code node returns an empty array (which short-circuits execution).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Every 10 Minutes Trigger** | n8n-nodes-base.scheduleTrigger | Automatically polls the workflow every 10 minutes. | None | Fetch Due Airtable Records | ## Start workflow checks<br><br>Contains the scheduled trigger for automatic polling and a manual trigger for test runs. These two entry nodes are stacked together on the left side of the canvas and both start the same Airtable lookup. |
| **Manual Execution Trigger** | n8n-nodes-base.manualTrigger | Allows manual test runs of the workflow. | None | Fetch Due Airtable Records | ## Start workflow checks<br><br>Contains the scheduled trigger for automatic polling and a manual trigger for test runs. These two entry nodes are stacked together on the left side of the canvas and both start the same Airtable lookup. |
| **Fetch Due Airtable Records** | n8n-nodes-base.airtable | Queries Airtable for overdue records matching specific status and date filters. | Every 10 Minutes Trigger, Manual Execution Trigger | Filter Duplicate Alerts | ## Find due records<br><br>Queries Airtable for records that are due and passes the matching items into the notification filtering step. |
| **Filter Duplicate Alerts** | n8n-nodes-base.code | Prevents duplicate alerts using static memory, sorts items, and formats the email body. | Fetch Due Airtable Records | Send Due Alert Email | ## Filter and notify<br><br>Uses custom code to prevent duplicate alerts, then sends the remaining due-action notifications through Gmail. These nodes form the right-side processing and output cluster. |
| **Send Due Alert Email** | n8n-nodes-base.gmail | Sends the compiled summary email to the internal recipient. | Filter Duplicate Alerts | None | ## Filter and notify<br><br>Uses custom code to prevent duplicate alerts, then sends the remaining due-action notifications through Gmail. These nodes form the right-side processing and output cluster. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Triggers:**
   - Add a **Schedule Trigger** node (`Every 10 Minutes Trigger`). Set the interval field to `minutes` and the minutes interval to `10`.
   - Add a **Manual Trigger** node (`Manual Execution Trigger`) and place it alongside the schedule trigger.
2. **Add the Airtable Node:**
   - Create an **Airtable** node (`Fetch Due Airtable Records`).
   - Connect both the Schedule Trigger and Manual Trigger outputs to this node.
   - Configure credentials (Airtable API/Personal Access Token).
   - Set **Resource**: `Record`, **Operation**: `Search`.
   - Provide your target **Base ID** and **Table ID**.
   - In **Options**, add the fields: `Company`, `Status`, `Next Action`, and `Next Action At`.
   - Set the **Filter by Formula** parameter to:  
     `AND({Next Action At} != BLANK(), {Next Action At} <= NOW(), NOT(OR({Status}='Done',{Status}='Closed',{Status}='Rejected',{Status}='Suppressed')))`
3. **Add the Code Node:**
   - Create a **Code** node (`Filter Duplicate Alerts`).
   - Connect the output of `Fetch Due Airtable Records` to this node.
   - Set **Mode** to `Run Once for All Items`.
   - Insert the deduplication JavaScript snippet provided in the source workflow to process items, maintain global static state memory, filter duplicates, sort chronologically, and output structured JSON objects containing `subject` and `message`.
4. **Add the Gmail Node:**
   - Create a **Gmail** node (`Send Due Alert Email`).
   - Connect the output of `Filter Duplicate Alerts` to this node.
   - Configure credentials (Gmail OAuth2).
   - Set **Resource**: `Message`, **Operation**: `Send`.
   - Set **Send To**: `you@example.com` (replace with your target internal email address).
   - Set **Subject** to expression: `={{ $json.subject }}`
   - Set **Message** to expression: `={{ $json.message }}`
5. **Testing and Verification:**
   - Run a manual execution using the `Manual Execution Trigger` with a sample overdue record in Airtable to verify that the Code node formats the message correctly and the Gmail node successfully dispatches the alert without triggering duplicate alerts on immediate re-runs.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Adapted from CLEAR MERIT's Internal / Working Demo operations system. Sanitized to remove credentials and private identifiers. Validated on n8n version 2.39.5. | Internal Operations / System Demo |