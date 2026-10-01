Add burned-in subtitles to Google Drive videos with Shotstack

https://n8nworkflows.xyz/workflows/add-burned-in-subtitles-to-google-drive-videos-with-shotstack-19788


# Add burned-in subtitles to Google Drive videos with Shotstack

### 1. Workflow Overview

This workflow automates the process of generating open-captioned videos from raw video files uploaded to a designated Google Drive folder. It targets content creators, social media managers, and video production pipelines looking to eliminate manual transcription and subtitle-burning tasks. 

The execution logic is structured into three consecutive functional blocks:
- **1.1 Input Reception & Preparation:** Monitors a specific Google Drive directory for incoming video files, updates file permissions to grant public read access, and submits the video asset to the Shotstack API along with configuration parameters for automated speech-to-text transcription and subtitle rendering.
- **1.2 Polling & Status Verification:** Enters a polling loop that queries the Shotstack API every 10 seconds to verify render status, evaluating whether the processing is complete, still running, or failed.
- **1.3 File Retrieval & Storage:** Downloads the successfully rendered MP4 video file from the temporary URL provided by Shotstack and uploads the final captioned copy back to Google Drive with an updated filename.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preparation
- **Overview:** This block detects newly added video files in Google Drive, adjusts their sharing permissions so external services can access them, and initiates the automated subtitle burning process via Shotstack.
- **Nodes Involved:** 
  - `New video in the folder`
  - `Let Shotstack read the file`
  - `Shotstack: burn in the captions`
- **Node Details:**
  - **New video in the folder**
    - *Type and Technical Role:* `n8n-nodes-base.googleDriveTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to poll a specific Google Drive folder every minute (`pollTimes`) for newly created files (`fileCreated`).
    - *Key Expressions/Variables:* None (serves as the primary data emitter).
    - *Input/Output Connections:* Output connects to `Let Shotstack read the file`.
    - *Edge Cases/Failures:* Failure to authenticate with Google Drive or selecting an output folder identical to the watched folder will cause infinite trigger loops.
  - **Let Shotstack read the file**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action Node)
    - *Configuration Choices:* Performs a file-sharing operation (`operation: share`) setting permissions to allow anyone with the link to view (`permissionsValues: { role: 'reader', type: 'anyone' }`).
    - *Key Expressions/Variables:* Uses `={{ $json.id }}` to target the file ID from the preceding trigger node.
    - *Input/Output Connections:* Input from `New video in the folder`; output connects to `Shotstack: burn in the captions`.
    - *Edge Cases/Failures:* Google API permission errors or restricted enterprise sharing settings can prevent public link generation.
  - **Shotstack: burn in the captions**
    - *Type and Technical Role:* `@shotstack/n8n-nodes-shotstack.shotstack` (Action Node)
    - *Configuration Choices:* Executes a POST render request (`resource: render`, `operation: postRender`) targeting an HD MP4 output format (`resolution: 'hd'`). It utilizes a layered timeline containing a rich-caption asset mapped to the source video alias and a video asset sourcing the direct Google Drive download URL.
    - *Key Expressions/Variables:* Uses `={{ $('New video in the folder').first().json.id }}` inside the asset source URL to construct the Google Drive direct download link.
    - *Input/Output Connections:* Input from `Let Shotstack read the file`; output connects to `Wait 10 seconds`.
    - *Edge Cases/Failures:* Silent video clips, corrupted media files, or unsupported audio tracks will cause Shotstack to reject or fail the render.

#### 1.2 Polling & Status Verification
- **Overview:** This block manages asynchronous video processing by periodically querying the Shotstack API for render status updates and branching the workflow based on completion or failure.
- **Nodes Involved:** 
  - `Wait 10 seconds`
  - `Check the render`
  - `Is it done?`
  - `Did it fail?`
  - `Stop and say why`
- **Node Details:**
  - **Wait 10 seconds**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control Node)
    - *Configuration Choices:* Pauses workflow execution for a fixed duration of 10 seconds (`amount: 10`, `unit: seconds`).
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input from `Shotstack: burn in the captions` or the retry branch of `Did it fail?`; output connects to `Check the render`.
    - *Edge Cases/Failures:* None typical, though excessive execution counts can hit rate limits if render durations are exceptionally long.
  - **Check the render**
    - *Type and Technical Role:* `@shotstack/n8n-nodes-shotstack.shotstack` (Action Node)
    - *Configuration Choices:* Retrieves the current render status and asset metadata (`resource: render`, `operation: getRender`) using the render ID generated by the initial POST request.
    - *Key Expressions/Variables:* Automatically inherits the `id` from the preceding Shotstack response context.
    - *Input/Output Connections:* Input from `Wait 10 seconds`; output connects to `Is it done?`.
    - *Edge Cases/Failures:* API timeouts or network drops during status checks.
  - **Is it done?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching Node)
    - *Configuration Choices:* Evaluates whether the render process has completed successfully.
    - *Key Expressions/Variables:* Left value: `={{ $json.status }}`, Operator: Equals, Right value: `done`.
    - *Input/Output Connections:* Input from `Check the render`. True branch connects to `Download it`; false branch connects to `Did it fail?`.
    - *Edge Cases/Failures:* None.
  - **Did it fail?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching Node)
    - *Configuration Choices:* Evaluates whether the render process has encountered an unrecoverable failure state.
    - *Key Expressions/Variables:* Left value: `={{ $json.status }}`, Operator: Equals, Right value: `failed`.
    - *Input/Output Connections:* Input from the false branch of `Is it done?`. True branch connects to `Stop and say why`; false branch loops back to `Wait 10 seconds` (to handle "queued" or "rendering" states).
    - *Edge Cases/Failures:* None.
  - **Stop and say why**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (Error Handling Node)
    - *Configuration Choices:* Halts workflow execution and throws a descriptive error message when a render fails.
    - *Key Expressions/Variables:* `=Shotstack could not finish this render: {{ $json.error || $json.status }}`
    - *Input/Output Connections:* Input from the true branch of `Did it fail?`. No outputs.
    - *Edge Cases/Failures:* Fallback to status code if detailed error strings are absent from the API payload.

#### 1.3 File Retrieval & Storage
- **Overview:** This block downloads the completed, captioned video file from Shotstack's temporary hosting URL and uploads the final asset back to Google Drive.
- **Nodes Involved:** 
  - `Download it`
  - `Save the captioned copy`
- **Node Details:**
  - **Download it**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action Node)
    - *Configuration Choices:* Performs an HTTP GET request to fetch binary file data (`responseFormat: file`).
    - *Key Expressions/Variables:* `={{ $json.url }}` (utilizing the temporary output URL returned by Shotstack upon completion).
    - *Input/Output Connections:* Input from the true branch of `Is it done?`; output connects to `Save the captioned copy`.
    - *Edge Cases/Failures:* Expired URLs or large file payloads causing memory limits to exceed on the n8n instance.
  - **Save the captioned copy**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Action Node)
    - *Configuration Choices:* Uploads the binary file to Google Drive using root or selected folder destinations, appending a descriptive suffix to the original filename.
    - *Key Expressions/Variables:* `={{ $('New video in the folder').first().json.name }} (captioned)`
    - *Input/Output Connections:* Input from `Download it`. No outputs.
    - *Edge Cases/Failures:* Insufficient storage space in Google Drive or naming conflicts with existing files.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `New video in the folder` | `n8n-nodes-base.googleDriveTrigger` | Watches a Google Drive folder for new video uploads | None | `Let Shotstack read the file` | ## Add subtitles to every video you drop in a Google Drive folder<br><br>Shotstack transcribes the speech itself and burns the subtitles into the video, so no separate transcription service is needed.<br><br>### How it works<br>- A new file in the watched Google Drive folder starts the workflow.<br>- The file is shared by link so Shotstack can read it over the internet.<br>- Shotstack transcribes the speech and renders a new MP4 with the subtitles burnt in.<br>- The workflow waits for the render, then saves the captioned copy back to Drive.<br><br>### Setup steps<br>- Connect your Google Drive credentials and pick the folder to watch.<br>- Connect your Shotstack credentials. A free API key is enough to start.<br>- Pick a different Drive folder for the output, or the workflow will trigger on its own result and never stop.<br><br>### Good to know<br>- The video must contain speech. A silent clip fails the render.<br>- Very large Drive files return a scan warning page instead of the video. Host those elsewhere.<br><br>[How captions work](https://shotstack.io/?utm_source=n8n&utm_medium=template&utm_campaign=caption-drive-folder) |
| `Let Shotstack read the file` | `n8n-nodes-base.googleDrive` | Updates Google Drive file sharing permissions for public access | `New video in the folder` | `Shotstack: burn in the captions` | ## Watch Drive and send the video<br>Each new file is shared by link, then handed to Shotstack to transcribe and caption. |
| `Shotstack: burn in the captions` | `@shotstack/n8n-nodes-shotstack.shotstack` | Submits render job with automated transcription and open captions | `Let Shotstack read the file` | `Wait 10 seconds` | ## Watch Drive and send the video<br>Each new file is shared by link, then handed to Shotstack to transcribe and caption. |
| `Wait 10 seconds` | `n8n-nodes-base.wait` | Pauses execution for polling intervals | `Shotstack: burn in the captions`, `Did it fail?` (false) | `Check the render` | ## Wait for the render<br>Shotstack answers with a render id and works in the background. This asks for the status every 10 seconds, moves on at `done`, and stops with Shotstack's reason at `failed`. |
| `Check the render` | `@shotstack/n8n-nodes-shotstack.shotstack` | Queries Shotstack API for current render status | `Wait 10 seconds` | `Is it done?` | ## Wait for the render<br>Shotstack answers with a render id and works in the background. This asks for the status every 10 seconds, moves on at `done`, and stops with Shotstack's reason at `failed`. |
| `Is it done?` | `n8n-nodes-base.if` | Checks if render status equals `done` | `Check the render` | `Download it`, `Did it fail?` | ## Wait for the render<br>Shotstack answers with a render id and works in the background. This asks for the status every 10 seconds, moves on at `done`, and stops with Shotstack's reason at `failed`. |
| `Did it fail?` | `n8n-nodes-base.if` | Checks if render status equals `failed` | `Is it done?` (false) | `Stop and say why`, `Wait 10 seconds` | ## Wait for the render<br>Shotstack answers with a render id and works in the background. This asks for the status every 10 seconds, moves on at `done`, and stops with Shotstack's reason at `failed`. |
| `Stop and say why` | `n8n-nodes-base.stopAndError` | Terminates execution and reports error details | `Did it fail?` (true) | None | ## Wait for the render<br>Shotstack answers with a render id and works in the background. This asks for the status every 10 seconds, moves on at `done`, and stops with Shotstack's reason at `failed`. |
| `Download it` | `n8n-nodes-base.httpRequest` | Downloads the rendered MP4 file binary data | `Is it done?` (true) | `Save the captioned copy` | ## Save the captioned copy<br>Fetch the finished file, download it and put it back in Drive. Use a different folder from the one you watch. |
| `Save the captioned copy` | `n8n-nodes-base.googleDrive` | Uploads the captioned MP4 back to Google Drive | `Download it` | None | ## Save the captioned copy<br>Fetch the finished file, download it and put it back in Drive. Use a different folder from the one you watch. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Google Drive Trigger** node (`n8n-nodes-base.googleDriveTrigger`).
   - Set Event to `File Created`.
   - Configure polling time to `Every Minute`.
   - Select your target input folder using the folder picker.
   - *Credentials:* Connect your Google Drive account OAuth2.

2. **Configure File Sharing:**
   - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`).
   - Set Resource to `File` and Operation to `Share`.
   - Set File ID to `={{ $json.id }}`.
   - Configure permissions: Role = `Reader`, Type = `Anyone`.
   - Connect output of `New video in the folder` to this node.

3. **Configure Shotstack Render Submission:**
   - Add a **Shotstack** node (`@shotstack/n8n-nodes-shotstack.shotstack`).
   - Set Resource to `Render` and Operation to `Post Render`.
   - Paste the following timeline edit payload into the Edit field:
     ```json
     {
       "timeline": {
         "tracks": [
           {
             "clips": [
               {
                 "asset": {
                   "type": "rich-caption",
                   "src": "alias://source"
                 },
                 "start": 0,
                 "length": "end"
               }
             ]
           },
           {
             "clips": [
               {
                 "alias": "source",
                 "asset": {
                   "type": "video",
                   "src": "https://drive.google.com/uc?export=download&confirm=t&id={{ $('New video in the folder').first().json.id }}"
                 },
                 "start": 0,
                 "length": "auto"
               }
             ]
           }
         ]
       },
       "output": {
         "format": "mp4",
         "resolution": "hd"
       }
     }
     ```
   - *Credentials:* Connect your Shotstack API key (Sandbox or Production).
   - Connect output of the previous sharing node to this node.

4. **Set Up Polling Loop:**
   - Add a **Wait** node (`n8n-nodes-base.wait`).
   - Set Unit to `Seconds` and Amount to `10`.
   - Connect output of the Shotstack render node to this node.

5. **Check Render Status:**
   - Add a second **Shotstack** node (`@shotstack/n8n-nodes-shotstack.shotstack`).
   - Set Resource to `Render` and Operation to `Get Render`.
   - Connect output of the Wait node to this node.

6. **Add Completion Check Branch:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Name it `Is it done?`.
   - Set condition: Left Value = `={{ $json.status }}`, Operator = `Equals`, Right Value = `done`.
   - Connect output of `Check the render` to this node.

7. **Add Failure Check Branch:**
   - Add a second **If** node (`n8n-nodes-base.if`).
   - Name it `Did it fail?`.
   - Set condition: Left Value = `={{ $json.status }}`, Operator = `Equals`, Right Value = `failed`.
   - Connect the *false* output port of `Is it done?` to this node.

8. **Configure Error Termination:**
   - Add a **Stop and Error** node (`n8n-nodes-base.stopAndError`).
   - Set Error Message to: `=Shotstack could not finish this render: {{ $json.error || $json.status }}`.
   - Connect the *true* output port of `Did it fail?` to this node.
   - Connect the *false* output port of `Did it fail?` back to the input of the **Wait 10 seconds** node to complete the retry loop.

9. **Configure File Download:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Download it`.
   - Set Method to `GET`.
   - Set URL to `={{ $json.url }}`.
   - Under Options, configure Response Format to `File`.
   - Connect the *true* output port of `Is it done?` to this node.

10. **Configure Final Upload:**
    - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`).
    - Name it `Save the captioned copy`.
    - Set Resource to `File` and Operation to `Upload` (or default create behavior).
    - Set File Name expression to: `={{ $('New video in the folder').first().json.name }} (captioned)`.
    - Select a destination folder distinct from the watched input folder.
    - Connect output of `Download it` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| How captions work documentation and guides | [Shotstack Captions Guide](https://shotstack.io/?utm_source=n8n&utm_medium=template&utm_campaign=caption-drive-folder) |
| Requirement warning: Silent video clips will cause render failures | Media must contain clear speech for automated transcription. |
| Folder configuration constraint: Use separate input and output folders | Prevents infinite loop triggers on the output of the workflow itself. |