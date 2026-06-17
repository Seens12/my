---
name: grill-with-docs
description: Full-cycle session that stress-tests a plan against the existing domain model and codebase, sharpens terminology, implements the agreed decisions, updates documentation (CONTEXT.md, ADRs) inline, then verifies the result with codebase QA, browser QA, a scoped security review, and a final codebase re-verification before reporting back. Use when the user wants to stress-test a plan against their project's language and documented decisions and then actually ship and verify the result, not just produce a plan.
disable-model-invocation: true
---

## Session language

Conduct the entire session — questions, recommendations, QA reports, security findings, the final report, everything you say to the user — in Russian, regardless of what language the user's plan, codebase, comments, or commit history are in. Do not translate code, identifiers, file paths, log output, or quoted source material; only your own commentary is in Russian. Write CONTEXT.md and ADR content in whatever language the project's existing documentation already uses (normally English, matching the domain terms in the code) unless the user says otherwise — the artifact language and the conversation language are independent choices.

## Session artifacts

Three files anchor the session. Create them in Phase 0:

- `docs/session-log.md` — running log of every decision, term change, ADR, and accepted/deferred improvement suggestion. Single source of truth across every session this skill has ever run for this repo. Survives session crashes and gives the next session continuity.
- `.grill-state.json` — checkpoint after each phase: current `phase`, `status`, decisions so far, files touched, what's next. Used to resume a crashed or paused session in a new one.
- `docs/backlog.md` — ideas surfaced during the session that the user deferred, or that hit the rate-limit. Cleared only after the user reviews.

`docs/session-log.md` never forks into dated copies — that would break "single source of truth." If the file already exists from a prior session, append a new `## Session {date}` header to the top and log new decisions underneath it. Read the whole file in Phase 0 — do not re-derive decisions that were already settled.

Never write an actual secret value (API key, password, token, connection string) into any of these three files, even when Phase 0 or Phase 4 finds one — reference its location (`{file}:{line}`) instead. These files are meant to be read by humans and often committed; treat them with the same care as the codebase itself.

`.grill-state.json` is scratch checkpoint state, not a deliverable — add it to `.gitignore` if it isn't already, so phase-by-phase churn doesn't pollute commit history. `docs/session-log.md`, `docs/backlog.md`, `CONTEXT.md`, and ADRs are real artifacts meant to be committed.

## Stop conditions

Stop and surface to the user — do not guess through these:

1. Plan is infeasible as described (library doesn't exist, API doesn't support the claim, contradicts load-bearing existing code)
2. A QA failure can only be fixed at the root by reopening a Phase 1 decision
3. A security finding is architectural (the chosen auth model is wrong, not just a missing check)
4. Scope has drifted to >2x the original Phase 1 in-scope list
5. 3+ consecutive decisions received no per-decision linkage — the framing is wrong, ask
6. The user rejected 3+ suggestions in a row — directional mismatch, ask before continuing
7. The dependency or feature you want to recommend cannot be verified to exist or behave as claimed in *this* codebase

## Phase gate between phases

Default behavior: after completing each phase, ask "Phase X complete. Proceed to Y? [y/n]". No response within a reasonable window counts as "y" — proceed. The user can also pre-approve "just run through to the end"; honor that, but the final report still surfaces every gate the user skipped. **If a stop condition fires, the gate is mandatory — silence and pre-approval do not skip it.**

The phase-internal gates inside Phase 3 (3a → 3b → Phase 4) and at the end of Phase 4 (→ Phase 5) follow this same default; see those phases for their exact wording.

## Phase 0 — Codebase reconnaissance (silent, before first question)

Before asking anything, build a complete picture of the affected surface area. Do not output anything to the user during this phase — work silently.

1. Check whether `.grill-state.json` already exists. If it exists and `status` is not `complete`, this is a **resume**: read it to find the last completed phase, read `docs/session-log.md` in full, and pick up after that phase instead of restarting Phase 1 from scratch or re-deriving settled decisions. Still run the rest of this reconnaissance list once — the codebase may have changed since the crash. If no `.grill-state.json` exists, or its `status` is `complete`, this is a fresh session — proceed normally.
2. Read the full file tree — identify all modules, packages, and layers
3. Trace the call chain related to the plan: entry points → core logic → persistence/IO → outbound effects
4. Read every file the plan will touch or that calls into it
5. Read every file those files import from (one level deep minimum)
6. Identify shared state, global config, and cross-cutting concerns (auth, logging, error handling, transactions, feature flags)
7. Note all existing tests for the affected area, and how to run them — test command, framework, fixtures, required env vars
8. Note how to run the application locally — dev server command, seed data, anything needed for Phase 3's QA later
9. Check whether `CONTEXT-MAP.md` exists at the root. If so, this is a multi-context repo — read it, then read every `CONTEXT.md` and `docs/adr/` it points to that's plausibly relevant to the plan. If only a root `CONTEXT.md` exists, single context. If neither exists, note that both will be created lazily (see CONTEXT-FORMAT.md). Also read any `README` inside affected packages and any prior `docs/session-log.md`.
10. Re-read `CONTEXT.md` (or the relevant per-context files) and all ADRs fresh — do not rely on content seen earlier in this conversation
11. Unless step 1 found a resumable session, create `docs/session-log.md` (or append a dated section) and the initial `.grill-state.json` with `phase: "0"`, `status: "in_progress"`

After reconnaissance, produce an internal dependency snapshot (not shown to the user):

- Which modules does the plan mutate?
- Which callers depend on those modules?
- Which modules does the plan depend on?
- Where are the transaction/consistency boundaries?
- Where are the I/O boundaries (DB, queue, external API)?
- Are there background jobs, crons, or async workers touching this area?
- Are there any alternate entry points (CLI, webhook, queue consumer) that bypass the main path?
- **Preliminary blast radius estimate** — order-of-magnitude: how many files likely to be mutated, how many tests likely to be touched. Sanity-check the plan's scale.

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

### Capture upfront (before the first domain question)

Two artifacts must be agreed before drilling into decisions:

**Scope fence** — append to session-log:
```
## Scope fence ({date})
In scope: {list}
Out of scope: {list}
Non-goals: {list}
```

**Acceptance criteria** — 3–7 concrete, testable scenarios. Each must be verifiable in Phase 5. Append to session-log:
```
## Acceptance criteria ({date})
- [ ] {scenario: what user/system does, what should happen}
- [ ] {scenario 2}
...
```

If the user can't articulate criteria, ask. Without them, Phase 5 has nothing to verify against. Every decision in Phase 2 must be traceable to a criterion or a scope-fence item — otherwise it's scope drift, surface it.

### Per-decision linkage

Every decision, before being accepted, must declare what observable change it produces. Pick at least one:

- code change in `{file/area}`
- doc change in `{CONTEXT.md term / ADR}`
- test in `{file}`
- or: **no observable change — terminological resolution only** (e.g., choosing a canonical name when code is unchanged)

If none of these applies, surface it — the decision is probably not a real decision. Log the linkage in session-log alongside the decision.

### Decision log entry format

After each decision is accepted, append to `docs/session-log.md`:

```
## {ISO timestamp} — {short title}
**Decision**: {what was decided}
**Why**: {rationale, 1–3 sentences}
**Links**: {file paths affected; for terms → CONTEXT.md#{section}; for ADRs → docs/adr/NNNN-{slug}.md}
**Acceptance criterion**: {which Phase 1 criterion this satisfies, or "scope-fence: in-scope"}
**Status**: accepted → implemented → verified | follow-up | rejected | superseded by {timestamp} (update as you go — see Phase 5 for what each terminal status means, and "Reopening a decision" above for `superseded`)
```

When the decision is itself a term resolution or an ADR, also add this compact line to the same entry — it's what gets pulled into the final report's "Решения" section:

For terms: `[TERM] {term}: {definition} → CONTEXT.md#{section}`

For ADRs: `[ADR] {slug} → docs/adr/NNNN-{slug}.md`

### Rate-limit on improvement suggestions

Maximum **2 active suggestions per decision node**. If more surface, write them to `docs/backlog.md` and reference the file in chat, not the items themselves:

```
- [DEFER] {idea} — {why relevant right now} — {date} {session}
```

The user reviews the backlog at session end. Ideas that should die there, die there. Don't restate deferred suggestions in chat.

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

### Reopening a decision

A decision that turns out to be wrong later — a QA failure that traces back to it (stop condition 2), a contradiction surfaced during the session, or an unverified assumption that Phase 5 found to be load-bearing and false — is never edited in place. Append a **new** session-log entry for the corrected decision, then go back and set the old entry's **Status** to `superseded by {new timestamp}`, mirroring the pattern ADR-FORMAT.md uses for ADRs. If Phase 2 already implemented the old decision, the new decision's implementation step must revert or modify that code — don't leave both versions sitting in the codebase. Re-run whatever tests and QA checks the old decision had already passed; passing once doesn't carry over to a changed decision.

### Hidden dependency probe

For each decision node, explicitly check:

- Is there any background job, cron, or async worker that touches this?
- Is there any other entry point (CLI, webhook, queue consumer) that bypasses the path we're looking at?
- Is there shared mutable state (cache, singleton, module-level variable) that this change could affect?
- Are there feature flags, env-specific branches, or tenant overrides in this path?

If you cannot answer these from the codebase — say so. Do not assume the answer is "no".

### Existing ADR cross-check

Before offering to create an ADR, re-read all existing ADRs (not from memory — re-read the files). If the new decision would contradict an existing ADR, surface that to the user before proceeding. Use the "superseded by ADR-NNNN" pattern from ADR-FORMAT.md instead of letting the contradiction sit. Cross-check is cheap; contradictions discovered in Phase 5 are expensive.

### Proactive improvement suggestions

Once the core question for a decision node is settled, look at what's already being touched and surface adjacent improvements that are genuinely comparable in scope — not a wishlist, just the things a maintainer would naturally consider given the code you're already in. For example: while adding a rate limiter, note that it's a natural place to also support a per-tenant override; while touching a search endpoint, note that it could take an extra filter parameter with little added cost; while reshaping a module's boundary, note an alternative internal structure that would make a likely next change easier.

Rules for these suggestions:

- Hard cap: 2 per decision node, with overflow going to `docs/backlog.md` instead of chat — see "Rate-limit on improvement suggestions" above for the mechanics.
- Present each one as a labelled, optional proposal — never bundle it into the decision silently. The user must be able to see clearly what was asked for versus what you're additionally proposing.
- For each suggestion, state: why it's relevant given what's already being touched, and the rough cost/complexity it would add.
- Let the user accept, defer, or reject it explicitly before moving on. A deferred suggestion still gets recorded in the final report so it isn't lost.
- Stay inside the plan's actual domain. Don't pad the list with generic best-practice suggestions ("add more tests", "add logging everywhere") unless they're concretely tied to the code being changed right now.
- These are suggestions, not commitments — do not implement a suggestion in Phase 2 unless the user accepted it.

### Domain awareness

Phase 0 (step 9) already determined whether this is a single- or multi-context repo, and read the relevant `CONTEXT.md` file(s). Use that here when framing questions and resolving terms. For reference, the two layouts:

#### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   ├── adr/
│   │   ├── 0001-event-sourced-orders.md
│   │   └── 0002-postgres-for-write-model.md
│   └── session-log.md             ← added by this skill
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   ├── adr/                          ← system-wide decisions
│   └── session-log.md
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       └── CONTEXT.md
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed. When a repo has multiple `docs/adr/` folders, ADR-FORMAT.md's "Multi-context repos" section governs which folder a given decision belongs in and how numbering works across them.

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
- **After each implementation: run the tests for the files you just touched** (the project's test command from Phase 0). Catch regressions in the act — don't defer to Phase 3.
- **Update the session-log entry**: `accepted → implemented` once the change passes its local test run. If the test run failed, keep `accepted` and note the blocker; do not mark implemented.
- Keep a running list, per decision, of every file touched — Phase 5 will need it.

## Phase 3 — QA

Run QA only after Phase 2's changes for the current branch are in place. Codebase QA always comes before browser QA — don't exercise the UI on code you haven't already confirmed builds and passes its own tests.

### 3a. Codebase QA

- Run the project's existing automated tests for the affected area. If none exist for the new behaviour, write minimal tests covering it before relying on manual inspection.
- **Test-the-test**: for each new test, run it once with the fix reverted (or the bug reintroduced by temporarily reverting the change). It must fail. If it passes with the fix reverted, the test is not actually checking the fix — fix the test before continuing. This catches the "I wrote a test that always passes" failure mode.
- Run linters, type-checkers, and the build if the project has them.
- Re-read every changed file and confirm it matches what the decision log says was agreed.
- Report each check as pass/fail with the command used and the relevant output — not just a verdict.

### 3b. Browser / API QA — conditional

- **If the repo has a UI** (web app, mobile shell, electron, etc.) and a way to run it (dev server + browser/automation): start the application the way Phase 0 determined it's normally run. Exercise the actual user-facing flow the change affects, the way a user would. Note exactly what you did and what you observed.
- **If the repo is backend-only / CLI / library / data pipeline**: run **API/CLI QA** with the same standard. Exercise the actual user-facing flow (curl, CLI invocation, library call from a small script). Note the command, the input, the output. Don't paraphrase "it works."
- **If unclear which applies**: ask once, then proceed. Don't keep re-asking on every iteration.

### Failure handling

If anything fails in 3a or 3b:

1. Don't patch the symptom in isolation. Trace why it failed: re-read the failing file, then walk one level out — every caller, every file it imports from, every file that imports it.
2. Check whether the same root cause appears anywhere else in the codebase — same pattern, same assumption, copy-pasted logic. If so, that's part of the same fix, not a separate bug to defer.
3. Apply the fix at the root cause, not the surface symptom.
4. Re-run the failing check plus everything you identified as downstream of the fix.
5. Log the failure, the root cause, the files touched to fix it, and the re-test result in the final report — don't silently resolve and move on as if nothing happened.

Repeat 3a → 3b until both pass, or until you hit a failure that needs a decision from the user (e.g., the fix would require reopening a Phase 1 decision) — in that case, stop QA and surface the conflict instead of guessing.

### Phase gate (3a → 3b → Phase 4)

After 3a passes, ask: "Phase 3a passed. Proceed to 3b? [y/n]". Same after 3b passes → Phase 4. If the user explicitly says no, write current state to `.grill-state.json` with `phase: "3a"`, `status: "passed-awaiting-3b"` (or the equivalent for 3b) and stop.

## Phase 4 — Security review

Scope this to what actually changed in Phase 2 during this session — this is not a general audit of the whole codebase, stay inside the surface area identified in Phase 0.

Check specifically for:

- Input validation and injection risk on any new or modified input path
- Authn/authz boundaries — does the change cross a permission boundary, and is that boundary still enforced?
- Secrets handling — are credentials, tokens, or keys touched, logged, or exposed by the change?
- Data exposure across context boundaries — especially relevant if a `CONTEXT.md` decision touched data ownership (e.g., who's allowed to read a field that crossed from one context to another)
- New third-party dependencies introduced for this change, and their footprint

For each finding, report: severity, exact location, and whether you fixed it now or it needs to be raised as a follow-up decision with the user. If fixing it changes behaviour already covered by Phase 3, re-run the relevant QA checks.

### Phase gate (4 → 5)

After Phase 4 completes, ask: "Security review done. Proceed to Phase 5? [y/n]".

## Phase 5 — Final codebase re-verification

1. Re-read the dependency snapshot from Phase 0.
2. **Acceptance criteria sweep**: go through the criteria from Phase 1. For each: `pass` / `fail` / `skipped (reason)`. This is the primary definition of done. Any `fail` blocks session close — surface it.
3. **Decision closure**: for every entry in `docs/session-log.md`, the status must be terminal:
   - `verified` (implemented and acceptance criterion met), or
   - `follow-up` (explicitly tagged with what blocks it), or
   - `rejected` (decision was overturned, with link to the overturning decision), or
   - `superseded by {timestamp}` (decision was reopened and replaced — see "Reopening a decision" in Phase 1)
   Any `accepted` or `implemented` without one of these terminals is a hole — surface it.
4. **Term ↔ code consistency check**: extract every term defined in `CONTEXT.md` (and any per-context `CONTEXT.md` files). For each, grep the codebase for the canonical term and its "Avoid" aliases. Flag three failure modes:
   - Term defined but never used in code → possibly dead language; ask the user whether to drop it from the glossary
   - Term used in code but never defined → gap in the glossary; add it
   - Term used in code inconsistently with its `CONTEXT.md` definition → likely a real bug; surface the location
5. **ADR consistency check**: re-read all ADRs. For each, check that the code it references still matches. If a later change has drifted from an ADR without a "superseded by" pointer, surface it.
6. For each decision: confirm the implementation actually matches the decision log, not your memory of having implemented it.
7. Re-read every file changed in Phase 2 through 4 in its current, final state — don't trust a diff or your recollection of writing it.
8. List any files or modules in the affected surface area that were never read during the session.
9. List any assumptions made during the session that couldn't be verified from the code.
10. If any unverified assumption is load-bearing, reopen the relevant decision before closing rather than noting it and moving on.

Write the final `.grill-state.json` with `phase: "5"`, `status: "complete"`. Output this sweep to the user so they can see exactly what was and wasn't checked.

## Session boundary

When Phase 5 is complete and the user has confirmed the summary, output the structured report below and stop. Don't start new work beyond what's in the report — anything flagged as a follow-up, a deferred suggestion, or a backlog item is for the user to bring back as a new session, not for you to pick up unprompted.

Report format:

```
## Критерии приёма
- [PASS/FAIL/SKIP] {критерий}: {доказательство — команда, вывод или наблюдение}

## Решения
- [DECISION] {что решили} → docs/session-log.md#{timestamp}
- [TERM] {термин}: {определение, добавленное/обновлённое в CONTEXT.md}
- [ADR] {файл ADR, если создавался}

## Предложенные улучшения
- [ПРИНЯТО / ОТЛОЖЕНО / ОТКЛОНЕНО] {предложение}: {почему релевантно, во что обошлось бы}

## Бэклог
- {N} пунктов отложено в docs/backlog.md — проверить в конце сессии

## QA — кодовая база
- [PASS/FAIL] {проверка}: {команда и результат}

## QA — браузер / API
- [PASS/FAIL] {сценарий}: {что делали, что наблюдали}

## Найденные и исправленные баги
- {баг}: {корневая причина}; {затронутые файлы}; {фикс}; {результат повторного теста}

## Проверка безопасности
- [серьёзность] {находка}: {исправлено сейчас / вынесено как отдельное решение}

## Итоговая перепроверка кодовой базы
- {что перечитано и подтверждено}

## Соответствие терминов и кода
- {термин}: {defined | unused | used-but-undefined | inconsistent} — {evidence}

## Соответствие ADR
- {ADR}: {still valid | superseded by ADR-NNNN | drifted — {details}}

## Неподтверждённые предположения
- {...}

## Файлы, которые не были прочитаны
- {...}
```
