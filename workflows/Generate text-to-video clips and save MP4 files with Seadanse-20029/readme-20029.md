Generate text-to-video clips and save MP4 files with Seadanse

https://n8nworkflows.xyz/workflows/generate-text-to-video-clips-and-save-mp4-files-with-seadanse-20029


# Generate text-to-video clips and save MP4 files with Seadanse

### 1. Workflow Overview

This workflow automates the generation of text-to-video clips using the Seadanse API. It provides a robust, state-polling mechanism to handle asynchronous video rendering jobs, securely fetches temporary download grants, and saves the resulting MP4 files to a designated local storage directory. 

The execution flow is structured into the following logical blocks:
- **1.1 Input Initialization & Validation:** Triggers the workflow execution manually, defines baseline generation properties, and validates parameters to ensure integrity before API submission.
- **1.2 Generation Submission:** Formats the request payload, sends it to the Seadanse generation endpoint with idempotency controls, and captures the initial job receipt.
- **1.3 Status Polling & Loop Controller:** Iteratively waits and checks the generation progress against a strict timeout window until the remote job completes or fails.
- **1.4 Secure Retrieval & Local Storage:** Extracts completed file metadata, requests a short-lived download grant, downloads the binary asset securely without leaking credentials, and writes the MP4 file to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization & Validation
**Overview:** This block initializes user-defined parameters for the video generation request and performs strict validation checks on configuration properties before interacting with external services.
**Nodes Involved:** `Start`, `Configure`, `Validate configuration`

- **Start**
  - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger Node)
  - *Configuration Choices:* Initiates execution manually on demand.
  - *Key Expressions or Variables:* None.
  - *Input/Output Connections:* Output connects to `Configure`.
  - *Edge Cases / Failure Types:* Manual trigger requires active operator interaction; no automatic error handling needed.

- **Configure**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
  - *Configuration Choices:* Establishes hardcoded workflow variables including text prompt, model selection (`seedance-2.0-mini`), duration, aspect ratio, resolution, max credits, unique idempotency request ID, image path, output directory, start timestamp, and a 30-minute timeout limit.
  - *Key Expressions or Variables:* Uses `Date.now()` to establish baseline execution timing.
  - *Input/Output Connections:* Input from `Start`; output connects to `Validate configuration`.
  - *Edge Cases / Failure Types:* Using a duplicate `requestId` incorrectly across distinct generations may cause idempotency conflicts.

- **Validate configuration**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation / Validation Node)
  - *Configuration Choices:* Validates that `requestId` has been customized and matches expected formatting, ensures prompt length is between 1 and 4000 characters, and verifies that `maxCredits` is a finite positive number.
  - *Key Expressions or Variables:* Evaluates `$json.requestId`, `$json.prompt`, and `$json.maxCredits`.
  - *Input/Output Connections:* Input from `Configure`; output connects to `Build generation`.
  - *Edge Cases / Failure Types:* Throws explicit runtime errors if default placeholder IDs or invalid parameter boundaries are detected.

---

#### 2.2 Generation Submission
**Overview:** This block prepares the clean payload parameters and dispatches a POST request to the Seadanse API to kick off asynchronous video rendering.
**Nodes Involved:** `Build generation`, `Generate video`, `Keep receipt`

- **Build generation**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
  - *Configuration Choices:* Extracts clean generation attributes from the initial configuration block, filtering out execution metadata.
  - *Key Expressions or Variables:* References `$('Configure').first().json`.
  - *Input/Output Connections:* Input from `Validate configuration`; output connects to `Generate video`.
  - *Edge Cases / Failure Types:* Missing configuration keys will cause property mapping errors.

- **Generate video**
  - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration Node)
  - *Configuration Choices:* Executes an HTTP POST request to `https://seadanse.com/api/v1/generations` with a 120-second timeout. Uses generic HTTP Header Authentication and passes the unique idempotency key via headers.
  - *Key Expressions or Variables:* Request body set via `={{ $json }}`, `Idempotency-Key` header populated via `={{ $('Configure').first().json.requestId }}`.
  - *Input/Output Connections:* Input from `Build generation`; output connects to `Keep receipt`.
  - *Edge Cases / Failure Types:* Authentication errors due to invalid API keys, rate limits, or network timeouts.

- **Keep receipt**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
  - *Configuration Choices:* Verifies API submission success and extracts the unique run identifier (`runId`).
  - *Key Expressions or Variables:* Evaluates `$json.success` and `$json.data?.runId`.
  - *Input/Output Connections:* Input from `Generate video`; output connects to `Wait 10 seconds`.
  - *Edge Cases / Failure Types:* Throws an error if the generation receipt is missing or indicates failure.

---

#### 2.3 Status Polling & Loop Controller
**Overview:** This block manages an asynchronous polling loop, pausing execution between status checks and evaluating job states against timeout constraints.
**Nodes Involved:** `Wait 10 seconds`, `Get generation`, `Check status`, `Finished`

- **Wait 10 seconds**
  - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control Node)
  - *Configuration Choices:* Pauses workflow execution for 10 seconds to allow remote video rendering progress.
  - *Key Expressions or Variables:* None.
  - *Input/Output Connections:* Inputs from `Keep receipt` and `Finished` (conditional loop); output connects to `Get generation`.
  - *Edge Cases / Failure Types:* None.

- **Get generation**
  - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration Node)
  - *Configuration Choices:* Performs an HTTP GET request to check the current status of the active job.
  - *Key Expressions or Variables:* Dynamic URL construction: `={{ 'https://seadanse.com/api/v1/generations/' + $('Keep receipt').first().json.runId }}`.
  - *Input/Output Connections:* Input from `Wait 10 seconds`; output connects to `Check status`.
  - *Edge Cases / Failure Types:* Transient network errors or invalid run IDs.

- **Check status**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation / Logic Node)
  - *Configuration Choices:* Validates job status validity (`queued`, `processing`, `succeeded`, `failed`). Enforces timeout thresholds based on elapsed execution time.
  - *Key Expressions or Variables:* Compares current timestamp against `startedAt` and `timeoutMinutes`.
  - *Input/Output Connections:* Input from `Get generation`; output connects to `Finished`.
  - *Edge Cases / Failure Types:* Throws runtime errors if the job status returns `failed` or if the maximum timeout window (30 minutes) is exceeded.

- **Finished**
  - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control Node)
  - *Configuration Choices:* Evaluates whether the generation status is strictly equal to `succeeded`.
  - *Key Expressions or Variables:* Condition checks `={{ $json.status === 'succeeded' }}`.
  - *Input/Output Connections:* Input from `Check status`; True branch outputs to `Select files`, False branch loops back to `Wait 10 seconds`.
  - *Edge Cases / Failure Types:* None.

---

#### 2.4 Secure Retrieval & Local Storage
**Overview:** This block extracts successful file references, obtains temporary authorized download grants, downloads the binary MP4 asset securely, and writes it to the local filesystem.
**Nodes Involved:** `Select files`, `Get download grant`, `Validate grant`, `Download MP4`, `Save MP4`

- **Select files**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
  - *Configuration Choices:* Parses the successful response items, maps file endpoints, and validates file URL schemas.
  - *Key Expressions or Variables:* Evaluates `$json.files` and maps `runId`, `fileId`, and `downloadUrl`.
  - *Input/Output Connections:* Input from `Finished` (True branch); output connects to `Get download grant`.
  - *Edge Cases / Failure Types:* Throws an error if no files are returned or if endpoints fail structural validation.

- **Get download grant**
  - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration Node)
  - *Configuration Choices:* Requests a temporary download grant URL. Configured with redirect tracking disabled (`followRedirects: false`) and response format set to text/full response to capture redirect headers.
  - *Key Expressions or Variables:* Dynamic URL construction: `={{ 'https://seadanse.com' + $json.downloadUrl }}`.
  - *Input/Output Connections:* Input from `Select files`; output connects to `Validate grant`.
  - *Edge Cases / Failure Types:* Expired temporary grants or unauthorized access attempts.

- **Validate grant**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
  - *Configuration Choices:* Ensures the download grant response returned an HTTP 307 redirect status and contains a valid secure location header.
  - *Key Expressions or Variables:* Validates `r.statusCode === 307` and checks `r.headers?.location`.
  - *Input/Output Connections:* Input from `Get download grant`; output connects to `Download MP4`.
  - *Edge Cases / Failure Types:* Throws an error if grant acquisition fails or redirection headers are missing.

- **Download MP4**
  - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration Node)
  - *Configuration Choices:* Downloads the binary file from the validated grant URL. **Crucial:** Authentication headers are omitted here to prevent sending API keys to third-party file storage buckets. Response format is explicitly configured as binary file output (`data`).
  - *Key Expressions or Variables:* Target URL set via `={{ $json.grantUrl }}`.
  - *Input/Output Connections:* Input from `Validate grant`; output connects to `Save MP4`.
  - *Edge Cases / Failure Types:* Large file download timeouts or expired grant links.

- **Save MP4**
  - *Type and Technical Role:* `n8n-nodes-base.readWriteFile` (File System Node)
  - *Configuration Choices:* Writes binary file data to the local disk using a dynamic filename combining output directory path, run ID, and file ID.
  - *Key Expressions or Variables:* File path set via `={{ $('Configure').first().json.outputDirectory + '/seadanse-' + $('Select files').item.json.runId + '-' + $('Select files').item.json.fileId + '.mp4' }}`. Operation set to `write`.
  - *Input/Output Connections:* Input from `Download MP4`; no outgoing connections.
  - *Edge Cases / Failure Types:* Insufficient disk write permissions or missing allowlisted directories on self-hosted instances.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start | `n8n-nodes-base.manualTrigger` | Trigger Node | None | Configure | Generate text-to-video clips with Seadanse<br><br>### How it works<br>For content creators using self-hosted n8n, this workflow turns a text prompt into a video and saves an MP4 locally. It submits once, checks progress every ten seconds for up to thirty minutes, then obtains a temporary download grant. The media download does not receive your API key. Output filenames include the run and file IDs.<br><br>### Setup<br>1. Use a paid-eligible [Seadanse](https://seadanse.com) account with sufficient credits. Create an API key with generation read/write scopes. Generation consumes credits; this template is free.<br>2. Create an n8n Header Auth credential: Authorization = Bearer YOUR_KEY. Select it on Generate video, Get generation, Get download grant. Never add it to Download MP4.<br>3. Mount a writable /files directory and allow n8n file access. Edit Configure: prompt, unique requestId, maxCredits and outputDirectory.<br>4. Check [current pricing and model options](https://seadanse.com/developers/api-guide.md), then execute. Mini, four seconds and 480p are sample settings.<br><br>### Customization<br>Change model settings and budget in Configure. Retry uncertain submissions with the same requestId and exact body. A timeout does not cancel the job. Recover by querying its run ID; download or save failures do not require another generation. Cloud storage adaptation is not included in this tested self-hosted export. |
| Configure | `n8n-nodes-base.code` | Data Transformation | Start | Validate configuration | ## 1. Configure inputs and credentials |
| Validate configuration | `n8n-nodes-base.code` | Data Transformation | Configure | Build generation | ## 1. Configure inputs and credentials |
| Build generation | `n8n-nodes-base.code` | Data Transformation | Validate configuration | Generate video | Submit once and retain the receipt |
| Generate video | `n8n-nodes-base.httpRequest` | External Integration | Build generation | Keep receipt | Submit once and retain the receipt |
| Keep receipt | `n8n-nodes-base.code` | Data Transformation | Generate video | Wait 10 seconds | Submit once and retain the receipt |
| Wait 10 seconds | `n8n-nodes-base.wait` | Flow Control | Keep receipt, Finished | Get generation | Wait and check the existing run |
| Get generation | `n8n-nodes-base.httpRequest` | External Integration | Wait 10 seconds | Check status | Wait and check the existing run |
| Check status | `n8n-nodes-base.code` | Data Transformation | Get generation | Finished | Wait and check the existing run |
| Finished | `n8n-nodes-base.if` | Flow Control | Check status | Select files, Wait 10 seconds | Wait and check the existing run |
| Select files | `n8n-nodes-base.code` | Data Transformation | Finished | Get download grant | Download privately and save the MP4 |
| Get download grant | `n8n-nodes-base.httpRequest` | External Integration | Select files | Validate grant | Download privately and save the MP4 |
| Validate grant | `n8n-nodes-base.code` | Data Transformation | Get download grant | Download MP4 | Download privately and save the MP4 |
| Download MP4 | `n8n-nodes-base.httpRequest` | External Integration | Validate grant | Save MP4 | Download privately and save the MP4 |
| Save MP4 | `n8n-nodes-base.readWriteFile` | File System | Download MP4 | None | Download privately and save the MP4 |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create the Manual Trigger Node:**
   - Add a **Manual Trigger** node. Name it `Start`. Leave default configurations.
2. **Add the Configuration Node:**
   - Add a **Code** node named `Configure`.
   - Set mode to `Run Once for All Items`.
   - Insert JavaScript code returning the initial JSON object containing `prompt`, `model`, `durationSeconds`, `resolution`, `aspectRatio`, `maxCredits`, `requestId`, `imagePath`, `outputDirectory`, `startedAt`, and `timeoutMinutes`.
   - Connect `Start` to `Configure`.
3. **Add Configuration Validation:**
   - Add a **Code** node named `Validate configuration`.
   - Add validation logic verifying `requestId` formatting, prompt character length (1–4000), and valid `maxCredits`.
   - Connect `Configure` to `Validate configuration`.
4. **Prepare the Generation Payload:**
   - Add a **Code** node named `Build generation` to extract request parameters from the configuration step.
   - Connect `Validate configuration` to `Build generation`.
5. **Submit the Generation Request:**
   - Add an **HTTP Request** node named `Generate video`.
   - Set method to `POST`, URL to `https://seadanse.com/api/v1/generations`, and timeout to `120000ms`.
   - Configure authentication: Select **Generic Credential Type** -> **HTTP Header Auth** (ensure you have configured your Seadanse API key credential beforehand).
   - Add header: `Idempotency-Key` set to `={{ $('Configure').first().json.requestId }}`.
   - Set body content type to JSON, populated via expression `={{ $json }}`.
   - Connect `Build generation` to `Generate video`.
6. **Capture the Run Receipt:**
   - Add a **Code** node named `Keep receipt` to validate success and return `$json.data`.
   - Connect `Generate video` to `Keep receipt`.
7. **Implement the Polling Loop (Wait & Status Check):**
   - Add a **Wait** node named `Wait 10 seconds`. Set unit to `seconds` and amount to `10`. Connect `Keep receipt` to it.
   - Add an **HTTP Request** node named `Get generation`. Set method to `GET` and URL to `={{ 'https://seadanse.com/api/v1/generations/' + $('Keep receipt').first().json.runId }}`. Configure HTTP Header Auth. Connect `Wait 10 seconds` to `Get generation`.
   - Add a **Code** node named `Check status` to validate run status responses and check timeout thresholds. Connect `Get generation` to `Check status`.
   - Add an **If** node named `Finished`. Configure condition: Left Value `={{ $json.status === 'succeeded' }}`, Operator `Boolean`, operation `true`. Connect `Check status` to `Finished`.
   - Configure the loop back: Connect the `False` branch of `Finished` back into `Wait 10 seconds`.
8. **Process Files & Obtain Download Grants:**
   - Add a **Code** node named `Select files` to extract file entries from the successful job payload. Connect the `True` branch of `Finished` to `Select files`.
   - Add an **HTTP Request** node named `Get download grant`. Set method to `GET`, URL to `={{ 'https://seadanse.com' + $json.downloadUrl }}`. Configure HTTP Header Auth. Under options, disable redirect following (`followRedirects: false`) and set response format to text/full response. Connect `Select files` to `Get download grant`.
   - Add a **Code** node named `Validate grant` to confirm an HTTP 307 status and check the location header. Connect `Get download grant` to `Validate grant`.
9. **Download and Save the Binary MP4 Asset:**
   - Add an **HTTP Request** node named `Download MP4`. Set method to `GET`, URL to `={{ $json.grantUrl }}`. **Important:** Do *not* attach authentication credentials here. Set response format to `File` with output property name `data`. Connect `Validate grant` to `Download MP4`.
   - Add a **Read/Write Files from Disk** node named `Save MP4`. Set operation to `Write`, file path to `={{ $('Configure').first().json.outputDirectory + '/seadanse-' + $('Select files').item.json.runId + '-' + $('Select files').item.json.fileId + '.mp4' }}`. Connect `Download MP4` to `Save MP4`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| API Guide & Current Pricing | [Seadanse API Guide](https://seadanse.com/developers/api-guide.md) |
| Security & API Key Management | [Seadanse Security Settings](https://seadanse.com/settings/security) |