Create captioned vertical video highlights with Zvid, ElevenLabs and OpenRouter

https://n8nworkflows.xyz/workflows/create-captioned-vertical-video-highlights-with-zvid--elevenlabs-and-openrouter-20180


# Create captioned vertical video highlights with Zvid, ElevenLabs and OpenRouter

### 1. Workflow Overview

This workflow automates the process of transforming a long-form video (provided via an MP4 URL) into professional, captioned 1080x1920 vertical Shorts. It ingests a video, transcribes its audio with word-level timestamps using ElevenLabs, uses an LLM via OpenRouter to identify highlight segments, generates dynamic layout projects via Zvid, and either outputs editor drafts or fully rendered MP4 video links.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Handles incoming requests via Webhook or manual testing, normalizes source video metadata (handling Google Drive links and invalid landing pages), and establishes global configurations (branding, timeouts, clip dimensions).
- **1.2 Transcription & Analysis:** Downloads a byte-range sample of the video, executes Speech-to-Text via ElevenLabs Scribe, and builds an indexed transcript representation.
- **1.3 AI Highlight Selection:** Queries an LLM via OpenRouter to select non-overlapping, high-value clip ranges matching the configured duration and word counts.
- **1.4 Project Build & Validation:** Translates selected word ranges into exact timestamps, constructs a detailed Zvid render JSON project (complete with layouts, subtitles, branding, progress bar, and scrims), and validates it against the Zvid API.
- **1.5 Conditional Evaluation & Dry Run:** Evaluates the `dryRun` flag. If enabled, saves clips as Zvid editor drafts and returns credit quotes; if disabled, proceeds to production rendering.
- **1.6 Batch Rendering & Safety Management:** Sequentially processes assets through Zvid render submission, managing API capacity limits (HTTP 429) via bounded backoff strategies.
- **1.7 Polling & Completion Verification:** Polls active render jobs until completion, manages timeout boundaries, and extracts final deliverable video URLs.
- **1.8 Response & Asset Inspection:** Compiles final results into a structured webhook payload and allows optional binary file downloads for immediate review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Accepts incoming requests from automated triggers or manual testing, normalizes URL structures (such as Google Drive sharing links), validates video accessibility, and loads rendering configuration variables.
- **Nodes Involved:** `Test manually`, `Video webhook`, `Sample video`, `Config`, `Video request`
- **Node Details:**
  - **Test manually**
    - Type: `n8n-nodes-base.manualTrigger`
    - Role: Initiates workflow execution manually for testing.
    - Configuration: Default parameters.
    - Input/Output: None / Outputs execution trigger to `Sample video`.
  - **Video webhook**
    - Type: `n8n-nodes-base.webhook`
    - Role: Receives external HTTP POST requests containing target video parameters.
    - Configuration: Path: `video-to-shorts`, Method: `POST`, Response Mode: `responseNode`.
    - Input/Output: External HTTP POST / Outputs payload data to `Video request`.
  - **Sample video**
    - Type: `n8n-nodes-base.set`
    - Role: Provides a built-in sample MP4 URL and title when testing manually.
    - Configuration: Raw mode JSON output containing a default sample video URL.
    - Input/Output: `Test manually` / Outputs sample dataset to `Video request`.
  - **Video request**
    - Type: `n8n-nodes-base.code`
    - Role: Normalizes webhooks and manual inputs, filters out invalid URLs (e.g., YouTube or watch pages), and transforms Google Drive links into direct download links.
    - Configuration: JavaScript snippet handling URL cleaning and validation.
    - Input/Output: `Video webhook` or `Sample video` / Outputs normalized video metadata to `Config`.
  - **Config**
    - Type: `n8n-nodes-base.set`
    - Role: Establishes global parameters including API endpoints, clip counts, target durations, brand colors, typography, and execution modes (`dryRun`).
    - Configuration: Raw JSON object containing global settings.
    - Input/Output: `Video request` / Outputs configuration parameters to `Fetch video sample`.

#### 2.2 Transcription & Analysis
- **Overview:** Fetches a byte-range sample of the target video and submits it to ElevenLabs Scribe for high-precision word-level transcription.
- **Nodes Involved:** `Fetch video sample`, `Transcribe (Scribe)`, `Read transcript`
- **Node Details:**
  - **Fetch video sample**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Downloads the initial segment of the source video up to the maximum transcription size limit.
    - Configuration: Uses HTTP Range header derived from `maxTranscribeMB`, binary output mode, max 3 tries with exponential retry.
    - Input/Output: `Config` / Outputs binary file data (`data`) to `Transcribe (Scribe)`.
  - **Transcribe (Scribe)**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Sends the video binary sample to the ElevenLabs Speech-to-Text API.
    - Configuration: POST to `https://api.elevenlabs.io/v1/speech-to-text`, multipart form-data, uses generic HTTP Header Auth.
    - Input/Output: `Fetch video sample` / Outputs transcription JSON response to `Read transcript`.
  - **Read transcript**
    - Type: `n8n-nodes-base.code`
    - Role: Normalizes Scribe response data into clean, ASCII-sanitized word objects with exact timestamps and compiles an indexed prompt for the LLM.
    - Configuration: JavaScript text-cleaning and indexing engine.
    - Input/Output: `Transcribe (Scribe)` / Outputs word arrays and prompt text to `Pick highlights`.

#### 2.3 AI Highlight Selection
- **Overview:** Queries an OpenRouter LLM model to analyze the indexed transcript and select optimal, non-overlapping highlight ranges.
- **Nodes Involved:** `Pick highlights`, `Prepare clips`
- **Node Details:**
  - **Pick highlights**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Communicates with OpenRouter to process the structured prompt and return targeted segment indices.
    - Configuration: POST to `https://api.openrouter.ai/api/v1/chat/completions`, JSON body containing model selections, uses OpenRouter API credentials.
    - Input/Output: `Read transcript` / Outputs AI completion response to `Prepare clips`.
  - **Prepare clips**
    - Type: `n8n-nodes-base.code`
    - Role: Maps LLM-returned word indices back to exact start/end timestamps, enforces min/max clip duration constraints, and handles fallbacks if the model output is malformed.
    - Configuration: JavaScript validation and slicing engine.
    - Input/Output: `Pick highlights` / Outputs structured clip objects to `Build project JSON`.

#### 2.4 Project Build & Validation
- **Overview:** Assembles complete Zvid project definitions (visuals, captions, scrims, and progress bars) for each clip and validates them against the Zvid API.
- **Nodes Involved:** `Build project JSON`, `Validate project (free)`, `Check validation`
- **Node Details:**
  - **Build project JSON**
    - Type: `n8n-nodes-base.code`
    - Role: Translates clip parameters into comprehensive 1080x1920 Zvid project payloads incorporating watermarks, subtitles, and layout styling.
    - Configuration: JavaScript generator function (`buildShortsClip`).
    - Input/Output: `Prepare clips` / Outputs Zvid project payload to `Validate project (free)`.
  - **Validate project (free)**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Submits the project JSON to the Zvid validation endpoint to verify syntax, credit requirements, and configuration viability.
    - Configuration: Resource: `render`, Operation: `validate`, JSON payload source.
    - Input/Output: `Build project JSON` / Outputs validation response to `Check validation`.
  - **Check validation**
    - Type: `n8n-nodes-base.code`
    - Role: Evaluates validation results, throwing detailed errors if the payload is rejected or passing valid configurations forward.
    - Configuration: JavaScript response checker.
    - Input/Output: `Validate project (free)` / Outputs validated items to `Dry run?`.

#### 2.5 Conditional Evaluation & Dry Run
- **Overview:** Determines whether to execute a dry run (saving drafts and quoting credits) or proceed to production rendering.
- **Nodes Involved:** `Dry run?`, `Save draft to editor`, `Dry run summary`
- **Node Details:**
  - **Dry run?**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates the `dryRun` configuration flag.
    - Configuration: Checks `{{ $('Config').first().json.dryRun }}`.
    - Input/Output: `Check validation` / True branch goes to `Save draft to editor`; False branch goes to `Render one asset at a time`.
  - **Save draft to editor**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Saves clip configurations as editable projects inside Zvid.
    - Configuration: Resource: `project`, Operation: `create`, max 3 tries with error continuation.
    - Input/Output: `Dry run?` (True) / Outputs draft project IDs to `Dry run summary`.
  - **Dry run summary**
    - Type: `n8n-nodes-base.code`
    - Role: Compiles credit estimations, warning logs, and editor links into a dry-run report.
    - Configuration: JavaScript summary aggregator.
    - Input/Output: `Save draft to editor` / Outputs final report to `Respond with clips`.

#### 2.6 Batch Rendering & Safety Management
- **Overview:** Iterates through items sequentially and handles render job submissions with robust handling of API rate limits (HTTP 429).
- **Nodes Involved:** `Render one asset at a time`, `Prepare render attempt`, `Submit render`, `Check submission rejection`, `Wait for render capacity`
- **Node Details:**
  - **Render one asset at a time**
    - Type: `n8n-nodes-base.splitInBatches`
    - Role: Batches validated clips into individual items to ensure sequential, safe rendering.
    - Configuration: Batch size: 1.
    - Input/Output: `Dry run?` (False) / Outputs single asset item to `Prepare render attempt` (or `Run summary` when complete).
  - **Prepare render attempt**
    - Type: `n8n-nodes-base.code`
    - Role: Injects submission timestamps and attempt counters for deadline tracking.
    - Configuration: JavaScript snippet.
    - Input/Output: `Render one asset at a time` / Outputs metadata-enriched item to `Submit render`.
  - **Submit render**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Submits the render payload to the Zvid rendering engine asynchronously.
    - Configuration: Resource: `render`, Operation: `create`, `waitForCompletion: false`.
    - Input/Output: `Prepare render attempt` / Success outputs to `Attach job to clip`; Error outputs to `Check submission rejection`.
  - **Check submission rejection**
    - Type: `n8n-nodes-base.code`
    - Role: Inspects submission failures; isolates retryable rate-limit errors (HTTP 429) from other failures and calculates backoff intervals.
    - Configuration: JavaScript error parser and backoff calculator.
    - Input/Output: `Submit render` (Error branch) / Outputs wait duration to `Wait for render capacity`.
  - **Wait for render capacity**
    - Type: `n8n-nodes-base.wait`
    - Role: Pauses execution during rate-limiting periods before re-attempting submission.
    - Configuration: Dynamic duration based on `retrySeconds`.
    - Input/Output: `Check submission rejection` / Outputs delayed trigger back to `Submit render`.

#### 2.7 Polling & Completion Verification
- **Overview:** Attaches job identifiers to active items, periodically polls render status, and handles timeouts or failures.
- **Nodes Involved:** `Attach job to clip`, `Wait`, `Get render status`, `Merge job status`, `Render finished?`, `Still rendering?`
- **Node Details:**
  - **Attach job to clip**
    - Type: `n8n-nodes-base.code`
    - Role: Links the newly created render job ID to the active clip asset.
    - Configuration: JavaScript snippet.
    - Input/Output: `Submit render` (Success) / Outputs job-linked asset to `Wait`.
  - **Wait**
    - Type: `n8n-nodes-base.wait`
    - Role: Pauses execution between polling requests based on configured polling intervals.
    - Configuration: Unit: seconds, amount driven by `pollSeconds`.
    - Input/Output: `Attach job to clip` or `Still rendering?` / Outputs polling trigger to `Get render status`.
  - **Get render status**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Queries the Zvid API for current render job progress.
    - Configuration: Resource: `render`, Operation: `get`, uses `jobId`.
    - Input/Output: `Wait` / Outputs job status JSON to `Merge job status`.
  - **Merge job status**
    - Type: `n8n-nodes-base.code`
    - Role: Combines current job state details with original asset metadata.
    - Configuration: JavaScript merger.
    - Input/Output: `Get render status` / Outputs merged status to `Render finished?`.
  - **Render finished?**
    - Type: `n8n-nodes-base.if`
    - Role: Checks if the render state equals `completed`.
    - Configuration: Condition: `{{ $json.state === 'completed' }}`.
    - Input/Output: `Merge job status` / True branch routes to `Render one asset at a time` (next batch item); False branch routes to `Still rendering?`.
  - **Still rendering?**
    - Type: `n8n-nodes-base.code`
    - Role: Validates whether the render job has failed or exceeded the maximum timeout limit.
    - Configuration: JavaScript timeout/error validator.
    - Input/Output: `Render finished?` (False) / Outputs item back to `Wait` if still active.

#### 2.8 Response & Asset Inspection
- **Overview:** Aggregates completed render items, delivers the final webhook response, and provides optional binary video previews.
- **Nodes Involved:** `Run summary`, `Clips response`, `Respond with clips`, `Video ready to watch?`, `▶ Watch video`
- **Node Details:**
  - **Run summary**
    - Type: `n8n-nodes-base.code`
    - Role: Standardizes completed clip records with resulting media URLs and credit charges.
    - Configuration: JavaScript aggregator.
    - Input/Output: `Render one asset at a time` (Batch complete) / Outputs item list to `Clips response` and `Video ready to watch?`.
  - **Clips response**
    - Type: `n8n-nodes-base.code`
    - Role: Combines all rendered clip outputs into a single consolidated JSON webhook payload.
    - Configuration: JavaScript payload builder.
    - Input/Output: `Run summary` / Outputs consolidated payload to `Respond with clips`.
  - **Respond with clips**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Role: Returns the final webhook response to the original caller.
    - Configuration: Responds with first incoming item.
    - Input/Output: `Clips response` (or `Dry run summary`) / Outputs HTTP response.
  - **Video ready to watch?**
    - Type: `n8n-nodes-base.if`
    - Role: Verifies if deliverables are valid rendered URLs and dryRun is false.
    - Configuration: Condition checking URL format and dry run status.
    - Input/Output: `Run summary` / True branch routes to `▶ Watch video`.
  - **▶ Watch video**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Downloads the rendered MP4 video binary file for internal n8n execution inspection.
    - Configuration: GET request to `$json.videoUrl`, response format: file.
    - Input/Output: `Video ready to watch?` (True) / Outputs binary video file.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Test manually | `n8n-nodes-base.manualTrigger` | Initiates workflow execution manually for testing. | None | Sample video | ## 1. Choose settings and an entry point<br><br>Complete the overview setup first. Use the manual trigger for a test, then check the trigger settings and workflow timezone before activation. |
| Video webhook | `n8n-nodes-base.webhook` | Receives external HTTP POST requests containing target video parameters. | External HTTP POST | Video request | ## 1. Choose settings and an entry point<br><br>Complete the overview setup first. Use the manual trigger for a test, then check the trigger settings and workflow timezone before activation. |
| Sample video | `n8n-nodes-base.set` | Provides a built-in sample MP4 URL and title when testing manually. | Test manually | Video request | ## 1. Choose settings and an entry point<br><br>Complete the overview setup first. Use the manual trigger for a test, then check the trigger settings and workflow timezone before activation. |
| Video request | `n8n-nodes-base.code` | Normalizes webhooks and manual inputs, filters out invalid URLs, and handles Google Drive links. | Video webhook, Sample video | Config | ## 2. Accept a direct video URL<br><br>Test with Sample video or POST to the webhook. Choose fit to retain the frame or fill to crop for portrait output. |
| Config | `n8n-nodes-base.set` | Establishes global parameters including API endpoints, clip counts, target durations, brand colors, typography, and execution modes (`dryRun`). | Video request | Fetch video sample | ## 1. Choose settings and an entry point<br><br>Complete the overview setup first. Use the manual trigger for a test, then check the trigger settings and workflow timezone before activation. |
| Fetch video sample | `n8n-nodes-base.httpRequest` | Downloads the initial segment of the source video up to the maximum transcription size limit. | Config | Transcribe (Scribe) | ## 3. Transcribe the source sample<br><br>Only the fetched sample is transcribed and available for highlight selection. Speech-to-text can incur charges before rendering. |
| Transcribe (Scribe) | `n8n-nodes-base.httpRequest` | Sends the video binary sample to the ElevenLabs Speech-to-Text API. | Fetch video sample | Read transcript | ## 3. Transcribe the source sample<br><br>Only the fetched sample is transcribed and available for highlight selection. Speech-to-text can incur charges before rendering. |
| Read transcript | `n8n-nodes-base.code` | Normalizes Scribe response data into clean, ASCII-sanitized word objects with exact timestamps and compiles an indexed prompt for the LLM. | Transcribe (Scribe) | Pick highlights | ## 3. Transcribe the source sample<br><br>Only the fetched sample is transcribed and available for highlight selection. Speech-to-text can incur charges before rendering. |
| Pick highlights | `n8n-nodes-base.httpRequest` | Communicates with OpenRouter to process the structured prompt and return targeted segment indices. | Read transcript | Prepare clips | ## 4. Choose timed highlights<br><br>Keep the selected clips within the transcript’s source timestamps. Review that each excerpt preserves the speaker’s meaning. |
| Prepare clips | `n8n-nodes-base.code` | Maps LLM-returned word indices back to exact start/end timestamps, enforces min/max clip duration constraints, and handles fallbacks. | Pick highlights | Build project JSON | ## 4. Choose timed highlights<br><br>Keep the selected clips within the transcript’s source timestamps. Review that each excerpt preserves the speaker’s meaning. |
| Build project JSON | `n8n-nodes-base.code` | Translates clip parameters into comprehensive 1080x1920 Zvid project payloads incorporating watermarks, subtitles, and layout styling. | Prepare clips | Validate project (free) | ## 5. Build and validate the design<br><br>Prepare the media and video design, then validate the payload and credit quote. Resolve validation errors before submitting a render. |
| Validate project (free) | `@zvid/n8n-nodes-zvid.zvid` | Submits the project JSON to the Zvid validation endpoint to verify syntax, credit requirements, and configuration viability. | Build project JSON | Check validation | ## 5. Build and validate the design<br><br>Prepare the media and video design, then validate the payload and credit quote. Resolve validation errors before submitting a render. |
| Check validation | `n8n-nodes-base.code` | Evaluates validation results, throwing detailed errors if the payload is rejected or passing valid configurations forward. | Validate project (free) | Dry run? | ## 5. Build and validate the design<br><br>Prepare the media and video design, then validate the payload and credit quote. Resolve validation errors before submitting a render. |
| Dry run? | `n8n-nodes-base.if` | Evaluates the `dryRun` configuration flag. | Check validation | Save draft to editor, Render one asset at a time | ## 6. Choose an optional editor preview<br><br>dryRun=true returns an editor draft and credit quote. The default false branch renders and spends Zvid credits; upstream AI services may charge separately. |
| Save draft to editor | `@zvid/n8n-nodes-zvid.zvid` | Saves clip configurations as editable projects inside Zvid. | Dry run? | Dry run summary | ## 6. Choose an optional editor preview<br><br>dryRun=true returns an editor draft and credit quote. The default false branch renders and spends Zvid credits; upstream AI services may charge separately. |
| Dry run summary | `n8n-nodes-base.code` | Compiles credit estimations, warning logs, and editor links into a dry-run report. | Save draft to editor | Respond with clips | ## 6. Choose an optional editor preview<br><br>dryRun=true returns an editor draft and credit quote. The default false branch renders and spends Zvid credits; upstream AI services may charge separately. |
| Render one asset at a time | `n8n-nodes-base.splitInBatches` | Batches validated clips into individual items to ensure sequential, safe rendering. | Dry run?, Render finished? | Prepare render attempt, Run summary | ## 7. Submit one asset safely<br><br>Render one asset at a time. Retry only explicit HTTP 429 rejections after at least 30 seconds, up to timeoutMinutes. Other submission errors stop for inspection; an accepted job is never automatically resubmitted. |
| Prepare render attempt | `n8n-nodes-base.code` | Injects submission timestamps and attempt counters for deadline tracking. | Render one asset at a time | Submit render | ## 7. Submit one asset safely<br><br>Render one asset at a time. Retry only explicit HTTP 429 rejections after at least 30 seconds, up to timeoutMinutes. Other submission errors stop for inspection; an accepted job is never automatically resubmitted. |
| Submit render | `@zvid/n8n-nodes-zvid.zvid` | Submits the render payload to the Zvid rendering engine asynchronously. | Prepare render attempt, Wait for render capacity | Attach job to clip, Check submission rejection | ## 7. Submit one asset safely<br><br>Render one asset at a time. Retry only explicit HTTP 429 rejections after at least 30 seconds, up to timeoutMinutes. Other submission errors stop for inspection; an accepted job is never automatically resubmitted. |
| Check submission rejection | `n8n-nodes-base.code` | Inspects submission failures; isolates retryable rate-limit errors (HTTP 429) from other failures and calculates backoff intervals. | Submit render | Wait for render capacity | ## 7. Submit one asset safely<br><br>Render one asset at a time. Retry only explicit HTTP 429 rejections after at least 30 seconds, up to timeoutMinutes. Other submission errors stop for inspection; an accepted job is never automatically resubmitted. |
| Wait for render capacity | `n8n-nodes-base.wait` | Pauses execution during rate-limiting periods before re-attempting submission. | Check submission rejection | Submit render | ## 7. Submit one asset safely<br><br>Render one asset at a time. Retry only explicit HTTP 429 rejections after at least 30 seconds, up to timeoutMinutes. Other submission errors stop for inspection; an accepted job is never automatically resubmitted. |
| Attach job to clip | `n8n-nodes-base.code` | Links the newly created render job ID to the active clip asset. | Submit render | Wait | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Wait | `n8n-nodes-base.wait` | Pauses execution between polling requests based on configured polling intervals. | Attach job to clip, Still rendering? | Get render status | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Get render status | `@zvid/n8n-nodes-zvid.zvid` | Queries the Zvid API for current render job progress. | Wait | Merge job status | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Merge job status | `n8n-nodes-base.code` | Combines current job state details with original asset metadata. | Get render status | Render finished? | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Render finished? | `n8n-nodes-base.if` | Checks if the render state equals `completed`. | Merge job status | Render one asset at a time, Still rendering? | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Still rendering? | `n8n-nodes-base.code` | Validates whether the render job has failed or exceeded the maximum timeout limit. | Render finished? | Wait | ## 8. Render and wait for completion<br><br>Submit the render, then check its status until completion or timeout. Keep each result paired with its source item. Check Zvid before retrying a timed-out job. |
| Run summary | `n8n-nodes-base.code` | Standardizes completed clip records with resulting media URLs and credit charges. | Render one asset at a time | Clips response, Video ready to watch? | ## 9. Return the clip links<br><br>The webhook response contains the completed clip URLs. Allow sufficient caller timeout for transcription and rendering. |
| Clips response | `n8n-nodes-base.code` | Combines all rendered clip outputs into a single consolidated JSON webhook payload. | Run summary | Respond with clips | ## 9. Return the clip links<br><br>The webhook response contains the completed clip URLs. Allow sufficient caller timeout for transcription and rendering. |
| Respond with clips | `n8n-nodes-base.respondToWebhook` | Returns the final webhook response to the original caller. | Clips response, Dry run summary | None | ## 9. Return the clip links<br><br>The webhook response contains the completed clip URLs. Allow sufficient caller timeout for transcription and rendering. |
| Video ready to watch? | `n8n-nodes-base.if` | Verifies if deliverables are valid rendered URLs and dryRun is false. | Run summary | ▶ Watch video | ## 10. Review the finished media<br><br>Open Watch video → Binary → data → View. Select each output item for batches; image outputs open as images. Empty links and preview results skip the download. Each request times out after 30 seconds; completed links remain available. |
| ▶ Watch video | `n8n-nodes-base.httpRequest` | Downloads the rendered MP4 video binary file for internal n8n execution inspection. | Video ready to watch? | None | ## 10. Review the finished media<br><br>Open Watch video → Binary → data → View. Select each output item for batches; image outputs open as images. Empty links and preview results skip the download. Each request times out after 30 seconds; completed links remain available. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Install Community Node:** Navigate to **Settings → Community nodes** in your n8n instance and install `@zvid/n8n-nodes-zvid`.
2. **Create Credentials:** Set up three credentials:
   - **Zvid API:** API Key and base URL `https://api.zvid.io`.
   - **ElevenLabs Header Auth:** Header name `xi-api-key` with your ElevenLabs API key.
   - **OpenRouter API:** OpenRouter API key.
3. **Create Entry Nodes:**
   - Add a **Manual Trigger** (`Test manually`).
   - Add a **Webhook** (`Video webhook`) with path `video-to-shorts`, method `POST`, and response mode `responseNode`.
   - Add a **Set** node (`Sample video`) in raw mode with JSON output containing a sample `videoUrl` and `title`.
4. **Add Request Normalizer:**
   - Add a **Code** node (`Video request`). Connect both `Video webhook` and `Sample video` outputs to it. Insert the request normalization JavaScript logic to handle Google Drive URLs and validate direct video links.
5. **Add Configuration Node:**
   - Add a **Set** node (`Config`) configured in raw mode. Set global parameters such as `apiUrl`, `clipCount`, `clipSeconds`, `scribeModel`, `llmModel`, `brandAccent`, `dryRun`, etc. Connect `Video request` output to `Config`.
6. **Configure Transcription Stream:**
   - Add an **HTTP Request** node (`Fetch video sample`). Set method to GET, URL to `={{ $('Video request').first().json.videoUrl }}`, response format to file, and add a custom `Range` header using `maxTranscribeMB`. Connect `Config` to it.
   - Add an **HTTP Request** node (`Transcribe (Scribe)`). Method: POST, URL: `https://api.elevenlabs.io/v1/speech-to-text`, body type: multipart-form-data with file binary field, model ID, and word timestamps. Attach the ElevenLabs credential. Connect `Fetch video sample` to it.
   - Add a **Code** node (`Read transcript`) to parse Scribe output words into timestamps and construct the LLM prompt. Connect `Transcribe (Scribe)` to it.
7. **Configure AI Highlight Selection:**
   - Add an **HTTP Request** node (`Pick highlights`). Method: POST, URL: `https://api.openrouter.ai/api/v1/chat/completions`, JSON body incorporating prompt text and JSON response format. Attach OpenRouter credentials. Connect `Read transcript` to it.
   - Add a **Code** node (`Prepare clips`) to parse LLM selections into exact clip time windows. Connect `Pick highlights` to it.
8. **Configure Project Builder & Validator:**
   - Add a **Code** node (`Build project JSON`) running the `buildShortsClip` generation function for 1080x1920 portrait layouts, captions, scrims, and progress bars. Connect `Prepare clips` to it.
   - Add a **Zvid** node (`Validate project (free)`). Resource: `render`, Operation: `validate`, JSON source. Attach Zvid credentials. Connect `Build project JSON` to it.
   - Add a **Code** node (`Check validation`) to review validation responses. Connect `Validate project (free)` to it.
9. **Configure Dry Run / Render Branching:**
   - Add an **If** node (`Dry run?`) checking `{{ $('Config').first().json.dryRun }}`. Connect `Check validation` to it.
   - **True Branch:** Add a **Zvid** node (`Save draft to editor`) (Resource: `project`, Operation: `create`), followed by a **Code** node (`Dry run summary`) and **Respond with clips**.
   - **False Branch:** Add a **Split In Batches** node (`Render one asset at a time`) with batch size 1.
10. **Configure Rendering & Polling Loop:**
    - Add a **Code** node (`Prepare render attempt`). Connect to `Render one asset at a time`.
    - Add a **Zvid** node (`Submit render`) (Resource: `render`, Operation: `create`, `waitForCompletion: false`). Connect `Prepare render attempt` to it.
    - Handle submission errors: connect the error output of `Submit render` to a **Code** node (`Check submission rejection`), followed by a **Wait** node (`Wait for render capacity`), looping back into `Submit render`.
    - Handle successful submissions: connect the success output of `Submit render` to a **Code** node (`Attach job to clip`), followed by a **Wait** node (`Wait`), a **Zvid** node (`Get render status`), a **Code** node (`Merge job status`), and an **If** node (`Render finished?`).
    - If not finished, connect `Render finished?` false branch to a **Code** node (`Still rendering?`) which loops back to the `Wait` node. If finished, route back to `Render one asset at a time`.
11. **Configure Final Response & Review:**
    - When batch processing completes, route `Render one asset at a time` to a **Code** node (`Run summary`), then to `Clips response`, and finally to **Respond with clips**.
    - Optionally connect `Run summary` to an **If** node (`Video ready to watch?`) and an **HTTP Request** node (`▶ Watch video`) to download final binaries.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zvid Community Node Installation | [Settings → Community nodes Guide](https://github.com/Zvid-io/zvid-n8n) |
| Zvid API Keys Workspace Configuration | [Zvid Dashboard API Keys](https://app.zvid.io/api-keys) |
| Detailed Setup & Configuration Documentation | [Video-to-Shorts Workflow Documentation](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/video-to-shorts.md) |