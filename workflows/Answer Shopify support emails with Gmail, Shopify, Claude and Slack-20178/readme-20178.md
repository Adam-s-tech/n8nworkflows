Answer Shopify support emails with Gmail, Shopify, Claude and Slack

https://n8nworkflows.xyz/workflows/answer-shopify-support-emails-with-gmail--shopify--claude-and-slack-20178


# Answer Shopify support emails with Gmail, Shopify, Claude and Slack

### 1. Workflow Overview

This workflow automates the handling of incoming customer support emails for a Shopify store by combining Gmail, Shopify's GraphQL API, Anthropic's Claude AI, and Slack. Its primary purpose is to draft context-aware, accurate customer support responses derived directly from real order data and store policies while ensuring that sensitive actions (such as refunds, cancellations, or uncertain inquiries) are strictly gated for human review. 

The logic is grouped into three core functional blocks:
- **1.1 Input Reception & Normalization:** Monitors the inbox for unread support emails, injects centralized store configurations, extracts and normalizes sender metadata, and parses order references.
- **1.2 Data Lookup & AI Processing:** Authenticates with Shopify, queries the Admin GraphQL API for order details, formats the records into text briefs, and queries Claude using structured JSON schema constraints to classify intent and generate a response draft.
- **1.3 Routing, Decision & Action:** Evaluates the AI's output and safety parameters via code rules to route emails into four paths: automatic sending via Gmail, saving a review draft in Gmail with a Slack notification, escalating directly to Slack, or ignoring spam.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization

##### Overview
This block captures incoming unread customer emails from Gmail, injects store-wide configuration variables, cleans and normalizes message bodies, and extracts embedded order numbers or falls back to sender email searches.

##### Nodes Involved
- `New support email`
- `Settings`
- `Normalise the email`
- `Find the order reference`

##### Node Details

###### New support email
- **Type and Technical Role:** `n8n-nodes-base.gmailTrigger` (v1.3) — Triggers execution when new unread emails arrive in the Gmail inbox.
- **Configuration Choices:** Polling interval set to every minute; filters for `INBOX` label and `unread` status. `simple` option set to `false` to capture raw payload fields.
- **Key Expressions / Variables:** None.
- **Input / Output:** Input: None (Trigger). Output: Raw Gmail message object.
- **Version-Specific Requirements:** v1.3.
- **Edge Cases / Potential Failure Types:** Authentication token expiration or revocation; API rate limits if traffic spikes.

###### Settings
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.4) — Acts as a centralized configuration store for operational parameters.
- **Configuration Choices:** Assigns static variables: `storeName`, `shopSubdomain`, `returnWindowDays`, `shippingPromise`, `brandVoice`, `autoSendIntents`, `confidenceBar`, `slackChannel`, `signature`, and `disclosureLine`.
- **Key Expressions / Variables:** None (static property assignments).
- **Input / Output:** Input: `New support email`. Output: Original items combined with assigned settings properties.
- **Version-Specific Requirements:** v3.4.
- **Edge Cases / Potential Failure Types:** Typographical errors in subdomain or channel names will cause downstream API integration failures.

###### Normalise the email
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.4) — Cleans up raw email fields, isolates sender details, and strips quoted reply histories from the text body.
- **Configuration Choices:** Maps and transforms fields to establish standard data formats.
- **Key Expressions / Variables:** Uses ternary and regular expression splitting logic to sanitize email addresses (`fromAddress`), extract first names (`fromName`), trim subjects, truncate message text to 4000 characters, evaluate `isReply`, and map identifiers (`gmailMessageId`, `threadId`, `receivedAt`).
- **Input / Output:** Input: `Settings`. Output: Sanitized and structured email object.
- **Version-Specific Requirements:** v3.4.
- **Edge Cases / Potential Failure Types:** Malformed email headers lacking standard `from` objects may return empty strings, requiring downstream fallback logic.

###### Find the order reference
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript execution node that parses subject lines and body text for order references.
- **Configuration Choices:** Scans message text for patterns matching `#123` or order reference keywords.
- **Key Expressions / Variables:** Uses RegEx matching to extract 3 to 7-digit order strings and builds Shopify search queries (`name:#XXXX` or fallback `email:"address"`).
- **Input / Output:** Input: `Normalise the email`. Output: Items augmented with `orderName`, `searchQuery`, and `lookupBy`.
- **Version-Specific Requirements:** v2.
- **Edge Cases / Potential Failure Types:** Ambiguous numbers matching the pattern without being order IDs may cause incorrect lookups (mitigated by subsequent email-matching validations).

---

#### Block 1.2: Data Lookup & AI Processing

##### Overview
This block obtains a secure access token from Shopify, executes a GraphQL query to fetch relevant order details, compiles a structured plain-text data brief, and prompts Claude via the Anthropic Messages API using strict JSON schema validation.

##### Nodes Involved
- `Get a Shopify token`
- `Look up the order in Shopify`
- `Prepare the brief`
- `Classify and draft with Claude`

##### Node Details

###### Get a Shopify token
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Requests a short-lived OAuth access token from Shopify.
- **Configuration Choices:** POST method to `https://{{ $('Settings').first().json.shopSubdomain }}.myshopify.com/admin/oauth/access_token`. Uses `client_credentials` grant type via `httpCustomAuth`. Configured with `onError: continueRegularOutput`, `retryOnFail: true`, and 3 max retries.
- **Key Expressions / Variables:** Dynamically targets the shop subdomain from the Settings node.
- **Input / Output:** Input: `Find the order reference`. Output: JSON object containing `access_token`.
- **Version-Specific Requirements:** v4.2.
- **Edge Cases / Potential Failure Types:** Invalid client credentials or expired app installations will fail token generation.

###### Look up the order in Shopify
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Fetches order data using Shopify's GraphQL Admin API.
- **Configuration Choices:** POST method to GraphQL endpoint. Uses `X-Shopify-Access-Token` header.
- **Key Expressions / Variables:** Uses `JSON.stringify` to construct a GraphQL query fetching order status, financial status, fulfillment status, totals, addresses, fulfillments, tracking, and line items based on `searchQuery`.
- **Input / Output:** Input: `Get a Shopify token`. Output: Raw GraphQL response JSON containing order edges.
- **Version-Specific Requirements:** v4.2.
- **Edge Cases / Potential Failure Types:** API rate limiting or invalid query strings will return errors captured by downstream nodes.

###### Prepare the brief
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript node that processes Shopify API responses and pairs them with email records.
- **Configuration Choices:** Validates order lookup results, checks if the order's customer email matches the sender's address, and compiles order attributes into readable summary lines.
- **Key Expressions / Variables:** JavaScript logic assessing `orderFound`, `emailMatches`, and formatting line items, tracking information, and dates into `orderText`.
- **Input / Output:** Input: `Look up the order in Shopify`. Output: Enhanced item containing order text, match flags, and lookup error states.
- **Version-Specific Requirements:** v2.
- **Edge Cases / Potential Failure Types:** Malformed API responses or empty query edges handled gracefully via conditional array checks.

###### Classify and draft with Claude
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (v4.2) — Communicates with the Anthropic Messages API to classify email intent and draft a response.
- **Configuration Choices:** POST request to `https://api.anthropic.com/v1/messages` authenticated via `httpHeaderAuth` (`x-api-key`). Configured with strict JSON schema enforcement (`json_schema`), `effort: 'low'`, and `max_tokens: 1500`.
- **Key Expressions / Variables:** Constructs dynamic system prompts referencing store name, return window, shipping promises, brand voice, and strict operational rules. Dynamically formats user messages using customer email headers, body text, thread status, and order records.
- **Input / Output:** Input: `Prepare the brief`. Output: Anthropic API response containing structured JSON text blocks.
- **Version-Specific Requirements:** v4.2.
- **Edge Cases / Potential Failure Types:** API timeouts (90s configured limit), server refusals, or rate limits.

---

#### Block 1.3: Routing, Decision & Action

##### Overview
This block evaluates the AI's classification and confidence metrics against strict store policies, routes the message accordingly, and performs the corresponding external action: replying directly via Gmail, creating a review draft accompanied by a Slack alert, escalating directly to Slack, or discarding spam.

##### Nodes Involved
- `Decide the route`
- `Route`
- `Reply to the customer`
- `Save a draft for review`
- `Ask the team to review`
- `Escalate to a person`
- `Nothing to do`

##### Node Details

###### Decide the route
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — JavaScript decision engine implementing strict guardrails over AI outputs.
- **Configuration Choices:** Evaluates whether intents are authorized for auto-send, checks confidence thresholds against the `confidenceBar`, verifies email-matching constraints, and detects thread replies.
- **Key Expressions / Variables:** Computes destination routes (`send`, `draft`, `escalate`, `skip`), generates formatted reply and draft bodies with signatures and disclosures, and builds `threadLink` URLs.
- **Input / Output:** Input: `Classify and draft with Claude`. Output: Item enriched with route decisions, reasons, and formatted message strings.
- **Version-Specific Requirements:** v2.
- **Edge Cases / Potential Failure Types:** JSON parsing errors from malformed model blocks default safely to escalation paths.

###### Route
- **Type and Technical Role:** `n8n-nodes-base.switch` (v3.2) — Directs workflow execution based on the evaluated route property.
- **Configuration Choices:** Evaluates `{{ $json.route }}` against four strict outputs: `send`, `draft`, `escalate`, and a fallback `extra` route.
- **Key Expressions / Variables:** Evaluates string equality against route keys.
- **Input / Output:** Input: `Decide the route`. Output: Branched execution paths.
- **Version-Specific Requirements:** v3.2.
- **Edge Cases / Potential Failure Types:** Unhandled custom route strings route to the fallback output.

###### Reply to the customer
- **Type and Technical Role:** `n8n-nodes-base.gmail` (v2.2) — Automatically replies to customer emails within the existing thread.
- **Configuration Choices:** Resource: `message`, Operation: `reply`. Options: `appendAttribution: false`, `replyToSenderOnly: true`.
- **Key Expressions / Variables:** Uses `{{ $json.replyText }}` and `{{ $json.gmailMessageId }}`.
- **Input / Output:** Input: `Route` (send branch). Output: Sent Gmail message confirmation.
- **Version-Specific Requirements:** v2.2.
- **Edge Cases / Potential Failure Types:** Invalid message IDs or missing thread context will cause delivery failures.

###### Save a draft for review
- **Type and Technical Role:** `n8n-nodes-base.gmail` (v2.2) — Creates a draft reply inside the customer's email thread for human verification.
- **Configuration Choices:** Resource: `draft`, Operation: `create`. Configures recipient, subject, and thread bindings.
- **Key Expressions / Variables:** Uses `{{ $json.draftText }}`, `{{ $json.fromAddress }}`, `{{ $json.threadId }}`, and `{{ $json.draftSubject }}`.
- **Input / Output:** Input: `Route` (draft branch). Output: Created draft object.
- **Version-Specific Requirements:** v2.2.
- **Edge Cases / Potential Failure Types:** Missing thread IDs prevent proper draft threading in Gmail clients.

###### Ask the team to review
- **Type and Technical Role:** `n8n-nodes-base.slack` (v2.2) — Notifies the support team in Slack that a draft is awaiting review.
- **Configuration Choices:** Selects channel by name from Settings variables.
- **Key Expressions / Variables:** Formats a multi-line notification string containing sender info, intent, confidence score, waiting reason, summary, order reference, and direct thread links.
- **Input / Output:** Input: `Save a draft for review`. Output: Slack API response.
- **Version-Specific Requirements:** v2.2.
- **Edge Cases / Potential Failure Types:** Incorrect Slack channel names or missing bot scopes will prevent message delivery.

###### Escalate to a person
- **Type and Technical Role:** `n8n-nodes-base.slack` (v2.2) — Sends a high-priority escalation alert to the support team for complex inquiries.
- **Configuration Choices:** Selects channel by name from Settings variables.
- **Key Expressions / Variables:** Formats escalation message containing sender details, unclassified intents, reasons, summaries, subjects, orders, and thread links.
- **Input / Output:** Input: `Route` (escalate branch). Output: Slack API response.
- **Version-Specific Requirements:** v2.2.
- **Edge Cases / Potential Failure Types:** Slack credential misconfiguration.

###### Nothing to do
- **Type and Technical Role:** `n8n-nodes-base.noOp` (v1) — Terminal node for spam or newsletter emails that require no action.
- **Configuration Choices:** None.
- **Key Expressions / Variables:** None.
- **Input / Output:** Input: `Route` (fallback branch). Output: None.
- **Version-Specific Requirements:** v1.
- **Edge Cases / Potential Failure Types:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `New support email` | `n8n-nodes-base.gmailTrigger` | Polls inbox for unread support messages | None (Trigger) | `Settings` | ## Answer Shopify support emails with Claude, draft refunds for review and escalate to Slack... |
| `Settings` | `n8n-nodes-base.set` | Injects global configuration variables | `New support email` | `Normalise the email` | ## 1. Catch and prepare... |
| `Normalise the email` | `n8n-nodes-base.set` | Cleans email bodies and normalizes headers | `Settings` | `Find the order reference` | ## 1. Catch and prepare... |
| `Find the order reference` | `n8n-nodes-base.code` | Extracts order numbers or fallback emails | `Normalise the email` | `Get a Shopify token` | ## 1. Catch and prepare... |
| `Get a Shopify token` | `n8n-nodes-base.httpRequest` | Requests OAuth token from Shopify | `Find the order reference` | `Look up the order in Shopify` | ## 2. Look up and draft... |
| `Look up the order in Shopify` | `n8n-nodes-base.httpRequest` | Queries Shopify GraphQL API for order data | `Get a Shopify token` | `Prepare the brief` | ## 2. Look up and draft... |
| `Prepare the brief` | `n8n-nodes-base.code` | Compiles order records into text briefs | `Look up the order in Shopify` | `Classify and draft with Claude` | ## 2. Look up and draft... |
| `Classify and draft with Claude` | `n8n-nodes-base.httpRequest` | Prompts Claude API for intent and reply JSON | `Prepare the brief` | `Decide the route` | ## 2. Look up and draft... |
| `Decide the route` | `n8n-nodes-base.code` | Applies safety rules and determines routing | `Classify and draft with Claude` | `Route` | ## 3. Decide and act... |
| `Route` | `n8n-nodes-base.switch` | Switches workflow path based on route decision | `Decide the route` | `Reply to the customer`, `Save a draft for review`, `Escalate to a person`, `Nothing to do` | ## 3. Decide and act... |
| `Reply to the customer` | `n8n-nodes-base.gmail` | Sends automatic email reply via Gmail | `Route` | None | ## 3. Decide and act... |
| `Save a draft for review` | `n8n-nodes-base.gmail` | Saves review draft in Gmail thread | `Route` | `Ask the team to review` | ## 3. Decide and act... |
| `Ask the team to review` | `n8n-nodes-base.slack` | Sends Slack notification for review draft | `Save a draft for review` | None | ## 3. Decide and act... |
| `Escalate to a person` | `n8n-nodes-base.slack` | Sends Slack escalation alert | `Route` | None | ## 3. Decide and act... |
| `Nothing to do` | `n8n-nodes-base.noOp` | Terminal node for filtered spam | `Route` | None | ## 3. Decide and act... |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps in sequence:

1. **Create Trigger Node:**
   - Add a **Gmail Trigger** node named `New support email`.
   - Set parameters: Polling `everyMinute`, filters with `INBOX` label and `unread` status, and `simple` set to `false`.
   - Configure a valid Gmail OAuth2 credential.

2. **Configure Settings Node:**
   - Add a **Set** node named `Settings`.
   - Add string/number assignments for: `storeName` (string), `shopSubdomain` (string), `returnWindowDays` (number, e.g., 30), `shippingPromise` (string), `brandVoice` (string), `autoSendIntents` (string, comma-separated), `confidenceBar` (number, e.g., 0.8), `slackChannel` (string, e.g., `#support`), `signature` (string), and `disclosureLine` (string).
   - Connect `New support email` to `Settings`.

3. **Normalize Email Content:**
   - Add a **Set** node named `Normalise the email`.
   - Map normalized expressions for `fromAddress`, `fromName`, `subject`, `text` (with quote stripping), `isReply`, `gmailMessageId`, `threadId`, and `receivedAt`.
   - Connect `Settings` to `Normalise the email`.

4. **Extract Order References:**
   - Add a **Code** node named `Find the order reference`.
   - Insert JavaScript code to run RegEx matching against subject and body text for order numbers (e.g., `#1042`), establishing search queries for Shopify.
   - Connect `Normalise the email` to `Find the order reference`.

5. **Authenticate with Shopify:**
   - Add an **HTTP Request** node named `Get a Shopify token`.
   - Set method to `POST`, URL to `https://{{ $('Settings').first().json.shopSubdomain }}.myshopify.com/admin/oauth/access_token`, and content type to `form-urlencoded`.
   - Configure a Custom Auth credential (`httpCustomAuth`) with JSON: `{"body": {"client_id": "your_client_id", "client_secret": "your_secret"}}`.
   - Set body parameters with `grant_type: client_credentials`. Enable error continuation and retries.
   - Connect `Find the order reference` to `Get a Shopify token`.

6. **Query Shopify Orders:**
   - Add an **HTTP Request** node named `Look up the order in Shopify`.
   - Set method to `POST`, URL to `https://{{ $('Settings').first().json.shopSubdomain }}.myshopify.com/admin/api/2026-07/graphql.json`.
   - Pass headers: `X-Shopify-Access-Token` (from previous node) and `content-type: application/json`.
   - Provide the GraphQL query payload searching orders by name or email.
   - Connect `Get a Shopify token` to `Look up the order in Shopify`.

7. **Prepare Briefs:**
   - Add a **Code** node named `Prepare the brief`.
   - Insert JavaScript logic to validate order search results, verify customer email matching, and generate plain-text summaries (`orderText`).
   - Connect `Look up the order in Shopify` to `Prepare the brief`.

8. **Integrate Claude AI:**
   - Add an **HTTP Request** node named `Classify and draft with Claude`.
   - Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`.
   - Configure HTTP Header Auth credential (`x-api-key`) with your Anthropic API key.
   - Set headers: `anthropic-version: 2023-06-01`, `anthropic-beta: server-side-fallback-2026-07-01`, `content-type: application/json`.
   - Provide JSON body utilizing model `claude-opus-5`, strict JSON schema configurations, system instructions, and customer email payloads.
   - Connect `Prepare the brief` to `Classify and draft with Claude`.

9. **Evaluate Routing Rules:**
   - Add a **Code** node named `Decide the route`.
   - Insert JavaScript evaluation rules checking confidence scores, intent whitelists, email matches, and reply statuses to determine whether to `send`, `draft`, `escalate`, or `skip`.
   - Connect `Classify and draft with Claude` to `Decide the route`.

10. **Implement Switch Routing:**
    - Add a **Switch** node named `Route`.
    - Configure rules checking `{{ $json.route }}` for values `send`, `draft`, and `escalate`, with fallback handling.
    - Connect `Decide the route` to `Route`.

11. **Configure Action Nodes:**
    - **Reply to the customer:** Add a **Gmail** node (resource `message`, operation `reply`) connected to the `send` output. Use `{{ $json.replyText }}`.
    - **Save a draft for review:** Add a **Gmail** node (resource `draft`, operation `create`) connected to the `draft` output. Use `{{ $json.draftText }}`.
    - **Ask the team to review:** Add a **Slack** node connected after `Save a draft for review` targeting the configured channel with review notifications. Configure Slack OAuth2 credentials.
    - **Escalate to a person:** Add a **Slack** node connected to the `escalate` output targeting the configured channel with escalation alerts.
    - **Nothing to do:** Add a **NoOp** node connected to the fallback output.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Companion article explaining design decisions, cost per email, and reasoning behind thresholds | [Amplence Blog](https://amplence.com/blog/n8n-shopify-customer-support-workflow) |
| Built by Amplence | Creator Attribution |