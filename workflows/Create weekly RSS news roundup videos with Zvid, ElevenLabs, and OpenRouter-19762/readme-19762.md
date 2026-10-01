Create weekly RSS news roundup videos with Zvid, ElevenLabs, and OpenRouter

https://n8nworkflows.xyz/workflows/create-weekly-rss-news-roundup-videos-with-zvid--elevenlabs--and-openrouter-19762


# Create weekly RSS news roundup videos with Zvid, ElevenLabs, and OpenRouter

### 1. Workflow Overview

This workflow automates the creation of vertical weekly news roundup videos (such as Instagram Reels or TikToks) derived from RSS feeds. It fetches recent news items, deduplicates and filters them based on a lookback window, leverages an AI language model to script a countdown, sources matching stock media and background music, generates synchronized AI narration with word-level timings, as well as assembles, validates, and renders a complete branded video.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Manually or automatically initiates the workflow and sets up all global branding, AI parameters, and rendering options.
- **1.2 RSS Feed Ingestion & Pooling:** Plans feed requests, reads multiple RSS feeds concurrently, and pools/deduplicates recent stories.
- **1.3 AI Script Curation:** Sends pooled headlines to an LLM via OpenRouter to select the top five stories and formulate a structured countdown script.
- **1.4 Stock Media Sourcing:** Expands the curated stories into individual tasks and queries Zvid's media library for matching video clips.
- **1.5 Background Music Selection:** Searches, shortlists, and validates audio tracks based on file size constraints and duration limits.
- **1.6 Voiceover Generation & Processing:** Generates time-aligned TTS audio via ElevenLabs, converts character alignments to word timings, and uploads the audio binary to Zvid.
- **1.7 Project Assembly & Validation:** Constructs the comprehensive Zvid project JSON (handling scenes, animations, titles, and karaoke captions) and validates it against the Zvid API.
- **1.8 Execution Branching & Rendering:** Evaluates the `dryRun` configuration flag to either generate an editor draft link or submit a paid render job, polling until completion.
- **1.9 Output Review:** Optionally downloads and views the finalized MP4 binary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block serves as the workflow's entry points and establishes the foundational configuration parameters used across downstream nodes.
- **Nodes Involved:** `Test manually`, `Every Friday at 4pm`, `Config`
- **Node Details:**
  - **`Test manually`** (`n8n-nodes-base.manualTrigger`)
    - *Technical Role:* Manual entry point for testing executions.
    - *Configuration:* Default parameters.
    - *Input/Output:* No inputs; outputs a single empty trigger item to `Config`.
    - *Edge Cases:* None.
  - **`Every Friday at 4pm`** (`n8n-nodes-base.scheduleTrigger`)
    - *Technical Role:* Automated cron-like trigger firing every Friday at 16:00.
    - *Configuration:* Rule set to trigger weekly on day 5 at hour 16.
    - *Input/Output:* No inputs; outputs a scheduled trigger item to `Config`.
    - *Edge Cases:* Relies on the n8n instance's host timezone configuration.
  - **`Config`** (`n8n-nodes-base.set`)
    - *Technical Role:* Sets global variables (API endpoints, niche, RSS feed URLs, lookback windows, brand color codes, typography, LLM models, voice settings, and render options).
    - *Configuration:* Raw JSON output mode containing an extensive configuration object. Key properties include `apiUrl`, `feedUrls`, `lookbackDays`, `llmModel`, `voiceId`, `dryRun`, and timeout settings.
    - *Input/Output:* Receives input from either trigger; outputs the configuration object.
    - *Edge Cases:* Malformed JSON structures or empty `feedUrls` will propagate downstream execution errors.

#### 2.2 RSS Feed Ingestion & Pooling
This block processes configured URLs, queries the RSS feeds, aggregates items, and creates a clean text pool for AI curation.
- **Nodes Involved:** `Plan feeds`, `Read feeds`, `Pool headlines`
- **Node Details:**
  - **`Plan feeds`** (`n8n-nodes-base.code`)
    - *Technical Role:* Transforms the `feedUrls` array into individual items for parallel feed reading.
    - *Configuration:* JavaScript execution validating HTTP/HTTPS URLs from `Config`. Throws an error if no usable URLs exist.
    - *Input/Output:* Input from `Config`; outputs multiple items (one per valid feed URL).
    - *Edge Cases:* Throws if `feedUrls` is empty or invalid.
  - **`Read feeds`** (`n8n-nodes-base.rssFeedRead`)
    - *Technical Role:* Fetches items from each RSS feed URL.
    - *Configuration:* URL set via expression `={{ $json.feedUrl }}`; retry on fail enabled (max 3 tries, 5000ms interval); `onError` set to continue regular output.
    - *Input/Output:* Input: feed URL items; Output: raw RSS feed items.
    - *Edge Cases:* Dead feeds or network timeouts are handled gracefully via `continueRegularOutput`, contributing no items without failing the entire workflow.
  - **`Pool headlines`** (`n8n-nodes-base.code`)
    - *Technical Role:* Merges feed items, deduplicates headlines, filters by lookback window, and constructs the LLM prompt.
    - *Configuration:* JavaScript execution filtering items based on `lookbackDays` from the `Config` node, producing a maximum of 40 pooled items, and generating a structured prompt text.
    - *Input/Output:* Input: all fetched RSS items; Output: single item containing `promptText`, `pool`, `nowIso`, and `pooled` count.
    - *Edge Cases:* Throws an error if no usable stories are found within the lookback window across all feeds.

#### 2.3 AI Script Curation
This block interacts with an LLM via OpenRouter to select the top stories and format them into a structured script.
- **Nodes Involved:** `Curate top five`, `Parse rundown`
- **Node Details:**
  - **`Curate top five`** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Sends the curation prompt to OpenRouter and requests JSON-formatted output.
    - *Configuration:* POST request to `https://openrouter.ai/api/v1/chat/completions`, using predefined `openRouterApi` credentials. Body sends the configured `llmModel` and forces JSON object response formatting. Retry on fail enabled (3 attempts).
    - *Input/Output:* Input: curation prompt from `Pool headlines`; Output: LLM chat completion response.
    - *Edge Cases:* API rate limits, invalid credentials, or model timeouts. Retries handle temporary network interruptions.
  - **`Parse rundown`** (`n8n-nodes-base.code`)
    - *Technical Role:* Parses the LLM's JSON response, normalizes typographic characters to ASCII, validates stories, maps sources back to original feed items, and compiles the final narration script.
    - *Configuration:* JavaScript execution extracting `choices[0].message.content`, parsing JSON, performing string normalization (`toAscii`), and validating minimum word counts.
    - *Input/Output:* Input: LLM HTTP response; Output: single item containing structured `intro`, `outro`, `stories`, `musicTag`, `narration`, and metadata.
    - *Edge Cases:* Model outputting invalid JSON or a script shorter than 40 words triggers an explicit error.

#### 2.4 Stock Media Sourcing
This block splits the curated stories and queries Zvid's stock video library to find relevant background footage for each scene.
- **Nodes Involved:** `Expand stories`, `Search stock clips`, `Pick story clips`
- **Node Details:**
  - **`Expand stories`** (`n8n-nodes-base.code`)
    - *Technical Role:* Converts the array of stories into individual execution items for parallel stock media searching.
    - *Configuration:* JavaScript mapping each story to an individual item containing its index, headline, and `visualQuery`.
    - *Input/Output:* Input: parsed rundown; Output: one item per story.
    - *Edge Cases:* Empty story arrays yield no downstream searches.
  - **`Search stock clips`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Searches Zvid's stock media repository for video clips matching the query.
    - *Configuration:* Resource: `stockMedia`, Operation: `search`, stock type: `video`, query set via expression `={{ $json.visualQuery }}`, page size: 20. `onError` set to continue regular output; retry on fail enabled.
    - *Input/Output:* Input: story visual queries; Output: stock search results containing item lists.
    - *Edge Cases:* Missing stock items or API errors are caught and allowed to continue, defaulting to fallback scenes downstream.
  - **`Pick story clips`** (`n8n-nodes-base.code`)
    - *Technical Role:* Evaluates and selects the best b-roll clip per story based on orientation (portrait preferred) and duration.
    - *Configuration:* JavaScript processing stock search responses against target scene durations.
    - *Input/Output:* Input: stock search results; Output: single item containing `sceneMedia` array and `missing` clip count.
    - *Edge Cases:* Unmatched or missing media results in `null` entries, which trigger branded gradient fallback panels in the video builder rather than failing the execution.

#### 2.5 Background Music Selection
This block queries, shortlists, and validates background audio tracks from Zvid's music library.
- **Nodes Involved:** `Music queries`, `Find background music`, `Shortlist music`, `Check music asset`, `Pick music`
- **Node Details:**
  - **`Music queries`** (`n8n-nodes-base.code`)
    - *Technical Role:* Generates search queries combining the LLM-selected music tag with broad fallbacks (`cinematic`, `ambient`).
    - *Configuration:* JavaScript extracting `musicTag` from parsed rundown and outputting unique query strings.
    - *Input/Output:* Input: parsed rundown; Output: multiple query items.
    - *Edge Cases:* None.
  - **`Find background music`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Searches Zvid's stock audio library for tracks matching each query.
    - *Configuration:* Resource: `stockMedia`, Operation: `search`, stock type: `audio`, query set via expression `={{ $json.query }}`, page size: 10. `onError` set to continue.
    - *Input/Output:* Input: music query items; Output: audio search results.
    - *Edge Cases:* Empty results from specific queries are filtered out safely downstream.
  - **`Shortlist music`** (`n8n-nodes-base.code`)
    - *Technical Role:* Deduplicates and shortlists audio candidates based on duration limits.
    - *Configuration:* JavaScript filtering tracks by duration constraints (between 45 seconds and `maxMusicSeconds`) and sorting by duration.
    - *Input/Output:* Input: audio search results; Output: shortisted candidate items.
    - *Edge Cases:* Emits a fallback health check probe URL if no tracks match the criteria.
  - **`Check music asset`** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Probes shortlisted audio candidate URLs via HTTP HEAD requests to verify availability and file size.
    - *Configuration:* HTTP Method: HEAD, URL set via expression `={{ $json.probeUrl }}`, timeout: 15000ms, options configured to never error on non-200 responses and return full response headers.
    - *Input/Output:* Input: shortlisted music candidates; Output: HTTP response objects with headers.
    - *Edge Cases:* Broken URLs or network errors result in non-200 status codes, which are evaluated and rejected in the subsequent node.
  - **`Pick music`** (`n8n-nodes-base.code`)
    - *Technical Role:* Selects the first downloadable candidate complying with the `maxMusicBytes` file size cap.
    - *Configuration:* JavaScript iterating through probe results, checking `Content-Length` headers, and collapsing the output into a single music bed object.
    - *Input/Output:* Input: HEAD request probe results; Output: single item containing the chosen `music` track object, count, and rejected list.
    - *Edge Cases:* If all tracks fail validation, `music` is set to `null`, omitting background music without failing the run.

#### 2.6 Voiceover Generation & Processing
This block generates narration audio with precise character timings via ElevenLabs, processes word groupings, and uploads the audio binary to Zvid.
- **Nodes Involved:** `Generate voiceover`, `Voice + timings`, `Upload voiceover`
- **Node Details:**
  - **`Generate voiceover`** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Requests time-aligned text-to-speech generation from ElevenLabs.
    - *Configuration:* POST request to `https://api.elevenlabs.io/v1/text-to-speech/{voiceId}/with-timestamps`, authenticated via generic HTTP Header Auth (`xi-api-key`). Query parameter `output_format=mp3_44100_128`. Body contains narration text, model ID, and voice settings (`stability`, `similarity_boost`). Timeout: 120000ms. Retry on fail enabled.
    - *Input/Output:* Input: parsed rundown script; Output: JSON containing `audio_base64` and character alignment metadata.
    - *Edge Cases:* Character quota exhaustion, invalid voice IDs, or API timeouts.
  - **`Voice + timings`** (`n8n-nodes-base.code`)
    - *Technical Role:* Parses ElevenLabs character-level timestamps, groups non-whitespace runs into word timing objects, and packages the MP3 binary.
    - *Configuration:* JavaScript validation of audio payload, parsing characters/start/end times into word arrays, and constructing n8n binary data (`audio/mpeg`).
    - *Input/Output:* Input: ElevenLabs API response; Output: single item with `words`, `duration`, `wordCount`, and binary `data` payload.
    - *Edge Cases:* Missing audio base64 or absent alignment data throws an explicit error.
  - **`Upload voiceover`** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Uploads the generated MP3 binary to Zvid's storage API to obtain a public URL for the render engine.
    - *Configuration:* POST request to `{{ $json.apiUrl }}/api/uploads`, multipart form-data content type, authenticated via predefined `zvidApi` credential, sending input binary data field `data`. Retry on fail enabled. *(Note: Uses HTTP Request instead of the Zvid node due to multipart upload operation limitations in the published community node).*
    - *Input/Output:* Input: binary audio from previous node; Output: JSON containing the hosted file URL.
    - *Edge Cases:* Network errors or invalid Zvid credentials during file upload.

#### 2.7 Project Assembly & Validation
This block assembles the complete Zvid project JSON (defining scenes, typography, animations, audio tracks, and karaoke subtitles) and validates it against the Zvid API.
- **Nodes Involved:** `Build project JSON`, `Validate project (free)`, `Check validation`
- **Node Details:**
  - **`Build project JSON`** (`n8n-nodes-base.code`)
    - *Technical Role:* Compiles the Zvid project payload by structuring intro scenes, countdown story scenes (5 to 1 with media/fallback panels, headline cards, and source tags), outro CTA scenes, timed karaoke captions, and audio tracks.
    - *Configuration:* Comprehensive JavaScript builder function processing configuration settings, story data, stock media, music selections, voiceover URLs, and word timings.
    - *Input/Output:* Input: data aggregated from Config, Parse rundown, Pick story clips, Pick music, Upload voiceover, and Voice + timings; Output: single item containing the complete `payload` and metadata `meta`.
    - *Edge Cases:* Missing stories or unparseable timing structures throw rendering preparation errors.
  - **`Validate project (free)`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Submits the project payload to Zvid's free validation endpoint to check schema correctness and calculate credit requirements.
    - *Configuration:* Resource: `render`, Operation: `validate`, source set to JSON, passing `projectJson` stringified payload.
    - *Input/Output:* Input: project payload; Output: validation result containing `valid`, `creditsRequired`, `warnings`, and error details.
    - *Edge Cases:* Schema violations or unsupported parameter values return validation errors.
  - **`Check validation`** (`n8n-nodes-base.code`)
    - *Technical Role:* Inspects validation results and surfaces detailed field errors if validation fails.
    - *Configuration:* JavaScript evaluation of response status and error arrays. Throws formatted error details if `statusCode !== 200`.
    - *Input/Output:* Input: validation API response; Output: enriched build payload with `creditsRequired`, `warnings`, and `schemaVersion`.
    - *Edge Cases:* Rejects execution immediately if Zvid flags schema validation errors.

#### 2.8 Execution Branching & Rendering
This block evaluates whether to execute a dry run (saving an editor draft) or proceed with a live, paid render job with polling.
- **Nodes Involved:** `Dry run?`, `Save draft to editor`, `Dry run summary`, `Submit render`, `Wait`, `Get render status`, `Render finished?`, `Still rendering?`, `Run summary`
- **Node Details:**
  - **`Dry run?`** (`n8n-nodes-base.if`)
    - *Technical Role:* Branches execution based on the `dryRun` configuration flag.
    - *Configuration:* Condition evaluates `{{ $('Config').first().json.dryRun }}` as boolean `true`.
    - *Input/Output:* Input: validated project data; Output: True branch (Dry run) or False branch (Live render).
    - *Edge Cases:* None.
  - **`Save draft to editor`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Creates a project draft in the Zvid editor workspace when `dryRun` is enabled.
    - *Configuration:* Resource: `project`, Operation: `create`, setting project JSON and project name. `onError` set to continue regular output; retry on fail enabled.
    - *Input/Output:* Input: validated project data; Output: Zvid project ID and metadata.
    - *Edge Cases:* Draft creation failures are caught gracefully to prevent blocking the dry-run summary.
  - **`Dry run summary`** (`n8n-nodes-base.code`)
    - *Technical Role:* Generates a comprehensive dry-run report including editor preview links, credit quotes, and metadata without spending render credits.
    - *Configuration:* JavaScript compiling dry run metrics and assembling the editor link.
    - *Input/Output:* Input: draft project response and validation data; Output: dry-run summary report.
    - *Edge Cases:* None.
  - **`Submit render`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Submits the project for live video rendering, consuming Zvid credits.
    - *Configuration:* Resource: `render`, Operation: `create`, render type: `video`, `waitForCompletion` set to `false`. Retry on fail enabled.
    - *Input/Output:* Input: validated project data; Output: render job ID and initial status.
    - *Edge Cases:* API errors or insufficient credits.
  - **`Wait`** (`n8n-nodes-base.wait`)
    - *Technical Role:* Pauses execution between polling attempts to check render progress.
    - *Configuration:* Amount set via expression `={{ $('Config').first().json.pollSeconds }}` (default 10 seconds).
    - *Input/Output:* Input: job status or poll loop; Output: delayed execution signal.
    - *Edge Cases:* None.
  - **`Get render status`** (`@zvid/n8n-nodes-zvid.zvid`)
    - *Technical Role:* Queries the current status of the submitted render job.
    - *Configuration:* Resource: `render`, Operation: `get`, job ID set via expression `={{ $('Submit render').first().json.jobId }}`, `waitForCompletion` set to `false`. Retry on fail enabled.
    - *Input/Output:* Input: wait signal; Output: job status object (states: `queued`, `rendering`, `completed`, `failed`).
    - *Edge Cases:* Temporary API connection drops during polling.
  - **`Render finished?`** (`n8n-nodes-base.if`)
    - *Technical Role:* Checks whether the render job state has successfully completed.
    - *Configuration:* Condition evaluates `{{ $json.state === 'completed' }}`.
    - *Input/Output:* Input: job status object; Output: True (proceed to run summary) or False (check if still rendering).
    - *Edge Cases:* None.
  - **`Still rendering?`** (`n8n-nodes-base.code`)
    - *Technical Role:* Validates render progress, detects failures, and enforces timeout limits to prevent infinite polling loops.
    - *Configuration:* JavaScript checking for `state === 'failed'` (throwing job failure reasons) and calculating maximum allowed polls based on `timeoutMinutes` and `pollSeconds`.
    - *Input/Output:* Input: incomplete job status; Output: loops back to `Wait`.
    - *Edge Cases:* Throws an error if rendering fails or exceeds the configured timeout duration (default 20 minutes).
  - **`Run summary`** (`n8n-nodes-base.code`)
    - *Technical Role:* Compiles the final execution report for a successful live render.
    - *Configuration:* JavaScript extracting the final video URL from job results and compiling metadata.
    - *Input/Output:* Input: completed job status and validation data; Output: live run summary object containing `videoUrl`, `jobId`, and `creditsCharged`.
    - *Edge Cases:* Handles string or object result formats from the render job.

#### 2.9 Output Review
This block optionally fetches and outputs the finished MP4 video binary for review.
- **Nodes Involved:** `Video ready to watch?`, `▶ Watch video`
- **Node Details:**
  - **`Video ready to watch?`** (`n8n-nodes-base.if`)
    - *Technical Role:* Determines whether the run was live and generated a valid video URL before attempting to download the binary.
    - *Configuration:* Condition evaluates `{{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
    - *Input/Output:* Input: live run summary; Output: True branch (download video) or False branch (skip download).
    - *Edge Cases:* Dry runs or failed URLs skip binary fetching.
  - **`▶ Watch video`** (`n8n-nodes-base.httpRequest`)
    - *Technical Role:* Downloads the final rendered MP4 video file into n8n's binary data store for direct preview.
    - *Configuration:* GET request to `{{ $json.videoUrl }}`, response format set to file outputting to property name `data`. Retry on fail enabled; `onError` set to continue regular output.
    - *Input/Output:* Input: video URL; Output: binary MP4 file stored in item binary data.
    - *Edge Cases:* Network drops during large file downloads are handled by retry policies and `continueRegularOutput`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Test manually` | `manualTrigger` | Manual entry point for testing executions. | None | `Config` | 1. Choose settings and an entry point |
| `Every Friday at 4pm` | `scheduleTrigger` | Automated schedule trigger firing weekly on Fridays at 4pm. | None | `Config` | 1. Choose settings and an entry point |
| `Config` | `set` | Defines global workflow variables, branding, and render parameters. | `Test manually`, `Every Friday at 4pm` | `Plan feeds` | 1. Choose settings and an entry point |
| `Plan feeds` | `code` | Maps configured feed URLs into individual items. | `Config` | `Read feeds` | 2. Collect recent headlines |
| `Read feeds` | `rssFeedRead` | Fetches RSS feed items from target URLs. | `Plan feeds` | `Pool headlines` | 2. Collect recent headlines |
| `Pool headlines` | `code` | Deduplicates feed items, filters by lookback window, and builds LLM prompt. | `Read feeds` | `Curate top five` | 2. Collect recent headlines |
| `Curate top five` | `httpRequest` | Requests script curation and top 5 story selection from OpenRouter. | `Pool headlines` | `Parse rundown` | 3. Write and check the script |
| `Parse rundown` | `code` | Normalizes LLM JSON output, sanitizes text, and compiles narration script. | `Curate top five` | `Expand stories` | 3. Write and check the script |
| `Expand stories` | `code` | Splits curated stories into individual items for stock searching. | `Parse rundown` | `Search stock clips` | 4. Choose footage for each scene |
| `Search stock clips` | `zvid` | Searches Zvid's media library for video clips matching visual queries. | `Expand stories` | `Pick story clips` | 4. Choose footage for each scene |
| `Pick story clips` | `code` | Selects optimal b-roll clips per story or assigns fallback panels. | `Search stock clips` | `Music queries` | 4. Choose footage for each scene |
| `Music queries` | `code` | Generates tag-matched and fallback background music queries. | `Pick story clips` | `Find background music` | 5. Check the background music |
| `Find background music` | `zvid` | Searches Zvid's audio catalogue for music tracks. | `Music queries` | `Shortlist music` | 5. Check the background music |
| `Shortlist music` | `code` | Shortlists and sorts music tracks by duration constraints. | `Find background music` | `Check music asset` | 5. Check the background music |
| `Check music asset` | `httpRequest` | Probes music asset URLs via HTTP HEAD to verify size and availability. | `Shortlist music` | `Pick music` | 5. Check the background music |
| `Pick music` | `code` | Selects the first valid downloadable music track within size limits. | `Check music asset` | `Generate voiceover` | 5. Check the background music |
| `Generate voiceover` | `httpRequest` | Requests time-aligned text-to-speech audio from ElevenLabs. | `Pick music` | `Voice + timings` | 6. Create narration and timings |
| `Voice + timings` | `code` | Parses character alignments into word timings and packages MP3 binary. | `Generate voiceover` | `Upload voiceover` | 6. Create narration and timings |
| `Upload voiceover` | `httpRequest` | Uploads the MP3 binary to Zvid storage to obtain a public URL. | `Voice + timings` | `Build project JSON` | 6. Create narration and timings |
| `Build project JSON` | `code` | Assembles the complete Zvid project JSON payload with scenes and subtitles. | `Upload voiceover` | `Validate project (free)` | 7. Build and validate the design |
| `Validate project (free)` | `zvid` | Validates the project payload against Zvid's API and gets credit quotes. | `Build project JSON` | `Check validation` | 7. Build and validate the design |
| `Check validation` | `code` | Inspects validation results and surfaces errors if invalid. | `Validate project (free)` | `Dry run?` | 7. Build and validate the design |
| `Dry run?` | `if` | Branches execution based on the `dryRun` configuration setting. | `Check validation` | `Save draft to editor`, `Submit render` | 8. Choose an optional editor preview |
| `Save draft to editor` | `zvid` | Creates an editor draft project in Zvid when `dryRun` is enabled. | `Dry run?` | `Dry run summary` | 8. Choose an optional editor preview |
| `Dry run summary` | `code` | Compiles a dry-run summary report with editor preview links and cost quotes. | `Save draft to editor` | None | 8. Choose an optional editor preview |
| `Submit render` | `zvid` | Submits the validated project payload for live video rendering. | `Dry run?` | `Wait` | 9. Render and wait for completion |
| `Wait` | `wait` | Pauses execution between status polling intervals. | `Submit render`, `Still rendering?` | `Get render status` | 9. Render and wait for completion |
| `Get render status` | `zvid` | Queries the current render job status from Zvid. | `Wait` | `Render finished?` | 9. Render and wait for completion |
| `Render finished?` | `if` | Checks if the render job state has successfully completed. | `Get render status` | `Run summary`, `Still rendering?` | 9. Render and wait for completion |
| `Still rendering?` | `code` | Enforces timeout limits and detects failed render states. | `Render finished?` | `Wait` | 9. Render and wait for completion |
| `Run summary` | `code` | Compiles the final execution report for a successful live render. | `Render finished?` | `Video ready to watch?` | 9. Render and wait for completion |
| `Video ready to watch?` | `if` | Verifies that a valid live video URL was produced. | `Run summary` | `▶ Watch video`, None | 10. Review the finished media |
| `▶ Watch video` | `httpRequest` | Downloads the completed MP4 video file into n8n binary storage. | `Video ready to watch?` | None | 10. Review the finished media |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Install Prerequisites:**
   - Go to **Settings → Community nodes** in your n8n instance and install **`@zvid/n8n-nodes-zvid`**.
   - Ensure you have active API keys for **Zvid** (`https://app.zvid.io/api-keys`), **OpenRouter**, and **ElevenLabs** (with Text-to-Speech permission).

2. **Create Trigger and Configuration Nodes:**
   - **Node 1:** Create a **Manual Trigger** (`n8n-nodes-base.manualTrigger`). Name it `Test manually`.
   - **Node 2:** Create a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`). Name it `Every Friday at 4pm`, set recurrence to weekly, trigger at day 5 (Friday) at hour 16.
   - **Node 3:** Create a **Set** node (`n8n-nodes-base.set`). Name it `Config`. Set mode to raw JSON and paste the configuration object containing your target niche, feed URLs (`feedUrls`), branding colors, model selections (`llmModel`), voice configuration (`voiceId`), and `dryRun: false`.
   - *Connections:* Connect both triggers (`Test manually` and `Every Friday at 4pm`) to `Config`.

3. **Build the RSS Ingestion Pipeline:**
   - **Node 4:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Plan feeds`. Add JavaScript to parse `Config.feedUrls` into individual items. Connect `Config` to `Plan feeds`.
   - **Node 5:** Create an **RSS Feed Read** node (`n8n-nodes-base.rssFeedRead`). Name it `Read feeds`. Set URL expression to `={{ $json.feedUrl }}`, enable retries (3 attempts, 5000ms wait), and set error handling to continue regular output. Connect `Plan feeds` to `Read feeds`.
   - **Node 6:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Pool headlines`. Add JavaScript to merge, deduplicate, and filter RSS items by lookback window, generating the LLM prompt. Connect `Read feeds` to `Pool headlines`.

4. **Set Up AI Script Curation:**
   - **Node 7:** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Curate top five`. Set method to `POST`, URL to `https://openrouter.ai/api/v1/chat/completions`, select **OpenRouter API** credentials, and configure JSON body with system/user prompt formatting (`response_format: { type: "json_object" }`). Enable retries. Connect `Pool headlines` to `Curate top five`.
   - **Node 8:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Parse rundown`. Add JavaScript to parse LLM JSON, normalize text to ASCII (`toAscii`), validate stories, and compile narration text. Connect `Curate top five` to `Parse rundown`.

5. **Build Stock Media Sourcing:**
   - **Node 9:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Expand stories`. Add JavaScript to split stories into individual items. Connect `Parse rundown` to `Expand stories`.
   - **Node 10:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Search stock clips`. Select Zvid API credentials. Set resource to `stockMedia`, operation to `search`, stock type to `video`, and query to `={{ $json.visualQuery }}`. Enable error continuation and retries. Connect `Expand stories` to `Search stock clips`.
   - **Node 11:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Pick story clips`. Add JavaScript to select the best portrait or long-enough video clip per story or fallback to null. Connect `Search stock clips` to `Pick story clips`.

6. **Set Up Background Music Sourcing:**
   - **Node 12:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Music queries`. Add JavaScript to extract music tags and build fallback queries. Connect `Pick story clips` to `Music queries`.
   - **Node 13:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Find background music`. Set resource to `stockMedia`, operation to `search`, stock type to `audio`, and query to `={{ $json.query }}`. Connect `Music queries` to `Find background music`.
   - **Node 14:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Shortlist music`. Add JavaScript to filter tracks by duration (45s to `maxMusicSeconds`). Connect `Find background music` to `Shortlist music`.
   - **Node 15:** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Check music asset`. Set method to `HEAD`, URL to `={{ $json.probeUrl }}`, timeout to 15000ms, and configure options to never error on non-200 responses. Connect `Shortlist music` to `Check music asset`.
   - **Node 16:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Pick music`. Add JavaScript to validate `Content-Length` headers against `maxMusicBytes` and select the first valid track. Connect `Check music asset` to `Pick music`.

7. **Configure Voiceover & Timings:**
   - **Node 17:** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Generate voiceover`. Set method to `POST`, URL to `=https://api.elevenlabs.io/v1/text-to-speech/{{ $('Config').first().json.voiceId }}/with-timestamps`, configure Generic HTTP Header Auth (`xi-api-key`), set query parameter `output_format=mp3_44100_128`, and provide JSON body with text and voice settings. Connect `Pick music` to `Generate voiceover`.
   - **Node 18:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Voice + timings`. Add JavaScript to parse ElevenLabs character timestamps into word timing objects and package binary data (`audio/mpeg`, `voiceover.mp3`). Connect `Generate voiceover` to `Voice + timings`.
   - **Node 19:** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Upload voiceover`. Set method to `POST`, URL to `={{ $('Config').first().json.apiUrl }}/api/uploads`, content type to `multipart-form-data`, select Zvid API credentials, and map binary field `file` to input data field `data`. Connect `Voice + timings` to `Upload voiceover`.

8. **Construct Project Assembly & Validation:**
   - **Node 20:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Build project JSON`. Add JavaScript to assemble scenes, titles, countdowns, b-roll, audio tracks, and karaoke captions into the Zvid project payload. Connect `Upload voiceover` to `Build project JSON`.
   - **Node 21:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Validate project (free)`. Set resource to `render`, operation to `validate`, source to `json`, and `projectJson` to `={{ JSON.stringify(({ payload: $json.payload }).payload) }}`. Connect `Build project JSON` to `Validate project (free)`.
   - **Node 22:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Check validation`. Add JavaScript to inspect validation results and throw detailed errors on failure. Connect `Validate project (free)` to `Check validation`.

9. **Configure Rendering & Polling Logic:**
   - **Node 23:** Create an **If** node (`n8n-nodes-base.if`). Name it `Dry run?`. Set condition to evaluate `{{ $('Config').first().json.dryRun }}` equals `true`. Connect `Check validation` to `Dry run?`.
   - **True Branch (Dry Run):**
     - **Node 24:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Save draft to editor`. Set resource to `project`, operation to `create`, passing project JSON and name. Enable error continuation. Connect to `Dry run?` (true branch).
     - **Node 25:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Dry run summary`. Compiles the dry run report and editor link. Connect `Save draft to editor` to `Dry run summary`.
   - **False Branch (Live Render):**
     - **Node 26:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Submit render`. Set resource to `render`, operation to `create`, render type `video`, `waitForCompletion: false`. Connect `Dry run?` (false branch) to `Submit render`.
     - **Node 27:** Create a **Wait** node (`n8n-nodes-base.wait`). Name it `Wait`. Set amount to `={{ $('Config').first().json.pollSeconds }}` seconds. Connect `Submit render` (and `Still rendering?`) to `Wait`.
     - **Node 28:** Create a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`). Name it `Get render status`. Set resource to `render`, operation to `get`, job ID `={{ $('Submit render').first().json.jobId }}`, `waitForCompletion: false`. Connect `Wait` to `Get render status`.
     - **Node 29:** Create an **If** node (`n8n-nodes-base.if`). Name it `Render finished?`. Condition: `{{ $json.state === 'completed' }}`. Connect `Get render status` to `Render finished?`.
     - **Node 30:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Still rendering?`. Validates render failure states and timeout limits. Connect `Render finished?` (false branch) to `Still rendering?`, and loop back to `Wait`.
     - **Node 31:** Create a **Code** node (`n8n-nodes-base.code`). Name it `Run summary`. Compiles final live run report and video URL. Connect `Render finished?` (true branch) to `Run summary`.
     - **Node 32:** Create an **If** node (`n8n-nodes-base.if`). Name it `Video ready to watch?`. Condition checks `{!$json.dryRun && /^https?:\/\//i.test(($json.videoUrl))}`. Connect `Run summary` to `Video ready to watch?`.
     - **Node 33:** Create an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `▶ Watch video`. GET request to `{{ $json.videoUrl }}`, outputting response as file binary `data`. Connect `Video ready to watch?` (true branch) to `▶ Watch video`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zvid Community Node Installation Guide | [Zvid Integration on n8n](https://n8n.io/integrations/zvid/) |
| Zvid API Keys Generation | [Zvid API Keys Management](https://app.zvid.io/api-keys) |
| Multipart Voice Upload Implementation Note | Multipart voice upload uses an HTTP Request node because the published Zvid community node lacks that specific file upload operation. |