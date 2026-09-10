# Quality Gates

Status: DRAFT — 2026-09-10

Every automated check that stands between an agent's PR and production. This file is the
substance behind the claim that a human need not read day-to-day diffs — **if these gates
are weak, that claim is false.**

## Principles

1. **Fail closed.** A gate that errors, times out, or can't run has *failed*. There is no
   "skip on infrastructure flake" path. Flaky gates get fixed, not bypassed.
2. **Agents cannot modify gates.** Enforced structurally — see "Protecting the gates".
3. **Each gate names what it protects.** A gate nobody can justify gets deleted, because
   an unexplained red build trains everyone to override.
4. **Cheap gates first.** Fail in seconds where possible; spend money on evals last.

## Gate ladder

Ordered by cost. Each stage runs only if the previous passed.

| # | Gate | Protects against | Runtime |
|---|---|---|---|
| 0 | Intent binding | Scope creep, spec drift | seconds |
| 1 | Static analysis | Type errors, lint, dead code | seconds |
| 2 | Contract | Frontend/backend drift | seconds |
| 3 | Unit tests | Logic regressions | ~1 min |
| 4 | **Design invariants** | **Architectural erosion** | ~1 min |
| 5 | Integration tests | Wiring, DB, queue | ~5 min |
| 6 | Migration safety | Data loss, irreversible schema | seconds |
| 7 | Security | Secrets, vulns, injection | ~1 min |
| 8 | **Cost regression** | **Silent unit-economics blowout** | ~1 min |
| 9 | **Evals** | **Quality regressions the tests can't see** | ~10 min, costs money |
| 10 | Runtime verification | "Passes CI, doesn't actually work" | ~5 min |

---

## 0. Intent binding

- PR must reference exactly one intent record (`/intents/NNNN-*.md`) with `status: ready`.
- **The record's acceptance criteria must be unmodified relative to the base branch.**
  If the PR edits them, the gate fails outright. Editing criteria to match the
  implementation converts a failed build into a passing one — it is the single most
  dangerous move available to a building agent, so it's blocked mechanically rather than
  discouraged in prose.
- Files changed must be plausibly within the record's stated scope; a PR touching areas
  the record never mentions is flagged for the review agent.

## 1. Static analysis

`ruff` (lint + format), `mypy --strict` on backend; `eslint` + `tsc --noEmit` on frontend.
Warnings are errors — the factory has no human to notice a warning.

Custom lint rules encoding our decisions:
- **No hardcoded model ID** outside the LLM gateway and `config` (`../design/data-model.md`
  invariant 6). Greppable, so it's a rule.
- **No `datetime.now()` / `Date.now()` in business logic** — breaks test determinism *and*
  silently invalidates prompt caches (`../design/llm-and-cost.md`).
- **No direct `Anthropic()` construction** outside the gateway.
- **No agent loop in core pipeline modules** — `../design/architecture.md` restricts loops
  to source discovery and ask-the-dossier.

## 2. Contract

Regenerate the OpenAPI schema and the TS client; `git diff --exit-code`. Any drift fails.
This is the one control keeping the two-language decision from compounding into silent
frontend/backend mismatches (`../design/architecture.md`).

## 3. Unit tests

- Every acceptance criterion in the intent record maps to at least one test. The review
  agent audits this mapping (`review-policy.md`) — coverage percentage alone does not
  prove the *right* things are tested.
- Frozen clock, seeded randomness, no network. A unit test that hits the network is a
  failed unit test.
- LLM calls mocked via Pydantic AI `TestModel` / `FunctionModel`.

## 4. Design invariants ⭐

The seven invariants in `../design/data-model.md` as executable tests. **These are the
gates that stop architectural erosion**, which is exactly the failure mode of a codebase
nobody reads.

| # | Invariant | How it's checked |
|---|---|---|
| 1 | No user/intent identifier in `article_summaries` / `article_embeddings` | Schema assertion + static check on the summarisation prompt builder |
| 2 | `dossiers.research_direction` is `NOT NULL` | Schema assertion + a test that dossier creation without a direction fails |
| 3 | Every `view_items` row has `score_reasons` | Schema `NOT NULL` + integration test |
| 4 | Every LLM call writes an `llm_usage` row | Integration test counting rows before/after a pipeline run |
| 5 | Job handlers are idempotent | **Property test: run every handler twice, assert identical DB state and no second LLM charge** |
| 6 | No hardcoded model ID outside gateway/`config` | Lint rule (gate 1) |
| 7 | `article_embeddings.model_id` matches configured model | Assertion on write + drift check |

Invariant 1 deserves special note: violating it silently destroys the cost model
(`../design/llm-and-cost.md` §2) without breaking a single feature. Nothing would look
wrong until the bill arrived. That is precisely the class of defect a human diff-reviewer
would also miss — so it must be a test.

## 5. Integration tests

Real Postgres (ephemeral container), real queue, mocked LLM and HTTP.
- Full pipeline on a fixture corpus: feeds → articles → summaries → stories → view items.
- Ingestion replay: recorded HTTP fixtures for feeds and article pages, including the
  ugly ones — malformed XML, paywalls, 429s, navigation-chrome-instead-of-article.
  These are the documented failure modes in `../design/ingestion.md`; each needs a case.
- Worker crash/restart mid-job → no duplicate rows, no double charge.

## 6. Migration safety

- Every Alembic migration has a tested `downgrade`.
- **Destructive operations** (`DROP`, `ALTER … TYPE`, `NOT NULL` on an existing populated
  column) auto-escalate to human review (`review-policy.md`), regardless of stated risk.
- Migration runs against a copy of production-shaped data, not just an empty schema.
- Two-phase rule: schema change and code change that depends on it ship as separate PRs,
  so either can roll back independently.

## 7. Security

- Secret scanning on the diff; no credential ever committed.
- Dependency audit (`pip-audit`, `npm audit`); new high-severity CVE fails.
- **Prompt-injection surface review** — article text is *untrusted input* that flows into
  LLM prompts. Any PR that changes how fetched content enters a prompt is flagged for
  human review. A hostile publisher embedding instructions in an article is a real threat
  to a system that summarises and synthesises unattended.
- No new outbound network destination without an explicit intent record.

## 8. Cost regression ⭐

The cost model is a design constraint (`../design/llm-and-cost.md`), so it's a gate.

- The pipeline integration run records token usage per stage; CI compares against a
  committed baseline. **A >20% per-article or per-synthesis increase fails**, and must be
  either justified in the intent record or fixed.
- Batch-path assertion: the summarisation stage must actually use the Batch API. If a
  refactor silently drops batching, our dominant cost line doubles with no other symptom.
- New LLM call sites outside the gateway fail gate 1; new call sites *inside* the gateway
  must declare a stage and tier.

## 9. Evals ⭐ — quality regressions tests can't see

Implemented with **Pydantic Evals** (`../design/architecture.md`). This is the gate that
distinguishes "the code runs" from "the product is good", and no unit test substitutes
for it.

| Eval | Measures | Baseline |
|---|---|---|
| **Summarisation quality** | Faithfulness to source, coverage of key claims, no hallucinated facts | Committed scores per model tier |
| **Downstream synthesis quality** | Dossier quality *given* those summaries — the compounding-error check | Committed |
| **Story clustering** | Precision/recall on a hand-labelled multi-outlet corpus | Committed |
| **Intent compilation** | Does NL → structured Intent produce the expected primitive/topics/direction-ask? | Golden set |
| **Ranking** | Are the right stories surfaced for a known intent? | Golden set |

Rules:
- Evals run on any PR touching prompts, model config, ranking, or extraction; smoke-only
  otherwise (they cost money and take minutes).
- **A quality regression fails the build**, exactly like a type error.
- Evals are non-deterministic. Use score thresholds with tolerance bands, not exact
  matches, and require N runs for a regression to count — a single bad sample is noise,
  not a signal. A gate that flaps gets ignored, which is worse than no gate.
- **The summarisation-tier decision** (`../design/llm-and-cost.md` §3a — open-weight vs
  Haiku) is settled by this harness, and stays a standing measurement rather than a
  one-time call.

## 10. Runtime verification

CI green is not proof the thing works. Deploy the PR to an ephemeral environment and
exercise the actual changed flow — for a UI change, drive it; for a pipeline change, run
real articles through and inspect the output. Post-deploy: health checks, error-rate and
cost-per-hour monitors, **auto-rollback on breach**.

---

## Protecting the gates

The obvious failure mode: an agent, unable to make a PR pass, makes the gate pass instead.
Blocked structurally:

1. **CODEOWNERS** on `.github/workflows/`, `docs/factory/`, all gate config, eval
   baselines, and cost baselines → human approval required.
2. **A PR that modifies a gate, a baseline, or a threshold always requires human review**,
   regardless of risk level (`review-policy.md`). No exceptions, including "the baseline
   was wrong".
3. **Branch protection**: required status checks, no force-push, no admin bypass by
   automation.
4. **Deleting or skipping a test** (`@pytest.mark.skip`, `.skip(`, removed assertions) is
   detected in the diff and escalates to human review.

> If you are an agent reading this and a gate is blocking you: **escalate. Do not weaken
> the gate, relax a threshold, skip a test, or edit a baseline.** A blocked PR is a normal
> outcome and costs the project very little. A weakened gate is permanent damage to the
> only thing making unattended development safe.

## Related
- `review-policy.md` — what happens after the gates pass
- `intent-record-template.md` — the spec these check against
- `../design/data-model.md` — source of the invariants in gate 4
- `../design/llm-and-cost.md` — source of the baselines in gate 8
