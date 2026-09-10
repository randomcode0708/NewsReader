# The Agent Development Factory

Status: DRAFT — 2026-09-10
Phase 2. Input: `../design/`. This directory defines **how software gets built**, not what.

## The goal, stated precisely

> **Intent record in → reviewed, tested, deployed PR out. The human reviews the intent
> records and the gates — not the day-to-day diffs.**

That last clause is the whole design constraint. If a human is not reading every diff,
then **every property we care about must be machine-checkable**, and the checks must be
adversarial enough that "all green" genuinely means "safe to merge".

## The honest risk

An agent factory without adequate gates does not produce no output — it produces
*confidently wrong* output at high velocity, which is worse than no output because it
looks finished. Anything that cannot be checked by a machine must either become
checkable, or stay on the human's desk. There is no third option.

We therefore define the human's real job as **three narrow, high-leverage roles**:

| Role | What the human does | Frequency |
|---|---|---|
| **Author** | Writes/approves intent records — the *what* and *why* | Per feature |
| **Gate-keeper** | Owns the quality gates themselves; reviews changes *to the gates* | Rare, high-scrutiny |
| **Escalation** | Handles what the gates flag as un-automatable | Exception only |

The human never becomes a diff reviewer. But they do own the gates, and **a PR that
weakens a gate is the one PR type that always gets human review** (`review-policy.md`).

## Flow

```
  ┌──────────────────┐
  │ 1. INTENT RECORD │  human-authored (or agent-drafted, human-approved)
  │  /intents/*.md   │  the what + why + acceptance criteria
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 2. PLAN          │  agent restates intent as a change plan
  │                  │  ── STOPS if the intent is ambiguous or violates a design invariant
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 3. BUILD         │  agent implements on a branch, one intent = one PR
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 4. GATES         │  quality-gates.md — types, tests, invariants, evals, cost,
  │                  │  security, migrations. All must pass. No overrides by agents.
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 5. AUTO-REVIEW   │  adversarial agent review against the intent + design docs
  │                  │  ── a DIFFERENT agent than the one that wrote it
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 6. MERGE         │  auto-merge if green AND not in the human-review list
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ 7. VERIFY        │  post-deploy checks; auto-rollback on failure
  └──────────────────┘
```

## Files

| File | Contents |
|---|---|
| `intent-record-template.md` | The unit of work. One record → one PR. |
| `quality-gates.md` | Every automated check, and what each one protects |
| `review-policy.md` | What auto-merges, what escalates to a human, and why |

## Principles

1. **The intent record is the specification.** If behaviour isn't in the record or the
   design docs, it isn't in scope. Agents don't get to invent requirements.
2. **Design invariants are enforced, not documented.** The seven invariants in
   `../design/data-model.md` are tests, not prose.
3. **The reviewer is not the author.** Self-review by the generating agent is worth very
   little; review runs as a separate pass with fresh context.
4. **Gates fail closed.** A gate that errors is a gate that failed. No "skip on timeout".
5. **Agents cannot weaken their own gates.** Enforced by `review-policy.md` +
   CODEOWNERS.
6. **Cost and quality are gates, not dashboards.** A PR that regresses summary quality or
   blows the cost model fails, the same as a type error.

## Related
- `../design/` — the technical constraints these gates enforce
- `../product/` — the intent behind the software
- `/intents/` — the live queue of intent records
