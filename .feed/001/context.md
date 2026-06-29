# Context — 001

## Goal
Pipeline infrastructure test. Validate that /dive reads feed.md and produces a structurally valid context.md.

## Scope
### In scope
- /dive → context.md pipeline flow

### Out of scope
- /product, /tasks, /exec
- Any real feature

## Domain & Definitions
None.

## Requirements
1. context.md is written to .feed/001/context.md
2. context.md contains all required sections per the /dive skill format

## Constraints
None.

## Dependencies
None.

## Edge Cases & Decisions
- This is a pure infrastructure test with no real feature intent

## Success Criteria
- context.md exists at .feed/001/context.md
- context.md contains all required sections

## Open Items
None.
