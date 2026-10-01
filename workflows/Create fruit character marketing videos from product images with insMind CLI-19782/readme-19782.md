Create fruit character marketing videos from product images with insMind CLI

https://n8nworkflows.xyz/workflows/create-fruit-character-marketing-videos-from-product-images-with-insmind-cli-19782


# Create fruit character marketing videos from product images with insMind CLI

### 1. Workflow Overview

This workflow automates the generation of short fruit or vegetable character marketing videos from product images. It exposes a webhook endpoint to receive product images and customization parameters, interfaces with the `insMind CLI` to discover available models and tools, submits an asynchronous video generation task, polls for results, and returns the final video URL.

The logical execution follows a linear, sequential pattern divided into five distinct operational blocks:
- **1.1 Input Reception & Authentication:** Receives incoming POST requests via webhook and validates that the `insMind CLI` environment is authenticated.
- **1.2 Discovery & Command Construction:** Queries the CLI for available tools and models, validates inputs, and compiles a customized prompt and shell command.
- **1.3 Job Submission & Task ID Extraction:** Inspects the selected tool and dispatches the asynchronous generation job, capturing the assigned Task ID.
- **1.4 Execution Delay & Result Polling:** Pauses execution briefly to allow generation processing before querying the CLI for task status and output.
- **1.5 Response Formatting & Delivery:** Parses the output from the CLI task status check, determines success or failure, and responds to the webhook caller with status payloads and URLs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Authentication
- **Overview:** Receives raw webhook payloads containing image references and metadata, immediately followed by checking the local CLI execution environment's authentication status.
- **Nodes Involved:** `Create Video Request`, `Check insMind Login`
- **Node Details:**
  - **Create Video Request**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (v2) – Acts as the primary HTTP entry point for the workflow.
    - *Configuration Choices:* Configured to listen for `POST` requests on the relative path `insmind-fruit-product-video-plain` using manual response management (`responseMode: responseNode`).
    - *Key Expressions/Variables:* Expects parameters in the incoming JSON body: `product_image_path`, `product_image_url`, `product_name`, `campaign_goal`, `story_language`, `duration_seconds`, `aspect_ratio`, and `insmind_tool`.
    - *Connections:* Input: None (Trigger); Output: `Check insMind Login`.
    - *Version-specific requirements:* Uses version 2 webhook configurations.
    - *Edge cases:* Missing required image parameters result in downstream execution failures.
  - **Check insMind Login**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Executes shell commands on the host operating system.
    - *Configuration Choices:* Runs the status validation command `insmind-cli auth status`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: `Create Video Request`; Output: `Discover Tools`.
    - *Edge cases:* Fails if the `insMind CLI` is not installed or if the hosting container user lacks valid authentication credentials.

---

#### 2.2 Discovery & Command Construction
- **Overview:** Queries the local environment for operational tools and available AI models, then compiles a context-aware promotional prompt and executable shell script based on inputs.
- **Nodes Involved:** `Discover Tools`, `Discover Models`, `Build Submit Command`
- **Node Details:**
  - **Discover Tools**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell command execution node.
    - *Configuration Choices:* Runs `insmind-cli tool list` to fetch available tooling options.
    - *Connections:* Input: `Check insMind Login`; Output: `Discover Models`.
  - **Discover Models**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell command execution node.
    - *Configuration Choices:* Runs `insmind-cli model list` to retrieve model configurations.
    - *Connections:* Input: `Discover Tools`; Output: `Build Submit Command`.
  - **Build Submit Command**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Custom JavaScript evaluation block.
    - *Configuration Choices:* Parses webhook body data, cleans input strings, handles fallbacks for missing durations/aspect ratios, selects a compatible image-to-video tool dynamically, and builds a cinematic prompt structure.
    - *Key Expressions/Variables:* References data from `Create Video Request` and `Discover Tools`. Uses helper functions like `shellQuote()` and `chooseTool()`.
    - *Connections:* Input: `Discover Models`; Output: `Inspect Selected Tool`.
    - *Edge cases:* Throws an explicit error if neither `product_image_path` nor `product_image_url` is supplied, or if no compatible video tool can be parsed from the CLI tool list.

---

#### 2.3 Job Submission & Task ID Extraction
- **Overview:** Verifies the configuration of the targeted generation tool, submits the video generation job asynchronously, and extracts the unique Task ID from stdout/stderr.
- **Nodes Involved:** `Inspect Selected Tool`, `Submit Async Job`, `Extract Task ID`
- **Node Details:**
  - **Inspect Selected Tool**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell execution node.
    - *Configuration Choices:* Dynamically inspects configuration parameters for the selected tool using `insmind-cli tool get <selected_tool>`.
    - *Key Expressions/Variables:* Uses `={{ 'insmind-cli tool get ' + ''' + $json.selected_tool + ''' }}`.
    - *Connections:* Input: `Build Submit Command`; Output: `Submit Async Job`.
  - **Submit Async Job**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell execution node.
    - *Configuration Choices:* Executes the complete shell submission script formulated in the upstream code node.
    - *Key Expressions/Variables:* Uses `={{ $node["Build Submit Command"].json["submit_command"] }}`.
    - *Connections:* Input: `Inspect Selected Tool`; Output: `Extract Task ID`.
    - *Edge cases:* Generation limits, invalid file paths, or CLI service outages cause command non-zero exits.
  - **Extract Task ID**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – JavaScript processing node.
    - *Configuration Choices:* Scans combined stdout and stderr streams token-by-token for identifiers matching task definitions (`taskid`, `task_id`, `id`) or fallback UUID-like patterns.
    - *Key Expressions/Variables:* Reads `$json.stdout` and `$json.stderr`.
    - *Connections:* Input: `Submit Async Job`; Output: `Wait for Generation`.
    - *Edge cases:* Throws an error if no task ID string can be resolved from command feedback.

---

#### 2.4 Execution Delay & Result Polling
- **Overview:** Introduces a controlled delay to allow server-side processing before requesting task completion updates from the CLI.
- **Nodes Involved:** `Wait for Generation`, `Get Task Result`
- **Node Details:**
  - **Wait for Generation**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell execution node acting as a timer.
    - *Configuration Choices:* Executes `sleep 45` to pause workflow execution for 45 seconds.
    - *Connections:* Input: `Extract Task ID`; Output: `Get Task Result`.
  - **Get Task Result**
    - *Type and Technical Role:* `n8n-nodes-base.executeCommand` (v1) – Shell execution node.
    - *Configuration Choices:* Retrieves final or intermediate task status using the generated polling command.
    - *Key Expressions/Variables:* Uses `={{ $node["Extract Task ID"].json["task_get_command"] }}`.
    - *Connections:* Input: `Wait for Generation`; Output: `Parse Result`.

---

#### 2.5 Response Formatting & Delivery
- **Overview:** Evaluates task log outputs for downloadable media links, categorizes workflow processing states, and responds to the webhook caller with status objects and HTTP response codes.
- **Nodes Involved:** `Parse Result`, `Return Result`
- **Node Details:**
  - **Parse Result**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – JavaScript expression evaluator.
    - *Configuration Choices:* Tokenizes CLI response outputs to search for valid protocols (`http://` or `https://`) accompanied by media identifiers (`video`, `result`, `download`, `mp4`, `mov`, `webm`). Determines operational health flags (`success`, `workflow_status`, `result_url`, `raw_task_output`).
    - *Connections:* Input: `Get Task Result`; Output: `Return Result`.
  - **Return Result**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.1) – Webhook response node.
    - *Configuration Choices:* Returns structured JSON responses. Sets dynamic HTTP status codes based on task success (HTTP 200 for completed outputs, HTTP 202 for pending queues or failures).
    - *Key Expressions/Variables:* Response code uses `={{ $json.success ? 200 : 202 }}` and body uses `={{ $json }}`.
    - *Connections:* Input: `Parse Result`; Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview Note | `n8n-nodes-base.stickyNote` | Visual documentation block explaining purpose and referencing official insMind CLI docs (https://www.insmind.com/cli). | None | None | Overview<br>This template creates short fruit or vegetable character marketing videos from product images. It exposes a webhook for product image input, uses insMind CLI as the generation engine, submits an async video generation task, and returns a JSON response with the task outcome and result URL when available. Official insMind CLI docs: https://www.insmind.com/cli |
| Setup Note | `n8n-nodes-base.stickyNote` | Visual guide detailing requirements for self-hosted instances, Execute Command enablement, and CLI installation (https://www.insmind.com/cli/install.md). | None | None | Setup<br>Use a self-hosted n8n instance with Execute Command enabled. Install insmind-cli on the same worker or container where n8n runs. Follow the official install guide at https://www.insmind.com/cli/install.md. Log in on that machine or container user with insmind-cli auth login, then confirm insmind-cli auth status works before activating this workflow. |
| Input Note | `n8n-nodes-base.stickyNote` | Instructions describing payload formatting options for webhook endpoints (`product_image_path`, `product_image_url`, optional parameters). | None | None | Webhook input<br>Send a POST request with product_image_path or product_image_url. Optional fields include product_name, campaign_goal, story_language, duration_seconds, aspect_ratio, and insmind_tool. Local image paths must be readable by the n8n worker or container. If insmind_tool is omitted, the workflow chooses a compatible video tool from insMind CLI discovery output. |
| insMind Flow Note | `n8n-nodes-base.stickyNote` | Explains the structural execution flow including tool discovery, authentication checking, prompt assembly, and async task results retrieval. | None | None | insMind CLI flow<br>The workflow checks login status, runs tool discovery and model discovery, builds a marketing video prompt, inspects the selected tool, submits an async generation request, extracts the returned task ID, waits briefly, then requests the task result. This keeps the workflow aligned with currently available insMind CLI tools instead of hard-coding an uncertain model. |
| Output Note | `n8n-nodes-base.stickyNote` | Details expected JSON output parameters (`success`, `workflow_status`, `result_url`, `raw_task_output`) and error troubleshooting guidelines. | None | None | Output and troubleshooting<br>The webhook response contains success, workflow_status, result_url, and raw_task_output. A completed status means a result URL was found. A pending status means the task may still be processing and can be checked again in insMind CLI. A failed status usually means authentication, unreadable image input, an unsupported tool choice, or a generation error returned by the task. |
| Create Video Request | `n8n-nodes-base.webhook` | Receives incoming HTTP POST payloads containing input assets and generation parameters. | None | Check insMind Login | |
| Check insMind Login | `n8n-nodes-base.executeCommand` | Validates host CLI authentication credentials. | Create Video Request | Discover Tools | |
| Discover Tools | `n8n-nodes-base.executeCommand` | Queries available `insMind CLI` utilities. | Check insMind Login | Discover Models | |
| Discover Models | `n8n-nodes-base.executeCommand` | Queries available AI models via CLI. | Discover Tools | Build Submit Command | |
| Build Submit Command | `n8n-nodes-base.code` | Evaluates payload arguments, handles default fallbacks, selects tools dynamically, and builds the CLI invocation script. | Discover Models | Inspect Selected Tool | |
| Inspect Selected Tool | `n8n-nodes-base.executeCommand` | Fetches details and requirements for the selected tool. | Build Submit Command | Submit Async Job | |
| Submit Async Job | `n8n-nodes-base.executeCommand` | Submits asynchronous generation requests. | Inspect Selected Tool | Extract Task ID | |
| Extract Task ID | `n8n-nodes-base.code` | Parses command standard output and error streams to extract unique Task IDs. | Submit Async Job | Wait for Generation | |
| Wait for Generation | `n8n-nodes-base.executeCommand` | Pauses workflow execution for a fixed duration (`sleep 45`) to allow generation processing. | Extract Task ID | Get Task Result | |
| Get Task Result | `n8n-nodes-base.executeCommand` | Requests current task execution status and media availability. | Wait for Generation | Parse Result | |
| Parse Result | `n8n-nodes-base.code` | Parses result logs for downloadable media links and categorizes workflow health states. | Get Task Result | Return Result | |
| Return Result | `n8n-nodes-base.respondToWebhook` | Returns formatted JSON output and custom HTTP response codes to callers. | Parse Result | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow** on a self-hosted instance where the `Execute Command` node is fully enabled and permitted.
2. **Install and authenticate `insmind-cli`** on the underlying worker or container running the n8n process. Run `insmind-cli auth login` via container shell access, followed by `insmind-cli auth status` to confirm operational readiness. Reference the installation guide at `https://www.insmind.com/cli/install.md`.
3. **Node 1: Create Video Request**
   - Type: `n8n-nodes-base.webhook`
   - Parameters: Set **HTTP Method** to `POST`, **Path** to `insmind-fruit-product-video-plain`, and **Response Mode** to `Response Node`.
4. **Node 2: Check insMind Login**
   - Type: `n8n-nodes-base.executeCommand`
   - Parameters: Set **Command** to `insmind-cli auth status`.
   - Connect input to *Create Video Request*.
5. **Node 3: Discover Tools**
   - Type: `n8n-nodes-base.executeCommand`
   - Parameters: Set **Command** to `insmind-cli tool list`.
   - Connect input to *Check insMind Login*.
6. **Node 4: Discover Models**
   - Type: `n8n-nodes-base.executeCommand`
   - Parameters: Set **Command** to `insmind-cli model list`.
   - Connect input to *Discover Tools*.
7. **Node 5: Build Submit Command**
   - Type: `n8n-nodes-base.code`
   - Parameters: Set **Mode** to `Run Once for All Items` and paste the JavaScript payload mapping logic provided in the reference script to process payload variables (`product_image_path`, `product_image_url`, `duration_seconds`, `aspect_ratio`, `product_name`, `campaign_goal`, `story_language`, `insmind_tool`), assemble prompt arrays, and compute `submit_command`.
   - Connect input to *Discover Models*.
8. **Node 6: Inspect Selected Tool**
   - Type: `n8n-nodes-base.executeCommand`
   - Parameters: Set **Command** to `={{ 'insmind-cli tool get ' + ''' + $json.selected_tool + ''' }}`.
   - Connect input to *Build Submit Command*.
9. **Node 7: Submit Async Job**
   - Type: `n8n-nodes-base.executeCommand`
   - Parameters: Set **Command** to `={{ $node["Build Submit Command"].json["submit_command"] }}`.
   - Connect input to *Inspect Selected Tool*.
10. **Node 8: Extract Task ID**
    - Type: `n8n-nodes-base.code`
    - Parameters: Set **Mode** to `Run Once for All Items` and insert token parsing logic designed to extract task identifiers (`task_id` or UUID strings) from command output streams, outputting `task_id` and `task_get_command`.
    - Connect input to *Submit Async Job*.
11. **Node 9: Wait for Generation**
    - Type: `n8n-nodes-base.executeCommand`
    - Parameters: Set **Command** to `sleep 45`.
    - Connect input to *Extract Task ID*.
12. **Node 10: Get Task Result**
    - Type: `n8n-nodes-base.executeCommand`
    - Parameters: Set **Command** to `={{ $node["Extract Task ID"].json["task_get_command"] }}`.
    - Connect input to *Wait for Generation*.
13. **Node 11: Parse Result**
    - Type: `n8n-nodes-base.code`
    - Parameters: Set **Mode** to `Run Once for All Items` and insert script logic scanning command strings for valid URLs containing media formats (`mp4`, `mov`, `webm`), returning structured flags (`success`, `workflow_status`, `result_url`, `raw_task_output`).
    - Connect input to *Get Task Result*.
14. **Node 12: Return Result**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Parameters: Set **Respond With** to `JSON`, set **Response Body** expression to `={{ $json }}`, and set **Response Code** expression under options to `={{ $json.success ? 200 : 202 }}`.
    - Connect input to *Parse Result*.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official insMind CLI Documentation | [https://www.insmind.com/cli](https://www.insmind.com/cli) |
| CLI Installation Guide | [https://www.insmind.com/cli/install.md](https://www.insmind.com/cli/install.md) |