Monitor Entra ID high-risk users with Microsoft Graph, TheHive and Slack

https://n8nworkflows.xyz/workflows/monitor-entra-id-high-risk-users-with-microsoft-graph--thehive-and-slack-19996


# Monitor Entra ID high-risk users with Microsoft Graph, TheHive and Slack

### 1. Workflow Overview

This workflow is designed to automate Security Operations Center (SOC) monitoring and incident response for Microsoft Entra ID Protection. Its primary purpose is to poll for high-risk identity detections, group and enrich them with context from Microsoft Graph, correlate risk factors, deduplicate incidents within TheHive, and broadcast structured alerts to a dedicated Slack channel. 

The workflow execution is divided into the following logical blocks:
- **1.1 Trigger & Detection Ingestion:** Scheduled execution that polls Microsoft Entra ID Protection for high-risk detections using automatic pagination.
- **1.2 Data Aggregation:** Normalizes and groups raw detections per user to consolidate distinct event signals into single investigation objects.
- **1.3 Identity Enrichment:** Queries Microsoft Graph for risky-user profiles and recent sign-in telemetry.
- **1.4 Correlation & Incident Compilation:** Evaluates risk metrics, determines priority classification (Standard vs. Critical), and constructs Markdown-formatted incident descriptions.
- **1.5 Incident Deduplication & Synchronization:** Queries TheHive for existing open alerts, deciding whether to update an existing record or provision a new one.
- **1.6 Notification & SOC Dispatch:** Formulates a notification payload and posts a summary message to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Detection Ingestion
This block starts the automation on a fixed schedule and pulls high-risk detection logs from Microsoft Entra ID.
- **Nodes Involved:** `Every 30 Minutes`, `Query Risk Detections`
- **Node Details:**
  - **Every 30 Minutes**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration Choices:* Configured to trigger execution every 30 minutes.
    - *Key Expressions:* None.
    - *Input/Output:* Output connects to `Query Risk Detections`.
    - *Edge Cases / Failures:* Missed executions if the n8n instance is offline.
  - **Query Risk Detections**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / API Polling)
    - *Configuration Choices:* Sends a GET request to Microsoft Graph endpoint `https://graph.microsoft.com/v1.0/identityProtection/riskDetections` with query parameter `$filter` set to fetch high-risk detections (`riskLevel eq 'high'`) from the last 20 minutes (`detectedDateTime ge ...`). Uses automatic response pagination via `@odata.nextLink`.
    - *Key Expressions:* 
      - Query filter: `={{ $now.minus({ minutes: 20 }).toUTC().toISO() }}`
      - Pagination next URL: `={{ $response.body["@odata.nextLink"] }}`
      - Pagination complete expression: `={{ $response.body["@odata.nextLink"] === undefined }}`
    - *Input/Output:* Input from `Every 30 Minutes`; output connects to `Group & Aggregate Per User`.
    - *Credential Requirement:* `microsoftEntraOAuth2Api` (Requires `IdentityRiskEvent.Read.All`).
    - *Edge Cases / Failures:* Token expiration, Microsoft Graph API rate limiting (HTTP 429), or query syntax evaluation errors.

#### 2.2 Data Aggregation
This block flattens the retrieved detection pages and groups individual risk events by target user principal/ID.
- **Nodes Involved:** `Group & Aggregate Per User`
- **Node Details:**
  - **Group & Aggregate Per User**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* JavaScript execution mode processing items into memory. Extracts array elements, filters for high-risk attributes, and aggregates unique properties like risk states, detection types, IP addresses, countries, and timestamps.
    - *Key Expressions:* Custom JS array mapping and reduction logic.
    - *Input/Output:* Input from `Query Risk Detections`; output connects to `Enrich Risky User`.
    - *Edge Cases / Failures:* Empty item arrays resulting in zero downstream execution paths.

#### 2.3 Identity Enrichment
This block queries Microsoft Graph for user-specific identity profiles and recent sign-in history.
- **Nodes Involved:** `Enrich Risky User`, `Enrich Recent Sign-Ins`
- **Node Details:**
  - **Enrich Risky User**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* GET request targeting `https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/{{ $json.userId }}`. Configured to never error on non-2xx responses (`neverError: true`), ensuring workflow continuity if user objects are missing.
    - *Key Expressions:* URL interpolation: `={{ $json.userId }}`
    - *Input/Output:* Input from `Group & Aggregate Per User`; output connects to `Enrich Recent Sign-Ins`.
    - *Credential Requirement:* `microsoftEntraOAuth2Api` (Requires `IdentityRiskyUser.Read.All`).
    - *Edge Cases / Failures:* 404 Not Found if the user ID has been deleted or purged in Azure AD; mitigated via `neverError`.
  - **Enrich Recent Sign-Ins**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* GET request to `https://graph.microsoft.com/v1.0/auditLogs/signIns` with query parameters filtering by user ID (`userId eq '{{ $json.id }}'`), limited to top 5 results sorted by descending creation date. Configured with `neverError: true`.
    - *Key Expressions:* Filter query: `={{ $json.id }}`
    - *Input/Output:* Input from `Enrich Risky User`; output connects to `Assess, Correlate & Build Incident`.
    - *Credential Requirement:* `microsoftEntraOAuth2Api` (Requires `AuditLog.Read.All`).
    - *Edge Cases / Failures:* Graph API audit log latency or missing sign-in entries.

#### 2.4 Correlation & Incident Compilation
Analyzes contextual signals against security heuristics to prioritize the incident and generate detailed markdown documentation.
- **Nodes Involved:** `Assess, Correlate & Build Incident`
- **Node Details:**
  - **Assess, Correlate & Build Incident**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation & Correlation Engine)
    - *Configuration Choices:* Correlates multiple data streams (`Group & Aggregate Per User`, `Enrich Risky User`, `Enrich Recent Sign-Ins`). Evaluates conditions such as privileged roles, impossible travel, confirmed compromise, and device non-compliance to set severity levels (`CRITICAL` severity 4 vs `HIGH` severity 3). Compiles a full Markdown report (`description`) and a unique deduplication reference (`sourceRef`).
    - *Key Expressions:* Cross-node reference functions using `$('Node Name').all()`.
    - *Input/Output:* Input from `Enrich Recent Sign-Ins`; output connects to `Query Open TheHive Alerts`.
    - *Edge Cases / Failures:* Data misalignment across data streams if node execution ordering shifts.

#### 2.5 Incident Deduplication & Synchronization
Queries TheHive to check for existing open alerts, preventing duplicate investigations for the same active incident.
- **Nodes Involved:** `Query Open TheHive Alerts`, `Deduplication Decision`, `Alert Already Open?`, `Update TheHive Alert`, `Create TheHive Alert`, `Merge Alert Paths`
- **Node Details:**
  - **Query Open TheHive Alerts**
    - *Type and Technical Role:* `n8n-nodes-base.theHiveProject` (ITSM / SOAR Integration)
    - *Configuration Choices:* Executes a structured JSON query against TheHive 5 to list open alerts matching type `entra-id-risk` and status `New`. Executed once per batch (`executeOnce: true`).
    - *Key Expressions:* Query payload structure defining filter parameters.
    - *Input/Output:* Input from `Assess, Correlate & Build Incident`; output connects to `Deduplication Decision`.
    - *Credential Requirement:* TheHive 5 API credentials.
    - *Edge Cases / Failures:* Network timeouts or authentication failures communicating with the TheHive instance.
  - **Deduplication Decision**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Logic Processor)
    - *Configuration Choices:* Maps generated incidents against the list of open TheHive alerts by comparing `sourceRef`, appending boolean `alertExists` and `existingAlertId` properties.
    - *Key Expressions:* JavaScript map/reduce logic.
    - *Input/Output:* Input from `Query Open TheHive Alerts`; output connects to `Alert Already Open?`.
  - **Alert Already Open?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Evaluates whether `alertExists` evaluates to `true`.
    - *Key Expressions:* `={{ $json.alertExists }}` (Boolean evaluation).
    - *Input/Output:* Input from `Deduplication Decision`. True branch routes to `Update TheHive Alert`; false branch routes to `Create TheHive Alert`.
  - **Update TheHive Alert**
    - *Type and Technical Role:* `n8n-nodes-base.theHiveProject` (Action)
    - *Configuration Choices:* Updates an existing alert’s severity and description fields.
    - *Key Expressions:* `={{ $json.severity }}`, `={{ $json.description }}`.
    - *Input/Output:* Input from `Alert Already Open?` (True branch); output connects to `Merge Alert Paths`.
  - **Create TheHive Alert**
    - *Type and Technical Role:* `n8n-nodes-base.theHiveProject` (Action)
    - *Configuration Choices:* Creates a new alert of type `entra-id-risk` with associated observables (User UPN, Primary IP).
    - *Key Expressions:* 
      - Title: `=Entra ID high-risk user detected: {{ $json.user }}`
      - Source Reference: `={{ $json.sourceRef }}`
      - Observables: `={{ $json.userPrincipalName }}`, `={{ $json.primaryIp }}`
    - *Input/Output:* Input from `Alert Already Open?` (False branch); output connects to `Merge Alert Paths`.
  - **Merge Alert Paths**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Flow Control)
    - *Configuration Choices:* Combines data from both the update and creation branches back into a single downstream stream.
    - *Input/Output:* Inputs from `Update TheHive Alert` and `Create TheHive Alert`; output connects to `Build Slack Notification`.

#### 2.6 Notification & SOC Dispatch
Constructs the final alert message payload and dispatches it to the designated Slack security channel.
- **Nodes Involved:** `Build Slack Notification`, `Post to Security Channel`
- **Node Details:**
  - **Build Slack Notification**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Payload Builder)
    - *Configuration Choices:* Formulates a structured Block Kit or Markdown-formatted text string incorporating severity emojis, risk indicators, affected users, and TheHive reference IDs.
    - *Key Expressions:* Cross-node reference `$('Deduplication Decision').item.json` combined with string concatenations.
    - *Input/Output:* Input from `Merge Alert Paths`; output connects to `Post to Security Channel`.
  - **Post to Security Channel**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Notification Output)
    - *Configuration Choices:* Posts a text message to a configured Slack channel using OAuth2 authentication with Markdown formatting enabled.
    - *Key Expressions:* Text: `={{ $json.slackText }}`
    - *Input/Output:* Input from `Build Slack Notification`; terminal node.
    - *Credential Requirement:* Slack OAuth2 API credentials.
    - *Edge Cases / Failures:* Slack API rate limits, invalid channel configurations, or missing bot scopes (`chat:write`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Every 30 Minutes** | scheduleTrigger | Triggers the workflow on a schedule | None | Query Risk Detections | Detect high-risk activity |
| **Query Risk Detections** | httpRequest | Queries MS Graph for high-risk detections | Every 30 Minutes | Group & Aggregate Per User | Detect high-risk activity |
| **Group & Aggregate Per User** | code | Groups raw detections per user | Query Risk Detections | Enrich Risky User | Group by user |
| **Enrich Risky User** | httpRequest | Fetches risky user details from MS Graph | Group & Aggregate Per User | Enrich Recent Sign-Ins | Enrich context |
| **Enrich Recent Sign-Ins** | httpRequest | Fetches user sign-in audit logs | Enrich Risky User | Assess, Correlate & Build Incident | Enrich context |
| **Assess, Correlate & Build Incident** | code | Correlates signals and builds report | Enrich Recent Sign-Ins | Query Open TheHive Alerts | Correlate risk |
| **Query Open TheHive Alerts** | theHiveProject | Lists active alerts in TheHive | Assess, Correlate & Build Incident | Deduplication Decision | Prevent duplicate alerts |
| **Deduplication Decision** | code | Matches incidents against open alerts | Query Open TheHive Alerts | Alert Already Open? | Prevent duplicate alerts |
| **Alert Already Open?** | if | Routes based on existing alert status | Deduplication Decision | Update TheHive Alert, Create TheHive Alert | Prevent duplicate alerts |
| **Update TheHive Alert** | theHiveProject | Updates an existing TheHive alert | Alert Already Open? | Merge Alert Paths | Create or update |
| **Create TheHive Alert** | theHiveProject | Creates a new alert in TheHive | Alert Already Open? | Merge Alert Paths | Create or update |
| **Merge Alert Paths** | merge | Merges update and create paths | Update TheHive Alert, Create TheHive Alert | Build Slack Notification | |
| **Build Slack Notification** | code | Formulates Slack message payload | Merge Alert Paths | Post to Security Channel | Notify the SOC |
| **Post to Security Channel** | slack | Posts summary alert to Slack | Build Slack Notification | None | Notify the SOC |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually within n8n:

1. **Trigger Setup:**
   - Create a **Schedule Trigger** node named `Every 30 Minutes`. Set the interval rule to run every `30` minutes.
2. **Detection Retrieval:**
   - Add an **HTTP Request** node named `Query Risk Detections`.
   - Connect `Every 30 Minutes` to this node.
   - Configure method as `GET`, URL as `https://graph.microsoft.com/v1.0/identityProtection/riskDetections`.
   - Add query parameter `$filter` with value `={{ $now.minus({ minutes: 20 }).toUTC().toISO() }}` (ensure filtering for `riskLevel eq 'high'`).
   - Enable pagination in options: set mode to *Response Contains Next URL*, mapping the next URL to `={{ $response.body["@odata.nextLink"] }}` and completion condition to `={{ $response.body["@odata.nextLink"] === undefined }}`.
   - Configure credentials to use `Microsoft Entra OAuth2 API`.
3. **Data Aggregation:**
   - Add a **Code** node named `Group & Aggregate Per User`.
   - Connect `Query Risk Detections` to it.
   - Insert the JavaScript logic provided in the source workflow to parse raw arrays, filter for high-risk events, and group properties by user identity.
4. **Enrichment Nodes:**
   - Add an **HTTP Request** node named `Enrich Risky User`. Connect `Group & Aggregate Per User` to it.
     - Set URL to `https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/{{ $json.userId }}`.
     - Enable `Never Error` in options. Configure `Microsoft Entra OAuth2 API` credentials.
   - Add an **HTTP Request** node named `Enrich Recent Sign-Ins`. Connect `Enrich Risky User` to it.
     - Set URL to `https://graph.microsoft.com/v1.0/auditLogs/signIns`.
     - Add query parameters: `$filter` = `userId eq '{{ $json.id }}'`, `$top` = `5`, `$orderby` = `createdDateTime desc`.
     - Enable `Never Error` in options. Configure `Microsoft Entra OAuth2 API` credentials.
5. **Correlation & Incident Compilation:**
   - Add a **Code** node named `Assess, Correlate & Build Incident`. Connect `Enrich Recent Sign-Ins` to it.
   - Implement the correlation script that evaluates administrative privileges, multiple countries/IPs, and non-compliant devices, generating severity scores (`3` or `4`), markdown `description` text, and a stable `sourceRef`.
6. **TheHive Deduplication & Integration:**
   - Add a **TheHive** node named `Query Open TheHive Alerts`. Connect the correlation node to it.
     - Set resource to `Query`, supplying the JSON array configuration filtering for alerts where type is `entra-id-risk` and status is `New`. Enable `Execute Once`.
   - Add a **Code** node named `Deduplication Decision`. Connect `Query Open TheHive Alerts` to it to map existing alerts against incoming incidents.
   - Add an **If** node named `Alert Already Open?`. Connect `Deduplication Decision` to it.
     - Set condition to evaluate if `{{ $json.alertExists }}` equals `true`.
   - Add an **Update** **TheHive** node named `Update TheHive Alert`. Connect the `True` output of the If node to it. Configure parameters to map `severity` and `description`.
   - Add a **Create** **TheHive** node named `Create TheHive Alert`. Connect the `False` output of the If node to it. Configure fields for `type` (`entra-id-risk`), `title`, `source`, `sourceRef`, `severity`, `description`, and add observables for user UPN (`mail`) and IP (`ip`).
   - Add a **Merge** node named `Merge Alert Paths`. Connect outputs from both `Update TheHive Alert` (index 0) and `Create TheHive Alert` (index 1) into this node.
7. **Slack Notification:**
   - Add a **Code** node named `Build Slack Notification`. Connect `Merge Alert Paths` to it.
   - Insert JavaScript to format Slack Markdown text blocks (`slackText`).
   - Add a **Slack** node named `Post to Security Channel`. Connect `Build Slack Notification` to it.
   - Configure action to post message to channel using Slack OAuth2 credentials, mapping the text parameter to `={{ $json.slackText }}` with markdown enabled.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Monitor Entra ID high-risk users with Microsoft Graph, TheHive and Slack | Workflow primary design specification |
| Required Microsoft Graph API Scopes | `IdentityRiskEvent.Read.All`, `IdentityRiskyUser.Read.All`, `AuditLog.Read.All` |
| TheHive Integration Requirements | TheHive 5 instance supporting alert creation, querying, and updating via API |