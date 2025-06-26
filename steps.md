Belows are steps I design for the evaluation process of questions and answers for a pdf file, lets review it

---

### **Step 1: Answer-Type Conformance Check (The Triage)**

#### **Goal**

To immediately determine if the AI's answer has the correct _structure_ for the question being asked. This step acts as a smart "triage" to filter out fundamentally flawed responses before wasting time on detailed fact-checking.

#### **Core Principle**

We are not yet checking if the answer is _correct_, but if it's a _valid attempt_. A question that expects a "Yes/No" answer shouldn't receive a vague paragraph. A question about a contradiction shouldn't be answered by picking one side. This step validates the answer's fundamental shape.

#### **Detailed Inputs for the LLM Judge**

1.  The ground-truth `question`.
2.  The ground-truth `answer_type` (e.g., `definitive`, `contradiction_report`, `no_information`).
3.  The `generated_answer` from the AI system.

#### **The LLM Judge Prompt**

```
You are an expert evaluator. Your task is to check if a Generated Answer's structure conforms to the Expected Answer Type for the given Question.

**Question:** "[question]"
**Expected Answer Type:** "[answer_type]"
**Generated Answer:** "[generated_answer]"

Based on the Expected Answer Type, analyze the structure of the Generated Answer and provide a single classification from the list below.

---
**A. If Expected Answer Type is `definitive`:**
- `conforms`: The answer provides a direct, conclusive statement.
- `non_conforming_evasive`: The answer discusses the topic but avoids giving the required conclusion.
- `non_conforming_irrelevant`: The answer is off-topic.

**B. If Expected Answer Type is `contradiction_report`:**
- `conforms`: The answer explicitly states that the source contains conflicting or ambiguous information.
- `non_conforming_picks_one_side`: The answer gives one of the conflicting facts but ignores the other.
- `non_conforming_fails_to_identify`: The answer says it doesn't know or gives other irrelevant info.

**C. If Expected Answer Type is `no_information`:**
- `conforms`: The answer correctly states that the information is not available in the source document.
- `non_conforming_hallucinates`: The answer invents an answer instead of admitting the information is missing.
---

**Your Classification:**
```

#### **Illustrative Examples**

**Example 1.1: The Contradiction Test**

- **Context:** Page 10 says "The project was approved for a Q2 start." A table on page 45 lists the "Project Kickoff" date as "July 1st."
- **Ground Truth:**
  - `question`: "What was the start date of the project?"
  - `answer_type`: `contradiction_report`

| Scenario                                | Generated Answer                                                                                                                      | LLM Judge's Analysis                                                                                                                                                         | Result                                                                                                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. The Good Answer** (Handles Nuance) | "The start date is ambiguous. The document states it was approved for a Q2 start, but the kickoff meeting was held on July 1st (Q3)." | The answer's structure matches the `contradiction_report` type. It correctly identifies and presents the conflict.                                                           | **`conforms`**. The evaluation proceeds. This AI is sophisticated.                                                                                                  |
| **B. The Bad Answer** (Misses Nuance)   | "The project start date was in Q2."                                                                                                   | The answer fails to report the contradiction. It picks one piece of information and presents it as the sole truth. This does not conform to the `contradiction_report` type. | **`non_conforming_picks_one_side`**. **Penalty applied.** This AI is not trustworthy for nuanced tasks. It gives a deceptively simple answer to a complex question. |

**Why this matters:** This step immediately separates AIs that can handle real-world ambiguity from those that naively grab the first fact they find.

### **Step 2 (Revised): Factual Verification, Focus, and Hallucination Auditing**

#### **Goal**

To perform a two-stage audit of the answer. First, we measure its completeness against the _required_ facts. Second, we audit any "extra" information against the original source text to distinguish between unfocused-but-true statements (**low focus**) and fabricated statements (**hallucinations**).

#### **Core Principle**

An answer's quality has two dimensions of truthfulness:

1.  **Completeness:** Did it include all the facts _required_ to answer the question? (Measured against `atomic_facts`).
2.  **Fidelity:** Is everything it says grounded in the source document? (Measured against the `source_chunk`).

A failure in #1 means the answer is incomplete. A failure in #2 means the answer is untrustworthy.

#### **Detailed Inputs for the LLM Judge**

1.  The `generated_answer`.
2.  The list of ground-truth `atomic_facts`.
3.  The original `source_chunk` of text from the document.

#### **The LLM Judge Prompt (Now a Two-Part Process)**

The evaluation is now a chain of two prompts to ensure a rigorous audit.

**Process Part 1: Initial Verification against `atomic_facts`**

The first prompt is the same as before, designed to find what's missing and what might be "extra."

```
You are a meticulous fact-checker.

**Generated Answer:** "[generated_answer]"
**Required Atomic Facts:** [List of atomic_facts]

**Task:**
1.  For each Atomic Fact, classify its presence in the Generated Answer as `full_match`, `partial_match`, or `no_match`.
2.  Identify any substantive claims in the Generated Answer that are NOT covered by the Atomic Facts. List these as `unverified_statements`.

**Provide your output in this exact JSON format:**
{
  "fact_verification": [ ... ],
  "unverified_statements": [ "[claim_1]", "[claim_2]" ]
}
```

**Process Part 2: The Hallucination Audit (The Arbiter)**

This part only runs if the `unverified_statements` list from Part 1 is not empty. The LLM Judge now acts as an arbiter, using the source text as the ground truth.

```
You are an auditor. For each statement in the `unverified_statements` list, you must determine if it is supported by the provided `source_chunk`.

**Source Chunk:** "[The original text from the document]"
**Unverified Statements:** [List of unverified_statements from Part 1]

**Task:**
For each statement, classify it as:
- `supported_by_source`: The statement is demonstrably true based on the source chunk.
- `not_supported_by_source`: The statement is NOT supported by the source chunk. This is a hallucination.

**Provide your output in this exact JSON format:**
{
  "audit_results": [
    {"statement": "[claim_1]", "status": "[classification]"},
    {"statement": "[claim_2]", "status": "[classification]"}
  ]
}
```

#### **Scoring with the New, More Nuanced Results**

This two-part process gives us richer data to calculate more precise scores.

- **`Factual Score` (Completeness):** Unchanged. Calculated from the `fact_verification` results. Measures if the answer included what was required.
- **`Hallucination Score` (Fidelity):** Now much more accurate. It is set to `0` **only if** the audit in Part 2 finds one or more statements with the status `not_supported_by_source`.
- **`Focus Score` (Conciseness):** This is a **new score** that punishes the model for being "chatty" but not for lying. It is set to `0` if the audit in Part 2 finds one or more statements with the status `supported_by_source`. Otherwise, it's `1`.

#### **Illustrative Example**

- **Context:** A paragraph on page 50 states: "The 'Vanguard' project, led by Maria Flores, had a performance target of 1 million TPS. Final testing showed a result of 1.2 million TPS."
- **Ground Truth:**
  - `question`: "Did the 'Vanguard' system meet its performance target?"
  - `atomic_facts`: [`"The 'Vanguard' system's target was 1 million TPS."`, `"The final result was 1.2 million TPS."`]
  - `source_chunk`: The full paragraph text above.
- **Generated Answer:** "Yes, the 'Vanguard' system met its target. It achieved 1.2 million TPS against a goal of 1 million TPS. The project was led by Maria Flores."

**How the Evaluation Unfolds:**

1.  **Part 1 - Initial Verification:**

    - The Judge analyzes the answer against the `atomic_facts`.
    - `fact_verification`: Both facts get `full_match`.
    - `unverified_statements`: `["The project was led by Maria Flores."]`

2.  **Part 2 - Hallucination Audit:**

    - The Judge is given the `source_chunk` and the unverified statement.
    - It checks if "The project was led by Maria Flores" is supported by the source text. It is.
    - `audit_results`: `{"statement": "The project was led by Maria Flores.", "status": "supported_by_source"}`

3.  **Final Scoring:**
    - **`Factual Score`:** `(1 + 1) / 2 = 1.0`. (Perfect completeness).
    - **`Hallucination Score`:** `1`. (No statements were `not_supported_by_source`. The AI is trustworthy).
    - **`Focus Score`:** `0`. (An extraneous but true fact was included. The AI is unfocused/chatty).

**Why this is a crucial improvement:**

This revised workflow prevents us from unfairly punishing a truthful AI. We can now distinguish between three distinct answer profiles:

- **Good Answer:** High Factual, Hallucination, and Focus scores.
- **Incomplete Answer:** Low Factual Score, but high Hallucination and Focus scores.
- **Unfocused Answer:** High Factual and Hallucination scores, but a low Focus Score.
- **Untrustworthy Answer:** A Hallucination Score of 0, which is the most severe failure.

This gives us a far more accurate and actionable diagnosis of the AI's behavior.

---

### **Step 3 (Refined & Finalized): Conclusion & Reasoning Verification**

#### **Goal**

To evaluate if the AI correctly produced the required conclusion from a reasoning process, and optionally, to assess the transparency and trustworthiness of that process. This step applies to Level 2 and 3 questions.

#### **Core Principle**

We will follow a modular, two-part evaluation.

1.  **Part A (Correctness):** We first check the most important thing: Is the final answer correct? This is a mandatory, non-negotiable check.
2.  **Part B (Trust & Transparency):** We then perform an optional, separate check: Did the AI explain its reasoning, and was that explanation sound? This measures the AI's reliability.

#### **Required Ground-Truth Fields (Simplified)**

- **`atomic_facts`**: The raw evidence.
- **`reasoning_steps`**: The logic to get from facts to the answer.
- **`final_answer`**: The single, definitive, bottom-line answer (e.g., `"25%"`, `"Yes"`, `"$120,000"`). This is simple to generate and provides a clear target.

---

### **Part A: The Correctness Score (Mandatory)**

This is the primary evaluation of the reasoning task.

- **Inputs to LLM Judge:**

  1.  The `generated_answer`.
  2.  The ground-truth `final_answer`.

- **LLM Judge Prompt:**

  ```
  You are an evaluator. Your task is to determine if the final conclusion in a Generated Answer is correct.

  **Generated Answer:** "[generated_answer]"
  **Correct Final Answer:** "[final_answer]"

  ---
  **Your Task:**
  1.  Carefully read the Generated Answer and identify its main, bottom-line conclusion.
  2.  Compare this conclusion to the "Correct Final Answer". They must be semantically equivalent (e.g., "$1.2 million" is the same as "1,200,000 dollars"; "Yes, it requires a review" contains the correct answer "Yes").

  **Provide a single classification:**
  - `correct_and_present`: The correct final answer is found within the Generated Answer.
  - `incorrect_or_absent`: The final answer is wrong, or the Generated Answer fails to state a final conclusion.
  ```

- **Scoring (The `Reasoning_Accuracy_Score`):**
  - `correct_and_present` -> **1.0**
  - `incorrect_or_absent` -> **0.0**
    This is a strict, binary score. For Levels 2 and 3, the answer is either right or wrong.

---

### **Part B: The Trust & Transparency Score (Optional)**

This secondary evaluation assesses the quality of the AI's explanation.

- **Inputs to LLM Judge:**

  1.  The `generated_answer`.
  2.  The `atomic_facts` (for context).
  3.  The `reasoning_steps` (for context).

- **LLM Judge Prompt:**

  ```
  You are an analyst evaluating the transparency of an AI's reasoning. Do not worry about whether the final number is correct. Focus only on the quality of the explanation provided.

  **Generated Answer:** "[generated_answer]"
  **Context (Facts & Logic):** [Provide atomic_facts and reasoning_steps]

  ---
  **Your Task:**
  Review the Generated Answer and classify the quality of its explanation. An explanation is any text that describes how the conclusion was reached.

  **Provide a single classification:**
  - `clear_and_correct_explanation`: The answer provides an explanation, and that explanation is logically sound and consistent with the provided context.
  - `no_explanation_provided`: The answer gives a conclusion but provides no explanation for how it was reached.
  - `flawed_explanation`: The answer provides an explanation, but it is logically incorrect, contains errors, or is confusing.
  ```

- **Scoring (The `Explanation_Quality_Score`):**
  - `clear_and_correct_explanation` -> **1.0** (Trustworthy and transparent)
  - `no_explanation_provided` -> **0.5** (Correct but opaque; a moderate flaw)
  - `flawed_explanation` -> **0.0** (Untrustworthy; a critical flaw)

### **Why This Refined Step 3 is Strong**

- **Practical:** It relies on a simple-to-create `final_answer` field, making the entire project feasible.
- **Focused:** It separates the evaluation of **correctness** from the evaluation of **explanation**. This gives a clearer, more actionable diagnosis of the AI's performance.
- **Robust:** It correctly identifies the most dangerous failure mode—an AI that gives a `flawed_explanation`—and penalizes it most heavily in the trust score.
- **Modular:** The project lead can decide whether to run only the mandatory "Correctness" check or to also run the optional "Trust" check, depending on the evaluation goals.

### **Step 4: Nuance Verification**

#### **Goal**

To evaluate the AI's ability to handle two subtle but critical aspects of language and information:

1.  **Attribution:** Correctly identifying and reporting that a piece of information is an opinion or statement from a specific source, not an objective fact.
2.  **Judgment:** Refraining from injecting its own unstated analysis or evaluative language (like "good," "bad," "successful") into what should be a factual answer.

#### **Core Principle**

A truly advanced AI doesn't just act as a search engine; it acts as a careful reporter. It understands provenance and the difference between fact and opinion. This step audits that specific capability. A failure here doesn't mean the answer is factually wrong, but it does mean the AI is less reliable and less suitable for tasks requiring high fidelity.

#### **Detailed Inputs for the LLM Judge**

1.  The `generated_answer`.
2.  The `atomic_facts` list from our ground truth. (The Judge will be instructed to pay special attention to facts of type `Attributed Statements`).
3.  The `opinions_from_answer` list from our ground truth. (This list contains interpretive judgments _we_ made in our `ideal_answer`, serving as a baseline for what constitutes an "opinion").

#### **The LLM Judge Prompt**

```
You are a nuance and source analyst. Your task is to audit a Generated Answer for two specific qualities: correct attribution of opinions and avoidance of unstated judgments.

**Generated Answer:** "[generated_answer]"

**Context for your analysis:**
- **Atomic Facts from Source:** [Full list of atomic_facts]
- **Example Judgments (from our Ideal Answer):** [List of opinions_from_answer]

---
**Part 1: Attribution Check**

1.  First, review the "Atomic Facts" list and identify any that are **Attributed Statements** (e.g., "Project Manager Sarah Chen stated...", "According to the Gartner report...").
2.  Now, read the Generated Answer. For each Attributed Statement that is present in the answer, did the AI correctly credit the source? Or did it present the opinion as an objective fact?

**Provide a single classification for Attribution Quality:**
- `correctly_attributed`: All attributed statements mentioned in the answer were correctly attributed to their source.
- `failed_to_attribute`: At least one attributed statement was mentioned in the answer but was presented as objective fact without sourcing.
- `not_applicable`: The Generated Answer did not mention any of the Attributed Statements.

---
**Part 2: Unstated Judgment Check**

1.  First, review the "Example Judgments" to understand what kind of interpretive language to look for (e.g., "highly successful," "a significant risk," "a poor outcome").
2.  Now, read the Generated Answer. Does it contain similar evaluative or interpretive language that is **not** a direct quote from an attributed source in the Atomic Facts?

**Provide a single classification for Judgment Injection:**
- `made_unstated_judgment`: The answer contains its own analysis or opinion (e.g., calling a project "successful" when the source only provides the data).
- `stated_only_facts_and_quotes`: The answer strictly sticks to reporting the facts and correctly attributed quotes.
- `not_applicable`: This check is not relevant for this question type.
```

#### **Scoring and Flag Generation**

This step produces qualitative flags, not a numerical score, because these are nuanced traits.

- **`Attribution_Flag`**:

  - `correctly_attributed` -> **PASSED**
  - `failed_to_attribute` -> **FAILED**
  - `not_applicable` -> `N/A`

- **`Judgment_Flag`**:
  - `stated_only_facts_and_quotes` -> **PASSED**
  - `made_unstated_judgment` -> **FAILED**
  - `not_applicable` -> `N/A`

#### **Illustrative Examples**

**Example 4.1: The Attribution Failure**

- **Context:** Source text says, "Project Manager Sarah Chen commented, 'This is a phenomenal result.'"
- **Ground Truth:**
  - `atomic_facts`: [`"Project Manager Sarah Chen described the result as 'phenomenal'."`]
- **Generated Answer:** "The project achieved a phenomenal result."

| LLM Judge's Analysis - **Attribution Check:** The Judge sees the AI used the word "phenomenal" but did not attribute it to Sarah Chen. | **`Attribution_Flag`:** **FAILED**. <br><br> **Why this matters:** The AI has presented a subjective opinion as an objective fact. This is a subtle but serious error that erodes trust. |

**Example 4.2: The Unstated Judgment Failure**

- **Context:** Source text says, "The system processed 1.2M TPS against a target of 1M TPS." (No evaluative words are used).
- **Ground Truth:**
  - `atomic_facts`: [`"The system processed 1.2M TPS."`, `"The target was 1M TPS."`]
  - `opinions_from_answer`: [`"The conclusion that the performance was 'highly successful' is an interpretive summary."`]
- **Generated Answer:** "The system's performance was highly successful, processing 1.2M TPS and beating its 1M TPS target."

| LLM Judge's Analysis - **Judgment Check:** The Judge sees the phrase "highly successful." This is an evaluative statement not present in the source facts. It matches the type of opinion we flagged in our ground truth. | **`Judgment_Flag`:** **FAILED**. <br><br> **Why this matters:** The AI is adding its own layer of analysis without being asked to. While "successful" might seem correct, a high-fidelity system should not do this unless explicitly prompted. It shows a lack of discipline in sticking to the source. |

This step provides crucial qualitative flags that help us build a complete profile of the AI's behavior, complementing the numerical scores from the other steps.

### **Step 5: Aggregation and The Final Scorecard**

#### **Goal**

To synthesize the detailed, multi-faceted outputs from Steps 1-4 into a single, structured object—the "Scorecard." This Scorecard serves as the definitive record of the AI's performance on one specific question-answer pair.

#### **Core Principle**

Raw LLM outputs are useful but not immediately readable. This step translates the structured JSONs and classifications from the Judge into a clean, hierarchical report. It calculates the final numerical scores and presents the qualitative flags in a clear format, adding a high-level summary for quick analysis.

#### **Process**

This is a deterministic data aggregation process performed by a script or program.

1.  **Gather Inputs:** The system collects all the JSON outputs and classifications generated by the LLM Judge for a single question:

    - From Step 1: The `triage_status` (e.g., `conforms`).
    - From Step 2: The `fact_verification` list, the `hallucinated_statements` list, and the `supported_by_source` list.
    - From Step 3: The `conclusion_classification` (e.g., `correct_and_present`) and the `explanation_classification` (e.g., `clear_and_correct_explanation`).
    - From Step 4: The classifications for `Attribution Quality` and `Unstated Judgment`.

2.  **Calculate Final Scores:** The script computes the final scores based on the predefined logic:

    - **`Factual_Score`:** Calculated from the `fact_verification` list (`full_match`=1, `partial_match`=0.5, `no_match`=0).
    - **`Focus_Score`:** Set to `0` if any `supported_by_source` statements were found, otherwise `1`.
    - **`Hallucination_Score`:** Set to `0` if any `hallucinated_statements` were found, otherwise `1`.
    - **`Reasoning_Accuracy_Score`:** `1.0` if conclusion was `correct_and_present`, otherwise `0.0`.
    - **`Explanation_Quality_Score`:** Mapped from the classification (e.g., `clear_and_correct_explanation` -> 1.0).

3.  **Generate Final Flags:** The script converts the classifications from Step 4 into simple `PASSED`/`FAILED` flags.

    - **`Attribution_Flag`:** `FAILED` if `failed_to_attribute`, otherwise `PASSED`.
    - **`Judgment_Flag`:** `FAILED` if `made_unstated_judgment`, otherwise `PASSED`.

4.  **Assemble the Scorecard Object:** All this information is structured into a final JSON object.

---

### **The Final Scorecard Structure**

This is the definitive output for a single question.

```json
{
  "question_id": "viper-review-001",
  "question_text": "According to company policy, does Project Viper require a formal review by the PMO?",
  "difficulty_level": 3,
  "triage_status": "conforms",
  "scores": {
    "factual_score": 1.0,
    "focus_score": 1,
    "hallucination_score": 1,
    "reasoning_accuracy_score": 1.0,
    "explanation_quality_score": 1.0
  },
  "flags": {
    "attribution_flag": "N/A",
    "judgment_flag": "PASSED"
  },
  "llm_judge_diagnostics": {
    "fact_verification_details": [
      {
        "fact": "Policy: >10% over budget requires review.",
        "status": "full_match"
      },
      { "fact": "Viper Budget: $100,000.", "status": "full_match" },
      { "fact": "Viper Actuals: $115,000.", "status": "full_match" }
    ],
    "reasoning_conclusion_status": "correct_and_present",
    "reasoning_explanation_status": "clear_and_correct_explanation",
    "hallucinated_statements": [],
    "unfocused_statements": []
  },
  "generated_answer_text": "Yes, Project Viper requires a formal review because its costs of $115,000 exceeded its budget of $100,000 by 15%, which is over the 10% policy threshold."
}
```

---

### **Illustrative Example: A More Complex Failure**

Let's see how this Scorecard captures a nuanced failure.

- **Question:** "What was the profit margin for Project Orion?"
- **Generated Answer:** "The profit margin was 25%. Project Orion was a very successful project."

**The resulting Scorecard might look like this:**

```json
{
  "question_id": "orion-margin-001",
  "question_text": "What was the profit margin for Project Orion?",
  "difficulty_level": 2,
  "triage_status": "conforms",
  "scores": {
    "factual_score": 1.0, // It included all the necessary facts (implicitly)
    "focus_score": 1, // No extra *facts* were added
    "hallucination_score": 1, // No facts were hallucinated
    "reasoning_accuracy_score": 1.0, // The final answer "25%" was correct
    "explanation_quality_score": 0.5 // No explanation was provided
  },
  "flags": {
    "attribution_flag": "N/A",
    "judgment_flag": "FAILED" // It made an unstated judgment ("very successful")
  },
  "llm_judge_diagnostics": {
    // ... detailed outputs ...
    "reasoning_explanation_status": "no_explanation_provided"
  },
  "generated_answer_text": "The profit margin was 25%. Project Orion was a very successful project."
}
```

**Why this is powerful:** A simple "pass/fail" would say this answer is correct. Our Scorecard provides a much deeper diagnosis: "The AI got the right number (`Reasoning_Accuracy`=1.0), but its response was opaque (`Explanation_Quality`=0.5) and it injected its own unstated opinion (`Judgment_Flag`=FAILED). This indicates a correct but untrustworthy and unsophisticated system."

You are right to ask. Step 5 creates the detailed report for a _single_ question. But the ultimate goal of this entire project is to evaluate the AI system _as a whole_.

So, no, that is not all. The final, and arguably most important, step is to zoom out from the individual scorecards and create the high-level, system-wide analysis.

---

### **Step 6: System-Level Reporting & Analytics**

#### **Goal**

To aggregate the hundreds or thousands of individual Scorecards (from Step 5) into a comprehensive dashboard and analytical report. This report should answer the key question: "How good is this AI system, and where are its specific strengths and weaknesses?"

#### **Core Principle**

Individual data points are noise; aggregated trends are insight. This step moves from micro-level analysis (one question) to macro-level conclusions (system-wide behavior). It's where we generate the actionable intelligence for developers, product managers, and leadership.

#### **Process**

This is a data analysis and visualization step, performed by code (e.g., Python scripts using libraries like Pandas and Matplotlib/Seaborn, or by loading the data into a BI tool like Tableau or Power BI).

1.  **Load the Data:** Ingest all the JSON Scorecard objects generated in Step 5 into a structured format like a database or a Pandas DataFrame.

2.  **Calculate Aggregate Metrics:** Compute system-wide averages for all the key numerical scores:

    - Overall Average `Factual_Score`
    - Overall Average `Reasoning_Accuracy_Score`
    - Overall Average `Explanation_Quality_Score`

3.  **Calculate Failure Rates:** Compute percentages for the binary scores and flags:

    - **`Hallucination Rate`**: Percentage of questions where `Hallucination_Score` was 0.
    - **`Unfocused Rate`**: Percentage of questions where `Focus_Score` was 0.
    - **`Attribution Failure Rate`**: Percentage of relevant questions where `Attribution_Flag` was `FAILED`.
    - **`Judgment Failure Rate`**: Percentage of relevant questions where `Judgment_Flag` was `FAILED`.

4.  **Segment the Analysis:** This is the most critical part. The overall averages are good, but the real insights come from slicing the data by the metadata in our ground-truth dataset. We calculate the metrics above for different segments:

    - **By Difficulty Level:** How does `Reasoning_Accuracy_Score` change from Level 1 to Level 2 to Level 3?
    - **By Source Type:** Is the `Factual_Score` lower for facts from `tables` versus `text`?
    - **By Question Type:** Does the AI hallucinate more on `no_information` questions? Does it fail attribution on questions involving `Attributed Statements`?

5.  **Visualize the Results:** Create clear charts and tables to present the findings. Bar charts are excellent for comparing segments, and tables are good for detailed breakdowns.

---

### **The Final Deliverable: The System Evaluation Report**

This is the ultimate output of the entire project. It's a document or dashboard that might include sections like:

#### **1. Executive Summary (The "At a Glance" View)**

- **Overall Correctness Score:** A blended score of Factual and Reasoning accuracy.
- **Trust & Safety Score:** A score based on Hallucination, Attribution, and Judgment failure rates.
- **Top 3 Strengths:** e.g., "Excellent factual recall from prose," "Strong on Level 2 calculations."
- **Top 3 Weaknesses:** e.g., "Struggles with reasoning on Level 3 policy questions," "Frequently presents opinions as facts."

#### **2. Deep Dive: Performance by Difficulty**

- A bar chart showing the `Reasoning_Accuracy_Score` for Level 1, 2, and 3.
  - _Insight Example:_ "The model is nearly perfect at direct lookup (L1) and simple synthesis (L2), but its performance drops by 40% on complex deduction (L3), indicating a weakness in multi-step logical inference."

#### **3. Deep Dive: Performance by Information Source**

- A bar chart showing the `Factual_Score` for questions sourced from `text`, `table`, `chart`, and `figure`.
  - _Insight Example:_ "The model's ability to extract facts from tables is significantly lower than from text. This suggests a weakness in its document layout understanding or table parsing capabilities."

#### **4. Deep Dive: Trust and Safety Analysis**

- **Hallucination Rate:** "The system exhibits a very low hallucination rate of 0.5%, making it generally safe."
- **Attribution Failure Rate:** "The system failed to attribute opinions in 35% of cases where it was required. This is a significant training gap for high-fidelity applications."
- **Judgment Failure Rate:** "The system injected its own unstated opinions in 15% of answers, showing a tendency to be overly interpretive."

#### **5. Actionable Recommendations**

- "To improve performance, focus on fine-tuning with more examples of multi-step logical deduction (Level 3)."
- "The data extraction module for tables needs to be reviewed and improved."
- "Specialized training is required to teach the model the difference between presenting facts and attributing opinions."

This final step transforms our detailed, per-question data into strategic business and engineering intelligence, which is the true value of building this comprehensive evaluation framework.
