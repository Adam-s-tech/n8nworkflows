Fill missing Wix SEO titles and meta descriptions in Wix with Gemini

https://n8nworkflows.xyz/workflows/fill-missing-wix-seo-titles-and-meta-descriptions-in-wix-with-gemini-19685


# Fill missing Wix SEO titles and meta descriptions in Wix with Gemini

### 1. Workflow Overview

This workflow automates the audit and generation of Search Engine Optimization (SEO) titles and meta descriptions for Wix websites using Google’s Gemini AI. It addresses the common issue where Wix generates generic titles (e.g., “Page Name | My Site”) and leaves meta descriptions blank. The workflow evaluates static pages, blog posts, and store products, generates optimized text based on actual page content, validates compliance with strict character and formatting rules, and saves approved changes back to Wix.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution on a weekly schedule and defines global parameters (site ID, item types, character constraints, and dry-run state).
- **1.2 SEO Tag Retrieval & Validation:** Expands item types, queries the Wix SEO Metatags API, and validates the HTTP response to handle errors gracefully.
- **1.3 Gap Analysis:** Filters items to process only those lacking custom tags, respecting `noindex` rules and existing manual overrides.
- **1.4 Product Data Enrichment:** Fetches rich text descriptions, info sections, options, and price ranges specifically for Wix Stores products.
- **1.5 AI Generation & Validation:** Formulates a single batched prompt for Google Gemini, enforces a strict JSON schema, and evaluates generated tags for length limits, formatting errors, and duplicates.
- **1.6 Execution & Reporting:** Evaluates whether the workflow is in dry-run mode, patches approved updates to Wix via single-item endpoints, and compiles a comprehensive execution report.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Establishes the execution trigger and loads persistent configuration parameters required by downstream nodes.
- **Nodes Involved:** `When Monday at 9am`, `Set Wix Configuration`
- **Node Details:**
  - **`When Monday at 9am`**
    - Type: `ScheduleTrigger`
    - Role: Initiates workflow execution weekly on Mondays at 09:00 AM.
    - Configuration: Trigger interval set to weekly (Monday, 09:00).
    - Connections: Outputs to `Set Wix Configuration`.
  - **`Set Wix Configuration`**
    - Type: `Set` (Edit Fields)
    - Role: Stores global constants including Wix site credentials, target item types, brand names, batch limits, character thresholds, and execution modes.
    - Configuration: Manual mode with assignments for variables such as `wix_site_id`, `item_types`, `brand_name`, `dry_run`, `max_items_per_run`, `title_max_chars`, `desc_min_chars`, `desc_max_chars`, `gemini_model`, `skip_noindex`, and `publish_drafted_types`.
    - Connections: Input from `When Monday at 9am`; output to `Parse Item Types`.

#### 2.2 SEO Tag Retrieval & Validation
- **Overview:** Converts comma-separated item types into individual processing streams, queries Wix for current SEO data, and intercepts API errors.
- **Nodes Involved:** `Parse Item Types`, `Get Wix SEO Tags`, `Validate Wix SEO Response`
- **Node Details:**
  - **`Parse Item Types`**
    - Type: `Code` (JavaScript)
    - Role: Splits the comma-separated `item_types` configuration string into discrete items.
    - Configuration: Runs once for all items; splits `item_types` by comma, trims, converts to uppercase, and outputs individual items containing `itemType`.
    - Connections: Input from `Set Wix Configuration`; output to `Get Wix SEO Tags`.
  - **`Get Wix SEO Tags`**
    - Type: `HttpRequest`
    - Role: Fetches resolved SEO metatags for each item type from the Wix API.
    - Configuration: GET request to `https://www.wixapis.com/seo-metatags-server/v1/item-seo-tags/{{ $json.itemType }}`. Uses Generic Header Authentication (`Authorization`) and includes a custom header `wix-site-id` referencing the configuration node. Configured with a maximum of 3 retries and a 3-second wait between attempts. Response handling set to never error out and return the full response.
    - Connections: Input from `Parse Item Types`; output to `Validate Wix SEO Response`.
  - **`Validate Wix SEO Response`**
    - Type: `Code` (JavaScript)
    - Role: Evaluates HTTP status codes from the Wix SEO API and throws descriptive errors for unauthorized access (401/403), schema mismatches (428), or generic failures.
    - Configuration: Runs once for all items; inspects `statusCode` and `body` payload.
    - Connections: Input from `Get Wix SEO Tags`; output to `Identify Missing Tags`.

#### 2.3 Gap Analysis
- **Overview:** Inspects returned SEO tags to identify items missing titles or descriptions while respecting `noindex` directives and manual edits.
- **Nodes Involved:** `Identify Missing Tags`, `If Tags Missing`, `Log Unchanged Status`
- **Node Details:**
  - **`Identify Missing Tags`**
    - Type: `Code` (JavaScript)
    - Role: Filters out pages marked with `noindex` (if enabled) and items featuring existing custom manual overrides. Enforces batch size and ensures fair distribution across item types.
    - Configuration: Evaluates tags against configured thresholds; normalizes slugs and packages payload for downstream consumption. Outputs a bypass flag (`__none: true`) if no items require processing.
    - Connections: Input from `Validate Wix SEO Response`; output to `If Tags Missing`.
  - **`If Tags Missing`**
    - Type: `If`
    - Role: Branches execution depending on whether actionable items are found.
    - Configuration: Checks if `itemId` is present and not empty.
    - Connections: Input from `Identify Missing Tags`; outputs true path to `Prepare Product ID Lookup` and false path to `Log Unchanged Status`.
  - **`Log Unchanged Status`**
    - Type: `Code` (JavaScript)
    - Role: Compiles an informative summary report when no items require SEO tag generation.
    - Configuration: Aggregates reasons for skipped items (e.g., `noindex`, already written by hand).
    - Connections: Input from `If Tags Missing` (false branch); terminates flow branch.

#### 2.4 Product Data Enrichment
- **Overview:** Queries the Wix Stores API to retrieve rich product copies, info sections, options, and pricing for store items.
- **Nodes Involved:** `Prepare Product ID Lookup`, `Post Product Search`, `Combine Product Data`
- **Node Details:**
  - **`Prepare Product ID Lookup`**
    - Type: `Code` (JavaScript)
    - Role: Prepares a filtered search query targeting items classified as `STORES_PRODUCT`.
    - Configuration: Extracts product IDs from the batch; constructs a filter payload requesting `DESCRIPTION` and `INFO_SECTION` fields.
    - Connections: Input from `If Tags Missing` (true branch); output to `Post Product Search`.
  - **`Post Product Search`**
    - Type: `HttpRequest`
    - Role: Executes a search request against the Wix Stores API.
    - Configuration: POST request to `https://www.wixapis.com/stores/v3/products/search`. Uses Header Authentication (`Authorization`) with `wix-site-id` header. Error handling set to continue regular output.
    - Connections: Input from `Prepare Product ID Lookup`; output to `Combine Product Data`.
  - **`Combine Product Data`**
    - Type: `Code` (JavaScript)
    - Role: Flattens Ricos rich content structures from Wix product responses into plain text and merges them into the item context.
    - Configuration: Parses product descriptions, info sections, variants, and price ranges, appending them to the item's source text.
    - Connections: Input from `Post Product Search`; output to `Create Gemini API Request`.

#### 2.5 AI Generation & Validation
- **Overview:** Formulates a structured prompt, calls the Google Gemini API, and validates the output against strict SEO constraints.
- **Nodes Involved:** `Create Gemini API Request`, `Post to Gemini API`, `Validate Generated Tags`
- **Node Details:**
  - **`Create Gemini API Request`**
    - Type: `Code` (JavaScript)
    - Role: Constructs a batched prompt containing item details and enforces a rigid JSON response schema.
    - Configuration: Sets prompt rules (character limits, exclusion of clickbait, ALL CAPS, exclamation marks, emojis, or unsupported pricing/claims). Configures `generationConfig` with `responseMimeType: "application/json"` and a strict array schema requiring `itemId`, `title`, and `metaDescription`.
    - Connections: Input from `Combine Product Data`; output to `Post to Gemini API`.
  - **`Post to Gemini API`**
    - Type: `HttpRequest`
    - Role: Submits the prompt to the Google Gemini model.
    - Configuration: POST request to `https://generativelanguage.googleapis.com/v1beta/models/{{ $('Set Wix Configuration').first().json.gemini_model }}:generateContent`. Uses Header Authentication (`x-goog-api-key`). Configured with up to 5 retries and a 5-second wait between attempts.
    - Connections: Input from `Create Gemini API Request`; output to `Validate Generated Tags`.
  - **`Validate Generated Tags`**
    - Type: `Code` (JavaScript)
    - Role: Parses Gemini’s JSON response and validates every suggestion against character length thresholds, prohibited symbols, duplicates, and similarity to existing tags.
    - Configuration: Filters valid items into an `accepted` array and invalid items into a `rejected` array with specific failure reasons.
    - Connections: Input from `Post to Gemini API`; output to `Check Dry Run Status`.

#### 2.6 Execution & Reporting
- **Overview:** Evaluates dry-run parameters, updates Wix via single-item PATCH requests if live, and generates a final audit report.
- **Nodes Involved:** `Check Dry Run Status`, `Construct Write Requests`, `Patch Tags on Wix`, `Log Changes Reported`
- **Node Details:**
  - **`Check Dry Run Status`**
    - Type: `If`
    - Role: Determines whether to halt execution for a dry run or proceed with live updates.
    - Configuration: Evaluates `dry_run` configuration parameter (boolean).
    - Connections: Input from `Validate Generated Tags`; outputs true path directly to `Log Changes Reported` and false path to `Construct Write Requests`.
  - **`Construct Write Requests`**
    - Type: `Code` (JavaScript)
    - Role: Prepares individual PATCH payloads, merging new title and description tags while preserving other existing tags (such as canonical links or `robots` metatags).
    - Configuration: Determines publication rules depending on whether the item type is drafted (`STATIC_PAGE`, `BLOG_POST`) or published immediately.
    - Connections: Input from `Check Dry Run Status` (false branch); output to `Patch Tags on Wix`.
  - **`Patch Tags on Wix`**
    - Type: `HttpRequest`
    - Role: Sends individual PATCH requests to update SEO metatags for each approved item in Wix.
    - Configuration: PATCH request to `={{ $json.url }}` using Header Authentication (`Authorization`) and `wix-site-id` header. Configured with 3 retries and 3-second intervals. Response handling set to never error out and return full response.
    - Connections: Input from `Construct Write Requests`; output to `Log Changes Reported`.
  - **`Log Changes Reported`**
    - Type: `Code` (JavaScript)
    - Role: Aggregates write results, API statuses, rejections, and success metrics into a final structured output report.
    - Configuration: Inspects HTTP responses from patch actions or bypasses metrics if running in dry-run mode.
    - Connections: Inputs from `Check Dry Run Status` (true branch) and `Patch Tags on Wix`; terminates workflow execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Monday at 9am | scheduleTrigger | Initiates workflow execution weekly on Mondays at 9 AM. | None | Set Wix Configuration | ## Fill missing Wix SEO titles and meta descriptions with Gemini<br><br>Wix writes a title like "Page Name \| My Site" for every page, product and blog post,<br>and leaves the meta description empty. Google then invents its own snippet.<br><br>This workflow finds every item with no real SEO tags, writes a title and meta<br>description from the page's own content, and saves them back. It fills gaps only, so<br>anything you wrote by hand stays untouched.<br><br>### How it works<br><br>1. Reads the SEO tags on your pages, blog posts and store products.<br>2. Keeps only the items Wix reports as having no custom tags, so no AI quota is spent finding them.<br>3. Pulls real product copy, options and prices for product pages.<br>4. Sends the batch to Gemini once, using a strict JSON schema.<br>5. Checks every suggestion for length, duplicates, emoji and shouting.<br>6. Saves the survivors, then reports what changed and what was rejected.<br><br>### Setup steps<br><br>- [ ] Create a Wix API key with View SEO Settings, Manage SEO Settings and Read Products<br>- [ ] Add it as Header Auth named `Authorization` on the three Wix HTTP nodes<br>- [ ] Add your Gemini key as Header Auth named `x-goog-api-key` on the Gemini node only<br>- [ ] Paste your Wix site ID into Set Wix Configuration<br>- [ ] Run once with `dry_run` true, read the report, then set it to false<br><br>### Customization<br><br>Set Wix Configuration holds the item types, batch size and character limits. To change<br>the tone of the copy, edit the numbered rules in Create Gemini API Request.<br><br>**Note:** page and blog tags save to the site draft, so publish in the Wix editor to<br>take them live. Store products go live straight away. |
| Set Wix Configuration | set | Stores global workflow settings and credentials configuration. | When Monday at 9am | Parse Item Types | ## Schedule and settings<br><br>Runs weekly, then sets the Wix site, item types, brand name, batch size, character limits and dry-run mode that every later step reads. |
| Parse Item Types | code | Splits item types string into an array of individual items. | Set Wix Configuration | Get Wix SEO Tags | ## Read the SEO tags from Wix<br><br>Expands the configured item types into one row each, fetches the current SEO tags per type, and turns a bad response into a readable error instead of a silent failure. |
| Get Wix SEO Tags | httpRequest | Fetches current SEO tags from the Wix API per item type. | Parse Item Types | Validate Wix SEO Response | ## Read the SEO tags from Wix<br><br>Expands the configured item types into one row each, fetches the current SEO tags per type, and turns a bad response into a readable error instead of a silent failure. |
| Validate Wix SEO Response | code | Validates API responses and intercepts authentication or catalog errors. | Get Wix SEO Tags | Identify Missing Tags | ## Read the SEO tags from Wix<br><br>Expands the configured item types into one row each, fetches the current SEO tags per type, and turns a bad response into a readable error instead of a silent failure. |
| Identify Missing Tags | code | Filters out items with manual overrides or noindex directives. | Validate Wix SEO Response | If Tags Missing | ## Find the gaps<br><br>Keeps items Wix reports as having no custom tags, skipping anything noindex or already written by hand. Wix does this filtering, so no AI quota is spent finding them. When nothing needs doing, the run explains why instead of returning an empty result. |
| If Tags Missing | if | Branches workflow execution depending on whether actionable items exist. | Identify Missing Tags | Prepare Product ID Lookup, Log Unchanged Status | ## Find the gaps<br><br>Keeps items Wix reports as having no custom tags, skipping anything noindex or already written by hand. Wix does this filtering, so no AI quota is spent finding them. When nothing needs doing, the run explains why instead of returning an empty result. |
| Log Unchanged Status | code | Compiles an explanation report when no items require updates. | If Tags Missing | None | ## Find the gaps<br><br>Keeps items Wix reports as having no custom tags, skipping anything noindex or already written by hand. Wix does this filtering, so no AI quota is spent finding them. When nothing needs doing, the run explains why instead of returning an empty result. |
| Prepare Product ID Lookup | code | Prepares search parameters targeting store product items. | If Tags Missing | Post Product Search | ## Add the product copy<br><br>Pulls real descriptions, info sections, options and price range from the Wix Stores catalog so product pages get specific copy instead of a guess from the page name. |
| Post Product Search | httpRequest | Queries the Wix Stores API for product content and specifications. | Prepare Product ID Lookup | Combine Product Data | ## Add the product copy<br><br>Pulls real descriptions, info sections, options and price range from the Wix Stores catalog so product pages get specific copy instead of a guess from the page name. |
| Combine Product Data | code | Flattens Ricos rich text content and merges product context. | Post Product Search | Create Gemini API Request | ## Add the product copy<br><br>Pulls real descriptions, info sections, options and price range from the Wix Stores catalog so product pages get specific copy instead of a guess from the page name. |
| Create Gemini API Request | code | Formulates prompt instructions and JSON schema for Gemini. | Combine Product Data | Post to Gemini API | ## Write and check the copy<br><br>One Gemini call covers the whole batch. Every suggestion is then checked for length, duplicates, emoji and shouting, and anything that fails is dropped with a reason. |
| Post to Gemini API | httpRequest | Submits batch items to Google Gemini for content generation. | Create Gemini API Request | Validate Generated Tags | ## Write and check the copy<br><br>One Gemini call covers the whole batch. Every suggestion is then checked for length, duplicates, emoji and shouting, and anything that fails is dropped with a reason. |
| Validate Generated Tags | code | Evaluates AI responses against formatting, length, and duplicate criteria. | Post to Gemini API | Check Dry Run Status | ## Write and check the copy<br><br>One Gemini call covers the whole batch. Every suggestion is then checked for length, duplicates, emoji and shouting, and anything that fails is dropped with a reason. |
| Check Dry Run Status | if | Directs flow based on dry-run configuration setting. | Validate Generated Tags | Log Changes Reported, Construct Write Requests | ## Save back to Wix<br><br>Dry run stops here and reports. Otherwise each item is patched individually and the real HTTP status of every write is reported, because the Wix bulk endpoint returns successes it never performed. |
| Construct Write Requests | code | Prepares PATCH payloads while preserving existing unmanaged tags. | Check Dry Run Status | Patch Tags on Wix | ## Save back to Wix<br><br>Dry run stops here and reports. Otherwise each item is patched individually and the real HTTP status of every write is reported, because the Wix bulk endpoint returns successes it never performed. |
| Patch Tags on Wix | httpRequest | Executes individual PATCH requests to update tags in Wix. | Construct Write Requests | Log Changes Reported | ## Save back to Wix<br><br>Dry run stops here and reports. Otherwise each item is patched individually and the real HTTP status of every write is reported, because the Wix bulk endpoint returns successes it never performed. |
| Log Changes Reported | code | Compiles the final execution report with success and failure metrics. | Patch Tags on Wix, Check Dry Run Status | None | ## Save back to Wix<br><br>Dry run stops here and reports. Otherwise each item is patched individually and the real HTTP status of every write is reported, because the Wix bulk endpoint returns successes it never performed. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`).
   - Configure rule: Interval set to weekly, trigger on Monday at 09:00.
   - Name the node: `When Monday at 9am`.

2. **Configure Global Settings:**
   - Add a **Set** (Edit Fields) node (`n8n-nodes-base.set`). Name it `Set Wix Configuration`.
   - Add manual assignments:
     - `wix_site_id` (String): `PASTE_YOUR_WIX_SITE_ID_HERE`
     - `item_types` (String): `STATIC_PAGE,BLOG_POST,STORES_PRODUCT`
     - `brand_name` (String): `Your Brand`
     - `dry_run` (Boolean): `true`
     - `max_items_per_run` (Number): `20`
     - `title_max_chars` (Number): `60`
     - `desc_min_chars` (Number): `120`
     - `desc_max_chars` (Number): `158`
     - `gemini_model` (String): `gemini-3.6-flash`
     - `skip_noindex` (Boolean): `true`
     - `publish_drafted_types` (Boolean): `false`
   - Connect `When Monday at 9am` to `Set Wix Configuration`.

3. **Parse Item Types:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Parse Item Types`.
   - Set mode to `Run Once for All Items`.
   - Paste JavaScript code:
     ```javascript
     const cfg = $input.first().json;
     return String(cfg.item_types || '')
       .split(',').map(s => s.trim().toUpperCase()).filter(Boolean)
       .map(t => ({ json: { itemType: t } }));
     ```
   - Connect `Set Wix Configuration` to `Parse Item Types`.

4. **Retrieve Wix SEO Tags:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Get Wix SEO Tags`.
   - Configuration:
     - Method: `GET`
     - URL: `https://www.wixapis.com/seo-metatags-server/v1/item-seo-tags/{{ $json.itemType }}`
     - Authentication: Generic Credential Type -> Header Auth (`Authorization`)
     - Headers: Add parameter `wix-site-id` with value `={{ $('Set Wix Configuration').first().json.wix_site_id }}`
     - Options -> Response -> Response: Set to never error out and return full response.
     - Retry on Fail: Enabled (3 retries, 3000ms wait).
   - Connect `Parse Item Types` to `Get Wix SEO Tags`.

5. **Validate Wix SEO Response:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate Wix SEO Response`.
   - Set mode to `Run Once for All Items`.
   - Paste JavaScript code to check status codes (401, 403, 428, 200) and throw explicit validation errors.
   - Connect `Get Wix SEO Tags` to `Validate Wix SEO Response`.

6. **Identify Missing Tags (Gap Analysis):**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Identify Missing Tags`.
   - Set mode to `Run Once for All Items`.
   - Implement filtering logic to check `noindex`, existing manual overrides (`hasOverride`), character lengths, and bucket limits based on `max_items_per_run`.
   - Connect `Validate Wix SEO Response` to `Identify Missing Tags`.

7. **Branch Based on Missing Tags:**
   - Add an **If** node (`n8n-nodes-base.if`). Name it `If Tags Missing`.
   - Condition: Check if `={{ $json.itemId }}` is not empty.
   - Connect `Identify Missing Tags` to `If Tags Missing`.
   - **False Branch:** Connect to a **Code** node named `Log Unchanged Status` (`n8n-nodes-base.code`) to log reasons why no changes were required.
   - **True Branch:** Connect to the product lookup sequence.

8. **Prepare and Execute Product Lookup:**
   - Add a **Code** node named `Prepare Product ID Lookup` (`n8n-nodes-base.code`). Filter item IDs where `itemType === 'STORES_PRODUCT'` and prepare a search query payload.
   - Connect true branch of `If Tags Missing` to `Prepare Product ID Lookup`.
   - Add an **HTTP Request** node named `Post Product Search` (`n8n-nodes-base.httpRequest`).
     - Configuration: POST to `https://www.wixapis.com/stores/v3/products/search`. Use Header Auth (`Authorization`) and `wix-site-id` header. Set error handling to continue regular output.
   - Connect `Prepare Product ID Lookup` to `Post Product Search`.

9. **Combine Product Data:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Combine Product Data`.
   - Set mode to `Run Once for All Items`.
   - Implement recursive flat-text extraction for Ricos rich content nodes (description and info sections) and append them to item context.
   - Connect `Post Product Search` to `Combine Product Data`.

10. **Create Gemini API Request:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Create Gemini API Request`.
    - Set mode to `Run Once for All Items`.
    - Construct the batched prompt adhering to structural rules (character counts, formatting constraints) and configure `generationConfig` with `responseMimeType: "application/json"` and strict `responseSchema`.
    - Connect `Combine Product Data` to `Create Gemini API Request`.

11. **Post to Gemini API:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Post to Gemini API`.
    - Configuration:
      - Method: `POST`
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/{{ $('Set Wix Configuration').first().json.gemini_model }}:generateContent`
      - Authentication: Generic Credential Type -> Header Auth (`x-goog-api-key`)
      - Send Body: JSON body (`={{ JSON.stringify($json) }}`)
      - Retry on Fail: Enabled (5 retries, 5000ms wait).
    - Connect `Create Gemini API Request` to `Post to Gemini API`.

12. **Validate Generated Tags:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate Generated Tags`.
    - Set mode to `Run Once for All Items`.
    - Parse Gemini JSON output, verify against length rules, filter out emojis, exclamation marks, all-caps strings, and duplicate entries, partitioning results into `accepted` and `rejected` arrays.
    - Connect `Post to Gemini API` to `Validate Generated Tags`.

13. **Check Dry Run Status & Construct Writes:**
    - Add an **If** node (`n8n-nodes-base.if`). Name it `Check Dry Run Status`.
    - Condition: Check if `={{ $('Set Wix Configuration').first().json.dry_run }}` is true.
    - **True Branch:** Connect directly to `Log Changes Reported`.
    - **False Branch:** Connect to a **Code** node named `Construct Write Requests` (`n8n-nodes-base.code`) which constructs single-item PATCH payloads, merges existing unmanaged tags, and handles publishing parameters.

14. **Patch Tags on Wix:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Patch Tags on Wix`.
    - Configuration:
      - Method: `PATCH`
      - URL: `={{ $json.url }}`
      - Authentication: Generic Credential Type -> Header Auth (`Authorization`)
      - Headers: Add parameter `wix-site-id` with value `={{ $('Set Wix Configuration').first().json.wix_site_id }}`
      - Send Body: JSON body (`={{ JSON.stringify($json.body) }}`)
      - Options -> Response -> Response: Set to never error out and return full response.
      - Retry on Fail: Enabled (3 retries, 3000ms wait).
    - Connect `Construct Write Requests` to `Patch Tags on Wix`.

15. **Log Changes Reported:**
    - Add a **Code** node (`n8n-nodes-base.code`). Name it `Log Changes Reported`.
    - Set mode to `Run Once for All Items`.
    - Aggregate write execution results, success counts, API errors, and rejections into a comprehensive audit report.
    - Connect both `Check Dry Run Status` (true branch) and `Patch Tags on Wix` to `Log Changes Reported`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Create a Wix API key with permissions for **View SEO Settings**, **Manage SEO Settings**, and **Read Products**. Add it as Header Auth named `Authorization` with no Bearer prefix. | Wix API Authentication Setup |
| Set your Wix site ID in the configuration node and attach it via the `wix-site-id` header on Wix HTTP nodes. | Wix Site Identification |
| Create a Google Gemini API key and add it as Header Auth named `x-goog-api-key` on the Gemini HTTP request node. | Google Gemini Authentication Setup |
| Run the workflow once with `dry_run` set to `true` to audit proposed changes before performing live writes. | Execution Safety & Testing |
| Static page and blog post tags are written to the site draft. Publish the site in the Wix editor to take changes live (store product updates publish immediately). | Wix Publishing Behavior |