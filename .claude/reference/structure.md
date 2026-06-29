# Structure Reference

> This document describes the directory layout, file roles, and naming conventions of the AI workflow system.  
> It contains no skill-specific or pipeline-specific information — only where data lives and what each file represents.

---

## Overview

All workflow content lives under a single root directory:

```
.feed/
```

Each **session** is a subdirectory inside `.feed/`. A session is one discrete unit of work — a single idea, task, or product you want to develop.

---

## Directory Layout

```
.feed/
├── 001/
│   ├── feed.md
│   ├── context.md
│   ├── product.md
│   ├── state.json
│   └── tasks/
│       ├── 001_task.md
│       └── 002_task.md
├── 002/
│   ├── feed.md
│   ├── context.md
│   ├── product.md
│   ├── state.json
│   └── tasks/
│       └── 001_task.md
└── 003/
    └── feed.md
```

---

## Session Identifier

Each session directory is named with a **zero-padded 3-digit integer**:

```
001, 002, 003, ..., 099, 100, ..., 999
```

- Identifiers are assigned sequentially and never reused.
- A session is referenced by its identifier alone (e.g. `001`) or by a direct path alias to any file within it (e.g. `@.feed/001/feed.md`).

---

## Files

### `feed.md` — Session Input

**Written by:** you (the human)  
**Read by:** `/dive` skill that needs to understand the intent of this session

The starting point of a session. Write whatever captures what you are trying to do — a rough idea, a description, a list of requirements, a problem statement, or a mix. No structure is enforced.

**Example:**

```markdown
I want to build a thing that helps me keep track of foo.

Foo has a name, a status, and a date. I need to be able to create foo,
list all foos, and mark a foo as done. There should be some way to filter
by status.

Not sure yet about the tech stack.
```

---

### `context.md` — Resolved Context

**Written by:** `/dive` skill  
**Read by:** `/product` skill that produces `product.md`

The result of a clarification process — all ambiguities from `feed.md` have been surfaced and resolved, assumptions are made explicit, and the intent is fully understood.

**Example:**

```markdown
# Context

## What we're building
A personal tracker for foo. Each foo has a name, a status (pending / done),
and a creation date. The user can create foos, list them, filter by status,
and mark any foo as done.

## Resolved
- No editing of foo name after creation
- Filtering is client-side for now
- No authentication required in the first version
- Stack: foo-lang with a foo-db backend

## Out of scope
- Notifications
- Bulk operations
```

---

### `product.md` — Product Requirements Document

**Written by:** `/product` skill  
**Read by:** `/tasks` skill that produces task files

A structured requirements document derived from `context.md`. Defines what is being built, its scope, and what done looks like.

**Example:**

```markdown
# Product

## Goal
A lightweight tracker for managing foos.

## Scope
- Create a foo
- List all foos
- Filter foos by status
- Mark a foo as done

## Non-Goals
- User accounts
- Notifications
- Bulk actions

## Acceptance Criteria
- A foo can be created with a name
- The list shows all foos with their status and date
- Filtering by status shows only matching foos
- Marking done is irreversible
```

---

### `tasks/NNN_task.md` — Individual Tasks

**Written by:** `/tasks` skill  
**Read by:** `/exec` skill

Each task is a separate file inside the `tasks/` subdirectory. Files are named with a zero-padded 3-digit index followed by `_task`:

```
tasks/001_task.md
tasks/002_task.md
```

Each file represents one atomic unit of work.

**Example (`tasks/001_task.md`):**

```markdown
# Task 001

## Description
Define the data model for foo.

## Details
- Foo has: id, name, status, createdAt
- Status is an enum: pending | done
- id is auto-generated

## Acceptance Criteria
- Model is defined and matches the agreed schema
- Status defaults to pending on creation
```

---

### `state.json` — Session State

**Written by:** all skills  
**Read by:** all skills

Tracks the current state of this session. Written and updated by every skill as the pipeline progresses.

> The schema and full specification of `state.json` are defined in `reference/state.md`.

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Session directory | Zero-padded 3-digit integer | `001`, `042`, `999` |
| Task files | `NNN_task.md`, zero-padded 3-digit | `001_task.md`, `042_task.md` |
| File names | Lowercase, no spaces | `feed.md`, `product.md` |
| Path alias | `@` prefix + path relative to project root | `@.feed/001/feed.md` |
| Session reference | Identifier alone or path alias | `001` or `@.feed/001/feed.md` |