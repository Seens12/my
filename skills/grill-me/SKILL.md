---
name: grill-with-docs
description: Full-cycle session that stress-tests a plan against the existing domain model and codebase, sharpens terminology, implements the agreed decisions, updates documentation (CONTEXT.md, ADRs) inline, then verifies the result with codebase QA, browser QA, a scoped security review, and a final codebase re-verification before reporting back. Use when the user wants to stress-test a plan against their project's language and documented decisions and then actually ship and verify the result, not just produce a plan.
disable-model-invocation: true
---

## Session language

Conduct the entire session — questions, recommendations, QA reports, security findings, the final report, everything you say to the user — in Russian, regardless of what language the user's plan, codebase, comments, or commit history are in. Do not translate code, identifiers, file paths, log output, or quoted source material; only your own commentary is in Russian. Write CONTEXT.md and ADR content in whatever language the project's existing documentation already uses (normally English, matching the domain terms in the code) unless the user says otherwise — the artifact language and the conversation language are independent choices.

## Phase 0 — Codebase reconnaissance (silent, before first question)

Before asking anything, build a complete picture of the affected surface area. Do not output anything to the user during this phase — work silently.

1. Read the full file tree — identify all modules, packages, and layers
2. Trace the call chain related to the plan: entry points → core logic → persistence/IO → outbound effects
3. Read every file the plan will touch or that calls into it
4. Read every file those files import from (one level deep minimum)
5. Identify shared state, global config, and cross-cutting concerns (auth, logging, error handling, transactions, feature flags)
6. Note all existing tests for the affected area, and how to run them — test command, framework, fixtures, required env vars
7. Note how to run the application locally — dev server command, seed data, anything needed for Phase 3's browser QA later
8. Read `CONTEXT.md`, all ADRs, and any `README` inside affected packages
9. Re-read `CONTEXT.md` and all ADRs fresh — do not rely on content seen earlier in this conversation

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
- **Grounded over speculative**: only recommend an approach you have actually confirmed is feasible in this codebase — check that the libraries, APIs, framework features, or infrastructure it depends on actually exist and behave the way you're assuming before proposing it. If you can't verify feasibility from the code, say so explicitly instead of recommending it as if it were checked. Never recommend something just because it would typically work in a project like this — verify it works in *this* one.

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

### Proactive improvement suggestions

Once the core question for a decision node is settled, look at what's already being touched and surface adjacent improvements that are genuinely comparable in scope — not a wishlist, just the things a maintainer would naturally consider given the code you're already in. For example: while adding a rate limiter, note that it's a natural place to also support a per-tenant override; while touching a search endpoint, note that it could take an extra filter parameter with little added cost; while reshaping a module's boundary, note an alternative internal structure that would make a likely next change easier.

Rules for these suggestions:

- Present each one as a labelled, optional proposal — never bundle it into the decision silently. The user must be able to see clearly what was asked for versus what you're additionally proposing.
- For each suggestion, state: why it's relevant given what's already being touched, and the rough cost/complexity it would add.
- Let the user accept, defer, or reject it explicitly before moving on. A deferred suggestion still gets recorded in the final report so it isn't lost.
- Stay inside the plan's actual domain. Don't pad the list with generic best-practice suggestions ("add more tests", "add logging everywhere") unless they're concretely tied to the code being changed right now.
- These are suggestions, not commitments — do not implement a suggestion in Phase 2 unless the user accepted it.

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

## Phase 2 — Implementation

As each decision is reached, implement it before moving to the next branch of the design tree — don't batch every code change to the end of the session.

- Make the smallest change that satisfies the decision as agreed. Do not generalize beyond what was actually discussed and accepted — speculative scope is exactly what Phase 1 was supposed to rule out.
- Keep documentation and code changes distinct: `CONTEXT.md`/ADR updates happen as described in Phase 1, code edits happen here. A decision that's purely terminological may have no code change at all.
- Implement an accepted improvement suggestion from Phase 1 alongside the decision it was attached to, not bundled invisibly into unrelated code.
- Keep a running list, per decision, of every file touched — Phase 5 will need it.

## Phase 3 — QA

Run QA only after Phase 2's changes for the current branch are in place. Codebase QA always comes before browser QA — don't exercise the UI on code you haven't already confirmed builds and passes its own tests.

### 3a. Codebase QA

- Run the project's existing automated tests for the affected area. If none exist for the new behaviour, write minimal tests covering it before relying on manual inspection.
- Run linters, type-checkers, and the build if the project has them.
- Re-read every changed file and confirm it matches what the decision log says was agreed.
- Report each check as pass/fail with the command used and the relevant output — not just a verdict.

### 3b. Browser QA

Only proceed once 3a passes.

- Start the application the way Phase 0 determined it's normally run.
- Exercise the actual user-facing flow the change affects, the way a user would — don't assume the backend behaviour observed in 3a implies the UI behaves correctly too; check it directly.
- Note exactly what you did and what you observed, not just "it works."

### Failure handling

If anything fails in 3a or 3b:

1. Don't patch the symptom in isolation. Trace why it failed: re-read the failing file, then walk one level out — every caller, every file it imports from, every file that imports it.
2. Check whether the same root cause appears anywhere else in the codebase — same pattern, same assumption, copy-pasted logic. If so, that's part of the same fix, not a separate bug to defer.
3. Apply the fix at the root cause, not the surface symptom.
4. Re-run the failing check plus everything you identified as downstream of the fix.
5. Log the failure, the root cause, the files touched to fix it, and the re-test result in the final report — don't silently resolve and move on as if nothing happened.

Repeat 3a → 3b until both pass, or until you hit a failure that needs a decision from the user (e.g., the fix would require reopening a Phase 1 decision) — in that case, stop QA and surface the conflict instead of guessing.

## Phase 4 — Security review

Scope this to what actually changed in Phase 2 during this session — this is not a general audit of the whole codebase, stay inside the surface area identified in Phase 0.

Check specifically for:

- Input validation and injection risk on any new or modified input path
- Authn/authz boundaries — does the change cross a permission boundary, and is that boundary still enforced?
- Secrets handling — are credentials, tokens, or keys touched, logged, or exposed by the change?
- Data exposure across context boundaries — especially relevant if a `CONTEXT.md` decision touched data ownership (e.g., who's allowed to read a field that crossed from one context to another)
- New third-party dependencies introduced for this change, and their footprint

For each finding, report: severity, exact location, and whether you fixed it now or it needs to be raised as a follow-up decision with the user. If fixing it changes behaviour already covered by Phase 3, re-run the relevant QA checks.

## Phase 5 — Final codebase re-verification

1. Re-read the dependency snapshot from Phase 0.
2. For each decision: confirm the implementation actually matches the decision log, not your memory of having implemented it.
3. Re-read every file changed in Phase 2 through 4 in its current, final state — don't trust a diff or your recollection of writing it.
4. List any files or modules in the affected surface area that were never read during the session.
5. List any assumptions made during the session that couldn't be verified from the code.
6. If any unverified assumption is load-bearing, reopen the relevant decision before closing rather than noting it and moving on.

Output this sweep to the user so they can see exactly what was and wasn't checked.

## Session boundary

When Phase 5 is complete and the user has confirmed the summary, output the structured report below and stop. Don't start new work beyond what's in the report — anything flagged as a follow-up or a deferred suggestion is for the user to bring back as a new session, not for you to pick up unprompted.

Report format:

```
## Решения
- [DECISION] {что решили}
- [TERM] {термин}: {определение, добавленное/обновлённое в CONTEXT.md}
- [ADR] {файл ADR, если создавался}

## Предложенные улучшения
- [ПРИНЯТО / ОТЛОЖЕНО / ОТКЛОНЕНО] {предложение}: {почему релевантно, во что обошлось бы}

## QA — кодовая база
- [PASS/FAIL] {проверка}: {команда и результат}

## QA — браузер
- [PASS/FAIL] {сценарий}: {что делали, что наблюдали}

## Найденные и исправленные баги
- {баг}: {корневая причина}; {затронутые файлы}; {фикс}; {результат повторного теста}

## Проверка безопасности
- [серьёзность] {находка}: {исправлено сейчас / вынесено как отдельное решение}

## Итоговая перепроверка кодовой базы
- {что перечитано и подтверждено}

## Неподтверждённые предположения
- {...}

## Файлы, которые не были прочитаны
- {...}
```
