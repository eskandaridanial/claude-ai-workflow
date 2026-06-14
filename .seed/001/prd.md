# Seed: 003 — PRD: Add a string utility function

## Meta
id: 003
stage: prd

## Problem Statement

Users need a simple string utility function to capitalize the first letter of a string.

## Solution

Add a `capitalize` function that capitalizes the first letter of a string, with comprehensive tests using TDD.

---

## User Stories

### US-001: Capitalize Function
**As a** developer
**I want** to use a `capitalize` function
**So that** I can capitalize the first letter of strings in my code

---

## Implementation Decisions

### ID-001: Function Location
**Decision**: Create a `string_utils.py` file in the project's utils directory
**Rationale**: Keeps utility functions organized and reusable
**Tradeoffs**: None - simple organizational decision

### ID-002: TDD Approach
**Decision**: Write tests first using pytest, then implement the function
**Rationale**: Validates the TDD pipeline integration for seed 002
**Tradeoffs**: Tests must pass before implementation is considered complete

---

## Testing Decisions

### TD-001: Test Coverage
**Module**: string_utils
**What to test**: capitalize function with various inputs (normal, empty, single char, already capitalized)
**Prior art**: Standard pytest test patterns

---

## Out of Scope

- External API changes
- Database changes
- UI changes

---

## Further Notes

This PRD is for validation of the TDD pipeline integration from seed 002.