Chase unanswered quotes by SMS with Twilio and a webhook

https://n8nworkflows.xyz/workflows/chase-unanswered-quotes-by-sms-with-twilio-and-a-webhook-19773


# Chase unanswered quotes by SMS with Twilio and a webhook

### 1. Workflow Overview

This workflow automates the process of following up on sent quotes via SMS using Twilio. It is specifically designed for service-oriented businesses (such as contractors, plumbers, HVAC technicians, and cleaning services) that want to reduce lost deals due to customer silence without requiring an external database. 

The core logic operates in a background sequence triggered by an incoming HTTP webhook. It normalizes customer data, calculates localized follow-up schedules, and sends up to three sequenced nudge messages. Before sending each nudge, the workflow queries Twilio to verify whether the customer has replied since the chase began. If a reply is detected, the workflow immediately halts the sequence and notifies the business owner. If all three nudges pass without a response, the workflow closes the sequence and sends a final notification.

The workflow is grouped into the following logical blocks:
- **1.1 Input Reception & Configuration:** Receives quote details, defines business and schedule settings, normalizes contact data, and immediately responds to the webhook caller.
- **1.2 Validation & Initialization:** Validates the processed contact data and alerts the business owner via SMS that the chase has started.
- **1.3 First Follow-Up Cycle:** Waits for the scheduled time, checks for customer replies via Twilio, branches based on the reply status, and sends the first nudge if no reply is found.
- **1.4 Second Follow-Up Cycle:** Waits for the second interval, checks for inbound replies, branches logic, and sends the second nudge if unanswered.
- **1.5 Third Follow-Up Cycle & Completion:** Waits for the final interval, checks for replies one last time, sends the third nudge, and closes the chase sequence if no response is received.
- **1.6 Early Termination Handling:** Captures cases where the customer replies during any of the check phases and immediately alerts the business owner while terminating further automated follow-ups.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
This block ingests incoming quote metadata, registers global business preferences, and normalizes phone numbers and personalized text templates for downstream consumption.

- **Quote sent**
  - **Type & Technical Role:** Webhook node (`n8n-nodes-base.webhook`). Acts as the primary entry point, listening for incoming HTTP POST requests containing quote parameters.
  - **Configuration:** Configured with path `quote-chase`, HTTP method `POST`, and `responseMode` set to return via a separate response node.
  - **Expressions/Variables:** None.
  - **Connections:** Inputs: None (Trigger). Outputs: Connects to `Your settings`.
  - **Version Requirements:** Version 2.1.
  - **Edge Cases & Failure Types:** Incorrect HTTP method or payload timeouts.
  - **Sub-workflow Reference:** None.

- **Your settings**
  - **Type & Technical Role:** Edit Fields (Set) node (`n8n-nodes-base.set`). Defines static parameters such as business name, phone numbers, time zone, wait intervals, and message templates.
  - **Configuration:** Assigns values for `business_name`, `business_number`, `owner_cell`, `timezone`, wait durations (`wait_1_days`, `wait_2_days`, `wait_3_days`), and nudge message templates (`nudge_1_text`, `nudge_2_text`, `nudge_3_text`).
  - **Expressions/Variables:** Uses literal string and number assignments.
  - **Connections:** Inputs: `Quote sent`. Outputs: Connects to `Prepare the chase`.
  - **Version Requirements:** Version 3.4.
  - **Edge Cases & Failure Types:** Missing configuration fields or incorrect time zone strings.
  - **Sub-workflow Reference:** None.

- **Prepare the chase**
  - **Type & Technical Role:** Code node (`n8n-nodes-base.code`). Normalizes incoming raw data, formats phone numbers to E.164, compiles custom text templates, and calculates precise UTC timestamps for scheduled nudges based on local time zones.
  - **Configuration:** JavaScript execution block reading from `Your settings` and `Quote sent`.
  - **Expressions/Variables:** Uses `DateTime.now().setZone(tz)`, regular expressions for phone parsing, and template replacement strings (`{name}`, `{business}`, `{amount}`, `{job}`, `{link}`).
  - **Connections:** Inputs: `Your settings`. Outputs: Connects to `Answer the webhook`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failure Types:** Malformed phone numbers causing `ok` to evaluate to `false`.
  - **Sub-workflow Reference:** None.

- **Answer the webhook**
  - **Type & Technical Role:** Respond to Webhook node (`n8n-nodes-base.respondToWebhook`). Sends an immediate acknowledgment back to the webhook caller.
  - **Configuration:** Responds with JSON containing `{"ok": true}`.
  - **Expressions/Variables:** `={{ { "ok": true } }}`
  - **Connections:** Inputs: `Prepare the chase`. Outputs: Connects to `Usable phone?`.
  - **Version Requirements:** Version 1.5.
  - **Edge Cases & Failure Types:** Network drop before response transmission.
  - **Sub-workflow Reference:** None.

#### 2.2 Validation & Initialization
This block checks whether the provided phone number and contact settings are valid before alerting the business owner that a new quote sequence has started.

- **Usable phone?**
  - **Type & Technical Role:** If node (`n8n-nodes-base.if`). Evaluates whether the preparation script flagged the contact details as valid.
  - **Configuration:** Loose type validation evaluating a boolean condition.
  - **Expressions/Variables:** `={{ $('Prepare the chase').first().json.ok === true }}`
  - **Connections:** Inputs: `Answer the webhook`. Outputs: True branch connects to `Tell the owner`. False branch terminates.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Evaluates to false if phone digits are insufficient or configuration numbers are missing.
  - **Sub-workflow Reference:** None.

- **Tell the owner**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends an SMS notification to the business owner confirming that the quote chase has initiated.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses expressions pointing to `owner_cell`, `business_number`, and `owner_start` from `Prepare the chase`.
  - **Connections:** Inputs: `Usable phone?` (True branch). Outputs: Connects to `Wait for nudge 1`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio authentication failure, insufficient funds, or invalid destination number.
  - **Sub-workflow Reference:** None.

#### 2.3 First Follow-Up Cycle
This block handles the execution delay for the first reminder, checks Twilio message history for customer replies, evaluates if a reply occurred, and either stops the workflow or sends the first nudge.

- **Wait for nudge 1**
  - **Type & Technical Role:** Wait node (`n8n-nodes-base.wait`). Pauses execution until the specific timestamp calculated for the first nudge.
  - **Configuration:** Resume set to `specificTime`.
  - **Expressions/Variables:** `={{ $('Prepare the chase').first().json.nudge_1_at }}`
  - **Connections:** Inputs: `Tell the owner`. Outputs: Connects to `Check for a reply 1`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** Execution engine restarts or workflow deactivation during the wait period.
  - **Sub-workflow Reference:** None.

- **Check for a reply 1**
  - **Type & Technical Role:** HTTP Request node (`n8n-nodes-base.httpRequest`). Queries the Twilio Messages API to retrieve inbound text messages from the customer.
  - **Configuration:** GET request to Twilio API using predefined Twilio credentials with query parameters filtering by `From`, `To`, and `DateSent>`.
  - **Expressions/Variables:** Dynamic URL construction utilizing account SID from previous Twilio execution outputs.
  - **Connections:** Inputs: `Wait for nudge 1`. Outputs: Connects to `Replied before 1?`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** API rate limits or invalid Twilio credentials.
  - **Sub-workflow Reference:** None.

- **Replied before 1?**
  - **Type & Technical Role:** If node (`n8n-nodes-base.if`). Checks if any retrieved message has a timestamp greater than or equal to the chase start time.
  - **Configuration:** Loose type validation evaluating array methods.
  - **Expressions/Variables:** `={{ ($json.messages || []).some(m => new Date(m.date_sent) >= new Date($('Prepare the chase').first().json.started_at)) }}`
  - **Connections:** Inputs: `Check for a reply 1`. Outputs: True branch connects to `Tell the owner they replied`. False branch connects to `Nudge 1`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Timezone mismatch between local system time and API message timestamps.
  - **Sub-workflow Reference:** None.

- **Nudge 1**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends the first automated reminder SMS to the customer.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses `phone`, `business_number`, and `nudge_1` variables from `Prepare the chase`.
  - **Connections:** Inputs: `Replied before 1?` (False branch). Outputs: Connects to `Wait for nudge 2`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio messaging gateway errors.
  - **Sub-workflow Reference:** None.

#### 2.4 Second Follow-Up Cycle
This block executes the second waiting interval, checks for inbound replies, and sends the second reminder if the customer remains unresponsive.

- **Wait for nudge 2**
  - **Type & Technical Role:** Wait node (`n8n-nodes-base.wait`). Pauses execution until the timestamp for the second nudge.
  - **Configuration:** Resume set to `specificTime`.
  - **Expressions/Variables:** `={{ $('Prepare the chase').first().json.nudge_2_at }}`
  - **Connections:** Inputs: `Nudge 1`. Outputs: Connects to `Check for a reply 2`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** Workflow persistence failures during extended wait durations.
  - **Sub-workflow Reference:** None.

- **Check for a reply 2**
  - **Type & Technical Role:** HTTP Request node (`n8n-nodes-base.httpRequest`). Queries Twilio Messages API for inbound replies received prior to the second nudge.
  - **Configuration:** GET request using Twilio API credentials with query parameters.
  - **Expressions/Variables:** Dynamic URL matching account SID extraction.
  - **Connections:** Inputs: `Wait for nudge 2`. Outputs: Connects to `Replied before 2?`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Network timeout reaching Twilio endpoints.
  - **Sub-workflow Reference:** None.

- **Replied before 2?**
  - **Type & Technical Role:** If node (`n8n-nodes-base.if`). Evaluates whether an inbound message arrived since the chase started.
  - **Configuration:** Loose type validation.
  - **Expressions/Variables:** `={{ ($json.messages || []).some(m => new Date(m.date_sent) >= new Date($('Prepare the chase').first().json.started_at)) }}`
  - **Connections:** Inputs: `Check for a reply 2`. Outputs: True branch connects to `Tell the owner they replied`. False branch connects to `Nudge 2`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Empty message object arrays.
  - **Sub-workflow Reference:** None.

- **Nudge 2**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends the second automated reminder SMS to the customer.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses `nudge_2` template variables.
  - **Connections:** Inputs: `Replied before 2?` (False branch). Outputs: Connects to `Wait for nudge 3`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio API delivery failures.
  - **Sub-workflow Reference:** None.

#### 2.5 Third Follow-Up Cycle & Completion
This final cycle handles the third reminder, checks for a final response, sends the last nudge, and closes out the chase sequence if no reply is received.

- **Wait for nudge 3**
  - **Type & Technical Role:** Wait node (`n8n-nodes-base.wait`). Pauses execution until the timestamp for the final reminder.
  - **Configuration:** Resume set to `specificTime`.
  - **Expressions/Variables:** `={{ $('Prepare the chase').first().json.nudge_3_at }}`
  - **Connections:** Inputs: `Nudge 2`. Outputs: Connects to `Check for a reply 3`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failure Types:** Execution state loss during the wait period.
  - **Sub-workflow Reference:** None.

- **Check for a reply 3**
  - **Type & Technical Role:** HTTP Request node (`n8n-nodes-base.httpRequest`). Queries the Twilio API for any inbound customer messages prior to the final nudge.
  - **Configuration:** GET request utilizing Twilio credentials and query filters.
  - **Expressions/Variables:** Dynamic URL construction for account SID.
  - **Connections:** Inputs: `Wait for nudge 3`. Outputs: Connects to `Replied before 3?`.
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failure Types:** Authentication token expiration or revocation.
  - **Sub-workflow Reference:** None.

- **Replied before 3?**
  - **Type & Technical Role:** If node (`n8n-nodes-base.if`). Verifies if the customer replied before the final message is dispatched.
  - **Configuration:** Loose type validation.
  - **Expressions/Variables:** `={{ ($json.messages || []).some(m => new Date(m.date_sent) >= new Date($('Prepare the chase').first().json.started_at)) }}`
  - **Connections:** Inputs: `Check for a reply 3`. Outputs: True branch connects to `Tell the owner they replied`. False branch connects to `Nudge 3`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases & Failure Types:** Malformed date objects in message comparisons.
  - **Sub-workflow Reference:** None.

- **Nudge 3**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends the third and final reminder SMS to the customer.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses `nudge_3` template string.
  - **Connections:** Inputs: `Replied before 3?` (False branch). Outputs: Connects to `Tell the owner it closed`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio delivery errors.
  - **Sub-workflow Reference:** None.

- **Tell the owner it closed**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends a closing SMS notification to the business owner indicating that all three nudges were sent without a customer response.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses `owner_cell`, `business_number`, and `owner_done` variables.
  - **Connections:** Inputs: `Nudge 3`. Outputs: None (Terminal node).
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio API transmission failure.
  - **Sub-workflow Reference:** None.

#### 2.6 Early Termination Handling
This section manages the workflow branch executed when a customer reply is detected during any of the validation check phases.

- **Tell the owner they replied**
  - **Type & Technical Role:** Twilio node (`n8n-nodes-base.twilio`). Sends an alert SMS to the business owner indicating that the customer has replied, stopping any further automated nudges.
  - **Configuration:** Resource set to `sms`, operation set to `send`.
  - **Expressions/Variables:** Uses `owner_cell`, `business_number`, and `owner_replied` variables.
  - **Connections:** Inputs: `Replied before 1?`, `Replied before 2?`, and `Replied before 3?` (True branches). Outputs: None (Terminal node).
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failure Types:** Twilio gateway connection failures.
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Quote sent | n8n-nodes-base.webhook | Receives incoming quote details via POST webhook | None | Your settings | ## Chase unanswered quotes by SMS, and stop the moment they reply<br><br>**Who it is for:** service businesses that send quotes and lose jobs to silence. Plumbers, HVAC, roofers, cleaners, contractors.<br><br>**How it works**<br>1. Post a quote to the webhook with a phone, name, amount, job and an optional quote link.<br>2. You get a text saying the chase has started.<br>3. Up to three nudges go out, by default 2, 5 and 9 days later, each at 10am in your time zone.<br>4. Before every nudge it asks Twilio whether the customer has texted your number since the quote went out. If they have, the chase stops and you get told.<br>5. If they never reply, you get a text when the chase closes.<br><br>There is no database to set up. Twilio already knows whether they replied.<br><br>**Setup steps**<br>1. Select your Twilio credential on the five Twilio nodes and the three reply checks.<br>2. Fill in **Your settings**: business name, your Twilio number, your cell, your time zone. Edit the nudge wording if you like.<br>3. Activate the workflow and POST to the production URL of **Quote sent**. |
| Your settings | n8n-nodes-base.set | Defines business name, phone numbers, time zone, and nudge settings | Quote sent | Prepare the chase | ## Chase unanswered quotes by SMS, and stop the moment they reply<br><br>**Who it is for:** service businesses that send quotes and lose jobs to silence. Plumbers, HVAC, roofers, cleaners, contractors.<br><br>**How it works**<br>1. Post a quote to the webhook with a phone, name, amount, job and an optional quote link.<br>2. You get a text saying the chase has started.<br>3. Up to three nudges go out, by default 2, 5 and 9 days later, each at 10am in your time zone.<br>4. Before every nudge it asks Twilio whether the customer has texted your number since the quote went out. If they have, the chase stops and you get told.<br>5. If they never reply, you get a text when the chase closes.<br><br>There is no database to set up. Twilio already knows whether they replied.<br><br>**Setup steps**<br>1. Select your Twilio credential on the five Twilio nodes and the three reply checks.<br>2. Fill in **Your settings**: business name, your Twilio number, your cell, your time zone. Edit the nudge wording if you like.<br>3. Activate the workflow and POST to the production URL of **Quote sent**. |
| Prepare the chase | n8n-nodes-base.code | Normalizes payload data, formats numbers, and calculates schedule timestamps | Your settings | Answer the webhook | ## Chase unanswered quotes by SMS, and stop the moment they reply<br><br>**Who it is for:** service businesses that send quotes and lose jobs to silence. Plumbers, HVAC, roofers, cleaners, contractors.<br><br>**How it works**<br>1. Post a quote to the webhook with a phone, name, amount, job and an optional quote link.<br>2. You get a text saying the chase has started.<br>3. Up to three nudges go out, by default 2, 5 and 9 days later, each at 10am in your time zone.<br>4. Before every nudge it asks Twilio whether the customer has texted your number since the quote went out. If they have, the chase stops and you get told.<br>5. If they never reply, you get a text when the chase closes.<br><br>There is no database to set up. Twilio already knows whether they replied.<br><br>**Setup steps**<br>1. Select your Twilio credential on the five Twilio nodes and the three reply checks.<br>2. Fill in **Your settings**: business name, your Twilio number, your cell, your time zone. Edit the nudge wording if you like.<br>3. Activate the workflow and POST to the production URL of **Quote sent**. |
| Answer the webhook | n8n-nodes-base.respondToWebhook | Returns immediate JSON success response to webhook caller | Prepare the chase | Usable phone? | ### 1. Receive the quote<br>Answers the webhook straight away, then runs the chase in the background. A missing or bad phone number stops it here. |
| Usable phone? | n8n-nodes-base.if | Verifies if phone number and configuration are valid | Answer the webhook | Tell the owner (True) | ### 1. Receive the quote<br>Answers the webhook straight away, then runs the chase in the background. A missing or bad phone number stops it here. |
| Tell the owner | n8n-nodes-base.twilio | Sends initial SMS alert to business owner | Usable phone? | Wait for nudge 1 | ### 1. Receive the quote<br>Answers the webhook straight away, then runs the chase in the background. A missing or bad phone number stops it here. |
| Wait for nudge 1 | n8n-nodes-base.wait | Pauses execution until the scheduled first nudge time | Tell the owner | Check for a reply 1 | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Check for a reply 1 | n8n-nodes-base.httpRequest | Queries Twilio API for inbound customer messages | Wait for nudge 1 | Replied before 1? | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Replied before 1? | n8n-nodes-base.if | Checks if a customer reply was received since chase started | Check for a reply 1 | Tell the owner they replied (True), Nudge 1 (False) | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Nudge 1 | n8n-nodes-base.twilio | Sends the first reminder SMS to the customer | Replied before 1? | Wait for nudge 2 | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Wait for nudge 2 | n8n-nodes-base.wait | Pauses execution until the scheduled second nudge time | Nudge 1 | Check for a reply 2 | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Check for a reply 2 | n8n-nodes-base.httpRequest | Queries Twilio API for inbound customer messages | Wait for nudge 2 | Replied before 2? | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Replied before 2? | n8n-nodes-base.if | Checks if a customer reply was received before second nudge | Check for a reply 2 | Tell the owner they replied (True), Nudge 2 (False) | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Nudge 2 | n8n-nodes-base.twilio | Sends the second reminder SMS to the customer | Replied before 2? | Wait for nudge 3 | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Wait for nudge 3 | n8n-nodes-base.wait | Pauses execution until the scheduled third nudge time | Nudge 2 | Check for a reply 3 | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Check for a reply 3 | n8n-nodes-base.httpRequest | Queries Twilio API for inbound customer messages | Wait for nudge 3 | Replied before 3? | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Replied before 3? | n8n-nodes-base.if | Checks if a customer reply was received before third nudge | Check for a reply 3 | Tell the owner they replied (True), Nudge 3 (False) | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Nudge 3 | n8n-nodes-base.twilio | Sends the third reminder SMS to the customer | Replied before 3? | Tell the owner it closed | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Tell the owner it closed | n8n-nodes-base.twilio | Sends final SMS notification that chase has closed | Nudge 3 | None | ### 3b. No answer<br>Three nudges, nothing back. You hear about it and the chase closes. |
| Tell the owner they replied | n8n-nodes-base.twilio | Alerts business owner that customer has replied, stopping chase | Replied before 1?, Replied before 2?, Replied before 3? | None | ### 3a. They replied<br>The chase stops and the conversation is yours. |
| Note 0cd41c | n8n-nodes-base.stickyNote | Overview and setup instructions | None | None | ## Chase unanswered quotes by SMS, and stop the moment they reply<br><br>**Who it is for:** service businesses that send quotes and lose jobs to silence. Plumbers, HVAC, roofers, cleaners, contractors.<br><br>**How it works**<br>1. Post a quote to the webhook with a phone, name, amount, job and an optional quote link.<br>2. You get a text saying the chase has started.<br>3. Up to three nudges go out, by default 2, 5 and 9 days later, each at 10am in your time zone.<br>4. Before every nudge it asks Twilio whether the customer has texted your number since the quote went out. If they have, the chase stops and you get told.<br>5. If they never reply, you get a text when the chase closes.<br><br>There is no database to set up. Twilio already knows whether they replied.<br><br>**Setup steps**<br>1. Select your Twilio credential on the five Twilio nodes and the three reply checks.<br>2. Fill in **Your settings**: business name, your Twilio number, your cell, your time zone. Edit the nudge wording if you like.<br>3. Activate the workflow and POST to the production URL of **Quote sent**. |
| Note 10ff86 | n8n-nodes-base.stickyNote | Explains input ingestion block | None | None | ### 1. Receive the quote<br>Answers the webhook straight away, then runs the chase in the background. A missing or bad phone number stops it here. |
| Note d925a7 | n8n-nodes-base.stickyNote | Explains wait, check, and nudge cycles | None | None | ### 2. Wait, check, nudge<br>Three rounds. Each one waits until 10am on the day, asks Twilio for any text from the customer since the quote went out, and only sends a nudge if there is none. |
| Note de494e | n8n-nodes-base.stickyNote | Explains early termination on reply | None | None | ### 3a. They replied<br>The chase stops and the conversation is yours. |
| Note e14ed5 | n8n-nodes-base.stickyNote | Explains closing action when no answer is received | None | None | ### 3b. No answer<br>Three nudges, nothing back. You hear about it and the chase closes. |
| Note 76ab9c | n8n-nodes-base.stickyNote | Testing tips for wait intervals | None | None | **Testing tip:** set the three waits to 0.002 (about three minutes) in **Your settings**, use your own cell as the customer, and reply to a nudge to watch it stop. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create the Webhook Node**
   - Add a **Webhook** node named `Quote sent`.
   - Set HTTP Method to `POST`.
   - Set Path to `quote-chase`.
   - Set Response Mode to `Response Node`.

2. **Create the Settings Node**
   - Add an **Edit Fields (Set)** node named `Your settings`.
   - Connect `Quote sent` output to `Your settings`.
   - Add string assignments: `business_name` (`Your Business`), `business_number` (`+1234567890`), `owner_cell` (`+1234567890`), `timezone` (`America/New_York`), `nudge_1_text`, `nudge_2_text`, and `nudge_3_text` with appropriate templates.
   - Add number assignments: `wait_1_days` (`2`), `wait_2_days` (`3`), `wait_3_days` (`4`).

3. **Create the Preparation Code Node**
   - Add a **Code** node named `Prepare the chase`.
   - Connect `Your settings` output to `Prepare the chase`.
   - Insert the JavaScript code logic to normalize input properties, parse phone numbers to E.164 format, build template replacements, and compute UTC timestamps based on the configured timezone.

4. **Create the Webhook Response Node**
   - Add a **Respond to Webhook** node named `Answer the webhook`.
   - Connect `Prepare the chase` output to `Answer the webhook`.
   - Set Response Body to JSON: `={{ { "ok": true } }}`.

5. **Create the Validation Node**
   - Add an **If** node named `Usable phone?`.
   - Connect `Answer the webhook` output to `Usable phone?`.
   - Set condition expression to check `={{ $('Prepare the chase').first().json.ok === true }}`.

6. **Create the Initial Notification Node**
   - Add a **Twilio** node named `Tell the owner`.
   - Connect the True output of `Usable phone?` to `Tell the owner`.
   - Configure resource to `sms` and operation to `send`. Set To, From, and Message parameters using expressions referencing `Prepare the chase` output fields.
   - Configure your **Twilio account credentials**.

7. **Build the First Follow-Up Cycle**
   - **Wait Node:** Add a **Wait** node named `Wait for nudge 1`. Set resume condition to `specificTime` with expression `={{ $('Prepare the chase').first().json.nudge_1_at }}`. Connect `Tell the owner` to this node.
   - **HTTP Request Node:** Add an **HTTP Request** node named `Check for a reply 1`. Set method to `GET`, URL to the Twilio Messages endpoint using account SID, and authentication to predefined Twilio credentials. Configure query parameters for `From`, `To`, `DateSent>`, and `PageSize`. Connect `Wait for nudge 1` to this node.
   - **If Node:** Add an **If** node named `Replied before 1?`. Configure expression to check array messages: `={{ ($json.messages || []).some(m => new Date(m.date_sent) >= new Date($('Prepare the chase').first().json.started_at)) }}`. Connect `Check for a reply 1` to this node.
   - **Twilio Node (Nudge):** Add a **Twilio** node named `Nudge 1`. Connect the False output of `Replied before 1?` to `Nudge 1`. Set To, From, and Message using `Prepare the chase` data.
   - **Twilio Node (Reply Alert):** Add a **Twilio** node named `Tell the owner they replied`. Connect the True output of `Replied before 1?` to this node. Set message to `owner_replied`.

8. **Build the Second Follow-Up Cycle**
   - **Wait Node:** Add a **Wait** node named `Wait for nudge 2` (resume at `nudge_2_at`). Connect `Nudge 1` to this node.
   - **HTTP Request Node:** Add an **HTTP Request** node named `Check for a reply 2` configured identically to the first check. Connect `Wait for nudge 2` to this node.
   - **If Node:** Add an **If** node named `Replied before 2?` checking message timestamps. Connect `Check for a reply 2` to this node.
   - **Twilio Node (Nudge):** Add a **Twilio** node named `Nudge 2`. Connect the False output of `Replied before 2?` to `Nudge 2`.
   - **Twilio Node (Reply Alert):** Connect the True output of `Replied before 2?` to the existing `Tell the owner they replied` node.

9. **Build the Third Follow-Up Cycle & Completion**
   - **Wait Node:** Add a **Wait** node named `Wait for nudge 3` (resume at `nudge_3_at`). Connect `Nudge 2` to this node.
   - **HTTP Request Node:** Add an **HTTP Request** node named `Check for a reply 3` configured identically to previous checks. Connect `Wait for nudge 3` to this node.
   - **If Node:** Add an **If** node named `Replied before 3?` checking message timestamps. Connect `Check for a reply 3` to this node.
   - **Twilio Node (Nudge):** Add a **Twilio** node named `Nudge 3`. Connect the False output of `Replied before 3?` to `Nudge 3`.
   - **Twilio Node (Reply Alert):** Connect the True output of `Replied before 3?` to the existing `Tell the owner they replied` node.
   - **Twilio Node (Closed Alert):** Add a **Twilio** node named `Tell the owner it closed`. Connect `Nudge 3` output to this node. Set message to `owner_done`.

10. **Finalize Credentials & Testing**
    - Ensure all Twilio nodes and HTTP Request reply checks share the same configured **Twilio account credential** (Account SID and Auth Token).
    - Activate the workflow and configure your quoting system to POST payloads to the webhook production URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for service businesses sending quotes (plumbers, HVAC, roofers, cleaners, contractors) to prevent lost deals due to customer silence. | Workflow Use Case Overview |
| Uses Twilio message history as an implicit state database rather than requiring an external database setup. | Architecture Design Note |
| Recommended testing tip: Set wait intervals to `0.002` (approx. 3 minutes) in **Your settings**, use your own cell as the customer number, and reply to verify automated shutdown. | Testing & Quality Assurance |