# /dive Skill

## References

Before doing anything else, read and internalize these documents in full:

- `.claude/reference/structure.md` — the directory layout, file roles, and naming conventions of the workflow
- `.claude/reference/pipeline.md` — the overall pipeline flow and how stages connect
- `.claude/reference/state.md` — the state.json schema and update rules

Do not proceed until you have read all three.

---

## Purpose

`/dive` is the first skill in the pipeline. It reads the human-written `feed.md` for a session and conducts a relentless, structured interview with the human until every ambiguity is resolved, every assumption is challenged, and both the agent and the human are fully aligned on what is being built and why.

The output is `context.md` — a complete, unambiguous document that the `/product` skill can work from without needing to ask anything further.

No code changes. No implementation. Clarification only.

---

## Invocation

```
/dive <session>
/dive <session> --focus <area>
```

| Argument | Description |
|---|---|
| `<session>` | A session identifier (`001`) or a path alias (`@.feed/001/feed.md`) |
| `--focus <area>` | Optional. Re-opens an already-resolved session to re-interview the human about one specific area, leaving the rest of `context.md` untouched. |

### Resolving the input

1. Resolve `<session>` to a session directory (e.g. `.feed/001/`)
2. Confirm `feed.md` exists at `.feed/001/feed.md`
3. Confirm `feed.md` is not empty

### Error handling

| Condition | Response |
|---|---|
| Session identifier does not exist | Tell the human the session directory `.feed/NNN/` was not found. List existing sessions if any. |
| `feed.md` does not exist | Tell the human no `feed.md` was found at the expected path. Ask them to create it and re-run. |
| `feed.md` exists but is empty | Tell the human `feed.md` is empty. Ask them to write their intent into it and re-run. |
| `feed.md` is too vague to extract any intent | Do not abort. Proceed with the interview — this is exactly what the interview is for. Start with broad, open questions. |
| `--focus <area>` given but `context.md` does not exist yet | Ignore `--focus`; run a normal full first-pass interview instead. |

---

## Two Modes

### Mode A — First pass (no `--focus`)

Runs the full interview from scratch against `feed.md`. Used when `context.md` does not exist yet.

### Mode B — Focused re-entry (`--focus <area>`)

Used when `context.md` already exists and the human (or a later stage) has identified that one specific area needs to be revisited — for example, a new edge case surfaced during `/product` or `/tasks`, or the human simply wants to rethink one part of the decision.

Behavior:

1. Read the existing `context.md` in full for surrounding context.
2. Run a short, targeted interview scoped **only** to `<area>`. Do not re-litigate sections that are not related.
3. Apply a **surgical patch** to `context.md` — update only the section(s) affected by `<area>`. Leave everything else byte-for-byte as it was.
4. Check whether `/product` and/or `/tasks` have already run for this session (via `state.json`). If either has, mark those stages `stale` per the rules in `.claude/reference/state.md`. Tell the human explicitly which downstream stages are now stale and will need to be re-run.

`--focus` can be invoked at any point in the pipeline's lifetime, not just before `/product` runs.

---

## State

`/dive` follows the general stage lifecycle defined in `.claude/reference/state.md` (see "Stage Lifecycle"), applied to the `dive` stage:

- **Mode A** is a first run: state.md's "first run" rules apply.
- **Mode B** is a re-entry: state.md's "re-entry" rules apply — `started_at` is preserved from the original run, and on completion, any of `product` / `tasks` that had already completed are marked `stale`.

No `/dive`-specific state handling exists beyond what `state.md` already defines.

---

## Codebase Exploration

If `feed.md` describes something tied to an existing codebase (a bug fix, a new feature on an existing system, a refactor), explore the codebase automatically on the first pass before asking the human anything:

- Locate relevant files, modules, or entry points
- Read them to understand the current state
- Use this to inform your questions and pre-answer what you already know

Explore freely — do not ask permission before reading code. When a finding is used to shape a question or settle a point, confirm it with the human before treating it as final:

> "I found `FooService.java` which appears to handle X. It currently does Y. I'll assume this is the right place unless you tell me otherwise — does that sound right?"

This is a read-only stage. No file in the codebase is ever modified by `/dive`.

---

## Interview Process

### First pass

Read `feed.md` in full (and the relevant section of `context.md` in Mode B). Identify:

- What the human is trying to achieve (the goal)
- What is known vs. what is ambiguous
- What assumptions the human is making, explicitly or implicitly
- What is missing that the next stage will need

If the input is codebase-related, explore the relevant code first, then start the interview informed by both the document and what you found.

### Questioning rules

- Ask a maximum of **3 questions per round**. This is a pacing limit on a single round, not a cap on the interview as a whole — keep running rounds for as many rounds as it takes until the completeness check below passes. There is no limit on total questions asked across the whole interview.
- Group related questions together in the same round.
- Every question must include:
  - **Why it matters** — what downstream impact it has on implementation or design
  - **Options** — 2–4 concrete choices that fit the problem, not open-ended prompts
  - **A recommended answer** — your best read given what you know, stated clearly
- Challenge assumptions hard. If the human's input implies something that may not be the right approach, say so directly and explain why.
- Do not let the conversation drift. If an answer opens a tangent unrelated to the current session (or, in Mode B, unrelated to `<area>`), acknowledge it and redirect.
- Do not accept vague answers. If an answer is unclear or incomplete, ask again with a tighter framing.

### Assumption challenging

When the human states or implies an assumption, surface it explicitly:

> "You mentioned X. I want to challenge that — if we do X, it means Y and Z downstream. Are you sure X is the right call, or would [alternative] serve you better here?"

Do not accept the first answer if it is underspecified. Push until the answer is concrete and its implications are understood by both sides.

### Completeness check

Before ending the interview, run through this checklist internally. Do not move to writing `context.md` until every item that applies to this session (or, in Mode B, to `<area>`) is resolved:

- [ ] The goal is stated unambiguously
- [ ] The scope boundary is clear (what is in, what is out)
- [ ] All entities, concepts, or domain terms are defined
- [ ] All constraints are known (technical, business, time, resource)
- [ ] All dependencies on existing systems or code are identified
- [ ] All edge cases that affect design decisions are resolved
- [ ] The success criteria are concrete and testable
- [ ] No open question remains that would block the next stage

### Alignment signal

When you believe all questions are resolved, state it explicitly:

> "I think we have everything we need. Here is what I understand we have agreed on: [brief bullet summary]. Does anything need to change before I write `context.md`?"

Wait for the human to confirm, correct, or add. Only write `context.md` after explicit confirmation.

---

## Output: `context.md`

Write to `.feed/NNN/context.md`.

This document is the authoritative input for `/product`. Write it for an AI agent, not a human — be explicit, structured, and leave nothing implicit. Every decision made during the interview must appear here.

### Format

```markdown
# Context — [session identifier]

## Goal
One or two sentences. What is being built and why.

## Scope
### In scope
- ...

### Out of scope
- ...

## Domain & Definitions
Key terms, entities, and concepts used in this session and what they mean.
- Term: definition

## Requirements
Numbered list. Each requirement is concrete and testable.
1. ...
2. ...

## Constraints
Technical, business, or other constraints that must be respected.
- ...

## Dependencies
Existing systems, services, files, or modules this work touches or depends on.
- ...

## Edge Cases & Decisions
Specific cases that were discussed and the decision reached for each.
- Case: decision and rationale

## Success Criteria
How we know this is done. Concrete and verifiable.
- ...

## Open Items
Anything explicitly deferred, out of scope for now, or to be revisited later.
- ...
```

All sections must be present. If a section has nothing to say, write `None.` — do not omit the section.

### Mode B writes

In Mode B, only the section(s) relevant to `<area>` are rewritten. All other sections are copied through unchanged from the previous version of `context.md`.

---

## Examples

### Example 1 — Challenging a vague goal

**feed.md:**
```
I want to add caching to the foo service.
```

**Bad /dive response:**
> "What kind of caching do you want?"

**Good /dive response:**
> **Why this matters:** The type and layer of caching determines where it lives in the stack, what invalidation strategy we need, and how it affects consistency guarantees.
>
> I explored the codebase and found `FooService` at `src/foo/FooService.java`. It currently calls `FooRepository.findById()` on every request with no caching. I'll assume this is what you want to cache — correct me if not.
>
> **Question 1 — What problem are we solving with caching?**
> - A) Response times are too slow for end users
> - B) Database is under too much load
> - C) Both A and B
> - D) Something else
>
> → My recommendation: **B or C** — without knowing the traffic pattern, I'd assume DB load is the driver. Tell me if latency is the primary concern.
>
> **Question 2 — What is the acceptable staleness of cached data?**
> - A) Always fresh — cache must invalidate on every write
> - B) Eventually consistent — a few seconds of stale data is fine
> - C) Long-lived — data changes rarely, TTL of minutes or hours is acceptable
>
> → My recommendation: **A** — given that this is a backoffice service, stale data is likely unacceptable. But confirm.
>
> **Question 3 — Where should the cache live?**
> - A) In-process (JVM heap, e.g. Caffeine)
> - B) Distributed (e.g. Redis)
> - C) No preference — recommend based on existing infrastructure
>
> → My recommendation: **C** — I'll check what's already in the stack before suggesting.

---

### Example 2 — Catching a missing constraint

**feed.md:**
```
Build a task assignment feature. Managers can assign tasks to team members.
```

**Good /dive question:**
> **Why this matters:** Without knowing how many tasks and team members exist, we may over- or under-engineer the assignment UI and backend query patterns.
>
> **Question — What is the scale we're designing for?**
> - A) Small team, <20 members, <100 tasks at a time
> - B) Medium team, 20–100 members, hundreds of tasks
> - C) Large org, 100+ members, thousands of tasks
> - D) Unknown — design for extensibility
>
> → My recommendation: **A or D** — start with the simplest model and note where it would need to change if scale grows.

---

### Example 3 — Blocking a drifting conversation

**Human mid-interview:**
> "While we're at it, maybe we should also redesign the whole notification system."

**Good /dive response:**
> That's worth discussing separately — it's a different session. I've noted it so you can open a new feed for it. For now let's stay focused on [original goal]. Back to the open question: [restate the last unanswered question].

---

### Example 4 — Confirming a codebase finding

> I found `FooConfig.java` at `src/config/FooConfig.java`. It currently sets a global timeout of 30s for all outbound calls. If your feed implies we're adding a new outbound call, this timeout will apply unless we override it.
>
> I'll assume 30s is acceptable for the new call unless you tell me otherwise. Does that sound right, or do we need a different timeout for this specific case?

---

### Example 5 — Using `--focus`

**Invocation:**
```
/dive 001 --focus error-handling
```

**Context:** `context.md` already exists and `/tasks` has already run. The human realized the error-handling behavior for the foo creation endpoint was never actually pinned down.

**Good /dive response:**
> Re-opening session `001`, focused on **error-handling**. I'll only touch this part of `context.md` — everything else stays as-is.
>
> I checked the current `context.md`. The `Edge Cases & Decisions` section doesn't mention what happens when foo creation fails validation. Looking at `FooController.java`, I see other endpoints return a structured error body with a `code` and `message` field.
>
> **Question — What should happen when foo creation fails validation?**
> - A) Return 400 with the existing structured error format (`code` + `message`), matching other endpoints
> - B) Return 400 with a simple string message
> - C) Something else
>
> → My recommendation: **A** — for consistency with the rest of the API.
>
> [After confirmation] — Updating `context.md`'s Edge Cases & Decisions section now. Heads up: `product.md` and the task files in `tasks/` were generated before this was resolved, so I'm marking both `product` and `tasks` as **stale** in `state.json`. You'll want to re-run `/product` and `/tasks` to pick up this change.