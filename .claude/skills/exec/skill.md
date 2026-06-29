# /exec Skill

## References

Before doing anything else, read and internalize these documents in full:

- `reference/structure.md` — the directory layout, file roles, and naming conventions of the workflow
- `reference/pipeline.md` — the overall pipeline flow and how stages connect
- `reference/state.md` — the state.json schema and update rules, including the generic stage lifecycle
- `reference/tdd.md` — the testing discipline that all implementation produced by this skill must follow

Do not proceed until you have read all four.

---

## Purpose

`/exec` is the final skill in the pipeline. It takes a single task file and implements it — writing the actual code, configuration, or other changes that satisfy the task's Details and Verification sections.

This is the only stage in the pipeline that modifies the codebase. Every previous stage (`/dive`, `/product`, `/tasks`) is read-only and produces planning documents; `/exec` is where those plans become real changes.

Every implementation produced by `/exec` must have a passing test, per `reference/tdd.md`. A task is not done until its tests exist, run, and pass.

---

## Invocation

```
/exec <task-file>
```

| Argument | Description |
|---|---|
| `<task-file>` | A path to a single task file, e.g. `.feed/001/tasks/003_task.md` or `@.feed/001/tasks/003_task.md` |

`/exec` operates on exactly one task per invocation. It does not take a session identifier and does not work through multiple tasks automatically.

### Resolving the input

1. Resolve `<task-file>` to a concrete path.
2. Confirm the file exists and is not empty.
3. Identify the session this task belongs to (the parent of `tasks/`) to locate `state.json`.
4. Read the task's `Dependencies` section.

### Error handling

| Condition | Response |
|---|---|
| Task file does not exist | Tell the human the path was not found. Do not proceed. |
| Task file is empty | Tell the human the task file appears empty or malformed. Do not proceed. |
| `state.json` for the session is missing | Tell the human the session has no state file, which shouldn't happen if `/tasks` ran correctly. Do not proceed without it — recreate it only if the human explicitly confirms how. |
| One or more declared dependencies are not `completed` in `state.json` | Stop and tell the human which dependency task(s) are not done yet, and that this task cannot be safely started until they are. Do not proceed without explicit override from the human. |
| Task has no `Dependencies` (`None.`) | Proceed normally. |

---

## Process

### Step 1 — Read the task in full

Read all sections of the task file: Source, Description, Dependencies, Details, Verification. The Details section is the specification; the Verification section is the definition of done.

If anything in Details is genuinely ambiguous at the implementation level (not a product decision — a technical detail like "which existing utility to reuse"), resolve it by reading the surrounding codebase. Do not guess where the codebase already shows the convention to follow.

### Step 2 — Implement with tests, per `reference/tdd.md`

Write the implementation and its test(s) together, in whichever order is natural for the change. Both must exist before the task is considered done.

- Every condition listed in the task's Verification section must be covered by at least one test.
- Tests must be real tests per `reference/tdd.md` — they must actually exercise the implementation and be capable of failing.
- Follow the existing codebase's testing conventions (framework, file location, naming) rather than introducing new ones.

### Step 3 — Run the tests

Execute the relevant test suite — not just the new test(s), but the surrounding suite likely to be affected by this change.

- All relevant tests must pass. A task with any failing test is not done.
- If running tests reveals a regression in unrelated code, stop and surface it to the human before proceeding — do not silently modify unrelated code to force the suite to pass.
- If a test framework or dependency needs to be installed to run tests, do so as part of this step.

### Step 4 — Verify against the task's Verification section

Go through each checkbox in the task's Verification section explicitly. For each one, confirm it is satisfied by a passing test or a directly observable check. Do not mark a task done with an unverified checkbox.

---

## State

`/exec` follows the general stage lifecycle defined in `reference/state.md` ("Stage Lifecycle"), applied to **this specific task's entry** in the `tasks` block of `state.json` (not to a top-level pipeline stage):

- On start: set this task's `status` to `in_progress`, `started_at` to the current UTC timestamp.
- During the work: update `state.json` after meaningful checkpoints (e.g. implementation written, tests written, tests passing) so an interruption doesn't lose progress.
- On completion: set this task's `status` to `completed`, `completed_at` to the current UTC timestamp — only once all of Step 2–4 above are satisfied.

No other `/exec`-specific state handling exists beyond what `state.md` already defines. `/exec` never modifies the `stages` block — only its own task's entry under `tasks`.

---

## Boundaries

- `/exec` only ever touches the codebase for the single task it was given. It does not implement other tasks, even adjacent ones, even if they look trivial.
- `/exec` does not modify `feed.md`, `context.md`, `product.md`, or any task file other than updating its own task's state.
- `/exec` does not make product or scope decisions. If the task's Details turn out to be wrong or insufficient at the implementation level in a way that isn't a simple technical convention question, stop and surface it to the human rather than improvising a product decision.

---

## Examples

### Example 1 — Implementing a task with TDD

**Task file (`tasks/001_task.md`):**
```markdown
# Task 001 — Create PromoCode entity

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

**Good /exec behavior:**
1. No dependencies — proceed.
2. Write the `PromoCode` entity class with the specified fields and JPA annotations for the unique constraint.
3. Write a test that attempts to persist two `PromoCode` rows with the same `code` and asserts a constraint violation is thrown.
4. Run the test — confirm it fails against a version of the entity without the constraint (to confirm the test is real), then confirm it passes with the constraint in place.
5. Run the full test suite for the module to check for regressions.
6. Check both Verification boxes — entity compiles and maps correctly (confirmed by a successful persist in the test setup), and the unique constraint is enforced (confirmed by the constraint-violation test).
7. Mark task `001` `completed` in `state.json`.

---

### Example 2 — Blocked by an unmet dependency

**Invocation:**
```
/exec .feed/001/tasks/003_task.md
```

**state.json excerpt:**
```json
"tasks": {
  "001": { "status": "completed", ... },
  "002": { "status": "not_started", ... },
  "003": { "status": "not_started", ... }
}
```

**Task 003's Dependencies:**
```
- Task 001
- Task 002
```

**Good /exec response:**
> Task `003` depends on task `002`, which has not been started yet. I can't safely implement `003` until `002` is done — it likely relies on what `002` produces (e.g. a repository method this task's endpoint will call). Recommend running `/exec` on task `002` first.

---

### Example 3 — A test that isn't real

**Bad test (rejected):**
```java
@Test
void testCreatePromoCode() {
    PromoCode code = new PromoCode();
    assertTrue(true);
}
```

This test cannot fail under any implementation, correct or broken. It does not satisfy `reference/tdd.md` and does not count toward the task's Verification.

**Good test:**
```java
@Test
void duplicateCodeViolatesUniqueConstraint() {
    promoCodeRepository.save(new PromoCode("SAVE10", ...));
    PromoCode duplicate = new PromoCode("SAVE10", ...);

    assertThrows(DataIntegrityViolationException.class, () -> {
        promoCodeRepository.saveAndFlush(duplicate);
    });
}
```

This test fails if the unique constraint is missing and passes once it's correctly in place — it actually verifies the behavior the task requires.