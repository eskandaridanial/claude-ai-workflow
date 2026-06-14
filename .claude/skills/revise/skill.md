---
name: revise
description: Pick up a task from .seed/{number}/tasks/ and implement it. Handles AFK tasks autonomously and pauses for human input on HITL tasks. Use when the user wants to execute a task slice produced by the break skill.
---

# Implement Task

Execute a single vertical slice from `.seed/{number}/tasks/NNN-*.md`, respecting its type (AFK or HITL) and its dependency chain.

---

## 1. Resolve the task

### If the user provided an explicit task path

If the user passed a path like `.seed/{number}/tasks/NNN-*.md` directly, use it. Skip auto-resolution entirely.

### Otherwise, auto-resolve

Resolve the seed number using the same priority chain as `prd` and `break`:
1. Inherit from a prior session (`align`, `prd`, or `break`) if already established.
2. Scan `.seed/` and infer from context.
3. Ask the user: _"Which seed should I work on? (e.g. 001)"_

Once the seed is known, **find the first incomplete task**:
- List all files in `.seed/{number}/tasks/` sorted numerically ascending (`001`, `002`, ...).
- Scan each task file in order and check whether all acceptance criteria are checked off (`[x]`).
- Pick the first task that has at least one unchecked criterion (`[ ]`).
- If all tasks are complete, tell the user and stop.

Running `implement` with no arguments always continues from where the last session left off.

---

## 2. Check blockers

Read the "Blocked by" field in the task file.

- If blockers are listed, check each referenced task file and verify all its acceptance criteria are checked off (`[x]`).
- If any blocker is incomplete, **stop**. Tell the user which task is blocking and why. Do not proceed.
- If there are no blockers, or all blockers are complete, continue.

---

## 3. Read supporting context

Before touching any code, read:
- `.seed/{number}/init.md` — original story intent
- `.seed/{number}/prd.md` — full requirements and implementation decisions
- The task file itself — acceptance criteria, user stories addressed

Explore the relevant parts of the codebase to understand the current state. Do not assume — verify.

---

## 4. Branch on task type

### AFK task

The task can be implemented and verified without human interaction.

1. Plan the implementation privately. Identify which files to create or modify.
2. Follow the `tdd` skill: write one test, make it pass, repeat. Do not write all tests first.
3. Implement the slice end-to-end — schema, logic, API, tests — as described in the task.
4. Run tests and confirm they pass.
5. Update the task file: check off each completed acceptance criterion (`- [x]`).
6. Report what was done in a short summary. No approval needed.

### HITL task

The task requires a human decision before or during implementation.

1. Read the task and identify exactly what the decision point is. Common cases:
   - An architectural choice not resolved in the PRD
   - A design review or visual sign-off
   - An ambiguous requirement with multiple valid interpretations
   - An external dependency that needs a human to configure or approve
2. **Stop and surface the decision** clearly to the user:
   - What you know so far
   - What the specific decision is
   - The options you see, with trade-offs
   - Which option you recommend and why
3. Wait for the user's response.
4. Once the decision is made, record it in the task file under a `## Decisions` section.
5. Continue implementing as AFK from that point forward, following the same TDD loop.
6. Update acceptance criteria checkboxes when done.

---

## 5. Finish

After implementation:

- All acceptance criteria in the task file must be checked `[x]`.
- If any criterion could not be met, explain why and leave it unchecked with a note.
- Do not modify the parent PRD or other task files.
- If the implementation revealed scope that belongs in a new task, describe it briefly — do not create the task file yourself. Let the user decide whether to add it via `break`.