Create AI research briefs and LinkedIn/newsletter drafts with Gemini and RSS

https://n8nworkflows.xyz/workflows/create-ai-research-briefs-and-linkedin-newsletter-drafts-with-gemini-and-rss-19682


# Create AI research briefs and LinkedIn/newsletter drafts with Gemini and RSS

### 1. Workflow Overview

This workflow is a comprehensive, human-in-the-loop intelligence pipeline designed to automate the collection, filtering, ranking, and drafting of research and content briefs from RSS and Atom feeds. Its primary target use case is editorial teams, AI researchers, and automation agencies seeking to monitor industry trends (e.g., TechCrunch and The Verge) based on specific keywords, generate structured research summaries via Google Gemini, and produce ready-to-review social media (LinkedIn) and newsletter drafts without automated publishing risks.

The execution logic is structured into the following distinct functional blocks:
- **1.1 Input Reception & Validation:** Triggers execution manually, defines configuration parameters (feeds, keywords, lookback windows, AI toggles, and demo scenarios), and validates inputs against security and structural rules.
- **1.2 Feed Ingestion & Parsing:** Iterates through configured feeds, handles switching between live HTTPS requests and mock data (demo mode), parses RSS/Atom XML structures, and records failed requests gracefully.
- **1.3 Evidence Selection & Ranking:** Deduplicates incoming stories, excludes previously seen URLs, filters by recency and keyword matches, calculates a relevance score, and generates a standardized source register.
- **1.4 AI Research Briefing:** Checks if AI is enabled, routes to mock or live Google Gemini models to generate a structured research briefing, and validates source-ID citations.
- **1.5 Content Drafting & Validation:** Generates LinkedIn and newsletter copy via Google Gemini based on the validated briefing, enforces length limits and strict citation rules, and gates outputs based on verification success.
- **1.6 Review Bundle Rendering & Export:** Consolidates metadata, review issues, markdown summaries, source registers, and CSV data into a final exportable JSON package.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Initializes the workflow execution, injects core configuration parameters, and validates structural integrity, host allowlists, and sanitization parameters.
- **Nodes Involved:** `Run Manually`, `Settings`, `Validate Configuration`
- **Node Details:**
  - **Run Manually** (`n8n-nodes-base.manualTrigger`)
    - *Type/Role:* Trigger node. Starts execution upon manual operator interaction.
    - *Config:* Default properties (no parameters required).
    - *Connections:* Input: None | Output: `Settings`.
    - *Edge Cases:* None.
  - **Settings** (`n8n-nodes-base.set`)
    - *Type/Role:* Parameter definition node. Stores raw configuration JSON containing operational modes, feeds, keywords, lookback hours, and known URLs.
    - *Config:* Mode: `raw`, JSON output defines parameters including `mode` (`live`/`demo`), `demoScenario`, `enableAI`, `topic`, `audience`, `keywords`, `lookbackHours`, `maxStories`, `allowedFeedHosts`, and `feeds`.
    - *Connections:* Input: `Run Manually` | Output: `Validate Configuration`.
    - *Edge Cases:* Misconfigured JSON structures or unescaped strings.
  - **Validate Configuration** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript execution node. Validates configuration parameters, cleans text fields, applies canonical URL parsing, and enforces domain allowlists.
    - *Config:* Custom JS validating numerical ranges, keyword lengths, feed count limits (1–6), and ensuring feeds use HTTPS matching `allowedFeedHosts`.
    - *Expressions:* Uses `$input.first().json`.
    - *Connections:* Input: `Settings` | Output: `Expand Feeds`.
    - *Edge Cases:* Throws explicit runtime errors if parameters violate schema limits (e.g., `lookbackHours` outside 1–720).

#### 2.2 Feed Ingestion & Parsing
- **Overview:** Iterates through configured RSS/Atom feeds one by one, fetches XML data over HTTP (or generates mock XML in demo mode), parses the XML payload into JSON, extracts article entries, and tracks feed retrieval failures.
- **Nodes Involved:** `Expand Feeds`, `Process Feeds`, `Demo Feed Mode`, `Demo Feed XML`, `Fetch Feed`, `Normalize HTTP Feed`, `Prepare XML`, `Valid XML Input`, `Parse XML`, `Extract RSS or Atom`, `Record Feed Failure`
- **Node Details:**
  - **Expand Feeds** (`n8n-nodes-base.code`)
    - *Type/Role:* Data mapping node. Splits the feed array from the configuration into individual items for loop processing.
    - *Config:* JavaScript mapping array elements.
    - *Connections:* Input: `Validate Configuration` | Output: `Process Feeds`.
    - *Edge Cases:* Empty feed arrays resulting in zero iteration runs.
  - **Process Feeds** (`n8n-nodes-base.splitInBatches`)
    - *Type/Role:* Flow control node (Loop). Iterates over feed items one at a time.
    - *Config:* Default batch settings.
    - *Connections:* Input: `Expand Feeds`, `Extract RSS or Atom`, `Record Feed Failure` | Output: `Rank Relevant Sources` (when loop completes), `Demo Feed Mode` (per iteration).
    - *Edge Cases:* Infinite loops if reset conditions are bypassed.
  - **Demo Feed Mode** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional routing node. Determines whether to fetch live web data or use simulated demo feeds.
    - *Config:* Condition: `={{ $json.config.mode === 'demo' }}` equals `true`.
    - *Connections:* Input: `Process Feeds` | Output: True branch to `Demo Feed XML`, False branch to `Fetch Feed`.
    - *Edge Cases:* Unhandled mode strings falling back unexpectedly.
  - **Demo Feed XML** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Generates mock RSS/Atom XML payloads based on selected demo scenarios (`normal`, `empty`, `feed-failure`, etc.).
    - *Config:* Custom JavaScript generating XML strings.
    - *Connections:* Input: `Demo Feed Mode` | Output: `Prepare XML`.
    - *Edge Cases:* Malformed XML output if scenario parameters mismatch.
  - **Fetch Feed** (`n8n-nodes-base.httpRequest`)
    - *Type/Role:* HTTP Request node. Fetches remote RSS/Atom XML content over HTTPS.
    - *Config:* URL: `={{ $json.feed.url }}`, Timeout: `30000ms`, Response Format: Text (assigned to property `xml`), Redirects disabled (`followRedirects: false`), Retry on fail enabled (3 tries, 2000ms delay), Error handling set to continue regular output.
    - *Connections:* Input: `Demo Feed Mode` | Output: `Normalize HTTP Feed`.
    - *Edge Cases:* DNS resolution failures, SSL handshake errors, HTTP 404/500 responses, or timeouts.
  - **Normalize HTTP Feed** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Merges HTTP response text with feed context and flags request errors.
    - *Config:* Custom JS combining payload properties.
    - *Connections:* Input: `Fetch Feed` | Output: `Prepare XML`.
    - *Edge Cases:* Missing payload properties on failed HTTP requests.
  - **Prepare XML** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript validation node. Inspects XML string length, scans for dangerous directives (`<!DOCTYPE`, `<!ENTITY`), and validates fetch status.
    - *Config:* Custom JS validation returning `fetchOk` boolean and `failureReason`.
    - *Connections:* Input: `Demo Feed XML`, `Normalize HTTP Feed` | Output: `Valid XML Input`.
    - *Edge Cases:* Oversized XML payloads (>1,000,000 characters).
  - **Valid XML Input** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional branch node. Checks whether `fetchOk` is true.
    - *Config:* Condition: `={{ $json.fetchOk }}` equals `true`.
    - *Connections:* Input: `Prepare XML` | Output: True branch to `Parse XML`, False branch to `Record Feed Failure`.
    - *Edge Cases:* False positives on corrupted XML.
  - **Parse XML** (`n8n-nodes-base.xml`)
    - *Type/Role:* XML parser node. Converts raw RSS/Atom XML string into a structured JavaScript object.
    - *Config:* Data Property: `xml`, Options: Explicit root enabled, explicit array disabled, attribute key `$` and character key `_`. Error handling set to continue regular output.
    - *Connections:* Input: `Valid XML Input` | Output: `Extract RSS or Atom`.
    - *Edge Cases:* Parsing errors on malformed XML documents.
  - **Extract RSS or Atom** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Extracts articles, titles, canonical URLs, publication dates, and excerpts from parsed RSS (`channel.item`) or Atom (`feed.entry`) structures.
    - *Config:* Custom JS extraction logic enforcing extraction limits (`maxPerFeed`).
    - *Connections:* Input: `Parse XML` | Output: `Process Feeds` (loops back for next feed).
    - *Edge Cases:* Unrecognized feed schemas resulting in empty article lists.
  - **Record Feed Failure** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Records feed failure reasons into a structured log object.
    - *Config:* Custom JS building failure arrays.
    - *Connections:* Input: `Valid XML Input` | Output: `Process Feeds` (loops back for next feed).
    - *Edge Cases:* None.

#### 2.3 Evidence Selection & Ranking
- **Overview:** Aggregates all parsed feed outputs, deduplicates articles by canonical URL, filters out known URLs and stale/undated articles, scores remaining entries against keyword relevance, and limits the output to top stories.
- **Nodes Involved:** `Rank Relevant Sources`, `Has Relevant Sources`
- **Node Details:**
  - **Rank Relevant Sources** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Executes deduplication, date/lookback boundary checks, keyword matching, scoring algorithms, and assigns source identifiers (`S001`, `S002`, etc.).
    - *Config:* Custom JS accessing `$('Validate Configuration').first().json` and aggregating all input items.
    - *Connections:* Input: `Process Feeds` | Output: `Has Relevant Sources`.
    - *Edge Cases:* Zero matching stories resulting in empty `sources` arrays.
  - **Has Relevant Sources** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional branch node. Verifies if valid sources were found.
    - *Config:* Condition: `={{ $json.hasSources }}` equals `true`.
    - *Connections:* Input: `Rank Relevant Sources` | Output: True branch to `Demo Brief Mode`, False branch to `No Stories Result`.
    - *Edge Cases:* Boolean evaluation failures on malformed JSON packages.

#### 2.4 AI Research Briefing
- **Overview:** Evaluates whether AI processing is enabled and whether demo mode is active; invokes Google Gemini to synthesize a structured research briefing citing source IDs, and validates the returned citations against available sources.
- **Nodes Involved:** `Demo Brief Mode`, `AI Enabled`, `Demo Brief Response`, `Create Research Brief`, `Validate Brief Citations`, `Brief Citation Gate`
- **Node Details:**
  - **Demo Brief Mode** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional branch node. Checks if operating in demo mode for briefings.
    - *Config:* Condition: `={{ $json.config.mode === 'demo' }}` equals `true`.
    - *Connections:* Input: `Has Relevant Sources` | Output: True branch to `Demo Brief Response`, False branch to `AI Enabled`.
    - *Edge Cases:* None.
  - **AI Enabled** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional branch node. Checks if `enableAI` is set to true.
    - *Config:* Condition: `={{ $json.config.enableAI }}` equals `true`.
    - *Connections:* Input: `Demo Brief Mode` | Output: True branch to `Create Research Brief`, False branch to `Evidence Only Result`.
    - *Edge Cases:* Boolean type mismatches.
  - **Demo Brief Response** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Simulates a Google Gemini API response structure containing a demo research briefing.
    - *Config:* Custom JS building JSON response strings.
    - *Connections:* Input: `Demo Brief Mode` | Output: `Validate Brief Citations`.
    - *Edge Cases:* Simulation scenario mismatches (e.g., `invalid-brief`).
  - **Create Research Brief** (`@n8n/n8n-nodes-langchain.googleGemini`)
    - *Type/Role:* Google Gemini LLM node. Generates an original research briefing JSON object from provided feed excerpts.
    - *Config:* Model: `models/gemini-3.5-flash-lite`, Operation: `message`, Resource: `text`, Temperature: `0.4`, Max Output Tokens: `8192`, System Message: Strict instructions restricting generation to supplied evidence without inventing facts, JSON output enabled. Retry on fail enabled (3 attempts).
    - *Credentials Required:* Google Gemini API.
    - *Connections:* Input: `AI Enabled` | Output: `Validate Brief Citations`.
    - *Edge Cases:* Rate limits, API authentication errors, malformed JSON responses, or refusal triggers.
  - **Validate Brief Citations** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript validation node. Parses LLM output and validates headline lengths, finding counts, summary limits, uncertainty notes, and ensuring source IDs match available sources.
    - *Config:* Custom JS validation functions.
    - *Connections:* Input: `Create Research Brief`, `Demo Brief Response` | Output: `Brief Citation Gate`.
    - *Edge Cases:* Syntax errors in LLM JSON output.
  - **Brief Citation Gate** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional gate node. Evaluates `briefValid` boolean status.
    - *Config:* Condition: `={{ $json.briefValid }}` equals `true`.
    - *Connections:* Input: `Validate Brief Citations` | Output: True branch to `Demo Content Mode`, False branch to `Needs Editorial Review`.
    - *Edge Cases:* Undefined validation flags.

#### 2.5 Content Drafting & Validation
- **Overview:** Generates LinkedIn and newsletter copy via Google Gemini based on the verified research briefing, validates citation accuracy and character length constraints, and routes content through quality gates.
- **Nodes Involved:** `Demo Content Mode`, `Demo Content Response`, `Draft Content Pack`, `Validate Content Citations`, `Content Citation Gate`, `Ready for Human Review`, `Needs Editorial Review`, `Evidence Only Result`, `No Stories Result`
- **Node Details:**
  - **Demo Content Mode** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional branch node. Checks if operating in demo mode for content drafting.
    - *Config:* Condition: `={{ $json.config.mode === 'demo' }}` equals `true`.
    - *Connections:* Input: `Brief Citation Gate` | Output: True branch to `Demo Content Response`, False branch to `Draft Content Pack`.
    - *Edge Cases:* None.
  - **Demo Content Response** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Simulates Google Gemini responses for LinkedIn and newsletter drafts.
    - *Config:* Custom JS generating mock JSON packages.
    - *Connections:* Input: `Demo Content Mode` | Output: `Validate Content Citations`.
    - *Edge Cases:* Scenario overrides (`invalid-content`).
  - **Draft Content Pack** (`@n8n/n8n-nodes-langchain.googleGemini`)
    - *Type/Role:* Google Gemini LLM node. Drafts LinkedIn (max 2500 chars) and newsletter (max 7000 chars) copy targeting the specified audience.
    - *Config:* Model: `models/gemini-3.5-flash`, Operation: `message`, Resource: `text`, Temperature: `0.4`, Max Output Tokens: `8192`, System Message: Strict instructions restricting claims to supplied brief and sources without URLs, JSON output enabled. Retry on fail enabled (3 attempts).
    - *Credentials Required:* Google Gemini API.
    - *Connections:* Input: `Demo Content Mode` | Output: `Validate Content Citations`.
    - *Edge Cases:* LLM exceeding character limits or hallucinating URLs.
  - **Validate Content Citations** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript validation node. Verifies draft text lengths, validates source ID citations against the source register, and checks for prohibited URLs in draft text.
    - *Config:* Custom JS validation logic.
    - *Connections:* Input: `Draft Content Pack`, `Demo Content Response` | Output: `Content Citation Gate`.
    - *Edge Cases:* Malformed JSON structures.
  - **Content Citation Gate** (`n8n-nodes-base.if`)
    - *Type/Role:* Conditional gate node. Evaluates `contentValid` boolean status.
    - *Config:* Condition: `={{ $json.contentValid }}` equals `true`.
    - *Connections:* Input: `Validate Content Citations` | Output: True branch to `Ready for Human Review`, False branch to `Needs Editorial Review`.
    - *Edge Cases:* Unhandled validation exceptions.
  - **Ready for Human Review** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Assigns status `'review_ready'` to successfully validated content packages.
    - *Config:* Custom JS object property assignment.
    - *Connections:* Input: `Content Citation Gate` | Output: `Render Review Bundle`.
    - *Edge Cases:* None.
  - **Needs Editorial Review** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Assigns status `'needs_review'` when validation failures occur in briefing or content checks.
    - *Config:* Custom JS object property assignment.
    - *Connections:* Input: `Brief Citation Gate`, `Content Citation Gate` | Output: `Render Review Bundle`.
    - *Edge Cases:* None.
  - **Evidence Only Result** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Assigns status `'evidence_only'` when AI is disabled.
    - *Config:* Custom JS setting review reasons and status.
    - *Connections:* Input: `AI Enabled` | Output: `Render Review Bundle`.
    - *Edge Cases:* None.
  - **No Stories Result** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Assigns status `'no_stories'` when zero articles match criteria.
    - *Config:* Custom JS setting fallback review reasons.
    - *Connections:* Input: `Has Relevant Sources` | Output: `Render Review Bundle`.
    - *Edge Cases:* None.

#### 2.6 Review Bundle Rendering & Export
- **Overview:** Formats the final review package into structured Markdown and CSV formats, packages metadata, and converts the payload into a downloadable JSON file.
- **Nodes Involved:** `Render Review Bundle`, `Download Review Bundle`
- - **Node Details:**
  - **Render Review Bundle** (`n8n-nodes-base.code`)
    - *Type/Role:* JavaScript code node. Compiles topic headers, statuses, review issues, research briefings, uncertainty notes, content drafts, source registers, feed failures, and converts the source list into escaped CSV format.
    - *Config:* Custom JS string concatenation and CSV generation.
    - *Connections:* Input: `Ready for Human Review`, `Needs Editorial Review`, `Evidence Only Result`, `No Stories Result` | Output: `Download Review Bundle`.
    - *Edge Cases:* Character escaping issues in CSV/Markdown generators.
  - **Download Review Bundle** (`n8n-nodes-base.convertToFile`)
    - *Type/Role:* File conversion node. Exports the bundle object into a downloadable JSON file (`research-content-bundle.json`).
    - *Config:* Operation: `toJson`, Options: File name `research-content-bundle.json`, format enabled.
    - *Connections:* Input: `Render Review Bundle` | Output: None (Terminal node).
    - *Edge Cases:* Large payloads exceeding system memory thresholds.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Manually | n8n-nodes-base.manualTrigger | Trigger workflow execution | None | Settings | Workflow overview<br>Step 1 - Configure the run |
| Settings | n8n-nodes-base.set | Define runtime configuration | Run Manually | Validate Configuration | Workflow overview<br>Step 1 - Configure the run |
| Validate Configuration | n8n-nodes-base.code | Validate and sanitize settings | Settings | Expand Feeds | Step 1 - Configure the run |
| Expand Feeds | n8n-nodes-base.code | Map feeds array for iteration | Validate Configuration | Process Feeds | Step 2 - Collect and parse feeds |
| Process Feeds | n8n-nodes-base.splitInBatches | Iterate through feeds sequentially | Expand Feeds, Extract RSS or Atom, Record Feed Failure | Rank Relevant Sources, Demo Feed Mode | Step 2 - Collect and parse feeds |
| Demo Feed Mode | n8n-nodes-base.if | Route between demo and live mode | Process Feeds | Demo Feed XML, Fetch Feed | Step 2 - Collect and parse feeds |
| Demo Feed XML | n8n-nodes-base.code | Generate mock XML for testing | Demo Feed Mode | Prepare XML | Step 2 - Collect and parse feeds |
| Fetch Feed | n8n-nodes-base.httpRequest | Fetch RSS/Atom feed over HTTPS | Demo Feed Mode | Normalize HTTP Feed | Step 2 - Collect and parse feeds |
| Normalize HTTP Feed | n8n-nodes-base.code | Normalize HTTP feed response | Fetch Feed | Prepare XML | Step 2 - Collect and parse feeds |
| Prepare XML | n8n-nodes-base.code | Validate XML size and safety | Demo Feed XML, Normalize HTTP Feed | Valid XML Input | Step 2 - Collect and parse feeds |
| Valid XML Input | n8n-nodes-base.if | Check if feed fetch succeeded | Prepare XML | Parse XML, Record Feed Failure | Step 2 - Collect and parse feeds |
| Parse XML | n8n-nodes-base.xml | Parse XML string to object | Valid XML Input | Extract RSS or Atom | Step 2 - Collect and parse feeds |
| Extract RSS or Atom | n8n-nodes-base.code | Extract articles from parsed feed | Parse XML | Process Feeds | Step 2 - Collect and parse feeds |
| Record Feed Failure | n8n-nodes-base.code | Log failed feed fetches | Valid XML Input | Process Feeds | Step 2 - Collect and parse feeds |
| Rank Relevant Sources | n8n-nodes-base.code | Deduplicate, filter, and score stories | Process Feeds | Has Relevant Sources | Step 3 - Select evidence and create brief |
| Has Relevant Sources | n8n-nodes-base.if | Check if valid sources exist | Rank Relevant Sources | Demo Brief Mode, No Stories Result | Step 3 - Select evidence and create brief |
| Demo Brief Mode | n8n-nodes-base.if | Check if demo mode for briefing | Has Relevant Sources | Demo Brief Response, AI Enabled | Step 3 - Select evidence and create brief |
| AI Enabled | n8n-nodes-base.if | Check if AI generation is enabled | Demo Brief Mode | Create Research Brief, Evidence Only Result | Step 3 - Select evidence and create brief |
| Demo Brief Response | n8n-nodes-base.code | Simulate AI briefing response | Demo Brief Mode | Validate Brief Citations | Step 3 - Select evidence and create brief |
| Create Research Brief | @n8n/n8n-nodes-langchain.googleGemini | Generate research brief via Gemini | AI Enabled | Validate Brief Citations | Step 3 - Select evidence and create brief |
| Validate Brief Citations | n8n-nodes-base.code | Validate brief structure and citations | Create Research Brief, Demo Brief Response | Brief Citation Gate | Step 3 - Select evidence and create brief |
| Brief Citation Gate | n8n-nodes-base.if | Gate execution on valid brief | Validate Brief Citations | Demo Content Mode, Needs Editorial Review | Step 3 - Select evidence and create brief |
| Demo Content Mode | n8n-nodes-base.if | Check if demo mode for content | Brief Citation Gate | Demo Content Response, Draft Content Pack | Step 4 - Draft and validate content |
| Demo Content Response | n8n-nodes-base.code | Simulate AI content response | Demo Content Mode | Validate Content Citations | Step 4 - Draft and validate content |
| Draft Content Pack | @n8n/n8n-nodes-langchain.googleGemini | Draft LinkedIn/newsletter copy | Demo Content Mode | Validate Content Citations | Step 4 - Draft and validate content |
| Validate Content Citations | n8n-nodes-base.code | Validate content drafts and citations | Draft Content Pack, Demo Content Response | Content Citation Gate | Step 4 - Draft and validate content |
| Content Citation Gate | n8n-nodes-base.if | Gate execution on valid content | Validate Content Citations | Ready for Human Review, Needs Editorial Review | Step 4 - Draft and validate content |
| Ready for Human Review | n8n-nodes-base.code | Set status to review ready | Content Citation Gate | Render Review Bundle | Step 5 - Review and download |
| Needs Editorial Review | n8n-nodes-base.code | Set status to needs review | Brief Citation Gate, Content Citation Gate | Render Review Bundle | Step 5 - Review and download |
| Evidence Only Result | n8n-nodes-base.code | Set status for disabled AI | AI Enabled | Render Review Bundle | Step 5 - Review and download |
| No Stories Result | n8n-nodes-base.code | Set status for zero articles | Has Relevant Sources | Render Review Bundle | Step 5 - Review and download |
| Render Review Bundle | n8n-nodes-base.code | Compile Markdown, CSV, and JSON bundle | Ready for Human Review, Needs Editorial Review, Evidence Only Result, No Stories Result | Download Review Bundle | Step 5 - Review and download |
| Download Review Bundle | n8n-nodes-base.convertToFile | Export bundle as downloadable JSON file | Render Review Bundle | None | Step 5 - Review and download |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to recreate the workflow manually in n8n:

1. **Create Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run Manually`.

2. **Add Configuration Node:**
   - Create a **Set** node (`n8n-nodes-base.set`) named `Settings`.
   - Set mode to `raw` and populate the JSON output with your desired configuration (`mode`, `demoScenario`, `enableAI`, `topic`, `audience`, `keywords`, `lookbackHours`, `maxStories`, `allowedFeedHosts`, and `feeds`).
   - Connect `Run Manually` to `Settings`.

3. **Add Configuration Validation:**
   - Create a **Code** node (`n8n-nodes-base.code`) named `Validate Configuration`. Paste the configuration validation and sanitization script provided in the source workflow.
   - Connect `Settings` to `Validate Configuration`.

4. **Add Feed Expansion and Looping Logic:**
   - Create a **Code** node named `Expand Feeds` to map the `feeds` array into individual items. Connect `Validate Configuration` to `Expand Feeds`.
   - Create a **Split In Batches** node named `Process Feeds`. Connect `Expand Feeds` to `Process Feeds`.

5. **Add Mode Switching & Fetching Branch:**
   - Create an **If** node named `Demo Feed Mode` with condition `{{ $json.config.mode === 'demo' }}` equals `true`. Connect `Process Feeds` to `Demo Feed Mode`.
   - *True branch:* Create a **Code** node named `Demo Feed XML` to generate mock RSS XML, connecting to `Prepare XML`.
   - *False branch:* Create an **HTTP Request** node named `Fetch Feed` (`url: "={{ $json.feed.url }}"`, method GET, response format text assigned to `xml`, timeout `30000ms`, redirects disabled, retry on fail enabled). Connect to `Normalize HTTP Feed`.
   - Create a **Code** node named `Normalize HTTP Feed` to format HTTP responses, connecting to `Prepare XML`.

6. **Add XML Validation & Parsing:**
   - Create a **Code** node named `Prepare XML` to validate payload length and security directives. Connect `Demo Feed XML` and `Normalize HTTP Feed` to `Prepare XML`.
   - Create an **If** node named `Valid XML Input` with condition `={{ $json.fetchOk }}` equals `true`. Connect `Prepare XML` to `Valid XML Input`.
   - *True branch:* Create an **XML** node named `Parse XML` (data property `xml`, explicit root enabled, explicit array disabled). Connect to `Extract RSS or Atom`.
   - Create a **Code** node named `Extract RSS or Atom` to extract article objects, and loop it back to `Process Feeds`.
   - *False branch:* Create a **Code** node named `Record Feed Failure` to log failures, and loop it back to `Process Feeds`.

7. **Add Evidence Ranking & Filtering:**
   - When `Process Feeds` completes its loop, connect its completion output to a **Code** node named `Rank Relevant Sources` (aggregating all input items and matching against `Validate Configuration`).
   - Create an **If** node named `Has Relevant Sources` with condition `={{ $json.hasSources }}` equals `true`. Connect `Rank Relevant Sources` to `Has Relevant Sources`.
   - *False branch:* Create a **Code** node named `No Stories Result` setting status `'no_stories'`, connecting to `Render Review Bundle`.

8. **Add AI Briefing Branch:**
   - *True branch:* Create an **If** node named `Demo Brief Mode` with condition `={{ $json.config.mode === 'demo' }}` equals `true`. Connect `Has Relevant Sources` to `Demo Brief Mode`.
   - *True branch (Demo):* Create a **Code** node named `Demo Brief Response`, connecting to `Validate Brief Citations`.
   - *False branch (Live):* Create an **If** node named `AI Enabled` with condition `={{ $json.config.enableAI }}` equals `true`. Connect `Demo Brief Mode` to `AI Enabled`.
   - *False branch (AI Disabled):* Create a **Code** node named `Evidence Only Result` setting status `'evidence_only'`, connecting to `Render Review Bundle`.
   - *True branch (AI Enabled):* Create a **Google Gemini** node named `Create Research Brief` (`models/gemini-3.5-flash-lite`, text operation, JSON output enabled, temperature `0.4`). Configure Google Gemini API credentials. Connect to `Validate Brief Citations`.
   - Create a **Code** node named `Validate Brief Citations` to inspect brief structure and citation IDs. Connect to `Brief Citation Gate` (**If** node evaluating `briefValid === true`).
   - *False branch:* Create a **Code** node named `Needs Editorial Review` setting status `'needs_review'`, connecting to `Render Review Bundle`.

9. **Add Content Drafting & Validation Branch:**
   - *True branch:* Create an **If** node named `Demo Content Mode` with condition `={{ $json.config.mode === 'demo' }}` equals `true`. Connect `Brief Citation Gate` to `Demo Content Mode`.
   - *True branch (Demo):* Create a **Code** node named `Demo Content Response`, connecting to `Validate Content Citations`.
   - *False branch (Live):* Create a **Google Gemini** node named `Draft Content Pack` (`models/gemini-3.5-flash`, text operation, JSON output enabled, temperature `0.4`). Configure Google Gemini API credentials. Connect to `Validate Content Citations`.
   - Create a **Code** node named `Validate Content Citations` to verify character limits and citation validity. Connect to `Content Citation Gate` (**If** node evaluating `contentValid === true`).
   - *False branch:* Connect to `Needs Editorial Review`.
   - *True branch:* Create a **Code** node named `Ready for Human Review` setting status `'review_ready'`, connecting to `Render Review Bundle`.

10. **Add Bundle Rendering & Export:**
    - Connect `Ready for Human Review`, `Needs Editorial Review`, `Evidence Only Result`, and `No Stories Result` into a **Code** node named `Render Review Bundle` to format Markdown and CSV outputs.
    - Create a **Convert to File** node named `Download Review Bundle` (`toJson` operation, file name `research-content-bundle.json`). Connect `Render Review Bundle` to `Download Review Bundle`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow Disclaimer | The text and logic originate exclusively from an automated n8n workflow respecting content safety policies. |
| Operational Requirement | Human review is strictly required before publishing any AI-generated drafts or briefing materials. |