Generate structured research reports with OpenAI, Wikipedia, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/generate-structured-research-reports-with-openai--wikipedia--google-sheets--and-slack-19710


# Generate structured research reports with OpenAI, Wikipedia, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates end-to-end web research, content extraction, AI analysis, and structured reporting based on any incoming research topic. It serves as an automated research assistant suitable for market research, technical summaries, and newsletter content drafting.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Validation:** Captures the research request via webhook, assigns variables, and validates that a topic has been provided.
- **1.2 Search Query Generation:** Leverages OpenAI to formulate focused Wikipedia search queries covering various perspectives (overview, trends, benefits, challenges, future outlook).
- **1.3 Web Search & Content Extraction:** Executes web searches on Wikipedia, iterates through results, extracts raw page content, and cleans the text by stripping HTML tags and unwanted artifacts.
- **1.4 AI Analysis & Report Formatting:** Merges the gathered research data into a unified context, feeds it to OpenAI to generate a multi-section research report, and parses the output into structured JSON fields.
- **1.5 Persistence, Notification & Response:** Stores the formatted report in Google Sheets, sends a completion notification to Slack, and returns the final structured report to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Receives the raw HTTP POST request, extracts the research topic parameter, and verifies its existence before proceeding.
- **Nodes Involved:** `Webhook`, `Receive Research Topic`, `Validate Research Topic`
- **Node Details:**
  - **Webhook** (`n8n-nodes-base.webhook`)
    - *Role:* Entry point for incoming HTTP POST requests.
    - *Configuration:* Path `/research-assistant`, HTTP method `POST`, response mode set to `responseNode`. Enabled retry on fail.
    - *Input/Output:* No inputs; outputs to `Receive Research Topic`.
    - *Failure Types:* Webhook timeout if processing takes too long.
  - **Receive Research Topic** (`n8n-nodes-base.set`)
    - *Role:* Assigns the incoming webhook payload property to a standardized variable.
    - *Configuration:* Creates an assignment `topic` with value `={{ $json.body.topic }}`.
    - *Input/Output:* Input from `Webhook`; output to `Validate Research Topic`.
    - *Failure Types:* Expression evaluation failure if `body.topic` is missing.
  - **Validate Research Topic** (`n8n-nodes-base.if`)
    - *Role:* Ensures the research topic is not empty.
    - *Configuration:* Evaluates `={{ $json.topic }}` using a strict type validation, requiring the string to be `notEmpty`.
    - *Input/Output:* Input from `Receive Research Topic`; output (true branch) to `z`.
    - *Failure Types:* Halts execution if the topic is empty (false branch is unhandled).

#### 2.2 Search Query Generation
- **Overview:** Uses an OpenAI language model to transform a single research topic into five specialized search queries.
- **Nodes Involved:** `z`, `Parse Search Queries`
- **Node Details:**
  - **z** (`@n8n/n8n-nodes-langchain.openAi`)
    - *Role:* Generates structured search queries via an LLM.
    - *Configuration:* Uses model `gpt-4o-mini` with a prompt instructing the model to output a JSON array of 5 queries covering general overview, latest developments, benefits, challenges, and future trends.
    - *Credentials:* OpenAI (PAID) (AI-ML Team).
    - *Input/Output:* Input from `Validate Research Topic`; output to `Parse Search Queries`.
    - *Failure Types:* API rate limits, authentication errors, or malformed LLM JSON output.
  - **Parse Search Queries** (`n8n-nodes-base.code`)
    - *Role:* Cleans the LLM text output and parses it into individual items.
    - *Configuration:* JavaScript code strips markdown code blocks (` ```json `), parses the JSON string, and maps each query into a separate n8n item.
    - *Input/Output:* Input from `z`; output to `Search Web`.
    - *Failure Types:* Syntax errors if the LLM output is not valid JSON.

#### 2.3 Web Search & Content Extraction
- **Overview:** Searches Wikipedia using the generated queries, loops through individual search hits, extracts plaintext article content, and cleans the data.
- **Nodes Involved:** `Search Web`, `Prepare Search Results`, `Process Search Results`, `Extract Website Content`, `Clean Website Content`
- **Node Details:**
  - **Search Web** (`n8n-nodes-base.httpRequest`)
    - *Role:* Queries the Wikipedia API for matching pages.
    - *Configuration:* GET request to `https://en.wikipedia.org/w/api.php` with query parameters: `action=query`, `list=search`, `srsearch={{ $json.query }}`, `format=json`, `srlimit=5`.
    - *Input/Output:* Input from `Parse Search Queries`; output to `Prepare Search Results`.
    - *Failure Types:* Network timeouts, DNS resolution failures, or API schema changes.
  - **Prepare Search Results** (`n8n-nodes-base.code`)
    - *Role:* Flattens and formats raw Wikipedia search response arrays.
    - *Configuration:* JavaScript extracts search items, constructs Wikipedia URLs (`https://en.wikipedia.org/wiki/...`), strips HTML tags from snippets, and retains page IDs and timestamps.
    - *Input/Output:* Input from `Search Web`; output to `Process Search Results`.
    - *Failure Types:* Empty search results array handling.
  - **Process Search Results** (`n8n-nodes-base.splitInBatches`)
    - *Role:* Iterates through search results sequentially.
    - *Configuration:* Standard batch processing loop node.
    - *Input/Output:* Input from `Prepare Search Results`; output loops to `Extract Website Content`.
  - **Extract Website Content** (`n8n-nodes-base.httpRequest`)
    - *Role:* Fetches the raw plaintext extract for a given Wikipedia title.
    - *Configuration:* GET request to `https://en.wikipedia.org/w/api.php` with query parameters: `action=query`, `prop=extracts`, `explaintext=true`, `titles={{ $json.title }}`, `format=json`.
    - *Input/Output:* Input from `Process Search Results`; output to `Clean Website Content`.
    - *Failure Types:* Invalid page titles or rate-limiting by Wikipedia.
  - **Clean Website Content** (`n8n-nodes-base.code`)
    - *Role:* Normalizes and structures the extracted webpage content alongside original search metadata.
    - *Configuration:* JavaScript extracts page content extracts from the API response and pairs them with titles, URLs, snippets, and page IDs.
    - *Input/Output:* Input from `Extract Website Content`; output to `Merge Research Data`.
    - *Failure Types:* Missing page extract properties.

#### 2.4 AI Analysis & Report Formatting
- **Overview:** Combines all processed research documents into a single analysis context, runs AI synthesis to generate a comprehensive report, and parses sections into structured JSON attributes.
- **Nodes Involved:** `Merge Research Data`, `AI Research Engine`, `Research Report Formatter`
- **Node Details:**
  - **Merge Research Data** (`n8n-nodes-base.merge`)
    - *Role:* Consolidates individual research snippets into a unified dataset.
    - *Configuration:* Default merge settings.
    - *Input/Output:* Input from `Clean Website Content`; output to `AI Research Engine`.
  - **AI Research Engine** (`@n8n/n8n-nodes-langchain.openAi`)
    - *Role:* Synthesizes gathered research into a structured report.
    - *Configuration:* Uses model `gpt-4o-mini`. The prompt requests an 8-section report: Executive Summary, Key Findings, Important Facts, Benefits, Challenges, Latest Trends, Future Outlook, and Source References, restricted strictly to the provided research content.
    - *Credentials:* OpenAI (PAID) (AI-ML Team).
    - *Input/Output:* Input from `Merge Research Data`; output to `Research Report Formatter`.
    - *Failure Types:* Token limit overflows or incomplete LLM generations.
  - **Research Report Formatter** (`n8n-nodes-base.code`)
    - *Role:* Parses the markdown report generated by the LLM into distinct property keys.
    - *Configuration:* Custom JavaScript validating that the report is non-empty, then using Regular Expressions to extract individual sections based on markdown headings (`### Executive Summary`, etc.) and returning a structured JSON payload with a timestamp (`generatedAt`).
    - *Input/Output:* Input from `AI Research Engine`; output to `Update row in sheet`.
    - *Failure Types:* Throws explicit error `'AI research report is empty.'` if the report output is missing; regex mismatch if headings deviate from expected format.

#### 2.5 Persistence, Notification & Response
- **Overview:** Archives the report data in a Google Sheets spreadsheet, sends a notification message to a Slack channel, and returns the final report payload to the webhook client.
- **Nodes Involved:** `Update row in sheet`, `Send a message`, `Return Research Response`
- **Node Details:**
  - **Update row in sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends or updates research report rows in a Google Sheet.
    - *Configuration:* Operation `appendOrUpdate`, matching on column `Topic`. Mapped columns: `Topic` (`={{ $json.topic }}`), `Date` (`={{ $json.generatedAt }}`), `Summary` (`={{ $json.executiveSummary }}`), `Result` (`={{ $json.report }}`). Document ID `160CLK9fYwpdcmch14-30N5hMXsf93GIS35mS90KLGzU`, Sheet ID `258575867`.
    - *Input/Output:* Input from `Research Report Formatter`; output to `Send a message`.
    - *Failure Types:* Google Sheets API authentication failure, quota limits, or schema mismatches (missing columns).
  - **Send a message** (`n8n-nodes-base.slack`)
    - *Role:* Dispatches a completion notification to a Slack channel.
    - *Configuration:* Sends text: `AI Personal Research Assistant\n\nResearch completed successfully.\n\nTopic:\n{{ $('Research Report Formatter').item.json.topic }}\n\nThe final research report has been generated and stored successfully.` to channel ID `C0ATZFFKRUH` (`n8n-general`).
    - *Credentials:* Slack credential configuration required.
    - *Input/Output:* Input from `Update row in sheet`; output to `Return Research Response`.
    - *Failure Types:* Slack API errors, invalid token scopes, or non-existent channel IDs.
  - **Return Research Response** (`n8n-nodes-base.respondToWebhook`)
    - *Role:* Returns the execution result back to the original HTTP webhook caller.
    - *Configuration:* Standard webhook response node.
    - *Input/Output:* Input from `Send a message`; output loop back to `Process Search Results` (note: loop return edge in metadata).
    - *Failure Types:* Connection drop before response transmission.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook` | `n8n-nodes-base.webhook` | Receives a research topic from the user/application | None | `Receive Research Topic` | Research Request Trigger |
| `Receive Research Topic` | `n8n-nodes-base.set` | Validates and formats the research request | `Webhook` | `Validate Research Topic` | Input Validation Layer |
| `Validate Research Topic` | `n8n-nodes-base.if` | Validates and formats the research request | `Receive Research Topic` | `z` | Input Validation Layer |
| `z` | `@n8n/n8n-nodes-langchain.openAi` | Uses AI to convert the research topic into several useful search queries | `Validate Research Topic` | `Parse Search Queries` | Generate Search Queries |
| `Parse Search Queries` | `n8n-nodes-base.code` | Process queries | `z` | `Search Web` | Process queries |
| `Search Web` | `n8n-nodes-base.httpRequest` | Searches Google or use search API through the standard HTTP Request node. | `Parse Search Queries` | `Prepare Search Results` | Search the web |
| `Prepare Search Results` | `n8n-nodes-base.code` | Extract Search URLs | `Search Web` | `Process Search Results` | Extract Search URLs |
| `Process Search Results` | `n8n-nodes-base.splitInBatches` | Processes search results one by one. | `Prepare Search Results` | `Extract Website Content` | Loop Through Search Results |
| `Extract Website Content` | `n8n-nodes-base.httpRequest` | Fetches the actual webpage content from each search result. | `Process Search Results` | `Clean Website Content` | Extract Website Content |
| `Clean Website Content` | `n8n-nodes-base.code` | Removes unnecessary HTML, scripts, navigation text, and other unwanted content before sending it to the AI. | `Extract Website Content` | `Merge Research Data` | Clean Website Content |
| `Merge Research Data` | `n8n-nodes-base.merge` | Combines all contents into one document. | `Clean Website Content` | `AI Research Engine` | Merge Research Data |
| `AI Research Engine` | `@n8n/n8n-nodes-langchain.openAi` | Uses OpenAI to analyze all collected information. | `Merge Research Data` | `Research Report Formatter` | AI Research Analysis |
| `Research Report Formatter` | `n8n-nodes-base.code` | Formats the AI output | `AI Research Engine` | `Update row in sheet` | Generate Final Report |
| `Update row in sheet` | `n8n-nodes-base.googleSheets` | Stores the report. | `Research Report Formatter` | `Send a message` | Save Research |
| `Send a message` | `n8n-nodes-base.slack` | Sends research completion notification. | `Update row in sheet` | `Return Research Response` | Notify User |
| `Return Research Response` | `n8n-nodes-base.respondToWebhook` | Returns the final report through webhook. | `Send a message` | `Process Search Results` | Return Response |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`). Set HTTP Method to `POST`, Path to `research-assistant`, and Response Mode to `Response Node`.
2. **Setup Input Processing & Validation:**
   - Add a **Set** node (`Receive Research Topic`). Create an assignment named `topic` with expression `={{ $json.body.topic }}`. Connect `Webhook` output to this node.
   - Add an **If** node (`Validate Research Topic`). Configure a condition where `={{ $json.topic }}` is `notEmpty` (strict validation). Connect `Receive Research Topic` output to this node.
3. **Configure Search Query Generation:**
   - Add an **OpenAI Chat Model** node (`z`). Set model to `gpt-4o-mini`. Provide a system/user prompt requesting 5 focused web search queries returned strictly as a JSON array. Connect OpenAI credential (`openAiApi`). Connect the True branch of `Validate Research Topic` to this node.
   - Add a **Code** node (`Parse Search Queries`). Add JavaScript to strip markdown code blocks (` ```json `), parse the array, and return items formatted with `{ query }`. Connect `z` output to this node.
4. **Implement Web Search & Extraction:**
   - Add an **HTTP Request** node (`Search Web`). Set method to `GET`, URL to `https://en.wikipedia.org/w/api.php`, and query parameters: `action=query`, `list=search`, `srsearch={{ $json.query }}`, `format=json`, `srlimit=5`. Connect `Parse Search Queries` output to this node.
   - Add a **Code** node (`Prepare Search Results`). Add JavaScript to map search results into objects containing `title`, encoded `url`, cleaned `snippet`, `pageId`, and `timestamp`. Connect `Search Web` output to this node.
   - Add a **Split In Batches** node (`Process Search Results`). Connect `Prepare Search Results` output to this node.
   - Add an **HTTP Request** node (`Extract Website Content`). Set method to `GET`, URL to `https://en.wikipedia.org/w/api.php`, and query parameters: `action=query`, `prop=extracts`, `explaintext=true`, `titles={{ $json.title }}`, `format=json`. Connect the loop output of `Process Search Results` to this node.
   - Add a **Code** node (`Clean Website Content`). Add JavaScript to extract the page extract and merge it with original search metadata (`title`, `url`, `snippet`, `pageId`). Connect `Extract Website Content` output to this node.
5. **Set up AI Synthesis & Formatting:**
   - Add a **Merge** node (`Merge Research Data`) using default settings. Connect `Clean Website Content` output to this node.
   - Add an **OpenAI Chat Model** node (`AI Research Engine`). Set model to `gpt-4o-mini`. Add a prompt instructing the model to analyze research content and output structured sections (Executive Summary, Key Findings, Important Facts, Benefits, Challenges, Latest Trends, Future Outlook, Source References). Connect OpenAI credential (`openAiApi`). Connect `Merge Research Data` output to this node.
   - Add a **Code** node (`Research Report Formatter`). Add JavaScript to validate that the AI output is non-empty and use RegExp extraction to isolate each markdown section into dedicated JSON properties, adding a `generatedAt` ISO timestamp. Connect `AI Research Engine` output to this node.
6. **Configure Persistence, Notification & Response:**
   - Add a **Google Sheets** node (`Update row in sheet`). Configure Operation as `appendOrUpdate`, matching on column `Topic`. Set Document ID (`160CLK9fYwpdcmch14-30N5hMXsf93GIS35mS90KLGzU`) and Sheet ID (`258575867`). Map columns: `Topic` (`={{ $json.topic }}`), `Date` (`={{ $json.generatedAt }}`), `Summary` (`={{ $json.executiveSummary }}`), `Result` (`={{ $json.report }}`). Connect Google Sheets credentials. Connect `Research Report Formatter` output to this node.
   - Add a **Slack** node (`Send a message`). Configure resource as `Message`, action as `Post`, select `channel`, set target channel ID (`C0ATZFFKRUH`), and provide the completion notification text including the topic expression. Connect Slack credentials. Connect `Update row in sheet` output to this node.
   - Add a **Respond to Webhook** node (`Return Research Response`). Connect `Send a message` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Sticky Note Guide** | [n8n Sticky Notes Documentation](https://docs.n8n.io/workflows/components/sticky-notes/) |
| **WeblineIndia Services & Development Support** | [AI automation development](https://www.weblineindia.com/process-automation-solutions.html) |