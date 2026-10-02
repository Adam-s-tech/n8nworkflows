Detect GitHub Actions supply-chain risks with Gemini and Slack

https://n8nworkflows.xyz/workflows/detect-github-actions-supply-chain-risks-with-gemini-and-slack-20291


# Detect GitHub Actions supply-chain risks with Gemini and Slack

### 1. Workflow Overview

The **Detect GitHub Actions supply-chain risks with Gemini and Slack** workflow is an automated security scanner designed to continuously monitor GitHub repositories for supply-chain vulnerabilities, risky patterns, and unpinned GitHub Actions. Running on a daily schedule, it performs static analysis and leverages Google Gemini to assess modifications in workflow files, subsequently alerting security teams via Slack and recording tracking data in an n8n Data Table.

The logical execution of the workflow is structured into five distinct functional blocks:

- **1.1 Initialization and Baseline Loading:** Triggers the automation on a daily schedule, establishes the global security configuration parameters, and initializes or loads the local inventory tracking database.
- **1.2 Repository Enumeration and File Discovery:** Connects to the GitHub API to discover target repositories, filters out forks or archived projects, lists files within the `.github/workflows` directory, and isolates YAML files while identifying changes since the last execution.
- **1.3 Static Security Analysis and Pinning Resolution:** Downloads newly modified workflow files, executes pattern-matching rule checks for common supply-chain attacks, and queries the GitHub GraphQL API to resolve exact commit SHAs for unpinned action references.
- **1.4 AI Review and Instant Alerting:** Prioritizes altered workflows by risk score, submits high-priority targets to Google Gemini for a structured security verdict, and dispatches immediate notification alerts to Slack for malicious or critical discoveries.
- **1.5 Data Persistence, Digest Reporting, and Remediation:** Aggregates scan metrics into the inventory store, utilizes Google Gemini to generate a daily security digest broadcasted to Slack, and conditionally opens GitHub security issues for private repositories.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Baseline Loading
- **Overview:** This block establishes the operational cadence of the workflow, provisions global settings variables, and ensures the local state-tracking database exists before inventory ingestion.
- **Nodes Involved:** `Every Day 7 AM`, `Settings`, `Create Inventory Table`, `Load Inventory`.
- **Node Details:**
  - **Every Day 7 AM** (`n8n-nodes-base.scheduleTrigger`)
    - *Technical Role:* Entry point triggering the workflow execution daily at 07:00 AM.
    - *Configuration:* Interval configured to trigger at hour 7 every day.
    - *Connections:* Input: None; Output: `Settings`.
    - *Failure Types:* None standard; missed executions occur if n8n is offline.
  - **Settings** (`n8n-nodes-base.set`)
    - *Technical Role:* Stores user-defined configuration variables (`github_owner`, `owner_type`, filtering rules, max thresholds, and feature flags).
    - *Configuration:* Assignments mapping string, number, and boolean configuration keys.
    - *Expressions:* Provides global configuration parameters utilized by downstream HTTP requests and evaluation nodes.
    - *Connections:* Input: `Every Day 7 AM`; Output: `Create Inventory Table`.
    - *Failure Types:* Invalid user parameter types.
  - **Create Inventory Table** (`n8n-nodes-base.dataTable`)
    - *Technical Role:* Provisions the `gha_security_inventory` Data Table schema on initial execution if it does not already exist.
    - *Configuration:* Resource: Table, Operation: Create, Table Name: `gha_security_inventory`, Option: Create if not exists (`true`). Executed only once (`executeOnce: true`).
    - *Connections:* Input: `Settings`; Output: `Load Inventory`.
    - *Failure Types:* Database permission or table creation locking issues.
  - **Load Inventory** (`n8n-nodes-base.dataTable`)
    - *Technical Role:* Retrieves all stored workflow state rows from the baseline inventory table to facilitate differential comparison.
    - *Configuration:* Resource: Row, Operation: Get, Return All: `true`, Table Name: `gha_security_inventory`. Executed only once (`executeOnce: true`), with data outputs forced (`alwaysOutputData: true`).
    - *Connections:* Input: `Create Inventory Table`; Output: `List Repositories`.
    - *Failure Types:* Data Table read timeouts or missing table schema.

#### 1.2 Repository Enumeration and File Discovery
- **Overview:** This block lists organization or user repositories from GitHub, filters out unwanted targets (such as forks or disabled repos), iterates through `.github/workflows` directories, isolates valid YAML files, and detects changes against the loaded inventory baseline.
- **Nodes Involved:** `List Repositories`, `Filter Repositories`, `Limit Repositories`, `List Workflow Files`, `Keep YAML Files`, `Classify Changes`, `Changed?`.
- **Node Details:**
  - **List Repositories** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Fetches repositories associated with the configured owner via GitHub REST API with built-in pagination handling.
    - *Configuration:* URL evaluated dynamically based on `owner_type` (`org` vs `user`), Authentication: Predefined credential type (`githubApi`), Headers: `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`. Pagination set up to process up to 20 pages via response header links.
    - *Expressions:* `={{ $('Settings').first().json.owner_type === 'user' ? 'https://api.github.com/user/repos?...' : 'https://api.github.com/orgs/' + ... }}`
    - *Connections:* Input: `Load Inventory`; Output: `Filter Repositories`.
    - *Failure Types:* GitHub API rate-limiting, authentication failures, invalid token scopes.
  - **Filter Repositories** (`n8n-nodes-base.filter`)
    - *Technical Role:* Excludes archived repositories, disabled repositories, and unwanted forks depending on settings.
    - *Configuration:* JavaScript condition evaluating repository properties (`archived`, `disabled`, `fork`, and `repo_name_filter` regex match).
    - *Connections:* Input: `List Repositories`; Output: `Limit Repositories`.
    - *Failure Types:* Malformed regular expression in `repo_name_filter`.
  - **Limit Repositories** (`n8n-nodes-base.limit`)
    - *Technical Role:* Restricts the maximum number of repositories processed in a single run.
    - *Configuration:* Max Items bound to `={{ $('Settings').first().json.max_repositories }}`.
    - *Connections:* Input: `Filter Repositories`; Output: `List Workflow Files`.
    - *Failure Types:* None.
  - **List Workflow Files** (`n8n-nodes-base.github`)
    - *Technical Role:* Enumerates the contents of the `.github/workflows` directory for each target repository using the GitHub API.
    - *Configuration:* Resource: File, Operation: List, Owner: `={{ $json.owner.login }}`, Repository: `={{ $json.name }}`, File Path: `.github/workflows`. Error handling: Continue regular output on error.
    - *Connections:* Input: `Limit Repositories`; Output: `Keep YAML Files`.
    - *Failure Types:* 404 errors when `.github/workflows` does not exist in a repository (handled by error continuation).
  - **Keep YAML Files** (`n8n-nodes-base.filter`)
    - *Technical Role:* Filters list outputs to retain only standard files with `.yml` or `.yaml` extensions.
    - *Configuration:* Conditions verifying `type === 'file'` and filename matching regex `/\\.ya?ml$/i`.
    - *Connections:* Input: `List Workflow Files`; Output: `Classify Changes`.
    - *Failure Types:* None.
  - **Classify Changes** (`n8n-nodes-base.code`)
    - *Technical Role:* Compares current workflow file SHAs against historical inventory records to categorize items as `new`, `changed`, or `unchanged`.
    - *Configuration:* Custom JavaScript block loading inventory state and generating normalized file metadata objects.
    - *Connections:* Input: `Keep YAML Files`; Output: `Changed?`.
    - *Failure Types:* JSON parsing errors on historical details storage.
  - **Changed?** (`n8n-nodes-base.if`)
    - *Technical Role:* Routes files for deep security inspection only if they are new or have uncommitted changes; otherwise, bypasses inspection for unchanged files.
    - *Configuration:* Condition evaluating whether `change_type` equals `'new'` or `'changed'`.
    - *Connections:* Input: `Classify Changes`; Outputs: True branch to `Fetch Workflow File`, False branch to `Collect Results`.
    - *Failure Types:* None.

#### 1.3 Static Security Analysis and Pinning Resolution
- **Overview:** This block downloads raw file content for modified workflows, performs comprehensive rule-based security audits, builds a batch GraphQL query to check unpinned actions, and maps commit SHAs for version pinning suggestions.
- **Nodes Involved:** `Fetch Workflow File`, `Audit Rules`, `Build Pin Query`, `Resolve Pinned SHAs`, `Attach Pin Suggestions`.
- **Node Details:**
  - **Fetch Workflow File** (`n8n-nodes-base.github`)
    - *Technical Role:* Downloads the raw file content for newly identified or altered workflow files.
    - *Configuration:* Resource: File, Operation: Get, Owner and Repository dynamic expressions, As Binary Property: `false`. Error handling: Continue regular output.
    - *Connections:* Input: `Changed?` (True); Output: `Audit Rules`.
    - *Failure Types:* API timeouts, truncated responses, unreadable file encodings.
  - **Audit Rules** (`n8n-nodes-base.code`)
    - *Technical Role:* Executes a series of static regex and semantic security checks across workflow lines to detect supply-chain vulnerabilities (such as script injections, unpinned actions, secret dumping, exfiltration endpoints, and runner backdoors).
    - *Configuration:* Run once for each item JavaScript parser processing base64 decoded content lines.
    - *Connections:* Input: `Fetch Workflow File`; Output: `Build Pin Query`.
    - *Failure Types:* Script parsing exceptions on anomalous syntax structures.
  - **Build Pin Query** (`n8n-nodes-base.code`)
    - *Technical Role:* Aggregates unpinned action references across all audited files and constructs an optimized, batched GitHub GraphQL query to query their latest commit SHAs.
    - *Configuration:* JavaScript collector extracting unique action references and formatting GraphQL query strings.
    - *Connections:* Input: `Audit Rules`; Output: `Resolve Pinned SHAs`.
    - *Failure Types:* Exceeding batch query size limits (capped internally at 60 items).
  - **Resolve Pinned SHAs** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Executes the batched GraphQL request against the GitHub API to resolve commit object IDs (OIDs) for unpinned references.
    - *Configuration:* URL: `https://api.github.com/graphql`, Method: POST, Authentication: Predefined credential type (`githubApi`), JSON body parameter containing the query string. Error handling: Continue regular output.
    - *Connections:* Input: `Build Pin Query`; Output: `Attach Pin Suggestions`.
    - *Failure Types:* GraphQL query complexity limitations or rate limits.
  - **Attach Pin Suggestions** (`n8n-nodes-base.code`)
    - *Technical Role:* Maps resolved commit OIDs back to unpinned action references, generates ready-to-paste pinning suggestions, calculates file risk scores, and selects candidate files for subsequent AI review.
    - *Configuration:* JavaScript mapping function evaluating risk score weighting and capping AI reviews by `max_ai_reviews`.
    - *Connections:* Input: `Resolve Pinned SHAs`; Output: `Collect Results` and `Needs AI Review?`.
    - *Failure Types:* Unmatched GraphQL response properties.

#### 1.4 AI Review and Instant Alerting
- **Overview:** This block filters files requiring artificial intelligence evaluation, executes Google Gemini structured chat completions to produce security verdicts, parses the outputs, and immediately dispatches alerts to Slack for critical or malicious findings.
- **Nodes Involved:** `Needs AI Review?`, `AI Workflow Reviewer`, `Gemini (Reviewer)`, `Verdict Parser`, `Format Verdict`, `Alert Needed?`, `Slack Security Alert`.
- **Node Details:**
  - **Needs AI Review?** (`n8n-nodes-base.filter`)
    - *Technical Role:* Isolates files flagged for AI inspection by the previous selection logic.
    - *Configuration:* Condition evaluating `ai_review === true`.
    - *Connections:* Input: `Attach Pin Suggestions`; Output: `AI Workflow Reviewer`.
    - *Failure Types:* None.
  - **AI Workflow Reviewer** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Technical Role:* Orchestrates the LangChain chat model execution, prompting Google Gemini to assess untrusted workflow contents for malicious indicators.
    - *Configuration:* Max Tries: 3, Retry on Fail: `true`. Linked to model and output parser nodes.
    - *Connections:* Inputs: `Needs AI Review?`, `Gemini (Reviewer)`, `Verdict Parser`; Output: `Format Verdict`.
    - *Failure Types:* Model API errors, rate-limiting, output parsing validation failures.
  - **Gemini (Reviewer)** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Technical Role:* Provides the underlying Google Gemini chat model implementation (`models/gemini-3.8-flash`) for file reviews.
    - *Configuration:* Authentication: Google Gemini API key credential.
    - *Connections:* Output: Connected to `AI Workflow Reviewer` as Language Model.
    - *Failure Types:* Invalid API key credentials or service outages.
  - **Verdict Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Technical Role:* Enforces a strict JSON schema on the Gemini review output, validating fields such as `verdict`, `confidence`, `summary`, `evidence`, and `recommended_action`.
    - *Configuration:* Manual schema type definition containing enum constraints for verdicts (`malicious`, `suspicious`, `risky`, `ok`) and confidence levels.
    - *Connections:* Output: Connected to `AI Workflow Reviewer` as Output Parser.
    - *Failure Types:* Schema validation errors if the LLM output drifts from the specified structure.
  - **Format Verdict** (`n8n-nodes-base.code`)
    - *Technical Role:* Normalizes structured AI verdicts and compiles formatted alert payloads for Slack notification.
    - *Configuration:* Run once for each item JavaScript formatting text strings with HTML-safe entity replacements.
    - *Connections:* Input: `AI Workflow Reviewer`; Outputs: `Alert Needed?` and `Collect Results`.
    - *Failure Types:* None.
  - **Alert Needed?** (`n8n-nodes-base.filter`)
    - *Technical Role:* Determines whether an instant Slack notification must be sent based on severity thresholds.
    - *Configuration:* Condition evaluating whether the verdict is `malicious` or `suspicious`, or if critical rule findings count exceeds zero.
    - *Connections:* Input: `Format Verdict`; Output: `Slack Security Alert`.
    - *Failure Types:* None.
  - **Slack Security Alert** (`n8n-nodes-base.slack`)
    - *Technical Role:* Posts real-time security alerts to the designated Slack channel.
    - *Configuration:* Resource: Message, Select: Channel, Channel ID: `#security-alerts`, Authentication: Predefined Slack credential.
    - *Connections:* Input: `Alert Needed?`; Output: None (Terminal node for this branch).
    - *Failure Types:* Slack token permission errors or missing channel access.

#### 1.5 Data Persistence, Digest Reporting, and Remediation
- **Overview:** This block consolidates scan results, upserts inventory records into the Data Table, generates an AI-powered daily summary digest, posts it to Slack, and conditionally opens GitHub issues in private repositories.
- **Nodes Involved:** `Collect Results`, `Prepare Inventory Rows`, `Save Inventory`, `Build Security Report`, `AI Security Brief`, `Gemini (Brief)`, `Brief Parser`, `Render Slack Digest`, `Post Daily Digest`, `Plan GitHub Issues`, `Create GitHub Issue`.
- **Node Details:**
  - **Collect Results** (`n8n-nodes-base.merge`)
    - *Technical Role:* Merges unchanged file records, processed workflow outputs, and AI verdict objects into a unified stream.
    - *Configuration:* Mode: Append, Number of Inputs: 3.
    - *Connections:* Inputs: `Changed?` (False), `Format Verdict`, `Attach Pin Suggestions`; Outputs: `Prepare Inventory Rows`, `Build Security Report`, `Plan GitHub Issues`.
    - *Failure Types:* Stream synchronization mismatches.
  - **Prepare Inventory Rows** (`n8n-nodes-base.code`)
    - *Technical Role:* Prepares structured rows containing calculated risk scores, findings counts, and serialized details for Data Table persistence.
    - *Configuration:* JavaScript object transformer with string truncation logic for payload size control.
    - *Connections:* Input: `Collect Results`; Output: `Save Inventory`.
    - *Failure Types:* Data size exceeding table limits.
  - **Save Inventory** (`n8n-nodes-base.dataTable`)
    - *Technical Role:* Upserts workflow scan findings and baseline metrics into the `gha_security_inventory` Data Table.
    - *Configuration:* Resource: Row, Operation: Upsert, Match Type: All conditions, Filters matching on `key`.
    - *Connections:* Input: `Prepare Inventory Rows`; Output: None.
    - *Failure Types:* Database write failures or constraint violations.
  - **Build Security Report** (`n8n-nodes-base.code`)
    - *Technical Role:* Aggregates statistics across all scanned repositories to produce a comprehensive summary report for the daily AI brief.
    - *Configuration:* JavaScript aggregation script calculating rule counts, riskiest repositories, and severity distributions.
    - *Connections:* Input: `Collect Results`; Output: `AI Security Brief`.
    - *Failure Types:* None.
  - **AI Security Brief** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Technical Role:* Prompts Google Gemini to synthesize the scan report into a developer-friendly daily security digest.
    - *Configuration:* Max Tries: 3, Retry on Fail: `true`. Linked to model and output parser nodes.
    - *Connections:* Inputs: `Build Security Report`, `Gemini (Brief)`, `Brief Parser`; Output: `Render Slack Digest`.
    - *Failure Types:* LLM timeout or structural parsing deviations.
  - **Gemini (Brief)** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Technical Role:* Provides the chat model backend (`models/gemini-3.8-flash`) for generating the daily security brief.
    - *Configuration:* Authentication: Google Gemini API key credential.
    - *Connections:* Output: Connected to `AI Security Brief` as Language Model.
    - *Failure Types:* API credential errors.
  - **Brief Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Technical Role:* Enforces a strict JSON schema on the security brief containing `headline`, `summary`, `priority_actions`, and `quick_win`.
    - *Configuration:* Manual schema type definition.
    - *Connections:* Output: Connected to `AI Security Brief` as Output Parser.
    - *Failure Types:* Schema parsing exceptions.
  - **Render Slack Digest** (`n8n-nodes-base.code`)
    - *Technical Role:* Converts the aggregated scan report and AI brief structure into a formatted Markdown Slack message.
    - *Configuration:* Run once for each item JavaScript template string builder.
    - *Connections:* Input: `AI Security Brief`; Output: `Post Daily Digest`.
    - *Failure Types:* None.
  - **Post Daily Digest** (`n8n-nodes-base.slack`)
    - *Technical Role:* Broadcasts the daily security digest message to the configured Slack channel.
    - *Configuration:* Resource: Message, Channel ID: `#security-alerts`, Authentication: Predefined Slack credential.
    - *Connections:* Input: `Render Slack Digest`; Output: None (Terminal node).
    - *Failure Types:* Slack API connectivity or token scope errors.
  - **Plan GitHub Issues** (`n8n-nodes-base.code`)
    - *Technical Role:* Drafts GitHub issue payloads for private repositories containing critical or high-severity security findings when issue creation is enabled.
    - *Configuration:* JavaScript filter and markdown sanitizer checking `create_github_issues` settings.
    - *Connections:* Input: `Collect Results`; Output: `Create GitHub Issue`.
    - *Failure Types:* None.
  - **Create GitHub Issue** (`n8n-nodes-base.github`)
    - *Technical Role:* Automatically opens a security issue within private repositories that contain unaddressed severe findings.
    - *Configuration:* Resource: Issue, Operation: Create, Owner and Repository dynamic bindings, Title and Body bindings.
    - *Connections:* Input: `Plan GitHub Issues`; Output: None (Terminal node).
    - *Failure Types:* Missing `issues: write` token permissions on private repositories resulting in 403 Forbidden errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Main Note | `n8n-nodes-base.stickyNote` | Workflow documentation, architecture overview, setup instructions, and requirements. | None | None | Detect supply-chain risks in GitHub Actions workflows with Gemini and Slack... |
| Step 1 Note | `n8n-nodes-base.stickyNote` | Documentation note covering settings and inventory setup. | None | None | 1. Settings & inventory... |
| Step 2 Note | `n8n-nodes-base.stickyNote` | Documentation note covering repository enumeration and file discovery. | None | None | 2. Find workflow files... |
| Step 3 Note | `n8n-nodes-base.stickyNote` | Documentation note covering security checks and SHA resolution. | None | None | 3. Security checks... |
| Step 4 Note | `n8n-nodes-base.stickyNote` | Documentation note covering AI reviews and instant alerts. | None | None | 4. AI review & instant alerts... |
| Step 5 Note | `n8n-nodes-base.stickyNote` | Documentation note covering data saving, reporting, and issue creation. | None | None | 5. Save, report & issues... |
| Every Day 7 AM | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow execution daily at 7:00 AM. | None | Settings | |
| Settings | `n8n-nodes-base.set` | Stores global configuration parameters and thresholds. | Every Day 7 AM | Create Inventory Table | |
| Create Inventory Table | `n8n-nodes-base.dataTable` | Provisions the inventory table schema if it does not exist. | Settings | Load Inventory | |
| Load Inventory | `n8n-nodes-base.dataTable` | Loads historical workflow records for differential comparison. | Create Inventory Table | List Repositories | |
| List Repositories | `n8n-nodes-base.httpRequest` | Fetches repositories from GitHub with pagination support. | Load Inventory | Filter Repositories | |
| Filter Repositories | `n8n-nodes-base.filter` | Filters out archived repositories, disabled repos, and forks. | List Repositories | Limit Repositories | |
| Limit Repositories | `n8n-nodes-base.limit` | Restricts the maximum number of processed repositories. | Filter Repositories | List Workflow Files | |
| List Workflow Files | `n8n-nodes-base.github` | Lists files in the `.github/workflows` directory. | Limit Repositories | Keep YAML Files | |
| Keep YAML Files | `n8n-nodes-base.filter` | Retains only valid YAML workflow files. | List Workflow Files | Classify Changes | |
| Classify Changes | `n8n-nodes-base.code` | Compares current file SHAs with inventory to detect changes. | Keep YAML Files | Changed? | |
| Changed? | `n8n-nodes-base.if` | Routes new or changed files for deep inspection. | Classify Changes | Fetch Workflow File, Collect Results | |
| Fetch Workflow File | `n8n-nodes-base.github` | Downloads raw file content for modified workflows. | Changed? | Audit Rules | |
| Audit Rules | `n8n-nodes-base.code` | Performs static rule-based security audits on workflow code. | Fetch Workflow File | Build Pin Query | |
| Build Pin Query | `n8n-nodes-base.code` | Builds a batched GraphQL query to resolve unpinned action SHAs. | Audit Rules | Resolve Pinned SHAs | |
| Resolve Pinned SHAs | `n8n-nodes-base.httpRequest` | Resolves commit SHAs for unpinned actions via GitHub GraphQL. | Build Pin Query | Attach Pin Suggestions | |
| Attach Pin Suggestions | `n8n-nodes-base.code` | Maps resolved SHAs, generates pinning lines, and selects AI targets. | Resolve Pinned SHAs | Collect Results, Needs AI Review? | |
| Needs AI Review? | `n8n-nodes-base.filter` | Isolates files selected for artificial intelligence review. | Attach Pin Suggestions | AI Workflow Reviewer | |
| AI Workflow Reviewer | `@n8n/n8n-nodes-langchain.chainLlm` | Orchestrates Google Gemini evaluation of untrusted workflows. | Needs AI Review?, Gemini (Reviewer), Verdict Parser | Format Verdict | |
| Gemini (Reviewer) | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the Gemini chat model backend for workflow reviews. | None | AI Workflow Reviewer | |
| Verdict Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces a structured JSON schema on the reviewer output. | None | AI Workflow Reviewer | |
| Format Verdict | `n8n-nodes-base.code` | Formats AI review verdicts and prepares Slack alert text. | AI Workflow Reviewer | Alert Needed?, Collect Results | |
| Alert Needed? | `n8n-nodes-base.filter` | Evaluates whether an immediate Slack security alert is required. | Format Verdict | Slack Security Alert | |
| Slack Security Alert | `n8n-nodes-base.slack` | Posts immediate security alerts to the target Slack channel. | Alert Needed? | None | |
| Collect Results | `n8n-nodes-base.merge` | Merges unchanged, processed, and evaluated workflow data. | Changed?, Format Verdict, Attach Pin Suggestions | Prepare Inventory Rows, Build Security Report, Plan GitHub Issues | |
| Prepare Inventory Rows | `n8n-nodes-base.code` | Prepares structured inventory data for database upsertion. | Collect Results | Save Inventory | |
| Save Inventory | `n8n-nodes-base.dataTable` | Upserts scan results and metrics into the inventory table. | Prepare Inventory Rows | None | |
| Build Security Report | `n8n-nodes-base.code` | Aggregates overall scan metrics for the daily security digest. | Collect Results | AI Security Brief | |
| AI Security Brief | `@n8n/n8n-nodes-langchain.chainLlm` | Prompts Gemini to synthesize the daily security briefing. | Build Security Report, Gemini (Brief), Brief Parser | Render Slack Digest | |
| Gemini (Brief) | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the Gemini chat model backend for the daily digest. | None | AI Security Brief | |
| Brief Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces a structured JSON schema on the daily brief output. | None | AI Security Brief | |
| Render Slack Digest | `n8n-nodes-base.code` | Converts scan reports and AI briefs into formatted Slack text. | AI Security Brief | Post Daily Digest | |
| Post Daily Digest | `n8n-nodes-base.slack` | Broadcasts the daily security digest to the target Slack channel. | Render Slack Digest | None | |
| Plan GitHub Issues | `n8n-nodes-base.code` | Drafts GitHub security issues for private repositories. | Collect Results | Create GitHub Issue | |
| Create GitHub Issue | `n8n-nodes-base.github` | Automatically opens issues in private repositories with vulnerabilities. | Plan GitHub Issues | None | |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow manually in n8n, follow these sequential steps:

1. **Trigger Node:**
   - Create a **Schedule Trigger** node named `Every Day 7 AM`. Configure the rule interval to trigger daily at hour `7`.
2. **Settings Node:**
   - Add a **Set** node named `Settings`. Define string parameters: `github_owner` (target organization or username), `owner_type` (`org` or `user`), `repo_name_filter` (`""`), `trusted_action_owners` (`actions,github`). Define boolean parameters: `include_forks` (`false`), `create_github_issues` (`false`). Define number parameters: `max_repositories` (`100`), `max_ai_reviews` (`10`).
   - *Connection:* `Every Day 7 AM` $\rightarrow$ `Settings`.
3. **Inventory Setup Nodes:**
   - Add a **Data Table** node named `Create Inventory Table`. Set resource to `table`, operation to `create`, table name to `gha_security_inventory`, and enable option `Create If Not Exists`. Define 16 columns: `key` (string), `repo` (string), `path` (string), `sha` (string), `visibility` (string), `first_seen` (string), `last_seen` (string), `last_changed` (string), `risk_score` (number), `critical` (number), `high` (number), `medium` (number), `low` (number), `ai_verdict` (string), `ai_summary` (string), `details` (string). Enable `Execute Once`.
   - Add a **Data Table** node named `Load Inventory`. Set resource to `row`, operation to `get`, return all to `true`, table name to `gha_security_inventory`. Enable `Execute Once` and `Always Output Data`.
   - *Connections:* `Settings` $\rightarrow$ `Create Inventory Table` $\rightarrow$ `Load Inventory`.
4. **Repository Listing & Filtering Nodes:**
   - Add an **HTTP Request** node named `List Repositories`. Configure URL dynamically using `owner_type` and `github_owner` from the Settings node. Configure pagination via response header links (limit 20 pages). Set authentication to **GitHub API** credentials with headers `Accept: application/vnd.github+json` and `X-GitHub-Api-Version: 2022-11-28`. Enable `Execute Once`.
   - Add a **Filter** node named `Filter Repositories` to exclude archived/disabled repositories and unauthorized forks.
   - Add a **Limit** node named `Limit Repositories` capping items to `={{ $('Settings').first().json.max_repositories }}`.
   - *Connections:* `Load Inventory` $\rightarrow$ `List Repositories` $\rightarrow$ `Filter Repositories` $\rightarrow$ `Limit Repositories`.
5. **File Enumeration & Classification Nodes:**
   - Add a **GitHub** node named `List Workflow Files`. Set resource to `file`, operation to `list`, repository owner to `={{ $json.owner.login }}`, repository name to `={{ $json.name }}`, and file path to `.github/workflows`. Set error handling to continue regular output.
   - Add a **Filter** node named `Keep YAML Files` to filter files matching `.yml` or `.yaml`.
   - Add a **Code** node named `Classify Changes` containing JavaScript logic to compare current file SHAs against historical inventory records.
   - Add an **If** node named `Changed?` evaluating whether `change_type` is `'new'` or `'changed'`.
   - *Connections:* `Limit Repositories` $\rightarrow$ `List Workflow Files` $\rightarrow$ `Keep YAML Files` $\rightarrow$ `Classify Changes` $\rightarrow$ `Changed?`.
6. **File Audit & SHA Resolution Nodes:**
   - Add a **GitHub** node named `Fetch Workflow File` (connected to the true output of `Changed?`). Set resource to `file`, operation to `get`, owner to `={{ $json.owner }}`, repository to `={{ $json.name }}`, and file path to `={{ $json.path }}`. Set error handling to continue regular output.
   - Add a **Code** node named `Audit Rules` to perform regex-based security checks.
   - Add a **Code** node named `Build Pin Query` to aggregate unpinned actions into a GraphQL query.
   - Add an **HTTP Request** node named `Resolve Pinned SHAs`. Set URL to `https://api.github.com/graphql`, method to `POST`, authentication to **GitHub API** credentials, and send body as JSON containing the GraphQL query. Set error handling to continue regular output.
   - Add a **Code** node named `Attach Pin Suggestions` to map resolved commit SHAs, attach pinning suggestions, calculate risk scores, and flag files for AI review.
   - *Connections:* `Changed?` (True) $\rightarrow$ `Fetch Workflow File` $\rightarrow$ `Audit Rules` $\rightarrow$ `Build Pin Query` $\rightarrow$ `Resolve Pinned SHAs` $\rightarrow$ `Attach Pin Suggestions`.
7. **AI Review & Alerting Nodes:**
   - Add a **Filter** node named `Needs AI Review?` filtering items where `ai_review === true`.
   - Add a **Google Gemini Chat Model** node named `Gemini (Reviewer)` configured with `models/gemini-3.8-flash` and a Google Gemini API credential.
   - Add a **Structured Output Parser** node named `Verdict Parser` configured with the required JSON schema (`verdict`, `confidence`, `summary`, `evidence`, `recommended_action`).
   - Add an **AI Agent / Basic LLM Chain** node named `AI Workflow Reviewer` connected to `Needs AI Review?`, using `Gemini (Reviewer)` and `Verdict Parser`.
   - Add a **Code** node named `Format Verdict` to normalize review outputs and build Slack message strings.
   - Add a **Filter** node named `Alert Needed?` checking for malicious/suspicious verdicts or critical findings.
   - Add a **Slack** node named `Slack Security Alert`. Set resource to message, channel to `#security-alerts`, and authentication to a Slack credential.
   - *Connections:* `Attach Pin Suggestions` $\rightarrow$ `Needs AI Review?` $\rightarrow$ `AI Workflow Reviewer` $\rightarrow$ `Format Verdict` $\rightarrow$ `Alert Needed?` $\rightarrow$ `Slack Security Alert`.
8. **Persistence, Reporting & Issue Creation Nodes:**
   - Add a **Merge** node named `Collect Results` configured for append mode with 3 inputs. Connect inputs from `Changed?` (False branch), `Format Verdict`, and `Attach Pin Suggestions`.
   - Add a **Code** node named `Prepare Inventory Rows` and connect it to `Collect Results`.
   - Add a **Data Table** node named `Save Inventory`. Set resource to row, operation to upsert, table name to `gha_security_inventory`, matching columns on `key`.
   - Add a **Code** node named `Build Security Report` connected to `Collect Results`.
   - Add a **Google Gemini Chat Model** node named `Gemini (Brief)` configured with `models/gemini-3.8-flash`.
   - Add a **Structured Output Parser** node named `Brief Parser` configured with the daily digest schema (`headline`, `summary`, `priority_actions`, `quick_win`).
   - Add an **AI Agent / Basic LLM Chain** node named `AI Security Brief` connected to `Build Security Report`, `Gemini (Brief)`, and `Brief Parser`.
   - Add a **Code** node named `Render Slack Digest` to format the summary into a Slack message.
   - Add a **Slack** node named `Post Daily Digest` posting to `#security-alerts`.
   - Add a **Code** node named `Plan GitHub Issues` connected to `Collect Results`.
   - Add a **GitHub** node named `Create GitHub Issue`. Set resource to issue, operation to create, owner and repository dynamic bindings.
   - *Connections:*
     - `Collect Results` $\rightarrow$ `Prepare Inventory Rows` $\rightarrow$ `Save Inventory`.
     - `Collect Results` $\rightarrow$ `Build Security Report` $\rightarrow$ `AI Security Brief` $\rightarrow$ `Render Slack Digest` $\rightarrow$ `Post Daily Digest`.
     - `Collect Results` $\rightarrow$ `Plan GitHub Issues` $\rightarrow$ `Create GitHub Issue`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **API Credentials Required** | Requires a GitHub Personal Access Token (classic with `repo` scope, or fine-grained with Contents: Read, Metadata: Read, and optional Issues: Write), a free Google Gemini API key from Google AI Studio, and a Slack Bot token with channel post permissions. |
| **Data Tables Prerequisite** | This workflow utilizes built-in n8n Data Tables (introduced in n8n 2.x) to manage persistent state across executions without requiring external databases. |
| **Security Architecture** | Static rule audits execute locally without AI interaction, and AI prompts treat repository workflow contents as untrusted data to prevent prompt injection bypasses. |