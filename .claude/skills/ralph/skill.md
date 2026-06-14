# Role

`ralph` picks up a task from `.seed/{number}/tasks/` and implements it. It handles AFK (autonomous) tasks without human interaction and pauses for HITL (Human-In-The-Loop) tasks when a human decision is required.

`ralph` is the fourth and final stage in the `grill` → `prd` → `issues` → `ralph` pipeline.

---

## Input Contract

`ralph` receives:
- A task file (`.seed/{number}/tasks/NNN-*.md`) produced by `issues`
- The parent PRD (`.seed/{number}/prd.md`) for context
- The requirements file (`.seed/{number}/requirements.md`) for additional context
- The seed init.md (`.seed/{number}/init.md`) for original intent

**Expected input state:**
- The task file exists and is readable
- All blocker tasks (if any) have their acceptance criteria checked off

**Example of good input:**
A task file with:
- Parent PRD reference
- What to build (concise description)
- Acceptance criteria (numbered list)
- Blocked by (empty or references to completed task files)
- User stories addressed

---

## Instructions

### 1. Resolve the seed number and task

Resolve the seed number:
1. If a prior `issues` session has already established the seed number, use it.
2. Otherwise, list `.seed/` and find the directory that matches the current story.
3. If still unknown, ask the user: _"Which seed and task should I implement? (e.g. seed 001, task 002)"_

Read the task file from `.seed/{number}/tasks/NNN-*.md`.

### 2. Check blockers

Read the "Blocked by" field in the task file.

- If blockers are listed, check each referenced task file and verify all its acceptance criteria are checked off (`[x]`).
- If any blocker is incomplete, **stop**. Tell the user which task is blocking and why. Do not proceed.
- If there are no blockers, or all blockers are complete, continue.

### 3. Read supporting context

Before touching any code, read:
- `.seed/{number}/init.md` — original story intent
- `.seed/{number}/prd.md` — full requirements and implementation decisions
- `.seed/{number}/requirements.md` — requirements from `grill`
- The task file itself — acceptance criteria, user stories addressed

Explore the relevant parts of the codebase to understand the current state. Do not assume — verify.

### 4. Check confidence threshold

If the upstream `issues` stage emitted a confidence value, apply the threshold bands:
- `confidence ≥ 0.7` — proceed autonomously
- `0.4 ≤ confidence < 0.7` — proceed with warnings surfaced to user
- `confidence < 0.4` — halt and surface the low confidence to user before proceeding

### 5. Branch on task type

#### AFK task

The task can be implemented and verified without human interaction.

1. Plan the implementation privately. Identify which files to create or modify.
2. **Invoke `tdd` via `Skill` tool** for every AFK implementation slice:
   - Pass `task_file`, `acceptance_criteria`, `language`, and `mode: "pipeline"` to `tdd`
   - `tdd` runs the red-green-refactor loop autonomously and returns structured output
   - Do not write all tests first — follow the tracer bullet approach
3. **Validate TDD output** before declaring SUCCESS:
   - `status` must be `SUCCESS`
   - `red_green_refactor_order` must be `true`
   - `tests_passed` must be `true`
   - `timestamps.test_written_at` must exist and be non-null
   - If any validation fails, emit `status: FAILED` and halt
4. **Reject green-red anti-pattern**: if `tdd` returns `red_green_refactor_order: false`, emit `status: FAILED` — tests were written after implementation, which is a contract violation
5. Implement the slice end-to-end — schema, logic, API, tests — as described in the task.
6. Run tests and confirm they pass.
7. Update the task file: check off each completed acceptance criterion (`- [x]`).
8. Report what was done in a short summary. No approval needed.

#### HITL task

The task requires a human decision before or during implementation.

1. Read the task and identify exactly what the decision point is. Common cases:
   - An architectural choice not resolved in the PRD
   - A design review or visual sign-off
   - An ambiguous requirement with multiple valid interpretations
   - An external dependency that needs a human to configure or approve
2. **For mixed AFK/HITL tasks**: If the task has some AFK acceptance criteria:
   - Run `tdd` via `Skill` tool for the AFK criteria first
   - Pause at the HITL boundary when `tdd` emits `hitl_needed: true`
   - Surface the decision to the user
   - After the decision, resume `tdd` for remaining AFK criteria
3. **Stop and surface the decision** clearly to the user:
   - What you know so far
   - What the specific decision is
   - The options you see, with trade-offs
   - Which option you recommend and why
4. Wait for the user's response.
5. Once the decision is made, record it in the task file under a `## Decisions` section.
6. Continue implementing as AFK from that point forward, following the same TDD loop.
7. Update acceptance criteria checkboxes when done.

### 6. Finish

After implementation:

- All acceptance criteria in the task file must be checked `[x]`.
- If any criterion could not be met, explain why and leave it unchecked with a note.
- Do not modify the parent PRD or other task files.
- If the implementation revealed scope that belongs in a new task, describe it briefly — do not create the task file yourself. Let the user decide whether to add it via `issues`.

---

## Output Contract

`ralph` updates the task file and produces implementation changes.

**Output schema (JSON in fenced code block):**
```json
{
  "seed": "001",
  "stage": "ralph",
  "task_id": "001",
  "filename": "001-add-oauth2-providers.md",
  "status": "completed | blocked | halted",
  "acceptance_criteria_completed": ["AC-001", "AC-002"],
  "acceptance_criteria_skipped": [],
  "decisions": [
    {
      "decision": "The decision made",
      "rationale": "Why this decision was made"
    }
  ],
  "new_scope": ["Any scope that belongs in a new task"]
}
```

---

## Failure Behavior

### Missing input

**Trigger:** The task file does not exist.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "ralph",
  "summary": "Missing task file",
  "warnings": [],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe what was missing, halt. Do not proceed.

### Malformed input

**Trigger:** The task file exists but is missing required fields (What to build, Acceptance criteria, Blocked by).

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "ralph",
  "summary": "Malformed task file",
  "warnings": ["Missing required field: What to build"],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe the parse error or missing fields, halt. Do not proceed.

### Incomplete input / blocked

**Trigger:** A task listed in "Blocked by" has incomplete acceptance criteria.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "ralph",
  "summary": "Blocker task is incomplete",
  "warnings": [".seed/001/tasks/000-setup.md has unchecked acceptance criteria"],
  "confidence": 0.0
}
```
Emit `status: FAILED`, identify which task is blocking and why, halt. Do not proceed.

### Low confidence threshold

**Trigger:** Upstream confidence is below 0.4.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "ralph",
  "summary": "Upstream confidence too low to proceed",
  "warnings": ["issues confidence was 0.3"],
  "confidence": 0.3
}
```
Emit `status: FAILED`, surface the low confidence to the user, halt. Do not proceed autonomously.

### Implementation failure

**Trigger:** Implementation encounters an error that cannot be resolved (missing dependency, conflicting requirement, etc.).

**Handling:**
```json
{
  "status": "PARTIAL",
  "stage": "ralph",
  "summary": "Implementation partially complete",
  "warnings": [
    "Could not complete AC-003:第三方API超时",
    "New scope discovered:需要重试机制"
  ],
  "confidence": 0.5
}
```
Emit `status: PARTIAL` with warnings. Update which acceptance criteria were completed vs. skipped. Surface to user for guidance.

---

## Status Block

After completing the implementation, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "ralph",
  "summary": "One-line description of what was accomplished",
  "tests_written": true,
  "red_green_refactor_order": true,
  "tdd_invoked": true,
  "tdd_test_count": 5,
  "tdd_test_files": ["path/to/test1.ts"],
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```

**TDD evidence fields:**
- `tests_written`: `true` if tests were written, `false` otherwise
- `red_green_refactor_order`: `true` if tests were written before implementation
- `tdd_invoked`: `true` if `tdd` was invoked via `Skill` tool
- `tdd_test_count`: number of tests written by `tdd`
- `tdd_test_files`: array of test file paths produced by `tdd`
