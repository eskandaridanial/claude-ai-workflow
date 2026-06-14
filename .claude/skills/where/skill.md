# Role

`where` shows the current state of the pipeline across all seeds or for a specific seed. It provides aggregate visibility into what stage each seed is at, task completion status, and overall pipeline health.

`where` is a pipeline-adjacent skill that improves ease of use and visibility. It does not produce artifacts that feed into other pipeline stages.

---

## Input Contract

`where` receives:
- No explicit input files — it reads from `.seed/{number}/` directories
- Optional: a seed number argument (e.g., `001`) to show detailed view

**Expected input state:**
- The `.seed/` directory exists and contains seed directories
- Each seed directory contains stage output files (requirements.md, prd.md, tasks/*.md)

**How it determines state:**
- `grill` done: `.seed/{number}/requirements.md` exists
- `prd` done: `.seed/{number}/prd.md` exists
- `issues` done: `.seed/{number}/tasks/` directory exists with task files
- `ralph` in progress: task files exist with mixed DONE/IN_PROGRESS/PENDING states
- Task status: read from task files — `[x]` = DONE, no checkbox or `[ ]` = PENDING/IN_PROGRESS

**Timestamps (state.json):**
- Read timestamps from `.seed/{number}/state.json` if it exists
- If a stage is complete but not in state.json, add it with current timestamp
- If a task is complete but not in state.json, add it with current timestamp
- Timestamps in ISO 8601 format: `YYYY-MM-DDTHH:mm:ssZ`

---

## Instructions

### 1. Resolve the seed number

If the user invoked `/where 001` (with a seed number), use that seed.
If the user invoked `/where` (no args), show all seeds.

### 2. Read seed directories

List `.seed/` and read each seed directory to determine stage completion.

For each seed:
1. Check if `.seed/{number}/requirements.md` exists → grill done
2. Check if `.seed/{number}/prd.md` exists → prd done
3. Check if `.seed/{number}/tasks/` exists → issues done
4. Read task files in `.seed/{number}/tasks/` to determine ralph progress:
   - Count total tasks
   - Count DONE tasks (files with `[x]` checked acceptance criteria)
   - Count AFK vs HITL tasks (from task frontmatter or content)
   - Identify blocked tasks (tasks with `blocked_by` referencing incomplete tasks)

### 3. Create or update state.json

**This step runs when a specific seed number is provided (e.g., `/where 001`).**

After determining stage completion, create or update `.seed/{number}/state.json`:

1. **If state.json does not exist:** Create it with current timestamps for all completed stages
2. **If state.json exists:** Compare against current state and add missing timestamps for newly completed stages
3. **Write the updated state.json** back to the seed directory

**state.json schema:**
```json
{
  "seed": "001",
  "stages": {
    "grill": { "completed": "2026-06-14T10:30:00Z" },
    "prd": { "completed": "2026-06-14T10:45:00Z" },
    "issues": { "completed": "2026-06-14T11:00:00Z" }
  },
  "tasks": {
    "001-setup": { "completed": "2026-06-14T11:15:00Z" },
    "002-auth": { "completed": "2026-06-14T11:20:00Z" }
  }
}
```

**Rules:**
- Only add timestamps for stages/tasks that are DONE
- Do not overwrite existing timestamps
- Use ISO 8601 format: `YYYY-MM-DDTHH:mm:ssZ`
- Use current time when creating new timestamps

### 4. Format output

**`/where` (all seeds summary):**
```
Seed 001: grill=DONE prd=DONE issues=5tasks(3AFK/2HITL) ralph=2/5 | Seed 002: grill=IN_PROGRESS
```

Format per seed: `Seed {NNN}: {stage}={STATUS} {stage}={STATUS} ...`

Stage statuses:
- `grill`: DONE | IN_PROGRESS | PENDING
- `prd`: DONE | IN_PROGRESS | PENDING
- `issues`: DONE | IN_PROGRESS | PENDING
- `ralph`: {done}/{total}tasks({AFK}AFK/{HITL}HITL) | IN_PROGRESS | PENDING

**`/where 001` (detailed tree view):**
```
Seed 001: AI Workflow Pipeline
├── grill: ✓ DONE (confidence: 0.85, completed 2026-06-14 10:30)
├── prd:   ✓ DONE (confidence: 0.75, completed 2026-06-14 10:45)
├── issues: ✓ DONE (5 tasks: 3 AFK, 2 HITL, completed 2026-06-14 11:00)
└── ralph: ⟳ IN PROGRESS
    ├── 001-setup: ✓ DONE (completed 2026-06-14 11:15)
    ├── 002-auth:  ⟳ IN PROGRESS (ralph running)
    ├── 003-api:   ○ PENDING (blocked by 002)
    ├── 004-ui:    ○ PENDING (blocked by 002)
    └── 005-docs:  ○ PENDING (AFK)
```

Note: Timestamps are read from `.seed/{number}/state.json`. If state.json doesn't exist, show "N/A". If a stage/task is in progress, show no timestamp.

**Upcoming work section** (shown in detailed view):
```
## Upcoming work
1. /ralph 002 005 — Task 005: pipeline skill (run --next) (AFK, unblocked)
2. /ralph 002 006 — Task 006: pipeline skill (run --all) (AFK, blocked by 005)
3. /ralph 002 007 — Task 007: pipeline skill (edge cases) (AFK, blocked by 005)
```

**Rules for upcoming work:**
1. Only show tasks that can actually run (not blocked by incomplete dependencies)
2. Sort by task number (001, 002, 003...)
3. Show top 3
4. AFK tasks are runnable; HITL tasks are shown but marked as HITL
5. If all tasks complete, show "All tasks complete"
6. For each task, show: command to run, task title, type, blocked status

### 5. Handle seed not found

If `/where 999` is invoked and seed 999 does not exist:
```
Seed 999: not found
No seed directory exists at .seed/999/
```

### 6. Emit Status Block

After completing, emit a trailing Status Block.

---

## Output Contract

`where` produces terminal output (text), not files.

**Output schema:**
```
{all-seeds-summary | detailed-tree | error-message}
```

---

## Failure Behavior

### No seeds found

**Trigger:** `.seed/` directory is empty or does not exist.

**Handling:**
```
No seeds found.
Create a seed with /grill @.seed/{number}/init.md first.
```

### Seed not found

**Trigger:** Seed directory does not exist.

**Handling:**
```
Seed {NNN}: not found
No seed directory exists at .seed/{NNN}/
```

### Malformed task files

**Trigger:** Task files exist but cannot be parsed.

**Handling:** Log a warning and continue. Show "?" for status of unparseable tasks.

---

## Status Block

After completing, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "where",
  "summary": "One-line description of what was shown",
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```
