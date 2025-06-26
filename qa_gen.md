### **Final Consolidated Plan**

#### **1. Core Objective**

To create a high-quality, structured, ground-truth dataset of Question-Answer pairs from a 200-page PDF. This dataset will serve as a robust benchmark to evaluate the capabilities of other AI systems in the following areas:

- **Factual Recall:** Retrieving correct and relevant information.
- **Information Synthesis:** Integrating multiple pieces of information.
- **Logical Reasoning:** Performing calculations or deductions.
- **Nuance Comprehension:** Handling ambiguity, contradictions, and attributed opinions.

#### **2. Finalized Output Structure**

For each generated item, we will provide the following eight fields:

- **`question`**: The question posed to the system.
- **`answer`**: The ideal, human-crafted answer, which is comprehensive and well-grounded.
- **`atomic_facts`**: A list of the smallest, verifiable units of information from the source, sufficient to reconstruct the answer. (Detailed definition in Section 3).
- **`reasoning_steps`**: The logical steps or calculations required to bridge the `atomic_facts` to the `answer`.
- **`opinions_from_answer`**: Interpretive judgments or conclusions present in our `answer` that are not explicitly stated in the source.
- **`source_type`**: The category of the content from which the information was primarily drawn, defined as follows:
  - **`text`**: Standard paragraph text.
  - **`table`**: Information extracted from a table structure. We treat the table's **title, footnotes, and the data within it as a single, holistic unit.**
  - **`figure`**: Information from a graphical figure (like a diagram, flowchart, or schematic). The **caption and any embedded text are considered part of the figure.**
  - **`chart`**: Information from a data visualization (like a bar chart, pie chart, or line graph). Its **title, legend, and captions are all part of the chart.**
- **`relative_page`**: The page number(s) within the current chunk where the source information is located.
- **`chunk_id`**: Absolute identifier for the chunk.

#### **3. Detailed Definition of `atomic_facts`**

An `atomic_fact` is a discrete, verifiable statement from the source document, falling into one of these six categories:

1.  **Quantitative & Qualitative Data**
2.  **Declarative Statements of State or Event**
3.  **Causal and Relational Statements**
4.  **Prescriptive Statements (Rules, Policies, Requirements)**
5.  **Definitions and Terminology**
6.  **Attributed Statements (Opinions, Beliefs, Quotes)**

#### **4. Question Difficulty Levels**

- **Level 1 - Direct Factual Lookup:** Direct search-and-locate from a single, explicit location.
- **Level 2 - Contained Synthesis & Calculation:** Requires integration of information from several sentences or paragraphs within a single section.
- **Level 3 - Holistic Deduction:** Requires holistic understanding, abstraction, and reasoning across multiple sections or the entire document.

#### **5. Guiding Principles & Edge Case Handling**

- **Calculations:** For answers requiring calculation, `atomic_facts` will contain the raw inputs, and `reasoning_steps` will detail the logic.
- **Contradictions:** If the document presents conflicting data, the `answer` will explicitly state the contradiction, and the `atomic_facts` will list both conflicting data points.
- **Information in Visuals:** Information from tables, charts, and figures will be treated as a first-class source, adhering to the holistic definitions specified in Section 2 under `source_type`.
- **The Reconstruction Principle:** The list of `atomic_facts` must be sufficient for a third party to independently reconstruct and validate the `answer` without reading the source document.
