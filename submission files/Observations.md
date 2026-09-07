# Agent Evaluation Observations & Analysis

## Overview & Performance Summary

 Across five evaluation runs (see screenshot of Bedrock Evaluations page), overall system accuracy improved significantly from **65%** in Iteration 1 to **95%** in Iteration 2 through prompt refinements and clearer test definition.

| Iteration | Accuracy | Key Changes / Focus Areas |
| :--- | :--- | :--- |
| **Iteration 1** | **65%** | Initial system prompt with basic category definitions and single-turn evaluation prompts. |
| **Iteration 2** | **95%** | Refined prompt boundary definitions, explicit edge-case mapping, and disambiguated evaluation test cases. |

---

## Key Observations

### 1. Intent Routing Is Highly Reliable
Across all test runs, **100% of user requests were eventually routed correctly** into one of the three primary paths:
* **Bug Reports**
* **Platform Questions (FAQ)**
* **Other Requests / Human Handoff**

Explicitly listing ambiguous edge cases in the system prompt eliminated initial category misclassifications.

---

### 2. Limitations of Pure Prose Prompting (Non-Determinism)
While intent routing succeeded, controlling the precise multi-step runtime behavior of an LLM purely through natural language instructions (*prose*) demonstrated inherent non-determinism:

* **Instruction Adherence Variance:** Natural language prompts instruct the model to gather missing fields *one at a time* (e.g., asking for `environment` before `stepsToReproduce`). However, stochastic decoding occasionally led the model to skip directly to filing attempts.

---

### 3. State Management Needs During Data Collection
The contrast between Iteration 1 (65%) and Iteration 2 (95%) highlighted a critical architectural insight regarding multi-turn data gathering (e.g., bug reporting):

* **In-Context Memory vs. Structured State:** Relying entirely on LLM conversation history to track parameter presence (`description`, `stepsToReproduce`, `environment`) introduces failure modes when user inputs are ambiguous or non-standard.
* **The Case for External State Control:** To achieve 100% reliable multi-turn execution, state management (e.g., a formal state machine or structured slot-filling schema in the agent framework) is far superior to relying on natural language prompts alone to evaluate field completion.

