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

## Pipeline Visibility: `/where`

**What it does:** Shows the current state of the pipeline across all seeds or for a specific seed.

**Invoke:**

| Command | Description |
|---------|-------------|
| `/where` | Shows one-line summary of all seeds |
| `/where 001` | Shows detailed tree view for seed 001 |

**`/where` output example:**
```
Seed 001: grill=DONE prd=DONE issues=5tasks(3AFK/2HITL) ralph=2/5 | Seed 002: grill=IN_PROGRESS
```

**`/where 001` output example:**
```
Seed 001: AI Workflow Pipeline
├── grill: ✓ DONE (completed 2026-06-14 10:30:00Z)
├── prd:   ✓ DONE (completed 2026-06-14 10:45:00Z)
├── issues: ✓ DONE (5 tasks: 3 AFK, 2 HITL, completed 2026-06-14 11:00:00Z)
└── ralph: ⟳ IN PROGRESS
    ├── 001-setup: ✓ DONE (completed 2026-06-14 11:15:00Z)
    ├── 002-auth:  ⟳ IN PROGRESS (ralph running)
    ├── 003-api:   ○ PENDING (blocked by 002)
    ├── 004-ui:    ○ PENDING (blocked by 002)
    └── 005-docs:  ○ PENDING (AFK)

## Upcoming work
1. /ralph 001 003 — Task 003: Add API endpoints (AFK, blocked by 002)
2. /ralph 001 005 — Task 005: Add documentation (AFK, unblocked)
```

**What `/where` shows:**
- Stage status icons: ✓ DONE, ⟳ IN_PROGRESS, ○ PENDING
- Timestamps in ISO 8601 format (from `state.json`)
- Task breakdown with AFK/HITL classification
- Blocked task indicators
- Confidence scores for each stage
- Confidence trend with actionable hints when confidence drops
- Upcoming work (next 3 runnable tasks)

**Confidence trend:**
When confidence drops between stages, `/where` shows:
```
├── prd:   ✓ DONE (confidence: 0.65, completed 2026-06-14 10:45:00Z)
│   ⚠ Low confidence — open question: how should we handle password reset?
│   💡 Run `/grill --focus auth` to clarify auth requirements
```

**Actionable hint definition:**
A hint is actionable if it:
1. Includes a specific command to run
2. Directly resolves the open question that caused the confidence drop
3. References the specific skill/stage that needs attention

---

## Pipeline Auto-Run: `/pipeline`

**What it does:** Manages pipeline execution with auto-run capabilities.

**Invoke:**

| Command | Description |
|---------|-------------|
| `/pipeline run --next` | Run the next unblocked AFK task (asks for confirmation) |
| `/pipeline run --next --force` | Run the next unblocked AFK task (no confirmation) |
| `/pipeline run --all` | Run all remaining AFK tasks in order |

**`/pipeline run --next` behavior:**
1. Reads fresh state from disk
2. Identifies the next unblocked AFK task
3. Shows confirmation: `Ready to run: /ralph 002 005 — Task 005: pipeline skill (run --next). Proceed? [y/N]`
4. On confirm: invokes ralph for that task
5. On reject: shows "Cancelled" and exits
6. Updates `state.json` after completion

**`/pipeline run --all` behavior:**
1. Identifies ALL remaining AFK tasks (not just the next one)
2. Runs them in task number order
3. Shows progress: `Running 001... 002... 003...`
4. Stops at HITL tasks (surfaces decision point for human input)
5. Stops on failure (marks task as FAILED, does not continue)
6. Updates `state.json` after each task

---

## Edge Case Handling

The pipeline handles edge cases gracefully:

### Blocked Task Refusal
```
Task 003 blocked by 002. Complete 002 first.
Run `/ralph 002 002` to complete the blocker.
```

### HITL Task Halting
```
Task 003 is HITL: requires design review. What is your decision?
[Pipeline halts and waits for human input]
```

### Failure Recovery
```
Task 003 FAILED: compile error. Fix the error and run `/ralph 002 003` to retry.
```
- Failed task is marked as FAILED in `state.json`
- Next run resumes from the failed task (never skips ahead)
- Never auto-rollback; always resume forward

### Manual Intervention Detection
- Pipeline always reads fresh state from disk before every action
- If you manually run a skill while pipeline is tracking, it detects the state change
- Never assumes cached state is correct

---

## Understanding Status Blocks

Every skill emits a Status Block at the end of its output:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "grill | prd | issues | ralph | where | pipeline",
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
|------------|---------|
| `≥ 0.7` | `ralph` proceeds autonomously |
| `0.4 – 0.7` | `ralph` proceeds but surfaces warnings |
| `< 0.4` | `ralph` halts and asks for guidance |

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

### Check pipeline status

```
You: /where
```

### See detailed status of a seed

```
You: /where 001
```

### Auto-run next task with confirmation

```
You: /pipeline run --next
```

### Auto-run without confirmation

```
You: /pipeline run --next --force
```

### Run all remaining AFK tasks

```
You: /pipeline run --all
```

### Resume after a failure

```
You: /ralph 001 001
     ...
     Status: FAILED, "Blocker task is incomplete"
```

Check the blocker:
```
You: cat .seed/001/tasks/001-setup.md
```

Fix it or complete it, then retry:
```
You: /ralph 001 001
```

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
| `.seed/{number}/state.json` | Pipeline state (timestamps, confidence history, task status) |
| `.claude/contract/pipeline.md` | Full pipeline contract (technical reference) |
| `.claude/contract/usage.md` | This document |

---

## Skill Locations

| Skill | Location |
|-------|----------|
| `grill` | `.claude/skills/grill/skill.md` |
| `prd` | `.claude/skills/prd/skill.md` |
| `issues` | `.claude/skills/issues/skill.md` |
| `ralph` | `.claude/skills/ralph/skill.md` |
| `tdd` | `.claude/skills/tdd/skill.md` |
| `where` | `.claude/skills/where/skill.md` |
| `pipeline` | `.claude/skills/pipeline/skill.md` |

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

### "No tasks to run"

All AFK tasks are complete or blocked. Use `/where 001` to see the current state.

### "Task blocked by NNN"

The task you tried to run is blocked by an incomplete task. Complete the blocking task first.
