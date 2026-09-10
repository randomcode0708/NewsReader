# The Development Lifecycle

Status: DRAFT — 2026-09-10

**What an engineer actually does, day to day.** This is the operating manual; the other
files in this directory are the rules it runs under.

## Framework choice

> DECISION (2026-09-10): **Adopt [GitHub Spec Kit](https://github.com/github/spec-kit) as
> the spec→code framework, driven from Claude Code, with
> [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) as the
> autonomous builder.** We do not invent our own intent→code scaffolding.
>
> Rationale: Spec Kit's seven phases (Constitution → Specify → Clarify → Plan → Tasks →
> Analyze → Implement) already implement the flow this project needs, including two things
> an earlier bespoke design under-specified: a first-class **constitution** for
> non-negotiable rules, and a **`/clarify`** phase that kills ambiguity *before* code is
> written rather than during review. It is open-source, CLI-based, and tool-agnostic
> (30+ integrations), so lock-in is low. Reported ~3–10× first-pass success improvement on
> non-trivial tasks.
>
> Rejected: **AWS Kiro** — an entire IDE, which means leaving Claude Code; its EARS
> requirements notation is worth borrowing, but not at that price. **Tessl** —
> "spec-as-source, code never hand-edited" is the most aggressive position in the category
> and carries too much vendor risk for this project. **Bespoke** — we tried; it was
> reinventing Spec Kit with fewer phases.
>
> What stays ours: `quality-gates.md` and `review-policy.md`. No framework knows our
> shared-enrichment invariant, our cost baselines, or our eval thresholds.

## The loop

```
╔═ LOCAL — your actual day, in Claude Code ═══════════════════════════════╗
║                                                                          ║
║  /constitution   ONCE. Encodes docs/design invariants as binding rules.   ║
║                  Changing it later = high-scrutiny human review.         ║
║        │                                                                 ║
║        ▼                                                                 ║
║  /specify        Describe the feature BEHAVIOURALLY. What is true after   ║
║                  this ships that isn't now? Not how.                     ║
║        │                                                                 ║
║        ▼                                                                 ║
║  /clarify        Agent asks; you answer. ⭐ THE HIGHEST-LEVERAGE STEP.    ║
║                  Every ambiguity resolved here is a PR that doesn't       ║
║                  get built wrong. Do not skip it to save time.          ║
║        │                                                                 ║
║        ▼                                                                 ║
║  /plan           Agent proposes a technical plan. YOU READ THIS.          ║
║                  Checked against docs/design/. Cheapest place to catch    ║
║                  a wrong approach — before any code exists.              ║
║        │                                                                 ║
║        ▼                                                                 ║
║  /tasks          Decompose into small, reviewable, independent units.     ║
║        │                                                                 ║
║        ▼                                                                 ║
║  /taskstoissues  → GitHub issues. Handoff point.                          ║
╚════════│═════════════════════════════════════════════════════════════════╝
         ▼
╔═ AUTONOMOUS — no human in the loop ═════════════════════════════════════╗
║  claude-code-action   issue → branch → implementation → PR               ║
║        ▼                                                                 ║
║  CI gates             quality-gates.md — 10 stages, fail closed          ║
║        ▼                                                                 ║
║  review agent         adversarial, fresh context, never the author       ║
║        ▼                                                                 ║
║  merge decision       review-policy.md — currently Stage A: ALL to you   ║
╚══════════════════════════════════════════════════════════════════════════╝
```

**Your day-to-day is the top box.** Write specs, answer clarifying questions, approve
plans, watch the queue. You touch diffs only where policy escalates them.

## Where the leverage actually is

Not in the code generation — that is the commodity part now. It's in:

| Step | Why it matters most |
|---|---|
| **`/constitution`** | Written once, constrains every future PR. Highest leverage per minute spent in the entire project. |
| **`/clarify`** | An ambiguity resolved here costs one sentence. The same ambiguity discovered in review costs a rebuild; discovered in production, an incident. |
| **`/plan` review** | Reading a plan is minutes; reading the resulting diff is an hour. Reject wrong approaches at the plan. |

Corollary: **if you're spending your day reading diffs, something upstream is broken.**
The fix is a better constitution or a more thorough clarify — not faster reading.

## The autonomy ladder

> DECISION (2026-09-10): **Start at Stage A — every PR gets human review.** Advance only
> on evidence.
>
> Rationale: a 2026 finding that **~75% of AI coding agents broke working code in CI
> workflows** — review capacity, not model capability, is the binding constraint. Our
> gates are unproven until they have run against real failures on this codebase. Starting
> permissive means discovering the gates are inadequate in production.

| Stage | Auto-merge | Exit criterion to advance |
|---|---|---|
| **A** *(current)* | Nothing. Full spec→code automation, human reviews every PR. | ≥20 merged PRs, and every defect that reached review was **also** caught by a gate — proving the gates, not the human, are doing the work. |
| **B** | `risk: low` only — docs, tests, logging, copy. | ≥20 more, no low-risk incident. |
| **C** | Anything green and off the escalation list (`review-policy.md`). | — |

At Stage A the human is still reading diffs — deliberately, and temporarily. The point of
Stage A is not to review code; it is to **calibrate the gates by finding what they miss.**
Every defect you catch that a gate didn't is a gate bug: fix the gate, not just the PR.

> Advancement is a human decision backed by incident data. An agent may not propose that
> the project is ready to advance — see `review-policy.md`.

## Repository layout

```
specs/                      Spec Kit working area — one dir per feature
  001-source-backoff/
    spec.md                 /specify + /clarify output — the WHAT and WHY
    plan.md                 /plan output — the technical approach
    tasks.md                /tasks output — the decomposition
docs/factory/constitution.md   The binding rules (also wired to Spec Kit)
CLAUDE.md                   Repo conventions the action reads on every run
.github/workflows/          Gates + the claude-code-action wiring
```

Specs are committed and kept after merge. The spec history records *why*, which the git
history alone does not.

## What still needs a human, permanently

Spec Kit and the action automate *construction*. They do not automate:

- **Deciding what to build** — `/specify` needs an author with product judgement.
- **Resolving genuine ambiguity** — `/clarify` asks; someone has to actually know.
- **Owning the gates** — see `review-policy.md`; agents cannot modify their own gates.
- **Judging quality regressions** — an eval score drop is a signal; deciding whether it's
  acceptable is a product call.
- **Design changes** — anything contradicting a `> DECISION` in `docs/design/`.

## Related
- `constitution.md` — the rules agents are bound by
- `quality-gates.md` — the automated checks
- `review-policy.md` — merge vs escalate, and the Stage A definition
- `../design/` — the technical constraints the constitution encodes
