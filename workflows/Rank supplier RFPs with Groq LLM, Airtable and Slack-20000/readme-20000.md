Rank supplier RFPs with Groq LLM, Airtable and Slack

https://n8nworkflows.xyz/workflows/rank-supplier-rfps-with-groq-llm--airtable-and-slack-20000


# Rank supplier RFPs with Groq LLM, Airtable and Slack

### 1. Workflow Overview

This workflow functions as an automated procurement evaluation and ranking engine. Its primary purpose is to retrieve pending supplier RFP responses, score them qualitatively via a Groq-hosted LLM using distinct evaluation rubrics, apply a business-strategy-weighted mathematical model, rank the suppliers into a leaderboard, and persist the updates back to Airtable while broadcasting results to Slack.

The logic is grouped into the following functional blocks:
- **1.1 Initialization & Strategy Fetch:** Manually triggers the workflow, queries global settings from Airtable to identify the active strategic priority, and maps it to a corresponding evaluation prompt rubric.
- **1.2 Supplier Data Ingestion:** Queries Airtable for pending vendor submissions and passes them into a batch processing loop.
- **1.3 AI Analysis & Weighted Scoring:** Evaluates each supplier’s ESG practices and risk mitigation strategies using an LLM configured with structured JSON outputs, then calculates a final normalized score based on price, lead time, and AI sub-scores.
- **1.4 Ranking & System Output:** Reassembles the processed batch, sorts suppliers by score to assign leaderboard rankings, updates each Airtable record, and sends a notification broadcast to Slack.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Initialization & Strategy Fetch

##### Overview
This block initiates the execution, retrieves the current enterprise priority setting from a configuration database, and prepares the corresponding dynamic system prompt for the AI evaluation phase.

##### Nodes Involved
- `Start Workflow`
- `Fetch Priority`
- `Prompts`
- `System Prompt`

##### Node Details

- **Start Workflow**
  - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Trigger node). Acts as the manual entry point for test runs.
  - *Configuration Choices:* Standard manual invocation settings without webhook parameters.
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Input: None; Output: Connects to `Fetch Priority`.
  - *Version-Specific Requirements:* v1.
  - *Edge Cases / Potential Failures:* None.
  - *Sub-Workflow Reference:* None.

- **Fetch Priority**
  - *Type and Technical Role:* `n8n-nodes-base.airtable` (Airtable node). Searches the "Global Settings" base to retrieve the active strategic priority value.
  - *Configuration Choices:* Search operation targeting base `appufBeEMzlDG6ASg`, table `tbl4WUc84SUKdPQeu`.
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Input: `Start Workflow`; Output: Connects to `Prompts`.
  - *Version-Specific Requirements:* v2.2.
  - *Edge Cases / Potential Failures:* Airtable API authentication failure, incorrect Base/Table IDs, or missing configuration records.
  - *Sub-Workflow Reference:* None.

- **Prompts**
  - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields node). Defines static string variables containing the system evaluation rubrics (`ESG_Focus`, `Cost_Reduction`, `Speed_to_Market`, and `Balanced`).
  - *Configuration Choices:* Set to execute once (`executeOnce: true`) with four string assignments outlining scoring instructions (1–10 scale definitions) for the LLM.
  - *Key Expressions or Variables:* Static evaluation prompt strings.
  - *Input and Output Connections:* Input: `Fetch Priority`; Output: Connects to `System Prompt`.
  - *Version-Specific Requirements:* v3.5.
  - *Edge Cases / Potential Failures:* None.
  - *Sub-Workflow Reference:* None.

- **System Prompt**
  - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields node). Dynamically selects the active system prompt based on the priority returned by Airtable.
  - *Configuration Choices:* Assigns a variable named `System_Prompt`.
  - *Key Expressions or Variables:* `={{ $json[$('Fetch Priority').first().json.fields['Active Value']] }}`
  - *Input and Output Connections:* Input: `Prompts`; Output: Connects to `Fetch Pending Suppliers`.
  - *Version-Specific Requirements:* v3.5.
  - *Edge Cases / Potential Failures:* If the value returned by `Fetch Priority` does not exactly match one of the defined prompt keys, the expression evaluates to undefined.
  - *Sub-Workflow Reference:* None.

---

#### 1.2 Supplier Data Ingestion

##### Overview
This block queries the target Airtable base to extract all supplier records marked with a "Pending" processing status and queues them into a loop for sequential item-by-item processing.

##### Nodes Involved
- `Fetch Pending Suppliers`
- `Batch Processing Loop`

##### Node Details

- **Fetch Pending Suppliers**
  - *Type and Technical Role:* `n8n-nodes-base.airtable` (Airtable node). Searches for supplier records matching a specific view filter.
  - *Configuration Choices:* Search operation targeting base `appXRp1pD9Qtm5Un4`, table `tbl8s52LdLqPz4jdl`, using the view `viwNWbkPzAaPODaog` ("Pending records").
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Input: `System Prompt`; Output: Connects to `Batch Processing Loop`.
  - *Version-Specific Requirements:* v2.2.
  - *Edge Cases / Potential Failures:* API rate limits, invalid table/view IDs, or empty search results.
  - *Sub-Workflow Reference:* None.

- **Batch Processing Loop**
  - *Type and Technical Role:* `n8n-nodes-base.splitInBatches` (Split In Batches node). Iterates through the retrieved list of supplier records one item at a time.
  - *Configuration Choices:* Default batch options.
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Input: `Fetch Pending Suppliers` and loopback from `Calculate Weighted Score`; Output: Loops to `Evaluate Supplier (LLM)` for processing, and branches out to `Sort by Score` when the loop is exhausted.
  - *Version-Specific Requirements:* v3.
  - *Edge Cases / Potential Failures:* Infinite loops if loop completion wiring is misconfigured.
  - *Sub-Workflow Reference:* None.

---

#### 1.3 AI Analysis & Weighted Scoring

##### Overview
For each supplier in the batch, this block prompts a Groq-hosted LLM to evaluate qualitative fields against the active prompt rubric, enforces a strict JSON output structure, and executes custom JavaScript to calculate a weighted numerical score based on price, lead time, and AI metrics.

##### Nodes Involved
- `Evaluate Supplier (LLM)`
- `Groq LLM Engine`
- `Enforce JSON Schema`
- `Extract AI Scores`
- `Calculate Weighted Score`

##### Node Details

- **Evaluate Supplier (LLM)**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI / Basic LLM Chain node). Orchestrates the LLM text generation request using the supplier's qualitative submissions and the dynamic system prompt.
  - *Configuration Choices:* Uses defined prompt type with message binding.
  - *Key Expressions or Variables:* 
    - Text prompt: `=Please evaluate the following supplier submission:\n\nSupplier Name: {{ $json.fields['Supplier Name'] }}\nESG Practices: {{ $json.fields['ESG Practices'] }}\nRisk Mitigation Strategy: {{ $json.fields['Risk Mitigation Strategy'] }}`
    - System message binding: `={{ $('System Prompt').item.json.System_Prompt }}`
  - *Input and Output Connections:* Input: `Batch Processing Loop`, `Groq LLM Engine` (AI Model link), `Enforce JSON Schema` (Output Parser link); Output: Connects to `Extract AI Scores`.
  - *Version-Specific Requirements:* v1.9.
  - *Edge Cases / Potential Failures:* LLM provider timeouts, rate-limiting, or token limit overages.
  - *Sub-Workflow Reference:* None.

- **Groq LLM Engine**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (Groq Chat Model node). Provides the underlying language model infrastructure.
  - *Configuration Choices:* Configured to use model `openai/gpt-oss-120b`.
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Output (AI Model): Connects to `Evaluate Supplier (LLM)`.
  - *Version-Specific Requirements:* v1.
  - *Edge Cases / Potential Failures:* Invalid Groq API credentials (`groqApi` credential ID `AnindwaoyRCy8KlP`).
  - *Sub-Workflow Reference:* None.

- **Enforce JSON Schema**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser node). Ensures the LLM response adheres strictly to a predefined schema containing integer scores and string rationales.
  - *Configuration Choices:* Manual schema definition requiring `ESG_Score`, `ESG_Reasoning`, `Risk_Score`, and `Risk_Reasoning`.
  - *Key Expressions or Variables:* JSON Schema structure enforcing integer types (1–10) and string descriptions with `additionalProperties: false`.
  - *Input and Output Connections:* Output (AI Output Parser): Connects to `Evaluate Supplier (LLM)`.
  - *Version-Specific Requirements:* v1.3.
  - *Edge Cases / Potential Failures:* Schema validation errors if the LLM output drifts from the required format.
  - *Sub-Workflow Reference:* None.

- **Extract AI Scores**
  - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields node). Maps the structured parser output into clean root-level numeric and string attributes.
  - *Configuration Choices:* Assigns `ESG_Score`, `ESG_Reasoning`, `Risk_Score`, and `Risk_Reasoning`.
  - *Key Expressions or Variables:* 
    - `={{ $json.output.ESG_Score }}`
    - `={{ $json.output.ESG_Reasoning }}`
    - `={{ $json.output.Risk_Score }}`
    - `={{ $json.output.Risk_Reasoning }}`
  - *Input and Output Connections:* Input: `Evaluate Supplier (LLM)`; Output: Connects to `Calculate Weighted Score`.
  - *Version-Specific Requirements:* v3.5.
  - *Edge Cases / Potential Failures:* Missing keys in the output object.
  - *Sub-Workflow Reference:* None.

- **Calculate Weighted Score**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Code node - JavaScript). Executes custom logic to normalize price and lead time metrics, apply the dynamic weight matrix based on active strategy, compute a final score out of 100, and package the payload.
  - *Configuration Choices:* Custom JS evaluation block.
  - *Key Expressions or Variables:* Pulls cross-node references via `$('Batch Processing Loop').item.json` and `$('Fetch Priority')`.
  - *Input and Output Connections:* Input: `Extract AI Scores`; Output: Connects back to `Batch Processing Loop` for the next iteration.
  - *Version-Specific Requirements:* v2.
  - *Edge Cases / Potential Failures:* Undefined field references if Airtable columns do not match `Total Bid Price` or `Lead Time (Days)`.
  - *Sub-Workflow Reference:* None.

---

#### 1.4 Ranking & System Output

##### Overview
Once the batch loop finishes processing all suppliers, this block sorts the dataset by final score, assigns sequential leaderboard ranks, updates Airtable records with the results, and broadcasts the summary to Slack.

##### Nodes Involved
- `Sort by Score`
- `Assign Leaderboard Rank`
- `Update Supplier Record`
- `Broadcast Final Leaderboard`

##### Node Details

- **Sort by Score**
  - *Type and Technical Role:* `n8n-nodes-base.sort` (Sort node). Sorts the completed batch of evaluated items in descending order based on the computed final score.
  - *Configuration Choices:* Sort field `finalScore`, order `descending`.
  - *Key Expressions or Variables:* None.
  - *Input and Output Connections:* Input: `Batch Processing Loop` (when batch completes); Output: Connects to `Assign Leaderboard Rank`.
  - *Version-Specific Requirements:* v1.
  - *Edge Cases / Potential Failures:* Empty item arrays if no records were processed.
  - *Sub-Workflow Reference:* None.

- **Assign Leaderboard Rank**
  - *Type and Technical Role:* `n8n-nodes-base.code` (Code node - JavaScript). Iterates through the sorted items to assign sequential integer rank values (`batchRank`).
  - *Configuration Choices:* Custom JS iterating over `$input.all()`.
  - *Key Expressions or Variables:* `item.json.batchRank = index + 1;`
  - *Input and Output Connections:* Input: `Sort by Score`; Output: Connects to `Update Supplier Record` and `Broadcast Final Leaderboard`.
  - *Version-Specific Requirements:* v2.
  - *Edge Cases / Potential Failures:* None.
  - *Sub-Workflow Reference:* None.

- **Update Supplier Record**
  - *Type and Technical Role:* `n8n-nodes-base.airtable` (Airtable node). Updates the supplier records in Airtable with the calculated scores, reasoning rationales, priority applied, processing status, and leaderboard rank.
  - *Configuration Choices:* Operation set to `update` targeting base `appXRp1pD9Qtm5Un4`, table `tbl8s52LdLqPz4jdl`, matching on `Supplier Name`.
  - *Key Expressions or Variables:* Mappings include:
    - `BatchRank`: `={{ $json.batchRank }}`
    - `ESG_Score`: `={{ $json.breakdown.esgScore }}`
    - `Risk_Score`: `={{ $json.breakdown.riskScore }}`
    - `Total Score`: `={{ $json.finalScore }}`
    - `ESG Rationale`: `={{ $json.aiReasoning.esg }}`
    - `Supplier Name`: `={{ $json.supplierName }}`
    - `Priority Applied`: `={{ $json.priorityApplied }}`
    - `Processing Status`: `Evaluated`
  - *Input and Output Connections:* Input: `Assign Leaderboard Rank`; Output: None (Terminal node).
  - *Version-Specific Requirements:* v2.2.
  - *Edge Cases / Potential Failures:* Airtable column mismatch or token expiration.
  - *Sub-Workflow Reference:* None.

- **Broadcast Final Leaderboard**
  - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack node). Posts the final leaderboard results to a designated Slack channel.
  - *Configuration Choices:* Executes once (`executeOnce: true`) with webhook routing configured.
  - *Key Expressions or Variables:* None specified beyond default message payload configurations.
  - *Input and Output Connections:* Input: `Assign Leaderboard Rank`; Output: None (Terminal node).
  - *Version-Specific Requirements:* v2.7.
  - *Edge Cases / Potential Failures:* Invalid Slack channel ID or unauthenticated Slack credentials.
  - *Sub-Workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Start Workflow` | `n8n-nodes-base.manualTrigger` | Manual entry point for test runs. | None | `Fetch Priority` | ## Initialization & Strategy Fetch<br>Retrieves the active strategic priority from the database and dynamically configures the corresponding AI system prompt for the upcoming evaluation. |
| `Fetch Priority` | `n8n-nodes-base.airtable` | Queries Airtable Global Settings for active strategic priority. | `Start Workflow` | `Prompts` | ## Initialization & Strategy Fetch<br>Retrieves the active strategic priority from the database and dynamically configures the corresponding AI system prompt for the upcoming evaluation. |
| `Prompts` | `n8n-nodes-base.set` | Defines static evaluation prompt rubric strings. | `Fetch Priority` | `System Prompt` | ## Initialization & Strategy Fetch<br>Retrieves the active strategic priority from the database and dynamically configures the corresponding AI system prompt for the upcoming evaluation. |
| `System Prompt` | `n8n-nodes-base.set` | Selects active system prompt based on fetched priority. | `Prompts` | `Fetch Pending Suppliers` | ## Initialization & Strategy Fetch<br>Retrieves the active strategic priority from the database and dynamically configures the corresponding AI system prompt for the upcoming evaluation. |
| `Fetch Pending Suppliers` | `n8n-nodes-base.airtable` | Retrieves pending supplier submissions from Airtable. | `System Prompt` | `Batch Processing Loop` | ## Supplier Data Ingestion<br>Queries the database for all pending supplier submissions and initiates the batching loop to process each record individually through the evaluation pipeline. |
| `Batch Processing Loop` | `n8n-nodes-base.splitInBatches` | Iterates through supplier records one by one. | `Fetch Pending Suppliers`, `Calculate Weighted Score` | `Evaluate Supplier (LLM)` (Loop), `Sort by Score` (Exit) | ## Supplier Data Ingestion<br>Queries the database for all pending supplier submissions and initiates the batching loop to process each record individually through the evaluation pipeline. |
| `Groq LLM Engine` | `@n8n/n8n-nodes-langchain.lmChatGroq` | Provides LLM backend for qualitative analysis. | None | `Evaluate Supplier (LLM)` | ## AI Analysis & Weighted Scoring<br>Utilizes an AI model to evaluate qualitative responses with structured outputs, then applies a dynamic mathematical weighting matrix to calculate the final supplier score. |
| `Enforce JSON Schema` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Ensures AI output matches strict JSON structure. | None | `Evaluate Supplier (LLM)` | ## AI Analysis & Weighted Scoring<br>Utilizes an AI model to evaluate qualitative responses with structured outputs, then applies a dynamic mathematical weighting matrix to calculate the final supplier score. |
| `Evaluate Supplier (LLM)` | `@n8n/n8n-nodes-langchain.chainLlm` | Evaluates supplier ESG and risk practices via AI. | `Batch Processing Loop`, `Groq LLM Engine`, `Enforce JSON Schema` | `Extract AI Scores` | ## AI Analysis & Weighted Scoring<br>Utilizes an AI model to evaluate qualitative responses with structured outputs, then applies a dynamic mathematical weighting matrix to calculate the final supplier score. |
| `Extract AI Scores` | `n8n-nodes-base.set` | Extracts scores and reasoning strings from AI output. | `Evaluate Supplier (LLM)` | `Calculate Weighted Score` | ## AI Analysis & Weighted Scoring<br>Utilizes an AI model to evaluate qualitative responses with structured outputs, then applies a dynamic mathematical weighting matrix to calculate the final supplier score. |
| `Calculate Weighted Score` | `n8n-nodes-base.code` | Normalizes metrics and calculates final weighted score. | `Extract AI Scores` | `Batch Processing Loop` | ## AI Analysis & Weighted Scoring<br>Utilizes an AI model to evaluate qualitative responses with structured outputs, then applies a dynamic mathematical weighting matrix to calculate the final supplier score. |
| `Sort by Score` | `n8n-nodes-base.sort` | Sorts processed supplier batch by final score. | `Batch Processing Loop` | `Assign Leaderboard Rank` | ## Ranking & System Output<br>Reassembles the processed batch, sorts the suppliers by final score to assign a leaderboard rank, updates the database, and sends a team notification. |
| `Assign Leaderboard Rank` | `n8n-nodes-base.code` | Assigns sequential rank numbers to sorted suppliers. | `Sort by Score` | `Update Supplier Record`, `Broadcast Final Leaderboard` | ## Ranking & System Output<br>Reassembles the processed batch, sorts the suppliers by final score to assign a leaderboard rank, updates the database, and sends a team notification. |
| `Update Supplier Record` | `n8n-nodes-base.airtable` | Updates Airtable supplier records with scores and status. | `Assign Leaderboard Rank` | None | ## Ranking & System Output<br>Reassembles the processed batch, sorts the suppliers by final score to assign a leaderboard rank, updates the database, and sends a team notification. |
| `Broadcast Final Leaderboard` | `n8n-nodes-base.slack` | Broadcasts final rankings to Slack. | `Assign Leaderboard Rank` | None | ## Ranking & System Output<br>Reassembles the processed batch, sorts the suppliers by final score to assign a leaderboard rank, updates the database, and sends a team notification. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Node:** Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Start Workflow`.
2. **Fetch Strategic Priority:** Add an **Airtable** (`n8n-nodes-base.airtable`) node named `Fetch Priority`. Set the operation to `search`, configure your Airtable Base ID (`appufBeEMzlDG6ASg`) and Global Settings Table ID (`tbl4WUc84SUKdPQeu`). Connect `Start Workflow` to this node.
3. **Define Rubrics:** Add an Edit Fields (**Set**) (`n8n-nodes-base.set`) node named `Prompts`. Set `executeOnce` to true and add four string assignments: `ESG_Focus`, `Cost_Reduction`, `Speed_to_Market`, and `Balanced`, populating each with their respective evaluation rubrics. Connect `Fetch Priority` to `Prompts`.
4. **Configure System Prompt:** Add an Edit Fields (**Set**) (`n8n-nodes-base.set`) node named `System Prompt`. Create an assignment named `System_Prompt` using the expression:
   `={{ $json[$('Fetch Priority').first().json.fields['Active Value']] }}`
   Connect `Prompts` to `System Prompt`.
5. **Fetch Supplier Submissions:** Add an **Airtable** (`n8n-nodes-base.airtable`) node named `Fetch Pending Suppliers`. Set operation to `search`, configure your target Supplier Base ID (`appXRp1pD9Qtm5Un4`), Table ID (`tbl8s52LdLqPz4jdl`), and View ID (`viwNWbkPzAaPODaog` for "Pending records"). Connect `System Prompt` to this node.
6. **Setup Batch Loop:** Add a **Split In Batches** (`n8n-nodes-base.splitInBatches`) node named `Batch Processing Loop`. Connect `Fetch Pending Suppliers` to it.
7. **Configure AI Model & Parser:** 
   - Add a Groq Chat Model (`@n8n/n8n-nodes-langchain.lmChatGroq`) node named `Groq LLM Engine`. Configure credential (`groqApi`) and select model `openai/gpt-oss-120b`.
   - Add a Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) node named `Enforce JSON Schema`. Define the manual schema requiring properties `ESG_Score` (integer), `ESG_Reasoning` (string), `Risk_Score` (integer), and `Risk_Reasoning` (string) with `additionalProperties: false`.
8. **Configure LLM Chain:** Add a Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) node named `Evaluate Supplier (LLM)`. Connect `Groq LLM Engine` to its AI model input and `Enforce JSON Schema` to its output parser input. Set the prompt text to evaluate the supplier name, ESG practices, and risk mitigation strategy, and bind the system message to `={{ $('System Prompt').item.json.System_Prompt }}`. Connect the main output of `Batch Processing Loop` to this node.
9. **Extract AI Output:** Add an Edit Fields (**Set**) (`n8n-nodes-base.set`) node named `Extract AI Scores`. Map `ESG_Score`, `ESG_Reasoning`, `Risk_Score`, and `Risk_Reasoning` to extract data from `$json.output`. Connect `Evaluate Supplier (LLM)` to `Extract AI Scores`.
10. **Calculate Weighted Scores:** Add a **Code** (`n8n-nodes-base.code`) node named `Calculate Weighted Score`. Insert JavaScript logic to extract Airtable fields, parse AI scores, look up active weights based on strategy priority, normalize price and lead time to a 1–10 scale, and compute the final weighted score out of 100. Connect `Extract AI Scores` to `Calculate Weighted Score`, and wire the output back to `Batch Processing Loop`.
11. **Sort Processed Batch:** Add a **Sort** (`n8n-nodes-base.sort`) node named `Sort by Score`. Set the sort field to `finalScore` with a descending order. Connect the loop exit branch of `Batch Processing Loop` to `Sort by Score`.
12. **Assign Leaderboard Ranks:** Add a **Code** (`n8n-nodes-base.code`) node named `Assign Leaderboard Rank` with JavaScript to iterate over sorted items and assign sequential `batchRank` values. Connect `Sort by Score` to this node.
13. **Update Database Records:** Add an **Airtable** (`n8n-nodes-base.airtable`) node named `Update Supplier Record`. Set operation to `update`, configure base and table IDs, match on `Supplier Name`, and map `BatchRank`, scores, rationales, total score, priority applied, and processing status (`Evaluated`). Connect `Assign Leaderboard Rank` to this node.
14. **Broadcast Notification:** Add a **Slack** (`n8n-nodes-base.slack`) node named `Broadcast Final Leaderboard`. Configure Slack credentials and destination channel. Connect `Assign Leaderboard Rank` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Credits & Assistance | Contact WeblineIndia for custom automation design and workflow support. |
| Airtable Setup Requirement | Ensure supplier tables contain exact column names matching code expectations (`Total Bid Price`, `Lead Time (Days)`). |