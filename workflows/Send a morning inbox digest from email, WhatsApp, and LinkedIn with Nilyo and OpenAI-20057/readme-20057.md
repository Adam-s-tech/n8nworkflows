Send a morning inbox digest from email, WhatsApp, and LinkedIn with Nilyo and OpenAI

https://n8nworkflows.xyz/workflows/send-a-morning-inbox-digest-from-email--whatsapp--and-linkedin-with-nilyo-and-openai-20057


# Send a morning inbox digest from email, WhatsApp, and LinkedIn with Nilyo and OpenAI

### 1. Workflow Overview

This workflow automates the generation and delivery of a daily morning intelligence briefing by consolidating communication streams from WhatsApp, email, and LinkedIn. It runs on a scheduled basis, queries multiple platforms via the Nilyo community integration, synthesizes the gathered data using an OpenAI language model, and delivers a categorized, concise digest back to the user via WhatsApp.

The workflow logic is divided into four distinct functional blocks:
- **1.1 Schedule & Configuration Setup:** Initializes execution daily at 08:00 and defines global configuration parameters such as the target recipient and the historical lookback window.
- **1.2 Multi-Channel Data Retrieval & Synchronization:** Fan-out architecture that concurrently queries WhatsApp, email, and LinkedIn sources with fault-tolerant error handling, followed by a multi-input synchronization merge.
- **1.3 Data Aggregation & AI Processing:** Combines the synchronized records into a unified dataset and leverages an OpenAI chat model to generate a plain-text briefing grouped by urgency.
- **1.4 Delivery:** Transmits the finalized AI-generated briefing to the designated contact via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration Setup
- **Overview:** Triggers the workflow execution every morning at a fixed time and injects required contextual variables (recipient name and lookback duration) into the data stream.
- **Nodes Involved:** 
  - `Daily Schedule Trigger`
  - `Set Recipient and Hours`
- **Node Details:**
  - **Daily Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Entry point initiating execution.
    - *Configuration:* Trigger rule configured for 08:00 AM daily.
    - *Inputs:* None.
    - *Outputs:* Triggers `Set Recipient and Hours`.
    - *Edge Cases:* Missed triggers if the n8n instance is offline at 08:00 AM (depends on n8n catchup policies).
  - **Set Recipient and Hours** (`n8n-nodes-base.set`)
    - *Role:* Data injection node setting up parameters for subsequent API calls and delivery.
    - *Configuration:* Defines two fixed assignments: `digest_recipient` (string, value: "Me") and `lookback_hours` (number, value: "16").
    - *Inputs:* `Daily Schedule Trigger`.
    - *Outputs:* Fan-out output connected to all three fetch nodes simultaneously.
    - *Edge Cases:* Expression evaluation failures if downstream nodes attempt to reference missing property keys.

#### 2.2 Multi-Channel Data Retrieval & Synchronization
- **Overview:** Executes parallel data gathering operations across WhatsApp, email, and LinkedIn via Nilyo tools with error resiliency enabled, consolidating the separate execution branches into a single stream.
- **Nodes Involved:**
  - `Fetch WhatsApp Messages`
  - `Fetch Emails`
  - `Fetch LinkedIn Invitations`
  - `Merge Sources Data`
- **Node Details:**
  - **Fetch WhatsApp Messages** (`n8n-nodes-nilyo.nilyo`)
    - *Role:* Retrieves recent WhatsApp messages via Nilyo tool execution.
    - *Configuration:* Resource set to `tool`, tool name set to `messaging_list_recent_messages`, operation set to `call`. Passes dynamic arguments evaluating the lookback window.
    - *Key Expressions:* `={{ JSON.stringify({ provider: 'whatsapp', after: $now.minus({ hours: $json.lookback_hours }).toUTC().toISO(), chat_type: 'all', max_chats: 25 }) }}`
    - *Inputs:* `Set Recipient and Hours`.
    - *Outputs:* `Merge Sources Data` (Input Index 0).
    - *Error Handling:* `onError: "continueRegularOutput"` ensures pipeline continuity if WhatsApp API fails.
    - *Edge Cases:* Rate-limiting, API credential invalidation, or malformed ISO date generation.
  - **Fetch Emails** (`n8n-nodes-nilyo.nilyo`)
    - *Role:* Retrieves recent incoming emails through Nilyo tool execution.
    - *Configuration:* Resource set to `tool`, tool name set to `email_list_messages`, operation set to `call`.
    - *Key Expressions:* `={{ JSON.stringify({ after: $now.minus({ hours: $json.lookback_hours }).toUTC().toISO(), limit: 25 }) }}`
    - *Inputs:* `Set Recipient and Hours`.
    - *Outputs:* `Merge Sources Data` (Input Index 1).
    - *Error Handling:* `onError: "continueRegularOutput"`.
    - *Edge Cases:* Mailbox authorization expiration, empty message arrays.
  - **Fetch LinkedIn Invitations** (`n8n-nodes-nilyo.nilyo`)
    - *Role:* Retrieves pending incoming LinkedIn invitations.
    - *Configuration:* Resource set to `linkedin`, operation set to `listInvitations`, invitation type set to `received`.
    - *Inputs:* `Set Recipient and Hours`.
    - *Outputs:* `Merge Sources Data` (Input Index 2).
    - *Error Handling:* `onError: "continueRegularOutput"`.
    - *Edge Cases:* LinkedIn API connectivity timeouts or account disconnection within Nilyo.
  - **Merge Sources Data** (`n8n-nodes-base.merge`)
    - *Role:* Synchronizes three independent execution branches into a single item list.
    - *Configuration:* Mode configured for 3 inputs (`numberInputs: 3`).
    - *Inputs:* `Fetch WhatsApp Messages` (Input 0), `Fetch Emails` (Input 1), `Fetch LinkedIn Invitations` (Input 2).
    - *Outputs:* `Aggregate All Data`.
    - *Edge Cases:* Execution stalls if any input branch hangs without returning items (mitigated by error continuation settings on preceding nodes).

#### 2.3 Data Aggregation & AI Processing
- **Overview:** Flattens the multi-source dataset into a single JSON payload and utilizes an OpenAI language model chain to draft a structured textual briefing.
- **Nodes Involved:**
  - `Aggregate All Data`
  - `Generate Briefing`
  - `OpenAI Chat Model`
- **Node Details:**
  - **Aggregate All Data** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates multiple items from the merge node into one unified dataset object.
    - *Configuration:* Aggregation mode set to `aggregateAllItemData`.
    - *Inputs:* `Merge Sources Data`.
    - *Outputs:* `Generate Briefing`.
    - *Edge Cases:* Payload size limitations if communication volume within the lookback window is exceptionally high.
  - **Generate Briefing** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Role:* LLM chain node formatting the prompt and querying the connected language model.
    - *Configuration:* Prompt type set to `define`. Restricts payload character length to 12,000 characters.
    - *Key Expressions:* Prompt template explicitly requests grouping into "needs an answer today", "can wait", and "ignore", with constraints to keep output under 200 words and strictly grounded in the provided data: `=Write a short morning briefing in plain text from this data. Group it as: needs an answer today, can wait, ignore. Name the sender and the channel on each line, keep it under 200 words, and never mention anything that is not in the data.\n\n{{ JSON.stringify($json.data).slice(0, 12000) }}`
    - *Inputs:* Main input from `Aggregate All Data`; AI Language Model input from `OpenAI Chat Model`.
    - *Outputs:* `Dispatch Digest via WhatsApp`.
    - *Edge Cases:* Token limit overruns, API rate limits, or hallucinations if data structures change unexpectedly.
  - **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Language model provider configuration sub-node.
    - *Configuration:* Standard OpenAI chat model parameters using configured OpenAI credentials.
    - *Inputs:* Connected to `Generate Briefing` via `ai_languageModel` connection.
    - *Outputs:* None (provides model context).
    - *Credentials:* `openAiApi Credential`.
    - *Edge Cases:* OpenAI API outages, insufficient account quota, or invalid credentials.

#### 2.4 Delivery
- **Overview:** Takes the generated plain-text briefing and transmits it to the specified contact using the Nilyo WhatsApp messaging integration.
- **Nodes Involved:**
  - `Dispatch Digest via WhatsApp`
- **Node Details:**
  - **Dispatch Digest via WhatsApp** (`n8n-nodes-nilyo.nilyo`)
    - *Role:* Sends the synthesized text message to the target recipient via WhatsApp.
    - *Configuration:* Resource set to `messaging`, operation set to `sendToContact`, provider set to `whatsapp`.
    - *Key Expressions:* 
      - Text body: `={{ $json.text }}`
      - Recipient name: `={{ $('Set Recipient and Hours').first().json.digest_recipient }}`
    - *Inputs:* `Generate Briefing`.
    - *Outputs:* None (Terminal node).
    - *Credentials:* `nilyoApi Credential`.
    - *Edge Cases:* Invalid recipient name matching, WhatsApp API delivery failures, or phone number resolution errors in Nilyo.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Template description** | `n8n-nodes-base.stickyNote` | Documentation and setup guide | None | None | ## Send a morning digest of email, WhatsApp and LinkedIn to WhatsApp<br><br>### Who is it for<br>Anyone whose incoming work is spread across a mailbox, WhatsApp and LinkedIn, and who wants one short briefing instead of three inboxes to open.<br><br>### How it works<br>Every weekday at 8, the workflow reads three sources through Nilyo on your own accounts, each limited to the lookback window: WhatsApp messages received, emails received and pending LinkedIn invitations. A merge node waits for the three branches so the briefing is written once, an LLM writes a briefing sorted by what needs an answer today, and the text is sent to you on WhatsApp. Each read is set to continue on error, so a provider that is temporarily unavailable does not cancel the digest.<br><br>### How to set up<br>1. Create a personal token in Nilyo (Agent access) and store it in a **Nilyo API** credential.<br><br>2. Connect the mailbox, WhatsApp and LinkedIn in Nilyo.<br>3. In **Set Recipient and Hours**, set the WhatsApp recipient of the digest and the lookback window in hours.<br>4. Add your OpenAI credential to the chat model node.<br><br>### Requirements<br>A Nilyo account with the accounts connected, and an OpenAI account.<br><br>### How to customize<br>Send the briefing to Slack or by email, change the schedule, drop a source, or ask the model for a different structure in the prompt.<br><br>### Community node<br>This template uses the **Nilyo** community node (`n8n-nodes-nilyo`), verified by n8n. On n8n Cloud, enable Verified Community Nodes in the Admin Panel and add Nilyo from the canvas. On self-hosted n8n, install `n8n-nodes-nilyo` from Settings > Community nodes. |
| **Step 1** | `n8n-nodes-base.stickyNote` | Documentation note for trigger and settings | None | None | ## 1. Choose the window<br><br>The schedule decides when the digest runs. **Set Recipient and Hours** holds the two values to adapt: who receives it and how far back to look. |
| **Step 2** | `n8n-nodes-base.stickyNote` | Documentation note for fetch and merge blocks | None | None | ## 2. Read three accounts, then wait<br><br>Three Nilyo reads on your own accounts, each limited to the lookback window. They continue on error, so one unavailable provider does not cancel the digest. The 3-input merge waits for every branch — without it the digest would be sent once per source. |
| **Step 3** | `n8n-nodes-base.stickyNote` | Documentation note for aggregation and LLM processing | None | None | ## 3. Group it and write<br><br>Everything becomes a single item, so the model runs once instead of once per message. It writes the briefing sorted by what needs an answer today. |
| **Daily Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Triggers execution daily at 08:00 | None | Set Recipient and Hours | |
| **Set Recipient and Hours** | `n8n-nodes-base.set` | Sets recipient name and lookback time window | Daily Schedule Trigger | Fetch WhatsApp Messages, Fetch Emails, Fetch LinkedIn Invitations | ## 1. Choose the window<br><br>The schedule decides when the digest runs. **Set Recipient and Hours** holds the two values to adapt: who receives it and how far back to look. |
| **Fetch WhatsApp Messages** | `n8n-nodes-nilyo.nilyo` | Retrieves recent WhatsApp messages via Nilyo tool | Set Recipient and Hours | Merge Sources Data | ## 2. Read three accounts, then wait<br><br>Three Nilyo reads on your own accounts, each limited to the lookback window. They continue on error, so one unavailable provider does not cancel the digest. The 3-input merge waits for every branch — without it the digest would be sent once per source. |
| **Fetch Emails** | `n8n-nodes-nilyo.nilyo` | Retrieves recent emails via Nilyo tool | Set Recipient and Hours | Merge Sources Data | ## 2. Read three accounts, then wait<br><br>Three Nilyo reads on your own accounts, each limited to the lookback window. They continue on error, so one unavailable provider does not cancel the digest. The 3-input merge waits for every branch — without it the digest would be sent once per source. |
| **Fetch LinkedIn Invitations** | `n8n-nodes-nilyo.nilyo` | Retrieves pending LinkedIn invitations via Nilyo | Set Recipient and Hours | Merge Sources Data | ## 2. Read three accounts, then wait<br><br>Three Nilyo reads on your own accounts, each limited to the lookback window. They continue on error, so one unavailable provider does not cancel the digest. The 3-input merge waits for every branch — without it the digest would be sent once per source. |
| **Merge Sources Data** | `n8n-nodes-base.merge` | Synchronizes the three data fetch branches | Fetch WhatsApp Messages, Fetch Emails, Fetch LinkedIn Invitations | Aggregate All Data | ## 2. Read three accounts, then wait<br><br>Three Nilyo reads on your own accounts, each limited to the lookback window. They continue on error, so one unavailable provider does not cancel the digest. The 3-input merge waits for every branch — without it the digest would be sent once per source. |
| **Aggregate All Data** | `n8n-nodes-base.aggregate` | Combines merged items into a single dataset | Merge Sources Data | Generate Briefing | ## 3. Group it and write<br><br>Everything becomes a single item, so the model runs once instead of once per message. It writes the briefing sorted by what needs an answer today. |
| **Generate Briefing** | `@n8n/n8n-nodes-langchain.chainLlm` | LLM chain generating the categorized text briefing | Aggregate All Data (Main), OpenAI Chat Model (AI Model) | Dispatch Digest via WhatsApp | ## 3. Group it and write<br><br>Everything becomes a single item, so the model runs once instead of once per message. It writes the briefing sorted by what needs an answer today. |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides the OpenAI LLM configuration for the chain | None | Generate Briefing | ## 3. Group it and write<br><br>Everything becomes a single item, so the model runs once instead of once per message. It writes the briefing sorted by what needs an answer today. |
| **Dispatch Digest via WhatsApp** | `n8n-nodes-nilyo.nilyo` | Sends the generated briefing back via WhatsApp | Generate Briefing | None | ## 4. Send it to you<br><br>Nilyo sends the finished briefing to your own WhatsApp. This is the only step that writes anything out. |
| **Step 4** | `n8n-nodes-base.stickyNote` | Documentation note for final dispatch node | None | None | ## 4. Send it to you<br><br>Nilyo sends the finished briefing to your own WhatsApp. This is the only step that writes anything out. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in an n8n instance:

1. **Install Prerequisites:**
   - Ensure the community node `n8n-nodes-nilyo` is installed (via Settings > Community nodes on self-hosted instances, or enabled in the Admin Panel on n8n Cloud).
2. **Create the Trigger:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`). Name it `Daily Schedule Trigger`. Configure the interval rule to trigger at hour `8` and minute `0`.
3. **Configure Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`). Name it `Set Recipient and Hours`.
   - Add two assignments:
     - Name: `digest_recipient`, Type: `String`, Value: `Me`
     - Name: `lookback_hours`, Type: `Number`, Value: `16`
   - Connect `Daily Schedule Trigger` output to `Set Recipient and Hours`.
4. **Create Data Fetch Branches:**
   - **WhatsApp Fetch:**
     - Add a **Nilyo** node (`n8n-nodes-nilyo.nilyo`). Name it `Fetch WhatsApp Messages`.
     - Set Resource to `tool`, Tool Name to `messaging_list_recent_messages`, Operation to `call`.
     - Set Tool Arguments to expression: `={{ JSON.stringify({ provider: 'whatsapp', after: $now.minus({ hours: $json.lookback_hours }).toUTC().toISO(), chat_type: 'all', max_chats: 25 }) }}`
     - Configure node error setting: **On Error** -> *Continue (using output).*
     - Connect `Set Recipient and Hours` to this node.
   - **Email Fetch:**
     - Add a **Nilyo** node (`n8n-nodes-nilyo.nilyo`). Name it `Fetch Emails`.
     - Set Resource to `tool`, Tool Name to `email_list_messages`, Operation to `call`.
     - Set Tool Arguments to expression: `={{ JSON.stringify({ after: $now.minus({ hours: $json.lookback_hours }).toUTC().toISO(), limit: 25 }) }}`
     - Configure node error setting: **On Error** -> *Continue (using output).*
     - Connect `Set Recipient and Hours` to this node.
   - **LinkedIn Fetch:**
     - Add a **Nilyo** node (`n8n-nodes-nilyo.nilyo`). Name it `Fetch LinkedIn Invitations`.
     - Set Resource to `linkedin`, Operation to `listInvitations`, Invitation Type to `received`.
     - Configure node error setting: **On Error** -> *Continue (using output).*
     - Connect `Set Recipient and Hours` to this node.
5. **Merge Branches:**
   - Add a **Merge** node (`n8n-nodes-base.merge`). Name it `Merge Sources Data`.
   - Set Number of Inputs to `3`.
   - Connect `Fetch WhatsApp Messages` to Input 0, `Fetch Emails` to Input 1, and `Fetch LinkedIn Invitations` to Input 2.
6. **Aggregate Data:**
   - Add an **Aggregate** node (`n8n-nodes-base.aggregate`). Name it `Aggregate All Data`.
   - Set Aggregate to `Aggregate All Item Data`.
   - Connect `Merge Sources Data` output to this node.
7. **Configure AI Generation:**
   - Add an **Advanced AI -> Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`). Name it `Generate Briefing`.
   - Set Prompt Type to `define`.
   - Set Text prompt to:
     ```text
     =Write a short morning briefing in plain text from this data. Group it as: needs an answer today, can wait, ignore. Name the sender and the channel on each line, keep it under 200 words, and never mention anything that is not in the data.

     {{ JSON.stringify($json.data).slice(0, 12000) }}
     ```
   - Connect `Aggregate All Data` to the main input of `Generate Briefing`.
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Name it `OpenAI Chat Model`. Connect its `ai_languageModel` output to the corresponding input on `Generate Briefing`.
8. **Configure Output Delivery:**
   - Add a **Nilyo** node (`n8n-nodes-nilyo.nilyo`). Name it `Dispatch Digest via WhatsApp`.
   - Set Resource to `messaging`, Operation to `sendToContact`, Provider to `whatsapp`.
   - Set Text parameter to: `={{ $json.text }}`
   - Set Recipient Name parameter to: `={{ $('Set Recipient and Hours').first().json.digest_recipient }}`
   - Connect `Generate Briefing` main output to `Dispatch Digest via WhatsApp`.
9. **Credentials Setup:**
   - Create a **Nilyo API** credential using a valid Nilyo personal token (Agent access) and assign it to all three Nilyo nodes and the final dispatch node.
   - Create an **OpenAI API** credential and assign it to the `OpenAI Chat Model` node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Node Requirement | This template requires the verified `n8n-nodes-nilyo` community integration package. |
| Credential Scope | All Nilyo nodes share a single Nilyo API credential configured with Agent access. |