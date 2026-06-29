# Pipeline Reference

> This document describes the overall flow of the AI workflow — what stages exist, what order they run in, and how they connect to each other.  
> It does not describe the internal details of any skill. Each skill has its own reference document.

---

## Overview

The pipeline is a linear sequence of stages. Each stage takes the output of the previous one as its input and produces a document that the next stage reads. A session moves forward through the pipeline one stage at a time.

```
feed.md → /dive → context.md → /product → product.md → /tasks → tasks/ → /exec
```

---

## Stages

### 1. Feed

The pipeline starts with a `feed.md` file written by you. This is the raw input — whatever you need to communicate to get the session going. It can be a product idea, a bug description, a feature request, a vague thought, or anything in between.

No format is required. The only goal is to capture enough intent for the next stage to work with.

---

### 2. `/dive`

`/dive` reads `feed.md` and opens a dialogue with you. It asks questions, challenges assumptions, and keeps going until the problem is fully understood and all ambiguities are resolved. Nothing is assumed; everything unclear gets surfaced and settled.

The output is `context.md` — a complete, unambiguous description of the problem that the next stage can work from without needing to ask anything further.

---

### 3. `/product`

`/product` reads `context.md` and produces `product.md` — a product requirements document. It takes the resolved understanding from the previous stage and turns it into a structured definition of what is being built: scope, goals, non-goals, and acceptance criteria.

---

### 4. `/tasks`

`/tasks` reads `product.md` and breaks the work down into individual, atomic tasks. Each task is written as a separate file under `tasks/`. Tasks are scoped so that they can be worked on independently and, where possible, simultaneously.

---

### 5. `/exec`

`/exec` works on a single task file. It takes one task, does the implementation, and produces the result. It is run once per task.

---

## State

Every skill reads and updates `state.json` at the start and end of its run. This file tracks the current progress of the session — which stages have completed and what the current state of the pipeline is.

The schema and full specification of `state.json` are defined in `reference/state.md`.