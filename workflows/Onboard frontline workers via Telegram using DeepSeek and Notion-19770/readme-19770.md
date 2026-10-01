Onboard frontline workers via Telegram using DeepSeek and Notion

https://n8nworkflows.xyz/workflows/onboard-frontline-workers-via-telegram-using-deepseek-and-notion-19770


# Onboard frontline workers via Telegram using DeepSeek and Notion

### 1. Workflow Overview

This workflow automates the end-to-end onboarding process for frontline and hourly workers directly through Telegram, leveraging artificial intelligence for document verification and Notion as a live HR database. It handles initial registrations, document submissions, policy acknowledgments, and proactive daily reminders for incomplete onboarding records.

The system logic is divided into five functional blocks:
- **1.1 Input Reception & Routing:** Listens for incoming Telegram updates and categorizes them by intent (`/start`, document/photo uploads, `AGREE`, or unexpected inputs).
- **1.2 Onboarding Registration & Status Check:** Manages new employee onboarding records in Notion, detecting whether a user is registering for the first time or returning.
- **1.3 Document Ingestion, AI Extraction & Notion Storage:** Downloads files from Telegram, processes PDFs or image assets using DeepSeek AI for structured data extraction, updates Notion statuses, and securely attaches the raw files to the employee's Notion page.
- **1.4 Policy Acknowledgment & Completion:** Processes employee agreement to company policies, issues shift schedules, and notifies hiring managers.
- **1.5 Daily Compliance & Overdue Management:** Runs a scheduled daily job to scan for pending or stalled onboarding items and triggers automated reminders for employees and alerts for managers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing
- **Overview:** Acts as the primary entry point for all interactions, capturing webhook payloads from Telegram and routing messages down distinct operational paths based on predefined criteria.
- **Nodes Involved:** 
  - `When New Telegram Message`
  - `Route by Message Type`
- **Node Details:**
  - **When New Telegram Message** (`n8n-nodes-base.telegramTrigger`)
    - *Role:* Webhook trigger listening for `message` updates from Telegram.
    - *Configuration:* Subscribed to message updates. Requires a valid Telegram Bot API credential.
    - *Input/Output:* Inputs: None (Trigger). Outputs: Connects to `Route by Message Type`.
    - *Edge Cases:* Webhook registration failures or token invalidation.
  - **Route by Message Type** (`n8n-nodes-base.switch`)
    - *Role:* Branching router directing payloads based on message content or attachments.
    - *Configuration:* Evaluates expressions against three conditions (Output 0: text equals `/start`; Output 1: `message.document` or `message.photo` is not empty; Output 2: text equals `AGREE`). Uses a fallback output for unrecognized inputs.
    - *Input/Output:* Inputs: `When New Telegram Message`. Outputs: Four separate branches (Onboarding, Document, Policy, Help Fallback).

---

#### 2.2 Onboarding Registration & Status Check
- **Overview:** Initializes employee metadata from incoming registration requests, queries Notion to verify prior records, and determines whether to issue a welcome-back message or establish a new employee profile.
- **Nodes Involved:**
  - `Set Employee Fields`
  - `Find Existing Record in Notion`
  - `Normalize Existing Record Check`
  - `Check If Returning Employee`
  - `Send Welcome Back Telegram`
  - `Create Notion Onboarding Record`
  - `Send Welcome and Checklist`
- **Node Details:**
  - **Set Employee Fields** (`n8n-nodes-base.set`)
    - *Role:* Extracts and structures sender metadata.
    - *Configuration:* Maps `chatId`, `firstName`, `lastName` (defaulting to empty string), and `onboardingDate` using expression helpers (`$now.format('yyyy-MM-dd')`).
    - *Input/Output:* Inputs: `Route by Message Type` (Branch 0). Outputs: `Find Existing Record in Notion`.
  - **Find Existing Record in Notion** (`n8n-nodes-base.notion`)
    - *Role:* Queries the Notion database for existing entries matching the user's chat ID.
    - *Configuration:* Resource: `databasePage`, Operation: `getAll`, Filter type: Manual condition where `ChatId|rich_text` equals `={{ $json.chatId }}`.
    - *Input/Output:* Inputs: `Set Employee Fields`. Outputs: `Normalize Existing Record Check`.
  - **Normalize Existing Record Check** (`n8n-nodes-base.code`)
    - *Role:* Standardizes Notion search results into a unified boolean flag and payload object.
    - *Configuration:* Custom JavaScript verifying item length, checking for page IDs, and packaging context for downstream evaluation.
    - *Input/Output:* Inputs: `Find Existing Record in Notion`. Outputs: `Check If Returning Employee`.
  - **Check If Returning Employee** (`n8n-nodes-base.if`)
    - *Role:* Decision gate splitting new applicants from returning users.
    - *Configuration:* Condition checks if `={{ $json.exists }}` equals `true`.
    - *Input/Output:* Inputs: `Normalize Existing Record Check`. Outputs: True branch (`Send Welcome Back Telegram`), False branch (`Create Notion Onboarding Record`).
  - **Send Welcome Back Telegram** (`n8n-nodes-base.telegram`)
    - *Role:* Sends a notification to returning workers.
    - *Configuration:* Custom markdown text displaying user name and current onboarding status.
    - *Input/Output:* Inputs: `Check If Returning Employee` (True). Outputs: None (Terminal node for this branch).
  - **Create Notion Onboarding Record** (`n8n-nodes-base.notion`)
    - *Role:* Creates a new database row for first-time applicants.
    - *Configuration:* Operation: `create`, setting page title to full name, and mapping properties (`ChatId`, `Status` set to `Started`, `OnboardingDate`).
    - *Input/Output:* Inputs: `Check If Returning Employee` (False). Outputs: `Send Welcome and Checklist`.
  - **Send Welcome and Checklist** (`n8n-nodes-base.telegram`)
    - *Role:* Sends onboarding instructions and the document checklist to new users.
    - *Configuration:* Text template instructing users to submit government IDs, address proof, and certifications.
    - *Input/Output:* Inputs: `Create Notion Onboarding Record`. Outputs: None (Terminal node for this step).

---

#### 2.3 Document Ingestion, AI Extraction & Notion Storage
- **Overview:** Downloads submitted files from Telegram, checks whether they are PDFs or images, extracts structured text or encodes the image to base64, sends payloads to DeepSeek AI for verification, updates Notion, and uploads the raw asset.
- **Nodes Involved:**
  - `Download Document from Telegram`
  - `Check Document File Type`
  - `Extract Text from PDF`
  - `Prepare Text for DeepSeek Extraction`
  - `Convert Doc to Base64`
  - `Post to DeepSeek API`
  - `Parse DeepSeek Response`
  - `Find Notion Page Using ChatId`
  - `Update Notion Status to Submitted`
  - `Send Doc Received Confirmation`
  - `Create File Upload Request`
  - `Post File Upload to Notion`
  - `Attach Binary Data for Upload`
  - `Post File Content to Notion`
  - `Build Notion Attach Block`
  - `Attach File Block to Notion`
- **Node Details:**
  - **Download Document from Telegram** (`n8n-nodes-base.telegram`)
    - *Role:* Downloads binary files associated with incoming messages.
    - *Configuration:* Resource: `file`, evaluation dynamic `fileId` logic targeting either `message.document` or the highest-resolution `message.photo`.
    - *Input/Output:* Inputs: `Route by Message Type` (Branch 1). Outputs: `Check Document File Type`.
  - **Check Document File Type** (`n8n-nodes-base.if`)
    - *Role:* Determines whether the file is a PDF or an image.
    - *Configuration:* Checks MIME type or file extension for `pdf`.
    - *Input/Output:* Inputs: `Download Document from Telegram`. Outputs: True (`Extract Text from PDF`), False (`Convert Doc to Base64`).
  - **Extract Text from PDF** (`n8n-nodes-base.extractFromFile`)
    - *Role:* Extracts textual data from PDF binaries.
    - *Configuration:* Operation set to `pdf`.
    - *Input/Output:* Inputs: `Check Document File Type` (True). Outputs: `Prepare Text for DeepSeek Extraction`.
  - **Prepare Text for DeepSeek Extraction** (`n8n-nodes-base.code`)
    - *Role:* Sanitizes extracted text and formats the API request body for DeepSeek's text model.
    - *Configuration:* Slices text to 6,000 characters and constructs a strict JSON extraction prompt (`documentType`, `fullName`, `idNumber`, `expiryDate`, `isReadable`).
    - *Input/Output:* Inputs: `Extract Text from PDF`. Outputs: `Post to DeepSeek API`.
  - **Convert Doc to Base64** (`n8n-nodes-base.code`)
    - *Role:* Converts image binaries into base64 strings for vision processing.
    - *Configuration:* Uses `this.helpers.getBinaryDataBuffer()` to extract file buffers and format inline base64 image data URLs.
    - *Input/Output:* Inputs: `Check Document File Type` (False). Outputs: `Post to DeepSeek API`.
  - **Post to DeepSeek API** (`n8n-nodes-base.httpRequest`)
    - *Role:* Sends document content to DeepSeek for analysis.
    - *Configuration:* Method: `POST`, Endpoint: `https://api.deepseek.com/chat/completions`, Content-Type: `raw` (application/json), Authentication: Generic HTTP Header Auth.
    - *Input/Output:* Inputs: `Prepare Text for DeepSeek Extraction` or `Convert Doc to Base64`. Outputs: `Parse DeepSeek Response`.
  - **Parse DeepSeek Response** (`n8n-nodes-base.code`)
    - *Role:* Parses the AI's JSON output and structures metadata for database insertion.
    - *Configuration:* Sanitizes markdown code blocks, extracts structured fields, and determines status values (`Docs Verified` or `Docs Submitted`).
    - *Input/Output:* Inputs: `Post to DeepSeek API`. Outputs: `Find Notion Page Using ChatId`.
  - **Find Notion Page Using ChatId** (`n8n-nodes-base.notion`)
    - *Role:* Locates the target Notion database page ID matching the worker's chat ID.
    - *Configuration:* Resource: `databasePage`, Operation: `getAll`, Filter: `ChatId|rich_text` equals `={{ $json.chatId }}`.
    - *Input/Output:* Inputs: `Parse DeepSeek Response`. Outputs: Splits to `Update Notion Status to Submitted` and `Create File Upload Request`.
  - **Update Notion Status to Submitted** (`n8n-nodes-base.notion`)
    - *Role:* Updates the employee record with verification status, submission date, and AI summary notes.
    - *Configuration:* Operation: `update`, properties mapping `Status`, `DocsSubmittedAt`, and `Notes`.
    - *Input/Output:* Inputs: `Find Notion Page Using ChatId`. Outputs: `Send Doc Received Confirmation`.
  - **Send Doc Received Confirmation** (`n8n-nodes-base.telegram`)
    - *Role:* Acknowledges receipt of documents and presents the company policy summary.
    - *Configuration:* Sends message prompting the user to reply `AGREE`.
    - *Input/Output:* Inputs: `Update Notion Status to Submitted`. Outputs: None.
  - **Create File Upload Request** (`n8n-nodes-base.code`)
    - *Role:* Initializes the upload object configuration for Notion's native file upload API.
    - *Configuration:* Determines file extensions and generates unique payloads containing metadata (`filename`, `content_type`).
    - *Input/Output:* Inputs: `Find Notion Page Using ChatId`. Outputs: `Post File Upload to Notion`.
  - **Post File Upload to Notion** (`n8n-nodes-base.httpRequest`)
    - *Role:* Requests an authenticated upload URL from Notion.
    - *Configuration:* Method: `POST`, Endpoint: `https://api.notion.com/v1/file_uploads`, Header: `Notion-Version: 2022-06-28`.
    - *Input/Output:* Inputs: `Create File Upload Request`. Outputs: `Attach Binary Data for Upload`.
  - **Attach Binary Data for Upload** (`n8n-nodes-base.code`)
    - *Role:* Reattaches the original binary stream to the API upload payload.
    - *Configuration:* Passes upload metadata alongside the binary data object.
    - *Input/Output:* Inputs: `Post File Upload to Notion`. Outputs: `Post File Content to Notion`.
  - **Post File Content to Notion** (`n8n-nodes-base.httpRequest`)
    - *Role:* Uploads the actual binary stream to the temporary Notion upload URL.
    - *Configuration:* Method: `POST`, Content-Type: `multipart-form-data` with form field `file`.
    - *Input/Output:* Inputs: `Attach Binary Data for Upload`. Outputs: `Build Notion Attach Block`.
  - **Build Notion Attach Block** (`n8n-nodes-base.code`)
    - *Role:* Formats the Notion block payload structure for referencing the uploaded file.
    - *Configuration:* Sets block type dynamically (`pdf` or `image`) using file upload IDs.
    - *Input/Output:* Inputs: `Post File Content to Notion`. Outputs: `Attach File Block to Notion`.
  - **Attach File Block to Notion** (`n8n-nodes-base.httpRequest`)
    - *Role:* Appends the uploaded file block as a child block on the employee's Notion page.
    - *Configuration:* Method: `PATCH`, Endpoint: `https://api.notion.com/v1/blocks/{pageId}/children`.
    - *Input/Output:* Inputs: `Build Notion Attach Block`. Outputs: None.

---

#### 2.4 Policy Acknowledgment & Completion
- **Overview:** Listens for the `AGREE` keyword, updates Notion records to confirm policy acknowledgment, delivers shift schedules, and alerts designated managers.
- **Nodes Involved:**
  - `Find Policy Page in Notion`
  - `Update Notion Policy Status`
  - `Send Shift Schedule`
  - `Notify Manager of Progress`
- **Node Details:**
  - **Find Policy Page in Notion** (`n8n-nodes-base.notion`)
    - *Role:* Locates the applicant's Notion page following policy confirmation.
    - *Configuration:* Resource: `databasePage`, Operation: `getAll`, Filter: `ChatId|rich_text` equals `={{ $json.message.chat.id }}`.
    - *Input/Output:* Inputs: `Route by Message Type` (Branch 2). Outputs: `Update Notion Policy Status`.
  - **Update Notion Policy Status** (`n8n-nodes-base.notion`)
    - *Role:* Updates the employee's onboarding status in Notion.
    - *Configuration:* Operation: `update`, setting `Status` select property to `Policy Acknowledged`.
    - *Input/Output:* Inputs: `Find Policy Page in Notion`. Outputs: `Send Shift Schedule`.
  - **Send Shift Schedule** (`n8n-nodes-base.telegram`)
    - *Role:* Dispatches shift details and management contact info to the verified employee.
    - *Configuration:* Text message containing onboarding completion details.
    - *Input/Output:* Inputs: `Update Notion Policy Status`. Outputs: `Notify Manager of Progress`.
  - **Notify Manager of Progress** (`n8n-nodes-base.telegram`)
    - *Role:* Alerts managers that an employee has completed onboarding steps.
    - *Configuration:* Sends notification to a hardcoded manager chat ID.
    - *Input/Output:* Inputs: `Send Shift Schedule`. Outputs: None.
  - **Send Help Message** (`n8n-nodes-base.telegram`)
    - *Role:* Fallback handler responding to unrecognized messages.
    - *Configuration:* Guides users on available commands (`/start`, document uploads, `AGREE`).
    - *Input/Output:* Inputs: `Route by Message Type` (Fallback Branch). Outputs: None.

---

#### 2.5 Daily Compliance & Overdue Management
- **Overview:** Executes a daily cron schedule to catch stalled onboarding applications, sending reminder notifications to employees and escalating overdue items to managers.
- **Nodes Involved:**
  - `Schedule Daily Compliance Check`
  - `Fetch Pending Onboarding Records`
  - `Filter Overdue Records (48hrs)`
  - `Send Reminder to Employee`
  - `Alert Manager of Overdue Docs`
- **Node Details:**
  - **Schedule Daily Compliance Check** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Cron trigger executing daily at 9:00 AM.
    - *Configuration:* Cron expression: `0 9 * * *`.
    - *Input/Output:* Inputs: None (Trigger). Outputs: `Fetch Pending Onboarding Records`.
  - **Fetch Pending Onboarding Records** (`n8n-nodes-base.notion`)
    - *Role:* Retrieves all pending candidate records from Notion.
    - *Configuration:* Filters records where `Status` equals `Started` or `Docs Submitted`, with `returnAll` enabled.
    - *Input/Output:* Inputs: `Schedule Daily Compliance Check`. Outputs: `Filter Overdue Records (48hrs)`.
  - **Filter Overdue Records (48hrs)** (`n8n-nodes-base.filter`)
    - *Role:* Filters out records that have been pending for more than 48 hours.
    - *Configuration:* Evaluates if `property_onboardingdate` is before `={{ $now.minus({hours: 48}) }}`.
    - *Input/Output:* Inputs: `Fetch Pending Onboarding Records`. Outputs: Splits into `Send Reminder to Employee` and `Alert Manager of Overdue Docs`.
  - **Send Reminder to Employee** (`n8n-nodes-base.telegram`)
    - *Role:* Sends a friendly reminder nudge to stalled workers.
    - *Configuration:* Target chat ID mapped from `property_chatid`.
    - *Input/Output:* Inputs: `Filter Overdue Records (48hrs)`. Outputs: None.
  - **Alert Manager of Overdue Docs** (`n8n-nodes-base.telegram`)
    - *Role:* Alerts managers regarding overdue onboarding steps.
    - *Configuration:* Sends warning message containing candidate names to manager chat IDs.
    - *Input/Output:* Inputs: `Filter Overdue Records (48hrs)`. Outputs: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation banner | None | None | ## Frontline Worker Onboarding Automation... |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Telegram intake routing... |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Check onboarding status... |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Returning worker reply... |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Create new onboarding... |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Download and classify document... |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Extract document data... |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Find employee record... |
| `Sticky Note8` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Confirm documents received... |
| `Sticky Note9` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Prepare Notion upload... |
| `Sticky Note10` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Attach uploaded document... |
| `Sticky Note11` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Policy acknowledgement flow... |
| `Sticky Note12` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Help fallback response... |
| `Sticky Note13` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Daily overdue scan... |
| `Sticky Note14` | `n8n-nodes-base.stickyNote` | Block documentation | None | None | ## Overdue reminder alerts... |
| `When New Telegram Message` | `n8n-nodes-base.telegramTrigger` | Triggers on incoming Telegram messages | None | `Route by Message Type` | ## Frontline Worker Onboarding Automation... |
| `Route by Message Type` | `n8n-nodes-base.switch` | Routes messages by type/content | `When New Telegram Message` | `Set Employee Fields`, `Download Document from Telegram`, `Find Policy Page in Notion`, `Send Help Message` | ## Telegram intake routing... |
| `Set Employee Fields` | `n8n-nodes-base.set` | Prepares user metadata from chat payload | `Route by Message Type` | `Find Existing Record in Notion` | ## Check onboarding status... |
| `Find Existing Record in Notion` | `n8n-nodes-base.notion` | Searches Notion DB by chat ID | `Set Employee Fields` | `Normalize Existing Record Check` | ## Check onboarding status... |
| `Normalize Existing Record Check` | `n8n-nodes-base.code` | Normalizes existence check output | `Find Existing Record in Notion` | `Check If Returning Employee` | ## Check onboarding status... |
| `Check If Returning Employee` | `n8n-nodes-base.if` | Branches returning vs new users | `Normalize Existing Record Check` | `Send Welcome Back Telegram`, `Create Notion Onboarding Record` | ## Check onboarding status... |
| `Send Welcome Back Telegram` | `n8n-nodes-base.telegram` | Sends greeting to returning users | `Check If Returning Employee` | None | ## Returning worker reply... |
| `Create Notion Onboarding Record` | `n8n-nodes-base.notion` | Creates new applicant row in Notion | `Check If Returning Employee` | `Send Welcome and Checklist` | ## Create new onboarding... |
| `Send Welcome and Checklist` | `n8n-nodes-base.telegram` | Sends initial welcome and document checklist | `Create Notion Onboarding Record` | None | ## Create new onboarding... |
| `Download Document from Telegram` | `n8n-nodes-base.telegram` | Downloads attached documents/photos | `Route by Message Type` | `Check Document File Type` | ## Download and classify document... |
| `Check Document File Type` | `n8n-nodes-base.if` | Checks if file is PDF or image | `Download Document from Telegram` | `Extract Text from PDF`, `Convert Doc to Base64` | ## Download and classify document... |
| `Extract Text from PDF` | `n8n-nodes-base.extractFromFile` | Extracts text from PDF binary | `Check Document File Type` | `Prepare Text for DeepSeek Extraction` | ## Extract document data... |
| `Prepare Text for DeepSeek Extraction` | `n8n-nodes-base.code` | Formats extracted text for DeepSeek | `Extract Text from PDF` | `Post to DeepSeek API` | ## Extract document data... |
| `Convert Doc to Base64` | `n8n-nodes-base.code` | Encodes images to base64 for vision API | `Check Document File Type` | `Post to DeepSeek API` | ## Extract document data... |
| `Post to DeepSeek API` | `n8n-nodes-base.httpRequest` | Sends document payload to DeepSeek AI | `Prepare Text for DeepSeek Extraction`, `Convert Doc to Base64` | `Parse DeepSeek Response` | ## Extract document data... |
| `Parse DeepSeek Response` | `n8n-nodes-base.code` | Parses DeepSeek AI extraction response | `Post to DeepSeek API` | `Find Notion Page Using ChatId` | ## Extract document data... |
| `Find Notion Page Using ChatId` | `n8n-nodes-base.notion` | Finds candidate Notion page ID | `Parse DeepSeek Response` | `Update Notion Status to Submitted`, `Create File Upload Request` | ## Find employee record... |
| `Update Notion Status to Submitted` | `n8n-nodes-base.notion` | Updates Notion status and notes | `Find Notion Page Using ChatId` | `Send Doc Received Confirmation` | ## Confirm documents received... |
| `Send Doc Received Confirmation` | `n8n-nodes-base.telegram` | Confirms document receipt & shares policy | `Update Notion Status to Submitted` | None | ## Confirm documents received... |
| `Find Policy Page in Notion` | `n8n-nodes-base.notion` | Finds page for policy acknowledgement | `Route by Message Type` | `Update Notion Policy Status` | ## Policy acknowledgement flow... |
| `Update Notion Policy Status` | `n8n-nodes-base.notion` | Updates status to Policy Acknowledged | `Find Policy Page in Notion` | `Send Shift Schedule` | ## Policy acknowledgement flow... |
| `Send Shift Schedule` | `n8n-nodes-base.telegram` | Sends schedule details to employee | `Update Notion Policy Status` | `Notify Manager of Progress` | ## Policy acknowledgement flow... |
| `Notify Manager of Progress` | `n8n-nodes-base.telegram` | Sends completion notice to manager | `Send Shift Schedule` | None | ## Policy acknowledgement flow... |
| `Send Help Message` | `n8n-nodes-base.telegram` | Sends fallback help menu instructions | `Route by Message Type` | None | ## Help fallback response... |
| `Schedule Daily Compliance Check` | `n8n-nodes-base.scheduleTrigger` | Triggers check daily at 9:00 AM | None | `Fetch Pending Onboarding Records` | ## Daily overdue scan... |
| `Fetch Pending Onboarding Records` | `n8n-nodes-base.notion` | Fetches active onboarding records | `Schedule Daily Compliance Check` | `Filter Overdue Records (48hrs)` | ## Daily overdue scan... |
| `Filter Overdue Records (48hrs)` | `n8n-nodes-base.filter` | Filters records stalled > 48 hours | `Fetch Pending Onboarding Records` | `Send Reminder to Employee`, `Alert Manager of Overdue Docs` | ## Overdue reminder alerts... |
| `Send Reminder to Employee` | `n8n-nodes-base.telegram` | Sends reminder notice to worker | `Filter Overdue Records (48hrs)` | None | ## Overdue reminder alerts... |
| `Alert Manager of Overdue Docs` | `n8n-nodes-base.telegram` | Sends overdue alert to manager | `Filter Overdue Records (48hrs)` | None | ## Overdue reminder alerts... |
| `Create File Upload Request` | `n8n-nodes-base.code` | Prepares Notion file upload metadata | `Find Notion Page Using ChatId` | `Post File Upload to Notion` | ## Prepare Notion upload... |
| `Post File Upload to Notion` | `n8n-nodes-base.httpRequest` | Requests upload URL from Notion API | `Create File Upload Request` | `Attach Binary Data for Upload` | ## Prepare Notion upload... |
| `Attach Binary Data for Upload` | `n8n-nodes-base.code` | Prepares multipart binary upload context | `Post File Upload to Notion` | `Post File Content to Notion` | ## Prepare Notion upload... |
| `Post File Content to Notion` | `n8n-nodes-base.httpRequest` | Uploads binary stream to Notion URL | `Attach Binary Data for Upload` | `Build Notion Attach Block` | ## Attach uploaded document... |
| `Build Notion Attach Block` | `n8n-nodes-base.code` | Builds file block payload structure | `Post File Content to Notion` | `Attach File Block to Notion` | ## Attach uploaded document... |
| `Attach File Block to Notion` | `n8n-nodes-base.httpRequest` | Appends file block to Notion page | `Build Notion Attach Block` | None | ## Attach uploaded document... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Input & Routing Setup
1. Create a **Telegram Trigger** node named `When New Telegram Message`. Configure credentials (`Telegram API`) and listen to the `message` update type.
2. Create a **Switch** node named `Route by Message Type`. Set up three routing rules:
   - Rule 0: `{{ $json.message.text }}` equals `/start`.
   - Rule 1: `{{ $json.message.document }}` is not empty OR `{{ $json.message.photo }}` is not empty.
   - Rule 2: `{{ $json.message.text }}` equals (case-insensitive) `AGREE`.
   - Set the fallback output for unrecognized inputs.

#### Step 2: Onboarding Registration Branch (`/start`)
3. Connect Rule 0 of the Switch to a **Set** node named `Set Employee Fields`. Map fields: `chatId` (`{{ $json.message.chat.id }}`), `firstName` (`{{ $json.message.from.first_name }}`), `lastName` (`{{ $json.message.from.last_name || '' }}`), and `onboardingDate` (`{{ $now.format('yyyy-MM-dd') }}`).
4. Add a Notion node named `Find Existing Record in Notion`. Set resource to `databasePage`, operation to `getAll`, filter manually by `ChatId|rich_text` equals `={{ $json.chatId }}` using a Notion API credential.
5. Add a **Code** node named `Normalize Existing Record Check` to parse whether the Notion page exists (`true`/`false`) and pass existing status attributes.
6. Add an **If** node named `Check If Returning Employee` evaluating if `exists` equals `true`.
   - **True Branch:** Connect to a **Telegram** node (`Send Welcome Back Telegram`) addressing the user by name with their current status.
   - **False Branch:** Connect to a **Notion** node (`Create Notion Onboarding Record`) creating a database page with properties `ChatId`, `Status` (`Started`), and `OnboardingDate`.
7. Chain a **Telegram** node (`Send Welcome and Checklist`) to send the onboarding document checklist.

#### Step 3: Document Processing, AI Extraction & Notion Storage Branch
8. Connect Rule 1 of the Switch to a **Telegram** node (`Download Document from Telegram`). Set resource to `file` and evaluate `fileId` dynamically using document IDs or the last photo array index.
9. Connect to an **If** node (`Check Document File Type`) evaluating if the MIME type or extension contains `pdf`.
10. **PDF Path:** Connect to an **Extract From File** node (`Extract Text from PDF`), followed by a **Code** node (`Prepare Text for DeepSeek Extraction`) limiting text to 6,000 characters and setting the JSON prompt.
11. **Image Path:** Connect to a **Code** node (`Convert Doc to Base64`) using `this.helpers.getBinaryDataBuffer()` to encode images as base64 data URLs.
12. Route both paths into an **HTTP Request** node (`Post to DeepSeek API`). Set method to `POST`, URL to `https://api.deepseek.com/chat/completions`, Content-Type to `raw` (`application/json`), and configure an `HTTP Header Auth` credential for DeepSeek.
13. Pass the API output to a **Code** node (`Parse DeepSeek Response`) to clean markdown syntax, parse extracted JSON, and formulate summary strings.
14. Add a Notion node (`Find Notion Page Using ChatId`) to find the corresponding database page by chat ID. Split output into two parallel branches:
    - **Branch A (Status Update):** Connect to a Notion **Update** node (`Update Notion Status to Submitted`) updating status values, submission dates, and notes, followed by a Telegram confirmation node (`Send Doc Received Confirmation`).
    - **Branch B (File Upload):** Connect to a **Code** node (`Create File Upload Request`) establishing file upload metadata, an **HTTP Request** node (`Post File Upload to Notion`) hitting `https://api.notion.com/v1/file_uploads` with header `Notion-Version: 2022-06-28`, a **Code** node (`Attach Binary Data for Upload`) binding binary streams, an **HTTP Request** node (`Post File Content to Notion`) uploading via multipart form data, a **Code** node (`Build Notion Attach Block`) structuring the block object, and a final **HTTP Request** node (`Attach File Block to Notion`) sending a `PATCH` request to `https://api.notion.com/v1/blocks/{pageId}/children`.

#### Step 4: Policy Acknowledgment Branch (`AGREE`)
15. Connect Rule 2 of the Switch to a Notion node (`Find Policy Page in Notion`) filtering by chat ID.
16. Connect to a Notion update node (`Update Notion Policy Status`) setting `Status` to `Policy Acknowledged`.
17. Chain a **Telegram** node (`Send Shift Schedule`) with shift and manager details, followed by another **Telegram** node (`Notify Manager of Progress`) alerting the manager chat ID.
18. Connect the Switch fallback output to a **Telegram** node (`Send Help Message`) for unrecognized commands.

#### Step 5: Daily Compliance Overdue Scan
19. Create a **Schedule Trigger** node (`Schedule Daily Compliance Check`) configured with cron expression `0 9 * * *`.
20. Connect to a Notion node (`Fetch Pending Onboarding Records`) retrieving database pages where status equals `Started` or `Docs Submitted` (with `returnAll` enabled).
21. Add a **Filter** node (`Filter Overdue Records (48hrs)`) ensuring `property_onboardingdate` is older than `={{ $now.minus({hours: 48}) }}`.
22. Split the output into two **Telegram** nodes: `Send Reminder to Employee` (sending nudges to workers) and `Alert Manager of Overdue Docs` (alerting managers).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Frontline Worker Onboarding Automation Template | Main workflow template design utilizing Telegram, DeepSeek AI, and Notion databases. |
| Notion API Integration Requirements | Requires Notion integration tokens with database read/write permissions and `Notion-Version: 2022-06-28` headers for native file uploads. |
| DeepSeek AI Endpoint Reference | Standard model endpoint integration configured for `https://api.deepseek.com/chat/completions`. |