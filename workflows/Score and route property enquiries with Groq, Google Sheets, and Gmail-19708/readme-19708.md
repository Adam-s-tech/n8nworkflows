Score and route property enquiries with Groq, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/score-and-route-property-enquiries-with-groq--google-sheets--and-gmail-19708


# Score and route property enquiries with Groq, Google Sheets, and Gmail

### 1. Workflow Overview

This workflow automates the ingestion, validation, AI-powered qualification, segmentation, and persistence of property enquiries received from an external website contact form. It processes real-time lead data, scores buying intent via an external Large Language Model (LLM), updates separate Google Sheets pipelines based on lead characteristics, alerts internal agents via email, and responds asynchronously to the calling application.

The workflow logic is categorized into six functional blocks:
- **1.1 Input Reception & Validation:** Ingests HTTP POST payloads from webhooks, verifies mandatory structural requirements (such as email presence), and diverts incomplete payloads to an error logging and response path.
- **1.2 Incomplete Lead Handling:** Captures malformed requests without an email address, logs them into an error-tracking Google Sheet, and returns an HTTP 400 response.
- **1.3 Normalization & Classification:** Flattens incoming JSON data structures and analyzes email domain patterns to segregate enquirers into "Personal" or "Business" classifications.
- **1.4 AI Intent Scoring & Fallback:** Interacts with the Groq Chat Completions API using the `llama-3.3-70b-versatile` model to evaluate buying intent on a 1–10 scale. Includes error-handling branches to inject a fallback score if the AI service fails.
- **1.5 Segmented Routing & Persistence:** Evaluates lead classifications and upserts records into designated Google Sheets trackers using the email address as a unique matching key.
- **1.6 Notification & Completion:** Dispatches an HTML-formatted summary email to internal real estate agents via Gmail and returns an HTTP 200 success acknowledgement to the originating webhook caller.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Validation
- **Overview:** Acts as the entry gateway for the workflow, capturing HTTP POST requests from external contact forms and enforcing a strict validation rule to guarantee that an email address exists before deeper processing occurs.
- **Nodes Involved:**
  - `When Enquiry Posted to Webhook`
  - `If Email Provided`

- **Node Details:**
  - **`When Enquiry Posted to Webhook`**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Listens for incoming HTTP POST requests at a designated path.
    - *Configuration Choices:* Method set to `POST`. Error handling configured via `continueRegularOutput`.
    - *Key Expressions or Variables:* Uses runtime execution context parameters.
    - *Input/Output Connections:* Inputs: None (Trigger). Outputs: Connects to `If Email Provided`.
    - *Version-specific Requirements:* Version 2.1.
    - *Edge Cases / Failure Types:* Network latency, incorrect payload format, or timeout errors from the origin form.

  - **`If Email Provided`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Evaluates payload data to verify the existence of an email attribute.
    - *Configuration Choices:* Condition set to check whether the evaluated string is non-empty (`notEmpty`).
    - *Key Expressions or Variables:* `={{ $json.body?.email || $json.payload?.email || $json.email }}`
    - *Input/Output Connections:* Inputs: `When Enquiry Posted to Webhook`. Outputs: True branch connects to `Classify Enquirer Type`; False branch connects to `Append Incomplete Enquiry to Sheets`.
    - *Version-specific Requirements:* Version 2.3.
    - *Edge Cases / Failure Types:* Malformed JSON payloads where the email property is missing or nested unexpectedly.

---

#### Block 1.2: Handle Missing Email
- **Overview:** Manages validation failures by recording rejected, incomplete submissions into a designated Google Sheet and returning a structured HTTP 400 Bad Request error to the webhook client.
- **Nodes Involved:**
  - `Append Incomplete Enquiry to Sheets`
  - `Respond 400 Missing Email`

- **Node Details:**
  - **`Append Incomplete Enquiry to Sheets`**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Persistence). Appends raw submission data to a centralized error-tracking sheet.
    - *Configuration Choices:* Operation set to `append`. Document target configured for "Incomplete Lead".
    - *Key Expressions or Variables:* Standard document list mapping.
    - *Input/Output Connections:* Inputs: `If Email Provided` (False branch). Outputs: Connects to `Respond 400 Missing Email`.
    - *Version-specific Requirements:* Version 4.7.
    - *Edge Cases / Failure Types:* Google API rate limits, invalid spreadsheet credentials, or quota exhaustion.

  - **`Respond 400 Missing Email`**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (HTTP Response). Terminates the request lifecycle with an explicit HTTP 400 status code.
    - *Configuration Choices:* Response code configured to `400`. Response format set to JSON returning `{"error": "Email is required"}`.
    - *Key Expressions or Variables:* Static JSON payload.
    - *Input/Output Connections:* Inputs: `Append Incomplete Enquiry to Sheets`. Outputs: None (Terminal node).
    - *Version-specific Requirements:* Version 1.5.
    - *Edge Cases / Failure Types:* Client disconnection before receiving the response payload.

---

#### Block 1.3: Normalization & Classification
- **Overview:** Flattens incoming payloads and performs domain-level string analysis to categorize the enquirer as either a personal buyer or a business investor.
- **Nodes Involved:**
  - `Classify Enquirer Type`

- **Node Details:**
  - **`Classify Enquirer Type`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Executes custom JavaScript code to extract fields and evaluate domain string patterns.
    - *Configuration Choices:* Custom ECMAScript processing. Throws an explicit error if the email string is missing or invalid.
    - *Key Expressions or Variables:* 
      ```javascript
      let rawData = $input.first().json;
      let bodyData = rawData.body || rawData.payload || rawData;
      // Evaluates free email provider domains against corporate domain patterns
      ```
    - *Input/Output Connections:* Inputs: `If Email Provided` (True branch). Outputs: Connects to `Post to Groq AI for Intent Score`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* Edge cases involving unconventional TLD structures or internationalized domain names (IDNs) causing string splitting errors.

---

#### Block 1.4: AI Intent Scoring & Fallback
- **Overview:** Sends lead data to the Groq Chat Completions endpoint to generate an integer score (1–10) representing buying intent, providing a static fallback value if the external LLM call fails.
- **Nodes Involved:**
  - `Post to Groq AI for Intent Score`
  - `Integrate AI Score with Enquiry`
  - `Fallback Buying Intent Score`

- **Node Details:**
  - **`Post to Groq AI for Intent Score`**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration). Calls the Groq cloud inference API.
    - *Configuration Choices:* Method set to `POST`. Target URL: `https://api.groq.com/openai/v1/chat/completions`. Authentication using predefined credential type `groqApi`. Error handling configured via `continueErrorOutput`.
    - *Key Expressions or Variables:* Uses model `llama-3.3-70b-versatile` with inline prompt templates referencing `{{ $json.name }}`, `{{ $json.email }}`, and `{{ $json.phone }}`.
    - *Input/Output Connections:* Inputs: `Classify Enquirer Type`. Outputs: Main success output connects to `Integrate AI Score with Enquiry`; Error output connects to `Fallback Buying Intent Score`.
    - *Version-specific Requirements:* Version 4.4.
    - *Edge Cases / Failure Types:* API timeouts, token rate limit throttling (`HTTP 429`), or malformed JSON choices returned by the LLM.

  - **`Integrate AI Score with Enquiry`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses the LLM response text, extracts the numerical intent score, and merges it with original lead data.
    - *Configuration Choices:* Custom script parsing `choices[0].message.content`. Sets a boolean flag `scored: true`.
    - *Key Expressions or Variables:* Uses item linking via `$('Classify Enquirer Type').item.json`.
    - *Input/Output Connections:* Inputs: `Post to Groq AI for Intent Score` (Success branch). Outputs: Connects to `If Enquirer is Personal`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* LLM outputting extraneous conversational text instead of a pure number (mitigated by `parseInt` and fallback parsing).

  - **`Fallback Buying Intent Score`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Fallback Transformation). Generates a safe default payload if the primary Groq API call fails.
    - *Configuration Choices:* Assigns a static score of `5` and sets `scored: false`.
    - *Key Expressions or Variables:* References upstream context from `$('Classify Enquirer Type').item.json`.
    - *Input/Output Connections:* Inputs: `Post to Groq AI for Intent Score` (Error branch). Outputs: Connects to `If Enquirer is Personal`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* Context loss if upstream node references fail due to drastic payload mutations.

---

#### Block 1.5: Segmented Routing & Persistence
- **Overview:** Evaluates the classification parameter and updates the appropriate Google Sheets database document using an upsert methodology based on the enquirer's email address.
- **Nodes Involved:**
  - `If Enquirer is Personal`
  - `Update Personal Leads in Sheets`
  - `Update Business Leads in Sheets`

- **Node Details:**
  - **`If Enquirer is Personal`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router). Branches execution based on whether the lead type matches personal or business classifications.
    - *Configuration Choices:* Evaluates `={{ $json.leadType }}` against string value `Personal` with case insensitivity enabled.
    - *Key Expressions or Variables:* `={{ $json.leadType }}`
    - *Input/Output Connections:* Inputs: `Integrate AI Score with Enquiry` or `Fallback Buying Intent Score`. Outputs: True branch connects to `Update Personal Leads in Sheets`; False branch connects to `Update Business Leads in Sheets`.
    - *Version-specific Requirements:* Version 2.3.
    - *Edge Cases / Failure Types:* Unexpected lead type strings causing routing logic to fall through incorrectly.

  - **`Update Personal Leads in Sheets`**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Persistence). Upserts personal lead records into a target Google Spreadsheet.
    - *Configuration Choices:* Operation set to `appendOrUpdate`. Matching column configured as `email`. Maps parameters: `name`, `email`, `phone`, `Score`, `LeadType`, and `Scored`.
    - *Key Expressions or Variables:* `={{ $json.name }}`, `={{ $json.score }}`, `={{ $json.scored ? 'Yes' : 'No' }}`, etc.
    - *Input/Output Connections:* Inputs: `If Enquirer is Personal` (True branch). Outputs: Connects to `Send Enquiry Notification Email`.
    - *Version-specific Requirements:* Version 4.7.
    - *Edge Cases / Failure Types:* Google Sheets API write locks, duplicate record collision issues, or unmapped sheet structures.

  - **`Update Business Leads in Sheets`**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Persistence). Upserts business investor lead records into a target Google Spreadsheet.
    - *Configuration Choices:* Operation set to `appendOrUpdate`. Matching column configured as `email`. Maps parameters identically to the personal sheet node.
    - *Key Expressions or Variables:* `={{ $json.name }}`, `={{ $json.email }}`, `={{ $json.score }}`, etc.
    - *Input/Output Connections:* Inputs: `If Enquirer is Personal` (False branch). Outputs: Connects to `Send Enquiry Notification Email`.
    - *Version-specific Requirements:* Version 4.7.
    - *Edge Cases / Failure Types:* Same as personal sheet node (quota constraints, credential auth expiration).

---

#### Block 1.6: Notification & Completion
- **Overview:** Dispatches a structured HTML alert email to internal sales agents and returns a final HTTP 200 execution confirmation back to the website webhook caller.
- **Nodes Involved:**
  - `Send Enquiry Notification Email`
  - `Respond 200 Enquiry Processed`

- **Node Details:**
  - **`Send Enquiry Notification Email`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration). Sends rich HTML notification emails through Gmail API integrations.
    - *Configuration Choices:* Operation set to send message. Dynamic subject line formulation. Rich HTML body template incorporating inline styles, warning badges for fallback scores, and lead meta-data.
    - *Key Expressions or Variables:* Uses node-linking expressions such as `{{ $('Integrate AI Score with Enquiry').item.json.name }}` and `{{ $json.name }}`.
    - *Input/Output Connections:* Inputs: Receives inputs from both `Update Personal Leads in Sheets` and `Update Business Leads in Sheets`. Outputs: Connects to `Respond 200 Enquiry Processed`.
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases / Failure Types:* Gmail OAuth2 token expiration, daily sending quota limits, or invalid recipient formatting.

  - **`Respond 200 Enquiry Processed`**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (HTTP Response). Concludes the HTTP request lifecycle successfully.
    - *Configuration Choices:* Response code configured to `200`. Response format set to JSON returning a success status object.
    - *Key Expressions or Variables:* Static JSON payload: `{"status": "success", "message": "Lead processed, scored, and saved successfully."}`.
    - *Input/Output Connections:* Inputs: `Send Enquiry Notification Email`. Outputs: None (Terminal node).
    - *Version-specific Requirements:* Version 1.5.
    - *Edge Cases / Failure Types:* Client connection dropping before receiving the confirmation packet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation placeholder outlining full workflow architecture, setup steps, and customization options. | None | None | Property Enquiry Scoring & Routing<br><br>How it works<br>This workflow receives property enquiries from a website webhook, validates that an email address was supplied, and rejects incomplete submissions with a logged 400 response. Valid enquiries are classified as personal or business, scored for buying intent using Groq AI with a fallback score if AI is unavailable, then routed into the appropriate Google Sheet. After saving the lead, it emails agents and returns a successful webhook response.<br><br>Setup steps<br>- Configure the website form to POST enquiry payloads to the n8n webhook URL.<br>- Connect Google Sheets credentials and select/create sheets for incomplete enquiries, personal buyer leads, and business investor leads with matching columns.<br>- Add the Groq API authorization details to the HTTP Request node and confirm the model, prompt, and expected score format.<br>- Connect Gmail credentials and configure recipient agents, subject, and message content for the notification email.<br>- Verify the Respond to Webhook nodes return the desired 200 and 400 response bodies for the website integration.<br><br>Customization<br>Adjust the classification code, Groq scoring prompt, fallback score rules, sheet mappings, and notification recipients to match your lead qualification criteria and sales process. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Entry cluster documentation note. | None | None | Receive and validate enquiry<br><br>Entry cluster that receives a property enquiry from the website and checks whether the required email field is present before continuing. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Lower-left validation failure path documentation note. | None | None | Handle missing email<br><br>Lower-left validation failure path that records incomplete enquiries in Google Sheets and returns a 400 response to the webhook caller. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Central processing cluster documentation note. | None | None | Classify and score intent<br><br>Central processing cluster that flattens and classifies the enquirer, calls Groq AI to score buying intent, merges the AI score with the enquiry data, and provides a fallback score if the AI call fails. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Isolated routing decision documentation note. | None | None | Route by enquirer type<br><br>Isolated routing decision that sends the enriched lead to the personal buyer or business investor storage path based on the earlier classification. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Right-side storage cluster documentation note. | None | None | Save segmented leads<br><br>Right-side storage cluster that appends qualified enquiries to separate Google Sheets for personal buyer leads and business investor leads. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Final output cluster documentation note. | None | None | Notify and confirm success<br><br>Final output cluster that emails agents about the new enquiry and returns a 200 processed response to the website webhook. |
| `When Enquiry Posted to Webhook` | `n8n-nodes-base.webhook` | Listens for incoming HTTP POST requests containing lead details. | None | `If Email Provided` | Receive and validate enquiry<br>Entry cluster that receives a property enquiry from the website and checks whether the required email field is present before continuing. |
| `If Email Provided` | `n8n-nodes-base.if` | Verifies that an email address exists within the incoming payload. | `When Enquiry Posted to Webhook` | `Classify Enquirer Type`, `Append Incomplete Enquiry to Sheets` | Receive and validate enquiry<br>Entry cluster that receives a property enquiry from the website and checks whether the required email field is present before continuing. |
| `Append Incomplete Enquiry to Sheets` | `n8n-nodes-base.googleSheets` | Records submissions lacking an email address into an error-tracking Google Sheet. | `If Email Provided` | `Respond 400 Missing Email` | Handle missing email<br>Lower-left validation failure path that records incomplete enquiries in Google Sheets and returns a 400 response to the webhook caller. |
| `Respond 400 Missing Email` | `n8n-nodes-base.respondToWebhook` | Returns an HTTP 400 error response to the client for missing emails. | `Append Incomplete Enquiry to Sheets` | None | Handle missing email<br>Lower-left validation failure path that records incomplete enquiries in Google Sheets and returns a 400 response to the webhook caller. |
| `Classify Enquirer Type` | `n8n-nodes-base.code` | Normalizes incoming payload structures and classifies the enquirer domain type. | `If Email Provided` | `Post to Groq AI for Intent Score` | Classify and score intent<br>Central processing cluster that flattens and classifies the enquirer, calls Groq AI to score buying intent, merges the AI score with the enquiry data, and provides a fallback score if the AI call fails. |
| `Post to Groq AI for Intent Score` | `n8n-nodes-base.httpRequest` | Calls the Groq Chat Completions API (`llama-3.3-70b-versatile`) to score lead buying intent. | `Classify Enquirer Type` | `Integrate AI Score with Enquiry`, `Fallback Buying Intent Score` | Classify and score intent<br>Central processing cluster that flattens and classifies the enquirer, calls Groq AI to score buying intent, merges the AI score with the enquiry data, and provides a fallback score if the AI call fails. |
| `Integrate AI Score with Enquiry` | `n8n-nodes-base.code` | Parses the LLM response text and merges the extracted score with the lead payload. | `Post to Groq AI for Intent Score` | `If Enquirer is Personal` | Classify and score intent<br>Central processing cluster that flattens and classifies the enquirer, calls Groq AI to score buying intent, merges the AI score with the enquiry data, and provides a fallback score if the AI call fails. |
| `Fallback Buying Intent Score` | `n8n-nodes-base.code` | Provides a default score of 5 if the primary AI request fails. | `Post to Groq AI for Intent Score` | `If Enquirer is Personal` | Classify and score intent<br>Central processing cluster that flattens and classifies the enquirer, calls Groq AI to score buying intent, merges the AI score with the enquiry data, and provides a fallback score if the AI call fails. |
| `If Enquirer is Personal` | `n8n-nodes-base.if` | Routes the enriched lead payload to the appropriate personal or business storage path. | `Integrate AI Score with Enquiry`, `Fallback Buying Intent Score` | `Update Personal Leads in Sheets`, `Update Business Leads in Sheets` | Route by enquirer type<br>Isolated routing decision that sends the enriched lead to the personal buyer or business investor storage path based on the earlier classification. |
| `Update Personal Leads in Sheets` | `n8n-nodes-base.googleSheets` | Upserts personal buyer lead data into the designated Google Sheet. | `If Enquirer is Personal` | `Send Enquiry Notification Email` | Save segmented leads<br>Right-side storage cluster that appends qualified enquiries to separate Google Sheets for personal buyer leads and business investor leads. |
| `Update Business Leads in Sheets` | `n8n-nodes-base.googleSheets` | Upserts business investor lead data into the designated Google Sheet. | `If Enquirer is Personal` | `Send Enquiry Notification Email` | Save segmented leads<br>Right-side storage cluster that appends qualified enquiries to separate Google Sheets for personal buyer leads and business investor leads. |
| `Send Enquiry Notification Email` | `n8n-nodes-base.gmail` | Dispatches an HTML-formatted notification email to sales agents via Gmail. | `Update Personal Leads in Sheets`, `Update Business Leads in Sheets` | `Respond 200 Enquiry Processed` | Notify and confirm success<br>Final output cluster that emails agents about the new enquiry and returns a 200 processed response to the website webhook. |
| `Respond 200 Enquiry Processed` | `n8n-nodes-base.respondToWebhook` | Returns an HTTP 200 success response to the website webhook caller. | `Send Enquiry Notification Email` | None | Notify and confirm success<br>Final output cluster that emails agents about the new enquiry and returns a 200 processed response to the website webhook. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step numbered guide to recreate the workflow in an n8n instance from scratch:

1. **Create Webhook Trigger Node:**
   - Create a new node of type `n8n-nodes-base.webhook`.
   - Name it `When Enquiry Posted to Webhook`.
   - Set **HTTP Method** to `POST`.
   - Set **Options -> On Error** to `Continue Regular Output`.

2. **Create Email Validation Node:**
   - Create a node of type `n8n-nodes-base.if` named `If Email Provided`.
   - Connect the output of `When Enquiry Posted to Webhook` to `If Email Provided`.
   - Configure a condition where the left value is `={{ $json.body?.email || $json.payload?.email || $json.email }}` and the operator is set to `Is Not Empty`.

3. **Configure the Incomplete Lead Path (Error Handling Branch):**
   - Create a node of type `n8n-nodes-base.googleSheets` named `Append Incomplete Enquiry to Sheets`.
   - Connect the `false` output of `If Email Provided` to this node.
   - Set **Operation** to `Append`. Select your target "Incomplete Lead" document and sheet.
   - Create a node of type `n8n-nodes-base.respondToWebhook` named `Respond 400 Missing Email`.
   - Connect `Append Incomplete Enquiry to Sheets` to this node.
   - Set **Response Code** to `400` and **Response Body** to `{"error": "Email is required"}`.

4. **Create Normalization & Classification Node:**
   - Create a node of type `n8n-nodes-base.code` named `Classify Enquirer Type`.
   - Connect the `true` output of `If Email Provided` to this node.
   - Paste the JavaScript code snippet that extracts `name`, `email`, and `phone` safely, checks for email existence, and classifies the domain against free email providers into `Personal` or `Business` lead types.

5. **Create Groq AI Integration Node:**
   - Create a node of type `n8n-nodes-base.httpRequest` named `Post to Groq AI for Intent Score`.
   - Connect `Classify Enquirer Type` to this node.
   - Set **Method** to `POST` and **URL** to `https://api.groq.com/openai/v1/chat/completions`.
   - Configure **Authentication** using a pre-defined credential of type `Groq API` (`groqApi`).
   - Set **Body Content Type** to `JSON` and insert the JSON body payload containing model `llama-3.3-70b-versatile` and system/user messages prompting for an integer score from 1 to 10.
   - Set **Options -> On Error** to `Continue Error Output`.

6. **Create AI Integration & Fallback Nodes:**
   - Create a node of type `n8n-nodes-base.code` named `Integrate AI Score with Enquiry`.
   - Connect the primary (success) output of `Post to Groq AI for Intent Score` to this node.
   - Add script logic to extract the intent score from `aiOutput.choices[0].message.content`, parse it as an integer, and combine it with original data from `$('Classify Enquirer Type').item.json`.
   - Create a node of type `n8n-nodes-base.code` named `Fallback Buying Intent Score`.
   - Connect the secondary (error) output of `Post to Groq AI for Intent Score` to this node.
   - Add script logic returning a default score of `5` and setting `scored: false`.

7. **Create Routing Conditional Node:**
   - Create a node of type `n8n-nodes-base.if` named `If Enquirer is Personal`.
   - Connect both `Integrate AI Score with Enquiry` and `Fallback Buying Intent Score` outputs into this node.
   - Configure the condition to check if `={{ $json.leadType }}` equals `Personal` (with case insensitivity enabled).

8. **Create Segmented Google Sheets Storage Nodes:**
   - Create a node of type `n8n-nodes-base.googleSheets` named `Update Personal Leads in Sheets`.
   - Connect the `true` output of `If Enquirer is Personal` to this node.
   - Set **Operation** to `Append or Update`. Set **Matching Columns** to `email`. Map sheet columns (`name`, `email`, `phone`, `Score`, `LeadType`, `Scored`) to corresponding incoming JSON properties.
   - Create a node of type `n8n-nodes-base.googleSheets` named `Update Business Leads in Sheets`.
   - Connect the `false` output of `If Enquirer is Personal` to this node.
   - Configure identically to the personal sheets node, pointing to the "Business" spreadsheet target.

9. **Create Agent Notification Node:**
   - Create a node of type `n8n-nodes-base.gmail` named `Send Enquiry Notification Email`.
   - Connect the outputs of both `Update Personal Leads in Sheets` and `Update Business Leads in Sheets` to this node.
   - Configure **Authentication** using valid Gmail OAuth2 credentials.
   - Set the subject line expression to `=New Lead Alert: {{ $json.name }}` and provide the HTML-formatted message body utilizing expressions pointing to upstream lead attributes.

10. **Create Success Response Node:**
    - Create a node of type `n8n-nodes-base.respondToWebhook` named `Respond 200 Enquiry Processed`.
    - Connect `Send Enquiry Notification Email` to this node.
    - Set **Response Code** to `200` and **Response Body** to JSON containing status success confirmation messages.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Property Enquiry Scoring & Routing Architecture Reference | Internal workflow design specification for automated lead qualification using Groq AI, Google Sheets, and Gmail integrations. |