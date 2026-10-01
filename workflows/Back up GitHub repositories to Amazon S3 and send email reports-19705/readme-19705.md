Back up GitHub repositories to Amazon S3 and send email reports

https://n8nworkflows.xyz/workflows/back-up-github-repositories-to-amazon-s3-and-send-email-reports-19705


# Back up GitHub repositories to Amazon S3 and send email reports

### 1. Workflow Overview

This workflow automates the nightly backup of GitHub repositories by downloading each repository as a tarball, uploading it to an Amazon S3 bucket, pruning backups older than a defined retention window, and emailing a comprehensive summary report. It targets developers, DevOps engineers, and organizations needing reliable automated off-site code archiving.

The workflow is categorized into the following logical blocks:
- **1.1 Schedule and Validation:** Triggers execution nightly, assigns backup configurations, and validates settings against common placeholders.
- **1.2 Repository Discovery:** Identifies the target GitHub account or organization and performs paginated API calls to list all owned repositories.
- **1.3 Repository Backup Loop:** Iterates through each repository individually, downloads the tarball archive with retry handling, uploads it to Amazon S3, and records execution metrics.
- **1.4 Retention Cleanup:** Lists existing archives in S3, filters files exceeding the retention limit, and conditionally deletes expired objects.
- **1.5 Reporting:** Aggregates logs across all iteration cycles and dispatches an SMTP status email.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Validation
- **Overview:** Initializes the backup pipeline on a nightly schedule, declares environmental parameters, and performs guardrail validations to fail fast if defaults or placeholders are detected.
- **Nodes Involved:** `When Nightly at 2 AM`, `Set Backup Parameters`, `Validate Backup Settings`.
- **Node Details:**
  - **When Nightly at 2 AM**
    - Type and technical role: `Schedule Trigger` initiates execution via cron.
    - Configuration choices: Configured with cron expression `0 2 * * *`.
    - Connections: Outputs to `Set Backup Parameters`.
    - Edge cases/Failure types: None (trigger node).
  - **Set Backup Parameters**
    - Type and technical role: `Set` (Edit Fields) defines workflow variables.
    - Configuration choices: Establishes `github_owner`, `s3_bucket`, `s3_prefix`, `retention_days`, `notify_email`, and `notify_from`.
    - Key expressions or variables: Static string and number assignments.
    - Connections: Input from trigger; output to `Validate Backup Settings`.
    - Edge cases/Failure types: Misconfigured placeholder strings.
  - **Validate Backup Settings**
    - Type and technical role: `Code` node for fail-fast error checking.
    - Configuration choices: Custom JavaScript assessing whether placeholders like `you@example.com` or `your-backup-bucket` are present.
    - Key expressions or variables: Reads `$input.first().json`.
    - Connections: Input from `Set Backup Parameters`; output to `Fetch GitHub Account Info`.
    - Edge cases/Failure types: Throws explicit configuration errors halting execution immediately if validation rules fail.

#### 2.2 Repository Discovery
- **Overview:** Determines whether the target string points to an organization or user account and queries the GitHub REST API with pagination to retrieve a complete repository list.
- **Nodes Involved:** `Fetch GitHub Account Info`, `List GitHub Repos`.
- **Node Details:**
  - **Fetch GitHub Account Info**
    - Type and technical role: `HTTP Request` retrieves authenticated user or target user profile metadata.
    - Configuration choices: Uses predefined GitHub API credentials. Dynamically switches between `/user` and `/users/{owner}` based on the `github_owner` parameter.
    - Key expressions or variables: `={{ String($json.github_owner || '').trim() ? 'https://api.github.com/users/' + String($json.github_owner).trim() : 'https://api.github.com/user' }}`
    - Connections: Input from `Validate Backup Settings`; output to `List GitHub Repos`.
    - Version-specific requirements: HTTP Request v4.2.
    - Edge cases/Failure types: Authentication token expiration, insufficient scope permissions, or invalid user/org names.
  - **List GitHub Repos**
    - Type and technical role: `HTTP Request` fetches repositories using pagination.
    - Configuration choices: Evaluates target type (`Organization`, user, or authenticated account) and loops pagination via `per_page=100` and page incrementing until an empty array is returned.
    - Key expressions or variables: `={{ $json.type === 'Organization' ? 'https://api.github.com/orgs/' + $json.login + '/repos' : ... }}`
    - Connections: Input from `Fetch GitHub Account Info`; output to `Process Each Repository`.
    - Edge cases/Failure types: Rate limiting by GitHub API or network timeouts on massive organizations.

#### 2.3 Repository Backup Loop
- **Overview:** Processes repositories one by one, fetches their tarball archives, handles retry logic and empty repositories, and uploads payloads directly to S3.
- **Nodes Involved:** `Process Each Repository`, `Download Repo Archive`, `Upload Archive to S3`, `Log Repository Backup Status`.
- **Node Details:**
  - **Process Each Repository**
    - Type and technical role: `Split In Batches` controls iteration flow.
    - Configuration choices: Batch size set to 1 to bound peak memory usage to a single archive file.
    - Connections: Inputs from `List GitHub Repos` and `Log Repository Backup Status` (loop back); outputs to `Download Repo Archive` and `List S3 Backups` (post-loop branch).
  - **Download Repo Archive**
    - Type and technical role: `HTTP Request` downloads repository source as a tarball file.
    - Configuration choices: Response format set to `File`, automatic retry enabled (3 tries, 5000ms interval), error output routed explicitly via `continueErrorOutput`.
    - Key expressions or variables: `=https://api.github.com/repos/{{ $json.full_name }}/tarball`
    - Connections: Input from `Process Each Repository`; outputs to `Upload Archive to S3` (success) and `Log Repository Backup Status` (error branch).
    - Edge cases/Failure types: 404 errors for empty repositories with no commits, private repo permission blocks, or network timeouts.
  - **Upload Archive to S3**
    - Type and technical role: `AWS S3` node uploads compressed repository tarballs.
    - Configuration choices: Operation set to `Upload`, automated retries configured (3 tries).
    - Key expressions or variables: `={{ $('Set Backup Parameters').first().json.s3_prefix }}/{{ $('Process Each Repository').item.json.name }}-$now.format('yyyyLLdd-HHmmss')}.tar.gz`
    - Connections: Input from `Download Repo Archive`; output to `Log Repository Backup Status`.
    - Edge cases/Failure types: AWS credential misconfigurations, IAM policy restriction failures (`s3:PutObject`), or bucket region mismatches.
  - **Log Repository Backup Status**
    - Type and technical role: `Code` node aggregates individual iteration outcomes.
    - Configuration choices: Distinguishes between empty repositories (size 0 with 404 responses) and actual network/API failures.
    - Connections: Inputs from `Upload Archive to S3` and error output of `Download Repo Archive`; output loops back to `Process Each Repository`.

#### 2.4 Retention Cleanup
- **Overview:** Inspects the S3 bucket after all uploads complete, identifies objects older than the configured retention policy, and deletes expired files.
- **Nodes Involved:** `List S3 Backups`, `Filter Expired Backups`, `Check Expired Backups`, `Delete Expired Backup`.
- **Node Details:**
  - **List S3 Backups**
    - Type and technical role: `AWS S3` node retrieves all stored bucket objects under the designated prefix.
    - Configuration choices: Operation set to `Get All`, `alwaysOutputData` enabled to prevent workflow termination on empty buckets.
    - Connections: Input triggered once from `Process Each Repository` upon completion; output to `Filter Expired Backups`.
  - **Filter Expired Backups**
    - Type and technical role: `Code` node evaluates file modification timestamps.
    - Configuration choices: Computes a unified cutoff timestamp based on `retention_days` and collects expired items, defaulting to a fallback payload if no objects are stale.
    - Connections: Input from `List S3 Backups`; output to `Check Expired Backups`.
  - **Check Expired Backups**
    - Type and technical role: `If` conditional node guarding the deletion step.
    - Configuration choices: Evaluates whether the incoming payload contains a valid non-empty `key`.
    - Connections: Input from `Filter Expired Backups`; outputs to `Delete Expired Backup` (true) and `Generate Backup Report` (false).
  - **Delete Expired Backup**
    - Type and technical role: `AWS S3` node deletes obsolete backup files.
    - Configuration choices: Operation set to `Delete`, error handling configured to continue regular output on failure.
    - Key expressions or variables: `={{ $json.key }}`
    - Connections: Input from `Check Expired Backups`; output to `Generate Backup Report`.
    - Edge cases/Failure types: Insufficient IAM permissions (`s3:DeleteObject`) or objects deleted concurrently.

#### 2.5 Reporting
- **Overview:** Compiles execution metrics, success counts, failures, skipped repositories, and deletion summaries into an email notification body.
- **Nodes Involved:** `Generate Backup Report`, `Send Backup Report Email`.
- **Node Details:**
  - **Generate Backup Report**
    - Type and technical role: `Code` node aggregates multi-iteration batch outputs and S3 cleanup lists.
    - Configuration choices: Iterates safely across all loop runs using historical index lookups (`.all(0, run)`).
    - Connections: Inputs from `Check Expired Backups` and `Delete Expired Backup`; output to `Send Backup Report Email`.
  - **Send Backup Report Email**
    - Type and technical role: `Email Send` (SMTP) dispatches the text-formatted run status report.
    - Configuration choices: Plain text format, supports fallback sender addresses if `notify_from` is left blank.
    - Key expressions or variables: `={{ $json.body }}`, `={{ $json.subject }}`
    - Connections: Input from `Generate Backup Report`; output terminates workflow.
    - Edge cases/Failure types: Incorrect SMTP credentials, mail server rejection limits, or untrusted sender domains.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration overview. | None | None | Back up every repository you own... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Section note for schedule and settings. | None | None | Schedule and settings |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Section note for discovery phase. | None | None | Discover GitHub repositories |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Section note for repository iteration. | None | None | Back up each repository |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Section note for retention pruning. | None | None | Apply retention cleanup |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Section note for reporting phase. | None | None | Report backup results |
| When Nightly at 2 AM | `n8n-nodes-base.scheduleTrigger` | Triggers execution nightly at 02:00. | None | Set Backup Parameters | Schedule and settings |
| Set Backup Parameters | `n8n-nodes-base.set` | Declares variables for bucket, prefix, retention, and notifications. | When Nightly at 2 AM | Validate Backup Settings | Schedule and settings |
| Validate Backup Settings | `n8n-nodes-base.code` | Fails fast if placeholder defaults are detected. | Set Backup Parameters | Fetch GitHub Account Info | Schedule and settings |
| Fetch GitHub Account Info | `n8n-nodes-base.httpRequest` | Retrieves account type and metadata. | Validate Backup Settings | List GitHub Repos | Discover GitHub repositories |
| List GitHub Repos | `n8n-nodes-base.httpRequest` | Paginates through account repositories. | Fetch GitHub Account Info | Process Each Repository | Discover GitHub repositories |
| Process Each Repository | `n8n-nodes-base.splitInBatches` | Loops through repositories sequentially. | List GitHub Repos, Log Repository Backup Status | Download Repo Archive, List S3 Backups | Back up each repository |
| Download Repo Archive | `n8n-nodes-base.httpRequest` | Downloads tarball archives with retry options. | Process Each Repository | Upload Archive to S3, Log Repository Backup Status | Back up each repository |
| Upload Archive to S3 | `n8n-nodes-base.awsS3` | Uploads repo tarball to object storage. | Download Repo Archive | Log Repository Backup Status | Back up each repository |
| List S3 Backups | `n8n-nodes-base.awsS3` | Lists existing backup objects under prefix. | Process Each Repository | Filter Expired Backups | Apply retention cleanup |
| Filter Expired Backups | `n8n-nodes-base.code` | Identifies backups older than retention policy. | List S3 Backups | Check Expired Backups | Apply retention cleanup |
| Check Expired Backups | `n8n-nodes-base.if` | Ensures objects exist before triggering delete. | Filter Expired Backups | Delete Expired Backup, Generate Backup Report | Apply retention cleanup |
| Delete Expired Backup | `n8n-nodes-base.awsS3` | Deletes expired backups from S3 bucket. | Check Expired Backups | Generate Backup Report | Apply retention cleanup |
| Generate Backup Report | `n8n-nodes-base.code` | Aggregates all logs and formulates report. | Check Expired Backups, Delete Expired Backup | Send Backup Report Email | Report backup results |
| Send Backup Report Email | `n8n-nodes-base.emailSend` | Sends summary report via SMTP. | Generate Backup Report | None | Report backup results |
| Log Repository Backup Status | `n8n-nodes-base.code` | Records success, skip, or failure status per repo. | Download Repo Archive, Upload Archive to S3 | Process Each Repository | Back up each repository |
| Warning: deletes files | `n8n-nodes-base.stickyNote` | Warning reminder concerning retention deletions. | None | None | Deletes files... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**
   - Node Type: `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`)
   - Name: `When Nightly at 2 AM`
   - Configuration: Set interval rule to Cron Expression `0 2 * * *`.
2. **Create Settings Node**
   - Node Type: `Set` (`n8n-nodes-base.set`)
   - Name: `Set Backup Parameters`
   - Configuration: Add string parameters `github_owner` (default blank), `s3_bucket` (`your-backup-bucket`), `s3_prefix` (`github-backups`), `notify_email` (`you@example.com`), `notify_from` (blank), and number parameter `retention_days` (`30`).
3. **Create Validation Node**
   - Node Type: `Code` (`n8n-nodes-base.code`)
   - Name: `Validate Backup Settings`
   - Configuration: Add JavaScript validation logic checking for placeholder strings and throwing errors on invalid configurations.
4. **Create GitHub Account Info Node**
   - Node Type: `HTTP Request` (`n8n-nodes-base.httpRequest`)
   - Name: `Fetch GitHub Account Info`
   - Configuration: Set authentication to `GitHub API` credentials. Use expression to determine URL based on `github_owner`.
5. **Create List Repositories Node**
   - Node Type: `HTTP Request` (`n8n-nodes-base.httpRequest`)
   - Name: `List GitHub Repos`
   - Configuration: Configure pagination (`page` incremented by `$pageCount + 1`, complete when response length is 0) and query parameters (`per_page: 100`, `type: owner`). Authenticate via GitHub credentials.
6. **Create Batch Loop Node**
   - Node Type: `Split In Batches` (`n8n-nodes-base.splitInBatches`)
   - Name: `Process Each Repository`
   - Configuration: Default options (batch size 1).
7. **Create Download Archive Node**
   - Node Type: `HTTP Request` (`n8n-nodes-base.httpRequest`)
   - Name: `Download Repo Archive`
   - Configuration: Set URL to `https://api.github.com/repos/{{ $json.full_name }}/tarball`. Response format: `File`. Enable retries (3 attempts, 5000ms wait). Set `On Error` to continue using error output. Authenticate via GitHub credentials.
8. **Create Upload S3 Node**
   - Node Type: `AWS S3` (`n8n-nodes-base.awsS3`)
   - Name: `Upload Archive to S3`
   - Configuration: Operation: `Upload`. Bucket name from parameters. File name expression: `={{ $('Set Backup Parameters').first().json.s3_prefix }}/{{ $('Process Each Repository').item.json.name }}-$now.format('yyyyLLdd-HHmmss')}.tar.gz`. Configure AWS credentials.
9. **Create Status Logger Node**
   - Node Type: `Code` (`n8n-nodes-base.code`)
   - Name: `Log Repository Backup Status`
   - Configuration: Add JavaScript logic to parse successful runs, HTTP error codes, and empty repositories. Connect output back to `Process Each Repository`.
10. **Create List S3 Backups Node**
    - Node Type: `AWS S3` (`n8n-nodes-base.awsS3`)
    - Name: `List S3 Backups`
    - Configuration: Operation: `Get All`. Folder key from parameters. Enable `Always Output Data`. Configure AWS credentials. Connect input from completion of `Process Each Repository`.
11. **Create Filter Expired Node**
    - Node Type: `Code` (`n8n-nodes-base.code`)
    - Name: `Filter Expired Backups`
    - Configuration: Add retention cutoff calculation script filtering old files.
12. **Create Condition Check Node**
    - Node Type: `If` (`n8n-nodes-base.if`)
    - Name: `Check Expired Backups`
    - Configuration: Condition check that `{{ $json.key }}` is not empty.
13. **Create Delete Expired S3 Node**
    - Node Type: `AWS S3` (`n8n-nodes-base.awsS3`)
    - Name: `Delete Expired Backup`
    - Configuration: Operation: `Delete`. File key: `={{ $json.key }}`. Configure AWS credentials.
14. **Create Report Generator Node**
    - Node Type: `Code` (`n8n-nodes-base.code`)
    - Name: `Generate Backup Report`
    - Configuration: Script aggregating loop histories and S3 deletion results into summary objects and email body text.
15. **Create Email Send Node**
    - Node Type: `Email Send` (`n8n-nodes-base.emailSend`)
    - Name: `Send Backup Report Email`
    - Configuration: Configure SMTP credentials. Set `To Email`, `From Email`, subject, and text expressions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full bucket and IAM setup guide | [datadrifter.io setup guide](https://datadrifter.io/github-backup-s3-bucket-iam-setup/) |
| Creator attribution | Made by DataDrifter |