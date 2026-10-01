Find relevant US job listings by role and location with Jobright forms

https://n8nworkflows.xyz/workflows/find-relevant-us-job-listings-by-role-and-location-with-jobright-forms-19836


# Find relevant US job listings by role and location with Jobright forms

### 1. Workflow Overview

This workflow exposes an interactive n8n form that queries Jobright’s public visitor job-search endpoint based on user input (job title, location, and remote preference). It retrieves up to five relevant job listings, normalizes the data, formats the output as a styled HTML list, and displays the results directly in the browser upon form submission. If any step in the pipeline fails, the workflow catches the error, formats a clean HTML error message, and presents it to the user.

The execution logic is divided into the following functional blocks:
- **1.1 Input Reception & Validation Setup:** Captures user criteria via an n8n Form trigger and validates parameters before constructing the outbound payload.
- **1.2 Job Search & API Integration:** Executes a POST request to Jobright’s anonymous job-search endpoint and validates the response structure.
- **1.3 Data Processing & HTML Rendering:** Normalizes the raw job listings, constructs a responsive HTML layout with secure URL sanitization, and renders the success view.
- **1.4 Error Handling & Fallback:** Catches errors across the pipeline, formats an HTML failure message, and presents it via a dedicated form completion step.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation Setup
- **Overview:** This block initializes the workflow via a web form, collects user inputs (job title, location, remote preference), and translates them into a structured query payload for the Jobright API.
- **Nodes Involved:** 
  - `Job search form`
  - `Build search request`
- **Node Details:**
  - **Job search form**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (v2.2) — Acts as the entry point, rendering a web form to users.
    - *Configuration Choices:* Configured with path `find-relevant-jobs-with-jobright`, button label "Find jobs", and response mode set to output of the last node. Includes three fields: `Job Title` (text, required), `Location` (text, optional), and `Remote Preference` (dropdown with options: *Any*, *Remote only*, *Onsite only*, *Hybrid only*, required).
    - *Key Expressions or Variables:* Uses static form field definitions.
    - *Input and Output Connections:* Input: None (Trigger); Output: `Build search request`.
    - *Edge Cases / Potential Failures:* Form submissions may lack required fields if client-side validation is bypassed, though n8n enforces required field rules natively.
  - **Build search request**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that validates user inputs and formats the request body.
    - *Configuration Choices:* Truncates/trims strings, validates job title length (1–200 characters), maps remote work preferences to numeric identifiers (`Remote only`: `[2]`, `Onsite only`: `[1]`, `Hybrid only`: `[3]`), and defines a default search radius of 25 miles for US locations.
    - *Key Expressions or Variables:* Reads `$input.first().json['Job Title']`, `Location`, and `['Remote Preference']`. Throws explicit errors on invalid inputs.
    - *Input and Output Connections:* Input: `Job search form`; Output: Main output connects to `Search Jobright jobs`, error output connects to `Explain search failure`.
    - *Edge Cases / Potential Failures:* Throws an error if the job title exceeds 200 characters or is left empty, triggering the error handling branch.

#### 2.2 Job Search & API Integration
- **Overview:** Sends the constructed search criteria payload to Jobright's visitor endpoint via an HTTP POST request and verifies that the response contains valid job data.
- **Nodes Involved:** 
  - `Search Jobright jobs`
  - `Validate and format jobs`
- **Node Details:**
  - **Search Jobright jobs**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Performs an outbound HTTP POST request to an external API.
    - *Configuration Choices:* URL: `https://jobright.ai/swan/recommend/visitor-list/jobs?count=5&position=0&sortCondition=0&useLegacySearch=false`. Method: POST. Body content type: Raw (`application/json`). Timeout: 120,000ms.
    - *Key Expressions or Variables:* Body expression: `={{ JSON.stringify($json.body) }}`
    - *Input and Output Connections:* Input: `Build search request`; Output: Main output connects to `Validate and format jobs`, error output connects to `Explain search failure`.
    - *Edge Cases / Potential Failures:* Network timeouts, DNS resolution errors, or HTTP 4xx/5xx responses from Jobright will trigger the error branch (`Explain search failure`).
  - **Validate and format jobs**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node parsing and normalizing API response arrays.
    - *Configuration Choices:* Checks `response.success === true` and verifies `response.result.jobList` is an array. Slices results to a maximum of 5 items, extracting rank, ID, title, company, location strings, work models, employment types, and fallback apply URLs.
    - *Key Expressions or Variables:* Reads `$input.first().json`.
    - *Input and Output Connections:* Input: `Search Jobright jobs`; Output: Main output connects to `Render job results`, error output connects to `Explain search failure`.
    - *Edge Cases / Potential Failures:* Unexpected API schema shifts or missing job lists throw an error, routing to the failure handler.

#### 2.3 Data Processing & HTML Rendering
- **Overview:** Transforms the normalized job objects into a clean, responsive HTML list with security sanitization and presents the final response to the user via the form completion UI.
- **Nodes Involved:** 
  - `Render job results`
  - `Show relevant jobs`
- **Node Details:**
  - **Render job results**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node generating styled HTML markup.
    - *Configuration Choices:* Implements HTML special character escaping and URL protocol verification (allowing only `https:` links to prevent XSS). Assembles list items (`<li>`) and search metadata summaries.
    - *Key Expressions or Variables:* Reads `$input.first().json.jobs` and cross-references data from the `Build search request` node using `$('Build search request').first().json`.
    - *Input and Output Connections:* Input: `Validate and format jobs`; Output: Main output connects to `Show relevant jobs`, error output connects to `Explain search failure`.
    - *Edge Cases / Potential Failures:* Malformed URLs or script injections in job attributes are neutralized via escaping and URL protocol validation.
  - **Show relevant jobs**
    - *Type and Technical Role:* `n8n-nodes-base.form` (v2.3) — Form completion node that terminates the workflow session and renders text/HTML back to the user's browser.
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, Response text: `={{ $json.html }}`.
    - *Key Expressions or Variables:* `={{ $json.html }}`
    - *Input and Output Connections:* Input: `Render job results`; Output: None (Terminal node).
    - *Edge Cases / Potential Failures:* Browser rendering issues if malformed HTML slips through (mitigated by the escaping functions in the preceding node).

#### 2.4 Error Handling & Fallback
- **Overview:** Captures errors from upstream nodes, formats a readable error message wrapped in clean HTML, and displays the failure state in the browser.
- **Nodes Involved:** 
  - `Explain search failure`
  - `Show search failure`
- **Node Details:**
  - **Explain search failure**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node extracting error messages and formatting a fallback HTML page.
    - *Configuration Choices:* Sanitizes error strings and builds an error presentation layout.
    - *Key Expressions or Variables:* Reads `$input.first().json.error?.message` or `item.message`.
    - *Input and Output Connections:* Input: Connected via error output paths from `Build search request`, `Search Jobright jobs`, `Validate and format jobs`, and `Render job results`. Output: `Show search failure`.
    - *Edge Cases / Potential Failures:* Undefined error structures default to a generic failure message.
  - **Show search failure**
    - *Type and Technical Role:* `n8n-nodes-base.form` (v2.3) — Form completion node rendering the error message to the user.
    - *Configuration Choices:* Operation: `completion`, Respond with: `showText`, Response text: `={{ $json.html }}`.
    - *Key Expressions or Variables:* `={{ $json.html }}`
    - *Input and Output Connections:* Input: `Explain search failure`; Output: None (Terminal node).
    - *Edge Cases / Potential Failures:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Job search form | n8n-nodes-base.formTrigger | Triggers the workflow via a web form and collects job criteria. | None (Trigger) | Build search request | ## Find relevant jobs by role and location<br><br>This workflow provides a no-credential job search form powered by Jobright's anonymous visitor job-search endpoint.<br><br>**Flow:** job title, location, and remote preference → Jobright search → validate response → show up to five jobs.<br><br>Results are described as relevant search results, not personalized matches or match scores. No resume, account, API key, or third-party service is required. |
| Build search request | n8n-nodes-base.code | Validates form inputs and builds the search payload. | Job search form | Search Jobright jobs, Explain search failure | ## Find relevant jobs by role and location<br><br>This workflow provides a no-credential job search form powered by Jobright's anonymous visitor job-search endpoint.<br><br>**Flow:** job title, location, and remote preference → Jobright search → validate response → show up to five jobs.<br><br>Results are described as relevant search results, not personalized matches or match scores. No resume, account, API key, or third-party service is required. |
| Search Jobright jobs | n8n-nodes-base.httpRequest | Sends a POST request to Jobright’s public API. | Build search request | Validate and format jobs, Explain search failure | ## Find relevant jobs by role and location<br><br>This workflow provides a no-credential job search form powered by Jobright's anonymous visitor job-search endpoint.<br><br>**Flow:** job title, location, and remote preference → Jobright search → validate response → show up to five jobs.<br><br>Results are described as relevant search results, not personalized matches or match scores. No resume, account, API key, or third-party service is required. |
| Validate and format jobs | n8n-nodes-base.code | Validates API response and normalizes job records. | Search Jobright jobs | Render job results, Explain search failure | ## Setup and customization<br><br>1. Import the workflow and open **Job search form**.<br>2. Test with `Software Engineer`, `San Francisco, CA`, and a remote preference.<br>3. Activate the workflow to use its production form URL.<br><br>Remote mapping is `1 = Onsite`, `2 = Remote`, and `3 = Hybrid`. To customize: change the result count (maximum used here: 5), radius, or output styling. Keep the public positioning search-based and avoid claiming personalized matching. |
| Render job results | n8n-nodes-base.code | Renders jobs into a secure HTML list. | Validate and format jobs | Show relevant jobs, Explain search failure | ## Setup and customization<br><br>1. Import the workflow and open **Job search form**.<br>2. Test with `Software Engineer`, `San Francisco, CA`, and a remote preference.<br>3. Activate the workflow to use its production form URL.<br><br>Remote mapping is `1 = Onsite`, `2 = Remote`, and `3 = Hybrid`. To customize: change the result count (maximum used here: 5), radius, or output styling. Keep the public positioning search-based and avoid claiming personalized matching. |
| Show relevant jobs | n8n-nodes-base.form | Displays the successful HTML search results to the user. | Render job results | None (Terminal) | ## Setup and customization<br><br>1. Import the workflow and open **Job search form**.<br>2. Test with `Software Engineer`, `San Francisco, CA`, and a remote preference.<br>3. Activate the workflow to use its production form URL.<br><br>Remote mapping is `1 = Onsite`, `2 = Remote`, and `3 = Hybrid`. To customize: change the result count (maximum used here: 5), radius, or output styling. Keep the public positioning search-based and avoid claiming personalized matching. |
| Explain search failure | n8n-nodes-base.code | Formats error messages into an HTML failure page. | Build search request, Search Jobright jobs, Validate and format jobs, Render job results | Show search failure | |
| Show search failure | n8n-nodes-base.form | Displays the HTML error page to the user. | Explain search failure | None (Terminal) | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node (`n8n-nodes-base.formTrigger`, v2.2).
   - Set Path to `find-relevant-jobs-with-jobright`, Button Label to `Find jobs`, and Form Title to `Find relevant jobs by role and location`.
   - Add three form fields:
     1. Type: `Text`, Field Label: `Job Title`, Placeholder: `Software Engineer`, Required: `true`.
     2. Type: `Text`, Field Label: `Location`, Placeholder: `San Francisco, CA`, Required: `false`.
     3. Type: `Dropdown`, Field Label: `Remote Preference`, Options: `Any`, `Remote only`, `Onsite only`, `Hybrid only`, Required: `true`.
   - Set Response Mode to `Last Node`.

2. **Create the Request Builder Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Build search request`.
   - Set Mode to `Run Once for All Items`.
   - Paste the validation and payload generation JavaScript snippet that reads the form inputs, sanitizes strings, maps remote preferences, and outputs `{ json: { jobTitle, location, remotePreference, body } }`.
   - Enable **Continue On Fail** (`onError: continueErrorOutput`).

3. **Create the HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, v4.2) named `Search Jobright jobs`.
   - Set Method to `POST`, URL to `https://jobright.ai/swan/recommend/visitor-list/jobs?count=5&position=0&sortCondition=0&useLegacySearch=false`.
   - Set Body Content Type to `Raw` (`application/json`).
   - Set Body expression to `={{ JSON.stringify($json.body) }}`.
   - Set timeout options to `120000ms` and response format to `JSON`.
   - Enable **Continue On Fail** (`onError: continueErrorOutput`).

4. **Create the Validation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Validate and format jobs`.
   - Paste the JavaScript snippet to verify `response.success === true` and extract/normalize up to 5 job results into clean properties (`rank`, `jobId`, `jobTitle`, `company`, `location`, `workModel`, `employmentType`, `url`).
   - Enable **Continue On Fail** (`onError: continueErrorOutput`).

5. **Create the HTML Renderer Node:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Render job results`.
   - Paste the JavaScript snippet that encodes special characters, verifies HTTPS link safety, builds HTML card elements, and compiles the final page markup into `{ json: { html, jobs, query } }`.
   - Enable **Continue On Fail** (`onError: continueErrorOutput`).

6. **Create the Success Form Completion Node:**
   - Add a **Form** node (`n8n-nodes-base.form`, v2.3) named `Show relevant jobs`.
   - Set Operation to `Completion`, Respond With to `Show Text`, and Response Text expression to `={{ $json.html }}`.

7. **Create the Error Handling Branch:**
   - Add a **Code** node (`n8n-nodes-base.code`, v2) named `Explain search failure`.
   - Paste the JavaScript snippet to capture upstream error messages, escape HTML entities, and output a styled error page HTML string.
   - Add a **Form** node (`n8n-nodes-base.form`, v2.3) named `Show search failure`.
   - Set Operation to `Completion`, Respond With to `Show Text`, and Response Text expression to `={{ $json.html }}`.

8. **Establish Connections:**
   - `Job search form` (main) $\rightarrow$ `Build search request`
   - `Build search request` (main) $\rightarrow$ `Search Jobright jobs`
   - `Build search request` (error) $\rightarrow$ `Explain search failure`
   - `Search Jobright jobs` (main) $\rightarrow$ `Validate and format jobs`
   - `Search Jobright jobs` (error) $\rightarrow$ `Explain search failure`
   - `Validate and format jobs` (main) $\rightarrow$ `Render job results`
   - `Validate and format jobs` (error) $\rightarrow$ `Explain search failure`
   - `Render job results` (main) $\rightarrow$ `Show relevant jobs`
   - `Render job results` (error) $\rightarrow$ `Explain search failure`
   - `Explain search failure` (main) $\rightarrow$ `Show search failure`

9. **Credentials & External Dependencies:**
   - No credentials or API keys are required; the workflow interacts exclusively with Jobright's public visitor endpoints.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Job search results are relevance-based and are not personalized match scores. Find more matching jobs at Jobright. | [Jobright Jobs Platform](https://jobright.ai/jobs) |