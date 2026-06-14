# Pipeline Contract

This document is the source of truth for the `grill` → `prd` → `issues` → `ralph` pipeline. It defines the end-to-end handoff schema, shared conventions, and inter-agent compatibility rules.

---

## Pipeline Stages

| Stage | Skill | Input | Output |
|-------|-------|-------|--------|
| 1 | `grill` | `.seed/{number}/init.md` | `.seed/{number}/requirements.md` |
| 2 | `prd` | `.seed/{number}/requirements.md` | `.seed/{number}/prd.md` |
| 3 | `issues` | `.seed/{number}/prd.md` | `.seed/{number}/tasks/NNN-*.md` |
| 4 | `ralph` | `.seed/{number}/tasks/NNN-*.md` | Implementation + updated task file |

### Auxiliary Skills

| Skill | Role |
|-------|------|
| `tdd` | Test-driven development with red-green-refactor loop |

---

## File Handoff Schema

### Stage 1: `grill` output → `requirements.md`

```json
{
  "seed": "001",
  "stage": "grill",
  "problem_statement": "string",
  "solution": "string",
  "requirements": [
    {
      "id": "REQ-001",
      "category": "functional | non-functional | constraint",
      "description": "string",
      "priority": "must-have | should-have | nice-to-have",
      "notes": "string"
    }
  ],
  "user_stories": [
    {
      "id": "US-001",
      "actor": "string",
      "feature": "string",
      "benefit": "string"
    }
  ],
  "assumptions": ["string"],
  "open_questions": ["string"]
}
```

### Stage 2: `prd` output → `prd.md`

```json
{
  "seed": "001",
  "stage": "prd",
  "problem_statement": "string",
  "solution": "string",
  "user_stories": [
    {
      "id": "US-001",
      "as_an": "string",
      "i_want": "string",
      "so_that": "string"
    }
  ],
  "implementation_decisions": [
    {
      "id": "ID-001",
      "decision": "string",
      "rationale": "string",
      "tradeoffs": "string"
    }
  ],
  "testing_decisions": [
    {
      "module": "string",
      "what_to_test": "string",
      "prior_art": "string"
    }
  ],
  "out_of_scope": ["string"],
  "further_notes": "string"
}
```

### Stage 3: `issues` output → `tasks/NNN-*.md`

```json
{
  "seed": "001",
  "stage": "issues",
  "task_id": "001",
  "filename": "001-add-oauth2-providers.md",
  "parent_prd": ".seed/001/prd.md",
  "title": "string",
  "type": "AFK | HITL",
  "what_to_build": "string",
  "acceptance_criteria": ["string"],
  "blocked_by": [".seed/001/tasks/000-title.md"] | [],
  "user_stories_addressed": ["US-001", "US-002"]
}
```

### Stage 4: `ralph` output → updated task file

```json
{
  "seed": "001",
  "stage": "ralph",
  "task_id": "001",
  "filename": "001-add-oauth2-providers.md",
  "status": "completed | blocked | halted",
  "acceptance_criteria_completed": ["AC-001", "AC-002"],
  "acceptance_criteria_skipped": [],
  "decisions": [
    {
      "decision": "string",
      "rationale": "string"
    }
  ],
  "new_scope": ["string"]
}
```

---

## Status Block Schema

Every agent must emit a trailing Status Block as a fenced JSON code block.

```json
{
  "status": "SUCCESS | FAILED | PARTIAL",
  "stage": "grill | prd | issues | ralph",
  "summary": "One-line description of what was accomplished",
  "warnings": ["warning 1", "warning 2"],
  "confidence": 0.0-1.0
}
```

### Status Values

| Value | Meaning |
|-------|---------|
| `SUCCESS` | Completed, output is valid and ready for downstream |
| `FAILED` | Halted, no useful output, downstream should not proceed |
| `PARTIAL` | Completed but with warnings, output is usable but compromised |

### Confidence Threshold Bands

| Confidence | Behavior |
|------------|----------|
| `≥ 0.7` | Proceed autonomously (AFK) |
| `0.4 – 0.7` | Proceed with warnings surfaced to user |
| `< 0.4` | Halt and surface to user before proceeding |

---

## Failure Modes

All failure modes are organized by input type. Each skill must handle:

### Missing Input

**Trigger:** Required input file does not exist.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "skill-name",
  "summary": "Missing required input file",
  "warnings": [],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe what was missing, halt. Do not proceed.

### Malformed Input

**Trigger:** Input file exists but is not parseable or is missing required fields.

**Handling:**
```json
{
  "status": "FAILED",
  "stage": "skill-name",
  "summary": "Malformed input file",
  "warnings": ["Description of parse error or missing fields"],
  "confidence": 0.0
}
```
Emit `status: FAILED`, describe the parse error, halt. Do not proceed.

### Incomplete Input

**Trigger:** Input file is parseable but lacks sufficient detail.

**Handling:**
```json
{
  "status": "PARTIAL",
  "stage": "skill-name",
  "summary": "Insufficient input for full processing",
  "warnings": ["List of gaps or concerns"],
  "confidence": 0.0-0.5
}
```
Emit `status: PARTIAL` with warnings. Proceed with caution or halt and surface to user.

---

## Skill File Structure

All skill files share this 6-section structure:

1. **Role** — Who the agent is and what it does
2. **Input Contract** — What it expects from upstream (prose + example)
3. **Instructions** — How to do the work
4. **Output Contract** — What it guarantees downstream (fenced JSON code block)
5. **Failure Behavior** — Failure modes by input type: Missing / Malformed / Incomplete
6. **Status Block** — Trailing fenced JSON code block

---

## Agent Reference Convention

Agents reference each other via inline code spans:

- `grill`
- `prd`
- `issues`
- `ralph`
- `tdd`

---

## Task Types

| Type | Meaning |
|------|---------|
| `AFK` (Autonomous) | Can be implemented and merged without human interaction |
| `HITL` (Human-In-The-Loop) | Requires human interaction — architectural decision, design review, or ambiguous requirement |

---

## Naming Conventions

- Seed directories: `.seed/{number}/` with zero-padded 3-digit numbers (`001`, `002`, etc.)
- Task files: `NNN-short-title.md` (e.g., `001-add-oauth2-providers.md`)
- Skill files: `skill.md` in `.claude/skills/{skill-name}/`
- Pipeline contract: `.claude/contract/pipeline.md`
