Score LinkedIn post engagers as leads with Periodix Actions, Claude, and Google Sheets

https://n8nworkflows.xyz/workflows/score-linkedin-post-engagers-as-leads-with-periodix-actions--claude--and-google-sheets-19840


# Score LinkedIn post engagers as leads with Periodix Actions, Claude, and Google Sheets

### 1. Workflow Overview

This workflow automates the process of finding relevant LinkedIn discussions by topic, extracting users who engaged with those posts (via reactions and comments), filtering them against an Ideal Customer Profile (ICP), scoring them with AI, generating personalized connection notes, and logging the results into Google Sheets.

The architecture is organized into the following logical blocks:
- **1.1 Trigger & Configuration:** Initializes the automated run on a daily schedule and defines campaign settings such as topic keywords, ICP criteria, offers, and result limits.
- **1.2 Post Discovery & Extraction:** Searches LinkedIn for relevant posts based on topic keywords and extracts unique post identifiers and text summaries.
- **1.3 Engagement Extraction & Normalization:** Iterates through each discovered post to collect reactions and comments, standardizing the data format across sources.
- **1.4 Filtering & AI Processing:** Merges, deduplicates, and pre-filters engagers by ICP keywords, evaluates them using an external AI service (Claude), and parses the structured output.
- **1.5 Data Routing & Storage:** Separates leads based on their AI score threshold and saves qualified prospects ("Hot") and unqualified prospects ("Cold") into distinct Google Sheets tabs.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
- **Overview:** Sets up the execution schedule and centralizes all campaign variables required for LinkedIn searches, ICP matching, and score thresholds.
- **Nodes Involved:** `Run Daily`, `Campaign Config`
- **Node Details:**
  - **Run Daily**
    - *Type and Technical Role:* Schedule Trigger node (`n8n-nodes-base.scheduleTrigger`) initiating the workflow automatically.
    - *Configuration Choices:* Configured to trigger daily at hour 09:00.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects to `Campaign Config`.
    - *Edge Cases / Potential Failures:* None.
  - **Campaign Config**
    - *Type and Technical Role:* Set node (`n8n-nodes-base.set`) acting as a central configuration repository.
    - *Configuration Choices:* Assigns parameters including `topic_keywords`, `icp_keywords`, `icp_description`, `offer`, `score_threshold`, `posts_limit`, `reactions_limit`, and `comments_limit`.
    - *Key Expressions or Variables:* Static string and number assignments.
    - *Input/Output Connections:* Input from `Run Daily`; output connects to `Find Topic Posts`.
    - *Edge Cases / Potential Failures:* Missing or misspelled parameter keys can cause downstream expression evaluation errors.

#### 2.2 Post Discovery & Extraction
- **Overview:** Searches LinkedIn via the Periodix Actions integration to find posts matching specified topic keywords and standardizes the retrieved post records.
- **Nodes Involved:** `Find Topic Posts`, `Extract Posts`
- **Node Details:**
  - **Find Topic Posts**
    - *Type and Technical Role:* Periodix Actions LinkedIn node (`@periodix/n8n-nodes-actions.linkedIn`) for searching content.
    - *Configuration Choices:* Uses resource `search`, constructs a URL via `encodeURIComponent` using `topic_keywords`, and sets limits based on configuration.
    - *Key Expressions or Variables:* `={{ $("Campaign Config").first().json.posts_limit }}` and `={{ "https://www.linkedin.com/search/results/content/?keywords=" + encodeURIComponent($("Campaign Config").first().json.topic_keywords) }}`
    - *Input/Output Connections:* Input from `Campaign Config`; output connects to `Extract Posts`.
    - *Credentials:* Periodix Actions account (`periodixActionsApi`).
    - *Edge Cases / Potential Failures:* API authentication failures, rate limits, or invalid profile selections.
  - **Extract Posts**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`) parsing raw search results.
    - *Configuration Choices:* Iterates through items, filters out duplicate post IDs, and truncates post text to 500 characters.
    - *Key Expressions or Variables:* Custom JavaScript iterating over `$input.all()`.
    - *Input/Output Connections:* Input from `Find Topic Posts`; output connects to `Loop Posts`.
    - *Edge Cases / Potential Failures:* Malformed JSON payloads from the LinkedIn search API.

#### 2.3 Engagement Extraction & Normalization
- **Overview:** Loops through each extracted post to retrieve both reactions and comments in parallel, normalizing the user profiles into a uniform structure.
- **Nodes Involved:** `Loop Posts`, `Current Post`, `Get Post Reactions`, `Normalize Reactions`, `Get Post Comments`, `Normalize Comments`, `Merge Post Engagers`, `Collect All Engagers`
- **Node Details:**
  - **Loop Posts**
    - *Type and Technical Role:* Split In Batches node (`n8n-nodes-base.splitInBatches`) for handling posts sequentially.
    - *Configuration Choices:* Processes items in batches.
    - *Input/Output Connections:* Input from `Extract Posts` and `Merge Post Engagers`; outputs connect to `Current Post` and `Collect All Engagers`.
  - **Current Post**
    - *Type and Technical Role:* NoOp node (`n8n-nodes-base.noOp`) acting as a distribution anchor for the active post.
    - *Input/Output Connections:* Input from `Loop Posts`; outputs connect to `Get Post Reactions` and `Get Post Comments`.
  - **Get Post Reactions**
    - *Type and Technical Role:* Periodix Actions LinkedIn node (`@periodix/n8n-nodes-actions.linkedIn`).
    - *Configuration Choices:* Resource `post`, operation `getReactions`, configured with user limits. Error handling set to continue regular output on failure.
    - *Key Expressions or Variables:* `={{ $("Campaign Config").first().json.reactions_limit }}` and `={{ $json.post_id }}`
    - *Credentials:* Periodix Actions account (`periodixActionsApi`).
    - *Input/Output Connections:* Input from `Current Post`; output connects to `Normalize Reactions`.
  - **Normalize Reactions**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`).
    - *Configuration Choices:* Maps reaction payloads into standard schema fields (`source: 'reaction'`, name, headline, company, provider_id, profile_url, matched_post, post_url).
    - *Input/Output Connections:* Input from `Get Post Reactions`; output connects to `Merge Post Engagers` (Input 1).
  - **Get Post Comments**
    - *Type and Technical Role:* Periodix Actions LinkedIn node (`@periodix/n8n-nodes-actions.linkedIn`).
    - *Configuration Choices:* Resource `post`, operation `getComments`, configured with user limits. Error handling set to continue regular output on failure.
    - *Key Expressions or Variables:* `={{ $("Campaign Config").first().json.comments_limit }}` and `={{ $json.post_id }}`
    - *Credentials:* Periodix Actions account (`periodixActionsApi`).
    - *Input/Output Connections:* Input from `Current Post`; output connects to `Normalize Comments`.
  - **Normalize Comments**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`).
    - *Configuration Choices:* Maps comment payloads into standard schema fields (`source: 'comment'`, handling nested author objects).
    - *Input/Output Connections:* Input from `Get Post Comments`; output connects to `Merge Post Engagers` (Input 2).
  - **Merge Post Engagers**
    - *Type and Technical Role:* Merge node (`n8n-nodes-base.merge`).
    - *Configuration Choices:* Combines normalized reactions and comments streams.
    - *Input/Output Connections:* Inputs from `Normalize Reactions` and `Normalize Comments`; output loops back to `Loop Posts`.
  - **Collect All Engagers**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`).
    - *Configuration Choices:* Gathers all merged engager items once the loop completes.
    - *Key Expressions or Variables:* `={{ $('Merge Post Engagers').all() }}`
    - *Input/Output Connections:* Input from `Loop Posts`; output connects to `Dedupe and Pre-filter by ICP`.

#### 2.4 Filtering & AI Processing
- **Overview:** Deduplicates engagers across posts, applies keyword pre-filtering to save API costs, scores prospects via Claude AI, and parses the returned JSON.
- **Nodes Involved:** `Dedupe and Pre-filter by ICP`, `Score and Draft with AI`, `Parse AI Output`
- **Node Details:**
  - **Dedupe and Pre-filter by ICP**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`).
    - *Configuration Choices:* Removes duplicate profiles based on `provider_id` or `profile_url`, and checks headline/company against comma-separated ICP keywords.
    - *Input/Output Connections:* Input from `Collect All Engagers`; output connects to `Score and Draft with AI`.
  - **Score and Draft with AI**
    - *Type and Technical Role:* HTTP Request node (`n8n-nodes-base.httpRequest`).
    - *Configuration Choices:* POST request to Anthropic Messages API (`https://api.anthropic.com/v1/messages`) using model `claude-haiku-4-5-20251001`. Configured with request batching (batch size 1, 1500ms interval), retry on fail, and error handling set to continue regular output.
    - *Key Expressions or Variables:* Dynamic JSON payload incorporating profile details, matched post content, ICP description, and campaign offer.
    - *Credentials:* Anthropic API credential (`anthropicApi`).
    - *Input/Output Connections:* Input from `Dedupe and Pre-filter by ICP`; output connects to `Parse AI Output`.
    - *Edge Cases / Potential Failures:* API rate limits, invalid JSON responses, or authentication errors.
  - **Parse AI Output**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`).
    - *Configuration Choices:* Executes once per item (`runOnceForEachItem`), stripping markdown code blocks from the AI response and parsing JSON into score, reason, and connection note attributes.
    - *Input/Output Connections:* Input from `Score and Draft with AI`; output connects to `Hot Lead`.

#### 2.5 Data Routing & Storage
- **Overview:** Evaluates the AI score against the configured threshold and routes leads to either the "Hot" or "Cold" Google Sheets destination.
- **Nodes Involved:** `Hot Lead`, `Save Hot Lead`, `Save Cold Lead`
- **Node Details:**
  - **Hot Lead**
    - *Type and Technical Role:* If node (`n8n-nodes-base.if`).
    - *Configuration Choices:* Checks if `$json.score` is greater than or equal to `score_threshold`.
    - *Key Expressions or Variables:* `={{ $json.score }}` vs `={{ $('Campaign Config').first().json.score_threshold }}`
    - *Input/Output Connections:* Input from `Parse AI Output`; true branch connects to `Save Hot Lead`, false branch connects to `Save Cold Lead`.
  - **Save Hot Lead**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`).
    - *Configuration Choices:* Operation `append`, appending row data to the sheet named `Hot`.
    - *Key Expressions or Variables:* Mapped fields for name, score, reason, source, company, headline, post URL, profile URL, provider ID, matched post, and connection note.
    - *Credentials:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
    - *Input/Output Connections:* Input from `Hot Lead` (true branch).
  - **Save Cold Lead**
    - *Type and Technical Role:* Google Sheets node (`n8n-nodes-base.googleSheets`).
    - *Configuration Choices:* Operation `append`, appending row data to the sheet named `Cold`.
    - *Key Expressions or Variables:* Mapped fields for name, score, reason, source, company, headline, profile URL, and matched post.
    - *Credentials:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
    - *Input/Output Connections:* Input from `Hot Lead` (false branch).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Daily | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 09:00 | None | Campaign Config | # Find LinkedIn posts by topic and turn their comments and likes into leads with Periodix Actions and AI... |
| Campaign Config | n8n-nodes-base.set | Defines campaign parameters and limits | Run Daily | Find Topic Posts | ## 1. Configure once<br>Set topic keywords, ICP keywords, description, offer, and score threshold in Campaign Config. |
| Find Topic Posts | @periodix/n8n-nodes-actions.linkedIn | Searches LinkedIn posts by topic | Campaign Config | Extract Posts | ## 2. Find matching posts, not just brand mentions<br>Periodix Actions searches LinkedIn for posts on any topic you choose, across many posts at once, no manual post ID needed. |
| Extract Posts | n8n-nodes-base.code | Normalizes and filters unique post IDs | Find Topic Posts | Loop Posts | ## 2. Find matching posts, not just brand mentions<br>Periodix Actions searches LinkedIn for posts on any topic you choose, across many posts at once, no manual post ID needed. |
| Loop Posts | n8n-nodes-base.splitInBatches | Iterates through posts | Extract Posts, Merge Post Engagers | Collect All Engagers, Current Post | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Current Post | n8n-nodes-base.noOp | Distributes current post context | Loop Posts | Get Post Reactions, Get Post Comments | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Get Post Reactions | @periodix/n8n-nodes-actions.linkedIn | Fetches reactions for a post | Current Post | Normalize Reactions | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Normalize Reactions | n8n-nodes-base.code | Normalizes reaction user payloads | Get Post Reactions | Merge Post Engagers | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Get Post Comments | @periodix/n8n-nodes-actions.linkedIn | Fetches comments for a post | Current Post | Normalize Comments | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Normalize Comments | n8n-nodes-base.code | Normalizes comment user payloads | Get Post Comments | Merge Post Engagers | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Merge Post Engagers | n8n-nodes-base.merge | Merges reactions and comments streams | Normalize Reactions, Normalize Comments | Loop Posts | ## 3. Pull engagement per post<br>For each matching post, Periodix Actions fetches reactions and comments, no cookies or scraping. |
| Collect All Engagers | n8n-nodes-base.code | Gathers all merged engagers after loop | Loop Posts | Dedupe and Pre-filter by ICP | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Dedupe and Pre-filter by ICP | n8n-nodes-base.code | Deduplicates and pre-filters by keywords | Collect All Engagers | Score and Draft with AI | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Score and Draft with AI | n8n-nodes-base.httpRequest | Scores fit and drafts note via Claude | Dedupe and Pre-filter by ICP | Parse AI Output | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Parse AI Output | n8n-nodes-base.code | Parses AI JSON output | Score and Draft with AI | Hot Lead | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Hot Lead | n8n-nodes-base.if | Routes leads based on score threshold | Parse AI Output | Save Hot Lead, Save Cold Lead | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Save Hot Lead | n8n-nodes-base.googleSheets | Logs qualified leads to Hot tab | Hot Lead | None | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |
| Save Cold Lead | n8n-nodes-base.googleSheets | Logs unqualified leads to Cold tab | Hot Lead | None | ## 4. Score, personalize, and save<br>AI scores each engager and drafts a note referencing the actual post. Hot and Cold leads are logged separately. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node**
   - Type: Schedule Trigger (`n8n-nodes-base.scheduleTrigger`)
   - Name: `Run Daily`
   - Parameters: Set rule interval to trigger at hour `9`.
2. **Create the Configuration Node**
   - Type: Set (`n8n-nodes-base.set`)
   - Name: `Campaign Config`
   - Parameters: Add assignments for `topic_keywords` (string), `icp_keywords` (string), `icp_description` (string), `offer` (string), `score_threshold` (number), `posts_limit` (number), `reactions_limit` (number), and `comments_limit` (number).
   - Connection: Connect `Run Daily` output to `Campaign Config`.
3. **Create the LinkedIn Search Node**
   - Type: Periodix Actions LinkedIn (`@periodix/n8n-nodes-actions.linkedIn`)
   - Name: `Find Topic Posts`
   - Credentials: Configure Periodix Actions account.
   - Parameters: Set resource to `search`, select your connected LinkedIn profile, set limit to `={{ $("Campaign Config").first().json.posts_limit }}`, and search URL to `={{ "https://www.linkedin.com/search/results/content/?keywords=" + encodeURIComponent($("Campaign Config").first().json.topic_keywords) }}`.
   - Connection: Connect `Campaign Config` output to `Find Topic Posts`.
4. **Create the Post Extraction Code Node**
   - Type: Code (`n8n-nodes-base.code`)
   - Name: `Extract Posts`
   - Parameters: Paste JavaScript to filter unique post IDs and truncate post text.
   - Connection: Connect `Find Topic Posts` output to `Extract Posts`.
5. **Create the Loop Node**
   - Type: Split In Batches (`n8n-nodes-base.splitInBatches`)
   - Name: `Loop Posts`
   - Connection: Connect `Extract Posts` output to `Loop Posts`.
6. **Create the Distribution Anchor Node**
   - Type: NoOp (`n8n-nodes-base.noOp`)
   - Name: `Current Post`
   - Connection: Connect output 1 of `Loop Posts` to `Current Post`.
7. **Create Reaction Extraction Nodes**
   - *Get Post Reactions:* Periodix Actions LinkedIn node (`@periodix/n8n-nodes-actions.linkedIn`). Resource `post`, operation `getReactions`, profile ID, limit `={{ $("Campaign Config").first().json.reactions_limit }}`, post ID `={{ $json.post_id }}`. Enable "On Error: Continue Regular Output". Connect `Current Post` output to it.
   - *Normalize Reactions:* Code node (`n8n-nodes-base.code`). Map reaction data fields. Connect `Get Post Reactions` output to it.
8. **Create Comment Extraction Nodes**
   - *Get Post Comments:* Periodix Actions LinkedIn node (`@periodix/n8n-nodes-actions.linkedIn`). Resource `post`, operation `getComments`, profile ID, limit `={{ $("Campaign Config").first().json.comments_limit }}`, post ID `={{ $json.post_id }}`. Enable "On Error: Continue Regular Output". Connect `Current Post` output to it.
   - *Normalize Comments:* Code node (`n8n-nodes-base.code`). Map comment author and data fields. Connect `Get Post Comments` output to it.
9. **Merge Engagers and Loop Back**
   - Type: Merge (`n8n-nodes-base.merge`)
   - Name: `Merge Post Engagers`
   - Connections: Input 1 from `Normalize Reactions`, Input 2 from `Normalize Comments`. Connect output to `Loop Posts` to complete the loop cycle.
10. **Collect All Engagers Node**
    - Type: Code (`n8n-nodes-base.code`)
    - Name: `Collect All Engagers`
    - Parameters: Return `={{ $('Merge Post Engagers').all() }}`.
    - Connection: Connect output 2 of `Loop Posts` to `Collect All Engagers`.
11. **Create Deduplication & Pre-Filter Node**
    - Type: Code (`n8n-nodes-base.code`)
    - Name: `Dedupe and Pre-filter by ICP`
    - Parameters: Add JavaScript to deduplicate profiles and filter by ICP keywords.
    - Connection: Connect `Collect All Engagers` output to `Dedupe and Pre-filter by ICP`.
12. **Create AI Scoring & Drafting Node**
    - Type: HTTP Request (`n8n-nodes-base.httpRequest`)
    - Name: `Score and Draft with AI`
    - Credentials: Configure Anthropic API key.
    - Parameters: Method POST, URL `https://api.anthropic.com/v1/messages`, headers (`anthropic-version: 2023-06-01`, `content-type: application/json`), batching enabled (size 1, interval 1500ms), retry on fail enabled, error handling set to continue regular output. JSON body containing the prompt referencing profile details, ICP description, offer, and matched post.
    - Connection: Connect `Dedupe and Pre-filter by ICP` output to `Score and Draft with AI`.
13. **Create AI Output Parser Node**
    - Type: Code (`n8n-nodes-base.code`)
    - Name: `Parse AI Output`
    - Parameters: Mode set to run once for each item, parsing raw JSON text from Claude.
    - Connection: Connect `Score and Draft with AI` output to `Parse AI Output`.
14. **Create Conditional Routing Node**
    - Type: If (`n8n-nodes-base.if`)
    - Name: `Hot Lead`
    - Parameters: Condition checking if score is greater than or equal to `={{ $('Campaign Config').first().json.score_threshold }}`.
    - Connection: Connect `Parse AI Output` output to `Hot Lead`.
15. **Create Storage Nodes**
    - *Save Hot Lead:* Google Sheets node (`n8n-nodes-base.googleSheets`). Credentials configured via Google Sheets OAuth2 API. Operation `append`, sheet name `Hot`. Map all profile and lead parameters. Connect True output of `Hot Lead` to this node.
    - *Save Cold Lead:* Google Sheets node (`n8n-nodes-base.googleSheets`). Credentials configured via Google Sheets OAuth2 API. Operation `append`, sheet name `Cold`. Map core profile and score parameters. Connect False output of `Hot Lead` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Periodix Actions Integration | Built with Periodix Actions (rated 4.9/5 on G2). Free trial available at [actions.periodix.net](https://actions.periodix.net). |
| Platform Compatibility | Works on both n8n Cloud and self-hosted instances. |
| Cost & Efficiency | Provides direct LinkedIn infrastructure access without browser scrapers, CAPTCHAs, or scraper fingerprints. |