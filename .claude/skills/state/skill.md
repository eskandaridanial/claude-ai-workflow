---
name: state
description: Report the current status of a session's pipeline progress, or list all sessions, by reading state.json. Use when the human wants to know where a session stands without changing anything.
---

# /state Skill

## References

Before doing anything else, read and internalize these documents in full:

- `reference/structure.md` — the directory layout, file roles, and naming conventions of the workflow
- `reference/state.md` — the state.json schema, including the stage lifecycle and the `stale` status

Do not proceed until you have read both.

---

## Purpose

`/state` reports where a session currently stands in the pipeline by reading its `state.json`. It is purely informational — it never writes to `state.json`, never modifies any file, and never suggests what to run next. It only tells the human what has happened so far.

---

## Invocation

```
/state
/state <session>
```

| Argument | Description |
|---|---|
| *(none)* | Show a one-line status summary for every session under `.feed/` |
| `<session>` | A session identifier (`001`) or a path alias (`@.feed/001/`). Show the full status breakdown for that one session. |

---

## Mode A — No argument: all sessions

List every session directory under `.feed/`. For each one, read its `state.json` and produce a single summary line.

### Error handling

| Condition | Response |
|---|---|
| `.feed/` does not exist or has no session directories | Tell the human no sessions exist yet, and that a session starts by creating `.feed/NNN/feed.md`. |
| A session directory exists but has no `state.json` | Show it in the list with status `uninitialized` rather than failing the whole command. |

### Output format

A short table, one row per session, ordered by session identifier:

```
Session   Stage      Tasks        Overall
------- -------- ----------- -----------------
001       exec       3/5 done    in progress
002       tasks      not started not started
003       product    completed   pipeline done (no tasks yet)
```

The `Stage` column shows the furthest stage that is `completed` or `in_progress` (whichever is most advanced). `Tasks` summarizes completed-vs-total task count if `/tasks` has run, otherwise `not started`. `Overall` is a short plain-language read of where things stand for that session.

After the table, give a one-line plain-text summary, e.g.:

> 3 sessions found. `001` is mid-implementation, `002` hasn't started `/tasks` yet, `003` is fully planned but has no tasks generated.

---

## Mode B — Specific session: full breakdown

### Resolving the input

1. Resolve `<session>` to a session directory (e.g. `.feed/001/`)
2. Confirm `state.json` exists at `.feed/001/state.json`

### Error handling

| Condition | Response |
|---|---|
| Session identifier does not exist | Tell the human the session directory `.feed/NNN/` was not found. |
| `state.json` does not exist for an existing session | Tell the human this session has not been initialized yet — no skill has run against it. Recommend starting with `/dive`. |
| `state.json` exists but fails to parse | Tell the human the state file appears corrupted, and show the raw content so they can inspect it. Do not attempt to fix or rewrite it. |

### Output

Give both a short plain-text summary and a detailed table, in that order.

**1. Plain-text summary** — two or three sentences, written for a human skimming quickly:

> Session `001` has completed `/dive` and `/product`. `/tasks` is in progress. No tasks have been created yet, so there's nothing for `/exec` to work on.

If any stage is `stale`, call it out explicitly and say why, by referencing what triggered it (the focused re-entry that occurred), if that's inferable from the data available:

> Note: `product` is marked **stale** — `context.md` was revised after `product.md` was last generated, so `product.md` may no longer be accurate.

**2. Detailed table** — every stage, in pipeline order, plus every task if the `tasks` block is non-empty:

```
Stage      Status        Started              Completed
--------  ------------  -------------------  -------------------
dive       completed     2026-06-16T10:01:00Z 2026-06-16T10:45:00Z
product    stale         2026-06-16T10:46:00Z 2026-06-16T11:20:00Z
tasks      not_started   —                    —

Task       Status        Started              Completed
--------  ------------  -------------------  -------------------
001        completed     2026-06-16T12:00:00Z 2026-06-16T12:30:00Z
002        in_progress   2026-06-16T12:31:00Z —
003        not_started   —                    —
```

If the `tasks` block is empty, omit the task table and state plainly that no tasks exist yet for this session.

---

## Boundaries

- `/state` never writes to `state.json` or any other file.
- `/state` never recommends a next command to run — it reports status only.
- `/state` never modifies, fixes, or regenerates a session's documents, even if it notices something is `stale` or inconsistent. It only surfaces what it finds.

---

## Examples

### Example 1 — No argument, multiple sessions

**Invocation:**
```
/state
```

**Good response:**
```
Session   Stage      Tasks        Overall
------- -------- ----------- -----------------
001       exec       2/4 done    in progress
002       dive       not started in progress
003       tasks      0/3 done    tasks generated, none started
```

3 sessions found. `001` is mid-implementation with 2 of 4 tasks done. `002` is still in the clarification interview. `003` has a full task breakdown ready but `/exec` hasn't started on any of them.

---

### Example 2 — Specific session, fully on track

**Invocation:**
```
/state 001
```

**Good response:**

Session `001` has completed `/dive`, `/product`, and `/tasks`. 4 tasks exist; 2 are done, 1 is in progress, 1 hasn't started.

```
Stage      Status        Started              Completed
--------  ------------  -------------------  -------------------
dive       completed     2026-06-16T10:01:00Z 2026-06-16T10:45:00Z
product    completed     2026-06-16T10:46:00Z 2026-06-16T11:20:00Z
tasks      completed     2026-06-16T11:21:00Z 2026-06-16T11:40:00Z

Task       Status        Started              Completed
--------  ------------  -------------------  -------------------
001        completed     2026-06-16T12:00:00Z 2026-06-16T12:30:00Z
002        completed     2026-06-16T12:31:00Z 2026-06-16T13:00:00Z
003        in_progress   2026-06-16T13:01:00Z —
004        not_started   —                    —
```

---

### Example 3 — Specific session, with a stale stage

**Invocation:**
```
/state 002
```

**Good response:**

Session `002` has a `dive` re-entry that invalidated downstream work. `context.md` was last updated after `product.md` was generated, so `product` is marked stale and should be regenerated before trusting it. `tasks` has not run yet.

```
Stage      Status        Started              Completed
--------  ------------  -------------------  -------------------
dive       completed     2026-06-20T09:00:00Z 2026-06-20T09:50:00Z
product    stale         2026-06-20T09:55:00Z 2026-06-20T10:20:00Z
tasks      not_started   —                    —
```

No tasks exist yet for this session.

---

### Example 4 — Session with no state.json

**Invocation:**
```
/state 005
```

**Good response:**

> Session `005` exists but has no `state.json` — no skill has run against it yet. If `feed.md` is already written, the next step would be starting with `/dive`.