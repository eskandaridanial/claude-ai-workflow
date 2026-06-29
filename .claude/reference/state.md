# State Reference

> This document defines the schema, lifecycle, and update rules for `state.json`.  
> Every skill reads this file at the start of its run and updates it as it progresses.

---

## Location

Each session has its own state file:

```
.feed/001/state.json
```

It is created at the beginning of a session, before any skill runs, and lives alongside the session's documents for its entire lifetime.

---

## Purpose

`state.json` is the single source of truth for where a session currently stands. It answers:

- Which stages have run, are running, or have not started yet
- When each stage started and when it completed
- What tasks exist and the status of each one
- Enough context to resume a session that was interrupted at any point

All skills read this file first to understand the current state of the session before doing any work, and write to it whenever their state changes.

---

## Graceful Shutdown

Because skills update `state.json` continuously as they progress — not only at completion — a session can be safely interrupted at any point and resumed later. No work is silently lost. The file always reflects the last known state.

---

## Schema

```json
{
  "session": "001",
  "created_at": "2026-06-16T10:00:00Z",
  "updated_at": "2026-06-16T11:42:00Z",
  "stages": {
    "dive": {
      "status": "completed",
      "started_at": "2026-06-16T10:01:00Z",
      "completed_at": "2026-06-16T10:45:00Z"
    },
    "product": {
      "status": "in_progress",
      "started_at": "2026-06-16T10:46:00Z",
      "completed_at": null
    },
    "tasks": {
      "status": "not_started",
      "started_at": null,
      "completed_at": null
    }
  },
  "tasks": {
    "001": {
      "status": "not_started",
      "started_at": null,
      "completed_at": null
    }
  }
}
```

---

## Fields

### Top-level

| Field | Type | Description |
|---|---|---|
| `session` | string | The session identifier this state file belongs to |
| `created_at` | ISO 8601 string | When this state file was first created |
| `updated_at` | ISO 8601 string | Last time any field in this file was written |

---

### `stages`

Tracks the lifecycle of each pipeline stage. Every stage has the same shape:

| Field | Type | Description |
|---|---|---|
| `status` | string | One of: `not_started`, `in_progress`, `completed`, `stale` |
| `started_at` | ISO 8601 string or null | When the skill began its run |
| `completed_at` | ISO 8601 string or null | When the skill finished and its output was written |

A stage marked `stale` was previously `completed`, but an upstream document it depended on has since changed, meaning its own output may no longer be accurate. A stale stage must be re-run before its output is trusted by the next stage.

#### Stage completion criteria

| Stage | Marked `completed` when |
|---|---|
| `dive` | All questions are resolved, all assumptions are explicit, human and agent are aligned, and `context.md` is written |
| `product` | All requirements are clear to the agent and `product.md` is written |
| `tasks` | All task files are written under `tasks/` |

---

### `tasks`

Tracks each individual task produced by the `/tasks` stage. Keys are task identifiers matching the task filenames (e.g. `"001"` for `tasks/001_task.md`).

Each task entry has the same shape as a stage:

| Field | Type | Description |
|---|---|---|
| `status` | string | One of: `not_started`, `in_progress`, `completed` |
| `started_at` | ISO 8601 string or null | When `/exec` began working on this task |
| `completed_at` | ISO 8601 string or null | When `/exec` finished this task |

Task entries are added to this block by the `/tasks` stage when task files are created. Initially all tasks have status `not_started`.

---

## Stage Lifecycle

This section defines, generically, how any skill must read and write `state.json` around its own stage — on a normal first run, while it runs, on completion, and when it is re-entered to revisit something already done.

### On start — first run

If `state.json` does not exist yet, create it using the schema in [Initial State](#initial-state).

Set the stage's `status` to `in_progress` and its `started_at` to the current UTC timestamp. Write the file.

### On start — re-entry (e.g. a focused re-run on an already-completed or stale stage)

Set the stage's `status` to `in_progress`. Keep the existing `started_at` from the prior run unchanged — re-entry is a continuation of the same stage, not a new beginning. Write the file.

### During the run

Update `state.json` after every meaningful checkpoint, not only at the very end. This ensures that if the session is interrupted partway through, no progress is silently lost and the file always reflects the last known state.

### On completion

Set the stage's `status` to `completed` and its `completed_at` to the current UTC timestamp.

If this was a re-entry and any downstream stage had already completed using this stage's now-updated output, mark each of those downstream stages `stale` (see [Update Rules](#update-rules)). Write the file.

---

## Initial State

When a session is created, `state.json` is written with all stages and tasks set to `not_started` and all timestamps set to `null`:

```json
{
  "session": "001",
  "created_at": "2026-06-16T10:00:00Z",
  "updated_at": "2026-06-16T10:00:00Z",
  "stages": {
    "dive": {
      "status": "not_started",
      "started_at": null,
      "completed_at": null
    },
    "product": {
      "status": "not_started",
      "started_at": null,
      "completed_at": null
    },
    "tasks": {
      "status": "not_started",
      "started_at": null,
      "completed_at": null
    }
  },
  "tasks": {}
}
```

---

## Update Rules

- `updated_at` is refreshed on every write, regardless of which field changed.
- A stage moves to `in_progress` when it begins and its `started_at` is set.
- A stage moves to `completed` only when its output document(s) are fully written and verified.
- A stage must never skip `in_progress` — every stage passes through `not_started` → `in_progress` → `completed`.
- If a completed stage's upstream input changes after the fact (e.g. its source document is patched), the stage itself stays `completed` but every stage *after* it that already ran is marked `stale`. A `stale` stage keeps its existing `completed_at` timestamp until it is re-run.
- A `stale` stage returns to `in_progress` when it is re-run, and back to `completed` when it finishes — same as a normal first run.
- The `tasks` block starts empty (`{}`) and is populated by the `/tasks` stage as task files are created.
- Tasks follow the same status progression as stages.
- No field is ever deleted. Null is used for timestamps that have not occurred yet.