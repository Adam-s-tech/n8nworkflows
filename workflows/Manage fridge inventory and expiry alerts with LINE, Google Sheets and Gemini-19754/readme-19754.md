Manage fridge inventory and expiry alerts with LINE, Google Sheets and Gemini

https://n8nworkflows.xyz/workflows/manage-fridge-inventory-and-expiry-alerts-with-line--google-sheets-and-gemini-19754


# Manage fridge inventory and expiry alerts with LINE, Google Sheets and Gemini

### 1. Workflow Overview

This workflow automates fridge inventory management using LINE messaging, Google Sheets, and Google Gemini AI. It functions through two independent entry points: an interactive item-registration flow triggered by user images sent via LINE, and a daily automated check that detects expiring ingredients, generates meal suggestions using AI, and sends proactive notifications.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception:** Listens for incoming webhook events from LINE and downloads the attached food package image.
- **1.2 AI Processing & Inventory Registration:** Analyzes the image using Google Gemini to extract structured food details, appends them to a Google Sheets inventory, and replies to the user via LINE.
- **1.3 Automated Expiry Monitoring:** Triggers daily at 9:00 AM, reads the inventory from Google Sheets, and filters items approaching their expiration dates.
- **1.4 AI Meal Generation & Notification:** Aggregates expiring items, uses Google Gemini to generate concise Japanese meal suggestions, and pushes a notification to the user via LINE.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
This block captures inbound webhook requests from the LINE Messaging API whenever a user sends an image, retrieving the corresponding binary file for subsequent analysis.

- **Nodes Involved:** `LINE Webhook Trigger`, `Fetch LINE Image`
- **Node Details:**
  - **LINE Webhook Trigger**
    - *Type and Technical Role:* Webhook node (`n8n-nodes-base.webhook`), acts as the entry point for HTTP POST requests originating from LINE.
    - *Configuration Choices:* Path set to `line-fridge`, HTTP method set to `POST`.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input: None (Trigger); Output: Connects to `Fetch LINE Image`.
    - *Edge Cases / Potential Failure Types:* Webhook URL misconfiguration in the LINE developer console; invalid payload structure if non-image messages are sent.
  - **Fetch LINE Image**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`), fetches binary image content from the LINE API.
    - *Configuration Choices:* Uses generic HTTP Header Authentication. Response format configured as file/binary.
    - *Key Expressions or Variables:* `={{"https://api-data.line.me/v2/bot/message/" + $('LINE Webhook Trigger').first().json.body.events[0].message.id + "/content"}}`
    - *Input/Output Connections:* Input: `LINE Webhook Trigger`; Output: Connects to `Food Info Extraction Agent`.
    - *Edge Cases / Potential Failure Types:* Expired or invalid LINE Channel Access Token; network timeouts when downloading large images.

#### 2.2 AI Processing & Inventory Registration
This block processes the downloaded food package image through an AI agent, extracts structured metadata, persists the record into Google Sheets, and confirms registration back to the user via LINE.

- **Nodes Involved:** `Food Info Extraction Agent`, `Gemini Image Reading Model`, `Append Food to Sheets`, `Reply to LINE Registration`
- **Node Details:**
  - **Food Info Extraction Agent**
    - *Type and Technical Role:* LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`), processes images using structured prompts to extract specific food attributes.
    - *Configuration Choices:* Prompt type set to define. Configured with a system prompt enforcing exact English field labels, Japanese field values, and ISO-8601 date formatting. Retry enabled on failure.
    - *Key Expressions or Variables:* None (relies on attached binary image input).
    - *Input/Output Connections:* Input: `Fetch LINE Image`, `Gemini Image Reading Model`; Output: Connects to `Append Food to Sheets`.
    - *Edge Cases / Potential Failure Types:* Unreadable text on packaging resulting in fallback values ("不明" / "読み取れません").
  - **Gemini Image Reading Model**
    - *Type and Technical Role:* Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`), provides the LLM backend for the extraction agent.
    - *Configuration Choices:* Model name set to `models/gemini-2.5-flash`.
    - *Input/Output Connections:* Input: Google Gemini API credentials; Output: Connects to `Food Info Extraction Agent` via `ai_languageModel`.
  - **Append Food to Sheets**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`), appends new inventory rows.
    - *Configuration Choices:* Operation set to `append`, mapping mode set to define below, targeting the "Inventory" sheet.
    - *Key Expressions or Variables:* 
      - `Registered At`: `=$now.toFormat('yyyy-LL-dd HH:mm')`
      - `Food Name`: `={{ $json.output.match(/Food Name:\s*(.+)/)?.[1]?.trim() }}`
      - `Expiration Date`: `={{ $json.output.match(/Expiration Date:\s*(.+)/)?.[1]?.trim() }}`
      - `Storage Location`: `={{ $json.output.match(/Storage Location:\s*(.+)/)?.[1]?.trim() }}`
      - `Status`: `"In Stock"`
      - `Notes`: `={{ $json.output.match(/Notes:\s*(.+)/)?.[1]?.trim() }}`
      - `AI Output`: `={{ $json.output }}`
      - `LINE User ID`: `={{ $('LINE Webhook Trigger').first().json.body.events[0].source.userId }}`
    - *Input/Output Connections:* Input: `Food Info Extraction Agent`; Output: Connects to `Reply to LINE Registration`.
    - *Edge Cases / Potential Failure Types:* Google Sheets API rate limits or authentication token expiration; regex matching failures if the LLM deviates from the output format.
  - **Reply to LINE Registration**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`), sends a reply message to the LINE user.
    - *Configuration Choices:* POST request to `https://api.line.me/v2/bot/message/reply`. Uses generic HTTP Header Authentication.
    - *Key Expressions or Variables:* Uses `replyToken` from the webhook trigger and extracts registered property values from `Append Food to Sheets`.
    - *Input/Output Connections:* Input: `Append Food to Sheets`; Output: None.
    - *Edge Cases / Potential Failure Types:* Expired reply tokens (LINE reply tokens expire shortly after issuance).

#### 2.3 Automated Expiry Monitoring
This block executes on a daily schedule to retrieve current inventory records and filter items that are currently in stock and nearing their expiration date.

- **Nodes Involved:** `When Daily at 9am`, `Read Inventory in Sheets`, `Filter Soon to Expire`, `Collect Expiring Ingredients`
- **Node Details:**
  - **When Daily at 9am**
    - *Type and Technical Role:* Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`), initiates the daily workflow execution.
    - *Configuration Choices:* Interval rule configured to trigger at hour 9.
    - *Input/Output Connections:* Input: None (Trigger); Output: Connects to `Read Inventory in Sheets`.
  - **Read Inventory in Sheets**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`), reads all rows from the inventory spreadsheet.
    - *Configuration Choices:* Operation set to default read (get rows), targeting the "Inventory" sheet.
    - *Input/Output Connections:* Input: `When Daily at 9am`; Output: Connects to `Filter Soon to Expire`.
  - **Filter Soon to Expire**
    - *Type and Technical Role:* Filter node (`n8n-nodes-base.filter`), isolates inventory items matching specific criteria.
    - *Configuration Choices:* Combinator set to `and` with three conditions:
      1. `状態` (Status) equals `在庫あり` (In Stock)
      2. `賞味期限` (Expiration Date) is after or equals current date (`$now.toFormat('yyyy-LL-dd')`)
      3. `賞味期限` (Expiration Date) is before or equals 3 days from current date (`$now.plus({ days: 3 }).toFormat('yyyy-LL-dd')`)
    - *Input/Output Connections:* Input: `Read Inventory in Sheets`; Output: Connects to `Collect Expiring Ingredients`.
    - *Edge Cases / Potential Failure Types:* Date format mismatches between spreadsheet entries and expression evaluations.
  - **Collect Expiring Ingredients**
    - *Type and Technical Role:* Aggregate node (`n8n-nodes-base.aggregate`), merges filtered items into a single consolidated list.
    - *Configuration Choices:* Aggregate type set to `aggregateAllItemData`, destination field named `data`.
    - *Input/Output Connections:* Input: `Filter Soon to Expire`; Output: Connects to `Meal Suggestion Agent`.

#### 2.4 AI Meal Generation & Notification
This block transforms the aggregated expiring inventory list into concise Japanese meal ideas via Google Gemini and dispatches a push notification to the user via LINE.

- **Nodes Involved:** `Meal Suggestion Agent`, `Gemini Meal Suggestion Model`, `Notify LINE on Expiry`
- **Node Details:**
  - **Meal Suggestion Agent**
    - *Type and Technical Role:* LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`), compiles meal recommendations based on structured inventory data.
    - *Configuration Choices:* Prompt type set to define. Includes strict formatting rules (Japanese language, max 250 characters, no markdown, no asterisks, no emojis).
    - *Key Expressions or Variables:* `={{ JSON.stringify($json.data, null, 2) }}`
    - *Input/Output Connections:* Input: `Collect Expiring Ingredients`, `Gemini Meal Suggestion Model`; Output: Connects to `Notify LINE on Expiry`.
  - **Gemini Meal Suggestion Model**
    - *Type and Technical Role:* Google Gemini Chat Model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`), provides the LLM backend for the meal suggestion agent.
    - *Configuration Choices:* Model name set to `models/gemini-2.5-flash`.
    - *Input/Output Connections:* Input: Google Gemini API credentials; Output: Connects to `Meal Suggestion Agent` via `ai_languageModel`.
  - **Notify LINE on Expiry**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`), sends a LINE push notification containing meal ideas.
    - *Configuration Choices:* POST request to `https://api.line.me/v2/bot/message/push`. Uses generic HTTP Header Authentication.
    - *Key Expressions or Variables:* 
      - Recipient ID: `={{ $('Collect Expiring Ingredients').first().json.data[0]["LINE userId"] }}`
      - Message Text: Combines static header with `={{ $json.output }}`.
    - *Input/Output Connections:* Input: `Meal Suggestion Agent`; Output: None.
    - *Edge Cases / Potential Failure Types:* Push messaging limits or missing `LINE userId` if manual entries lack user IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Overview documentation and setup instructions. | None | None | ## AI Fridge Inventory Management Agent<br><br>### How it works<br><br>This workflow manages a fridge inventory through LINE and Google Sheets. When a user sends a food image to LINE, it downloads the image, uses Gemini to extract food details, stores them in a sheet, and replies with a registration confirmation. On a daily schedule, it reads the inventory, filters items nearing expiry, asks Gemini for meal suggestions, and pushes a LINE notification.<br><br>### Setup steps<br><br>- Configure the LINE webhook URL to point to the n8n Webhook node and add the required LINE Messaging API access token to the HTTP Request nodes.<br>- Connect Google Sheets credentials and select the spreadsheet/sheet used for the fridge inventory, including columns for food name, quantity, registration date, and expiry date.<br>- Configure Google Gemini credentials for both AI Agent model sub-nodes: image reading and meal suggestion generation.<br>- Set the Daily Check schedule frequency and configure the LINE recipient or group ID for expiry push notifications.<br><br>### Customization<br><br>Adjust the expiry filter threshold, the inventory sheet schema, and the AI prompts for food extraction or meal suggestions to match household preferences. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Explains LINE image reception block. | None | None | ## Receive LINE image<br><br>Starts the image-registration flow from a LINE webhook and downloads the submitted image content for processing. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Explains food extraction AI block. | None | None | ## Extract food details<br><br>Uses an AI agent with a Gemini image-reading model to identify food information from the downloaded LINE image. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Explains saving and confirmation block. | None | None | ## Save and confirm item<br><br>Adds the extracted food details to the Google Sheets inventory and sends a LINE reply confirming registration. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Explains expiry monitoring block. | None | None | ## Find expiring inventory<br><br>Runs the scheduled daily check, retrieves inventory rows from Google Sheets, filters expiring items, and aggregates them for AI processing. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Explains meal suggestion and notification block. | None | None | ## Suggest meals and notify<br><br>Generates meal ideas for soon-to-expire items using Gemini and sends the resulting expiry notification through LINE. |
| LINE Webhook Trigger | `n8n-nodes-base.webhook` | Receives POST webhook events from LINE. | None | Fetch LINE Image | |
| Fetch LINE Image | `n8n-nodes-base.httpRequest` | Downloads binary image content from LINE. | LINE Webhook Trigger | Food Info Extraction Agent | |
| Append Food to Sheets | `n8n-nodes-base.googleSheets` | Appends extracted food details to inventory sheet. | Food Info Extraction Agent | Reply to LINE Registration | |
| Reply to LINE Registration | `n8n-nodes-base.httpRequest` | Sends confirmation reply message via LINE. | Append Food to Sheets | None | |
| When Daily at 9am | `n8n-nodes-base.scheduleTrigger` | Triggers the daily inventory check at 9:00 AM. | None | Read Inventory in Sheets | |
| Read Inventory in Sheets | `n8n-nodes-base.googleSheets` | Reads all rows from the inventory Google Sheet. | When Daily at 9am | Filter Soon to Expire | |
| Filter Soon to Expire | `n8n-nodes-base.filter` | Filters rows for in-stock items expiring within 3 days. | Read Inventory in Sheets | Collect Expiring Ingredients | |
| Collect Expiring Ingredients | `n8n-nodes-base.aggregate` | Consolidates filtered expiring items into a single array. | Filter Soon to Expire | Meal Suggestion Agent | |
| Meal Suggestion Agent | `@n8n/n8n-nodes-langchain.agent` | Generates Japanese meal ideas from expiring items. | Collect Expiring Ingredients, Gemini Meal Suggestion Model | Notify LINE on Expiry | |
| Notify LINE on Expiry | `n8n-nodes-base.httpRequest` | Sends expiry push notification via LINE. | Meal Suggestion Agent | None | |
| Gemini Meal Suggestion Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM model for meal suggestion generation. | None | Meal Suggestion Agent | |
| Food Info Extraction Agent | `@n8n/n8n-nodes-langchain.agent` | Extracts structured food details from food images. | Fetch LINE Image, Gemini Image Reading Model | Append Food to Sheets | |
| Gemini Image Reading Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM model for image parsing and text extraction. | None | Food Info Extraction Agent | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`).
   - Set HTTP Method to `POST` and Path to `line-fridge`.
2. **Add LINE Image Fetcher**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Configure authentication using HTTP Header Auth (`formula-line-fridge` credential with LINE Channel Access Token).
   - Set URL to `={{"https://api-data.line.me/v2/bot/message/" + $('LINE Webhook Trigger').first().json.body.events[0].message.id + "/content"}}`.
   - Set response format to file. Connect input from **LINE Webhook Trigger**.
3. **Configure Food Extraction Agent**
   - Add a **LangChain Agent** (`@n8n/n8n-nodes-langchain.agent`) named `Food Info Extraction Agent`.
   - Add a **Google Gemini Chat Model** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) configured with model `models/gemini-2.5-flash` and connect it to the agent's AI language model input. Configure with Google Gemini API credentials.
   - Set the agent prompt to enforce extraction of `Food Name`, `Expiration Date`, `Storage Location`, and `Notes` in Japanese format.
   - Connect input from **Fetch LINE Image**.
4. **Append Data to Google Sheets**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Configure Google Sheets OAuth2 credentials and select the target document and sheet ("Inventory").
   - Set operation to `append` and map columns using expressions to parse output fields (`Food Name`, `Expiration Date`, `Storage Location`, `Status` as "In Stock", `Notes`, `AI Output`, `LINE User ID`, and `Registered At`).
   - Connect input from **Food Info Extraction Agent**.
5. **Send LINE Reply Confirmation**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Configure authentication with HTTP Header Auth (LINE credential).
   - Set URL to `https://api.line.me/v2/bot/message/reply`, method to `POST`.
   - Provide a JSON body extracting `replyToken` and constructing a confirmation message using data from the sheet/webhook.
   - Connect input from **Append Food to Sheets**.
6. **Set Up Daily Schedule Trigger**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to trigger at hour 9.
7. **Read and Filter Inventory**
   - Add a **Google Sheets** node to read rows from the inventory sheet, connected to **When Daily at 9am**.
   - Add a **Filter** node (`n8n-nodes-base.filter`) connected to the Google Sheets read node.
   - Configure filter conditions: Status equals `在庫あり`, Expiration Date is after or equals `$now.toFormat('yyyy-LL-dd')`, and Expiration Date is before or equals `$now.plus({ days: 3 }).toFormat('yyyy-LL-dd')`.
   - Add an **Aggregate** node (`n8n-nodes-base.aggregate`) set to `aggregateAllItemData` with destination field `data`, connected from the filter node.
8. **Configure Meal Suggestion Agent & Notification**
   - Add a **LangChain Agent** (`@n8n/n8n-nodes-langchain.agent`) named `Meal Suggestion Agent`.
   - Add a second **Google Gemini Chat Model** node configured with `models/gemini-2.5-flash` and connect it to the agent.
   - Set the agent prompt to generate Japanese meal ideas under strict formatting constraints based on input JSON data.
   - Connect input from the Aggregate node.
   - Add an **HTTP Request** node configured for LINE push notifications (`https://api.line.me/v2/bot/message/push`), using HTTP Header Auth, sending a JSON body containing the recipient's `LINE userId` and the generated AI output text. Connect input from **Meal Suggestion Agent**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow utilizes the LINE Messaging API webhook and push/reply endpoints. | Requires an active LINE Channel Access Token configured in n8n credentials. |
| Integrates with Google Gemini (PaLM) API models for multimodal image reading and text generation. | Requires valid Google Gemini API credentials. |
| Relies on a specific Google Sheets schema (Registered At, Food Name, Expiration Date, Storage Location, Status, Notes, AI Output, LINE User ID). | Ensure column headers and sheet names match workflow parameter mappings. |