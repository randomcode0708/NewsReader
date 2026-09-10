# NewsReader — Documentation Index

This `docs/` tree is the **single source of truth** for the project. It is written
to be consumed by AI agents as much as by humans. Every artifact is self-contained
and cross-links to related artifacts by relative path.

## Reading order

1. `product/` — **Phase 1: Product & validation** (what we're building and why)
2. `design/` — **Phase 1.5: Technical design** (how it's architected — the input to the factory)
3. `factory/` — **Phase 2: Agent development factory** (how software gets built)

## Directory map

```
docs/
  product/
    vision.md            Researcher-in-a-news-reader's-face thesis, success metrics
    market-analysis.md   Competitor teardown, TAM/SAM/SOM, positioning, monetization
    personas.md          Who we build for (multi-persona)
    source-discovery.md  Intent -> suggested domains -> suggested sites -> user selection
    view-primitives.md   The 3 core view types (Edition / Stream / Dossier); Dossier is the heart
    intent-model.md      How user intent becomes a structured query + ranking policy
  design/                Phase 1.5 — technical design (input to the factory)
    architecture.md      Stack, pipeline stages, shared-vs-per-user split, job queue
    data-model.md        Authoritative schema + invariants the factory must enforce
    ingestion.md         RSS-first crawler, dedupe, source discovery/registry, legal posture
    llm-and-cost.md      Model pricing, cost per user by usage, model tiering, caching/batch
  factory/               Phase 2 — how software gets built (not what)
    README.md            The factory flow, the human's three roles, the honest risk
    intent-record-template.md  The unit of work: one record -> one PR
    quality-gates.md     Every automated check and what it protects
    review-policy.md     What auto-merges, what escalates to a human
intents/                 One file per intent record; each drives a PR
```

## Conventions

- **Status header**: every artifact starts with `Status: DRAFT | REVIEW | LOCKED` and a date.
- **Dates are absolute** (e.g. `2026-09-10`), never relative.
- **Decisions** are recorded inline as `> DECISION (date): ...` blocks so agents can trace rationale.
- **Open questions** are tagged `> OPEN: ...` so they are greppable.
- Cross-references use relative paths, e.g. `see product/intent-model.md`.
