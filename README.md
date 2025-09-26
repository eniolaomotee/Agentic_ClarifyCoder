# Agentic_ClarifyCoder

Overview
--------

This codebase implements an **agentic flow** for evaluating problem statements and generating code solutions using LLMs (Gemini in this case).

The system follows a **multi-round reasoning process**:

1.  **Initial Attempt** → Generate candidate code directly from the problem statement.

2.  **Clarification Phase** → If ambiguities are detected, generate clarifying questions.

3.  **Answer Phase** → Answer the clarifying question using the original problem statement(an LLM evaluator answers this).

4.  **Refinement Phase** → Generate improved/final code using the clarification.

5.  **Testing Phase** → Evaluate generated code against test cases.

This loop simulates how a **human developer** would read, ask clarifying questions, and refine their implementation.

* * * * *

Components
----------

### 1\. `run_agentic_flow(row)`

-   **Purpose**: Orchestrates the full agentic reasoning loop for one dataset row (problem + test case).

-   **Input**: A Pandas row containing:

    -   `problem_statement`

    -   `test_case`

-   **Process**:

    -   Calls `round_1_generation` to generate initial code.

    -   Calls `round_2_implementation` to decide if clarification is needed.

    -   If clarification is required, invokes `clarifying_question_answer` to resolve ambiguities.

    -   Calls Gemini again to produce the refined implementation.

-   **Output**: `(initial_code, final_code, number_iterations, asked_follow_up_question, is_good_question)`.

* * * * *

### 2\. `clarifying_question_answer(gemini_model, original_question, clarifying_question, modified_question_category)`

-   **Purpose**: Evaluates whether a clarifying question is good, and answers it if possible.

-   **Input**:

    -   Gemini model instance.

    -   Original problem statement.

    -   Clarifying question.

    -   Category of missing information.

-   **Output**: Dict:

    ```
    {
      "is_good_question": true|false,
      "answer": "<direct answer or empty>"
    }
    ```

-   **Notes**:

    -   Enforces strict JSON output.

    -   Uses `parse_json_from_output` to handle cleanup.

* * * * *

### 3\. `parse_json_from_output(response_text)`

-   **Purpose**: Extracts JSON from model output.

-   **Input**: Raw text from Gemini.

-   **Output**: Python dict if valid JSON, otherwise defaults to `{"is_good_question": false, "answer": ""}`.

* * * * *

### 4\. `test_generated_code(code, test_case)`

-   **Purpose**: Runs generated Python code against provided test cases.

-   **Input**: Code string + test case description.

-   **Output**: Boolean pass/fail or structured test result.

-   **Notes**:

    -   Used to compute **pass@1** and **pass@final** metrics.

* * * * *

### 5\. Dataset Handling

-   **Input Dataset**: Contains problem statements, categories, and test cases.

-   **Iteration**: `new_df.iterrows()` loops through dataset rows, feeding them into `run_agentic_flow`.

-   **Outputs**:

    -   Initial code vs final code.

    -   Whether clarifying questions were asked.

    -   Test pass results.

-   **Metrics**:

    -   `pass@1` → correctness of first attempt.

    -   `pass@final` → correctness after clarifications.

* * * * *

### 6\. Error Handling

-   **Timeouts**: Colab secret retrieval (`userdata.get`) may fail; must handle `TimeoutException`.

-   **Malformed JSON**: Warnings printed when Gemini outputs non-JSON.

-   **Missing Arguments**: Functions must always be passed required args (e.g., `clarifying_question_answer` needed `modified_question_category`).

* * * * *

Example Flow
------------

1.  **Input Row**:

    ```
    {
      "problem_statement": "Check if in a list of numbers, any two numbers are closer than threshold.",
      "test_case": "has_close_elements([1.0, 2.8, 3.0], 0.3) == True"
    }
    ```

2.  **Round 1 (Initial)**:

    ```
    def has_close_elements(numbers, threshold):
        return any(abs(x-y) < threshold for x in numbers for y in numbers)
    ```

3.  **Round 2 (Clarification)**:

    -   Question: *"What is the definition of 'closer' in the context of the problem statement?"*

    -   System marks it as GOOD and answers: `"closer" means absolute difference < threshold`.

4.  **Round 3 (Refinement)**:

    -   Gemini regenerates code with clarified definition.

5.  **Testing**:

    -   Runs against provided test case(s).

    -   Logs results.

* * * * *

Usage
-----

```
import google.generativeai as genai
from google.colab import userdata

# Configure Gemini
genai.configure(api_key=userdata.get("GOOGLE_API_KEY"))
gemini = genai.GenerativeModel("gemini-2.5-pro")

# Run on dataset
for index, row in new_df.iterrows():
    results = run_agentic_flow(row)
    print(results)
```

* * * * *

