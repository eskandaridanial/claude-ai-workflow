# Pipeline Usage Guide

A practical guide to running the `grill` → `prd` → `issues` → `ralph` pipeline.

---

## Quick Start

```
You: /grill @.seed/001/init.md
     /prd
     /issues
     /ralph 001 001
```

Each skill reads from and writes to `.seed/{number}/` files. No manual file creation needed.

---

## Stage 1: `grill`

**What it does:** Interviews you relentlessly until shared understanding is reached. Writes structured requirements.

**Invoke:**
```
/grill @.seed/001/init.md
```

**Input:** `.seed/{number}/init.md` (the seed story)

**Output:** `.seed/{number}/requirements.md`

**What to expect:**
- `grill` asks questions one at a time
- For each question, it gives a recommended answer
- You confirm, modify, or reject
- When done, `grill` writes `requirements.md`

**Status Block example:**
```json
{
  "status": "SUCCESS",
  "stage": "grill",
  "summary": "Extracted 12 requirements across 5 user stories",
  "warnings": [],
  "confidence": 0.85
}
```

---

## Stage 2: `prd`

**What it does:** Generates a Product Requirements Document from the requirements.

**Invoke:**
```
/prd
```

**Input:** `.seed/{number}/requirements.md` (from `grill`)

**Output:** `.seed/{number}/prd.md`

**What to expect:**
- `prd` reads the requirements
- It explores the codebase to understand the current state
- It generates a comprehensive PRD with user stories, implementation decisions, testing decisions
- You may be asked clarifying questions

**Status Block example:**
```json
{
  "status": "SUCCESS",
  "stage": "prd",
  "summary": "Generated PRD with 8 user stories and 5 implementation decisions",
  "warnings": ["Open question: How should we handle password reset?"],
  "confidence": 0.75
}
```

---

## Stage 3: `issues`

**What it does:** Breaks the PRD into independently-workable tasks (vertical slices).

**Invoke:**
```
/issues
```

**Input:** `.seed/{number}/prd.md` (from `prd`)

**Output:** `.seed/{number}/tasks/NNN-*.md` (one file per task)

**What to expect:**
- `issues` proposes a task breakdown
- Each task is a thin vertical slice (tracer bullet) through all layers
- Tasks are marked as `AFK` (autonomous) or `HITL` (human-in-the-loop)
- You quiz and iterate until you approve the breakdown
- `issues` creates the task files

**Task types:**
- **AFK**: Can be implemented and merged without human interaction
- **HITL**: Requires human decision — architectural choice, design review, etc.

**Status Block example:**
```json
{
  "status": "SUCCESS",
  "stage": "issues",
  "summary": "Created 5 tasks: 3 AFK, 2 HITL",
  "warnings": [],
  "confidence": 0.9
}
```

---

## Stage 4: `ralph`

**What it does:** Implements a single task.

**Invoke:**
```
/ralph 001 001
```
Where `001` is the seed number and `001` is the task number.

**Input:** `.seed/{number}/tasks/NNN-*.md` (from `issues`)

**Output:** Updated task file with checked acceptance criteria + implementation

**What to expect for AFK tasks:**
- `ralph` plans the implementation
- Follows TDD loop: write one test, make it pass, repeat
- Implements end-to-end
- Updates the task file with checked acceptance criteria
- Reports what was done

**What to expect for HITL tasks:**
- `ralph` identifies the decision point
- Surfaces the decision with options and trade-offs
- Recommends an option
- Waits for your response
- Records the decision in the task file
- Continues as AFK

**Status Block example:**
```json
{
  "status": "SUCCESS",
  "stage": "ralph",
  "summary": "Implemented OAuth2 providers end-to-end",
  "warnings": [],
  "confidence": 0.95
}
```

---

## Understanding Status Blocks

Every skill emits a Status Block at the end of its output:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "grill | prd | issues | ralph",
  "summary": "One-line description",
  "warnings": ["Any concerns"],
  "confidence": 0.0-1.0
}
```

### Status Values

| Status | Meaning | What to do |
|--------|---------|------------|
| `SUCCESS` | Completed, output is valid | Proceed to next stage |
| `PARTIAL` | Completed but with warnings | Review warnings, decide whether to proceed |
| `FAILED` | Halted, no useful output | Fix the issue before proceeding |

### Confidence Threshold

| Confidence | Behavior |
|------------|----------|
| `≥ 0.7` | `ralph` proceeds autonomously |
| `0.4 – 0.7` | `ralph` proceeds but surfaces warnings |
| `< 0.4` | `ralph` halts and asks for guidance |

### When You See a Warning

**Example warning from `prd`:**
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

You can:
1. Proceed anyway (the agent will do its best)
2. Go back and fill the gap (run `grill` again for that specific area)
3. Ask the agent to halt and surface the issue

### When You See a Failure

**Example failure from `issues`:**
```json
{
  "status": "FAILED",
  "stage": "issues",
  "summary": "Missing prd.md from prd",
  "warnings": [],
  "confidence": 0.0
}
```

This means `prd` hasn't been run yet, or the PRD file is missing. Run `/prd` first.

---

## Common Scenarios

### Start a new pipeline from scratch

```
You: /grill @.seed/001/init.md
     ... (answer questions) ...
     /prd
     ... (answer questions if any) ...
     /issues
     ... (review and approve task breakdown) ...
     /ralph 001 001
     ... (ralph implements) ...
     /ralph 001 002
     ... (continue with next task) ...
```

### Resume after a failure

```
You: /ralph 001 001
     ...
     Status: FAILED, "Blocker task is incomplete"
```

Check the blocker:
```
You: cat .seed/001/tasks/000-setup.md
```

Fix it or complete it, then retry:
```
You: /ralph 001 001
```

### Get architectural input mid-pipeline

```
You: /revise 001 003
```

`revise` is a standalone skill for architectural decisions. Use it when:
- A task is marked HITL and you need to make a decision
- You want to explore architectural options before breaking down tasks
- You need a design review

### Run TDD on a specific module

```
You: /tdd auth/oauth2
```

`tdd` is a standalone skill for test-driven development. Use it when:
- You want to add tests before implementing
- You need to fix a bug using TDD
- You want integration tests

---

## File Locations

| File | Purpose |
|------|---------|
| `.seed/{number}/init.md` | Original seed story |
| `.seed/{number}/requirements.md` | `grill` output |
| `.seed/{number}/prd.md` | `prd` output |
| `.seed/{number}/tasks/NNN-*.md` | `issues` output (task files) |
| `.claude/contract/pipeline.md` | Full pipeline contract (technical reference) |
| `.claude/contract/usage.md` | This document |

---

## Troubleshooting

### "Missing requirements.md"

`grill` hasn't been run yet. Run `/grill @.seed/{number}/init.md` first.

### "Missing prd.md"

`prd` hasn't been run yet. Run `/prd` first.

### "Blocker task is incomplete"

The task you're trying to run depends on another task that isn't done. Check the "Blocked by" field in the task file and complete that task first.

### "Upstream confidence too low"

The upstream stage had low confidence in its output. Review the warnings and decide whether to:
1. Proceed anyway
2. Go back and fill the gaps
3. Ask for clarification

### Task marked HITL

The task requires a human decision. `ralph` will surface the decision point and wait for your input before proceeding.
