# Intent Record

Status: DRAFT — 2026-09-10

An **intent record** is the unit of work in the factory. One record → one branch → one PR.
It is the *specification*: if a behaviour is not in the record or in `../design/`, it is
out of scope and an agent must not invent it.

Records live in `/intents/` as `NNNN-short-slug.md`. They are committed — the intent
history is as valuable as the git history, because it records *why*.

> Naming note: an intent **record** (a unit of engineering work) is unrelated to a user
> **Intent** (the product concept in `../product/intent-model.md`). Unfortunate collision;
> the docs always qualify which one.

---

## Template

```markdown
---
id: 0042
title: <imperative phrase — "Add per-source poll backoff">
status: draft | ready | in-progress | in-review | merged | abandoned
author: <human>
created: YYYY-MM-DD
design_refs:                      # design docs this depends on — agent MUST read these
  - design/ingestion.md#fetch-pipeline
  - design/data-model.md#sources
risk: low | medium | high         # drives review policy — see review-policy.md
---

## Why
<The problem, in the user's or the system's terms. Not the solution.>

## What
<The change, described behaviourally. What is true after this ships that isn't now?>

## Acceptance criteria
<Numbered, individually testable. This is what the gates check against and what the
review agent audits. Vague criteria are the single most common cause of a bad PR.>

1. ...
2. ...

## Out of scope
<Explicit non-goals. Prevents scope creep — the most common agent failure mode.>

## Design invariants touched
<Which of the invariants in design/data-model.md this could affect, if any.
"None" is a valid and common answer — but it must be stated, not omitted.>

## Open questions
<If any exist, status MUST be `draft`. A record with open questions is not buildable.>
```

---

## Rules

### For the author
- **Describe behaviour, not implementation.** "Summaries must cite their source article"
  is an intent. "Add a `citations` column" is a design decision that belongs in
  `../design/`, or in the agent's plan.
- **Acceptance criteria must be individually testable.** If you can't imagine the
  assertion, the criterion is too vague. This is where quality is won or lost.
- **One coherent change per record.** If it needs "and" between unrelated things, split it.
- **State risk honestly.** It drives whether a human sees the PR (`review-policy.md`).
  Under-stating risk to route around review is the one way to defeat this system.

### For the building agent
- **Read every `design_refs` document before planning.** They are the binding constraints.
- **STOP and escalate, do not guess**, when:
  - an acceptance criterion is ambiguous or self-contradictory
  - the change would violate a design invariant
  - the intent conflicts with a `> DECISION` block in `../design/` or `../product/`
  - implementing it requires a decision the record doesn't make
- **Never edit the acceptance criteria to match what you built.** Escalate instead. This
  is the highest-severity process violation in the factory: it converts a failed build
  into a silently passing one.
- **Never weaken a gate to make a PR pass.** See `review-policy.md`.
- Scope is the record. Noticed something else worth fixing? File a new intent record.

### Risk levels

| Risk | Examples | Review consequence |
|---|---|---|
| `low` | Copy changes, logging, additive non-nullable-free columns, test additions | Auto-merge if green |
| `medium` | New pipeline stage, ranking changes, UI flows, new endpoints | Auto-merge if green + review agent approves |
| `high` | Schema migrations, auth, cost-model-affecting changes, prompt changes on shared summarisation, anything touching a design invariant | **Human review required** |

Anything the author is unsure about is `high`. The cost of over-classifying is a few
minutes of human attention; the cost of under-classifying is a silent production defect.

---

## Worked example

```markdown
---
id: 0007
title: Back off polling for repeatedly-failing sources
status: ready
author: masood
created: 2026-09-10
design_refs:
  - design/ingestion.md#fetch-pipeline
  - design/data-model.md#shared-tables
risk: medium
---

## Why
A dead or blocked feed is currently retried at its normal cadence forever. It wastes
worker capacity, hammers a publisher that may have blocked us deliberately, and buries
the real signal that a source needs attention.

## What
Sources that fail repeatedly are polled progressively less often, and a source that has
clearly died surfaces to its subscribers rather than failing silently.

## Acceptance criteria
1. A `poll_source` job that fails increments `sources.failure_count` and records the
   error in `sources.last_error`.
2. The next poll is scheduled with exponential backoff based on `failure_count`,
   capped at 24 hours.
3. A successful poll resets `failure_count` to 0 and clears `last_error`.
4. At `failure_count >= 10`, the source is marked dead and every intent subscribing to
   it surfaces a user-visible warning.
5. Backoff is computed from a injected clock, not `datetime.now()`, and is unit-tested
   at failure counts 1, 5, 10, and 25.
6. A source failing then succeeding does not lose articles published during the outage
   (bounded by feed retention).

## Out of scope
- Auto-discovering a replacement source. Separate intent.
- Distinguishing failure *kinds* (404 vs 429 vs parse error). Separate intent —
  though `last_error` should capture enough to make that possible later.

## Design invariants touched
None. This touches `sources` only; no change to `article_summaries`, the shared-enrichment
boundary, or LLM usage.

## Open questions
None.
```

Note what makes this record buildable: every criterion maps to an assertion, the
invariant question is answered explicitly rather than skipped, and "out of scope" heads
off the two adjacent features an agent would otherwise be tempted to build.

## Related
- `quality-gates.md` — what runs against the resulting PR
- `review-policy.md` — what `risk` does
- `../design/` — the binding constraints
