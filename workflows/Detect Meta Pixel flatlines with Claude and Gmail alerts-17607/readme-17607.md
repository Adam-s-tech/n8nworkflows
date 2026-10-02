Detect Meta Pixel flatlines with Claude and Gmail alerts

https://n8nworkflows.xyz/workflows/detect-meta-pixel-flatlines-with-claude-and-gmail-alerts-17607


# Detect Meta Pixel flatlines with Claude and Gmail alerts

### 1. Workflow Overview

This workflow automates the health monitoring of Meta (Facebook) ad account pixels by checking their inactivity levels through the Meta Marketing API, analyzing inactive pixels using Anthropic Claude to determine potential root causes, and generating a structured report delivered via Gmail. 

The workflow groups its processing logic into three primary functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution and establishes evaluation thresholds (flatline hours, warning windows, and handling of un-fired pixels).
- **1.2 Pixel Fetching & AI Analysis:** Queries the Meta Marketing API for pixel statuses, calculates inactivity durations, flags anomalies, and leverages Claude AI to generate concise structural diagnoses for inactive pixels.
- **1.3 Report Compilation & Delivery:** Formats the AI-generated insights into a readable flatline report and dispatches the alert via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the workflow execution and sets up operational parameters that dictate how pixel inactivity is measured and categorized.
- **Nodes Involved:** 
  - `Run manually`
  - `Set config: flatline thresholds`

- **Node Details:**
  - **Run manually**
    - *Type & Role:* `n8n-nodes-base.manualTrigger` — Initiates workflow execution on-demand.
    - *Configuration:* Default parameters.
    - *Input/Output:* Output connected to `Set config: flatline thresholds`.
    - *Failure Modes:* Manual execution limits automation; can be replaced with a schedule trigger for continuous monitoring.
  - **Set config: flatline thresholds**
    - *Type & Role:* `n8n-nodes-base.code` — Defines threshold variables (`FLATLINE_HOURS: 24`, `WARN_HOURS: 12`, `IGNORE_NEVER_FIRED: true`).
    - *Configuration:* JavaScript code block returning a configuration JSON object.
    - *Input/Output:* Input from `Run manually`; output connected to `Fetch pixels & last-fired time (Meta)`.
    - *Failure Modes:* Syntax errors in the custom JS block.

---

#### 2.2 Pixel Fetching & AI Analysis
- **Overview:** Pulls live pixel data from Meta, evaluates runtime conditions against defined thresholds, and queries Anthropic Claude to analyze root causes for inactive pixels.
- **Nodes Involved:**
  - `Fetch pixels & last-fired time (Meta)`
  - `Detect flatlined pixels`
  - `Build AI diagnosis prompt`
  - `Diagnose pixel flatlines with Claude AI`

- **Node Details:**
  - **Fetch pixels & last-fired time (Meta)**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Communicates with the Meta Graph API to list all ad pixels, their IDs, names, and last fired timestamps.
    - *Configuration:* GET request to `https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/adspixels`. Query parameters include `access_token`, `fields` (`id,name,last_fired_time`), and `limit` (`200`).
    - *Input/Output:* Input from `Set config: flatline thresholds`; output connected to `Detect flatlined pixels`.
    - *Failure Modes:* Authentication failure (invalid `META_ACCESS_TOKEN`), invalid Ad Account ID (`META_AD_ACCOUNT_ID`), or rate limiting by Meta.
  - **Detect flatlined pixels**
    - *Type & Role:* `n8n-nodes-base.code` — Processes raw Meta API responses, calculates hours of inactivity, assigns statuses (`ok`, `warn`, `flatline`, `never_fired`), and isolates alerts.
    - *Configuration:* JavaScript code block referencing thresholds from the configuration node.
    - *Input/Output:* Input from `Fetch pixels & last-fired time (Meta)`; output connected to `Build AI diagnosis prompt`.
    - *Failure Modes:* Unexpected data structures returned by the Meta API.
  - **Build AI diagnosis prompt**
    - *Type & Role:* `n8n-nodes-base.code` — Formats a structured prompt and system instructions for Claude, limiting payloads to the top 20 flatlined pixels to avoid truncation.
    - *Configuration:* JavaScript code defining the model (`claude-haiku-4-5`), max tokens (`3000`), system instructions, and user payload.
    - *Input/Output:* Input from `Detect flatlined pixels`; output connected to `Diagnose pixel flatlines with Claude AI`.
    - *Failure Modes:* Context construction failures if input arrays are empty or malformed.
  - **Diagnose pixel flatlines with Claude AI**
    - *Type & Role:* `n8n-nodes-base.httpRequest` — Sends the prompt payload to the Anthropic Messages API to receive structured troubleshooting insights.
    - *Configuration:* POST request to `https://api.anthropic.com/v1/messages`. Headers include `x-api-key` (`{{ $env.ANTHROPIC_API_KEY }}`), `anthropic-version` (`2023-06-01`), and `content-type` (`application/json`). Body is sourced from the preceding expression.
    - *Input/Output:* Input from `Build AI diagnosis prompt`; output connected to `Print pixel flatline report`.
    - *Failure Modes:* API timeouts, invalid Anthropic API keys, or JSON parsing exceptions if Claude returns unexpected formats.

---

#### 2.3 Report Compilation & Delivery
- **Overview:** Parses the AI diagnostic output, builds a human-readable summary report, and dispatches it via Gmail.
- **Nodes Involved:**
  - `Print pixel flatline report`
  - `Send a message`

- **Node Details:**
  - **Print pixel flatline report**
    - *Type & Role:* `n8n-nodes-base.code` — Extracts and sanitizes Claude's JSON response, combining context metrics and individual diagnoses into a text report.
    - *Configuration:* JavaScript code containing robust JSON regex cleanup and string array builders.
    - *Input/Output:* Input from `Diagnose pixel flatlines with Claude AI`; output connected to `Send a message`.
    - *Failure Modes:* AI response syntax errors falling back to default messaging.
  - **Send a message**
    - *Type & Role:* `n8n-nodes-base.gmail` — Dispatches the generated email containing the Meta Pixel Flatline Report.
    - *Configuration:* Message content mapped via expression `{{ $json.report }}`. Requires pre-configured Gmail OAuth2 credentials.
    - *Input/Output:* Input from `Print pixel flatline report`; terminal node.
    - *Failure Modes:* Expired Gmail OAuth2 credentials, network timeouts, or invalid recipient configurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run manually | n8n-nodes-base.manualTrigger | Initiates workflow | None | Set config: flatline thresholds | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert |
| Set config: flatline thresholds | n8n-nodes-base.code | Sets operational thresholds | Run manually | Fetch pixels & last-fired time (Meta) | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 1. Read pixels\nList every pixel and when it last fired. |
| Fetch pixels & last-fired time (Meta) | n8n-nodes-base.httpRequest | Fetches pixels from Meta Graph API | Set config: flatline thresholds | Detect flatlined pixels | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 1. Read pixels\nList every pixel and when it last fired. |
| Detect flatlined pixels | n8n-nodes-base.code | Analyzes inactivity against thresholds | Fetch pixels & last-fired time (Meta) | Build AI diagnosis prompt | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 2. Detect flatlines\nFlag pixels quiet past the threshold, then Claude diagnoses why. |
| Build AI diagnosis prompt | n8n-nodes-base.code | Prepares payloads for Claude | Detect flatlined pixels | Diagnose pixel flatlines with Claude AI | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to Check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 2. Detect flatlines\nFlag pixels quiet past the threshold, then Claude diagnoses why. |
| Diagnose pixel flatlines with Claude AI | n8n-nodes-base.httpRequest | Calls Anthropic API for diagnostics | Build AI diagnosis prompt | Print pixel flatline report | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 2. Detect flatlines\nFlag pixels quiet past the threshold, then Claude diagnoses why. |
| Print pixel flatline report | n8n-nodes-base.code | Parses AI output into text report | Diagnose pixel flatlines with Claude AI | Send a message | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 3. Report\nPrint the flatline report (swap for Slack/email). |
| Send a message | n8n-nodes-base.gmail | Emails the compiled report | Print pixel flatline report | None | Meta Pixel Flatline Detector: Catches Meta pixels that have silently stopped firing, before you lose days of conversion tracking. Alerts with an AI diagnosis of the likely cause.\n\n### How it works\n- Lists every pixel on your ad account and reads when each last fired, from the Meta Marketing API.\n- Flags any pixel that has not received an event for longer than your flatline threshold, or has never fired.\n- Claude explains the most likely cause (a site deploy removed the tag, consent blocking, a GTM misfire, low traffic) and the first thing to check.\n\n### Setup\n1. Add to your environment: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID, META_API_VERSION, ANTHROPIC_API_KEY.\n2. Run with the manual trigger, or attach a Schedule trigger to check every few hours.\n\n### Customization\nIn the Config node, tune FLATLINE_HOURS and WARN_HOURS to your traffic volume. Swap the report node for a Slack or email node to get paged the moment a pixel goes dark.\n\nBuilt by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert<br><br>## 3. Report\nPrint the flatline report (swap for Slack/email). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node 1:** Add a `Manual Trigger` node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Create Node 2:** Add a `Code` node (`n8n-nodes-base.code`) named `Set config: flatline thresholds`. Connect `Run manually`'s output to this node. Set the JS code to:
   ```javascript
   return [{ json: {
     FLATLINE_HOURS: 24,
     WARN_HOURS: 12,
     IGNORE_NEVER_FIRED: true,
   } }];
   ```
3. **Create Node 3:** Add an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Fetch pixels & last-fired time (Meta)`. Connect `Set config: flatline thresholds` to it.
   - Method: `GET`
   - URL: `=https://graph.facebook.com/{{ $env.META_API_VERSION || 'v21.0' }}/{{ $env.META_AD_ACCOUNT_ID }}/adspixels`
   - Query Parameters: Add parameters `access_token` (`={{ $env.META_ACCESS_TOKEN }}`), `fields` (`id,name,last_fired_time`), and `limit` (`200`).
4. **Create Node 4:** Add a `Code` node (`n8n-nodes-base.code`) named `Detect flatlined pixels`. Connect `Fetch pixels & last-fired time (Meta)` to it. Insert processing logic to calculate hours quiet and categorize statuses.
5. **Create Node 5:** Add a `Code` node (`n8n-nodes-base.code`) named `Build AI diagnosis prompt`. Connect `Detect flatlined pixels` to it. Populate the Claude system instructions, user constraints, and model payload (`claude-haiku-4-5`).
6. **Create Node 6:** Add an `HTTP Request` node (`n8n-nodes-base.httpRequest`) named `Diagnose pixel flatlines with Claude AI`. Connect `Build AI diagnosis prompt` to it.
   - Method: `POST`
   - URL: `https://api.anthropic.com/v1/messages`
   - Specify Body: `Using JSON`
   - JSON Body: `={{ $json.body }}`
   - Header Parameters: `x-api-key` (`={{ $env.ANTHROPIC_API_KEY }}`), `anthropic-version` (`2023-06-01`), and `content-type` (`application/json`).
7. **Create Node 7:** Add a `Code` node (`n8n-nodes-base.code`) named `Print pixel flatline report`. Connect `Diagnose pixel flatlines with Claude AI` to it to parse the markdown/JSON response from Claude and build the summary string.
8. **Create Node 8:** Add a `Gmail` node (`n8n-nodes-base.gmail`) named `Send a message`. Connect `Print pixel flatline report` to it.
   - Credentials: Configure valid Gmail OAuth2 credentials.
   - Message: `={{ $json.report }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Done-for-you automation & tracking services provided by nocode.expert | [nocode.expert website](https://nocode.expert) |