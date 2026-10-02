Create Veo 3.1 social videos from Google Sheets with Gemini and PostWire

https://n8nworkflows.xyz/workflows/create-veo-3-1-social-videos-from-google-sheets-with-gemini-and-postwire-20251


# Create Veo 3.1 social videos from Google Sheets with Gemini and PostWire

### 1. Workflow Overview

This workflow automates the daily generation and multi-platform distribution of short-form vertical AI videos. It pulls pending video concepts from a Google Sheet, leverages Google Gemini to authoring video generation prompts and social media captions, renders an 8-second vertical clip using Veo 3.1, and publishes the media via the PostWire API. 

The execution logic is divided into four functional blocks:
- **1.1 Input Reception & State Management:** Triggers daily at 09:00, reads pending records marked as "ready" from Google Sheets, restricts processing to a single item, and transitions its status to "generating".
- **1.2 Content Planning:** Utilizes Google Gemini to generate structured metadata, including a Veo video prompt and platform-specific social copy, before mapping these elements into working variables.
- **1.3 Video Generation & Cloud Staging:** Requests an upload slot from PostWire, prompts Google Veo 3.1 to generate a 9:16 video binary, uploads the stream directly to PostWire, and finalizes the asset to retrieve a hosted media URL.
- **1.4 Publishing & Logging:** Dispatches the media across specified social networks via PostWire, evaluates success metrics using conditional logic, updates the originating Google Sheet row with tracking links, and triggers an email alert via Gmail if failures occur.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & State Management
- **Overview:** Initiates the automation daily, queries a Google Sheet for unproccessed row data, claims the top candidate, and locks its status to prevent duplicate execution attempts.
- **Nodes Involved:** `Every day at 9:00`, `Read the video ideas`, `Take the next idea`, `Mark the idea as generating`.

- **Node Details:**
  - **Every day at 9:00**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration Choices:* Configured to trigger daily at hour 09:00.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects to `Read the video ideas`.
    - *Version-Specific Requirements:* v1.2.
    - *Edge Cases/Failure Types:* Execution skipped if n8n instance is offline during the specific hour window.
  - **Read the video ideas**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration Choices:* Operation set to `read`, using filters to match rows where `lookupColumn` is "status" and `lookupValue` is "ready". Spreadsheet and sheet targets configured via UI.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input from `Every day at 9:00`, output to `Take the next idea`.
    - *Version-Specific Requirements:* v4.7.
    - *Edge Cases/Failure Types:* Google API rate limits, invalid spreadsheet credentials, or missing columns.
  - **Take the next idea**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Data Transformation)
    - *Configuration Choices:* Max items restricted to `1`.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input from `Read the video ideas`, output to `Mark the idea as generating`.
    - *Version-Specific Requirements:* v1.
    - *Edge Cases/Failure Types:* Empty input array if no rows match the "ready" filter.
  - **Mark the idea as generating**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration Choices:* Operation set to `update`, matching by `row_number`.
    - *Key Expressions or Variables:* `={{ $json.row_number }}` targeting the `row_number` column; status set explicitly to `generating`.
    - *Input/Output Connections:* Input from `Take the next idea`, output to `Plan the video with Gemini`.
    - *Version-Specific Requirements:* v4.7.
    - *Edge Cases/Failure Types:* Row locking conflicts or missing primary match keys.

---

#### Block 1.2: Content Planning
- **Overview:** Sends the selected idea and style parameters to Google Gemini to formulate structured prompt instructions and tailored social captions, then normalizes the resulting payload into operational variables.
- **Nodes Involved:** `Plan the video with Gemini`, `Read the plan`.

- **Node Details:**
  - **Plan the video with Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI / Transform)
    - *Configuration Choices:* Model set to `models/gemini-2.5-flash`, resource set to `text`, operation set to `message`, and forced JSON output enabled. Includes a rigorous system prompt detailing required keys (`veo_prompt`, `tiktok`, `instagram`, `youtube_title`, `youtube`, `linkedin`, `facebook`).
    - *Key Expressions or Variables:* `={{ 'Idea: ' + $('Take the next idea').item.json.idea + '\nStyle: ' + ($('Take the next idea').item.json.style || 'cinematic, natural light') }}`
    - *Input/Output Connections:* Input from `Mark the idea as generating`, output to `Read the plan`.
    - *Version-Specific Requirements:* v1.2.
    - *Edge Cases/Failure Types:* Gemini API timeouts, billing restrictions, or malformed JSON generation.
  - **Read the plan**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Sets multi-variable assignments mapping the Gemini JSON output and upstream item metadata.
    - *Key Expressions or Variables:* 
      - `plan`: `={{ JSON.parse($('Plan the video with Gemini').item.json.content.parts.map(p => p.text || '').join('')) }}`
      - `idea`: `={{ $('Take the next idea').item.json.idea }}`
      - `networks`: `={{ String($('Take the next idea').item.json.networks || 'tiktok, instagram, youtube, facebook').split(',').map(n => n.trim().toLowerCase()).filter(Boolean) }}`
      - `row_number`: `={{ $('Take the next idea').item.json.row_number }}`
    - *Input/Output Connections:* Input from `Plan the video with Gemini`, output to `Get an upload slot`.
    - *Version-Specific Requirements:* v3.4.
    - *Edge Cases/Failure Types:* Expression failure if Gemini returns text outside standard part blocks or invalid JSON structure.

---

#### Block 1.3: Video Generation & Cloud Staging
- **Overview:** Obtains an ingestion URL from PostWire, passes the generated visual prompt to Google Veo to render an 8-second MP4 file, uploads the binary asset, and resolves the final hosted resource path.
- **Nodes Involved:** `Get an upload slot`, `Generate the clip with Veo 3.1`, `Upload the clip`, `Get the video link`.

- **Node Details:**
  - **Get an upload slot**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / API Integration)
    - *Configuration Choices:* HTTP POST to `https://postwire.io/api/media/upload-url`, authentication using generic Header Auth (`Authorization: Bearer ...`). Response configuration set to `neverError: true`. Body payload sends explicit JSON content type.
    - *Key Expressions or Variables:* `={{ JSON.stringify({ content_type: 'video/mp4' }) }}`
    - *Input/Output Connections:* Input from `Read the plan`, output to `Generate the clip with Veo 3.1`.
    - *Version-Specific Requirements:* v4.2.
    - *Edge Cases/Failure Types:* Expired API tokens, network dropouts, or PostWire service unavailability.
  - **Generate the clip with Veo 3.1**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (AI / Media Generation)
    - *Configuration Choices:* Model set to `models/veo-3.1-fast-generate-preview`, resource set to `video`, operation set to `generate`, configured with 9:16 aspect ratio and an 8-second duration.
    - *Key Expressions or Variables:* `={{ $('Read the plan').item.json.plan.veo_prompt }}`
    - *Input/Output Connections:* Input from `Get an upload slot`, output to `Upload the clip`.
    - *Version-Specific Requirements:* v1.2.
    - *Edge Cases/Failure Types:* Insufficient quota, billing errors in Google AI Studio, or prompt safety rejections.
  - **Upload the clip**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / API Integration)
    - *Configuration Choices:* HTTP PUT request targeting the dynamic upload URL. Sends binary data using `inputDataFieldName: 'data'` with `contentType: 'binaryData'`.
    - *Key Expressions or Variables:* `={{ $('Get an upload slot').item.json.upload_url }}`
    - *Input/Output Connections:* Input from `Generate the clip with Veo 3.1`, output to `Get the video link`.
    - *Version-Specific Requirements:* v4.2.
    - *Edge Cases/Failure Types:* Large payload network termination or expired temporary signed upload URLs.
  - **Get the video link**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / API Integration)
    - *Configuration Choices:* HTTP GET request to `https://postwire.io/api/media/finalize`, authenticated via Header Auth. Query parameters configured to pass the storage path.
    - *Key Expressions or Variables:* Query parameter `path`: `={{ $('Get an upload slot').item.json.path }}`
    - *Input/Output Connections:* Input from `Upload the clip`, output to `Publish with PostWire`.
    - *Version-Specific Requirements:* v4.2.
    - *Edge Cases/Failure Types:* Finalization failure if upload is incomplete or corrupted.

---

#### Block 1.4: Publishing & Logging
- **Overview:** Submits the finalized video asset along with custom platform captions to PostWire, assesses publishing status, records operational results back into Google Sheets, and dispatches a notification email via Gmail if errors are detected.
- **Nodes Involved:** `Publish with PostWire`, `Every network published?`, `Mark the idea as posted`, `Write the fix in the row`, `Email me the fix`.

- **Node Details:**
  - **Publish with PostWire**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / API Integration)
    - *Configuration Choices:* HTTP POST request to `https://postwire.io/api/post`, authenticated via Header Auth. Constructs a detailed request body mapping platforms, localized text variants, titles, and idempotency protection rules.
    - *Key Expressions or Variables:* Inline IIFE extracting plans and networks, formatting per-platform payloads, and setting an idempotency key string format: `veo-idea-row-{{row_number}}-{{idea_snippet}}`.
    - *Input/Output Connections:* Input from `Get the video link`, output to `Every network published?`.
    - *Version-Specific Requirements:* v4.2.
    - *Edge Cases/Failure Types:* Platform token disconnects (e.g., expired Instagram or TikTok authorizations stored in PostWire).
  - **Every network published?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Logic / Flow Control)
    - *Configuration Choices:* Evaluates whether publishing results returned arrays with items and verified that all target results report `ok === true`.
    - *Key Expressions or Variables:* `={{ ($json.results || []).length > 0 && $json.results.every(r => r.ok) }}`
    - *Input/Output Connections:* Input from `Publish with PostWire`. True branch outputs to `Mark the idea as posted`; False branch outputs to `Write the fix in the row`.
    - *Version-Specific Requirements:* v2.2.
    - *Edge Cases/Failure Types:* Partial network success flagged as failure.
  - **Mark the idea as posted**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration Choices:* Operation set to `update`, matching by `row_number`. Updates status to `posted`, records current ISO timestamp, and lists generated link objects.
    - *Key Expressions or Variables:* 
      - `posted_at`: `={{ $now.toISO() }}`
      - `links`: `={{ ($json.results || []).filter(r => r.ok).map(r => r.platform + ' ' + (r.url || '')).join(' | ') }}`
    - *Input/Output Connections:* Input from `Every network published?` (True branch), terminal node.
    - *Version-Specific Requirements:* v4.7.
    - *Edge Cases/Failure Types:* Sheet connection errors during update.
  - **Write the fix in the row**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Action)
    - *Configuration Choices:* Operation set to `update`, matching by `row_number`. Dynamically determines error states and logs targeted remediation descriptions.
    - *Key Expressions or Variables:* 
      - `status`: `={{ ($json.posted || 0) > 0 ? 'partly posted' : 'needs fix' }}`
      - `note`: `={{ $json.error ? $json.error : ($json.results || []).filter(r => !r.ok).map(r => r.platform + ': ' + (r.fix || r.error)).join(' | ') }}`
    - *Input/Output Connections:* Input from `Every network published?` (False branch), output to `Email me the fix`.
    - *Version-Specific Requirements:* v4.7.
    - *Edge Cases/Failure Types:* Undefined error structures from PostWire API.
  - **Email me the fix**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action / Communication)
    - *Configuration Choices:* Sends a plaintext notification email reporting publishing issues to a configured recipient address.
    - *Key Expressions or Variables:* 
      - `subject`: `={{ 'Veo video for "' + $('Read the plan').item.json.idea + '" was not published everywhere' }}`
      - `message`: Inline IIFE formatting detailed failure reasons per target platform.
    - *Input/Output Connections:* Input from `Write the fix in the row`, terminal node.
    - *Version-Specific Requirements:* v2.1.
    - *Edge Cases/Failure Types:* Gmail OAuth authorization loss or invalid recipient address.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Documentation and setup guide | None | None | ## Create Veo 3.1 videos and post them everywhere<br>Turn ideas in Google Sheets into AI videos: Gemini writes the Veo 3.1 prompt and a caption per network, Veo renders an 8-second vertical clip, and PostWire posts it to TikTok, Instagram Reels, YouTube Shorts and Facebook.<br><br>### How it works<br>1. **Every day at 9:00** takes the first idea marked ready in the sheet and marks it as generating.<br>2. **Plan the video with Gemini** returns the Veo prompt and a caption for each network, plus the YouTube title.<br>3. **Generate the clip with Veo 3.1** renders it in 9:16 and the file goes straight to a PostWire upload slot: no Drive, no public link.<br>4. **Publish with PostWire** posts it to every network with its own text. The row is the idempotency key, so a retry never posts twice.<br>5. The row gets the post links, or the exact fix plus a **Gmail** report when a network refused.<br><br>### Setup<br>1. Create a free account at postwire.io, connect your networks and copy your API key. In n8n add a **Header Auth** credential (name `Authorization`, value `Bearer pw_live_…`) and select it in the three PostWire nodes.<br>2. Create a sheet with the columns idea, style, networks, status, posted_at, links and note, and mark ideas ready. Pick it in the four Sheets nodes.<br>3. Add a Google Gemini credential. Veo needs billing enabled in Google AI Studio (about one dollar per 8-second Veo 3.1 Fast clip); the captions fit the free tier.<br>4. Connect Gmail and put your address in **Email me the fix**.<br><br>### Customize<br>Switch to Veo 3.1 (not Fast) for quality, or add linkedin in the networks column. |
| **Sticky Note 2** | `n8n-nodes-base.stickyNote` | Section header documentation | None | None | ## 1. Next idea<br>The first idea marked ready, claimed so it is never generated twice. |
| **Sticky Note 3** | `n8n-nodes-base.stickyNote` | Section header documentation | None | None | ## 2. Plan with Gemini<br>One Veo prompt and a caption for each network. |
| **Sticky Note 4** | `n8n-nodes-base.stickyNote` | Section header documentation | None | None | ## 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Sticky Note 5** | `n8n-nodes-base.stickyNote` | Section header documentation | None | None | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |
| **Every day at 9:00** | `n8n-nodes-base.scheduleTrigger` | Daily schedule execution trigger | None | Read the video ideas | ## 1. Next idea<br>The first idea marked ready, claimed so it is never generated twice. |
| **Read the video ideas** | `n8n-nodes-base.googleSheets` | Read rows matching 'ready' status | Every day at 9:00 | Take the next idea | ## 1. Next idea<br>The first idea marked ready, claimed so it is never generated twice. |
| **Take the next idea** | `n8n-nodes-base.limit` | Restrict item stream to first item | Read the video ideas | Mark the idea as generating | ## 1. Next idea<br>The first idea marked ready, claimed so it is never generated twice. |
| **Mark the idea as generating** | `n8n-nodes-base.googleSheets` | Update row status to 'generating' | Take the next idea | Plan the video with Gemini | ## 1. Next idea<br>The first idea marked ready, claimed so it is never generated twice. |
| **Plan the video with Gemini** | `@n8n/n8n-nodes-langchain.googleGemini` | Generate prompt and social copy | Mark the idea as generating | Read the plan | ## 2. Plan with Gemini<br>One Veo prompt and a caption for each network. |
| **Read the plan** | `n8n-nodes-base.set` | Parse and map Gemini JSON output | Plan the video with Gemini | Get an upload slot | ## 2. Plan with Gemini<br>One Veo prompt and a caption for each network. |
| **Get an upload slot** | `n8n-nodes-base.httpRequest` | Request temporary PostWire upload URL | Read the plan | Generate the clip with Veo 3.1 | ## 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Generate the clip with Veo 3.1** | `@n8n/n8n-nodes-langchain.googleGemini` | Render 9:16 video via Veo | Get an upload slot | Upload the clip | ## 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Upload the clip** | `n8n-nodes-base.httpRequest` | Upload binary video to PostWire | Generate the clip with Veo 3.1 | Get the video link | ## 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Get the video link** | `n8n-nodes-base.httpRequest` | Finalize upload and fetch media URL | Upload the clip | Publish with PostWire | ## 3. Render with Veo 3.1 and upload<br>An upload slot, the 9:16 clip from Veo, then its link. No public link needed. |
| **Publish with PostWire** | `n8n-nodes-base.httpRequest` | Distribute video across platforms | Get the video link | Every network published? | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |
| **Every network published?** | `n8n-nodes-base.if` | Check if all network posts succeeded | Publish with PostWire | Mark the idea as posted,<br>Write the fix in the row | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |
| **Mark the idea as posted** | `n8n-nodes-base.googleSheets` | Log successful links and posted status | Every network published? | None | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |
| **Write the fix in the row** | `n8n-nodes-base.googleSheets` | Log error details and status | Every network published? | Email me the fix | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |
| **Email me the fix** | `n8n-nodes-base.gmail` | Send email alert for failed posts | Write the fix in the row | None | ## 4. Publish and log<br>Each network with its own text; the links or the fix go back to the row, and refusals are emailed. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually inside n8n:

1. **Trigger Node:** Create a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`), name it `Every day at 9:00`, and set its interval to trigger daily at hour `9`.
2. **Read Rows Node:** Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`), name it `Read the video ideas`, set operation to `read`, configure filters for `status` equal to `ready`, and select your target spreadsheet document and sheet.
3. **Limit Node:** Add a **Limit** node (`n8n-nodes-base.limit`), name it `Take the next idea`, and set maximum items to `1`.
4. **Update Status Node:** Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`), name it `Mark the idea as generating`, set operation to `update`, map matching columns to `row_number`, and assign `status` to `generating`.
5. **AI Planning Node:** Add a **Google Gemini Chat Model / Advanced AI** node (`@n8n/n8n-nodes-langchain.googleGemini`), name it `Plan the video with Gemini`, set model ID to `models/gemini-2.5-flash`, resource to `text`, operation to `message`, and enable JSON output. Configure the system message with the structural prompt rules and bind the user message to `={{ 'Idea: ' + $('Take the next idea').item.json.idea + '\nStyle: ' + ($('Take the next idea').item.json.style || 'cinematic, natural light') }}`. Connect a Google Gemini API credential.
6. **Set/Parse Plan Node:** Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`), name it `Read the plan`, and create assignments for `plan` (using `JSON.parse` over Gemini message parts), `idea`, `networks`, and `row_number`.
7. **PostWire Upload Slot Node:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`), name it `Get an upload slot`, set method to `POST`, URL to `https://postwire.io/api/media/upload-url`, and send JSON body `={{ JSON.stringify({ content_type: 'video/mp4' }) }}`. Configure authentication using generic Header Auth (`Authorization: Bearer <your_api_key>`). Set response option to `neverError: true`.
8. **Video Generation Node:** Add a **Google Gemini** node (`@n8n/n8n-nodes-langchain.googleGemini`), name it `Generate the clip with Veo 3.1`, set model ID to `models/veo-3.1-fast-generate-preview`, resource to `video`, operation to `generate`, aspect ratio to `9:16`, duration to `8` seconds, and prompt expression to `={{ $('Read the plan').item.json.plan.veo_prompt }}`.
9. **Upload Binary Node:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`), name it `Upload the clip`, set method to `PUT`, URL expression to `={{ $('Get an upload slot').item.json.upload_url }}`, send body to true with content type `binaryData`, and input field name to `data`.
10. **Finalize Video Node:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`), name it `Get the video link`, set method to `GET`, URL to `https://postwire.io/api/media/finalize`, configure Header Auth, and add query parameter `path` valued at `={{ $('Get an upload slot').item.json.path }}`. Set response option to `neverError: true`.
11. **Publish Post Node:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`), name it `Publish with PostWire`, set method to `POST`, URL to `https://postwire.io/api/post`, configure Header Auth, and pass the detailed JSON body mapping platforms, localized text variants, and idempotency keys. Set response option to `neverError: true`.
12. **Conditional Check Node:** Add an **If** node (`n8n-nodes-base.if`), name it `Every network published?`, and add a boolean condition expression verifying that results are populated and all returned statuses are true: `={{ ($json.results || []).length > 0 && $json.results.every(r => r.ok) }}`.
13. **Success Sheet Update Node:** Connect the `true` output of the If node to a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Mark the idea as posted`. Set operation to `update`, match by `row_number`, and set status to `posted`, `posted_at` to `={{ $now.toISO() }}`, and `links` to formatted result URLs.
14. **Error Sheet Update Node:** Connect the `false` output of the If node to a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Write the fix in the row`. Set operation to `update`, match by `row_number`, set status dynamically based on posting count, and map `note` and `posted_at`.
15. **Email Alert Node:** Add a **Gmail** node (`n8n-nodes-base.gmail`), name it `Email me the fix`, configure your target recipient email, map subject and message bodies dynamically from upstream publishing payloads, and connect Gmail OAuth2 credentials.
16. **Wiring Connections:** Connect the nodes sequentially following the direct paths defined in Block 1 through Block 4.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostWire platform integration setup and API credentials generation | [postwire.io](https://postwire.io) |
| Google AI Studio billing requirements for Veo 3.1 video generation | Google AI Studio Console |