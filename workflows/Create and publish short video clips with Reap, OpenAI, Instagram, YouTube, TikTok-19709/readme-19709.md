Create and publish short video clips with Reap, OpenAI, Instagram, YouTube, TikTok

https://n8nworkflows.xyz/workflows/create-and-publish-short-video-clips-with-reap--openai--instagram--youtube--tiktok-19709


# Create and publish short video clips with Reap, OpenAI, Instagram, YouTube, TikTok

### 1. Workflow Overview

This workflow automates the end-to-end production and multi-platform publishing of short-form video clips derived from a source video URL. It ingests a target video, uses the Reap API to generate AI-powered portrait clips, selects the highest-scoring clip based on virality metrics, creates tailored social media metadata via OpenAI, validates content and publishing parameters against platform-specific constraints, and distributes the final video to Instagram Reels, YouTube Shorts, and TikTok (via Zernio).

The execution logic is structured into the following functional blocks:
- **1.1 Input Reception & Validation:** Ingests source video URLs, applies initial configurations, and validates inputs before starting processing.
- **1.2 AI Video Processing (Reap):** Initiates a clipping project with Reap, polls the project status until complete, fetches generated clips, and selects the highest-scoring option.
- **1.3 AI Content Generation (OpenAI):** Configures social content parameters, generates platform-specific captions and hashtags, and validates them against character and constraint rules.
- **1.4 Publishing Preparation & Routing:** Sets global publishing parameters (including dry-run mode), validates them, and routes the execution branch to enabled platforms (Instagram, YouTube, and/or TikTok).
- **1.5 Instagram Publishing Branch:** Creates an Instagram media container via the Graph API, polls its processing status until ready, and publishes the Reel.
- **1.6 YouTube Publishing Branch:** Downloads the final selected clip and uploads it directly to YouTube as a Short.
- **1.7 TikTok Publishing Branch (Zernio):** Obtains an upload URL, downloads the video, merges upload parameters, uploads the media to Zernio, creates a TikTok post, and manages capacity-related retry logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation
- **Overview:** Initializes the workflow execution manually, defines source video and processing preferences, and validates structural integrity before contacting external services.
- **Nodes Involved:** 
  - `When clicking ‘Execute workflow’`
  - `Content Input`
  - `Video Settings`
  - `Validate Input`
- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - *Type & Technical Role:* Manual Trigger (v1). Initiates the workflow on-demand.
    - *Configuration:* Default manual trigger settings.
    - *Input / Output:* Inputs: None. Outputs: Triggers `Content Input`.
  - **Content Input**
    - *Type & Technical Role:* Set Node (v3.5). Sets initial parameters containing the source video URL.
    - *Configuration:* Defines variables such as `video_url`.
    - *Input / Output:* Inputs: `When clicking ‘Execute workflow’`. Outputs: `Video Settings`.
  - **Video Settings**
    - *Type & Technical Role:* Set Node (v3.5). Appends video generation configurations (e.g., target duration, aspect ratio settings).
    - *Configuration:* Sets payload variables required by the Reap API integration.
    - *Input / Output:* Inputs: `Content Input`. Outputs: `Validate Input`.
  - **Validate Input**
    - *Type & Technical Role:* Code Node (v2). Executes custom JavaScript to ensure the video URL is reachable and parameters are correctly formatted.
    - *Configuration:* Validates schema and checks for missing required fields. Throws an error if validation fails.
    - *Input / Output:* Inputs: `Video Settings`. Outputs: `Reap - Create Clip Project`.
    - *Edge Cases:* Malformed URLs or unsupported video formats will halt execution here.

#### 1.2 AI Video Processing (Reap)
- **Overview:** Sends the source video to Reap to initiate portrait clipping, polls the processing status asynchronously, fetches the generated clips, and programmatically selects the highest virality-scoring clip.
- **Nodes Involved:**
  - `Reap - Create Clip Project`
  - `Reap - Wait Before Status Check`
  - `Reap - Check Project Status`
  - `Reap - Project Completed?`
  - `Reap - Fetch Generated Clips`
  - `Reap - Select Best Clip`
- **Node Details:**
  - **Reap - Create Clip Project**
    - *Type & Technical Role:* HTTP Request (v4.5). Calls the Reap API to start a new clipping project.
    - *Configuration:* POST request using Bearer Token authentication against the Reap endpoint, passing the `video_url`.
    - *Input / Output:* Inputs: `Validate Input`. Outputs: `Reap - Wait Before Status Check`.
  - **Reap - Wait Before Status Check**
    - *Type & Technical Role:* Wait Node (v1.1). Pauses execution to allow backend video processing.
    - *Configuration:* Duration-based pause.
    - *Input / Output:* Inputs: `Reap - Create Clip Project`, `Reap - Project Completed?` (false branch). Outputs: `Reap - Check Project Status`.
  - **Reap - Check Project Status**
    - *Type & Technical Role:* HTTP Request (v4.5). Queries the Reap API for project completion updates.
    - *Configuration:* GET request referencing the project ID returned from creation.
    - *Input / Output:* Inputs: `Reap - Wait Before Status Check`. Outputs: `Reap - Project Completed?`.
  - **Reap - Project Completed?**
    - *Type & Technical Role:* IF Node (v2.3). Evaluates whether the Reap project status equals "completed".
    - *Configuration:* Condition checks project status field.
    - *Input / Output:* Inputs: `Reap - Check Project Status`. Outputs: True branch goes to `Reap - Fetch Generated Clips`; False branch loops back to `Reap - Wait Before Status Check`.
  - **Reap - Fetch Generated Clips**
    - *Type & Technical Role:* HTTP Request (v4.5). Retrieves the list of generated short-form clips and associated metadata from Reap.
    - *Configuration:* GET request to retrieve generated clip assets.
    - *Input / Output:* Inputs: `Reap - Project Completed?` (True). Outputs: `Reap - Select Best Clip`.
  - **Reap - Select Best Clip**
    - *Type & Technical Role:* Code Node (v2). Iterates through the returned clips and identifies the one with the highest virality score.
    - *Configuration:* JavaScript array reduction/sorting based on score parameters.
    - *Input / Output:* Inputs: `Reap - Fetch Generated Clips`. Outputs: `Social Content Settings`.

#### 1.3 AI Content Generation (OpenAI)
- **Overview:** Defines tone and platform parameters, invokes OpenAI to draft platform-optimized captions and hashtags, and parses/validates the generated copy against platform limits.
- **Nodes Involved:**
  - `Social Content Settings`
  - `AI - Generate Platform Content`
  - `Parse AI Content`
  - `Validate Platform Content`
- **Node Details:**
  - **Social Content Settings**
    - *Type & Technical Role:* Set Node (v3.5). Sets configuration variables for the LLM prompt (e.g., target tone, language, hashtag counts).
    - *Configuration:* Defines prompt instructions and metadata constraints.
    - *Input / Output:* Inputs: `Reap - Select Best Clip`. Outputs: `AI - Generate Platform Content`.
  - **AI - Generate Platform Content**
    - *Type & Technical Role:* OpenAI LangChain Node (v2.3). Generates tailored captions for Instagram, TikTok, and YouTube.
    - *Configuration:* Uses OpenAI credentials and a structured prompt combining clip metadata and social settings.
    - *Input / Output:* Inputs: `Social Content Settings`. Outputs: `Parse AI Content`.
    - *Edge Cases:* API rate limits or quota exhaustion.
  - **Parse AI Content**
    - *Type & Technical Role:* Code Node (v2). Extracts and structures the LLM response into distinct JSON payloads for each social platform.
    - *Configuration:* JSON parsing and cleaning script.
    - *Input / Output:* Inputs: `AI - Generate Platform Content`. Outputs: `Validate Platform Content`.
  - **Validate Platform Content**
    - *Type & Technical Role:* Code Node (v2). Ensures generated captions comply with platform character limits and hashtag requirements.
    - *Configuration:* Conditional length checks and string sanitization.
    - *Input / Output:* Inputs: `Parse AI Content`. Outputs: `Publishing Settings`.

#### 1.4 Publishing Preparation & Routing
- **Overview:** Establishes global publishing controls (such as dry-run flags), validates publishing parameters, and fans out execution to platform-specific routing nodes.
- **Nodes Involved:**
  - `Publishing Settings`
  - `Validate Publishing Settings`
  - `Publish to Instagram?`
  - `Publish to YouTube?`
  - `Publish to TikTok?`
- **Node Details:**
  - **Publishing Settings**
    - *Type & Technical Role:* Set Node (v3.5). Configures global publishing toggles (`dry_run`, platform enable/disable flags, credentials parameters).
    - *Configuration:* Sets boolean switches for each destination platform.
    - *Input / Output:* Inputs: `Validate Platform Content`. Outputs: `Validate Publishing Settings`.
  - **Validate Publishing Settings**
    - *Type & Technical Role:* Code Node (v2). Confirms that required configuration IDs and credentials exist for every enabled platform.
    - *Configuration:* Validates that enabled platforms have corresponding configuration tokens supplied.
    - *Input / Output:* Inputs: `Publishing Settings`. Outputs: Fans out concurrently to `Publish to Instagram?`, `Publish to YouTube?`, and `Publish to TikTok?`.
  - **Publish to Instagram?**
    - *Type & Technical Role:* IF Node (v2.3). Checks if Instagram publishing is enabled and dry-run conditions permit execution.
    - *Configuration:* Evaluates `publishing.instagram_enabled`.
    - *Input / Output:* Inputs: `Validate Publishing Settings`. Outputs: `Instagram - Create Reel Container`.
  - **Publish to YouTube?**
    - *Type & Technical Role:* IF Node (v2.3). Checks if YouTube publishing is enabled.
    - *Configuration:* Evaluates `publishing.youtube_enabled`.
    - *Input / Output:* Inputs: `Validate Publishing Settings`. Outputs: `Download Final Video`.
  - **Publish to TikTok?**
    - *Type & Technical Role:* IF Node (v2.3). Checks if TikTok publishing via Zernio is enabled.
    - *Configuration:* Evaluates `publishing.tiktok_enabled`.
    - *Input / Output:* Inputs: `Validate Publishing Settings`. Outputs: `Zernio - Create Upload URL` and `TikTok - Download Video`.

#### 1.5 Instagram Publishing Branch
- **Overview:** Creates an Instagram media container through the Graph API, periodically checks processing status until the container is ready, and publishes the Reel.
- **Nodes Involved:**
  - `Instagram - Create Reel Container`
  - `Wait1`
  - `Instagram - Check Reel Status`
  - `Instagram - Reel Ready?`
  - `Instagram - Publish Reel`
- **Node Details:**
  - **Instagram - Create Reel Container**
    - *Type & Technical Role:* HTTP Request (v4.5). Submits the video URL and caption to initialize an Instagram Reel media container.
    - *Configuration:* POST request using Meta Graph API Bearer token, passing video URL and caption text.
    - *Input / Output:* Inputs: `Publish to Instagram?`. Outputs: `Wait1`.
  - **Wait1**
    - *Type & Technical Role:* Wait Node (v1.1). Pauses execution briefly to allow Meta servers to process the video container.
    - *Configuration:* Time-based delay.
    - *Input / Output:* Inputs: `Instagram - Create Reel Container`, `Instagram - Reel Ready?` (False branch). Outputs: `Instagram - Check Reel Status`.
  - **Instagram - Check Reel Status**
    - *Type & Technical Role:* HTTP Request (v4.5). Queries the Meta Graph API for container processing status.
    - *Configuration:* GET request using container ID and Bearer token.
    - *Input / Output:* Inputs: `Wait1`. Outputs: `Instagram - Reel Ready?`.
  - **Instagram - Reel Ready?**
    - *Type & Technical Role:* IF Node (v2.3). Evaluates whether the container status indicates processing is complete.
    - *Configuration:* Checks status code/string in response.
    - *Input / Output:* Inputs: `Instagram - Check Reel Status`. Outputs: True branch goes to `Instagram - Publish Reel`; False branch loops back to `Wait1`.
  - **Instagram - Publish Reel**
    - *Type & Technical Role:* HTTP Request (v4.5). Publishes the processed container as a live Instagram Reel.
    - *Configuration:* POST request to trigger publication via Meta Graph API.
    - *Input / Output:* Inputs: `Instagram - Reel Ready?` (True). Outputs: Terminal node.
    - *Edge Cases:* API token expiration or Meta rate limiting.

#### 1.6 YouTube Publishing Branch
- **Overview:** Downloads the selected video file and uploads it directly to YouTube as a Short with generated metadata.
- **Nodes Involved:**
  - `Download Final Video`
  - `YouTube - Upload Short`
- **Node Details:**
  - **Download Final Video**
    - *Type & Technical Role:* HTTP Request (v4.5). Downloads the video binary from the Reap storage URL.
    - *Configuration:* GET request returning binary data.
    - *Input / Output:* Inputs: `Publish to YouTube?`. Outputs: `YouTube - Upload Short`.
  - **YouTube - Upload Short**
    - *Type & Technical Role:* YouTube Node (v1). Uploads the binary video file to YouTube with title, description, and tags.
    - *Configuration:* Uses YouTube OAuth2 credentials, setting category, privacy, and generated metadata fields.
    - *Input / Output:* Inputs: `Download Final Video`. Outputs: Terminal node.
    - *Edge Cases:* OAuth token revocation, quota limits, or oversized video files.

#### 1.7 TikTok Publishing Branch (Zernio)
- **Overview:** Initiates a Zernio upload session, downloads the video binary, merges upload data, uploads the asset to Zernio, creates a TikTok post, and handles temporary capacity errors with automated retries.
- **Nodes Involved:**
  - `Zernio - Create Upload URL`
  - `TikTok - Download Video`
  - `Zernio - Merge Upload Data`
  - `Zernio - Upload Video`
  - `Zernio - Create TikTok Post`
  - `Zernio - TikTok Capacity Error?`
  - `Zernio - Wait Before Retry`
  - `Zernio - Retry TikTok Post`
  - `Zernio - Retry Again?`
  - `Zernio - Wait Before Next Retry`
- **Node Details:**
  - **Zernio - Create Upload URL**
    - *Type & Technical Role:* HTTP Request (v4.5). Requests an authenticated temporary upload target from Zernio.
    - *Configuration:* POST request with Zernio Bearer token.
    - *Input / Output:* Inputs: `Publish to TikTok?`. Outputs: `Zernio - Merge Upload Data`.
  - **TikTok - Download Video**
    - *Type & Technical Role:* HTTP Request (v4.5). Downloads the selected video binary for Zernio processing.
    - *Configuration:* GET request returning binary data.
    - *Input / Output:* Inputs: `Publish to TikTok?`. Outputs: `Zernio - Merge Upload Data`.
  - **Zernio - Merge Upload Data**
    - *Type & Technical Role:* Merge Node (v3.2). Combines the upload URL context with the downloaded video binary.
    - *Configuration:* Merge mode set to combine inputs.
    - *Input / Output:* Inputs: `Zernio - Create Upload URL` and `TikTok - Download Video`. Outputs: `Zernio - Upload Video`.
  - **Zernio - Upload Video**
    - *Type & Technical Role:* HTTP Request (v4.5). Uploads the video binary to the Zernio-provided upload endpoint.
    - *Configuration:* PUT/POST binary upload request with Bearer authentication.
    - *Input / Output:* Inputs: `Zernio - Merge Upload Data`. Outputs: `Zernio - Create TikTok Post`.
  - **Zernio - Create TikTok Post**
    - *Type & Technical Role:* HTTP Request (v4.5). Instructs Zernio to publish the uploaded video to TikTok.
    - *Configuration:* POST request passing Zernio account ID, caption, and video reference.
    - *Input / Output:* Inputs: `Zernio - Upload Video`. Outputs: `Zernio - TikTok Capacity Error?`.
  - **Zernio - TikTok Capacity Error?**
    - *Type & Technical Role:* IF Node (v2.3). Checks if the API response indicates a temporary "at capacity" error.
    - *Configuration:* Evaluates error response codes/messages.
    - *Input / Output:* Inputs: `Zernio - Create TikTok Post`. Outputs: True branch goes to `Zernio - Wait Before Retry`.
  - **Zernio - Wait Before Retry**
    - *Type & Technical Role:* Wait Node (v1.1). Pauses execution before attempting a retry.
    - *Configuration:* Duration-based pause.
    - *Input / Output:* Inputs: `Zernio - TikTok Capacity Error?` (True). Outputs: `Zernio - Retry TikTok Post`.
  - **Zernio - Retry TikTok Post**
    - *Type & Technical Role:* HTTP Request (v4.5). Re-submits the TikTok post creation request to Zernio.
    - *Configuration:* Identical POST request to `Zernio - Create TikTok Post`.
    - *Input / Output:* Inputs: `Zernio - Wait Before Retry`. Outputs: `Zernio - Retry Again?`.
  - **Zernio - Retry Again?**
    - *Type & Technical Role:* IF Node (v2.3). Evaluates whether the retry attempt failed with a persistent capacity error.
    - *Configuration:* Error evaluation condition.
    - *Input / Output:* Inputs: `Zernio - Retry TikTok Post`. Outputs: True branch goes to `Zernio - Wait Before Next Retry`.
  - **Zernio - Wait Before Next Retry**
    - *Type & Technical Role:* Wait Node (v1.1). Secondary pause for exponential backoff handling.
    - *Configuration:* Extended duration pause.
    - *Input / Output:* Inputs: `Zernio - Retry Again?` (True). Outputs: `Zernio - Retry TikTok Post`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When clicking ‘Execute workflow’` | manualTrigger | Initiates manual execution | None | `Content Input` | |
| `Content Input` | set | Sets source video URL | `When clicking ‘Execute workflow’` | `Video Settings` | |
| `Video Settings` | set | Sets video clipping configuration | `Content Input` | `Validate Input` | |
| `Validate Input` | code | Validates input parameters | `Video Settings` | `Reap - Create Clip Project` | |
| `Reap - Create Clip Project` | httpRequest | Starts Reap video clipping project | `Validate Input` | `Reap - Wait Before Status Check` | |
| `Reap - Wait Before Status Check` | wait | Pauses for project processing | `Reap - Create Clip Project`, `Reap - Project Completed?` | `Reap - Check Project Status` | |
| `Reap - Check Project Status` | httpRequest | Queries Reap project status | `Reap - Wait Before Status Check` | `Reap - Project Completed?` | |
| `Reap - Project Completed?` | if | Checks if clipping is complete | `Reap - Check Project Status` | `Reap - Fetch Generated Clips`, `Reap - Wait Before Status Check` | |
| `Reap - Fetch Generated Clips` | httpRequest | Retrieves generated clips | `Reap - Project Completed?` | `Reap - Select Best Clip` | |
| `Reap - Select Best Clip` | code | Selects highest-scoring clip | `Reap - Fetch Generated Clips` | `Social Content Settings` | |
| `Social Content Settings` | set | Configures AI prompt settings | `Reap - Select Best Clip` | `AI - Generate Platform Content` | |
| `AI - Generate Platform Content` | openAi | Generates platform metadata | `Social Content Settings` | `Parse AI Content` | |
| `Parse AI Content` | code | Parses AI text response | `AI - Generate Platform Content` | `Validate Platform Content` | |
| `Validate Platform Content` | code | Validates metadata constraints | `Parse AI Content` | `Publishing Settings` | |
| `Publishing Settings` | set | Sets global publishing parameters | `Validate Platform Content` | `Validate Publishing Settings` | |
| `Validate Publishing Settings` | code | Validates publishing configuration | `Publishing Settings` | `Publish to Instagram?`, `Publish to YouTube?`, `Publish to TikTok?` | |
| `Publish to Instagram?` | if | Routes execution if IG enabled | `Validate Publishing Settings` | `Instagram - Create Reel Container` | |
| `Publish to YouTube?` | if | Routes execution if YouTube enabled | `Validate Publishing Settings` | `Download Final Video` | |
| `Publish to TikTok?` | if | Routes execution if TikTok enabled | `Validate Publishing Settings` | `Zernio - Create Upload URL`, `TikTok - Download Video` | |
| `Instagram - Create Reel Container` | httpRequest | Creates IG media container | `Publish to Instagram?` | `Wait1` | |
| `Wait1` | wait | Pauses for IG container processing | `Instagram - Create Reel Container`, `Instagram - Reel Ready?` | `Instagram - Check Reel Status` | |
| `Instagram - Check Reel Status` | httpRequest | Queries IG container status | `Wait1` | `Instagram - Reel Ready?` | |
| `Instagram - Reel Ready?` | if | Checks if IG Reel is ready | `Instagram - Check Reel Status` | `Instagram - Publish Reel`, `Wait1` | |
| `Instagram - Publish Reel` | httpRequest | Publishes Instagram Reel | `Instagram - Reel Ready?` | None | |
| `Download Final Video` | httpRequest | Downloads video binary for YouTube | `Publish to YouTube?` | `YouTube - Upload Short` | |
| `YouTube - Upload Short` | youTube | Uploads video as YouTube Short | `Download Final Video` | None | |
| `Zernio - Create Upload URL` | httpRequest | Requests Zernio upload URL | `Publish to TikTok?` | `Zernio - Merge Upload Data` | |
| `TikTok - Download Video` | httpRequest | Downloads video binary for TikTok | `Publish to TikTok?` | `Zernio - Merge Upload Data` | |
| `Zernio - Merge Upload Data` | merge | Combines upload URL and binary | `Zernio - Create Upload URL`, `TikTok - Download Video` | `Zernio - Upload Video` | |
| `Zernio - Upload Video` | httpRequest | Uploads video binary to Zernio | `Zernio - Merge Upload Data` | `Zernio - Create TikTok Post` | |
| `Zernio - Create TikTok Post` | httpRequest | Creates TikTok post via Zernio | `Zernio - Upload Video` | `Zernio - TikTok Capacity Error?` | |
| `Zernio - TikTok Capacity Error?` | if | Checks for Zernio capacity errors | `Zernio - Create TikTok Post` | `Zernio - Wait Before Retry` | |
| `Zernio - Wait Before Retry` | wait | Pauses before Zernio retry | `Zernio - TikTok Capacity Error?` | `Zernio - Retry TikTok Post` | |
| `Zernio - Retry TikTok Post` | httpRequest | Retries TikTok post creation | `Zernio - Wait Before Retry` | `Zernio - Retry Again?` | |
| `Zernio - Retry Again?` | if | Evaluates retry success | `Zernio - Retry TikTok Post` | `Zernio - Wait Before Next Retry` | |
| `Zernio - Wait Before Next Retry` | wait | Extended pause for exponential backoff | `Zernio - Retry Again?` | `Zernio - Retry TikTok Post` | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger and Inputs:**
   - Add a **Manual Trigger** node (`When clicking ‘Execute workflow’`).
   - Add a **Set** node (`Content Input`) to define the `video_url` parameter.
   - Add a **Set** node (`Video Settings`) to configure clip duration, aspect ratio, and Reap preferences.
   - Add a **Code** node (`Validate Input`) to ensure the source URL is valid. Connect: Trigger $\rightarrow$ `Content Input` $\rightarrow$ `Video Settings` $\rightarrow$ `Validate Input`.

2. **Configure Reap Processing Loop:**
   - Add an **HTTP Request** node (`Reap - Create Clip Project`) configured as a POST request using HTTP Bearer Auth with your Reap API key. Connect from `Validate Input`.
   - Add a **Wait** node (`Reap - Wait Before Status Check`). Connect from `Reap - Create Clip Project`.
   - Add an **HTTP Request** node (`Reap - Check Project Status`) configured as a GET request to poll the project endpoint. Connect from `Reap - Wait Before Status Check`.
   - Add an **IF** node (`Reap - Project Completed?`) checking if status equals `completed`. Connect true to `Reap - Fetch Generated Clips` and false back to `Reap - Wait Before Status Check`.
   - Add an **HTTP Request** node (`Reap - Fetch Generated Clips`) to download clip lists. Connect from `Reap - Project Completed?`.
   - Add a **Code** node (`Reap - Select Best Clip`) to select the highest-scoring clip based on virality metrics. Connect from `Reap - Fetch Generated Clips`.

3. **Configure OpenAI Metadata Generation:**
   - Add a **Set** node (`Social Content Settings`) to set tone and hashtag constraints. Connect from `Reap - Select Best Clip`.
   - Add an **OpenAI** LangChain node (`AI - Generate Platform Content`) configured with OpenAI credentials and a prompt incorporating clip context and social settings. Connect from `Social Content Settings`.
   - Add a **Code** node (`Parse AI Content`) to structure the model output. Connect from `AI - Generate Platform Content`.
   - Add a **Code** node (`Validate Platform Content`) to check character limits. Connect from `Parse AI Content`.

4. **Configure Publishing Parameters and Routing:**
   - Add a **Set** node (`Publishing Settings`) to configure global toggles (`dry_run`, platform enable switches, credentials). Connect from `Validate Platform Content`.
   - Add a **Code** node (`Validate Publishing Settings`) to verify required credentials. Connect from `Publishing Settings`.
   - Add three **IF** nodes (`Publish to Instagram?`, `Publish to YouTube?`, `Publish to TikTok?`) connected in parallel from `Validate Publishing Settings`.

5. **Build Instagram Branch:**
   - From `Publish to Instagram?`, add an **HTTP Request** (`Instagram - Create Reel Container`) using Meta Graph API Bearer Auth.
   - Add a **Wait** node (`Wait1`), an **HTTP Request** (`Instagram - Check Reel Status`), an **IF** node (`Instagram - Reel Ready?`), and an **HTTP Request** (`Instagram - Publish Reel`). Wire them sequentially, looping false back to `Wait1`.

6. **Build YouTube Branch:**
   - From `Publish to YouTube?`, add an **HTTP Request** (`Download Final Video`) to retrieve binary data from the Reap URL.
   - Add a **YouTube** node (`YouTube - Upload Short`) configured with YouTube OAuth2 credentials, mapping binary input and metadata fields.

7. **Build TikTok (Zernio) Branch:**
   - From `Publish to TikTok?`, branch to an **HTTP Request** (`Zernio - Create Upload URL`) and an **HTTP Request** (`TikTok - Download Video`).
   - Merge both paths using a **Merge** node (`Zernio - Merge Upload Data`).
   - Add an **HTTP Request** (`Zernio - Upload Video`), an **HTTP Request** (`Zernio - Create TikTok Post`), an **IF** node (`Zernio - TikTok Capacity Error?`), a **Wait** node (`Zernio - Wait Before Retry`), an **HTTP Request** (`Zernio - Retry TikTok Post`), an **IF** node (`Zernio - Retry Again?`), and a **Wait** node (`Zernio - Wait Before Next Retry`) to manage retry logic.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Reap AI Video Clipping Service | https://reap.video |
| Zernio TikTok Publishing Integration | Setup instructions and required account configuration are documented within the workflow environment. |
| Execution Recommendation | It is strongly recommended to run initial executions with the `dry_run` parameter enabled in `Publishing Settings` to validate generated clips, captions, and platform metadata before initiating live publishing. |