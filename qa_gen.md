### **Gold-Standard QA Dataset: Final Project Plan (Version 3.0)**

#### **1. Vision & Core Objective**

Our objective is to engineer a gold-standard, ground-truth dataset of Question-Answer pairs derived from a 200-page source document. This dataset is not merely a collection of facts; it is a sophisticated diagnostic tool designed to rigorously benchmark and evaluate the capabilities of advanced AI systems.

The evaluation will focus on the following core competencies:

- **Precision Recall:** The ability to retrieve specific, correct, and relevant information.
- **Multi-Hop Synthesis:** The ability to connect and integrate information from multiple locations or formats (e.g., text and a table).
- **Analytical Reasoning:** The ability to perform logical deductions, calculations, and inferences based on the provided context.
- **Nuance & Ambiguity Comprehension:** The ability to identify and correctly handle contradictions, attributed opinions, and subtleties within the source text.
- **Intent-Driven Formatting:** The ability to understand the user's intent and structure the answer in the most appropriate format (e.g., list, summary, direct answer).

---

#### **2. The Ground-Truth Data Schema**

Each item in the dataset will be a JSON object conforming to the following nine-field schema. This structure ensures that every question-answer pair is transparent, verifiable, and rich with metadata for analysis.

**2.1. `question`**

- **Definition:** The natural language question posed to the AI system.
- **Purpose:** Simulates a real-world user query.

**2.2. `answer`**

- **Definition:** The ideal, human-crafted answer. It is comprehensive, well-written, and directly addresses the `question`.
- **Purpose:** Serves as the "gold standard" against which the AI's generated response is compared.

**2.3. `answer_type`**

- **Definition:** A classification of the expected answer format, reflecting the primary intent of the question.
- **Purpose:** Allows for evaluating an AI's ability to structure its response correctly.
- **Supported Types:**
  - **`definitive`**: An answer that provides a single, conclusive resolution to the question. The answer itself may be a simple fact (a number, a name, a date) or a complex, multi-sentence conclusion derived from extensive reasoning. The key is that it resolves the question with a final, unambiguous verdict, rather than providing an open-ended summary or a list of items.
  - `list`: A set of distinct items, formatted with bullets or numbers.
  - `summary`: A condensed, prose-based overview of a broad topic.
  - `instructional`: A sequence of actionable steps or a procedure.
  - `contradiction_report`: An explicit statement that the source contains conflicting information, presenting both sides.
  - `no_information`: A clear statement that the information is not available in the source document.

**2.4. `atomic_facts`**

- **Definition:** A list of the smallest, discrete, and verifiable units of information extracted directly from the source document. The collection of these facts must be sufficient to independently reconstruct the `answer`.
- **Purpose:** Provides a transparent, verifiable chain of evidence from the source to the answer, enabling programmatic validation.
- **Categories of Atomic Facts:**
  1.  **Quantitative & Qualitative Data** (e.g., "Revenue was $10M," "The casing is red.")
  2.  **Declarative Statements of State or Event** (e.g., "The project was completed in Q2.")
  3.  **Causal and Relational Statements** (e.g., "Increased marketing led to a 15% rise in sales.")
  4.  **Prescriptive Statements (Rules, Policies)** (e.g., "All reports must be submitted by Friday.")
  5.  **Definitions and Terminology** (e.g., "'High-Risk' is defined as any project over $1M.")
  6.  **Attributed Statements (Opinions, Quotes)** (e.g., "The CEO stated, 'The future is bright.'")

**2.5. `reasoning_steps`**

- **Definition:** An explicit, human-written description of the logical steps, calculations, or inferences required to get from the `atomic_facts` to the final `answer`. This field is null if no reasoning is needed.
- **Purpose:** Makes the thought process transparent and allows for evaluation of an AI's reasoning capabilities, separate from its recall ability.

**2.6. `opinions_from_answer`**

- **Definition:** A list of any interpretive judgments, conclusions, or syntheses present in the `answer` that are not explicitly stated as a single fact in the source. This often overlaps with `reasoning_steps` but focuses on the interpretive leap.
- **Purpose:** Isolates subjective or synthesized elements of the answer for nuanced evaluation.

**2.7. `source_type`**

- **Definition:** The primary category of the content from which the `atomic_facts` were drawn.
- **Purpose:** Tests the AI's ability to extract information from diverse content formats.
- **Supported Types:**
  - `text`: Standard paragraph text.
  - `table`: A data table. The title, headers, data cells, and footnotes are considered a single holistic unit.
  - `figure`: A graphical figure like a diagram, flowchart, or schematic. The caption and any embedded text are part of the figure.
  - `chart`: A data visualization like a bar chart or line graph. The title, legend, axes, and captions are part of the chart.

**2.8. `relative_page` & `chunk_id`**

- **Definition:** `relative_page` is the page number(s) where the source information is located. `chunk_id` is the absolute identifier for the specific data chunk provided to the system.
- **Purpose:** Provides precise source location for traceability and debugging.

---

#### **3. Methodology & Generation Strategy**

**3.1. The Reconstruction Principle**
This is the central guiding principle of our dataset creation: **The `atomic_facts` and `reasoning_steps` must contain all the necessary information for a third party to independently reconstruct the `answer` without access to the original document.** This principle guarantees the self-contained verifiability of each data item.

**3.2. Tiered Question Complexity**
Questions will be intentionally crafted across three tiers of difficulty to test a range of AI capabilities:

- **Tier 1: Pinpoint Retrieval.** Questions that can be answered from a single, explicit location (a sentence, a table row, a data point on a chart). These test foundational factual recall.
- **Tier 2: Contained Synthesis.** Questions requiring the integration of multiple facts from within a single section or from a single complex visual (e.g., combining two rows in a table, summarizing a paragraph). These test basic synthesis and reasoning.
- **Tier 3: Cross-Domain Abstraction.** Questions that demand a holistic understanding of the document, requiring the AI to connect information across disparate sections, infer relationships, or identify overarching themes. These test advanced reasoning and abstraction.

---

#### **4. Guiding Protocols & Edge Case Handling**

To ensure consistency and quality, annotators will adhere to the following protocols:

- **Calculations:** For answers requiring math, the `atomic_facts` will contain the raw numerical inputs (e.g., ["Fact 1: Project A cost $5M", "Fact 2: Project B cost $3M"]). The `reasoning_steps` will detail the operation (e.g., ["Step 1: Sum the cost of Project A and Project B ($5M + $3M) to find the total."]).
- **Contradictions:** When the source presents conflicting information, a question will be specifically designed to expose it. The `answer_type` will be `contradiction_report`, the `answer` will clearly state "The document provides conflicting information...", and the `atomic_facts` will list both conflicting data points.
- **"No Information" Scenarios:** Questions will be intentionally created about plausible but absent topics. For these, the `answer_type` is `no_information`, the `answer` is a definitive statement like "The document does not contain information on this topic," and the `atomic_facts` and `reasoning_steps` fields will be empty. This directly tests for grounding and prevention of hallucination.
- **Determining `answer_type`:** If a question could fit multiple types, the classification will be based on the _primary user intent_. For example, "What are the three main risk factors?" has a primary intent of `list`. "Based on the criteria in the report, was the project a success?" has a primary intent of `definitive`, even if the explanation is long.
