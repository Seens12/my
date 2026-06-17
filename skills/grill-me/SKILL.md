---
name: grill-with-docs
description: Grilling session that challenges your plan against the existing domain model, sharpens terminology, and updates documentation (CONTEXT.md, ADRs) inline as decisions crystallise. Use when user wants to stress-test a plan against their project's language and documented decisions.
disable-model-invocation: true
---

## Phase 0 — Codebase reconnaissance (silent, before first question)

Before asking anything, build a complete picture of the affected surface area. Do not output anything to the user during this phase — work silently.

1. Read the full file tree — identify all modules, packages, and layers
2. Trace the call chain related to the plan: entry points → core logic → persistence/IO → outbound effects
3. Read every file the plan will touch or that calls into it
4. Read every file those files import from (one level deep minimum)
5. Identify shared state, global config, and cross-cutting concerns (auth, logging, error handling, transactions, feature flags)
6. Note all existing tests for the affected area
7. Read `CONTEXT.md`, all ADRs, and any `README` inside affected packages
8. Re-read `CONTEXT.md` and all ADRs fresh — do not rely on content seen earlier in this conversation

After reconnaissance, produce an internal dependency snapshot (not shown to the user):

- Which modules does the plan mutate?
- Which callers depend on those modules?
- Which modules does the plan depend on?
- Where are the transaction/consistency boundaries?
- Where are the I/O boundaries (DB, queue, external API)?
- Are there background jobs, crons, or async workers touching this area?
- Are there any alternate entry points (CLI, webhook, queue consumer) that bypass the main path?

You will reference this snapshot throughout the session.

## Phase 1 — Grilling

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

Ask the questions one at a time, waiting for feedback on each question before continuing.

### Question ordering

Ask in this order:
1. Questions that block other decisions (load-bearing choices)
2. Questions that resolve terminology conflicts with `CONTEXT.md`
3. Questions about edge cases and consistency boundaries
4. Questions about implementation details

### Recommendation bias

When proposing a recommended answer, prefer solutions that are:

- **Safe over clever**: a boring, explicit, auditable solution beats a terse one with hidden assumptions
- **Reversible over optimal**: if two options solve the problem, prefer the one that's easier to undo
- **Explicit over implicit**: make side effects, state mutations, and error paths visible in the design — even if it adds boilerplate
- **Consistent with existing patterns**: if the codebase already solves a similar problem in a specific way, default to that — deviation needs justification

If the safest option is also the hardest to implement, say so explicitly and explain the trade-off. Do not simplify away from safety to save implementation effort.

### Decision validation loop

For every decision reached during the session:

1. Re-read the relevant source files (not from memory — re-read the actual file)
2. Check: does this decision contradict any existing behaviour?
3. Check: does this decision assume something that isn't true in the current code?
4. Check: are there callers or dependents that this decision silently breaks?
5. If any check fails — surface the contradiction before accepting the decision

### Hidden dependency probe

For each decision node, explicitly check:

- Is there any background job, cron, or async worker that touches this?
- Is there any other entry point (CLI, webhook, queue consumer) that bypasses the path we're looking at?
- Is there shared mutable state (cache, singleton, module-level variable) that this change could affect?
- Are there feature flags, env-specific branches, or tenant overrides in this path?

If you cannot answer these from the codebase — say so. Do not assume the answer is "no".

### Domain awareness

During codebase exploration, also look for existing documentation:

#### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"

When a term is updated or overridden, record the change inline in `CONTEXT.md` under a `## Changelog` section with date and reason. Do not silently overwrite previous definitions.

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account' — do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, re-read the relevant files and check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible — which is right?"

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up — capture them as they happen. Use the format in [CONTEXT-FORMAT.md](CONTEXT-FORMAT.md).

Don't couple `CONTEXT.md` to implementation details. Only include terms that are meaningful to domain experts.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](ADR-FORMAT.md).

## Phase 2 — Pre-close safety sweep

Before declaring the session complete, re-verify explicitly:

1. Re-read the dependency snapshot from Phase 0
2. For each decision made: confirm it is consistent with the snapshot
3. List any files or modules you did NOT read that might be affected
4. List any assumptions you made that you could not verify from code
5. If any unverified assumptions are load-bearing — reopen the relevant decision before closing

Output this sweep to the user so they can see what was and wasn't checked.

## Session boundary

When the sweep is complete and the user has confirmed the summary of decisions, output a structured decision log and STOP.

Do not begin implementation. Do not suggest next steps beyond: "Run `/tdd` to begin implementation."

The decision log format:

```
## Decisions

- [DECISION] {what was decided}
- [TERM] {term}: {definition added or updated in CONTEXT.md}
- [ADR] {adr file created, if any}

## Unverified assumptions

- {list anything that could not be confirmed from code}

## Files not read

- {list any files in the affected area that were not read}
```
