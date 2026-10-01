Audit website SEO with a multi-agent Anthropic Claude spider

https://n8nworkflows.xyz/workflows/audit-website-seo-with-a-multi-agent-anthropic-claude-spider-20009


# Audit website SEO with a multi-agent Anthropic Claude spider

### 1. Workflow Overview

This workflow automates a multi-agent SEO audit by crawling a website starting from a specified URL, extracting technical and on-page SEO signals, evaluating those signals using six specialized Anthropic Claude AI agents, and compiling a structured JSON report. 

The process is organized into the following logical blocks:
- **1.1 Input Reception & Configuration:** Receives the audit request via webhook or manual testing, parses the target URL and limits, and assigns core evaluation thresholds.
- **1.2 Crawl Control & Queue Management:** Iteratively pulls URLs from the crawl queue, tracking visited pages and enforcing maximum page and depth constraints.
- **1.3 Page Retrieval & Parsing:** Applies a configurable delay, fetches page HTML via HTTP, parses key SEO elements (title, metadata, headings, links, word count, schema), and updates the queue with newly discovered internal links.
- **1.4 AI Agent Evaluation & Synthesis:** Dispatches the parsed page data concurrently to six specialized Anthropic Claude agents (Technical, Intent, Entity, Internal Link, Content Quality, and Architecture), combines their outputs, and appends the synthesized page diagnosis to the crawl state before looping back.
- **1.5 Final Reporting:** Aggregates all page diagnoses into a site-wide summary report and returns the final JSON payload via webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the audit workflow from an external POST request or manual trigger, extracts target parameters, and sets up configuration variables for crawling and analysis.
- **Nodes Involved:** 
  - `When Crawl Requested`
  - `Manual Test Trigger`
  - `Parse Crawl Request`
  - `Set Crawl Configuration`

- **Node Details:**
  - **When Crawl Requested (`n8n-nodes-base.webhook`)**
    - *Technical Role:* Webhook trigger listening for incoming POST requests.
    - *Configuration:* Path set to `seo-spider-crawl-request`, responds via a separate node.
    - *Input/Output:* Output connects to `Parse Crawl Request`.
    - *Failure Types:* Invalid HTTP method or incorrect payload format.
  - **Manual Test Trigger (`n8n-nodes-base.manualTrigger`)**
    - *Technical Role:* Manual execution trigger for testing.
    - *Input/Output:* Output connects to `Parse Crawl Request`.
  - **Parse Crawl Request (`n8n-nodes-base.code`)**
    - *Technical Role:* JavaScript code node extracting target URL, host, and crawl limits (`maxPages`, `maxDepth`) with safe defaults.
    - *Variables Used:* `$input.first().json.body`
    - *Input/Output:* Inputs from triggers; output connects to `Set Crawl Configuration`.
    - *Failure Types:* Malformed URL throwing URI parsing errors.
  - **Set Crawl Configuration (`n8n-nodes-base.set`)**
    - *Technical Role:* Assigns static thresholds and metadata variables for the spider.
    - *Configuration Choices:* Sets `crawlDelaySeconds` (3), `thinContentWordThreshold` (300), `titleMaxLength` (60), `metaDescriptionMaxLength` (160), `anthropicModel` (`claude-sonnet-4-6`), and `userAgent`.
    - *Input/Output:* Input from `Parse Crawl Request`; output connects to `Retrieve URL from Queue`.

---

#### 2.2 Crawl Control & Queue Management
- **Overview:** Manages the URL queue iteratively, ensuring the crawler respects stop conditions such as empty queues or maximum page limits.
- **Nodes Involved:**
  - `Retrieve URL from Queue`
  - `Check Queue Status`

- **Node Details:**
  - **Retrieve URL from Queue (`n8n-nodes-base.code`)**
    - *Technical Role:* Pulls the next URL and depth level from the active queue array.
    - *Variables Used:* `item.queue`
    - *Input/Output:* Inputs from configuration or `Synthesize SEO Report` (loop-back); output connects to `Check Queue Status`.
  - **Check Queue Status (`n8n-nodes-base.if`)**
    - *Technical Role:* Evaluates whether the crawl should proceed or terminate.
    - *Configuration Choices:* Branches if `hasNext` is false or if `pagesCrawled` is greater than or equal to `maxPages`.
    - *Input/Output:* Input from queue retrieval; outputs connect to `Build Final SEO Report` (true path) or `Wait for Crawl Delay` (false path).

---

#### 2.3 Page Retrieval & Parsing
- **Overview:** Fetches target HTML content responsibly with rate limiting and extracts structured on-page SEO signals and internal links.
- **Nodes Involved:**
  - `Wait for Crawl Delay`
  - `Fetch Page HTML Content`
  - `Parse HTML Content`
  - `Update Crawl Queue`

- **Node Details:**
  - **Wait for Crawl Delay (`n8n-nodes-base.wait`)**
    - *Technical Role:* Enforces a polite delay between HTTP requests.
    - *Configuration Choices:* 3 seconds delay.
    - *Input/Output:* Input from queue check; output connects to `Fetch Page HTML Content`.
  - **Fetch Page HTML Content (`n8n-nodes-base.httpRequest`)**
    - *Technical Role:* HTTP GET request to download target page HTML.
    - *Configuration Choices:* Response format set to text, passes custom User-Agent header, configured with `continueOnFail: true`.
    - *Input/Output:* Input from wait node; output connects to `Parse HTML Content`.
    - *Failure Types:* DNS lookup failure, timeout, or HTTP 4xx/5xx errors.
  - **Parse HTML Content (`n8n-nodes-base.code`)**
    - *Technical Role:* JavaScript code node performing regular expression parsing to extract title, meta description, canonical, robots meta, H1/H2 headings, schema detection, image statistics, word count, and segregated internal/external links.
    - *Input/Output:* Input from HTTP request; output connects to `Update Crawl Queue`.
  - **Update Crawl Queue (`n8n-nodes-base.code`)**
    - *Technical Role:* Appends newly discovered internal links to the crawl queue while respecting maximum depth limits and avoiding duplicate visits.
    - *Input/Output:* Input from HTML parsing; outputs fan out to all six Anthropic agent HTTP request nodes.

---

#### 2.4 AI Agent Evaluation & Synthesis
- **Overview:** Dispatches extracted page data to six specialist Claude agents concurrently, parses their JSON responses, and synthesizes a combined page diagnosis.
- **Nodes Involved:**
  - `Post to Technical Agent API`
  - `Post to Intent Agent API`
  - `Post to Entity Agent API`
  - `Post to Internal Link API`
  - `Post to Content Quality API`
  - `Post to Architecture API`
  - `Combine Agent Outputs`
  - `Synthesize SEO Report`

- **Node Details:**
  - **Anthropic HTTP Request Nodes (`n8n-nodes-base.httpRequest`)** *(Technical, Intent, Entity, Internal Link, Content Quality, Architecture)*
    - *Technical Role:* Sends tailored prompts and page metrics to the Anthropic Messages API (`https://api.anthropic.com/v1/messages`).
    - *Configuration Choices:* POST method, JSON body generation using `JSON.stringify()`, max tokens set to 700, custom system prompts per agent specialty, generic HTTP Header Authentication for API key, and retry logic (`maxTries: 2`, `waitBetweenTries: 3000`, `continueErrorOutput`).
    - *Input/Output:* Inputs from `Update Crawl Queue`; outputs connect to specific input indexes (0 through 5) on `Combine Agent Outputs`.
    - *Failure Types:* API rate limits (HTTP 429), authentication errors (HTTP 401), invalid JSON response formatting from LLM.
  - **Combine Agent Outputs (`n8n-nodes-base.merge`)**
    - *Technical Role:* Merges the parallel responses from all six agent API calls by position.
    - *Configuration Choices:* Mode: Combine by position across 6 inputs.
    - *Input/Output:* Inputs from 6 agent API nodes; output connects to `Synthesize SEO Report`.
  - **Synthesize SEO Report (`n8n-nodes-base.code`)**
    - *Technical Role:* Parses individual agent responses, calculates overall and agent-specific scores, sorts issues by severity (`high`, `medium`, `low`), and pushes the page diagnosis into the accumulated results array before looping back to the queue.
    - *Input/Output:* Input from merge node; output connects back to `Retrieve URL from Queue`.

---

#### 2.5 Final Reporting
- **Overview:** Compiles accumulated page diagnoses into a site-wide executive report containing average scores, top recurring issues, and worst-performing pages.
- **Nodes Involved:**
  - `Build Final SEO Report`
  - `Return SEO Report via Webhook`

- **Node Details:**
  - **Build Final SEO Report (`n8n-nodes-base.code`)**
    - *Technical Role:* JavaScript code node aggregating statistics across all crawled pages.
    - *Input/Output:* Input from `Check Queue Status` (true branch); output connects to `Return SEO Report via Webhook`.
  - **Return SEO Report via Webhook (`n8n-nodes-base.respondToWebhook`)**
    - *Technical Role:* Responds to the original webhook request with the complete JSON summary report.
    - *Configuration Choices:* Respond with JSON.
    - *Input/Output:* Input from report builder node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration overview. | None | None | ## SEO Spider That Thinks<br><br>### How it works<br><br>This workflow receives an SEO crawl request via webhook or manual test trigger, parses the request, applies crawl configuration, and iterates through a URL queue until the crawl is complete or a limit is reached. For each page, it waits according to the crawl delay, fetches the HTML, parses SEO-relevant page data, updates the crawl queue, and sends the page findings to multiple Claude-based specialist agents. The agent outputs are merged into a synthesized SEO diagnosis per page, then the workflow loops back for the next URL before building and returning a final SEO report.<br><br>### Setup steps<br><br>- Configure the webhook URL and expected request payload for the starting URL, crawl limits, and any crawl options used by the parse request code.<br>- Add Anthropic API credentials or headers for each Claude HTTP Request node, including the API key, model, version header, and message payload format.<br>- Review the Set - Config node values for crawl delay, thin content threshold, title length limit, and meta description length limit.<br>- Ensure the custom Code nodes are configured for your site rules, queue structure, HTML parsing expectations, and final report format.<br>- Use the Manual Trigger node for test runs before enabling the webhook in production.<br><br>### Customization<br><br>Adjust crawl limits, delay, SEO thresholds, Claude prompts, and the specialist agent mix to match the depth of audit required. The workflow can also be extended to store reports in a database, send alerts, or export results to a document or spreadsheet. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Entry cluster documentation. | None | None | ## Receive crawl request<br><br>Entry cluster that starts the workflow from either a webhook or manual trigger, parses the incoming crawl request, and applies the base crawl configuration. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Crawl control documentation. | None | None | ## Control crawl queue<br><br>Selects the next URL to crawl and decides whether the crawl should continue or branch to final report generation when the queue is empty or limits are reached. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Page fetching and parsing documentation. | None | None | ## Fetch and parse page<br><br>Applies the crawl delay, fetches the current page HTML, extracts page data, and updates the crawl queue with discovered URLs before analysis begins. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Core SEO agents documentation. | None | None | ## Core SEO agents<br><br>Runs the upper Claude agent cluster for technical SEO, search intent, and entity analysis on the parsed page data. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Content structure agents documentation. | None | None | ## Content structure agents<br><br>Runs the lower Claude agent cluster for internal linking, content quality, and site architecture analysis. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Page diagnosis synthesis documentation. | None | None | ## Synthesize page diagnosis<br><br>Combines all specialist agent responses and creates a consolidated SEO diagnosis for the current page, then loops back to process the next URL. |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Final report return documentation. | None | None | ## Return final report<br><br>Builds the completed crawl-level SEO report and sends it back through the webhook response path. |
| When Crawl Requested | `n8n-nodes-base.webhook` | Webhook trigger for incoming crawl requests. | None | Parse Crawl Request | Receive crawl request |
| Manual Test Trigger | `n8n-nodes-base.manualTrigger` | Manual test execution trigger. | None | Parse Crawl Request | Receive crawl request |
| Parse Crawl Request | `n8n-nodes-base.code` | Parses initial payload and initializes state variables. | When Crawl Requested, Manual Test Trigger | Set Crawl Configuration | Receive crawl request |
| Set Crawl Configuration | `n8n-nodes-base.set` | Sets crawl constants, thresholds, and AI model parameters. | Parse Crawl Request | Retrieve URL from Queue | Receive crawl request |
| Retrieve URL from Queue | `n8n-nodes-base.code` | Selects the next URL from the crawl queue. | Set Crawl Configuration, Synthesize SEO Report | Check Queue Status | Control crawl queue |
| Check Queue Status | `n8n-nodes-base.if` | Evaluates stop conditions (empty queue or max pages). | Retrieve URL from Queue | Build Final SEO Report, Wait for Crawl Delay | Control crawl queue |
| Wait for Crawl Delay | `n8n-nodes-base.wait` | Implements crawl delay rate-limiting. | Check Queue Status | Fetch Page HTML Content | Fetch and parse page |
| Fetch Page HTML Content | `n8n-nodes-base.httpRequest` | Downloads HTML content of the target URL. | Wait for Crawl Delay | Parse HTML Content | Fetch and parse page |
| Parse HTML Content | `n8n-nodes-base.code` | Extracts on-page SEO signals and links from HTML. | Fetch Page HTML Content | Update Crawl Queue | Fetch and parse page |
| Update Crawl Queue | `n8n-nodes-base.code` | Updates visited list and enqueues new internal links. | Parse HTML Content | Post to Technical Agent API, Post to Intent Agent API, Post to Entity Agent API, Post to Internal Link API, Post to Content Quality API, Post to Architecture API | Fetch and parse page |
| Post to Technical Agent API | `n8n-nodes-base.httpRequest` | Calls Claude for technical SEO analysis. | Update Crawl Queue | Combine Agent Outputs (Input 0) | Core SEO agents |
| Post to Intent Agent API | `n8n-nodes-base.httpRequest` | Calls Claude for search intent analysis. | Update Crawl Queue | Combine Agent Outputs (Input 1) | Core SEO agents |
| Post to Entity Agent API | `n8n-nodes-base.httpRequest` | Calls Claude for entity and topical analysis. | Update Crawl Queue | Combine Agent Outputs (Input 2) | Core SEO agents |
| Post to Internal Link API | `n8n-nodes-base.httpRequest` | Calls Claude for internal linking strategy audit. | Update Crawl Queue | Combine Agent Outputs (Input 3) | Content structure agents |
| Post to Content Quality API | `n8n-nodes-base.httpRequest` | Calls Claude for content quality and word count audit. | Update Crawl Queue | Combine Agent Outputs (Input 4) | Content structure agents |
| Post to Architecture API | `n8n-nodes-base.httpRequest` | Calls Claude for site architecture and silo audit. | Update Crawl Queue | Combine Agent Outputs (Input 5) | Content structure agents |
| Combine Agent Outputs | `n8n-nodes-base.merge` | Merges the responses from all 6 AI agents. | Post to Technical Agent API, Post to Intent Agent API, Post to Entity Agent API, Post to Internal Link API, Post to Content Quality API, Post to Architecture API | Synthesize SEO Report | Synthesize page diagnosis |
| Synthesize SEO Report | `n8n-nodes-base.code` | Merges agent findings into a scored page diagnosis. | Combine Agent Outputs | Retrieve URL from Queue | Synthesize page diagnosis |
| Build Final SEO Report | `n8n-nodes-base.code` | Aggregates all page diagnoses into a site-wide summary. | Check Queue Status | Return SEO Report via Webhook | Return final report |
| Return SEO Report via Webhook | `n8n-nodes-base.respondToWebhook` | Returns the JSON report to the webhook caller. | Build Final SEO Report | None | Return final report |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Triggers:**
   - Add a **Webhook** node (`When Crawl Requested`): set HTTP method to `POST`, path to `seo-spider-crawl-request`, and response mode to `responseNode`.
   - Add a **Manual Trigger** node (`Manual Test Trigger`).

2. **Setup Request Parsing & Configuration:**
   - Add a **Code** node (`Parse Crawl Request`) connected to both triggers. Use JavaScript to extract `startUrl`, `maxPages`, and `maxDepth` from the incoming body.
   - Add a **Set** node (`Set Crawl Configuration`) to define variables: `crawlDelaySeconds` (3), `thinContentWordThreshold` (300), `titleMaxLength` (60), `metaDescriptionMaxLength` (160), `anthropicModel` (`claude-sonnet-4-6`), and `userAgent`.

3. **Build Queue Control Logic:**
   - Add a **Code** node (`Retrieve URL from Queue`) to pop the next URL from the queue array.
   - Add an **If** node (`Check Queue Status`) with conditions checking if `hasNext` is false or `pagesCrawled >= maxPages`. Connect the true branch to the final reporting cluster and the false branch to the crawl delay.

4. **Build Fetch & Parse Pipeline:**
   - Add a **Wait** node (`Wait for Crawl Delay`) set to 3 seconds.
   - Add an **HTTP Request** node (`Fetch Page HTML Content`) to fetch `{{ $json.currentUrl }}` with the custom User-Agent header, enabling `continueOnFail`.
   - Add a **Code** node (`Parse HTML Content`) to extract title, meta description, headings, word count, schema status, images, and links.
   - Add a **Code** node (`Update Crawl Queue`) to update the visited list and enqueue valid internal links within `maxDepth`.

5. **Deploy the 6 Specialist AI Agents:**
   - Create six **HTTP Request** nodes (`Post to Technical Agent API`, `Post to Intent Agent API`, `Post to Entity Agent API`, `Post to Internal Link API`, `Post to Content Quality API`, `Post to Architecture API`) pointing to `https://api.anthropic.com/v1/messages`.
   - Configure each with generic HTTP Header Authentication for Anthropic, adding headers `anthropic-version: 2023-06-01` and `content-type: application/json`.
   - Configure JSON request bodies with appropriate system prompts requesting strict JSON output containing `score`, `issues` (with `severity`), and `summary`. Enable retry on fail.

6. **Synthesize & Loop:**
   - Add a **Merge** node (`Combine Agent Outputs`) set to combine 6 inputs by position.
   - Add a **Code** node (`Synthesize SEO Report`) to parse agent outputs, calculate overall scores, and append the page diagnosis to the results array. Connect its output back to `Retrieve URL from Queue` to establish the loop.

7. **Build Reporting Cluster:**
   - Connect the true branch of `Check Queue Status` to a **Code** node (`Build Final SEO Report`) that calculates average scores, top recurring issues, and worst-performing pages.
   - Connect the report builder to a **Respond to Webhook** node (`Return SEO Report via Webhook`) responding with JSON.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SEO Spider That Thinks | Overview of workflow architecture and custom multi-agent audit pattern. |