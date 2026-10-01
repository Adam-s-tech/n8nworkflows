Create vertical product promo videos from Google Sheets or Shopify with Zvid

https://n8nworkflows.xyz/workflows/create-vertical-product-promo-videos-from-google-sheets-or-shopify-with-zvid-19763


# Create vertical product promo videos from Google Sheets or Shopify with Zvid

### 1. Workflow Overview

This workflow automates the creation of professional, vertical (1080x1920) product promotional videos using the **Zvid** rendering engine, pulling product data from either **Google Sheets** or a **Shopify** store. It can be triggered manually for testing or automatically on a 30-minute schedule. 

The workflow is organized into the following logical functional blocks:
- **1.1 Input Reception & Configuration:** Handles manual or scheduled triggers, initializes global branding/runtime variables, and determines whether to fetch products from Google Sheets or Shopify.
- **1.2 Product Queue & Normalization:** Retrieves raw product data, verifies availability, and normalizes disparate sources (Sheet rows vs. Shopify JSON responses) into a unified internal data structure.
- **1.3 Asset Verification & Project Building:** Probes background audio accessibility via an HTTP HEAD request and executes custom JavaScript to construct a three-scene Zvid project JSON (Hook, Offer, Call-to-Action) equipped with adaptive typography, image padding preservation, and pan/zoom motion layers.
- **1.4 Project Validation & Preview Handling:** Validates the compiled project payload against Zvid’s API to estimate credit requirements. If `dryRun` is enabled, it saves a draft directly in the Zvid editor and stops; otherwise, it proceeds to production rendering.
- **1.5 Video Rendering & Polling Loop:** Submits the render job to Zvid, pauses via an iterative wait/poll loop until rendering completes, and fails fast if timeouts or errors occur.
- **1.6 Result Storage & Media Retrieval:** Branches based on input mode to update Google Sheets (marking the row as done and writing the video URL) or internal global static data (storing Shopify markers), optionally downloading the finished binary video file for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes workflow execution via manual test triggers or cron-style scheduling, feeding global parameters into a centralized configuration node and routing execution to the selected data source branch.
- **Nodes Involved:** `Test manually`, `Every 30 minutes`, `Config`, `Source?`.
- **Node Details:**
  - **`Test manually`**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger Node).
    - *Configuration:* Default.
    - *Input/Output:* Output connects to `Config`.
    - *Edge Cases:* None.
  - **`Every 30 minutes`**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node).
    - *Configuration:* Interval rule set to every `30` minutes.
    - *Input/Output:* Output connects to `Config`.
    - *Edge Cases:* Ensure n8n instance timezone aligns with operational expectations.
  - **`Config`**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation / Initialization).
    - *Configuration:* Raw JSON mode outputting global configuration variables (`apiUrl`, `source`, `brandName`, `brandColor`, `dryRun`, `musicUrl`, etc.).
    - *Key Expressions:* Defines operational toggles like `={{ String($('Config').first().json.source || 'sheet').toLowerCase() }}`.
    - *Input/Output:* Inputs from both triggers; output connects to `Source?`.
  - **`Source?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates if configuration source equals `'sheet'` (case-insensitive).
    - *Key Expressions:* `={{ String($('Config').first().json.source || 'sheet').toLowerCase() === 'sheet' }}`.
    - *Input/Output:* Input from `Config`; True branch connects to `Read products sheet`, False branch connects to `Poll new products`.

---

#### 2.2 Product Queue & Normalization
- **Overview:** Fetches product catalog rows from Google Sheets or queries the Shopify Admin REST API, extracts the next actionable item, and normalizes payload structures while preventing duplicate processing.
- **Nodes Involved:** `Read products sheet`, `Pick next product`, `Poll new products`, `New product?`, `Product found?`, `Nothing to render`.
- **Node Details:**
  - **`Read products sheet`**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration / Read Operation).
    - *Configuration:* Reads rows from a specified Google Sheet document and tab, executing up to 3 retries on failure.
    - *Input/Output:* Input from `Source?` (True); output connects to `Pick next product`.
    - *Edge Cases:* Authentication expiration, missing spreadsheet IDs/tab names, empty rows.
  - **`Pick next product`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation / Logic).
    - *Configuration:* JavaScript function scanning row items to locate the first row where the `Status` column is empty. Validates presence of `Title` and `ImageUrl1`.
    - *Input/Output:* Input from `Read products sheet`; output connects to `New product?`.
  - **`Poll new products`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request).
    - *Configuration:* HTTP GET request to Shopify Admin API (`/admin/api/{version}/products.json`). Uses Generic Header Auth (`X-Shopify-Access-Token`). Configured with `neverError: true` and `continueRegularOutput` to prevent workflow crashes on API hiccups.
    - *Input/Output:* Input from `Source?` (False); output connects to `New product?`.
    - *Edge Cases:* Invalid API tokens, rate-limiting (HTTP 429), unreachable shop domains.
  - **`New product?`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Normalization / Deduplication).
    - *Configuration:* Normalizes both Sheet and Shopify inputs into a unified schema (`{ found, title, price, compareAtPrice, imageUrls[], tagline, ... }`). Handles Shopify static data deduplication against `global` workflow static data to track the latest processed item.
    - *Input/Output:* Inputs receive data from either `Pick next product` or `Poll new products`; output connects to `Product found?`.
  - **`Product found?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates boolean flag `$json.found`.
    - *Input/Output:* Input from `New product?`; True branch connects to `Check music`, False branch connects to `Nothing to render`.
  - **`Nothing to render`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Termination Handler).
    - *Configuration:* Generates a graceful success status payload indicating zero items required processing.
    - *Input/Output:* Input from `Product found?` (False).

---

#### 2.3 Asset Verification & Project Building
- **Overview:** Verifies background music track availability via network probe and executes an algorithmic layout builder to assemble a three-scene Zvid project JSON.
- **Nodes Involved:** `Check music`, `Build project JSON`.
- **Node Details:**
  - **`Check music`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Network Probe).
    - *Configuration:* HTTP HEAD request to the configured `musicUrl` with `neverError: true`.
    - *Input/Output:* Input from `Product found?` (True); output connects to `Build project JSON`.
    - *Edge Cases:* Broken audio links, oversized files exceeding `maxMusicBytes`.
  - **`Build project JSON`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Complex Layout Engine).
    - *Configuration:* Comprehensive JavaScript layout generator assembling a 1080x1920 vertical video structure containing three scenes (Hook, Offer, CTA), adaptive typography fitting loops, image containment plates, and kinetic motion keyframes.
    - *Input/Output:* Input from `Check music`; output connects to `Validate project (free)`.

---

#### 2.4 Project Validation & Preview Handling
- **Overview:** Validates the assembled project configuration against the Zvid API, evaluates credit requirements, and branches to either generate an interactive web editor draft or proceed to automated rendering.
- **Nodes Involved:** `Validate project (free)`, `Check validation`, `Dry run?`, `Save draft to editor`, `Dry run summary`.
- **Node Details:**
  - **`Validate project (free)`**
    - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (API Operation - Validate).
    - *Configuration:* Uses Zvid API credentials; validates payload schema and retrieves credit quotes.
    - *Input/Output:* Input from `Build project JSON`; output connects to `Check validation`.
  - **`Check validation`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Error Extractor).
    - *Configuration:* Inspects validation response; throws descriptive field-level errors if validation fails (HTTP $\neq$ 200).
    - *Input/Output:* Input from `Validate project (free)`; output connects to `Dry run?`.
  - **`Dry run?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates `dryRun` configuration flag (`={{ $('Config').first().json.dryRun }}`).
    - *Input/Output:* Input from `Check validation`; True branch connects to `Save draft to editor`, False branch connects to `Submit render`.
  - **`Save draft to editor`**
    - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (API Operation - Create Project Draft).
    - *Configuration:* Creates an editable project within the Zvid web editor. Configured with `onError: continueRegularOutput` and retry logic.
    - *Input/Output:* Input from `Dry run?` (True); output connects to `Dry run summary`.
  - **`Dry run summary`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Reporting Node).
    - *Configuration:* Assembles a summary object containing editor preview URLs, estimated credit costs, and execution metadata without consuming render credits.
    - *Input/Output:* Input from `Save draft to editor`.

---

#### 2.5 Video Rendering & Polling Loop
- **Overview:** Submits the validated project payload to the Zvid rendering engine as an asynchronous job, polling execution status at configured intervals until completion or timeout.
- **Nodes Involved:** `Submit render`, `Wait`, `Get render status`, `Render finished?`, `Still rendering?`.
- **Node Details:**
  - **`Submit render`**
    - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (API Operation - Create Render Job).
    - *Configuration:* Submits render payload (`renderType: 'video'`), disabling immediate synchronous completion waiting (`waitForCompletion: false`).
    - *Input/Output:* Input from `Dry run?` (False); output connects to `Wait`.
  - **`Wait`**
    - *Type & Technical Role:* `n8n-nodes-base.wait` (Delay Node).
    - *Configuration:* Pauses execution for a duration defined by `pollSeconds` in Config.
    - *Input/Output:* Inputs from `Submit render` and `Still rendering?`; output connects to `Get render status`.
  - **`Get render status`**
    - *Type & Technical Role:* `@zvid/n8n-nodes-zvid.zvid` (API Operation - Get Job Status).
    - *Configuration:* Queries render job status using the generated `jobId`.
    - *Input/Output:* Input from `Wait`; output connects to `Render finished?`.
  - **`Render finished?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Checks if job state equals `'completed'` (`={{ $json.state === 'completed' }}`).
    - *Input/Output:* Input from `Get render status`; True branch connects to `Sheet mode?`, False branch connects to `Still rendering?`.
  - **`Still rendering?`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Loop Guard / Timeout Controller).
    - *Configuration:* Validates job state for failures and enforces maximum polling timeout thresholds based on `timeoutMinutes` and `pollSeconds`.
    - *Input/Output:* Input from `Render finished?` (False); output loops back to `Wait`.

---

#### 2.6 Result Storage & Media Retrieval
- **Overview:** Updates originating record structures (Google Sheets row updates or Shopify static data tracking) upon successful render completion, and optionally retrieves binary video files for downstream inspection.
- **Nodes Involved:** `Sheet mode?`, `Mark row done`, `Run summary`, `Video ready to watch?`, `▶ Watch video`.
- **Node Details:**
  - **`Sheet mode?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Evaluates if source mode equals `'sheet'`.
    - *Input/Output:* Input from `Render finished?` (True); True branch connects to `Mark row done`, False branch connects to `Run summary`.
  - **`Mark row done`**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Integration / Update Operation).
    - *Configuration:* Updates the source Google Sheet row matching `row_number`, setting `Status` to done and writing the generated `VideoUrl`.
    - *Input/Output:* Input from `Sheet mode?` (True); output connects to `Run summary`.
  - **`Run summary`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Reporting & State Persistence).
    - *Configuration:* Compiles final execution metrics, updates global workflow static data for Shopify product IDs, and formats production run reports.
    - *Input/Output:* Inputs from `Mark row done` and `Sheet mode?` (False); output connects to `Video ready to watch?`.
  - **`Video ready to watch?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration:* Verifies execution was not a dry run and that a valid HTTP video URL exists.
    - *Input/Output:* Input from `Run summary`; True branch connects to `▶ Watch video`.
  - **`▶ Watch video`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Binary File Download).
    - *Configuration:* HTTP GET request downloading the completed video asset into binary property `data` with file response format.
    - *Input/Output:* Input from `Video ready to watch?` (True).
    - *Edge Cases:* Transient network errors during file download.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Test manually | n8n-nodes-base.manualTrigger | Trigger workflow manually | None | Config | 1. Choose settings and an entry point |
| Every 30 minutes | n8n-nodes-base.scheduleTrigger | Trigger workflow on interval | None | Config | 1. Choose settings and an entry point |
| Config | n8n-nodes-base.set | Initialize configuration | Test manually, Every 30 minutes | Source? | 1. Choose settings and an entry point |
| Source? | n8n-nodes-base.if | Route execution by source type | Config | Read products sheet, Poll new products | 1. Choose settings and an entry point |
| Read products sheet | n8n-nodes-base.googleSheets | Read product rows from sheet | Source? | Pick next product | 2. Select a product source |
| Pick next product | n8n-nodes-base.code | Select first empty status row | Read products sheet | New product? | 2. Select a product source |
| Poll new products | n8n-nodes-base.httpRequest | Query Shopify Admin API | Source? | New product? | 2. Select a product source |
| New product? | n8n-nodes-base.code | Normalize & deduplicate product data | Pick next product, Poll new products | Product found? | 2. Select a product source |
| Product found? | n8n-nodes-base.if | Check if product exists | New product? | Check music, Nothing to render | 2. Select a product source |
| Nothing to render | n8n-nodes-base.code | Handle empty queue state | Product found? | None | 2. Select a product source |
| Check music | n8n-nodes-base.httpRequest | Probe audio asset availability | Product found? | Build project JSON | 4. Build and validate the design |
| Build project JSON | n8n-nodes-base.code | Generate 3-scene Zvid project JSON | Check music | Validate project (free) | 4. Build and validate the design |
| Validate project (free) | @zvid/n8n-nodes-zvid.zvid | Validate payload & get quote | Build project JSON | Check validation | 4. Build and validate the design |
| Check validation | n8n-nodes-base.code | Parse validation errors/quotes | Validate project (free) | Dry run? | 4. Build and validate the design |
| Dry run? | n8n-nodes-base.if | Route based on dryRun flag | Check validation | Save draft to editor, Submit render | 5. Choose an optional editor preview |
| Save draft to editor | @zvid/n8n-nodes-zvid.zvid | Create Zvid web editor draft | Dry run? | Dry run summary | 5. Choose an optional editor preview |
| Dry run summary | n8n-nodes-base.code | Generate dry run report | Save draft to editor | None | 5. Choose an optional editor preview |
| Submit render | @zvid/n8n-nodes-zvid.zvid | Submit video render job | Dry run? | Wait | 6. Render and wait for completion |
| Wait | n8n-nodes-base.wait | Poll delay timer | Submit render, Still rendering? | Get render status | 6. Render and wait for completion |
| Get render status | @zvid/n8n-nodes-zvid.zvid | Query render job status | Wait | Render finished? | 6. Render and wait for completion |
| Render finished? | n8n-nodes-base.if | Check if rendering is complete | Get render status | Sheet mode?, Still rendering? | 6. Render and wait for completion |
| Still rendering? | n8n-nodes-base.code | Enforce timeout & retry guards | Render finished? | Wait | 6. Render and wait for completion |
| Sheet mode? | n8n-nodes-base.if | Route based on input source | Render finished? | Mark row done, Run summary | 7. Save the completed result |
| Mark row done | n8n-nodes-base.googleSheets | Update sheet with status & URL | Sheet mode? | Run summary | 7. Save the completed result |
| Run summary | n8n-nodes-base.code | Compile final production report | Mark row done, Sheet mode? | Video ready to watch? | 7. Save the completed result |
| Video ready to watch? | n8n-nodes-base.if | Verify valid video URL | Run summary | ▶ Watch video | 8. Review the finished media |
| ▶ Watch video | n8n-nodes-base.httpRequest | Download binary video asset | Video ready to watch? | None | 8. Review the finished media |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Install Community Node:**
   - Go to **Settings → Community nodes** in n8n and install **`@zvid/n8n-nodes-zvid`**.
2. **Create Triggers & Configuration:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) named `Test manually`.
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Every 30 minutes` configured for every 30 minutes.
   - Add a **Set** node (`n8n-nodes-base.set`) named `Config`. Set mode to raw JSON and paste the configuration schema defining branding, API URLs, `source: "sheet"`, and `dryRun: false`.
   - Connect both triggers to `Config`.
3. **Setup Source Router:**
   - Add an **If** node named `Source?`. Set condition to evaluate if `Config` source equals `'sheet'`. Connect `Config` output to this node.
4. **Build Sheet Branch:**
   - Add a **Google Sheets** node named `Read products sheet` (Resource: `Sheet`, Operation: `Read`). Select your target document and tab. Connect `Source?` True branch to it.
   - Add a **Code** node named `Pick next product`. Insert JavaScript that finds the first row with an empty `Status` column, validating `Title` and `ImageUrl1`. Connect `Read products sheet` to it.
5. **Build Shopify Branch:**
   - Add an **HTTP Request** node named `Poll new products`. Set URL to `https://{{ $('Config').first().json.shopDomain }}.myshopify.com/admin/api/{{ $('Config').first().json.apiVersion }}/products.json?limit=5&order=created_at+desc`. Configure Generic Header Authentication (`X-Shopify-Access-Token`), `neverError: true`, and `continueRegularOutput`. Connect `Source?` False branch to it.
6. **Normalize Product Data:**
   - Add a **Code** node named `New product?` to unify Sheet and Shopify data schemas and handle static data deduplication. Connect both `Pick next product` and `Poll new products` outputs to it.
   - Add an **If** node named `Product found?` checking `={{ $json.found }}`. Connect `New product?` to it.
   - Add a **Code** node named `Nothing to render` connected to the False branch of `Product found?`.
7. **Build Project Payload:**
   - Add an **HTTP Request** node named `Check music` (Method: `HEAD`, URL: `={{ $('Config').first().json.musicUrl }}`). Connect True branch of `Product found?` to it.
   - Add a **Code** node named `Build project JSON` containing the 3-scene Zvid project compilation script (handling SVG generation, adaptive text measurement, and photo containing plates). Connect `Check music` to it.
8. **Validate & Preview:**
   - Add a **Zvid** node (`@zvid/n8n-nodes-zvid.zvid`) named `Validate project (free)` (Resource: `Render`, Operation: `Validate`). Select your Zvid API credential. Connect `Build project JSON` to it.
   - Add a **Code** node named `Check validation` to parse credit quotes and errors. Connect `Validate project (free)` to it.
   - Add an **If** node named `Dry run?` evaluating `={{ $('Config').first().json.dryRun }}`. Connect `Check validation` to it.
   - Add a **Zvid** node named `Save draft to editor` (Resource: `Project`, Operation: `Create`) with `continueRegularOutput`. Connect True branch of `Dry run?` to it.
   - Add a **Code** node named `Dry run summary` connected to `Save draft to editor`.
9. **Render & Poll Loop:**
   - Add a **Zvid** node named `Submit render` (Resource: `Render`, Operation: `Create`, Render Type: `Video`, Wait for Completion: `false`). Connect False branch of `Dry run?` to it.
   - Add a **Wait** node named `Wait` set to poll seconds. Connect `Submit render` to it.
   - Add a **Zvid** node named `Get render status` (Resource: `Render`, Operation: `Get`). Connect `Wait` to it.
   - Add an **If** node named `Render finished?` checking `={{ $json.state === 'completed' }}`. Connect `Get render status` to it.
   - Add a **Code** node named `Still rendering?` to enforce timeout/failure limits. Connect False branch of `Render finished?` to it, and loop its output back into `Wait`.
10. **Save Results & Review:**
    - Add an **If** node named `Sheet mode?` checking if source is `'sheet'`. Connect True branch of `Render finished?` to it.
    - Add a **Google Sheets** node named `Mark row done` (Resource: `Sheet`, Operation: `Update`, matching on `row_number`). Connect True branch of `Sheet mode?` to it.
    - Add a **Code** node named `Run summary` connected to `Mark row done` and False branch of `Sheet mode?`.
    - Add an **If** node named `Video ready to watch?` checking valid video URLs. Connect `Run summary` to it.
    - Add an **HTTP Request** node named `▶ Watch video` configured with response format `File` and output property `data`. Connect True branch of `Video ready to watch?` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zvid Community Node Installation | [Zvid Integration Guide](https://n8n.io/integrations/zvid/) |
| Zvid API Keys Management | [Zvid API Keys Portal](https://app.zvid.io/api-keys) |
| Shopify Admin REST API Documentation | [Shopify REST API Docs](https://shopify.dev/docs/api/admin-rest/latest/resources/product) |