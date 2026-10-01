Audit missing error workflows using the n8n API and Slack

https://n8nworkflows.xyz/workflows/audit-missing-error-workflows-using-the-n8n-api-and-slack-19730


# Audit missing error workflows using the n8n API and Slack

### 1. Workflow Overview

This workflow automates the weekly auditing of a self-hosted n8n instance to ensure that all active, published workflows have an error handler workflow explicitly attached in their settings. Running every Monday morning, it queries the n8n public API, filters out irrelevant workflows (such as archived ones, error triggers, and specifically tagged exempt workflows), evaluates error coverage gaps, and reports the findings to a Slack channel.

The logic is structured into three functional blocks:
- **1.1 Trigger and Configuration:** Initiates the audit on a weekly schedule and defines global filter parameters (ignore tags and clean-state alert preferences).
- **1.2 Data Retrieval and Gap Analysis:** Queries the n8n API to fetch all active workflows, parses and filters the dataset via custom JavaScript, and compiles a structured gap report.
- **1.3 Conditional Notification:** Evaluates whether notification criteria are met and either dispatches an alert to Slack or terminates via a no-op endpoint.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger and Configuration
- **Overview:** Initializes the audit process on a schedule and injects user-defined configuration properties to control filtering and notification behavior.
- **Nodes Involved:** `Every Monday morning`, `Settings`

- **Node Details:**
  - **Every Monday morning**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
    - *Configuration Choices:* Configured to fire weekly on Mondays at 09:00 AM.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs: None (Entry point). Outputs: Connected to `Settings`.
    - *Version-specific Requirements:* Version 1.3.
    - *Edge Cases/Failure Types:* Missed executions if the n8n instance is offline at the scheduled time.

  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Parameters Node)
    - *Configuration Choices:* Defines two static assignment parameters: `ignoreTag` (string: `"no-error-workflow"`) and `alertWhenClean` (boolean: `false`).
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs: `Every Monday morning`. Outputs: Connected to `Get published workflows`.
    - *Version-specific Requirements:* Version 3.5.
    - *Edge Cases/Failure Types:* Typographical errors in variable names referenced downstream.

---

#### Block 1.2: Data Retrieval and Gap Analysis
- **Overview:** Queries the n8n public API for published workflows, processes the returned array to filter out non-target workflows, and computes coverage metrics.
- **Nodes Involved:** `Get published workflows`, `Find workflows with no error workflow`

- **Node Details:**
  - **Get published workflows**
    - *Type and Technical Role:* `n8n-nodes-base.n8n` (API Integration Node)
    - *Configuration Choices:* Interacts with the n8n public API, filtering explicitly for active workflows (`activeWorkflows: true`).
    - *Key Expressions/Variables:* Relies on authenticated instance credentials.
    - *Input/Output:* Inputs: `Settings`. Outputs: Connected to `Find workflows with no error workflow`.
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases/Failure Types:* Authentication failure (invalid API key), network timeout, or API rate limits.
    - *Sub-workflow Reference:* None.

  - **Find workflows with no error workflow**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Processing Node)
    - *Configuration Choices:* Executes custom JavaScript to iterate through the API response. Excludes archived workflows, error-handler workflows (containing `Error Trigger` nodes), and workflows tagged with the configured `ignoreTag`. Compiles remaining unprotected workflows into an alphabetized list and generates a report string and a boolean flag (`shouldPost`).
    - *Key Expressions/Variables:* `$('Settings').first().json`, `$input.all()`.
    - *Input/Output:* Inputs: `Get published workflows`. Outputs: Connected to `Anything worth saying?`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Unhandled data structure shifts in n8n API responses.

---

#### Block 1.3: Conditional Notification
- **Overview:** Assesses whether gaps were found or if a clean-state report is forced, routing the execution to either post an alert to Slack or complete quietly.
- **Nodes Involved:** `Anything worth saying?`, `Post the gaps to Slack`, `Everything is covered`

- **Node Details:**
  - **Anything worth saying?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router Node)
    - *Configuration Choices:* Evaluates the boolean expression `{{ $json.shouldPost }}`.
    - *Key Expressions/Variables:* `={{ $json.shouldPost }}`.
    - *Input/Output:* Inputs: `Find workflows with no error workflow`. Outputs: True branch connects to `Post the gaps to Slack`; False branch connects to `Everything is covered`.
    - *Version-specific Requirements:* Version 2.3.
    - *Edge Cases/Failure Types:* Evaluation failure if upstream JSON structure is modified.

  - **Post the gaps to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration Node)
    - *Configuration Choices:* Posts a formatted Markdown warning message containing the audit report to a designated Slack channel.
    - *Key Expressions/Variables:* `={{':warning: *n8n error workflow audit*\n\n' + $json.report}}`.
    - *Input/Output:* Inputs: `Anything worth saying?` (True branch). Outputs: None (Terminal node).
    - *Version-specific Requirements:* Version 2.7.
    - *Edge Cases/Failure Types:* Slack credential expiration, missing channel permissions, or API downtime.

  - **Everything is covered**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Placeholder / End Node)
    - *Configuration Choices:* Acts as a pass-through termination point when no gaps are discovered and clean-state alerts are disabled.
    - *Key Expressions/Variables:* None.
    - *Input/Output:* Inputs: `Anything worth saying?` (False branch). Outputs: None (Terminal node).
    - *Version-specific Requirements:* Version 1.
    - *Edge Cases/Failure Types:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Documentation and Setup Guide | None | None | ## Audit error workflow coverage in n8n and report the gaps to Slack<br><br>### Who's it for<br><br>Anyone running self-hosted n8n with more than a handful of published workflows. An error workflow in n8n is not instance-wide: every workflow has to name it individually in its own settings. That checkbox is easy to forget, so most instances quietly accumulate published workflows that fail with nobody watching. This template finds them.<br><br>### How it works<br><br>On a weekly schedule it reads every workflow from your own instance through the n8n public API, and keeps only the ones that can actually fail silently: published, not archived, and not an error handler themselves. Whatever is left with no error workflow attached goes on a list, and the list is posted to Slack. When it is empty nothing is sent, so a quiet channel means full coverage.<br><br>### How to set up<br><br>1. In n8n, go to **Settings > n8n API** and create an API key.<br>2. Add an n8n API credential in the **Get published workflows** node, using that key and your instance URL.<br>3. Connect Slack in the **Post the gaps to Slack** node and pick a channel.<br>4. Run it once with **Test workflow**, then publish it.<br><br>### Requirements<br><br>Self-hosted n8n with the public API enabled, and a Slack workspace.<br><br>### How to customize the workflow<br><br>The **Settings** node holds the two values worth changing. `ignoreTag` names a tag for workflows allowed to run without a handler, so they stop appearing. Set `alertWhenClean` to true to get the all-clear every week instead of silence. Move the schedule to daily if your instance changes often, and swap Slack for email or Telegram without touching anything upstream. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## Read the instance<br>Every Monday morning, pull the full list of workflows from the n8n public API. `Settings` holds the two values you are meant to change. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## Find the gaps<br>Keeps published, non-archived workflows that are not error handlers, and lists the ones with no error workflow attached. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Documentation Block | None | None | ## Say something only if it matters<br>Nothing is posted when every workflow is covered. Set `alertWhenClean` to true if you would rather get the all-clear as well. |
| **Every Monday morning** | `n8n-nodes-base.scheduleTrigger` | Schedule Trigger | None | Settings | |
| **Settings** | `n8n-nodes-base.set` | Configuration Assignment | Every Monday morning | Get published workflows | |
| **Get published workflows** | `n8n-nodes-base.n8n` | API Retrieval | Settings | Find workflows with no error workflow | |
| **Find workflows with no error workflow** | `n8n-nodes-base.code` | Data Filtering & Report Generation | Get published workflows | Anything worth saying? | |
| **Anything worth saying?** | `n8n-nodes-base.if` | Conditional Routing | Find workflows with no error workflow | Post the gaps to Slack, Everything is covered | |
| **Post the gaps to Slack** | `n8n-nodes-base.slack` | Notification Dispatcher | Anything worth saying? | None | |
| **Everything is covered** | `n8n-nodes-base.noOp` | Termination (No-Op) | Anything worth saying? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every Monday morning`.
   - Set the trigger rule interval to weekly on Mondays at 09:00.
2. **Create the Configuration Node:**
   - Add a **Set** node named `Settings`.
   - Add two string/boolean assignments:
     - Name: `ignoreTag`, Type: String, Value: `no-error-workflow`
     - Name: `alertWhenClean`, Type: Boolean, Value: `false`
   - Connect `Every Monday morning` to `Settings`.
3. **Create the API Retrieval Node:**
   - Add an **n8n** node named `Get published workflows`.
   - Set the Resource to `Workflow` and configure filters to active workflows only (`activeWorkflows: true`).
   - Configure credentials to use your self-hosted instance's n8n API key.
   - Connect `Settings` to `Get published workflows`.
4. **Create the Processing Node:**
   - Add a **Code** node named `Find workflows with no error workflow`.
   - Paste the following JavaScript into the code block:
     ```javascript
     const settings = $('Settings').first().json;
     const ignoreTag = String(settings.ignoreTag || '').trim().toLowerCase();
     const alertWhenClean = settings.alertWhenClean === true;

     const uncovered = [];
     let published = 0;

     for (const item of $input.all()) {
       const w = item.json;
       if (!w.active || w.isArchived) continue;

       const nodes = Array.isArray(w.nodes) ? w.nodes : [];
       if (nodes.some((n) => n.type === 'n8n-nodes-base.errorTrigger')) continue;

       const tags = (w.tags || []).map((t) => String(t.name || t).toLowerCase());
       if (ignoreTag && tags.includes(ignoreTag)) continue;

       published++;
       if (!(w.settings && w.settings.errorWorkflow)) {
         uncovered.push({ id: w.id, name: w.name, updatedAt: w.updatedAt || null });
       }
     }

     uncovered.sort((a, b) => String(a.name).localeCompare(String(b.name)));

     const covered = published - uncovered.length;
     const lines = uncovered.map((w) => '• ' + w.name);
     const report = uncovered.length
       ? `*${uncovered.length} of ${published} published workflows have no error workflow attached*\\n\\n` + lines.join('\\n')
       : `All ${published} published workflows have an error workflow attached.`;

     return [{
       json: {
         published,
         covered,
         uncoveredCount: uncovered.length,
         uncovered,
         report,
         shouldPost: uncovered.length > 0 || alertWhenClean,
       },
     }];
     ```
   - Connect `Get published workflows` to `Find workflows with no error workflow`.
5. **Create the Conditional Router:**
   - Add an **If** node named `Anything worth saying?`.
   - Set the condition left value to `={{ $json.shouldPost }}` with an operation type of `Boolean` and value set to `true`.
   - Connect `Find workflows with no error workflow` to `Anything worth saying?`.
6. **Create the Action and Termination Nodes:**
   - Add a **Slack** node named `Post the gaps to Slack`.
   - Configure the message text expression as: `={{':warning: *n8n error workflow audit*\n\n' + $json.report}}`.
   - Configure Slack OAuth2 credentials and select your target notification channel.
   - Connect the True output (index 0) of `Anything worth saying?` to `Post the gaps to Slack`.
   - Add a **No-Op** node named `Everything is covered`.
   - Connect the False output (index 1) of `Anything worth saying?` to `Everything is covered`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Self-hosted n8n instance requirement with public API access enabled. | [n8n Public API Documentation](https://docs.n8n.io/api/) |
| Slack integration prerequisites (Workspace access and channel posting permissions). | [n8n Slack Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/) |