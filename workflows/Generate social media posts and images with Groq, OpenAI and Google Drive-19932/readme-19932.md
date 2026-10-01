Generate social media posts and images with Groq, OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/generate-social-media-posts-and-images-with-groq--openai-and-google-drive-19932


# Generate social media posts and images with Groq, OpenAI and Google Drive

### 1. Workflow Overview

This workflow is designed to automate the creation of social media copy and corresponding visual assets from a single user submission. The primary use case is content generation for multiple platforms (Instagram, LinkedIn, and X) derived from a simple text idea and a selected image resolution.

The logic is divided into four distinct functional blocks:
- **1.1 Input Reception & Prompt Generation:** Collects user parameters via an n8n Form and utilizes a Groq-powered AI agent to transform rough ideas into an optimized image-generation prompt.
- **1.2 Image Generation & Storage:** Generates the image asset using OpenAI based on the structured prompt and chosen resolution, then securely uploads the file to a specified Google Drive folder.
- **1.3 Social Copy Generation:** Leverages a secondary Groq-powered AI agent to draft platform-specific posts for Instagram, LinkedIn, and X (Twitter) using the generated prompt context.
- **1.4 HTML Formatting & Delivery:** Combines the generated copy and the Google Drive shareable asset link into a responsive HTML page via a final AI agent, presenting the output back to the user through an interactive form completion page.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Prompt Generation
- **Overview:** This block establishes the user entry point, captures the initial creative concept alongside the target layout resolution, and refines the input into a structured prompt suitable for image generation models.
- **Nodes Involved:** 
  - `Form Submission Trigger`
  - `Prompt Creation Agent`
  - `Language Model Processor`
  - `Output Structuring Parser`
- **Node Details:**
  - **Form Submission Trigger**
    - *Type & Role:* `n8n-nodes-base.formTrigger` (v2.6) — Serves as the webhook-based entry point capturing form data.
    - *Configuration:* Configured with fields `input` (required string) and `Resolution` (dropdown with options: `1024x1024`, `1024x1536`, `1536x1024`).
    - *Expressions:* None.
    - *Connections:* Input: None (Trigger). Output: `Prompt Creation Agent`.
    - *Edge Cases:* Missing required input fields will prevent the workflow from initializing.
  - **Prompt Creation Agent**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (v3.1) — LangChain AI agent processing user inputs into standardized visual instructions.
    - *Configuration:* System message: `"You're a helpful image genertion prompt generator your task is to see what user want and then generate a prompt"`. Prompt type set to define.
    - *Expressions:* Input text: `={{ $json.input }}`.
    - *Connections:* Input: `Form Submission Trigger`, `Language Model Processor`, `Output Structuring Parser`. Output: `AI Image Generator`.
    - *Edge Cases:* API rate limits or downtime on the Groq endpoint will cause failures.
  - **Language Model Processor**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (v1) — Chat model provider integration for the prompt generation agent.
    - *Configuration:* Uses model `openai/gpt-oss-120b`. Requires Groq account credentials.
    - *Connections:* Input: None. Output: Connected to `Prompt Creation Agent` via `ai_languageModel`.
    - *Edge Cases:* Invalid credentials cause authentication errors.
  - **Output Structuring Parser**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — Enforces JSON structure on the AI agent's output.
    - *Configuration:* JSON Schema Example: `{"prompt": "null"}`.
    - *Connections:* Input: None. Output: Connected to `Prompt Creation Agent` via `ai_outputParser`.
    - *Edge Cases:* Model output failing to adhere strictly to the JSON schema results in parsing execution errors.

---

#### Block 1.2: Image Generation & Storage
- **Overview:** This block executes the graphical asset generation using OpenAI's image model and stores the binary output directly into a configured Google Drive directory to yield a persistent URL.
- **Nodes Involved:**
  - `AI Image Generator`
  - `Upload to Google Drive`
- **Node Details:**
  - **AI Image Generator**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (v2.3) — Generates an image file based on structured text prompts.
    - *Configuration:* Resource set to `image`, model ID configured to `gpt-image-1-mini`. Uses AI gateway managed credentials.
    - *Expressions:* Prompt: `={{ $json.output.prompt }}`. Size option: `={{ $('Form Submission Trigger').item.json.Resolution }}`.
    - *Connections:* Input: `Prompt Creation Agent`. Output: `Upload to Google Drive`.
    - *Edge Cases:* Resolution mismatches or token/quota exhaustion on OpenAI accounts.
  - **Upload to Google Drive**
    - *Type & Role:* `n8n-nodes-base.googleDrive` (v3) — Uploads binary image data to cloud storage.
    - *Configuration:* Target drive: `My Drive`. Target folder ID: `1gArI_U5zOHpNTl9xRCIongCHE7TzX5yz` (`image genaration`).
    - *Connections:* Input: `AI Image Generator`. Output: `Post Content Generator`.
    - *Edge Cases:* Insufficient Google OAuth permissions or missing target folder IDs will interrupt execution.

---

#### Block 1.3: Social Copy Generation
- **Overview:** Takes the conceptual prompt data and generates contextual, platform-tailored copy for Instagram, LinkedIn, and X (Twitter) using a Groq-backed LangChain agent.
- **Nodes Involved:**
  - `Post Content Generator`
  - `Chat Model Execution`
  - `Output Parsing Agent`
- **Node Details:**
  - **Post Content Generator**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (v3.1) — Orchestrates the copywriting task across multiple platforms.
    - *Configuration:* Prompt type defined with execution parameters referencing previous nodes.
    - *Expressions:* Text: `you have to make a post for instagram linkedin twitter (x) {{ $('Prompt Creation Agent').item.json.output.prompt }}`.
    - *Connections:* Input: `Upload to Google Drive`, `Chat Model Execution`, `Output Parsing Agent`. Output: `HTML Generation Agent`.
    - *Edge Cases:* Unhandled formatting characters in prompt variables can trigger parsing errors.
  - **Chat Model Execution**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (v1) — Language model backend for post generation.
    - *Configuration:* Model set to `openai/gpt-oss-120b` with Groq account credentials.
    - *Connections:* Output: Connected to `Post Content Generator` via `ai_languageModel`.
  - **Output Parsing Agent**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — Validates that generated social copies match specific keys.
    - *Configuration:* JSON schema expects: `{"instagram": "null", "linkedin": "null", "twitter (x)": "null"}`.
    - *Connections:* Output: Connected to `Post Content Generator` via `ai_outputParser`.

---

#### Block 1.4: HTML Formatting & Delivery
- **Overview:** Consolidates all generated social copy and the cloud storage image link into a clean HTML document layout, then presents the final payload to the end user via an n8n form completion response.
- **Nodes Involved:**
  - `HTML Generation Agent`
  - `Groq Chat Model Execution`
  - `Structured Output Parser`
  - `Display Final Result`
- **Node Details:**
  - **HTML Generation Agent**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (v3.1) — Converts plain text posts and image URLs into cohesive HTML code.
    - *Configuration:* System message defines structural display layout rules (Instagram block, LinkedIn block, Twitter block, and a redirect button pointing to the image URL).
    - *Expressions:* Text inputs mapped from prior social copy nodes and the Google Drive `webViewLink`.
    - *Connections:* Input: `Post Content Generator`, `Groq Chat Model Execution`, `Structured Output Parser`. Output: `Display Final Result`.
  - **Groq Chat Model Execution**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGroq` (v1) — Model provider powering the HTML formatting agent.
    - *Configuration:* Uses model `openai/gpt-oss-120b`.
    - *Connections:* Output: Connected to `HTML Generation Agent` via `ai_languageModel`.
  - **Structured Output Parser**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) — Enforces a predictable JSON return payload containing the generated code.
    - *Configuration:* JSON schema: `{"html file code": "null"}`.
    - *Connections:* Output: Connected to `HTML Generation Agent` via `ai_outputParser`.
  - **Display Final Result**
    - *Type & Role:* `n8n-nodes-base.form` (v2.5) — Form completion node rendering the final UI layout.
    - *Configuration:* Operation set to `completion`, response mode configured to show text.
    - *Expressions:* Response text: `={{ $json.output['html file code'] }}`.
    - *Connections:* Input: `HTML Generation Agent`. Output: None (Terminal node).
    - *Edge Cases:* Malformed HTML strings may render broken designs inside the browser form view.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Form Submission Trigger** | `n8n-nodes-base.formTrigger` | Collects user topic/idea and resolution parameters | None (Trigger) | Prompt Creation Agent | ## AI Post Generation<br><br>### How it works<br><br>This workflow starts from a form submission, uses a Groq-powered AI agent to turn the user input into structured prompts, then generates an image with OpenAI. The generated image is uploaded to Google Drive, after which another AI agent writes the post content and a final agent converts it into structured HTML. The finished HTML is shown back to the user in a result form.<br><br>### Setup steps<br><br>- Configure the Form trigger with the fields needed to collect the post topic, audience, style, or other prompt inputs.<br>- Add valid Groq credentials for each LangChain chat model used by the AI agents, and confirm the structured output parsers match the expected JSON schemas.<br>- Add OpenAI credentials and configure the image generation model, size, and prompt mapping from the prompt generator output.<br>- Connect Google Drive credentials and choose the target folder or upload settings for storing generated images.<br>- Configure the final Result form to display the generated HTML or any other fields returned by the HTML Code Generator.<br><br>### Customization<br><br>You can customize the form fields, agent prompts, structured output schemas, image style settings, Drive destination folder, and final HTML template to match different social platforms or content formats. <br><br>## Collect and shape prompt<br><br>Starts the workflow from the user form and uses the first AI agent, supported by its Groq model and structured output parser, to turn the submission into a clean image-generation prompt. |
| **Prompt Creation Agent** | `@n8n/n8n-nodes-langchain.agent` | Transforms user text into an optimized image prompt | Form Submission Trigger | AI Image Generator | ## AI Post Generation... (see above) <br><br>## Collect and shape prompt... (see above) |
| **Language Model Processor** | `@n8n/n8n-nodes-langchain.lmChatGroq` | Groq chat model for prompt generation | None | Prompt Creation Agent | ## AI Post Generation... (see above) <br><br>## Collect and shape prompt... (see above) |
| **Output Structuring Parser** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output structure for prompt generation | None | Prompt Creation Agent | ## AI Post Generation... (see above) <br><br>## Collect and shape prompt... (see above) |
| **AI Image Generator** | `@n8n/n8n-nodes-langchain.openAi` | Generates graphical asset via OpenAI | Prompt Creation Agent | Upload to Google Drive | ## AI Post Generation... (see above)<br><br>## Generate and store image<br><br>Creates the visual asset with OpenAI and uploads the generated file to Google Drive so the later post-generation step can reference it. |
| **Upload to Google Drive** | `n8n-nodes-base.googleDrive` | Uploads image file to Google Drive folder | AI Image Generator | Post Content Generator | ## AI Post Generation... (see above)<br><br>## Generate and store image... (see above) |
| **Post Content Generator** | `@n8n/n8n-nodes-langchain.agent` | Generates multi-platform social media copy | Upload to Google Drive | HTML Generation Agent | ## AI Post Generation... (see above)<br><br>## Write post content<br><br>Uses a second AI agent, with its nearby Groq model and structured parser, to generate the actual post content based on the uploaded image and earlier prompt context. |
| **Chat Model Execution** | `@n8n/n8n-nodes-langchain.lmChatGroq` | Groq chat model for social copy generation | None | Post Content Generator | ## AI Post Generation... (see above)<br><br>## Write post content... (see above) |
| **Output Parsing Agent** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Validates structured copy output keys | None | Post Content Generator | ## AI Post Generation... (see above)<br><br>## Write post content... (see above) |
| **HTML Generation Agent** | `@n8n/n8n-nodes-langchain.agent` | Combines text copy and image link into HTML code | Post Content Generator | Display Final Result | ## AI Post Generation... (see above)<br><br>## Format and return HTML<br><br>Converts the generated post into structured HTML using the final AI agent and then presents the finished result to the user through the output form. |
| **Groq Chat Model Execution** | `@n8n/n8n-nodes-langchain.lmChatGroq` | Groq chat model for HTML formatting agent | None | HTML Generation Agent | ## AI Post Generation... (see above)<br><br>## Format and return HTML... (see above) |
| **Structured Output Parser** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output for HTML code block | None | HTML Generation Agent | ## AI Post Generation... (see above)<br><br>## Format and return HTML... (see above) |
| **Display Final Result** | `n8n-nodes-base.form` | Renders the HTML layout as a form completion view | HTML Generation Agent | None | ## AI Post Generation... (see above)<br><br>## Format and return HTML... (see above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**
   - Type: `n8n-nodes-base.formTrigger`
   - Parameters: Set form title to `Image Generation Agent`, form description to `Fill the form for image generation`.
   - Form Fields: Add a text field with label `input` (required) and a dropdown field with label `Resolution` having options `1024x1024`, `1024x1536`, and `1536x1024`.
2. **Create Prompt Creation Agent & Sub-nodes**
   - Create an Agent node (`@n8n/n8n-nodes-langchain.agent`) named `Prompt Creation Agent`. Set system message to `"You are a helpful image genertion prompt generator your task is to see what user want and then generate a prompt"`. Set text input to `={{ $json.input }}`.
   - Attach `Language Model Processor` (`@n8n/n8n-nodes-langchain.lmChatGroq`) using model `openai/gpt-oss-120b` and assign Groq account credentials. Connect via `ai_languageModel`.
   - Attach `Output Structuring Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`) with schema example `{"prompt": "null"}`. Connect via `ai_outputParser`.
3. **Create Image Generation Node**
   - Create an OpenAI node (`@n8n/n8n-nodes-langchain.openAi`) named `AI Image Generator`.
   - Parameters: Resource = `image`, Model ID = `gpt-image-1-mini`.
   - Prompt expression: `={{ $json.output.prompt }}`.
   - Size option expression: `={{ $('Form Submission Trigger').item.json.Resolution }}`.
   - Connect main input from `Prompt Creation Agent`.
4. **Create Google Drive Upload Node**
   - Create a Google Drive node (`n8n-nodes-base.googleDrive`) named `Upload to Google Drive`.
   - Parameters: Name = `image`, Drive = `My Drive`, Folder ID = `1gArI_U5zOHpNTl9xRCIongCHE7TzX5yz`. Configure Google Drive OAuth2 credentials.
   - Connect main input from `AI Image Generator`.
5. **Create Post Content Generator & Sub-nodes**
   - Create an Agent node named `Post Content Generator`. Set text input to `you have to make a post for instagram linkedin twitter (x) {{ $('Prompt Creation Agent').item.json.output.prompt }}`.
   - Attach `Chat Model Execution` (`lmChatGroq`) using model `openai/gpt-oss-120b` with Groq credentials via `ai_languageModel`.
   - Attach `Output Parsing Agent` (`outputParserStructured`) with schema `{"instagram": "null", "linkedin": "null", "twitter (x)": "null"}` via `ai_outputParser`.
   - Connect main input from `Upload to Google Drive`.
6. **Create HTML Generation Agent & Sub-nodes**
   - Create an Agent node named `HTML Generation Agent`. Set system message to handle conversion to formatted HTML layouts with an image check button.
   - Text input expression:
     ```text
     Instagram Post : {{ $json.output.instagram }}
     LinkedIn Post : {{ $json.output.linkedin }}
     Twitter (X) Post: {{ $json.output['twitter (x)'] }}

     Image Link : {{ $('Upload to Google Drive').item.json.webViewLink }}
     ```
   - Attach `Groq Chat Model Execution` (`lmChatGroq`) using model `openai/gpt-oss-120b` via `ai_languageModel`.
   - Attach `Structured Output Parser` (`outputParserStructured`) with schema `{"html file code": "null"}` via `ai_outputParser`.
   - Connect main input from `Post Content Generator`.
7. **Create Final Result Node**
   - Create a Form node (`n8n-nodes-base.form`) named `Display Final Result`.
   - Parameters: Operation = `completion`, Respond with = `showText`.
   - Response text expression: `={{ $json.output['html file code'] }}`.
   - Connect main input from `HTML Generation Agent`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Image generation repository folder | [Google Drive Folder Link](https://drive.google.com/drive/folders/1gArI_U5zOHpNTl9xRCIongCHE7TzX5yz) |