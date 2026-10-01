Detect India VIX regime shifts with GPT-4.1, Google Sheets, and Slack alerts

https://n8nworkflows.xyz/workflows/detect-india-vix-regime-shifts-with-gpt-4-1--google-sheets--and-slack-alerts-20005


# Detect India VIX regime shifts with GPT-4.1, Google Sheets, and Slack alerts

### 1. Workflow Overview

This workflow automates the daily tracking of the India VIX volatility index, evaluates whether the market has transitioned into a new volatility regime, enriches shifts with AI-generated educational risk context, updates persistent storage, and broadcasts alerts via Slack. It functions as a risk management and monitoring tool for trading operations.

The workflow logic is categorized into six functional blocks:
- **1.1 Input Reception & Validation:** Triggers daily at market-close equivalent hours, pulls raw VIX market metrics over HTTP, and validates data integrity (numeric checks, non-negative bounds).
- **1.2 Data Normalization & State Retrieval:** Structures validated market inputs, queries Google Sheets to fetch historical regime states, and classifies current VIX levels into specific volatility buckets.
- **1.3 Regime Comparison & Gating:** Compares current classification against stored history, calculates percentage deltas, and routes execution conditionally based on whether a regime transition occurred.
- **1.4 AI Context Generation:** Invokes OpenAI’s language model to build structured, JSON-formatted educational risk parameters and normalizes the AI output payload.
- **1.5 State Persistence & Auditing:** Prepares updated state values, writes current parameters back to Google Sheets, and logs transition histories.
- **1.6 Alert Dispatch:** Formulates human-readable markdown messages and pushes structured notifications to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Initiates the automation daily, acquires remote metrics, and enforces strict validation checks to prevent malformed numerical inputs from corrupting downstream states.
- **Nodes Involved:** 
  - `Daily VIX Check`
  - `Fetch VIX Market Data`
  - `Validate VIX Market Data`
- **Node Details:**
  - **Daily VIX Check**
    - *Type & Technical Role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) / Time-based execution initiator.
    - *Configuration:* Interval rule set to execute daily at hour 15, minute 30 (`triggerAtHour: 15`, `triggerAtMinute: 30`).
    - *Inputs/Outputs:* Input: None (Trigger) | Output: Triggers `Fetch VIX Market Data`.
    - *Edge Cases:* Timezone configuration mismatch on the n8n host server can shift execution away from expected local market times.
  - **Fetch VIX Market Data**
    - *Type & Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`) / External API data fetcher.
    - *Configuration:* GET request to `https://httpbin.org/get?index=INDIA_VIX&value=24.7&timestamp=2026-08-26T15%3A30%3A00%2B05%3A30`, enforcing JSON response formatting.
    - *Inputs/Outputs:* Input: `Daily VIX Check` | Output: `Validate VIX Market Data`.
    - *Edge Cases:* Network timeouts, HTTP 4xx/5xx errors, or schema changes from the upstream data provider.
  - **Validate VIX Market Data**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / JavaScript validation filter.
    - *Configuration:* Evaluates payload using `Number.isFinite()` and checks for values $\ge 0$. Throws manual errors for invalid types or negative indices.
    - *Inputs/Outputs:* Input: `Fetch VIX Market Data` | Output: `Normalize VIX Market Data`.
    - *Edge Cases:* Missing payload nodes resulting in `undefined` parameters trapped by strict parsing rules.

#### 2.2 Data Normalization & State Retrieval
- **Overview:** Standardizes validated records into uniform formats, pulls previous session states from Google Sheets, and tags current metrics with qualitative risk thresholds.
- **Nodes Involved:**
  - `Normalize VIX Market Data`
  - `Read Previous VIX Regime State`
  - `Classify Current VIX Regime`
- **Node Details:**
  - **Normalize VIX Market Data**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Data normalizer.
    - *Configuration:* JavaScript parsing block mapping dynamic input fields to a strict schema (`index_name`, `vix_value`, `timestamp`). Fallbacks to `new.Date().toISOString()` if timestamps are absent.
    - *Inputs/Outputs:* Input: `Validate VIX Market Data` | Output: `Read Previous VIX Regime State`.
  - **Read Previous VIX Regime State**
    - *Type & Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`) / Database reader.
    - *Configuration:* Reads from document ID `1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY`, sheet name `CurrentState` (`gid=0`). `alwaysOutputData` enabled.
    - *Inputs/Outputs:* Input: `Normalize VIX Market Data` | Output: `Classify Current VIX Regime`.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Edge Cases:* Empty sheets or API quota exhaustion.
  - **Classify Current VIX Regime**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Threshold classifier.
    - *Configuration:* Conditional branching assigning strings based on `vix_value`: `< 15` (`LOW`), `< 20` (`NORMAL`), `< 30` (`ELEVATED`), else (`CRISIS`).
    - *Inputs/Outputs:* Input: `Read Previous VIX Regime State` | Output: `Compare VIX Regime State`.

#### 2.3 Regime Comparison & Gating
- **Overview:** Evaluates historical persistence against current conditions, computes shift percentage deltas, and isolates processing paths for state transitions.
- **Nodes Involved:**
  - `Compare VIX Regime State`
  - `VIX Regime Changed?`
  - `No Regime Change Log`
- **Node Details:**
  - **Compare VIX Regime State**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Analytical comparator.
    - *Configuration:* Computes percentage change relative to previous values and flags boolean changes (`regime_changed`) when current and previous states mismatch.
    - *Inputs/Outputs:* Input: `Classify Current VIX Regime` | Output: `VIX Regime Changed?`.
  - **VIX Regime Changed?**
    - *Type & Technical Role:* If (`n8n-nodes-base.if`) / Conditional router.
    - *Configuration:* Evaluates strict boolean condition: `{{ $json.regime_changed }}` equals `true`.
    - *Inputs/Outputs:* Input: `Compare VIX Regime State` | Outputs: True branch (`Generate AI Risk Context`), False branch (`No Regime Change Log`).
  - **No Regime Change Log**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Termination logger.
    - *Configuration:* Emits standard JSON log payload indicating static volatility conditions (`NO_REGIME_CHANGE`).
    - *Inputs/Outputs:* Input: `VIX Regime Changed?` (False) | Output: None (Terminal node).

#### 2.4 AI Context Generation
- **Overview:** Consults OpenAI to synthesize educational risk guidelines and position-sizing frameworks structured specifically around detected transition states.
- **Nodes Involved:**
  - `Generate AI Risk Context`
  - `Normalize AI Risk Context`
- **Node Details:**
  - **Generate AI Risk Context**
    - *Type & Technical Role:* OpenAI Chat Model Node (`@n8n/n8n-nodes-langchain.openAi`) / AI prompt executioner.
    - *Configuration:* Uses model `gpt-4.1-nano` with `jsonOutput: true`. Prompt injects index names, current VIX scores, previous/new regimes, and requested JSON schema structures (`regime_summary`, `risk_guidance`, `position_sizing_context`, `disclaimer`).
    - *Inputs/Outputs:* Input: `VIX Regime Changed?` (True) | Output: `Normalize AI Risk Context`.
    - *Credentials:* OpenAI API (Paid tier).
    - *Edge Cases:* Rate limits, API downtime, or output truncation producing invalid JSON schemas.
  - **Normalize AI Risk Context**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Parsing validator.
    - *Configuration:* Multi-tier fallback JavaScript parser checking for object, stringified JSON, or raw message content keys to ensure downstream nodes safely consume structured data.
    - *Inputs/Outputs:* Input: `Generate AI Risk Context` | Output: `Prepare Current VIX State`.

#### 2.5 State Persistence & Auditing
- **Overview:** Formulates state updates, upserts operational rows in Google Sheets, and writes historical transition audit logs.
- **Nodes Involved:**
  - `Prepare Current VIX State`
  - `Update Current VIX State`
  - `Log VIX Regime Transition`
- **Node Details:**
  - **Prepare Current VIX State**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Payload assembler.
    - *Configuration:* Assembles fixed `state_id` (`VIX-001`) alongside dynamic values.
    - *Inputs/Outputs:* Input: `Normalize AI Risk Context` | Output: `Update Current VIX State`.
  - **Update Current VIX State**
    - *Type & Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`) / Database writer.
    - *Configuration:* Operation: `appendOrUpdate`, Sheet: `CurrentState`, Matching column: `state_id`.
    - *Inputs/Outputs:* Input: `Prepare Current VIX State` | Output: `Log VIX Regime Transition`.
    - *Credentials:* Google Sheets OAuth2 API.
  - **Log VIX Regime Transition**
    - *Type & Technical Role:* Google Sheets (`n8n-nodes-base.googleSheets`) / Audit logger.
    - *Configuration:* Operation: `append`, Sheet: `RegimeHistory` (`gid=2104533653`), Document ID: `1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY`.
    - *Inputs/Outputs:* Input: `Update Current VIX State` | Output: `Prepare VIX Slack Alert`.
    - *Credentials:* Google Sheets OAuth2 API.

#### 2.6 Alert Dispatch
- **Overview:** Concatenates structural metrics and AI outputs into markdown templates before broadcasting messages to Slack.
- **Nodes Involved:**
  - `Prepare VIX Slack Alert`
  - `Send VIX Regime Alert`
- **Node Details:**
  - **Prepare VIX Slack Alert**
    - *Type & Technical Role:* Code (`n8n-nodes-base.code`) / Message formatter.
    - *Configuration:* Joins scalar properties, percentage changes, risk context summaries, and disclaimers into a line-break-delimited string property named `slack_message`.
    - *Inputs/Outputs:* Input: `Log VIX Regime Transition` | Output: `Send VIX Regime Alert`.
  - **Send VIX Regime Alert**
    - *Type & Technical Role:* Slack (`n8n-nodes-base.slack`) / Webhook message dispatcher.
    - *Configuration:* Posts message via channel selector targeting channel ID `C0B1LNY15GW` (`n8n-workflow-testing`).
    - *Inputs/Outputs:* Input: `Prepare VIX Slack Alert` | Output: None (Terminal node).
    - *Credentials:* Slack API.
    - *Edge Cases:* Bot token scope validation failures or archiving of target Slack channels.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily VIX Check | scheduleTrigger | Time-based execution trigger | None | Fetch VIX Market Data | Market Data Collection |
| Fetch VIX Market Data | httpRequest | External API data fetcher | Daily VIX Check | Validate VIX Market Data | Market Data Collection |
| Validate VIX Market Data | code | Numerical validation filter | Fetch VIX Market Data | Normalize VIX Market Data | Market Data Collection |
| Normalize VIX Market Data | code | Data normalizer & standardizer | Validate VIX Market Data | Read Previous VIX Regime State | VIX Data Preparation |
| Read Previous VIX Regime State | googleSheets | Database state reader | Normalize VIX Market Data | Classify Current VIX Regime | VIX Data Preparation |
| Classify Current VIX Regime | code | Threshold volatility classifier | Read Previous VIX Regime State | Compare VIX Regime State | VIX Data Preparation |
| Compare VIX Regime State | code | Analytical comparator & delta calculator | Classify Current VIX Regime | VIX Regime Changed? | Regime Transition Detection |
| VIX Regime Changed? | if | Conditional workflow branch router | Compare VIX Regime State | Generate AI Risk Context, No Regime Change Log | Regime Transition Detection |
| No Regime Change Log | code | Static state logger & terminator | VIX Regime Changed? | None | Regime Transition Detection |
| Generate AI Risk Context | openAi | AI context builder via LLM | VIX Regime Changed? | Normalize AI Risk Context | AI Risk Context Generation |
| Normalize AI Risk Context | code | AI payload verification & parser | Generate AI Risk Context | Prepare Current VIX State | AI Risk Context Generation |
| Prepare Current VIX State | code | Payload assembler for database upsert | Normalize AI Risk Context | Update Current VIX State | State Storage and Alerting |
| Update Current VIX State | googleSheets | Persistent state upsert | Prepare Current VIX State | Log VIX Regime Transition | State Storage and Alerting |
| Log VIX Regime Transition | googleSheets | Audit transition history writer | Update Current VIX State | Prepare VIX Slack Alert | State Storage and Alerting |
| Prepare VIX Slack Alert | code | Markdown message constructor | Log VIX Regime Transition | Send VIX Regime Alert | State Storage and Alerting |
| Send VIX Regime Alert | slack | Slack channel notification broadcaster | Prepare VIX Slack Alert | None | State Storage and Alerting |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Daily VIX Check`.
   - Set parameters to trigger daily at `15:30`.
2. **Add Market Data Fetching:**
   - Add an **HTTP Request** node named `Fetch VIX Market Data`. Connect `Daily VIX Check` output to it.
   - Set method to `GET`, URL to `https://httpbin.org/get?index=INDIA_VIX&value=24.7&timestamp=2026-08-26T15%3A30%3A00%2B05%3A30`.
3. **Setup Data Validation:**
   - Add a **Code** node named `Validate VIX Market Data`. Connect `Fetch VIX Market Data` output to it.
   - Insert JavaScript validation parsing logic to check for finite numbers and non-negative values.
4. **Normalize Market Feed:**
   - Add a **Code** node named `Normalize VIX Market Data`. Connect `Validate VIX Market Data` output.
   - Map parameters into standardized `index_name`, `vix_value`, and `timestamp` fields.
5. **Read Historical State:**
   - Add a **Google Sheets** node named `Read Previous VIX Regime State`. Connect `Normalize VIX Market Data`.
   - Configure document ID (`1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY`) and sheet name (`CurrentState`). Enable `alwaysOutputData`.
   - Link valid Google Sheets OAuth2 credentials.
6. **Classify Volatility Regime:**
   - Add a **Code** node named `Classify Current VIX Regime`. Connect `Read Previous VIX Regime State`.
   - Apply boundary criteria: `< 15` (`LOW`), `< 20` (`NORMAL`), `< 30` (`ELEVATED`), else (`CRISIS`).
7. **Compare States:**
   - Add a **Code** node named `Compare VIX Regime State`. Connect `Classify Current VIX Regime`.
   - Compute percentage shifts and boolean transition parameters.
8. **Add Conditional Branching:**
   - Add an **If** node named `VIX Regime Changed?`. Connect `Compare VIX Regime State`.
   - Set conditions to evaluate if `{{ $json.regime_changed }}` equals boolean `true`.
9. **Configure No-Change Terminal Path:**
   - Add a **Code** node named `No Regime Change Log` and connect it to the `false` output of `VIX Regime Changed?`.
10. **Configure AI Generation Path:**
    - Add an **OpenAI** node named `Generate AI Risk Context` connected to the `true` output of `VIX Regime Changed?`.
    - Select model `gpt-4.1-nano`, enable JSON output, and add system prompts injecting context metrics. Configure OpenAI API credentials.
    - Add a **Code** node named `Normalize AI Risk Context` to parse incoming AI message objects cleanly.
11. **Prepare and Write State Updates:**
    - Add a **Code** node named `Prepare Current VIX State` to define state structural templates (`state_id: VIX-001`).
    - Add a **Google Sheets** node named `Update Current VIX State`. Set operation to `appendOrUpdate`, matching columns to `state_id`, sheet to `CurrentState`.
    - Add a second **Google Sheets** node named `Log VIX Regime Transition`. Set operation to `append`, sheet to `RegimeHistory`.
12. **Construct and Send Alerts:**
    - Add a **Code** node named `Prepare VIX Slack Alert` to build formatted Markdown string templates (`slack_message`).
    - Add a **Slack** node named `Send VIX Regime Alert`. Target channel ID `C0B1LNY15GW` and provide valid Slack credentials. Connect the output of `Prepare VIX Slack Alert`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| VIX Regime Monitor Spreadsheet Reference | [VIX Regime Monitor Template](https://docs.google.com/spreadsheets/d/1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY/edit?usp=drivesdk) |
| Current State Sheet Range Reference | [CurrentState Sheet View](https://docs.google.com/spreadsheets/d/1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY/edit#gid=0) |
| Regime History Log Sheet Reference | [RegimeHistory Sheet View](https://docs.google.com/spreadsheets/d/1xp7iTKgJh78dGNks4NsZTikjTJnRkz-4tSVSkyBpfGY/edit#gid=2104533653) |