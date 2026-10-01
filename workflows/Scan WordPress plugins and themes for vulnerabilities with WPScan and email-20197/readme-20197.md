Scan WordPress plugins and themes for vulnerabilities with WPScan and email

https://n8nworkflows.xyz/workflows/scan-wordpress-plugins-and-themes-for-vulnerabilities-with-wpscan-and-email-20197


# Scan WordPress plugins and themes for vulnerabilities with WPScan and email

### 1. Workflow Overview

This workflow provides an automated nightly inventory and vulnerability assessment of WordPress plugins, themes, and core components. It interfaces with the WordPress REST API to fetch installed components, checks them against the WPScan vulnerability database while respecting daily API rate limits, logs audit data into an n8n Data Table, and delivers structured email reports regarding security status.

The workflow logic is divided into five functional blocks:
- **1.1 Schedule & Configuration Validation:** Triggers nightly, initializes scan parameters, and performs pre-flight assertions to prevent wasted API budget on invalid settings.
- **1.2 Environment Reconnaissance & Quota Assessment:** Queries the WPScan API status endpoint, retrieves installed WordPress extensions via REST API, and pulls historical check timestamps from internal storage.
- **1.3 Prioritization & Quota Allocation:** Evaluates component age, activation status, and API remaining request quotas to build an execution queue.
- **1.4 Component Vulnerability Iteration:** Sequentially queries the WPScan database for each queued component, filters results against the specific installed version, and upserts check results into the database.
- **1.5 Report Compilation & Dispatch:** Aggregates findings, un-checked items, and errors into a plaintext report, evaluates alert criteria, and transmits the email via SMTP.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Schedule & Configuration Validation

#### Overview
This block initiates the nightly audit sequence, declares base operational settings (such as target site URL and rate limits), and validates input configurations to ensure no API quota is burned due to malformed data.

#### Nodes Involved
- `Every night at 03:30`
- `Scan settings`
- `Check your settings`

#### Node Details

##### Every night at 03:30
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` — Triggers workflow execution on a fixed cron-like schedule.
- **Configuration Choices:** Configured to fire daily at 03:30.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: None (Root trigger). Output: Connects to `Scan settings`.
- **Version-Specific Requirements:** Type version 1.2.
- **Edge Cases / Potential Failure Types:** Fails to execute if the n8n instance is offline or execution queue workers are down during the trigger window.

##### Scan settings
- **Type and Technical Role:** `n8n-nodes-base.set` — Establishes core configuration variables for the execution context.
- **Configuration Choices:** Assigns static parameters: `site_url`, `wp_core_version`, `max_lookups_per_run`, `notify_email`, `notify_from`, and `alert_only_on_findings`.
- **Key Expressions / Variables:** None (Static assignments).
- **Input / Output Connections:** Input: `Every night at 03:30`. Output: Connects to `Check your settings`.
- **Version-Specific Requirements:** Type version 3.4.
- **Edge Cases / Potential Failure Types:** Placeholder domains (e.g., `example.com`) will be caught by downstream validation.

##### Check your settings
- **Type and Technical Role:** `n8n-nodes-base.code` — Executes JavaScript validation logic to verify input parameters before network requests occur.
- **Configuration Choices:** Validates URL schema, checks email addresses for placeholder patterns, confirms limits are positive numbers, and sanitizes input trailing slashes. Throws an explicit `Error` if assertions fail.
- **Key Expressions / Variables:** Reads input item JSON (`$json`).
- **Input / Output Connections:** Input: `Scan settings`. Output: Connects to `Ask WPScan how much quota is left`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Potential Failure Types:** Throws runtime errors if `site_url` is invalid, `notify_email` points to an example domain, or parameters are incorrectly typed, halting execution before consuming API quotas.

---

### Block 1.2: Environment Reconnaissance & Quota Assessment

#### Overview
This block interrogates external APIs and internal storage to determine available API consumption limits, retrieve the target site's installed extension inventory, and collect historical execution data.

#### Nodes Involved
- `Ask WPScan how much quota is left`
- `List the plugins`
- `List the themes`
- `Read the scan history`

#### Node Details

##### Ask WPScan how much quota is left
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Communicates with the WPScan status endpoint to retrieve API usage quotas.
- **Configuration Choices:** Targets `https://wpscan.com/api/v3/status` using Generic Header Authentication (`httpHeaderAuth`).
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Check your settings`. Output: Connects to `List the plugins`.
- **Version-Specific Requirements:** Type version 4.2.
- **Edge Cases / Potential Failure Types:** Authentication token expiration yields a 401 Unauthorized error; network timeouts abort the sequence.

##### List the plugins
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Retrieves all installed plugins from the target WordPress instance via REST API.
- **Configuration Choices:** Resolves URL dynamically from settings, uses HTTP Basic Authentication (`httpBasicAuth`), and appends a custom `User-Agent` header to prevent bot-blocking firewalls.
- **Key Expressions / Variables:** `={{ $('Check your settings').first().json.site_url }}/wp-json/wp/v2/plugins`
- **Input / Output Connections:** Input: `Ask WPScan how much quota is left`. Output: Connects to `List the themes`.
- **Version-Specific Requirements:** Type version 4.2.
- **Edge Cases / Potential Failure Types:** WordPress application passwords with insufficient privileges return 401/403 errors; incorrect base URLs trigger 404 or connection refused errors.

##### List the themes
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Retrieves installed themes from the target WordPress instance via REST API.
- **Configuration Choices:** Resolves endpoint dynamically, uses HTTP Basic Authentication, attaches a custom `User-Agent` header, and is set to `executeOnce`.
- **Key Expressions / Variables:** `={{ $('Check your settings').first().json.site_url }}/wp-json/wp/v2/themes`
- **Input / Output Connections:** Input: `List the plugins`. Output: Connects to `Read the scan history`.
- **Version-Specific Requirements:** Type version 4.2.
- **Edge Cases / Potential Failure Types:** Similar authentication and connectivity considerations as the plugin retrieval node.

##### Read the scan history
- **Type and Technical Role:** `n8n-nodes-base.dataTable` — Reads historical check intervals and state from an internal n8n Data Table.
- **Configuration Choices:** Targets the `wpscan_scan_history` Data Table, operation set to `get`, returns all rows where component is not empty, and configured with `executeOnce`.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `List the themes`. Output: Connects to `Plan the lookups`.
- **Version-Specific Requirements:** Type version 1.
- **Edge Cases / Potential Failure Types:** Fails if the `wpscan_scan_history` table does not exist or lacks the required schema columns.

---

### Block 1.3: Prioritization & Quota Allocation

#### Overview
This block aggregates all discovered components, cross-references their last-checked timestamps, applies active-status weightings, and bounds the workload to fit within remaining API rate limits.

#### Nodes Involved
- `Plan the lookups`
- `Anything to look up?`

#### Node Details

##### Plan the lookups
- **Type and Technical Role:** `n8n-nodes-base.code` — Executes complex sorting, filtering, and budgeting logic in JavaScript.
- **Configuration Choices:** Normalizes plugin and theme arrays, identifies components with missing version numbers, enforces quota limits (the minimum between remaining daily quota and `max_lookups_per_run`), prioritizes active components, and sorts remaining items by oldest check time.
- **Key Expressions / Variables:** Evaluates data flows from `Check your settings`, `Ask WPScan how much quota is left`, `List the themes`, `List the plugins`, and `Read the scan history`.
- **Input / Output Connections:** Input: `Read the scan history`. Output: Connects to `Anything to look up?`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Potential Failure Types:** Emits a sentinel object (`nothingToScan: true`) if no work is found or available quota is zero, preventing branch starvation.

##### Anything to look up?
- **Type and Technical Role:** `n8n-nodes-base.if` — Evaluates operational flow based on scan availability.
- **Configuration Choices:** Evaluates whether `nothingToScan` is true.
- **Key Expressions / Variables:** `={{ $json.nothingToScan !== true }}`
- **Input / Output Connections:** Input: `Plan the lookups`. Outputs: 
  - True branch (Work exists): Connects to `One component at a time`.
  - False branch (No work): Connects directly to `Build the report`.
- **Version-Specific Requirements:** Type version 2.2.
- **Edge Cases / Potential Failure Types:** Misconfigured evaluation logic could bypass reporting when components are exhausted.

---

### Block 1.4: Component Vulnerability Iteration

#### Overview
This block processes items iteratively, queries the WPScan API for each target component, filters raw vulnerability records to isolate those matching the exact installed version, and upserts check results into storage.

#### Nodes Involved
- `One component at a time`
- `Look it up in the vulnerability database`
- `Match against your version`
- `Update the scan history`

#### Node Details

##### One component at a time
- **Type and Technical Role:** `n8n-nodes-base.splitInBatches` — Loops through items sequentially to throttle external API calls.
- **Configuration Choices:** Processes items in batch iterations without resetting context.
- **Key Expressions / Variables:** None.
- **Input / Output Connections:** Input: `Anything to look up?` (True branch). Outputs:
  - Loop iteration output: Connects to `Look it up in the vulnerability database`.
  - Loop completion output: Connects to `Build the report`.
- **Version-Specific Requirements:** Type version 3.
- **Edge Cases / Potential Failure Types:** Infinite loops if batch pointers fail to advance or upstream data arrays mutate mid-iteration.

##### Look it up in the vulnerability database
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` — Queries the WPScan endpoint for specific plugin, theme, or core records.
- **Configuration Choices:** Dynamically constructs endpoints based on component type (`wordpresses`, `themes`, or `plugins`), utilizes Generic Header Authentication (`httpHeaderAuth`), and suppresses failure exceptions (`neverError: true`, `fullResponse: true`).
- **Key Expressions / Variables:** `=https://wpscan.com/api/v3/{{ $json.kind === 'core' ? 'wordpresses' : ($json.kind === 'theme' ? 'themes' : 'plugins') }}/{{ $json.kind === 'core' ? $json.slug.replace(/\./g, '') : $json.slug }}`
- **Input / Output Connections:** Input: `One component at a time`. Output: Connects to `Match against your version`.
- **Version-Specific Requirements:** Type version 4.2.
- **Edge Cases / Potential Failure Types:** Returns status code 404 for custom/commercial plugins absent from the database; HTTP 429 or 401 indicate rate exhaustion or invalid authorization tokens.

##### Match against your version
- **Type and Technical Role:** `n8n-nodes-base.code` — Executes JavaScript version comparison algorithms against raw WPScan vulnerability arrays.
- **Configuration Choices:** Implements strict version parsing (`cmp`), checks introduced and fixed-in version markers, categorizes vulnerability severities into explicit risk bands, and structures the findings payload.
- **Key Expressions / Variables:** Pulls data from `One component at a time` and input HTTP response nodes.
- **Input / Output Connections:** Input: `Look it up in the vulnerability database`. Output: Connects to `Update the scan history`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Potential Failure Types:** Non-standard version strings can produce comparison anomalies if they deviate from semantic versioning structures.

##### Update the scan history
- **Type and Technical Role:** `n8n-nodes-base.dataTable` — Writes or updates check outcomes in the audit Data Table.
- **Configuration Choices:** Targets table `wpscan_scan_history`, operation set to `upsert`, matches rows based on the `component` column.
- **Key Expressions / Variables:** Mappings use incoming JSON properties: `component`, `kind`, `slug`, `version`, `findings`, `last_checked`, and `affected_count`.
- **Input / Output Connections:** Input: `Match against your version`. Output: Loops back to `One component at a time` to continue batch execution.
- **Version-Specific Requirements:** Type version 1.
- **Edge Cases / Potential Failure Types:** Fails if Data Table locks occur or table constraints are violated.

---

### Block 1.5: Report Compilation & Dispatch

#### Overview
This block aggregates all execution metrics, formats a comprehensive plaintext vulnerability and status report, evaluates dispatch rules, and sends notification emails via SMTP.

#### Nodes Involved
- `Build the report`
- `Worth emailing?`
- `Email the report`

#### Node Details

##### Build the report
- **Type and Technical Role:** `n8n-nodes-base.code` — Aggregates all iteration indices, formats statistical tallies, constructs report body strings, and determines email dispatch triggers.
- **Configuration Choices:** Iterates through all loop executions via `.all(0, i)`, separates errored, missing, and successful checks, categorizes severity totals, and sets a boolean `send` flag based on findings, errors, or configuration parameters.
- **Key Expressions / Variables:** Reads execution context from `Check your settings`, `Match against your version`, and `Plan the lookups`.
- **Input / Output Connections:** Inputs: `Anything to look up?` (False branch) and `One component at a time` (Loop completion). Output: Connects to `Worth emailing?`.
- **Version-Specific Requirements:** Type version 2.
- **Edge Cases / Potential Failure Types:** Memory overhead can increase if batch loops contain exceptionally large component lists.

##### Worth emailing?
- **Type and Technical Role:** `n8n-nodes-base.if` — Controls conditional routing for email notifications.
- **Configuration Choices:** Evaluates the `send` evaluation flag produced by the report builder.
- **Key Expressions / Variables:** `={{ $json.send }}`
- **Input / Output Connections:** Input: `Build the report`. Outputs:
  - True branch: Connects to `Email the report`.
  - False branch: Terminates execution path without emailing.
- **Version-Specific Requirements:** Type version 2.2.
- **Edge Cases / Potential Failure Types:** If misconfigured, critical error alerts could be suppressed when `alert_only_on_findings` is mismatched.

##### Email the report
- **Type and Technical Role:** `n8n-nodes-base.emailSend` — Transmits the compiled vulnerability report via SMTP.
- **Configuration Choices:** Configured to send plaintext emails using dynamic subject and body parameters mapped from upstream JSON.
- **Key Expressions / Variables:** 
  - To: `={{ $('Check your settings').first().json.notify_email }}`
  - From: `={{ $('Check your settings').first().json.notify_from }}`
  - Subject: `={{ $json.subject }}`
  - Text: `={{ $json.body }}`
- **Input / Output Connections:** Input: `Worth emailing?` (True branch). Output: None (Terminal node).
- **Version-Specific Requirements:** Type version 2.1.
- **Edge Cases / Potential Failure Types:** SMTP connection timeouts, invalid authentication credentials, or strict sender domain restrictions (`notify_from`) will cause email delivery failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `About this template` | `stickyNote` | Documentation and setup guide | None | None | Scan WordPress plugins and themes for vulnerabilities with WPScan... Full setup guide: [datadrifter.io](https://datadrifter.io/wordpress-plugin-vulnerabilities-wpscan-api/) |
| `Section 1` | `stickyNote` | Section visual grouping | None | None | Settings<br>Your site, report address and lookup limit live in **Scan settings**. The check stops the run before any API quota is spent. |
| `Section 2` | `stickyNote` | Section visual grouping | None | None | What is installed, and the budget<br>Reads today's WPScan quota, your plugins and themes, and when each was last checked. |
| `Section 3` | `stickyNote` | Section visual grouping | None | None | Plan what to check<br>Active components first, then the ones checked longest ago, within the quota. |
| `Section 4` | `stickyNote` | Section visual grouping | None | None | Check each component<br>Looks each one up in WPScan, keeps only issues affecting your installed version and records the check. |
| `Section 5` | `stickyNote` | Section visual grouping | None | None | Report<br>Findings first, then everything that was not checked and why. |
| `Warning: admin-level password` | `stickyNote` | Security advisory | None | None | Admin-level password<br>Listing plugins needs an administrator Application Password. Use one only for this workflow and revoke it when you stop. |
| `Every night at 03:30` | `scheduleTrigger` | Triggers workflow execution nightly | None | `Scan settings` | |
| `Scan settings` | `set` | Declares workflow operational variables | `Every night at 03:30` | `Check your settings` | Settings<br>Your site, report address and lookup limit live in **Scan settings**. The check stops the run before any API quota is spent. |
| `Check your settings` | `code` | Validates configuration parameters | `Scan settings` | `Ask WPScan how much quota is left` | Settings<br>Your site, report address and lookup limit live in **Scan settings**. The check stops the run before any API quota is spent. |
| `Ask WPScan how much quota is left` | `httpRequest` | Fetches remaining WPScan API requests | `Check your settings` | `List the plugins` | What is installed, and the budget<br>Reads today's WPScan quota, your plugins and themes, and when each was last checked. |
| `List the plugins` | `httpRequest` | Retrieves installed plugins from WordPress | `Ask WPScan how much quota is left` | `List the themes` | What is installed, and the budget<br>Reads today's WPScan quota, your plugins and themes, and when each was last checked. |
| `List the themes` | `httpRequest` | Retrieves installed themes from WordPress | `List the plugins` | `Read the scan history` | What is installed, and the budget<br>Reads today's WPScan quota, your plugins and themes, and when each was last checked. |
| `Read the scan history` | `dataTable` | Reads historical scan timestamps | `List the themes` | `Plan the lookups` | What is installed, and the budget<br>Reads today's WPScan quota, your plugins and themes, and when each was last checked. |
| `Plan the lookups` | `code` | Prioritizes and budgets component checks | `Read the scan history` | `Anything to look up?` | Plan what to check<br>Active components first, then the ones checked longest ago, within the quota. |
| `Anything to look up?` | `if` | Checks if work items exist | `Plan the lookups` | `One component at a time`, `Build the report` | Plan what to check<br>Active components first, then the ones checked longest ago, within the quota. |
| `One component at a time` | `splitInBatches` | Loops through components sequentially | `Anything to look up?` | `Build the report`, `Look it up in the vulnerability database` | Check each component<br>Looks each one up in WPScan, keeps only issues affecting your installed version and records the check. |
| `Look it up in the vulnerability database` | `httpRequest` | Queries WPScan vulnerability database | `One component at a time` | `Match against your version` | Check each component<br>Looks each one up in WPScan, keeps only issues affecting your installed version and records the check. |
| `Match against your version` | `code` | Filters vulnerabilities against installed version | `Look it up in the vulnerability database` | `Update the scan history` | Check each component<br>Looks each one up in WPScan, keeps only issues affecting your installed version and records the check. |
| `Update the scan history` | `dataTable` | Upserts scan results to Data Table | `Match against your version` | `One component at a time` | Check each component<br>Looks each one up in WPScan, keeps only issues affecting your installed version and records the check. |
| `Build the report` | `code` | Formats scan findings into a report | `Anything to look up?`, `One component at a time` | `Worth emailing?` | Report<br>Findings first, then everything that was not checked and why. |
| `Worth emailing?` | `if` | Evaluates if notification email is needed | `Build the report` | `Email the report` | Report<br>Findings first, then everything that was not checked and why. |
| `Email the report` | `emailSend` | Sends the report via SMTP | `Worth emailing?` | None | Report<br>Findings first, then everything that was not checked and why. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create the Data Table:**
   - Create a new n8n Data Table named `wpscan_scan_history`.
   - Add the following columns:
     - `component` (Type: String, designated match column)
     - `kind` (Type: String)
     - `slug` (Type: String)
     - `version` (Type: String)
     - `last_checked` (Type: String)
     - `findings` (Type: String)
     - `affected_count` (Type: Number)

2. **Setup Credentials:**
   - **WordPress REST API:** Create an administrator user in WordPress, generate an Application Password, and set up an n8n **HTTP Basic Auth** credential with these details.
   - **WPScan API:** Obtain an API token from WPScan and create an n8n **Header Auth** credential with name `Authorization` and value `Token token=YOUR_TOKEN`.
   - **SMTP Server:** Configure an SMTP credential for outbound email delivery.

3. **Place and Configure Nodes:**
   - **Node 1 (`Every night at 03:30`):** Add a `Schedule Trigger` node configured for daily execution at `03:30`.
   - **Node 2 (`Scan settings`):** Add a `Set (Edit Fields)` node. Define string parameters: `site_url` (`https://your-site.example.com`), `wp_core_version` (`""`), `notify_email` (`you@example.com`), `notify_from` (`""`). Define number parameter: `max_lookups_per_run` (`20`). Define boolean parameter: `alert_only_on_findings` (`false`).
   - **Node 3 (`Check your settings`):** Add a `Code` node using JavaScript to validate settings parameters against placeholders and malformed URIs.
   - **Node 4 (`Ask WPScan how much quota is left`):** Add an `HTTP Request` node. Set URL to `https://wpscan.com/api/v3/status` and select the WPScan Header Auth credential.
   - **Node 5 (`List the plugins`):** Add an `HTTP Request` node. Set URL to `={{ $('Check your settings').first().json.site_url }}/wp-json/wp/v2/plugins`. Configure HTTP Basic Auth and add a custom `User-Agent` header (`Mozilla/5.0...`).
   - **Node 6 (`List the themes`):** Add an `HTTP Request` node. Set URL to `={{ $('Check your settings').first().json.site_url }}/wp-json/wp/v2/themes`. Configure HTTP Basic Auth, custom `User-Agent` header, and enable `Execute Once`.
   - **Node 7 (`Read the scan history`):** Add a `Data Table` node. Select table `wpscan_scan_history`, operation `Get`, return all rows, and enable `Execute Once`.
   - **Node 8 (`Plan the lookups`):** Add a `Code` node to aggregate inputs, calculate remaining quota, prioritize active and least-recently-checked components, and output candidate arrays.
   - **Node 9 (`Anything to look up?`):** Add an `If` node. Set condition to check `{{ $json.nothingToScan !== true }}`.
   - **Node 10 (`One component at a time`):** Add a `Split In Batches` node configured to iterate through items sequentially.
   - **Node 11 (`Look it up in the vulnerability database`):** Add an `HTTP Request` node. Set URL to dynamic evaluation for core, themes, or plugins (`https://wpscan.com/api/v3/...`). Configure Header Auth, and set response options to never error and return full response.
   - **Node 12 (`Match against your version`):** Add a `Code` node to execute semantic version comparisons (`cmp`), filter active vulnerabilities, and structure findings.
   - **Node 13 (`Update the scan history`):** Add a `Data Table` node. Select table `wpscan_scan_history`, operation `Upsert`, matching column `component`, mapping fields to incoming JSON.
   - **Node 14 (`Build the report`):** Add a `Code` node to aggregate loop execution data, construct the plaintext report body, and evaluate conditional flags.
   - **Node 15 (`Worth emailing?`):** Add an `If` node evaluating `={{ $json.send }}`.
   - **Node 16 (`Email the report`):** Add a `Send Email` node. Configure recipient, sender, subject, and text parameters from upstream JSON data using the SMTP credential.

4. **Establish Connections:**
   - Connect nodes sequentially matching the dependency graph outlined in Section 2 and Section 3 (including the loopback connection from `Update the scan history` back to `One component at a time`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full setup guide and documentation | [datadrifter.io](https://datadrifter.io/wordpress-plugin-vulnerabilities-wpscan-api/) |
| Project Credits | Made by DataDrifter |