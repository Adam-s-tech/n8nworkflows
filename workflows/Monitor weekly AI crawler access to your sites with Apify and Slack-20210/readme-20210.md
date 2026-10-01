Monitor weekly AI crawler access to your sites with Apify and Slack

https://n8nworkflows.xyz/workflows/monitor-weekly-ai-crawler-access-to-your-sites-with-apify-and-slack-20210


# Monitor weekly AI crawler access to your sites with Apify and Slack

### 1. Workflow Overview

This workflow is an automated weekly monitoring system designed to track whether AI web crawlers (such as GPTBot, ClaudeBot, PerplexityBot, and Google-Extended) can access a specified list of websites. Targeted primarily at SEO professionals, content teams, and web publishers, it detects changes in robots.txt rules, `llms.txt` files, and `noai` page signals.

The execution logic is structured into three functional blocks:
- **1.1 Schedule & Input Initialization:** Triggers automatically every Monday morning and prepares the target website list alongside a persistent monitor identifier.
- **1.2 Execution & Evaluation:** Interacts with the Apify actor ecosystem to assess crawler accessibility, evaluating state changes against a historical baseline and filtering for relevant alterations.
- **1.3 Aggregation & Notification:** Consolidates all detected changes into a single, structured summary message and delivers it to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Input Initialization
- **Overview:** Initiates the workflow on a weekly cadence and injects the configuration data containing target domains and the historical comparison key.
- **Nodes Involved:** 
  - `Every Monday`
  - `Websites to watch`

- **Node Details:**
  - **Every Monday**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire every week on Monday at 08:00 AM.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Websites to watch`.
    - *Version-specific Requirements:* Version 1.2.
    - *Edge Cases/Failure Types:* Missed executions if the n8n instance is offline at the scheduled time.

  - **Websites to watch**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation / Assignment Node)
    - *Configuration Choices:* Assigns an array of domain strings to a `websites` property and a string identifier to `monitorName`.
    - *Key Expressions/Variables:* `={{ ['nytimes.com', 'theguardian.com', 'python.org'] }}`
    - *Input/Output Connections:* Input from `Every Monday`; output connects to `Check AI crawler access`.
    - *Version-specific Requirements:* Version 3.4.
    - *Edge Cases/Failure Types:* Malformed JSON arrays or empty domain lists.

---

#### 2.2 Execution & Evaluation
- **Overview:** Executes the remote Apify actor to check crawler accessibility rules and filters the output to retain only sites with first-time or modified states.
- **Nodes Involved:**
  - `Check AI crawler access`
  - `New or changed?`

- **Node Details:**
  - **Check AI crawler access**
    - *Type & Technical Role:* `@apify/n8n-nodes-apify.apify` (External Service Integration Node)
    - *Configuration Choices:* Executes the `tidytools/ai-crawler-access-checker` actor with 1024 MB memory allocation and a $0.50 maximum total charge constraint. Passes custom body parameters evaluating input websites and the monitor name.
    - *Key Expressions/Variables:* 
      ```json
      ={
        "websites": {{ JSON.stringify($json.websites) }},
        "monitorName": "{{ $json.monitorName }}",
        "outputOnlyChanges": true,
        "includeFixSnippets": false
      }
      ```
    - *Input/Output Connections:* Input from `Websites to watch`; output connects to `New or changed?`.
    - *Version-specific Requirements:* Version 1. Requires valid Apify API credentials.
    - *Edge Cases/Failure Types:* API rate limits, authentication failures, actor execution timeouts, or exceeding the maximum cost threshold ($0.50).

  - **New or changed?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Routing Node)
    - *Configuration Choices:* Evaluates whether the comparison status returned by the Apify actor indicates a new monitor entry or an updated access policy.
    - *Key Expressions/Variables:* `={{ ['first', 'changed'].includes($json.comparison?.status) }}`
    - *Input/Output Connections:* Input from `Check AI crawler access`; output connects to `Build Slack message`.
    - *Version-specific Requirements:* Version 2.2.
    - *Edge Cases/Failure Types:* Missing `comparison` object in the incoming payload causing evaluation errors.

---

#### 2.3 Aggregation & Notification
- **Overview:** Formats all filtered status changes into a single markdown-styled message and publishes it to the target Slack channel.
- **Nodes Involved:**
  - `Build Slack message`
  - `Send to Slack`

- **Node Details:**
  - **Build Slack message**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Transformation Node)
    - *Configuration Choices:* Iterates through all incoming items, parses changed bots, policy shifts, `llms.txt`, and `noai` directives, then compiles them into a unified notification payload.
    - *Key Expressions/Variables:* Utilizes custom JavaScript processing via `$input.all()`.
    - *Input/Output Connections:* Input from `New or changed?`; output connects to `Send to Slack`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* TypeError if expected properties (such as `comparison` or `blockedBots`) are undefined.

  - **Send to Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration Node)
    - *Configuration Choices:* Posts a message to a named channel (`#seo-alerts`) with workflow link inclusion disabled.
    - *Key Expressions/Variables:* `={{ $json.text }}`
    - *Input/Output Connections:* Input from `Build Slack message`; terminal node in the workflow.
    - *Version-specific Requirements:* Version 2.3. Requires valid Slack OAuth2 or Bot Token credentials with chat permissions.
    - *Edge Cases/Failure Types:* Invalid channel names, missing channel permissions, or Slack API downtime.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Every Monday** | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow weekly | None | Websites to watch | Weekly AI crawler access check with Slack alerts<br><br>**Who it's for:** SEO and content teams, publishers and agencies who need to know whether GPTBot, ClaudeBot, PerplexityBot, Google-Extended and other AI crawlers can read their websites (or clients' and competitors' sites), and when that changes.<br><br>### How it works<br>1. Every Monday, **AI Crawler Access Checker** (tidytools/ai-crawler-access-checker on Apify) reads robots.txt, llms.txt and the home page `noai` tags of your sites.<br>2. The monitor name compares each site with last week, so only new or changed sites come back: a bot that went from allowed to blocked, a policy change, a new llms.txt.<br>3. One Slack message lists them. A week without changes posts nothing.<br><br>### Setup<br>1. Add your Apify credential (API key or OAuth) to the Apify node.<br>2. Put your domains and a monitor name in **Websites to watch**.<br>3. Choose your Slack credential and channel in **Send to Slack**.<br><br>The first run saves a baseline and posts one line per site.<br><br>### Cost<br>$2 per 1,000 sites checked, unchanged sites $0.50 per 1,000 (Apify pay-per-event, Sept 2026). 20 sites weekly is about $0.04 a month. The Apify node has a $0.50 cost cap.<br><br>Uses the verified Apify community node (`@apify/n8n-nodes-apify`). |
| **Websites to watch** | `n8n-nodes-base.set` | Defines target domains and monitor ID | Every Monday | Check AI crawler access | ## 1. Your websites<br>Domains or URLs, plus a monitor name. Use a different monitor name for each list. |
| **Check AI crawler access** | `@apify/n8n-nodes-apify.apify` | Executes Apify actor to check crawler access | Websites to watch | New or changed? | ## 2. Check AI crawler access<br>Only new or changed sites are returned (`outputOnlyChanges`). Unchanged sites are still checked at the re-check price. |
| **New or changed?** | `n8n-nodes-base.if` | Filters for first-time checks or policy changes | Check AI crawler access | Build Slack message | ## 3. Alert on changes<br>One Slack message per run. Swap Slack for Email, Teams or Discord if you like. |
| **Build Slack message** | `n8n-nodes-base.code` | Formats change data into a unified message payload | New or changed? | Send to Slack | ## 3. Alert on changes<br>One Slack message per run. Swap Slack for Email, Teams or Discord if you like. |
| **Send to Slack** | `n8n-nodes-base.slack` | Delivers the alert to the configured Slack channel | Build Slack message | None | ## 3. Alert on changes<br>One Slack message per run. Swap Slack for Email, Teams or Discord if you like. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually reconstruct the workflow in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Every Monday`.
   - Set the interval rule to trigger weekly on Mondays at `08:00`.

2. **Create the Input Data Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Websites to watch`.
   - Add an assignment of type `Array` named `websites` with value: `={{ ['nytimes.com', 'theguardian.com', 'python.org'] }}`.
   - Add an assignment of type `String` named `monitorName` with value: `my-sites`.
   - Connect `Every Monday` to `Websites to watch`.

3. **Create the Apify Integration Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`).
   - Name it `Check AI crawler access`.
   - Configure credentials: Add or select your Apify API Token/OAuth credential.
   - Set `Resource` to `Actors` and `Operation` to `Run actor and get dataset`.
   - Select the actor `tidytools/ai-crawler-access-checker` (`fHmxS3tQMVb1EfQBu`).
   - Set `Memory` to `1024` MB and `Max Total Charge (USD)` to `0.5`.
   - Set the `Custom Body` parameter to evaluate expressions dynamically:
     ```json
     ={
       "websites": {{ JSON.stringify($json.websites) }},
       "monitorName": "{{ $json.monitorName }}",
       "outputOnlyChanges": true,
       "includeFixSnippets": false
     }
     ```
   - Connect `Websites to watch` to `Check AI crawler access`.

4. **Create the Condition Filter Node:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Name it `New or changed?`.
   - Set the condition rule using a JavaScript expression: `={{ ['first', 'changed'].includes($json.comparison?.status) }}`.
   - Connect `Check AI crawler access` to `New or changed?`.

5. **Create the Message Builder Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Build Slack message`.
   - Insert the following JavaScript logic into the code editor:
     ```javascript
     const fmt = (list) => (list && list.length ? list.join(', ') : 'none');
     const items = $input.all();
     const lines = items.map(({ json: s }) => {
       const c = s.comparison || {};
       if (c.status === 'first') {
         return `• *${s.site}*: now monitored. Policy: ${s.policy}. Blocked: ${fmt(s.blockedBots)}.`;
       }
       const parts = (c.changedBots || []).map((b) => `${b.bot} ${b.before} → ${b.after}`);
       if (c.policyChanged) parts.push(`policy ${c.policyChanged.before} → ${c.policyChanged.after}`);
       if (c.llmsTxtChanged) parts.push(`llms.txt ${s.llmsTxt && s.llmsTxt.found ? 'added' : 'removed'}`);
       if (c.noAiChanged) parts.push(`noai meta tag ${s.noAiDirective ? 'added' : 'removed'}`);
       if (c.contentSignalChanged) parts.push('Content-Signal changed');
       return `• *${s.site}*: ${parts.join('; ') || 'changed'}. Blocked now: ${fmt(s.blockedBots)}.`;
     });
     const changed = items.filter((i) => (i.json.comparison || {}).status === 'changed').length;
     const header = changed
       ? `:robot_face: AI crawler access changed on ${changed} site(s)`
       : ':robot_face: AI crawler monitor: first check saved';
     return [{ json: { text: `${header}\n${lines.join('\n')}`, sites: items.length, changed } }];
     ```
   - Connect the `true` output of `New or changed?` to `Build Slack message`.

6. **Create the Slack Notification Node:**
   - Add a **Slack** node (`n8n-nodes-base.slack`).
   - Name it `Send to Slack`.
   - Configure credentials: Add or select your Slack Bot/OAuth2 credential.
   - Set `Select` to `channel` and provide the target channel name (e.g., `#seo-alerts`).
   - Set the `Text` parameter to: `={{ $json.text }}`.
   - Under `Other Options`, ensure `Include Link To Workflow` is set to `false`.
   - Connect `Build Slack message` to `Send to Slack`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor Reference | Uses the verified community actor `tidytools/ai-crawler-access-checker` available on the [Apify Store](https://console.apify.com/actors/fHmxS3tQMVb1EfQBu/input). |
| Pricing Considerations | Pay-per-event pricing applies based on Apify's active schedule. The actor includes a cost cap configured at $0.50 per execution. |