# /product Skill

## References

Before doing anything else, read and internalize these documents in full:

- `.claude/reference/structure.md` — the directory layout, file roles, and naming conventions of the workflow
- `.claude/reference/pipeline.md` — the overall pipeline flow and how stages connect
- `.claude/reference/state.md` — the state.json schema and update rules, including the generic stage lifecycle

Do not proceed until you have read all three.

---

## Purpose

`/product` is the second skill in the pipeline. It reads `context.md` — the fully resolved understanding produced by `/dive` — and acts as a technical product manager, turning that understanding into a structured product requirements document with detailed user stories.

The output is `product.md`, the authoritative input for `/tasks`.

No code changes. No implementation. This stage defines *what* is being built and *why* it satisfies the resolved context — not how it will be built.

---

## Role

Act as a technical product manager. This means:

- Translate resolved context into concrete scope, not vague aspiration.
- Write user stories that a technical team could pick up and build from without further interpretation.
- Think in terms of value and verification: every story exists because it serves the goal in `context.md`, and every story has acceptance criteria that prove it was done correctly.
- Do not introduce new product decisions that `context.md` didn't settle. Your job is to structure and elaborate the resolved understanding, not to make new judgment calls about what should be built.

---

## Invocation

```
/product <session>
/product <session> --focus <area>
```

| Argument | Description |
|---|---|
| `<session>` | A session identifier (`001`) or a path alias (`@.feed/001/context.md`) |
| `--focus <area>` | Optional. Re-opens an already-written `product.md` to revisit one specific area, leaving the rest untouched. |

### Resolving the input

1. Resolve `<session>` to a session directory (e.g. `.feed/001/`)
2. Confirm `context.md` exists at `.feed/001/context.md`
3. Confirm `context.md` is not empty
4. Check `state.json` — confirm `stages.dive.status` is `completed`. If it is `stale`, warn the human that `context.md` may be out of date relative to a more recent `/dive` re-entry elsewhere, and confirm whether to proceed anyway.

### Error handling

| Condition | Response |
|---|---|
| Session identifier does not exist | Tell the human the session directory `.feed/NNN/` was not found. |
| `context.md` does not exist | Tell the human `/dive` has not been run yet (or did not complete) for this session. Recommend running `/dive <session>` first. Do not proceed. |
| `context.md` exists but is empty | Treat as equivalent to missing. Recommend re-running `/dive`. |
| `stages.dive.status` is `not_started` or `in_progress` | Tell the human `/dive` has not finished. Do not proceed. |
| `stages.dive.status` is `stale` | Warn the human and ask for confirmation before proceeding, since `context.md` may not reflect the latest resolved understanding. |
| `--focus <area>` given but `product.md` does not exist yet | Ignore `--focus`; run a normal full first-pass instead. |

---

## Two Modes

### Mode A — First pass (no `--focus`)

Reads `context.md` in full and produces a complete `product.md` from scratch.

### Mode B — Focused re-entry (`--focus <area>`)

Used when `product.md` already exists and one specific area needs revisiting — for example, a new user story, a scope change, or a story whose acceptance criteria turned out to be wrong.

Behavior:

1. Read the existing `product.md` in full for surrounding context.
2. Resolve only what's needed for `<area>` — using the same trivial/real gap rule below.
3. Apply a surgical patch to `product.md` — update only the section(s) or story(ies) affected by `<area>`. Leave everything else unchanged.
4. Check `state.json` for whether `/tasks` has already completed. If so, mark it `stale` per `.claude/reference/state.md`. Tell the human explicitly that `/tasks` will need to be re-run.

---

## Handling Gaps in `context.md`

`context.md` should already be fully resolved by the time `/product` runs. Occasionally a gap still surfaces during the PRD-writing process. When this happens, classify the gap before acting on it:

### Trivial gap — resolve directly

A gap is trivial if it has **no impact on scope or architecture** — it's a naming choice, a formatting detail, an ordering preference, or similarly cosmetic. For trivial gaps:

- Pick a sensible default.
- State the assumption inline in `product.md` where relevant (e.g. "Assumption: endpoint named `POST /foos`, no constraint specified in context").
- Do not interrupt the human with a question for this.
- You may ask a single quick confirming question if you want one, but it should never block progress.

### Real gap — send back to `/dive`

A gap is real if resolving it would affect scope, behavior, data model, user-facing flow, or any architectural decision. For real gaps:

- Do not guess. Do not invent a product decision `context.md` didn't make.
- Stop and tell the human explicitly what is missing and why it matters for the PRD.
- Recommend running `/dive <session> --focus <area>` to resolve it, naming the specific area.
- Do not write `product.md` (or the affected portion of it, in Mode B) until the gap is resolved and `context.md` is updated.

---

## State

`/product` follows the general stage lifecycle defined in `.claude/reference/state.md` ("Stage Lifecycle"), applied to the `product` stage:

- **Mode A** is a first run: the "first run" rules apply.
- **Mode B** is a re-entry: the "re-entry" rules apply — `started_at` is preserved from the original run, and on completion, `tasks` is marked `stale` if it had already completed.

No `/product`-specific state handling exists beyond what `state.md` already defines.

---

## Output: `product.md`

Write to `.feed/NNN/product.md`.

This document is the authoritative input for `/tasks`. Write it for an AI agent, not a human — be explicit and structured. Every user story must be independently understandable without re-reading `context.md`.

### Format

```markdown
# Product — [session identifier]

## Goal
One or two sentences. What is being built and why. (Derived from context.md's Goal.)

## Scope
### In scope
- ...

### Out of scope
- ...

## User Stories

### US-1: [Short title]
**As a** [role]
**I want** [goal]
**So that** [benefit]

**Acceptance Criteria:**
- [ ] ...
- [ ] ...

**Notes:** (assumptions made, edge cases this story covers, dependencies on other stories — omit if none)

### US-2: [Short title]
...

## Non-Functional Requirements
Performance, security, scalability, observability, or other cross-cutting requirements that apply across stories.
- ...

## Assumptions
Any assumption made by /product while writing this document (including resolved trivial gaps).
- ...

## Out of Scope / Deferred
Carried over from context.md's Open Items, plus anything explicitly deferred during this stage.
- ...
```

All sections must be present. If a section has nothing to say, write `None.` — do not omit the section.

### User story rules

- Every story uses the As a / I want / So that format exactly.
- Every story has at least one acceptance criterion, written as a checkbox so it can be tracked.
- Acceptance criteria are concrete and verifiable — "the system returns X when Y" not "the system works correctly."
- Stories are independently sized where possible — small enough that `/tasks` can map each into one or a few tasks, large enough to represent real user value.
- Number stories sequentially (`US-1`, `US-2`, ...) and never renumber existing stories, even during a Mode B patch — new stories from a focused re-entry get the next available number.

### Mode B writes

In Mode B, only the section(s) or story(ies) relevant to `<area>` are rewritten. Everything else is copied through unchanged from the previous version of `product.md`, including story numbering.

---

## Examples

### Example 1 — Turning context into a user story

**Relevant excerpt from context.md:**
```
## Requirements
1. Admin can create a promo code with a code string, expiry date, and usage limit.
2. A promo code cannot be redeemed after its expiry date.
```

**Good /product output:**
```markdown
### US-1: Admin creates a promo code
**As an** admin
**I want** to create a promo code with a code string, expiry date, and usage limit
**So that** I can run time-bound promotions with controlled usage

**Acceptance Criteria:**
- [ ] POST /promo-codes accepts code, expiresAt, and usageLimit
- [ ] A created code defaults to active
- [ ] Creating a code with a duplicate code string returns 409

### US-2: Expired codes cannot be redeemed
**As a** system
**I want** to reject redemption attempts for expired codes
**So that** promotions don't run past their intended window

**Acceptance Criteria:**
- [ ] Redemption checks expiresAt against the current time
- [ ] An expired code returns a 4xx with a clear reason
- [ ] Expiry is checked at redemption time, not at creation time

**Notes:** Depends on US-1 existing (a code must be creatable before it can be redeemed).
```

---

### Example 2 — Resolving a trivial gap directly

**Situation:** `context.md` doesn't specify the exact endpoint path for listing promo codes.

**Good /product behavior:**
> (No question asked.) In `product.md`:
> ```
> ## Assumptions
> - Listing endpoint assumed to be `GET /promo-codes`, following REST convention; not specified in context.md.
> ```

---

### Example 3 — Escalating a real gap

**Situation:** `context.md` defines that promo codes have a usage limit, but never resolves what happens when a code's usage limit is reached mid-redemption under concurrent requests.

**Good /product response:**
> I can't write the acceptance criteria for the redemption story without knowing the concurrency behavior here — this affects the data model and possibly requires a locking strategy, which is an architectural decision, not something I should assume.
>
> **Gap:** `context.md` doesn't resolve what happens when two redemptions race against the last unit of a promo code's usage limit.
>
> Recommend running:
> ```
> /dive 001 --focus concurrent-redemption-limits
> ```
> I'll hold off on writing the redemption-related stories until this is resolved. The rest of `product.md` is unaffected and I'll proceed with it now.

---

### Example 4 — Using `--focus`

**Invocation:**
```
/product 001 --focus usage-limit-story
```

**Context:** `product.md` already exists. `/dive` was re-entered earlier with `--focus concurrent-redemption-limits` and resolved the gap from Example 3. `context.md` now has the answer; `/tasks` has not yet run.

**Good /product response:**
> Re-opening session `001`, focused on the usage-limit story. I'll only touch this part of `product.md`.
>
> `context.md` now specifies that redemption uses a row-level lock to prevent overuse. Writing `US-3` to cover this:
>
> ```markdown
> ### US-3: Concurrent redemptions cannot exceed usage limit
> **As a** system
> **I want** to serialize redemption attempts against a code's remaining usage
> **So that** concurrent requests can never push usage past the configured limit
>
> **Acceptance Criteria:**
> - [ ] Redemption acquires a row-level lock on the promo code before checking usage
> - [ ] When usage is exhausted, all but the winning concurrent request receive a 4xx
> - [ ] No two redemptions can be counted against the same final unit of usage
> ```
>
> `state.json` did not have `tasks` marked completed yet, so nothing downstream needs to be flagged stale.