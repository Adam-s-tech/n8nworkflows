Back up workflows to Nextcloud and send completion alerts in Discord

https://n8nworkflows.xyz/workflows/back-up-workflows-to-nextcloud-and-send-completion-alerts-in-discord-19805


# Back up workflows to Nextcloud and send completion alerts in Discord

### 1. Workflow Overview

This workflow automates the process of backing up all n8n workflows to a Nextcloud storage folder and sends a completion notification via Discord. It can be triggered manually or automatically on a daily schedule.

The logical blocks are organized as follows:
- **1.1 Input Reception:** Initiates the workflow via a scheduled Cron trigger or a manual execution button.
- **1.2 Collect Workflow List:** Connects to the n8n API to retrieve all workflows, splits the payload into individual items, and initializes an iteration loop.
- **1.3 Fetch and Upload Backups:** Iterates through each workflow ID to download its complete definition, transforms the JSON data into a binary file, and uploads it to a designated Nextcloud directory.
- **1.4 Send Completion Notice:** Detects the end of the batch loop and posts a confirmation message to a Discord channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Provides dual entry points for the backup process, allowing either automated daily execution or on-demand manual triggers.
- **Nodes Involved:** `Scheduled Workflow Backup`, `Manual Execution Trigger`
- **Node Details:**
  - **Scheduled Workflow Backup**
    - *Type and Technical Role:* Cron trigger node (`n8n-nodes-base.cron`). Executes the workflow automatically.
    - *Configuration Choices:* Configured to trigger daily at hour 4.
    - *Input/Output Connections:* Output connects to `Retrieve All Workflows`.
    - *Edge Cases/Failures:* Relies on the n8n instance time zone settings; inaccurate server time will cause unexpected execution schedules.
  - **Manual Execution Trigger**
    - *Type and Technical Role:* Manual trigger node (`n8n-nodes-base.manualTrigger`). Allows users to test or run the workflow on-demand within the n8n editor.
    - *Configuration Choices:* Default configuration with no parameters.
    - *Input/Output Connections:* Output connects to `Retrieve All Workflows`.
    - *Edge Cases/Failures:* None.

#### 2.2 Collect Workflow List
- **Overview:** Connects to the local n8n instance API to gather all workflows, flattens the resulting array into individual workflow items, and passes them to a batch loop controller.
- **Nodes Involved:** `Retrieve All Workflows`, `Split Workflow IDs`, `Batch Process Workflows`
- **Node Details:**
  - **Retrieve All Workflows**
    - *Type and Technical Role:* n8n integration node (`n8n-nodes-base.n8n`). Fetches a list of all workflows from the instance using API credentials.
    - *Configuration Choices:* Uses default filters to pull all workflows. Requires `n8nApi` credentials.
    - *Input/Output Connections:* Inputs from `Scheduled Workflow Backup` and `Manual Execution Trigger`; output connects to `Split Workflow IDs`.
    - *Edge Cases/Failures:* Authentication errors if the n8n API key lacks permissions or has expired.
  - **Split Workflow IDs**
    - *Type and Technical Role:* Item list utility node (`n8n-nodes-base.splitOut`). Separates the array of workflows into distinct items based on the `id` field.
    - *Configuration Choices:* `fieldToSplitOut` set to `id`, with `include` set to retain all other fields.
    - *Input/Output Connections:* Input from `Retrieve All Workflows`; output connects to `Error Handling / Batch Process Workflows`.
    - *Edge Cases/Failures:* Failure if the upstream node returns an empty or malformed array.
  - **Batch Process Workflows**
    - *Type and Technical Role:* Flow control loop node (`n8n-nodes-base.splitInBatches`). Iterates through items sequentially.
    - *Configuration Choices:* Default batch settings.
    - *Input/Output Connections:* Input from `Split Workflow IDs` and loopback from `Upload to NextCloud`. Output branch 0 goes to `Fetch Workflow Data`; output branch 1 goes to `Notify Discord of Completion`.
    - *Edge Cases/Failures:* Infinite loops if loop continuation logic is misconfigured.

#### 2.3 Fetch and Upload Backups
- **Overview:** Retrieves the full JSON definition of each workflow iteratively, converts the data structure into a binary file format, and uploads it to a target Nextcloud directory.
- **Nodes Involved:** `Fetch Workflow Data`, `Transfer Workflow File`, `Upload to NextCloud`
- **Node Details:**
  - **Fetch Workflow Data**
    - *Type and Technical Role:* n8n integration node (`n8n-nodes-base.n8n`). Retrieves the detailed JSON configuration for a specific workflow ID.
    - *Configuration Choices:* Operation set to `get`, with `workflowId` evaluated dynamically as `={{ $json.id }}`. Requires `n8nApi` credentials.
    - *Input/Output Connections:* Input from `Batch Process Workflows`; output connects to `Transfer Workflow File`.
    - *Edge Cases/Failures:* API timeouts or missing workflow IDs if a workflow is deleted mid-execution.
  - **Transfer Workflow File**
    - *Type and Technical Role:* Data transformation node (`n8n-nodes-base.moveBinaryData`). Converts JSON payload data into binary file attachments.
    - *Configuration Choices:* Mode set to `jsonToBinary` with UTF-8 encoding. Sets the dynamic file name to `={{ $json.name }}.json`.
    - *Input/Output Connections:* Input from `Fetch Workflow Data`; output connects to `Upload to NextCloud`.
    - *Edge Cases/Failures:* Expression evaluation failures if the workflow name contains unsupported special characters.
  - **Upload to NextCloud**
    - *Type and Technical Role:* Cloud storage node (`n8n-nodes-base.nextCloud`). Uploads binary files to Nextcloud.
    - *Configuration Choices:* Path set to `=/N8N Workflow Backup/{{ $('Fetch Workflow Data').item.json.name.replaceAll("/:", "_") }}.json`, with binary data upload enabled. Error handling set to `continueRegularOutput`. Requires `nextCloudApi` credentials.
    - *Input/Output Connections:* Input from `Transfer Workflow File`; output loops back to `Batch Process Workflows`.
    - *Edge Cases/Failures:* Network timeouts, authentication issues, or missing target directories on the Nextcloud server. `continueRegularOutput` prevents a single failed upload from halting the entire batch.

#### 2.4 Send Completion Notice
- **Overview:** Triggers a notification message via Discord once the iteration loop finishes processing all workflow backups.
- **Nodes Involved:** `Notify Discord of Completion`
- **Node Details:**
  - **Notify Discord of Completion**
    - *Type and Technical Role:* Messaging notification node (`n8n-nodes-base.discord`). Sends a completion alert to a Discord server channel.
    - *Configuration Choices:* Resource set to `message` using OAuth2 authentication. Content evaluates to `N8N Workflow Backup Complete for {{ $json.Timestamp }}!`. Requires `discordOAuth2Api` credentials.
    - *Input/Output Connections:* Input from the completion branch of `Batch Process Workflows`.
    - *Edge Cases/Failures:* Invalid channel or guild IDs, missing OAuth2 scopes, or rate limiting by the Discord API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Backs up n8n workflows to NextCloud... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Start backup triggers... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Collect workflow list... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Fetch and upload backups... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation container | None | None | ## Send completion notice... |
| Manual Execution Trigger | `n8n-nodes-base.manualTrigger` | Manual entry point | None | Retrieve All Workflows | ## Start backup triggers... |
| Scheduled Workflow Backup | `n8n-nodes-base.cron` | Automated timer entry point | None | Retrieve All Workflows | ## Start backup triggers... |
| Retrieve All Workflows | `n8n-nodes-base.n8n` | Fetch all workflows from API | Manual Execution Trigger, Scheduled Workflow Backup | Split Workflow IDs | ## Collect workflow list... |
| Split Workflow IDs | `n8n-nodes-base.splitOut` | Flatten workflow array | Retrieve All Workflows | Batch Process Workflows | ## Collect workflow list... |
| Batch Process Workflows | `n8n-nodes-base.splitInBatches` | Loop controller | Split Workflow IDs, Upload to NextCloud | Fetch Workflow Data, Notify Discord of Completion | ## Collect workflow list... |
| Fetch Workflow Data | `n8n-nodes-base.n8n` | Fetch individual workflow JSON | Batch Process Workflows | Transfer Workflow File | ## Fetch and upload backups... |
| Transfer Workflow File | `n8n-nodes-base.moveBinaryData` | Convert JSON to binary | Fetch Workflow Data | Upload to NextCloud | ## Fetch and upload backups... |
| Upload to NextCloud | `n8n-nodes-base.nextCloud` | Upload file to cloud storage | Transfer Workflow File | Batch Process Workflows | ## Fetch and upload backups... |
| Notify Discord of Completion | `n8n-nodes-base.discord` | Send final alert message | Batch Process Workflows | None | ## Send completion notice... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Triggers:**
   - Add a **Cron** node named `Scheduled Workflow Backup`. Set the trigger time to hour `4`.
   - Add a **Manual Trigger** node named `Manual Execution Trigger`.
2. **Collect and Split Workflow Data:**
   - Add an **n8n** node named `Retrieve All Workflows`. Configure it to connect to your n8n API credentials and leave filters empty. Connect both triggers to this node.
   - Add a **Split Out** node named `Split Workflow IDs`. Set `Field to Split Out` to `id` and include all other fields. Connect `Retrieve All Workflows` to this node.
   - Add a **Split In Batches** node named `Batch Process Workflows`. Connect `Split Workflow IDs` to it.
3. **Configure the Processing Loop:**
   - Add an **n8n** node named `Fetch Workflow Data`. Set the operation to `get` and the workflow ID expression to `={{ $json.id }}` using your n8n API credentials. Connect output branch 0 of `Batch Process Workflows` to this node.
   - Add a **Move Binary Data** node named `Transfer Workflow File`. Set the mode to `jsonToBinary`, encoding to `utf8`, and the file name expression to `={{ $json.name }}.json`. Connect `Fetch Workflow Data` to this node.
   - Add a **Nextcloud** node named `Upload to NextCloud`. Set the file path expression to `=/N8N Workflow Backup/{{ $('Fetch Workflow Data').item.json.name.replaceAll("/:", "_") }}.json` and enable binary data upload. Set error handling to `Continue On Fail` (`continueRegularOutput`). Configure your Nextcloud credentials. Connect `Transfer Workflow File` to this node.
   - Loop the output of `Upload to NextCloud` back to the input of `Batch Process Workflows`.
4. **Configure Completion Notification:**
   - Add a **Discord** node named `Notify Discord of Completion`. Set the resource to `message`, authentication to `OAuth2` with your Discord credentials, and message content to `N8N Workflow Backup Complete for {{ $json.Timestamp }}!`.
   - Connect output branch 1 of `Batch Process Workflows` (loop completion branch) to `Notify Discord of Completion`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Filename character replacement | Slashes and colons in workflow names are replaced with underscores (`_`) to maintain operating system file path compatibility. |
| Error handling strategy | The Nextcloud upload node is configured to continue regular execution on failure to prevent a single corrupted or unauthorized upload from blocking the entire backup loop. |