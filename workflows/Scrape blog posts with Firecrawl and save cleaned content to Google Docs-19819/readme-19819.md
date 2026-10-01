Scrape blog posts with Firecrawl and save cleaned content to Google Docs

https://n8nworkflows.xyz/workflows/scrape-blog-posts-with-firecrawl-and-save-cleaned-content-to-google-docs-19819


# Scrape blog posts with Firecrawl and save cleaned content to Google Docs

### 1. Workflow Overview

This workflow is designed to automate the extraction of content from multiple blog post URLs, clean the resulting markdown text, and consolidate the output into a single, newly created Google Document. It is intended for content aggregation, research, and backup workflows where clean text extraction from web pages is required.

The logic is divided into three functional blocks:
- **1.1 Input Reception:** Manually triggers the execution and initiates the batch scraping job.
- **1.2 Scraping & Polling Loop:** Submits URLs to Firecrawl, waits, polls for completion, handles failures, and loops until the job succeeds.
- **1.3 Content Processing & Output:** Creates a destination Google Document, transforms raw scraped markdown into clean plain text, and writes the consolidated output to the document.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Initializes the workflow execution manually on demand.
- **Nodes Involved:** `When clicking ‘Execute workflow’`
- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` (v1) - Acts as the entry point for manual testing and execution.
    - *Configuration choices:* Default settings.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Outputs to `Batch Scrape Blog Post URLs`.
    - *Edge cases:* None.

#### 2.2 Scraping & Polling Loop
- **Overview:** Submits a batch of URLs to Firecrawl for scraping, polls the job status periodically until completion, and terminates early if the job fails.
- **Nodes Involved:** `Batch Scrape Blog Post URLs`, `Wait 1 Minute`, `Check Batch Scrape Status`, `If Scrape Completed`, `If Scrape Failed`, `Batch Scrape Failed`
- **Node Details:**
  - **Batch Scrape Blog Post URLs**
    - *Type and technical role:* `@mendable/n8n-nodes-firecrawl.firecrawl` (v1.1) - Submits a batch scraping request to Firecrawl.
    - *Configuration choices:* Operation set to `batchScrape` with 3 maximum tries and a 5-second wait between tries on failure.
    - *Key expressions or variables:* Array of hardcoded target URLs.
    - *Input and output connections:* Input from `When clicking ‘Execute workflow’`; output to `Wait 1 Minute`.
    - *Credentials:* `Firecrawl Dummy Account` (`firecrawlApi`).
    - *Edge cases:* Invalid URLs or API authentication errors.
  - **Wait 1 Minute**
    - *Type and technical role:* `n8n-nodes-base.wait` (v1.1) - Pauses workflow execution to allow the scraping job time to process.
    - *Configuration choices:* Unit set to `minutes`, amount set to `1`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input from `Batch Scrape Blog Post URLs` and `If Scrape Failed` (loop); output to `Check Batch Scrape Status`.
  - **Check Batch Scrape Status**
    - *Type and technical role:* `@mendable/n8n-nodes-firecrawl.firecrawl` (v1.1) - Checks the current status of the active batch job.
    - *Configuration choices:* Operation set to `batchScrapeStatus` with retry mechanisms configured.
    - *Key expressions or variables:* `={{ $json.data.id }}`
    - *Input and output connections:* Input from `Wait 1 Minute`; output to `If Scrape Completed`.
    - *Credentials:* `Firecrawl Dummy Account` (`firecrawlApi`).
    - *Edge cases:* API timeout or missing batch ID reference.
  - **If Scrape Completed**
    - *Type and technical role:* `n8n-nodes-base.if` (v2.3) - Evaluates whether the scraping job has finished successfully.
    - *Configuration choices:* Checks if job status equals `completed`.
    - *Key expressions or variables:* `={{ $json.status }}`
    - *Input and output connections:* Input from `Check Batch Scrape Status`; outputs true path to `Create Google Doc` and false path to `If Scrape Failed`.
  - **If Scrape Failed**
    - *Type and technical role:* `n8n-nodes-base.if` (v2.3) - Evaluates whether the scraping job failed.
    - *Configuration choices:* Checks if job status equals `failed`.
    - *Key expressions or variables:* `={{ $json.status }}`
    - *Input and output connections:* Input from `If Scrape Completed` (false branch); outputs true path to `Batch Scrape Failed` and false path to `Wait 1 Minute` (continuing the poll).
  - **Batch Scrape Failed**
    - *Type and technical role:* `n8n-nodes-base.stopAndError` (v1) - Halts execution and throws an explicit error.
    - *Configuration choices:* Custom error message configured.
    - *Key expressions or variables:* `=Firecrawl batch scrape failed (status: {{ $json.status }})`
    - *Input and output connections:* Input from `If Scrape Failed`.

#### 2.3 Content Processing & Output
- **Overview:** Generates a new Google Doc, cleans the scraped markdown data to remove noise, and updates the document with the consolidated text.
- **Nodes Involved:** `Create Google Doc`, `Clean Scraped Markdown`, `Update Google Doc`
- **Node Details:**
  - **Create Google Doc**
    - *Type and technical role:* `n8n-nodes-base.googleDocs` (v2) - Creates a blank document in Google Drive.
    - *Configuration choices:* Operation set to create, with 3 maximum retries on failure.
    - *Key expressions or variables:* `=Firecrawl ID: {{ $('Batch Scrape Blog Post URLs').item.json.data.id }}`
    - *Input and output connections:* Input from `If Scrape Completed`; output to `Clean Scraped Markdown`.
    - *Credentials:* `Google Docs account (Dummy)` (`googleDocsOAuth2Api`).
    - *Edge cases:* Insufficient OAuth2 scopes or permission errors.
  - **Clean Scraped Markdown**
    - *Type and technical role:* `n8n-nodes-base.code` (v2) - Executes JavaScript to sanitize and structure raw markdown content.
    - *Configuration choices:* Custom JS code parsing page metadata, removing navigational headers/footers, stripping image tags, converting markdown links to plain text, and normalizing spacing.
    - *Key expressions or variables:* `$('If Scrape Completed').item.json.data`
    - *Input and output connections:* Input from `Create Google Doc`; output to `Update Google Doc`.
    - *Edge cases:* Missing metadata fields or unexpected markdown structures.
  - **Update Google Doc**
    - *Type and technical role:* `n8n-nodes-base.googleDocs` (v2) - Inserts text into the specified Google Document.
    - *Configuration choices:* Operation set to `update` with `insert` action mode.
    - *Key expressions or variables:* Document URL retrieved via `={{ $('Create Google Doc').item.json.id }}`; text payload retrieved via `={{ $json.data }}`.
    - *Input and output connections:* Input from `Clean Scraped Markdown`.
    - *Credentials:* `Google Docs account (Dummy)` (`googleDocsOAuth2Api`).
    - *Edge cases:* Payload size limits or document lock issues.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point | None | Batch Scrape Blog Post URLs | Start workflow<br><br>Manual trigger to kick off the fetch. |
| Batch Scrape Blog Post URLs | @mendable/n8n-nodes-firecrawl.firecrawl | Submits URLs for batch scraping | When clicking ‘Execute workflow’ | Wait 1 Minute | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| Wait 1 Minute | n8n-nodes-base.wait | Pauses workflow execution | Batch Scrape Blog Post URLs, If Scrape Failed | Check Batch Scrape Status | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| Check Batch Scrape Status | @mendable/n8n-nodes-firecrawl.firecrawl | Checks batch scrape status | Wait 1 Minute | If Scrape Completed | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| If Scrape Completed | n8n-nodes-base.if | Verifies if scrape job is completed | Check Batch Scrape Status | Create Google Doc, If Scrape Failed | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| If Scrape Failed | n8n-nodes-base.if | Verifies if scrape job has failed | If Scrape Completed | Batch Scrape Failed, Wait 1 Minute | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| Batch Scrape Failed | n8n-nodes-base.stopAndError | Halts workflow on failure | If Scrape Failed | None | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Scrape URLs and poll for completion<br><br>Submits a batch scrape to Firecrawl, then waits and checks status until the job is done. Stops with an error if Firecrawl reports the scrape failed, instead of polling forever. |
| Create Google Doc | n8n-nodes-base.googleDocs | Creates a new Google Document | If Scrape Completed | Clean Scraped Markdown | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Clean content and save to Google Doc<br><br>Creates a Google Doc, strips markdown noise from the scraped pages, then inserts the cleaned text. |
| Clean Scraped Markdown | n8n-nodes-base.code | Sanitizes scraped markdown data | Create Google Doc | Update Google Doc | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scrape Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Clean content and save to Google Doc<br><br>Creates a Google Doc, strips markdown noise from the scraped pages, then inserts the cleaned text. |
| Update Google Doc | n8n-nodes-base.googleDocs | Inserts text into the Google Doc | Clean Scraped Markdown | None | Scrape Blog Posts with Firecrawl and Save to Google Docs<br><br>### How it works<br><br>1. Manually triggers a batch scrape of a list of blog post URLs via Firecrawl.<br>2. Waits 1 minute, then polls Firecrawl for the batch scrape status.<br>3. Repeats the wait-and-check loop until the scrape job reports completed or failed.<br>4. Stops with a clear error if the scrape job fails, instead of polling indefinitely.<br>5. Creates a new Google Doc once the scrape finishes.<br>6. Cleans the scraped markdown (strips nav/footer, links, and formatting) in a Code node.<br>7. Inserts the cleaned text into the Google Doc.<br><br>### Setup steps<br><br>- [ ] Add a Firecrawl API credential and select it on both Firecrawl nodes.<br>- [ ] Add Google Docs OAuth2 credentials and select them on both Google Docs nodes.<br>- [ ] Replace the hardcoded URL list in "Batch Scraped Blog Post URLs" with your own target URLs.<br><br>### Customization<br><br>Adjust the wait duration or the footer marker text in "Clean Scraped Markdown" to match a different source site. Network calls (Firecrawl, Google Docs) retry up to 3 times on transient failure.<br><br>Clean content and save to Google Doc<br><br>Creates a Google Doc, strips markdown noise from the scraped pages, then inserts the cleaned text. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`, v1). Name it `When clicking ‘Execute workflow’`.
2. **Create Firecrawl Batch Scrape Node:**
   - Add a **Firecrawl** node (`@mendable/n8n-nodes-firecrawl.firecrawl`, v1.1). Name it `Batch Scrape Blog Post URLs`.
   - Set operation to `batchScrape`.
   - Set parameters with an array of target URLs:
     ```json
     [
       "https:.....",
       "https:.....",
       "https:....."
     ]
     ```
   - Enable retry options: set `maxTries` to `3` and `waitBetweenTries` to `5000`.
   - Configure credentials: select your `Firecrawl API` credential.
   - Connect `When clicking ‘Execute workflow’` output to this node.
3. **Create Wait Node:**
   - Add a **Wait** node (`n8n-nodes-base.wait`, v1.1). Name it `Wait 1 Minute`.
   - Set unit to `minutes` and amount to `1`.
   - Connect `Batch Scrape Blog Post URLs` output to this node.
4. **Create Firecrawl Status Check Node:**
   - Add a **Firecrawl** node (`@mendable/n8n-nodes-firecrawl.firecrawl`, v1.1). Name it `Check Batch Scrape Status`.
   - Set operation to `batchScrapeStatus`.
   - Set Batch ID parameter: `={{ $json.data.id }}`.
   - Configure retries (`maxTries`: `3`, `waitBetweenTries`: `5000`) and the `Firecrawl API` credential.
   - Connect `Wait 1 Minute` output to this node.
5. **Create Completion Check Node:**
   - Add an **If** node (`n8n-nodes-base.if`, v2.3). Name it `If Scrape Completed`.
   - Configure condition: Left Value `={{ $json.status }}`, Operator `equals`, Right Value `completed`.
   - Connect `Check Batch Scrape Status` output to this node.
6. **Create Failure Check Node:**
   - Add an **If** node (`n8n-nodes-base.if`, v2.3). Name it `If Scrape Failed`.
   - Configure condition: Left Value `={{ $json.status }}`, Operator `equals`, Right Value `failed`.
   - Connect the `false` output of `If Scrape Completed` to this node.
7. **Create Error Stop Node:**
   - Add a **Stop and Error** node (`n8n-nodes-base.stopAndError`, v1). Name it `Batch Scrape Failed`.
   - Set Error Message: `=Firecrawl batch scrape failed (status: {{ $json.status }})`.
   - Connect the `true` output of `If Scrape Failed` to this node.
   - Connect the `false` output of `If Scrape Failed` back to `Wait 1 Minute` to complete the polling loop.
8. **Create Google Document Creation Node:**
   - Add a **Google Docs** node (`n8n-nodes-base.googleDocs`, v2). Name it `Create Google Doc`.
   - Set operation to create document.
   - Set Title parameter: `=Firecrawl ID: {{ $('Batch Scrape Blog Post URLs').item.json.data.id }}`.
   - Configure retries and select your `Google Docs OAuth2 API` credential.
   - Connect the `true` output of `If Scrape Completed` to this node.
9. **Create Code Sanitization Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2). Name it `Clean Scraped Markdown`.
   - Insert the JavaScript code snippet to extract and parse page metadata, drop unwanted navigation/footer text, remove markdown syntax noise, and return consolidated content.
   - Connect `Create Google Doc` output to this node.
10. **Create Google Document Update Node:**
    - Add a **Google Docs** node (`n8n-nodes-base.googleDocs`, v2). Name it `Update Google Doc`.
    - Set operation to `update`, document URL to `={{ $('Create Google Doc').item.json.id }}`, and configure actions UI with an `insert` action containing text `={{ $json.data }}`.
    - Configure retries and select your `Google Docs OAuth2 API` credential.
    - Connect `Clean Scraped Markdown` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Network calls to external integrations (Firecrawl and Google Docs) are pre-configured to retry up to 3 times on transient failures. | Reliability and Error Handling |
| Footer markers and navigation elements stripped in the Code node can be customized to match different target website structures. | Customization Note |