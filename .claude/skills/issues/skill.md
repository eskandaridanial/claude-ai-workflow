# Role

`issues` breaks a PRD into independently-workable tasks and writes each as a local markdown file. It decomposes the PRD produced by `prd` into vertical slices (tracer bullets) that `ralph` can implement.

`issues` is the third stage in the `grill` → `prd` → `issues` → `ralph` pipeline.

---

## Input Contract

`issues` receives:
- The PRD file (`.seed/{number}/prd.md`) produced by `prd`
- The requirements file (`.seed/{number}/requirements.md`) from `grill` for additional context

**Expected input state:**
- `prd.md` exists and contains a valid PRD with user stories and implementation decisions
- The file is readable and has all required sections

**Example of good input:**
A PRD with:
- Problem Statement
- Solution
- User Stories (numbered list)
- Implementation Decisions
- Testing Decisions
- Out of Scope
- Further Notes

---

## Instructions

### 1. Resolve the seed number

Resolve the seed number:
1. If a prior `prd` session has already established the seed number, use it.
2. Otherwise, list `.seed/` and find the directory that matches the current story.
3. If still unknown, ask the user: _"Which seed number should I break down? (e.g. 001)"_

All output goes to `.seed/{number}/tasks/`.

### 2. Read the PRD

Read `.seed/{number}/prd.md` to understand the full requirements.

Also read `.seed/{number}/requirements.md` for additional context.

### 3. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code.

### 4. Draft vertical slices

Break the PRD into **tracer bullet** tasks. Each task is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK':
- **HITL** (Human-In-The-Loop): requires human interaction — an architectural decision, design review, or ambiguous requirement
- **AFK** (Autonomous): can be implemented and merged without human interaction

<vertical-slice_rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice_rules>

### 5. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:
- **Title**: short descriptive name
- **Type**: HITL / AFK
- **Blocked by**: which other tasks (if any) must complete first
- **User stories covered**: which user stories from the PRD this addresses

Ask the user:
- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the correct slices marked as HITL and AFK?

Iterate until the user approves the breakdown.

### 6. Create the task files

For each approved slice, write a markdown file in `.seed/{number}/tasks/` using the naming pattern `NNN-short-title.md` (e.g. `001-add-oauth2-providers.md`).

Number tasks starting from `001`, incrementing by one. Check what files already exist in `.seed/{number}/tasks/` and continue from the next available number. Zero-pad to 3 digits.

Create files in dependency order (blockers first) so you can reference real filenames in the "Blocked by" field.

Do NOT use `gh issue create` or any GitHub CLI commands. Do NOT reference GitHub issue numbers. Use local filenames for all cross-references.

---

## Output Contract

`issues` writes to `.seed/{number}/tasks/NNN-*.md` (one file per task).

**Output schema for each task file:**
```json
{
  "seed": "001",
  "stage": "issues",
  "task_id": "001",
  "filename": "001-add-oauth2-providers.md",
  "parent_prd": ".seed/001/prd.md",
  "title": "Add OAuth2 providers",
  "type": "AFK | HITL",
  "what_to_build": "A concise description of this vertical slice",
  "acceptance_criteria": [
    "Criterion 1",
    "Criterion 2",
    "Criterion 3"
  ],
  "blocked_by": [".seed/001/tasks/000-placeholder.md"] | [],
  "user_stories_addressed": ["US-001", "US-002"]
}
```

**Markdown task file format:**
```markdown
## Parent PRD
`.seed/{number}/prd.md`

## What to build
A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

## Acceptance criteria
- [ ] Criterion 1 [AFK]
- [ ] Criterion 2 [HITL: requires design review]
- [ ] Criterion 3 [AFK]

## Blocked by
- `.seed/{number}/tasks/NNN-title.md` (if any)
Or "None - can start immediately" if no blockers.

## User stories addressed
- US-001
- US-002
```

**AFK/HITL Marker Syntax:**

Each acceptance criterion must be marked as AFK (autonomous) or HITL (Human-In-The-Loop):

- `[AFK]` — Can be implemented and verified without human interaction. TDD runs on this criterion.
- `[HITL: reason]` — Requires a human decision before implementation. TDD pauses at this boundary. Include a brief reason: `requires design review`, `ambiguous requirement`, `needs external approval`, etc.

**Examples:**
```
- [ ] User can login with OAuth2 [AFK]
- [ ] Design the OAuth2 provider selection UI [HITL: requires design review]
- [ ] Add logout functionality [AFK]
- [ ] Handle session timeout behavior [HITL: ambiguous requirement]
```

**Rules:**
- Every acceptance criterion must have either `[AFK]` or `[HITL: ...]` marker
- If a criterion is HITL, include a brief reason after the colon
- AFK criteria are processed first by `ralph` using TDD
- HITL criteria halt `ralph`, which surfaces the decision to the user before proceeding

---

## Failure Behavior

### Missing input

**Trigger:** `prd.md` does not exist.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "issues",
  "summary": "Missing prd.md from prd",
  "warnings": [],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe what was missing, halt. Do not proceed.

### Malformed input

**Trigger:** `prd.md` exists but is not readable as markdown, or is missing required sections (user stories, implementation decisions).

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "issues",
  "summary": "Malformed prd.md",
  "warnings": ["Missing required section: User Stories"],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe the parse error or missing sections, halt. Do not proceed.

### Incomplete input

**Trigger:** `prd.md` is readable but lacks sufficient detail to create meaningful tasks (e.g., no user stories, no implementation decisions).

**Handling:**
```json
{
  "status": "PARTIAL",
  "stage": "issues",
  "summary": "PRD lacks sufficient detail for task decomposition",
  "warnings": [
    "No implementation decisions found",
    "User stories are too vague"
  ],
  "confidence": 0.3
}
```
Emit `status: PARTIAL` with warnings. Ask the user for clarification before proceeding. If gaps cannot be filled, halt with `status: FAILED`.

---

## Status Block

After completing the task breakdown, emit a trailing Status Block:

```json
{
  "status": "SUCCESS | PARTIAL | FAILED",
  "stage": "issues",
  "summary": "One-line description of what was accomplished",
  "warnings": ["Any caveats or concerns"],
  "confidence": 0.0-1.0
}
```
