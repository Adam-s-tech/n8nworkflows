Detect idle device activity from Prey screenshots with Claude Vision

https://n8nworkflows.xyz/workflows/detect-idle-device-activity-from-prey-screenshots-with-claude-vision-19686


# Detect idle device activity from Prey screenshots with Claude Vision

### 1. Workflow Overview

This workflow is designed to detect device idleness by evaluating recent screenshot reports fetched from the Prey tracking service. Using Anthropic’s Claude Vision model, it analyzes a series of chronologically ordered screenshots to determine if meaningful on-screen changes have occurred within a specified timeframe. It returns a structured JSON verdict containing an idle boolean flag and a summary of the visual analysis.

The workflow logic is categorized into the following functional blocks:

- **1.1 Input Reception & Configuration:** Receives triggers either via a production webhook (`POST /prey-idle-check`) or a manual execution, subsequently initializing global configuration parameters such as the analysis window, minimum report thresholds, and rate-limiting rules.
- **1.2 Cooldown, Reports & Window Filtering:** Enforces a workflow-level cooldown timer using static data. If the cooldown has expired, it queries the Prey API for device reports, filters out reports lacking screenshots, ensures sufficient data span and volume within the designated window, and evenly samples up to the maximum permitted number of images.
- **1.3 Download & Claude Vision Analysis:** Downloads the binary image data for each sampled screenshot, packages them into a single multimodal request payload for the Anthropic Messages API, queries Claude Vision, and parses the returned JSON verdict to output the final activity state.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** Establishes entry points for the workflow and defines core environmental variables and operational limits.
- **Nodes Involved:** 
  - `Webhook`
  - `When clicking ‘Execute workflow’`
  - `Set — Window config`

- **Node Details:**
  - **Webhook**
    - *Type and technical role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration choices:* Listens for incoming `POST` requests on path `prey-idle-check`.
    - *Input and output connections:* Input: None; Output: `Set — Window config`.
    - *Edge cases:* Missing headers or incorrect HTTP methods will cause the webhook to reject incoming payloads.
  - **When clicking ‘Execute workflow’**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` (Trigger)
    - *Configuration choices:* Manual execution for testing and debugging.
    - *Input and output connections:* Input: None; Output: `Set — Window config`.
  - **Set — Window config**
    - *Type and technical role:* `n8n-nodes-base.set` (Transform)
    - *Configuration choices:* Sets numeric parameters: `window_minutes` (30), `min_reports` (3), `min_span_minutes` (10), `max_images` (6), and `cooldown_minutes` (10).
    - *Key expressions or variables used:* Static numeric assignments.
    - *Input and output connections:* Input: `Webhook` or `When clicking ‘Execute workflow’`; Output: `Code — Cooldown gate`.

---

#### 1.2 Cooldown, Reports & Window Filtering
- **Overview:** Evaluates rate-limiting constraints, fetches raw reports from the Prey API, and validates data sufficiency based on time windows and sample thresholds.
- **Nodes Involved:**
  - `Code — Cooldown gate`
  - `IF — Cooldown passed?`
  - `NoOp — Result: skipped (cooldown)`
  - `Get device reports`
  - `Code — Filter report window`
  - `IF — Enough screenshots?`
  - `NoOp — Result: insufficient data`

- **Node Details:**
  - **Code — Cooldown gate**
    - *Type and technical role:* `n8n-nodes-base.code` (Logic)
    - *Configuration choices:* Reads workflow static global data to evaluate elapsed time since `last_analysis_at`.
    - *Input and output connections:* Input: `Set — Window config`; Output: `IF — Cooldown passed?`.
    - *Edge cases:* Static data resets or behaves differently in non-production environments; manual tests are never blocked by this gate.
  - **IF — Cooldown passed?**
    - *Type and technical role:* `n8n-nodes-base.if` (Routing)
    - *Configuration choices:* Checks if `skipped` is false.
    - *Input and output connections:* Input: `Code — Cooldown gate`; Output (True): `Get device reports`, Output (False): `NoOp — Result: skipped (cooldown)`.
  - **NoOp — Result: skipped (cooldown)**
    - *Type and technical role:* `n8n-nodes-base.noOp` (Terminal/Passthrough)
    - *Configuration choices:* Serves as a graceful exit point when execution is rate-limited.
    - *Input and output connections:* Input: `IF — Cooldown passed?` (False branch); Output: None.
  - **Get device reports**
    - *Type and technical role:* `@preyproject/n8n-nodes-prey.prey` (Integration)
    - *Configuration choices:* Operation `getDeviceReports` using Prey account credentials targeting a specific device ID (`dccc4e`).
    - *Input and output connections:* Input: `IF — Cooldown passed?` (True branch); Output: `Code — Filter report window`.
    - *Edge cases:* API rate limits, invalid device IDs, or authentication token expiration.
  - **Code — Filter report window**
    - *Type and technical role:* `n8n-nodes-base.code` (Logic/Transform)
    - *Configuration choices:* Filters reports with screenshots, anchors the window to the most recent report, validates minimum count and time spans, and evenly samples items down to `max_images`.
    - *Key expressions or variables used:* References upstream configuration via `$('Set — Window config').first().json`.
    - *Input and output connections:* Input: `Get device reports`; Output: `IF — Enough screenshots?`.
  - **IF — Enough screenshots?**
    - *Type and technical role:* `n8n-nodes-base.if` (Routing)
    - *Configuration choices:* Evaluates boolean flag `{{ $json.enough }}`.
    - *Input and output connections:* Input: `Code — Filter report window`; Output (True): `HTTP — Download screenshot`, Output (False): `NoOp — Result: insufficient data`.
  - **NoOp — Result: insufficient data**
    - *Type and technical role:* `n8n-nodes-base.noOp` (Terminal/Passthrough)
    - *Configuration choices:* Acts as an exit point when criteria for minimum reports or time spans fail.
    - *Input and output connections:* Input: `IF — Enough screenshots?` (False branch); Output: None.

---

#### 1.3 Download & Claude Vision Analysis
- **Overview:** Downloads binary image content for each sampled report, structures a multimodal prompt for Anthropic Claude, executes the API request, and parses the resulting output into an actionable idle status.
- **Nodes Involved:**
  - `HTTP — Download screenshot`
  - `Code — Build Claude vision request`
  - `HTTP — Claude vision (Anthropic)`
  - `Code — Activity verdict`

- **Node Details:**
  - **HTTP — Download screenshot**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (Integration)
    - *Configuration choices:* Downloads image files via GET requests using Prey credentials. Configured with up to 3 retries.
    - *Key expressions or variables used:* `={{ $json.screenshot_url }}`
    - *Input and output connections:* Input: `IF — Enough screenshots?` (True branch); Output: `Code — Build Claude vision request`.
    - *Edge cases:* Broken image links, timeouts, or remote server errors during binary fetching.
  - **Code — Build Claude vision request**
    - *Type and technical role:* `n8n-nodes-base.code` (Transform)
    - *Configuration choices:* Iterates over binary items, encodes buffers to base64, maps MIME types, and formats the JSON payload structure expected by the Anthropic Messages API.
    - *Input and output connections:* Input: `HTTP — Download screenshot`; Output: `HTTP — Claude vision (Anthropic)`.
  - **HTTP — Claude vision (Anthropic)**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (Integration)
    - *Configuration choices:* POST request to `https://api.anthropic.com/v1/messages` using Anthropic API credentials, setting header `anthropic-version: 2023-06-01`. Configured with up to 2 retries.
    - *Key expressions or variables used:* `={{ JSON.stringify($json.anthropic_body) }}`
    - *Input and output connections:* Input: `Code — Build Claude vision request`; Output: `Code — Activity verdict`.
    - *Edge cases:* API throttling, invalid model names, oversized payloads, or malformed authentication headers.
  - **Code — Activity verdict**
    - *Type and technical role:* `n8n-nodes-base.code` (Transform/Logic)
    - *Configuration choices:* Stamps the global static cooldown clock, parses the raw text output from Claude (stripping markdown code blocks if present), and formats the final verdict object.
    - *Input and output connections:* Input: `HTTP — Claude vision (Anthropic)`; Output: None (Workflow completion terminal node).
    - *Edge cases:* Unparseable model responses default to an `idle: null` state with a corresponding reason descriptor.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook` | `n8n-nodes-base.webhook` | Production Trigger | None | `Set — Window config` | ## 1. Trigger & configuration<br><br>Webhook for production, manual trigger for testing. The analysis window lives in the Set node. |
| `When clicking ‘Execute workflow’` | `n8n-nodes-base.manualTrigger` | Manual Trigger | None | `Set — Window config` | ## 1. Trigger & configuration<br><br>Webhook for production, manual trigger for testing. The analysis window lives in the Set node. |
| `Set — Window config` | `n8n-nodes-base.set` | Configuration Assignment | `Webhook`, `When clicking ‘Execute workflow’` | `Code — Cooldown gate` | ## 1. Trigger & configuration<br><br>Webhook for production, manual trigger for testing. The analysis window lives in the Set node. |
| `Code — Cooldown gate` | `n8n-nodes-base.code` | Rate Limit Logic | `Set — Window config` | `IF — Cooldown passed?` | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `IF — Cooldown passed?` | `n8n-nodes-base.if` | Execution Branching | `Code — Cooldown gate` | `Get device reports`, `NoOp — Result: skipped (cooldown)` | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `NoOp — Result: skipped (cooldown)` | `n8n-nodes-base.noOp` | Terminal Passthrough | `IF — Cooldown passed?` | None | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `Get device reports` | `@preyproject/n8n-nodes-prey.prey` | API Data Retrieval | `IF — Cooldown passed?` | `Code — Filter report window` | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `Code — Filter report window` | `n8n-nodes-base.code` | Data Filtering & Sampling | `Get device reports` | `IF — Enough screenshots?` | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `IF — Enough screenshots?` | `n8n-nodes-base.if` | Condition Evaluation | `Code — Filter report window` | `HTTP — Download screenshot`, `NoOp — Result: insufficient data` | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `NoOp — Result: insufficient data` | `n8n-nodes-base.noOp` | Terminal Passthrough | `IF — Enough screenshots?` | None | ## 2. Cooldown, reports & window filter<br><br>Skips the run if the last analysis was less than cooldown_minutes ago. Otherwise fetches the device reports and keeps the ones inside the window — requiring min_reports screenshots spanning at least min_span_minutes — then samples down to max_images. |
| `HTTP — Download screenshot` | `n8n-nodes-base.httpRequest` | Binary File Retrieval | `IF — Enough screenshots?` | `Code — Build Claude vision request` | ## 3. Download & Claude vision analysis<br><br>Downloads each screenshot as binary and sends the full set to the Anthropic API in a single request. |
| `Code — Build Claude vision request` | `n8n-nodes-base.code` | Payload Formatting | `HTTP — Download screenshot` | `HTTP — Claude vision (Anthropic)` | ## 3. Download & Claude vision analysis<br><br>Downloads each screenshot as binary and sends the full set to the Anthropic API in a single request. |
| `HTTP — Claude vision (Anthropic)` | `n8n-nodes-base.httpRequest` | AI Model Integration | `Code — Build Claude vision request` | `Code — Activity verdict` | ## 3. Download & Claude vision analysis<br><br>Downloads each screenshot as binary and sends the full set to the Anthropic API in a single request. |
| `Code — Activity verdict` | `n8n-nodes-base.code` | Response Parsing | `HTTP — Claude vision (Anthropic)` | None | ## 3. Download & Claude vision analysis<br><br>Downloads each screenshot as binary and sends the full set to the Anthropic API in a single request. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Nodes:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`). Set the HTTP Method to `POST` and the Path to `prey-idle-check`.
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) for testing purposes.
2. **Configure Global Settings:**
   - Add a **Set** node named `Set — Window config` (`n8n-nodes-base.set`).
   - Connect both the **Webhook** and **Manual Trigger** outputs to this node.
   - Assign the following number parameters: `window_minutes` (30), `min_reports` (3), `min_span_minutes` (10), `max_images` (6), and `cooldown_minutes` (10).
3. **Implement Rate Limiting:**
   - Add a **Code** node named `Code — Cooldown gate` (`n8n-nodes-base.code`) connected to `Set — Window config`. Paste logic to check global static data against `cooldown_minutes`.
   - Add an **IF** node named `IF — Cooldown passed?` (`n8n-nodes-base.if`) to check if `skipped` is false.
   - Add a **NoOp** node named `NoOp — Result: skipped (cooldown)` (`n8n-nodes-base.noOp`) and connect it to the `false` branch of the IF node.
4. **Fetch Device Reports:**
   - Add a Prey node named `Get device reports` (`@preyproject/n8n-nodes-prey.prey`). Connect it to the `true` branch of `IF — Cooldown passed?`.
   - Configure the operation as `getDeviceReports`, select your target device ID, and configure a **Prey account** credential.
5. **Filter and Sample Data:**
   - Add a **Code** node named `Code — Filter report window` (`n8n-nodes-base.code`) connected to `Get device reports`.
   - Add an **IF** node named `IF — Enough screenshots?` (`n8n-nodes-base.if`) checking `{{ $json.enough }}`.
   - Add a **NoOp** node named `NoOp — Result: insufficient data` (`n8n-nodes-base.noOp`) and connect it to the `false` branch.
6. **Download Images:**
   - Add an **HTTP Request** node named `HTTP — Download screenshot` (`n8n-nodes-base.httpRequest`) connected to the `true` branch of `IF — Enough screenshots?`.
   - Set URL to `={{ $json.screenshot_url }}`, configure response format as `File`, set authentication to Predefined Credential Type using the **Prey account** credential, and enable retries (3 attempts).
7. **Build and Execute AI Request:**
   - Add a **Code** node named `Code — Build Claude vision request` (`n8n-nodes-base.code`) connected to `HTTP — Download screenshot` to assemble base64 payloads and text prompts.
   - Add an **HTTP Request** node named `HTTP — Claude vision (Anthropic)` (`n8n-nodes-base.httpRequest`) connected to the build node.
   - Configure the POST request to `https://api.anthropic.com/v1/messages`, set headers (`anthropic-version: 2023-06-01`), pass JSON body stringified from upstream, and select an **Anthropic account** credential. Enable retries (2 attempts).
8. **Finalize Verdict:**
   - Add a **Code** node named `Code — Activity verdict` (`n8n-nodes-base.code`) connected to the Anthropic HTTP request to parse the JSON response and update the global static cooldown timer.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Prey Idle detection from screenshots (Overview, Setup, Customization) | Documented within the main sticky note block (`sticky-main`) of the workflow. |