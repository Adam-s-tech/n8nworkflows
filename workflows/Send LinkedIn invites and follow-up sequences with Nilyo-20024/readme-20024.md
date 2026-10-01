Send LinkedIn invites and follow-up sequences with Nilyo

https://n8nworkflows.xyz/workflows/send-linkedin-invites-and-follow-up-sequences-with-nilyo-20024


# Send LinkedIn invites and follow-up sequences with Nilyo

### 1. Workflow Overview

This workflow automates LinkedIn outreach and multi-step follow-up sequences using the Nilyo API and verified community nodes. It operates on a daily schedule or manual execution, processing two distinct operational branches: an **Invitation Pipeline** that sources and invites prospects from LinkedIn search queries, and a **Sequence Pipeline** that evaluates recent connections, reads conversation histories, halts sequences upon prospect replies, and dispatches scheduled opening messages or follow-up texts.

The system is logically structured into the following functional blocks:

- **1.1 Trigger & Configuration**: Initiates execution via schedule or manual test, establishing central workflow variables, parameters, message templates, and dry-run flags.
- **1.2 Invitation Candidate Discovery**: Fetches active pending invitations and translates user-selected search configurations (URLs, saved searches, or JSON parameter objects) into executable Nilyo tool calls.
- **1.3 Invitation Batch Filtering**: Cross-references search results against pending and connected profiles, filters out duplicates, caps volume according to daily limits, and honors dry-run test modes.
- **1.4 Invitation Dispatch & Throttling**: Renders personalized invitation notes, executes delivery via the Nilyo API, interprets delivery outcomes (skipping cooldowns or halting on rate limits), and enforces pacing delays between requests.
- **1.5 Sequence Audience Identification**: Retrieves recent connections from LinkedIn and filters prospects against a sliding time-window configuration.
- **1.6 Conversation Analysis**: Locates and retrieves full chat histories for targeted participants to evaluate response states and message counts.
- **1.7 Message Decision Logic**: Evaluates thread state to determine if messages should be sent, paused for spacing delays, or aborted due to inbound replies.
- **1.8 Message Delivery**: Determines whether to push messages into an existing chat thread or initiate a new conversation sequence via Nilyo.

---

### 2. Block-by-Block Analysis

---

### 1.1 Trigger & Configuration

#### Overview
This block establishes entry points for the workflow and initializes all operational parameters, including search criteria, rate limits, message bodies, spacing rules, and dry-run safety flags.

#### Nodes Involved
- `Every Day at 9:30 AM` (`n8n-nodes-base.scheduleTrigger`)
- `Manual Workflow Test` (`n8n-nodes-base.manualTrigger`)
- `Set Workflow Parameters` (`n8n-nodes-base.set`)

#### Node Details

- **Every Day at 9:30 AM**
  - **Type & Technical Role**: Schedule Trigger node executing the workflow daily at 09:30.
  - **Configuration Choices**: Configured with a daily interval rule targeting hour 9, minute 30.
  - **Key Expressions / Variables**: None.
  - **Input / Output Connections**: Input: None (Root node). Output: `Set Workflow Parameters`.
  - **Version Requirements**: `typeVersion: 1.2`
  - **Edge Cases & Failure Types**: Schedule execution relies on n8n core worker availability.
  - **Sub-workflow Reference**: None.

- **Manual Workflow Test**
  - **Type & Technical Role**: Manual Trigger node allowing developers to execute the workflow on demand for testing.
  - **Configuration Choices**: Standard manual trigger with default settings.
  - **Key Expressions / Variables**: None.
  - **Input / Output Connections**: Input: None (Root node). Output: `Set Workflow Parameters`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

- **Set Workflow Parameters**
  - **Type & Technical Role**: Edit Fields (Set) node defining all control parameters, search queries, template strings, and timing configurations.
  - **Configuration Choices**: Assigns boolean flags (`dry_run: true`), search configuration strings (`source`, `search_url`, `search_product`, `search_tool`, `search_arguments`, `saved_search_tool`, `saved_search_id`), pacing limits (`invitations_per_day: 25`, `seconds_between_invitations: 90`), message templates (`invitation_note`, `opening_message`, `follow_up_1`, `follow_up_2`, `follow_up_3`), and temporal windows (`days_between_messages: 4`, `sequence_window_days: 30`).
  - **Key Expressions / Variables**: Static assignment values consumed downstream across both parallel branches.
  - **Input / Output Connections**: Input: `Every Day at 9:30 AM`, `Manual Workflow Test`. Output: `Check Pending Invitations`, `Check New Connections`.
  - **Version Requirements**: `typeVersion: 3.4`
  - **Edge Cases & Failure Types**: Missing or malformed JSON inside `search_arguments` can cause downstream parsing errors.
  - **Sub-workflow Reference**: None.

---

### 1.2 Invitation Candidate Discovery

#### Overview
This block inspects existing pending LinkedIn invitations to prevent duplicate invites, then translates the chosen search configuration into an executable Nilyo API query.

#### Nodes Involved
- `Check Pending Invitations` (`n8n-nodes-nilyo.nilyo`)
- `Construct Search API Call` (`n8n-nodes-base.code`)
- `Execute People Search` (`n8n-nodes-nilyo.nilyo`)

#### Node Details

- **Check Pending Invitations**
  - **Type & Technical Role**: Community node calling Nilyo to list active sent invitations.
  - **Configuration Choices**: Resource: `linkedin`, Operation: `listInvitations`, Invitation Type: `sent`. Error handling is set to `continueRegularOutput` with `alwaysOutputData: true`.
  - **Key Expressions / Variables**: None.
  - **Input / Output Connections**: Input: `Set Workflow Parameters`. Output: `Construct Search API Call`.
  - **Version Requirements**: `typeVersion: 1` (Requires `n8n-nodes-nilyo`).
  - **Edge Cases & Failure Types**: API authentication errors or rate limits will be passed down via regular output due to error handling configuration.
  - **Sub-workflow Reference**: None.

- **Construct Search API Call**
  - **Type & Technical Role**: JavaScript Code node translating parameters from the settings node into a unified Nilyo tool name and arguments payload.
  - **Configuration Choices**: Reads `source` parameter; branches logic to handle `saved_search`, raw `parameters`, or default `search_url` configurations.
  - **Key Expressions / Variables**: Accesses `$('Set Workflow Parameters').first().json`.
  - **Input / Output Connections**: Input: `Check Pending Invitations`. Output: `Execute People Search`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Throws an explicit error if `search_arguments` contains invalid JSON strings.
  - **Sub-workflow Reference**: None.

- **Execute People Search**
  - **Type & Technical Role**: Community node invoking a Nilyo dynamic tool to retrieve LinkedIn search results.
  - **Configuration Choices**: Resource: `tool`, Tool Name: `={{ $json.tool }}`, Operation: `call`, Tool Arguments: `={{ $json.args }}`.
  - **Key Expressions / Variables**: Evaluates dynamic tool name and argument payload expressions.
  - **Input / Output Connections**: Input: `Construct Search API Call`. Output: `Shortlist Candidates to Invite`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: Tool execution failures or invalid parameter schemas from LinkedIn search queries.
  - **Sub-workflow Reference**: None.

---

### 1.3 Invitation Batch Filtering

#### Overview
This block filters raw search results by eliminating profiles with pending invites, existing first-degree or self connections, and duplicates, while enforcing daily caps and dry-run boundaries.

#### Nodes Involved
- `Shortlist Candidates to Invite` (`n8n-nodes-base.code`)
- `If Dry Run` (`n8n-nodes-base.if`)
- `Batch Process Invitations` (`n8n-nodes-base.splitInBatches`)

#### Node Details

- **Shortlist Candidates to Invite**
  - **Type & Technical Role**: JavaScript Code node filtering search results and constructing clean candidate objects.
  - **Configuration Choices**: Compiles user IDs from pending invitations, iterates search items to filter out duplicates, self connections, and first-degree connections, derives first names from display names, and slices the array to `invitations_per_day`.
  - **Key Expressions / Variables**: Accesses `$('Set Workflow Parameters')` and `$('Check Pending Invitations')`.
  - **Input / Output Connections**: Input: `Execute People Search`. Output: `If Dry Run`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Profiles missing `id` attributes or unparseable display names are safely ignored.
  - **Sub-workflow Reference**: None.

- **If Dry Run**
  - **Type & Technical Role**: Conditional Branching node determining whether execution should proceed with live actions or halt.
  - **Configuration Choices**: Evaluates whether `dry_run` is enabled or execution mode is set to test. Condition uses loose type validation.
  - **Key Expressions / Variables**: `={{ $('Set Workflow Parameters').first().json.dry_run || $execution.mode === 'test' }}`
  - **Input / Output Connections**: Input: `Shortlist Candidates to Invite`. Output: `Batch Process Invitations` (True branch; false branch terminates).
  - **Version Requirements**: `typeVersion: 2.2`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

- **Batch Process Invitations**
  - **Type & Technical Role**: Loop Controller node processing invitation candidates one item at a time.
  - **Configuration Choices**: Batch size set to default (1 item per iteration), with reset option disabled.
  - **Key Expressions / Variables**: Iterates items from `If Dry Run`.
  - **Input / Output Connections**: Input: `If Dry Run`, `Wait Between Invitations`. Output: Loop body (`Compose Invitation Note`) / Loop completion (empty).
  - **Version Requirements**: `typeVersion: 3`
  - **Edge Cases & Failure Types**: Empty input arrays complete the loop immediately.
  - **Sub-workflow Reference**: None.

---

### 1.4 Invitation Dispatch & Throttling

#### Overview
This block renders personalized invitation notes, dispatches invitations via Nilyo, interprets error outcomes to prevent retry loops on cooldowns, and pauses between requests.

#### Nodes Involved
- `Compose Invitation Note` (`n8n-nodes-base.code`)
- `Dispatch Invitation` (`n8n-nodes-nilyo.nilyo`)
- `Parse Invitation Outcome` (`n8n-nodes-base.code`)
- `If Stop Inviting` (`n8n-nodes-base.if`)
- `Wait Between Invitations` (`n8n-nodes-base.wait`)

#### Node Details

- **Compose Invitation Note**
  - **Type & Technical Role**: JavaScript Code node personalizing invitation note templates.
  - **Configuration Choices**: Replaces `{first_name}` and `{display_name}` placeholders inside the configured `invitation_note`.
  - **Key Expressions / Variables**: Accesses current item data and `Set Workflow Parameters`.
  - **Input / Output Connections**: Input: `Batch Process Invitations`. Output: `Dispatch Invitation`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Missing placeholder variables fall back safely.
  - **Sub-workflow Reference**: None.

- **Dispatch Invitation**
  - **Type & Technical Role**: Community node calling Nilyo to send a LinkedIn connection invitation.
  - **Configuration Choices**: Resource: `linkedin`, Operation: `sendInvitation`, User ID: `={{ $json.user_id }}`, Message: `={{ $json.note }}`. Error handling set to `continueRegularOutput`.
  - **Key Expressions / Variables**: Evaluates `$json.user_id` and `$json.note`.
  - **Input / Output Connections**: Input: `Compose Invitation Note`. Output: `Parse Invitation Outcome`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Limits**: Network timeouts or API errors are captured via regular output.
  - **Sub-workflow Reference**: None.

- **Parse Invitation Outcome**
  - **Type & Technical Role**: JavaScript Code node categorizing API response codes into operational outcomes.
  - **Configuration Choices**: Inspects error codes; classifies outcomes into `sent`, `skipped` (for existing invites or cooldowns), `stop_for_today` (for quotas and rate limits), or `failed`.
  - **Key Expressions / Variables**: Accesses `$json` response data and `Compose Invitation Note`.
  - **Input / Output Connections**: Input: `Dispatch Invitation`. Output: `If Stop Inviting`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Unrecognized error payloads default to `failed`.
  - **Sub-workflow Reference**: None.

- **If Stop Inviting**
  - **Type & Technical Role**: Conditional Branching node checking if rate limits or quota thresholds require halting the daily run.
  - **Configuration Choices**: Evaluates whether `outcome` equals `stop_for_today`.
  - **Key Expressions / Variables**: `={{ $json.outcome }}`
  - **Input / Output Connections**: Input: `Parse Invitation Outcome`. Output: True branch (terminates run) / False branch (`Wait Between Invitations`).
  - **Version Requirements**: `typeVersion: 2.2`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

- **Wait Between Invitations**
  - **Type & Technical Role**: Wait node introducing configured delays between consecutive invitation dispatches.
  - **Configuration Choices**: Unit: `seconds`, Amount: `={{ $('Set Workflow Parameters').first().json.seconds_between_invitations }}`. Webhook ID configured.
  - **Key Expressions / Variables**: Evaluates `seconds_between_invitations` parameter.
  - **Input / Output Connections**: Input: `If Stop Inviting`. Output: `Batch Process Invitations`.
  - **Version Requirements**: `typeVersion: 1.1`
  - **Edge Cases & Failure Types**: Long wait durations depend on n8n execution persistence.
  - **Sub-workflow Reference**: None.

---

### 1.5 Sequence Audience Identification

#### Overview
This block retrieves active LinkedIn connections and filters them to identify individuals accepted within the specified sequence time window.

#### Nodes Involved
- `Check New Connections` (`n8n-nodes-nilyo.nilyo`)
- `Identify Sequence Participants` (`n8n-nodes-base.code`)
- `Batch Process Participants` (`n8n-nodes-base.splitInBatches`)

#### Node Details

- **Check New Connections**
  - **Type & Technical Role**: Community node invoking Nilyo to list authenticated user connections.
  - **Configuration Choices**: Resource: `tool`, Tool Name: `linkedin_list_my_connections`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({}) }}`.
  - **Key Expressions / Variables**: None.
  - **Input / Output Connections**: Input: `Set Workflow Parameters`. Output: `Identify Sequence Participants`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: API connectivity or authorization failures.
  - **Sub-workflow Reference**: None.

- **Identify Sequence Participants**
  - **Type & Technical Role**: JavaScript Code node filtering connections by acceptance date relative to the sequence window.
  - **Configuration Choices**: Computes acceptance timestamp limit using `sequence_window_days`, discards older connections, and extracts user IDs, display names, and first names.
  - **Key Expressions / Variables**: Accesses `Set Workflow Parameters` and input connection records.
  - **Input / Output Connections**: Input: `Check New Connections`. Output: `Batch Process Participants`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Connections lacking `user.id` or `created_at` timestamps are filtered out.
  - **Sub-workflow Reference**: None.

- **Batch Process Participants**
  - **Type & Technical Role**: Loop Controller node iterating through sequence participants one at a time.
  - **Configuration Choices**: Batch size set to 1 item per iteration, with reset option disabled.
  - **Key Expressions / Variables**: Iterates items from `Identify Sequence Participants`.
  - **Input / Output Connections**: Input: `Identify Sequence Participants`, `If Message to Send` (false branch), `Continue Existing Chat`, `Initiate New Chat Sequence`. Output: Loop body (`Locate Conversation Thread`) / Loop completion (empty).
  - **Version Requirements**: `typeVersion: 3`
  - **Edge Cases & Failure Types**: Empty input arrays complete the loop immediately.
  - **Sub-workflow Reference**: None.

---

### 1.6 Conversation Analysis

#### Overview
This block locates conversation threads for target participants and retrieves message histories to enable contextual message sequencing.

#### Nodes Involved
- `Locate Conversation Thread` (`n8n-nodes-nilyo.nilyo`)
- `Analyze Conversation Content` (`n8n-nodes-nilyo.nilyo`)

#### Node Details

- **Locate Conversation Thread**
  - **Type & Technical Role**: Community node calling Nilyo to find existing chat threads by user ID.
  - **Configuration Choices**: Resource: `tool`, Tool Name: `messaging_find_chat_by_user`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({ user_id: $json.user_id }) }}`. Error handling set to `continueRegularOutput` with `alwaysOutputData: true`.
  - **Key Expressions / Variables**: Evaluates `$json.user_id`.
  - **Input / Output Connections**: Input: `Batch Process Participants`. Output: `Analyze Conversation Content`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: Unreadable chats or lookup failures output error objects handled downstream.
  - **Sub-workflow Reference**: None.

- **Analyze Conversation Content**
  - **Type & Technical Role**: Community node listing messages within a located chat thread.
  - **Configuration Choices**: Resource: `messaging`, Operation: `listMessages`, Chat ID: `={{ $json.id }}`, Limit: 50. Error handling set to `continueRegularOutput` with `alwaysOutputData: true`.
  - **Key Expressions / Variables**: Evaluates `$json.id` from thread lookup.
  - **Input / Output Connections**: Input: `Locate Conversation Thread`. Output: `Determine Today's Message`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: Chat history retrieval errors pass error flags into the decision logic.
  - **Sub-workflow Reference**: None.

---

### 1.7 Message Decision Logic

#### Overview
This block evaluates conversation histories to determine whether sequences should be halted due to inbound replies, paused due to spacing rules, or advanced to the next scheduled message step.

#### Nodes Involved
- `Determine Today's Message` (`n8n-nodes-base.code`)
- `If Message to Send` (`n8n-nodes-base.if`)
- `If Sequence Dry Run` (`n8n-nodes-base.if`)
- `Verify Existing Conversation` (`n8n-nodes-base.if`)

#### Node Details

- **Determine Today's Message**
  - **Type & Technical Role**: JavaScript Code node executing comprehensive safety guards and message sequencing logic.
  - **Configuration Choices**: Validates read safety (fails closed on unreadable threads), checks for inbound prospect replies (halts sequence if prospect replied), calculates step indexes based on outbound message counts and invitation note status, evaluates spacing delays (`days_between_messages`), and verifies duplicate-prevention texts.
  - **Key Expressions / Variables**: Accesses parameters from `Set Workflow Parameters`, participant data, chat objects, and message arrays.
  - **Input / Output Connections**: Input: `Analyze Conversation Content`. Output: `If Message to Send`.
  - **Version Requirements**: `typeVersion: 2`
  - **Edge Cases & Failure Types**: Unreadable chat threads or missing acceptance dates return safety actions (`skip_unreadable`, `wait`) rather than sending messages.
  - **Sub-workflow Reference**: None.

- **If Message to Send**
  - **Type & Technical Role**: Conditional Branching node checking whether the calculated action requires sending a message.
  - **Configuration Choices**: Evaluates whether `$json.action` equals `send`.
  - **Key Expressions / Variables**: `={{ $json.action }}`
  - **Input / Output Connections**: Input: `Determine Today's Message`. Output: True branch (`If Sequence Dry Run`) / False branch (`Batch Process Participants` loop continuation).
  - **Version Requirements**: `typeVersion: 2.2`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

- **If Sequence Dry Run**
  - **Type & Technical Role**: Conditional Branching node respecting dry-run mode for message dispatches.
  - **Configuration Choices**: Evaluates whether dry-run mode or test execution is active.
  - **Key Expressions / Variables**: `={{ $('Set Workflow Parameters').first().json.dry_run || $execution.mode === 'test' }}`
  - **Input / Output Connections**: Input: `If Message to Send`. Output: True branch (`Verify Existing Conversation`) / False branch (`Batch Process Participants` loop continuation).
  - **Version Requirements**: `typeVersion: 2.2`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

- **Verify Existing Conversation**
  - **Type & Technical Role**: Conditional Branching node determining whether to reply to an existing chat thread or initiate a new one.
  - **Configuration Choices**: Evaluates whether `$json.chat_id` is not empty.
  - **Key Expressions / Variables**: `={{ $json.chat_id }}`
  - **Input / Output Connections**: Input: `If Sequence Dry Run`. Output: True branch (`Continue Existing Chat`) / False branch (`Initiate New Chat Sequence`).
  - **Version Requirements**: `typeVersion: 2.2`
  - **Edge Cases & Failure Types**: None.
  - **Sub-workflow Reference**: None.

---

### 1.8 Message Delivery

#### Overview
This block dispatches selected messages into existing chat threads or initiates new conversations via Nilyo before returning control to the participant processing loop.

#### Nodes Involved
- `Continue Existing Chat` (`n8n-nodes-nilyo.nilyo`)
- `Initiate New Chat Sequence` (`n8n-nodes-nilyo.nilyo`)

#### Node Details

- **Continue Existing Chat**
  - **Type & Technical Role**: Community node calling Nilyo to send a message within an existing LinkedIn chat thread.
  - **Configuration Choices**: Resource: `linkedin`, Operation: `sendMessage`, Chat ID: `={{ $json.chat_id }}`, Text: `={{ $json.text }}`. Error handling set to `continueRegularOutput`.
  - **Key Expressions / Variables**: Evaluates `$json.chat_id` and `$json.text`.
  - **Input / Output Connections**: Input: `Verify Existing Conversation`. Output: `Batch Process Participants`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: Network errors or API limits caught via regular output.
  - **Sub-workflow Reference**: None.

- **Initiate New Chat Sequence**
  - **Type & Technical Role**: Community node calling Nilyo to start a new conversation thread with a user.
  - **Configuration Choices**: Resource: `tool`, Tool Name: `linkedin_start_conversation`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({ linkedin_user_id: $json.user_id, text: $json.text }) }}`. Error handling set to `continueRegularOutput`.
  - **Key Expressions / Variables**: Evaluates `$json.user_id` and `$json.text`.
  - **Input / Output Connections**: Input: `Verify Existing Conversation`. Output: `Batch Process Participants`.
  - **Version Requirements**: `typeVersion: 1`
  - **Edge Cases & Failure Types**: API errors captured via regular output.
  - **Sub-workflow Reference**: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template description | n8n-nodes-base.stickyNote | Workflow documentation and setup guide | None | None | ## Invite LinkedIn search results and run a follow-up sequence with Nilyo<br><br>### Who is it for<br>Founders, recruiters and sales people who build a shortlist on LinkedIn Classic, Sales Navigator or Recruiter and want the invitations, the opening message and the follow-ups handled on their own account, without a scraper.<br><br>### How it works<br>One daily run does two things. **Invitations**: the shortlist comes from the source you choose, people with a pending invitation or already connected are removed, the batch is capped and sent one at a time with a pause. LinkedIn refusals are read precisely — already invited or a three-week cooldown skips that person, a quota stops the run until tomorrow.<br><br>**Sequence**: for every connection accepted inside the window it reads the conversation. One inbound message and the sequence stops for good. Otherwise the next message goes out once your last one is old enough. If the invitation carried a note, that note was the opening message and the sequence starts at follow-up 1.<br><br>### Setup steps<br>- Store a Nilyo personal token in a **Nilyo API** credential.<br>- Open **Set Workflow Parameters**, the only node you need to edit: source, daily cap, pause, the four messages and the spacing. `{first_name}` is replaced.<br>- Keep `dry_run` on and run the manual trigger once: you get the shortlist and the planned messages without sending anything. Turn it off and publish.<br>- Leave the Code nodes alone. They hold the guards that stop a message being sent twice; the sending rules live in the settings above.<br><br>### Requirements<br>A Nilyo account with LinkedIn connected. Sales Navigator or Recruiter only if you source from them.<br><br>### Customization<br>Empty a follow-up to shorten the sequence, change the spacing, or replace the search with your own list of user ids.<br><br>### Community node<br>This template uses the **Nilyo** community node (`n8n-nodes-nilyo`), verified by n8n. On n8n Cloud, enable Verified Community Nodes in the Admin Panel and add Nilyo from the canvas. On self-hosted n8n, install `n8n-nodes-nilyo` from Settings > Community nodes. |
| Start | n8n-nodes-base.stickyNote | Start and configuration overview note | None | None | ## Start and configure<br><br>The schedule runs it once a day; the manual trigger is there to try it. **Set Workflow Parameters** holds everything you edit — the source, the daily cap, the delays, the four messages and the dry run. Both branches read from it. |
| Find | n8n-nodes-base.stickyNote | Find invitation candidates overview note | None | None | ## 1. Find invitation candidates<br><br>Pending invitations are read first, so nobody is invited twice. The search call is built from the source you chose: a Classic, Sales Navigator or Recruiter URL, explicit filters, or a saved search. |
| Filter | n8n-nodes-base.stickyNote | Filter batch overview note | None | None | ## 2. Filter the batch<br><br>People already connected or already invited are dropped, the rest is capped at the daily limit, and the dry run stops here. One invitation at a time — never a parallel loop. |
| Send | n8n-nodes-base.stickyNote | Send invitation overview note | None | None | ## 3. Send one invitation<br><br>The note is rendered for this person; an empty note in the settings means an invitation without one. LinkedIn's refusals are read precisely rather than retried. |
| Throttle | n8n-nodes-base.stickyNote | Throttle and outcome handling overview note | None | None | ## 4. Skip, pause, or stop for the day<br><br>Already invited or in cooldown skips that person. A quota stops the whole run until tomorrow's schedule. Anything else pauses and continues, so a refusal is never retried. |
| Audience | n8n-nodes-base.stickyNote | Sequence audience identification overview note | None | None | ## 5. Who is in the sequence<br><br>Connections accepted inside the window. The acceptance date is what places someone in the sequence, so a connection from last year is not messaged today. |
| History | n8n-nodes-base.stickyNote | Conversation history analysis overview note | None | None | ## 6. Read the conversation first<br><br>Nothing is decided from a counter. The thread itself is read, and a conversation that cannot be read blocks that person for this run rather than being treated as empty. |
| Decide | n8n-nodes-base.stickyNote | Message decision logic overview note | None | None | ## 7. What is due today<br><br>The number of messages you sent is the step index. One inbound message ends the sequence for good, and a message whose text is already in the thread is never sent again. |
| Deliver | n8n-nodes-base.stickyNote | Message delivery overview note | None | None | ## 8. Reply, or open the thread<br><br>Into the existing conversation when there is one, otherwise a new thread. Then back to the loop: one message per person per run, whatever happens. |
| Every Day at 9:30 AM | n8n-nodes-base.scheduleTrigger | Daily schedule trigger at 09:30 AM | None | Set Workflow Parameters | ## Start and configure<br><br>The schedule runs it once a day; the manual trigger is there to try it. **Set Workflow Parameters** holds everything you edit — the source, the daily cap, the delays, the four messages and the dry run. Both branches read from it. |
| Manual Workflow Test | n8n-nodes-base.manualTrigger | Manual execution trigger for testing | None | Set Workflow Parameters | ## Start and configure<br><br>The schedule runs it once a day; the manual trigger is there to try it. **Set Workflow Parameters** holds everything you edit — the source, the daily cap, the delays, the four messages and the dry run. Both branches read from it. |
| Set Workflow Parameters | n8n-nodes-base.set | Defines workflow variables, search config, and message templates | Every Day at 9:30 AM, Manual Workflow Test | Check Pending Invitations, Check New Connections | ## Start and configure<br><br>The schedule runs it once a day; the manual trigger is there to try it. **Set Workflow Parameters** holds everything you edit — the source, the daily cap, the delays, the four messages and the dry run. Both branches read from it. |
| Check Pending Invitations | n8n-nodes-nilyo.nilyo | Retrieves active sent LinkedIn invitations | Set Workflow Parameters | Construct Search API Call | ## 1. Find invitation candidates<br><br>Pending invitations are read first, so nobody is invited twice. The search call is built from the source you chose: a Classic, Sales Navigator or Recruiter URL, explicit filters, or a saved search. |
| Construct Search API Call | n8n-nodes-base.code | Translates settings into Nilyo search tool arguments | Check Pending Invitations | Execute People Search | ## 1. Find invitation candidates<br><br>Pending invitations are read first, so nobody is invited twice. The search call is built from the source you chose: a Classic, Sales Navigator or Recruiter URL, explicit filters, or a saved search. |
| Execute People Search | n8n-nodes-nilyo.nilyo | Executes LinkedIn people search via Nilyo tool | Construct Search API Call | Shortlist Candidates to Invite | ## 1. Find invitation candidates<br><br>Pending invitations are read first, so nobody is invited twice. The search call is built from the source you chose: a Classic, Sales Navigator or Recruiter URL, explicit filters, or a saved search. |
| Shortlist Candidates to Invite | n8n-nodes-base.code | Filters search results and caps daily invitation volume | Execute People Search | If Dry Run | ## 2. Filter the batch<br><br>People already connected or already invited are dropped, the rest is capped at the daily limit, and the dry run stops here. One invitation at a time — never a parallel loop. |
| If Dry Run | n8n-nodes-base.if | Bypasses live sending when dry run mode is enabled | Shortlist Candidates to Invite | Batch Process Invitations | ## 2. Filter the batch<br><br>People already connected or already invited are dropped, the rest is capped at the daily limit, and the dry run stops here. One invitation at a time — never a parallel loop. |
| Batch Process Invitations | n8n-nodes-base.splitInBatches | Iterates invitation candidates sequentially | If Dry Run, Wait Between Invitations | Compose Invitation Note | ## 2. Filter the batch<br><br>People already connected or already invited are dropped, the rest is capped at the daily limit, and the dry run stops here. One invitation at a time — never a parallel loop. |
| Compose Invitation Note | n8n-nodes-base.code | Personalizes invitation note templates | Batch Process Invitations | Dispatch Invitation | ## 3. Send one invitation<br><br>The note is rendered for this person; an empty note in the settings means an invitation without one. LinkedIn's refusals are read precisely rather than retried. |
| Dispatch Invitation | n8n-nodes-nilyo.nilyo | Sends LinkedIn invitation via Nilyo API | Compose Invitation Note | Parse Invitation Outcome | ## 3. Send one invitation<br><br>The note is rendered for this person; an empty note in the settings means an invitation without one. LinkedIn's refusals are read precisely rather than retried. |
| Parse Invitation Outcome | n8n-nodes-base.code | Classifies invitation API response codes | Dispatch Invitation | If Stop Inviting | ## 4. Skip, pause, or stop for the day<br><br>Already invited or in cooldown skips that person. A quota stops the whole run until tomorrow's schedule. Anything else pauses and continues, so a refusal is never retried. |
| If Stop Inviting | n8n-nodes-base.if | Halts execution if quota or rate limits are hit | Parse Invitation Outcome | Wait Between Invitations | ## 4. Skip, pause, or stop for the day<br><br>Already invited or in cooldown skips that person. A quota stops the whole run until tomorrow's schedule. Anything else pauses and continues, so a refusal is never retried. |
| Wait Between Invitations | n8n-nodes-base.wait | Pauses between invitation dispatches | If Stop Inviting | Batch Process Invitations | ## 4. Skip, pause, or stop for the day<br><br>Already invited or in cooldown skips that person. A quota stops the whole run until tomorrow's schedule. Anything else pauses and continues, so a refusal is never retried. |
| Check New Connections | n8n-nodes-nilyo.nilyo | Lists authenticated LinkedIn connections | Set Workflow Parameters | Identify Sequence Participants | ## 5. Who is in the sequence<br><br>Connections accepted inside the window. The acceptance date is what places someone in the sequence, so a connection from last year is not messaged today. |
| Identify Sequence Participants | n8n-nodes-base.code | Filters connections within sequence time window | Check New Connections | Batch Process Participants | ## 5. Who is in the sequence<br><br>Connections accepted inside the window. The acceptance date is what places someone in the sequence, so a connection from last year is not messaged today. |
| Batch Process Participants | n8n-nodes-base.splitInBatches | Iterates sequence participants sequentially | Identify Sequence Participants, If Message to Send, If Sequence Dry Run, Continue Existing Chat, Initiate New Chat Sequence | Locate Conversation Thread | ## 5. Who is in the sequence<br><br>Connections accepted inside the window. The acceptance date is what places someone in the sequence, so a connection from last year is not messaged today. |
| Locate Conversation Thread | n8n-nodes-nilyo.nilyo | Finds existing chat thread by user ID | Batch Process Participants | Analyze Conversation Content | ## 6. Read the conversation first<br><br>Nothing is decided from a counter. The thread itself is read, and a conversation that cannot be read blocks that person for this run rather than being treated as empty. |
| Analyze Conversation Content | n8n-nodes-nilyo.nilyo | Lists messages within located chat thread | Locate Conversation Thread | Determine Today's Message | ## 6. Read the conversation first<br><br>Nothing is decided from a counter. The thread itself is read, and a conversation that cannot be read blocks that person for this run rather than being treated as empty. |
| Determine Today's Message | n8n-nodes-base.code | Evaluates replies, step index, and spacing rules | Analyze Conversation Content | If Message to Send | ## 7. What is due today<br><br>The number of messages you sent is the step index. One inbound message ends the sequence for good, and a message whose text is already in the thread is never sent again. |
| If Message to Send | n8n-nodes-base.if | Checks whether a message action is required | Determine Today's Message | If Sequence Dry Run, Batch Process Participants | ## 7. What is due today<br><br>The number of messages you sent is the step index. One inbound message ends the sequence for good, and a message whose text is already in the thread is never sent again. |
| If Sequence Dry Run | n8n-nodes-base.if | Bypasses message delivery during dry runs | If Message to Send | Verify Existing Conversation, Batch Process Participants | ## 7. What is due today<br><br>The number of messages you sent is the step index. One inbound message ends the sequence for good, and a message whose text is already in the thread is never sent again. |
| Verify Existing Conversation | n8n-nodes-base.if | Checks whether chat ID exists for direct reply | If Sequence Dry Run | Continue Existing Chat, Initiate New Chat Sequence | ## 8. Reply, or open the thread<br><br>Into the existing conversation when there is one, otherwise a new thread. Then back to the loop: one message per person per run, whatever happens. |
| Continue Existing Chat | n8n-nodes-nilyo.nilyo | Sends message in existing chat thread | Verify Existing Conversation | Batch Process Participants | ## 8. Reply, or open the thread<br><br>Into the existing conversation when there is one, otherwise a new thread. Then back to the loop: one message per person per run, whatever happens. |
| Initiate New Chat Sequence | n8n-nodes-nilyo.nilyo | Initiates new chat sequence with user | Verify Existing Conversation | Batch Process Participants | ## 8. Reply, or open the thread<br><br>Into the existing conversation when there is one, otherwise a new thread. Then back to the loop: one message per person per run, whatever happens. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Prerequisites & Community Nodes**:
   - Ensure `n8n-nodes-nilyo` is installed (via Settings > Community nodes on self-hosted instances, or enabled via Verified Community Nodes in the n8n Cloud Admin Panel).
   - Create a credential of type **Nilyo API** (`nilyoApi`) using your Nilyo personal token. Ensure your Nilyo account is connected to LinkedIn.

2. **Trigger Nodes Setup**:
   - Create a **Schedule Trigger** node (`Every Day at 9:30 AM`), configure rule interval for hour 9, minute 30.
   - Create a **Manual Trigger** node (`Manual Workflow Test`).

3. **Parameters Node Setup**:
   - Create an **Edit Fields (Set)** node (`Set Workflow Parameters`).
   - Add assignments for `dry_run` (boolean: `true`), `source` (string: `"search_url"`), `search_url` (string), `search_product` (string: `"classic"`), `search_tool` (string), `search_arguments` (string), `saved_search_tool` (string), `saved_search_id` (string), `invitations_per_day` (number: `25`), `seconds_between_invitations` (number: `90`), `invitation_note` (string), `opening_message` (string), `follow_up_1` (string), `follow_up_2` (string), `follow_up_3` (string), `days_between_messages` (number: `4`), and `sequence_window_days` (number: `30`).
   - Connect both triggers (`Every Day at 9:30 AM` and `Manual Workflow Test`) to `Set Workflow Parameters`.

4. **Invitation Pipeline Setup**:
   - **Check Pending Invitations**: Add a Nilyo node (`n8n-nodes-nilyo.nilyo`), set Resource to `linkedin`, Operation to `listInvitations`, Invitation Type to `sent`. Enable `continueRegularOutput` error handling and `alwaysOutputData`. Connect `Set Workflow Parameters` output to this node.
   - **Construct Search API Call**: Add a Code node (`n8n-nodes-base.code`), paste the JavaScript snippet reading `source` and constructing the Nilyo tool arguments payload. Connect `Check Pending Invitations` to this node.
   - **Execute People Search**: Add a Nilyo node, set Resource to `tool`, Tool Name to `={{ $json.tool }}`, Operation to `call`, Tool Arguments to `={{ $json.args }}`. Connect `Construct Search API Call` to this node.
   - **Shortlist Candidates to Invite**: Add a Code node, paste the filtering script removing pending/existing connections, deriving first names, and capping to `invitations_per_day`. Connect `Execute People Search` to this node.
   - **If Dry Run**: Add an If node, set condition to evaluate `={{ $('Set Workflow Parameters').first().json.dry_run || $execution.mode === 'test' }}` equals `true`. Connect `Shortlist Candidates to Invite` to this node. Connect the `false` output to nothing (terminates test run).
   - **Batch Process Invitations**: Add a Split in Batches node with batch size 1. Connect the `true` output of `If Dry Run` to this node.
   - **Compose Invitation Note**: Add a Code node rendering `{first_name}` and `{display_name}` placeholders. Connect loop output of `Batch Process Invitations` to this node.
   - **Dispatch Invitation**: Add a Nilyo node, set Resource to `linkedin`, Operation to `sendInvitation`, User ID to `={{ $json.user_id }}`, Message to `={{ $json.note }}`. Enable `continueRegularOutput`. Connect `Compose Invitation Note` to this node.
   - **Parse Invitation Outcome**: Add a Code node classifying outcomes into `sent`, `skipped`, `stop_for_today`, or `failed`. Connect `Dispatch Invitation` to this node.
   - **If Stop Inviting**: Add an If node evaluating `={{ $json.outcome }}` equals `stop_for_today`. Connect `Parse Invitation Outcome` to this node. Connect `true` output to nothing (stops daily run).
   - **Wait Between Invitations**: Add a Wait node, Unit: `seconds`, Amount: `={{ $('Set Workflow Parameters').first().json.seconds_between_invitations }}`. Connect `false` output of `If Stop Inviting` to this node. Connect output back to `Batch Process Invitations`.

5. **Sequence Pipeline Setup**:
   - **Check New Connections**: Add a Nilyo node, Resource: `tool`, Tool Name: `linkedin_list_my_connections`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({}) }}`. Connect `Set Workflow Parameters` output to this node.
   - **Identify Sequence Participants**: Add a Code node filtering connections by `sequence_window_days`. Connect `Check New Connections` to this node.
   - **Batch Process Participants**: Add a Split in Batches node with batch size 1. Connect `Identify Sequence Participants` to this node (and subsequent loop return paths).
   - **Locate Conversation Thread**: Add a Nilyo node, Resource: `tool`, Tool Name: `messaging_find_chat_by_user`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({ user_id: $json.user_id }) }}`. Enable `continueRegularOutput` and `alwaysOutputData`. Connect loop output of `Batch Process Participants` to this node.
   - **Analyze Conversation Content**: Add a Nilyo node, Resource: `messaging`, Operation: `listMessages`, Chat ID: `={{ $json.id }}`, Limit: 50. Enable `continueRegularOutput` and `alwaysOutputData`. Connect `Locate Conversation Thread` to this node.
   - **Determine Today's Message**: Add a Code node containing thread analysis, inbound reply checks, step indexing, and spacing validation logic. Connect `Analyze Conversation Content` to this node.
   - **If Message to Send**: Add an If node evaluating `={{ $json.action }}` equals `send`. Connect `Determine Today's Message` to this node. Connect `false` output back to `Batch Process Participants` (skips/waits).
   - **If Sequence Dry Run**: Add an If node evaluating dry run / test mode. Connect `true` output of `If Message to Send` to this node. Connect `false` output back to `Batch Process Participants`.
   - **Verify Existing Conversation**: Add an If node evaluating `={{ $json.chat_id }}` is not empty. Connect `true` output of `If Sequence Dry Run` to this node.
   - **Continue Existing Chat**: Add a Nilyo node, Resource: `linkedin`, Operation: `sendMessage`, Chat ID: `={{ $json.chat_id }}`, Text: `={{ $json.text }}`. Enable `continueRegularOutput`. Connect `true` output of `Verify Existing Conversation` to this node. Connect output back to `Batch Process Participants`.
   - **Initiate New Chat Sequence**: Add a Nilyo node, Resource: `tool`, Tool Name: `linkedin_start_conversation`, Operation: `call`, Tool Arguments: `={{ JSON.stringify({ linkedin_user_id: $json.user_id, text: $json.text }) }}`. Enable `continueRegularOutput`. Connect `false` output of `Verify Existing Conversation` to this node. Connect output back to `Batch Process Participants`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Node Requirement | Uses the verified community node `n8n-nodes-nilyo`. On n8n Cloud, enable Verified Community Nodes in Admin Panel. On self-hosted instances, install via Settings > Community nodes. |
| Operational Best Practice | Keep `dry_run` enabled (`true`) and run the workflow manually via the test trigger to review shortlisted invites and planned messages before activating the daily schedule. |