# Review Policy

Status: DRAFT — 2026-09-10

What happens after the gates pass. Defines exactly what auto-merges, what a review agent
inspects, and what always reaches a human.

## Two-stage review

**Stage 1 — automated review agent.** Runs on every PR that clears the gates.
**Stage 2 — human.** Runs only on the escalation list below.

> DECISION (2026-09-10): **The reviewing agent is never the authoring agent, and starts
> with fresh context.** A model reviewing its own work in the same session mostly
> rationalises it — it has already committed to the approach and will defend it. The
> reviewer gets the intent record, the diff, and the design docs; **not** the author's
> reasoning or session history.

## Stage 1 — the review agent

Its job is **adversarial**: find reasons this PR should not merge. A review that says
"looks good" without evidence of having tried to break it is a failed review.

Checklist, in priority order:

1. **Acceptance-criteria audit.** For each criterion in the intent record: is it actually
   implemented, and is there a test that would fail if it broke? Name the test. A
   criterion with no corresponding test is a blocking finding — this is the single
   highest-value check the reviewer performs.
2. **Scope audit.** Anything in this diff not required by the record? Unrequested
   refactors, speculative abstractions, "while I was in here" changes → blocking.
3. **Design-invariant audit.** Re-check the seven invariants against the *diff*, not just
   the test results. Tests can be wrong; the reviewer reads the code.
4. **Decision conflicts.** Does anything contradict a `> DECISION` block in `../design/`
   or `../product/`? Quote the decision and the conflicting code.
5. **Failure modes.** What breaks with an empty feed, a 10MB article, a duplicate job, a
   dead source, a hostile publisher, a clock skew? Are the documented failure modes in
   `../design/ingestion.md` handled?
6. **Cost implications.** Any new LLM call, loop over articles, or prompt growth? Does it
   match the cost gate's assumptions?
7. **Test quality, not test count.** Do the tests assert on behaviour, or do they assert
   that the code does what the code does? Tautological tests are a blocking finding.

Output: `approve` or `request-changes` with specific, actionable findings. The reviewer
may **not** approve a PR whose findings it hasn't resolved, and may not merge.

> The review agent must not be given the ability to modify gates, baselines, or branch
> protection. Its only outputs are an approval decision and comments.

## Stage 2 — human review escalation list

A PR reaches a human if **any** of these are true. This is the boundary of the
"human doesn't read diffs" claim, and it is deliberately wide at the start.

| Trigger | Why |
|---|---|
| `risk: high` in the intent record | Author's judgement |
| **Modifies any gate, threshold, eval baseline, or cost baseline** | The one PR type that can disable the safety net |
| **Modifies `docs/factory/` or `docs/design/`** | Changing the rules ≠ following them |
| **Destructive migration** (`DROP`, type change, `NOT NULL` on populated column) | Irreversible data loss |
| **Deletes or skips tests**, or removes assertions | Indistinguishable from hiding a failure |
| Touches auth, secrets, or permissions | Blast radius |
| **Changes the shared summarisation prompt** | Affects every user's corpus and the cost model at once |
| **Changes how fetched content enters a prompt** | Prompt-injection surface (`quality-gates.md` §7) |
| Review agent requested changes twice on the same PR | The loop isn't converging; a human should look |
| Any gate was overridden | Should be impossible; if it happened, investigate |
| Diff exceeds a size threshold | Large diffs defeat automated review — split the intent |

Everything else, with green gates and an agent approval, **auto-merges**.

## Merge rules

- **One intent record = one PR.** Squash-merge; the commit message references the record.
- Auto-merge requires: all gates green **and** review-agent approval **and** not on the
  escalation list.
- Human-review PRs never auto-merge, even when green.
- A PR open >7 days goes stale → close and re-plan. Long-lived agent branches rot against
  a moving main.

## Post-merge

- Deploy to staging automatically; production deploys on a schedule or on demand.
- **Auto-rollback** on health-check failure, error-rate spike, or cost-per-hour breach.
- A rollback **auto-files a new intent record** describing what regressed. Failures
  re-enter the queue as work, rather than relying on someone remembering.

## Escalation protocol (for agents)

When blocked, an agent must **stop and surface**, never work around. Blocked is a normal,
cheap outcome; a silent workaround is expensive and often invisible.

| Situation | Action |
|---|---|
| Ambiguous acceptance criterion | Stop. Comment on the intent record with the specific ambiguity and the interpretations. |
| Would violate a design invariant | Stop. Escalate — invariants are not overridable by an intent record. |
| Intent conflicts with a design `> DECISION` | Stop. The conflict is a decision for the human, not a thing to resolve unilaterally. |
| A gate fails and you can't fix the code | Stop. **Never** weaken the gate. |
| The fix requires a design change | Stop. Design changes are their own PR with human review. |
| You noticed an unrelated problem | File a new intent record. Do not fix it here. |

## Reviewing this policy

The escalation list starts deliberately wide. Narrowing it is legitimate — but only
**with evidence**: a category of change that has auto-merged cleanly many times, with no
incidents, is a candidate for removal.

> DECISION (2026-09-10): **The escalation list is narrowed only by a human, backed by
> data, never by an agent proposing it's ready.** Narrowing this list is the highest-
> leverage way to break the factory, because it looks like progress.

## Related
- `quality-gates.md` — what must pass before review
- `intent-record-template.md` — the `risk` field driving escalation
- `README.md` — the human's three roles
