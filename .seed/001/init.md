# Seed: 001 — Sample task: Add a string utility function

## Meta
id:   001
type: task

## Goal
Add a simple string utility function `capitalize` that capitalizes the first letter of a string.

## Context
This is a sample task for end-to-end validation of the TDD pipeline integration. It tests that TDD is invoked for every AFK slice and status blocks contain TDD evidence fields.

## Scope

### In
- A `capitalize` function that capitalizes the first letter
- Tests for the function

### Out
- No external API changes
- No database changes

## Acceptance criteria
- [ ] `capitalize("hello")` returns `"Hello"`
- [ ] `capitalize("Hello")` returns `"Hello"` (already capitalized)
- [ ] `capitalize("")` returns `""` (empty string)
- [ ] `capitalize("a")` returns `"A"` (single character)

## Notes
This is a sample task for validation only.
