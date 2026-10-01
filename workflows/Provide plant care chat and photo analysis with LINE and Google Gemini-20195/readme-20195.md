Provide plant care chat and photo analysis with LINE and Google Gemini

https://n8nworkflows.xyz/workflows/provide-plant-care-chat-and-photo-analysis-with-line-and-google-gemini-20195


# Provide plant care chat and photo analysis with LINE and Google Gemini

### 1. Workflow Overview

This workflow powers a LINE-based AI assistant designed to identify plants, diagnose plant health from uploaded images, and answer natural language plant care questions. It utilizes Google Gemini for multimodal AI analysis and text generation, and four n8n Data Tables (`plant_users`, `plants`, `plant_observations`, and `conversations`) to persist user state, profiles, diagnostic observations, and conversation history.

The logic is grouped into the following functional blocks:

- **1.1 Input Reception & Routing:** Accepts webhook events from the LINE Messaging API and routes incoming events based on their message type (`image` or `text`).
- **1.2 Image Ingestion & Context Enrichment:** Downloads binary image payloads from LINE, queries the user profile, active plant records, and past chat history, and aggregates them into a unified context payload.
- **1.3 AI Plant Image Analysis:** Sends the aggregated context and binary image to Google Gemini (`gemini-3.8-flash`) to parse botanical features, health statuses, and potential symptoms.
- **1.4 Plant Profile Upsert & Matching:** Branches execution depending on whether the analyzed plant matches the user's active plant, represents a new plant, or is too uncertain to categorize, performing database insertions or updates accordingly.
- **1.5 Observation Storage & Image Reply:** Saves confirmed observations to the database, formats a multilingual diagnostic summary (supporting English and Japanese), and posts the response back to LINE.
- **1.6 Text Chat Ingestion & State Retrieval:** Loads the chat user profile, retrieves the active plant, queries historical plant observations and conversation logs, and aggregates them for text interactions.
- **1.7 AI Chat Generation & Storage:** Invokes Google Gemini (`gemini-3-flash-preview`) to generate contextual care responses and persists both user input and assistant output to conversation history.
- **1.8 Language Preference Management & Chat Reply:** Evaluates whether an explicit language change was requested, updates user language preferences when necessary, and sends the final text reply via LINE.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Routing
- **Overview:** Accepts HTTP POST requests from the LINE Messaging API webhook and branches the processing stream depending on whether the user sent an image or a text message.
- **Nodes Involved:** 
  - `LINE Webhook Trigger`
  - `Route by Message Type`
- **Node Details:**
  - **LINE Webhook Trigger**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook entry point)
    - *Configuration:* HTTP method set to `POST`, webhook path configured to `plantcare-line-webhook`.
    - *Inputs / Outputs:* Input: External LINE API POST request. Output: Raw webhook event payload.
    - *Edge Cases / Failure Types:* Webhook endpoint URL misconfiguration in the LINE Developer Console.
  - **Route by Message Type**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Conditional router)
    - *Configuration:* Evaluates `{{ $('LINE Webhook Trigger').item.json.body.events[0].message.type }}` against string values `image` (Output 0) and `text` (Output 1).
    - *Inputs / Outputs:* Input: LINE Webhook event. Outputs: Image branch or Text branch.
    - *Edge Cases / Failure Types:* Unsupported message types (e.g., stickers, audio) will fail matching unless unhandled paths are managed.

---

#### Block 1.2: Image Ingestion & Context Enrichment
- **Overview:** Downloads the binary image file from LINE's content servers, queries user state, active plant records, and previous chat logs, then merges everything into a structured context object.
- **Nodes Involved:**
  - `Fetch LINE Image`
  - `Query Image User`
  - `Retrieve Image Active Plant`
  - `Fetch Image Chat History`
  - `Aggregate Image Chat History`
  - `Build Image Context`
  - `Combine Image Context Data`
- **Node Details:**
  - **Fetch LINE Image**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Binary downloader)
    - *Configuration:* HTTP GET request to `https://api-data.line.me/v2/bot/message/{messageId}/content`. Response format set to `file`. Uses Generic HTTP Header Authentication.
    - *Credentials:* `Header Auth account` (LINE Channel Access Token).
    - *Edge Cases / Failure Types:* Expired message IDs or invalid access tokens.
  - **Query Image User**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Queries `plant_users` table where `line_user_id` matches the incoming LINE event sender ID. `Always Output Data` enabled.
  - **Retrieve Image Active Plant**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Queries `plants` table using `active_plant_id` retrieved from the user record. Defaults to `__NO_ACTIVE_PLANT__` if unassigned.
  - **Fetch Image Chat History**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Retrieves conversation logs from the `conversations` table filtered by `line_user_id`.
  - **Aggregate Image Chat History**
    - *Type & Technical Role:* `n8n-nodes-base.aggregate` (Data aggregator)
    - *Configuration:* Aggregates all item data into an array destination field named `conversation_history`.
  - **Build Image Context**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Maps user ID, preferred language, active plant object, and filtered conversation history into explicit properties.
  - **Combine Image Context Data**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Combiner)
    - *Configuration:* Mode set to `combine` using `combineByPosition` to merge the context object with the binary image file from `Fetch LINE Image`.

---

#### Block 1.3: AI Plant Image Analysis
- **Overview:** Sends the combined image payload and contextual metadata to Google Gemini to evaluate plant identification, health status, and symptoms, normalizing the JSON response.
- **Nodes Involved:**
  - `Image Analysis Agent`
  - `Set Plant Analysis Data`
- **Node Details:**
  - **Image Analysis Agent**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (Multimodal AI node)
    - *Configuration:* Resource: `image`, Operation: `analyze`, Model: `models/gemini-3.8-flash`. Input type set to `binary`. System prompt enforces strict JSON schema output including `is_plant`, `context_match`, `common_name`, `scientific_name`, `confidence`, `health_status`, `visible_symptoms`, `possible_causes`, `care_advice`, `needs_another_photo`, and `photo_request_reason`.
    - *Credentials:* `PlantCare AI - Gemini` (Google Palm API).
    - *Edge Cases / Failure Types:* Malformed JSON outputs from the LLM or API rate limits.
  - **Set Plant Analysis Data**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Parses the raw text response from Gemini into a structured JavaScript object: `={{ JSON.parse($('Image Analysis Agent').item.json.content.parts[0].text) }}`.

---

#### Block 1.4: Plant Profile Upsert & Matching
- **Overview:** Evaluates the `context_match` property from the AI analysis to determine whether to update the active plant, record a new plant entity, or classify the image as uncertain.
- **Nodes Involved:**
  - `Route by Image Context`
  - `Query Existing Plant`
  - `Check Plant Existence`
  - `Update Existing Plant`
  - `Record New Plant`
  - `Merge Plant Data`
  - `Set Active Plant Data`
- **Node Details:**
  - **Route by Image Context**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Conditional router)
    - *Configuration:* Branches execution based on `$json.analysis.context_match`: `same_active_plant` (Output 0), `new_plant` (Output 1), and `uncertain` (Output 2).
  - **Query Existing Plant**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Queries `plants` table matching `line_user_id` and the analyzed `scientific_name`.
  - **Check Plant Existence**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional checker)
    - *Configuration:* Evaluates whether `$json.plant_id` exists.
  - **Update Existing Plant**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database update)
    - *Configuration:* Operation: `update`. Updates `common_name`, `scientific_name`, `identification_confidence`, and `updated_at`.
  - **Record New Plant**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database insertion)
    - *Configuration:* Operation: `append`. Creates a new record in `plants` with a generated composite `plant_id`.
  - **Merge Plant Data**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Combiner)
    - *Configuration:* Merges data streams from either `Update Existing Plant` or `Record New Plant`.
  - **Set Active Plant Data**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Sets active plant properties when `context_match` evaluates to `same_active_plant`.

---

#### Block 1.5: Observation Storage & Image Reply
- **Overview:** Links the user to the plant, logs plant diagnostic observations to the data table, generates localized reply text, and transmits the response back to LINE.
- **Nodes Involved:**
  - `Query Plant User`
  - `Check Plant User Existence`
  - `Update Plant User Data`
  - `Record New Plant User`
  - `Merge Plant User Data`
  - `Store Plant Observation`
  - `Set Plant Reply Text`
  - `Set Uncertain Image Reply`
  - `Post Plant Analysis Reply`
- **Node Details:**
  - **Query Plant User**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Queries `plant_users` table by `line_user_id`.
  - **Check Plant User Existence**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional checker)
    - *Configuration:* Evaluates whether `$json.line_user_id` exists.
  - **Update Plant User Data**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database update)
    - *Configuration:* Operation: `update`. Updates `active_plant_id` and `updated_at`.
  - **Record New Plant User**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database insertion)
    - *Configuration:* Operation: `append`. Creates a new entry in `plant_users` with default preferred language set to `en`.
  - **Merge Plant User Data**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Combiner)
    - *Configuration:* Merges user update/creation branches.
  - **Store Plant Observation**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database insertion)
    - *Configuration:* Operation: `append`. Inserts observation details (health status, symptoms, causes, care advice, confidence) into `plant_observations`.
  - **Set Plant Reply Text**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Constructs a formatted multilingual markdown/text message template based on the user's preferred language (`en` or `ja`).
  - **Set Uncertain Image Reply**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Prepares a fallback notification message when plant identification is uncertain.
  - **Post Plant Analysis Reply**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API caller)
    - *Configuration:* HTTP POST request to `https://api.line.me/v2/bot/message/reply` with reply token and text payload. Uses HTTP Header Authentication.
    - *Credentials:* `Header Auth account`.

---

#### Block 1.6: Text Chat Ingestion & State Retrieval
- **Overview:** Fetches user profiles, retrieves active plants, loads past observation records and conversation histories for text-based interactions, and formats them into context variables.
- **Nodes Involved:**
  - `Query Chat User`
  - `Retrieve Active Plant`
  - `Query Plant Observations`
  - `Aggregate Observations`
  - `Fetch Chat History`
  - `Aggregate Chat History`
  - `Build Chat Context`
- **Node Details:**
  - **Query Chat User**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Queries `plant_users` table for the incoming user ID.
  - **Retrieve Active Plant**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Retrieves active plant details from `plants` table using `active_plant_id`.
  - **Query Plant Observations**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Fetches up to 5 recent records from `plant_observations` ordered by `observed_at`.
  - **Aggregate Observations**
    - *Type & Technical Role:* `n8n-nodes-base.aggregate` (Data aggregator)
    - *Configuration:* Combines observation items into `observation_history`.
  - **Fetch Chat History**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database lookup)
    - *Configuration:* Fetches up to 10 historical conversation messages from `conversations` ordered by `created_at`.
  - **Aggregate Chat History**
    - *Type & Technical Role:* `n8n-nodes-base.aggregate` (Data aggregator)
    - *Configuration:* Combines chat logs into `conversation_history`.
  - **Build Chat Context**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Assembles user message, active plant profile, observation history, and conversation history into a structured payload.

---

#### Block 1.7: AI Chat Generation & Storage
- **Overview:** Sends structured chat context to Google Gemini to generate natural language plant care advice, parses the response, and logs both user messages and assistant replies to the database.
- **Nodes Involved:**
  - `Generate Care Reply`
  - `Set Chat Response Data`
  - `Store User Message`
  - `Store Assistant Reply`
- **Node Details:**
  - **Generate Care Reply**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.googleGemini` (LLM chat node)
    - *Configuration:* Model: `models/gemini-3-flash-preview`. System prompt defines conversational constraints, plain text formatting rules for LINE, and strict JSON output structure (`language`, `language_changed`, `reply`).
    - *Credentials:* `PlantCare AI - Gemini`.
  - **Set Chat Response Data**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data transformer)
    - *Configuration:* Parses Gemini's JSON output to extract `reply_text`, `detected_language`, and `language_changed` boolean flag.
  - **Store User Message**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database insertion)
    - *Configuration:* Operation: `append`. Records user messages with role `user` in the `conversations` table.
  - **Store Assistant Reply**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database insertion)
    - *Configuration:* Operation: `append`. Records assistant responses with role `assistant` in the `conversations` table.

---

#### Block 1.8: Language Preference Management & Chat Reply
- **Overview:** Evaluates whether the user requested a language switch, updates user language preferences in the database when applicable, and sends the final plain-text response through LINE.
- **Nodes Involved:**
  - `Check Language Change`
  - `Update User Language Setting`
  - `Post Chat Reply`
- **Node Details:**
  - **Check Language Change**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional checker)
    - *Configuration:* Evaluates whether `{{ $('Set Chat Response Data').item.json.language_changed }}` equals `true`.
  - **Update User Language Setting**
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database update)
    - *Configuration:* Operation: `update`. Updates `preferred_language` in `plant_users` to the newly detected language.
  - **Post Chat Reply**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API caller)
    - *Configuration:* HTTP POST request to `https://api.line.me/v2/bot/message/reply` sending plain text responses. Uses HTTP Header Authentication.
    - *Credentials:* `Header Auth account`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| LINE Webhook Trigger | n8n-nodes-base.webhook | Webhook entry point | None | Route by Message Type | ## AI Plant Care Assistant for LINE... |
| Route by Message Type | n8n-nodes-base.switch | Conditional router | LINE Webhook Trigger | Fetch LINE Image, Query Chat User | ## AI Plant Care Assistant for LINE... |
| Fetch LINE Image | n8n-nodes-base.httpRequest | Binary downloader | Route by Message Type | Query Image User, Combine Image Context Data | ## Receive LINE events |
| Query Image User | n8n-nodes-base.dataTable | Database lookup | Fetch LINE Image | Retrieve Image Active Plant | ## Fetch incoming image |
| Retrieve Image Active Plant | n8n-nodes-base.dataTable | Database lookup | Query Image User | Fetch Image Chat History | ## Load image user state |
| Fetch Image Chat History | n8n-nodes-base.dataTable | Database lookup | Retrieve Image Active Plant | Aggregate Image Chat History | ## Build image context |
| Aggregate Image Chat History | n8n-nodes-base.aggregate | Data aggregator | Fetch Image Chat History | Build Image Context | ## Build image context |
| Build Image Context | n8n-nodes-base.set | Data transformer | Aggregate Image Chat History | Combine Image Context Data | ## Build image context |
| Combine Image Context Data | n8n-nodes-base.merge | Combiner | Fetch LINE Image, Build Image Context | Image Analysis Agent | ## Build image context |
| Image Analysis Agent | @n8n/n8n-nodes-langchain.googleGemini | Multimodal AI node | Combine Image Context Data | Set Plant Analysis Data | ## Analyze plant image |
| Set Plant Analysis Data | n8n-nodes-base.set | Data transformer | Image Analysis Agent | Route by Image Context | ## Analyze plant image |
| Route by Image Context | n8n-nodes-base.switch | Conditional router | Set Plant Analysis Data | Set Active Plant Data, Query Existing Plant, Set Uncertain Image Reply | ## Route image result |
| Set Active Plant Data | n8n-nodes-base.set | Data transformer | Route by Image Context | Query Plant User | ## Upsert recognized plant |
| Query Existing Plant | n8n-nodes-base.dataTable | Database lookup | Route by Image Context | Check Plant Existence | ## Upsert recognized plant |
| Check Plant Existence | n8n-nodes-base.if | Conditional checker | Query Existing Plant | Update Existing Plant, Record New Plant | ## Upsert recognized plant |
| Update Existing Plant | n8n-nodes-base.dataTable | Database update | Check Plant Existence | Merge Plant Data | ## Upsert recognized plant |
| Record New Plant | n8n-nodes-base.dataTable | Database insertion | Check Plant Existence | Merge Plant Data | ## Upsert recognized plant |
| Merge Plant Data | n8n-nodes-base.merge | Combiner | Update Existing Plant, Record New Plant | Query Plant User | ## Upsert recognized plant |
| Query Plant User | n8n-nodes-base.dataTable | Database lookup | Merge Plant Data, Set Active Plant Data | Check Plant User Existence | ## Link user to plant |
| Check Plant User Existence | n8n-nodes-base.if | Conditional checker | Query Plant User | Update Plant User Data, Record New Plant User | ## Link user to plant |
| Update Plant User Data | n8n-nodes-base.dataTable | Database update | Check Plant User Existence | Merge Plant User Data | ## Link user to plant |
| Record New Plant User | n8n-nodes-base.dataTable | Database insertion | Check Plant User Existence | Merge Plant User Data | ## Link user to plant |
| Merge Plant User Data | n8n-nodes-base.merge | Combiner | Update Plant User Data, Record New Plant User | Store Plant Observation | ## Link user to plant |
| Store Plant Observation | n8n-nodes-base.dataTable | Database insertion | Merge Plant User Data | Set Plant Reply Text | ## Save analysis reply |
| Set Plant Reply Text | n8n-nodes-base.set | Data transformer | Store Plant Observation | Post Plant Analysis Reply | ## Save analysis reply |
| Set Uncertain Image Reply | n8n-nodes-base.set | Data transformer | Route by Image Context | Post Plant Analysis Reply | ## Save analysis reply |
| Post Plant Analysis Reply | n8n-nodes-base.httpRequest | API caller | Set Plant Reply Text, Set Uncertain Image Reply | None | ## Save analysis reply |
| Query Chat User | n8n-nodes-base.dataTable | Database lookup | Route by Message Type | Retrieve Active Plant | ## Load chat plant state |
| Retrieve Active Plant | n8n-nodes-base.dataTable | Database lookup | Query Chat User | Query Plant Observations | ## Load chat plant state |
| Query Plant Observations | n8n-nodes-base.dataTable | Database lookup | Retrieve Active Plant | Aggregate Observations | ## Load chat plant state |
| Aggregate Observations | n8n-nodes-base.aggregate | Data aggregator | Query Plant Observations | Fetch Chat History | ## Load chat plant state |
| Fetch Chat History | n8n-nodes-base.dataTable | Database lookup | Aggregate Observations | Aggregate Chat History | ## Prepare chat context |
| Aggregate Chat History | n8n-nodes-base.aggregate | Data aggregator | Fetch Chat History | Build Chat Context | ## Prepare chat context |
| Build Chat Context | n8n-nodes-base.set | Data transformer | Aggregate Chat History | Generate Care Reply | ## Prepare chat context |
| Generate Care Reply | @n8n/n8n-nodes-langchain.googleGemini | LLM chat node | Build Chat Context | Set Chat Response Data | ## Generate chat response |
| Set Chat Response Data | n8n-nodes-base.set | Data transformer | Generate Care Reply | Store User Message | ## Generate chat response |
| Store User Message | n8n-nodes-base.dataTable | Database insertion | Set Chat Response Data | Store Assistant Reply | ## Store chat exchange |
| Store Assistant Reply | n8n-nodes-base.dataTable | Database insertion | Store User Message | Check Language Change | ## Store chat exchange |
| Check Language Change | n8n-nodes-base.if | Conditional checker | Store Assistant Reply | Update User Language Setting, Post Chat Reply | ## Update language reply |
| Update User Language Setting | n8n-nodes-base.dataTable | Database update | Check Language Change | Post Chat Reply | ## Update language reply |
| Post Chat Reply | n8n-nodes-base.httpRequest | API caller | Check Language Change, Update User Language Setting | None | ## Update language reply |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the n8n Data Tables:**
   - Create a Data Table named `plant_users` with columns: `line_user_id` (string), `active_plant_id` (string), `preferred_language` (string), `created_at` (dateTime), `updated_at` (dateTime).
   - Create a Data Table named `plants` with columns: `plant_id` (string), `line_user_id` (string), `common_name` (string), `scientific_name` (string), `nickname` (string), `identification_confidence` (number), `created_at` (dateTime), `updated_at` (dateTime).
   - Create a Data Table named `plant_observations` with columns: `observation_id` (string), `plant_id` (string), `line_user_id` (string), `observed_at` (dateTime), `health_status` (string), `visible_symptoms` (string), `possible_causes` (string), `care_advice` (string), `confidence` (number), `line_message_id` (string).
   - Create a Data Table named `conversations` with columns: `line_user_id` (string), `plant_id` (string), `role` (string), `message` (string), `created_at` (dateTime).

2. **Configure Credentials:**
   - Create an HTTP Header Auth credential (`Header Auth account`) containing your LINE Channel Access Token.
   - Create a Google Gemini API credential (`PlantCare AI - Gemini`) using your Google Palm/Gemini API key.

3. **Build Input Reception & Routing:**
   - Add a **Webhook Trigger** node (`LINE Webhook Trigger`), set method to `POST`, and path to `plantcare-line-webhook`.
   - Add a **Switch** node (`Route by Message Type`), routing based on `{{ $('LINE Webhook Trigger').item.json.body.events[0].message.type }}` for values `image` and `text`.

4. **Build Image Handling Path:**
   - Add an **HTTP Request** node (`Fetch LINE Image`) configured to `GET` `https://api-data.line.me/v2/bot/message/{{ $('LINE Webhook Trigger').item.json.body.events[0].message.id }}/content` with response format set to file, using LINE header auth.
   - Add **Data Table** nodes (`Query Image User`, `Retrieve Image Active Plant`, `Fetch Image Chat History`) to fetch user state, active plant profile, and history.
   - Add an **Aggregate** node (`Aggregate Image Chat History`) to group chat logs into `conversation_history`.
   - Add a **Set** node (`Build Image Context`) to organize variables into a clean context payload, then connect it and the image fetcher to a **Merge** node (`Combine Image Context Data`).
   - Add a **Google Gemini** node (`Image Analysis Agent`) configured for image analysis using `models/gemini-3.8-flash` with the system prompt enforcing strict JSON output.
   - Add a **Set** node (`Set Plant Analysis Data`) to parse the JSON output: `={{ JSON.parse($('Image Analysis Agent').item.json.content.parts[0].text) }}`.

5. **Build Plant Upsert and Observation Logic:**
   - Add a **Switch** node (`Route by Image Context`) evaluating `analysis.context_match` for `same_active_plant`, `new_plant`, and `uncertain`.
   - For `new_plant`/`uncertain`, add a **Data Table** (`Query Existing Plant`), an **If** node (`Check Plant Existence`), and corresponding **Update** and **Append** nodes (`Update Existing Plant`, `Record New Plant`), joined by a **Merge** node (`Merge Plant Data`).
   - For `same_active_plant`, add a **Set** node (`Set Active Plant Data`).
   - Add **Data Table** lookup and conditional nodes (`Query Plant User`, `Check Plant User Existence`, `Update Plant User Data`, `Record New Plant User`, `Merge Plant User Data`) to link the user to the plant.
   - Add a **Data Table** node (`Store Plant Observation`) to append observation records to `plant_observations`.
   - Add **Set** nodes (`Set Plant Reply Text` and `Set Uncertain Image Reply`) to build localized markdown responses, followed by an **HTTP Request** node (`Post Plant Analysis Reply`) to send the message via LINE.

6. **Build Text Chat Handling Path:**
   - Add **Data Table** lookup nodes (`Query Chat User`, `Retrieve Active Plant`, `Query Plant Observations`, `Fetch Chat History`) to load chat state and history.
   - Add **Aggregate** nodes (`Aggregate Observations`, `Aggregate Chat History`) to structure historical arrays.
   - Add a **Set** node (`Build Chat Context`) to compile user messages and context.
   - Add a **Google Gemini** node (`Generate Care Reply`) using `models/gemini-3-flash-preview` with a system prompt enforcing plain text formatting and strict JSON output.
   - Add a **Set** node (`Set Chat Response Data`) to extract chat replies, detected language, and language change flags.
   - Add **Data Table** nodes (`Store User Message`, `Store Assistant Reply`) to persist conversation history.
   - Add an **If** node (`Check Language Change`) to check if `language_changed` is true. If true, update preferred language via **Data Table** (`Update User Language Setting`).
   - Add an **HTTP Request** node (`Post Chat Reply`) to deliver the final plain text response through the LINE messaging API.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Plant Care Assistant for LINE workflow overview and setup guide. | Internal workflow documentation and template design. |
| LINE Messaging API Webhook integration guide. | [LINE Developers Documentation](https://developers.line.biz/en/docs/messaging-api/) |
| Google Gemini API multimodal integration reference. | [Google AI for Developers](https://ai.google.dev/) |