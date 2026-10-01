Reconcile daily Stripe revenue with ApogeoAPI, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/reconcile-daily-stripe-revenue-with-apogeoapi--google-sheets--and-slack-19794


# Reconcile daily Stripe revenue with ApogeoAPI, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates the daily financial reconciliation of successful Stripe charges across multiple currencies, normalizes them into a single reporting currency using live exchange rates from ApogeoAPI, logs the ledger entry into Google Sheets, and triggers a Slack notification if exchange rates are stale or if any transactions fail conversion.

The workflow is structured into four primary functional blocks:

- **1.1 Input Reception & Triggering:** Handles execution initiation either through a daily automated schedule or via manual testing using pre-defined sample charge datasets.
- **1.2 Live Rate Retrieval & Data Normalization:** Fetches live foreign exchange rates, isolates yesterday’s successful Stripe transactions (accounting for refunds and non-standard currency decimals), and aggregates the totals into a unified reporting currency.
- **1.3 Ledger Recording:** Appends the final structured reconciliation metrics, breakdown details, and exception logs directly into a designated Google Sheets worksheet.
- **1.4 Anomaly Detection & Alerting:** Evaluates reconciliation health flags and automatically dispatches warning alerts to a designated Slack channel when discrepancies or stale FX data are identified.

---

### 2. Block-by-Block Analysis

---

### 1.1 Input Reception & Triggering

#### Overview
This block initiates the execution flow. It provides a dual-path entry mechanism: an automated daily cron-based trigger for production runs and a manual trigger paired with mock financial data for testing and validation.

#### Nodes Involved
- `When Daily at 9am` (`n8n-nodes-base.scheduleTrigger`)
- `Manual Execution Trigger` (`n8n-nodes-base.manualTrigger`)
- `Fetch Stripe Charges` (`n8n-nodes-base.stripe`)
- `Generate Sample Stripe Charges` (`n8n-nodes-base.code`)

#### Node Details

##### When Daily at 9am
- **Type and Technical Role:** Schedule Trigger node configured to execute the workflow every day at 09:00.
- **Configuration Choices:** Rule interval set to trigger at hour 9 on a daily basis.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: None (Trigger); Output: `Fetch Stripe Charges`.
- **Version-specific Requirements:** Version 1.2.
- **Edge Cases or Potential Failure Types:** Timezone mismatches if the n8n instance environment timezone differs from UTC expectations used downstream.

##### Manual Execution Trigger
- **Type and Technical Role:** Manual Trigger node allowing users to run the workflow on-demand inside the n8n UI.
- **Configuration Choices:** Default parameters.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: None (Trigger); Output: `Generate Sample Stripe Charges`.
- **Version-specific Requirements:** Version 1.0.
- **Edge Cases or Potential Failure Types:** None.

##### Fetch Stripe Charges
- **Type and Technical Role:** Stripe node configured to retrieve all successful or historical charge records from the connected account.
- **Configuration Choices:** Resource set to `charge`, operation set to `getAll`, `returnAll` set to `true`.
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Input: `When Daily at 9am`; Output: `Fetch Exchange Rates`.
- **Version-specific Requirements:** Version 1.0.
- **Edge Cases or Potential Failure Types:** Stripe API rate limits, invalid or expired Stripe credentials, or network timeouts when pulling large transaction volumes.
- **Credentials:** Requires Stripe API credentials (`REPLACE_WITH_YOUR_STRIPE_CRED`).

##### Generate Sample Stripe Charges
- **Type and Technical Role:** JavaScript Code node that builds an array of mock Stripe charges for testing manual execution paths.
- **Configuration Choices:** Custom JavaScript calculating timestamps for yesterday at noon UTC and defining multi-currency test payloads (including zero-decimal currencies like JPY/CLP and partially refunded items).
- **Key Expressions or Variables:** Uses native JavaScript `Date` and `Math` libraries.
- **Input and Output Connections:** Input: `Manual Execution Trigger`; Output: `Fetch Exchange Rates`.
- **Version-specific Requirements:** Version 2.0.
- **Edge Cases or Potential Failure Types:** Script execution errors if JavaScript syntax is modified incorrectly.

---

### 1.2 Live Rate Retrieval & Data Normalization

#### Overview
This block fetches comprehensive exchange rates from ApogeoAPI using USD as a base currency, evaluates previous-day boundaries in UTC, filters successful Stripe or sample charges net of refunds, applies decimal correction standards, and calculates aggregated financial totals.

#### Nodes Involved
- `Fetch Exchange Rates` (`n8n-nodes-apogeoapi.apogeoAPI`)
- `Normalize and Aggregate Data` (`n8n-nodes-base.code`)

#### Node Details

##### Fetch Exchange Rates
- **Type and Technical Role:** Community API node that requests current foreign exchange rates from ApogeoAPI.
- **Configuration Choices:** Resource set to `exchangeRate`, operation set to `listRates`, base currency set to `USD`. Configured to execute once per run (`executeOnce`).
- **Key Expressions or Variables:** None.
- **Input and Output Connections:** Inputs: `Fetch Stripe Charges`, `Generate Sample Stripe Charges`; Output: `Normalize and Aggregate Data`.
- **Version-specific Requirements:** Requires community node package `n8n-nodes-apogeoapi`.
- **Edge Cases or Potential Failure Types:** ApogeoAPI service outages, rate limit blocks, or expired credentials.
- **Credentials:** Requires ApogeoAPI credentials (`REPLACE_WITH_YOUR_CRED_ID`).

##### Normalize and Aggregate Data
- **Type and Technical Role:** JavaScript Code node that executes core financial transformation logic, handling date filtering, currency decimal adjustments, FX conversion, and summary metrics generation.
- **Configuration Choices:** Custom JavaScript processing incoming ApogeoAPI JSON, parsing Stripe or sample node payloads based on availability, applying zero- and three-decimal currency rules, and compiling structured reconciliation data.
- **Key Expressions or Variables:** 
  - `={{ $input.first().json }}`
  - `={{ $('Fetch Stripe Charges').all() }}`
  - `={{ $('Generate Sample Stripe Charges').all() }}`
- **Input and Output Connections:** Input: `Fetch Exchange Rates`; Outputs: `Append Data to Ledger Sheet`, `Check for Anomalies`.
- **Version-specific Requirements:** Version 2.0.
- **Edge Cases or Potential Failure Types:** Missing reporting currency rates in the ApogeoAPI payload will throw a manual script error. Malformed charge properties or missing timestamps may result in empty filtering arrays.

---

### 1.3 Ledger Recording

#### Overview
This block maps the consolidated financial summary and currency breakdowns generated by the normalization script and writes a new row into the specified Google Sheets reconciliation ledger.

#### Nodes Involved
- `Append Data to Ledger Sheet` (`n8n-nodes-base.googleSheets`)

#### Node Details

##### Append Data to Ledger Sheet
- **Type and Technical Role:** Google Sheets node that appends formatted rows to an existing spreadsheet.
- **Configuration Choices:** Operation set to `append`, mapping mode set to define columns below. Target document identified by ID and sheet name set to `Reconciliation`.
- **Key Expressions or Variables:**
  - Date: `={{ $json.date }}`
  - Breakdown: `={{ JSON.stringify($json.per_currency) }}`
  - Grand Total: `={{ $json.grand_total_reporting }}`
  - Rates Stale: `={{ $json.rates_stale }}`
  - Unconverted: `={{ JSON.stringify($json.unconverted) }}`
  - Charge Count: `={{ $json.charge_count }}`
  - USD/EUR Rate: `={{ $json.usd_eur_rate }}`
  - Reporting Currency: `={{ $json.reporting_currency }}`
- **Input and Output Connections:** Input: `Normalize and Aggregate Data`; Output: None.
- **Version-specific Requirements:** Version 4.5.
- **Edge Cases or Potential Failure Types:** Google API rate limits, permission errors on the target spreadsheet, or schema mismatches if sheet column headers do not match property keys.
- **Credentials:** Requires Google Sheets OAuth2 API credentials (`REPLACE_WITH_YOUR_SHEETS_CRED`).

---

### 1.4 Anomaly Detection & Alerting

#### Overview
This block evaluates the health flags produced during data normalization, determining whether exchange rates are flagged as stale or if any transactions failed currency conversion, and routes a detailed alert message to Slack if anomalies are detected.

#### Nodes Involved
- `Check for Anomalies` (`n8n-nodes-base.if`)
- `Post Anomaly Alert to Slack` (`n8n-nodes-base.slack`)

#### Node Details

##### Check for Anomalies
- **Type and Technical Role:** Conditional routing node that inspects the review flag.
- **Configuration Choices:** Condition checks if `$json.needs_review` equals boolean `true` using loose type validation.
- **Key Expressions or Variables:** `={{ $json.needs_review }}`
- **Input and Output Connections:** Input: `Normalize and Aggregate Data`; True Output: `Post Anomaly Alert to Slack`; False Output: None.
- **Version-specific Requirements:** Version 2.2.
- **Edge Cases or Potential Failure Types:** Type validation mismatches if boolean evaluations are passed as strings.

##### Post Anomaly Alert to Slack
- **Type and Technical Role:** Slack integration node that posts structured alert notifications to a designated messaging channel.
- **Configuration Choices:** Target channel configured as `#finance`. Text parameter populated via template expressions summarizing the reconciliation date, FX freshness, unconverted counts, and converted totals.
- **Key Expressions or Variables:**
  - `=⚠️ *Revenue reconciliation for {{ $json.date }} needs review*`
  - `{{ $json.rates_stale ? 'Exchange rates were stale, last updated ' + $json.rates_last_updated + '.' : 'Exchange rates were fresh.' }}`
  - `{{ $json.unconverted.length ? $json.unconverted.length + ' charge(s) could not be converted: ' + $json.unconverted.map(c => c.id + ' (' + c.currency + ')').join(', ') + '.' : 'All charges were converted.' }}`
  - `Converted total: *{{ $json.reporting_currency }} {{ $json.grand_total_reporting }}* from {{ $json.charge_count }} charges. Verify before closing the books.`
- **Input and Output Connections:** Input: `Check for Anomalies` (True branch); Output: None.
- **Version-specific Requirements:** Version 2.2.
- **Edge Cases or Potential Failure Types:** Invalid Slack channel names, missing token permissions for posting messages, or expired credentials.
- **Credentials:** Requires Slack API credentials (`REPLACE_WITH_YOUR_SLACK_CRED`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Overview documentation and setup instructions for the complete workflow. | None | None | ## Reconcile daily multi-currency revenue with live exchange rates<br><br>### How it works<br><br>This workflow runs daily at 09:00, or manually with sample Stripe-like charges for testing. It retrieves live exchange rates, normalizes multi-currency revenue into a base reporting view, aggregates the results, then writes a ledger row to Google Sheets. If the reconciliation indicates a condition that needs review, it sends a Slack alert for follow-up.<br><br>### Setup steps<br><br>- Configure Stripe credentials and confirm the charge listing filters match the intended daily reconciliation window.<br>- Configure ApogeoAPI credentials or connection settings for live exchange-rate retrieval.<br>- Connect Google Sheets credentials and select the target spreadsheet and ledger worksheet.<br>- Connect Slack credentials and choose the channel or recipient for review alerts.<br>- Verify the schedule timezone and run the manual test trigger with the sample charges before enabling the daily schedule.<br><br>### Customization<br><br>Adjust the normalization and aggregation code to change the base currency, date window, revenue fields, or anomaly thresholds. The sample charges node can also be edited to test additional currencies and edge cases. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation highlighting the trigger and charge input block. | None | None | ## Trigger and charge input<br><br>Starts the workflow either on the daily schedule with real Stripe charges or through a manual test path using sample charge data. These parallel input lanes are spatially aligned on the left side of the canvas. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation highlighting the rates retrieval and core reconciliation logic block. | None | None | ## Rates and reconciliation<br><br>Fetches current exchange rates from ApogeoAPI and runs the central normalization and aggregation logic that reconciles the charge data across currencies. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation highlighting the ledger recording and anomaly review alert outputs block. | None | None | ## Ledger and review outputs<br><br>Records the aggregated reconciliation result in Google Sheets and evaluates whether the result needs human review. If the condition is met, it sends a Slack alert from the nearby review branch. |
| When Daily at 9am | `n8n-nodes-base.scheduleTrigger` | Automatically triggers the production workflow daily at 09:00. | None | Fetch Stripe Charges | ## Trigger and charge input<br><br>Starts the workflow either on the daily schedule with real Stripe charges or through a manual test path using sample charge data. These parallel input lanes are spatially aligned on the left side of the canvas. |
| Manual Execution Trigger | `n8n-nodes-base.manualTrigger` | Allows manual workflow execution for testing. | None | Generate Sample Stripe Charges | ## Trigger and charge input<br><br>Starts the workflow either on the daily schedule with real Stripe charges or through a manual test path using sample charge data. These parallel input lanes are spatially aligned on the left side of the canvas. |
| Fetch Stripe Charges | `n8n-nodes-base.stripe` | Retrieves historical and successful transaction charges from Stripe. | When Daily at 9am | Fetch Exchange Rates | ## Trigger and charge input<br><br>Starts the workflow either on the daily schedule with real Stripe charges or through a manual test path using sample charge data. These parallel input lanes are spatially aligned on the left side of the canvas. |
| Generate Sample Stripe Charges | `n8n-nodes-base.code` | Generates sample multi-currency mock transaction charges for manual tests. | Manual Execution Trigger | Fetch Exchange Rates | ## Trigger and charge input<br><br>Starts the workflow either on the daily schedule with real Stripe charges or through a manual test path using sample charge data. These parallel input lanes are spatially aligned on the left side of the canvas. |
| Fetch Exchange Rates | `n8n-nodes-apogeoapi.apogeoAPI` | Retrieves current exchange rates from ApogeoAPI. | Fetch Stripe Charges, Generate Sample Stripe Charges | Normalize and Aggregate Data | ## Rates and reconciliation<br><br>Fetches current exchange rates from ApogeoAPI and runs the central normalization and aggregation logic that reconciles the charge data across currencies. |
| Normalize and Aggregate Data | `n8n-nodes-base.code` | Processes, converts, and aggregates financial transaction data. | Fetch Exchange Rates | Append Data to Ledger Sheet, Check for Anomalies | ## Rates and reconciliation<br><br>Fetches current exchange rates from ApogeoAPI and runs the central normalization and aggregation logic that reconciles the charge data across currencies. |
| Append Data to Ledger Sheet | `n8n-nodes-base.googleSheets` | Appends daily financial reconciliation records into Google Sheets. | Normalize and Aggregate Data | None | ## Ledger and review outputs<br><br>Records the aggregated reconciliation result in Google Sheets and evaluates whether the result needs human review. If the condition is met, it sends a Slack alert from the nearby review branch. |
| Check for Anomalies | `n8n-nodes-base.if` | Evaluates review flags to decide if an alert should be triggered. | Normalize and Aggregate Data | Post Anomaly Alert to Slack | ## Ledger and review outputs<br><br>Records the aggregated reconciliation result in Google Sheets and evaluates whether the result needs human review. If the condition is met, it sends a Slack alert from the nearby review branch. |
| Post Anomaly Alert to Slack | `n8n-nodes-base.slack` | Sends warning alerts to Slack when exchange rates are stale or conversions fail. | Check for Anomalies | None | ## Ledger and review outputs<br><br>Records the aggregated reconciliation result in Google Sheets and evaluates whether the result needs human review. If the condition is met, it sends a Slack alert from the nearby review branch. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to recreate the workflow in n8n:

1. **Create Sticky Notes (Optional for documentation):**
   - Add four sticky notes with the text contents provided in the Summary Table above to document the canvas sections visually.

2. **Set up Input Triggers:**
   - Add a **Schedule Trigger** node (`When Daily at 9am`). Set its rule interval to trigger every day at hour `9`.
   - Add a **Manual Trigger** node (`Manual Execution Trigger`).

3. **Set up Data Input Nodes:**
   - Add a **Stripe** node (`Fetch Stripe Charges`). Connect its input to `When Daily at 9am`. Set resource to `charge`, operation to `getAll`, and enable `returnAll`. Configure your Stripe API credentials (`REPLACE_WITH_YOUR_STRIPE_CRED`).
   - Add a **Code** node (`Generate Sample Stripe Charges`). Connect its input to `Manual Execution Trigger`. Set mode to JavaScript and paste code that generates mock transaction objects (including properties for `id`, `amount`, `currency`, `status`, `amount_refunded`, and `created`).

4. **Set up Exchange Rate Retrieval:**
   - Add an **ApogeoAPI** node (`Fetch Exchange Rates`). Connect inputs from both `Fetch Stripe Charges` and `Generate Sample Stripe Charges`. Set resource to `exchangeRate`, operation to `listRates`, and base currency to `USD`. Enable `executeOnce`. Configure your ApogeoAPI credentials (`REPLACE_WITH_YOUR_CRED_ID`). *Note: Requires the community node package `n8n-nodes-apogeoapi` installed.*

5. **Set up Normalization and Aggregation Logic:**
   - Add a **Code** node (`Normalize and Aggregate Data`). Connect its input to `Fetch Exchange Rates`. Set mode to JavaScript and paste the normalization script that parses the FX rates, filters yesterday's UTC charges, adjusts for currency decimal standards, computes currency breakdowns, and builds review flags.

6. **Set up Google Sheets Integration:**
   - Add a **Google Sheets** node (`Append Data to Ledger Sheet`). Connect its input to the first output of `Normalize and Aggregate Data`. Set operation to `append`, mapping mode to define columns below, document ID to your target Google Sheet ID, and sheet name to `Reconciliation`. Map the columns as follows:
     - `date`: `={{ $json.date }}`
     - `breakdown`: `={{ JSON.stringify($json.per_currency) }}`
     - `grand_total`: `={{ $json.grand_total_reporting }}`
     - `rates_stale`: `={{ $json.rates_stale }}`
     - `unconverted`: `={{ JSON.stringify($json.unconverted) }}`
     - `charge_count`: `={{ $json.charge_count }}`
     - `usd_eur_rate`: `={{ $json.usd_eur_rate }}`
     - `reporting_currency`: `={{ $json.reporting_currency }}`
   - Configure your Google Sheets OAuth2 API credentials (`REPLACE_WITH_YOUR_SHEETS_CRED`).

7. **Set up Anomaly Branch and Slack Alerts:**
   - Add an **If** node (`Check for Anomalies`). Connect its input to the second output of `Normalize and Aggregate Data`. Set condition to evaluate if `={{ $json.needs_review }}` is `true`.
   - Add a **Slack** node (`Post Anomaly Alert to Slack`). Connect its input to the `true` output branch of `Check for Anomalies`. Set channel to `#finance` and populate the text field with the alert template expression verifying dates, stale statuses, and unconverted counts. Configure your Slack API credentials (`REPLACE_WITH_YOUR_SLACK_CRED`).

8. **Validation:**
   - Execute the manual trigger path first to verify that sample records process correctly through normalization, logging, and alert branches before enabling the automated daily schedule.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Stripe Currency Decimals Reference | Used in normalization logic for handling non-standard currency units ([Stripe Currencies Documentation](https://docs.stripe.com/currencies)) |
| ApogeoAPI Community Node Requirement | Requires installation of `n8n-nodes-apogeoapi` to fetch live exchange rates |