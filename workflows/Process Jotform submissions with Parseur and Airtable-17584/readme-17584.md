Process Jotform submissions with Parseur and Airtable

https://n8nworkflows.xyz/workflows/process-jotform-submissions-with-parseur-and-airtable-17584


# Process Jotform submissions with Parseur and Airtable

### 1. Workflow Overview

The **Process Jotform submissions with Parseur and Airtable** workflow automates the intake of form responses from Jotform, validates and downloads attached documents, extracts information using Parseur, and logs the details into an Airtable base. It ensures that only submissions containing an uploaded document proceed through the extraction and storage pipeline.

The logical execution is grouped into three main blocks:
- **1.1 Form Intake & Validation:** Captures new Jotform submissions, evaluates whether an uploaded file is present, and branches the flow accordingly.
- **1.2 File Download & Parsing:** Constructs a secure download URL, retrieves the binary file from Jotform, and routes the payload and metadata to Parseur for data extraction.
- **1.3 Database Search & Record Creation:** Queries Airtable to check for existing entries using the Submission ID and creates a new record containing the submitter details and document URL.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Form Intake & Validation
- **Overview:** This block listens for incoming Jotform submissions, inspects the form data to confirm if an attachment was included, and routes valid submissions forward while isolating submissions without attachments.
- **Nodes Involved:** 
  - `New Form Submission Received`
  - `Check If File Uploaded`
  - `Skip - No File Uploaded`
- **Node Details:**
  - **New Form Submission Received**
    - *Type and Technical Role:* `n8n-nodes-base.jotFormTrigger` (Trigger). Listens for webhook events corresponding to new form submissions.
    - *Configuration Choices:* Form ID set to `262+1234567890`; `onlyAnswers` set to `false` to retain metadata.
    - *Key Expressions or Variables:* None (triggers on event).
    - *Input and Output Connections:* Input: None (Trigger) | Output: `Check If File Uploaded` (Main index 0).
    - *Version-Specific Requirements:* Version 1. Requires valid Jotform integration credentials.
    - *Edge Cases or Potential Failure Types:* Webhook delivery failures if Jotform API endpoints change or network blocks incoming requests.
  - **Check If File Uploaded**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates conditional expressions against incoming JSON payloads.
    - *Configuration Choices:* Evaluates condition using strict type validation and case-sensitive matching.
    - *Key Expressions or Variables:* Left Value: `={{ $json.pretty }}` | Operator: Contains | Right Value: `Upload Document:`
    - *Input and Output Connections:* Input: `New Form Submission Received` | Output True: `Build File Download URL` (Index 0) | Output False: `Skip - No File Uploaded` (Index 1).
    - *Version-Specific Requirements:* Version 2.3.
    - *Edge Cases or Potential Failure Types:* Field label naming discrepancies in Jotform can cause the condition to fail incorrectly.
  - **Skip - No File Uploaded**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Flow Control / Termination). Acts as a placeholder terminal node for payloads lacking a file.
    - *Configuration Choices:* Default empty configuration.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `Check If File Uploaded` (False branch) | Output: None.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases or Potential Failure Types:* None.

---

#### Block 1.2: File Download & Parsing
- **Overview:** This block constructs a direct download link for the Jotform attachment, fetches the binary file over HTTP, and forwards the data along with form field attributes to Parseur.
- **Nodes Involved:**
  - `Build File Download URL`
  - `Download Uploaded File`
  - `Send File to Parseur for Extraction`
- **Node Details:**
  - **Build File Download URL**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Constructs the target file URL string using dynamic variables.
    - *Configuration Choices:* Creates a single string property named `fileURL`.
    - *Key Expressions or Variables:* `=https://www.jotform.com/uploads/{{ $json.username }}/{{ $json.formID }}/{{ $json.submissionID }}/{{ $json.pretty.split("Upload Document:")[1].trim() }}?apiKey=1342bdf101c496efa0a9a7b5bcb5d25a`
    - *Input and Output Connections:* Input: `Check If File Uploaded` (True branch) | Output: `Download Uploaded File`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases or Potential Failure Types:* Hardcoded API key expiration or malformed string splitting if the Jotform response format shifts.
  - **Download Uploaded File**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Network Request). Downloads binary data from the generated URL.
    - *Configuration Choices:* URL mapped to `={{ $json.fileURL }}`; response format set to `file`; custom User-Agent header supplied.
    - *Key Expressions or Variables:* `={{ $json.fileURL }}`
    - *Input and Output Connections:* Input: `Build File Download URL` | Output: `Send File to Parseur for Extraction`.
    - *Version-Specific Requirements:* Version 4.4.
    - *Edge Cases or Potential Failure Types:* HTTP 403/404 errors due to invalid API keys or broken file paths.
  - **Send File to Parseur for Extraction**
    - *Type and Technical Role:* `n8n-nodes-parseur.parseur` (Integration / OCR). Submits text content and documents to a Parseur workspace inbox.
    - *Configuration Choices:* Operation set to `uploadText`; sender and recipient email addresses defined.
    - *Key Expressions or Variables:* Dynamic HTML body mapping form fields (Full Name, Email, Phone Number, Service Type, Submission ID, Form Name, and Upload Document URL) using parent node references.
    - *Input and Output Connections:* Input: `Download Uploaded File` | Output: `Check Existing Airtable Record`.
    - *Version-Specific Requirements:* Version 1. Requires valid Parseur credentials.
    - *Edge Cases or Potential Failure Types:* Authentication errors with Parseur API or rate limits on document processing.

---

#### Block 1.3: Database Search & Record Creation
- **Overview:** This block queries Airtable to check for an existing row matching the current submission ID, ensuring duplicates are avoided before appending the new submission data.
- **Nodes Involved:**
  - `Check Existing Airtable Record`
  - `Save Submission to Airtable`
- **Node Details:**
  - **Check Existing Airtable Record**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Integration). Searches an Airtable table using a custom formula.
    - *Configuration Choices:* Base ID: `app3jRazfZCjG6ufL`, Table ID: `tblH7GGEzTL8PUKXk`, Operation: `search`, `alwaysOutputData` enabled.
    - *Key Expressions or Variables:* Formula: `={Submission ID} = '{{ $('New Form Submission Received').item.json.submissionID }}'`
    - *Input and Output Connections:* Input: `Send File to Parseur for Extraction` | Output: `Save Submission to Airtable`.
    - *Version-Specific Requirements:* Version 2.2. Requires Airtable Personal Access Token credentials.
    - *Edge Cases or Potential Failure Types:* Formula syntax errors or API permission issues restricting read access to the base.
  - **Save Submission to Airtable**
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Integration). Creates a new record inside the specified Airtable base.
    - *Configuration Choices:* Base ID: `app3jRazfZCjG6ufL`, Table ID: `tblH7GGEzTL8PUKXk`, Operation: `create`, mapping mode configured to define field values explicitly.
    - *Key Expressions or Variables:* Maps `Email`, `Full Name`, `Phone Number`, `Service Type`, `Submission ID`, and `Upload Document` from incoming node data (`New Form Submission Received` and `Build File Download URL`).
    - *Input and Output Connections:* Input: `Check Existing Airtable Record` | Output: None.
    - *Version-Specific Requirements:* Version 2.2. Requires Airtable Personal Access Token credentials.
    - *Edge Cases or Potential Failure Types:* Schema mismatches (e.g., trying to write text to a field configured as a multi-select or attachment type in Airtable).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `New Form Submission Received` | `n8n-nodes-base.jotFormTrigger` | Trigger on new form submission | None | `Check If File Uploaded` | Automated Jotform Intake & File Processing Pipeline<br><br>### How It Works<br>* **Form Trigger:** Listens for new submissions coming from Jotform (Form ID: `262001763321040`).<br>* **Attachment Check:** Evaluates whether the submission contains an uploaded document.<br>* **File Download:** Constructs the Jotform download URL (using submission metadata and API key) and downloads the binary file via an HTTP request.<br><br>### Setup Checklist<br>1. **Jotform API:** Verify Jotform credentials in `New Form Submission Received` and check that the API key in `Build File Download URL` is active.<br>2. **Parseur Account:** Ensure Parseur credentials (`GNvtRXHGKcIiINiI`) are linked to the upload step.<br>3. **Airtable Integration:** Confirm Personal Access Token permissions and verify Base ID (`app3jRazfZCjG6ufL`) and Table ID (`tblH7GGEzTL8PUKXk`).<br><br>### Customization<br>* **Target Form ID:** Update the Jotform ID parameter to hook into a different intake form.<br>* **Airtable Fields:** Adjust field mappings inside `Save Submission to Airtable` to capture additional form questions.<br><br>## 1. Form Intake & File Download |
| `Check If File Uploaded` | `n8n-nodes-base.if` | Check if uploaded file exists | `New Form Submission Received` | `Build File Download URL`, `Skip - No File Uploaded` | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 1. Form Intake & File Download |
| `Skip - No File Uploaded` | `n8n-nodes-base.noOp` | End flow if no file is present | `Check If File Uploaded` | None | Automated Jotform Intake & File Processing Pipeline ... |
| `Build File Download URL` | `n8n-nodes-base.set` | Build file download URL string | `Check If File Uploaded` | `Download Uploaded File` | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 1. Form Intake & File Download |
| `Download Uploaded File` | `n8n-nodes-base.httpRequest` | Download file from Jotform via HTTP | `Build File Download URL` | `Send File to Parseur for Extraction` | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 2. Parsing & OCR Submission |
| `Send File to Parseur for Extraction` | `n8n-nodes-parseur.parseur` | Send payload and file to Parseur | `Download Uploaded File` | `Check Existing Airtable Record` | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 2. Parsing & OCR Submission |
| `Check Existing Airtable Record` | `n8n-nodes-base.airtable` | Search Airtable for existing entry | `Send File to Parseur for Extraction` | `Save Submission to Airtable` | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 3. Database Search & Record Creation |
| `Save Submission to Airtable` | `n8n-nodes-base.airtable` | Create new Airtable record | `Check Existing Airtable Record` | None | Automated Jotform Intake & File Processing Pipeline ...<br><br>## 3. Database Search & Record Creation |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Jotform Trigger** (`n8n-nodes-base.jotFormTrigger`) named `New Form Submission Received`.
   - Set parameters: Form ID to `262+1234567890`, `onlyAnswers` to `false`.
   - Configure valid Jotform credentials.

2. **Add the Validation Branch:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Check If File Uploaded`.
   - Set condition: Left Value `={{ $json.pretty }}`, Operator `contains`, Right Value `Upload Document:`.
   - Connect `New Form Submission Received` output to `Check If File Uploaded`.

3. **Add the No-Operation (Skip) Node:**
   - Add a **No-Op** node (`n8n-nodes-base.noOp`) named `Skip - No File Uploaded`.
   - Connect the `False` output (index 1) of `Check If File Uploaded` to this node.

4. **Construct the File URL:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`) named `Build File Download URL`.
   - Connect the `True` output (index 0) of `Check If File Uploaded` to this node.
   - Configure a string assignment named `fileURL` with value:
     `=https://www.jotform.com/uploads/{{ $json.username }}/{{ $json.formID }}/{{ $json.submissionID }}/{{ $json.pretty.split("Upload Document:")[1].trim() }}?apiKey=1342bdf101c496efa0a9a7b5bcb5d25a`

5. **Download the Binary File:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Download Uploaded File`.
   - Connect `Build File Download URL` to this node.
   - Set URL to `={{ $json.fileURL }}`.
   - Set Response Format to `File`.
   - Add a header parameter: Name `User-Agent`, Value `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36`.

6. **Integrate with Parseur:**
   - Add a **Parseur** node (`n8n-nodes-parseur.parseur`) named `Send File to Parseur for Extraction`.
   - Connect `Download Uploaded File` to this node.
   - Configure operation to `uploadText`, sender to `user@example.com`, recipient to `user@example.com`, and subject to `=New Jotform Submission - {{ $('New Form Submission Received').item.json.formTitle }}`.
   - Set `body_html` using expression mapping form fields (`Full Name`, `Email`, `Phone Number`, `Service Type`, `Submission ID`, `Form Name`, and `Upload Document`).
   - Configure Parseur credentials.

7. **Search Airtable Records:**
   - Add an **Airtable** node (`n8n-nodes-base.airtable`) named `Check Existing Airtable Record`.
   - Connect `Send File to Parseur for Extraction` to this node.
   - Configure operation to `search`, Base ID to `app3jRazfZCjG6ufL`, Table ID to `tblH7GGEzTL8PUKXk`, and enable `alwaysOutputData`.
   - Set Filter by Formula: `={Submission ID} = '{{ $('New Form Submission Received').item.json.submissionID }}'`
   - Configure Airtable Personal Access Token credentials.

8. **Save Record to Airtable:**
   - Add an **Airtable** node (`n8n-nodes-base.airtable`) named `Save Submission to Airtable`.
   - Connect `Check Existing Airtable Record` to this node.
   - Configure operation to `create`, Base ID to `app3jRazfZCjG6ufL`, Table ID to `tblH7GGEzTL8PUKXk`.
   - Map columns explicitly:
     - `Email`: `={{ $('New Form Submission Received').item.json.rawRequest["Email"] }}`
     - `Full Name`: `={{ $('New Form Submission Received').item.json.rawRequest["Full Name"] }}`
     - `Phone Number`: `={{ $('New Form Submission Received').item.json.rawRequest["Phone Number"] }}`
     - `Service Type`: `={{ $('New Form Submission Received').item.json.rawRequest["Service Type"] }}`
     - `Submission ID`: `={{ $('New Form Submission Received').item.json.submissionID }}`
     - `Upload Document`: `={{ $('Build File Download URL').item.json.fileURL }}`
   - Use the same Airtable credentials as the previous step.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Jotform Intake Form Configuration | Jotform Form ID: `262001763321040` (Reference Base ID) |
| Parseur Workspace Credentials | Workspace integration linked via API / Email Inbox (`GNvtRXHGKcIiINiI`) |
| Airtable Schema Reference | Base ID: `app3jRazfZCjG6ufL`, Table ID: `tblH7GGEzTL8PUKXk` |