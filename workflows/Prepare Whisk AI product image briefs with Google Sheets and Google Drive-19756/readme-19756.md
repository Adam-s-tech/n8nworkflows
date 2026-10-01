Prepare Whisk AI product image briefs with Google Sheets and Google Drive

https://n8nworkflows.xyz/workflows/prepare-whisk-ai-product-image-briefs-with-google-sheets-and-google-drive-19756


# Prepare Whisk AI product image briefs with Google Sheets and Google Drive

### 1. Workflow Overview

This workflow automates the preparation of structured image generation briefs for [Whisk AI](https://whisk-ai.io/), tracks the associated tasks in Google Sheets, and provides an authenticated return channel to collect, validate, and archive finished images in Google Drive. Because Whisk AI does not offer an automated API, the generation process remains a human-in-the-loop task, while n8n manages data integrity, prompt standardization, status tracking, and cloud storage.

The logical execution divides into two independent operational pathways (forms):
- **Block 1: Brief Generation & Workspace Setup** – Accepts product details via an authenticated form, validates inputs against structural constraints, assembles a standardized AI prompt alongside a unique task ID, logs the record to Google Sheets, and presents the user with manual instructions and a link to the return portal.
- **Block 2: Asset Validation, Matching & Archiving** – Accepts completed image uploads via a second authenticated form, validates metadata and binary size constraints, looks up the corresponding task in Google Sheets, verifies Google Drive for existing archive entries to support safe sequential retries, stores the asset, and updates the tracking spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Brief Generation & Workspace Setup

- **Overview:** This block handles inbound product parameters, initializes configuration constants, validates data types and URL structures, builds a standardized prompt based on scene presets, logs the task to Google Sheets, and serves a completion form with instructions for manual generation in Whisk AI.
- **Nodes Involved:**
  - `Create image brief`
  - `Configure brief workspace`
  - `Validate brief inputs`
  - `Build reusable image brief`
  - `Record brief in Google Sheets`
  - `Show prompt and manual creation steps`
  - `Explain invalid brief`

- **Node Details:**

  - **Create image brief**
    - *Type and technical role:* `n8n-nodes-base.formTrigger` — Acts as the entry point for users submitting new product image briefs.
    - *Configuration choices:* Secured via HTTP Basic Auth. Exposes form fields: `product_name` (text, required), `source_image_url` (text, required), `scene` (dropdown with options: "Daylight desk", "Studio pedestal", "Cozy home", required), `aspect_ratio` (dropdown with options: "1:1", "4:5", "9:16", required), and `extra_instructions` (textarea, optional). Response mode is set to `lastNode`.
    - *Key expressions or variables:* None (trigger node).
    - *Input and output connections:* Input: None; Output: `Configure brief workspace`.
    - *Version-specific requirements:* Form Trigger version 2.4.
    - *Edge cases or potential failure types:* Unauthorized access if HTTP Basic Auth fails; invalid field entries handled downstream.

  - **Configure brief workspace**
    - *Type and technical role:* `n8n-nodes-base.set` — Injects configuration variables required for downstream Google Sheets operations and redirect links.
    - *Configuration choices:* Sets `spreadsheet_id` to `REPLACE_WITH_GOOGLE_SHEET_ID`, `sheet_name` to `Briefs`, and `return_form_url` to the production URL of the return form.
    - *Key expressions or variables:* Static string assignments.
    - *Input and output connections:* Input: `Create image brief`; Output: `Validate brief inputs`.
    - *Version-specific requirements:* Set node version 3.4.
    - *Edge cases or potential failure types:* Failure to replace placeholder IDs will cause downstream Google Sheets API errors.

  - **Validate brief inputs**
    - *Type and strict role:* `n8n-nodes-base.if` — Evaluates payload validity against formatting and length rules.
    - *Configuration choices:* Validates that `product_name` is 1–120 characters, `source_image_url` is a valid HTTPS URL up to 2048 characters, `scene` and `aspect_ratio` match allowed whitelist options, and `extra_instructions` do not exceed 1200 characters.
    - *Key expressions or variables:* 
      ```javascript
      ={{ $json.product_name.trim().length > 0 && $json.product_name.length <= 120 && /^https:\/\/[^\s]+$/i.test($json.source_image_url) && $json.source_image_url.length <= 2048 && ['Daylight desk','Studio pedestal','Cozy home'].includes($json.scene) && ['1:1','4:5','9:16'].includes($json.aspect_ratio) && ($json.extra_instructions || '').length <= 1200 }}
      ```
    - *Input and output connections:* Input: `Configure brief workspace`; Output (True): `Build reusable image brief`; Output (False): `Explain invalid brief`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* Strict type validation failures route traffic to the error explanation page.

  - **Build reusable image brief**
    - *Type and technical role:* `n8n-nodes-base.set` — Compiles form inputs into a standardized object containing a unique task ID, tailored prompt text, initial status, and timestamps.
    - *Configuration choices:* Generates `brief_id`, formats product name and URL, maps scene descriptions to descriptive prompt segments, appends aspect ratio specs, sets initial `status` to `awaiting_creation`, and establishes ISO timestamps for `created_at` and `updated_at`.
    - *Key expressions or variables:* 
      - `brief_id`: `={{ 'WHISK-' + $workflow.id + '-' + $execution.id }}`
      - `prompt`: Generates context-aware instructions combining the product name, scene template, aspect ratio, and safety constraints.
      - `created_at` / `updated_at`: `={{ $now.toISO() }}`
    - *Input and output connections:* Input: `Validate brief inputs` (True branch); Output: `Record brief in Google Sheets`.
    - *Version-specific requirements:* Set node version 3.4.
    - *Edge cases or potential failure types:* Expression evaluation faults if upstream properties are missing.

  - **Record brief in Google Sheets**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Appends the generated brief row to the configured tracking spreadsheet.
    - *Configuration choices:* Resource: `sheet`, Operation: `append`. Uses OAuth2 authentication. Cell formatting set to `RAW`.
    - *Key expressions or variables:* Spreadsheet ID and Sheet name reference `$('Configure brief workspace')`. Column values map directly to upstream payload fields (`brief_id`, `product_name`, `source_image_url`, `scene`, `aspect_ratio`, `prompt`, `status`, `created_at`, `drive_file_id`, `drive_file_url`, `review_notes`, `updated_at`).
    - *Input and output connections:* Input: `Build reusable image brief`; Output: `Show prompt and manual creation steps`.
    - *Version-specific requirements:* Google Sheets node version 4.7.
    - *Edge cases or potential failure types:* Authentication expiry, incorrect Sheet ID, or header mismatch.

  - **Show prompt and manual creation steps**
    - *Type and technical role:* `n8n-nodes-base.form` — Displays a completion web page to the user containing the generated task ID, scene details, formatted prompt, step-by-step instructions for Whisk AI, and a link to the return form.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Key expressions or variables:* Sanitizes and interpolates `brief_id`, `scene`, `aspect_ratio`, `prompt`, and `return_form_url` using HTML-safe string transformations.
    - *Input and output connections:* Input: `Record brief in Google Sheets`; Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* XSS sanitization failures if upstream text includes unescaped markup.

  - **Explain invalid brief**
    - *Type and technical role:* `n8n-nodes-base.form` — Renders an error page instructing the user to correct input parameters if validation fails.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Key expressions or variables:* Static HTML error text.
    - *Input and output connections:* Input: `Validate brief inputs` (False branch); Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

---

#### 2.2 Asset Validation, Matching & Archiving

- **Overview:** This block captures finished image submissions, validates task ID formats and file size/MIME types, queries Google Sheets and Google Drive to locate existing task records or archived files, handles duplicate safety checks, and manages file uploads or sequential retries before updating the spreadsheet and showing a completion receipt.
- **Nodes Involved:**
  - `Return completed image`
  - `Configure archive workspace`
  - `Validate task ID and image`
  - `Find original brief`
  - `Require one matching brief`
  - `Find an already archived image`
  - `Require an unambiguous archive`
  - `Restore uploaded binary`
  - `Already archived`
  - `Use existing file`
  - `Allow first asset upload`
  - `Upload completed image`
  - `Use uploaded file`
  - `Update original brief with asset`
  - `Show archived asset`
  - `Explain invalid asset`
  - `Explain unmatched brief`
  - `Explain duplicate archive`
  - `Explain missing previous asset`

- **Node Details:**

  - **Return completed image**
    - *Type and technical role:* `n8n-nodes-base.formTrigger` — Entry point for returning generated images.
    - *Configuration choices:* Secured via HTTP Basic Auth. Collects `brief_id` (text, required), `asset` (file upload supporting PNG, JPEG, WEBP, required, single file), and `review_notes` (textarea, optional). Response mode: `lastNode`.
    - *Key expressions or variables:* None (trigger node).
    - *Input and output connections:* Input: None; Output: `Configure archive workspace`.
    - *Version-specific requirements:* Form Trigger version 2.4.
    - *Edge cases or potential failure types:* Authentication failures or missing file attachments.

  - **Configure archive workspace**
    - *Type and technical role:* `n8n-nodes-base.set` — Injects configuration parameters for archiving operations.
    - *Configuration choices:* Sets `spreadsheet_id`, `sheet_name`, `drive_folder_id`, and `max_upload_bytes` (configured to `10485760` bytes / 10 MB).
    - *Key expressions or variables:* Static assignments.
    - *Input and output connections:* Input: `Return completed image`; Output: `Validate task ID and image`.
    - *Version-specific requirements:* Set node version 3.4.
    - *Edge cases or potential failure types:* Unconfigured placeholder folder IDs will cause Google Drive API failures.

  - **Validate task ID and image**
    - *Type and technical role:* `n8n-nodes-base.if` — Validates format compliance for task IDs, binary asset presence, MIME types, file sizes, and review note lengths.
    - *Configuration choices:* Evaluates regex matching for `WHISK-[A-Za-z0-9_-]+-[0-9]+`, checks binary MIME type (`image/png`, `image/jpeg`, `image/webp`), enforces size limits (`0 < size <= max_upload_bytes`), and checks optional review note character limits.
    - *Key expressions or variables:* 
      ```javascript
      ={{ /^WHISK-[A-Za-z0-9_-]+-[0-9]+$/.test(($json.brief_id || '').trim()) && !!$binary.asset && ['image/png','image/jpeg','image/webp'].includes($binary.asset.mimeType) && $json.asset.size > 0 && $json.asset.size <= $json.max_upload_bytes && ($json.review_notes || '').length <= 2000 }}
      ```
    - *Input and output connections:* Input: `Configure archive workspace`; Output 1 (True): `Find original brief` & `Restore uploaded binary`; Output 2 (False): `Explain invalid asset`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* Unsupported file extensions or oversized images route to the error handler.

  - **Find original brief**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Searches the tracking sheet for a row matching the submitted `brief_id`.
    - *Configuration choices:* Resource: `sheet`, Operation: `read`. Uses filters UI matching `brief_id`. `alwaysOutputData` enabled.
    - *Key expressions or variables:* Lookup value references `$('Return completed image').first().json.brief_id.trim()`.
    - *Input and output connections:* Input: `Validate task ID and image` (True branch); Output: `Require one matching brief`.
    - *Version-specific requirements:* Google Sheets node version 4.7.
    - *Edge cases or potential failure types:* Missing tasks result in empty arrays, caught by downstream logic.

  - **Require one matching brief**
    - *Type and technical role:* `n8n-nodes-base.if` — Ensures exactly one unique valid brief exists for the task ID and that its status is valid.
    - *Configuration choices:* Evaluates whether `Find original brief` returns exactly one row matching allowed lifecycle statuses (`awaiting_creation`, `needs_review`, `approved`, `needs_revision`).
    - *Key expressions or variables:* 
      ```javascript
      ={{ $('Find original brief').all().length === 1 && $json.brief_id === $('Return completed image').first().json.brief_id.trim() && ['awaiting_creation','needs_review','approved','needs_revision'].includes($json.status) }}
      ```
    - *Input and output connections:* Input: `Find original brief`; Output (True): `Find an already archived image`; Output (False): `Explain unmatched brief`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* Zero matches or duplicate task records trigger error routing.

  - **Find an already archived image**
    - *Type and technical role:* `n8n-nodes-base.googleDrive` — Searches the designated Google Drive folder for existing files matching the task ID prefix.
    - *Configuration choices:* Resource: `fileFolder`, Operation: `search`. Limits results to 2. Query filters by parent folder ID, active status (`trashed = false`), and name prefix containing the `brief_id`. `alwaysOutputData` enabled.
    - *Key expressions or variables:* Query constructed using `drive_folder_id` and `brief_id`.
    - *Input and output connections:* Input: `Require one matching brief`; Output: `Require an unambiguous archive`.
    - *Version-specific requirements:* Google Drive node version 3.
    - *Edge cases or potential failure types:* Network timeouts or permission errors accessing Google Drive.

  - **Require an unambiguous archive**
    - *Type and technical role:* `n8n-nodes-base.if` — Verifies that search results for existing archives do not contain duplicate conflicting files (expects 0 or 1 file).
    - *Configuration choices:* Evaluates whether `Find an already archived image` returns exactly one or zero results.
    - *Key expressions or variables:* `={{ $('Find an already archived image').all().length === 1 }}`
    - *Input and output connections:* Input: `Find an already archived image`; Output (True): `Restore uploaded binary` (Input index 1); Output (False): `Explain duplicate archive`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* Multiple files matching the task ID halt execution to prevent overwriting ambiguity.

  - **Restore uploaded binary**
    - *Type and technical role:* `n8n-nodes-base.merge` — Combines binary stream data from the initial form submission back into the main workflow flow after asynchronous API lookups.
    - *Configuration choices:* Mode: `combine`, Clash handling: `preferInput2`, Combine by: `combineByPosition`. Two inputs connected.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input 0: `Validate task ID and image`; Input 1: `Require an unambiguous archive`; Output: `Already archived`.
    - *Version-specific requirements:* Merge node version 3.2.
    - *Edge cases or potential failure types:* Binary payload detachment if stream handling drops data across asynchronous Google API calls.

  - **Already archived**
    - *Type and technical role:* `n8n-nodes-base.if` — Branches logic based on whether an archived file was already found in Google Drive for this task.
    - *Configuration choices:* Evaluates `!!$json.id`.
    - *Key expressions or variables:* `={{ !!$json.id }}`
    - *Input and output connections:* Input: `Restore uploaded binary`; Output (True): `Use existing file`; Output (False): `Allow first asset upload`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* None.

  - **Use existing file**
    - *Type and technical role:* `n8n-nodes-base.set` — Formats file metadata to reuse the existing Google Drive file on a retry.
    - *Configuration choices:* Assigns `file_id`, constructs `file_url`, and sets `reused` to `yes`.
    - *Key expressions or variables:* `file_url` assembled using `https://drive.google.com/file/d/` + ID + `/view`.
    - *Input and output connections:* Input: `Already archived` (True branch); Output: `Update original brief with asset`.
    - *Version-specific requirements:* Set node version 3.4.
    - *Edge cases or potential failure types:* None.

  - **Allow first asset upload**
    - *Type and technical role:* `n8n-nodes-base.if` — Verifies that the task status is awaiting creation and lacks an existing Drive file ID before uploading.
    - *Configuration choices:* Evaluates brief status and absence of `drive_file_id`.
    - *Key expressions or variables:* `={{ $('Find original brief').first().json.status === 'awaiting_creation' && !$('Find original brief').first().json.drive_file_id }}`
    - *Input and output connections:* Input: `Already archived` (False branch); Output (True): `Upload completed image`; Output (False): `Explain missing previous asset`.
    - *Version-specific requirements:* IF node version 2.2.
    - *Edge cases or potential failure types:* State mismatch if a file exists without a corresponding Drive ID.

  - **Upload completed image**
    - *Type and technical role:* `n8n-nodes-base.googleDrive` — Uploads the binary asset to Google Drive.
    - *Configuration choices:* Resource: `file`, Operation: `upload`. Target folder references `drive_folder_id`. Sets custom app property `whiskBriefId`. Filename constructed from brief ID, sanitized product name, and file extension.
    - *Key expressions or variables:* Dynamic filename generation based on brief ID and MIME type mapping.
    - *Input and output connections:* Input: `Allow first asset upload` (True branch); Output: `Use uploaded file`.
    - *Version-specific requirements:* Google Drive node version 3.
    - *Edge cases or potential failure types:* Storage quota limits or network timeouts during upload.

  - **Use uploaded file**
    - *Type and technical role:* `n8n-nodes-base.set` — Formats file metadata for a newly uploaded Google Drive asset.
    - *Configuration choices:* Assigns `file_id`, constructs `file_url`, and sets `reused` to `no`.
    - *Key expressions or variables:* Standard file URL construction.
    - *Input and output connections:* Input: `Upload completed image`; Output: `Update original brief with asset`.
    - *Version-specific requirements:* Set node version 3.4.
    - *Edge cases or potential failure types:* None.

  - **Update original brief with asset**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Updates the tracking sheet row with the Drive file reference, updated timestamp, review status, and creation notes.
    - *Configuration choices:* Resource: `sheet`, Operation: `update`. Matching column: `brief_id`. Cell formatting: `RAW`.
    - *Key expressions or variables:* Evaluates reuse state to preserve existing status or advance status to `needs_review`. Updates `review_notes` and `updated_at`.
    - *Input and output connections:* Input: `Use existing file` or `Use uploaded file`; Output: `Show archived asset`.
    - *Version-specific requirements:* Google Sheets node version 4.7.
    - *Edge cases or potential failure types:* Row locking conflicts or API write failures.

  - **Show archived asset**
    - *Type and technical role:* `n8n-nodes-base.form` — Displays a confirmation completion page with a link to the archived Google Drive image and review status.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Key expressions or variables:* Sanitizes and interpolates `brief_id`, `drive_file_url`, and `status`.
    - *Input and output connections:* Input: `Update original brief with asset`; Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

  - **Explain invalid asset**
    - *Type and technical role:* `n8n-nodes-base.form` — Displays an error page when uploaded asset validation fails.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Input and output connections:* Input: `Validate task ID and image` (False branch); Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

  - **Explain unmatched brief**
    - *Type and technical role:* `n8n-nodes-base.form` — Renders an error page when a submitted task ID cannot be matched to a unique active brief.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Input and output connections:* Input: `Require one matching brief` (False branch); Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

  - **Explain duplicate archive**
    - *Type and technical role:* `n8n-nodes-base.form` — Displays an error page when multiple archived files are found in Drive for a single task ID.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Input and output connections:* Input: `Require an unambiguous archive` (False branch); Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

  - **Explain missing previous asset**
    - *Type and technical role:* `n8n-nodes-base.form` — Renders an error page when a task state mismatch occurs during upload checks.
    - *Configuration choices:* Operation: `completion`, Respond with: `showText`.
    - *Input and output connections:* Input: `Allow first asset upload` (False branch); Output: None (Terminal node).
    - *Version-specific requirements:* Form node version 2.4.
    - *Edge cases or potential failure types:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Create image brief | `n8n-nodes-base.formTrigger` | Inbound product brief submission form | None | Configure brief workspace | # Prepare Whisk AI product image briefs and archive results with Google Sheets and Google Drive<br><br>## Who is this for?<br>E-commerce sellers and content teams who create product images manually in [Whisk AI](https://whisk-ai.io/) and need repeatable briefs, asset tracking, and review handoffs.<br><br>## How it works<br>The first form collects product details, a reference-image link, and a scene preset. It prepares a prompt and records a task in Google Sheets. Open Whisk, upload your reference, and generate the image manually. Return the downloaded file through the second form with its task ID. The workflow validates the task, archives the image in Google Drive, and updates the original row. Sequential retries reuse an existing archive.<br><br>## Setup<br>1. Create a tracking sheet with the column names in the setup note.<br>2. Connect Google Sheets and Google Drive credentials.<br>3. Configure both workspace nodes and the return-form URL.<br>4. Set Basic Auth credentials on both forms and test both branches.<br><br>## Requirements<br>n8n, Google service access, and browser access to Whisk. No Whisk API key is required. Image creation is manual; Whisk generation costs apply separately.<br><br>## Customize and review<br>Edit the scene prompts and upload limit. Check product shape, labels, and logos before approval. Use this with a trusted team; process returns sequentially. Create a new brief for each revision.<br><br>## One-time setup<br>1. Create a Google Sheet and name its tab **Briefs**. Paste these exact column names into row 1:<br><br>`brief_id \| product_name \| source_image_url \| scene \| aspect_ratio \| prompt \| status \| created_at \| drive_file_id \| drive_file_url \| review_notes \| updated_at`<br><br>2. Set the same sheet ID/tab in both **Configure ... workspace** nodes. Set the archive folder ID in the archive configuration.<br>3. Connect Google Sheets credentials to all three Sheets nodes and Google Drive credentials to both Drive nodes.<br>4. Set **HTTP Basic Auth credentials** on BOTH form triggers. Use this for a trusted internal team. Task IDs identify work, not people.<br>5. Copy the return form Production URL into **Configure brief workspace → return_form_url**. Publish/activate only after completing setup.<br><br>Reference links are retained for the operator; n8n does not fetch them or send them to Whisk. |
| Configure brief workspace | `n8n-nodes-base.set` | Sets Google Sheet ID, sheet name, and return form URL | Create image brief | Validate brief inputs | # Prepare Whisk AI product image briefs and archive results with Google Sheets and Google Drive<br><br>## Who is this for?<br>E-commerce sellers and content teams who create product images manually in [Whisk AI](https://whisk-ai.io/) and need repeatable briefs, asset tracking, and review handoffs.<br><br>## How it works<br>The first form collects product details, a reference-image link, and a scene preset. It prepares a prompt and records a task in Google Sheets. Open Whisk, upload your reference, and generate the image manually. Return the downloaded file through the second form with its task ID. The workflow validates the task, archives the image in Google Drive, and updates the original row. Sequential retries reuse an existing archive.<br><br>## Setup<br>1. Create a tracking sheet with the column names in the setup note.<br>2. Connect Google Sheets and Google Drive credentials.<br>3. Configure both workspace nodes and the return-form URL.<br>4. Set Basic Auth credentials on both forms and test both branches.<br><br>## Requirements<br>n8n, Google service access, and browser access to Whisk. No Whisk API key is required. Image creation is manual; Whisk generation costs apply separately.<br><br>## Customize and review<br>Edit the scene prompts and upload limit. Check product shape, labels, and logos before approval. Use this with a trusted team; process returns sequentially. Create a new brief for each revision.<br><br>## One-time setup<br>1. Create a Google Sheet and name its tab **Briefs**. Paste these exact column names into row 1:<br><br>`brief_id \| product_name \| source_image_url \| scene \| aspect_ratio \| prompt \| status \| created_at \| drive_file_id \| drive_file_url \| review_notes \| updated_at`<br><br>2. Set the same sheet ID/tab in both **Configure ... workspace** nodes. Set the archive folder ID in the archive configuration.<br>3. Connect Google Sheets credentials to all three Sheets nodes and Google Drive credentials to both Drive nodes.<br>4. Set **HTTP Basic Auth credentials** on BOTH form triggers. Use this for a trusted internal team. Task IDs identify work, not people.<br>5. Copy the return form Production URL into **Configure brief workspace → return_form_url**. Publish/activate only after completing setup.<br><br>Reference links are retained for the operator; n8n does not fetch them or send them to Whisk. |
| Validate brief inputs | `n8n-nodes-base.if` | Validates product name length, URL format, and enum selections | Configure brief workspace | Build reusable image brief, Explain invalid brief | ## 1 · Prepare and record the brief<br>Submit one product and one scene. The workflow assembles a reusable prompt and records a unique task. Google Sheets writes use RAW values. The final page links to Whisk and your configured return form.<br><br>No text-model account or Whisk API key is needed. |
| Build reusable image brief | `n8n-nodes-base.set` | Generates prompt, task ID, and initial metadata | Validate brief inputs | Record brief in Google Sheets | ## 1 · Prepare and record the brief<br>Submit one product and one scene. The workflow assembles a reusable prompt and records a unique task. Google Sheets writes use RAW values. The final page links to Whisk and your configured return form.<br><br>No text-model account or Whisk API key is needed. |
| Record brief in Google Sheets | `n8n-nodes-base.googleSheets` | Appends brief record to Google Sheets | Build reusable image brief | Show prompt and manual creation steps | ## 1 · Prepare and record the brief<br>Submit one product and one scene. The workflow assembles a reusable prompt and records a unique task. Google Sheets writes use RAW values. The final page links to Whisk and your configured return form.<br><br>No text-model account or Whisk API key is needed. |
| Show prompt and manual creation steps | `n8n-nodes-base.form` | Displays prompt, instructions, and return form link | Record brief in Google Sheets | None | ## 2 · Create the image manually<br>Open **[Whisk AI](https://whisk-ai.io/)** in your browser. Upload your product reference, paste the prompt, choose the ratio and generate. Download the result and submit it with the task ID in the second form.<br><br>This template does not automate Whisk or Google Labs. Whisk generation charges depend on your account and selections. |
| Explain invalid brief | `n8n-nodes-base.form` | Renders validation error page for brief inputs | Validate brief inputs | None | ## 1 · Prepare and record the brief<br>Submit one product and one scene. The workflow assembles a reusable prompt and records a unique task. Google Sheets writes use RAW values. The final page links to Whisk and your configured return form.<br><br>No text-model account or Whisk API key is needed. |
| Return completed image | `n8n-nodes-base.formTrigger` | Inbound asset return form | None | Configure archive workspace | ## One-time setup<br>1. Create a Google Sheet and name its tab **Briefs**. Paste these exact column names into row 1:<br><br>`brief_id \| product_name \| source_image_url \| scene \| aspect_ratio \| prompt \| status \| created_at \| drive_file_id \| drive_file_url \| review_notes \| updated_at`<br><br>2. Set the same sheet ID/tab in both **Configure ... workspace** nodes. Set the archive folder ID in the archive configuration.<br>3. Connect Google Sheets credentials to all three Sheets nodes and Google Drive credentials to both Drive nodes.<br>4. Set **HTTP Basic Auth credentials** on BOTH form triggers. Use this for a trusted internal team. Task IDs identify work, not people.<br>5. Copy the return form Production URL into **Configure brief workspace → return_form_url**. Publish/activate only after completing setup.<br><br>Reference links are retained for the operator; n8n does not fetch them or send them to Whisk. |
| Configure archive workspace | `n8n-nodes-base.set` | Sets Sheet ID, Drive folder ID, and upload limits | Return completed image | Validate task ID and image | ## One-time setup<br>1. Create a Google Sheet and name its tab **Briefs**. Paste these exact column names into row 1:<br><br>`brief_id \| product_name \| source_image_url \| scene \| aspect_ratio \| prompt \| status \| created_at \| drive_file_id \| drive_file_url \| review_notes \| updated_at`<br><br>2. Set the same sheet ID/tab in both **Configure ... workspace** nodes. Set the archive folder ID in the archive configuration.<br>3. Connect Google Sheets credentials to all three Sheets nodes and Google Drive credentials to both Drive nodes.<br>4. Set **HTTP Basic Auth credentials** on BOTH form triggers. Use this for a trusted internal team. Task IDs identify work, not people.<br>5. Copy the return form Production URL into **Configure brief workspace → return_form_url**. Publish/activate only after completing setup.<br><br>Reference links are retained for the operator; n8n does not fetch them or send them to Whisk. |
| Validate task ID and image | `n8n-nodes-base.if` | Validates task ID format, binary type, and size | Configure archive workspace | Find original brief, Restore uploaded binary, Explain invalid asset | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Find original brief | `n8n-nodes-base.googleSheets` | Looks up task record in Google Sheets | Validate task ID and image | Require one matching brief | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Require one matching brief | `n8n-nodes-base.if` | Ensures exactly one matching brief exists | Find original brief | Find an already archived image, Explain unmatched brief | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Find an already archived image | `n8n-nodes-base.googleDrive` | Searches Drive folder for existing task files | Require one matching brief | Require an unambiguous archive | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Require an unambiguous archive | `n8n-nodes-base.if` | Confirms zero or one existing archive match | Find an already archived image | Restore uploaded binary, Explain duplicate archive | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Restore uploaded binary | `n8n-nodes-base.merge` | Restores binary payload after async API checks | Validate task ID and image, Require an unambiguous archive | Already archived | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Already archived | `n8n-nodes-base.if` | Checks if a file was already archived for this task | Restore uploaded binary | Use existing file, Allow first asset upload | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Use existing file | `n8n-nodes-base.set` | Formats metadata to reuse existing Drive file | Already archived | Update original brief with asset | ## Retry and failure handling<br>- If Drive upload succeeds but updating Sheets fails, a **sequential** resubmission finds the task-prefixed filename and repairs the sheet link.<br>- Keep archived filenames unchanged. The workflow checks for duplicate matches and stops for review.<br>- Existing approval status and reviewer notes are preserved.<br>- Concurrent submissions for the same task are not transactionally locked; submit one return at a time.<br>- Google authorization/network failures are visible in n8n executions; retry after fixing the cause. No social or store publication is performed.<br>- File checks validate type metadata and size, not image authenticity. |
| Allow first asset upload | `n8n-nodes-base.if` | Checks task status and upload eligibility | Already archived | Upload completed image, Explain missing previous asset | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Upload completed image | `n8n-nodes-base.googleDrive` | Uploads completed image to Google Drive folder | Allow first asset upload | Use uploaded file | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Use uploaded file | `n8n-nodes-base.set` | Formats metadata for newly uploaded Drive file | Upload completed image | Update original brief with asset | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Update original brief with asset | `n8n-nodes-base.googleSheets` | Updates tracking sheet row with Drive file ID/URL | Use existing file, Use uploaded file | Show archived asset | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Show archived asset | `n8n-nodes-base.form` | Displays completion page with Drive link and status | Update original brief with asset | None | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Explain invalid asset | `n8n-nodes-base.form` | Renders validation error page for asset uploads | Validate task ID and image | None | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Explain unmatched brief | `n8n-nodes-base.form` | Renders error page for unmatched task IDs | Require one matching brief | None | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Explain duplicate archive | `n8n-nodes-base.form` | Renders error page for ambiguous Drive archive matches | Require an unambiguous archive | None | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |
| Explain missing previous asset | `n8n-nodes-base.form` | Renders error page for state mismatch on upload | Allow first asset upload | None | ## 3 · Match, archive and review<br>PNG/JPEG/WebP, at most 10 MB by default. A Merge node restores the original uploaded binary after Sheets/Drive lookups. Unknown tasks and ambiguous archive matches stop before upload. Files stay in your selected folder with its existing permissions.<br><br>Review in the sheet: `awaiting_creation → needs_review → approved`. For a new revision, create a new brief. |

---

### 4. Reproducing theWorkflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps in sequence:

1. **Create the Tracking Google Sheet:**
   - Create a Google Sheet and name its tab `Briefs`.
   - In row 1, add the exact column headers: `brief_id`, `product_name`, `source_image_url`, `scene`, `aspect_ratio`, `prompt`, `status`, `created_at`, `drive_file_id`, `drive_file_url`, `review_notes`, `updated_at`.

2. **Build the Brief Creation Branch:**
   - **Step 1:** Create a **Form Trigger** node named `Create image brief`. Configure HTTP Basic Auth credentials. Add form fields: `product_name` (text, required), `source_image_url` (text, required), `scene` (dropdown with options: "Daylight desk", "Studio pedestal", "Cozy home", default "Daylight desk"), `aspect_ratio` (dropdown with options: "1:1", "4:5", "9:16", default "1:1"), and `extra_instructions` (textarea, optional). Set response mode to `lastNode`.
   - **Step 2:** Add a **Set** node named `Configure brief workspace`. Assign `spreadsheet_id` (string), `sheet_name` (string, value `Briefs`), and `return_form_url` (string, set to return form URL). Connect `Create image brief` to this node.
   - **Step 3:** Add an **If** node named `Validate brief inputs`. Configure conditions to validate string lengths, HTTPS URL regex, and allowed enum values. Connect `Configure brief workspace` to it.
   - **Step 4:** Add a **Set** node named `Build reusable image brief`. Connect the `True` output of `Validate brief inputs`. Assign `brief_id` (`WHISK-$workflow.id-$execution.id`), `product_name`, `source_image_url`, `scene`, `aspect_ratio`, `prompt` (using the conditional scene template expression), `status` (`awaiting_creation`), timestamps (`$now.toISO()`), and empty string defaults for `drive_file_id`, `drive_file_url`, and `review_notes`.
   - **Step 5:** Add a **Google Sheets** node named `Record brief in Google Sheets`. Connect `Build reusable image brief`. Set resource to `sheet`, operation to `append`, mapping mode to define below, cell format to `RAW`, and use OAuth2 credentials. Map spreadsheet ID and sheet name from configuration nodes.
   - **Step 6:** Add a **Form** (completion) node named `Show prompt and manual creation steps`. Connect `Record brief in Google Sheets`. Set operation to `completion`, respond with `showText`, and insert HTML response rendering the task details, prompt, and Whisk instructions.
   - **Step 7:** Add a **Form** (completion) node named `Explain invalid brief`. Connect the `False` output of `Validate brief inputs`. Set operation to `completion`, respond with `showText`, and insert error instructions.

3. **Build the Asset Return & Archiving Branch:**
   - **Step 8:** Create a **Form Trigger** node named `Return completed image`. Configure HTTP Basic Auth credentials. Add form fields: `brief_id` (text, required), `asset` (file upload accepting `.png,.jpg,.jpeg,.webp`, required, single file), and `review_notes` (textarea, optional). Set response mode to `lastNode`.
   - **Step 9:** Add a **Set** node named `Configure archive workspace`. Connect `Return completed image`. Assign `spreadsheet_id`, `sheet_name` (`Briefs`), `drive_folder_id`, and `max_upload_bytes` (`10485760`).
   - **Step 10:** Add an **If** node named `Validate task ID and image`. Connect `Configure archive workspace`. Configure conditions to validate task ID regex, binary asset presence, MIME types, and file size constraints.
   - **Step 11:** Add a **Google Sheets** node named `Find original brief`. Connect the `True` output of `Validate task ID and image` (Output 0). Set resource to `sheet`, operation to `read`, filters UI matching `brief_id` to `$('Return completed image').first().json.brief_id.trim()`, and enable `alwaysOutputData`.
   - **Step 12:** Add an **If** node named `Require one matching brief`. Connect `Find original brief`. Configure conditions to verify exactly one matching row and valid lifecycle status.
   - **Step 13:** Add a **Google Drive** node named `Find an already archived image`. Connect the `True` output of `Require one matching brief`. Set resource to `fileFolder`, operation to `search`, limit to 2, query to search active files in parent folder containing `brief_id`, and enable `alwaysOutputData`.
   - **Step 14:** Add an **If** node named `Require an unambiguous archive`. Connect `Find an already archived image`. Configure condition to check that result count equals 1.
   - **Step 15:** Add a **Merge** node named `Restore uploaded binary`. Connect output 0 of `Validate task ID and image` to Input 0, and connect the `True` output of `Require an unambiguous archive` to Input 1. Set mode to `combine`, clash handling to `preferInput2`, and combine by position.
   - **Step 16:** Add an **If** node named `Already archived`. Connect `Restore uploaded binary`. Configure condition `!!$json.id`.
   - **Step 17:** Add a **Set** node named `Use existing file`. Connect the `True` output of `Already archived`. Assign `file_id`, `file_url`, and `reused` (`yes`).
   - **Step 18:** Add an **If** node named `Allow first asset upload`. Connect the `False` output of `Already archived`. Configure condition checking `awaiting_creation` status and empty Drive file ID.
   - **Step 19:** Add a **Google Drive** node named `Upload completed image`. Connect the `True` output of `Allow first asset upload`. Set resource to `file`, operation to `upload`, folder ID from configuration, filename expression using brief ID and sanitized product name, and app properties mapping `whiskBriefId`.
   - **Step 20:** Add a **Set** node named `Use uploaded file`. Connect `Upload completed image`. Assign `file_id`, `file_url`, and `reused` (`no`).
   - **Step 21:** Add a **Google Sheets** node named `Update original brief with asset`. Connect both `Use existing file` and `Use uploaded file`. Set resource to `sheet`, operation to `update`, matching columns `brief_id`, cell format `RAW`, and map status, Drive file ID, Drive file URL, review notes, and updated timestamp.
   - **Step 22:** Add a **Form** (completion) node named `Show archived asset`. Connect `Update original brief with asset`. Set operation to `completion`, respond with `showText`, and insert HTML success summary.
   - **Step 23:** Create failure handling **Form** nodes (`Explain invalid asset`, `Explain unmatched brief`, `Explain duplicate archive`, `Explain missing previous asset`) and connect them to their respective `False` branch outputs from the validation and check nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Whisk AI Tool | [Whisk AI Website](https://whisk-ai.io/) |
| Workflow Purpose | Organizes human-led creation and tracking of product image briefs without requiring an automated API key. |
| Operational Best Practices | Submit one return task at a time. Sequential retries reuse existing archived files, but concurrent submissions are not transactionally locked. |