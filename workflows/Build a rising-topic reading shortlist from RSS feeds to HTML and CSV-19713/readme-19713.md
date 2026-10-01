Build a rising-topic reading shortlist from RSS feeds to HTML and CSV

https://n8nworkflows.xyz/workflows/build-a-rising-topic-reading-shortlist-from-rss-feeds-to-html-and-csv-19713


# Build a rising-topic reading shortlist from RSS feeds to HTML and CSV

### 1. Workflow Overview

This workflow automates the aggregation, filtering, momentum analysis, and summarization of news from multiple public RSS/Atom feeds into a balanced reading shortlist, exportable as both an HTML brief and a CSV file. It is designed for editors, researchers, and information curators who monitor specific technology and current affairs themes across multiple publications.

The logical execution is organized into the following sequential blocks:
- **1.1 Trigger & Configuration:** Initiates execution either manually or via a scheduled cron trigger and injects a centralized configuration payload containing feed URLs, topic keywords, exclusion criteria, time windows, and analytical weights.
- **1.2 Feed Ingestion & Synchronization:** Concurrently fetches four distinct RSS/Atom feeds using fault-tolerant request settings and merges them into a unified execution stream.
- **1.3 Article Filtering & Normalization:** Validates publication dates, strips HTML tags, removes duplicate entries per publisher, filters out excluded keywords, and enforces per-feed article limits.
- **1.4 Topic Classification & Headline Grouping:** Scores and categorizes normalized articles against user-defined topics using keyword matching, then groups similar headlines into distinct story clusters based on text similarity and token analysis.
- **1.5 Momentum Analysis & Story Selection:** Compares recent coverage windows against baseline historical windows to calculate momentum, growth ratios, and signal quality warnings, then builds a prioritized, multi-publisher-favored reading shortlist.
- **1.6 Report Generation & Export:** Compiles the structured data into a self-contained HTML document and a properly escaped CSV dataset, outputting them as downloadable binary files.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
- **Overview:** Initializes the workflow execution pipeline on-demand or on a recurring weekday schedule and establishes the operational parameters and settings for subsequent analytical steps.
- **Nodes Involved:** `Manual Execution Trigger`, `Weekday schedule`, `Configure Feed Sources`

- **Node Details:**
  - **Manual Execution Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Entry point for manual user-initiated test runs.
    - *Configuration Choices:* Default manual trigger parameters.
    - *Input/Output Connections:* Output connects to `Configure Feed Sources`.
    - *Version-Specific Requirements:* Type Version 1.
    - *Edge Cases / Potential Failure Types:* None (local execution trigger).
  
  - **Weekday schedule**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` — Recurring automated trigger.
    - *Configuration Choices:* Cron expression set to run at `08:00` and `20:00` UTC, Monday through Friday (`0 0 8,20 * * 1-5`).
    - *Input/Output Connections:* Output connects to `Configure Feed Sources`.
    - *Version-Specific Requirements:* Type Version 1.2.
    - *Edge Cases / Potential Failure Types:* Requires workflow activation in n8n for cron execution.

  - **Configure Feed Sources**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Injects configuration JSON containing feeds, topics, keywords, exclusions, and algorithmic thresholds.
    - *Configuration Choices:* Raw JSON mode (`mode: "raw"`). Defines arrays for 4 RSS feeds (BBC Technology, The Guardian Technology, The Verge, Ars Technica), 5 topic categories (Artificial intelligence, Chips and devices, Cybersecurity, Space, Energy), exclusion keywords, and scoring thresholds (`freshHours: 12`, `baselineHours: 36`, `shortlistSize: 5`, etc.).
    - *Key Expressions or Variables:* Uses `{{ $now.toISO() }}` to stamp the generation timestamp.
    - *Input/Output Connections:* Inputs from `Manual Execution Trigger` and `Weekday schedule`. Outputs fan out to the 4 individual feed reader nodes.
    - *Version-Specific Requirements:* Type Version 3.4.
    - *Edge Cases / Potential Failure Types:* Malformed JSON syntax will cause downstream JavaScript validation errors.

---

#### 2.2 Feed Ingestion & Synchronization
- **Overview:** Fetches raw XML content concurrently from all configured public RSS/Atom feed URLs and waits for all requests to complete before proceeding, ensuring empty or failed feeds remain visible in downstream reports.
- **Nodes Involved:** `Fetch RSS Feed 1`, `Fetch RSS Feed 2`, `Fetch RSS Feed 3`, `Fetch RSS Feed 4`, `Merge RSS Feeds`

- **Node Details:**
  - **Fetch RSS Feed 1 / 2 / 3 / 4**
    - *Type and Technical Role:* `n8n-nodes-base.rssFeedRead` — Parses RSS/Atom XML feeds into structured JSON article objects.
    - *Configuration Choices:* URL parameter dynamically referenced from configuration. Error handling set to continue regular execution (`onError: "continueRegularOutput"`, `alwaysOutputData: true`) with automatic retries (`retryOnFail: true`, `maxTries: 2`, `waitBetweenTries: 1000`).
    - *Key Expressions or Variables:* 
      - Feed 1: `={{ $json.config.feeds[0].url }}`
      - Feed 2: `={{ $json.config.feeds[1].url }}`
      - Feed 3: `={{ $json.config.feeds[2].url }}`
      - Feed 4: `={{ $json.config.feeds[3].url }}`
    - *Input/Output Connections:* Input from `Configure Feed Sources`. Outputs connect to corresponding inputs (0 through 3) of `Merge RSS Feeds`.
    - *Version-Specific Requirements:* Type Version 1.1.
    - *Edge Cases / Potential Failure Types:* Network timeouts, DNS resolution failures, or invalid XML schemas. Handled via error continuation to prevent workflow crashes.

  - **Merge RSS Feeds**
    - *Type and Technical Role:* `n8n-nodes-base.merge` — Aggregates multi-input data streams into a synchronized combined output.
    - *Configuration Choices:* Merge Mode set to `Multiple Inputs` with 4 input streams enabled.
    - *Input/Output Connections:* Inputs 0–3 receive data from `Fetch RSS Feed 1` through `4`. Output connects to `Filter Articles by Criteria`.
    - *Version-Specific Requirements:* Type Version 3.. Requires an n8n core version supporting a 4-input merge node.
    - *Edge Cases / Potential Failure Types:* Mismatched stream arrival synchronization (mitigated by n8n's wait-for-all behavior on multi-input merge nodes).

---

#### 2.3 Article Filtering & Normalization
- **Overview:** Validates configuration integrity, parses publication dates, strips HTML formatting, discards out-of-window, future, or excluded articles, removes duplicate links per publisher, and caps maximum items per feed.
- **Nodes Involved:** `Filter Articles by Criteria`

- **Node Details:**
  - **Filter Articles by Criteria**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom JavaScript execution block for advanced data cleansing, schema validation, and normalization.
    - *Configuration Choices:* Executes custom ES6 JavaScript parsing feed items against inclusion/exclusion rules, URL canonicalization, and deduplication logic based on publisher and link hashes.
    - *Key Expressions or Variables:* Reads upstream configuration and feed outputs via `$('Configure Feed Sources').first().json` and `$('Fetch RSS Feed N').all()`.
    - *Input/Output Connections:* Input from `Merge RSS Feeds`. Output connects to `Assign Topics to Articles`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Throws explicit runtime errors if configuration parameters fail type checks, boundary validation, or array constraints.

---

#### 2.4 Topic Classification & Headline Grouping
- **Overview:** Scores and assigns each normalized article to its strongest matching topic based on keyword frequency in titles and summaries, then groups semantically similar headlines into unique story clusters.
- **Nodes Involved:** `Assign Topics to Articles`, `Group Article Headlines`

- **Node Details:**
  - **Assign Topics to Articles**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom JavaScript node performing weighted keyword matching.
    - *Configuration Choices:* Evaluates article titles (weight: 2) and summaries (weight: 1) against configured topic keyword arrays. Ties are broken by topic definition order. Unmatched articles are filtered out and counted.
    - *Input/Output Connections:* Input from `Filter Articles by Criteria`. Output connects to `Group Article Headlines`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Articles lacking any keyword matches are dropped; high volume of unmatched articles indicates overly narrow keyword configurations.

  - **Group Article Headlines**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom JavaScript clustering node.
    - *Configuration Choices:* Groups articles sharing identical canonical links/titles or high token similarity (`headlineSimilarity` threshold) within the same topic into singular story objects while preserving numeric differentiators.
    - *Input/Output Connections:* Input from `Assign Topics to Articles`. Output connects to `Analyze Topic Momentum`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Over-clustering or under-clustering if `headlineSimilarity` threshold is set inappropriately.

---

#### 2.5 Momentum Analysis & Story Selection
- **Overview:** Evaluates coverage velocity by comparing recent time windows against historical baselines, computes momentum scores, assigns status labels (Surging, Rising, Active, Cooling, Quiet), and curates a balanced reading shortlist.
- **Nodes Involved:** `Analyze Topic Momentum`, `Select Stories for Review`

- **Node Details:**
  - **Analyze Topic Momentum**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom analytical JavaScript calculation node.
    - *Configuration Choices:* Partitions article stories into fresh and baseline subsets based on `freshHours`. Calculates growth ratios, freshness scores, publisher diversity percentages, and composite momentum scores (Growth 60%, Freshness 25%, Diversity 15%). Generates signal quality warnings.
    - *Input/Output Connections:* Input from `Group Article Headlines`. Output connects to `Select Stories for Review`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Small sample sizes or limited baselines generate quality warnings (`Limited`).

  - **Select Stories for Review**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom curation and sorting node.
    - *Configuration Choices:* Filters fresh story candidates, prioritizes multi-publisher coverage, sorts by momentum and recency, and builds a curated shortlist capped by `shortlistSize` total items and `maxShortlistPerTopic` per category.
    - *Input/Output Connections:* Input from `Analyze Topic Momentum`. Output connects to `Compile Reading Brief`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Empty shortlists if no articles fall within the fresh time window.

---

#### 2.6 Report Generation & Export
- **Overview:** Transforms the curated dataset and topic metrics into a styled HTML briefing document and a properly structured CSV dataset, encoding both as downloadable binary files.
- **Nodes Involved:** `Compile Reading Brief`, `Generate HTML Report`, `Generate CSV Report`

- **Node Details:**
  - **Compile Reading Brief**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Custom template-rendering and string-formatting node.
    - *Configuration Choices:* Constructs a complete HTML string complete with embedded CSS styles, warning boxes, summary tables, feed health stats, and reading guides. Builds a comma-separated UTF-8 CSV string with formula-injection mitigation (`=`, `+`, `-`, `@` prefix neutralization).
    - *Input/Output Connections:* Input from `Select Stories for Review`. Outputs fan out to both `Generate HTML Report` and `Generate CSV Report`.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Large datasets may increase string generation memory usage.

  - **Generate HTML Report**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Binary file wrapper node.
    - *Configuration Choices:* Encodes HTML string output into base64 binary format (`mimeType: 'text/html'`, `fileName: 'momentum-brief.html'`).
    - *Input/Output Connections:* Input from `Compile Reading Brief`. Output execution data available for manual download.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* None under normal operation.

  - **Generate CSV Report**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Binary file wrapper node.
    - *Configuration Choices:* Encodes CSV string output with a UTF-8 BOM into base64 binary format (`mimeType: 'text/csv'`, `fileName: 'momentum-topics.csv'`).
    - *Input/Output Connections:* Input from `Compile Reading Brief`. Output execution data available for manual download.
    - *Version-Specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* None under normal operation.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & setup overview | None | None | ## Find rising RSS topics and build a balanced reading shortlist<br><br>### Who is it for?<br>For editors, researchers or anyone who follows the same subjects across several news feeds and wants a useful place to start reading.<br><br>### What it does<br>Reads four public RSS or Atom feeds, removes unwanted entries, assigns topics and groups similar headlines. It compares the last 12 hours with the previous 36 hours to spot changes in coverage.<br><br>The added shortlist picks up to five fresh stories. Stories covered by multiple publishers come first, with at most two picks per topic. Each pick includes source links and a short reason. This keeps one busy subject from filling the whole list.<br><br>### Setup<br>Open Configure Feed Sources (Edit Fields) and edit the JSON fields for feeds and keywords. Run the whole workflow manually. In each Download node, open Output → Binary → data and download the file. Review feed health before using the results.<br><br>### Requirements<br>An n8n version with a four-input Merge node and internet access to the feeds. No credentials, API services or extra packages.<br><br>### Customization<br>Change the keywords, time windows, shortlist size and limit per topic in Configure Feed Sources. Adjust the weekday schedule if needed. The workflow does not retain history or verify news; feed coverage may be incomplete. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Trigger documentation | None | None | ## Start and configure<br><br>Edit feed URLs, topic keywords and limits in Configure Feed Sources. Then run the whole workflow manually.<br><br>Weekday schedule runs at 08:00 and 20:00 UTC, Monday to Friday, after activation. Change the times and timezone before using it. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Ingestion documentation | None | None | ## Read and merge feeds<br><br>Read four public RSS/Atom feeds and wait for all four before processing. No credentials needed.<br><br>Empty or failed feeds remain visible in the report. Use the same publisher ID for feeds from one outlet. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Filtering documentation | None | None | ## Filter and tag articles<br><br>Remove old, undated, excluded and duplicate entries. Assign each article to its strongest keyword match. Titles count more than summaries; topic order breaks ties. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Analysis documentation | None | None | ## Analyze topic momentum<br><br>Group similar headlines and compare the latest 12 hours with the preceding 36 hours.<br><br>Rising and Surging need multiple stories and publishers. Limited means incomplete coverage. No history is stored between runs. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Selection documentation | None | None | ## Select and build brief<br><br>Pick up to 5 fresh stories, with at most 2 per topic. Groups covered by multiple publishers come first, then topic momentum, publisher count and recency.<br><br>Change shortlistSize and maxShortlistPerTopic in Configure Feed Sources. Each pick has links and a reason. Coverage does not verify the story. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Export documentation | None | None | ## Download files<br><br>Open either output node: Output → Binary → data → Download.<br><br>HTML: shortlist, links, topic table and feed health.<br><br>CSV: all topic metrics and shortlisted story counts.<br><br>Empty results still produce both files. Files stay in execution data according to your retention settings. |
| `Manual Execution Trigger` | `n8n-nodes-base.manualTrigger` | Run once after changing the settings. | None | `Configure Feed Sources` | |
| `Configure Feed Sources` | `n8n-nodes-base.set` | Edit the JSON fields here: feeds, topics, time windows and shortlist limits. | `Manual Execution Trigger`, `Weekday schedule` | `Fetch RSS Feed 1`, `Fetch RSS Feed 2`, `Fetch RSS Feed 3`, `Fetch RSS Feed 4` | |
| `Fetch RSS Feed 1` | `n8n-nodes-base.rssFeedRead` | Source 1 from Configure Feed Sources. Empty results and errors remain visible in the brief. | `Configure Feed Sources` | `Merge RSS Feeds` | |
| `Fetch RSS Feed 2` | `n8n-nodes-base.rssFeedRead` | Source 2 from Configure Feed Sources. Empty results and errors remain visible in the brief. | `Configure Feed Sources` | `Merge RSS Feeds` | |
| `Fetch RSS Feed 3` | `n8n-nodes-base.rssFeedRead` | Source 3 from Configure Feed Sources. Empty results and errors remain visible in the brief. | `Configure Feed Sources` | `Merge RSS Feeds` | |
| `Fetch RSS Feed 4` | `n8n-nodes-base.rssFeedRead` | Source 4 from Configure Feed Sources. Empty results and errors remain visible in the brief. | `Configure Feed Sources` | `Merge RSS Feeds` | |
| `Merge RSS Feeds` | `n8n-nodes-base.merge` | Wait for all four feeds before preparing the snapshot. | `Fetch RSS Feed 1`, `Fetch RSS Feed 2`, `Fetch RSS Feed 3`, `Fetch RSS Feed 4` | `Filter Articles by Criteria` | |
| `Filter Articles by Criteria` | `n8n-nodes-base.code` | Keep dated, recent articles. Remove exclusions and duplicate links from the same publisher. | `Merge RSS Feeds` | `Assign Topics to Articles` | |
| `Assign Topics to Articles` | `n8n-nodes-base.code` | Give each article its strongest matching topic. Topic order breaks ties. | `Filter Articles by Criteria` | `Group Article Headlines` | |
| `Group Article Headlines` | `n8n-nodes-base.code` | Count similar headlines as one story. Different numeric details stay separate. | `Assign Topics to Articles` | `Analyze Topic Momentum` | |
| `Analyze Topic Momentum` | `n8n-nodes-base.code` | Compare fresh coverage with the earlier window and flag weak source coverage. | `Group Article Headlines` | `Select Stories for Review` | |
| `Select Stories for Review` | `n8n-nodes-base.code` | Pick a short reading list with publisher links and a limit per topic. | `Analyze Topic Momentum` | `Compile Reading Brief` | |
| `Compile Reading Brief` | `n8n-nodes-base.code` | Build the reading shortlist, topic summary, source links and CSV. | `Select Stories for Review` | `Generate HTML Report`, `Generate CSV Report` | |
| `Generate HTML Report` | `n8n-nodes-base.code` | Open Output → Binary → data to download momentum-brief.html. | `Compile Reading Brief` | None | |
| `Generate CSV Report` | `n8n-nodes-base.code` | Open Output → Binary → data to download momentum-topics.csv. | `Compile Reading Brief` | None | |
| `Weekday schedule` | `n8n-nodes-base.scheduleTrigger` | 08:00 and 20:00 UTC, Monday to Friday. Publish or activate to enable. | None | `Configure Feed Sources` | |

---

### 4. Reproducing theWorkflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Triggers and Configuration Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`), set interval mode to `cronExpression`, and set the expression to `0 0 8,20 * * 1-5` (UTC, Mon–Fri).
   - Add an **Edit Fields (Set)** node named `Configure Feed Sources` (`n8n-nodes-base.set`). Set Mode to `Raw`, and populate the JSON payload with `config` (containing `feeds` array with 4 feed objects having `name`, `publisher`, and `url`; `topics` array with category names and keyword lists; `excludeKeywords`; and threshold parameters like `shortlistSize: 5`, `freshHours: 12`, `baselineHours: 36`, etc.) and `generatedAt: "{{ $now.toISO() }}"`.
   - Connect both `Manual Execution Trigger` and `Weekday schedule` outputs to `Configure Feed Sources`.

2. **Add Feed Readers and Merge Node:**
   - Add four **RSS Feed Read** nodes (`n8n-nodes-base.rssFeedRead`) named `Fetch RSS Feed 1` through `4`.
     - Set URL parameters to `={{ $json.config.feeds[0].url }}`, `={{ $json.config.feeds[1].url }}`, `={{ $json.config.feeds[2].url }}`, and `={{ $json.config.feeds[3].url }}` respectively.
     - Enable error handling options: `onError` to `continueRegularOutput`, enable retry on fail (`maxTries: 2`, `waitBetweenTries: 1000`), and check `Always Output Data`.
   - Connect the output of `Configure Feed Sources` to the input of all four `Fetch RSS Feed` nodes.
   - Add a **Merge** node named `Merge RSS Feeds` (`n8n-nodes-base.merge`) configured with `Number Inputs` set to `4`.
   - Connect `Fetch RSS Feed 1` through `4` outputs to inputs `0`, `1`, `2`, and `3` of `Merge RSS Feeds`.

3. **Add Processing and Filtering Code Nodes:**
   - Add a **Code** node named `Filter Articles by Criteria` (`n8n-nodes-base.code`). Paste the JavaScript logic that validates configuration constraints, parses dates, filters out excluded keywords and duplicate publisher links, and caps articles per feed. Connect `Merge RSS Feeds` output here.
   - Add a **Code** node named `Assign Topics to Articles` (`n8n-nodes-base.code`). Paste the topic-matching JavaScript logic that scores article titles and summaries against topic keywords. Connect `Filter Articles by Criteria` output here.
   - Add a **Code** node named `Group Article Headlines` (`n8n-nodes-base.code`). Paste the headline clustering JavaScript logic. Connect `Assign Topics to Articles` output here.

4. **Add Analysis, Selection, and Brief Compilation Nodes:**
   - Add a **Code** node named `Analyze Topic Momentum` (`n8n-nodes-base.code`). Paste the momentum ratio calculation and quality warning generation script. Connect `Group Article Headlines` output here.
   - Add a **Code** node named `Select Stories for Review` (`n8n-nodes-base.code`). Paste the shortlist curation and topic limiting script. Connect `Analyze Topic Momentum` output here.
   - Add a **Code** node named `Compile Reading Brief` (`n8n-nodes-base.code`). Paste the HTML template and CSV generation script. Connect `Select Stories for Review` output here.

5. **Add Binary Export Output Nodes:**
   - Add two **Code** nodes named `Generate HTML Report` and `Generate CSV Report` (`n8n-nodes-base.code`).
   - In `Generate HTML Report`, paste the base64 conversion script targeting `mimeType: 'text/html'` and `fileName: 'momentum-brief.html'`.
   - In `Generate CSV Report`, paste the base64 conversion script targeting `mimeType: 'text/csv'` and `fileName: 'momentum-topics.csv'`.
   - Connect the output of `Compile Reading Brief` to **both** export nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| No external credentials or API keys required | Workflow relies entirely on parsing public RSS/Atom endpoints over standard HTTP. |
| Execution retention and binary data access | Generated files remain accessible within n8n execution data (`Output → Binary → data`) according to instance retention policies. |