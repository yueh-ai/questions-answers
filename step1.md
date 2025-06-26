### **Step 1: Answer-Type Conformance Check (The Triage)**

#### **Goal**

To immediately determine if an AI's generated answer has the correct **structure** for the question being asked. This step acts as a smart "triage" to filter out fundamentally flawed responses before wasting time on detailed fact-checking.

#### **Core Principle**

We are not checking if the answer is _correct_, only if it's a _valid attempt_ in the right format. This simplified check focuses on the single most important structural requirement for each answer type.

#### **Supported Answer Types**

- `definitive`: Expects a single, conclusive answer, which may include supporting reasoning.
- `list`: Expects a set of distinct items, typically formatted with bullets or numbers.
- `summary`: Expects a condensed, prose-based overview of a topic.
- `instructional`: Expects a sequence of actionable steps.
- `contradiction_report`: Expects an explicit statement that the source contains conflicting information.
- `no_information`: Expects a statement that the information is not available in the source.

---

#### **The LLM Judge Prompt**

```
You are an expert evaluator. Your task is to check if a Generated Answer's structure conforms to the Expected Answer Type for the given Question.

**Question:** "[question]"
**Expected Answer Type:** "[answer_type]"
**Generated Answer:** "[generated_answer]"

Based on the Expected Answer Type, analyze the structure of the Generated Answer and provide a single classification from the two options provided for that type.

---
**A. If Expected Answer Type is `definitive`:**
- `conforms`: The answer provides a single, unambiguous conclusion, with or without reasoning.
- `non_conforming_indirect_or_evasive`: The answer fails to provide a direct conclusion, is off-topic, or provides a list instead of a conclusion.

**B. If Expected Answer Type is `list`:**
- `conforms`: The answer is formatted as a list of two or more items.
- `non_conforming_not_a_list`: The answer is formatted as prose, a single item, or is otherwise not a list.

**C. If Expected Answer Type is `summary`:**
- `conforms`: The answer is a cohesive block of prose that summarizes the topic.
- `non_conforming_not_a_summary`: The answer is a list, a single isolated fact, or an un-contextualized quote.

**D. If Expected Answer Type is `instructional`:**
- `conforms`: The answer provides a clear sequence of actionable steps.
- `non_conforming_descriptive`: The answer describes a system or outcome but not the steps to achieve it.

**E. If Expected Answer Type is `contradiction_report`:**
- `conforms`: The answer explicitly states that the source contains conflicting or ambiguous information.
- `non_conforming_fails_to_report_contradiction`: The answer picks one side, ignores the conflict, or says it doesn't know.

**F. If Expected Answer Type is `no_information`:**
- `conforms`: The answer correctly states that the information is not available in the source.
- `non_conforming_hallucinates`: The answer invents information instead of admitting it's missing.
---

**Your Classification:**
```

---

#### **Illustrative Examples**

**Example 1.1: The Definitive Answer Test**

- **Context:** A project post-mortem states: "...Given the primacy of the ticket reduction goal, the project is officially classified as a success."
- **Ground Truth:**
  - `question`: "Was the 'Odyssey' initiative a success?"
  - `answer_type`: `definitive`

| Scenario               | Generated Answer                                                                        | LLM Judge's Analysis                                                                     | Result                                                         |
| :--------------------- | :-------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| **A. The Good Answer** | "Yes, the initiative was considered a success because it surpassed its primary goal."   | The answer's structure `conforms` because it provides a direct, unambiguous conclusion.  | **`conforms`**.                                                |
| **B. The Bad Answer**  | "The initiative had mixed results. It reduced support tickets but missed the NPS goal." | This answer does not `conform`. It presents evidence but avoids the required conclusion. | **`non_conforming_indirect_or_evasive`**. **Penalty applied.** |

**Example 1.2: The List Test**

- **Context:** A document section lists five project leaders.
- **Ground Truth:**
  - `question`: "Who are the key stakeholders on the project?"
  - `answer_type`: `list`

| Scenario               | Generated Answer                                                                                                                        | LLM Judge's Analysis                                                                                       | Result                                                |
| :--------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- | :---------------------------------------------------- |
| **A. The Good Answer** | "The key stakeholders are: <ul><li>Chandra Patel</li><li>David Chen</li><li>Maria Garcia</li></ul>"                                     | The answer's structure is a list of multiple items, which correctly conforms to the `list` answer type.    | **`conforms`**.                                       |
| **B. The Bad Answer**  | "Stakeholder management is a critical component of this project's success. The governance philosophy emphasizes clear communication..." | The answer provides a prose summary instead of the requested list of names. This is a structural mismatch. | **`non_conforming_not_a_list`**. **Penalty applied.** |

**Example 1.3: The Summary Test**

- **Context:** A project charter reads: "Project 'Titan' is a multi-year initiative to modernize our legacy billing system. The project will migrate customer records to a new cloud platform and refactor the payment processing module."
- **Ground Truth:**
  - `question`: "Provide an overview of Project Titan."
  - `answer_type`: `summary`

| Scenario               | Generated Answer                                                                                                                                              | LLM Judge's Analysis                                                                                                 | Result                                                   |
| :--------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------- |
| **A. The Good Answer** | "Project Titan is an initiative focused on updating the company's old billing system by moving customer data to the cloud and rebuilding the payment module." | The answer is a cohesive block of prose that correctly summarizes the topic, meeting the `summary` type requirement. | **`conforms`**.                                          |
| **B. The Bad Answer**  | "The project will: <ul><li>Modernize the billing system</li><li>Migrate customer records</li><li>Refactor the payment module</li></ul>"                       | The answer is a bulleted list, not a prose summary. This is a structural violation.                                  | **`non_conforming_not_a_summary`**. **Penalty applied.** |

**Example 1.4: The Instructional Test**

- **Context:** A support article states: "To reset your password, first navigate to the login page. Click the 'Forgot Password?' link. Then, enter your registered email address and submit the form."
- **Ground Truth:**
  - `question`: "How do I reset my password?"
  - `answer_type`: `instructional`

| Scenario               | Generated Answer                                                                                                                           | LLM Judge's Analysis                                                                                                                        | Result                                                 |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :----------------------------------------------------- |
| **A. The Good Answer** | "1. Go to the login page.<br>2. Click 'Forgot Password?'.<br>3. Enter your email and submit."                                              | The answer provides a clear, numbered sequence of actionable steps, which conforms to the `instructional` type.                             | **`conforms`**.                                        |
| **B. The Bad Answer**  | "The system allows for password resets via an email-based process. This is a standard security feature designed to protect user accounts." | This answer describes the purpose of the feature but does not provide the steps for a user to follow. It is descriptive, not instructional. | **`non_conforming_descriptive`**. **Penalty applied.** |

**Example 1.5: The Contradiction Report Test**

- **Context:** A meeting summary says: "Per the marketing lead, the campaign will launch on May 1st. The engineering lead later noted that due to a technical dependency, the earliest possible launch date is May 7th."
- **Ground Truth:**
  - `question`: "When will the campaign launch?"
  - `answer_type`: `contradiction_report`

| Scenario               | Generated Answer                                                                                                                                 | LLM Judge's Analysis                                                                                                                        | Result                                                                   |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :----------------------------------------------------------------------- |
| **A. The Good Answer** | "The source contains conflicting information. Marketing states a launch date of May 1st, while engineering states the earliest date is May 7th." | The answer explicitly states that the source contains conflicting information, which is the correct structure for a `contradiction_report`. | **`conforms`**.                                                          |
| **B. The Bad Answer**  | "The campaign will launch on May 1st."                                                                                                           | The answer picks one side of the conflict without acknowledging the other. It fails to report the contradiction present in the source.      | **`non_conforming_fails_to_report_contradiction`**. **Penalty applied.** |

**Example 1.6: The No Information Test**

- **Context:** A project brief states: "The 'Apollo' project has been approved with a budget of $500,000. The project manager is Sarah Jenkins."
- **Ground Truth:**
  - `question`: "What is the timeline for the 'Apollo' project?"
  - `answer_type`: `no_information`

| Scenario               | Generated Answer                                                                 | LLM Judge's Analysis                                                                                                  | Result                                                  |
| :--------------------- | :------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------ |
| **A. The Good Answer** | "The provided document does not contain information about the project timeline." | The answer correctly states that the requested information is not available, conforming to the `no_information` type. | **`conforms`**.                                         |
| **B. The Bad Answer**  | "The 'Apollo' project is scheduled to be completed in six months."               | The answer invents a timeline that is not present in the source. This is a hallucination.                             | **`non_conforming_hallucinates`**. **Penalty applied.** |
