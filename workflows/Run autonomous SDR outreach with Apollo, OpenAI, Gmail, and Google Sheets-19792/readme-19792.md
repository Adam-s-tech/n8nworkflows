Run autonomous SDR outreach with Apollo, OpenAI, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/run-autonomous-sdr-outreach-with-apollo--openai--gmail--and-google-sheets-19792


# Run autonomous SDR outreach with Apollo, OpenAI, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates an end-to-end B2B Sales Development Representative (SDR) lifecycle. It operates across two primary pipelines: an outbound prospecting engine and an inbound reply-handling engine. The purpose of the workflow is to discover target leads daily based on specified Ideal Customer Profiles (ICPs), research company backgrounds and real-time news, generate personalized cold outreach emails using AI, track outreach in a CRM sheet, and dynamically process incoming responses to handle booking, questions, or opt-outs.

The workflow logic is categorized into the following functional blocks:
- **1.1 Lead Discovery and Filtering:** Triggers daily, defines target ICP parameters, and searches for matching contacts in Apollo while performing deduplication against existing CRM entries.
- **1.2 Company Research and AI Email Drafting:** Enriches prospect organization data, retrieves recent company news via web search, synthesizes context with OpenAI to draft personalized emails, sends them via Gmail, and logs the activity in Google Sheets.
- **1.3 Reply Detection and Intent Classification:** Polls incoming unread emails, verifies lead existence in the CRM database, and uses OpenAI to categorize the sentiment and intent of prospect replies.
- **1.4 Intent-Based Action Routing:** Evaluates classified reply intents through a conditional switch and executes downstream actions (such as scheduling calendar meetings, answering questions, or updating lead statuses).

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Discovery and Filtering
- **Overview:** Initializes the daily prospecting run, sets the criteria for target company sizes, titles, and regions, queries the Apollo API, breaks down the search array, and screens out contacts that already exist in the Google Sheets CRM.
- **Nodes Involved:** 
  - `Schedule Trigger - Daily Lead Gen`
  - `Set ICP Criteria`
  - `Apollo: Search Leads`
  - `Split Out Leads`
  - `Google Sheets: Check Existing Lead`

- **Node Details:**
  - **Schedule Trigger - Daily Lead Gen**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger node). Fires the automation interval every 24 hours.
    - *Configuration:* Interval set to 24 hours.
    - *Connections:* Input: None; Output: `Set ICP Criteria`.
    - *Edge Cases:* Missed executions if the n8n instance is offline.
  - **Set ICP Criteria**
    - *Type and Role:* `n8n-nodes-base.set` (Data transformation node). Establishes target demographic criteria including industry, employee count bounds, job titles, and regions.
    - *Configuration:* Assigns string and numeric variables (`industry`, `employeeRangeMin`, `employeeRangeMax`, `titles`, `locations`, `perPage`).
    - *Connections:* Input: `Schedule Trigger - Daily Lead Gen`; Output: `Apollo: Search Leads`.
    - *Edge Cases:* Formatting errors if values are left unescaped.
  - **Apollo: Search Leads**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (API integration node). Queries Apollo's mixed people search endpoint using criteria parameters.
    - *Configuration:* POST request to `https://api.apollo.io/v1/mixed_people/search` using JSON body mapping and custom header `X-Api-Key`.
    - *Key Expressions:* Maps payload arrays dynamically from `{{ $json.titles.split(',') }}` and size ranges.
    - *Connections:* Input: `Set ICP Criteria`; Output: `Split Out Leads`.
    - *Edge Cases:* Rate-limiting, authentication failures, or empty arrays returned by Apollo.
  - **Split Out Leads**
    - *Type and Role:* `n8n-nodes-base.splitOut` (Data restructuring node). Flattens the array of people returned from Apollo into individual items.
    - *Configuration:* Field to split out set to `people`.
    - *Connections:* Input: `Apollo: Search Leads`; Output: `Google Sheets: Check Existing Lead`.
    - *Edge Cases:* Fails if the `people` property is missing or null.
  - **Google Sheets: Check Existing Lead**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM lookup node). Searches the Leads spreadsheet to determine if an email already exists.
    - *Configuration:* Uses lookup column `Email` matching against incoming email payloads. `continueOnFail` is enabled.
    - *Connections:* Input: `Split Out Leads`; Output: `Apollo: Enrich Company`.
    - *Edge Cases:* API quotas, permission issues, or partial matches.

---

#### 2.2 Company Research and AI Email Drafting
- **Overview:** Enriches valid prospect organizations, performs a live web search for recent news updates, invokes OpenAI to draft hyper-personalized copy, parses the structural output, sends the email, and logs the outreach action.
- **Nodes Involved:**
  - `Apollo: Enrich Company`
  - `Web Search: Company News`
  - `OpenAI: Research & Draft Email`
  - `Code: Parse Email JSON`
  - `Gmail: Send Cold Email`
  - `Google Sheets: Log New Lead`

- **Node Details:**
  - **Apollo: Enrich Company**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (API integration node). Fetches deeper institutional details for the company domain.
    - *Configuration:* GET request to `https://api.apollo.io/v1/organizations/enrich` with domain query parameters and API key headers.
    - *Key Expressions:* `={{ $('Split Out Leads').item.json.organization.primary_domain }}`
    - *Connections:* Input: `Google Sheets: Check Existing Lead`; Output: `Web Search: Company News`.
    - *Edge Cases:* Unknown domains resulting in 404 responses from Apollo.
  - **Web Search: Company News**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (API integration node). Queries Serper news to gather recent press coverage on the target company.
    - *Configuration:* POST request to `https://google.serper.dev/news` using JSON bodies and `X-API-KEY` authentication.
    - *Key Expressions:* `={{ { "q": $('Split Out Leads').item.json.organization.name } }}`
    - *Connections:* Input: `Apollo: Enrich Company`; Output: `OpenAI: Research & Draft Email`.
    - *Edge Cases:* Low-yield search results returning empty arrays.
  - **OpenAI: Research & Draft Email**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (AI processing node). Synthesizes lead data, enrichment values, and recent news articles to write a concise cold outreach draft.
    - *Configuration:* Model set to `gpt-4o-mini` with a temperature of `0.7`.
    - *Key Expressions:* Constructs custom user prompts injecting upstream variables.
    - *Connections:* Input: `Web Search: Company News`; Output: `Code: Parse Email JSON`.
    - *Edge Cases:* Token limit ceiling, API downtime, or malformed JSON generation.
  - **Code: Parse Email JSON**
    - *Type and Role:* `n8n-nodes-base.code` (Script execution node). Cleans LLM output text, strips markdown backticks, and structures object properties for downstream mailing.
    - *Configuration:* JavaScript execution block handling string sanitization and fallback structures.
    - *Connections:* Input: `OpenAI: Research & Draft Email`; Output: `Gmail: Send Cold Email`.
    - *Edge Cases:* Syntax exceptions if the JSON block is completely malformed.
  - **Gmail: Send Cold Email**
    - *Type and Role:* `n8n-nodes-base.gmail` (Messaging node). Dispatches the drafted email to the prospect.
    - *Configuration:* Uses Gmail OAuth2 credentials. Sets recipient address, subject, and message body expressions.
    - *Connections:* Input: `Code: Parse Email JSON`; Output: `Google Sheets: Log New Lead`.
    - *Edge Cases:* OAuth token expiration, invalid sender addresses, or spam blocks.
  - **Google Sheets: Log New Lead**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM logging node). Appends a new tracking row recording contact metadata, initial status, and timestamps.
    - *Configuration:* Operation set to `append` using defined column mappings.
    - *Key Expressions:* `={{ $now.toISO() }}` for timestamp generation.
    - *Connections:* Input: `Gmail: Send Cold Email`; Output: None.
    - *Edge Cases:* Schema mismatches or sheet locks during write operations.

---

#### 2.3 Reply Detection and Intent Classification
- **Overview:** Continuously polls the Gmail inbox for unread responses, validates the sender against the CRM database, confirms lead existence, and leverages OpenAI to evaluate the conversational intent of the reply.
- **Nodes Involved:**
  - `Gmail Trigger: New Reply`
  - `Google Sheets: Match Lead by Email`
  - `IF: Lead Found`
  - `OpenAI: Classify Reply Intent`
  - `Code: Parse Intent JSON`

- **Node Details:**
  - **Gmail Trigger: New Reply**
    - *Type and Role:* `n8n-nodes-base.gmailTrigger` (Trigger node). Polls inbox messages matching specific filter constraints every minute.
    - *Configuration:* Query filter set to `is:unread -in:sent -category:promotions`. Polling frequency set to every minute.
    - *Connections:* Input: None; Output: `Google Sheets: Match Lead by Email`.
    - *Edge Cases:* High API quota usage due to frequent polling intervals.
  - **Google Sheets: Match Lead by Email**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM lookup node). Searches CRM entries using the incoming email address of the respondent.
    - *Configuration:* Matching columns set to `Email`. `continueOnFail` enabled.
    - *Connections:* Input: `Gmail Trigger: New Reply`; Output: `IF: Lead Found`.
    - *Edge Cases:* Unmatched replies from unknown email addresses.
  - **IF: Lead Found**
    - *Type and Role:* `n8n-nodes-base.if` (Conditional routing node). Validates whether the lookup returned any associated CRM records.
    - *Configuration:* Checks if input length is greater than 0.
    - *Connections:* Input: `Google Sheets: Match Lead by Email`; Output: `OpenAI: Classify Reply Intent`.
    - *Edge Cases:* Processing halts or drops items if the lead cannot be matched.
  - **OpenAI: Classify Reply Intent**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (AI processing node). Classifies the incoming message content into predefined categories (`interested`, `not_interested`, `question`, `other`).
    - *Configuration:* Model set to `gpt-4o-mini` with a temperature of `0.2`.
    - *Connections:* Input: `IF: Lead Found`; Output: `Code: Parse Intent JSON`.
    - *Edge Cases:* Ambiguous phrasing causing misclassified intents.
  - **Code: Parse Intent JSON**
    - *Type and Role:* `n8n-nodes-base.code` (Script execution node). Parses the classification payload returned by the LLM and merges it with lead properties.
    - *Configuration:* JavaScript parsing block with built-in fallbacks.
    - *Connections:* Input: `OpenAI: Classify Reply Intent`; Output: `Switch: Route by Intent`.
    - *Edge Cases:* Unhandled parsing exceptions resulting in default intent strings.

---

#### 2.4 Intent-Based Action Routing
- **Overview:** Directs the processed reply through a multi-branch switch based on intent categories, creating calendar bookings, sending automated responses, or updating CRM statuses accordingly.
- **Nodes Involved:**
  - `Switch: Route by Intent`
  - `Google Calendar: Create Meeting`
  - `Gmail: Send Meeting Confirmation`
  - `Google Sheets: Update Status - Meeting Booked`
  - `Google Sheets: Update Status - Not Interested`
  - `Gmail: Send Answer to Question`
  - `Google Sheets: Update Status - Replied to Question`
  - `Google Sheets: Update Status - Unsubscribed`

- **Node Details:**
  - **Switch: Route by Intent**
    - *Type and Role:* `n8n-nodes-base.switch` (Routing node). Evaluates the parsed `intent` field and distributes items to corresponding workflows.
    - *Configuration:* Routes based on string matches for `interested`, `not_interested`, `question`, and fallback options.
    - *Connections:* Input: `Code: Parse Intent JSON`; Output: Multiple downstream action nodes.
    - *Edge Cases:* Unmatched string literals triggering fallback outputs.
  - **Google Calendar: Create Meeting**
    - *Type and Role:* `n8n-nodes-base.googleCalendar` (Calendar integration node). Creates an intro call event on the primary calendar when a lead expresses interest.
    - *Configuration:* Sets start and end times dynamically offset from current timestamps, adding attendees via expressions.
    - *Key Expressions:* `={{ $now.plus({ days: 2 }).set({ hour: 15, minute: 0 }).toISO() }}`
    - *Connections:* Input: `Switch: Route by Intent`; Output: `Gmail: Send Meeting Confirmation`.
    - *Edge Cases:* Conflicting calendar slots or missing attendee permissions.
  - **Gmail: Send Meeting Confirmation**
    - *Type and Role:* `n8n-nodes-base.gmail` (Messaging node). Sends a booking confirmation email to the contact.
    - *Connections:* Input: `Google Calendar: Create Meeting`; Output: `Google Sheets: Update Status - Meeting Booked`.
    - *Edge Cases:* Invalid email payloads.
  - **Google Sheets: Update Status - Meeting Booked**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM logging node). Updates the lead record status to reflect a confirmed meeting.
    - *Configuration:* Operation set to `update` matching on `Email`.
    - *Connections:* Input: `Gmail: Send Meeting Confirmation`; Output: None.
    - *Edge Cases:* Missing rows or unindexed columns.
  - **Google Sheets: Update Status - Not Interested**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM logging node). Updates the lead status to reflect negative responses.
    - *Configuration:* Operation set to `update` matching on `Email`.
    - *Connections:* Input: `Switch: Route by Intent`; Output: None.
    - *Edge Cases:* Update failures due to network timeouts.
  - **Gmail: Send Answer to Question**
    - *Type and Role:* `n8n-nodes-base.gmail` (Messaging node). Replies to prospect inquiries using the AI-suggested answer.
    - *Configuration:* Sends email body mapped from `{{ $json.suggested_reply }}`.
    - *Connections:* Input: `Switch: Route by Intent`; Output: `Google Sheets: Update Status - Replied to Question`.
    - *Edge Cases:* Empty suggestion strings resulting in blank replies.
  - **Google Sheets: Update Status - Replied to Question**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM logging node). Updates the CRM status following question responses.
    - *Configuration:* Operation set to `update` matching on `Email`.
    - *Connections:* Input: `Gmail: Send Answer to Question`; Output: None.
    - *Edge Cases:* Failed row lookups.
  - **Google Sheets: Update Status - Unsubscribed**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (CRM logging node). Updates lead statuses to "Not Interested" when opt-out intent is detected.
    - *Configuration:* Operation set to `update` matching on `Email`.
    - *Connections:* Input: `Switch: Route by Intent`; Output: None.
    - *Edge Cases:* Data desynchronization.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 📌 Overview – How This Workflow Works | `n8n-nodes-base.stickyNote` | Overview documentation block. | None | None | ## 🤖 Autonomous SDR Agent – AI-Powered Lead Gen, Outreach & Reply Handling<br><br>### How it works<br>This workflow runs a full autonomous SDR loop across two parallel pipelines. **Outbound:** every morning, a schedule trigger searches Apollo for prospects matching your ICP (industry, title, company size). Each new lead is enriched with company details and a real-time web news search, then GPT-4o-mini drafts a hyper-personalised cold email referencing the company's recent news. The email is sent via Gmail and the lead is logged to Google Sheets. **Inbound reply handling:** a Gmail trigger fires on every new reply, matches it against the leads sheet, classifies the reply intent using GPT-4o-mini (meeting request, question, not interested, unsubscribed), and routes it to the right action — booking a calendar event + confirmation email, answering the question, or updating the CRM status.<br><br>### Setup steps<br>1. Add your **Apollo API key** to `Apollo: Search Leads` and `Apollo: Enrich Company`... |
| Section – ICP Lead Discovery & Deduplication | `n8n-nodes-base.stickyNote` | Visual grouping note for lead discovery. | None | None | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Section – Company Research & AI Email Drafting | `n8n-nodes-base.stickyNote` | Visual grouping note for email drafting. | None | None | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Section – Reply Detection & Intent Classification | `n8n-nodes-base.stickyNote` | Visual grouping note for reply detection. | None | None | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| Section – Intent-Based Action Routing | `n8n-nodes-base.stickyNote` | Visual grouping note for intent routing. | None | None | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| 🔑 Credentials & Configuration | `n8n-nodes-base.stickyNote` | Credential requirement notes. | None | None | ## 🔑 Credentials Required<br>- **Apollo API key** — lead search + company enrichment<br>- **Web search API key** — company news (SerpAPI, Tavily, or Brave)<br>- **OpenAI API** — email drafting + reply classification<br>- **Gmail OAuth2** — send + trigger<br>- **Google Sheets OAuth2** — leads CRM<br>- **Google Calendar OAuth2** — meeting booking |
| Google Sheets: Match Lead by Email | `n8n-nodes-base.googleSheets` | Looks up CRM records using email addresses. | `Gmail Trigger: New Reply` | `IF: Lead Found` | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| IF: Lead Found | `n8n-nodes-base.if` | Verifies whether a matching CRM record exists. | `Google Sheets: Match Lead by Email` | `OpenAI: Classify Reply Intent` | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| OpenAI: Classify Reply Intent | `@n8n/n8n-nodes-langchain.openAi` | Classifies reply intent into structured JSON. | `IF: Lead Found` | `Code: Parse Intent JSON` | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| Code: Parse Intent JSON | `n8n-nodes-base.code` | Parses LLM text into actionable intent metadata. | `OpenAI: Classify Reply Intent` | `Switch: Route by Intent` | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| Switch: Route by Intent | `n8n-nodes-base.switch` | Routes traffic based on classification intent. | `Code: Parse Intent JSON` | `Google Calendar: Create Meeting`, `Google Sheets: Update Status - Not Interested`, `Gmail: Send Answer to Question`, `Google Sheets: Update Status - Unsubscribed` | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Google Calendar: Create Meeting | `n8n-nodes-base.googleCalendar` | Books calendar events for interested prospects. | `Switch: Route by Intent` | `Gmail: Send Meeting Confirmation` | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Gmail: Send Meeting Confirmation | `n8n-nodes-base.gmail` | Sends booking confirmation emails. | `Google Calendar: Create Meeting` | `Google Sheets: Update Status - Meeting Booked` | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Google Sheets: Update Status - Meeting Booked | `n8n-nodes-base.googleSheets` | Updates CRM status to meeting booked. | `Gmail: Send Meeting Confirmation` | None | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Google Sheets: Update Status - Not Interested | `n8n-nodes-base.googleSheets` | Updates CRM status to not interested. | `Switch: Route by Intent` | None | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Gmail: Send Answer to Question | `n8n-nodes-base.gmail` | Replies to prospect questions with AI drafts. | `Switch: Route by Intent` | `Google Sheets: Update Status - Replied to Question` | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Google Sheets: Update Status - Replied to Question | `n8n-nodes-base.googleSheets` | Updates CRM status to answered question. | `Gmail: Send Answer to Question` | None | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |
| Gmail Trigger: New Reply | `n8n-nodes-base.gmailTrigger` | Polls inbox for incoming prospect replies. | None | `Google Sheets: Match Lead by Email` | ## 📨 Reply Detection & Intent Classification<br>A Gmail trigger fires on every new reply from a prospect. The lead is matched against the CRM sheet by email address. GPT-4o-mini classifies the reply intent — meeting request, question, not interested, or unsubscribed — before routing. |
| Apollo: Enrich Company | `n8n-nodes-base.httpRequest` | Fetches corporate profile enrichment data. | `Google Sheets: Check Existing Lead` | `Web Search: Company News` | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Web Search: Company News | `n8n-nodes-base.httpRequest` | Searches recent news via Serper API. | `Apollo: Enrich Company` | `OpenAI: Research & Draft Email` | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| OpenAI: Research & Draft Email | `@n8n/n8n-nodes-langchain.openAi` | Drafts personalized cold email copy using GPT-4o-mini. | `Web Search: Company News` | `Code: Parse Email JSON` | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Code: Parse Email JSON | `n8n-nodes-base.code` | Parses generated email copy and JSON payloads. | `OpenAI: Research & Draft Email` | `Gmail: Send Cold Email` | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Gmail: Send Cold Email | `n8n-nodes-base.gmail` | Sends outbound cold emails to prospects. | `Code: Parse Email JSON` | `Google Sheets: Log New Lead` | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Google Sheets: Log New Lead | `n8n-nodes-base.googleSheets` | Logs new outreach activities to the CRM spreadsheet. | `Gmail: Send Cold Email` | None | ## ✍️ Company Research & AI Email Drafting<br>For each new lead: enriches the company profile via Apollo, fetches recent company news via web search, and passes both to GPT-4o-mini to draft a personalised cold email referencing current events. The parsed email is sent via Gmail and the lead is logged. |
| Schedule Trigger - Daily Lead Gen | `n8n-nodes-base.scheduleTrigger` | Triggers the outbound lead pipeline daily. | None | `Set ICP Criteria` | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Set ICP Criteria | `n8n-nodes-base.set` | Defines buyer persona search parameters. | `Schedule Trigger - Daily Lead Gen` | `Apollo: Search Leads` | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Apollo: Search Leads | `n8n-nodes-base.httpRequest` | Queries Apollo API for matching contact profiles. | `Set ICP Criteria` | `Split Out Leads` | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Split Out Leads | `n8n-nodes-base.splitOut` | Flattens batch Apollo person arrays into single items. | `Apollo: Search Leads` | `Google Sheets: Check Existing Lead` | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Google Sheets: Check Existing Lead | `n8n-nodes-base.googleSheets` | Deduplicates contacts against the CRM sheet. | `Split Out Leads` | `Apollo: Enrich Company` | ## 🎯 ICP Lead Discovery & Deduplication<br>Fires daily. Sets your ICP filters (industry, title, location, company size) and searches Apollo for matching prospects. Splits the results into individual leads and checks each against the Leads sheet — existing contacts are skipped before enrichment. |
| Google Sheets: Update Status - Unsubscribed | `n8n-nodes-base.googleSheets` | Updates CRM status to reflect unsubscriptions. | `Switch: Route by Intent` | None | ## 🔀 Intent-Based Action Routing<br>A Switch node routes each classified reply to the right action: meeting requests → create Google Calendar event + send confirmation email + update sheet; questions → send a GPT-drafted answer + update sheet; not interested / unsubscribed → update CRM status only. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually within an n8n canvas:

1. **Create the Outbound Trigger Node**
   - Add a **Schedule Trigger** node. Set interval rules to run every 24 hours.
2. **Configure ICP Parameters**
   - Add a **Set** node connected downstream from the schedule trigger.
   - Configure assignments for: `industry` (e.g., Software Development), `employeeRangeMin` (11), `employeeRangeMax` (200), `titles` (Founder, CEO, Head of Sales, VP Sales), `locations` (United States, United Kingdom, India), and `perPage` (25).
3. **Connect Apollo Search**
   - Add an **HTTP Request** node. Method: `POST`, URL: `https://api.apollo.io/v1/mixed_people/search`.
   - Configure headers to include `Content-Type: application/json` and `X-Api-Key: YOUR_APOLLO_API_KEY`.
   - Format the JSON body expression using properties from the Set node (`$json.titles`, size ranges, locations).
4. **Split Lead Results**
   - Add a **Split Out** node. Set the field to split out to `people`.
5. **Deduplicate Leads via CRM**
   - Add a **Google Sheets** node configured for a lookup operation.
   - Select your target spreadsheet ID and sheet name (`Leads`). Set lookup column to `Email` matching against `{{ $json.email }}`. Enable `continueOnFail`.
6. **Enrich Company Data**
   - Add an **HTTP Request** node. Method: `GET`, URL: `https://api.apollo.io/v1/organizations/enrich`.
   - Add query parameter `domain` set to `={{ $('Split Out Leads').item.json.organization.primary_domain }}` and header `X-Api-Key: YOUR_APOLLO_API_KEY`.
7. **Fetch Company News**
   - Add an **HTTP Request** node. Method: `POST`, URL: `https://google.serper.dev/news`.
   - Configure headers: `X-API-KEY: YOUR_SERPER_API_KEY` and `Content-Type: application/json`.
   - Pass the company name in the JSON body payload.
8. **Draft Email with OpenAI**
   - Add an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`). Select model `gpt-4o-mini` with temperature `0.7`.
   - Set system prompt instructing the model to act as an SDR copywriter returning valid JSON with `subject` and `body` keys.
9. **Parse Email Output**
   - Add a **Code** node. Insert JavaScript logic to sanitize markdown fences, parse JSON strings safely, and construct standardized output properties (`email`, `firstName`, `lastName`, `company`, `domain`, `title`, `subject`, `body`).
10. **Send Cold Email via Gmail**
    - Add a **Gmail** node configured to send messages. Connect Gmail OAuth2 credentials, map recipient addresses, subject lines, and message bodies.
11. **Log Outbound Lead in CRM**
    - Add a **Google Sheets** node configured to `append` rows. Map columns for email, names, company data, initial status (`Sent`), and date timestamps (`{{ $now.toISO() }}`).
12. **Configure Inbound Reply Trigger**
    - Add a **Gmail Trigger** node. Set polling frequency to every minute with filter query: `is:unread -in:sent -category:promotions`.
13. **Match Incoming Leads**
    - Add a **Google Sheets** node to look up incoming respondent emails against the Leads spreadsheet with `continueOnFail` enabled.
14. **Validate Lead Existence**
    - Add an **IF** node checking if input row lengths are greater than 0.
15. **Classify Reply Intent**
    - Add an **OpenAI** node (`gpt-4o-mini`, temperature `0.2`) to evaluate reply text and return structured JSON containing `intent` and `suggested_reply`.
16. **Parse Intent Payload**
    - Add a **Code** node to sanitize and parse intent classification outputs.
17. **Route by Intent Using Switch**
    - Add a **Switch** node evaluating `$json.intent` across rules: `interested`, `not_interested`, `question`, and fallback paths.
18. **Implement Interested Pipeline**
    - Connect the first switch branch to a **Google Calendar** node to schedule meetings, followed by a **Gmail** confirmation node, and finalize with a **Google Sheets** update node setting status to `Meeting Booked`.
19. **Implement Not Interested / Unsubscribed Pipelines**
    - Connect the second and fourth switch branches to **Google Sheets** update nodes setting status to `Not Interested`.
20. **Implement Question Pipeline**
    - Connect the third switch branch to a **Gmail** node sending the AI-suggested answer, followed by a **Google Sheets** update node setting status to `Answered Question`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apollo API integration documentation | Used for prospect generation and company enrichment endpoints. |
| Serper News API integration | Used for real-time news retrieval to personalize outreach copy. |
| OpenAI API integration | Powers both cold email drafting and inbound reply classification. |