Create 15 podcast content assets with OpenAI, Notion, and Gmail

https://n8nworkflows.xyz/workflows/create-15-podcast-content-assets-with-openai--notion--and-gmail-20091


# Create 15 podcast content assets with OpenAI, Notion, and Gmail

### 1. Workflow Overview

This workflow automates the transformation of a single podcast episode into 15 structured marketing assets using OpenAI, organizes them inside a Notion review page, and orchestrates human-in-the-loop approvals via Gmail. It bridges podcast production and content marketing by generating short-form clips, quote graphic briefs, newsletters, SEO show notes, and social media posts, all vetted through a Notion-based review workflow.

The logic is grouped into the following operational blocks:
- **1.1 Episode Intake:** Collects user submissions (episode details and transcript/audio files) via an n8n Form trigger and normalizes the payload.
- **1.2 Transcript Processing:** Determines whether a transcript was provided directly or needs conversion via OpenAI Whisper, resulting in a clean, timestamped transcript.
- **1.3 AI Content Generation:** Configures show settings and makes three sequential calls to OpenAI (GPT-4o-mini) to generate clips, quotes, newsletters, show notes, and social media assets.
- **1.4 Notion Review Page & Initial Notification:** Packages generated assets into structured Notion blocks, publishes them in optimized batches to a Notion database, and emails the designated reviewer.
- **1.5 Approval Watcher & Lifecycle Routing:** Periodically polls the Notion database for status changes, routes approved pages to publishing readiness, and triggers revision requests with feedback notes if changes are required.

---

### 2. Block-by-Block Analysis

#### 2.1 Episode Intake
- **Overview:** Receives raw inputs from the user via a web form and extracts binary or text-based content to prepare a unified payload for downstream nodes.
- **Nodes Involved:** `Episode Submission Form`, `Prepare Episode Input`, `Transcript Provided?`
- **Node Details:**
  - **Episode Submission Form** (`n8n-nodes-base.formTrigger`)
    - *Role:* Entry point for the workflow, hosting a user-facing form to collect metadata (Episode Title, Guest Name, Episode URL, Target Keywords, Submitter Email) alongside a transcript file or audio file, or a pasted text transcript.
    - *Configuration:* Form fields defined for text inputs, file attachments (`.srt`, `.vtt`, `.txt`, `.mp3`, `.m4a`, `.wav`, `.mp4`), and text areas.
    - *Input/Output:* Output connects to `Prepare Episode Input`.
    - *Edge Cases:* Missing required fields (Title, Email) are caught natively by the form interface.
  - **Prepare Episode Input** (`n8n-nodes-base.code`)
    - *Role:* JavaScript execution node that parses binary attachments or text blocks to ensure valid content is present before proceeding.
    - *Configuration:* Custom script inspecting binary inputs or text fields. Throws an explicit error if neither a transcript nor an audio file is provided.
    - *Input/Output:* Input from `Episode Submission Form`; output connects to `Transcript Provided?`.
    - *Edge Cases:* Throws an error (`Please upload a transcript...`) if inputs are blank.
  - **Transcript Provided?** (`n8n-nodes-base.if`)
    - *Role:* Conditional router determining whether the workflow proceeds with text-based parsing or audio transcription.
    - *Configuration:* Evaluates `{{ $json.hasTranscript }}` as a boolean true/false condition.
    - *Input/Output:* Input from `Prepare Episode Input`. True branch leads to `Parse Transcript Timestamps`; False branch leads to `Transcribe Audio with Whisper`.

#### 2.2 Transcript Processing
- **Overview:** Normalizes incoming text-based transcripts into standard `[mm:ss]` timestamped lines or processes raw audio files via OpenAI Whisper to generate timestamped text.
- **Nodes Involved:** `Parse Transcript Timestamps`, `Transcribe Audio with Whisper`, `Format Whisper Segments`
- **Node Details:**
  - **Parse Transcript Timestamps** (`n8n-nodes-base.code`)
    - *Role:* Strips SRT/VTT metadata or formats raw text lines into clean `[mm:ss]` timestamps.
    - *Configuration:* Custom JavaScript regex parser handling timestamps and removing WebVTT tags.
    - *Input/Output:* Input from `Transcript Provided?` (True); output connects to `Set Show Settings`.
  - **Transcribe Audio with Whisper** (`n8n-nodes-base.httpRequest`)
    - *Role:* Sends audio binary data to OpenAI's Whisper API using multipart form data to get a verbose JSON transcription.
    - *Configuration:* POST request to `https://api.openai.com/v1/audio/transcriptions` using `openAiApi` credentials with model `whisper-1` and `response_format` set to `verbose_json`. Timeout set to 600,000ms.
    - *Input/Output:* Input from `Transcript Provided?` (False); output connects to `Format Whisper Segments`.
    - *Edge Cases:* Audio file size limitations enforced by OpenAI (must be under 25 MB). Network timeouts handled by the 10-minute timeout configuration.
  - **Format Whisper Segments** (`n8n-nodes-base.code`)
    - *Role:* Converts Whisper's verbose JSON segment array into standard `[mm:ss]` timestamped lines.
    - *Configuration:* Custom JavaScript mapping segment start times to human-readable time codes.
    - *Input/Output:* Input from `Transcribe Audio with Whisper`; output connects to `Set Show Settings`.

#### 2.3 AI Content Generation
- **Overview:** Injects global podcast metadata and sequentially prompts OpenAI's GPT-4o-mini model to generate 15 distinct marketing assets.
- **Nodes Involved:** `Set Show Settings`, `Generate Clips & Quote Briefs`, `Write Newsletter & Show Notes`, `Write Social Assets`
- **Node Details:**
  - **Set Show Settings** (`n8n-nodes-base.set`)
    - *Role:* Defines global environment variables (show name, host name, target audience, brand voice, reviewer email) and aggregates episode metadata alongside the processed transcript.
    - *Configuration:* Explicit string assignments for branding constants combined with expression lookups pointing back to `Prepare Episode Input` and transcript nodes.
    - *Input/Output:* Inputs from `Parse Transcript Timestamps` or `Format Whisper Segments`; output connects to `Generate Clips & Quote Briefs`.
  - **Generate Clips & Quote Briefs** (`@n8n/n8n-nodes-langchain.openAi`)
    - *Role:* First LLM call extracting 5 short-form clip ideas and 5 quote graphic briefs from the transcript.
    - *Configuration:* Uses model `gpt-4o-mini` with a temperature of `0.6` and enforced JSON output. System prompt directs structured JSON extraction of clips (hooks, timestamps, platforms) and quotes.
    - *Input/Output:* Input from `Set Show Settings`; output connects to `Write Newsletter & Show Notes`.
  - **Write Newsletter & Show Notes** (`@n8n/n8n-nodes-langchain.openAi`)
    - *Role:* Second LLM call creating an email newsletter draft and SEO-optimized show notes (chapters, summaries, key takeaways).
    - *Configuration:* Uses model `gpt-4o-mini` with temperature `0.6` and forced JSON output.
    - *Input/Output:* Input from `Generate Clips & Quote Briefs`; output connects to `Write Social Assets`.
  - **Write Social Assets** (`@n8n/n8n-nodes-langchain.openAi`)
    - *Role:* Third LLM call producing platform-specific social text: an 8-10 slide LinkedIn carousel, a LinkedIn post, and an X (Twitter) thread.
    - *Configuration:* Uses model `gpt-4o-mini` with temperature `0.6` and forced JSON output.
    - *Input/Output:* Input from `Write Newsletter & Show Notes`; output connects to `Package 15 Assets`.

#### 2.4 Notion Review Page & Initial Notification
- **Overview:** Formats all generated text blocks, respects Notion API block limits by batching payloads, creates a new database entry set to "Needs Review", and notifies the reviewer via Gmail.
- **Nodes Involved:** `Package 15 Assets`, `Create Notion Review Page`, `Split Block Batches`, `Append Assets to Notion Page`, `Email Reviewer`
- **Node Details:**
  - **Package 15 Assets** (`n8n-nodes-base.code`)
    - *Role:* Compiles outputs from all three AI generation nodes into an array of structured Notion block objects and chunks them into batches of 90 items.
    - *Configuration:* Custom JavaScript assembling callouts, headings, rich text paragraphs, dividers, and bullet lists.
    - *Input/Output:* Input from `Write Social Assets`; output connects to `Create Notion Review Page`.
  - **Create Notion Review Page** (`n8n-nodes-base.notion`)
    - *Role:* Creates the parent page in the specified Notion database with property values initialized for tracking.
    - *Configuration:* Resource: `databasePage`, Database ID configured via placeholder (`YOUR_NOTION_DATABASE_ID`), properties mapped for `Status` (`Needs Review`), `Guest`, `Asset Count`, and `Submitted By`.
    - *Input/Output:* Input from `Package 15 Assets`; output connects to `Split Block Batches`.
  - **Split Block Batches** (`n8n-nodes-base.code`)
    - *Role:* Prepares batched children blocks alongside the newly created page ID to align with Notion API requirements.
    - *Configuration:* Custom JavaScript mapping batch arrays to discrete execution items.
    - *Input/Output:* Input from `Create Notion Review Page`; output connects to `Append Assets to Notion Page`.
  - **Append Assets to Notion Page** (`n8n-nodes-base.httpRequest`)
    - *Role:* Appends content blocks to the Notion page using PATCH requests with built-in batching constraints.
    - *Configuration:* HTTP PATCH request to `https://api.notion.com/v1/blocks/{{ $json.pageId }}/children` with Notion version header (`2022-06-28`), using `notionApi` credentials and request batching (`batchSize: 1`, `batchInterval: 400ms`).
    - *Input/Output:* Input from `Split Block Batches`; output connects to `Email Reviewer`.
  - **Email Reviewer** (`n8n-nodes-base.gmail`)
    - *Role:* Sends an email notification to the designated reviewer containing direct links to the Notion review page.
    - *Configuration:* Uses Gmail OAuth2 credentials, pulling recipient address from show settings and constructing HTML message bodies with review instructions. Set to `executeOnce`.
    - *Input/Output:* Input from `Append Assets to Notion Page`.

#### 2.5 Approval Watcher & Lifecycle Routing
- **Overview:** Polls the Notion database for status adjustments, automatically routing approved entries toward publishing readiness while alerting the submitter, or routing revision requests back with feedback notes.
- **Nodes Involved:** `Watch Notion Review Queue`, `Route by Review Status`, `Mark Ready to Publish`, `Tell Submitter It's Approved`, `Mark In Revision`, `Send Revision Notes`
- **Node Details:**
  - **Watch Notion Review Queue** (`n8n-nodes-base.notionTrigger`)
    - *Role:* Polls the Notion database every minute to monitor page updates.
    - *Configuration:* Event configured to `pagedUpdatedInDatabase` with polling interval set to every minute against `YOUR_NOTION_DATABASE_ID`.
    - *Input/Output:* Output connects to `Route by Review Status`.
  - **Route by Review Status** (`n8n-nodes-base.switch`)
    - *Role:* Directs workflow execution based on the updated Notion status property.
    - *Configuration:* Switch rules evaluating `{{ $json.property_status }}` against two string values: `Approved` and `Changes Requested`.
    - *Input/Output:* Input from `Watch Notion Review Queue`; outputs branch to `Mark Ready to Publish` and `Mark In Revision`.
  - **Mark Ready to Publish** (`n8n-nodes-base.notion`)
    - *Role:* Updates the Notion database page status to "Ready to Publish" and logs the approval timestamp.
    - *Configuration:* Resource: `databasePage`, Operation: `update`, properties set to `Status` (`Ready to Publish`) and `Approved On` (`{{ $now.toISO() }}`).
    - *Input/Output:* Input from `Route by Review Status` (Approved branch); output connects to `Tell Submitter It's Approved`.
  - **Tell Submitter It's Approved** (`n8n-nodes-base.gmail`)
    - *Role:* Emails the original submitter confirming that their content pack has been approved.
    - *Configuration:* Uses Gmail OAuth2 credentials, fetching the recipient email from `{{ $('Watch Notion Review Queue').item.json.property_submitted_by }}`.
    - *Input/Output:* Input from `Mark Ready to Publish`.
  - **Mark In Revision** (`n8n-nodes-base.notion`)
    - *Role:* Updates the Notion database page status to "In Revision" when feedback is provided.
    - *Configuration:* Resource: `databasePage`, Operation: `update`, property `Status` set to `In Revision`.
    - *Input/Output:* Input from `Route by Review Status` (Changes Requested branch); output connects to `Send Revision Notes`.
  - **Send Revision Notes** (`n8n-nodes-base.gmail`)
    - *Role:* Emails review notes and feedback back to the episode submitter.
    - *Configuration:* Uses Gmail OAuth2 credentials, pulling review notes from Notion properties and targeting the submitter email.
    - *Input/Output:* Input from `Mark In Revision`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup guidelines | None | None | Podcast to 15 Assets with Notion Approval... |
| Section - Episode Intake | n8n-nodes-base.stickyNote | Groups the intake form and initial validation logic | None | None | Episode Intake: A hosted form collects episode details... |
| Section - Transcript with Timestamps | n8n-nodes-base.stickyNote | Groups transcript parsing and Whisper transcription logic | None | None | Transcript with Timestamps: SRT and VTT files are parsed... |
| Section - AI Asset Generation | n8n-nodes-base.stickyNote | Groups show settings configuration and OpenAI generation steps | None | None | Settings & AI Generation: Edit show name, host, audience... |
| Section - Notion Review Page | n8n-nodes-base.stickyNote | Groups Notion page structuring, batching, creation, and notification | None | None | Notion Review Page: Builds the page layout, creates the entry... |
| Section - Approval Watcher | n8n-nodes-base.stickyNote | Groups the database polling trigger and status switch | None | None | Approval Watcher: Polls the database every minute... |
| Section - Publish or Revise | n8n-nodes-base.stickyNote | Groups publishing status updates and email notification handlers | None | None | Publish or Revise: Approved pages move to Ready to Publish... |
| Credentials & Security | n8n-nodes-base.stickyNote | Documents required external API integrations | None | None | Credentials & Security: API key for OpenAI... |
| Episode Submission Form | n8n-nodes-base.formTrigger | Collects podcast submission details and files | None | Prepare Episode Input | Episode Intake: A hosted form collects episode details... |
| Prepare Episode Input | n8n-nodes-base.code | Normalizes intake data and validates files | Episode Submission Form | Transcript Provided? | Episode Intake: A hosted form collects episode details... |
| Transcript Provided? | n8n-nodes-base.if | Evaluates whether a transcript was uploaded or pasted | Prepare Episode Input | Parse Transcript Timestamps, Transcribe Audio with Whisper | Episode Intake: A hosted form collects episode details... |
| Parse Transcript Timestamps | n8n-nodes-base.code | Parses text transcripts into timestamped lines | Transcript Provided? | Set Show Settings | Transcript with Timestamps: SRT and VTT files are parsed... |
| Transcribe Audio with Whisper | n8n-nodes-base.httpRequest | Transcribes audio via OpenAI Whisper API | Transcript Provided? | Format Whisper Segments | Transcript with Timestamps: SRT and VTT files are parsed... |
| Format Whisper Segments | n8n-nodes-base.code | Formats Whisper JSON response into timestamped lines | Transcribe Audio with Whisper | Set Show Settings | Transcript with Timestamps: SRT and VTT files are parsed... |
| Set Show Settings | n8n-nodes-base.set | Sets global podcast metadata and environment variables | Parse Transcript Timestamps, Format Whisper Segments | Generate Clips & Quote Briefs | Settings & AI Generation: Edit show name, host, audience... |
| Generate Clips & Quote Briefs | @n8n/n8n-nodes-langchain.openAi | Generates short clips and quote briefs via OpenAI | Set Show Settings | Write Newsletter & Show Notes | Settings & AI Generation: Edit show name, host, audience... |
| Write Newsletter & Show Notes | @n8n/n8n-nodes-langchain.openAi | Generates email newsletter and SEO show notes via OpenAI | Generate Clips & Quote Briefs | Write Social Assets | Settings & AI Generation: Edit show name, host, audience... |
| Write Social Assets | @n8n/n8n-nodes-langchain.openAi | Generates social posts, threads, and carousel copy via OpenAI | Write Newsletter & Show Notes | Package 15 Assets | Settings & AI Generation: Edit show name, host, audience... |
| Package 15 Assets | n8n-nodes-base.code | Formats and chunks all assets into Notion blocks | Write Social Assets | Create Notion Review Page | Notion Review Page: Builds the page layout, creates the entry... |
| Create Notion Review Page | n8n-nodes-base.notion | Creates the parent review page in Notion | Package 15 Assets | Split Block Batches | Notion Review Page: Builds the page layout, creates the entry... |
| Split Block Batches | n8n-nodes-base.code | Splits blocks into smaller batches to respect API limits | Create Notion Review Page | Append Assets to Notion Page | Notion Review Page: Builds the page layout, creates the entry... |
| Append Assets to Notion Page | n8n-nodes-base.httpRequest | Appends content blocks to Notion in batches via HTTP PATCH | Split Block Batches | Email Reviewer | Notion Review Page: Builds the page layout, creates the entry... |
| Email Reviewer | n8n-nodes-base.gmail | Sends review notification email with Notion link | Append Assets to Notion Page | None | Notion Review Page: Builds the page layout, creates the entry... |
| Watch Notion Review Queue | n8n-nodes-base.notionTrigger | Polls Notion database for updates | None | Route by Review Status | Approval Watcher: Polls the database every minute... |
| Route by Review Status | n8n-nodes-base.switch | Routes pages based on Approval or Changes Requested status | Watch Notion Review Queue | Mark Ready to Publish, Mark In Revision | Approval Watcher: Polls the database every minute... |
| Mark Ready to Publish | n8n-nodes-base.notion | Updates Notion page status to Ready to Publish and logs date | Route by Review Status | Tell Submitter It's Approved | Publish or Revise: Approved pages move to Ready to Publish... |
| Tell Submitter It's Approved | n8n-nodes-base.gmail | Notifies submitter of approval status | Mark Ready to Publish | None | Publish or Revise: Approved pages move to Ready to Publish... |
| Mark In Revision | n8n-nodes-base.notion | Updates Notion page status to In Revision | Route by Review Status | Send Revision Notes | Publish or Revise: Approved pages move to Ready to Publish... |
| Send Revision Notes | n8n-nodes-base.gmail | Emails review feedback notes to submitter | Mark In Revision | None | Publish or Revise: Approved pages move to Ready to Publish... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Form Trigger:**
   - Add an **Episode Submission Form** node.
   - Configure form fields: `Episode Title` (Required), `Guest Name`, `Episode URL`, `Target Keywords`, `Your Email` (Email type, Required), `Transcript File` (File type, accepting `.srt,.vtt,.txt`), `Audio File` (File type, accepting `.mp3,.m4a,.wav,.mp4`), and `Or Paste Transcript` (Textarea).

2. **Add Input Preparation Logic:**
   - Add a **Code** node named `Prepare Episode Input`.
   - Insert JavaScript to inspect binary files and text inputs, ensuring either a transcript or audio is supplied, and output unified JSON properties (`episodeTitle`, `guestName`, `episodeUrl`, `keywords`, `submitterEmail`, `hasTranscript`, `transcriptRaw`).
   - Connect **Episode Submission Form** to **Prepare Episode Input**.

3. **Configure Branching for Transcription:**
   - Add an **If** node named `Transcript Provided?`. Set condition to evaluate `{{ $json.hasTranscript }}` as true.
   - Connect **Prepare Episode Input** to **Transcript Provided?**.

4. **Build the Text Parser Branch:**
   - Add a **Code** node named `Parse Transcript Timestamps`. Insert script to parse SRT/VTT structures into `[mm:ss]` lines.
   - Connect the **True** output of `Transcript Provided?` to `Parse Transcript Timestamps`.

5. **Build the Audio Transcription Branch:**
   - Add an **HTTP Request** node named `Transcribe Audio with Whisper`.
   - Configure: POST method to `https://api.openai.com/v1/audio/transcriptions`, Authentication set to pre-defined OpenAI credential, Content-Type `multipart-form-data`, Body parameters: `file` (from binary input `audio`), `model` (`whisper-1`), and `response_format` (`verbose_json`). Set timeout to `600000`.
   - Add a **Code** node named `Format Whisper Segments` to map verbose JSON segments to `[mm:ss]` lines.
   - Connect the **False** output of `Transcript Provided?` to `Transcribe Audio with Whisper`, and connect its output to `Format Whisper Segments`.

6. **Set Up Global Show Settings:**
   - Add a **Set** node named `Set Show Settings`.
   - Define string assignments for `showName`, `hostName`, `audience`, `brandVoice`, and `reviewerEmail`. Map `transcript`, `episodeTitle`, `guestName`, `episodeUrl`, `keywords`, and `submitterEmail` using expressions referencing prior nodes.
   - Connect both `Parse Transcript Timestamps` and `Format Whisper Segments` outputs to `Set Show Settings`.

7. **Configure AI Generation Nodes (OpenAI):**
   - Add three **OpenAI** nodes sequentially:
     - **Generate Clips & Quote Briefs**: Model `gpt-4o-mini`, temperature `0.6`, JSON output enabled. Use system prompt instructing extraction of 5 short clips and 5 quote briefs. Connect input from `Set Show Settings`.
     - **Write Newsletter & Show Notes**: Model `gpt-4o-mini`, temperature `0.6`, JSON output enabled. Use system prompt for newsletter and SEO show notes. Connect input from **Generate Clips & Quote Briefs**.
     - **Write Social Assets**: Model `gpt-4o-mini`, temperature `0.6`, JSON output enabled. Use system prompt for carousel, LinkedIn post, and X thread. Connect input from **Write Newsletter & Show Notes**.

8. **Package Assets for Notion:**
   - Add a **Code** node named `Package 15 Assets`. Insert script to build Notion blocks (callouts, headings, rich text, dividers) and chunk them into arrays of 90 blocks per batch.
   - Connect **Write Social Assets** to **Package 15 Assets**.

9. **Create and Populate the Notion Review Page:**
   - Add a **Notion** node named `Create Notion Review Page`. Configure resource as `Database Page`, Database ID (`YOUR_NOTION_DATABASE_ID`), and map properties: `Status` (select: `Needs Review`), `Guest` (rich text), `Asset Count` (number), and `Submitted By` (email).
   - Add a **Code** node named `Split Block Batches` to map batched children blocks against the new page ID.
   - Add an **HTTP Request** node named `Append Assets to Notion Page`. Configure PATCH method to `https://api.notion.com/v1/blocks/{{ $json.pageId }}/children`, JSON body stringifying children, and batching enabled (`batchSize: 1`, `batchInterval: 400ms`). Set Notion API credentials and version header `2022-06-28`.
   - Connect **Package 15 Assets** $\rightarrow$ **Create Notion Review Page** $\rightarrow$ **Split Block Batches** $\rightarrow$ **Append Assets to Notion Page**.

10. **Email the Reviewer:**
    - Add a **Gmail** node named `Email Reviewer`. Configure recipient as `{{ $('Set Show Settings').first().json.reviewerEmail }}`, subject, and HTML body linking to the Notion page. Set execution to once.
    - Connect **Append Assets to Notion Page** to **Email Reviewer**.

11. **Build the Approval Watcher & Lifecycle Routing Flow:**
    - Add a **Notion Trigger** node named `Watch Notion Review Queue`. Event: `pagedUpdatedInDatabase`, Poll interval: every minute, Database ID: `YOUR_NOTION_DATABASE_ID`.
    - Add a **Switch** node named `Route by Review Status`. Rules: `Approved` (when status equals "Approved") and `Changes Requested` (when status equals "Changes Requested"). Connect trigger to switch.
    - **Approved Branch:**
      - Add a **Notion** node named `Mark Ready to Publish`. Operation: update page, set `Status` to `Ready to Publish` and `Approved On` to `{{ $now.toISO() }}`.
      - Add a **Gmail** node named `Tell Submitter It's Approved` targeting `{{ $('Watch Notion Review Queue').item.json.property_submitted_by }}`.
      - Connect: **Route by Review Status** (Approved) $\rightarrow$ **Mark Ready to Publish** $\rightarrow$ **Tell Submitter It's Approved**.
    - **Changes Requested Branch:**
      - Add a **Notion** node named `Mark In Revision`. Operation: update page, set `Status` to `In Revision`.
      - Add a **Gmail** node named `Send Revision Notes` targeting the submitter email with review notes from `{{ $('Watch Notion Review Queue').item.json.property_review_notes }}`.
      - Connect: **Route by Review Status** (Changes Requested) $\rightarrow$ **Mark In Revision** $\rightarrow$ **Send Revision Notes**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a Notion database with properties matching the workflow (`Name`, `Status` (select), `Guest` (text), `Asset Count` (number), `Submitted By` (email), `Review Notes` (text), `Approved On` (date)) and share it with your Notion integration. | Notion Database Schema Setup |
| Audio files submitted for transcription via OpenAI Whisper must be under 25 MB. | OpenAI Whisper API Constraints |