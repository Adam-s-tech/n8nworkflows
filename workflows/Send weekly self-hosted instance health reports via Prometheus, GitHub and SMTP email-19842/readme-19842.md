Send weekly self-hosted instance health reports via Prometheus, GitHub and SMTP email

https://n8nworkflows.xyz/workflows/send-weekly-self-hosted-instance-health-reports-via-prometheus--github-and-smtp-email-19842


# Send weekly self-hosted instance health reports via Prometheus, GitHub and SMTP email

### 1. Workflow Overview

This workflow automates weekly health monitoring for a self-hosted n8n instance. Its primary purpose is to consolidate instance metrics, security audit findings, GitHub release data, and recent execution failures into a single HTML and plain-text report delivered via email. Target use cases include system administration, proactive security updates, and error tracking for self-hosted production environments.

The workflow logic is grouped into three sequential functional blocks:
- **1.1 Input Reception & Configuration:** Manages scheduled or manual triggers and defines baseline parameters (such as instance URLs, look-back windows, and email addresses).
- **1.2 Instance Data Collection:** Fetches local Prometheus metrics, executes an n8n internal security audit, queries the GitHub Releases API for stable n8n versions, and extracts failed executions and workflow names via the n8n public API.
- **1.3 Report Processing & Delivery:** Aggregates and transforms the collected datasets using custom JavaScript to build a structured health summary, then dispatches the report via SMTP email.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

##### Overview
This block initiates the reporting sequence either automatically every Monday morning or on-demand, establishing the environmental and configuration parameters required by downstream nodes.

##### Nodes Involved
- `Every Monday at 8am` (Schedule Trigger)
- `Run report now` (Manual Trigger)
- `Set report settings` (Set / Edit Fields)

##### Node Details

###### `Every Monday at 8am`
- **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Starts the workflow execution on a weekly schedule.
- **Configuration Choices:** Configured to trigger every week on Day 1 (Monday) at 08:00 AM.
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: None (Root node). Output: Connects to `Set report settings`.
- **Version-specific Requirements:** TypeVersion 1.2.
- **Edge Cases & Failure Types:** Missed executions if the n8n instance is offline at the scheduled time.

###### `Run report now`
- **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Trigger Node). Allows on-demand execution of the workflow.
- **Configuration Choices:** Default settings (no parameters required).
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: None (Root node). Output: Connects to `Set report settings`.
- **Version-specific Requirements:** TypeVersion 1.0.
- **Edge Cases & Failure Types:** None.

###### `Set report settings`
- **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Assigns configuration variables used throughout the pipeline.
- **Configuration Choices:** Defines five string/number fields: `instanceUrl`, `metricsUrl`, `lookbackDays`, `reportTo`, and `reportFrom`.
- **Key Expressions/Variables:** 
  - `instanceUrl`: `https://n8n.example.com`
  - `metricsUrl`: `http://localhost:5678/metrics`
  - `lookbackDays`: `7`
  - `reportTo`: `you@example.com`
  - `reportFrom`: `user@example.com`
- **Input/Output Connections:** Input: `Every Monday at 8am` or `Run report now`. Output: Connects to `Read version from metrics`.
- **Version-specific Requirements:** TypeVersion 3.4.
- **Edge Cases & Failure Types:** Invalid URL strings or malformed email addresses may cause failures in downstream HTTP or SMTP nodes.

---

#### Block 1.2: Instance Data Collection

##### Overview
This block queries multiple internal and external data sources sequentially. It reads instance Prometheus metrics, runs a security audit, pulls upstream GitHub release metadata, and queries the local n8n API for failed workflow executions.

##### Nodes Involved
- `Read version from metrics` (HTTP Request)
- `Run security audit` (n8n Internal API)
- `Get n8n releases from GitHub` (HTTP Request)
- `Get failed executions` (n8n Internal API)
- `Get workflow names` (n8n Internal API)

##### Node Details

###### `Read version from metrics`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Fetches Prometheus text-format metrics from the local n8n instance.
- **Configuration Choices:** Target URL points to `={{ $json.metricsUrl }}`. Timeout set to 10,000ms. Response format set to text.
- **Key Expressions/Variables:** `={{ $json.metricsUrl }}` (references data from `Set report settings`).
- **Input/Output Connections:** Input: `Set report settings`. Output: Connects to `Run security audit`.
- **Version-specific Requirements:** TypeVersion 4.2.
- **Edge Cases & Failure Types:** `onError` is set to `continueRegularOutput`. If `N8N_METRICS=true` is missing or the endpoint is unreachable, the node outputs empty data, and the subsequent script falls back to parsing version information from the security audit.

###### `Run security audit`
- **Type & Technical Role:** `n8n-nodes-base.n8n` (Internal Integration Node). Executes an n8n instance security audit.
- **Configuration Choices:** Resource: `audit`, Operation: `generate`, Categories: `instance`. Configured with `executeOnce: true` and `alwaysOutputData: true`.
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: `Read version from metrics`. Output: Connects to `Get n8n releases from GitHub`.
- **Version-specific Requirements:** TypeVersion 1. Requires valid n8n API credentials configured at the instance level.
- **Edge Cases & Failure Types:** `onError` is set to `continueRegularOutput`. Authentication failures occur if instance-level API keys are missing or misconfigured.

###### `Get n8n releases from GitHub`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Network Request Node). Retrieves stable release data from the official n8n GitHub repository.
- **Configuration Choices:** GET request to `https://api.github.com/repos/n8n-io/n8n/releases` with query parameter `per_page=100` and header `Accept: application/vnd.github+json`. Timeout: 20,000ms. `executeOnce: true`, `alwaysOutputData: true`.
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: `Run security audit`. Output: Connects to `Get failed executions`.
- **Version-specific Requirements:** TypeVersion 4.2.
- **Edge Cases & Failure Types:** GitHub API rate limiting (unauthenticated requests allow 60 requests per hour). Timeouts if GitHub is unreachable.

###### `Get failed executions`
- **Type & Technical Role:** `n8n-nodes-base.n8n` (Internal Integration Node). Queries the local n8n instance API to retrieve failed job executions.
- **Configuration Choices:** Resource: `execution`, Operation: `getAll`. Filter status set to `error`. Limit set to `250`. `executeOnce: true`, `alwaysOutputData: true`.
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: `Get n8n releases from GitHub`. Output: Connects to `Get workflow names`.
- **Version-specific Requirements:** TypeVersion 1. Requires valid n8n API credentials.
- **Edge Cases & Failure Types:** Returns up to 250 records; instances with high failure volumes will only report the most recent 250 errors.

###### `Get workflow names`
- **Type & Technical Role:** `n8n-nodes-base.n8n` (Internal Integration Node). Retrieves all workflow identifiers and display names from the local instance.
- **Configuration Choices:** Resource: `workflow`, Operation: `getAll`, Return All: `true`. `executeOnce: true`, `alwaysOutputData: true`.
- **Key Expressions/Variables:** None.
- **Input/Output Connections:** Input: `Get failed executions`. Output: Connects to `Build health report`.
- **Version-specific Requirements:** TypeVersion 1. Requires valid n8n API credentials.
- **Edge Cases & Failure Types:** Authentication failures or timeouts on instances containing thousands of workflows.

---

#### Block 1.3: Report Processing & Delivery

##### Overview
This block aggregates all upstream data payloads via a JavaScript processing node, formats the results into an HTML and plain-text health report, and transmits the email using configured SMTP credentials.

##### Nodes Involved
- `Build health report` (Code)
- `Send report by email` (Email / SMTP)

##### Node Details

###### `Build health report`
- **Type & Technical Role:** `n8n-nodes-base.code` (Data Processing Node). Executes JavaScript to normalize versions, parse security audit flags, group failed executions within the look-back window, and compile clean HTML and plain-text report payloads.
- **Configuration Choices:** Mode: `runOnceForAllItems`. JavaScript execution processes node outputs using expression helpers (`$('Node Name')`).
- **Key Expressions/Variables:** 
  - Reads settings from `$('Set report settings')`
  - Reads Prometheus data from `$('Read version from metrics')`
  - Reads audit outputs from `$('Run security audit')`
  - Reads GitHub releases from `$('Get n8n releases from GitHub')`
  - Reads workflow names and executions from `$('Get workflow names')` and `$('Get failed executions')`
- **Input/Output Connections:** Input: `Get workflow names`. Output: Connects to `Send report by email`.
- **Version-specific Requirements:** TypeVersion 2.
- **Edge Cases & Failure Types:** JavaScript runtime errors if expected JSON structures from Prometheus, GitHub, or internal APIs deviate from anticipated schemas.

###### `Send report by email`
- **Type & Technical Role:** `n8n-nodes-base.emailSend` (Communication Node). Sends the formatted health report via SMTP.
- **Configuration Choices:** Email Format set to `html`. Appends no attribution.
- **Key Expressions/Variables:**
  - `toEmail`: `={{ $('Set report settings').first().json.reportTo }}`
  - `fromEmail`: `={{ $('Set report settings').first().json.reportFrom }}`
  - `subject`: `={{ $json.subject }}`
  - `html`: `={{ $json.html }}`
- **Input/Output Connections:** Input: `Build health report`. Output: None (Terminal node).
- **Version-specific Requirements:** TypeVersion 2.1. Requires valid SMTP credentials.
- **Edge Cases & Failure Types:** SMTP authentication errors, network timeouts, or rejection by downstream mail transfer agents (MTAs) due to sender policy misconfigurations (SPF/DKIM).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - Overview` | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Weekly health report for your self-hosted n8n instance<br><br>Once a week, this workflow checks which n8n version you are running against the latest releases on GitHub, runs n8n's built-in security audit, and summarises every failed execution from the past week. You get one email with what needs attention.<br><br>Self-hosted n8n gets frequent security fixes, and they are often backported to older release lines. This report tells you when your line has a newer patch, not just when a new major version is out.<br><br>### How it works<br>1. Reads the running version from the instance's Prometheus metrics endpoint.<br>2. Runs the security audit (instance category), which also flags missing updates and security fixes.<br>3. Pulls recent stable releases from the n8n GitHub repository.<br>4. Lists failed executions through the n8n API and groups them by workflow.<br>5. Builds an HTML report and emails it.<br><br>### Setup<br>1. Set `N8N_METRICS=true` on the instance and restart it (see the red note).<br>2. Create an n8n API key (Settings > n8n API) and an **n8n API** credential with base URL `https://your-n8n-host/api/v1`. Select it in the three n8n nodes.<br>3. Create an **SMTP** credential and select it in **Send report by email**.<br>4. Fill in **Set report settings**: your public n8n URL, recipients and look-back window.<br>5. Run it once manually, then activate it.<br><br>### Customization tips<br>- Replace the email node with Slack or Microsoft Teams; the report is also available as plain text in `summary`.<br>- Change the schedule to daily if you run many production workflows. |
| `Sticky Note - Settings` | `n8n-nodes-base.stickyNote` | Section grouping for configuration | None | None | ## 1. Schedule and settings<br>Runs every Monday; edit your URL, recipients and look-back window here. |
| `Sticky Note - Collect` | `n8n-nodes-base.stickyNote` | Section grouping for data collection | None | None | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names. |
| `Sticky Note - Metrics` | `n8n-nodes-base.stickyNote` | Configuration warning for Prometheus metrics | None | None | ### Needs N8N_METRICS=true<br>Add `N8N_METRICS=true` to the n8n environment and restart. The default URL reads metrics over localhost, so `/metrics` does not need to be public; block it at your reverse proxy if it is. Without it, the version comes from the security audit when you are behind. |
| `Sticky Note - Report` | `n8n-nodes-base.stickyNote` | Section grouping for reporting and delivery | None | None | ## 3. Build and send<br>One HTML email with version status, audit findings and failures by workflow. |
| `Every Monday at 8am` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow weekly on Mondays at 8 AM | None | `Set report settings` | |
| `Run report now` | `n8n-nodes-base.manualTrigger` | Manually triggers the workflow on demand | None | `Set report settings` | |
| `Set report settings` | `n8n-nodes-base.set` | Assigns configuration variables (URLs, lookback window, emails) | `Every Monday at 8am`, `Run report now` | `Read version from metrics` | ## 1. Schedule and settings<br>Runs every Monday; edit your URL, recipients and look-back window here. |
| `Read version from metrics` | `n8n-nodes-base.httpRequest` | Fetches Prometheus metrics endpoint data | `Set report settings` | `Run security audit` | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names.<br>### Needs N8N_METRICS=true<br>Add `N8N_METRICS=true` to the n8n environment and restart. The default URL reads metrics over localhost, so `/metrics` does not need to be public; block it at your reverse proxy if it is. Without it, the version comes from the security audit when you are behind. |
| `Run security audit` | `n8n-nodes-base.n8n` | Generates an instance security audit report | `Read version from metrics` | `Get n8n releases from GitHub` | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names. |
| `Get n8n releases from GitHub` | `n8n-nodes-base.httpRequest` | Fetches stable release data from GitHub API | `Run security audit` | `Get failed executions` | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names. |
| `Get failed executions` | `n8n-nodes-base.n8n` | Retrieves failed execution records from the n8n API | `Get n8n releases from GitHub` | `Get workflow names` | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names. |
| `Get workflow names` | `n8n-nodes-base.n8n` | Retrieves all workflow names and IDs from the n8n API | `Get failed executions` | `Build health report` | ## 2. Collect instance data<br>Version, security audit, GitHub releases, failed executions and workflow names. |
| `Build health report` | `n8n-nodes-base.code` | Aggregates data and builds HTML/plain-text report payloads | `Get workflow names` | `Send report by email` | ## 3. Build and send<br>One HTML email with version status, audit findings and failures by workflow. |
| `Send report by email` | `n8n-nodes-base.emailSend` | Sends the compiled health report via SMTP | `Build health report` | None | ## 3. Build and send<br>One HTML email with version status, audit findings and failures by workflow. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** (`Every Monday at 8am`). Set interval parameters to weeks, trigger day to Monday, and hour to 8.
   - Add a **Manual Trigger** (`Run report now`) with default parameters.

2. **Configure Settings Node:**
   - Add a **Set** node (`Set report settings`). Connect both trigger nodes to its input.
   - Add the following assignments in the node parameters:
     - `instanceUrl` (String): `https://n8n.example.com`
     - `metricsUrl` (String): `http://localhost:5678/metrics`
     - `lookbackDays` (Number): `7`
     - `reportTo` (String): `you@example.com`
     - `reportFrom` (String): `user@example.com`

3. **Configure Metrics Reading:**
   - Add an **HTTP Request** node (`Read version from metrics`). Connect `Set report settings` to it.
   - Set Method to `GET`, URL to `={{ $json.metricsUrl }}`, timeout to `10000`, response format to `Text`, and output property name to `data`.
   - In node settings, enable **On Error: Continue Regular Output**.

4. **Configure Security Audit:**
   - Add an **n8n** node (`Run security audit`). Connect `Read version from metrics` to it.
   - Set Resource to `Audit`, Operation to `Generate`, and Category to `instance`. Enable `Execute Once` and `Always Output Data`.
   - Configure n8n API credentials pointing to your base URL (`https://your-n8n-host/api/v1`).
   - Enable **On Error: Continue Regular Output**.

5. **Configure GitHub Release Fetching:**
   - Add an **HTTP Request** node (`Get n8n releases from GitHub`). Connect `Run security audit` to it.
   - Set Method to `GET`, URL to `https://api.github.com/repos/n8n-io/n8n/releases`.
   - Add query parameter `per_page` = `100`. Add header `Accept` = `application/vnd.github+json`. Set timeout to `20000`. Enable `Execute Once` and `Always Output Data`.

6. **Configure Execution Retrieval:**
   - Add an **n8n** node (`Get failed executions`). Connect `Get n8n releases from GitHub` to it.
   - Set Resource to `Execution`, Operation to `Get Many` (or `Get All`), filter status to `error`, and limit to `250`. Enable `Execute Once` and `Always Output Data`. Use the same n8n API credential.

7. **Configure Workflow Name Retrieval:**
   - Add an **n8n** node (`Get workflow names`). Connect `Get failed executions` to it.
   - Set Resource to `Workflow`, Operation to `Get Many` (or `Get All`), and Return All to `true`. Enable `Execute Once` and `Always Output Data`. Use the same n8n API credential.

8. **Configure Report Builder Code Node:**
   - Add a **Code** node (`Build health report`). Connect `Get workflow names` to it.
   - Set Mode to `Run Once for All Items` and paste the JavaScript aggregation logic provided in the reference implementation to process inputs from previous nodes and construct the `subject`, `summary`, and `html` variables.

9. **Configure Email Dispatch Node:**
   - Add an **Email Send** node (`Send report by email`). Connect `Build health report` to it.
   - Set Email Format to `html`.
   - Map parameters:
     - `To Email`: `={{ $('Set report settings').first().json.reportTo }}`
     - `From Email`: `={{ $('Set report settings').first().json.reportFrom }}`
     - `Subject`: `={{ $json.subject }}`
     - `Html`: `={{ $json.html }}`
   - Configure valid **SMTP credentials** (Host, Port, User, Password, SSL/TLS settings).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Environment Variable Requirement | Requires `N8N_METRICS=true` configured in the self-hosted n8n environment to expose Prometheus metrics. |
| API Authentication Requirement | Requires a valid n8n API key configured as an n8n API credential to access audit, execution, and workflow endpoints. |