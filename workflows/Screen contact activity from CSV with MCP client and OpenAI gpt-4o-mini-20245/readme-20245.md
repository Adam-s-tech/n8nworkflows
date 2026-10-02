Screen contact activity from CSV with MCP client and OpenAI gpt-4o-mini

https://n8nworkflows.xyz/workflows/screen-contact-activity-from-csv-with-mcp-client-and-openai-gpt-4o-mini-20245


# Screen contact activity from CSV with MCP client and OpenAI gpt-4o-mini

### 1. Workflow Overview

This workflow automates the evaluation of organization contacts by processing data from an input CSV, scraping website and social media content via a Model Context Protocol (MCP) browser endpoint, analyzing the data using OpenAI (gpt-4o-mini) for activity status, and writing structured output to a results CSV.

The logical execution follows these functional blocks:
- **1.1 Input Reception & Preparation:** Manually triggers the workflow, reads the raw contacts CSV from disk, parses it, and feeds individual records into a batch processing loop.
- **1.2 MCP Browser Processing:** Dynamically generates browser navigation and evaluation steps per contact channel (website, Facebook, Instagram), executes them against an MCP server, and extracts/cleans the resulting text snapshots.
- **1.3 AI Verdict Assessment:** Constructs a strict evidence-based screening prompt, queries OpenAI to classify contact status (active/inactive/uncertain), parses the JSON response, and aggregates evidence excerpts and timestamps.
- **1.4 Output Aggregation & Safety Checks:** Validates row counts against a runaway loop safety threshold, aggregates completed contact evaluations, converts the dataset to CSV format, and saves the final report to the local file system while looping back to process subsequent contact batches.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation
- **Overview:** Initializes the workflow execution manually, ingests the contacts source file from the local filesystem, extracts tabular data, and prepares contact records for iterative batch execution.
- **Nodes Involved:** `Manual Start Trigger`, `Read CSV File`, `Extract CSV Data`, `Batch Process Contacts`.
- **Node Details:**
  - **Manual Start Trigger**
    - Type: `n8n-nodes-base.manualTrigger` (v1)
    - Technical Role: Entry point for manual workflow executions.
    - Input Connections: None. Output: `Read CSV File`.
    - Edge Cases: Requires manual operator action.
  - **Read CSV File**
    - Type: `n8n-nodes-base.readWriteFile` (v1.1)
    - Technical Role: Reads file binary data from the host filesystem.
    - Configuration: File path configured via `fileSelector` (`/path/to/input/contacts.csv`).
    - Input Connections: `Manual Start Trigger`. Output: `Extract CSV Data`.
    - Edge Cases: Missing file or permission errors on the n8n host path.
  - **Extract CSV Data**
    - Type: `n8n-nodes-base.extractFromFile` (v1.1)
    - Technical Role: Parses raw file binary data into structured JSON items.
    - Input Connections: `Read CSV File`. Output: `Batch Process Contacts`.
    - Edge Cases: Malformed CSV formatting or missing header rows.
  - **Batch Process Contacts**
    - Type: `n8n-nodes-base.splitInBatches` (v3)
    - Technical Role: Iterates through the list of contact records. Also acts as the loop completion target receiving final processed rows.
    - Input Connections: `Extract CSV Data`, `Format Output Row Data`. Output: `Check Row Count Safety` (branch 0), `Generate MCP Steps Code` (branch 1 / loop iteration).
    - Edge Cases: Infinite loops if loop termination logic fails.

#### 2.2 MCP Browser Processing
- **Overview:** Evaluates available communication URLs for each contact by generating browser automation steps, calling an MCP browser client, and parsing/cleaning textual snapshots.
- **Nodes Involved:** `Generate MCP Steps Code`, `Perform MCP Calls`, `Extract MCP Results`.
- **Node Details:**
  - **Generate MCP Steps Code**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Inspects contact URLs (`website_url`, `facebook_url`, `instagram_url`) and generates step definitions (`browser_navigate`, `browser_wait_for`, `browser_evaluate`).
    - Key Expressions: Evaluates item properties dynamically via JavaScript.
    - Input Connections: `Batch Process Contacts`. Output: `Perform MCP Calls`.
    - Edge Cases: Contacts lacking any valid URL generate a dummy `about:blank` step to maintain loop parity.
  - **Perform MCP Calls**
    - Type: `@n8n/n8n-nodes-langchain.mcpClient` (v1)
    - Technical Role: Interacts with the MCP endpoint using tool calls (`browser_navigate`, `browser_wait_for`, `browser_evaluate`).
    - Configuration: Endpoint URL configured to `http://localhost:8931/mcp`. Tool selected dynamically via `={{ $json.mcp_tool }}` and parameters via `={{ $json.mcp_params }}`. Configured with 2 retries and a 5000ms delay.
    - Input Connections: `Generate MCP Steps Code`. Output: `Extract MCP Results`.
    - Edge Cases: MCP endpoint downtime, network timeouts, or unreachable browser instances resulting in failed tool calls.
  - **Extract MCP Results**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Reconstructs channel context from original step mapping, stripping boilerplate cookies and UI noise from Facebook and Instagram captures.
    - Input Connections: `Perform MCP Calls`. Output: `Create Verdict Prompt`.
    - Edge Cases: Unexpected DOM structures breaking text regex replacement rules.

#### 2.3 AI Verdict Assessment
- **Overview:** Compiles scraped evidence into a strict prompt template, requests classification from OpenAI, and parses the structured JSON verdict.
- **Nodes Involved:** `Create Verdict Prompt`, `OpenAI Analyze Verdict`, `Parse Analysis Results`, `Format Output Row Data`.
- **Node Details:**
  - **Create Verdict Prompt**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Assembles an evidence-based screening prompt containing page text limits, current date constraints, and strict evaluation rules.
    - Input Connections: `Extract MCP Results`. Output: `OpenAI Analyze Verdict`.
    - Edge Cases: Prompt token length overruns if scraped text is overly verbose (mitigated by slice limits).
  - **OpenAI Analyze Verdict**
    - Type: `@n8n/n8n-nodes-langchain.openAi` (v2.1)
    - Technical Role: Queries the OpenAI chat model for classification.
    - Configuration: Model set to `gpt-4o-mini`. Uses OpenAI API credentials (`OpenAI TEMPLATE`).
    - Input Connections: `Create Verdict Prompt`. Output: `Parse Analysis Results`.
    - Edge Cases: API rate limits, authentication failures, or malformed provider responses.
  - **Parse Analysis Results**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Parses the raw model output string into structured JSON variables (`verdict`, `confidence`, `reasoning`, `checked_at`).
    - Input Connections: `OpenAI Analyze Verdict`. Output: `Format Output Row Data`.
    - Edge Cases: Non-JSON outputs or markdown code block wrappers from the model fall back safely to an `uncertain` verdict.
  - **Format Output Row Data**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Merges original contact properties, truncated evidence excerpts, and evaluation results into a unified output structure.
    - Input Connections: `Parse Analysis Results`. Output: `Batch Process Contacts` (loop continuation).
    - Edge Cases: Missing context variables if parent node references fail.

#### 2.4 Output Aggregation & Safety Checks
- **Overview:** Enforces safety constraints on processed row counts, converts the final dataset into CSV format, and writes the output file.
- **Nodes Involved:** `Check Row Count Safety`, `Convert to CSV File`, `Save CSV File`.
- **Node Details:**
  - **Check Row Count Safety**
    - Type: `n8n-nodes-base.code` (v2)
    - Technical Role: Validates that the number of items does not exceed a runaway threshold (`MAX_REASONABLE_ROWS = 500`).
    - Input Connections: `Batch Process Contacts` (triggered when batching completes). Output: `Convert to CSV File`.
    - Edge Cases: Throws an explicit error if item counts exceed expectations to prevent infinite loop generation.
  - **Convert to CSV File**
    - Type: `n8n-nodes-base.convertToFile` (v1.1)
    - Technical Role: Transforms aggregated JSON items into CSV file binary format.
    - Input Connections: `Check Row Count Safety`. Output: `Save CSV File`.
    - Edge Cases: Schema mismatches across items leading to misaligned CSV columns.
  - **Save CSV File**
    - Type: `n8n-nodes-base.readWriteFile` (v1.1)
    - Technical Role: Writes the generated CSV binary data to the designated local file system path.
    - Configuration: Operation set to `write`, path configured via `fileName` (`/path/to/output/results.csv`).
    - Input Connections: `Convert to CSV File`. Output: None.
    - Edge Cases: Write permission denials or insufficient disk space on the host environment.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation, setup steps, and overview | None | None | ## Contact Activity Screener TEMPLATE... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Initialize process documentation | None | None | ## Initialize process<br><br>Manually start and read input CSV. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Iterate contacts and check documentation | None | None | ## Iterate contacts and check<br><br>Iterate over contacts and perform sanity checks. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | MCP processing documentation | None | None | ## MCP processing<br><br>Build MCP steps and execute all channel calls. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | AI assessment documentation | None | None | ## AI assessment<br><br>Build a prompt, get verdict from OpenAI, and parse results. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | CSV output documentation | None | None | ## CSV output<br><br>Convert results to CSV and write output. |
| Manual Start Trigger | `n8n-nodes-base.manualTrigger` | Entry point for manual executions | None | Read CSV File | ## Initialize process<br><br>Manually start and read input CSV. |
| Read CSV File | `n8n-nodes-base.readWriteFile` | Reads input contacts CSV file | Manual Start Trigger | Extract CSV Data | ## Initialize process<br><br>Manually start and read input CSV. |
| Extract CSV Data | `n8n-nodes-base.extractFromFile` | Parses raw CSV binary data to items | Read CSV File | Batch Process Contacts | ## Initialize process<br><br>Manually start and read input CSV. |
| Batch Process Contacts | `n8n-nodes-base.splitInBatches` | Iterates over contacts in batches | Extract CSV Data, Format Output Row Data | Check Row Count Safety, Generate MCP Steps Code | ## Iterate contacts and check<br><br>Iterate over contacts and perform sanity checks. |
| Check Row Count Safety | `n8n-nodes-base.code` | Safety guard against runaway loops | Batch Process Contacts | Convert to CSV File | ## CSV output<br><br>Convert results to CSV and write output. |
| Convert to CSV File | `n8n-nodes-base.convertToFile` | Converts results items to CSV format | Check Row Count Safety | Save CSV File | ## CSV output<br><br>Convert results to CSV and write output. |
| Save CSV File | `n8n-nodes-base.readWriteFile` | Writes CSV data to local output path | Convert to CSV File | None | ## CSV output<br><br>Convert results to CSV and write output. |
| Generate MCP Steps Code | `n8n-nodes-base.code` | Generates browser automation steps for channels | Batch Process Contacts | Perform MCP Calls | ## MCP processing<br><br>Build MCP steps and execute all channel calls. |
| Perform MCP Calls | `@n8n/n8n-nodes-langchain.mcpClient` | Executes tool calls against MCP endpoint | Generate MCP Steps Code | Extract MCP Results | ## MCP processing<br><br>Build MCP steps and execute all channel calls. |
| Extract MCP Results | `n8n-nodes-base.code` | Cleans and extracts text snapshots per channel | Perform MCP Calls | Create Verdict Prompt | ## MCP processing<br><br>Build MCP steps and execute all channel calls. |
| Create Verdict Prompt | `n8n-nodes-base.code` | Constructs evidence-based prompt for AI | Extract MCP Results | OpenAI Analyze Verdict | ## AI assessment<br><br>Build a prompt, get verdict from OpenAI, and parse results. |
| OpenAI Analyze Verdict | `@n8n/n8n-nodes-langchain.openAi` | Queries OpenAI model for verdict assessment | Create Verdict Prompt | Parse Analysis Results | ## AI assessment<br><br>Build a prompt, get verdict from OpenAI, and parse results. |
| Parse Analysis Results | `n8n-nodes-base.code` | Parses model response into structured verdict fields | OpenAI Analyze Verdict | Format Output Row Data | ## AI assessment<br><br>Build a prompt, get verdict from OpenAI, and parse results. |
| Format Output Row Data | `n8n-nodes-base.code` | Formats final row with evidence and metadata | Parse Analysis Results | Batch Process Contacts | ## AI assessment<br><br>Build a prompt, get verdict from OpenAI, and parse results. |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the workflow manually in n8n:

1. **Create the Manual Trigger Node:**
   - Add a **Manual Start Trigger** node.
2. **Configure File Input Nodes:**
   - Add a **Read CSV File** (`n8n-nodes-base.readWriteFile`) node. Connect `Manual Start Trigger` output to it. Set `fileSelector` to `/path/to/input/contacts.csv`.
   - Add an **Extract CSV Data** (`n8n-nodes-base.extractFromFile`) node. Connect `Read CSV File` output to it. Leave options at default.
3. **Setup Batch Processing and Safety Checks:**
   - Add a **Batch Process Contacts** (`n8n-nodes-base.splitInBatches`) node. Connect `Extract CSV Data` output to it.
   - Add a **Check Row Count Safety** (`n8n-nodes-base.code` version 2) node. Connect output index `0` of `Batch Process Contacts` to it. Paste the JavaScript snippet enforcing `MAX_REASONABLE_ROWS = 500`.
   - Add a **Convert to CSV File** (`n8n-nodes-base.convertToFile`) node. Connect `Check Row Count Safety` output to it.
   - Add a **Save CSV File** (`n8n-nodes-base.readWriteFile`) node. Connect `Convert to CSV File` output to it. Set operation to `write` and `fileName` to `/path/to/output/results.csv`.
4. **Implement MCP Browser Processing:**
   - Add a **Generate MCP Steps Code** (`n8n-nodes-base.code` version 2) node. Connect output index `1` of `Batch Process Contacts` to it. Paste the JavaScript snippet iterating over `website`, `facebook`, and `instagram` URLs.
   - Add a **Perform MCP Calls** (`@n8n/n8n-nodes-langchain.mcpClient`) node. Connect `Generate MCP Steps Code` output to it. Configure `endpointUrl` to `http://localhost:8931/mcp`, set input mode to `json`, configure tool selector expression to `={{ $json.mcp_tool }}`, json input expression to `={{ $json.mcp_params }}`, enable retry on fail with 2 max tries and 5000ms delay.
   - Add an **Extract MCP Results** (`n8n-nodes-base.code` version 2) node. Connect `Perform MCP Calls` output to it. Paste the text cleaning and aggregation script.
5. **Implement AI Verdict Assessment:**
   - Add a **Create Verdict Prompt** (`n8n-nodes-base.code` version 2) node. Connect `Extract MCP Results` output to it. Paste the prompt construction script referencing scraped snapshot properties.
   - Add an **OpenAI Analyze Verdict** (`@n8n/n8n-nodes-langchain.openAi`) node. Connect `Create Verdict Prompt` output to it. Select model `gpt-4o-mini`, configure response values to use `={{ $json.verdict_prompt }}`, and link valid OpenAI API credentials (`OpenAI TEMPLATE`).
   - Add a **Parse Analysis Results** (`n8n-nodes-base.code` version 2) node. Connect `OpenAI Analyze Verdict` output to it. Paste the JSON parsing script with fallback handling.
   - Add a **Format Output Row Data** (`n8n-nodes-base.code` version 2) node. Connect `Parse Analysis Results` output to it. Paste the evidence excerpt and metadata formatting script.
6. **Complete the Loop Connection:**
   - Connect the output of **Format Output Row Data** back into the **Batch Process Contacts** (`n8n-nodes-base.splitInBatches`) node input to continue iteration over remaining contact rows.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow is provided as a template workflow for screening contact activity from CSV data using MCP browser clients and OpenAI models. | Workflow Title: Screen contact activity from CSV with MCP client and OpenAI gpt-4o-mini |