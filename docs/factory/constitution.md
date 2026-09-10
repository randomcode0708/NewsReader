# Constitution

Status: DRAFT — 2026-09-10

**The non-negotiable rules for this codebase.** Wired into Spec Kit as the project
constitution (`/constitution`) and read by every agent on every task.

These are not style preferences. Each rule exists because breaking it causes a specific,
named failure — usually one that **no test would catch and no diff review would notice**,
which is precisely why it must be law rather than convention.

> Changing this file requires human review, always (`review-policy.md`). An agent may
> propose an amendment in a spec; it may not merge one.

---

## I — Architecture

**1. Article enrichment is globally shared. Ranking and synthesis are per-user.**
An article is fetched, extracted, summarised, and embedded **exactly once**, regardless of
how many users or intents reference it. Nothing user-specific or intent-specific may enter
`article_summaries`, `article_embeddings`, or the summarisation prompt.
*Why:* per-user summarisation makes unit cost scale with `users × articles` instead of
`articles`. **Violating this breaks nothing visible — no test fails, no feature misbehaves.
It shows up only on the bill.** See `../design/llm-and-cost.md` §2.

**2. No LLM call outside the gateway.**
No direct client construction in feature code. Stages request a *capability tier*, never a
vendor or a model ID. *Why:* the gateway is the only place cost is metered, providers are
routed, and calls are mockable. A bypass silently defeats all three.

**3. No hardcoded model IDs.** They live in `config` and the gateway. *Why:* model tier is a
runtime-tunable cost knob — a Phase 1 requirement for keeping pricing decisions open
(`../product/market-analysis.md`).

**4. No agent loop in the core pipeline.**
INGEST → ENRICH → RANK → SYNTHESISE are deterministic, single-shot, schema-validated calls.
Agent loops are permitted **only** in source discovery and ask-the-dossier.
*Why:* the pipeline's correctness must be machine-checkable; loops make it
non-reproducible. See `../design/architecture.md`.

**5. Every job handler is idempotent.** Keyed by `idempotency_key`. Running twice must not
duplicate rows or double-charge an LLM call. *Why:* workers get killed by deploys, OOM, and
crashes. Retries are not exceptional; they are the normal case.

**6. The API contract is generated, never hand-written.**
FastAPI → OpenAPI → TS client. A hand-edited client or a drifted schema fails CI.
*Why:* it is the single control preventing the two-language decision from producing silent
frontend/backend mismatches that no test catches.

## II — Product invariants

**7. A Dossier always has a user-supplied `research_direction`.** `NOT NULL`. Never
inferred, never defaulted, never auto-created from Edition/Stream activity.
*Why:* unsteered accumulation produces a generic topic summary — exactly the commodity
output the product exists to differentiate from. See `../product/view-primitives.md`.

**8. Ranked items always carry `score_reasons`.** Every surfaced item can explain why it
appeared. *Why:* transparency is a product promise and the trust difference from black-box
feeds (`../product/intent-model.md`) — so it is a schema obligation, not a feature.

**9. Compiled intent structure is always visible and editable to the user.** NL is the
input, not a black box. *Why:* same promise as above.

**10. Never block a user on source selection; never limit them either.** Discovery
proposes; manual entry (URL and name search) is a peer path, not a fallback.
See `../product/source-discovery.md`.

## III — Sourcing ethics and law

**11. Always link back to the original article.** Every surfaced item carries its source
and URL.

**12. Summarise; never republish.** Our own summary plus at most a short quoted span. Never
the full article body.

**13. `extracted_text` is a working copy, not a library.** Used to produce a summary, then
TTL'd.

**14. Honour `robots.txt`, feed terms, and per-host rate limits.** Identify with a real
User-Agent and contact URL. Never be the reason a small publisher's server falls over.

**15. Never attempt to bypass a paywall.** Detect, mark, store the feed excerpt only.

*Why (11–15):* this is the publisher-friendly posture the market now expects, and the legal
footing the product stands on. An agent optimising for "better summaries" would otherwise
find full-text scraping an attractive shortcut.

## IV — Correctness discipline

**16. Determinism in business logic.** No `datetime.now()`, no unseeded randomness. Clocks
and randomness are injected. *Why:* breaks test reproducibility **and** silently invalidates
prompt caches.

**17. Untrusted input is untrusted.** Article text is fetched from the open internet and
flows into LLM prompts. Treat it as hostile: a publisher can embed instructions. Any change
to how fetched content enters a prompt requires human review.

**18. Every LLM call writes an `llm_usage` row.** Cost observability is a build
requirement, not an afterthought.

**19. Type checks are build failures.** `mypy --strict`, `tsc`, `ruff`. Warnings are errors —
there is no human reading output to notice a warning.

## V — Process (binding on agents)

**20. The spec is the specification.** If behaviour isn't in the spec or in `docs/design/`,
it is out of scope. Do not invent requirements. Do not build the adjacent feature you
noticed. File a new spec.

**21. Never edit acceptance criteria to match what you built.** Escalate instead.
*Why:* it converts a failed build into a silently passing one. This is the single most
damaging action available to a building agent.

**22. Never weaken a gate, threshold, baseline, or test to make a PR pass.**
Do not add skip markers, relax thresholds, delete assertions, or re-baseline evals.
*Why:* a blocked PR costs the project almost nothing. A weakened gate is permanent damage
to the only thing making unattended development safe.

**23. Stop and escalate rather than guess.** When an acceptance criterion is ambiguous, when
the work would violate a rule in this constitution, or when the spec conflicts with a
`> DECISION` block in `docs/design/` or `docs/product/` — stop. Being blocked is a normal,
cheap outcome.

**24. A rule in this constitution outranks a spec.** If a spec asks for something this file
forbids, the spec is wrong. Escalate; do not comply.

---

## Enforcement

Most of these are mechanically checked — see `quality-gates.md`. Rules 1–3, 5, 7–8, 16, 18
map to gate 4 (design invariants) and gate 1 (lint). Rules 20–24 are process, enforced by
gate 0 (intent binding), CODEOWNERS, and `review-policy.md`.

A rule that cannot be checked mechanically is a rule that will eventually be broken. When
adding one, add its check in the same PR.

## Related
- `sdlc.md` — how this is wired into the workflow
- `quality-gates.md` — the mechanical enforcement
- `../design/data-model.md` — source of the schema-level invariants
