# AI Workflow — User Guide

A structured pipeline for turning an idea into implemented, tested code — with a clarification stage that makes sure the AI actually understands what you want before anything gets built.

## The Big Picture

```
Your idea  →  /dive  →  /product  →  /tasks  →  /exec
 (feed.md)   (interview)  (PRD)      (task list)  (implementation)
```

Each stage builds on the previous one. You start by writing down what you want in plain language, and the pipeline progressively turns that into a clarified spec, then a product document, then a list of small tasks, then real code with tests.

You don't need to do all of this in one sitting. Each stage saves its output to a file, so you can stop and come back later.

---

## Getting Started

Every piece of work lives in a numbered **session** folder under `.feed/`, like `.feed/001/`. Each session is one self-contained thing you're building or fixing — pick a new number for each new idea.

To start a session, create:

```
.feed/001/feed.md
```

and write down what you want, in your own words. It doesn't need to be polished. A rough idea, a bug description, a feature request — anything that captures your intent is enough. The next stage exists specifically to help sharpen it.

**Example:**

```markdown
I want to add a way for admins to create promo codes. Each code should
have an expiry date and a usage limit. Not sure yet about the tech stack.
```

---

## The Stages

### 1. `/dive` — Clarify

```
/dive 001
```

This stage interviews you. It reads your `feed.md`, asks questions about anything ambiguous, challenges assumptions, and won't stop until both you and the AI are on the same page about exactly what's being built.

Expect it to:
- Ask a few questions at a time (not a giant wall of questions all at once)
- Explain *why* each question matters
- Offer concrete options with a recommendation, rather than open-ended "what do you want?" prompts
- Push back if something you said doesn't hold up, or if the conversation starts drifting off-topic
- Explore your codebase on its own when relevant, and just confirm with you what it found

Nothing is implemented at this stage — it's purely conversation. When everything is resolved, it writes a clarified summary to `context.md` and you move on.

**If something needs rethinking later:** you don't have to redo the whole interview. Use:

```
/dive 001 --focus <area>
```

to reopen just one part of the conversation — for example `/dive 001 --focus error-handling`. This only touches that one part of `context.md`; everything else you already agreed on stays untouched. If `/product` or `/tasks` already ran using the old understanding, you'll be told that they're now out of date and need a re-run.

---

### 2. `/product` — Define

```
/product 001
```

This stage takes the clarified understanding from `/dive` and turns it into a proper product requirements document — goals, scope, and detailed user stories with acceptance criteria. Think of it as a product manager writing the spec.

It mostly works on its own. For very minor details that don't affect what's being built (like an unspecified naming convention), it'll just make a sensible choice and note the assumption. If it hits something that genuinely needs a decision — something that would affect scope or behavior — it'll stop and tell you, and point you back to `/dive --focus` to resolve it.

Output: `product.md`.

`--focus` works here too, the same way it does for `/dive`, if you need to revise one part of the product document later.

---

### 3. `/tasks` — Break Down

```
/tasks 001
```

This stage takes `product.md` and breaks it into small, individually checkable tasks — written as separate files under `.feed/001/tasks/`. Each task is scoped to be as small as it reasonably can be while still being something you can verify works or doesn't, on its own.

Tasks that depend on each other are linked, so you (or `/exec`) know what order to work through them in.

Output: a set of files like `.feed/001/tasks/001_task.md`, `.feed/001/tasks/002_task.md`, etc.

---

### 4. `/exec` — Implement

```
/exec .feed/001/tasks/001_task.md
```

This is the only stage that actually touches your codebase. Give it one task file at a time, and it implements that task — writing both the code and the tests for it. A task isn't considered finished until its tests exist and pass.

If a task depends on another task that isn't done yet, `/exec` will tell you and won't proceed until that dependency is resolved.

Run `/exec` once per task, working through your task list in order.

---

## Typical Flow

```
1. Write .feed/001/feed.md describing what you want
2. /dive 001              → answer its questions until it says you're aligned
3. /product 001            → review product.md
4. /tasks 001               → review the generated tasks
5. /exec .feed/001/tasks/001_task.md
   /exec .feed/001/tasks/002_task.md
   ... (one per task, in dependency order)
```

## A Few Things Worth Knowing

- **You can stop and resume anytime.** Progress is tracked automatically, so closing a session mid-interview or mid-implementation doesn't lose your place.
- **Nothing gets implemented until `/exec`.** `/dive`, `/product`, and `/tasks` only ever read and write planning documents — your code is untouched until the final stage.
- **`/dive` will challenge you.** If something you ask for seems underspecified or risky, expect pushback and follow-up questions rather than the AI just going along with it.
- **Revising something doesn't mean starting over.** Use `/dive 001 --focus <area>` to fix one part of the understanding without redoing the whole conversation. You'll be told if anything downstream needs to be re-run as a result.
- **One task, one `/exec` call.** This keeps each implementation step small, reviewable, and easy to verify before moving to the next.