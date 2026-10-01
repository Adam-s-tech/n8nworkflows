Validate Pinterest Pin payloads locally with JSON-only checks

https://n8nworkflows.xyz/workflows/validate-pinterest-pin-payloads-locally-with-json-only-checks-19747


# Validate Pinterest Pin payloads locally with JSON-only checks

### 1. Workflow Overview

This workflow is designed to locally validate single-image Pinterest Pin payloads against the documented Forge F1 single-image profile before attempting any external publishing. Its primary target use cases include data sanitization, schema enforcement, and payload quality checks for automated publishing pipelines. The workflow runs entirely offline without making network requests, accessing credentials, or communicating with the Pinterest API.

The execution logic is grouped into three functional blocks:
- **1.1 Input Reception & Preparation:** Manually triggers the workflow, generates or accepts input items, and uses n8n's native URL parser to pre-parse destination and image URLs.
- **1.2 Validation Processing:** Applies the F1 validation engine via a JavaScript node to enforce strict schema requirements, URL safety, media rights declarations, and image metadata standards.
- **1.3 Routing & Branching:** Evaluates the boolean validation result and routes each processed item into either an accepted or rejected processing branch.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation
- **Overview:** Initializes the execution manually, provides synthetic test fixtures representing various payload states, and wraps the raw items while extracting URL path structures using native expressions.
- **Nodes Involved:** 
  - `Manual Test`
  - `Synthetic Examples - Replace Before Integration`
  - `Parse URL Syntax - Connect Your Items Here`
- **Node Details:**
  - **Manual Test**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Acts as the entry point for manual testing and execution.
    - *Configuration Choices:* Default parameters with no special inputs required.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects directly to `Synthetic Examples - Replace Before Integration`.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases/Failure Types:* None (manual execution only).
  - **Synthetic Examples - Replace Before Integration**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Generates an array of three static sample JSON objects representing different payload conditions (valid, incomplete/invalid types, and missing optional fields).
    - *Configuration Choices:* Mode set to `runOnceForAllItems`, executing JavaScript code to return mapped objects with paired item tracking.
    - *Key Expressions or Variables:* JavaScript array mapping returning `{json, pairedItem:{item:0}}`.
    - *Input/Output Connections:* Input from `Manual Test`; output connects to `Parse URL Syntax - Connect Your Items Here`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Yields hardcoded demo data; must be bypassed or replaced with upstream inputs in production integrations.
  - **Parse URL Syntax - Connect Your Items Here**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Transforms the incoming JSON structure to isolate the payload under test and pre-evaluates URL syntax safety.
    - *Configuration Choices:* Mode set to `raw` JSON output, discarding other incoming fields.
    - *Key Expressions or Variables:* Evaluates `_f1_input` as `$json` and computes `_f1_url_paths` by checking string types and invoking `.extractUrlPath()` on `destination_url` and `image_url`.
    - *Input/Output Connections:* Input from `Synthetic Examples - Replace Before Integration`; output connects to `Validate Pin Payload`.
    - *Version-Specific Requirements:* Version 3.4. Requires an n8n runtime supporting the native `extractUrlPath()` expression (tested on n8n 2.34.6).
    - *Edge Cases/Failure Types:* If upstream items are not flat JSON objects or lack string formats for URLs, the property evaluation gracefully returns `null`.

#### 2.2 Validation Processing
- **Overview:** Executes a comprehensive local validation script that inspects required fields, validates absolute HTTPS URLs, checks board ID formats, verifies media rights declarations, and reviews image dimensions and MIME types.
- **Nodes Involved:** 
  - `Validate Pin Payload`
- **Node Details:**
  - **Validate Pin Payload**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Core validation engine containing business logic rules for Pinterest Pin payloads.
    - *Configuration Choices:* Mode set to `runOnceForAllItems`, executing a comprehensive JavaScript validation function (`validatePin`) across all incoming items.
    - *Key Expressions or Variables:* Utilizes `$input.all()`, references `_f1_input` and `_f1_url_paths`, and establishes a shared `referenceTime` via `Date.now()`.
    - *Input/Output Connections:* Input from `Parse URL Syntax - Connect Your Items Here`; output connects to `Accepted?`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases/Failure Types:* Handles non-object payloads, missing required fields, non-HTTPS protocols, invalid URL encodings, expired schedules, and unsupported image MIME types (`image/jpeg`, `image/png`).

#### 2.3 Routing & Branching
- **Overview:** Evaluates the validity flag returned by the validation engine to direct items down separate paths depending on whether validation errors occurred.
- **Nodes Involved:** 
  - `Accepted?`
  - `ACCEPTED - Local Rules Passed`
  - `REJECTED - Correct All Errors`
- **Node Details:**
  - **Accepted?**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Conditional router splitting execution based on the validation status.
    - *Configuration Choices:* Strict type validation version 2, evaluating whether `{{ $json.valid }}` equals `true`.
    - *Key ExpressionsOr Variables:* `{{ $json.valid }}`
    - *Input/Output Connections:* Input from `Validate Pin Payload`; True branch connects to `ACCEPTED - Local Rules Passed`, False branch connects to `REJECTED - Correct All Errors`.
    - *Version-Specific Requirements:* Version 2.3.
    - *Edge Cases/Failure Types:* Expression evaluation failure if upstream JSON does not contain the `valid` boolean property.
  - **ACCEPTED - Local Rules Passed**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` — Placeholder terminal node for items that successfully pass all local validation rules.
    - *Configuration Choices:* Default No-Op configuration.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input connected to the True output of `Accepted?`.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases/Failure Types:* None.
  - **REJECTED - Correct All Errors**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` — Placeholder terminal node for items failing validation checks, containing structured error lists.
    - *Configuration Choices:* Default No-Op configuration.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input connected to the False output of `Accepted?`.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases/Failure Types:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Test | n8n-nodes-base.manualTrigger | Workflow entry point for manual testing | None | Synthetic Examples - Replace Before Integration | ## FORGE F1 · Local Pin validation<br>Run manually: three synthetic examples → two accepted and one rejected. No credentials, network calls, publishing, or schedules. ACCEPTED means local rules passed, not Pinterest approval.<br><br>**Integration:** Disconnect the synthetic example node and connect your own flat JSON items directly to **Parse URL Syntax - Connect Your Items Here**. Never keep demo rows connected to your downstream workflow. |
| Synthetic Examples - Replace Before Integration | n8n-nodes-base.code | Generates sample pin payload test records | Manual Test | Parse URL Syntax - Connect Your Items Here | ## FORGE F1 · Local Pin validation<br>Run manually: three synthetic examples → two accepted and one rejected. No credentials, network calls, publishing, or schedules. ACCEPTED means local rules passed, not Pinterest approval.<br><br>**Integration:** Disconnect the synthetic example node and connect your own flat JSON items directly to **Parse URL Syntax - Connect Your Items Here**. Never keep demo rows connected to your downstream workflow. |
| Parse URL Syntax - Connect Your Items Here | n8n-nodes-base.set | Pre-parses URL paths and wraps input items | Synthetic Examples - Replace Before Integration | Validate Pin Payload | ## FORGE F1 · Local Pin validation<br>Run manually: three synthetic examples → two accepted and one rejected. No credentials, network calls, publishing, or schedules. ACCEPTED means local rules passed, not Pinterest approval.<br><br>**Integration:** Disconnect the synthetic example node and connect your own flat JSON items directly to **Parse URL Syntax - Connect Your Items Here**. Never keep demo rows connected to your downstream workflow. |
| Validate Pin Payload | n8n-nodes-base.code | Executes local validation rules and schema checks | Parse URL Syntax - Connect Your Items Here | Accepted? | ## Input / output<br>Required: title, destination_url, board_id (digit string), image_url, media_rights_status (owned / licensed / public_domain). Optional: description, alt_text, image_metadata, scheduled_at.<br><br>Wrapper mapping: destination_url → link; image_url → media_source.url with source_type=image_url. Rights, metadata and schedule stay local.<br><br>Errors reject; warnings do not. Consult docs and run tests/run.cjs from the package. No URL, rights, board or credential verification is performed. |
| Accepted? | n8n-nodes-base.if | Routes items based on validation status | Validate Pin Payload | ACCEPTED - Local Rules Passed,<br>REJECTED - Correct All Errors | ## Input / output<br>Required: title, destination_url, board_id (digit string), image_url, media_rights_status (owned / licensed / public_domain). Optional: description, alt_text, image_metadata, scheduled_at.<br><br>Wrapper mapping: destination_url → link; image_url → media_source.url with source_type=image_url. Rights, metadata and schedule stay local.<br><br>Errors reject; warnings do not. Consult docs and run tests/run.cjs from the package. No URL, rights, board or credential verification is performed. |
| ACCEPTED - Local Rules Passed | n8n-nodes-base.noOp | Terminal node for valid payloads | Accepted? | None | |
| REJECTED - Correct All Errors | n8n-nodes-base.noOp | Terminal node for invalid payloads | Accepted? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Manual Chat/Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Manual Test`.
2. **Add the Synthetic Generator Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Synthetic Examples - Replace Before Integration`. Set mode to `Run Once for All Items` and language to `JavaScript`. Paste the example array returning `{json, pairedItem:{item:0}}`. Connect `Manual Test` output to this node.
3. **Add the URL Parser Wrapper Node:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`). Name it `Parse URL Syntax - Connect Your Items Here`. Set mode to `Raw` JSON output. Configure the expression to wrap `$json` into `_f1_input` and extract URL paths via `extractUrlPath()` for `destination_url` and `image_url`. Connect `Synthetic Examples - Replace Before Integration` output to this node.
4. **Add the Validation Engine Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate Pin Payload`. Set mode to `Run Once for All Items` and language to `JavaScript`. Insert the local validation function (`validatePin`) and reference clock mapping logic. Connect `Parse URL Syntax - Connect Your Items Here` output to this node.
5. **Add the Conditional Router Node:**
   - Add an **If** node (`n8n-nodes-base.if`). Name it `Accepted?`. Set condition type validation to `strict`, version `2`, checking if `{{ $json.valid }}` equals `true`. Connect `Validate Pin Payload` output to this node.
6. **Add the Terminal No-Op Nodes:**
   - Add two **No-Op** nodes (`n8n-nodes-base.noOp`). Name the first `ACCEPTED - Local Rules Passed` and connect it to the `True` output of `Accepted?`. Name the second `REJECTED - Correct All Errors` and connect it to the `False` output of `Accepted?`.
7. **Credentials & Configurations:**
   - No external credentials or API keys are required for this workflow. Ensure your n8n runtime is version 2.34.6 or higher to support native URL extraction expressions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| FORGE F1 by JusGro, version 1.0.0. Local validation only; no credentials or network calls required. | Engineering-cleared release package |