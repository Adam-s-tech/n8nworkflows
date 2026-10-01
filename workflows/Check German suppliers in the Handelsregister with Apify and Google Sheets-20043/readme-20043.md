Check German suppliers in the Handelsregister with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/check-german-suppliers-in-the-handelsregister-with-apify-and-google-sheets-20043


# Check German suppliers in the Handelsregister with Apify and Google Sheets

### 1. Workflow Overview

This workflow is designed for procurement, vendor onboarding, finance, and compliance teams to verify German business entities against the official German commercial register (*Handelsregister*). It provides an automated lookup pipeline triggered by an interactive web form, processes external API results, logs audit data into a spreadsheet, and renders structured status outputs directly back to the user.

The logical execution is grouped into three sequential functional blocks:
- **1.1 Input Reception:** Captures enterprise supplier query details via an n8n form interface (Company name, optional register number, and search behavior mode).
- **1.2 AI & External Processing:** Interacts with Apify’s official Germany Handelsregister Scraper actor to search the commercial register, handle fault states gracefully, and normalize search output vectors.
- **1.3 Data Logging and UI Feedback:** Evaluates match hierarchies, writes audit rows to Google Sheets, and completes the workflow execution by presenting dynamic validation statuses (Matched, No Match, or Register Unresponsive) to the submitter.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

##### Overview
This block initiates the execution cycle by exposing a web form interface to internal colleagues, harvesting the exact enterprise parameters required to execute downstream commercial registry lookups.

##### Nodes Involved
- `Supplier check form`

##### Node Details
- **Supplier check form**
  - **Type and technical role:** `n8n-nodes-base.formTrigger` (Trigger node). Acts as a webhook endpoint and web form generator.
  - **Configuration choices:** Configured with path `german-supplier-check`, button label `Check company`, and a completion response mode set to `lastNode`. Exposes three input fields: "Company name" (required text), "Register number (optional)" (text), and "Search type" (dropdown with options: "Exact company name" and "Keyword search").
  - **Key expressions or variables:** Outputs `$json['Company name']`, `$json['Register number (optional)']`, and `$json['Search type']`.
  - **Input and output connections:** No incoming inputs (Trigger node). Outputs directly to `Handelsregister lookup`.
  - **Version-specific requirements:** Uses version `2.2`.
  - **Edge cases or potential failure types:** Form submissions lacking required fields are intercepted natively by the form trigger validation logic.

---

#### 2.2 Register Lookup and Processing

##### Overview
This block executes the external API call to query the German commercial register via Apify, processes potential API runtime faults, and programmatically evaluates, sanitizes, and selects the best organizational match from returned registry payloads.

##### Nodes Involved
- `Handelsregister lookup`
- `Pick best match`

##### Node Details
- **Handelsregister lookup**
  - **Type and technical role:** `@apify/n8n-nodes-apify.apify` (Action node). Executes an Apify actor and waits for dataset retrieval.
  - **Configuration choices:** Uses actor ID `regdata~germany-handelsregister-scraper`, resource type `Actors`, and operation `Run actor and get dataset`. Configured with a memory limit of `512` MB, a strict cost ceiling (`maxTotalChargeUsd`) of `0.25`, and error handling set to `continueRegularOutput` with `alwaysOutputData` enabled.
  - **Key expressions or variables:** 
    `customBody`: `={{ JSON.stringify({ searchQuery: $json['Company name'], registerNumber: $json['Register number (optional)'] || undefined, exactMatch: $json['Search type'] !== 'Keyword search', maxResults: 5 }) }}`
  - **Input and output connections:** Input from `Supplier check form`. Output to `Pick best match`.
  - **Version-specific requirements:** Requires the verified community node package `@apify/n8n-nodes-apify` on self-hosted instances alongside valid Apify API credentials.
  - **Edge cases or potential failure types:** Network timeouts, upstream Apify throttling, credit exhaustion, or invalid input parameters causing actor faults (handled safely via error passing flags).

- **Pick best match**
  - **Type and technical role:** `n8n-nodes-base.code` (Action node). Runs custom JavaScript code to parse items, handle error states, score organizational name similarity, extract structural metadata, and normalize record attributes.
  - **Configuration choices:** Extracts query payloads from the form trigger node, sets up normalization regex helpers, filters payloads for matching entities, handles error objects, and generates a structured JSON payload containing timestamp, query parameters, match flags, legal forms, registry details, and a semicolon-delimited string of alternative matches.
  - **Key expressions or variables:** Uses `$input.all()`, `$now.toISO()`, and cross-node referencing (`$('Supplier check form').first().json`).
  - **Input and output connections:** Input from `Handelsregister lookup`. Output to `Log result to Google Sheets`.
  - **Version-specific requirements:** Uses version `2`.
  - **Edge cases or potential failure types:** JavaScript runtime exceptions if upstream fields contain unexpected null structures (mitigated via fallback string coercions).

---

#### 2.3 Data Logging and UI Feedback

##### Overview
This block logs the final structured compliance evaluation data into a permanent audit ledger on Google Sheets and updates the interactive web form completion screen to inform the user of the resolution status.

##### Nodes Involved
- `Log result to Google Sheets`
- `Show result`

##### Node Details
- **Log result to Google Sheets**
  - **Type and technical role:** `n8n-nodes-base.googleSheets` (Action node). Appends row data to a specified Google Sheets document.
  - **Configuration choices:** Operation set to `append`, mapping mode set to `autoMapInputData`. Document ID and Sheet Name reference dynamic list selectors.
  - **Key expressions or variables:** Automatically maps incoming keys from `Pick best match` (`checkedAt`, `query`, `matchFound`, `companyName`, `legalForm`, `registerCourt`, `registerNumber`, `status`, `seat`, `address`, `capital`, `euid`, `otherMatches`).
  - **Input and output connections:** Input from `Pick best match`. Output to `Show result`.
  - **Version-specific requirements:** Uses version `4.5`. Requires active Google Sheets OAuth2 credentials.
  - **Edge cases or potential failure types:** API permission errors, missing column headers in the target sheet matching the JSON keys, or quota limits.

- **Show result**
  - **Type and technical role:** `n8n-nodes-base.form` (Action node). Form response/completion display node.
  - **Configuration choices:** Operation set to `completion`. Renders dynamic titles and messages depending on the match status evaluation.
  - **Key expressions or variables:** 
    - `completionTitle`: `={{ $('Pick best match').first().json.matchFound === 'yes' ? $('Pick best match').first().json.companyName : ($('Pick best match').first().json.matchFound === 'no' ? 'No match found' : 'The register did not answer') }}`
    - `completionMessage`: `={{ $('Pick best match').first().json.matchFound === 'yes' ? [$('Pick best match').first().json.legalForm, $('Pick best match').first().json.registerCourt + ', ' + $('Pick best match').first().json.registerNumber, 'Status: ' + $('Pick best match').first().json.status, $('Pick best match').first().json.address].join(' | ') : ($('Pick best match').first().json.matchFound === 'no' ? 'Try a shorter name, Keyword search, or add the register number.' : 'Nothing was recorded as a match. Please submit the form again in a few minutes.') }}`
  - **Input and output connections:** Input from `Log result to Google Sheets`. No subsequent outgoing nodes (Terminal node).
  - **Version-specific requirements:** Uses version `1`.
  - **Edge cases or potential failure types:** Expression failure if referenced nodes return empty datasets.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| About this workflow | n8n-nodes-base.stickyNote | Documentation and setup guide for procurement/compliance teams. | None | None | ## Check German suppliers in the Handelsregister from a form<br><br>For procurement, vendor onboarding, finance and compliance teams that onboard German companies.<br><br>### How it works<br>1. A colleague enters a company name, and optionally a register number such as `HRB 6684`, in an n8n form.<br>2. The Apify Handelsregister actor looks the company up in the official German commercial register.<br>3. The best match (exact name first) is picked: register court, register number, legal form, status (current or deleted), seat, address, capital and EUID. Officer names are not copied.<br>4. The result is shown on the form's confirmation page and appended to Google Sheets. If the register does not answer, the form says so - it is never recorded as "no match".<br><br>### Setup<br>1. Add an **Apify API** credential to **Handelsregister lookup** ([free Apify plan with monthly usage credit](https://apify.com/regdata?fpr=getregdata)). On self-hosted n8n, install the verified community node `@apify/n8n-nodes-apify` first.<br>2. Create a sheet with the header row `checkedAt, query, matchFound, companyName, legalForm, registerCourt, registerNumber, status, seat, address, capital, euid, otherMatches` and select it in **Log result to Google Sheets**.<br>3. Activate the workflow and share the form's production URL.<br><br>### Cost<br>Billed on your Apify account per search plus per company returned - see the [Handelsregister actor page](https://apify.com/regdata/germany-handelsregister-scraper?fpr=getregdata). Up to 5 matches per search; the node stops a run above $0.25.<br><br>### Customization tips<br>- Start from a sheet instead: swap the form for a Google Sheets Trigger and write back with Update row.<br>- Add a German insolvency check: [Germany insolvency notices](https://apify.com/regdata/germany-insolvency-scraper?fpr=getregdata). |
| Section: form | n8n-nodes-base.stickyNote | Visual grouping block for form collection nodes. | None | None | ## 1. Form<br>Company name, optional register number, exact or keyword search. |
| Section: lookup | n8n-nodes-base.stickyNote | Visual grouping block for external registry query nodes. | None | None | ## 2. Register lookup<br>Official Handelsregister search. |
| Section: result | n8n-nodes-base.stickyNote | Visual grouping block for evaluation, logging, and completion nodes. | None | None | ## 3. Result<br>Best match to the sheet and the confirmation page. |
| Supplier check form | n8n-nodes-base.formTrigger | Captures search parameters via an interactive web form. | None | Handelsregister lookup | |
| Handelsregister lookup | @apify/n8n-nodes-apify.apify | Queries the official German commercial registry via Apify. | Supplier check form | Pick best match | |
| Pick best match | n8n-nodes-base.code | Normalizes search payloads and selects the highest-scoring organizational match. | Handelsregister lookup | Log result to Google Sheets | |
| Log result to Google Sheets | n8n-nodes-base.googleSheets | Appends normalized corporate registry attributes to Google Sheets. | Pick best match | Show result | |
| Show result | n8n-nodes-base.form | Renders status message and match metadata on the form completion screen. | Log result to Google Sheets | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Create a node of type `n8n-nodes-base.formTrigger` and name it `Supplier check form`.
   - Set Path option to `german-supplier-check`.
   - Set Button Label to `Check company`.
   - Set Form Title to `German supplier check`.
   - Set Form Description to `Look up a German company in the official commercial register (Handelsregister).`.
   - Add three Form Fields:
     1. Text field: Field Label = `Company name`, Placeholder = `e.g. Zalando SE`, Required = True.
     2. Text field: Field Label = `Register number (optional)`, Placeholder = `e.g. HRB 158855`, Required = False.
     3. Dropdown field: Field Label = `Search type`, Options = `Exact company name` and `Keyword search`, Required = True.
   - Configure Response Mode to `lastNode`.

2. **Create the Apify Lookup Node:**
   - Create a node of type `@apify/n8n-nodes-apify.apify` (requires community node package installation on self-hosted instances) and name it `Handelsregister lookup`.
   - Connect the output of `Supplier check form` to this node.
   - Set Resource to `Actors`, Operation to `Run actor and get dataset`.
   - Set Actor ID to `regdata~germany-handelsregister-scraper`.
   - Set Memory to `512`.
   - Set Max Total Charge (USD) to `0.25`.
   - Configure On Error behavior to `Continue (Using Regular Output)` and enable `Always Output Data`.
   - Set customBody expression to:
     `={{ JSON.stringify({ searchQuery: $json['Company name'], registerNumber: $json['Register number (optional)'] || undefined, exactMatch: $json['Search type'] !== 'Keyword search', maxResults: 5 }) }}`
   - Configure your Apify API credentials.

3. **Create the Code Processing Node:**
   - Create a node of type `n8n-nodes-base.code` and name it `Pick best match`.
   - Connect the output of `Handelsregister lookup` to this node.
   - Paste the JavaScript extraction block into the `jsCode` parameter:
     ```javascript
     const q = $('Supplier check form').first().json;
     const query = String(q['Company name'] || '').trim();
     const norm = s => String(s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
     const all = $input.all().map(i => i.json);
     const empty = { companyName: '', legalForm: '', registerCourt: '', registerNumber: '', status: '', seat: '', address: '', capital: '', euid: '', otherMatches: '' };
     const failed = all.find(h => h.error);
     if (failed) {
       return [{ json: { checkedAt: $now.toISO(), query, matchFound: 'check failed - try again', ...empty, otherMatches: String(failed.error).slice(0, 200) } }];
     }
     const hits = all.filter(h => h.companyName);
     const best = hits.find(h => norm(h.companyName) === norm(query)) || hits[0];
     if (!best) {
       return [{ json: { checkedAt: $now.toISO(), query, matchFound: 'no', ...empty } }];
     }
     return [{ json: {
       checkedAt: $now.toISO(),
       query,
       matchFound: 'yes',
       companyName: best.companyName,
       legalForm: best.legalForm || ((best.companyName.match(/(?:^|\s)(SE & Co\. KG|GmbH & Co\. KG|AG & Co\. KG|gGmbH|GmbH|UG \(haftungsbeschränkt\)|SE|AG|KGaA|KG|OHG|e\.K\.|eG|e\.V\.)\s*$/) || [])[1] || ''),
       registerCourt: best.registerCourt || '',
       registerNumber: best.registerNumber || '',
       status: best.status || '',
       seat: best.seat || '',
       address: best.address?.formatted || '',
       capital: best.capital?.formatted || '',
       euid: best.euid || '',
       otherMatches: hits.filter(h => h !== best).map(h => h.companyName + ' (' + (h.registerNumber || '') + ')').join('; '),
     } }];
     ```

4. **Create the Google Sheets Logging Node:**
   - Create a node of type `n8n-nodes-base.googleSheets` and name it `Log result to Google Sheets`.
   - Connect the output of `Pick best match` to this node.
   - Set Operation to `append`.
   - Select your target Document and Sheet name. Ensure your spreadsheet contains a header row matching: `checkedAt, query, matchFound, companyName, legalForm, registerCourt, registerNumber, status, seat, address, capital, euid, otherMatches`.
   - Set Mapping Mode to `autoMapInputData`.
   - Configure Google Sheets OAuth2 credentials.

5. **Create the Form Completion Node:**
   - Create a node of type `n8n-nodes-base.form` and name it `Show result`.
   - Connect the output of `Log result to Google Sheets` to this node.
   - Set Operation to `completion`.
   - Set Completion Title expression to:
     `={{ $('Pick best match').first().json.matchFound === 'yes' ? $('Pick best match').first().json.companyName : ($('Pick best match').first().json.matchFound === 'no' ? 'No match found' : 'The register did not answer') }}`
   - Set Completion Message expression to:
     `={{ $('Pick best match').first().json.matchFound === 'yes' ? [$('Pick best match').first().json.legalForm, $('Pick best match').first().json.registerCourt + ', ' + $('Pick best match').first().json.registerNumber, 'Status: ' + $('Pick best match').first().json.status, $('Pick best match').first().json.address].join(' | ') : ($('Pick best match').first().json.matchFound === 'no' ? 'Try a shorter name, Keyword search, or add the register number.' : 'Nothing was recorded as a match. Please submit the form again in a few minutes.') }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Platform Account & API Credentials | [Free Apify plan with monthly usage credit](https://apify.com/regdata?fpr=getregdata) |
| Apify Actor Documentation & Pricing (Germany Handelsregister Scraper) | [Handelsregister actor page](https://apify.com/regdata/germany-handelsregister-scraper?fpr=getregdata) |
| Optional Extension: German Insolvency Checks | [Germany insolvency notices](https://apify.com/regdata/germany-insolvency-scraper?fpr=getregdata) |