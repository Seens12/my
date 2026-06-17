# ADR Format

ADRs live in `docs/adr/` and use sequential numbering: `0001-slug.md`, `0002-slug.md`, etc.

Create the `docs/adr/` directory lazily — only when the first ADR is needed.

## Template

```
# {Short title of the decision}

{1-3 sentences: what's the context, what did we decide, and why.}
```

That's it. An ADR can be a single paragraph. The value is in recording *that* a decision was made and *why* — not in filling out sections.

## Optional sections

Only include these when they add genuine value. Most ADRs won't need them.

* **Status** frontmatter (`proposed | accepted | deprecated | superseded by ADR-NNNN`) — useful when decisions are revisited
* **Considered Options** — only when the rejected alternatives are worth remembering
* **Consequences** — only when non-obvious downstream effects need to be called out
* **Safety trade-off** — use when the chosen option is harder to implement but safer or more reversible than the alternative. State explicitly: what risk does this mitigate, and what was the simpler option that was rejected. This prevents a future engineer from "simplifying" a deliberate conservative choice.

## Numbering

Scan `docs/adr/` for the highest existing number and increment by one. In a multi-context repo, this is scoped per folder — see "Multi-context repos" below.

## Cross-check before writing

Before creating a new ADR, re-read every existing ADR (don't rely on memory). If the new decision contradicts, supersedes, or partially overlaps an existing one:

- **Contradicts**: stop. Surface the conflict to the user — do not write a contradictory ADR. Either revise the new decision, or mark the old one as `superseded by ADR-NNNN` in the same session.
- **Supersedes**: write the new ADR and update the old one's status to `superseded by ADR-NNNN` in the same change set. Don't leave the old one claiming authority.
- **Overlaps partially**: reference the existing ADR explicitly in the new one — "extends ADR-NNNN for the case of X" — so a future reader finds both.

Skipping this check is the most common way ADRs drift out of sync. A 30-second grep is cheap; a contradiction discovered during an incident is not.

## When to offer an ADR

All three of these must be true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will look at the code and wonder "why on earth did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If a decision is easy to reverse, skip it — you'll just reverse it. If it's not surprising, nobody will wonder why. If there was no real alternative, there's nothing to record beyond "we did the obvious thing."

Additionally: if a safer but more complex implementation was chosen over a simpler one — always write an ADR. The simpler path will be rediscovered; the reasons for rejecting it will not.

### What qualifies

* **Architectural shape.** "We're using a monorepo." "The write model is event-sourced, the read model is projected into Postgres."
* **Integration patterns between contexts.** "Ordering and Billing communicate via domain events, not synchronous HTTP."
* **Technology choices that carry lock-in.** Database, message bus, auth provider, deployment target. Not every library — just the ones that would take a quarter to swap out.
* **Boundary and scope decisions.** "Customer data is owned by the Customer context; other contexts reference it by ID only." The explicit no-s are as valuable as the yes-s.
* **Deliberate deviations from the obvious path.** "We're using manual SQL instead of an ORM because X." Anything where a reasonable reader would assume the opposite. These stop the next engineer from "fixing" something that was deliberate.
* **Constraints not visible in the code.** "We can't use AWS because of compliance requirements." "Response times must be under 200ms because of the partner API contract."
* **Rejected alternatives when the rejection is non-obvious.** If you considered GraphQL and picked REST for subtle reasons, record it — otherwise someone will suggest GraphQL again in six months.
* **Conservative choices made for safety.** If a simpler implementation was available but rejected because it was harder to reason about, less auditable, or more likely to fail silently — record it. Future readers will see complexity and try to remove it; the ADR explains why that complexity is load-bearing.

## Multi-context repos

When the repo has multiple contexts (see CONTEXT-FORMAT.md), a context may keep its own `docs/adr/` next to its `CONTEXT.md`. Each folder's numbering is independent — `src/billing/docs/adr/0001-...md` and the root `docs/adr/0001-...md` are not the same sequence and don't collide.

Where a decision goes depends on its blast radius, not on which file happened to prompt it:

- **Affects one context only** (an internal implementation choice inside Billing) → that context's `docs/adr/`.
- **Affects how contexts interact, or the system as a whole** (message bus choice, a new event contract between Ordering and Billing, a deployment target) → the root `docs/adr/`.

When unsure which applies, ask: does this decision constrain choices inside one context, or does it shape what's shared or how contexts talk to each other? The former is context-local; the latter is root-level.
