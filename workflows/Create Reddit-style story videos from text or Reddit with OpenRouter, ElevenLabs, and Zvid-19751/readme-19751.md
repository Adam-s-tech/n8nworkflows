Create Reddit-style story videos from text or Reddit with OpenRouter, ElevenLabs, and Zvid

https://n8nworkflows.xyz/workflows/create-reddit-style-story-videos-from-text-or-reddit-with-openrouter--elevenlabs--and-zvid-19751


# Create Reddit-style story videos from text or Reddit with OpenRouter, ElevenLabs, and Zvid

### 1. Workflow Overview

This workflow automates the creation of vertical, captioned story videos (optimized for TikTok, Reels, or Shorts) either from manual text inputs or from top posts on a configured Reddit subreddit. It integrates OpenRouter for AI script rewriting and scene breakdown, ElevenLabs for voiceover generation with precise word-level timestamps, and Zvid for stock media searching, background music matching, project validation, and final video rendering. 

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Sets up global workflow parameters, handles triggers (schedule or manual), and branches based on whether the story source mode is set to manual or Reddit.
- **1.2 Story Sourcing & AI Scripting:** Validates or fetches the source story, structures it into an AI prompt, sends it to OpenRouter to generate a script with scene beats, and parses the resulting JSON into individual shot segments.
- **1.3 Stock Media & Audio Shortlisting:** Searches Zvid's stock media library for portraits/videos matching each scene's visual query and queries/shortlists royalty-free background music tracks.
- **1.4 Voiceover Generation & Asset Upload:** Generates a synchronized voiceover via ElevenLabs, extracts per-character alignment data to compute word timings, and uploads the resulting MP3 to Zvid.
- **1.5 Project Assembly & Validation:** Builds a comprehensive Zvid project payload incorporating a custom cover card, timed b-roll/brand plates, karaoke-style subtitles, background audio, and an outro CTA card, followed by a free validation check.
- **1.6 Rendering, Polling, & Delivery:** Evaluates the dry-run setting (saving an editor draft if true, or submitting a paid render job if false), polls the Zvid render engine until completion, and downloads the final MP4 video file for review.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
- **Overview:** Initializes workflow configurations, manages scheduled or manual execution entry points, and splits execution paths depending on the designated story source mode.
- **Nodes Involved:** `Every day at 9am`, `Test manually`, `Config`, `Manual story?`
- **Node Details:**
  - **Every day at 9am**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration Choices:* Configured to trigger daily at hour 9 (9:00 AM).
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input: None. Output: Connects to `Config`.
    - *Edge Cases/Failures:* Ensure n8n instance timezones are correctly configured.
  - **Test manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger)
    - *Configuration Choices:* Standard manual execution button.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input: None. Output: Connects to `Config`.
    - *Edge Cases/Failures:* None.
  - **Config**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Raw JSON mode outputting global configuration parameters including API endpoints, subreddit sources, character limits, target video duration, voice settings, branding colors, typography, transition preferences, and execution modes (`dryRun`, `sourceMode`).
    - *Key Expressions/Variables:* Holds parameters like `apiUrl`, `sourceMode`, `minChars`, `maxChars`, `voiceId`, `dryRun`, etc.
    - *Input/Output:* Input: `Every day at 9am` or `Test manually`. Output: Connects to `Manual story?`.
    - *Edge Cases/Failures:* Invalid JSON or missing required fields (`manualStoryTitle`, `manualStoryText`) when `sourceMode` is set to `manual`.
  - **Manual story?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates if `sourceMode` equals `manual`.
    - *Key Expressions/Variables:* `={{ $('Config').first().json.sourceMode === 'manual' }}`
    - *Input/Output:* Input: `Config`. Output: True branch connects to `Pick story`; False branch connects to `Fetch top stories`.
    - *Edge Cases/Failures:* Case-sensitivity issues if `sourceMode` is misspelled.

---

#### Block 1.2: Story Sourcing & AI Scripting
- **Overview:** Fetches top Reddit posts via OAuth or validates manual text inputs, formats a prompt for OpenRouter, submits it to an LLM, and parses the returned JSON into actionable scene beats and shot timings.
- **Nodes Involved:** `Fetch top stories`, `Pick story`, `Write script`, `Parse script`, `Expand scenes`
- **Node Details:**
  - **Fetch top stories**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Sends a GET request to Reddit's OAuth API (`/r/{subreddit}/top`) with a limit of 25 posts, a 30s timeout, and structured headers.
    - *Key Expressions/Variables:* Uses `encodeURIComponent($('Config').first().json.subreddit)` and `redditUserAgent`.
    - *Input/Output:* Input: `Manual story?` (False branch). Output: Connects to `Pick story`.
    - *Credentials:* `redditOAuth2Api`
    - *Edge Cases/Failures:* HTTP 429 (Rate Limit), HTTP 401/403 (Invalid OAuth token or user agent), or empty listings.
  - **Pick story**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Validates character lengths, deduplicates against previously used posts stored in workflow global static data, filters out inappropriate or removed posts, and constructs an explicit LLM prompt requesting a JSON-formatted narrative script.
    - *Key Expressions/Variables:* Accesses `Config` limits and runtime evaluation variables.
    - *Input/Output:* Input: `Manual story?` (True branch) or `Fetch top stories`. Output: Connects to `Write script`.
    - *Edge Cases/Failures:* Throws explicit descriptive errors if manual text violates character bounds or if Reddit returns no eligible unused stories.
  - **Write script**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* POST request to OpenRouter API (`/api/v1/chat/completions`) requesting a strict JSON object response format.
    - *Key Expressions/Variables:* Uses `llmModel` from `Config` and `promptText` from previous node.
    - *Input/Output:* Input: `Pick story`. Output: Connects to `Parse script`.
    - *Credentials:* `openRouterApi`
    - *Edge Cases/Failures:* Model rate limits, timeouts, or failure to return valid JSON structures (mitigated by `retryOnFail: true`).
  - **Parse script**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Normalizes LLM output to ASCII, extracts hook and scene sentences, applies fallback logic if scenes are missing, segments long beats into separate shot objects based on `maxShotSeconds`, and calculates target word counts.
    - *Key Expressions/Variables:* Parses `choices[0].message.content`.
    - *Input/Output:* Input: `Write script`. Output: Connects to `Expand scenes`.
    - *Edge Cases/Failures:* Fails if the model returns zero content or insufficient scene beats.
  - **Expand scenes**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Flattens script scenes into individual shot items, allocating a video budget (up to 5 video clips per execution on free tiers) while assigning image stills to overflow or continuation shots.
    - *Key Expressions/Variables:* Tracks `videoBudget` count.
    - *Input/Output:* Input: `Parse script`. Output: Connects to `Search stock clips`.
    - *Edge Cases/Failures:* None.

---

#### Block 1.3: Stock Media & Audio Shortlisting
- **Overview:** Searches Zvid's stock library for portrait-oriented videos or images matching each shot's visual query, and queries/shortlists royalty-free background music tracks.
- **Nodes Involved:** `Search stock clips`, `Pick scene clip`, `Music queries`, `Find background music`, `Shortlist music`, `Check music asset`, `Pick music`
- **Node Details:**
  - **Search stock clips**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `stockMedia`, Operation: `search`. Queries Zvid's library using parameters `stockType` (video/image) and `stockQuery`.
    - *Key Expressions/Variables:* `={{ $json.mediaType }}`, `={{ $json.visualQuery }}`
    - *Input/Output:* Input: `Expand scenes`. Output: Connects to `Pick scene clip`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Handled via `onError: continueRegularOutput` and `alwaysOutputData: true` to prevent workflow crashes on empty searches.
  - **Pick scene clip**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Scores and ranks retrieved stock assets, prioritizing portrait orientation, adequate duration, and penalizing assets depicting people to protect faceless monetization safety guidelines.
    - *Key Expressions/Variables:* Uses regex `PEOPLE` to detect human-related tags/descriptions.
    - *Input/Output:* Input: `Search stock clips`. Output: Connects to `Music queries`.
    - *Edge Cases/Failures:* Throws an error if zero matching stock clips are found across all shots.
  - **Music queries**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Defines standard ambient stock search tags (`chill`, `ambient`, `calm`).
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Input: `Pick scene clip`. Output: Connects to `Find background music`.
    - *Edge Cases/Failures:* None.
  - **Find background music**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `stockMedia`, Operation: `search`, with `stockType` set to `audio`.
    - *Key Expressions/Variables:* `={{ $json.query }}`
    - *Input/Output:* Input: `Music queries`. Output: Connects to `Shortlist music`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Handled via `onError: continueRegularOutput`.
  - **Shortlist music**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Deduplicates music tracks and filters by duration constraints (between 45 seconds and `maxMusicSeconds`).
    - *Key Expressions/Variables:* Accesses `maxMusicSeconds` from `Config`.
    - *Input/Output:* Input: `Find background music`. Output: Connects to `Check music asset`.
    - *Edge Cases/Failures:* Passes fallback items if the pool is empty.
  - **Check music asset**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Performs a HEAD request on candidate audio URLs to verify downloadability and file size.
    - *Key Expressions/Variables:* `={{ $json.probeUrl }}`
    - *Input/Output:* Input: `Shortlist music`. Output: Connects to `Pick music`.
    - *Edge Cases/Failures:* Handled via `onError: continueRegularOutput` and `alwaysOutputData: true`.
  - **Pick music**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Selects the first track satisfying file size caps (`maxMusicBytes`) and collapses items into a single execution stream.
    - *Key Expressions/Variables:* Accesses `maxMusicBytes` from `Config`.
    - *Input/Output:* Input: `Check music asset`. Output: Connects to `Generate voiceover`.
    - *Edge Cases/Failures:* If all tracks fail validation, execution proceeds without background music rather than failing.

---

#### Block 1.4: Voiceover Generation & Asset Upload
- **Overview:** Sends the final narration string to ElevenLabs to generate speech audio with precise word-level timestamps, parses the alignment data, and uploads the MP3 binary to Zvid via multipart form data.
- **Nodes Involved:** `Generate voiceover`, `Voice + timings`, `Upload voiceover`
- *Note:* The `Upload voiceover` step uses an HTTP Request node instead of the Zvid community node due to multipart upload limitations in the published node version.
- **Node Details:**
  - **Generate voiceover**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* POST request to ElevenLabs API (`/v1/text-to-speech/{voiceId}/with-timestamps`), requesting 44.1kHz 128kbps MP3 format.
    - *Key Expressions/Variables:* Uses `voiceId`, `elevenModel`, `voiceStability`, `voiceSimilarity` from `Config`, and `narration` from `Parse script`.
    - *Input/Output:* Input: `Pick music`. Output: Connects to `Voice + timings`.
    - *Credentials:* `httpHeaderAuth` (ElevenLabs API Key)
    - *Edge Cases/Failures:* API quota exhaustion, invalid voice ID, or network timeouts (mitigated by `retryOnFail: true`).
  - **Voice + timings**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Decodes base64 audio data into binary format (`audio/mpeg`), extracts character alignment arrays, and groups non-whitespace characters into structured word-timing objects with start and end timestamps.
    - *Key Expressions/Variables:* Parses `res.audio_base64` and `res.alignment`.
    - *Input/Output:* Input: `Generate voiceover`. Output: Connects to `Upload voiceover`.
    - *Edge Cases/Failures:* Throws an error if alignment data is missing or if the ElevenLabs endpoint omitted `/with-timestamps`.
  - **Upload voiceover**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* POST request to `API_URL/api/uploads` using `multipart/form-data` with binary input data field name `data`.
    - *Key Expressions/Variables:* `={{ $('Config').first().json.apiUrl }}/api/uploads`
    - *Input/Output:* Input: `Voice + timings`. Output: Connects to `Build project JSON`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Upload timeouts or storage limits (mitigated by `retryOnFail: true`).

---

#### Block 1.5: Project Assembly & Validation
- **Overview:** Assembles the complete Zvid render project JSON (incorporating cover cards, script beats, media clips, karaoke subtitle configurations, audio beds, and outro CTA cards), and validates it using Zvid's free validation endpoint.
- **Nodes Involved:** `Build project JSON`, `Validate project (free)`, `Check validation`
- **Node Details:**
  - **Build project JSON**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Generates a fully structured Zvid project payload including SVG cover art, calculated beat spans aligned to speech word-maps, dynamic scene formatting, video/image layers, karaoke caption formatting, background/outro audio arrays, and channel watermarks.
    - *Key Expressions/Variables:* Aggregates variables from `Config`, `Pick story`, `Parse script`, `Pick scene clip`, `Pick music`, `Upload voiceover`, and `Voice + timings`.
    - *Input/Output:* Input: `Upload voiceover`. Output: Connects to `Validate project (free)`.
    - *Edge Cases/Failures:* Payload serialization or text sanitization issues.
  - **Validate project (free)**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `render`, Operation: `validate`. Submits the project JSON to verify schema compliance and credit requirements.
    - *Key Expressions/Variables:* `={{ JSON.stringify(({ payload: $json.payload }).payload) }}`
    - *Input/Output:* Input: `Build project JSON`. Output: Connects to `Check validation`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Network errors or schema validation errors.
  - **Check validation**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Evaluates validation status code. If validation fails, extracts detailed field-level error messages and throws an explicit exception.
    - *Key Expressions/Variables:* Inspects response status and validation errors.
    - *Input/Output:* Input: `Validate project (free)`. Output: Connects to `Dry run?`.
    - *Edge Cases/Failures:* Throws detailed Zvid error messages if payload validation fails.

---

#### Block 1.6: Rendering, Polling, & Delivery
- **Overview:** Evaluates whether `dryRun` is enabled (saving an editor draft if true) or proceeds with submitting a paid render job, polls the render task until completion, and downloads the resulting MP4 video file.
- **Nodes Involved:** `Dry run?`, `Save draft to editor`, `Dry run summary`, `Submit render`, `Wait`, `Get render status`, `Render finished?`, `Still rendering?`, `Run summary`, `Video ready to watch?`, `▶ Watch video`
- **Node Details:**
  - **Dry run?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Checks whether `dryRun` evaluates to true.
    - *Key Expressions/Variables:* `={{ $('Config').first().json.dryRun }}`
    - *Input/Output:* Input: `Check validation`. Output: True branch connects to `Save draft to editor`; False branch connects to `Submit render`.
    - *Edge Cases/Failures:* None.
  - **Save draft to editor**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `project`, Operation: `create`. Saves a project draft without rendering.
    - *Key Expressions/Variables:* Uses project name and payload.
    - *Input/Output:* Input: `Dry run?` (True branch). Output: Connects to `Dry run summary`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Handled via `onError: continueRegularOutput`.
  - **Dry run summary**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Formats a dry-run report including project metadata, credit quotes, and a direct web editor preview link (`editorLink`).
    - *Key Expressions/Variables:* Parses draft response objects.
    - *Input/Output:* Input: `Save draft to editor`. Output: None (Terminal node for dry runs).
    - *Edge Cases/Failures:* None.
  - **Submit render**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `render`, Operation: `create`, with `renderType` set to `video` and `waitForCompletion` set to `false`.
    - *Key Expressions/Variables:* `={{ JSON.stringify(({ payload: $json.payload }).payload) }}`
    - *Input/Output:* Input: `Dry run?` (False branch). Output: Connects to `Wait`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Insufficient account render credits.
  - **Wait**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control)
    - *Configuration Choices:* Pauses execution for a specified duration in seconds (`pollSeconds`, default 10s).
    - *Key Expressions/Variables:* `={{ $('Config').first().json.pollSeconds }}`
    - *Input/Output:* Input: `Submit render` or `Still rendering?`. Output: Connects to `Get render status`.
    - *Edge Cases/Failures:* None.
  - **Get render status**
    - *Type and Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (Community Node Integration)
    - *Configuration Choices:* Resource: `render`, Operation: `get`, querying status using `jobId`.
    - *Key Expressions/Variables:* `={{ $('Submit render').first().json.jobId }}`
    - *Input/Output:* Input: `Wait`. Output: Connects to `Render finished?`.
    - *Credentials:* `zvidApi`
    - *Edge Cases/Failures:* Network timeouts when querying status.
  - **Render finished?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Checks if `state === 'completed'`.
    - *Key Expressions/Variables:* `={{ $json.state === 'completed' }}`
    - *Input/Output:* Input: `Get render status`. Output: True branch connects to `Run summary`; False branch connects to `Still rendering?`.
    - *Edge Cases/Failures:* None.
  - **Still rendering?**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Fails fast if state is `failed`, or checks if polling timeout minutes have been exceeded; otherwise passes execution back to the `Wait` node.
    - *Key Expressions/Variables:* Compares run index against maximum allowed polls based on `timeoutMinutes` and `pollSeconds`.
    - *Input/Output:* Input: `Render finished?` (False branch). Output: Connects to `Wait`.
    - *Edge Cases/Failures:* Throws timeout errors if rendering exceeds `timeoutMinutes`.
  - **Run summary**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Processing)
    - *Configuration Choices:* Compiles the final live render report, extracts the finished video URL, and records successfully rendered Reddit post IDs to global static storage to prevent future duplicates.
    - *Key Expressions/Variables:* Accesses job results and build metadata.
    - *Input/Output:* Input: `Render finished?` (True branch). Output: Connects to `Video ready to watch?`.
    - *Edge Cases/Failures:* None.
  - **Video ready to watch?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Verifies that execution is not a dry run and that `videoUrl` matches a valid HTTP URL format.
    - *Key Expressions/Variables:* `={{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`
    - *Input/Output:* Input: `Run summary`. Output: True branch connects to `▶ Watch video`; False branch is empty.
    - *Edge Cases/Failures:* None.
  - **▶ Watch video**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration Choices:* Downloads the finished MP4 file binary to n8n output property `data`.
    - *Key Expressions/Variables:* `={{ $json.videoUrl }}`
    - *Input/Output:* Input: `Video ready to watch?` (True branch). Output: None (Terminal download node).
    - *Credentials:* None.
    - *Edge Cases/Failures:* Handled via `onError: continueRegularOutput`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every day at 9am | scheduleTrigger | Triggers workflow daily at 9 AM | None | Config | Workflow overview and setup |
| Test manually | manualTrigger | Manual execution entry point | None | Config | 1. Choose settings and an entry point |
| Config | set | Defines global workflow configuration | Every day at 9am, Test manually | Manual story? | 1. Choose settings and an [entry point](#) |
| Manual story? | if | Branches workflow based on source mode | Config | Pick story, Fetch top stories | 1. Choose settings and an entry point |
| Fetch top stories | httpRequest | Fetches top Reddit posts via OAuth | Manual story? | Pick story | 2. Choose a source story |
| Pick story | code | Validates, filters, and formats story input | Manual story?, Fetch top stories | Write script | 2. Choose a source story |
| Write script | httpRequest | Generates script JSON via OpenRouter LLM | Pick story | Parse script | 3. Write and check the script |
| Parse script | code | Normalizes text and segments shots | Write script | Expand scenes | 3. Write and check the script |
| Expand scenes | code | Allocates video budget and shot types | Parse script | Search stock clips | 4. Choose footage for each scene |
| Search stock clips | zvid | Searches Zvid stock library | Expand scenes | Pick scene clip | 4. Choose footage for each scene |
| Pick scene clip | code | Ranks and selects media per shot | Search stock clips | Music queries | 4. Choose footage for each scene |
| Music queries | code | Defines ambient music search tags | Pick scene clip | Find background music | 5. Check the background music |
| Find background music | zvid | Searches Zvid stock audio library | Music queries | Shortlist music | 5. Check the background music |
| Shortlist music | code | Filters and ranks music candidates | Find background music | Check music asset | 5. Check the background music |
| Check music asset | httpRequest | Verifies downloadability and file size | Shortlist music | Pick music | 5. Check the background music |
| Pick music | code | Selects final background track | Check music asset | Generate voiceover | 5. Check the background music |
| Generate voiceover | httpRequest | Generates speech audio with timestamps | Pick music | Voice + timings | 6. Create narration and timings |
| Voice + timings | code | Extracts word alignments and mp3 binary | Generate voiceover | Upload voiceover | 6. Create narration and timings |
| Upload voiceover | httpRequest | Uploads voiceover MP3 to Zvid | Voice + timings | Build project JSON | 6. Create narration and timings |
| Build project JSON | code | Assembles complete Zvid project payload | Upload voiceover | Validate project (free) | 7. Build and validate the design |
| Validate project (free) | zvid | Validates project schema and credits | Build project JSON | Check validation | 7. Build and validate the design |
| Check validation | code | Evaluates validation results and errors | Validate project (free) | Dry run? | 7. Build and validate the design |
| Dry run? | if | Branches between draft save and render | Check validation | Save draft to editor, Submit render | 8. Choose an optional editor preview |
| Save draft to editor | zvid | Creates a Zvid project draft | Dry run? | Dry run summary | 8. Choose an optional editor preview |
| Dry run summary | code | Outputs dry-run report and preview link | Save draft to editor | None | 8. Choose an optional editor preview |
| Submit render | zvid | Submits paid render job | Dry run? | Wait | 9. Render and wait for completion |
| Wait | wait | Pauses execution for polling interval | Submit render, Still rendering? | Get render status | 9. Render and wait for completion |
| Get render status | zvid | Queries Zvid render job status | Wait | Render finished? | 9. Render and wait for completion |
| Render finished? | if | Checks if rendering is completed | Get render status | Run summary, Still rendering? | 9. Render and wait for completion |
| Still rendering? | code | Validates poll state and timeout limits | Render finished? | Wait | 9. Render and wait for completion |
| Run summary | code | Compiles final live render report | Render finished? | Video ready to watch? | 9. Render and wait for completion |
| Video ready to watch? | if | Checks if output video is ready to download | Run summary | ▶ Watch video, None | 10. Review the finished media |
| ▶ Watch video | httpRequest | Downloads rendered MP4 file binary | Video ready to watch? | None | 10. Review the finished media |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Install Community Node:**
   - Go to **Settings → Community nodes** in n8n and install **`@zvid/n8n-nodes-zvid`** (Administrator privileges required).
2. **Create Triggers & Configuration:**
   - Add a **Schedule Trigger** node (`Every day at 9am`) set to trigger daily at hour `9`.
   - Add a **Manual Trigger** node (`Test manually`).
   - Add a **Set** node (`Config`). Set Mode to `Raw` and supply a JSON object containing your configuration parameters (`apiUrl`, `sourceMode`, `subreddit`, `minChars`, `maxChars`, `targetSeconds`, `llmModel`, `voiceId`, `voiceStability`, `voiceSimilarity`, `channelHandle`, `ctaText`, `dryRun`, etc.).
   - Add an **If** node (`Manual story?`) with condition: `={{ $('Config').first().json.sourceMode === 'manual' }}`. Connect both triggers into `Config`, and `Config` into `Manual story?`.
3. **Build Story Sourcing & Scripting Branch:**
   - **Reddit Path:** Create an **HTTP Request** node (`Fetch top stories`) connected to the *False* output of `Manual story?`. Method: `GET`, URL: `=https://oauth.reddit.com/r/{{ encodeURIComponent($('Config').first().json.subreddit) }}/top?t=day&limit=25&raw_json=1`. Set Header `User-Agent` to `={{ $('Config').first().json.redditUserAgent }}`. Configure credential: `redditOAuth2Api`.
   - **Story Picker:** Create a **Code** node (`Pick story`) connected to both `Manual story?` (*True*) and `Fetch top stories` (*False*). Paste the JavaScript code responsible for validating manual text or filtering Reddit posts and building the LLM prompt.
   - **LLM Script Writer:** Create an **HTTP Request** node (`Write script`) connected to `Pick story`. Method: `POST`, URL: `https://openrouter.ai/api/v1/chat/completions`, specify Body as JSON using `llmModel` and prompt text. Configure credential: `openRouterApi`. Enable `retryOnFail`.
   - **Script Parser:** Create a **Code** node (`Parse script`) connected to `Write script`. Paste the script parsing and shot-planning JavaScript code.
   - **Scene Expander:** Create a **Code** node (`Expand scenes`) connected to `Parse script`. Paste the scene expansion and video budget allocation code.
4. **Build Stock Media & Music Search:**
   - **Stock Search:** Add a **Zvid** node (`Search stock clips`) connected to `Expand scenes`. Resource: `stockMedia`, Operation: `search`. Map parameters `stockType` (`={{ $json.mediaType }}`) and `stockQuery` (`={{ $json.visualQuery }}`). Configure credential: `zvidApi`. Enable `alwaysOutputData` and `onError: continueRegularOutput`.
   - **Clip Picker:** Add a **Code** node (`Pick scene clip`) connected to `Search stock clips`. Paste the clip selection and anti-people scoring code.
   - **Music Branch:** Add a **Code** node (`Music queries`) connected to `Pick scene clip` to define tags (`chill`, `ambient`, `calm`).
   - **Find Music:** Add a **Zvid** node (`Find background music`) connected to `Music queries`. Resource: `stockMedia`, Operation: `search`, `stockType`: `audio`. Configure credential: `zvidApi`.
   - **Shortlist Music:** Add a **Code** node (`Shortlist music`) connected to `Find background music`.
   - **Check Music Asset:** Add an **HTTP Request** node (`Check music asset`) connected to `Shortlist music`. Method: `HEAD`, URL: `={{ $json.probeUrl }}`. Enable `alwaysOutputData` and `onError: continueRegularOutput`.
   - **Pick Music:** Add a **Code** node (`Pick music`) connected to `Check music asset`.
5. **Build Voiceover & Upload Branch:**
   - **Generate Voiceover:** Add an **HTTP Request** node (`Generate voiceover`) connected to `Pick music`. Method: `POST`, URL: `=https://api.elevenlabs.io/v1/text-to-speech/{voiceId}/with-timestamps`. Set Query Parameter `output_format` to `mp3_44100_128`. Configure credential: `httpHeaderAuth` (ElevenLabs API Key named `xi-api-key`). Enable `retryOnFail`.
   - **Voice & Timings:** Add a **Code** node (`Voice + timings`) connected to `Generate voiceover` to parse alignment data and output binary MP3 data (`audio/mpeg`).
   - **Upload Voiceover:** Add an **HTTP Request** node (`Upload voiceover`) connected to `Voice + timings`. Method: `POST`, URL: `={{ $('Config').first().json.apiUrl }}/api/uploads`. Content-Type: `multipart/form-data`, body parameter `file` from form binary data input field `data`. Configure credential: `zvidApi`. Enable `retryOnFail`.
6. **Build Project Assembly & Validation:**
   - **Project Builder:** Add a **Code** node (`Build project JSON`) connected to `Upload voiceover`. Paste the script assembling the complete Zvid payload (cover card, beats, subtitles, audios, visuals).
   - **Project Validation:** Add a **Zvid** node (`Validate project (free)`) connected to `Build project JSON`. Resource: `render`, Operation: `validate`. Pass project JSON. Configure credential: `zvidApi`.
   - **Validation Check:** Add a **Code** node (`Check validation`) connected to `Validate project (free)` to verify response status and extract error details if any.
7. **Build Rendering & Delivery Branch:**
   - **Dry Run Check:** Add an **If** node (`Dry run?`) connected to `Check validation`. Condition: `={{ $('Config').first().json.dryRun }}`.
   - **Dry Run Path:** Connect *True* branch to a **Zvid** node (`Save draft to editor`) (Resource: `project`, Operation: `create`) and follow with a **Code** node (`Dry run summary`).
   - **Live Render Path:** Connect *False* branch to a **Zvid** node (`Submit render`) (Resource: `render`, Operation: `create`, `renderType`: `video`, `waitForCompletion`: `false`). Configure credential: `zvidApi`.
   - **Polling Loop:**
     - Connect `Submit render` to a **Wait** node (`Wait`) set to unit `seconds` and amount `={{ $('Config').first().json.pollSeconds }}`.
     - Connect `Wait` to a **Zvid** node (`Get render status`) (Resource: `render`, Operation: `get`, Job ID: `={{ $('Submit render').first().json.jobId }}`). Configure credential: `zvidApi`.
     - Connect `Get render status` to an **If** node (`Render finished?`) with condition `={{ $json.state === 'completed' }}`.
     - Connect *False* branch of `Render finished?` to a **Code** node (`Still rendering?`) that checks failure states and timeouts, and connect its output back to the `Wait` node.
     - Connect *True* branch of `Render finished?` to a **Code** node (`Run summary`).
   - **Video Download:** Connect `Run summary` to an **If** node (`Video ready to watch?`) checking `={{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`. Connect its *True* output to an **HTTP Request** node (`▶ Watch video`) downloading the file binary into output property `data`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zvid Community Node Installation Guide | [Zvid Integration on n8n](https://n8n.io/integrations/zvid/) |
| Zvid API Keys Generation | [Zvid API Keys Portal](https://app.zvid.io/api-keys) |
| Workflow Credits & Attribution | Content generated utilizes public forum stories and automated stock assets. Ensure compliance with platform copyright and monetization policies before publishing. |