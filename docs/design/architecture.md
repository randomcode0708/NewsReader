# Architecture

Status: DRAFT — 2026-09-10
Phase 1.5. Input: everything in `../product/`. Output: consumed by `../factory/`.

## Stack

> DECISION (2026-09-10, revised): **Python backend + Next.js frontend + PostgreSQL,
> container-hosted on Fly.io.** Supersedes the earlier TypeScript-everywhere decision.
>
> Rationale — in order of weight:
> 1. **Article extraction quality.** `trafilatura` has no real TS equivalent. Extraction
>    gates summary quality, which gates synthesis quality. It is on the critical path.
> 2. **Embeddings.** Self-hosted `sentence-transformers` (see below) is only sane in
>    Python, and every article gets embedded.
> 3. **It removes the serverless-timeout constraint entirely** (see next section).
> 4. Mature data/ML tooling for clustering, evals, and prompt optimisation (DSPy).
>
> **Explicitly NOT the reason: agentic frameworks.** Our pipeline is a deterministic DAG,
> not an agent loop; see "On agent frameworks" below. Choosing Python for LangGraph would
> invite wrapping a fixed pipeline in non-determinism we cannot machine-check.
>
> **Accepted cost:** two languages doubles the factory's surface area — two toolchains,
> two test runners, two CI paths. This is a real tax on `../factory/`. It is contained by
> making the boundary a single generated contract (below), never hand-written types.

| Layer | Choice | Notes |
|---|---|---|
| Backend | Python 3.12+, FastAPI | Ingestion, pipeline, API |
| Frontend | Next.js (App Router), TypeScript | The reading UI |
| **Contract** | **OpenAPI → generated TS client** | **Neither side hand-writes types. Drift is a CI failure.** |
| DB | PostgreSQL + `pgvector` | Unchanged from prior design |
| ORM / migrations | SQLAlchemy 2.x + Alembic | Migrations are reviewable diffs — matters for agent PRs |
| Queue | Postgres-backed job table + long-lived workers | See below |
| Extraction | `trafilatura`, `feedparser` | See `ingestion.md` |
| Embeddings | `sentence-transformers`, self-hosted | See below |
| LLM calls | **Pydantic AI**, behind our gateway | Typed/structured; see "On agent frameworks" |
| LLM batch path | Anthropic SDK direct, if needed | Protects the batch discount; see `llm-and-cost.md` §3b |
| Evals | **Pydantic Evals** | The summarisation-tier eval; feeds `../factory/quality-gates.md` |
| Hosting | Fly.io — web, worker, Postgres | Container-based; no function timeouts |
| Typing/lint | `ruff` + `mypy --strict` / `eslint` + `tsc` | Enforced in CI |

> DECISION (2026-09-10): **The API contract is generated, not written.** FastAPI emits
> OpenAPI; the TS client is generated from it in CI. A hand-edited client, or a drifted
> schema, fails the build. This is what keeps the two-language cost from compounding —
> without it, the factory will silently produce frontend/backend type mismatches that no
> test catches.

> DECISION (2026-09-10): **Self-hosted embeddings via `sentence-transformers`.** Resolves
> the blocking OPEN in `data-model.md`. Every article is embedded, so this is a
> high-volume path: self-hosting makes marginal cost ≈ CPU time, removes a vendor rate
> limit from the ingestion critical path, and makes clustering fully testable offline with
> no network. The model choice fixes the `vector(N)` dimension in the schema — it must be
> recorded in `config` and in `article_embeddings.model_id`, because changing it later
> means re-embedding the entire corpus.

## Execution model

The previous design decomposed all work into tiny slices purely to survive Vercel
Function timeouts. **Container hosting removes that constraint**, and with it a large
amount of incidental complexity.

> DECISION (2026-09-10, revised): **Long-running workers are allowed.** A dedicated
> worker process pulls from the Postgres job queue and runs until done. Jobs are still
> queued, still retried, still bounded — but a single job may take minutes without
> special handling, and a synthesis run does not need to be split across invocations.

Two requirements survive the change, for correctness rather than platform reasons:

- **Idempotency.** Every job handler is keyed by `idempotency_key`; re-running must not
  double-charge an LLM call or duplicate rows. Workers get killed (deploys, OOM, crashes)
  and jobs get retried — this is unavoidable regardless of host.
- **Bounded units.** One job = one article, or one Dossier re-synthesis. Not "crawl
  everything". Keeps retries cheap and failures isolated.

What we drop: cron-tick slicing, resumable mid-job checkpointing, and the "escape hatch to
a container host" contingency — we now start there.

## On agent frameworks

Draw the line at **orchestration runtime vs typed call layer** — not at the word "agent".

> DECISION (2026-09-10): **No orchestration runtime in the core pipeline.** INGEST →
> ENRICH → RANK → SYNTHESISE is a deterministic DAG with known inputs and outputs at every
> step. The Postgres job queue already provides orchestration, retry, and observability.
> Adding LangGraph/CrewAI here buys nothing and costs a heavy dependency plus
> non-determinism in a system whose correctness must be machine-checkable
> (`../factory/quality-gates.md`). Note `AutoGen` is in maintenance mode (merged into
> Microsoft Agent Framework) — do not start there.

> DECISION (2026-09-10): **Use Pydantic AI as the typed LLM call layer**, behind our
> gateway. This *revises* an earlier, over-broad "no agent framework" stance — Pydantic AI
> is a library, not a runtime. It does not take over orchestration; the job queue still
> owns that.
>
> What it buys us, mapped to requirements we already committed to:
> | Requirement | Pydantic AI feature |
> |---|---|
> | Structured outputs everywhere (intent, summaries, source proposals) | Pydantic-model outputs with validation + bounded retry |
> | Machine-checkable correctness, no human diff review | `TestModel` / `FunctionModel` — deterministic agent tests, no network |
> | Eval harness for the summarisation-tier question (`llm-and-cost.md` §3a) | **Pydantic Evals** — datasets, scoring, regression tracking |
> | Provider-agnostic gateway (`llm-and-cost.md` §3b) | 20+ providers behind one interface |
> | Same idiom as the rest of the stack | Pydantic/FastAPI/SQLAlchemy family |
>
> Maturity: v1.0 Sept 2025, v2.0 June 2026. Stable enough to build on.
>
> **Use the typed-call layer everywhere; use the agent loop almost nowhere.** The loop is
> justified only where behaviour is genuinely exploratory:
> - **Source discovery** — search → evaluate → verify → register.
> - **"Ask the dossier"** — open-ended retrieval over an accumulated corpus.
>
> Every other stage calls a typed, single-shot, schema-validated model call. If a PR
> introduces an agent loop into INGEST/ENRICH/RANK/SYNTHESISE, that is a design violation.

> ⚠️ CONSTRAINT: **any abstraction we adopt must not cost us Anthropic Batch API access.**
> The 50% batch discount is load-bearing for the cost model (`llm-and-cost.md` §2a), and
> the batch path is the single highest-volume stage. If Pydantic AI does not expose native
> Anthropic batch, the summarisation stage calls the Anthropic SDK **directly**, behind the
> same gateway. One stage bypassing the abstraction is an acceptable price; silently
> doubling our dominant cost line is not. Verify before building that stage.

> OPEN: Redis for rate-limiting and caching is likely once ingestion scales. Deferred
> until measured.

## Component map

```
   Fly worker       ┌──────────────────────────────────────┐
   process(es) ────▶│  Job queue (Postgres `jobs` table)   │
   (long-lived)     │  claim → work → commit → retry       │
                    └───────────────┬──────────────────────┘
                                    │ dispatches by job kind
        ┌───────────────────────────┼────────────────────────────┐
        ▼                           ▼                            ▼
┌───────────────┐          ┌─────────────────┐          ┌────────────────┐
│ INGEST        │          │ ENRICH          │          │ SYNTHESISE     │
│ poll feeds    │          │ summarise       │          │ Dossier living │
│ extract text  │─────────▶│ embed           │─────────▶│ summary +      │
│ dedupe        │          │ cluster stories │          │ delta          │
└───────┬───────┘          └────────┬────────┘          └───────┬────────┘
        │                           │                           │
        ▼                           ▼                           ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │  PostgreSQL + pgvector    (see data-model.md)                   │
   │  sources · articles · article_summaries(SHARED) · intents ·      │
   │  stories · view_items · dossiers · dossier_sections             │
   └─────────────────────────────────────────────────────────────────┘
        ▲                           ▲                           ▲
        │                           │                           │
┌───────┴───────┐          ┌────────┴────────┐          ┌───────┴────────┐
│ INTENT        │          │ RANK            │          │ FastAPI        │
│ compile NL →  │          │ score items per │          │  └─OpenAPI─┐   │
│ structured    │          │ intent policy   │          │ Next.js UI ◀┘   │
│ discover srcs │          │                 │          │ Edition/Stream │
└───────────────┘          └─────────────────┘          │ /Dossier       │
                                                        └────────────────┘
```

Deployed as three Fly.io processes: **web** (FastAPI), **worker** (queue consumer,
scale-out by count), and **Postgres**. The Next.js frontend is a separate app consuming
the generated client.

## Pipeline stages

1. **INGEST** — poll RSS/Atom per source on a per-source schedule, extract article text,
   dedupe by URL + content hash. Writes `articles`. See `ingestion.md`.
2. **ENRICH** — summarise and embed each article **once, globally**; cluster into
   `stories` (same event across outlets). See the cost note below.
3. **INTENT** — compile NL → structured `Intent` (`../product/intent-model.md`), then run
   source discovery (`../product/source-discovery.md`). Runs on intent create/edit only.
4. **RANK** — score candidate stories against an intent's `ranking_policy`, materialise
   `view_items`.
5. **SYNTHESISE** — for Dossier intents, update the living summary + delta against the
   user's `research_direction`. The expensive stage; see `llm-and-cost.md`.
6. **READ** — render Edition / Stream / Dossier.

## The single most important architectural decision

> DECISION (2026-09-10): **Article enrichment is a globally shared resource; only
> ranking and synthesis are per-user.** An article is fetched, extracted, summarised, and
> embedded **exactly once** regardless of how many users or intents reference it. Its
> summary and embedding live in `article_summaries`, keyed by article, not by user.
>
> Rationale: per-article summarisation is the dominant cost driver
> (`llm-and-cost.md` §2). Per-user summarisation makes unit cost scale with
> `users × articles`; shared summarisation makes it scale with `articles` alone, and the
> marginal cost of a new user collapses to their ranking + synthesis only. This is the
> difference between viable and non-viable economics, and it is very hard to retrofit —
> it must be true from the first commit.
>
> Corollary: nothing user-specific may leak into the shared summarisation prompt. If a
> feature needs an intent-specific angle on an article, that is a *synthesis* concern
> (per-user, Dossier stage), not a summarisation concern.

## Runtime-tunable knobs (required by Phase 1)

`../product/market-analysis.md` deferred pricing and made this a design constraint: the
expensive levers must be **runtime configuration, not hardcoded values**, so pricing
stays free. A `config` table (or env-backed override) exposes at minimum:

| Knob | Affects |
|---|---|
| `model_tier` per pipeline stage | Cost/quality per stage — see `llm-and-cost.md` |
| `summary_depth` | Output tokens per article |
| `synthesis_frequency` per Dossier | How often the living summary is rebuilt |
| `batch_mode` on/off per stage | 50% cost cut, higher latency |
| `effort` per stage | Thinking depth |

No code path may hardcode a model ID or a synthesis cadence.

## Testability requirements (feeds the factory)

Because humans won't review day-to-day diffs (`../factory/`), the architecture must make
correctness machine-checkable:

- **All LLM calls behind a single typed gateway module.** One place to mock, one place to
  record fixtures, one place to meter cost. No bare `Anthropic()` construction in feature
  code. The gateway is also the seam where multi-provider routing plugs in
  (`llm-and-cost.md` §3) — provider choice must never leak into a pipeline stage.
- **All network fetching behind one fetcher module.** Enables replay-based ingestion tests
  with no live HTTP.
- **Pipeline stages are pure-ish functions** `(input, deps) -> output`, with I/O injected —
  so a stage can be tested without a database or a network.
- **Deterministic seeds and frozen clocks** in tests; no `datetime.now()` in business logic
  (also a prompt-cache invalidator — see `llm-and-cost.md`).
- **The API contract is generated and diff-checked in CI.** A drifted OpenAPI schema or a
  hand-edited TS client fails the build. This is the specific control that stops the
  two-language decision from compounding into silent frontend/backend mismatches.
- **`mypy --strict` on the backend, `tsc` on the frontend.** Type errors are build
  failures, not warnings — the factory has no human reading diffs to catch them.

## Related
- `data-model.md` — the schema this implies
- `ingestion.md` — INGEST + source discovery detail
- `llm-and-cost.md` — ENRICH/SYNTHESISE cost model and model tiering
- `../factory/quality-gates.md` — how these requirements get enforced
