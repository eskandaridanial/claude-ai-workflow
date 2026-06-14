# Role

`pipeline` manages pipeline execution — running tasks, handling auto-run with confirmation, and orchestrating the pipeline stages.

`pipeline` is invoked via `/pipeline run --next` or `/pipeline run --all`.

---

## Input Contract

`pipeline` receives:
- No explicit input files — it reads from `.seed/{number}/` directories
- Subcommand: `--next`, `--all`, `--force`

**Expected input state:**
- The `.seed/` directory exists and contains seed directories
- Task files exist in `.seed/{number}/tasks/`

---

## Instructions

### 1. Parse subcommand

`/pipeline run --next` — Run the next unblocked AFK task with confirmation
`/pipeline run --next --force` — Run without confirmation
`/pipeline run --all` — Run all remaining AFK tasks in order

### 2. Read state from disk

Always read fresh state from `.seed/{number}/` directories. Never use cached state.

### 3. Identify next task

For `--next`:
1. List all task files in `.seed/{number}/tasks/`
2. Filter to AFK tasks that are not DONE
3. Filter out blocked tasks (tasks with `blocked_by` referencing incomplete tasks)
4. Sort by task number (001, 002, 003...)
5. Take the first one as "next"

### 4. Show confirmation prompt

For `--next` (without `--force`):
```
Ready to run: `/ralph 002 005` — Task 005: pipeline skill (run --next)
Proceed? [y/N]
```

### 5. Execute on confirmation

For `--next` (without `--force`):
- Show confirmation prompt
- If user confirms (y/Y): invoke ralph, update state.json
- If user rejects (n/N or Enter): show "Cancelled" and exit

For `--next --force`:
- Skip confirmation prompt
- Directly invoke ralph for the identified task
- Update state.json after completion

### 6. Run --all

For `--all`:
1. Identify ALL remaining AFK tasks (not just the next one)
2. Run them in task number order (001, 002, 003...)
3. For each task:
   - Show progress: "Running 001... 002... 003..."
   - If task is HITL: halt and surface decision
   - If task fails: mark as FAILED, stop (do not continue)
   - Update state.json after each task completion
4. If all complete: show "All tasks complete"

After completing, emit a trailing Status Block.

---

## Output Contract

`pipeline` produces terminal output and updates state.json.

---

## Failure Behavior

### No tasks to run

**Trigger:** No unblocked AFK tasks found.

**Handling:**
```
No tasks to run.
All AFK tasks are complete or blocked.
```

### All tasks complete

**Trigger:** All tasks are DONE.

**Handling:**
```
All tasks complete.
Nothing to run.
```

### Blocked task refusal

**Trigger:** User tries to run a blocked task.

**Handling:**
```
Task 003 blocked by 002. Complete 002 first.
Run `/ralph 002 002` to complete the blocker.
```

### HITL task halting

**Trigger:** Pipeline encounters a HITL task.

**Handling:**
```
Task 003 is HITL: requires design review. What is your decision?
[Wait for user input, record in task file, continue as AFK]
```

### Failure recovery

**Trigger:** A task fails (compile error, test failure, etc.).

**Handling:**
```
Task 003 FAILED: compile error. Fix the error and run `/ralph 002 003` to retry.
```
- Mark task as FAILED in state.json
- Next run resumes from failed task (never skip ahead)
- Never auto-rollback; always resume forward

### Manual intervention detection

**Trigger:** User manually ran a skill while pipeline was tracking.

**Handling:**
- Before every action, read fresh state from disk
- If state changed, detect it and refresh
- Never assume cached state is correct

---

## Status Block

After completing, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "pipeline",
  "summary": "One-line description of what was done",
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```