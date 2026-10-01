Generate and nurture social leads from Reddit and Twitter with OpenAI and Airtable

https://n8nworkflows.xyz/workflows/generate-and-nurture-social-leads-from-reddit-and-twitter-with-openai-and-airtable-20046


# Generate and nurture social leads from Reddit and Twitter with OpenAI and Airtable

### 1. Workflow Overview

This workflow functions as an AI-powered lead generation, enrichment, drafting, and nurturing engine. It scans multiple digital platforms for buying-intent signals, profiles the prospects, filters messages through multi-LLM quality controls, enforces human-in-the-loop review, manages outreach delivery via email or manual channels, and runs autonomous pipeline maintenance and reporting tasks.

The functional blocks of the workflow include:
- **1.1 Input Reception & Intent Detection:** Scans Reddit (via Google Custom Search), Twitter/X, and RSS feeds every 15 minutes, merges and normalizes leads, deduplicates against Airtable, and classifies buying intent using OpenAI (GPT-4o).
- **1.2 Lead Profiling & Enrichment:** Performs profile lookups, checks for Apollo.io API availability, falls back to Google Search if needed, builds comprehensive buyer personas using Anthropic Claude, and computes weighted lead scores.
- **1.3 Multi-LLM Consensus & Drafting:** Routes leads by score class (HOT, WARM, COLD), executes a multi-step generation strategy (GPT-4o strategy, Claude 3.5 Sonnet writing, GPT-4o-mini quality gating) for hot leads, and generates single-model drafts for warm leads before queuing them.
- **1.4 Human-in-the-Loop Review:** Formats Slack review cards for channel #lead-review, logs review dispatch times, and tracks interaction records.
- **1.5 Premium Outreach & Conversion Funnel:** Routes approved drafts to email (with randomized delays), manual Reddit, or manual LinkedIn workflows, and handles incoming reply sentiment analysis.
- **1.6 Smart Nurture & Warmth Decay:** Evaluates active leads daily, applies warmth decay formulas based on inactivity, and uses OpenAI to determine automated nurture actions (Wait, Follow-up, Change Channel, Archive).
- **1.7 Competitive Intelligence & Command Center:** Scans competitor complaints periodically via Google Search, evaluates opportunities, generates weekly pipeline insights via OpenAI, and posts analytics reports to Slack.
- **1.8 Shadow Mode & Global Error Handling:** Provides a validation layer for comparing AI-generated drafts with human responses and runs a global error handler to capture execution failures, log them to Airtable, and alert Slack #lead-alerts.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Intent Detection
- **Overview:** This block executes on a 15-minute schedule, loads configuration variables, queries external platforms for buying-intent posts, normalizes payload structures, deduplicates records against Airtable, and filters out noise via OpenAI.
- **Nodes Involved:** `Scan Every 15min`, `Load Config`, `Reddit via Google`, `Twitter/X Search`, `RSS Feed Monitor`, `Merge Sources`, `Normalize Data`, `Check Duplicates`, `Filter Known`, `AI Intent Classifier`, `Filter Low Intent`, `Save New Lead`, `Track Detection`.
- **Node Details:**
  - **Scan Every 15min**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers workflow execution every 15 minutes.
    - Configuration: Cron-like interval rule set to 15 minutes.
    - Input/Output: None (Input) / Triggers `Load Config` (Output).
    - Edge Cases: Missed triggers if n8n instance is offline.
  - **Load Config**
    - Type: `n8n-nodes-base.code`
    - Role: Defines subreddits, intent keywords, and result limits.
    - Configuration: JavaScript code returning an array with configuration objects.
    - Input/Output: `Scan Every 15min` (Input) / `Reddit via Google`, `Twitter/X Search`, `RSS Feed Monitor` (Output).
    - Edge Cases: Invalid JS syntax.
  - **Reddit via Google**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Searches Reddit posts using Google Custom Search API.
    - Configuration: GET request to Google Custom Search API using query parameters derived from configuration keywords.
    - Input/Output: `Load Config` (Input) / `Merge Sources` (Output).
    - Edge Cases: API rate limits, authentication failures, missing API keys.
  - **Twitter/X Search**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Queries recent tweets matching intent keywords.
    - Configuration: GET request to Twitter/X API v2 recent search endpoint with bearer token authorization.
    - Input/Output: `Load Config` (Input) / `Merge Sources` (Output).
    - Credentials: Twitter/X Bearer Token.
    - Edge Cases: Rate limiting (429 errors), invalid bearer token.
  - **RSS Feed Monitor**
    - Type: `n8n-nodes-base.rssFeedRead`
    - Role: Reads RSS feed entries from Google Alerts.
    - Configuration: RSS feed URL lookup.
    - Input/Output: `Load Config` (Input) / `Merge Sources` (Output).
    - Edge Cases: Feed unavailable, malformed XML.
  - **Merge Sources**
    - Type: `n8n-nodes-base.merge`
    - Role: Combines items from Reddit, Twitter, and RSS inputs.
    - Configuration: Append mode.
    - Input/Output: `Reddit via Google`, `Twitter/X Search`, `RSS Feed Monitor` (Input) / `Normalize Data` (Output).
  - **Normalize Data**
    - Type: `n8n-nodes-base.code`
    - Role: Normalizes disparate data structures into a unified lead format.
    - Configuration: JavaScript processing extracting source, URL, content, author, and timestamp.
    - Input/Output: `Merge Sources` (Input) / `Check Duplicates` (Output).
  - **Check Duplicates**
    - Type: `n8n-nodes-base.airtable`
    - Role: Searches Airtable `LEADS` table for existing records matching the source URL.
    - Configuration: Airtable search operation with filter formula using `source_url`.
    - Input/Output: `Normalize Data` (Input) / `Filter Known` (Output).
    - Credentials: Airtable API token.
    - Edge Cases: Airtable connection failure, base/table ID mismatches.
  - **Filter Known**
    - Type: `n8n-nodes-base.code`
    - Role: Discards known duplicate leads or skipped posts.
    - Configuration: JavaScript filter verifying absence of existing ID.
    - Input/Output: `Check Duplicates` (Input) / `AI Intent Classifier` (Output).
  - **AI Intent Classifier**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Sends post content to OpenAI GPT-4o to score buying intent.
    - Configuration: POST request to OpenAI Chat Completions API with enforced JSON response format.
    - Input/Output: `Filter Known` (Input) / `Filter Low Intent` (Output).
    - Credentials: OpenAI API Key.
    - Edge Cases: OpenAI API downtime, rate limits, JSON parsing failures.
  - **Filter Low Intent**
    - Type: `n8n-nodes-base.code`
    - Role: Drops leads with intent scores below 30.
    - Configuration: JavaScript code parsing AI JSON response and filtering thresholds.
    - Input/Output: `AI Intent Classifier` (Input) / `Save New Lead`, `Track Detection` (Output).
  - **Save New Lead**
    - Type: `n8n-nodes-base.airtable`
    - Role: Persists newly detected leads into Airtable `LEADS` table with 'New' stage.
    - Configuration: Airtable create operation mapping lead properties.
    - Input/Output: `Filter Low Intent` (Input) / `Reddit Profile Search` (Output).
    - Credentials: Airtable API token.
  - **Track Detection**
    - Type: `n8n-nodes-base.airtable`
    - Role: Logs detection event metrics into Airtable `ANALYTICS` table.
    - Configuration: Airtable create operation.
    - Input/Output: `Filter Low Intent` (Input) / None (Terminal branch node).
    - Credentials: Airtable API token.

---

#### 2.2 Lead Profiling & Enrichment
- **Overview:** Enriches lead profiles using Apollo.io or a Google search fallback, builds detailed buyer personas via Anthropic Claude, calculates weighted scores, and updates Airtable records.
- **Nodes Involved:** `Reddit Profile Search`, `Has Apollo Key?`, `Apollo Enrichment`, `Google Fallback`, `Merge Enrichment`, `AI Lead Profiler`, `Calculate Score`, `Save Enriched Lead`, `Route by Score`.
- **Node Details:**
  - **Reddit Profile Search**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Searches Google for user profile details.
    - Configuration: GET request to Google Custom Search API.
    - Input/Output: `Save New Lead` (Input) / `Has Apollo Key?` (Output).
  - **Has Apollo Key?**
    - Type: `n8n-nodes-base.if`
    - Role: Checks whether the `APOLLO_API_KEY` environment variable is defined.
    - Configuration: Conditional statement checking environment variable.
    - Input/Output: `Reddit Profile Search` (Input) / `Apollo Enrichment` (True branch), `Google Fallback` (False branch).
  - **Apollo Enrichment**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Matches person and company data using Apollo.io API.
    - Configuration: POST request to Apollo people match endpoint.
    - Input/Output: `Has Apollo Key?` (Input) / `Merge Enrichment` (Output).
    - Credentials: Apollo API Key.
    - Edge Cases: Missing API key, quota exhaustion.
  - **Google Fallback**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Fallback enrichment via LinkedIn Google search queries.
    - Configuration: GET request to Google Custom Search API.
    - Input/Output: `Has Apollo Key?` (Input) / `Merge Enrichment` (Output).
  - **Merge Enrichment**
    - Type: `n8n-nodes-base.merge`
    - Role: Merges enrichment outputs from Apollo or Google fallback.
    - Configuration: Append mode.
    - Input/Output: `Apollo Enrichment`, `Google Fallback` (Input) / `AI Lead Profiler` (Output).
  - **AI Lead Profiler**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Builds buyer profiles using Anthropic Claude 3.5 Sonnet.
    - Configuration: POST request to Anthropic Messages API with JSON output instruction.
    - Input/Output: `Merge Enrichment` (Input) / `Calculate Score` (Output).
    - Credentials: Anthropic API Key.
    - Edge Cases: API rate limits, invalid JSON response.
  - **Calculate Score**
    - Type: `n8n-nodes-base.code`
    - Role: Computes weighted total scores based on intent, fit, engagement, and timing scores.
    - Configuration: JavaScript calculations assigning weights (Intent 40%, Fit 30%, Engagement 20%, Timing 10%).
    - Input/Output: `AI Lead Profiler` (Input) / `Save Enriched Lead` (Output).
  - **Save Enriched Lead**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates lead record in Airtable with enrichment data and total score.
    - Configuration: Airtable update operation matching by ID.
    - Input/Output: `Calculate Score` (Input) / `Route by Score` (Output).
    - Credentials: Airtable API token.
  - **Route by Score**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes leads based on classification (HOT, WARM, COLD).
    - Configuration: Switch routing rules targeting HOT, WARM, and COLD outputs.
    - Input/Output: `Save Enriched Lead` (Input) / `GPT-4o Strategy`, `Quick Draft (WARM)`, `Wait (Do Nothing)` (Output).

---

#### 2.3 Multi-LLM Consensus & Drafting
- **Overview:** Executes a multi-step AI drafting pipeline for HOT leads involving strategy formulation, message crafting via Claude, and a strict quality gate via OpenAI. For WARM leads, it uses a single-model generation step before queuing drafts in Airtable.
- **Nodes Involved:** `GPT-4o Strategy`, `Claude Message Craft`, `Quality Gate`, `Quick Draft (WARM)`, `Merge Drafts`, `Save to Queue`.
- **Node Details:**
  - **GPT-4o Strategy**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Formulates sales strategy for HOT leads using OpenAI GPT-4o.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Route by Score` (Input) / `Claude Message Craft` (Output).
    - Credentials: OpenAI API Key.
  - **Claude Message Craft**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Drafts outreach variants using Anthropic Claude 3.5 Sonnet.
    - Configuration: POST request to Anthropic Messages API.
    - Input/Output: `GPT-4o Strategy` (Input) / `Quality Gate` (Output).
    - Credentials: Anthropic API Key.
  - **Quality Gate**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Validates outreach drafts against spam and quality rules using OpenAI GPT-4o-mini.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Claude Message Craft` (Input) / `Merge Drafts` (Output).
    - Credentials: OpenAI API Key.
  - **Quick Draft (WARM)**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Generates outreach drafts for WARM leads using OpenAI GPT-4o-mini.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Route by Score` (Input) / `Merge Drafts` (Output).
    - Credentials: OpenAI API Key.
  - **Merge Drafts**
    - Type: `n8n-nodes-base.code`
    - Role: Standardizes and merges draft outputs from HOT and WARM paths.
    - Configuration: JavaScript code parsing JSON content and mapping fallback fields.
    - Input/Output: `Quality Gate`, `Quick Draft (WARM)` (Input) / `Save to Queue` (Output).
  - **Save to Queue**
    - Type: `n8n-nodes-base.airtable`
    - Role: Saves outreach drafts into the Airtable `OUTREACH_QUEUE` table.
    - Configuration: Airtable create operation.
    - Input/Output: `Merge Drafts` (Input) / `Format Slack Card` (Output).
    - Credentials: Airtable API token.

---

<h4>2.4 Human-in-the-Loop Review</h4>
- **Overview:** Formats lead data and draft variants into rich Slack cards and sends them to the `#lead-review` channel for human evaluation.
- **Nodes Involved:** `Format Slack Card`, `Send to #lead-review`, `Log Review Sent`, `Update Queue`, `Log Interaction`.
- **Node Details:**
  - **Format Slack Card**
    - Type: `n8n-nodes-base.code`
    - Role: Constructs formatted Slack message markdown containing score, source, user, persona, and draft variants.
    - Configuration: JavaScript string formatting.
    - Input/Output: `Save to Queue` (Input) / `Send to #lead-review` (Output).
  - **Send to #lead-review**
    - Type: `n8n-nodes-base.slack`
    - Role: Posts review card to Slack `#lead-review` channel.
    - Configuration: Slack post message action.
    - Input/Output: `Format Slack Card` (Input) / `Log Review Sent` (Output).
    - Credentials: Slack API.
  - **Log Review Sent**
    - Type: `n8n-nodes-base.code`
    - Role: Appends review timestamp and status properties to payload.
    - Configuration: JavaScript timestamp assignment.
    - Input/Output: `Send to #lead-review` (Input) / `Update Queue`, `Log Interaction` (Output).
  - **Update Queue**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates outreach queue status to 'Pending Review'.
    - Configuration: Airtable update operation matching by ID.
    - Input/Output: `Log Review Sent` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Log Interaction**
    - Type: `n8n-nodes-base.airtable`
    - Role: Logs review dispatch interaction into Airtable `INTERACTIONS` table.
    - Configuration: Airtable create operation.
    - Input/Output: `Log Review Sent` (Input) / None (Terminal).
    - Credentials: Airtable API token.

---

<h4>2.5 Premium Outreach & Conversion Funnel</h4>
- **Overview:** Routes approved messages by channel (Email, Reddit, LinkedIn), handles randomized delivery delays, dispatches emails via Gmail, logs outbound interactions, schedules follow-ups, and analyzes incoming replies for sentiment.
- **Nodes Involved:** `Route Channel`, `Random Delay`, `Send Email`, `Log Email`, `Schedule Follow-up`, `Update Contacted`, `Reddit Manual`, `LinkedIn Manual`, `Analyze Reply`, `Reply Sentiment`, `Send Calendly`, `Blacklist`, `Forward Question`.
- **Node Details:**
  - **Route Channel**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes outreach execution based on channel type (email, reddit, linkedin).
    - Configuration: Switch rules matching channel string values.
    - Input/Output: External/Queue Trigger (Input) / `Random Delay`, `Reddit Manual`, `LinkedIn Manual` (Output).
  - **Random Delay**
    - Type: `n8n-nodes-base.code`
    - Role: Introduces a randomized delay (45 to 120 seconds) before sending emails.
    - Configuration: JavaScript asynchronous sleep timer.
    - Input/Output: `Route Channel` (Input) / `Send Email` (Output).
  - **Send Email**
    - Type: `n8n-nodes-base.gmail`
    - Role: Sends approved outreach emails via Gmail.
    - Configuration: Gmail send message action with custom footers and subjects.
    - Input/Output: `Random Delay` (Input) / `Log Email` (Output).
    - Credentials: Gmail OAuth2.
    - Edge Cases: OAuth2 token expiry, missing recipient email address.
  - **Log Email**
    - Type: `n8n-nodes-base.airtable`
    - Role: Logs sent email interactions to Airtable `INTERACTIONS`.
    - Configuration: Airtable create operation.
    - Input/Output: `Send Email` (Input) / `Schedule Follow-up` (Output).
    - Credentials: Airtable API token.
  - **Schedule Follow-up**
    - Type: `n8n-nodes-base.code`
    - Role: Calculates the date for the first follow-up action (3 days out).
    - Configuration: JavaScript date manipulation.
    - Input/Output: `Log Email` (Input) / `Update Contacted` (Output).
  - **Update Contacted**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates lead stage to 'Contacted' and updates follow-up schedules.
    - Configuration: Airtable update operation matching by lead ID.
    - Input/Output: `Schedule Follow-up` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Reddit Manual**
    - Type: `n8n-nodes-base.slack`
    - Role: Posts ready-to-send Reddit message copy to `#outreach-ready`.
    - Configuration: Slack post message action.
    - Input/Output: `Route Channel` (Input) / None (Terminal).
    - Credentials: Slack API.
  - **LinkedIn Manual**
    - Type: `n8n-nodes-base.slack`
    - Role: Posts ready-to-send LinkedIn message copy to `#outreach-ready`.
    - Configuration: Slack post message action.
    - Input/Output: `Route Channel` (Input) / None (Terminal).
    - Credentials: Slack API.
  - **Analyze Reply**
    - Type: `n8n-nodes-base.code`
    - Role: Analyzes incoming reply body text to classify sentiment (positive, negative, question).
    - Configuration: JavaScript regex pattern matching.
    - Input/Output: Webhook/Inbox Trigger (Input) / `Reply Sentiment` (Output).
  - **Reply Sentiment**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes incoming replies based on classified sentiment.
    - Configuration: Switch routing rules for positive, negative, and question categories.
    - Input/Output: `Analyze Reply` (Input) / `Send Calendly`, `Blacklist`, `Forward Question` (Output).
  - **Send Calendly**
    - Type: `n8n-nodes-base.slack`
    - Role: Notifies team in Slack `#lead-review` with Calendly booking link for positive replies.
    - Configuration: Slack post message action.
    - Input/Output: `Reply Sentiment` (Input) / None (Terminal).
    - Credentials: Slack API.
  - **Blacklist**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates lead stage to 'Blacklisted' for unsubscribes or negative replies.
    - Configuration: Airtable update operation matching by lead ID.
    - Input/Output: `Reply Sentiment` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Forward Question**
    - Type: `n8n-nodes-base.slack`
    - Role: Forwards lead questions to Slack `#lead-review` for manual response.
    - Configuration: Slack post message action.
    - Input/Output: `Reply Sentiment` (Input) / None (Terminal).
    - Credentials: Slack API.

---

<h4>2.6 Smart Nurture & Warmth Decay</h4>
- **Overview:** Runs daily at 8:00 AM, fetches active leads, calculates warmth decay based on days of inactivity and email engagement, evaluates next actions using OpenAI, and updates or archives leads in Airtable.
- **Nodes Involved:** `Daily Nurture`, `Get Active Leads`, `Apply Warmth Decay`, `AI Next Action`, `Parse Decision`, `Nurture Action`, `Update Decayed`, `Archive Lead`, `Wait (Do Nothing)`.
- **Node Details:**
  - **Daily Nurture**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers nurture sweep every day at 8:00 AM.
    - Configuration: Cron expression `0 8 * * *`.
    - Input/Output: None (Input) / `Get Active Leads` (Output).
  - **Get Active Leads**
    - Type: `n8n-nodes-base.airtable`
    - Role: Queries Airtable for active leads in Nurturing, Contacted, or Enriched stages.
    - Configuration: Airtable search operation with filter formula.
    - Input/Output: `Daily Nurture` (Input) / `Apply Warmth Decay` (Output).
    - Credentials: Airtable API token.
  - **Apply Warmth Decay**
    - Type: `n8n-nodes-base.code`
    - Role: Calculates decayed scores based on inactivity days and email open status.
    - Configuration: JavaScript decay formula execution.
    - Input/Output: `Get Active Leads` (Input) / `AI Next Action` (Output).
  - **AI Next Action**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Uses OpenAI GPT-4o-mini to decide adaptive nurture strategy (WAIT, FOLLOW_UP, CHANGE_CHANNEL, ARCHIVE).
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Apply Warmth Decay` (Input) / `Parse Decision` (Output).
    - Credentials: OpenAI API Key.
  - **Parse Decision**
    - Type: `n8n-nodes-base.code`
    - Role: Parses AI response JSON for nurture decisions.
    - Configuration: JavaScript JSON parser with fallback.
    - Input/Output: `AI Next Action` (Input) / `Nurture Action` (Output).
  - **Nurture Action**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes leads based on parsed nurture decision.
    - Configuration: Switch rules matching follow up, change channel, and archive categories.
    - Input/Output: `Parse Decision` (Input) / `Update Decayed`, `Archive Lead` (Output).
  - **Update Decayed**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates lead record with decayed score and next action parameters.
    - Configuration: Airtable update operation matching by lead ID.
    - Input/Output: `Nurture Action` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Archive Lead**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates lead stage to 'Archived' when score falls below threshold.
    - Configuration: Airtable update operation matching by lead ID.
    - Input/Output: `Nurture Action` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Wait (Do Nothing)**
    - Type: `n8n-nodes-base.noOp`
    - Role: Placeholder node for leads requiring no action.
    - Configuration: Default No-Op configuration.
    - Input/Output: `Route by Score` or `Nurture Action` (Input) / None (Terminal).

---

<h4>2.7 Competitive Intelligence & Command Center</h4>
- **Overview:** Scans competitor complaints via Google Custom Search every 2 hours, scores opportunity sentiment with OpenAI, saves qualifying competitor leads, aggregates weekly pipeline metrics, generates AI insights, and posts weekly reports to Slack and Airtable.
- **Nodes Involved:** `Competitor Scan`, `Load Competitors`, `Search Complaints`, `Comp Sentiment`, `Is Opportunity?`, `Save Comp Lead`, `Weekly Report`, `Aggregate Pipeline`, `AI Insights`, `Post Report`, `Save Report`.
- **Node Details:**
  - **Competitor Scan**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers competitor monitoring every 2 hours.
    - Configuration: Cron expression `0 */2 * * *`.
    - Input/Output: None (Input) / `Load Competitors` (Output).
  - **Load Competitors**
    - Type: `n8n-nodes-base.airtable`
    - Role: Loads competitor records from Airtable `COMPETITORS`.
    - Configuration: Airtable search operation.
    - Input/Output: `Competitor Scan` (Input) / `Search Complaints` (Output).
    - Credentials: Airtable API token.
  - **Search Complaints**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Searches Google for Reddit complaints mentioning competitors.
    - Configuration: GET request to Google Custom Search API.
    - Input/Output: `Load Competitors` (Input) / `Comp Sentiment` (Output).
  - **Comp Sentiment**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Analyzes sentiment and switching intent of competitor mentions using OpenAI GPT-4o-mini.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Search Complaints` (Input) / `Is Opportunity?` (Output).
    - Credentials: OpenAI API Key.
  - **Is Opportunity?**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates whether opportunity score is 60 or higher.
    - Configuration: Condition checking parsed JSON opportunity score.
    - Input/Output: `Comp Sentiment` (Input) / `Save Comp Lead` (Output).
  - **Save Comp Lead**
    - Type: `n8n-nodes-base.airtable`
    - Role: Saves qualifying competitor lead into Airtable `LEADS`.
    - Configuration: Airtable create operation.
    - Input/Output: `Is Opportunity?` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Weekly Report**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Triggers weekly reporting every Sunday at 8:00 PM.
    - Configuration: Cron expression `0 20 * * 0`.
    - Input/Output: None (Input) / `Aggregate Pipeline` (Output).
  - **Aggregate Pipeline**
    - Type: `n8n-nodes-base.code`
    - Role: Aggregates pipeline metrics and report date ranges.
    - Configuration: JavaScript object aggregation.
    - Input/Output: `Weekly Report` (Input) / `AI Insights` (Output).
  - **AI Insights**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Generates weekly lead generation insights and predictions using OpenAI GPT-4o-mini.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Aggregate Pipeline` (Input) / `Post Report` (Output).
    - Credentials: OpenAI API Key.
  - **Post Report**
    - Type: `n8n-nodes-base.slack`
    - Role: Posts formatted weekly report to Slack `#lead-analytics`.
    - Configuration: Slack post message action.
    - Input/Output: `AI Insights` (Input) / `Save Report` (Output).
    - Credentials: Slack API.
  - **Save Report**
    - Type: `n8n-nodes-base.airtable`
    - Role: Saves weekly report content into Airtable `ANALYTICS`.
    - Configuration: Airtable create operation.
    - Input/Output: `Post Report` (Input) / None (Terminal).
    - Credentials: Airtable API token.

---

<h4>2.8 Shadow Mode & Global Error Handling</h4>
- **Overview:** Provides a parallel shadow mode validation layer for comparing AI drafts with human responses, and a global error handler that catches workflow exceptions, logs them to Airtable, and alerts Slack `#lead-alerts`.
- **Nodes Involved:** `Shadow Check`, `Log AI Draft`, `Shadow Notify`, `Compare AI vs Human`, `Save Shadow Score`, `Error Trigger`, `Format Error`, `Log Error`, `Is Critical?`, `Alert Team`.
- **Node Details:**
  - **Shadow Check**
    - Type: `n8n-nodes-base.code`
    - Role: Toggles shadow mode execution.
    - Configuration: JavaScript validation check.
    - Input/Output: External Draft Input (Input) / `Log AI Draft` (Output).
  - **Log AI Draft**
    - Type: `n8n-nodes-base.airtable`
    - Role: Logs AI-generated drafts into Airtable `SHADOW_MODE_LOG`.
    - Configuration: Airtable create operation.
    - Input/Output: `Shadow Check` (Input) / `Shadow Notify` (Output).
    - Credentials: Airtable API token.
  - **Shadow Notify**
    - Type: `n8n-nodes-base.slack`
    - Role: Notifies Slack `#shadow-mode` of AI drafts for manual comparison.
    - Configuration: Slack post message action.
    - Input/Output: `Log AI Draft` (Input) / `Compare AI vs Human` (Output).
    - Credentials: Slack API.
  - **Compare AI vs Human**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Compares AI draft and human response similarity using OpenAI GPT-4o-mini.
    - Configuration: POST request to OpenAI Chat Completions API.
    - Input/Output: `Shadow Notify` (Input) / `Save Shadow Score` (Output).
    - Credentials: OpenAI API Key.
  - **Save Shadow Score**
    - Type: `n8n-nodes-base.airtable`
    - Role: Updates shadow mode log with tone and overall match scores.
    - Configuration: Airtable update operation matching by ID.
    - Input/Output: `Compare AI vs Human` (Input) / None (Terminal).
    - Credentials: Airtable API token.
  - **Error Trigger**
    - Type: `n8n-nodes-base.errorTrigger`
    - Role: Global error trigger catching unhandled workflow exceptions.
    - Configuration: Default error trigger.
    - Input/Output: None (Input) / `Format Error` (Output).
  - **Format Error**
    - Type: `n8n-nodes-base.code`
    - Role: Formats error message, determines severity (CRITICAL, WARNING, INFO), and timestamp.
    - Configuration: JavaScript string analysis.
    - Input/Output: `Error Trigger` (Input) / `Log Error` (Output).
  - **Log Error**
    - Type: `n8n-nodes-base.airtable`
    - Role: Logs error details into Airtable `ERROR_LOGS`.
    - Configuration: Airtable create operation.
    - Input/Output: `Format Error` (Input) / `Is Critical?` (Output).
    - Credentials: Airtable API token.
  - **Is Critical?**
    - Type: `n8n-nodes-base.if`
    - Role: Evaluates whether error severity is CRITICAL.
    - Configuration: Condition checking severity string equality.
    - Input/Output: `Log Error` (Input) / `Alert Team` (Output).
  - **Alert Team**
    - Type: `n8n-nodes-base.slack`
    - Role: Sends error alerts to Slack `#lead-alerts`.
    - Configuration: Slack post message action.
    - Input/Output: `Is Critical?` (Input) / None (Terminal).
    - Credentials: Slack API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 📌 Branch 1 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 🔍 INTENT RADAR<br>### AI-Powered Buying Signal Detection<br><br>**Scans Reddit (via Google), Twitter & RSS every 15min for buying intent.**<br>No Reddit API = No ToS violation. Read-only monitoring.<br><br>`Keywords: "looking for", "need help with", "switching from", "alternative to"` |
| 📌 Branch 2 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 🧬 LEAD DNA — Deep Enrichment<br>### Transforms a username into a complete buyer profile<br><br>**Modular enrichment: works with or without paid APIs.**<br>Apollo.io → Email, Company, LinkedIn | Fallback → Google Search (free)<br>AI Profiler builds: Persona, Pain Points, Best Approach<br><br>`Score = Intent(40%) + Fit(30%) + Engagement(20%) + Timing(10%)` |
| 📌 Branch 3 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # ⚖️ MULTI-LLM CONSENSUS<br>### GPT-4o strategizes · Claude 3.5 writes · GPT-mini validates<br><br>**HOT leads (80+): Triple AI filter for surgical precision**<br>WARM leads: Single fast model (GPT-4o-mini)<br><br>`Quality Gate REJECTS: spammy, generic, too long, or AI-sounding messages` |
| 📌 Branch 4 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 🛑 HUMAN-IN-THE-LOOP<br>### Every message reviewed by a human before sending<br><br>**Slack #lead-review with Approve / Edit / Reject buttons**<br>AI assists, humans decide. Zero unsupervised auto-send.<br>Timeout: 4h for HOT, 24h for WARM → auto-archive (never auto-send) |
| 📌 Branch 5 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 📤 PREMIUM OUTREACH + CONVERSION FUNNEL<br>### Email auto-sends (after warm-up) · Reddit/LinkedIn = human-sent<br><br>**Warm emails only. Max 30/day. Random 45-120s delay. No tracking pixels.**<br>Conversion: Reply analysis → Calendly link → Demo booked 🎉<br><br>`Footer: "If this isn't relevant, let me know and I won't follow up."` |
| 📌 Branch 6 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 🔄 SMART NURTURE + 📉 WARMTH DECAY<br>### Scores decay 10%/day of inactivity. Treats leads like people.<br><br>**Day 0: 85 (HOT) → Day 3: 62 (WARM) → Day 7: 40 (COLD) → Day 14: auto-archive**<br>Re-engagement resets score. Email open slows decay. Reply = boost +20.<br>AI decides: WAIT / FOLLOW_UP / CHANGE_CHANNEL / ARCHIVE<br><br>`Not a dumb drip sequence — an adaptive system that respects buying cycles` |
| 📌 Branch 7 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 🛡️ COMPETITIVE INTELLIGENCE<br>### Monitors competitor complaints on Reddit<br><br>**Finds people ready to switch. Detects negative sentiment.**<br>High-opportunity leads auto-enter the enrichment pipeline. |
| 📌 Branch 8 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 📊 COMMAND CENTER<br>### Weekly AI-powered pipeline report<br><br>**Full visibility: leads/day, conversion rates, cost/lead, top keywords.**<br>AI generates insights + predictions. Slack #lead-analytics. |
| 📌 Branch 9 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 👻 SHADOW MODE — Validation Layer<br>### Runs in parallel without sending. Compares AI vs Human.<br><br>**2 weeks of silent validation before going live.**<br>AI drafts messages, human responds manually, GPT compares both.<br><br>`"Ran Shadow Mode on 50 leads. AI matched my responses at 92.4%."` |
| 📌 Branch 10 | n8n-nodes-base.stickyNote | Visual grouping | None | None | # ❌ GLOBAL ERROR HANDLER<br>### Catches all failures · Logs to Airtable · Alerts via Slack<br><br>**CRITICAL → pause workflow \| WARNING → alert + continue \| INFO → log only** |
| 📌 Results | n8n-nodes-base.stickyNote | Visual grouping | None | None | # 📊 LIVE RESULTS<br>━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━<br>Week 1: 52 detected → 14 enriched → 6 contacted → 2 replies → 1 demo<br>Week 2: 67 detected → 19 enriched → 8 contacted → 3 replies → 1 demo<br>Week 3: 71 detected → 22 enriched → 11 contacted → 4 replies → 2 demos<br>━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━<br>**Total: 190 leads \| 36% reply rate \| $0.43/lead \| 4 demos booked**<br><br>👻 Shadow Mode accuracy: **92.4%** match vs human responses<br>💰 Total cost: **$7-15/month**<br><br>_5 Quality Gates: Shadow Mode · Multi-LLM · HITL · Quality Gate · Warmth Decay_ |
| ⏰ Scan Every 15min | n8n-nodes-base.scheduleTrigger | Triggers workflow every 15 minutes | None | Load Config | |
| 📋 Load Config | n8n-nodes-base.code | Loads keywords and subreddits | Scan Every 15min | Reddit via Google, Twitter/X Search, RSS Feed Monitor | |
| 🔍 Reddit via Google | n8n-nodes-base.httpRequest | Searches Reddit via Google Custom Search | Load Config | Merge Sources | |
| 🐦 Twitter/X Search | n8n-nodes-base.httpRequest | Searches recent tweets | Load Config | Merge Sources | |
| 🌐 RSS Feed Monitor | n8n-nodes-base.rssFeedRead | Reads Google Alerts RSS feed | Load Config | Merge Sources | |
| 🔀 Merge Sources | n8n-nodes-base.merge | Merges multi-channel inputs | Reddit via Google, Twitter/X Search, RSS Feed Monitor | Normalize Data | |
| 🧹 Normalize Data | n8n-nodes-base.code | Normalizes posts into common format | Merge Sources | Check Duplicates | |
| 🚫 Check Duplicates | n8n-nodes-base.airtable | Checks Airtable for duplicate URLs | Normalize Data | Filter Known | |
| ✂️ Filter Known | n8n-nodes-base.code | Filters out known duplicate records | Check Duplicates | AI Intent Classifier | |
| 🧠 AI Intent Classifier | n8n-nodes-base.httpRequest | Scores intent via GPT-4o | Filter Known | Filter Low Intent | |
| ✂️ Filter Low Intent | n8n-nodes-base.code | Drops low-intent noise (<30 score) | AI Intent Classifier | Save New Lead, Track Detection | |
| 💾 Save New Lead | n8n-nodes-base.airtable | Saves new lead to Airtable LEADS | Filter Low Intent | Reddit Profile Search | |
| 📊 Track Detection | n8n-nodes-base.airtable | Tracks detection event metrics | Filter Low Intent | None | |
| 👤 Reddit Profile Search | n8n-nodes-base.httpRequest | Searches user profile via Google | Save New Lead | Has Apollo Key? | |
| 🔀 Has Apollo Key? | n8n-nodes-base.if | Checks for APOLLO_API_KEY | Reddit Profile Search | Apollo Enrichment, Google Fallback | |
| 🏢 Apollo Enrichment | n8n-nodes-base.httpRequest | Enriches lead via Apollo.io | Has Apollo Key? | Merge Enrichment | |
| 🔄 Google Fallback | n8n-nodes-base.httpRequest | Google search fallback enrichment | Has Apollo Key? | Merge Enrichment | |
| 🔀 Merge Enrichment | n8n-nodes-base.merge | Merges enrichment pathways | Apollo Enrichment, Google Fallback | AI Lead Profiler | |
| 🧠 AI Lead Profiler | n8n-nodes-base.httpRequest | Builds buyer profile via Claude 3.5 | Merge Enrichment | Calculate Score | |
| 📊 Calculate Score | n8n-nodes-base.code | Computes weighted total lead score | AI Lead Profiler | Save Enriched Lead | |
| 💾 Save Enriched Lead | n8n-nodes-base.airtable | Updates lead profile in Airtable | Calculate Score | Route by Score | |
| 🔀 Route by Score | n8n-nodes-base.switch | Routes by HOT, WARM, or COLD score | Save Enriched Lead | GPT-4o Strategy, Quick Draft (WARM), Wait (Do Nothing) | |
| 🧠 GPT-4o Strategy | n8n-nodes-base.httpRequest | Strategy generation via GPT-4o for HOT leads | Route by Score | Claude Message Craft | |
| ✍️ Claude Message Craft | n8n-nodes-base.httpRequest | Message crafting via Claude for HOT leads | GPT-4o Strategy | Quality Gate | |
| 🛡️ Quality Gate | n8n-nodes-base.httpRequest | Spam validation via GPT-4o-mini | Claude Message Craft | Merge Drafts | |
| 🧠 Quick Draft (WARM) | n8n-nodes-base.httpRequest | Quick draft generation for WARM leads | Route by Score | Merge Drafts | |
| 🔀 Merge Drafts | n8n-nodes-base.code | Merges and normalizes draft variants | Quality Gate, Quick Draft (WARM) | Save to Queue | |
| 💾 Save to Queue | n8n-nodes-base.airtable | Saves draft to Airtable OUTREACH_QUEUE | Merge Drafts | Format Slack Card | |
| 📋 Format Slack Card | n8n-nodes-base.code | Formats Slack review card markdown | Save to Queue | Send to #lead-review | |
| 💬 Send to #lead-review | n8n-nodes-base.slack | Posts review card to Slack #lead-review | Format Slack Card | Log Review Sent | |
| ⏰ Log Review Sent | n8n-nodes-base.code | Prepares review log payload | Send to #lead-review | Update Queue, Log Interaction | |
| 💾 Update Queue | n8n-nodes-base.airtable | Updates review status in Airtable | Log Review Sent | None | |
| 💾 Log Interaction | n8n-nodes-base.airtable | Logs review dispatch interaction | Log Review Sent | None | |
| 🔀 Route Channel | n8n-nodes-base.switch | Routes outreach by channel (email, reddit, linkedin) | Queue / Webhook Trigger | Random Delay, Reddit Manual, LinkedIn Manual | |
| ⏱️ Random Delay | n8n-nodes-base.code | Adds random delay (45-120s) | Route Channel | Send Email | |
| 📧 Send Email | n8n-nodes-base.gmail | Sends outreach email via Gmail | Random Delay | Log Email | |
| 💾 Log Email | n8n-nodes-base.airtable | Logs email interaction | Send Email | Schedule Follow-up | |
| 📅 Schedule Follow-up | n8n-nodes-base.code | Schedules follow-up action date | Log Email | Update Contacted | |
| 💾 Update Contacted | n8n-nodes-base.airtable | Updates lead stage to 'Contacted' | Schedule Follow-up | None | |
| 💬 Reddit Manual | n8n-nodes-base.slack | Posts Reddit manual outreach copy to Slack | Route Channel | None | |
| 🔗 LinkedIn Manual | n8n-nodes-base.slack | Posts LinkedIn manual outreach copy to Slack | Route Channel | None | |
| 🧠 Analyze Reply | n8n-nodes-base.code | Analyzes reply sentiment | Reply Trigger | Reply Sentiment | |
| 🔀 Reply Sentiment | n8n-nodes-base.switch | Routes by reply sentiment | Analyze Reply | Send Calendly, Blacklist, Forward Question | |
| 🎉 Send Calendly | n8n-nodes-base.slack | Sends Calendly link for positive replies | Reply Sentiment | None | |
| 🗑️ Blacklist | n8n-nodes-base.airtable | Blacklists unsubscribed leads | Reply Sentiment | None | |
| ❓ Forward Question | n8n-nodes-base.slack | Forwards prospect questions to Slack | Reply Sentiment | None | |
| ⏰ Daily Nurture | n8n-nodes-base.scheduleTrigger | Triggers nurture sweep daily at 8:00 AM | None | Get Active Leads | |
| 📋 Get Active Leads | n8n-nodes-base.airtable | Queries active leads from Airtable | Daily Nurture | Apply Warmth Decay | |
| 📉 Apply Warmth Decay | n8n-nodes-base.code | Calculates warmth decay scores | Get Active Leads | AI Next Action | |
| 🧠 AI Next Action | n8n-nodes-base.httpRequest | AI decides nurture action via GPT-4o-mini | Apply Warmth Decay | Parse Decision | |
| 🔀 Parse Decision | n8n-nodes-base.code | Parses AI nurture decision JSON | AI Next Action | Nurture Action | |
| 🔀 Nurture Action | n8n-nodes-base.switch | Routes by nurture decision | Parse Decision | Update Decayed, Archive Lead | |
| 💾 Update Decayed | n8n-nodes-base.airtable | Updates decayed scores in Airtable | Nurture Action | None | |
| 🗃️ Archive Lead | n8n-nodes-base.airtable | Archives lead in Airtable | Nurture Action | None | |
| ⏸️ Wait (Do Nothing) | n8n-nodes-base.noOp | No-op wait node | Route by Score / Nurture Action | None | |
| ⏰ Competitor Scan | n8n-nodes-base.scheduleTrigger | Triggers competitor scan every 2 hours | None | Load Competitors | |
| 📋 Load Competitors | n8n-nodes-base.airtable | Loads competitors from Airtable | Competitor Scan | Search Complaints | |
| 🔍 Search Complaints | n8n-nodes-base.httpRequest | Searches Reddit for competitor complaints via Google | Load Competitors | Comp Sentiment | |
| 🧠 Comp Sentiment | n8n-nodes-base.httpRequest | Analyzes competitor post sentiment via GPT-4o-mini | Search Complaints | Is Opportunity? | |
| 🔀 Is Opportunity? | n8n-nodes-base.if | Checks if opportunity score >= 60 | Comp Sentiment | Save Comp Lead | |
| 💾 Save Comp Lead | n8n-nodes-base.airtable | Saves qualifying competitor lead to Airtable | Is Opportunity? | None | |
| ⏰ Weekly Report | n8n-nodes-base.scheduleTrigger | Triggers weekly report Sundays at 8:00 PM | None | Aggregate Pipeline | |
| 📊 Aggregate Pipeline | n8n-nodes-base.code | Aggregates pipeline metrics | Weekly Report | AI Insights | |
| 🧠 AI Insights | n8n-nodes-base.httpRequest | Generates weekly insights via GPT-4o-mini | Aggregate Pipeline | Post Report | |
| 💬 Post Report | n8n-nodes-base.slack | Posts weekly report to Slack #lead-analytics | AI Insights | Save Report | |
| 💾 Save Report | n8n-nodes-base.airtable | Saves weekly report to Airtable ANALYTICS | Post Report | None | |
| 👻 Shadow Check | n8n-nodes-base.code | Toggles shadow mode execution | Draft Input | Log AI Draft | |
| 📝 Log AI Draft | n8n-nodes-base.airtable | Logs AI draft to SHADOW_MODE_LOG | Shadow Check | Shadow Notify | |
| 💬 Shadow Notify | n8n-nodes-base.slack | Notifies Slack #shadow-mode of AI draft | Log AI Draft | Compare AI vs Human | |
| 🧠 Compare AI vs Human | n8n-nodes-base.httpRequest | Compares AI draft vs human response via GPT-4o-mini | Shadow Notify | Save Shadow Score | |
| 💾 Save Shadow Score | n8n-nodes-base.airtable | Updates shadow mode log with match scores | Compare AI vs Human | None | |
| 🚨 Error Trigger | n8n-nodes-base.errorTrigger | Triggers on workflow errors | None | Format Error | |
| 📋 Format Error | n8n-nodes-base.code | Formats error message and severity | Error Trigger | Log Error | |
| 💾 Log Error | n8n-nodes-base.airtable | Logs error to Airtable ERROR_LOGS | Format Error | Is Critical? | |
| 🔀 Is Critical? | n8n-nodes-base.if | Checks if error severity is CRITICAL | Log Error | Alert Team | |
| 💬 Alert Team | n8n-nodes-base.slack | Alerts Slack #lead-alerts of errors | Is Critical? | None | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Airtable Base Setup
1. Create an Airtable base with the following seven tables:
   - `LEADS` (Fields: `stage`, `source`, `username`, `ai_reason`, `source_url`, `detected_at`, `intent_score`, `classification`, `source_content`, `persona`, `fit_score`, `pain_points`, `total_score`, `best_channel`, `last_action`, `next_action`, `last_action_date`, `next_action_date`, `email_opened`, `follow_ups_sent`)
   - `OUTREACH_QUEUE` (Fields: `lead`, `status`, `channel`, `lead_score`, `message_draft`, `classification`, `message_variant_b`)
   - `INTERACTIONS` (Fields: `lead`, `type`, `channel`, `content`, `direction`, `status`)
   - `ANALYTICS` (Fields: `date`, `event`, `score`, `source`, `content`)
   - `COMPETITORS` (Fields: `name`, `url`)
   - `SHADOW_MODE_LOG` (Fields: `lead`, `ai_draft`, `ai_channel`, `lead_score`, `tone_match`, `overall_match`)
   - `ERROR_LOGS` (Fields: `severity`, `node_name`, `error_message`)

#### Step 2: Configure Credentials
Set up the following credentials in n8n:
- **Airtable Token API** (`airtableTokenApi`)
- **OpenAI API** (`openAiApi` / Bearer token `YOUR_TOKEN_HERE`)
- **Anthropic API** (`x-api-key` header with `YOUR_ANTHROPIC_KEY`)
- **Slack API** (`slackApi`)
- **Gmail OAuth2** (`gmailOAuth2`)
- **Twitter/X Bearer Token** (Header configuration)

#### Step 3: Build Branch 1 (Intent Radar)
1. Create a `Schedule Trigger` node named `⏰ Scan Every 15min` (Interval: 15 minutes).
2. Create a `Code` node named `📋 Load Config` returning subreddits and intent keywords.
3. Create three HTTP Request nodes: `🔍 Reddit via Google` (Custom Search API), `🐦 Twitter/X Search` (Twitter API v2), and `🌐 RSS Feed Monitor` (Google Alerts RSS URL).
4. Connect `📋 Load Config` to all three search nodes.
5. Create a `Merge` node (`🔀 Merge Sources`, Append mode) and connect all three search nodes to it.
6. Create a `Code` node `🧹 Normalize Data` to clean and structure post items.
7. Create an Airtable node `🚫 Check Duplicates` (Operation: Search, filter formula `=FIND("{{ $json.url }}",{source_url})`).
8. Create a `Code` node `✂️ Filter Known` to drop existing duplicates.
9. Create an HTTP Request node `🧠 AI Intent Classifier` (POST to OpenAI GPT-4o with JSON object response format).
10. Create a `Code` node `✂️ Filter Low Intent` to drop scores <30.
11. Create two Airtable nodes connected to `✂️ Filter Low Intent`: `💾 Save New Lead` (Create operation, stage 'New') and `📊 Track Detection` (Create in ANALYTICS).

#### Step 4: Build Branch 2 (Lead DNA & Enrichment)
1. Create an HTTP Request node `👤 Reddit Profile Search` (Google Custom Search API).
2. Create an `If` node `🔀 Has Apollo Key?` checking `={{ $env.APOLLO_API_KEY !== undefined }}`.
3. Create HTTP Request node `🏢 Apollo Enrichment` (Apollo people match API) connected to the True branch.
4. Create HTTP Request node `🔄 Google Fallback` (Google Custom Search API) connected to the False branch.
5. Create a `Merge` node `🔀 Merge Enrichment` (Append mode) combining both enrichment paths.
6. Create HTTP Request node `🧠 AI Lead Profiler` (POST to Anthropic Claude 3.5 Sonnet).
7. Create a `Code` node `📊 Calculate Score` applying weighted scoring (40% intent, 30% fit, 20% engagement, 10% timing).
8. Create an Airtable node `💾 Save Enriched Lead` (Update operation on `LEADS`).
9. Create a `Switch` node `🔀 Route by Score` routing by classification (`HOT`, `WARM`, `COLD`).

#### Step 5: Build Branch 3 (Multi-LLM Consensus & Drafting)
1. From HOT output of `🔀 Route by Score`, create HTTP Request node `🧠 GPT-4o Strategy` (POST to OpenAI GPT-4o).
2. Connect to HTTP Request node `✍️ Claude Message Craft` (POST to Anthropic Claude 3.5 Sonnet).
3. Connect to HTTP Request node `🛡️ Quality Gate` (POST to OpenAI GPT-4o-mini).
4. From WARM output of `🔀 Route by Score`, create HTTP Request node `🧠 Quick Draft (WARM)` (POST to OpenAI GPT-4o-mini).
5. Create a `Code` node `🔀 Merge Drafts` combining outputs from `🛡️ Quality Gate` and `🧠 Quick Draft (WARM)`.
6. Create an Airtable node `💾 Save to Queue` (Create operation on `OUTREACH_QUEUE`).

#### Step 6: Build Branch 4 (Human-in-the-Loop Review)
1. Create a `Code` node `📋 Format Slack Card` formatting markdown.
2. Create a Slack node `💬 Send to #lead-review` targeting channel `#lead-review`.
3. Create a `Code` node `⏰ Log Review Sent`.
4. Create two Airtable nodes: `💾 Update Queue` (Update operation) and `💾 Log Interaction` (Create operation in `INTERACTIONS`).

#### Step 7: Build Branch 5 (Premium Outreach & Conversion Funnel)
1. Create a `Switch` node `🔀 Route Channel` matching channel values (`email`, `reddit`, `linkedin`).
2. For Email path: Create `Code` node `⏱️ Random Delay` (45-120s timeout), Gmail node `📧 Send Email`, Airtable node `💾 Log Email`, `Code` node `📅 Schedule Follow-up`, and Airtable node `💾 Update Contacted`.
3. For Reddit/LinkedIn paths: Create Slack nodes `💬 Reddit Manual` and `🔗 LinkedIn Manual` targeting `#outreach-ready`.
4. For Reply inbound path: Create `Code` node `Analyze Reply`, `Switch` node `🔀 Reply Sentiment`, Slack node `🎉 Send Calendly`, Airtable node `🗑️ Blacklist`, and Slack node `❓ Forward Question`.

#### Step 8: Build Branch 6 (Smart Nurture & Warmth Decay)
1. Create a `Schedule Trigger` node `⏰ Daily Nurture` set to cron expression `0 8 * * *`.
2. Create Airtable node `📋 Get Active Leads` searching active stages.
3. Create `Code` node `📉 Apply Warmth Decay`.
4. Create HTTP Request node `🧠 AI Next Action` (POST to OpenAI GPT-4o-mini).
5. Create `Code` node `🔀 Parse Decision`.
6. Create `Switch` node `🔀 Nurture Action` routing to Airtable nodes `💾 Update Decayed` and `🗃️ Archive Lead`, or No-Op node `⏸️ Wait (Do Nothing)`.

#### Step 9: Build Branch 7 & 8 (Competitive Intelligence & Command Center)
1. Competitor Scan: Create `Schedule Trigger` `⏰ Competitor Scan` (Cron `0 */2 * * *`), Airtable node `📋 Load Competitors`, HTTP Request node `🔍 Search Complaints`, HTTP Request node `🧠 Comp Sentiment`, `If` node `🔀 Is Opportunity?`, and Airtable node `💾 Save Comp Lead`.
2. Command Center: Create `Schedule Trigger` `⏰ Weekly Report` (Cron `0 20 * * 0`), `Code` node `📊 Aggregate Pipeline`, HTTP Request node `🧠 AI Insights`, Slack node `💬 Post Report` (`#lead-analytics`), and Airtable node `💾 Save Report` (`ANALYTICS`).

#### Step 10: Build Branch 9 & 10 (Shadow Mode & Error Handling)
1. Shadow Mode: Create `Code` node `👻 Shadow Check`, Airtable node `📝 Log AI Draft` (`SHADOW_MODE_LOG`), Slack node `💬 Shadow Notify` (`#shadow-mode`), HTTP Request node `🧠 Compare AI vs Human`, and Airtable node `💾 Save Shadow Score`.
2. Error Handler: Create `Error Trigger` node `🚨 Error Trigger`, `Code` node `📋 Format Error`, Airtable node `💾 Log Error` (`ERROR_LOGS`), `If` node `🔀 Is Critical?`, and Slack node `💬 Alert Team` (`#lead-alerts`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **No Reddit API to avoid ToS violations** | Read-only monitoring via Google Custom Search |
| **Warm emails rate-limited** | Maximum 30 emails per day with randomized 45-120 second delays |
| **Shadow Mode validation period** | Run in shadow mode for 2 weeks before enabling automated outreach |