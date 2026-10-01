Rank engineering candidates by role fit with weighted scoring in Judgment

https://n8nworkflows.xyz/workflows/rank-engineering-candidates-by-role-fit-with-weighted-scoring-in-judgment-19731


# Rank engineering candidates by role fit with weighted scoring in Judgment

### 1. Workflow Overview

This workflow automates the evaluation, scoring, and ranking of engineering candidate profiles across multiple role types using weighted scoring via the Judgment community node. It accepts distinct candidate profiles, normalizes them into a unified execution stream, and applies role-specific evaluation metrics across four distinct competencies: technical depth, leadership, system design, and generalist breadth. 

The logical execution follows a fan-out and fan-in pattern split into four functional blocks:

- **1.1 Input Reception & Preparation:** Initializes execution manually and provisions sample candidate datasets (`Priya Raman`, `Tomas Berg`, and `Dana Okafor`) with distinct skill distributions.
- **1.2 Candidate Stream Merging:** Combines the parallel profile inputs into a single unified stream for batch processing.
- **1.3 Multi-Role Evaluation:** Fan-out processing branches the candidate stream into three parallel evaluation paths utilizing the Judgment API node configured with role-specific weights (Senior IC, Engineering Manager, and Equal-Weight Baseline).
- **1.4 Report Data Formatting:** Normalizes the composite scores, converts them to percentage fit metrics, aggregates per-dimension breakdowns, and tags records by role profile for side-by-side comparison.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preparation
- **Overview:** Initializes the workflow execution manually and defines static candidate profiles featuring varied technical and leadership attributes.
- **Nodes Involved:** 
  - `Manual Test Trigger`
  - `Set Candidate A Profile`
  - `Set Candidate B Profile`
  - `Set Candidate C Profile`
- **Node Details:**
  - `Manual Test Trigger`
    - **Type & Role:** `n8n-nodes-base.manualTrigger` — Acts as the manual entry point for workflow execution.
    - **Configuration:** Default manual execution trigger.
    - **Inputs / Outputs:** Input: None | Output: Triggers downstream Set nodes.
    - **Edge Cases:** Requires manual invocation; not suitable for automated webhook or schedule triggers without replacement.
  - `Set Candidate A Profile`
    - **Type & Role:** `n8n-nodes-base.set` — Assigns candidate identifier and technical profile text emphasizing depth and design.
    - **Configuration:** Manual assignment mode creating `candidate` (string: "Priya Raman") and `profile` (string).
    - **Inputs / Outputs:** Input: `Manual Test Trigger` | Output: `Merge Candidate Profiles` (Input index 0).
  - `Set Candidate B Profile`
    - **Type & Role:** `n8n-nodes-base.set` — Assigns candidate identifier and technical profile text emphasizing leadership and breadth.
    - **Configuration:** Manual assignment mode creating `candidate` (string: "Tomas Berg") and `profile` (string).
    - **Inputs / Outputs:** Input: `Manual Test Trigger` | Output: `Merge Candidate Profiles` (Input index 1).
  - `Set Candidate C Profile`
    - **Type & Role:** `n8n-nodes-base.set` — Assigns candidate identifier and technical profile text exhibiting balanced traits across all dimensions.
    - **Configuration:** Manual assignment mode creating `candidate` (string: "Dana Okafor") and `profile` (string).
    - **Inputs / Outputs:** Input: `Manual Test Trigger` | Output: `Merge Candidate Profiles` (Input index 2).

#### 2.2 Candidate Stream Merging
- **Overview:** Consolidates the parallel candidate data streams into a single output stream, ensuring all candidate profiles are processed together in subsequent evaluation steps.
- **Nodes Involved:**
  - `Merge Candidate Profiles`
- **Node Details:**
  - `Merge Candidate Profiles`
    - **Type & Role:** `n8n-nodes-base.merge` — Combines multiple input streams into one.
    - **Configuration:** Append mode with 3 number inputs.
    - **Inputs / Outputs:** Input: `Set Candidate A Profile`, `Set Candidate B Profile`, `Set Candidate C Profile` | Output: Triggers `Evaluate IC Role Fit`, `Evaluate EM Role Fit`, and `Evaluate Balanced Role Fit` in parallel.

#### 2.3 Multi-Role Evaluation
- **Overview:** Evaluates the candidate profiles against a shared four-point rubric using the Judgment API community node, applying distinct weight sets for Senior IC, Engineering Manager, and Equal-Weight profiles.
- **Nodes Involved:**
  - `Evaluate IC Role Fit`
  - `Evaluate EM Role Fit`
  - `Evaluate Balanced Role Fit`
- **Node Details:**
  - `Evaluate IC Role Fit`
    - **Type & Role:** `n8n-nodes-judgment.judgment` — Evaluates candidate profiles against defined rubrics and calculates composite scores using field-based weights favoring depth and design.
    - **Configuration:** Resource: `score`, Operation: `composite`. Dimensions and shared levels defined via fields (`depth`, `leadership`, `design`, `generalist`). Weights configured as: Depth (4), Leadership (1), Design (4), Generalist (1).
    - **Expressions:** Field assignment uses `={{ $json.profile }}`.
    - **Inputs / Outputs:** Input: `Merge Candidate Profiles` | Output: `Prepare IC Report Data`.
    - **Credentials:** Requires `judgmentApi` credentials.
    - **Edge Cases:** Requires self-hosted instance with `n8n-nodes-judgment` installed; API timeouts or invalid API keys will cause execution failure.
  - `Evaluate EM Role Fit`
    - **Type & Role:** `n8n-nodes-judgment.judgment` — Evaluates candidate profiles with weights adjusted to favor leadership and generalist breadth for an Engineering Manager role.
    - **Configuration:** Resource: `score`, Operation: `composite`. Identical dimensions and shared levels as IC node. Weights configured as: Depth (1), Leadership (4), Design (2), Generalist (3).
    - **Expressions:** Field assignment uses `={{ $json.profile }}`.
    - **Inputs / Outputs:** Input: `Merge Candidate Profiles` | Output: `Prepare EM Report Data`.
    - **Credentials:** Requires `judgmentApi` credentials.
  - `Evaluate Balanced Role Fit`
    - **Type & Role:** `n8n-nodes-judgment.judgment` — Evaluates candidate profiles using JSON-defined dimensions and equal weights as a neutral baseline.
    - **Configuration:** Resource: `score`, Operation: `composite`. Dimensions and weights (`{"depth": 2, "leadership": 2, "design": 2, "generalist": 2}`) supplied via JSON expressions (`dimensionsMode: json`, `weightsMode: json`).
    - **Expressions:** State field uses `={{ $json.profile }}`.
    - **Inputs / Outputs:** Input: `Merge Candidate Profiles` | Output: `Prepare Balanced Report Data`.
    - **Credentials:** Requires `judgmentApi` credentials.

#### 2.4 Report Data Formatting
- **Overview:** Normalizes scoring outputs, converts raw composite metrics into percentage fit formats, maps per-dimension scores, and tags each row with its respective role profile.
- **Nodes Involved:**
  - `Prepare IC Report Data`
  - `Prepare EM Report Data`
  - `Prepare Balanced Report Data`
- **Node Details:**
  - `Prepare IC Report Data`
    - **Type & Role:** `n8n-nodes-base.set` — Formats and structures evaluation output into a unified report schema for the Senior IC role.
    - **Configuration:** Manual assignment mode. Creates fields: `roleProfile` ("Senior IC"), `candidate` (extracted via `{{ $('Merge Candidate Profiles').item.json.candidate }}`), `combinedScore`, `fitPercent` (calculated via `{{ Math.round($json.combinedScore * 100) }}`), and `perDimension` (mapped string array).
    - **Inputs / Outputs:** Input: `Evaluate IC Role Fit` | Output: None (Terminal node).
  - `Prepare EM Report Data`
    - **Type & Role:** `n8n-nodes-base.set` — Formats and structures evaluation output into a unified report schema for the Engineering Manager role.
    - **Configuration:** Manual assignment mode. Creates fields: `roleProfile` ("Engineering Manager"), `candidate` (extracted via `{{ $('Merge Candidate Profiles').item.json.candidate }}`), `combinedScore`, `fitPercent`, and `perDimension`.
    - **Inputs / Outputs:** Input: `Evaluate EM Role Fit` | Output: None (Terminal node).
  - `Prepare Balanced Report Data`
    - **Type & Role:** `n8n-nodes-base.set` — Formats and structures evaluation output into a unified report schema for the equal-weight baseline.
    - **Configuration:** Manual assignment mode. Creates fields: `roleProfile` ("Equal weights"), `candidate` (extracted via `{{ $('Merge Candidate Profiles').item.json.candidate }}`), `combinedScore`, `fitPercent`, and `perDimension`.
    - **Inputs / Outputs:** Input: `Evaluate Balanced Role Fit` | Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Manual Test Trigger` | `n8n-nodes-base.manualTrigger` | Initiates manual workflow execution | None | `Set Candidate A Profile`, `Set Candidate B Profile`, `Set Candidate C Profile` | Rank candidates by role fit with weighted scoring in Judgment <br> ### How it works <br> 1. Holds three candidate profiles, each as a Set node, chosen so no candidate is strongest on every dimension. <br> 2. Merges them into one stream, so all three are scored in a single pass. <br> 3. Rates each candidate on four dimensions in one judgment call: depth, leadership, design, and breadth. <br> 4. Scores those same ratings three times against three different weight sets, changing only the weights. <br> 5. Builds one report row per candidate per weighting, each tagged with the profile that produced it. <br> ### Setup steps <br> - [ ] Add your **Judgment API** credential to all three scoring nodes, and set the Base URL if you are not using the default provider. <br> - [ ] Install `n8n-nodes-judgment` from **Settings → Community Nodes**. See the note beside the first node. <br> - [ ] Replace the three candidate Set nodes with your own source, keeping one `profile` field per item. <br> - [ ] Edit the four dimensions and the shared five-point scale to match your rubric. Every scoring node must declare the same dimensions. <br> - [ ] Set the weights to match the roles you are hiring for, then compare the report nodes side by side. <br> ### Customization <br> Weights are applied in the node, not by the model, so adjusting a weight re-ranks the candidates without a single new API call. *Evaluate IC Role Fit* and *Evaluate EM Role Fit* differ only in their weight rows; copy either to score another role. *Evaluate Balanced Role Fit* supplies the same rubric as JSON, for when the weights come from elsewhere. <br> Candidate C is deliberately mid on every dimension, so the weights decide where they land. |
| `Set Candidate A Profile` | `n8n-nodes-base.set` | Assigns profile data for Candidate A | `Manual Test Trigger` | `Merge Candidate Profiles` | Extreme on depth and design, weak on leadership. Wins for a senior individual contributor role. |
| `Set Candidate B Profile` | `n8n-nodes-base.set` | Assigns profile data for Candidate B | `Manual Test Trigger` | `Merge Candidate Profiles` | Extreme on leadership and breadth, weak on hands-on depth. Wins for an engineering manager role. |
| `Set Candidate C Profile` | `n8n-nodes-base.set` | Assigns profile data for Candidate C | `Manual Test Trigger` | `Merge Candidate Profiles` | Balanced across every dimension, so the weights decide where this one lands. |
| `Merge Candidate Profiles` | `n8n-nodes-base.merge` | Consolidates parallel candidate items into a single stream | `Set Candidate A Profile`, `Set Candidate B Profile`, `Set Candidate C Profile` | `Evaluate IC Role Fit`, `Evaluate EM Role Fit`, `Evaluate Balanced Role Fit` | One item per candidate, so all three are scored in one pass. |
| `Evaluate IC Role Fit` | `n8n-nodes-judgment.judgment` | Evaluates candidate scores using Senior IC weights | `Merge Candidate Profiles` | `Prepare IC Report Data` | Weights favour depth and design. Change a weight here and the score changes with no new inference. |
| `Evaluate EM Role Fit` | `n8n-nodes-judgment.judgment` | Evaluates candidate scores using Engineering Manager weights | `Merge Candidate Profiles` | `Prepare EM Report Data` | The same four dimensions, weighted for a manager. Candidate B overtakes A without re-running the ratings. |
| `Evaluate Balanced Role Fit` | `n8n-nodes-judgment.judgment` | Evaluates candidate scores using JSON-defined equal weights | `Merge Candidate Profiles` | `Prepare Balanced Report Data` | Same dimensions and weights, but both come from JSON. Use this shape when the rubric is generated upstream rather than typed into rows. |
| `Prepare IC Report Data` | `n8n-nodes-base.set` | Formats Senior IC evaluation output into report schema | `Evaluate IC Role Fit` | None | One row per candidate under the senior IC weights, tagged with the role that produced it. |
| `Prepare EM Report Data` | `n8n-nodes-base.set` | Formats Engineering Manager evaluation output into report schema | `Evaluate EM Role Fit` | None | One row per candidate under the manager weights. Read both report nodes side by side to see the ranking invert. |
| `Prepare Balanced Report Data` | `n8n-nodes-base.set` | Formats balanced evaluation output into report schema | `Evaluate Balanced Role Fit` | None | Every dimension weighted equally, which is the baseline the two role profiles are compared against. |
| *(Install Note)* | *(Sticky Note)* | Documents community node installation prerequisites | None | None | ## Install the community node first <br> **Settings → Community Nodes → Install →** `n8n-nodes-judgment`, then add your API key on the **Judgment API** credential. Self-hosted only: community nodes cannot run on n8n Cloud. |

---

### 4. Reproducing the Workflow from Scratch

1. **Prerequisite Setup:**
   - Ensure you are running a self-hosted n8n instance (community nodes are unsupported on n8n Cloud).
   - Install the community node package `n8n-nodes-judgment` via **Settings → Community Nodes → Install**.
   - Create a new credential of type **Judgment API** and input your API key (and custom Base URL if applicable).

2. **Create the Trigger:**
   - Add a **Manual Chat Trigger** or **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Manual Test Trigger`.

3. **Create Candidate Profile Nodes:**
   - Add three **Set** nodes (`n8n-nodes-base.set`):
     - **Node 1 (`Set Candidate A Profile`):** Configure assignments with string fields: `candidate` = `Priya Raman`, `profile` = `Priya Raman\n\nNine years as a backend engineer, and nothing else...`
     - **Node 2 (`Set Candidate B Profile`):** Configure assignments with string fields: `candidate` = `Tomas Berg`, `profile` = `Tomas Berg\n\nEngineering manager for six years...`
     - **Node 3 (`Set Candidate C Profile`):** Configure assignments with string fields: `candidate` = `Dana Okafor`, `profile` = `Dana Okafor\n\nTen years across backend and data...`
   - Connect the output of `Manual Test Trigger` to all three Set nodes.

4. **Merge Profiles:**
   - Add a **Merge** node (`n8n-nodes-base.merge`) named `Merge Candidate Profiles`.
   - Set the mode to `Append` with `3` input numbers.
   - Connect `Set Candidate A Profile` to input 0, `Set Candidate B Profile` to input 1, and `Set Candidate C Profile` to input 2.

5. **Configure Evaluation Nodes (Fan-Out):**
   - **Evaluate IC Role Fit:**
     - Add a **Judgment** node (`n8n-nodes-judgment.judgment`). Select resource `score`, operation `composite`.
     - Configure 4 dimensions (`depth`, `leadership`, `design`, `generalist`) using shared levels (`No evidence`, `Some exposure`, `Solid working experience`, `Deep, with examples`, `Extensive, and clearly led work here`).
     - Set weights to: `depth` = 4, `leadership` = 1, `design` = 4, `generalist` = 1.
     - Set state field `profile` to expression: `={{ $json.profile }}`.
     - Assign the **Judgment API** credential.
   - **Evaluate EM Role Fit:**
     - Add a second **Judgment** node with identical dimensions and shared levels.
     - Set weights to: `depth` = 1, `leadership` = 4, `design` = 2, `generalist` = 3.
     - Set state field `profile` to expression: `={{ $json.profile }}`.
     - Assign the **Judgment API** credential.
   - **Evaluate Balanced Role Fit:**
     - Add a third **Judgment** node. Set `dimensionsMode` and `weightsMode` to `json`.
     - Provide the dimensions and weights array via JSON expressions (matching the structure specified in the node parameters for depth, leadership, design, and generalist with equal weight of 2).
     - Set state field `profile` to expression: `={{ $json.profile }}`.
     - Assign the **Judgment API** credential.
   - Connect the output of `Merge Candidate Profiles` to the input of all three Judgment nodes.

6. **Configure Report Formatting Nodes:**
   - Add three **Set** nodes (`n8n-nodes-base.set`) named `Prepare IC Report Data`, `Prepare EM Report Data`, and `Prepare Balanced Report Data`.
   - **For IC Report Data:** Connect input to `Evaluate IC Role Fit`. Configure assignments:
     - `roleProfile` (string) = `Senior IC`
     - `candidate` (string) = `={{ $('Merge Candidate Profiles').item.json.candidate }}`
     - `combinedScore` (number) = `={{ $json.combinedScore }}`
     - `fitPercent` (number) = `={{ Math.round($json.combinedScore * 100) }}`
     - `perDimension` (string) = `={{ $json.dimensions.map(d => d.name + ' ' + d.normalized.toFixed(2)).join(', ') }}`
   - **For EM Report Data:** Connect input to `Evaluate EM Role Fit`. Configure identical assignments except `roleProfile` = `Engineering Manager`.
   - **For Balanced Report Data:** Connect input to `Evaluate Balanced Role Fit`. Configure identical assignments except `roleProfile` = `Equal weights`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Install the community node first: `n8n-nodes-judgment` | Required for self-hosted instances via Settings → Community Nodes. |
| Judgment API Credential Setup | Required across all three Judgment evaluation nodes. |