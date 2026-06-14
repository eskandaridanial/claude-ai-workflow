# Role

`prd` generates a Product Requirements Document from the requirements extracted by `grill`. It takes the structured requirements from `grill` and produces a detailed PRD that `issues` then decomposes into tasks.

`prd` is the second stage in the `grill` → `prd` → `issues` → `ralph` pipeline.

---

## Input Contract

`prd` receives:
- The requirements file (`.seed/{number}/requirements.md`) produced by `grill`
- The seed init.md file (`.seed/{number}/init.md`) for additional context

**Expected input state:**
- `requirements.md` exists and contains a valid JSON object with the schema defined in `grill`'s output contract
- The file is parseable and has all required fields

**Example of good input:**
```json
{
  "seed": "001",
  "stage": "grill",
  "problem_statement": "Users need secure authentication",
  "solution": "Add OAuth2 with Google and GitHub",
  "requirements": [...],
  "user_stories": [...],
  "assumptions": [],
  "open_questions": []
}
```

---

## Instructions

### 1. Resolve the seed number

Resolve the seed number:
1. If a prior `grill` session has already established the seed number, use it.
2. Otherwise, list `.seed/` and find the directory that matches the current story.
3. If still unknown, ask the user: _"Which seed number should I use? (e.g. 001)"_

All output goes to `.seed/{number}/prd.md`.

### 2. Read the requirements file

Read `.seed/{number}/requirements.md` to understand what `grill` extracted.

Also read `.seed/{number}/init.md` for additional context.

### 3. Explore the codebase (optional)

If a question can be answered by exploring the codebase, explore it instead of asking the user.

### 4. Generate the PRD

Using the requirements and your understanding of the codebase, generate a comprehensive PRD using the template below.

### 5. Write the PRD

Write the PRD to `.seed/{number}/prd.md`. Create the directory if it doesn't exist.

---

## Output Contract

`prd` writes to `.seed/{number}/prd.md`.

**Output schema (JSON in fenced code block):**
```json
{
  "seed": "001",
  "stage": "prd",
  "problem_statement": "The problem from the user's perspective",
  "solution": "The solution from the user's perspective",
  "user_stories": [
    {
      "id": "US-001",
      "as_an": "actor",
      "i_want": "feature",
      "so_that": "benefit"
    }
  ],
  "implementation_decisions": [
    {
      "id": "ID-001",
      "decision": "The decision made",
      "rationale": "Why this decision was made",
      "tradeoffs": "What was traded off"
    }
  ],
  "testing_decisions": [
    {
      "module": "The module to test",
      "what_to_test": "What makes a good test",
      "prior_art": "Similar tests in the codebase"
    }
  ],
  "out_of_scope": ["List of things explicitly not included"],
  "further_notes": "Any additional notes"
}
```

---

## Failure Behavior

### Missing input

**Trigger:** `requirements.md` does not exist.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "prd",
  "summary": "Missing requirements.md from grill",
  "warnings": [],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe what was missing, halt. Do not proceed.

### Malformed input

**Trigger:** `requirements.md` exists but is not valid JSON, or is missing required fields (`problem_statement`, `requirements`, `user_stories`).

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "prd",
  "summary": "Malformed requirements.md",
  "warnings": ["Missing required field: user_stories"],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe the parse error or missing fields, halt. Do not proceed.

### Incomplete input

**Trigger:** `requirements.md` is valid JSON but has low confidence (below 0.4) or has significant `open_questions` that block PRD generation.

**Handling:**
```json
{
  "status": "PARTIAL",
  "stage": "prd",
  "summary": "Proceeding with incomplete requirements",
  "warnings": [
    "Confidence is 0.3 - proceeding with caution",
    "Open question: How should we handle password reset?"
  ],
  "confidence": 0.3
}
```
Emit `status: PARTIAL` with warnings. Proceed with PRD generation, noting the gaps. If gaps are blocking, halt and surface them to the user.

---

## Status Block

After completing the PRD, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "prd",
  "summary": "One-line description of what was accomplished",
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```
