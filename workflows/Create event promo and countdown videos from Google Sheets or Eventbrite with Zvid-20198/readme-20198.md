Create event promo and countdown videos from Google Sheets or Eventbrite with Zvid

https://n8nworkflows.xyz/workflows/create-event-promo-and-countdown-videos-from-google-sheets-or-eventbrite-with-zvid-20198


# Create event promo and countdown videos from Google Sheets or Eventbrite with Zvid

### 1. Workflow Overview

This workflow automates the creation of vertical event promotional and countdown reminder videos (announcements, 7-day reminders, and final calls) using data from Google Sheets or Eventbrite, rendered via Zvid. It is tailored for event organizers seeking to automate video marketing campaigns with configurable branding, music, and scheduling.

The logical flow is divided into six main blocks:
- **1.1 Input Reception & Music Check:** Initializes execution parameters via manual or scheduled triggers, establishes global configuration settings, and verifies the accessibility and size limits of background music assets.
- **1.2 Event Data Extraction & Urgency Evaluation:** Connects to either Google Sheets or the Eventbrite API, filters out past events, computes date differences, and selects the most urgent video type due for rendering based on predefined milestones.
- **1.3 Project Construction & Validation:** Programmatically builds a multi-scene Zvid project JSON (Hero/Alert, Detail Cards, and Call-to-Action) and validates it against Zvid's free validation API.
- **1.4 Rendering & Asynchronous Polling:** Handles execution routing based on dry-run settings, submits paid render jobs, and asynchronously polls Zvid for completion with built-in timeout and failure handling.
- **1.5 Data Write-Back & State Recording:** Formats completion results and updates the source Google Sheet or internal workflow ledger to prevent duplicate renders.
- **1.6 Media Preview & Retrieval:** Conditionally downloads and exposes the final rendered MP4 video binary for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Music Check
This block initializes the workflow execution, defines global campaign configurations, and checks the health of the background music URL.

- **Nodes Involved:**
  - `Test manually` (n8n-nodes-base.manualTrigger)
  - `Every day at 8am` (n8n-nodes-base.scheduleTrigger)
  - `Config` (n8n-nodes-base.set)
  - `Check music` (n8n-nodes-base.httpRequest)
  - `Music guard` (n8n-nodes-base.code)

- **Node Details:**
  - **Test manually**
    - *Type and Role:* Manual Trigger. Allows on-demand execution for testing.
    - *Configuration:* Default parameters.
    - *Connections:* Output to `Config`.
    - *Edge Cases:* None.
  - **Every day at 8am**
    - *Type and Role:* Schedule Trigger. Automatically starts the workflow daily at 08:00.
    - *Configuration:* Interval set to trigger at hour 8.
    - *Connections:* Output to `Config`.
    - *Edge Cases:* Ensure server timezone aligns with expected execution hours or adjust via `timezoneOffsetHours` in Config.
  - **Config**
    - *Type and Role:* Set Node (Raw JSON mode). Establishes global parameters including API endpoints, data source (`sheet` or `eventbrite`), brand styling, typography, fallback media, and render limits.
    - *Configuration:* Raw JSON output containing branding properties, color schemes (`baseColor`, `accentColor`), timing rules (`firstReminderDays`), and operational flags (`dryRun`, `maxRendersPerRun`).
    - *Connections:* Input from triggers; output to `Check music`.
    - *Edge Cases:* Invalid hex color codes or malformed JSON syntax will cause downstream rendering failures.
  - **Check music**
    - *Type and Role:* HTTP Request. Performs a `HEAD` request to verify the availability and content-length of the background music file.
    - *Configuration:* URL uses expression `{{ $('Config').first().json.musicUrl }}`; method `HEAD`; timeout 15,000ms; configured to never error on failure (`alwaysOutputData: true`).
    - *Connections:* Input from `Config`; output to `Music guard`.
    - *Edge Cases:* Unreachable URLs or servers blocking `HEAD` requests require graceful fallback handling.
  - **Music guard**
    - *Type and Role:* Code Node (JavaScript). Validates HTTP status codes and file size limits against `maxMusicBytes` (default 5MB).
    - *Configuration:* Custom JS parsing response headers and enforcing size/accessibility constraints.
    - *Connections:* Input from `Check music`; output to `Source?`.
    - *Edge Cases:* If music is oversized or unreachable, it falls back to rendering without a music bed rather than failing the entire run.

---

#### 2.2 Event Data Extraction & Urgency Evaluation
This block queries the chosen data source, normalizes event structures, calculates calendar day differences, and determines which single video type is due for production.

- **Nodes Involved:**
  - `Source?` (n8n-nodes-base.if)
  - `Fetch Eventbrite events` (n8n-nodes-base.httpRequest)
  - `Read events sheet` (n8n-nodes-base.googleSheets)
  - `Which renders today?` (n8n-nodes-base.code)
  - `Anything due?` (n8n-nodes-base.if)
  - `Nothing due today` (n8n-nodes-base.code)

- **Node Details:**
  - **Source?**
    - *Type and Role:* If Node. Routes execution based on whether the data source is set to `eventbrite` or `sheet`.
    - *Configuration:* Condition checks `{{ $('Config').first().json.source === 'eventbrite' }}`.
    - *Connections:* Input from `Music guard`; outputs to `Fetch Eventbrite events` (True) or `Read events sheet` (False).
    - *Edge Cases:* Invalid source string defaults to Google Sheets.
  - **Fetch Eventbrite events**
    - *Type and Role:* HTTP Request. Fetches the first 50 live events from the configured Eventbrite organization.
    - *Configuration:* Uses HTTP Bearer Auth; queries Eventbrite API v3 endpoint with pagination and venue expansion. Retries up to 3 times on failure.
    - *Connections:* Input from `Source?` (True); output to `Which renders today?`.
    - *Edge Cases:* Authentication expiration or API rate limits.
  - **Read events sheet**
    - *Type and Role:* Google Sheets Node. Reads rows from the designated event calendar spreadsheet.
    - *Configuration:* Resource `sheet`, operation `read`. Retries up to 3 times on failure.
    - *Connections:* Input from `Source?` (False); output to `Which renders today?`.
    - *Edge Cases:* Missing column headers or unlinked OAuth credentials will throw errors.
  - **Which renders today?**
    - *Type and Role:* Code Node (JavaScript). Normalizes event payloads, handles timezone offsets, calculates day differences, and evaluates which video variant (`promo`, `7day`, or `1day`) is due.
    - *Configuration:* Enforces rendering limits per run (`maxRendersPerRun`) and prioritizes the most urgent items.
    - *Connections:* Input from `Fetch Eventbrite events` or `Read events sheet`; output to `Anything due?`.
    - *Edge Cases:* Malformed date ISO strings are skipped and reported in the logs.
  - **Anything due?**
    - *Type and Role:* If Node. Verifies whether any events require video generation during the current run.
    - *Configuration:* Condition checks `{{ $json.due }}`.
    - *Connections:* Input from `Which renders today?`; outputs to `Build project JSON` (True) or `Nothing due today` (False).
    - *Edge Cases:* Normal operational state when schedules have no upcoming milestones.
  - **Nothing due today**
    - *Type and Role:* Code Node (JavaScript). Formats a clean success message when no events are due for video rendering.
    - *Configuration:* Summarizes checked events and skipped records.
    - *Connections:* Input from `Anything due?` (False); terminal branch.
    - *Edge Cases:* None.

---

#### 2.3 Project Construction & Validation
This block programmatically constructs the multi-scene Zvid project payload and validates it against Zvid's API before rendering.

- **Nodes Involved:**
  - `Build project JSON` (n8n-nodes-base.code)
  - `Validate project (free)` (@zvid/n8n-nodes-zvid.zvid)
  - `Check validation` (n8n-nodes-base.code)
  - `Dry run?` (n8n-nodes-base.if)
  - `Save draft to editor` (@zvid/n8n-nodes-zvid.zvid)
  - `Dry run summary` (n8n-nodes-base.code)

- **Node Details:**
  - **Build project JSON**
    - *Type and Role:* Code Node (JavaScript). Assembles a complete 1080x1920 vertical video project payload including SVG graphics, typography scaling, scene transitions, and optional audio tracks.
    - *Configuration:* Encapsulates complex layout calculations for promo, 7-day countdown, and final call scenes.
    - *Connections:* Input from `Anything due?` (True); output to `Validate project (free)`.
    - *Edge Cases:* Text overflow or missing mandatory fields (e.g., `EventName`) trigger explicit script errors.
  - **Validate project (free)**
    - *Type and Role:* Zvid Community Node. Submits the project payload to Zvid's validation endpoint to check for schema errors without incurring rendering charges.
    - *Configuration:* Resource `render`, operation `validate`.
    - *Connections:* Input from `Build project JSON`; output to `Check validation`.
    - *Edge Cases:* Network timeouts or invalid project schemas.
  - **Check validation**
    - *Type and Role:* Code Node (JavaScript). Inspects validation results and throws detailed error messages listing invalid fields if validation fails.
    - *Configuration:* Extracts validation errors or passes through credit cost estimations and warnings.
    - *Connections:* Input from `Validate project (free)`; output to `Dry run?`.
    - *Edge Cases:* Zvid API returning non-200 status codes.
  - **Dry run?**
    - *Type and Role:* If Node. Branches execution based on the `dryRun` configuration flag.
    - *Configuration:* Condition checks `{{ $('Config').first().json.dryRun }}`.
    - *Connections:* Input from `Check validation`; outputs to `Save draft to editor` (True) or `Submit render` (False).
    - *Edge Cases:* Draft creation requires Zvid node support for Project Creation (not supported in v0.1.8).
  - **Save draft to editor**
    - *Type and Role:* Zvid Community Node. Saves the project as a draft within the Zvid editor environment when dry run is enabled.
    - *Configuration:* Resource `project`, operation `create`.
    - *Connections:* Input from `Dry run?` (True); output to `Dry run summary`.
    - *Edge Cases:* Disabled or unsupported in older Zvid node versions.
  - **Dry run summary**
    - *Type and Role:* Code Node (JavaScript). Generates a summary report containing editor preview links for draft projects.
    - *Configuration:* Aggregates draft IDs and metadata.
    - *Connections:* Input from `Save draft to editor`; terminal branch for dry runs.
    - *Edge Cases:* None.

---

#### 2.4 Rendering & Asynchronous Polling
This block submits paid video render jobs to Zvid and manages asynchronous status polling until completion or timeout.

- **Nodes Involved:**
  - `Submit render` (@zvid/n8n-nodes-zvid.zvid)
  - `Attach job to event` (n8n-nodes-base.code)
  - `Wait` (n8n-nodes-base.wait)
  - `Get render status` (@zvid/n8n-nodes-zvid.zvid)
  - `Merge job status` (n8n-nodes-base.code)
  - `Render finished?` (n8n-nodes-base.if)
  - `Still rendering?` (n8n-nodes-base.code)

- **Node Details:**
  - **Submit render**
    - *Type and Role:* Zvid Community Node. Submits a paid video rendering job asynchronously.
    - *Configuration:* Resource `render`, operation `create`, render type `video`, `waitForCompletion: false`. Retries up to 3 times.
    - *Connections:* Input from `Dry run?` (False); output to `Attach job to event`.
    - *Edge Cases:* Insufficient Zvid credits will cause the API to reject the submission.
  - **Attach job to event**
    - *Type and Role:* Code Node (JavaScript). Attaches the returned `jobId` back to the event tracking object and strips heavy payload data to optimize memory usage during polling loops.
    - *Configuration:* Maps input items and appends `jobId`.
    - *Connections:* Input from `Submit render`; output to `Wait`.
    - *Edge Cases:* Missing job IDs throw an immediate error.
  - **Wait**
    - *Type and Role:* Wait Node. Pauses execution between status poll requests.
    - *Configuration:* Duration based on `pollSeconds` from Config (default 10 seconds).
    - *Connections:* Input from `Attach job to event` or `Still rendering?`; output to `Get render status`.
    - *Edge Cases:* None.
  - **Get render status**
    - *Type and Role:* Zvid Community Node. Queries the current status of a rendering job using its `jobId`.
    - *Configuration:* Resource `render`, operation `get`, `waitForCompletion: false`. Retries up to 3 times on failure.
    - *Connections:* Input from `Wait`; output to `Merge job status`.
    - *Edge Cases:* API timeouts or invalid job IDs.
  - **Merge job status**
    - *Type and Role:* Code Node (JavaScript). Combines the latest job status response with the carried event context.
    - *Configuration:* Zips input items by array index.
    - *Connections:* Input from `Get render status`; output to `Render finished?`.
    - *Edge Cases:* Array mismatch if items drop out during polling.
  - **Render finished?**
    - *Type and Role:* If Node. Checks if the render state has successfully completed.
    - *Configuration:* Condition checks `{{ $json.state === 'completed' }}`.
    - *Connections:* Input from `Merge job status`; outputs to `Prepare write-back` (True) or `Still rendering?` (False).
    - *Edge Cases:* Handles intermediate states such as `queued` or `rendering`.
  - **Still rendering?**
    - *Type and Role:* Code Node (JavaScript). Enforces timeout limits and checks for render failures.
    - *Configuration:* Calculates maximum poll iterations based on `timeoutMinutes` and `pollSeconds`. Throws an error if rendering fails or times out.
    - *Connections:* Input from `Render finished?` (False); output to `Wait` (loopback).
    - *Edge Cases:* Exceeding the timeout limit aborts the loop without updating the sheet, allowing retry on the next scheduled run.

---

#### 2.5 Data Write-Back & State Recording
This block formats completed render results and updates the Google Sheet or internal workflow ledger.

- **Nodes Involved:**
  - `Prepare write-back` (n8n-nodes-base.code)
  - `Sheet mode?` (n8n-nodes-base.if)
  - `Update event row` (n8n-nodes-base.googleSheets)
  - `Run summary` (n8n-nodes-base.code)

- **Node Details:**
  - **Prepare write-back**
    - *Type and Role:* Code Node (JavaScript). Extracts the final video URL from the render result and maps it to the correct column (`PromoUrl`, `Video7d`, or `Video1d`) while preserving existing column data.
    - *Configuration:* Updates status values and records sent states in workflow static data for Eventbrite mode.
    - *Connections:* Input from `Render finished?` (True); output to `Sheet mode?`.
    - *Edge Cases:* Missing URLs in completed render objects throw an error.
  - **Sheet mode?**
    - *Type and Role:* If Node. Determines whether to update a Google Sheet row or bypass sheet updates (for Eventbrite mode).
    - *Configuration:* Condition checks source type and presence of `rowNumber`.
    - *Connections:* Input from `Prepare write-back`; outputs to `Update event row` (True) or `Run summary` (False).
    - *Edge Cases:* None.
  - **Update event row**
    - *Type and Role:* Google Sheets Node. Updates the corresponding row with the new video URL and status value.
    - *Configuration:* Resource `sheet`, operation `update`, mapping columns by `row_number`. Retries up to 3 times on failure.
    - *Connections:* Input from `Sheet mode?` (True); output to `Run summary`.
    - *Edge Cases:* Sheet lock conflicts or credential revocation.
  - **Run summary**
    - *Type and Role:* Code Node (JavaScript). Compiles a final execution report summarizing generated videos, consumption metrics, and storage locations.
    - *Configuration:* Aggregates metrics for downstream preview steps.
    - *Connections:* Input from `Sheet mode?` (False) or `Update event row`; output to `Video ready to watch?`.
    - *Edge Cases:* None.

---

#### 2.6 Media Preview & Retrieval
This block conditionally downloads the rendered MP4 video binary for review.

- **Nodes Involved:**
  - `Video ready to watch?` (n8n-nodes-base.if)
  - `▶ Watch video` (n8n-nodes-base.httpRequest)

- **Node Details:**
  - **Video ready to watch?**
    - *Type and Role:* If Node. Validates whether the video is ready for download inspection.
    - *Configuration:* Strict condition checks `{{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
    - *Connections:* Input from `Run summary`; outputs to `▶ Watch video` (True) or terminal branch (False).
    - *Edge Cases:* Dry runs or missing URLs bypass the download step.
  - **▶ Watch video**
    - *Type and Role:* HTTP Request. Downloads the finished MP4 video file into binary workflow data.
    - *Configuration:* URL uses `{{ $json.videoUrl }}`; timeout 30,000ms; response format set to file with output property `data`. Retries up to 3 times.
    - *Connections:* Input from `Video ready to watch?` (True); terminal branch.
    - *Edge Cases:* Download failures do not invalidate the render URL, which remains accessible via the run summary.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Test manually | n8n-nodes-base.manualTrigger | Manual execution entry point | None | Config | Workflow overview and setup |
| Every day at 8am | n8n-nodes-base.scheduleTrigger | Daily automated trigger at 8 AM | None | Config | Workflow overview and setup |
| Config | n8n-nodes-base.set | Defines global configuration and campaign settings | Test manually, Every day at 8am | Check music | 1. Set up and check music |
| Check music | n8n-nodes-base.httpRequest | Performs HEAD request to check music URL availability | Config | Music guard | 1. Set up and check music |
| Music guard | n8n-nodes-base.code | Validates music file size and reachability | Check music | Source? | 1. Set up and check music |
| Source? | n8n-nodes-base.if | Routes execution between Eventbrite and Google Sheets | Music guard | Fetch Eventbrite events, Read events sheet | 2. Choose the event due today |
| Fetch Eventbrite events | n8n-nodes-base.httpRequest | Fetches live events from Eventbrite API | Source? | Which renders today? | 2. Choose the event due today |
| Read events sheet | n8n-nodes-base.googleSheets | Reads event calendar from Google Sheets | Source? | Which renders today? | 2. Choose the event due today |
| Which renders today? | n8n-nodes-base.code | Normalizes events and calculates due video types | Fetch Eventbrite events, Read events sheet | Anything due? | 2. Choose the event due today |
| Anything due? | n8n-nodes-base.if | Checks if any videos are due for rendering | Which renders today? | Build project JSON, Nothing due today | 2. Choose the event due today |
| Nothing due today | n8n-nodes-base.code | Handles cases where no events require videos | Anything due? | None (Terminal) | 2. Choose the event due today |
| Build project JSON | n8n-nodes-base.code | Builds multi-scene Zvid video project JSON | Anything due? | Validate project (free) | 3. Build, validate and choose the mode |
| Validate project (free) | @zvid/n8n-nodes-zvid.zvid | Validates Zvid project payload without rendering cost | Build project JSON | Check validation | 3. Build, validate and choose the mode |
| Check validation | n8n-nodes-base.code | Inspects validation results and reports errors | Validate project (free) | Dry run? | 3. Build, validate and choose the mode |
| Dry run? | n8n-nodes-base.if | Branches execution based on dry run setting | Check validation | Save draft to editor, Submit render | 3. Build, validate and choose the mode |
| Save draft to editor | @zvid/n8n-nodes-zvid.zvid | Saves project draft to Zvid editor | Dry run? | Dry run summary | 3. Build, validate and choose the mode |
| Dry run summary | n8n-nodes-base.code | Generates summary report for draft runs | Save draft to editor | None (Terminal) | 3. Build, validate and choose the mode |
| Submit render | @zvid/n8n-nodes-zvid.zvid | Submits paid video render job asynchronously | Dry run? | Attach job to event | 4. Render and wait for completion |
| Attach job to event | n8n-nodes-base.code | Attaches jobId to event context | Submit render | Wait | 4. Render and wait for completion |
| Wait | n8n-nodes-base.wait | Pauses execution between status poll requests | Attach job to event, Still rendering? | Get render status | 4. Render and wait for completion |
| Get render status | @zvid/n8n-nodes-zvid.zvid | Queries job completion status from Zvid API | Wait | Merge job status | 4. Render and wait for completion |
| Merge job status | n8n-nodes-base.code | Merges job status with carried event context | Get render status | Render finished? | 4. Render and wait for completion |
| Render finished? | n8n-nodes-base.if | Checks if render has successfully completed | Merge job status | Prepare write-back, Still rendering? | 4. Render and wait for completion |
| Still rendering? | n8n-nodes-base.code | Enforces timeout limits and checks for render failures | Render finished? | Wait | 4. Render and wait for completion |
| Prepare write-back | n8n-nodes-base.code | Formats completion results for write-back | Render finished? | Sheet mode? | 5. Record the completed video |
| Sheet mode? | n8n-nodes-base.if | Routes execution to sheet update or summary | Prepare write-back | Update event row, Run summary | 5. Record the completed video |
| Update event row | n8n-nodes-base.googleSheets | Updates Google Sheet row with completed video URL | Sheet mode? | Run summary | 5. Record the completed video |
| Run summary | n8n-nodes-base.code | Compiles execution metrics and final video URLs | Sheet mode?, Update event row | Video ready to watch? | 5. Record the completed video |
| Video ready to watch? | n8n-nodes-base.if | Checks if video is valid and ready for download | Run summary | ▶ Watch video, None | 6. Watch the finished video |
| ▶ Watch video | n8n-nodes-base.httpRequest | Downloads finished MP4 video file into binary data | Video ready to watch? | None (Terminal) | 6. Watch the finished video |
| Workflow overview and setup | n8n-nodes-base.stickyNote | Workflow documentation and setup guide | None | None | Workflow overview and setup |
| 1. Set up and check music | n8n-nodes-base.stickyNote | Block documentation for configuration and music check | None | None | 1. Set up and check music |
| 2. Choose the event due today | n8n-nodes-base.stickyNote | Block documentation for event extraction and urgency logic | None | None | 2. Choose the event due today |
| 3. Build, validate and choose the mode | n8n-nodes-base.stickyNote | Block documentation for project building and validation | None | None | 3. Build, validate and choose the mode |
| 4. Render and wait for completion | n8n-nodes-base.stickyNote | Block documentation for rendering and status polling | None | None | 4. Render and wait for completion |
| 5. Record the completed video | n8n-nodes-base.stickyNote | Block documentation for write-back and state recording | None | None | 5. Record the completed video |
| 6. Watch the finished video | n8n-nodes-base.stickyNote | Block documentation for media download and review | None | None | 6. Watch the finished video |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Prerequisite Setup:**
   - Install the community node `@zvid/n8n-nodes-zvid` via **Settings → Community nodes** in your self-hosted n8n instance.
   - Prepare a Google Sheet with a worksheet containing the following exact column headers: `EventName`, `DateISO`, `Venue`, `City`, `TicketUrl`, `ImageUrl`, `Status`, `PromoUrl`, `Video7d`, `Video1d`.

2. **Create Triggers and Configuration:**
   - **Node 1:** Create a `Manual Trigger` (`n8n-nodes-base.manualTrigger`). Name it `Test manually`.
   - **Node 2:** Create a `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`). Name it `Every day at 8am`, configured to trigger daily at hour 8.
   - **Node 3:** Create a `Set` node (`n8n-nodes-base.set`). Name it `Config`, set mode to `Raw`, and populate the JSON object with your API URLs, branding details (`brandName`, colors, fonts), and operational settings (`source: "sheet"`, `dryRun: false`, etc.). Connect both triggers to this node.

3. **Configure Music Verification:**
   - **Node 4:** Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`). Name it `Check music`. Set method to `HEAD`, URL to `={{ $('Config').first().json.musicUrl }}`, timeout to 15000ms, and enable `Always Output Data` with never-error response settings. Connect `Config` to this node.
   - **Node 5:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Music guard`. Paste the JavaScript code to validate the music file size and headers. Connect `Check music` to this node.

4. **Set Up Data Extraction & Urgency Routing:**
   - **Node 6:** Create an `If` node (`n8n-nodes-base.if`). Name it `Source?`. Set condition to evaluate `{{ $('Config').first().json.source === 'eventbrite' }}`. Connect `Music guard` to this node.
   - **Node 7:** Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`). Name it `Fetch Eventbrite events`. Configure URL for Eventbrite API v3 organization events and select an `HTTP Bearer Auth` credential. Connect the True output of `Source?` here.
   - **Node 8:** Create a `Google Sheets` node (`n8n-nodes-base.googleSheets`). Name it `Read events sheet`. Set resource to `sheet`, operation to `read`, and select your document and worksheet. Connect the False output of `Source?` here.
   - **Node 9:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Which renders today?`. Paste the urgency calculation script to parse dates and select due videos. Connect both `Fetch Eventbrite events` and `Read events sheet` outputs to this node.
   - **Node 10:** Create an `If` node (`n8n-nodes-base.if`). Name it `Anything due?`. Set condition to evaluate `{{ $json.due }}`. Connect `Which renders today?` to this node.
   - **Node 11:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Nothing due today`. Connect the False output of `Anything due?` here as a terminal branch.

5. **Build, Validate, and Route Renders:**
   - **Node 12:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Build project JSON`. Paste the project builder script. Connect the True output of `Anything due?` to this node.
   - **Node 13:** Create a Zvid node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Validate project (free)`. Set resource to `render`, operation to `validate`, and project JSON expression to `={{ JSON.stringify(({ payload: $json.payload }).payload) }}` using a Zvid API credential (`https://api.zvid.io`). Connect `Build project JSON` to this node.
   - **Node 14:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Check validation`. Connect `Validate project (free)` to this node.
   - **Node 15:** Create an `If` node (`n8n-nodes-base.if`). Name it `Dry run?`. Set condition to evaluate `{{ $('Config').first().json.dryRun }}`. Connect `Check validation` to this node.
   - **Node 16:** Create a Zvid node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Save draft to editor`. Set resource to `project`, operation to `create`. Connect the True output of `Dry run?` here.
   - **Node 17:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Dry run summary`. Connect `Save draft to editor` here as a terminal branch.

6. **Render, Poll, and Complete:**
   - **Node 18:** Create a Zvid node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Submit render`. Set resource to `render`, operation to `create`, render type to `video`, and `waitForCompletion` to false. Connect the False output of `Dry run?` here.
   - **Node 19:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Attach job to event`. Connect `Submit render` to this node.
   - **Node 20:** Create a `Wait` node (`n8n-nodes-base.wait`). Name it `Wait`, configured with seconds based on `={{ $('Config').first().json.pollSeconds }}`. Connect `Attach job to event` to this node.
   - **Node 21:** Create a Zvid node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Get render status`. Set resource to `render`, operation to `get`, and job ID expression to `={{ $json.jobId }}` (`waitForCompletion: false`). Connect `Wait` to this node.
   - **Node 22:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Merge job status`. Connect `Get render status` to this node.
   - **Node 23:** Create an `If` node (`n8n-nodes-base.if`). Name it `Render finished?`. Set condition to evaluate `{{ $json.state === 'completed' }}`. Connect `Merge job status` to this node.
   - **Node 24:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Still rendering?`. Connect the False output of `Render finished?` here, and loop its output back into the `Wait` node.

7. **Write-Back and Media Review:**
   - **Node 25:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Prepare write-back`. Connect the True output of `Render finished?` to this node.
   - **Node 26:** Create an `If` node (`n8n-nodes-base.if`). Name it `Sheet mode?`. Set condition to evaluate sheet mode and row presence. Connect `Prepare write-back` to this node.
   - **Node 27:** Create a `Google Sheets` node (`n8n-nodes-base.googleSheets`). Name it `Update event row`. Set resource to `sheet`, operation to `update`, matching by `row_number` with mapped status and URL columns using Google Sheets OAuth2 credentials. Connect the True output of `Sheet mode?` here.
   - **Node 28:** Create a `Code` node (`n8n-nodes-base.code`). Name it `Run summary`. Connect both the False output of `Sheet mode?` and `Update event row` to this node.
   - **Node 29:** Create an `If` node (`n8n-nodes-base.if`). Name it `Video ready to watch?`. Set condition to evaluate `{{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`. Connect `Run summary` to this node.
   - **Node 30:** Create an `HTTP Request` node (`n8n-nodes-base.httpRequest`). Name it `▶ Watch video`. Set URL to `={{ $json.videoUrl }}` with response format set to file (`outputPropertyName: "data"`). Connect the True output of `Video ready to watch?` here as the final terminal node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full setup and field reference guide | [GitHub Documentation](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/event-countdown-videos.md) |
| Zvid API Key generation | [Zvid Portal](https://app.zvid.io/api-keys) |
| Zvid Support and Contact | [Zvid Contact](https://zvid.io/contact) |