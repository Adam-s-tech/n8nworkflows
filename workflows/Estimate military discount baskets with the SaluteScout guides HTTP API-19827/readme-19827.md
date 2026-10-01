Estimate military discount baskets with the SaluteScout guides HTTP API

https://n8nworkflows.xyz/workflows/estimate-military-discount-baskets-with-the-salutescout-guides-http-api-19827


# Estimate military discount baskets with the SaluteScout guides HTTP API

### 1. Workflow Overview

This workflow is designed to automate the process of looking up verified brand discount guidelines and calculating pre-tax shopping basket totals. It accepts user inputs via an interactive n8n form, queries a public API for military discount articles, validates direct matches, and computes accurate monetary figures using integer-cent arithmetic with half-up rounding.

The logic is divided into three primary functional blocks:
- **1.1 Input Reception & Configuration:** Collects merchant and pricing inputs from the user via a web form and standardizes currency settings and result labeling.
- **1.2 API Search & Guide Validation:** Queries the public SaluteScout guides API to locate relevant brand articles and evaluates whether a direct brand match exists in the indexed results.
- **1.3 Mathematical Computation & Output:** Processes the eligible and excluded subtotals, applies caps (spend ceiling and allowance), calculates the estimated discount, and generates a formatted HTML response page.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block serves as the workflow entry point. It captures user input securely without requesting personal identification and standardizes the currency code and result heading parameters for downstream processing.
- **Nodes Involved:** 
  - `Collect basket and verified limits`
  - `Configure currency and heading`

- **Node Details:**
  - **Collect basket and verified limits**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Webhook-based Form Trigger, Version 2.4).
    - *Configuration Choices:* Configured as a public submission form (`responseMode: lastNode`, `authentication: none`). Collects text-based fields (`brand_query`, `eligible_subtotal`, `excluded_subtotal`, `discount_percent`, `eligible_spend_ceiling`, `remaining_discount_allowance`) and a mandatory dropdown acknowledgment (`terms_confirmation`).
    - *Key Expressions/Variables:* None (receives raw user input).
    - *Input/Output Connections:* Output connects directly to `Configure currency and heading`.
    - *Version-Specific Requirements:* Version 2.4.
    - *Edge Cases / Failure Types:* Empty inputs, incorrect decimal formatting, or failure to select the terms confirmation dropdown.

  - **Configure currency and heading**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields / Set node, Version 3.4).
    - *Configuration Choices:* Assigns constant configuration variables (`currency_code = USD`, `result_label = Pre-tax basket estimate`) while preserving existing input fields (`includeOtherFields: true`).
    - *Key Expressions/Variables:* Static string assignments.
    - *Input/Output Connections:* Input from `Collect basket and verified limits`; output connects to `Search published SaluteScout guides`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases / Failure Types:* Configuration corruption if downstream nodes attempt conversion (though none occurs).

---

#### 2.2 API Search & Guide Validation
- **Overview:** This block executes an HTTP GET request to the public SaluteScout guides API using the user's brand query. It searches for published documentation and checks for a direct, safe title or ID match.
- **Nodes Involved:**
  - `Search published SaluteScout guides`

- **Node Details:**
  - **Search published SaluteScout guides**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request node, Version 4.2).
    - *Configuration Choices:* Performs a `GET` request to `https://salutescout.com/api/agent/guides` with query parameters `q` (sanitized brand query) and `limit` (fixed to `5`). Configured to handle errors gracefully (`neverError: true`, `fullResponse: true`, `responseFormat: json`).
    - *Key Expressions/Variables:* `={{ String($json.brand_query || "").trim().slice(0, 120) }}`.
    - *Input/Output Connections:* Input from `Configure currency and heading`; output connects to `Validate lookup and calculate capped discount`.
    - *Version-Specific Requirements:* Version 4.2.
    - *Edge Cases / Failure Types:* Network timeout (10,000ms), DNS resolution failure, HTTP 5xx errors from the public endpoint, or rate limiting.

---

#### 2.3 Mathematical Computation & Output
- **Overview:** This block runs a custom JavaScript script to parse inputs safely using BigInt arithmetic (integer cents), applies spend ceilings and allowance caps, generates an HTML summary table, and presents the final results to the user via a form completion screen.
- **Nodes Involved:**
  - `Validate lookup and calculate capped discount`
  - `Show guide status and basket estimate`

- **Node Details:**
  - **Validate lookup and calculate capped discount**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code node, Version 2).
    - *Configuration Choices:* Executes once for all items (`runOnceForAllItems`). Validates currency format, term confirmation strings, and numerical bounds. Computes integer-cent arithmetic for eligible vs. excluded spend, spend ceilings, and remaining discount allowances. Uses half-up rounding.
    - *Key Expressions/Variables:* Accesses `$input.all()` and looks up parent item data from `$('Configure currency and heading').all()[index]?.json`.
    - *Input/Output Connections:* Input from `Search published SaluteScout guides`; output connects to `Show guide status and basket estimate`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases / Failure Types:* Throws handled errors on invalid decimal formats, missing terms confirmation, or negative numbers, routing errors into a fallback HTML error message.

  - **Show guide status and basket estimate**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Completion node, Version 2.4).
    - *Configuration Choices:* Operates in `completion` mode (`respondWith: text`). Dynamically sets the completion title and message based on the output of the preceding Code node.
    - *Key Expressions/Variables:* `={{ $json.title }}` and `={{ $json.result_html }}`.
    - *Input/Output Connections:* Input from `Validate lookup and calculate capped discount`; terminal node.
    - *Version-Specific Requirements:* Version 2.4.
    - *Edge Cases / Failure Types:* Rendering issues if HTML sanitization is bypassed (mitigated by explicit escaping in the Code node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Start here - purpose and setup** | `n8n-nodes-base.stickyNote` | Documentation & setup instructions | None | None | Look up a military discount guide and estimate a capped basket<br><br>### How it works<br>This form calls the public SaluteScout guide lookup API for the brand you enter, then turns your separately checked merchant terms into a repeatable pre-tax goods estimate. It accepts only a direct brand match in the returned guide title or ID; unrelated text mentions are not treated as a matching guide. It separates eligible goods from excluded goods, caps the eligible purchase base, applies your entered percentage, then caps the discount by the remaining allowance. A JavaScript Code node uses integer cents and half-up rounding. No live offer, military status, identity document or personal contact detail is requested or inferred.<br><br>### Setup<br>1. Review **Configure currency and heading**. Use one currency with two decimal minor units; no conversion occurs.<br>2. Select **Execute Workflow**, open the trigger's test URL and enter a brand and your checked terms.<br>3. The HTTP Request reads `https://salutescout.com/api/agent/guides`, documented at `https://salutescout.com/openapi.json`. No API key is needed. Its index can lag behind newly published pages, and a missing guide is not evidence of no merchant discount.<br>4. Enter `none` only for limits confirmed not applicable. Unknown terms should stop the calculation.<br>5. Fictional basket test: 120.00 eligible, 30.00 excluded, 10 percent, 100.00 ceiling, 8.00 remaining returns 8.00 discount and 142.00 goods total.<br>6. Publish only when ready to expose the production form.<br><br>No credentials or paid external services are needed. Tax, shipping, stacking, minimum spend and retailer rounding are not modeled. See the [source checklist and calculator](https://github.com/holaclea/military-discount-checklist). Related shopping guides are on our [website](https://salutescout.com/). |
| **Collect inputs and configure** | `n8n-nodes-base.stickyNote` | Block grouping note for inputs | None | None | ### 1. Collect checked terms<br>Form inputs stay as text to preserve decimal precision. Use `none` only for confirmed-inapplicable limits. The configuration node keeps the currency and result heading in one place. |
| **Look up a published guide and explain the estimate** | `n8n-nodes-base.stickyNote` | Block grouping note for lookup and logic | None | None | ### 2. Look up and estimate<br>The public API returns indexed site guides, not live merchant offers. Only a direct title or ID match is linked. The Code node applies checked limits using integer cents, then explains the result or an input correction. |
| **Collect basket and verified limits** | `n8n-nodes-base.formTrigger` | Webhook form entry point | None | Configure currency and heading | ### 1. Collect checked terms<br>Form inputs stay as text to preserve decimal precision. Use `none` only for confirmed-inapplicable limits. The configuration node keeps the currency and result heading in one place. |
| **Configure currency and heading** | `n8n-nodes-base.set` | Sets currency code and result label | Collect basket and verified limits | Search published SaluteScout guides | ### 1. Collect checked terms<br>Form inputs stay as text to preserve decimal precision. Use `none` only for confirmed-inapplicable limits. The configuration node keeps the currency and result heading in one place. |
| **Search published SaluteScout guides** | `n8n-nodes-base.httpRequest` | Queries SaluteScout API for guides | Configure currency and heading | Validate lookup and calculate capped discount | ### 2. Look up and estimate<br>The public API returns indexed site guides, not live merchant offers. Only a direct title or ID match is linked. The Code node applies checked limits using integer cents, then explains the result or an input correction. |
| **Validate lookup and calculate capped discount** | `n8n-nodes-base.code` | Parses API results and computes discount | Search published SaluteScout guides | Show guide status and basket estimate | ### 2. Look up and estimate<br>The public API returns indexed site guides, not live merchant offers. Only a direct title or ID match is linked. The Code node applies checked limits using integer cents, then explains the result or an input correction. |
| **Show guide status and basket estimate** | `n8n-nodes-base.form` | Displays completion page and summary table | Validate lookup and calculate capped discount | None | ### 2. Look up and estimate<br>The public API returns indexed site guides, not live merchant offers. Only a direct title or ID match is linked. The Code node applies checked limits using integer cents, then explains the result or an input correction. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Add a **Form Trigger** node (`n8n-nodes-base.formTrigger`, version 2.4).
   - Set the form title to `Military shopping basket estimate`.
   - Add text fields: `brand_query`, `eligible_subtotal`, `excluded_subtotal`, `discount_percent`, `eligible_spend_ceiling`, and `remaining_discount_allowance`.
   - Add a dropdown field named `terms_confirmation` with the option `I checked eligibility and current merchant terms`.
   - Set authentication to `none` and response mode to `lastNode`.

2. **Create the Set Node:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`, version 3.4).
   - Connect the output of the Form Trigger to this node.
   - Configure assignments to include `currency_code` (string: `USD`) and `result_label` (string: `Pre-tax basket estimate`), and keep other fields enabled (`includeOtherFields: true`).

3. **Create the HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, version 4.2).
   - Connect the Set node output to this node.
   - Set Method to `GET` and URL to `https://salutescout.com/api/agent/guides`.
   - Configure query parameters: `q` pointing to `={{ String($json.brand_query || "").trim().slice(0, 120) }}` and `limit` to `5`.
   - Configure header parameters: `Accept` (`application/json`) and `User-Agent` (`SaluteScout-n8n-template/1.0`).
   - Enable options: set timeout to `10000ms`, `neverError` to `true`, and `fullResponse` to `true`.

4. **Create the Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, version 2).
   - Connect the HTTP Request node output to this node.
   - Set mode to `Run Once for All Items`.
   - Insert the calculation and validation script handling BigInt cent parsing, regex validation, HTML escaping, and error handling as defined in the source.

5. **Create the Form Completion Node:**
   - Add a **Form** node (`n8n-nodes-base.form`, version 2.4).
   - Connect the Code node output to this node.
   - Set operation to `completion` and response format to `text`.
   - Set Completion Title to `={{ $json.title }}` and Completion Message to `={{ $json.result_html }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Source checklist and calculator source files | [GitHub Repository](https://github.com/holaclea/military-discount-checklist) |
| Related shopping guides and source notes | [SaluteScout Website](https://salutescout.com/) |
| API documentation reference | `https://salutescout.com/openapi.json` |