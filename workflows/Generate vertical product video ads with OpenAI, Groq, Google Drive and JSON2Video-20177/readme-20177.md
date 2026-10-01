Generate vertical product video ads with OpenAI, Groq, Google Drive and JSON2Video

https://n8nworkflows.xyz/workflows/generate-vertical-product-video-ads-with-openai--groq--google-drive-and-json2video-20177


# Generate vertical product video ads with OpenAI, Groq, Google Drive and JSON2Video

### 1. Workflow Overview

The **AI Product Video Generator** workflow automates the creation of vertical (9:16) product video advertisements by combining user-submitted product details and imagery with AI prompt generation, image editing, scriptwriting, voiceover synthesis, and cloud video rendering. Target use cases include generating social media video ads for platforms like TikTok, Instagram Reels, and YouTube Shorts directly from a form input.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Asset Staging:** Captures user product information and image via an n8n form submission and uploads the base image to Google Drive.
- **1.2 AI Prompt Generation:** Utilizes a Groq language model agent to analyze the product description and generate three cinematic, consistent scene prompts.
- **1.3 Sequential Scene Generation & Upload:** Iteratively downloads reference images, edits them using OpenAI's image model based on the generated prompts, and uploads each resulting scene to Google Drive.
- **1.4 Scriptwriting & Voiceover Synthesis:** Generates a unified voiceover script via a second Groq language model agent, converts it to speech using OpenAI, and uploads the audio asset to Google Drive.
- **1.5 Payload Assembly & Video Rendering:** Gathers all generated image assets and the audio track, structures a JSON payload for JSON2Video, submits the render job, checks its completion status, and returns the rendered MP4 file to the user via an n8n form completion response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Asset Staging
- **Overview:** Receives the user’s product description and product image via a web form, then stages the submitted image into Google Drive.
- **Nodes Involved:** `Get Data`, `Upload file`, `Split Info1`.
- **Node Details:**
  - **`Get Data`**
    - Type and role: `n8n-nodes-base.formTrigger` — Entry point that presents a form requesting a product description and product image.
    - Configuration: Configured with custom form title and fields (`Product Description` as required text, `Product Image` as required file).
    - Inputs/Outputs: Input: None (Trigger); Output: Passes binary image and text data.
    - Edge cases/Failures: Form submission timeout or missing required fields.
  - **`Upload file`**
    - Type and role: `n8n-nodes-base.googleDrive` — Uploads the binary product image received from the form to Google Drive as `Product_Sample_Image`.
    - Configuration: Uses Google Drive OAuth2 credentials, targets the root folder, maps input field `Product_Image`.
    - Inputs/Outputs: Input: `Get Data`; Output: Passes file metadata.
    - Edge cases/Failures: Google Drive authentication failure, insufficient storage quota, or incorrect folder permissions.
  - **`Split Info1`**
    - Type and role: `n8n-nodes-base.set` — Extracts and isolates the product description string into a dedicated assignment property.
    - Configuration: Maps `Product Description` from `{{ $('Get Data').item.json['Product Description'] }}`.
    - Inputs/Outputs: Input: `Upload file`; Output: Passes structured product description to the prompt generator.

#### 2.2 AI Prompt Generation
- **Overview:** Uses a Groq LLM agent backed by a structured output parser to translate the product description into three unique cinematic scene prompts.
- **Nodes Involved:** `Prompt Generator Agent`, `Groq`, `Output Parser`.
- **Node Details:**
  - **`Prompt Generator Agent`**
    - Type and role: `@n8n/n8n-nodes-langchain.agent` — AI Agent configured with a system message to act as an advertising creative director.
    - Configuration: Uses text input `{{ $json['Product Description'] }}` and enforces a strict JSON schema output via the linked structured output parser.
    - Inputs/Outputs: Input: `Split Info1` and `Groq` (Model) & `Output Parser`; Output: JSON containing `prompt 1`, `prompt 2`, and `prompt 3`.
    - Edge cases/Failures: LLM output failing to match the expected JSON schema.
  - **`Groq`**
    - Type and role: `@n8n/n8n-nodes-langchain.lmChatGroq` — Language model chat provider.
    - Configuration: Uses model `openai/gpt-oss-120b` with Groq API credentials.
    - Inputs/Outputs: Connected as an AI language model provider to `Prompt Generator Agent`.
  - **`Output Parser`**
    - Type and role: `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces strict JSON structure for the generated prompts (`prompt 1`, `prompt 2`, `prompt 3`).
    - Inputs/Outputs: Connected as an AI output parser to `Prompt Generator Agent`.

#### 2.3 Sequential Scene Generation & Upload
- **Overview:** Sequentially downloads a reference sample image from Google Drive, processes image edits via OpenAI based on prompts 1, 2, and 3, and uploads each generated scene back to Google Drive.
- **Nodes Involved:** `Get Example Image`, `Generate 1st image`, `Upload 1st image`, `Download Example Image1`, `Generate 2 image`, `Upload 2nd image`, `Download Example Image`, `Generate 3 image`, `Upload 3rd Image`.
- **Node Details:**
  - **`Get Example Image`**
    - Type and role: `n8n-nodes-base.googleDrive` — Downloads the base sample image file from Google Drive using a hardcoded file ID.
    - Configuration: Operation set to `download`, using file ID `1GAcHbWWELZzYoEXCNRwr4_X7tRA25-HP`.
    - Inputs/Outputs: Input: `Prompt Generator Agent`; Output: Passes binary image data.
  - **`Generate 1st image`**
    - Type and role: `@n8n/n8n-nodes-langchain.openAi` — Generates/edits an image using OpenAI's image model based on prompt 1.
    - Configuration: Operation set to `image` -> `edit`, model ID `gpt-image-1-mini`, prompt set to `={{ $json.output['prompt 1'] }}`.
    - Inputs/Outputs: Input: `Get Example Image`; Output: Passes generated image binary/URL data.
  - **`Upload 1st image`**
    - Type and role: `n8n-nodes-base.googleDrive` — Uploads the generated first scene image to the designated Google Drive folder.
    - Configuration: File name set to `Scene_1`, targets folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
    - Inputs/Outputs: Input: `Generate 1st image`; Output: Passes file metadata including `webContentLink`.
  - **`Download Example Image1`**
    - Type and role: `n8n-nodes-base.googleDrive` — Downloads a reference image file for the second scene processing step.
    - Configuration: Operation set to `download`, using file ID `1WUaO2Ogflz320IW3Yv4J4cs2KE_lwa2r`.
    - Inputs/Outputs: Input: `Upload 1st image`; Output: Passes binary image data.
  - **`Generate 2 image`**
    - Type and role: `@n8n/n8n-nodes-langchain.openAi` — Generates/edits the second scene image using prompt 2.
    - Configuration: Model ID `gpt-image-1-mini`, prompt set to `={{ $('Prompt Generator Agent').item.json.output['prompt 2'] }}`.
    - Inputs/Outputs: Input: `Download Example Image1`; Output: Passes generated image binary data.
  - **`Upload 2nd image`**
    - Type and role: `n8n-nodes-base.googleDrive` — Uploads the second scene image to Google Drive.
    - Configuration: File name set to `Scene_2`, targets folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
    - Inputs/Outputs: Input: `Generate 2 image`; Output: Passes file metadata.
  - **`Download Example Image`**
    - Type and role: `n8n-nodes-base.googleDrive` — Downloads a reference image file for the third scene processing step.
    - Configuration: Operation set to `download`, using file ID `1WUaO2Ogflz320IW3Yv4J4cs2KE_lwa2r`.
    - Inputs/Outputs: Input: `Upload 2nd image`; Output: Passes binary image data.
  - **`Generate 3 image`**
    - Type and role: `@n8n/n8n-nodes-langchain.openAi` — Generates/edits the third scene image using prompt 3.
    - Configuration: Model ID `gpt-image-1-mini`, prompt set to `={{ $('Prompt Generator Agent').item.json.output['prompt 3'] }}`.
    - Inputs/Outputs: Input: `Download Example Image`; Output: Passes generated image binary data.
  - **`Upload 3rd Image`**
    - Type and role: `n8n-nodes-base.googleDrive` — Uploads the third scene image to Google Drive.
    - Configuration: File name set to `Scene_3`, targets folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
    - Inputs/Outputs: Input: `Generate 3 image`; Output: Passes file metadata.

#### 2.4 Scriptwriting & Voiceover Synthesis
- **Overview:** Writes a continuous voiceover script based on all three scene prompts using a Groq agent, synthesizes speech via OpenAI, and uploads the audio file to Google Drive.
- **Nodes Involved:** `Script Writer Agent`, `Model`, `Structured Output Parser`, `Get Files`, `Download Scenes`, `Generate audio`, `Split Audio`, `Upload Audio`.
- **Node Details:**
  - **`Script Writer Agent`**
    - Type and role: `@n8n/n8n-nodes-langchain.agent` — AI Agent configured to write a short, natural voiceover script as a single paragraph.
    - Configuration: Uses input referencing prompts 1, 2, and 3 from `Prompt Generator Agent`, coupled with a structured output parser.
    - Inputs/Outputs: Input: `Upload 3rd Image`, `Model`, `Structured Output Parser`; Output: JSON containing the script string.
  - **`Model`**
    - Type and role: `@n8n/n8n-nodes-langchain.lmChatGroq` — Language model chat provider.
    - Configuration: Uses model `openai/gpt-oss-20b` with Groq API credentials.
    - Inputs/Outputs: Connected as an AI language model provider to `Script Writer Agent`.
  - **`Structured Output Parser`**
    - Type and role: `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a JSON schema containing the `script` key.
    - Inputs/Outputs: Connected as an AI output parser to `Script Writer Agent`.
  - **`Get Files`**
    - Type and role: `n8n-nodes-base.googleDrive` — Queries Google Drive to retrieve file metadata for all images in the specified agent folder (`1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`).
    - Configuration: Resource set to `fileFolder`, operation set to list/query with a mimeType filter for images.
    - Inputs/Outputs: Input: `Script Writer Agent`; Output: Returns list of file items.
  - **`Download Scenes`**
    - Type and role: `n8n-nodes-base.googleDrive` — Downloads individual scene image files based on IDs returned from `Get Files`.
    - Configuration: Operation set to `download`, file ID `={{ $json.id }}`.
    - Inputs/Outputs: Input: `Get Files`; Output: Passes binary files and branches to audio generation and merge nodes.
  - **`Generate audio`**
    - Type and role: `@n8n/n8n-nodes-langchain.openAi` — Generates text-to-speech audio using OpenAI based on the script.
    - Configuration: Resource set to `audio`, input set to `={{ $('Script Writer Agent').item.json.output.script }}`.
    - Inputs/Outputs: Input: `Download Scenes`; Output: Passes audio binary data.
  - **`Split Audio`**
    - Type and role: `n8n-nodes-base.code` — Custom JavaScript code node that extracts binary audio data from the OpenAI generation step.
    - Configuration: Custom JS code returning binary data `data`.
    - Inputs/Outputs: Input: `Generate audio`; Output: Passes clean audio binary.
  - **`Upload Audio`**
    - Type and role: `n8n-nodes-base.googleDrive` — Uploads the generated voiceover audio file to Google Drive.
    - Configuration: File name set to `audio_script`, targets folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
    - Inputs/Outputs: Input: `Split Audio`; Output: Passes audio file metadata including `webContentLink`.

#### 2.5 Payload Assembly & Video Rendering
- **Overview:** Merges scene image links with the audio link, formats a JSON payload for JSON2Video, triggers rendering, polls for completion, and returns the final video file via form completion.
- **Nodes Involved:** `Merge`, `Split Info`, `Make A Video`, `Wait 15 Sec`, `Check Video Status`, `Get Final Video`, `Form`.
- **Node Details:**
  - **`Merge`**
    - Type and role: `n8n-nodes-base.merge` — Combines the scene download streams and the uploaded audio file metadata.
    - Configuration: Default merge settings.
    - Inputs/Outputs: Inputs: `Download Scenes` (Input 0) and `Upload Audio` (Input 1); Output: Combined items.
  - **`Split Info`**
    - Type and role: `n8n-nodes-base.code` — Custom JavaScript node that constructs the JSON payload required by the JSON2Video API.
    - Configuration: Builds an Instagram-story resolution payload (1080x1920) containing 3 timed scenes (using web content links from `Upload 1st image`, `Upload 2nd image`, and `Upload 3rd Image`) alongside an audio track and classic subtitle configurations.
    - Inputs/Outputs: Input: `Merge`; Output: JSON payload object.
  - **`Make A Video`**
    - Type and role: `n8n-nodes-base.httpRequest` — Sends a POST request to the JSON2Video API to initiate video rendering.
    - Configuration: URL `https://api.json2video.com/v2/movies`, method POST, sends JSON body, includes headers `x-api-key` (hardcoded value `7Cp9ILxU6KODhv3CA9jMj5GCEiemXB9U92Pxsdr3`) and `Content-Type: application/json`.
    - Inputs/Outputs: Input: `Split Info`; Output: Returns project tracking information including project ID.
  - **`Wait 15 Sec`**
    - Type and role: `n8n-nodes-base.wait` — Pauses workflow execution for 15 seconds to allow JSON2Video to render the video.
    - Configuration: Amount set to 15 seconds.
    - Inputs/Outputs: Input: `Make A Video`; Output: Resumes execution.
  - **`Check Video Status`**
    - Type and role: `n8n-nodes-base.httpRequest` — Queries the JSON2Video API for the rendering status of the project.
    - Configuration: URL `=https://api.json2video.com/v2/movies?project={{ $json.project }}`, includes header `x-api-key`.
    - Inputs/Outputs: Input: `Wait 15 Sec`; Output: Returns movie rendering status and final movie URL.
  - **`Get Final Video`**
    - Type and role: `n8n-nodes-base.httpRequest` — Downloads the rendered MP4 video file from the URL provided in the status check response.
    - Configuration: URL `={{ $json.movie.url }}`, response format set to file (binary).
    - Inputs/Outputs: Input: `Check Video Status`; Output: Passes binary video file.
  - **`Form`**
    - Type and role: `n8n-nodes-base.form` — Form completion response node that returns the generated video file to the user.
    - Configuration: Operation set to `completion`, respond with binary, completion message: "Success Your Video Has Been Generated".
    - Inputs/Outputs: Input: `Get Final Video`; Output: End of workflow.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Get Data` | `n8n-nodes-base.formTrigger` | Web form input trigger for product description and image | None (Trigger) | `Upload file` | AI Product Video Generator / Get Data And Download Sample Image |
| `Groq` | `@n8n/n8n-nodes-langchain.lmChatGroq` | LLM chat model provider for prompt generation | None (AI Model) | `Prompt Generator Agent` | AI Product Video Generator |
| `Prompt Generator Agent` | `@n8n/n8n-nodes-langchain.agent` | Generates 3 cinematic scene prompts from product description | `Split Info1`, `Groq`, `Output Parser` | `Get Example Image` | AI Product Video Generator / Generate Prompt & Get Example Image |
| `Output Parser` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema for generated prompts | None (Parser) | `Prompt Generator Agent` | AI Product Video Generator |
| `Upload file` | `n8n-nodes-base.googleDrive` | Uploads submitted product image to Google Drive | `Get Data` | `Split Info1` | AI Product Video Generator / Get Data And Download Sample Image |
| `Generate 1st image` | `@n8n/n8n-nodes-langchain.openAi` | Generates/edits image for scene 1 | `Get Example Image` | `Upload 1st image` | AI Product Video Generator / Generate 1st Scene & Upload Scene 1 |
| `Generate 2 image` | `@n8n/n8n-nodes-langchain.openAi` | Generates/edits image for scene 2 | `Download Example Image1` | `Upload 2nd image` | AI Product Video Generator / Generate 2nd Scene & Upload Scene 2 |
| `Generate 3 image` | `@n8n/n8n-nodes-langchain.openAi` | Generates/edits image for scene 3 | `Download Example Image` | `Upload 3rd Image` | AI Product Video Generator / Generate 3rd Scene & Upload Scene 3 |
| `Make A Video` | `n8n-nodes-base.httpRequest` | Sends payload to JSON2Video API to render video | `Split Info` | `Wait 15 Sec` | Make Video & Wait 15 sec / Note : Add Your Json2video api key |
| `Check Video Status` | `n8n-nodes-base.httpRequest` | Polls JSON2Video API for rendering status | `Wait 15 Sec` | `Get Final Video` | Get Video Status & Get Final Video |
| `Generate audio` | `@n8n/n8n-nodes-langchain.openAi` | Converts voiceover script to audio via text-to-speech | `Download Scenes` | `Split Audio` | Generate Script And Upload Scene To Drive |
| `Model` | `@n8n/n8n-nodes-langchain.lmChatGroq` | LLM chat model provider for scriptwriting | None (AI Model) | `Script Writer Agent` | Generate Script And Get All Scenes |
| `Structured Output Parser` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON schema for script generation | None (Parser) | `Script Writer Agent` | Generate Script And Get All Scenes |
| `Upload Audio` | `n8n-nodes-base.googleDrive` | Uploads generated voiceover audio to Google Drive | `Split Audio` | `Merge` | Generate Script And Upload Scene To Drive |
| `Split Info` | `n8n-nodes-base.code` | Builds JSON payload for JSON2Video API | `Merge` | `Make A Video` | Merge Info & Split Info |
| `Merge` | `n8n-nodes-base.merge` | Merges scene download streams and audio file metadata | `Download Scenes`, `Upload Audio` | `Split Info` | Merge Info & Split Info |
| `Split Audio` | `n8n-nodes-base.code` | Extracts binary audio from generation node | `Generate audio` | `Upload Audio` | Generate Script And Upload Scene To Drive |
| `Get Files` | `n8n-nodes-base.googleDrive` | Queries Google Drive for scene images | `Script Writer Agent` | `Download Scenes` | Generate Script And Get All Scenes |
| `Get Final Video` | `n8n-nodes-base.httpRequest` | Downloads final rendered MP4 video file | `Check Video Status` | `Form` | Get Video Status & Get Final Video |
| `Split Info1` | `n8n-nodes-base.set` | Isolates product description string | `Upload file` | `Prompt Generator Agent` | Get Data And Download Sample Image |
| `Get Example Image` | `n8n-nodes-base.googleDrive` | Downloads sample reference image for scene 1 | `Prompt Generator Agent` | `Generate 1st image` | Generate Prompt & Get Example Image |
| `Upload 1st image` | `n8n-nodes-base.googleDrive` | Uploads scene 1 image to Google Drive | `Generate 1st image` | `Download Example Image1` | Generate 1st Scene & Upload Scene 1 |
| `Upload 2nd image` | `n8n-nodes-base.googleDrive` | Uploads scene 2 image to Google Drive | `Generate 2 image` | `Download Example Image` | Generate 2nd Scene & Upload Scene 2 |
| `Download Example Image` | `n8n-nodes-base.googleDrive` | Downloads reference image for scene 3 | `Upload 2nd image` | `Generate 3 image` | Generate 3rd Scene & Upload Scene 3 |
| `Download Example Image1` | `n8n-nodes-base.googleDrive` | Downloads reference image for scene 2 | `Upload 1st image` | `Generate 2 image` | Generate 2nd Scene & Upload Scene 2 |
| `Upload 3rd Image` | `n8n-nodes-base.googleDrive` | Uploads scene 3 image to Google Drive | `Generate 3 image` | `Script Writer Agent` | Generate 3rd Scene & Upload Scene 3 |
| `Script Writer Agent` | `@n8n/n8n-nodes-langchain.agent` | Writes voiceover script based on 3 scene prompts | `Upload 3rd Image`, `Model`, `Structured Output Parser` | `Get Files` | Generate Script And Get All Scenes |
| `Download Scenes` | `n8n-nodes-base.googleDrive` | Downloads scene images from Google Drive | `Get Files` | `Generate audio`, `Merge` | Generate Script And Upload Scene To Drive |
| `Wait 15 Sec` | `n8n-nodes-base.wait` | Pauses workflow for 15 seconds | `Make A Video` | `Check Video Status` | Make Video & Wait 15 sec |
| `Form` | `n8n-nodes-base.form` | Returns final generated video file to user | `Get Final Video` | None (End) | Rendeering The Final Video in Form |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Trigger:**
   - Add a **Form Trigger** node (`Get Data`). Set the form title to `AI Product Video Creation` and add two fields: `Product Description` (Required Text) and `Product Image` (Required File).
2. **Setup Initial Google Drive Upload:**
   - Connect a **Google Drive** node (`Upload file`) to `Get Data`. Configure it with Google Drive OAuth2 credentials, set the operation to upload, and map the input data field name to `Product_Image`.
3. **Isolate Description:**
   - Add a **Set** node (`Split Info1`) connected to `Upload file`. Assign a string field named `Product Description` with the value `={{ $('Get Data').item.json['Product Description'] }}`.
4. **Configure Prompt Generation AI Agent:**
   - Add an **AI Agent** node (`Prompt Generator Agent`). Set its text input to `={{ $json['Product Description'] }}` and configure its system message to act as an expert AI advertising creative director returning a JSON list of 3 prompts.
   - Attach a **Groq Chat Model** node (`Groq`) configured with model `openai/gpt-oss-120b` and Groq API credentials to the agent's AI language model input.
   - Attach a **Structured Output Parser** node (`Output Parser`) with a JSON schema defining `prompt 1`, `prompt 2`, and `prompt 3` to the agent's output parser input.
5. **Generate Scene 1:**
   - Add a **Google Drive** node (`Get Example Image`) set to download using file ID `1GAcHbWWELZzYoEXCNRwr4_X7tRA25-HP`. Connect it from `Prompt Generator Agent`.
   - Add an **OpenAI** node (`Generate 1st image`) set to image edit mode (`gpt-image-1-mini`), with the prompt set to `={{ $json.output['prompt 1'] }}`. Connect input from `Get Example Image`.
   - Add a **Google Drive** node (`Upload 1st image`) to upload the output as `Scene_1` to target folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
6. **Generate Scene 2:**
   - Add a **Google Drive** node (`Download Example Image1`) set to download using file ID `1WUaO2Ogflz320IW3Yv4J4cs2KE_lwa2r`. Connect from `Upload 1st image`.
   - Add an **OpenAI** node (`Generate 2 image`) set to image edit (`gpt-image-1-mini`), with the prompt set to `={{ $('Prompt Generator Agent').item.json.output['prompt 2'] }}`.
   - Add a **Google Drive** node (`Upload 2nd image`) to upload output as `Scene_2` to folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
7. **Generate Scene 3:**
   - Add a **Google Drive** node (`Download Example Image`) set to download using file ID `1WUaO2Ogflz320IW3Yv4J4cs2KE_lwa2r`. Connect from `Upload 2nd image`.
   - Add an **OpenAI** node (`Generate 3 image`) set to image edit (`gpt-image-1-mini`), with the prompt set to `={{ $('Prompt Generator Agent').item.json.output['prompt 3'] }}`.
   - Add a **Google Drive** node (`Upload 3rd Image`) to upload output as `Scene_3` to folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
8. **Configure Scriptwriting AI Agent:**
   - Add an **AI Agent** node (`Script Writer Agent`). Set its text input to combine prompts 1, 2, and 3 from `Prompt Generator Agent` and configure its system message to write a single-paragraph voiceover script.
   - Attach a **Groq Chat Model** node (`Model`) using model `openai/gpt-oss-20b` and Groq credentials.
   - Attach a **Structured Output Parser** node (`Structured Output Parser`) enforcing a JSON schema with a `script` property.
9. **Retrieve and Process Scenes & Audio:**
   - Add a **Google Drive** node (`Get Files`) configured to list files matching `mimeType contains 'image/' and '1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R' in parents and trashed = false`.
   - Add a **Google Drive** node (`Download Scenes`) set to download using file ID `={{ $json.id }}`.
   - Add an **OpenAI** node (`Generate audio`) set to audio resource, passing the script from `Script Writer Agent`.
   - Add a **Code** node (`Split Audio`) to extract binary audio data.
   - Add a **Google Drive** node (`Upload Audio`) to upload the audio file named `audio_script` to folder ID `1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`.
10. **Merge Assets & Build Payload:**
    - Add a **Merge** node (`Merge`) combining the downloaded scenes (Input 0) and uploaded audio metadata (Input 1).
    - Add a **Code** node (`Split Info`) to construct the JSON2Video payload with 9:16 vertical resolution (`instagram-story`), 3 timed scenes mapped to the scene image web content links, and subtitle configurations.
11. **Render Video and Return Response:**
    - Add an **HTTP Request** node (`Make A Video`) sending a POST request to `https://api.json2video.com/v2/movies` with header `x-api-key` and the JSON body from `Split Info`.
    - Add a **Wait** node (`Wait 15 Sec`) configured for 15 seconds.
    - Add an **HTTP Request** node (`Check Video Status`) querying `=https://api.json2video.com/v2/movies?project={{ $json.project }}` with the API key header.
    - Add an **HTTP Request** node (`Get Final Video`) downloading the file from `={{ $json.movie.url }}` with response format set to file.
    - Add a **Form** node (`Form`) set to completion mode, responding with binary data to return the final generated video file.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Add your JSON2Video API Key | Required for `Make A Video` and `Check Video Status` HTTP request nodes. |
| Google Drive Folder Access | Ensure target Google Drive folder (`1X1LWY1f2QEwW11gkzPa8Ps5ZkFRWb9_R`) exists, is accessible, and contains publicly accessible files for JSON2Video rendering. |
| Output Video Format | Generated video defaults to 1080×1920 (9:16 aspect ratio) suited for TikTok, Instagram Reels, and YouTube Shorts. |