Enrich GDPR research contacts in Airtable with Tavily, OpenAI, and Dropcontact

https://n8nworkflows.xyz/workflows/enrich-gdpr-research-contacts-in-airtable-with-tavily--openai--and-dropcontact-20084


# Enrich GDPR research contacts in Airtable with Tavily, OpenAI, and Dropcontact

### 1. Workflow Overview

The **GDPR Contact Enricher** workflow automates the extraction, enrichment, and storage of professional contacts based on research prompts stored in Airtable. It runs twice daily (at 09:00 and 14:00) to query pending requests, search the web for relevant entities, process them using AI, enrich them via a third-party provider, and update the database accordingly.

The architecture is divided into the following logical blocks:
- **1.1 Scheduled Request Intake:** Periodically triggers the workflow and fetches pending research prompts from Airtable, processing them sequentially.
- **1.2 Search and Qualify:** Executes a deep web search via Tavily and applies a programmatic score filter to validate the evidence.
- **1.3 AI Extraction:** Utilizes OpenAI to parse structured contact details from the qualified search output and appends tracking identifiers.
- **1.4 Enrichment and Callback Processing:** Formats and submits contacts to Dropcontact, pauses execution via a dynamic resume URL, and retrieves the enriched payload.
- **1.5 Data Recombination and Persistence:** Recombines context data, upserts individual enriched contacts into Airtable, and marks the parent research request as completed or failed.
- **1.6 Webhook Setup Utility (Disabled):** A manual utility branch intended for registering external webhook subscriptions if required.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Request Intake
- **Overview:** Initiates the automation process on a recurring schedule and reads unhandled records from Airtable, handling them one item at a time.
- **Nodes Involved:** `When Scheduled Run`, `Fetch Pending Requests`, `Process Items in Batches`.
- **Node Details:**
  - **When Scheduled Run**
    - *Type and Technical Role:* `scheduleTrigger` — Triggers execution based on a time-based cron rule.
    - *Configuration:* Rule configured to run daily at hours `9` and `14`.
    - *Input/Output:* No inputs; outputs execution timestamp to `Fetch Pending Requests`.
    - *Edge Cases:* Missed triggers due to system downtime (standard n8n catch-up behavior applies).
  - **Fetch Pending Requests**
    - *Type and Technical Role:* `airtable` — Queries records from an external Airtable base.
    - *Configuration:* Operation set to `search`, filter formula: `{Status} = 'Pending'`. Requires Airtable API credentials.
    - *Input/Output:* Input from `When Scheduled Run`; outputs an array of matching records to `Process Items in Batches`.
    - *Edge Cases:* Airtable API rate limits or authentication revocation.
  - **Process Items in Batches**
    - *Type and Technical Role:* `splitInBatches` — Controls iteration flow by processing items sequentially (batch size: 1).
    - *Configuration:* Options set to default (`reset: false`).
    - *Input/Output:* Input from `Fetch Pending Requests`; output 0 loops back to finish/fetch next, output 1 routes current item to `Post to Tavily API`.
    - *Edge Cases:* Infinite loops if batch indexing is improperly managed.

#### 2.2 Search and Qualify
- **Overview:** Performs an advanced web search using the Airtable prompt and filters the results using a confidence scoring threshold.
- **Nodes Involved:** `Post to Tavily API`, `Filter Confident Results`, `Check Usable Results`, `Log Low-Confidence Requests`.
- **Node Details:**
  - **Post to Tavily API**
    - *Type and Technical Role:* `httpRequest` — Communicates with the Tavily search endpoint.
    - *Configuration:* POST request to `https://api.tavily.com/search`. Body includes `query: "={{ $json.Prompt }}"`, `include_answer: "advanced"`, `search_depth: "advanced"`, `max_results: 10`. Authorization header uses `=Bearer {{ $env.TAVILY_API_KEY }}`.
    - *Input/Output:* Input from `Process Items in Batches`; outputs search result arrays to `Filter Confident Results`.
    - *Edge Cases:* Tavily API timeout, missing environment variable `TAVILY_API_KEY`.
  - **Filter Confident Results**
    - *Type and Technical Role:* `code` (JavaScript) — Evaluates search result confidence scores.
    - *Configuration:* Filters items where `result.score >= 0.7`. Sets `found_data` boolean flag (`true` or `false`).
    - *Input/Output:* Input from `Post to Tavily API`; outputs processed items with metadata flags to `Check Usable Results`.
    - *Edge Cases:* Empty result arrays causing script evaluation errors if unhandled.
  - **Check Usable Results**
    - *Type and Technical Role:* `if` — Branches the workflow based on data validity.
    - *Configuration:* Condition checks if `{{ $json.found_data }}` equals `true`.
    - *Input/Output:* Input from `Filter Confident Results`; output true routes to `OpenAI Extract Contacts`, output false routes to `Log Low-Confidence Requests`.
    - *Edge Cases:* Type mismatch on boolean evaluation.
  - **Log Low-Confidence Requests**
    - *Type and Technical Role:* `airtable` — Updates the request record status upon search failure.
    - *Configuration:* Operation set to `update`. Matches on `Prompt`, updates `Status` to `Failed - Low Confidence`.
    - *Input/Output:* Input from `Check Usable Results` (false branch); outputs update confirmation.
    - *Edge Cases:* Mismatched prompt strings preventing record updates.

#### 2.3 AI Extraction
- **Overview:** Sends valid search text to an LLM to extract structured contact data, then flattens the resulting array into individual items.
- **Nodes Involved:** `OpenAI Extract Contacts`, `Split Contacts`, `Assign Request ID`.
- **Node Details:**
  - **OpenAI Extract Contacts**
    - *Type and Technical Role:* `openAi` (LangChain integration) — Interacts with an OpenAI chat model.
    - *Configuration:* Model set to `=gpt-5-mini`. Response format forced to JSON object (`textFormat: json_object`). System prompt instructs extraction of `name`, `organization`, `role`, and `interests`. User prompt parses input via `={{ JSON.stringify($json, null, 2) }}`. Requires OpenAI credentials.
    - *Input/Output:* Input from `Check Usable Results` (true branch); outputs structured JSON response to `Split Contacts`.
    - *Edge Cases:* Malformed JSON responses from the model, token limit exceedance, rate limiting (mitigated by `retryOnFail`).
  - **Split Contacts**
    - *Type and Technical Role:* `splitOut` — Deconstructs an array into individual execution items.
    - *Configuration:* Field to split out set to `output[1].content[0].text.contacts`.
    - *Input/Output:* Input from `OpenAI Extract Contacts`; outputs individual contact objects to `Assign Request ID`.
    - *Edge Cases:* Missing array paths resulting in empty outputs.
  - **Assign Request ID**
    - *Type and Technical Role:* `set` (Edit Fields) — Injects parent workflow context into child contact items.
    - *Configuration:* Creates assignment `researchRequestId` valued at `={{ $('Process Items in Batches').first().json.id }}` while retaining other fields.
    - *Input/Output:* Input from `Split Contacts`; outputs enriched context items to `Prepare Data for Dropcontact` and `Merge Request and Results`.
    - *Edge Cases:* Scope resolution failures if batch node reference index becomes stale.

#### 2.4 Enrichment and Callback Processing
- **Overview:** Submits extracted names and organizations to Dropcontact for email and data enrichment, pauses execution using an async webhook wait mechanism, and fetches completed results.
- **Nodes Involved:** `Prepare Data for Dropcontact`, `Submit to Dropcontact API`, `Wait for Dropcontact Callback`, `Fetch Dropcontact Results`.
- **Node Details:**
  - **Prepare Data for Dropcontact**
    - *Type and Technical Role:* `code` (JavaScript) — Transforms item schema and injects dynamic n8n resume URLs.
    - *Configuration:* Maps `full_name` to `item.json.name` and `company` to `item.json.organization`. Captures `$execution.resumeUrl` into `custom_callback_url`.
    - *Input/Output:* Input from `Assign Request ID`; outputs formatted payload to `Submit to Dropcontact API`.
    - *Edge Cases:* Internal execution URL resolution failures.
  - **Submit to Dropcontact API**
    - *Type and Technical Role:* `httpRequest` — Transmits batch enrichment requests to Dropcontact.
    - *Configuration:* POST request to `https://api.dropcontact.com/v1/enrich/all`. Uses predefined Dropcontact API credentials.
    - *Input/Output:* Input from `Prepare Data for Dropcontact`; outputs request handle to `Wait for Dropcontact Callback`.
    - *Edge Cases:* API authentication failures, payload validation errors from Dropcontact.
  - **Wait for Dropcontact Callback**
    - *Type and Technical Role:* `wait` — Pauses workflow execution until an external webhook calls the generated resume URL.
    - *Configuration:* Fallback amount set to 90 seconds (though designed to resume asynchronously via webhook). Webhook ID configured.
    - *Input/Output:* Input from `Submit to Dropcontact API`; outputs resumed execution payload to `Fetch Dropcontact Results`.
    - *Edge Cases:* Webhook timeouts if Dropcontact fails to callback within expected limits or network partitions block incoming requests.
  - **Fetch Dropcontact Results**
    - *Type and Technical Role:* `dropcontact` — Retrieves enriched contact profiles.
    - *Configuration:* Operation set to `fetchRequest`. Parameter `requestId` set to `={{ $json.request_id }}`.
    - *Input/Output:* Input from `Wait for Dropcontact Callback`; outputs enriched profile data to `Merge Request and Results`.
    - *Edge Cases:* Missing request ID parameter.

#### 2.5 Data Recombination and Persistence
- **Overview:** Re-merges contact context, loops through enriched profiles to upsert them into Airtable, and concludes by marking the parent request as completed.
- **Nodes Involved:** `Merge Request and Results`, `Iterate Contacts in Batches`, `Upsert Contacts in Airtable`, `Complete Request in Airtable`.
- **Node Details:**
  - **Merge Request and Results**
    - *Type and Technical Role:* `merge` — Combines two data streams by position.
    - *Configuration:* Mode set to `combine`, combine strategy: `combineByPosition`.
    - *Input/Output:* Inputs from `Assign Request ID` and `Fetch Dropcontact Results`; outputs unified dataset to `Iterate Contacts in Batches`.
    - *Edge Cases:* Data stream length mismatches causing dropped records.
  - **Iterate Contacts in Batches**
    - *Type and Technical Role:* `splitInBatches` — Iterates through enriched contacts one by one.
    - *Configuration:* Default options.
    - *Input/Output:* Input from `Merge Request and Results`; output 0 routes to `Complete Request in Airtable` when complete, output 1 routes to `Upsert Contacts in Airtable`.
    - *Edge Cases:* Premature loop termination.
  - **Upsert Contacts in Airtable**
    - *Type and Technical Role:* `airtable` — Creates or updates contact records in the destination table.
    - *Configuration:* Operation set to `upsert`. Matching columns: `Name` and `Organization/Institution/Department/Region/Sector`. Mappings link Dropcontact output fields (`full_name`, `country`, `email`, `phone`, `company`, etc.) to Airtable columns.
    - *Input/Output:* Input from `Iterate Contacts in Batches`; loops back to `Iterate Contacts in Batches`.
    - *Edge Cases:* Airtable column mapping discrepancies or unhandled dropdown option constraints.
  - **Complete Request in Airtable**
    - *Type and Technical Role:* `airtable` — Updates parent research request status to completed.
    - *Configuration:* Operation set to `update`. Matches on `Prompt`, updates `Status` to `Completed`.
    - *Input/Output:* Input from `Iterate Contacts in Batches` (completion loop); routes back to `Process Items in Batches` to handle the next request.
    - *Edge Cases:* Missing prompt references.

#### 2.6 Webhook Setup Utility (Disabled)
- **Overview:** Provides an optional, manual trigger branch to register webhook subscriptions with Dropcontact if required.
- **Nodes Involved:** `Manual Webhook Initialization`, `Setup Dropcontact Webhook`.
- **Node Details:**
  - **Manual Webhook Initialization**
    - *Type and Technical Role:* `manualTrigger` — Allows manual execution of the setup utility branch.
    - *Configuration:* None. (Node is disabled by default).
    - *Input/Output:* Outputs manual trigger event to `Setup Dropcontact Webhook`.
  - **Setup Dropcontact Webhook**
    - *Type and Technical Role:* `httpRequest` — Registers webhook configurations with Dropcontact.
    - *Configuration:* POST request to `https://api.dropcontact.com/v1/webhooks/subscriptions`. Header includes `X-Access-Token` mapped from `{{ $env.DROPCONTACT_API_KEY }}`. (Node is disabled by default).
    - *Input/Output:* Input from `Manual Webhook Initialization`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation wrapper for workflow overview and setup steps. | None | None | ## GDPR Contact Enricher<br><br>### How it works<br><br>This workflow periodically pulls pending GDPR contact enrichment requests from Airtable and processes them one request at a time. For each request, it searches the web with Tavily, filters results by confidence, uses OpenAI to extract people, enriches the resulting contacts through Dropcontact, then writes contacts back to Airtable and marks the request complete. Requests without usable search evidence are marked as low-confidence instead of being enriched.<br><br>### Setup steps<br><br>- Configure Airtable credentials and verify the pending requests, contacts, completion, and low-confidence status fields match the node mappings.<br>- Add credentials or API keys for Tavily, OpenAI, and Dropcontact in the relevant HTTP/OpenAI/Dropcontact nodes.<br>- Run the manual webhook setup branch once to register the Dropcontact callback URL, and ensure the Wait node/webhook URL is reachable from Dropcontact.<br>- Review the schedule trigger interval and enable the workflow after testing with a small batch of pending requests.<br><br>### Customization<br><br>Adjust the Tavily query parameters, confidence threshold code, OpenAI extraction prompt, Dropcontact payload formatting, and Airtable field mappings to match the target market, data model, and GDPR review process. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation wrapper for webhook setup utility. | None | None | ## Webhook setup utility<br><br>A separate manual branch used to register the Dropcontact webhook before the automated enrichment flow runs. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation wrapper for scheduled intake. | None | None | ## Scheduled request intake<br><br>Starts on a schedule, reads pending Airtable research requests, and iterates through them one at a time. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation wrapper for search and qualification. | None | None | ## Search and qualify<br><br>Runs a Tavily deep search, filters results by confidence, and branches based on whether the evidence is usable; weak requests are marked low-confidence in Airtable. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation wrapper for AI contact extraction. | None | None | ## Extract and tag people<br><br>Uses OpenAI to extract people from usable search results, splits the extracted list into individual items, and attaches the originating research request ID. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation wrapper for Dropcontact execution. | None | None | ## Run Dropcontact enrichment<br><br>Formats extracted contacts for Dropcontact, submits the enrichment batch, waits for the callback, and fetches the completed enrichment result. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Documentation wrapper for context recombination. | None | None | ## Recombine request context<br><br>Merges the enriched Dropcontact data back with the original request/contact context before saving. |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Documentation wrapper for saving enriched contacts. | None | None | ## Save enriched contacts<br><br>Loops through enriched contacts, upserts each one to Airtable, then marks the overall request complete and returns to the next pending request. |
| `When Scheduled Run` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow execution on a fixed daily schedule. | None | `Fetch Pending Requests` | |
| `Fetch Pending Requests` | `n8n-nodes-base.airtable` | Queries Airtable for records where Status is Pending. | `When Scheduled Run` | `Process Items in Batches` | |
| `Process Items in Batches` | `n8n-nodes-base.splitInBatches` | Iterates through retrieved Airtable records sequentially. | `Fetch Pending Requests`, `Complete Request in Airtable` | `Post to Tavily API` (Output 1) | |
| `Post to Tavily API` | `n8n-nodes-base.httpRequest` | Performs web research queries via the Tavily API. | `Process Items in Batches` | `Filter Confident Results` | |
| `Filter Confident Results` | `n8n-nodes-base.code` | Evaluates search result confidence scores (threshold >= 0.7). | `Post to Tavily API` | `Check Usable Results` | |
| `Check Usable Results` | `n8n-nodes-base.if` | Routes workflow based on whether usable search data was found. | `Filter Confident Results` | `OpenAI Extract Contacts` (True), `Log Low-Confidence Requests` (False) | |
| `Log Low-Confidence Requests` | `n8n-nodes-base.airtable` | Updates Airtable record status to Failed - Low Confidence. | `Check Usable Results` | None | |
| `OpenAI Extract Contacts` | `@n8n/n8n-nodes-langchain.openAi` | Extracts structured contact JSON from search results using OpenAI. | `Check Usable Results` | `Split Contacts` | |
| `Split Contacts` | `n8n-nodes-base.splitOut` | Flattens the extracted contacts array into individual items. | `OpenAI Extract Contacts` | `Assign Request ID` | |
| `Assign Request ID` | `n8n-nodes-base.set` | Injects the parent research request ID into contact items. | `Split Contacts` | `Prepare Data for Dropcontact`, `Merge Request and Results` | |
| `Prepare Data for Dropcontact` | `n8n-nodes-base.code` | Formats contact payloads and injects dynamic n8n resume URLs. | `Assign Request ID` | `Submit to Dropcontact API` | |
| `Submit to Dropcontact API` | `n8n-nodes-base.httpRequest` | Submits enrichment batch to Dropcontact. | `Prepare Data for Dropcontact` | `Wait for Dropcontact Callback` | |
| `Wait for Dropcontact Callback` | `n8n-nodes-base.wait` | Pauses execution until an asynchronous callback resumes the workflow. | `Submit to Dropcontact API` | `Fetch Dropcontact Results` | |
| `Fetch Dropcontact Results` | `n8n-nodes-base.dropcontact` | Retrieves completed enrichment results using the request ID. | `Wait for Dropcontact Callback` | `Merge Request and Results` | |
| `Merge Request and Results` | `n8n-nodes-base.merge` | Combines initial context data with enriched profile results. | `Assign Request ID`, `Fetch Dropcontact Results` | `Iterate Contacts in Batches` | |
| `Iterate Contacts in Batches` | `n8n-nodes-base.splitInBatches` | Loops through enriched contacts individually for persistence. | `Merge Request and Results` | `Complete Request in Airtable` (Output 0), `Upsert Contacts in Airtable` (Output 1) | |
| `Upsert Contacts in Airtable` | `n8n-nodes-base.airtable` | Upserts individual enriched contact records into Airtable. | `Iterate Contacts in Batches` | `Iterate Contacts in Batches` | |
| `Complete Request in Airtable` | `n8n-nodes-base.airtable` | Marks the parent research request as Completed in Airtable. | `Iterate Contacts in Batches` | `Process Items in Batches` | |
| `Manual Webhook Initialization` | `n8n-nodes-base.manualTrigger` | Manual trigger for the webhook setup utility (Disabled). | None | `Setup Dropcontact Webhook` | |
| `Setup Dropcontact Webhook` | `n8n-nodes-base.httpRequest` | Registers webhook subscriptions with Dropcontact (Disabled). | `Manual Webhook Initialization` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Schedule Trigger** node (`When Scheduled Run`). Set the interval rule to trigger at hours `9` and `14`.
2. **Fetch Data from Airtable:**
   - Create an **Airtable** node (`Fetch Pending Requests`). Connect it to the Schedule Trigger.
   - Configure credentials (Airtable Personal Access Token). Set operation to `search`, base ID to `YOUR_AIRTABLE_BASE_ID`, table ID to `YOUR_RESEARCH_REQUESTS_TABLE_ID`, and filter formula to `{Status} = 'Pending'`.
3. **Setup Batch Processing:**
   - Add a **Split In Batches** node (`Process Items in Batches`) connected to `Fetch Pending Requests`. Set batch size to `1`.
4. **Perform Web Search:**
   - Add an **HTTP Request** node (`Post to Tavily API`) connected to output port 1 of the batch node.
   - Set Method to `POST`, URL to `https://api.tavily.com/search`.
   - Add Header `Authorization` = `=Bearer {{ $env.TAVILY_API_KEY }}`.
   - Add JSON Body parameters: `query` (`={{ $json.Prompt }}`), `include_answer` (`advanced`), `search_depth` (`advanced`), `max_results` (`10`). Enable `Retry On Fail`.
5. **Filter Search Results:**
   - Add a **Code** node (`Filter Confident Results`) connected to Tavily.
   - Insert JavaScript to iterate over items, filter results where `score >= 0.7`, and assign a boolean `found_data` flag (`true` or `false`).
6. **Branch Based on Confidence:**
   - Add an **If** node (`Check Usable Results`) connected to the filter node.
   - Set condition to evaluate if `{{ $json.found_data }}` equals `true`.
   - *False branch:* Connect to an **Airtable** node (`Log Low-Confidence Requests`). Set operation to `update`, matching column to `Prompt`, and update `Status` to `Failed - Low Confidence`.
7. **Extract Contacts with AI:**
   - *True branch:* Connect to an **OpenAI** node (`OpenAI Extract Contacts`).
   - Configure OpenAI credentials. Set model to `=gpt-5-mini`. Set Response Format to JSON object.
   - Set System Prompt to instruct extraction of `name`, `organization`, `role`, and `interests` into a JSON schema. Set User Prompt to `={{ JSON.stringify($json, null, 2) }}`. Enable `Retry On Fail`.
8. **Flatten and Tag Extracted Contacts:**
   - Add a **Split Out** node (`Split Contacts`) connected to OpenAI. Set field to split out to `output[1].content[0].text.contacts`.
   - Add a **Set (Edit Fields)** node (`Assign Request ID`) connected to Split Contacts. Create an assignment `researchRequestId` valued at `={{ $('Process Items in Batches').first().json.id }}` while retaining other fields.
9. **Enrich Contacts via Dropcontact:**
   - Add a **Code** node (`Prepare Data for Dropcontact`) connected to `Assign Request ID`.
   - Map `full_name` to `item.json.name`, `company` to `item.json.organization`, and assign `custom_callback_url` to `{{ $execution.resumeUrl }}`.
   - Connect to an **HTTP Request** node (`Submit to Dropcontact API`). Set Method to `POST`, URL to `https://api.dropcontact.com/v1/enrich/all`. Configure Dropcontact API authentication.
   - Connect to a **Wait** node (`Wait for Dropcontact Callback`). Configure webhook wait behavior (fallback 90 seconds).
   - Connect to a **Dropcontact** node (`Fetch Dropcontact Results`). Set operation to `fetchRequest` and `requestId` to `={{ $json.request_id }}`.
10. **Recombine Context and Enriched Data:**
    - Add a **Merge** node (`Merge Request and Results`). Connect input 0 to `Fetch Dropcontact Results` and input 1 to `Assign Request ID`. Set mode to `combine` by position.
11. **Upsert and Complete Records:**
    - Connect the merge node to a **Split In Batches** node (`Iterate Contacts in Batches`).
    - *Output 0 (Completion):* Connect to an **Airtable** node (`Complete Request in Airtable`). Set operation to `update`, match on `Prompt`, and update `Status` to `Completed`. Loop this back to `Process Items in Batches`.
    - *Output 1 (Upsert):* Connect to an **Airtable** node (`Upsert Contacts in Airtable`). Set operation to `upsert`, matching columns to `Name` and `Organization/Institution/Department/Region/Sector`. Map Dropcontact fields (`full_name`, `country`, `email`, `phone`, `interests`, `company`, etc.) to corresponding Airtable columns. Loop this back to `Iterate Contacts in Batches`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Airtable Database Interchangeability | Airtable can be swapped out for Excel or any other database; designed for non-technical users to submit enrichment requests. |
| External API Prerequisites | Requires valid API keys and credentials for Airtable (Personal Access Token), Tavily, OpenAI, and Dropcontact. |