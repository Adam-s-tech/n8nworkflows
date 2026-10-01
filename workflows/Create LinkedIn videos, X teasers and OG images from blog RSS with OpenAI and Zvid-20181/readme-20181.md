Create LinkedIn videos, X teasers and OG images from blog RSS with OpenAI and Zvid

https://n8nworkflows.xyz/workflows/create-linkedin-videos--x-teasers-and-og-images-from-blog-rss-with-openai-and-zvid-20181


# Create LinkedIn videos, X teasers and OG images from blog RSS with OpenAI and Zvid

### 1. Workflow Overview

This workflow automates the process of converting the latest post from a blog RSS feed into a coordinated social media content kit. It uses OpenRouter to generate tailored copywriting and Zvid to render visual media assets, producing a square LinkedIn video, a landscape X (Twitter) teaser video, and an Open Graph share image. 

The workflow is organized into six functional blocks based on direct node-to-node execution paths:
- **1.1 Input Reception & Feed Filtering:** Sets workflow variables, reads the RSS feed, identifies the newest post, and verifies if it has already been processed in a previous production run.
- **1.2 Source Article Retrieval & AI Copy Generation:** Fetches the full article page (falling back to RSS text if necessary), structures a constrained prompt, queries OpenRouter via Chat Completions, and parses the resulting JSON copy.
- **1.3 Asset Composition, Music Verification & Validation:** Probes the background music URL, dynamically builds the complete project JSON payloads for each enabled output format, and validates them against the Zvid API without incurring rendering charges.
- **1.4 Execution Branching (Preview or Render):** Evaluates the `dryRun` configuration flag to either generate draft projects in the Zvid editor (`dryRun: true`) or proceed to paid rendering (`dryRun: false`).
- **1.5 Sequential Asset Rendering & Polling:** Iterates through enabled assets one by one, submits render jobs, handles rate-limit rejections with adaptive delays, and polls job statuses until completion or timeout.
- **1.6 Post-Processing & Review:** Summarizes completed assets, updates the workflow execution state to track processing markers, and optionally fetches binary outputs for local review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Feed Filtering
- **Overview:** Initializes workflow parameters, queries the RSS feed for blog entries, and filters out previously processed posts to prevent duplicate content generation.
- **Nodes Involved:** `Every day at 9am`, `Test manually`, `Config`, `Read blog feed`, `Pick newest post`, `New post?`, `Nothing new today`.
- **Node Details:**
  - **Every day at 9am**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers the workflow daily at 9:00 AM in the workflow's configured timezone.
    - Input: None
    - Output: Connects to `Config`
    - Edge Cases: Relies on n8n instance timezone settings.
  - **Test manually**
    - Type: `n8n-nodes-base.manualTrigger`
    - Role: Allows manual execution for testing and debugging.
    - Input: None
    - Output: Connects to `Config`
  - **Config**
    - Type: `n8n-nodes-base.set`
    - Role: Defines global variables including API endpoints, RSS feed URL, branding properties, color schemes, font selections, AI model parameters (`openai/gpt-4.1-mini`), output toggles, asset durations, and execution flags (`dryRun`).
    - Configuration: Raw JSON configuration output containing visual styles, asset switches (`makeLinkedInVideo`, `makeXTeaser`, `makeOgImage`), and timeout limits.
    - Input: `Every day at 9am`, `Test manually`
    - Output: Connects to `Read blog feed`
  - **Read blog feed**
    - Type: `n8n-nodes-base.rssFeedRead`
    - Role: Fetches entries from the RSS or Atom feed URL specified in the configuration.
    - Configuration: Uses expression `={{ $('Config').first().json.feedUrl }}`. Configured with retry on failure (3 tries, 5000ms interval).
    - Input: `Config`
    - Output: Connects to `Pick newest post`
    - Failure Types: Network timeouts, invalid feed URL, or HTTP 4xx/5xx errors from the remote blog server.
  - **Pick newest post**
    - Type: `n8n-nodes-base.code`
    - Role: Sorts feed items by date, extracts the newest post, strips HTML tags from feed content, normalizes domain strings, and compares the post's unique identifier (`guid`) against workflow static data to check for previous processing.
    - Configuration: JavaScript block handling date parsing, HTML tag stripping regex, and static data lookup (`$getWorkflowStaticData('global')`).
    - Input: `Read blog feed`
    - Output: Connects to `New post?`
    - Edge Cases: Throws an error if the feed contains no readable posts. Note that static data only persists across production runs; manual test runs will always treat the post as new.
  - **New post?**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates whether a new, unprocessed post was found.
    - Configuration: Checks condition `={{ $json.found }}` (Boolean true).
    - Input: `Pick newest post`
    - Output: True branch connects to `Fetch post page`; False branch connects to `Nothing new today`.
  - **Nothing new today**
    - Type: `n8n-nodes-base.code`
    - Role: Terminates the workflow gracefully when no new posts are found by returning a status payload explaining that the newest post was already processed.
    - Input: `New post?` (False branch)
    - Output: None (Terminal node)

#### 2.2 Source Article Retrieval & AI Copy Generation
- **Overview:** Fetches the full article webpage for deeper context, prepares a strict prompting payload, queries OpenRouter for localized social copy, and parses/sanitizes the returned JSON structure.
- **Nodes Involved:** `Fetch post page`, `Prepare kit prompt`, `Write kit copy`, `Parse kit copy`.
- **Node Details:**
  - **Fetch post page**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Downloads the HTML content of the selected blog post to provide richer context than the RSS snippet.
    - Configuration: GET request using the post link. Timeout set to 20,000ms. Configured with `neverError: true` to continue execution even if page fetching fails.
    - Input: `New post?` (True branch)
    - Output: Connects to `Prepare kit prompt`
    - Failure Types: HTTP errors, site blocking, or long load times (handled by fallback to RSS feed text).
  - **Prepare kit prompt**
    - Type: `n8n-nodes-base.code`
    - Role: Compares page text length against feed text, extracts main article content using regex patterns (`article`, `main`, `body`), and constructs a constrained prompt instructing the AI to generate specific copy elements within rigid word limits and ASCII-only formats.
    - Configuration: JavaScript snippet assembling system/user instructions for `liHeadline`, `liPoints`, `teaserHook`, `teaserLine`, `ogTitle`, and `ogKicker`.
    - Input: `Fetch post page`
    - Output: Connects to `Write kit copy`
  - **Write kit copy**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Calls the OpenRouter Chat Completions API to generate social kit copy based on the prepared prompt.
    - Configuration: POST request to `https://openrouter.ai/api/v1/chat/completions` using JSON body with model `openai/gpt-4.1-mini` and `response_format: { type: "json_object" }`. Uses OpenRouter API credentials (`openRouterApi`). Configured with 3 retries.
    - Input: `Prepare kit prompt`
    - Output: Connects to `Parse kit copy`
    - Credentials Required: OpenRouter API Key.
    - Failure Types: API rate limits, insufficient account balance, network errors, or model refusals.
  - **Parse kit copy**
    - Type: `n8n-nodes-base.code`
    - Role: Parses the LLM's JSON response, normalizes typographic characters (curly quotes, em-dashes) to ASCII, strips unwanted punctuation, and validates that all mandatory copy fields and takeaway points meet minimum criteria.
    - Configuration: JavaScript block with JSON parsing, regular expression character replacement, and structural validation checks.
    - Input: `Write kit copy`
    - Output: Connects to `Check music`
    - Failure Types: Throws an error if the model returns invalid JSON or omits mandatory fields.

#### 2.3 Asset Composition, Music Verification & Validation
- **Overview:** Probes the background music URL to verify availability and file size limits, constructs structured project JSON payloads for enabled media outputs, and validates them against the Zvid API.
- **Nodes Involved:** `Check music`, `Music guard`, `Build project JSON`, `Validate project (free)`, `Check validation`.
- **Node Details:**
  - **Check music**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Performs an HTTP HEAD request on the configured background music URL to verify accessibility and file size.
    - Configuration: HEAD request with a 15,000ms timeout and `neverError: true`.
    - Input: `Parse kit copy`
    - Output: Connects to `Music guard`
    - Failure Types: Unreachable URL, network timeout, or HTTP 4xx/5xx responses.
  - **Music guard**
    - Type: `n8n-nodes-base.code`
    - Role: Evaluates the music probe results against the maximum allowed byte size (`maxMusicBytes`, default 5MB). Strips the audio track if unreachable or oversized to prevent render failures.
    - Configuration: JavaScript verification logic comparing `content-length` headers against size caps.
    - Input: `Check music`
    - Output: Connects to `Build project JSON`
  - **Build project JSON**
    - Type: `n8n-nodes-base.code`
    - Role: Programmatically constructs the complete Zvid render project payloads for the square LinkedIn video (1080x1080), X teaser video (1280x720), and Open Graph share image (1200x630) using unified branding, color palettes, and typographic scales.
    - Configuration: Comprehensive JavaScript builder function (`buildKit`) that clamps copy lengths, calculates word wraps, builds SVG layouts, and outputs an array of items (one per enabled format).
    - Input: `Music guard`
    - Output: Connects to `Validate project (free)`
    - Edge Cases: Throws an error if all output toggles (`makeLinkedInVideo`, `makeXTeaser`, `makeOgImage`) are disabled in the configuration.
  - **Validate project (free)**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Submits each asset payload to the free Zvid validation endpoint to check schema correctness and calculate required rendering credits without consuming credits.
    - Configuration: Resource: `render`, Operation: `validate`. Project JSON payload expression `={{ JSON.stringify(({ payload: $json.payload }).payload) }}`. Uses Zvid API credentials.
    - Input: `Build project JSON`
    - Output: Connects to `Check validation`
    - Credentials Required: Zvid API Key (`https://api.zvid.io`).
    - Failure Types: Zvid schema validation rejections due to malformed payloads or unsupported attribute values.
  - **Check validation**
    - Type: `n8n-nodes-base.code`
    - Role: Inspects validation responses for each asset piece, throwing descriptive errors if validation fails or compiling credit requirements and warnings if successful.
    - Configuration: JavaScript processing loop evaluating HTTP status codes and validation response flags.
    - Input: `Validate project (free)`
    - Output: Connects to `Dry run?`

#### 2.4 Execution Branching (Preview or Render)
- **Overview:** Evaluates the `dryRun` configuration parameter to determine whether to save draft projects to the Zvid editor or proceed with paid video and image rendering.
- **Nodes Involved:** `Dry run?`, `Save draft to editor`, `Dry run summary`.
- **Node Details:**
  - **Dry run?**
    - Type: `n8n-nodes-base.if`
    - Role: Branches workflow execution based on the `dryRun` setting in the configuration.
    - Configuration: Evaluates `={{ $('Config').first().json.dryRun }}` (Boolean).
    - Input: `Check validation`
    - Output: 
      - True branch connects to `Save draft to editor`
      - False branch connects to `Render one asset at a time`
  - **Save draft to editor**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Creates draft projects in the Zvid workspace instead of rendering final media files, returning editor links and credit quotes.
    - Configuration: Resource: `project`, Operation: `create`. Project JSON and project name expressions mapping to item payloads. Configured with 3 retries and `continueRegularOutput` error handling.
    - Input: `Dry run?` (True branch)
    - Output: Connects to `Dry run summary`
    - Credentials Required: Zvid API Key.
    - Version Requirements: Requires installed Zvid community node version supporting `Project → Create`.
  - **Dry run summary**
    - Type: `n8n-nodes-base.code`
    - Role: Compiles a summary of all draft pieces, calculating total estimated credits required and providing direct editor links for human review.
    - Configuration: JavaScript aggregation script mapping dry run results and editor URLs.
    - Input: `Save draft to editor`
    - Output: None (Terminal node for dry run execution)

#### 2.5 Sequential Asset Rendering & Polling
- **Overview:** Iterates through validated assets sequentially, submits paid render jobs, handles rate-limit rejections with adaptive waiting periods, and polls job statuses until completion or timeout.
- **Nodes Involved:** `Render one asset at a time`, `Prepare render attempt`, `Submit render`, `Attach job to piece`, `Check submission rejection`, `Wait for render capacity`, `Wait`, `Get render status`, `Merge job status`, `Render finished?`, `Still rendering?`.
- **Node Details:**
  - **Render one asset at a time**
    - Type: `n8n-nodes-base.splitInBatches`
    - Role: Ensures assets are rendered sequentially (batch size: 1) to manage concurrency and server capacity.
    - Configuration: Batch size set to `1`.
    - Input: `Dry run?` (False branch), `Render finished?` (False branch)
    - Output: Connects to `Prepare render attempt` (when items remain) and `Run summary` (when batch completes).
  - **Prepare render attempt**
    - Type: `n8n-nodes-base.code`
    - Role: Initializes submission timestamps and attempt counters for the current asset.
    - Input: `Render one asset at a time`
    - Output: Connects to `Submit render`
  - **Submit render**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Submits the project payload to the Zvid render engine for actual video or image production.
    - Configuration: Resource: `render`, Operation: `create`. Render type determined dynamically (`image` or `video`). `waitForCompletion` set to `false`. Configured with `continueErrorOutput`.
    - Input: `Prepare render attempt`, `Wait for render capacity`
    - Output: 
      - Main output connects to `Attach job to piece`
      - Error output connects to `Check submission rejection`
    - Credentials Required: Zvid API Key.
    - Failure Types: API rate limits (HTTP 429), capacity errors, or invalid payloads.
  - **Attach job to piece**
    - Type: `n8n-nodes-base.code`
    - Role: Extracts the assigned `jobId` from the render submission response and attaches it to the asset record along with start timestamps.
    - Input: `Submit render`
    - Output: Connects to `Wait`
  - **Check submission rejection**
    - Type: `n8n-nodes-base.code`
    - Role: Analyzes submission errors, isolating explicit HTTP 429 rate-limit rejections for retries while halting on other fatal errors.
    - Configuration: JavaScript error inspection parsing response headers for `retry-after` instructions.
    - Input: `Submit render` (Error output)
    - Output: Connects to `Wait for render capacity`
  - **Wait for render capacity**
    - Type: `n8n-nodes-base.wait`
    - Role: Pauses execution for a calculated duration when encountering rate limits before attempting submission again.
    - Configuration: Wait unit: seconds; amount expression: `={{ $json.retrySeconds }}`.
    - Input: `Check submission rejection`
    - Output: Connects to `Submit render`
  - **Wait**
    - Type: `n8n-nodes-base.wait`
    - Role: Pauses execution between status polling intervals to monitor background render progress.
    - Configuration: Wait unit: seconds; amount expression: `={{ $('Config').first().json.pollSeconds }}` (default 10s).
    - Input: `Attach job to piece`, `Still rendering?`
    - Output: Connects to `Get render status`
  - **Get render status**
    - Type: `@zvid/n8n-nodes-zvid.zvid`
    - Role: Queries the Zvid API for the current processing state of a specific render job.
    - Configuration: Resource: `render`, Operation: `get`. Job ID expression: `={{ $json.jobId }}`. `waitForCompletion: false`. Configured with 3 retries.
    - Input: `Wait`
    - Output: Connects to `Merge job status`
    - Credentials Required: Zvid API Key.
  - **Merge job status**
    - Type: `n8n-nodes-base.code`
    - Role: Merges the latest job status data (state, failed reasons, result URLs) into the asset tracking object.
    - Input: `Get render status`
    - Output: Connects to `Render finished?`
  - **Render finished?**
    - Type: `n8n-nodes-base.if`
    - Role: Checks whether the render job has successfully completed.
    - Configuration: Condition checks `={{ $json.state === 'completed' }}`.
    - Input: `Merge job status`
    - Output:
      - True branch connects to `Render one asset at a time` (to process the next asset in the batch loop)
      - False branch connects to `Still rendering?`
  - **Still rendering?**
    - Type: `n8n-nodes-base.code`
    - Role: Verifies whether an incomplete render job has failed or exceeded the maximum timeout limit (`timeoutMinutes`, default 20 mins), throwing an error if so, or passing the item back to the wait loop if still in progress.
    - Input: `Render finished?` (False branch)
    - Output: Connects to `Wait`
    - Failure Types: Render job failure on Zvid servers or processing timeout.

#### 2.6 Post-Processing & Review
- **Overview:** Summarizes completed render outputs, updates static data tracking markers to record successful kit completion, and fetches binary media for review.
- **Nodes Involved:** `Run summary`, `Video ready to watch?`, `▶ Watch video`.
- **Node Details:**
  - **Run summary**
    - Type: `n8n-nodes-base.code`
    - Role: Aggregates finished render results, calculates total credits charged across the kit, updates the workflow static data `lastGuid` marker only after all kit pieces have successfully rendered, and outputs structured media URLs.
    - Configuration: JavaScript aggregation referencing validation items and updating workflow global static data (`$getWorkflowStaticData('global')`).
    - Input: `Render one asset at a time` (Batch completion)
    - Output: Connects to `Video ready to watch?`
  - **Video ready to watch?**
    - Type: `n8n-nodes-base.if`
    - Role: Verifies that the execution was not a dry run and that a valid video URL is present before attempting to download media.
    - Configuration: Condition checks `={{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
    - Input: `Run summary`
    - Output: True branch connects to `▶ Watch video`; False branch is unconnected.
  - **▶ Watch video**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Downloads the rendered media file binary data for review and inspection.
    - Configuration: GET request using `={{ $json.videoUrl }}`. Timeout set to 30,000ms. Response format set to `file` with data property `data`. Configured with `continueRegularOutput` and 3 retries.
    - Input: `Video ready to watch?` (True branch)
    - Output: None (Terminal node)
    - Failure Types: Network timeouts or broken media storage links.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Workflow overview and setup | n8n-nodes-base.stickyNote | Explains workflow purpose, setup instructions, community node installation, credential configuration, and customization options. | None | None | ## Turn a blog article into a social content kit<br><br>### How it works<br>For content marketers: read the newest RSS article, fetch its text, and ask OpenRouter for concise copy grounded in that source. Zvid produces a square LinkedIn video, a landscape X teaser and an Open Graph image. Assets render one at a time. Review the returned files before publishing; social posting is not included.<br><br>### Setup<br>1. Install **@zvid/n8n-nodes-zvid** from **Settings → Community nodes** before credentials. A workspace owner/admin may need to install it.<br>2. Create a **Zvid API** credential with [your key](https://app.zvid.io/api-keys) and Base URL **https://api.zvid.io**. Select it on every Zvid node.<br>3. Connect an **OpenRouter API** credential to **Write kit copy**. In **Config**, replace **feedUrl**, **brandName**, **domainOverride** and the call to action. Use a public article page and an accessible music URL.<br>4. Choose outputs with **makeLinkedInVideo**, **makeXTeaser** and **makeOgImage**. Keep at least one enabled.<br>5. **dryRun=false** renders and spends Zvid credits; AI may charge separately. Optional **dryRun=true** returns drafts and quotes when the installed Zvid node exposes **Project → Create**.<br>6. Test manually and inspect every asset before enabling the daily schedule. The last-post marker persists only on production executions.<br><br>### Customization<br>Adjust colors, fonts, timing and output toggles in Config. [Full setup guide](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/blog-content-kit.md). |
| Section 1 - Select the newest unused article | n8n-nodes-base.stickyNote | Groups nodes responsible for scheduled triggers, configuration, RSS fetching, and newest post filtering. | None | None | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| Section 2 - Write copy from the source | n8n-nodes-base.stickyNote | Groups nodes fetching article web pages, building copy prompts, querying OpenRouter, and parsing/validating JSON responses. | None | None | ## 2. Write copy from the source<br><br>Fetch the article page, falling back to RSS text when needed. OpenRouter returns short copy for the three formats; check the claims against the article before sharing. |
| Section 3 - Build and validate each asset | n8n-nodes-base.stickyNote | Groups nodes checking background music availability, building project JSON payloads, and validating assets against Zvid. | None | None | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Section 4 - Preview or render | n8n-nodes-base.stickyNote | Groups nodes evaluating the dry-run configuration and generating Zvid editor draft projects. | None | None | ## 4. Preview or render<br><br>The default false branch renders. For dryRun=true, the draft branch requires Project → Create support in the installed Zvid node and returns editor links with a quote; AI may still charge. |
| Section 5 - Render one piece at a time | n8n-nodes-base.stickyNote | Groups nodes managing sequential asset rendering, submission retries, rate-limit waits, status polling, and execution timeouts. | None | None | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Section 6 - Review videos and the share image | n8n-nodes-base.stickyNote | Groups nodes summarizing render runs, updating tracking state, verifying render completion, and downloading binary media files. | None | None | ## 6. Review videos and the share image<br><br>Run summary returns each completed link and records the post after the kit finishes. Open Watch video → Binary → data → View; select each item. Videos play as MP4s and the PNG opens as an image. |
| Every day at 9am | n8n-nodes-base.scheduleTrigger | Triggers the workflow daily at 9:00 AM. | None | Config | |
| Test manually | n8n-nodes-base.manualTrigger | Triggers the workflow manually for testing. | None | Config | |
| Config | n8n-nodes-base.set | Defines global configuration variables, brand styling, timing, and output flags. | Every day at 9am, Test manually | Read blog feed | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| Read blog feed | n8n-nodes-base.rssFeedRead | Fetches blog posts from the configured RSS/Atom feed URL. | Config | Pick newest post | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| Pick newest post | n8n-nodes-base.code | Selects the newest RSS article, cleans HTML text, parses domain names, and checks static data for duplicates. | Read blog feed | New post? | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| New post? | n8n-nodes-base.if | Evaluates whether a new, unprocessed post was found. | Pick newest post | Fetch post page, Nothing new today | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| Nothing new today | n8n-nodes-base.code | Terminates execution when no new posts are available. | New post? | None | ## 1. Select the newest unused article<br><br>Set your feed and branding in Config. The daily trigger runs at 9am in the workflow timezone; production runs skip the last completed post, while manual tests can rebuild it. |
| Fetch post page | n8n-nodes-base.httpRequest | Downloads the full article HTML page for enhanced context. | New post? | Prepare kit prompt | ## 2. Write copy from the source<br><br>Fetch the article page, falling back to RSS text when needed. OpenRouter returns short copy for the three formats; check the claims against the article before sharing. |
| Prepare kit prompt | n8n-nodes-base.code | Extracts article text and formats a structured prompt for the AI model. | Fetch post page | Write kit copy | ## 2. Write copy from the source<br><br>Fetch the article page, falling back to RSS text when needed. OpenRouter returns short copy for the three formats; check the claims against the article before sharing. |
| Write kit copy | n8n-nodes-base.httpRequest | Calls OpenRouter Chat Completions API to generate social kit copy. | Prepare kit prompt | Parse kit copy | ## 2. Write copy from the source<br><br>Fetch the article page, falling back to RSS text when needed. OpenRouter returns short copy for the three formats; check the claims against the article before sharing. |
| Parse kit copy | n8n-nodes-base.code | Parses and sanitizes the LLM JSON response, normalizing typography to ASCII. | Write kit copy | Check music | ## 2. Write copy from the source<br><br>Fetch the article page, falling back to RSS text when needed. OpenRouter returns short copy for the three formats; check the claims against the article before sharing. |
| Check music | n8n-nodes-base.httpRequest | Sends an HTTP HEAD request to verify background music availability and file size. | Parse kit copy | Music guard | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Music guard | n8n-nodes-base.code | Validates music file size against plan caps and strips audio if invalid. | Check music | Build project JSON | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Build project JSON | n8n-nodes-base.code | Programmatically builds Zvid project JSON payloads for LinkedIn video, X teaser, and OG image. | Music guard | Validate project (free) | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Validate project (free) | @zvid/n8n-nodes-zvid.zvid | Validates project payloads against the Zvid API and calculates credit costs. | Build project JSON | Check validation | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Check validation | n8n-nodes-base.code | Inspects validation results for schema errors or compiles credit estimates. | Validate project (free) | Dry run? | ## 3. Build and validate each asset<br><br>Check background music availability and size. Build the enabled formats, then validate each design and credit quote before submitting any render. |
| Dry run? | n8n-nodes-base.if | Branches execution based on the `dryRun` configuration setting. | Check validation | Save draft to editor, Render one asset at a time | ## 4. Preview or render<br><br>The default false branch renders. For dryRun=true, the draft branch requires Project → Create support in the installed Zvid node and returns editor links with a quote; AI may still charge. |
| Save draft to editor | @zvid/n8n-nodes-zvid.zvid | Creates draft projects in the Zvid editor when `dryRun` is enabled. | Dry run? | Dry run summary | ## 4. Preview or render<br><br>The default false branch renders. For dryRun=true, the draft branch requires Project → Create support in the installed Zvid node and returns editor links with a quote; AI may still charge. |
| Dry run summary | n8n-nodes-base.code | Summarizes draft editor links and total estimated credit costs. | Save draft to editor | None | ## 4. Preview or render<br><br>The default false branch renders. For dryRun=true, the draft branch requires Project → Create support in the installed Zvid node and returns editor links with a quote; AI may still charge. |
| Submit render | @zvid/n8n-nodes-zvid.zvid | Submits project payloads to the Zvid render engine. | Prepare render attempt, Wait for render capacity | Attach job to piece, Check submission rejection | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Attach job to piece | n8n-nodes-base.code | Extracts the render job ID and records submission start time. | Submit render | Wait | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Wait | n8n-nodes-base.wait | Pauses execution between polling status checks. | Attach job to piece, Still rendering? | Get render status | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Get render status | @zvid/n8n-nodes-zvid.zvid | Queries the Zvid API for current render job status. | Wait | Merge job status | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Merge job status | n8n-nodes-base.code | Merges job state and result data into the asset record. | Get render status | Render finished? | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Render finished? | n8n-nodes-base.if | Checks if the render job state is 'completed'. | Merge job status | Render one asset at a time, Still rendering? | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Still rendering? | n8n-nodes-base.code | Evaluates render timeouts and failure conditions. | Render finished? | Wait | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Run summary | n8n-nodes-base.code | Aggregates finished render results, calculates total credits, and updates static data markers. | Render one asset at a time | Video ready to watch? | ## 6. Review videos and the share image<br><br>Run summary returns each completed link and records the post after the kit finishes. Open Watch video → Binary → data → View; select each item. Videos play as MP4s and the PNG opens as an image. |
| ▶ Watch video | n8n-nodes-base.httpRequest | Downloads rendered media binary data for review. | Video ready to watch? | None | ## 6. Review videos and the share image<br><br>Run summary returns each completed link and records the post after the kit finishes. Open Watch video → Binary → data → View; select each item. Videos play as MP4s and the PNG opens as an image. |
| Video ready to watch? | n8n-nodes-base.if | Verifies execution is not a dry run and that a valid video URL exists. | Run summary | ▶ Watch video | ## 6. Review videos and the share image<br><br>Run summary returns each completed link and records the post after the kit finishes. Open Watch video → Binary → data → View; select each item. Videos play as MP4s and the PNG opens as an image. |
| Render one asset at a time | n8n-nodes-base.splitInBatches | Splits output array into single-item batches for sequential rendering. | Dry run?, Render finished? | Run summary, Prepare render attempt | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Prepare render attempt | n8n-nodes-base.code | Initializes submit start times and attempt counters for the current asset. | Render one asset at a time | Submit render | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Check submission rejection | n8n-nodes-base.code | Inspects HTTP 429 rate-limit rejections and extracts retry-after durations. | Submit render | Wait for render capacity | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |
| Wait for render capacity | n8n-nodes-base.wait | Pauses execution following rate limits before resubmitting. | Check submission rejection | Submit render | ## 5. Render one piece at a time<br><br>Each accepted asset finishes before the next submission. Only explicit rate-limit rejections retry after a delay; timeoutMinutes bounds retries and polling. Check the existing Zvid job before rerunning a timeout. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n without importing the JSON, follow these sequential steps:

1. **Install Community Node:**
   - Go to **Settings → Community nodes** in your n8n instance and install `@zvid/n8n-nodes-zvid`. (A workspace owner or admin may be required).

2. **Configure Credentials:**
   - Create a **Zvid API** credential with base URL `https://api.zvid.io` using your API key from `https://app.zvid.io/api-keys`.
   - Create an **OpenRouter API** credential with your OpenRouter API key.

3. **Create Triggers and Configuration Nodes:**
   - **Node 1:** Add a `Schedule Trigger` (`Every day at 9am`), set rule interval to trigger at hour `9`.
   - **Node 2:** Add a `Manual Trigger` (`Test manually`).
   - **Node 3:** Add an Edit Fields (`Set`) node named `Config`. Set mode to raw JSON and paste your configuration object (`apiUrl`, `feedUrl`, `brandName`, `brandInk`, `brandAccent`, `llmModel`, `dryRun: false`, output toggles, durations, music URL, etc.).
     - *Connections:* Connect both `Every day at 9am` and `Test manually` to `Config`.

4. **Set Up Feed Reading and Filtering:**
   - **Node 4:** Add an RSS Feed Read (`RSS Feed Read`) node named `Read blog feed`. Set URL to expression `={{ $('Config').first().json.feedUrl }}`. Enable retry on fail (3 tries, 5000ms wait).
     - *Connection:* Connect `Config` output to `Read blog feed`.
   - **Node 5:** Add a Code (`Code`) node named `Pick newest post`. Insert JavaScript to sort feed items by date, strip HTML tags, extract domain, and check workflow static data (`$getWorkflowStaticData('global')`) for existing `lastGuid`.
     - *Connection:* Connect `Read blog feed` to `Pick newest post`.
   - **Node 6:** Add an If (`If`) node named `New post?`. Set condition to evaluate `={{ $json.found }}` equals `true`.
     - *Connection:* Connect `Pick newest post` to `New post?`.
   - **Node 7:** Add a Code (`Code`) node named `Nothing new today` connected to the `false` branch of `New post?`.

5. **Set Up Web Scraping and AI Copy Generation:**
   - **Node 8:** Add an HTTP Request (`HTTP Request`) node named `Fetch post page`. Set method to GET, URL to `={{ $('Pick newest post').first().json.link }}`, timeout to 20,000ms, and `neverError: true`.
     - *Connection:* Connect `New post?` (true branch) to `Fetch post page`.
   - **Node 9:** Add a Code (`Code`) node named `Prepare kit prompt` to clean page HTML and construct the structured prompt for OpenRouter.
     - *Connection:* Connect `Fetch post page` to `Prepare kit prompt`.
   - **Node 10:** Add an HTTP Request (`HTTP Request`) node named `Write kit copy`. Set method to POST, URL to `https://openrouter.ai/api/v1/chat/completions`, authentication to Predefined Credential Type (`openRouterApi`), and specify JSON body with model `openai/gpt-4.1-mini` and `response_format: { type: "json_object" }`. Enable 3 retries.
     - *Connection:* Connect `Prepare kit prompt` to `Write kit copy`.
   - **Node 11:** Add a Code (`Code`) node named `Parse kit copy` to parse JSON, normalize punctuation to ASCII, and validate required copy fields.
     - *Connection:* Connect `Write kit copy` to `Parse kit copy`.

6. **Set Up Music Verification, Payload Building, and Validation:**
   - **Node 12:** Add an HTTP Request (`HTTP Request`) node named `Check music`. Set method to HEAD, URL to `={{ $('Config').first().json.musicUrl }}`, timeout to 15,000ms, and `neverError: true`.
     - *Connection:* Connect `Parse kit copy` to `Check music`.
   - **Node 13:** Add a Code (`Code`) node named `Music guard` to verify music file size against `maxMusicBytes`.
     - *Connection:* Connect `Check music` to `Music guard`.
   - **Node 14:** Add a Code (`Code`) node named `Build project JSON` containing the `buildKit` layout generator script to produce payloads for LinkedIn video, X teaser, and OG image.
     - *Connection:* Connect `Music guard` to `Build project JSON`.
   - **Node 15:** Add a Zvid (`Zvid`) node named `Validate project (free)`. Set resource to `render`, operation to `validate`, and project JSON expression to `={{ JSON.stringify(({ payload: $json.payload }).payload) }}` using your Zvid credential.
     - *Connection:* Connect `Build project JSON` to `Validate project (free)`.
   - **Node 16:** Add a Code (`Code`) node named `Check validation` to parse validation responses and credit quotes.
     - *Connection:* Connect `Validate project (free)` to `Check validation`.

7. **Configure Execution Branching (Dry Run vs. Render):**
   - **Node 17:** Add an If (`If`) node named `Dry run?` evaluating `={{ $('Config').first().json.dryRun }}`.
     - *Connection:* Connect `Check validation` to `Dry run?`.
   - **Node 18 (Dry Run Branch):** Add a Zvid (`Zvid`) node named `Save draft to editor`. Set resource to `project`, operation to `create`, and configure project JSON and name expressions. Set `neverError: true` and 3 retries.
     - *Connection:* Connect `Dry run?` (true branch) to `Save draft to editor`.
   - **Node 19:** Add a Code (`Code`) node named `Dry run summary` to compile editor links and credit totals.
     - *Connection:* Connect `Save draft to editor` to `Dry run summary`.

8. **Configure Sequential Rendering and Polling Loop:**
   - **Node 20:** Add a Split In Batches (`Split In Batches`) node named `Render one asset at a time` with batch size `1`.
     - *Connection:* Connect `Dry run?` (false branch) and `Render finished?` (false branch) to `Render one asset at a time`.
   - **Node 21:** Add a Code (`Code`) node named `Prepare render attempt`.
     - *Connection:* Connect `Render one asset at a time` (batch items output) to `Prepare render attempt`.
   - **Node 22:** Add a Zvid (`Zvid`) node named `Submit render`. Set resource to `render`, operation to `create`, render type expression to `={{ $json.payload.type === "image" ? "image" : "video" }}`, project JSON expression, and `waitForCompletion: false`. Set error output enabled (`continueErrorOutput`).
     - *Connection:* Connect `Prepare render attempt` (and `Wait for render capacity`) to `Submit render`.
   - **Node 23:** Add a Code (`Code`) node named `Attach job to piece` to extract `jobId`.
     - *Connection:* Connect `Submit render` (main output) to `Attach job to piece`.
   - **Node 24:** Add a Wait (`Wait`) node named `Wait` set to seconds using `={{ $('Config').first().json.pollSeconds }}`.
     - *Connection:* Connect `Attach job to piece` (and `Still rendering?`) to `Wait`.
   - **Node 25:** Add a Zvid (`Zvid`) node named `Get render status`. Set resource to `render`, operation to `get`, and job ID expression to `={{ $json.jobId }}` (`waitForCompletion: false`).
     - *Connection:* Connect `Wait` to `Get render status`.
   - **Node 26:** Add a Code (`Code`) node named `Merge job status` to merge job state and result URLs.
     - *Connection:* Connect `Get render status` to `Merge job status`.
   - **Node 27:** Add an If (`If`) node named `Render finished?` checking `={{ $json.state === 'completed' }}`.
     - *Connection:* Connect `Merge job status` to `Render finished?`.
   - **Node 28:** Add a Code (`Code`) node named `Still rendering?` to handle timeouts and job failures.
     - *Connection:* Connect `Render finished?` (false branch) to `Still rendering?`, and connect its output back to `Wait`.
     - *Note:* Connect `Render finished?` (true branch) back to `Render one asset at a time` to loop to the next asset. When the batch loop finishes, connect `Render one asset at a time` (done output) to `Run summary`.

9. **Configure Post-Processing and Review:**
   - **Node 29:** Add a Code (`Code`) node named `Run summary` to compile final results, update static data execution markers (`lastGuid`), and total up credits.
     - *Connection:* Connect `Render one asset at a time` (done output) to `Run summary`.
   - **Node 30:** Add an If (`If`) node named `Video ready to watch?` checking `={{ !$json.dryRun && /^https?:\/\//i.test(String(($json.videoUrl) || '')) }}`.
     - *Connection:* Connect `Run summary` to `Video ready to watch?`.
   - **Node 31:** Add an HTTP Request (`HTTP Request`) node named `▶ Watch video`. Set method to GET, URL to `={{ $json.videoUrl }}`, response format to `file` (property `data`), with `neverError: true` and 3 retries.
     - *Connection:* Connect `Video ready to watch?` (true branch) to `▶ Watch video`.

10. **Error Handling & Rate-Limit Sub-Loop:**
    - Add a Code (`Code`) node named `Check submission rejection` connected to the error output of `Submit render`.
    - Add a Wait (`Wait`) node named `Wait for render capacity` using retry seconds calculated from rate-limit headers. Connect its output back to `Submit render`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full workflow setup and configuration guide | [GitHub Documentation](https://github.com/Zvid-io/zvid-n8n/blob/master/workflows/blog-content-kit.md) |
| Zvid API Key generation | [Zvid API Keys Portal](https://app.zvid.io/api-keys) |
| Zvid Community Node package (`@zvid/n8n-nodes-zvid`) | Installed via n8n Settings → Community nodes |
| Default sample RSS blog feed | [Google Blog RSS](https://blog.google/rss/) |
| Default sample background music | [Pixabay Audio Asset](https://cdn.pixabay.com/audio/2025/04/21/audio_ed6f0ed574.mp3) |