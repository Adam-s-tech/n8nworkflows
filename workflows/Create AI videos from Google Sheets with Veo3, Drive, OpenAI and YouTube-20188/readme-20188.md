Create AI videos from Google Sheets with Veo3, Drive, OpenAI and YouTube

https://n8nworkflows.xyz/workflows/create-ai-videos-from-google-sheets-with-veo3--drive--openai-and-youtube-20188


# Create AI videos from Google Sheets with Veo3, Drive, OpenAI and YouTube

### 1. Workflow Overview

This workflow automates the end-to-end process of generating AI-powered videos based on prompts stored in a Google Sheet, saving the assets to Google Drive, generating an SEO-optimized title using OpenAI, publishing the video to YouTube via an external posting service, and logging the final URLs back into the originating spreadsheet.

The execution logic is structured into four primary functional blocks:
- **1.1 Trigger and Data Intake:** Initiates the workflow either manually or via a timed schedule, then scans Google Sheets for unprocessed rows where the video output column is empty.
- **1.2 Video Generation and Polling:** Formats the prompt data, submits a generation request to the Google Veo3 engine via `fal.run`, and polls the processing queue until the rendering status returns `COMPLETED`.
- **1.3 AI Content Enhancement and Asset Retrieval:** Fetches the finalized video metadata, queries OpenAI (GPT-4o-mini) to build a YouTube-optimized title, and downloads the binary video file.
- **1.4 Publishing and Persistence:** Uploads the video file to Google Drive, pushes it alongside the generated title to `upload-post.com` for YouTube distribution, and writes both the direct video and YouTube links back to the Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Trigger and Data Intake

##### Overview
This block handles the manual or scheduled entry points of the workflow and retrieves new rows from Google Sheets that require video generation.

##### Nodes Involved
- When clicking ‘Test workflow’
- Schedule Trigger
- Get new video
- Set data

##### Node Details

- **When clicking ‘Test workflow’**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Entry point for manual execution.
  - **Configuration Choices:** Default parameters.
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Get new video`.
  - **Version Requirements:** v1.
  - **Edge Cases / Potential Failures:** None; strictly for developer testing.

- **Schedule Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Entry point for automated periodic execution.
  - **Configuration Choices:** Configured to run at minute-based intervals (recommended every 5 minutes).
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Get new video`.
  - **Version Requirements:** v1.2.
  - **Edge Cases / Potential Failures:** Ensure workflow activation status is set correctly to prevent missed triggers.

- **Get new video**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Retrieves rows from a target spreadsheet based on empty filter criteria.
  - **Configuration Choices:** Uses Google Sheets OAuth2 API. Filter configured to look up rows where the `VIDEO` column is empty.
  - **Key Expressions:** None.
  - **Connections:** Input from triggers; output connects to `Set data`.
  - **Version Requirements:** v4.5.
  - **Edge Cases / Potential Failures:** Authentication token expiration, missing sheet headers, or incorrect document/sheet IDs.

- **Set data**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Transforms raw row data into a formatted prompt payload.
  - **Configuration Choices:** Assigns a combined string mapping the prompt and video duration.
  - **Key Expressions:** `={{ $json.PROMPT }}\n\nDuration of the video: {{ $json.DURATION }}`
  - **Connections:** Input from `Get new video`; output connects to `Create Video`.
  - **Version Requirements:** v3.4.
  - **Edge Cases / Potential Failures:** Missing `PROMPT` or `DURATION` keys in the upstream item will output undefined strings.

---

#### Block 1.2: Video Generation and Polling

##### Overview
This block submits the generation request to the Google Veo3 API (`fal.run`), waits for processing, and checks status updates iteratively until rendering finishes.

##### Nodes Involved
- Create Video
- Wait 60 sec.
- Get status
- Completed?

##### Node Details

- **Create Video**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Submits a POST request to initiate AI video rendering.
  - **Configuration Choices:** POST method to `https://queue.fal.run/fal-ai/veo3` using HTTP Header Authentication.
  - **Key Expressions:** `={ "prompt": "{{$json.prompt}}" }`
  - **Connections:** Input from `Set data`; output connects to `Wait 60 sec.`.
  - **Version Requirements:** v4.2.
  - **Edge Cases / Potential Failures:** API key exhaustion, invalid payload structure, or remote service rate-limiting.

- **Wait 60 sec.**
  - **Type & Technical Role:** `n8n-nodes-base.wait` — Pauses workflow execution to allow asynchronous video rendering.
  - **Configuration Choices:** Set to an amount of 60 seconds.
  - **Key Expressions:** None.
  - **Connections:** Input from `Create Video` or `Completed?` (false branch); output connects to `Get status`.
  - **Version Requirements:** v1.1.
  - **Edge Cases / Potential Failures:** Long rendering times may require multiple polling loops.

- **Get status**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Queries the rendering queue for request progress.
  - **Configuration Choices:** GET request targeting the unique request ID endpoint using HTTP Header Authentication.
  - **Key Expressions:** `=https://queue.fal.run/fal-ai/veo3/requests/{{ $('Create Video').item.json.request_id }}/status`
  - **Connections:** Input from `Wait 60 sec.`; output connects to `Completed?`.
  - **Version Requirements:** v4.2.
  - **Edge Cases / Potential Failures:** Invalid `request_id` references or temporary network drops.

- **Completed?**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Evaluates whether the remote rendering job has finished.
  - **Configuration Choices:** Condition checks if `{{ $json.status }}` equals `COMPLETED`.
  - **Key Expressions:** `={{ $json.status }}` (compared against `COMPLETED`).
  - **Connections:** Input from `Get status`; True branch outputs to `Get Url Video`, False branch loops back to `Wait 60 sec.`.
  - **Version Requirements:** v2.2.
  - **Edge Cases / Potential Failures:** Infinite loops if the remote API fails to return a terminal state (`COMPLETED` or `FAILED`).

---

#### Block 1.3: AI Content Enhancement and Asset Retrieval

##### Overview
Once the video generation is confirmed, this block retrieves the definitive resource URL, generates an optimized YouTube title via OpenAI, and downloads the binary video file.

##### Nodes Involved
- Get Url Video
- Generate title
- Get File Video

##### Node Details

- **Get Url Video**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches the final metadata and download link for the completed video.
  - **Configuration Choices:** GET request using HTTP Header Authentication.
  - **Key Expressions:** `=https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`
  - **Connections:** Input from `Completed?` (True branch); output connects to `Generate title`.
  - **Version Requirements:** v4.2.
  - **Edge Cases / Potential Failures:** Missing `request_id` context.

- **Generate title**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.openAi` — Generates an SEO-optimized, engaging YouTube title.
  - **Configuration Choices:** Uses OpenAI model `gpt-4o-mini`. System instructions enforce a 60-character limit, keyword targeting, and matching input language.
  - **Key Expressions:** `=Input: {{ $('Get new video').item.json.PROMPT }}`
  - **Connections:** Input from `Get Url Video`; output connects to `Get File Video`.
  - **Version Requirements:** v1.8.
  - **Edge Cases / Potential Failures:** OpenAI rate limits, quota issues, or token authentication failures.

- **Get File Video**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Downloads the binary video file from the remote asset URL.
  - **Configuration Choices:** Standard GET request returning binary data.
  - **Key Expressions:** `={{ $('Get Url Video').item.json.video.url }}`
  - **Connections:** Input from `Generate title`; outputs split to `Upload Video` and `HTTP Request`.
  - **Version Requirements:** v4.2.
  - **Edge Cases / Potential Failures:** Large file payload timeouts or expired temporary file URLs.

---

#### Block 1.4: Publishing and Persistence

##### Overview
This final block saves the generated video to Google Drive, uploads it to YouTube via an external publishing API, and updates the original Google Sheet with both access links.

##### Nodes Involved
- Upload Video
- Update result
- HTTP Request (Upload-Post)
- Update Youtube URL

##### Node Details

- **Upload Video**
  - **Type & Technical Role:** `n8n-nodes-base.googleDrive` — Uploads the binary video to a specified Google Drive folder.
  - **Configuration Choices:** Uses Google Drive OAuth2 API. Dynamically names files using timestamps and original file names, targeting a specified folder ID.
  - **Key Expressions:** `={{ $now.format('yyyyLLddHHmmss') }}-{{ $('Get Url Video').item.json.video.file_name }}`
  - **Connections:** Input from `Get File Video`; output connects to `Update result`.
  - **Version Requirements:** v3.
  - **Edge Cases / Potential Failures:** Storage quota limits, incorrect folder IDs, or invalid OAuth scopes.

- **Update result**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Updates the originating row in Google Sheets with the video asset URL.
  - **Configuration Choices:** Operation set to `update`, matching rows by `row_number`.
  - **Key Expressions:** 
    - `VIDEO`: `={{ $('Get Url Video').item.json.video.url }}`
    - `row_number`: `={{ $('Get new video').item.json.row_number }}`
  - **Connections:** Input from `Upload Video`; terminal node (no output).
  - **Version Requirements:** v4.5.
  - **Edge Cases / Potential Failures:** Sheet mismatch or mismatched row identifier references.

- **HTTP Request (Upload-Post)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Uploads the video file and title to YouTube using `upload-post.com`.
  - **Configuration Choices:** POST request to `https://api.upload-post.com/api/upload` using multipart form-data. Uses HTTP Header Authentication.
  - **Key Expressions:** 
    - Title: `={{ $('Generate title').item.json.message.content }}`
    - Video binary input: `data`
  - **Connections:** Input from `Get File Video`; output connects to `Update Youtube URL`.
  - **Version Requirements:** v4.2.
  - **Edge Cases / Potential Failures:** Exceeding monthly free upload limits, unconfigured platform parameters, or invalid profile usernames (`YOUR_USERNAME`).

- **Update Youtube URL**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Updates the originating row in Google Sheets with the published YouTube link.
  - **Configuration Choices:** Operation set to `update`, matching rows by `row_number`.
  - **Key Expressions:** 
    - `YOUTUBE_URL`: `https://youtu.be/{{ $json.results.youtube.video_id }}`
    - `row_number`: `={{ $('Get new video').item.json.row_number }}`
  - **Connections:** Input from `HTTP Request (Upload-Post)`; terminal node (no output).
  - **Version Requirements:** v4.5.
  - **Edge Cases / Potential Failures:** Missing `video_id` in response payload or sheet row update conflicts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Test workflow’ | `n8n-nodes-base.manualTrigger` | Manual test entry point | None | Get new video | STEP 4 - MAIN FLOW<br>Start the workflow manually or periodically by hooking the "Schedule Trigger" node. It is recommended to set it at 5 minute intervals. |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Periodic automated entry point | None | Get new video | STEP 4 - MAIN FLOW<br>Start the workflow manually or periodically by hooking the "Schedule Trigger" node. It is recommended to set it at 5 minute intervals. |
| Get new video | `n8n-nodes-base.googleSheets` | Fetches unprocessed rows from the sheet | When clicking ‘Test workflow’, Schedule Trigger | Set data | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br><br>## STEP 1 - GOOGLE SHEET<br>Create a [Google Sheet like this](https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing).<br><br>Please insert:<br>- in the "PROMPT" column the accurate description of the video you want to create<br>- in the "DURATION" column the lenght of the video you want to create<br><br>Leave the "VIDEO" column unfilled. It will be inserted by the system once the video has been created |
| Set data | `n8n-nodes-base.set` | Formats prompt and duration string | Get new video | Create Video | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Create Video | `n8n-nodes-base.httpRequest` | Submits video generation job to fal.run | Set data | Wait 60 sec. | Set API Key created in Step 2<br><br>## STEP 2 - GET API KEY (YOURAPIKEY)<br>Create an account [here](https://fal.ai/) and obtain API KEY.<br>In the node "Create Image" set "Header Auth" and set:<br>- Name: "Authorization"<br>- Value: "Key YOURAPIKEY" |
| Wait 60 sec. | `n8n-nodes-base.wait` | Pauses execution for queue processing | Create Video, Completed? (False) | Get status | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Get status | `n8n-nodes-base.httpRequest` | Polls fal.run request progress status | Wait 60 sec. | Completed? | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Completed? | `n8n-nodes-base.if` | Evaluates if rendering status is COMPLETED | Get status | Get Url Video, Wait 60 sec. | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Get Url Video | `n8n-nodes-base.httpRequest` | Retrieves final video asset URL | Completed? (True) | Generate title | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Generate title | `@n8n/n8n-nodes-langchain.openAi` | Generates SEO-optimized YouTube title | Get Url Video | Get File Video | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Get File Video | `n8n-nodes-base.httpRequest` | Downloads binary video data | Generate title | Upload Video, HTTP Request | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Upload Video | `n8n-nodes-base.googleDrive` | Saves video file to Google Drive folder | Get File Video | Update result | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| Update result | `n8n-nodes-base.googleSheets` | Logs video URL back to Google Sheet row | Upload Video | None | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |
| HTTP Request | `n8n-nodes-base.httpRequest` | Uploads video and title to YouTube via upload-post.com | Get File Video | Update Youtube URL | Set YOUR_USERNAME in Step 3<br><br>## STEP 3 - Upload video on Youtube<br>- Find your API key in your [Upload-Post Manage Api Keys](https://app.upload-post.com/) 10 FREE uploads per month<br>- Set the the "Auth Header":<br>-- Name: Authorization<br>-- Value: Apikey YOUR_API_KEY_HERE<br>- Create profiles to manage your social media accounts. The "Profile" you choose will be used in the field YOUR_USRNAME (eg. test1 or test2).   |
| Update Youtube URL | `n8n-nodes-base.googleSheets` | Logs YouTube URL back to Google Sheet row | HTTP Request | None | # Generate AI Videos with Google Veo3, Save to Google Drive and Upload to YouTube<br><br>This workflow allows users to **generate AI videos** using **Google Veo3**, save them to **Google Drive**, generate optimized YouTube titles with GPT-4o, and **automatically upload them to YouTube** . The entire process is triggered from a Google Sheet that acts as the central interface for input and output.<br><br>IT automates video creation, uploading, and tracking, ensuring seamless integration between Google Sheets, Google Drive, Google Veo3, and YouTube.<br><br><br><br><br> |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**
   - Add a **Manual Trigger** (`When clicking ‘Test workflow’`).
   - Add a **Schedule Trigger** set to run every 5 minutes.
2. **Setup Data Intake:**
   - Add a **Google Sheets** node named `Get new video`. Set authentication via `Google Sheets OAuth2 API`. Configure operation to fetch rows using a filter on the `VIDEO` column (empty).
   - Add a **Set** node (`Set data`) with an assignment expression combining `PROMPT` and `DURATION` into a `prompt` variable.
3. **Setup AI Generation (Fal.run):**
   - Add an **HTTP Request** node named `Create Video`. Set method to `POST`, URL to `https://queue.fal.run/fal-ai/veo3`, and authenticate via HTTP Header Auth (`Authorization: Key YOUR_API_KEY`). Pass JSON body with the prompt.
   - Add a **Wait** node (`Wait 60 sec.`) set to 60 seconds.
   - Add an **HTTP Request** node named `Get status`. Set method to `GET`, URL referencing `https://queue.fal.run/fal-ai/veo3/requests/{{ $('Create Video').item.json.request_id }}/status`, using the same HTTP Header Auth.
   - Add an **If** node (`Completed?`) evaluating whether `{{ $json.status }}` equals `COMPLETED`. Connect the false output back to the **Wait** node.
4. **Setup AI Title Generation & Download:**
   - Add an **HTTP Request** node named `Get Url Video` (Method: `GET`, URL targeting the request ID endpoint) connected to the true output of the `Completed?` node.
   - Add an **OpenAI** node (`Generate title`) using model `gpt-4o-mini` with system instructions restricting titles to 60 characters and matching the input language.
   - Add an **HTTP Request** node named `Get File Video` to download the binary video file from the resolved URL.
5. **Setup Persistence & Publishing:**
   - Add a **Google Drive** node (`Upload Video`) using OAuth2 to save the binary file into a designated destination folder with a timestamped file name.
   - Add a **Google Sheets** node (`Update result`) to update the row’s `VIDEO` column with the asset URL.
   - Add an **HTTP Request** node (`HTTP Request` / upload-post.com) configured for multipart form-data (`POST` to `https://api.upload-post.com/api/upload`) with HTTP Header Auth (`Authorization: Apikey YOUR_API_KEY`). Pass parameters for `title`, `user` (profile username), `platform[]` (youtube), and binary video `data`.
   - Add a **Google Sheets** node (`Update Youtube URL`) to write the resulting YouTube URL (`https://youtu.be/{{ $json.results.youtube.video_id }}`) back to the corresponding row.
6. **Establish Connections:** Connect nodes strictly following the pathways outlined in Section 2 and Section 3.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheet Reference Template | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1pcoY9N_vQp44NtSRR5eskkL5Qd0N0BGq7Jh_4m-7VEQ/edit?usp=sharing) |
| Fal.ai Platform | [Fal.ai Platform](https://fal.ai/) |
| Upload-Post Management Console | [Upload-Post API Keys](https://app.upload-post.com/) |