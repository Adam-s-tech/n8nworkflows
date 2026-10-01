Grade LLM answers and route low-confidence results with Judgment API

https://n8nworkflows.xyz/workflows/grade-llm-answers-and-route-low-confidence-results-with-judgment-api-19720


# Grade LLM answers and route low-confidence results with Judgment API

### 1. Workflow Overview

This workflow evaluates AI-generated responses against reference answers using the Judgment API, computes confidence metrics and standardized scores, and routes outputs based on trust thresholds. The architecture separates automated, high-confidence grading results from low-confidence results requiring human intervention.

The logic groups into four functional blocks:
- **1.1 Trigger and Data Initialization:** Initializes the execution flow manually and supplies predefined evaluation datasets containing questions, generated answers, and references.
- **1.2 AI Evaluation Processing:** Interacts with the external Judgment API via a community node to run multi-criteria linguistic, factual, and quality assessments across six distinct parameters per item.
- **1.3 Normalization and Trust Scoring:** Transforms raw evaluation outputs into uniform schemas, calculates normalized $0\text{--}1$ scores, and applies conditional trust rules based on response types and confidence metrics.
- **1.4 Conditional Routing:** Evaluates trust determinations and branches execution paths into trusted logs/dashboards or manual review queues.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Initialization
- **Overview:** Starts the manual test execution and populates the execution context with a structured dataset of questions, answers, and reference texts.
- **Nodes Involved:** `When Clicking Test Workflow`, `Sample Answers`
- **Node Details:**
  - **When Clicking Test Workflow**
    - *Type:* `n8n-nodes-base.manualTrigger`
    - *Technical Role:* Entry point for manual executions.
    - *Configuration choices:* Default settings.
    - *Key expressions:* None.
    - *Input connections:* None.
    - *Output connections:* `Sample Answers` (Index 0).
    - *Version requirements:* Type Version 1.
    - *Potential failure types:* None.
  - **Sample Answers**
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Supplies raw mock data consisting of questions, generated LLM answers, and reference ground truth values.
    - *Configuration choices:* Custom JavaScript returning an array of 3 JSON objects.
    - *Key expressions:* None.
    - *Input connections:* `When Clicking Test Workflow` (Index 0).
    - *Output connections:* `Grade Answers` (Index 0).
    - *Version requirements:* Type Version 2.
    - *Potential failure types:* JavaScript syntax errors if modified incorrectly.

#### 2.2 AI Evaluation Processing
- **Overview:** Submits input triples to the Judgment API community node to perform parallel evaluations across six analytical categories.
- **Nodes Involved:** `Grade Answers`
- **Node Details:**
  - **Grade Answers**
    - *Type:* `n8n-nodes-judgment.judgment`
    - *Technical Role:* Communicates with the Judgment API to assess correctness, completeness, groundedness, relevance, clarity, and overall ship/edit/reject metrics.
    - *Configuration choices:* State set to `fields`; dynamically maps question, answer, and reference parameters. Configures custom levels, options, and descriptions for six evaluation criteria (`correct`, `complete`, `grounded`, `relevant`, `clarity`, `overall`).
    - *Key expressions:* 
      - `={{ $json.question }}`
      - `={{ $json.answer }}`
      - `={{ $json.reference }}`
    - *Input connections:* `Sample Answers` (Index 0).
    - *Output connections:* `Verdict` (Index 0).
    - *Version requirements:* Type Version 1; requires community node package `n8n-nodes-judgment`.
    - *Potential failure types:* Authentication errors due to missing or invalid API keys, network timeouts, or rate limit exceeded errors from the provider console (`console.typesafe.ai/keys`).

#### 2.3 Normalization and Trust Scoring
- **Overview:** Extracts evaluation payloads, maps conditional metrics into standardized attributes, computes normalized scores, and determines trust values.
- **Nodes Involved:** `Verdict`
- **Node Details:**
  - **Verdict**
    - *Type:* `n8n-nodes-base.set`
    - *Technical Role:* Data transformation node that structures evaluation outputs and assigns trust flags.
    - *Configuration choices:* Configured to include other fields and create six assignments (`grade`, `answerId`, `value`, `score0to1`, `judgeConfidence`, `trust`).
    - *Key expressions:*
      - `grade`: `={{ $json.id }}`
      - `answerId`: `={{ $json.raw.request_id ?? $json.requestId }}`
      - `value`: `={{ $json.type === 'choice' ? $json.choice : $json.value }}`
      - `score0to1`: `={{ $json.type === 'noul' ? $json.value : $json.type === 'choice' ? $json.answer.probabilities[$json.choice] : $json.value / (Object.keys($json.answer.legend).length - 1) }}`
      - `judgeConfidence`: `={{ $json.confidence }}`
      - `trust`: `={{ $json.type === 'noul' ? ($json.value >= 0.25 && $json.value <= 0.75 ? 'needs review' : 'trusted') : ($json.confidence !== null && $json.confidence >= 0.8 ? 'trusted' : 'needs review') }}`
    - *Input connections:* `Grade Answers` (Index 0).
    - *Output connections:* `Is the Grade Trusted?` (Index 0).
    - *Version requirements:* Type Version 3.4.
    - *Potential failure types:* Expression evaluation failures if expected JSON properties (`$json.type`, `$json.answer`) are missing or malformed.

#### 2.4 Conditional Routing
- **Overview:** Evaluates the computed trust string and splits workflow execution into automated reporting or human review paths.
- **Nodes Involved:** `Is the Grade Trusted?`, `Trusted Grades`, `Needs a Human Look`
- **Node Details:**
  - **Is the Grade Trusted?**
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Conditional router determining whether items pass the trust threshold.
    - *Configuration choices:* Uses loose type validation with a condition checking if `trust` equals `trusted`.
    - *Key expressions:* `={{ $json.trust }}`
    - *Input connections:* `Verdict` (Index 0).
    - *Output connections:* 
      - True: `Trusted Grades` (Index 0)
      - False: `Needs a Human Look` (Index 0)
    - *Version requirements:* Type Version 2.3.
    - *Potential failure types:* Type mismatch or unexpected values in the `trust` field.
  - **Trusted Grades**
    - *Type:* `n8n-nodes-base.noOp`
    - *Technical Role:* Terminal placeholder node for trusted, high-confidence evaluation results destined for logs or dashboards.
    - *Configuration choices:* Default empty configuration.
    - *Key expressions:* None.
    - *Input connections:* `Is the Grade Trusted?` (True branch).
    - *Output connections:* None.
    - *Version requirements:* Type Version 1.
    - *Potential failure types:* None.
  - **Needs a Human Look**
    - *Type:* `n8n-nodes-base.noOp`
    - *Technical Role:* Terminal placeholder node for shaky or low-confidence evaluations requiring manual review.
    - *Configuration choices:* Default empty configuration.
    - *Key expressions:* None.
    - *Input connections:* `Is the Grade Trusted?` (False branch).
    - *Output connections:* None.
    - *Version requirements:* Type Version 1.
    - *Potential failure types:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| How This Works | n8n-nodes-base.stickyNote | Explains workflow purpose, functionality, and customization paths. | None | None | ## LLM-as-a-Judge — Grade Answers and Route the Shaky Ones<br><br>### How it works<br><br>This workflow manually runs a small evaluation set of AI-generated answers, grades each answer with an LLM-as-a-judge step, then normalizes the result into fields such as grade, answer ID, value, and score. It checks whether each grade is trusted and routes confident results to a trusted path while sending shaky or low-confidence grades to a human-review path.<br><br>### Setup steps<br><br>- Configure the credentials and model/settings required by the judgment node used in “Grade Answers”.<br>- Edit “Sample Answers” with the questions and answer examples you want to evaluate.<br>- Review the threshold or condition in “Is the Grade Trusted?” so it matches your definition of a reliable grade.<br><br>### Customization<br><br>You can replace the no-op endpoints with actions such as saving trusted grades to a database, posting human-review tasks to Slack, or creating tickets for questionable grades. |
| Install Note | n8n-nodes-base.stickyNote | Documents community node installation requirements and credential setup. | None | None | ### ⚠️ Install the community node first<br><br>**Settings → Community Nodes → Install →** `n8n-nodes-judgment`<br><br>If the **Grade Answers** node shows as `[?]`, the package is not installed on this instance. The node type is `n8n-nodes-judgment.judgment`, so it only resolves once the package above is listed under **Settings → Community Nodes**.<br><br>Then add your API key on the **Judgment API** credential. The key comes from the provider console (for TypeSafe: console.typesafe.ai/keys).<br><br>This node makes typed judgements about text. It never writes text. |
| Verdict Rules | n8n-nodes-base.stickyNote | Details trust thresholds and confidence rules for different response types. | None | None | ### Verdict rules<br><br>Choice and Score answers set `lowConfidence` when confidence is below **0.8**. Noul answers carry no confidence at all, so they get a different rule: shaky when the probability sits **between 0.25 and 0.75**.<br><br>| Question | Trust is read from |<br>| --- | --- |<br>| `correct`, `complete`, `grounded`, `relevant` (Noul) | distance of `value` from 0.5 |<br>| `clarity` (Score) | `confidence` |<br>| `overall` (Choice) | `confidence` |<br><br>#### Why the Noul band rarely fires<br><br>A Noul answer near 0.9 is a confident *yes* and one near 0.1 is a confident *no*. Both are trustworthy even when the answer itself is bad. In the samples, **answer 3 scores 0.02 on `correct`** — a confident rejection, so it is trusted.<br><br>Only a genuinely split answer, close to 0.5, lands in the band. That is a real outcome, just a less common one, and worth writing down rather than treating as an accident.<br><br>Edit the rules in the **Verdict** node. |
| When Clicking Test Workflow | n8n-nodes-base.manualTrigger | Triggers manual test executions. | None | Sample Answers | |
| Sample Answers | n8n-nodes-base.code | Supplies mock data containing questions, answers, and references. | When Clicking Test Workflow | Grade Answers | |
| Grade Answers | n8n-nodes-judgment.judgment | Evaluates answers against criteria using the Judgment API. | Sample Answers | Verdict | |
| Verdict | n8n-nodes-base.set | Normalizes evaluation attributes and calculates trust verdicts. | Grade Answers | Is the Grade Trusted? | |
| Is the Grade Trusted? | n8n-nodes-base.if | Routes execution based on the calculated trust value. | Verdict | Trusted Grades, Needs a Human Look | |
| Trusted Output | n8n-nodes-base.stickyNote | Highlights expected metrics for trusted evaluation results. | None | None | **15 of the 18 grades** land here.<br><br>This is the reportable set: what you log, chart and aggregate. Nothing here needs a person, including answer 3's `overall` — a `reject` at **0.98** confidence, which is a grade you can act on today.<br><br>Wire your scoreboard, sheet or dashboard in here. |
| Trusted Grades | n8n-nodes-noOp | Acts as a terminal endpoint for trusted grades. | Is the Grade Trusted? | None | |
| Needs the Human | n8n-nodes-base.stickyNote | Details items routed to human review queues. | None | None | **3 of the 18 grades** land here. All three carry confidence, because only Choice and Score answers ever do:<br><br>| Answer | Grade | Confidence |<br>| --- | --- | --- |<br>| #1 | `overall` = `ship` | 0.72 |<br>| #2 | `overall` = `ship` | 0.45 |<br>| #3 | `clarity` = 1.61 | 0.41 |<br><br>None of these grades is *wrong*. The judge could not commit, which makes them exactly the rows to put in front of a person rather than automating on.<br><br>A Noul answer never appears here. One at 0.02 is a confident *no*, so a confidently bad answer is trusted. **`trusted` is not the same as `good`** — answer 3 is trusted *and* rejected, and you need both columns to decide anything. |
| Needs a Human Look | n8n-nodes-base.noOp | Acts as a terminal endpoint for items requiring human verification. | Is the Grade Trusted? | None | |
| Reading the Table | n8n-nodes-base.stickyNote | Explains schema mappings, axes, and confidence behaviors. | None | None | ### How to read the output<br><br>Six questions per answer, five of which produce one row each. `value` is the answer:<br><br>| ID | Question | `value` |<br>| --- | --- | --- |<br>| `correct` | Agrees with the reference | probability of yes |<br>| `complete` | Answers every part | probability of yes |<br>| `grounded` | Adds nothing unsupported | probability of yes |<br>| `relevant` | Answers the question asked | probability of yes |<br>| `clarity` | How well written | 0, 1 or 2 on the levels |<br>| `overall` | Ship, edit or reject | `value` = probability of the chosen option |<br><br>For `overall`, `answer.probabilities` holds the probability of *every* option, not just the winner.<br><br>### Two axes, not one<br><br>- **`score0to1`** — quality. How good is the answer?<br>- **`trust`** — certainty. Should this grade be acted on?<br><br>A verdict of `reject` with `trusted` is the *most* useful row in the table: the judge is sure the answer is bad. A `reject` with `needs review` means nobody should trust that either.<br><br>### Where confidence comes from<br><br>`judgeConfidence` is `null` on all four Noul questions by design. A probability near 0.5 already means "genuinely unsure", so the number would be redundant. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a `Manual Chat Trigger` or `When Clicking Test Workflow` node (`n8n-nodes-base.manualTrigger`).
2. **Create a Code Node for Sample Data:**
   - Add a `Code` node (`n8n-nodes-base.code`) named `Sample Answers`.
   - Set the mode to JavaScript and insert the array returning three test items containing `question`, `answer`, and `reference` properties.
   - Connect `When Clicking Test Workflow` to `Sample Answers`.
3. **Install and Configure the Judgment Node:**
   - Ensure the community node package `n8n-nodes-judgment` is installed via **Settings → Community Nodes**.
   - Create a new node of type `n8n-nodes-judgment.judgment` named `Grade Answers`.
   - Set the **State** parameter to `fields`.
   - Configure state fields mapping:
     - `question`: `={{ $json.question }}`
     - `answer`: `={{ $json.answer }}`
     - `reference`: `={{ $json.reference }}`
   - Configure evaluation questions:
     - `correct` (Noul type): Define true/false descriptions and question instructions regarding factual agreement.
     - `complete` (Noul type): Define coverage criteria.
     - `grounded` (Noul type): Define unsupported claim checks.
     - `relevant` (Noul type): Define topic relevance rules.
     - `clarity` (Score type): Configure three levels ranging from confusing to clear.
     - `overall` (Choice type): Configure options `ship`, `edit`, and `reject` with descriptions.
   - Set up credentials by adding a `Judgment API` credential with a valid API key from the provider console (`console.typesafe.ai/keys`).
   - Connect `Sample Answers` output to `Grade Answers`.
4. **Create a Set Node for Verdict Normalization:**
   - Add a `Set` (`n8n-nodes-base.set`) node named `Verdict`.
   - Enable `Include Other Fields`.
   - Add the following property assignments:
     - `grade` (string): `={{ $json.id }}`
     - `answerId` (string): `={{ $json.raw.request_id ?? $json.requestId }}`
     - `value` (string): `={{ $json.type === 'choice' ? $json.choice : $json.value }}`
     - `score0to1` (number): `={{ $json.type === 'noul' ? $json.value : $json.type === 'choice' ? $json.answer.probabilities[$json.choice] : $json.value / (Object.keys($json.answer.legend).length - 1) }}`
     - `judgeConfidence` (number): `={{ $json.confidence }}`
     - `trust` (string): `={{ $json.type === 'noul' ? ($json.value >= 0.25 && $json.value <= 0.75 ? 'needs review' : 'trusted') : ($json.confidence !== null && $json.confidence >= 0.8 ? 'trusted' : 'needs review') }}`
   - Connect `Grade Answers` to `Verdict`.
5. **Create an If Node for Trust Routing:**
   - Add an `If` (`n8n-nodes-base.if`) node named `Is the Grade Trusted?`.
   - Configure a condition where string `={{ $json.trust }}` equals `trusted`.
   - Connect `Verdict` to `Is the Grade Trusted?`.
6. **Create Terminal NoOp Nodes:**
   - Add a `NoOp` node named `Trusted Grades` (`n8n-nodes-base.noOp`).
   - Add a `NoOp` node named `Needs a Human Look` (`n8n-nodes-base.noOp`).
   - Connect the True branch of `Is the Grade Trusted?` (Index 0) to `Trusted Grades`.
   - Connect the False branch of `Is the Grade Trusted?` (Index 1) to `Needs a Human Look`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Judgment API Console Keys | [TypeSafe Console](https://console.typesafe.ai/keys) |
| Community Node Package Identifier | `n8n-nodes-judgment` |