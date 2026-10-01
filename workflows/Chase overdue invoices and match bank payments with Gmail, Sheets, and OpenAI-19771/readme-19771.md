Chase overdue invoices and match bank payments with Gmail, Sheets, and OpenAI

https://n8nworkflows.xyz/workflows/chase-overdue-invoices-and-match-bank-payments-with-gmail--sheets--and-openai-19771


# Chase overdue invoices and match bank payments with Gmail, Sheets, and OpenAI

### 1. Workflow Overview

This workflow automates Accounts Receivable (AR) management, combining bank payment reconciliation, customer communication, intelligent reply processing, and escalation workflows. It operates across multiple daily schedules and event triggers to process financial records stored in Google Sheets, communicate via Gmail, and use OpenAI for intent classification.

The logic is grouped into the following functional blocks:

- **1.1 Daily Chase Initiation & Ledger Ingestion:** Scheduled execution that loads customer profiles, open invoices, payment history, and the chase log from Google Sheets to evaluate collection actions.
- **1.2 Collection Decision & Customer Routing:** Evaluates customer aging, active payment promises, credit limits, and grace periods to determine whether to send a statement, escalate to an account manager, request collections approval, or take no action.
- **1.3 Payment Reconciliation & Matching:** A separate daily schedule that analyzes bank statement lines against open invoices using reference parsing, exact-amount matching, and multi-invoice combination rules, updating balances and flagging exceptions.
- **1.4 Inbound Reply Processing & AI Intent Extraction:** Polls Gmail for customer responses, uses an OpenAI model via LangChain to extract intent and entities, filters automated responses, and updates dispute or promise statuses in Google Sheets.
- **1.5 Error Handling:** Captures runtime workflow failures and alerts the finance team.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Chase Initiation & Ledger Ingestion
- **Overview:** Initializes the morning collection process on weekdays at 9:00 AM, sets overarching policy parameters, and ingests structural financial data from four Google Sheets tabs, aggregating rows into single payload arrays for downstream evaluation.
- **Nodes Involved:** 
  - `Daily Chase Trigger`
  - `Set Collection Policy`
  - `Read Customers from Sheets`
  - `Aggregate Customer Data`
  - `Read Invoices from Sheets`
  - `Aggregate Invoice Data`
  - `Read Payments from Sheets`
  - `Aggregate Payment Data`
  - `Read Chase Log from Sheets`
  - `Aggregate Chase History`
- **Node Details:**
  - **Daily Chase Trigger** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers the collection workflow at 9:00 AM, Monday through Friday (`0 9 * * 1-5`).
    - *Inputs:* None. *Outputs:* Triggers `Set Collection Policy`.
  - **Set Collection Policy** (`n8n-nodes-base.set`)
    - *Role:* Establishes execution configuration variables (sheet ID, stakeholder emails, grace days, thresholds, date formatting rules).
    - *Key Expressions:* Assigns static string and numeric parameters (`sheet_id`, `ar_clerk_email`, `finance_email`, `grace_days`, `chase_floor`, etc.).
    - *Inputs:* `Daily Chase Trigger`. *Outputs:* `Read Customers from Sheets`.
  - **Read Customers from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads customer master data from the "Customers" tab.
    - *Configuration:* Document ID bound dynamically to `{{ $('Set Collection Policy').first().json.sheet_id }}`.
    - *Inputs:* `Set Collection Policy`. *Outputs:* `Aggregate Customer Data`.
  - **Aggregate Customer Data** (`n8n-nodes-base.aggregate`)
    - *Role:* Folds individual row items into a single array under the field `rows`.
    - *Inputs:* `Read Customers from Sheets`. *Outputs:* `Read Invoices from Sheets`.
  - **Read Invoices from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads open and historical invoices from the "Invoices" tab.
    - *Inputs:* `Aggregate Customer Data`. *Outputs:* `Aggregate Invoice Data`.
  - **Aggregate Invoice Data** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates invoice rows into an array.
    - *Inputs:* `Read Invoices from Sheets`. *Outputs:* `Read Payments from Sheets`.
  - **Read Payments from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads recorded payment allocations from the "Payments" tab.
    - *Inputs:* `Aggregate Invoice Data`. *Outputs:* `Aggregate Payment Data`.
  - **Aggregate Payment Data** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates payment rows into an array.
    - *Inputs:* `Read Payments from Sheets`. *Outputs:* `Read Chase Log from Sheets`.
  - **Read Chase Log from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads historical contact records from the "Chase log" tab.
    - *Inputs:* `Aggregate Payment Data`. *Outputs:* `Aggregate Chase History`.
  - **Aggregate Chase History** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates chase history records into an array.
    - *Inputs:* `Read Chase Log from Sheets`. *Outputs:* `Decide Collection Actions`.

#### 2.2 Collection Decision & Customer Routing
- **Overview:** Executes core business logic to determine customer-level collection actions, loops through accounts sequentially, and routes execution based on policy outcomes (email statements, escalate internally, seek collections approval, or skip).
- **Nodes Involved:**
  - `Decide Collection Actions`
  - `Loop Over Customers`
  - `Route Collection Action`
  - `Email Overdue Statement`
  - `Set Chase Recorded`
  - `Alert the Account Manager`
  - `Set Escalation Recorded`
  - `Ask Before Collections Handover`
  - `Check Handover Decision`
  - `Set Handover Agreed`
  - `Set Handover Declined`
  - `Append to Chase Log`
  - `Nothing to Chase Today`
- **Node Details:**
  - **Decide Collection Actions** (`n8n-nodes-base.code`)
    - *Role:* Analyzes customer credit exposure, unallocated cash, active promises, disputes, and historical behavior to calculate an escalation ladder level and assign an action (`chase`, `escalate`, `handover`, or `none`).
    - *Inputs:* `Aggregate Chase History`. *Outputs:* `Loop Over Customers`.
  - **Loop Over Customers** (`n8n-nodes-base.splitInBatches`)
    - *Role:* Iterates through processed customer decisions one batch at a time (`batchSize: 1`).
    - *Inputs:* `Decide Collection Actions`, `Append to Chase Log`, `Nothing to Chase Today`. *Outputs:* `Route Collection Action`.
  - **Route Collection Action** (`n8n-nodes-base.switch`)
    - *Role:* Directs workflow execution based on the `action` property (`chase`, `escalate`, `handover`, or fallback).
    - *Inputs:* `Loop Over Customers`. *Outputs:* `Email Overdue Statement`, `Alert the Account Manager`, `Ask Before Collections Handover`, `Nothing to Chase Today`.
  - **Email Overdue Statement** (`n8n-nodes-base.gmail`)
    - *Role:* Sends a consolidated overdue statement to the customer.
    - *Configuration:* Uses Gmail OAuth2 credentials. Sends to `{{ $json.customer_email }}` with a dynamic subject and message body.
    - *Inputs:* `Route Collection Action` (Chase branch). *Outputs:* `Set Chase Recorded`.
  - **Set Chase Recorded** (`n8n-nodes-base.set`)
    - *Role:* Appends log metadata indicating a chase action was executed.
    - *Inputs:* `Email Overdue Statement`. *Outputs:* `Append to Chase Log`.
  - **Alert the Account Manager** (`n8n-nodes-base.gmail`)
    - *Role:* Escalates accounts exceeding credit limits, in dispute, or missing email addresses to the account manager.
    - *Inputs:* `Route Collection Action` (Needs a colleague branch). *Outputs:* `Set Escalation Recorded`.
  - **Set Escalation Recorded** (`n8n-nodes-base.set`)
    - *Role:* Prepares log metadata for internal account manager escalation.
    - *Inputs:* `Alert the Account Manager`. *Outputs:* `Append to Chase Log`.
  - **Ask Before Collections Handover** (`n8n-nodes-base.gmail`)
    - *Role:* Uses Gmail's `sendAndWait` operation with a 3-day wait window to request finance lead approval before legal collections handover.
    - *Inputs:* `Route Collection Action` (Collections branch). *Outputs:* `Check Handover Decision`.
  - **Check Handover Decision** (`n8n-nodes-base.if`)
    - *Role:* Evaluates whether the finance lead approved (`{{ $json.data.approved === true }}`) or rejected the handover.
    - *Inputs:* `Ask Before Collections Handover`. *Outputs:* `Set Handover Agreed`, `Set Handover Declined`.
  - **Set Handover Agreed** (`n8n-nodes-base.set`)
    - *Role:* Records approval for collections handover.
    - *Inputs:* `Check Handover Decision` (True branch). *Outputs:* `Append to Chase Log`.
  - **Set Handover Declined** (`n8n-nodes-base.set`)
    - *Role:* Records refusal of collections handover.
    - *Inputs:* `Check Handover Decision` (False branch). *Outputs:* `Append to Chase Log`.
  - **Append to Chase Log** (`n8n-nodes-base.googleSheets`)
    - *Role:* Writes the execution result (action, level, reason, amounts) back to the "Chase log" tab.
    - *Inputs:* `Set Chase Recorded`, `Set Escalation Recorded`, `Set Handover Agreed`, `Set Handover Declined`. *Outputs:* `Loop Over Customers` (to fetch the next batch).
  - **Nothing to Chase Today** (`n8n-nodes-base.noOp`)
    - *Role:* Handles accounts requiring no action, routing them back to the loop without writing to the log.
    - *Inputs:* `Route Collection Action` (Fallback branch). *Outputs:* `Loop Over Customers`.

#### 2.3 Payment Reconciliation & Matching
- **Overview:** Runs on weekday mornings at 7:00 AM to parse bank statement rows, apply matching algorithms against open invoices, update ledger balances, and route unallocated items to exceptions reporting.
- **Nodes Involved:**
  - `Daily Payment Trigger`
  - `Set Matching Rules`
  - `Read Bank Payments from Sheets`
  - `Aggregate Bank Payments`
  - `Read Open Invoices from Sheets`
  - `Aggregate Open Invoices`
  - `Allocate Payments to Invoices`
  - `Route Allocation Result`
  - `Update Invoice Balances`
  - `Mark Payment Allocated`
  - `Aggregate Payment Exceptions`
  - `Email Unmatched Payments`
- **Node Details:**
  - **Daily Payment Trigger** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers payment matching at 7:00 AM, Monday through Friday (`0 7 * * 1-5`).
    - *Inputs:* None. *Outputs:* `Set Matching Rules`.
  - **Set Matching Rules** (`n8n-nodes-base.set`)
    - *Role:* Defines matching parameters, fee tolerances, combination limits, and sheet identifiers.
    - *Inputs:* `Daily Payment Trigger`. *Outputs:* `Read Bank Payments from Sheets`.
  - **Read Bank Payments from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads unallocated bank transactions from the "Payments" tab.
    - *Inputs:* `Set Matching Rules`. *Outputs:* `Aggregate Bank Payments`.
  - **Aggregate Bank Payments** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates bank payment rows into an array.
    - *Inputs:* `Read Bank Payments from Sheets`. *Outputs:* `Read Open Invoices from Sheets`.
  - **Read Open Invoices from Sheets** (`n8n-nodes-base.googleSheets`)
    - *Role:* Reads open invoices from the "Invoices" tab.
    - *Inputs:* `Aggregate Bank Payments`. *Outputs:* `Aggregate Open Invoices`.
  - **Aggregate Open Invoices** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates open invoices into an array.
    - *Inputs:* `Read Open Invoices from Sheets`. *Outputs:* `Allocate Payments to Invoices`.
  - **Allocate Payments to Invoices** (`n8n-nodes-base.code`)
    - *Role:* Core matching engine. Attempts to match bank deposits to open invoices via reference string analysis, exact amount matching, multi-invoice combination calculation, or oldest-first fallback.
    - *Inputs:* `Aggregate Open Invoices`. *Outputs:* `Route Allocation Result`.
  - **Route Allocation Result** (`n8n-nodes-base.switch`)
    - *Role:* Routes matching outcomes based on item kind (`allocation`, `payment`, or `exception`).
    - *Inputs:* `Allocate Payments to Invoices`. *Outputs:* `Update Invoice Balances`, `Mark Payment Allocated`, `Aggregate Payment Exceptions`.
  - **Update Invoice Balances** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates invoice statuses, paid amounts, and payment references in the "Invoices" tab using upsert logic matching on `invoice_number`.
    - *Inputs:* `Route Allocation Result` (Invoice lines branch). *Outputs:* None (terminal sheet write).
  - **Mark Payment Allocated** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates transaction allocation status and matched invoice references in the "Payments" tab using upsert logic matching on `bank_ref`.
    - *Inputs:* `Route Allocation Result` (Bank lines branch). *Outputs:* None (terminal sheet write).
  - **Aggregate Payment Exceptions** (`n8n-nodes-base.aggregate`)
    - *Role:* Consolidates unmatched or ambiguous payment exceptions into a single list.
    - *Inputs:* `Route Allocation Result` (Needs a person branch). *Outputs:* `Email Unmatched Payments`.
  - **Email Unmatched Payments** (`n8n-nodes-base.gmail`)
    - *Role:* Emails a consolidated exception report to the AR clerk.
    - *Inputs:* `Aggregate Payment Exceptions`. *Outputs:* None.

#### 2.4 Inbound Reply Processing & AI Intent Extraction
- **Overview:** Polls Gmail every 15 minutes for customer replies, evaluates automated response headers, uses an OpenAI LLM to extract intent and entities, updates Google Sheets (promises and disputes), and routes human-in-the-loop items or internal alerts.
- **Nodes Involved:**
  - `When a Reply Arrives`
  - `Set Reply Rules`
  - `Extract Reply Intent`
  - `OpenAI GPT-4 Mini`
  - `Read What the Reply Says`
  - `Route Reply Intent`
  - `Write Promised Date`
  - `Flag Invoice Disputed`
  - `Report the Dispute Internally`
  - `Ask AR to Follow Up`
  - `No Action on Reply`
- **Node Details:**
  - **When a Reply Arrives** (`n8n-nodes-base.gmailTrigger`)
    - *Role:* Polls Gmail inbox every 15 minutes for relevant incoming collection replies.
    - *Configuration:* Polls messages with query `in:inbox -from:me newer_than:30d subject:(invoice OR overdue OR payment OR reminder)`.
    - *Inputs:* None. *Outputs:* `Set Reply Rules`.
  - **Set Reply Rules** (`n8n-nodes-base.set`)
    - *Role:* Sets configuration parameters (sheet ID, stakeholder emails, max promise days, date formatting) for reply handling.
    - *Inputs:* `When a Reply Arrives`. *Outputs:* `Extract Reply Intent`.
  - **Extract Reply Intent** (`@n8n/n8n-nodes-langchain.informationExtractor`)
    - *Role:* Uses structured LLM output generation to extract customer intent (`promise`, `dispute`, `paid`, `copy`, `other`), promised dates, invoice numbers, payment references, and summary notes.
    - *Inputs:* `Set Reply Rules` and `OpenAI GPT-4 Mini`. *Outputs:* `Read What the Reply Says`.
  - **OpenAI GPT-4 Mini** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
    - *Role:* Language model provider configuration for intent extraction.
    - *Inputs:* None. *Outputs:* Connected to `Extract Reply Intent` via `ai_languageModel`.
  - **Read What the Reply Says** (`n8n-nodes-base.code`)
    - *Role:* Validates email headers (RFC 3834 auto-responder detection), parses extracted dates, normalizes invoice numbers, and classifies actions (`promise`, `dispute`, `follow-up`, or `ignore`).
    - *Inputs:* `Extract Reply Intent`. *Outputs:* `Route Reply Intent`.
  - **Route Reply Intent** (`n8n-nodes-base.switch`)
    - *Role:* Directs workflow execution based on the processed reply action.
    - *Inputs:* `Read What the Reply Says`. *Outputs:* `Write Promised Date`, `Flag Invoice Disputed`, `Ask AR to Follow Up`, `No Action on Reply`.
  - **Write Promised Date** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates the invoice row in the "Invoices" tab with the new promised date and reply timestamp.
    - *Inputs:* `Route Reply Intent` (A date to pay branch). *Outputs:* None.
  - **Flag Invoice Disputed** (`n8n-nodes-base.googleSheets`)
    - *Role:* Updates invoice status to `disputed` and records dispute reasons in the "Invoices" tab.
    - *Inputs:* `Route Reply Intent` (A dispute branch). *Outputs:* `Report the Dispute Internally`.
  - **Report the Dispute Internally** (`n8n-nodes-base.gmail`)
    - *Role:* Alerts the account manager regarding a newly registered customer dispute.
    - *Inputs:* `Flag Invoice Disputed`. *Outputs:* None.
  - **Ask AR to Follow Up** (`n8n-nodes-base.gmail`)
    - *Role:* Forwards complex customer replies (payment claims, document requests) to the AR clerk for manual handling.
    - *Inputs:* `Route Reply Intent` (Needs a person branch). *Outputs:* None.
  - **No Action on Reply** (`n8n-nodes-base.noOp`)
    - *Role:* Terminates execution cleanly for ignored automated responses (e.g., out-of-office autoreplies).
    - *Inputs:* `Route Reply Intent` (Fallback branch). *Outputs:* None.

#### 2.5 Error Handling
- **Overview:** Captures unhandled workflow execution exceptions and dispatches failure alerts to the finance director.
- **Nodes Involved:**
  - `Workflow Failure Trigger`
  - `Alert on Workflow Failure`
- **Node Details:**
  - **Workflow Failure Trigger** (`n8n-nodes-base.errorTrigger`)
    - *Role:* Triggers whenever any node in the workflow throws an unhandled error.
    - *Inputs:* None. *Outputs:* `Alert on Workflow Failure`.
  - **Alert on Workflow Failure** (`n8n-nodes-base.gmail`)
    - *Role:* Sends failure details (error message, failed node name, execution URL) to the finance director.
    - *Inputs:* `Workflow Failure Trigger`. *Outputs:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation & metadata | None | None | ## Chase overdue invoices by customer and match bank payments with Gmail, Sheets and AI<br><br>### How it works... |
| Step 1 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Start the weekday chase run<br><br>Nine in the morning, Monday to Friday. **Set Collection Policy** is the only node in this branch you edit... |
| Daily Chase Trigger | n8n-nodes-base.scheduleTrigger | Cron schedule trigger | None | Set Collection Policy | |
| Set Collection Policy | n8n-nodes-base.set | Assigns global collection variables | Daily Chase Trigger | Read Customers from Sheets | |
| Step 2 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Load customers, invoices and history<br><br>Four tabs, read once. Each **Aggregate** folds a read into a single item... |
| Read Customers from Sheets | n8n-nodes-base.googleSheets | Reads customer data tab | Set Collection Policy | Aggregate Customer Data | |
| Aggregate Customer Data | n8n-nodes-base.aggregate | Aggregates customer records | Read Customers from Sheets | Read Invoices from Sheets | |
| Read Invoices from Sheets | n8n-nodes-base.googleSheets | Reads invoice data tab | Aggregate Customer Data | Aggregate Invoice Data | |
| Aggregate Invoice Data | n8n-nodes-base.aggregate | Aggregates invoice records | Read Invoices from Sheets | Read Payments from Sheets | |
| Read Payments from Sheets | n8n-nodes-base.googleSheets | Reads payment records tab | Aggregate Invoice Data | Aggregate Payment Data | |
| Aggregate Payment Data | n8n-nodes-base.aggregate | Aggregates payment records | Read Payments from Sheets | Read Chase Log from Sheets | |
| Read Chase Log from Sheets | n8n-nodes-base.googleSheets | Reads chase history tab | Aggregate Payment Data | Aggregate Chase History | |
| Aggregate Chase History | n8n-nodes-base.aggregate | Aggregates chase history records | Read Chase Log from Sheets | Decide Collection Actions | |
| Step 3 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Decide who to chase today<br><br>The node this template exists for. One decision per customer rather than per invoice... |
| Decide Collection Actions | n8n-nodes-base.code | Evaluates collection rules per customer | Aggregate Chase History | Loop Over Customers | |
| Loop Over Customers | n8n-nodes-base.splitInBatches | Batches customer decision payloads | Decide Collection Actions, Append to Chase Log, Nothing to Chase Today | Route Collection Action | |
| Route Collection Action | n8n-nodes-base.switch | Routes based on collection action | Loop Over Customers | Email Overdue Statement, Alert the Account Manager, Ask Before Collections Handover, Nothing to Chase Today | |
| Step 4 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Send one statement per customer<br><br>One email covering everything they owe, in the words the ladder allows... |
| Email Overdue Statement | n8n-nodes-base.gmail | Sends overdue statement email | Route Collection Action | Set Chase Recorded | |
| Set Chase Recorded | n8n-nodes-base.set | Sets chase log action metadata | Email Overdue Statement | Append to Chase Log | |
| Step 5 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Escalate to the account manager<br><br>Over the credit limit, in dispute, or no address on file... |
| Alert the Account Manager | n8n-nodes-base.gmail | Sends internal escalation email | Route Collection Action | Set Escalation Recorded | |
| Set Escalation Recorded | n8n-nodes-base.set | Sets escalation log metadata | Alert the Account Manager | Append to Chase Log | |
| Step 6 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Ask before handing to collections<br><br>The one step this workflow will not take on its own. Finance gets the account's whole history... |
| Ask Before Collections Handover | n8n-nodes-base.gmail | Requests approval for collections handover | Route Collection Action | Check Handover Decision | |
| Check Handover Decision | n8n-nodes-base.if | Evaluates finance approval | Ask Before Collections Handover | Set Handover Agreed, Set Handover Declined | |
| Set Handover Agreed | n8n-nodes-base.set | Sets handover approval metadata | Check Handover Decision | Append to Chase Log | |
| Set Handover Declined | n8n-nodes-base.set | Sets handover refusal metadata | Check Handover Decision | Append to Chase Log | |
| Step 7 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Write the chase log<br><br>Every branch ends in one row: who, when, at what level, and why... |
| Append to Chase Log | n8n-nodes-base.googleSheets | Appends record to chase log | Set Chase Recorded, Set Escalation Recorded, Set Handover Agreed, Set Handover Declined | Loop Over Customers | |
| Nothing to Chase Today | n8n-nodes-base.noOp | Handles accounts requiring no action | Route Collection Action | Loop Over Customers | |
| Step 8 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Start the payment matching run<br><br>Seven in the morning, two hours before the chase run... |
| Daily Payment Trigger | n8n-nodes-base.scheduleTrigger | Cron schedule for bank matching | None | Set Matching Rules | |
| Set Matching Rules | n8n-nodes-base.set | Sets bank matching configuration | Daily Payment Trigger | Read Bank Payments from Sheets | |
| Step 9 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Read payments and open invoices<br><br>The bank lines your bank or your bookkeeper drops into the Payments tab... |
| Read Bank Payments from Sheets | n8n-nodes-base.googleSheets | Reads unallocated bank payments | Set Matching Rules | Aggregate Bank Payments | |
| Aggregate Bank Payments | n8n-nodes-base.aggregate | Aggregates bank payment rows | Read Bank Payments from Sheets | Read Open Invoices from Sheets | |
| Read Open Invoices from Sheets | n8n-nodes-base.googleSheets | Reads open invoice records | Aggregate Bank Payments | Aggregate Open Invoices | |
| Aggregate Open Invoices | n8n-nodes-base.aggregate | Aggregates open invoice rows | Read Open Invoices from Sheets | Allocate Payments to Invoices | |
| Step 10 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Match payments to invoices<br><br>By the reference, by an exact amount, or by the one combination... |
| Allocate Payments to Invoices | n8n-nodes-base.code | Reconciles bank payments to invoices | Aggregate Open Invoices | Route Allocation Result | |
| Route Allocation Result | n8n-nodes-base.switch | Routes allocation outputs | Allocate Payments to Invoices | Update Invoice Balances, Mark Payment Allocated, Aggregate Payment Exceptions | |
| Step 11 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Write the balances back<br><br>The invoice takes its new balance, the bank line takes what it paid for... |
| Update Invoice Balances | n8n-nodes-base.googleSheets | Updates invoice payment statuses | Route Allocation Result | None | |
| Mark Payment Allocated | n8n-nodes-base.googleSheets | Updates bank payment records | Route Allocation Result | None | |
| Step 12 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Report what needs a person<br><br>One email, holding only what could not be settled... |
| Aggregate Payment Exceptions | n8n-nodes-base.aggregate | Aggregates allocation exceptions | Route Allocation Result | Email Unmatched Payments | |
| Email Unmatched Payments | n8n-nodes-base.gmail | Emails unmatched payment exceptions | Aggregate Payment Exceptions | None | |
| Step 13 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Catch replies to the chasers<br><br>Every reply to a chaser, read within fifteen minutes... |
| When a Reply Arrives | n8n-nodes-base.gmailTrigger | Polls Gmail for incoming replies | None | Set Reply Rules | |
| Set Reply Rules | n8n-nodes-base.set | Sets reply handling parameters | When a Reply Arrives | Extract Reply Intent | |
| Step 14 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Read the reply with AI<br><br>The model is asked what the customer wants... |
| Extract Reply Intent | @n8n/n8n-nodes-langchain.informationExtractor | Extracts structured intent from emails | Set Reply Rules, OpenAI GPT-4 Mini | Read What the Reply Says | |
| OpenAI GPT-4 Mini | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM chat model provider | None | Extract Reply Intent | |
| Step 15 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Work out what it means<br><br>Whether a person wrote it at all is settled by the headers... |
| Read What the Reply Says | n8n-nodes-base.code | Parses intents, dates, and headers | Extract Reply Intent | Route Reply Intent | |
| Route Reply Intent | n8n-nodes-base.switch | Routes processed reply actions | Read What the Reply Says | Write Promised Date, Flag Invoice Disputed, Ask AR to Follow Up, No Action on Reply | |
| Step 16 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Apply the reply to Sheets<br><br>A promised date holds the next chaser back until it has passed... |
| Write Promised Date | n8n-nodes-base.googleSheets | Updates promised date on invoice | Route Reply Intent | None | |
| Flag Invoice Disputed | n8n-nodes-base.googleSheets | Marks invoice status as disputed | Route Reply Intent | Report the Dispute Internally | |
| Step 17 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Hand the rest to a person<br><br>Claims of payment, requests for a copy, and disputes worth a human answer... |
| Report the Dispute Internally | n8n-nodes-base.gmail | Emails internal dispute notification | Flag Invoice Disputed | None | |
| Ask AR to Follow Up | n8n-nodes-base.gmail | Forwards unhandled replies to AR clerk | Route Reply Intent | None | |
| No Action on Reply | n8n-nodes-base.noOp | Nop node for automated replies | Route Reply Intent | None | |
| Step 18 | n8n-nodes-base.stickyNote | Block annotation | None | None | ## Report a failure of the workflow<br><br>A fourth entry point, firing only when a production run errors... |
| Workflow Failure Trigger | n8n-nodes-base.errorTrigger | Triggers on workflow execution error | None | Alert on Workflow Failure | |
| Alert on Workflow Failure | n8n-nodes-base.gmail | Emails failure notification to finance | Workflow Failure Trigger | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow in n8n manually, follow these ordered steps:

1. **Prerequisites & Credentials:**
   - Configure a **Google OAuth2** credential for Google Sheets and Gmail nodes.
   - Configure an **OpenAI API** credential for the LangChain language model.
   - Set up a Google Sheets spreadsheet containing four tabs named: `Customers`, `Invoices`, `Payments`, and `Chase log`.

2. **Branch 1: Daily Collection Run (9:00 AM)**
   - Add a **Schedule Trigger** (`Daily Chase Trigger`): Set cron expression `0 9 * * 1-5`.
   - Add a **Set** node (`Set Collection Policy`): Configure parameters (`sheet_id`, `ar_clerk_email`, `finance_email`, `grace_days`, etc.). Set `executeOnce` to true.
   - Add Google Sheets nodes to read data sequentially:
     - `Read Customers from Sheets` (Sheet Name: `Customers`, Document ID: `={{ $('Set Collection Policy').first().json.sheet_id }}`) -> Connect to an **Aggregate** node (`Aggregate Customer Data`, field: `rows`).
     - `Read Invoices from Sheets` (Sheet Name: `Invoices`) -> Connect to **Aggregate** (`Aggregate Invoice Data`, field: `rows`).
     - `Read Payments from Sheets` (Sheet Name: `Payments`) -> Connect to **Aggregate** (`Aggregate Payment Data`, field: `rows`).
     - `Read Chase Log from Sheets` (Sheet Name: `Chase log`) -> Connect to **Aggregate** (`Aggregate Chase History`, field: `rows`).
   - Add a **Code** node (`Decide Collection Actions`) to evaluate customer aging, promises, disputes, and assign ladder levels.
   - Add a **Split In Batches** node (`Loop Over Customers`) with `batchSize: 1`.
   - Add a **Switch** node (`Route Collection Action`) with rules routing on `{{ $json.action }}` (`chase`, `escalate`, `handover`, fallback).
   - *Chase Branch:* Connect to a **Gmail** node (`Email Overdue Statement`), then a **Set** node (`Set Chase Recorded`), then a **Google Sheets** (`Append to Chase Log`, operation: `append`). Connect back to `Loop Over Customers`.
   - *Escalate Branch:* Connect to a **Gmail** node (`Alert the Account Manager`), then a **Set** node (`Set Escalation Recorded`), then `Append to Chase Log`. Connect back to `Loop Over Customers`.
   - *Handover Branch:* Connect to a **Gmail** node (`Ask Before Collections Handover`, operation `sendAndWait`, approval type double). Connect to an **If** node (`Check Handover Decision`, checking `{{ $json.data.approved === true }}`).
     - True branch: Connect to **Set** (`Set Handover Agreed`) -> `Append to Chase Log` -> `Loop Over Customers`.
     - False branch: Connect to **Set** (`Set Handover Declined`) -> `Append to Chase Log` -> `Loop Over Customers`.
   - *Fallback Branch:* Connect to a **No Operation** node (`Nothing to Chase Today`) -> `Loop Over Customers`.

3. **Branch 2: Daily Payment Matching Run (7:00 AM)**
   - Add a **Schedule Trigger** (`Daily Payment Trigger`): Set cron expression `0 7 * * 1-5`.
   - Add a **Set** node (`Set Matching Rules`): Define tolerances, sheet ID, and allocation rules.
   - Add **Google Sheets** readers & aggregators:
     - Read `Payments` tab -> Aggregate (`Aggregate Bank Payments`).
     - Read `Invoices` tab -> Aggregate (`Aggregate Open Invoices`).
   - Add a **Code** node (`Allocate Payments to Invoices`) implementing the reconciliation algorithm.
   - Add a **Switch** node (`Route Allocation Result`) routing on `{{ $json.kind }}` (`allocation`, `payment`, `exception`).
   - *Allocation branch:* Connect to **Google Sheets** (`Update Invoice Balances`, operation `appendOrUpdate`, matching on `invoice_number`).
   - *Payment branch:* Connect to **Google Sheets** (`Mark Payment Allocated`, operation `appendOrUpdate`, matching on `bank_ref`).
   - *Exception branch:* Connect to **Aggregate** (`Aggregate Payment Exceptions`), then a **Gmail** node (`Email Unmatched Payments`) addressed to `ar_clerk_email`.

4. **Branch 3: Inbound Reply Processing**
   - Add a **Gmail Trigger** (`When a Reply Arrives`): Polls every 15 minutes with query `in:inbox -from:me newer_than:30d subject:(invoice OR overdue OR payment OR reminder)`.
   - Add a **Set** node (`Set Reply Rules`).
   - Add an **Information Extractor** node (`Extract Reply Intent`) connected to an **OpenAI Chat Model** node (`OpenAI GPT-4 Mini`, model: `OpenAI GPT-4 Mini`, temperature: `0`).
   - Add a **Code** node (`Read What the Reply Says`) to process headers and classify intents.
   - Add a **Switch** node (`Route Reply Intent`) routing on `{{ $json.action }}` (`promise`, `dispute`, `follow-up`, fallback).
   - *Promise branch:* Connect to **Google Sheets** (`Write Promised Date`, operation `appendOrUpdate`, matching on `invoice_number`).
   - *Dispute branch:* Connect to **Google Sheets** (`Flag Invoice Disputed`, operation `appendOrUpdate`) -> **Gmail** (`Report the Dispute Internally`).
   - *Follow-up branch:* Connect to **Gmail** (`Ask AR to Follow Up`).
   - *Fallback branch:* Connect to **No Operation** (`No Action on Reply`).

5. **Branch 4: Error Handling**
   - Add an **Error Trigger** node (`Workflow Failure Trigger`).
   - Connect to a **Gmail** node (`Alert on Workflow Failure`) addressed to the finance director.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automated AR ledger, payment matching, and chase workflow | n8n workflow integration template |
| Template tags | finance, accounting, ai, sales |