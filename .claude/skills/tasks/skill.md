# /tasks Skill

## References

Before doing anything else, read and internalize these documents in full:

- `reference/structure.md` — the directory layout, file roles, and naming conventions of the workflow
- `reference/pipeline.md` — the overall pipeline flow and how stages connect
- `reference/state.md` — the state.json schema and update rules, including the generic stage lifecycle

Do not proceed until you have read all three.

---

## Purpose

`/tasks` is the third skill in the pipeline. It reads `product.md` and breaks every user story down into atomic units of work, written as individual files under `tasks/`. Each task must be as small as possible while still being independently verifiable — completable and checkable on its own, without requiring another unfinished task to judge whether it succeeded.

The output is the `tasks/` directory, the input for `/exec`.

No code changes. No implementation. This stage decomposes *what* is being built into the smallest verifiable steps to build it.

---

## Role

Act as a technical lead breaking down approved requirements into an execution plan. This means:

- `product.md` is treated as complete and correct. Do not question its product decisions, and do not escalate gaps back to `/product` — if something seems underspecified, resolve it at the most reasonable, narrowest technical interpretation and proceed.
- Decompose, don't dilute: each task must be small, but it must still represent a real, coherent unit of work — not an arbitrary line-by-line split.
- Every task must be independently verifiable: it has a clear, checkable definition of "this works" that doesn't depend on a sibling task also being finished.

---

## Invocation

```
/tasks <session>
```

| Argument | Description |
|---|---|
| `<session>` | A session identifier (`001`) or a path alias (`@.feed/001/product.md`) |

### Resolving the input

1. Resolve `<session>` to a session directory (e.g. `.feed/001/`)
2. Confirm `product.md` exists at `.feed/001/product.md`
3. Confirm `product.md` is not empty
4. Check `state.json` — confirm `stages.product.status` is `completed`. If it is `stale`, warn the human that `product.md` may be out of date and confirm whether to proceed anyway.

### Error handling

| Condition | Response |
|---|---|
| Session identifier does not exist | Tell the human the session directory `.feed/NNN/` was not found. |
| `product.md` does not exist | Tell the human `/product` has not been run yet (or did not complete) for this session. Recommend running `/product <session>` first. Do not proceed. |
| `product.md` exists but is empty | Treat as equivalent to missing. Recommend re-running `/product`. |
| `stages.product.status` is `not_started` or `in_progress` | Tell the human `/product` has not finished. Do not proceed. |
| `stages.product.status` is `stale` | Warn the human and ask for confirmation before proceeding. |

---

## Decomposition Process

### Step 1 — Read every user story

Go through `product.md` story by story (`US-1`, `US-2`, ...). For each story, identify the distinct technical steps required to satisfy its acceptance criteria.

### Step 2 — Split into atomic tasks

A single user story becomes one or more tasks. Split a story into multiple tasks whenever it requires more than one independently verifiable unit of work to complete.

A task is atomic when:

- It does **one thing** — one entity, one endpoint, one function, one config change, one migration. Not a bundle of related things.
- It can be **verified on its own** — there is a concrete check (a test passes, an endpoint returns the right response, a query returns the right rows) that doesn't require any other unfinished task to be true first.
- It is **as small as it can be** while still being a coherent, meaningful unit — splitting further would produce a task with no independent verification (e.g. don't split "add a field" into "add the field" + "name the field").

If a task's verification step requires another task to exist first, that's a dependency (see below) — it does not mean the two should be merged into one task, unless the dependency is so tight that they cannot be verified separately at all.

### Step 3 — Identify dependencies

For each task, determine whether it requires another task to be completed first (e.g. a repository task depends on the entity task; an endpoint task depends on the service-layer task it calls).

- Record dependencies as references to other task IDs.
- A task with no dependencies can be started immediately.
- Dependencies must only point to tasks within the same session.
- Do not create circular dependencies. If a circular dependency seems to appear, the tasks are not actually atomic — recombine or re-split them until the dependency graph is acyclic.

### Step 4 — Number and write

Write each task to `.feed/NNN/tasks/NNN_task.md`, numbered sequentially starting from `001` in the order tasks are most sensibly executed (dependencies before dependents, where possible).

---

## State

`/tasks` follows the general stage lifecycle defined in `reference/state.md` ("Stage Lifecycle"), applied to the `tasks` stage.

When `/tasks` completes, it must also populate the `tasks` block in `state.json` — one entry per task file created, each initialized to `not_started` per the schema in `reference/state.md`. This population is part of what "completed" means for this stage; the `tasks` stage is not `completed` until both the task files exist and their entries exist in `state.json`.

No other `/tasks`-specific state handling exists beyond what `state.md` already defines.

---

## Output: `tasks/NNN_task.md`

Write each task to `.feed/NNN/tasks/NNN_task.md`, per the naming convention in `reference/structure.md`.

Write for an AI agent, not a human — be explicit. `/exec` must be able to pick up a single task file and implement it without needing to read `product.md` or `context.md` again.

### Format

```markdown
# Task NNN — [Short title]

## Source
US-N from product.md ([short reference to which story this came from])

## Description
One or two sentences. What this task does, precisely.

## Dependencies
- Task NNN (why it must come first)
- None (if no dependencies)

## Details
Concrete technical specification of what to do. Specific enough that no interpretation is needed:
- Exact names (classes, fields, endpoints, files) where determinable
- Exact behavior expected
- Anything from product.md's Notes or Assumptions that applies to this specific task

## Verification
How to check, independently of any other task, that this one is done correctly.
- [ ] Concrete, checkable condition
- [ ] Concrete, checkable condition
```

All sections must be present. If a section has nothing to say, write `None.` — do not omit the section.

### Task numbering

- Numbers are zero-padded 3-digit, sequential within the session, starting at `001`.
- Numbers are never reused, even if a task is later deleted or merged during a future re-run.

---

## Examples

### Example 1 — Splitting a story into atomic tasks

**Source story (`product.md`):**
```markdown
### US-1: Admin creates a promo code
**As an** admin
**I want** to create a promo code with a code string, expiry date, and usage limit
**So that** I can run time-bound promotions with controlled usage

**Acceptance Criteria:**
- [ ] POST /promo-codes accepts code, expiresAt, and usageLimit
- [ ] A created code defaults to active
- [ ] Creating a code with a duplicate code string returns 409
```

**Bad decomposition (not atomic — too coarse):**
```
001_task.md: "Implement promo code creation" (entity + repository + service + controller + validation, all in one task)
```

**Good decomposition:**
```markdown
# Task 001 — Create PromoCode entity

## Source
US-1 from product.md (promo code creation)

## Description
Define the PromoCode JPA entity.

## Dependencies
None.

## Details
- Fields: id (Long, PK), code (String, unique), expiresAt (LocalDateTime),
  usageLimit (Integer, nullable = unlimited), active (Boolean, default true)
- Table name: promo_codes
- Unique constraint on code

## Verification
- [ ] Entity compiles and maps to the promo_codes table
- [ ] Unique constraint on code is enforced at the DB level
```

```markdown
# Task 002 — Create PromoCodeRepository

## Source
US-1 from product.md (promo code creation)

## Description
Define the Spring Data repository for PromoCode, including a method to check
for an existing code string.

## Dependencies
- Task 001 (PromoCode entity must exist)

## Details
- Interface extends JpaRepository<PromoCode, Long>
- Add: boolean existsByCode(String code)

## Verification
- [ ] Repository compiles
- [ ] existsByCode returns true for an existing code, false otherwise (verified via a repository test)
```

```markdown
# Task 003 — Implement POST /promo-codes endpoint

## Source
US-1 from product.md (promo code creation)

## Description
Add the endpoint that creates a promo code from a request body containing
code, expiresAt, and usageLimit.

## Dependencies
- Task 001 (PromoCode entity)
- Task 002 (PromoCodeRepository, for duplicate-code check)

## Details
- POST /promo-codes
- Request body: { code: string, expiresAt: ISO date string, usageLimit: integer|null }
- On duplicate code (existsByCode returns true): respond 409
- On success: persist with active = true, respond 201 with the created resource

## Verification
- [ ] POST with a new code returns 201 and the code is persisted with active = true
- [ ] POST with an existing code returns 409
- [ ] POST with missing required fields returns 400
```

This is three tasks from one story, each independently verifiable, each small, with a clear dependency chain (003 depends on 001 and 002; 002 depends on 001; 001 has no dependencies).

---

### Example 2 — Recognizing when NOT to split further

**Bad over-splitting:**
```
001_task.md: "Add `code` field to PromoCode entity"
002_task.md: "Add `expiresAt` field to PromoCode entity"
003_task.md: "Add `usageLimit` field to PromoCode entity"
```

This is wrong — none of these fields can be meaningfully verified on their own; the entity itself is the atomic unit, not each individual field. Correct atomicity here is one task: "Create PromoCode entity" with all fields included (as in Example 1, Task 001).

---

### Example 3 — Resolving an underspecified detail without escalating

**Situation:** `product.md`'s acceptance criteria say "Creating a code with a duplicate code string returns 409" but doesn't specify the response body shape for the error.

**Good /tasks behavior:** Do not escalate to `/product`. Resolve at the narrowest reasonable technical interpretation and state it in the task's Details:

```markdown
## Details
- On duplicate code: respond 409 with body { "error": "DUPLICATE_CODE" }
  (response shape not specified in product.md; using the simplest convention consistent with a typical REST error response)
```