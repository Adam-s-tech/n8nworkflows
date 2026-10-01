Negotiate car purchase quotes with Gemini, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/negotiate-car-purchase-quotes-with-gemini--gmail--and-google-sheets-19811


# Negotiate car purchase quotes with Gemini, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates car dealership quote requests and negotiations. It processes inbound customer inquiries submitted via an n8n Form, verifies vehicle availability against a Google Sheets inventory, uses a Google Gemini AI model backed by Redis chat memory to negotiate pricing, logs leads into a CRM spreadsheet, and sends automated replies via Gmail.

The architecture is divided into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures web form submissions, standardizes input fields, and assigns a unique deal ID.
- **1.2 Inventory Verification & Configuration:** Loads dealership settings and queries a Google Sheets database to check vehicle availability.
- **1.3 Conditional Routing & Fallback:** Evaluates whether the requested vehicle exists in inventory, triggering a "not found" email notification if missing, or proceeding to deal preparation if found.
- **1.4 AI Sales Agent & Negotiation Processing:** Combines a Gemini LLM, Redis chat memory, deterministic pricing tools, and structured output parsing to evaluate customer offers, check pricing boundaries, and generate personalized sales replies.
- **1.5 CRM Logging & Customer Response:** Appends or updates lead data in a Google Sheets tracker and sends the final AI-generated quote email to the customer.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization
- **Overview:** Listens for incoming HTTP form submissions from prospective buyers and extracts/cleans relevant contact and vehicle identification details.
- **Nodes Involved:** 
  - `When Customer Form Submitted`
  - `Normalize Form Input`

##### Node Details:
- **When Customer Form Submitted**
  - *Type & Role:* `n8n-nodes-base.formTrigger` (Webhook/Trigger)
  - *Configuration:* Publishes an interactive web form with fields for Car selection (dropdown), Name, Email, Phone, and Message.
  - *Inputs/Outputs:* Input: None (Trigger). Output: Raw form payload.
  - *Edge Cases:* Missing required fields (handled natively by form validation); malformed inputs.

- **Normalize Form Input**
  - *Type & Role:* `n8n-nodes-base.set` (Data Transformation)
  - *Configuration:* Maps raw form fields into a normalized schema, generates a unique deal ID using date formatting and random alphanumeric strings, and extracts the `CAR-XXXX` ID via regex.
  - *Key Expressions:* `={{ 'DEAL-' + $now.toFormat('yyMMdd') + '-' + Math.random().toString(36).slice(2, 7).toUpperCase() }}`, `={{ $json.Car.match(/CAR-\\d+/)?.[0] }}`
  - *Inputs/Outputs:* Input: `When Customer Form Submitted`. Output: Normalized dataset.
  - *Edge Cases:* Regex failure if the car dropdown value changes format.

---

#### Block 1.2: Inventory Verification & Configuration
- **Overview:** Injects dealership metadata (name, sales rep name, maximum allowable discount) and queries a Google Sheets inventory database to retrieve specifications and pricing for the requested car.
- **Nodes Involved:**
  - `Set Dealer Configuration`
  - `Read Inventory from Sheets`

##### Node Details:
- **Set Dealer Configuration**
  - *Type & Role:* `n8n-nodes-base.set` (Data Enrichment)
  - *Configuration:* Appends constants such as dealership name (`Summit Auto Group`), sales representative name (`Alex`), and maximum discount percentage (`8`).
  - *Inputs/Outputs:* Input: `Normalize Form Input`. Output: Enriched payload containing both form data and dealer config.

- **Read Inventory from Sheets**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (Database Lookup)
  - *Configuration:* Connects via Google Service Account to filter a specified Google Sheet (`Car Inventory`) matching the `car_id` column against the normalized input. Configured with `alwaysOutputData: true`.
  - *Inputs/Outputs:* Input: `Set Dealer Configuration`. Output: Vehicle record (year, make, model, trim, mileage, list_price, floor_price, status).
  - *Edge Cases:* API authentication failures, invalid spreadsheet IDs, or empty search results.

---

#### Block 1.3: Conditional Routing & Fallback
- **Overview:** Branches workflow execution depending on whether the vehicle exists in inventory.
- **Nodes Involved:**
  - `If Vehicle Found`
  - `Send Car Not Found Email`
  - `Prepare Deal Data`

##### Node Details:
- **If Vehicle Found**
  - *Type & Role:* `n8n-nodes-base.if` (Router)
  - *Configuration:* Checks if the `car_id` field in the inventory lookup result is not empty.
  - *Inputs/Outputs:* Input: `Read Inventory from Sheets`. Output: True branch proceeds to deal preparation; False branch triggers the fallback email.

- **Send Car Not Found Email**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email Dispatch)
  - *Configuration:* Sends a plain-text notification via Gmail OAuth2 to the customer stating that the vehicle ID could not be found.
  - *Key Expressions:* Uses `.first()` references to pull customer name, email, and car ID from the configuration node.
  - *Inputs/Outputs:* Input: `If Vehicle Found` (False). Output: Confirmation of sent email.
  - *Edge Cases:* OAuth2 token expiration, invalid recipient address.

- **Prepare Deal Data**
  - *Type & Role:* `n8n-nodes-base.set` (Data Preparation)
  - *Configuration:* Merges vehicle specifications, cleans monetary fields by stripping non-numeric characters, and constructs a unified `vehicle_name` string.
  - *Key Expressions:* `={{ Number(String($json.list_price).replace(/[^0-9.]/g, '')) }}`, `={{ [$json.year, $json.make, $json.model, $json.trim].join(' ') }}`
  - *Inputs/Outputs:* Input: `If Vehicle Found` (True). Output: Cleaned deal object ready for AI processing.

---

#### Block 1.4: AI Sales Agent & Negotiation Processing
- **Overview:** Orchestrates a LangChain-based AI agent empowered with memory, determinism tools, and a structured output parser to manage the vehicle price negotiation.
- **Nodes Involved:**
  - `Car Sales AI Agent`
  - `Gemini Flash-Lite Model`
  - `Redis Chat Memory`
  - `Fetch Vehicle Details Tool`
  - `Calculate Offer Tool`
  - `Parse Structured Output`

##### Node Details:
- **Car Sales AI Agent**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Orchestrator)
  - *Configuration:* Configured as a prompt-defined agent with a maximum of 5 iterations. Enforces safety rules, pricing boundaries, and conversational tone via a comprehensive system prompt.
  - *Inputs/Outputs:* Input: `Prepare Deal Data`, Model, Memory, Tools, Output Parser. Output: JSON payload containing agent response properties.
  - *Edge Cases:* Iteration limit exceeded, hallucinated pricing, or tool execution errors.

- **Gemini Flash-Lite Model**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Language Model)
  - *Configuration:* Uses `models/gemini-3.5-flash-lite` with a temperature of `0.4` and a maximum output token limit of `800`.
  - *Credentials:* Google PaLM API.

- **Redis Chat Memory**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.memoryRedisChat` (Conversation State)
  - *Configuration:* Maintains session state mapped to the customer's email address (`sessionKey`) with a Time-To-Live (TTL) of 604,800 seconds and a context window length of 20 messages.
  - *Credentials:* Redis account.

- **Fetch Vehicle Details Tool**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.toolCode` (Custom Code Tool)
  - *Configuration:* Returns JSON containing public vehicle specifications, list price, and status.
  - *Inputs/Outputs:* Invoked by the AI Agent.

- **Calculate Offer Tool**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.toolCode` (Custom Code Tool)
  - *Configuration:* Executes a deterministic JavaScript engine that evaluates customer offers against vehicle list prices, floor prices, and maximum allowed dealer discounts. Returns decision codes (`ACCEPT`, `COUNTER`, `UNAVAILABLE`, `INVALID_OFFER`) along with approved quoting limits.
  - *Inputs/Outputs:* Invoked by the AI Agent.

- **Parse Structured Output**
  - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Schema Enforcement)
  - *Configuration:* Enforces a strict JSON schema containing `query` (boolean), `subject` (string), and `email_body_in_HTML` (string).
  - *Inputs/Outputs:* Feeds parsing rules into the AI Agent.

---

#### Block 1.5: CRM Logging & Customer Response
- **Overview:** Validates the AI response state, logs the transaction details to a secondary Google Sheet, and emails the negotiated response back to the customer.
- **Nodes Involved:**
  - `If Valid Query`
  - `Append or Update Lead in Sheets`
  - `Send Customer Reply Email`

##### Node Details:
- **If Valid Query**
  - *Type & Role:* `n8n-nodes-base.if` (Router)
  - *Configuration:* Evaluates whether `{{ $json.output.query }}` evaluates to `true`, ensuring leads are recorded only during active negotiations or follow-ups.
  - *Inputs/Outputs:* Input: `Car Sales AI Agent`. Output: True branch routes to lead logging.

- **Append or Update Lead in Sheets**
  - *Type & Role:* `n8n-nodes-base.googleSheets` (CRM Integration)
  - *Configuration:* Appends or updates rows in the `Car Inquiry Leads` spreadsheet (`Sheet1`) matched by `customer_email`. Maps deal metadata, customer contact details, and agent outputs.
  - *Credentials:* Google Service Account.
  - *Inputs/Outputs:* Input: `If Valid Query` (True). Output: Updated spreadsheet row confirmation.

- **Send Customer Reply Email**
  - *Type & Role:* `n8n-nodes-base.gmail` (Email Dispatch)
  - *Configuration:* Sends the AI-generated HTML response via Gmail OAuth2 to the customer's email address. Sets a dynamic subject line incorporating vehicle details, deal ID, and car ID.
  - *Key Expressions:* `={{ $json.agent_output.email_body_in_HTML }}`
  - *Credentials:* Gmail OAuth2.
  - *Inputs/Outputs:* Input: `Append or Update Lead in Sheets`. Output: Email sent confirmation.
  - *Edge Cases:* Unescaped double quotes in HTML body, invalid recipient addresses, or Gmail sending limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | *None* | *None* | ## Auto Dealership AI Quotation Agent... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Block documentation: Receive and normalize customer form | *None* | *None* | ## Receive and normalize customer form... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Block documentation: Configure dealer and check inventory | *None* | *None* | ## Configure dealer and check inventory... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Block documentation: Verify vehicle and prepare deal | *None* | *None* | ## Verify vehicle and prepare deal... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Block documentation: Process AI sales agent and respond | *None* | *None* | ## Process AI sales agent and respond... |
| When Customer Form Submitted | `n8n-nodes-base.formTrigger` | Triggers workflow on customer form submission | *None* | Normalize Form Input | ## Receive and normalize customer form... |
| Normalize Form Input | `n8n-nodes-base.set` | Normalizes form payload and generates deal ID | When Customer Form Submitted | Set Dealer Configuration | ## Receive and normalize customer form... |
| Set Dealer Configuration | `n8n-nodes-base.set` | Injects dealership configurations and rules | Normalize Form Input | Read Inventory from Sheets | ## Configure dealer and check inventory... |
| Read Inventory from Sheets | `n8n-nodes-base.googleSheets` | Queries inventory sheet by Car ID | Set Dealer Configuration | If Vehicle Found | ## Configure dealer and check inventory... |
| If Vehicle Found | `n8n-nodes-base.if` | Branches execution based on inventory match | Read Inventory from Sheets | Prepare Deal Data, Send Car Not Found Email | ## Verify vehicle and prepare deal... |
| Send Car Not Found Email | `n8n-nodes-base.gmail` | Sends fallback email if vehicle is missing | If Vehicle Found | *None* | ## Verify vehicle and prepare deal... |
| Prepare Deal Data | `n8n-nodes-base.set` | Cleans prices and formats vehicle names | If Vehicle Found | Car Sales AI Agent | ## Verify vehicle and prepare deal... |
| Gemini Flash-Lite Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides LLM backend for sales agent | *None* | Car Sales AI Agent | ## Process AI sales agent and respond... |
| Redis Chat Memory | `@n8n/n8n-nodes-langchain.memoryRedisChat` | Manages persistent conversation history | *None* | Car Sales AI Agent | ## Process AI sales agent and respond... |
| Fetch Vehicle DetailsTool | `@n8n/n8n-nodes-langchain.toolCode` | Tool providing vehicle specifications | *None* | Car Sales AI Agent | ## Process AI sales agent and respond... |
| Calculate Offer Tool | `@n8n/n8n-nodes-langchain.toolCode` | Deterministic pricing and margin calculation tool | *None* | Car Sales AI Agent | ## Process AI sales agent and respond... |
| Parse Structured Output | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output schema for agent responses | *None* | Car Sales AI Agent | ## Process AI sales agent and respond... |
| Car Sales AI Agent | `@n8n/n8n-nodes-agent` | Orchestrates AI negotiation and tool execution | Prepare Deal Data, Gemini Flash-Lite Model, Redis Chat Memory, Fetch Vehicle Details Tool, Calculate Offer Tool, Parse Structured Output | If Valid Query | ## Process AI sales agent and respond... |
| If Valid Query | `n8n-nodes-base.if` | Verifies whether the agent generated an active query | Car Sales AI Agent | Append or Update Lead in Sheets | ## Process AI sales agent and respond... |
| Append or Update Lead in Sheets | `n8n-nodes-base.googleSheets` | Logs or updates lead details in CRM sheet | If Valid Query | Send Customer Reply Email | ## Process AI sales agent and respond... |
| Send Customer Reply Email | `n8n-nodes-base.gmail` | Sends HTML quote response to customer | Append or Update Lead in Sheets | *None* | ## Process AI sales agent and respond... |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node named `When Customer Form Submitted`.
   - Configure form fields: `Car` (Dropdown with options: `Toyota Camry XSE CAR-1042`, `Honda Accord Touring CAR-1088`, `Ford Mustang GT CAR-1127`), `Name`, `Email`, `Phone`, and `Message` (Textarea).

2. **Add Input Normalization:**
   - Add a **Set** node named `Normalize Form Input`. Connect `When Customer Form Submitted` to it.
   - Add string assignments to generate `deal_id`, extract `car_id` via regex, and format customer details.

3. **Configure Dealership Settings:**
   - Add a **Set** node named `Set Dealer Configuration`. Connect `Normalize Form Input` to it.
   - Assign fields: `dealership_name` ("Summit Auto Group"), `sales_rep_name` ("Alex"), and `max_discount_percent` (8). Enable "Include Other Fields".

4. **Set Up Inventory Lookup:**
   - Add a **Google Sheets** node named `Read Inventory from Sheets`. Connect `Set Dealer Configuration` to it.
   - Set operation to `Lookup`. Configure document ID and sheet name (`Sheet1`). Set filter: lookup column `car_id` equal to `={{ $json.car_id }}`. Enable `Always Output Data`.

5. **Branch Based on Availability:**
   - Add an **If** node named `If Vehicle Found`. Connect `Read Inventory from Sheets` to it.
   - Set condition: Left value `={{ $json.car_id }}` is not empty.

6. **Configure Fallback Path (Vehicle Not Found):**
   - Add a **Gmail** node named `Send Car Not Found Email`. Connect the `false` output of `If Vehicle Found` to it.
   - Configure action to `Send`, set recipient to customer email, and compose a fallback message stating the vehicle ID could not be found.

7. **Prepare Deal Data (Vehicle Found):**
   - Add a **Set** node named `Prepare Deal Data`. Connect the `true` output of `If Vehicle Found` to it.
   - Assign cleaned numeric values for `mileage`, `list_price`, `floor_price`, and compile `vehicle_name`.

8. **Build the AI Agent & Supporting Nodes:**
   - Add a **Google Gemini Chat Model** node (`Gemini Flash-Lite Model`), set model to `models/gemini-3.5-flash-lite`, temperature `0.4`, and configure Google PaLM API credentials.
   - Add a **Redis Chat Memory** node (`Redis Chat Memory`), set session key to `={{ $json.customer_email }}`, TTL to `604800`, and configure Redis credentials.
   - Add a **Tool (Code)** node named `Fetch Vehicle Details Tool`. Insert JavaScript returning vehicle specifications from `Prepare Deal Data`.
   - Add a **Tool (Code)** node named `Calculate Offer Tool`. Insert the pricing negotiation logic script that evaluates customer offers against floor prices and maximum discounts.
   - Add a **Structured Output Parser** node (`Parse Structured Output`), configured with the manual JSON schema requiring `query`, `subject`, and `email_body_in_HTML`.
   - Add an **AI Agent** node (`Car Sales AI Agent`), set prompt type to define, map input text to `={{ $('Prepare Deal Data').first().json.customer_message }}`, and connect the Language Model, Memory, both Tools, and Output Parser. Connect `Prepare Deal Data` to its main input.

9. **Configure Response Handling & CRM Logging:**
   - Add an **If** node named `If Valid Query`. Connect `Car Sales AI Agent` to it. Set condition to check `={{ $json.output.query }}` equals `true`.
   - Add a **Google Sheets** node named `Append or Update Lead in Sheets`. Connect the `true` output of `If Valid Query` to it. Set operation to `Append or Update`, matching on `customer_email`, and map lead parameters.
   - Add a **Gmail** node named `Send Customer Reply Email`. Connect `Append or Update Lead in Sheets` to it. Set recipient to customer email, subject to dynamic deal/vehicle reference, and message body to `={{ $json.agent_output.email_body_in_HTML }}`. Configure Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Youtube Video | [https://youtu.be/0B5IAirPe5o](https://youtu.be/0B5IAirPe5o) |
| Source attribution disclaimer | Provided exclusively from an automated n8n workflow; complies fully with content and data policies. |