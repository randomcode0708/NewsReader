# The Agent Development Factory

Status: DRAFT — 2026-09-10
Phase 2. Input: `../design/`. This directory defines **how software gets built**, not what.

## The goal, stated precisely

> **Spec in → tested, reviewed, deployed PR out. The human's day is specs, clarifications,
> and plans — not diffs.**

That last clause is the design constraint. If a human is not reading every diff, then
**every property we care about must be machine-checkable**, and the checks must be
adversarial enough that "all green" genuinely means "safe to merge".

## The honest risk

A 2026 industry finding: **~75% of AI coding agents broke working code in CI workflows.
Review capacity, not model capability, is the binding constraint.**

An agent factory with weak gates does not produce no output — it produces *confidently
wrong* output at high velocity, which is worse than nothing because it looks finished.

Two consequences, both baked into this directory:

1. **We start at Stage A of the autonomy ladder** — full spec→code automation, but every
   PR is human-reviewed while the gates prove themselves (`sdlc.md`).
2. **Anything not machine-checkable either becomes checkable or stays on the human's
   desk.** There is no third option.

## We did not invent this

> DECISION (2026-09-10): **GitHub Spec Kit** for the spec→code lifecycle, **Claude Code +
> `anthropics/claude-code-action`** as the agent, **our own gates** on top. Full rationale
> and rejected alternatives in `sdlc.md`.

An earlier draft of this directory specified a bespoke intent-record format. That was
largely reinventing Spec Kit with fewer phases — notably missing a first-class
**constitution** and the **`/clarify`** phase. Replaced.

What is genuinely ours, because no framework can supply it: the **constitution** (our
design invariants as binding law), the **gates** (design-invariant tests, cost regression,
evals), and the **review policy**.

## The human's job

Not "nothing" — three narrow, high-leverage roles:

| Role | What they do | Frequency |
|---|---|---|
| **Author** | `/specify`, answer `/clarify`, approve `/plan` | Per feature |
| **Gate-keeper** | Own the gates; review changes *to* the gates | Rare, high-scrutiny |
| **Escalation** | Handle what the gates flag as un-automatable | Exception |

The target state is never becoming a diff reviewer. At Stage A they still are — temporarily,
and for a specific purpose: **calibrating the gates by finding what they miss.**

## Files

| File | Contents |
|---|---|
| **`sdlc.md`** | **The day-to-day lifecycle. Start here.** Framework choice, the loop, the autonomy ladder. |
| `constitution.md` | The non-negotiable rules, each with the failure it prevents |
| `quality-gates.md` | Every automated check and what it protects |
| `review-policy.md` | What auto-merges, what escalates, and why |

## Principles

1. **The spec is the specification.** Not in the spec or `../design/`? Out of scope.
2. **Design invariants are enforced, not documented.** They are tests, not prose.
3. **The reviewer is not the author.** Self-review in the same session is mostly
   rationalisation.
4. **Gates fail closed.** A gate that errors has failed. No skip-on-timeout.
5. **Agents cannot weaken their own gates.** Enforced structurally, not by policy.
6. **Cost and quality are gates, not dashboards.** A cost blowout or an eval regression
   fails the build like a type error.
7. **Leverage is upstream.** If you're spending your day reading diffs, fix the
   constitution or the clarify step — not your reading speed.

## Related
- `../design/` — the technical constraints these enforce
- `../product/` — the intent behind the software
- `/specs/` — the live queue of feature specs
