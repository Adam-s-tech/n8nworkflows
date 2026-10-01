Organize Gmail inbox labels with Jev TypeSafe AI (System One)

https://n8nworkflows.xyz/workflows/organize-gmail-inbox-labels-with-jev-typesafe-ai--system-one--19769


# Organize Gmail inbox labels with Jev TypeSafe AI (System One)

### 1. Workflow Overview

This workflow automates Gmail inbox management by periodically scanning for unlabelled emails, classifying them using the TypeSafe Jev AI engine, determining appropriate action levels based on statistical confidence, creating missing labels dynamically, and applying labels in a paginated loop until the backlog is cleared.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Polling:** Triggers execution on a fixed schedule and queries Gmail for unlabelled messages.
- **1.2 AI Processing & Decision Logic:** Sends email metadata and body text to TypeSafe Jev in batches, evaluating category fit and reply requirements to establish a confidence-weighted destination label.
- **1.3 Dynamic Label Management:** Fetches existing Gmail labels, diffs them against required labels, deduplicates, creates missing labels on-demand, and reloads the label inventory.
- **1.4 Application & Looping Execution:** Resolves string labels to fresh Gmail label IDs, applies the labels to messages, and evaluates loop conditions to drain backlogs up to a strict round limit.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Polling

- **Overview:** Initiates the workflow run on a regular schedule and retrieves a batch of unlabelled messages from Gmail, filtering out noise, drafts, and sent mail.
- **Nodes Involved:** 
  - `Every 5 Minutes`
  - `Fetch Unlabelled Emails`

- **Node Details:**
  - **`Every 5 Minutes`**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Fires execution on an interval of every 1 minute/minutes (rule configuration).
    - *Input/Output:* Inputs: None. Outputs: Triggers `Fetch Unlabelled Emails`.
    - *Edge Cases/Failures:* Missed triggers due to platform downtime or inactivity timeouts.
  - **`Fetch Unlabelled Emails`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node)
    - *Configuration Choices:* Operation: `getAll`. Query filter: `has:nouserlabels -in:chats -in:sent -in:draft`. Reader status: both. Simple mode disabled (`simple: false`).
    - *Input/Output:* Inputs: `Every 5 Minutes` (and loopback from `Check for Next Batch`). Outputs: `Classify Email with Jev`.
    - *Credentials:* `Gmail Account - OAuth2 API` (`gmailOAuth2`)
    - *Edge Cases/Failures:* Authentication revocation, API rate limits (HTTP 429), or empty query returns.

---

#### 2.2 AI Processing & Decision Logic

- **Overview:** Submits retrieved emails to TypeSafe Jev for semantic classification and confidence calculation, then evaluates thresholds to decide the final label or fallback category.
- **Nodes Involved:**
  - `Classify Email with Jev`
  - `Decide Label from Confidence`

- **Node Details:**
  - **`Classify Email with Jev`**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action Node)
    - *Configuration Choices:* Method: `POST`. URL: `https://api.typesafe.ai/v1/systemone`. Batching enabled with a batch size of 10. Timeout: 60,000ms. Error handling: `continueRegularOutput` enabled. Max retries: 3 with a 2-second wait.
    - *Key Expressions/Variables:* Constructs a JSON payload mapping `state` parameters (`from`, `to`, `date`, `subject`, `body` sliced to 16,000 characters, and filtered `gmail_signals`) and `questions` containing specific classification criteria and choices (`label` and `needs_reply`).
    - *Input/Output:* Inputs: `Fetch Unlabelled Emails`. Outputs: `Decide Label from Confidence`.
    - *Credentials:* `Jev Typesafe API Key` (`httpBearerAuth`)
    - *Edge Cases/Failures:* API timeouts, malformed responses, rate limits (429/529 errors handled via retry-on-fail and continue-on-error fallback routing).
  - **`Decide Label from Confidence`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
    - *Configuration Choices:* Evaluates confidence floors (`CONFIDENCE_FLOOR = 0.55`, `NEEDS_REPLY_FLOOR = 0.75`) against Jev's output to assign either the predicted choice, an escalation to `Action Required`, or a fallback to `Needs Review`.
    - *Key Expressions/Variables:* Uses `$runIndex` to pin current loop payloads via `$('Fetch Unlabelled Emails').all(0, $runIndex)`.
    - *Input/Output:* Inputs: `Classify Email with Jev`. Outputs: `List Gmail Labels`.
    - *Edge Cases/Failures:* Missing probability models or unparsable numerical conversions.

---

#### 2.3 Dynamic Label Management

- **Overview:** Inspects the user's Gmail labels, compares them against the required label names for the current batch, creates any missing labels uniquely, and refreshes label references.
- **Nodes Involved:**
  - `List Gmail Labels`
  - `Find Missing Labels`
  - `If Labels Missing`
  - `Create Gmail Label`
  - `Reload Gmail Labels`

- **Node Details:**
  - **`List Gmail Labels`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node)
    - *Configuration Choices:* Resource: `label`. Execution constraint: `executeOnce: true` (retrieves labels once per workflow execution).
    - *Input/Output:* Inputs: `Decide Label from Confidence`. Outputs: `Find Missing Labels`.
    - *Credentials:* `Gmail Account - OAuth2 API` (`gmailOAuth2`)
  - **`Find Missing Labels`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
    - *Configuration Choices:* Normalizes and diffs required labels against existing Gmail labels to isolate missing names and deduplicate entries. Emits a sentinel item if no work is required.
    - *Input/Output:* Inputs: `List Gmail Labels`. Outputs: `If Labels Missing`.
  - **`If Labels Missing`**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control Node)
    - *Configuration Choices:* Evaluates condition `{{ $json.create }}` using strict boolean operations.
    - *Input/Output:* Inputs: `Find Missing Labels`. Outputs: True branch goes to `Create Gmail Label`; False branch bypasses creation and flows directly to `Reload Gmail Labels`.
  - **`Create Gmail Label`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node)
    - *Configuration Choices:* Resource: `label`, Operation: `create`. Error handling: `continueRegularOutput` enabled with `alwaysOutputData: true` to catch naming conflicts without crashing the run.
    - *Key Expressions/Variables:* Label name parameter: `={{ $json.labelName }}`.
    - *Input/Output:* Inputs: `If Labels Missing` (True branch). Outputs: `Reload Gmail Labels`.
    - *Credentials:* `Gmail Account - OAuth2 API` (`gmailOAuth2`)
    - *Edge Cases/Failures:* Duplicate label name conflicts or reserved system label names.
  - **`Reload Gmail Labels`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node)
    - *Configuration Choices:* Resource: `label`. `executeOnce: true`. Convergence point for creation paths.
    - *Input/Output:* Inputs: `Create Gmail Label` and `If Labels Missing` (False branch). Outputs: `Resolve Label Ids`.
    - *Credentials:* `Gmail Account - OAuth2 API` (`gmailOAuth2`)

---

#### 2.4 Application & Looping Execution

- **Overview:** Maps resolved string label names to Gmail IDs, applies the labels to target emails, and manages the execution loop to continue pulling new batches until backlogs clear.
- **Nodes Involved:**
  - `Resolve Label Ids`
  - `Apply Label to Email`
  - `Check for Next Batch`

- **Node Details:**
  - **`Resolve Label Ids`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node)
    - *Configuration Choices:* Maps Gmail label IDs to decisions by normalized string matching. Implements fallback logic to route conflicting labels to `Needs Review`.
    - *Input/Output:* Inputs: `Reload Gmail Labels`. Outputs: `Apply Label to Email`.
  - **`Apply Label to Email`**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Action Node)
    - *Configuration Choices:* Operation: `addLabels`.
    - *Key Expressions/Variables:* Message ID: `={{ $json.messageId }}`, Label IDs: `={{ $json.labelId }}`.
    - *Input/Output:* Inputs: `Resolve Label Ids`. Outputs: `Check for Next Batch`.
    - *Credentials:* `Gmail Account - OAuth2 API` (`gmailOAuth2`)
    - *Edge Cases/Failures:* API message-not-found errors or permission restrictions on label manipulation.
  - **`Check for Next Batch`**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Loop Control Node)
    - *Configuration Choices:* Defines `BATCH_SIZE = 50` and `MAX_ROUNDS = 20`. Stops looping if no emails were labeled, a partial page is received, or the round ceiling is reached.
    - *Input/Output:* Inputs: `Apply Label to Email`. Outputs: Returns items back to `Fetch Unlabelled Emails` to continue the loop, or outputs an empty array to terminate execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Every 5 Minutes** | `scheduleTrigger` | Triggers workflow execution every 5 minutes. | None | Fetch Unlabelled Emails | Intelligent Email Organization with Jev for Gmail... |
| **Fetch Unlabelled Emails** | `gmail` | Queries unlabelled Gmail messages. | Every 5 Minutes, Check for Next Batch | Classify Email with Jev | Fetch unlabelled emails: Searches Gmail every five minutes for messages that carry no user label, 50 per round. |
| **Classify Email with Jev** | `httpRequest` | Sends email data to Jev for AI classification. | Fetch Unlabelled Emails | Decide Label from Confidence | Classify and decide: Jev returns a typed label and a calibrated confidence; code turns that into one label name. |
| **Decide Label from Confidence** | `code` | Evaluates confidence floors and assigns label names. | Classify Email with Jev | List Gmail Labels | Classify and decide: Jev returns a typed label and a calibrated confidence; code turns that into one label name. |
| **List Gmail Labels** | `gmail` | Fetches existing user and system labels once per run. | Decide Label from Confidence | Find Missing Labels | Create missing labels once: Diffs the needed names against Gmail and creates each new label exactly once, before any id is read. |
| **Find Missing Labels** | `code` | Diffs required and existing labels to isolate missing ones. | List Gmail Labels | If Labels Missing | Create missing labels once: Diffs the needed names against Gmail and creates each new label exactly once, before any id is read. |
| **If Labels Missing** | `if` | Branches execution depending on whether labels need creation. | Find Missing Labels | Create Gmail Label, Reload Gmail Labels | Create missing labels once: Diffs the needed names against Gmail and creates each new label exactly once, before any id is read. |
| **Create Gmail Label** | `gmail` | Creates a new unique label in Gmail. | If Labels Missing | Reload Gmail Labels | Create missing labels once: Diffs the needed names against Gmail and creates each new label exactly once, before any id is read. |
| **Reload Gmail Labels** | `gmail` | Refreshes the list of Gmail labels after creation steps. | If Labels Missing, Create Gmail Label | Resolve Label Ids | Create missing labels once: Diffs the needed names against Gmail and creates each new label exactly once, before any id is read. |
| **Resolve Label Ids** | `code` | Maps resolved string labels to fresh Gmail IDs. | Reload Gmail Labels | Apply Label to Email | Apply labels and loop: Attaches fresh label ids, applies them, then loops back until the backlog is drained. |
| **Apply Label to Email** | `gmail` | Applies the resolved Gmail label to the target message. | Resolve Label Ids | Check for Next Batch | Apply labels and loop: Attaches fresh label ids, applies them, then loops back until the backlog is drained. |
| **Check for Next Batch** | `code` | Controls loop iteration and pagination limits. | Apply Label to Email | Fetch Unlabelled Emails (Loop) | Apply labels and loop: Attaches fresh label ids, applies them, then loops back until the backlog is drained. |

---

### 4. Reproducing the Workflow from Scratch

Follow these manual steps to rebuild the workflow:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every 5 Minutes`.
   - Set interval configuration to run every `1` `minutes`.

2. **Add the Initial Fetch Node:**
   - Add a **Gmail** node named `Fetch Unlabelled Emails` (`getAll` operation).
   - Link `Every 5 Minutes` output to this node.
   - Configure parameters: Set `Simple` to `false`. Set query filter (`q`) to `has:nouserlabels -in:chats -in:sent -in:draft`.
   - Select your configured `Gmail Account - OAuth2 API` credential.

3. **Configure AI Classification:**
   - Add an **HTTP Request** node named `Classify Email with Jev` (`POST` method).
   - Set URL to `https://api.typesafe.ai/v1/systemone`. Enable batching (size 10), timeout (60,000ms), max retries (3), and set error handling to `Continue On Fail`.
   - Add your `Jev Typesafe API Key` credential via generic HTTP Bearer Auth.
   - Populate the JSON body with state extraction variables (`from`, `to`, `date`, `subject`, body slice, and filtered `gmail_signals`) alongside the taxonomy choices (`Action Required`, `Bank`, `Certifications`, `Event Invite`, `Follow-up Reminder`, `Inquiry`, `Invoice`, `Job Update`, `Newsletter`, `Personal`, `Receipt`, `Social / Networking`, `Spam / Junk`, `Subscription Renewal`, `System Notification`, `Tax Documents`, `Work Documents`, `Other / Uncategorized`) and `needs_reply` criteria.

4. **Implement Confidence & Decision Evaluation:**
   - Add a **Code** node named `Decide Label from Confidence`.
   - Insert JavaScript to parse Jev's answer probabilities, checking against `CONFIDENCE_FLOOR` (0.55) and `NEEDS_REPLY_FLOOR` (0.75), pinning data using `properties.all(0, $runIndex)`.

5. **List Gmail Labels:**
   - Add a **Gmail** node named `List Gmail Labels` (Resource: `label`, Operation: `get` or default label listing).
   - Enable `Execute Once` in node options. Select your Gmail OAuth2 credential.

6. **Find Missing Labels:**
   - Add a **Code** node named `Find Missing Labels`.
   - Insert JavaScript to normalize labels, compute the difference between wanted category names and existing Gmail labels, and emit missing items or a sentinel fallback.

7. **Conditional Flow Control:**
   - Add an **If** node named `If Labels Missing`.
   - Configure condition expression: `{{ $json.create }}` evaluates to `true`.

8. **Create Missing Labels:**
   - Add a **Gmail** node named `Create Gmail Label` (Resource: `label`, Operation: `create`).
   - Connect the True branch of `If Labels Missing` to this node. Set name parameter to `={{ $json.labelName }}`. Enable `Continue On Fail` with always output data. Select your Gmail OAuth2 credential.

9. **Reload Gmail Labels:**
   - Add a **Gmail** node named `Reload Gmail Labels` (Resource: `label`, Operation: `list/get`).
   - Enable `Execute Once`. Connect both the False branch of `If Labels Missing` and the output of `Create Gmail Label` to this node. Select your Gmail OAuth2 credential.

10. **Resolve Label IDs:**
    - Add a **Code** node named `Resolve Label Ids`.
    - Insert JavaScript to map fresh Gmail label IDs back onto email decisions by normalized string matching, falling back to `Needs Review` if a label creation failed.

11. **Apply Labels:**
    - Add a **Gmail** node named `Apply Label to Email` (Operation: `addLabels`).
    - Set Message ID to `={{ $json.messageId }}` and Label IDs to `={{ $json.labelId }}`. Select your Gmail OAuth2 credential.

12. **Configure Loop and Pagination Control:**
    - Add a **Code** node named `Check for Next Batch`.
    - Insert loop-guard JavaScript containing `BATCH_SIZE = 50` and `MAX_ROUNDS = 20`.
    - Connect the output of this node back to `Fetch Unlabelled Emails` to establish the iteration loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Intelligent Email Organization with Jev for Gmail | Workflow Title and Branding |
| TypeSafe System One API Reference | External AI Model Documentation (`https://api.typesafe.ai/v1/systemone`) |