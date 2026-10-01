Monitor KPI anomalies with Postgres, Slack, Gmail, GitHub and Claude

https://n8nworkflows.xyz/workflows/monitor-kpi-anomalies-with-postgres--slack--gmail--github-and-claude-19804


# Monitor KPI anomalies with Postgres, Slack, Gmail, GitHub and Claude

### 1. Workflow Overview

This workflow is an automated KPI Anomaly Watchdog designed to collect metric data from a PostgreSQL database, calculate robust baseline statistics, evaluate daily movements using median absolute deviations (MAD), and investigate anomalies via an OpenRouter (Claude) AI agent. Depending on the detected severity, it routes alerts to Slack, creates GitHub issues, dispatches approval emails via Gmail, processes alert feedback through secure webhooks, and compiles a weekly precision report to recommend threshold adjustments.

The workflow logic is categorized into four distinct functional blocks:

- **1.1 Metric Collection and Baselines:** Handles nightly data gathering, execution of individual metric SQL statements, observation storage, and baseline recomputations.
- **1.2 Detection, Investigation, and Routing:** Executes morning anomaly detection, applies suppression rules, invokes the AI agent for root-cause analysis, evaluates evidence, and routes alerts across channels based on severity.
- **1.3 Alert Disposition:** Provides a secure webhook endpoint to ingest external feedback on raised incidents, validating and recording dispositions back to the database.
- **1.4 Weekly Precision Report:** Generates a periodic performance assessment of triggered alerts, calculates precision metrics, and proposes threshold modifications.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Metric Collection and Baselines

#### Overview
This block executes nightly or on demand to fetch enabled metric configurations from the database, run their respective collection queries, store yesterday's observation records, and recompute statistical baselines using a 28-day median and median absolute deviation (MAD).

#### Nodes Involved
- `Collect On Demand`
- `Collect Nightly 01:00`
- `Load Metric Registry`
- `Run Metric Query`
- `Build Observation`
- `Store Observation`
- `Recompute Baselines`
- `Summarise Baseline Coverage`
- `Record Baseline Run`
- `Only Thin Coverage`
- `Warn Thin History`

#### Node Details

##### Collect On Demand
- **Type & Role:** `n8n-nodes-base.manualTrigger` — Entry point for manually executing the collection and baseline run.
- **Configuration:** Default parameters (no configuration required).
- **Key Expressions:** None.
- **Connections:** Output connects to `Load Metric Registry`.
- **Version Requirements:** v1
- **Edge Cases & Failure Types:** None (manual execution).

##### Collect Nightly 01:00
- **Type & Role:** `n8n-nodes-base.scheduleTrigger` — Entry point running automatically every day at 01:00 AM.
- **Configuration:** Configured with a daily schedule interval (`triggerAtHour: 1`, `triggerAtMinute: 0`).
- **Key Expressions:** None.
- **Connections:** Output connects to `Load Metric Registry`.
- **Version Requirements:** v1.3
- **Edge Cases & Failure Types:** Depends on n8n internal scheduler reliability.

##### Load Metric Registry
- **Type & Role:** `n8n-nodes-base.postgres` — Database reader node that loads all enabled metric definitions (`kpi_metrics`).
- **Configuration:** Executes an `executeQuery` operation to select metric keys, collection SQL, thresholds, and ownership details where `enabled = true`.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Collect On Demand`, `Collect Nightly 01:00`
  - Output: `Run Metric Query`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database connectivity drops, SQL syntax errors, or empty result sets if no metrics are enabled.

##### Run Metric Query
- **Type & Role:** `n8n-nodes-base.postgres` — Database runner node evaluating individual dynamic metric queries.
- **Configuration:** Executes custom SQL with error continuation enabled (`onError: continueRegularOutput`).
- **Key Expressions:** `={{ $json.metric_sql }}`
- **Connections:** 
  - Input: `Load Metric Registry`
  - Output: `Build Observation`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Query timeouts, database connection errors, or malformed metric SQL resulting in runtime error objects.

##### Build Observation
- **Type & Role:** `n8n-nodes-base.code` — Code execution node running JavaScript per item to parse metric query outputs and handle collection errors.
- **Configuration:** `runOnceForEachItem` mode.
- **Key Expressions:** Validates against `$('Load Metric Registry').item.json` and filters out non-numeric values or error states.
- **Connections:** 
  - Input: `Run Metric Query`
  - Output: `Store Observation`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Unexpected JSON payload shapes from target database tables.

##### Store Observation
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer node inserting recorded values into `kpi_observations`.
- **Configuration:** Performs an `executeQuery` insert with conflict handling (`on conflict (metric_key, observed_on) do update`).
- **Key Expressions:** Query replacement variables referencing `metric_key`, `value`, `unit`, and `collection_error`.
- **Connections:** 
  - Input: `Build Observation`
  - Output: `Recompute Baselines`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database constraint violations or write locks.

##### Recompute Baselines
- **Type & Role:** `n8n-nodes-base.postgres` — Database execution node recalculating 28-day medians, MAD, sample standard deviations, and day-of-week medians.
- **Configuration:** Executes once (`executeOnce: true`) with a complex PL/pgSQL aggregation query updating `kpi_baselines`.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Store Observation`
  - Output: `Summarise Baseline Coverage`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Heavy table scans on large observation history tables.

##### Summarise Baseline Coverage
- **Type & Role:** `n8n-nodes-base.code` — Code execution node evaluating baseline readiness against a minimum observation count (`MIN_OBS = 14`).
- **Configuration:** Evaluates raw input arrays to identify metrics with insufficient data.
- **Key Expressions:** Reads all items from `$input.all()`.
- **Connections:** 
  - Input: `Recompute Baselines`
  - Output: `Record Baseline Run`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Empty dataset inputs.

##### Record Baseline Run
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer recording baseline execution metadata into `kpi_baseline_runs`.
- **Configuration:** Executes an insert query using parameter replacements.
- **Key Expressions:** Query replacements mapping total metrics, ready metrics, thin counts, and stringified thin metric lists.
- **Connections:** 
  - Input: `Summarise Baseline Coverage`
  - Output: `Only Thin Coverage`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** JSON serialization mismatches.

##### Only Thin Coverage
- **Type & Role:** `n8n-nodes-base.filter` — Logic filter routing execution only if metrics lack sufficient baseline history.
- **Configuration:** Evaluates if `thin_count > 0`.
- **Key Expressions:** `={{ $('Summarise Baseline Coverage').item.json.thin_count }}`
- **Connections:** 
  - Input: `Record Baseline Run`
  - Output: `Warn Thin History`
- **Version Requirements:** v2.2
- **Edge Cases & Failure Types:** Boolean evaluation mismatches.

##### Warn Thin History
- **Type & Role:** `n8n-nodes-base.slack` — Slack integration node posting warnings regarding metrics with incomplete observation history.
- **Configuration:** Uses Slack access token authentication to send text messages to a designated channel.
- **Key Expressions:** Evaluates baseline summaries to construct dynamic warning strings listing immature metrics.
- **Connections:** 
  - Input: `Only Thin Coverage`
  - Output: None (Terminal node for this branch)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Slack authentication revocation or missing channel configurations.

---

### Block 1.2: Detection, Investigation, and Routing

#### Overview
Runs every morning at 07:00 AM to fetch yesterday's KPI observations joined with baselines, calculate robust z-scores, filter anomalies, suppress duplicate alerts, run an AI investigation agent, validate evidence constraints, and distribute alerts via Slack, GitHub issues, and Gmail.

#### Nodes Involved
- `Detect Every Morning 07:00`
- `Detection Settings`
- `Fetch Latest Vs Baseline`
- `Score Anomalies`
- `Keep Only Anomalies`
- `Fetch Open Incidents`
- `Suppress Duplicates`
- `New Incident`
- `Log Suppressed Alert`
- `Open Incident Record`
- `Investigator Model`
- `Investigation Schema`
- `Get Metric Definition`
- `Get Metric History`
- `Get Segment Breakdown`
- `Get Related Metric Moves`
- `List Recent Deployments`
- `Tool Think Out Loud`
- `Investigate Anomaly`
- `Check Evidence`
- `Record Investigation Failure`
- `Explanation Trusted`
- `Label Explained`
- `Label Unexplained`
- `Merge Explanation Paths`
- `Update Incident Findings`
- `Route By Severity`
- `Alert Critical In Slack`
- `File Investigation Issue`
- `Request Owner Acknowledgement`
- `Record Acknowledgement`
- `Alert Warning In Slack`
- `Record Warning Sent`
- `Record Info Only`

#### Node Details

##### Detect Every Morning 07:00
- **Type & Role:** `n8n-nodes-base.scheduleTrigger` — Scheduled trigger firing daily at 07:00 AM.
- **Configuration:** Interval set to 1 day at hour 7, minute 0.
- **Key Expressions:** None.
- **Connections:** Output connects to `Detection Settings`.
- **Version Requirements:** v1.3
- **Edge Cases & Failure Types:** Scheduler execution skips.

##### Detection Settings
- **Type & Role:** `n8n-nodes-base.set` — Parameter setup node defining global operational constants.
- **Configuration:** Manual assignment mode setting critical channels, warning channels, on-call emails, incident caps, cooldown hours, and minimum observation thresholds.
- **Key Expressions:** Static assignments.
- **Connections:** 
  - Input: `Detect Every Morning 07:00`
  - Output: `Fetch Latest Vs Baseline`
- **Version Requirements:** v3.4
- **Edge Cases & Failure Types:** Missing environment configurations.

##### Fetch Latest Vs Baseline
- **Type & Role:** `n8n-nodes-base.postgres` — Database reader joining observations, metrics, and baseline tables for yesterday's data.
- **Configuration:** Executes custom query joining `kpi_observations`, `kpi_metrics`, and `kpi_baselines`.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Detection Settings`
  - Output: `Score Anomalies`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Missing data entries for specific metrics on the target date.

##### Score Anomalies
- **Type & Role:** `n8n-nodes-base.code` — Code execution node calculating robust z-scores using median absolute deviation (MAD) or standard deviation.
- **Configuration:** JavaScript evaluation block applying day-of-week median seasonality checks and severity classifications (`critical`, `warning`, `info`).
- **Key Expressions:** References `$('Detection Settings').first().json`.
- **Connections:** 
  - Input: `Fetch Latest Vs Baseline`
  - Output: `Keep Only Anomalies`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Division by zero on unscaled flat series (handled via fallbacks).

##### Keep Only Anomalies
- **Type & Role:** `n8n-nodes-base.filter` — Filter node dropping non-anomalous metric records.
- **Configuration:** Condition verifying `is_anomaly === true`.
- **Key Expressions:** `={{ $json.is_anomaly }}`
- **Connections:** 
  - Input: `Score Anomalies`
  - Output: `Fetch Open Incidents`
- **Version Requirements:** v2.2
- **Edge Cases & Failure Types:** Strict type mismatch handling.

##### Fetch Open Incidents
- **Type & Role:** `n8n-nodes-base.postgres` — Database reader querying unresolved incidents from the last 7 days.
- **Configuration:** `executeOnce: true` with `alwaysOutputData: true` ensuring downstream deduplication nodes always receive input.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Keep Only Anomalies`
  - Output: `Suppress Duplicates`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database timeouts.

##### Suppress Duplicates
- **Type & Role:** `n8n-nodes-base.code` — Code execution node suppressing redundant alerts based on cooldown windows and maximum per-run incident caps.
- **Configuration:** JavaScript sorting anomalies by absolute z-score and checking active cooldown states.
- **Key Expressions:** References settings and upstream anomaly/incident lists.
- **Connections:** 
  - Input: `Fetch Open Incidents`
  - Output: `New Incident`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Invalid date parsing on incident timestamps.

##### New Incident
- **Type & Role:** `n8n-nodes-base.if` — Conditional router checking if an anomaly was suppressed.
- **Configuration:** Condition checks `suppressed === false`.
- **Key Expressions:** `={{ $json.suppressed }}`
- **Connections:** 
  - Input: `Suppress Duplicates`
  - Output: True branch to `Open Incident Record`, False branch to `Log Suppressed Alert`
- **Version Requirements:** v2.3
- **Edge Cases & Failure Types:** Type validation errors.

##### Log Suppressed Alert
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer recording suppressed alerts to `kpi_suppressed_alerts`.
- **Configuration:** Executes parameter-replaced insertion query.
- **Key Expressions:** Query replacements mapping metric key, observation date, z-score, severity, and suppression reason.
- **Connections:** 
  - Input: `New Incident` (False)
  - Output: None (Terminal node)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Constraint violations.

##### Open Incident Record
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer creating an active incident record in `kpi_incidents`.
- **Configuration:** Executes an insert returning generated incident IDs.
- **Key Expressions:** Query replacements binding metric key, observation values, z-scores, and severity.
- **Connections:** 
  - Input: `New Incident` (True)
  - Output: `Investigate Anomaly`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database connection drops.

##### Investigator Model
- **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — Chat model provider node configuring the LLM backend for root-cause investigation.
- **Configuration:** Uses OpenRouter with `anthropic/claude-opus-5`, zero temperature, and max 2 retries.
- **Key Expressions:** None.
- **Connections:** Output links to AI Language Model input of `Investigate Anomaly`.
- **Version Requirements:** v1
- **Edge Cases & Failure Types:** API rate limits, upstream service outages, or invalid API keys.

##### Investigation Schema
- **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Structured output parser ensuring the AI agent returns compliant JSON schemas.
- **Configuration:** Manual JSON schema enforcement requiring primary hypotheses, hypothesis classifications, evidence arrays, confidence scores, and false positive flags.
- **Key Expressions:** None.
- **Connections:** Output links to AI Output Parser input of `Investigate Anomaly`.
- **Version Requirements:** v1.3
- **Edge Cases & Failure Types:** LLM output format drifts failing JSON schema validation.

##### Get Metric Definition (Tool)
- **Type & Role:** `n8n-nodes-base.postgresTool` — AI tool node allowing the investigator agent to look up metric specifications.
- **Configuration:** Executes parameterised SQL querying `kpi_metrics`.
- **Key Expressions:** `={{ $fromAI('metric_key', 'The exact metric_key to look up', 'string') }}`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Missing or invalid tool input parameters from the model.

##### Get Metric History (Tool)
- **Type & Role:** `n8n-nodes-base.postgresTool` — AI tool node returning the last 60 daily values for a given metric.
- **Configuration:** Parameterised SQL selecting recent observations.
- **Key Expressions:** `={{ $fromAI('metric_key', 'The exact metric_key to pull history for', 'string') }}`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database timeouts.

##### Get Segment Breakdown (Tool)
- **Type & Role:** `n8n-nodes-base.postgresTool` — AI tool node returning dimension and segment delta values on specific dates.
- **Configuration:** Parameterised SQL querying `kpi_segment_observations`.
- **Key Expressions:** `={{ $fromAI('metric_key', ...) }}`, `={{ $fromAI('observed_on', ...) }}`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Missing segment data tables.

##### Get Related Metric Moves (Tool)
- **Type & Role:** `n8n-nodes-base.postgresTool` — AI tool node listing other metrics exhibiting movements exceeding 3 MADs on the same date.
- **Configuration:** Parameterised SQL querying concurrent anomalies.
- **Key Expressions:** `={{ $fromAI('observed_on', ...) }}`, `={{ $fromAI('exclude_metric_key', ...) }}`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Empty result sets.

##### List Recent Deployments (Tool)
- **Type & Role:** `n8n-nodes-base.githubTool` — GitHub integration tool node listing merged pull requests for the repository owning the metric's pipeline.
- **Configuration:** GitHub OAuth2/Access Token authentication querying repository pull requests with error continuation enabled.
- **Key Expressions:** `={{ $fromAI('repo_owner', ...) }}`, `={{ $fromAI('repo_name', ...) }}`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v1.1
- **Edge Cases & Failure Types:** Invalid repository names or GitHub API rate limits.

##### Think Out Loud (Tool)
- **Type & Role:** `@n8n/n8n-nodes-langchain.toolCode` — Scratchpad tool node enabling the agent to reason step-by-step.
- **Configuration:** JavaScript tool execution returning input query strings.
- **Key Expressions:** `return query;`
- **Connections:** Output links to AI Tool input of `Investigate Anomaly`.
- **Version Requirements:** v1.3
- **Edge Cases & Failure Types:** None.

##### Investigate Anomaly
- **Type & Role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent orchestrating tools and prompting the investigator model.
- **Configuration:** Defines system messages, max iterations (12), non-streaming execution, and error continuation (`onError: continueErrorOutput`).
- **Key Expressions:** Dynamic prompt inputs referencing upstream anomaly context, metric descriptions, values, and baselines.
- **Connections:** 
  - Input: `Open Incident Record`, `Investigator Model`, `Investigation Schema`, and all AI tools.
  - Output: True branch (success) to `Check Evidence`, False branch (error) to `Record Investigation Failure`.
- **Version Requirements:** v3.1
- **Edge Cases & Failure Types:** Agent execution timeouts, hallucinated tool arguments, or iteration limits reached.

##### Check Evidence
- **Type & Role:** `n8n-nodes-base.code` — Code node validating agent output against known metrics and evidence constraints.
- **Configuration:** JavaScript node verifying evidence count, counter-evidence presence, and fabricated metric keys.
- **Key Expressions:** References upstream agent outputs, open incidents, and baseline registries.
- **Connections:** 
  - Input: `Investigate Anomaly` (Success path)
  - Output: `Explanation Trusted`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Malformed agent outputs.

##### Record Investigation Failure
- **Type & Role:** `n8n-nodes-base.code` — Fallback code node handling agent execution errors without dropping the alert.
- **Configuration:** Sets empty hypotheses, unknown classes, and flags investigation failures.
- **Key Expressions:** Captures `$json.error.message`.
- **Connections:** 
  - Input: `Investigate Anomaly` (Error path)
  - Output: `Label Unexplained`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** None.

##### Explanation Trusted
- **Type & Role:** `n8n-nodes-base.if` — Conditional router verifying whether the investigator's explanation passes trust criteria.
- **Configuration:** Evaluates multiple conditions: evidence count `>= 2`, confidence `>= 0.6`, zero fabricated keys, non-empty hypothesis, and hypothesis class `!== 'unknown'`.
- **Key Expressions:** Evaluates evidence counts, confidence metrics, and fabrication flags.
- **Connections:** 
  - Input: `Check Evidence`
  - Output: True branch to `Label Explained`, False branch to `Label Unexplained`
- **Version Requirements:** v2.3
- **Edge Cases & Failure Types:** Strict boolean logic errors.

##### Label Explained
- **Type & Role:** `n8n-nodes-base.set` — Parameter setting node marking the explanation status as trusted.
- **Configuration:** Sets `explanation_status` to `'explained'` and formats descriptive success notes.
- **Key Expressions:** Dynamic assignment referencing evidence counts and confidence scores.
- **Connections:** 
  - Input: `Explanation Trusted` (True)
  - Output: `Merge Explanation Paths`
- **Version Requirements:** v3.4
- **Edge Cases & Failure Types:** None.

##### Label Unexplained
- **Type & Role:** `n8n-nodes-base.set` — Parameter setting node marking the explanation status as untrusted or discarded.
- **Configuration:** Sets `explanation_status` to `'unexplained'` and clears unreliable primary hypotheses.
- **Key Expressions:** Formats descriptive warning notes explaining why the explanation was rejected.
- **Connections:** 
  - Input: `Explanation Trusted` (False), `Record Investigation Failure`
  - Output: `Merge Explanation Paths`
- **Version Requirements:** v3.4
- **Edge Cases & Failure Types:** None.

##### Merge Explanation Paths
- **Type & Role:** `n8n-nodes-base.merge` — Append merge node consolidating explained and unexplained branches.
- **Configuration:** Append mode combining two input streams.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Label Explained`, `Label Unexplained`
  - Output: `Update Incident Findings`
- **Version Requirements:** v3.2
- **Edge Cases & Failure Types:** Asynchronous race conditions.

##### Update Incident Findings
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer updating `kpi_incidents` with investigation outcomes, evidence JSON, confidence scores, and hypotheses.
- **Configuration:** Executes custom update query returning updated incident states.
- **Key Expressions:** Query replacements binding incident IDs, explanation status, hypothesis classes, evidence arrays, and confidence levels.
- **Connections:** 
  - Input: `Merge Explanation Paths`
  - Output: `Route By Severity`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database write locks.

##### Route By Severity
- **Type & Role:** `n8n-nodes-base.switch` — Switch routing node directing alerts based on incident severity.
- **Configuration:** Rule-based routing for `critical` and `warning`, with an explicit fallback for `info` severity levels.
- **KeyExpressions:** `={{ $('Merge Explanation Paths').item.json.severity }}`
- **Connections:** 
  - Input: `Update Incident Findings`
  - Output: Critical branch to `Alert Critical In Slack`, Warning branch to `Alert Warning In Slack`, Fallback branch to `Record Info Only`
- **Version Requirements:** v3.4
- **Edge Cases & Failure Types:** Unexpected severity string values.

##### Alert Critical In Slack
- **Type & Role:** `n8n-nodes-base.slack` — Slack alert node posting critical anomaly notifications with investigation summaries and evidence breakdowns.
- **Configuration:** Uses Slack access token authentication with markdown formatting enabled (`onError: continueRegularOutput`).
- **Key Expressions:** Dynamically formats metric names, observed values, baselines, z-scores, evidence lists, and recommended next steps.
- **Connections:** 
  - Input: `Route By Severity` (Critical)
  - Output: `File Investigation Issue`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Slack rate limits or payload length limits.

##### File Investigation Issue
- **Type & Role:** `n8n-nodes-base.github` — GitHub integration node creating issues in target repositories for critical anomalies.
- **Configuration:** GitHub access token authentication creating issues in specified owner/repository targets with the `kpi-anomaly` label (`onError: continueRegularOutput`).
- **Key Expressions:** Formats markdown issue bodies containing incident metadata, hypotheses, evidence lists, and alternative explanations.
- **Connections:** 
  - Input: `Alert Critical In Slack`
  - Output: `Request Owner Acknowledgement`
- **Version Requirements:** v1.1
- **Edge Cases & Failure Types:** GitHub API validation errors or missing repository credentials.

##### Request Owner Acknowledgement
- **Type & Role:** `n8n-nodes-base.gmail` — Gmail integration node sending approval emails to metric owners with interactive response buttons.
- **Configuration:** Gmail OAuth2 authentication configured with a `sendAndWait` operation requesting approval with 8-hour wait limits (`onError: continueRegularOutput`).
- **Key Expressions:** `={{ $('Merge Explanation Paths').item.json.owner_email || $('Detection Settings').first().json.oncall_email }}`
- **Connections:** 
  - Input: `File Investigation Issue`
  - Output: `Record Acknowledgement`
- **Version Requirements:** v2.2
- **Edge Cases & Failure Types:** Email delivery failures or approval timeouts.

##### Record Acknowledgement
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer updating incident acknowledgement states based on Gmail approval responses.
- **Configuration:** Executes status update query (`acknowledged` vs `open`).
- **Key Expressions:** Query replacements evaluating approval responses (`$json.data.approved`).
- **Connections:** 
  - Input: `Request Owner Acknowledgement`
  - Output: None (Terminal node)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Missing approval payloads.

##### Alert Warning In Slack
- **Type & Role:** `n8n-nodes-base.slack` — Slack alert node posting warning-level anomaly notifications.
- **Configuration:** Slack access token authentication posting to the warning channel (`onError: continueRegularOutput`).
- **Key Expressions:** Formats warning notifications with z-scores and next steps.
- **Connections:** 
  - Input: `Route By Severity` (Warning)
  - Output: `Record Warning Sent`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Slack transmission errors.

##### Record Warning Sent
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer marking warning notifications as sent in `kpi_incidents`.
- **Configuration:** Executes update query setting `notified_at` and `notified_channel`.
- **Key Expressions:** References incident IDs.
- **Connections:** 
  - Input: `Alert Warning In Slack`
  - Output: None (Terminal node)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database connectivity drops.

##### Record Info Only
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer logging low-severity info anomalies without external messaging.
- **Configuration:** Executes update query setting status to `'logged'` and recording notification timestamps.
- **Key Expressions:** References incident IDs.
- **Connections:** 
  - Input: `Route By Severity` (Fallback/Info)
  - Output: None (Terminal node)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database connectivity drops.

---

### Block 1.3: Alert Disposition

#### Overview
Accepts external alert disposition feedback (confirmed, false positive, wont_fix) via a header-authenticated webhook, validates payload integrity, updates the incident record, and returns HTTP responses.

#### Nodes Involved
- `Disposition Webhook`
- `Validate Feedback`
- `Feedback Valid`
- `Record Disposition`
- `Confirm Feedback`
- `Reject Feedback`

#### Node Details

##### Disposition Webhook
- **Type & Role:** `n8n-nodes-base.webhook` — Webhook trigger node receiving external POST requests at `/kpi-alert-feedback`.
- **Configuration:** Uses header-auth authentication and response mode handled by downstream nodes.
- **Key Expressions:** None.
- **Connections:** Output connects to `Validate Feedback`.
- **Version Requirements:** v2.1
- **Edge Cases & Failure Types:** Invalid HTTP headers or unauthenticated requests.

##### Validate Feedback
- **Type & Role:** `n8n-nodes-base.code` — Code execution node validating incoming disposition payloads against allowed values (`confirmed`, `false_positive`, `wont_fix`).
- **Configuration:** JavaScript parsing body parameters and checking integer validity on incident IDs.
- **Key Expressions:** Parses `$json.body || $json`.
- **Connections:** 
  - Input: `Disposition Webhook`
  - Output: `Feedback Valid`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Malformed JSON bodies.

##### Feedback Valid
- **Type & Role:** `n8n-nodes-base.if` — Conditional router checking if feedback validation passed.
- **Configuration:** Condition checks `valid === true`.
- **Key Expressions:** `={{ $json.valid }}`
- **Connections:** 
  - Input: `Validate Feedback`
  - Output: True branch to `Record Disposition`, False branch to `Reject Feedback`
- **Version Requirements:** v2.3
- **Edge Cases & Failure Types:** Type evaluation mismatches.

##### Record Disposition
- **Type & Role:** `n8n-nodes-base.postgres` — Database writer updating `kpi_incidents` with validated dispositions, notes, and actor details.
- **Configuration:** Executes update query adjusting incident status based on disposition type.
- **Key Expressions:** Query replacements binding incident IDs, dispositions, notes, and actors.
- **Connections:** 
  - Input: `Feedback Valid` (True)
  - Output: `Confirm Feedback`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Missing incident IDs in the database.

##### Confirm Feedback
- **Type & Role:** `n8n-nodes-base.respondToWebhook` — Webhook response node returning HTTP 200 JSON confirmation.
- **Configuration:** Responds with JSON containing success status, incident ID, and accepted disposition.
- **Key Expressions:** References upstream validation payloads.
- **Connections:** 
  - Input: `Record Disposition`
  - Output: None (Terminal node)
- **Version Requirements:** v1.5
- **Edge Cases & Failure Types:** Connection resets before response dispatch.

##### Reject Feedback
- **Type & Role:** `n8n-nodes-base.respondToWebhook` — Webhook response node returning HTTP 400 error payloads for invalid feedback.
- **Configuration:** Configured with response code `400` responding with error problem arrays.
- **Key Expressions:** `={{ JSON.stringify({ ok: false, problems: $json.problems }) }}`
- **Connections:** 
  - Input: `Feedback Valid` (False)
  - Output: None (Terminal node)
- **Version Requirements:** v1.5
- **Edge Cases & Failure Types:** None.

---

### Block 1.4: Weekly Precision Report

#### Overview
Runs every Monday at 08:00 AM to query alert precision over a 30-day window, evaluate threshold modification recommendations, format comprehensive HTML and Slack reports, and distribute them via Gmail and Slack.

#### Nodes Involved
- `Weekly Report Cron`
- `Report Settings`
- `Query Alert Precision`
- `Recommend Threshold Changes`
- `Format Precision Report`
- `Send Precision Report`
- `Post Precision Summary`

#### Node Details

##### Weekly Report Cron
- **Type & Role:** `n8n-nodes-base.scheduleTrigger` — Scheduled trigger firing weekly on Mondays at 08:00 AM.
- **Configuration:** Interval set to weekly on day 1 at hour 8, minute 0.
- **Key Expressions:** None.
- **Connections:** Output connects to `Report Settings`.
- **Version Requirements:** v1.3
- **Edge Cases & Failure Types:** Scheduler timing skips.

##### Report Settings
- **Type & Role:** `n8n-nodes-base.set` — Parameter setup node defining reporting constants.
- **Configuration:** Manual assignment mode setting report emails, slack channels, window days (30), and minimum sample sizes (5).
- **Key Expressions:** Static assignments.
- **Connections:** 
  - Input: `Weekly Report Cron`
  - Output: `Query Alert Precision`
- **Version Requirements:** v3.4
- **Edge Cases & Failure Types:** Missing variable definitions.

##### Query Alert Precision
- **Type & Role:** `n8n-nodes-base.postgres` — Database reader querying incident precision, false positive counts, and confirmation ratios over the reporting window.
- **Configuration:** Executes aggregation query joining `kpi_incidents` and `kpi_metrics`.
- **Key Expressions:** None.
- **Connections:** 
  - Input: `Report Settings`
  - Output: `Recommend Threshold Changes`
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Database query timeouts.

##### Recommend Threshold Changes
- **Type & Role:** `n8n-nodes-base.code` — Code execution node analyzing alert precision and ignore rates to generate threshold recommendations (`hold`, `raise`, `lower`, `review`).
- **Configuration:** JavaScript evaluation block applying minimum sample constraints and precision thresholds.
- **Key Expressions:** References reporting settings and query results.
- **Connections:** 
  - Input: `Query Alert Precision`
  - Output: `Format Precision Report`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Division by zero on empty judge counts.

##### Format Precision Report
- **Type & Role:** `n8n-nodes-base.code` — Code execution node formatting precision analysis results into HTML email bodies and Slack summary strings.
- **Configuration:** JavaScript string templating constructing tables and bulleted action lists.
- **Key Expressions:** References precision metrics and recommendation arrays.
- **Connections:** 
  - Input: `Recommend Threshold Changes`
  - Output: `Send Precision Report`, `Post Precision Summary`
- **Version Requirements:** v2
- **Edge Cases & Failure Types:** Special character escaping issues in metric keys.

##### Send Precision Report
- **Type & Role:** `n8n-nodes-base.gmail` — Gmail integration node sending HTML precision reports to designated email addresses.
- **Configuration:** Gmail OAuth2 authentication with HTML email type (`onError: continueRegularOutput`).
- **Key Expressions:** `={{ $('Report Settings').first().json.report_email }}`
- **Connections:** 
  - Input: `Format Precision Report`
  - Output: None (Terminal node)
- **Version Requirements:** v2.2
- **Edge Cases & Failure Types:** SMTP/Gmail API rate limits.

##### Post Precision Summary
- **Type & Role:** `n8n-nodes-base.slack` — Slack integration node posting weekly precision summaries to alert channels.
- **Configuration:** Slack access token authentication posting markdown text (`onError: continueRegularOutput`).
- **Key Expressions:** `={{ $json.slack }}`
- **Connections:** 
  - Input: `Format Precision Report`
  - Output: None (Terminal node)
- **Version Requirements:** v2.7
- **Edge Cases & Failure Types:** Slack channel posting errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Collect On Demand` | `n8n-nodes-base.manualTrigger` | Manual entry point for metric collection | None | `Load Metric Registry` | A · Metric Collection And Baselines |
| `Collect Nightly 01:00` | `n8n-nodes-base.scheduleTrigger` | Nightly automated trigger at 01:00 | None | `Load Metric Registry` | A · Metric Collection And Baselines |
| `Load Metric Registry` | `n8n-nodes-base.postgres` | Loads enabled metrics from database | `Collect On Demand`, `Collect Nightly 01:00` | `Run Metric Query` | A · Metric Collection And Baselines |
| `Run Metric Query` | `n8n-nodes-base.postgres` | Executes individual metric SQL queries | `Load Metric Registry` | `Build Observation` | A · Metric Collection And Baselines |
| `Build Observation` | `n8n-nodes-base.code` | Parses query results and handles errors | `Run Metric Query` | `Store Observation` | A · Metric Collection And Baselines |
| `Store Observation` | `n8n-nodes-base.postgres` | Stores yesterday's observation records | `Build Observation` | `Recompute Baselines` | A · Metric Collection And Baselines |
| `Recompute Baselines` | `n8n-nodes-base.postgres` | Recalculates medians, MAD, and seasonality | `Store Observation` | `Summarise Baseline Coverage` | A · Metric Collection And Baselines |
| `Summarise Baseline Coverage` | `n8n-nodes-base.code` | Evaluates baseline observation history | `Recompute Baselines` | `Record Baseline Run` | A · Metric Collection And Baselines |
| `Record Baseline Run` | `n8n-nodes-base.postgres` | Logs baseline execution run stats | `Summarise Baseline Coverage` | `Only Thin Coverage` | A · Metric Collection And Baselines |
| `Only Thin Coverage` | `n8n-nodes-base.filter` | Filters metrics lacking sufficient history | `Record Baseline Run` | `Warn Thin History` | A · Metric Collection And Baselines |
| `Warn Thin History` | `n8n-nodes-base.slack` | Posts warning about immature baselines | `Only Thin Coverage` | None | A · Metric Collection And Baselines |
| `Detect Every Morning 07:00` | `n8n-nodes-base.scheduleTrigger` | Morning automated trigger at 07:00 | None | `Detection Settings` | B · Detection And Investigation |
| `Detection Settings` | `n8n-nodes-base.set` | Sets global detection thresholds | `Detect Every Morning 07:00` | `Fetch Latest Vs Baseline` | B · Detection And Investigation |
| `Fetch Latest Vs Baseline` | `n8n-nodes-base.postgres` | Fetches observations joined with baselines | `Detection Settings` | `Score Anomalies` | B · Detection And Investigation |
| `Score Anomalies` | `n8n-nodes-base.code` | Calculates robust z-scores and severity | `Fetch Latest Vs Baseline` | `Keep Only Anomalies` | B · Detection And Investigation |
| `Keep Only Anomalies` | `n8n-nodes-base.filter` | Filters records to keep only anomalies | `Score Anomalies` | `Fetch Open Incidents` | B · Detection And Investigation |
| `Fetch Open Incidents` | `n8n-nodes-base.postgres` | Queries unresolved open incidents | `Keep Only Anomalies` | `Suppress Duplicates` | B · Detection And Investigation |
| `Suppress Duplicates` | `n8n-nodes-base.code` | Suppresses duplicate alerts and caps runs | `Fetch Open Incidents` | `New Incident` | B · Detection And Investigation |
| `New Incident` | `n8n-nodes-base.if` | Branches based on suppression status | `Suppress Duplicates` | `Open Incident Record`, `Log Suppressed Alert` | B · Detection And Investigation |
| `Log Suppressed Alert` | `n8n-nodes-base.postgres` | Logs suppressed alerts to database | `New Incident` | None | B · Detection And Investigation |
| `Open Incident Record` | `n8n-nodes-base.postgres` | Creates active incident database records | `New Incident` | `Investigate Anomaly` | B · Detection And Investigation |
| `Investigator Model` | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Configures Claude chat model provider | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Investigation Schema` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured output parsing | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Get Metric Definition` | `n8n-nodes-base.postgresTool` | AI tool for looking up metric metadata | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Get Metric History` | `n8n-nodes-base.postgresTool` | AI tool for pulling historical observations | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Get Segment Breakdown` | `n8n-nodes-base.postgresTool` | AI tool for segment dimension breakdowns | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Get Related Metric Moves` | `n8n-nodes-base.postgresTool` | AI tool for identifying concurrent moves | None | `Investigate Anomaly` | B · Detection And Investigation |
| `List Recent Deployments` | `n8n-nodes-base.githubTool` | AI tool for listing recent GitHub PRs | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Think Out Loud` | `@n8n/n8n-nodes-langchain.toolCode` | Scratchpad reasoning tool for agent | None | `Investigate Anomaly` | B · Detection And Investigation |
| `Investigate Anomaly` | `@n8n/n8n-nodes-langchain.agent` | LangChain agent investigating root causes | `Open Incident Record`, AI Model/Tools | `Check Evidence`, `Record Investigation Failure` | B · Detection And Investigation |
| `Check Evidence` | `n8n-nodes-base.code` | Validates agent evidence and fabricated keys | `Investigate Anomaly` | `Explanation Trusted` | B · Detection And Investigation |
| `Record Investigation Failure` | `n8n-nodes-base.code` | Handles investigator execution failures | `Investigate Anomaly` | `Label Unexplained` | B · Detection And Investigation |
| `Explanation Trusted` | `n8n-nodes-base.if` | Verifies trustworthiness of explanation | `Check Evidence` | `Label Explained`, `Label Unexplained` | B · Detection And Investigation |
| `Label Explained` | `n8n-nodes-base.set` | Labels incident explanation as trusted | `Explanation Trusted` | `Merge Explanation Paths` | B · Detection And Investigation |
| `Label Unexplained` | `n8n-nodes-base.set` | Labels incident explanation as discarded | `Explanation Trusted`, `Record Investigation Failure` | `Merge Explanation Paths` | B · Detection And Investigation |
| `Merge Explanation Paths` | `n8n-nodes-base.merge` | Consolidates explanation branch paths | `Label Explained`, `Label Unexplained` | `Update Incident Findings` | B · Detection And Investigation |
| `Update Incident Findings` | `n8n-nodes-base.postgres` | Updates incident record with findings | `Merge Explanation Paths` | `Route By Severity` | B · Detection And Investigation |
| `Route By Severity` | `n8n-nodes-base.switch` | Routes alerts based on incident severity | `Update Incident Findings` | `Alert Critical In Slack`, `Alert Warning In Slack`, `Record Info Only` | B · Detection And Investigation |
| `Alert Critical In Slack` | `n8n-nodes-base.slack` | Posts critical alerts to Slack | `Route By Severity` | `File Investigation Issue` | B · Detection And Investigation |
| `File Investigation Issue` | `n8n-nodes-base.github` | Files GitHub issues for critical alerts | `Alert Critical In Slack` | `Request Owner Acknowledgement` | B · Detection And Investigation |
| `Request Owner Acknowledgement`| `n8n-nodes-base.gmail` | Requests email owner acknowledgement | `File Investigation Issue` | `Record Acknowledgement` | B · Detection And Investigation |
| `Record Acknowledgement` | `n8n-nodes-base.postgres` | Records owner email approval states | `Request Owner Acknowledgement` | None | B · Detection And Investigation |
| `Alert Warning In Slack` | `n8n-nodes-base.slack` | Posts warning alerts to Slack | `Route By Severity` | `Record Warning Sent` | B · Detection And Investigation |
| `Record Warning Sent` | `n8n-nodes-base.postgres` | Marks warning alerts as sent | `Alert Warning In Slack` | None | B · Detection And Investigation |
| `Record Info Only` | `n8n-nodes-base.postgres` | Logs info-level anomalies silently | `Route By Severity` | None | B · Detection And Investigation |
| `Disposition Webhook` | `n8n-nodes-base.webhook` | Webhook endpoint for alert feedback | None | `Validate Feedback` | C · Alert Disposition |
| `Validate Feedback` | `n8n-nodes-base.code` | Validates feedback disposition payloads | `Disposition Webhook` | `Feedback Valid` | C · Alert Disposition |
| `Feedback Valid` | `n8n-nodes-base.if` | Verifies feedback payload validity | `Validate Feedback` | `Record Disposition`, `Reject Feedback` | C · Alert Disposition |
| `Record Disposition` | `n8n-nodes-base.postgres` | Updates incident with validated feedback | `Feedback Valid` | `Confirm Feedback` | C · Alert Disposition |
| `Confirm Feedback` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 JSON confirmation | `Record Disposition` | None | C · Alert Disposition |
| `Reject Feedback` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 rejection payload | `Feedback Valid` | None | C · Alert Disposition |
| `Weekly Report Cron` | `n8n-nodes-base.scheduleTrigger` | Weekly report trigger on Mondays 08:00 | None | `Report Settings` | D · Weekly Precision Report |
| `Report Settings` | `n8n-nodes-base.set` | Configures report window and parameters | `Weekly Report Cron` | `Query Alert Precision` | D · Weekly Precision Report |
| `Query Alert Precision` | `n8n-nodes-base.postgres` | Queries alert precision over 30 days | `Report Settings` | `Recommend Threshold Changes` | D · Weekly Precision Report |
| `Recommend Threshold Changes` | `n8n-nodes-base.code` | Evaluates threshold change recommendations | `Query Alert Precision` | `Format Precision Report` | D · Weekly Precision Report |
| `Format Precision Report` | `n8n-nodes-base.code` | Formats HTML and Slack precision reports | `Recommend Threshold Changes` | `Send Precision Report`, `Post Precision Summary` | D · Weekly Precision Report |
| `Send Precision Report` | `n8n-nodes-base.gmail` | Sends precision report email via Gmail | `Format Precision Report` | None | D · Weekly Precision Report |
| `Post Precision Summary` | `n8n-nodes-base.slack` | Posts weekly precision summary to Slack | `Format Precision Report` | None | D · Weekly Precision Report |

---

### 4. Reproducing the Workflow from Scratch

To manually rebuild the workflow from scratch, follow these step-by-step instructions:

1. **Database Schema Setup:** Prepare your PostgreSQL database by creating the required tables (`kpi_metrics`, `kpi_observations`, `kpi_baselines`, `kpi_baseline_runs`, `kpi_incidents`, `kpi_suppressed_alerts`, and `kpi_segment_observations`). Populate `kpi_metrics` with active metric definitions and SQL collection queries.
2. **Configure Credentials:** Set up credentials in n8n for:
   - PostgreSQL (all database nodes)
   - Slack (AccessToken for alerting nodes)
   - GitHub (AccessToken for repository tools and issue filing)
   - Gmail (OAuth2 for approval emails and precision reports)
   - OpenRouter (API key for Claude investigator model)
3. **Build Block 1.1 (Collection & Baselines):**
   - Create `Collect On Demand` (`n8n-nodes-base.manualTrigger`) and `Collect Nightly 01:00` (`n8n-nodes-base.scheduleTrigger`, interval: daily at 01:00).
   - Create `Load Metric Registry` (`n8n-nodes-base.postgres`) selecting enabled metrics. Connect both triggers to it.
   - Create `Run Metric Query` (`n8n-nodes-base.postgres`) with `onError: continueRegularOutput`, using `={{ $json.metric_sql }}`. Connect registry output to it.
   - Create `Build Observation` (`n8n-nodes-base.code`) to sanitize collection output, connecting it after query execution.
   - Create `Store Observation` (`n8n-nodes-base.postgres`) with UPSERT logic, connecting it after observation building.
   - Create `Recompute Baselines` (`n8n-nodes-base.postgres`, `executeOnce: true`) executing median and MAD calculations. Connect it after observation storage.
   - Create `Summarise Baseline Coverage` (`n8n-nodes-base.code`) checking `MIN_OBS = 14`.
   - Create `Record Baseline Run` (`n8n-nodes-base.postgres`) logging baseline stats.
   - Create `Only Thin Coverage` (`n8n-nodes-base.filter`) checking `thin_count > 0`.
   - Create `Warn Thin History` (`n8n-nodes-base.slack`) alerting on immature metrics.
4. **Build Block 1.2 (Detection & Investigation):**
   - Create `Detect Every Morning 07:00` (`n8n-nodes-base.scheduleTrigger`, daily at 07:00) and link to `Detection Settings` (`n8n-nodes-base.set`) defining constants (`critical_channel`, `warning_channel`, `oncall_email`, caps, and cooldowns).
   - Create `Fetch Latest Vs Baseline` (`n8n-nodes-base.postgres`) and `Score Anomalies` (`n8n-nodes-base.code`) to calculate robust z-scores.
   - Create `Keep Only Anomalies` (`n8n-nodes-base.filter`) and `Fetch Open Incidents` (`n8n-nodes-base.postgres`, `executeOnce: true`, `alwaysOutputData: true`).
   - Create `Suppress Duplicates` (`n8n-nodes-base.code`) and `New Incident` (`n8n-nodes-base.if`).
   - Route true branch to `Open Incident Record` (`n8n-nodes-base.postgres`) and false branch to `Log Suppressed Alert` (`n8n-nodes-base.postgres`).
   - Configure the AI Investigation Agent:
     - Create `Investigator Model` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`, `anthropic/claude-opus-5`).
     - Create `Investigation Schema` (`@n8n/n8n-nodes-langchain.outputParserStructured`) enforcing the required output JSON schema.
     - Create AI tools: `Get Metric Definition`, `Get Metric History`, `Get Segment Breakdown`, `Get Related Metric Moves` (`n8n-nodes-base.postgresTool`), `List Recent Deployments` (`n8n-nodes-base.githubTool`), and `Think Out Loud` (`@n8n/n8n-nodes-langchain.toolCode`).
     - Create `Investigate Anomaly` (`@n8n/n8n-nodes-langchain.agent`) linking the model, schema, and all tools.
   - Create `Check Evidence` (`n8n-nodes-base.code`) and `Record Investigation Failure` (`n8n-nodes-base.code`).
   - Create `Explanation Trusted` (`n8n-nodes-base.if`) evaluating evidence thresholds (`evidence_count >= 2`, `confidence >= 0.6`, `fabricated_count === 0`).
   - Create `Label Explained` and `Label Unexplained` (`n8n-nodes-base.set`), merge them via `Merge Explanation Paths` (`n8n-nodes-base.merge`), and update incidents via `Update Incident Findings` (`n8n-nodes-base.postgres`).
   - Route severity via `Route By Severity` (`n8n-nodes-base.switch`):
     - Critical: `Alert Critical In Slack` -> `File Investigation Issue` (GitHub) -> `Request Owner Acknowledgement` (Gmail with approval options) -> `Record Acknowledgement`.
     - Warning: `Alert Warning In Slack` -> `Record Warning Sent`.
     - Info/Fallback: `Record Info Only`.
5. **Build Block 1.3 (Alert Disposition):**
   - Create `Disposition Webhook` (`n8n-nodes-base.webhook`, path `kpi-alert-feedback`, header-auth).
   - Create `Validate Feedback` (`n8n-nodes-base.code`) checking allowed dispositions (`confirmed`, `false_positive`, `wont_fix`).
   - Create `Feedback Valid` (`n8n-nodes-base.if`).
   - Route true to `Record Disposition` (`n8n-nodes-base.postgres`) -> `Confirm Feedback` (`n8n-nodes-base.respondToWebhook`, 200 OK).
   - Route false to `Reject Feedback` (`n8n-nodes-base.respondToWebhook`, 400 Bad Request).
6. **Build Block 1.4 (Weekly Precision Report):**
   - Create `Weekly Report Cron` (`n8n-nodes-base.scheduleTrigger`, Mondays at 08:00) -> `Report Settings` (`n8n-nodes-base.set`).
   - Create `Query Alert Precision` (`n8n-nodes-base.postgres`) -> `Recommend Threshold Changes` (`n8n-nodes-base.code`) -> `Format Precision Report` (`n8n-nodes-base.code`).
   - Dispatch outputs to `Send Precision Report` (`n8n-nodes-base.gmail`) and `Post Precision Summary` (`n8n-nodes-base.slack`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| KPI Anomaly Watchdog Purpose | Watches core business numbers and provides automated AI root-cause explanations with evidence checks rather than raw alerts. |
| Baseline Warmup Period | Requires at least 14 days of observation history before metrics are scored to prevent false positives. |
| Disposition Feedback Requirement | Requires webhook integration to capture human feedback (`confirmed`, `false_positive`, `wont_fix`) so weekly precision reports can accurately calculate threshold recommendations. |