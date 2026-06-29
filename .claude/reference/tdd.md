# TDD Reference

> This document defines the testing discipline that all implementation work in this workflow must follow.  
> It is skill-independent — it does not describe what `/exec` does, only what "tested" means and what rules govern tests.

---

## Principle

No implementation is complete without a test that verifies it. A task is not done because code was written; it is done because a test exists, runs, and passes, proving the task's verification criteria are actually met.

Tests are not optional, not an afterthought, and not something to add "later." A task without a passing test is an incomplete task, regardless of whether the code appears to work.

---

## Required, Not Ordered

Every unit of implementation must have a corresponding test, and that test must pass before the unit is considered done. The order in which the test and the implementation are written is not fixed:

- A test may be written first, then the implementation made to satisfy it.
- An implementation may be written first, then a test added that exercises it.

Either order is acceptable. What is **not** acceptable:

- Implementation with no test at all.
- A test that does not actually exercise the implementation (e.g. a test that always passes regardless of behavior).
- A test that is written but never run.
- Skipping or commenting out a failing test to move forward.

---

## What Counts as a Test

A valid test:

- Exercises the actual implementation, not a mock of the thing being tested.
- Has a clear pass/fail outcome — no manual judgment needed to interpret the result.
- Fails when the implementation is wrong or missing, and passes when it is correct. (If a test passes regardless of whether the implementation is right, it is not a real test.)
- Is automated — runnable without a human performing manual steps.

A test that doesn't fail under any incorrect implementation is not testing anything. Before trusting a passing test, it should be possible to imagine a broken implementation that this specific test would catch.

---

## Test Types

Match the test type to what's being verified. Not every unit needs every type:

| Type | When to use |
|---|---|
| Unit test | A single function, method, or class can be verified in isolation |
| Integration test | Behavior depends on the interaction between multiple components (e.g. a repository against a real or in-memory database) |
| Contract / API test | An endpoint or public interface's input/output behavior needs verifying |

Choose the narrowest test type that meaningfully verifies the behavior. Prefer unit tests where isolation is possible; reach for integration tests only when the behavior genuinely can't be verified without the real interaction.

---

## Coverage Expectations

A test (or set of tests) for a unit of work must cover:

- The primary success path — the expected behavior under normal, valid input.
- Each distinct failure or edge case that was explicitly defined for this unit (e.g. validation failure, duplicate entry, not-found case).

A unit of work is not tested merely because *a* test exists for it — the test(s) must cover the specific verification conditions defined for that unit. An empty-input test alone does not cover a duplicate-key rejection requirement; both need their own coverage if both are required behaviors.

---

## Running Tests

Writing a test is not sufficient — it must actually be executed, and it must pass.

- Run the full test suite relevant to the changed code, not only the newly written test, to catch regressions in surrounding code.
- A test suite with any failing test is a failing state. Do not treat the unit of work as done while any relevant test fails.
- If a pre-existing test starts failing because of a change, that failure must be resolved — either the change is wrong, or the old test's expectation is genuinely outdated and the test itself should be deliberately and explicitly updated (never silently deleted or skipped to make the suite pass).

---

## What This Document Does Not Cover

This document defines the testing discipline only. It does not describe:

- Which skill is responsible for writing or running tests — that is defined in the relevant skill's own documentation.
- Specific test frameworks, libraries, or tooling choices — those depend on the codebase and are determined contextually, not prescribed here.
- How task verification criteria are written — that is defined in `reference/structure.md` and the `/tasks` skill.