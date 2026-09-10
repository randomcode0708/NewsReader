# LLM Usage & Cost Model

Status: DRAFT — 2026-09-10

Per the deferred-pricing decision in `../product/market-analysis.md`, this document
**does not design to a price ceiling.** It reports **cost per user as a function of
usage**, so that when pricing is decided the economics are already characterised.

## 1. Model facts (verified 2026-09-10 against the Anthropic API reference)

| Model | ID | Context | Input $/1M | Output $/1M |
|---|---|---|---|---|
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | $5.00 | $25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | $3.00 ($2.00 intro thru 2026-08-31) | $15.00 ($10.00 intro) |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | $1.00 | $5.00 |

Cost levers available:
- **Batch API** — 50% off all token usage. Async; most batches finish within 1 hour, max 24h.
  Up to 100k requests per batch. **Ideal for our overnight enrichment**, which is not
  latency-sensitive.
- **Prompt caching** — cache reads ~0.1×, writes 1.25× (5-min TTL) or 2× (1-hour TTL).
  Break-even at 2 requests (5-min) / 3 requests (1-hour).
- **`effort`** (`low`|`medium`|`high`|`xhigh`|`max` inside `output_config`) — controls
  thinking depth and total token spend.
- **Adaptive thinking** — `thinking: {type: "adaptive"}`. Note it must be set
  *explicitly* on Opus 4.8; omitting the field runs **without** thinking.

## 2. Where the money goes

Two workloads dominate, and they have completely different scaling behaviour:

| Workload | Scales with | Shared or per-user? |
|---|---|---|
| **Article enrichment** (summarise + embed) | articles ingested | **SHARED globally** |
| **Dossier synthesis** (living summary + delta) | dossiers × cadence | **Per-user** |
| Intent compilation | intent creates/edits | Per-user, negligible |
| Ranking | can be non-LLM (embeddings + heuristics) | Per-user, near-zero |

> DECISION (2026-09-10): Enrichment is shared, per `architecture.md`. **This is the
> load-bearing cost decision.** Below, "shared" costs are platform costs amortised across
> all users; only "per-user" costs scale with headcount.

### 2a. Per-article enrichment cost

Assumptions: ~1,000-word article ≈ 1,400 input tokens + ~300 token prompt ≈ **1,700 in**;
summary ≈ **150 out**.

| Model | Standard | With Batch API (−50%) |
|---|---|---|
| Opus 4.8 | $0.0123 | **$0.0062** |
| Sonnet 5 | $0.0074 | **$0.0037** |
| Haiku 4.5 | $0.0025 | **$0.0012** |

At a platform ingesting **5,000 articles/day** (a broad shared corpus):

| Model (batch) | Cost/day | Cost/month |
|---|---|---|
| Opus 4.8 | $31 | ~$930 |
| Sonnet 5 | $18.50 | ~$555 |
| Haiku 4.5 | $6 | ~$180 |

**This is a platform cost, not a per-user cost.** It is flat whether we have 1 user or
10,000 — which is exactly why the shared-enrichment decision matters. Per-user
summarisation at the same volumes would multiply these figures by the user count.

### 2b. Dossier synthesis cost (per-user)

Assumptions: current living summary (~2,000 in) + new items since last run (~2,000 in) +
prompt ≈ **5,000 in**; rewritten summary + delta ≈ **1,500 out**.

| Model | Per re-synthesis | Daily cadence, 1 dossier/month |
|---|---|---|
| Opus 4.8 | $0.0625 | **~$1.88** |
| Sonnet 5 | $0.0375 | ~$1.13 |
| Haiku 4.5 | $0.0125 | ~$0.38 |

Prompt caching applies well here: the living summary is a stable prefix re-read on every
run, so cache reads at ~0.1× cut the input side materially once the prefix exceeds the
minimum cacheable size (**4,096 tokens on Opus 4.8** — a short living summary will
silently not cache).

### 2c. Cost per user as a function of usage

Per-user marginal cost ≈ **Dossier synthesis only** (enrichment is amortised, ranking is
near-zero). With Opus 4.8 synthesis:

| User profile | Dossiers | Cadence | Marginal $/user/month |
|---|---|---|---|
| Casual (Edition + Stream only) | 0 | — | **~$0.00** |
| Light researcher | 1 | daily | **~$1.90** |
| Typical researcher | 3 | daily | **~$5.60** |
| Heavy researcher | 10 | daily | **~$19** |
| Heavy, 6-hourly | 10 | 4×/day | **~$75** |

> Read this next to the deferred-pricing note in `../product/market-analysis.md`. It says
> plainly: **free-tier Edition/Stream users are nearly free to serve, and Dossier users
> are the entire variable cost.** That is a strong independent argument for the Phase 1
> decision that Dossier is the paid-tier anchor — arrived at from unit economics rather
> than from competitor pricing.

## 3. Model tiering

> DECISION (2026-09-10): **Model tier is runtime configuration per pipeline stage, never
> hardcoded** (required by `../product/market-analysis.md`). Default every stage to
> `claude-opus-4-8`. Any downgrade is an explicit owner decision backed by eval results
> (`../factory/quality-gates.md`), not a default we ship.

Proposed starting configuration, to be validated by evals before adopting:

| Stage | Proposed | Effort | Batch? | Reasoning |
|---|---|---|---|---|
| Intent compilation | `claude-opus-4-8` | `high` | No | Rare, user-facing, quality-critical |
| Article summarisation | *tier under eval — see §3a* | `low` | **Yes** | Highest volume — the eval that matters most |
| Story clustering | self-hosted embeddings | — | — | pgvector similarity; no generative call at all |
| Dossier synthesis | `claude-opus-4-8` | `high`/`xhigh` | No | The product's differentiator; do not cheapen |
| Delta / "since you last read" | `claude-opus-4-8` | `medium` | No | Small input, user-facing |

## 3a. Open-weight models for summarisation

Researched 2026-09-10. Summarisation is not a frontier task, and it is our highest-volume
stage — the one place an open-weight model could plausibly change the economics.

Same workload (1,700 in / 150 out), against current hosted open-weight pricing:

| Option | $/article | @5,000 articles/day |
|---|---|---|
| Claude Opus 4.8 (batch) | $0.0062 | ~$930/mo |
| Claude Sonnet 5 (batch) | $0.0037 | ~$555/mo |
| Claude Haiku 4.5 (batch) | $0.0012 | ~$180/mo |
| Llama 3.3 70B (DeepInfra, $0.10/$0.32 per M) | ~$0.00022 | **~$33/mo** |
| 8B-class (DeepInfra/Groq, ~$0.05–0.06 per M) | ~$0.00011 | **~$17/mo** |

≈5× cheaper than the cheapest Claude tier, ≈28× cheaper than Opus, on the platform's
dominant cost line. Open-weight frontier as of Sept 2026: GLM-5.3, Kimi K3, Qwen 3.8,
DeepSeek V4 Pro; Mistral/Gemma are Apache 2.0 and DeepSeek/GLM are MIT — all clean for
commercial use. **Specific versions churn monthly; treat the families as stable and the
version numbers as disposable.**

> OPEN — **the single highest-leverage question in the design.** The eval must judge
> candidates on **downstream Dossier synthesis quality**, not on summary quality in
> isolation. Summaries are *inputs* to synthesis, so a dropped causal link doesn't just
> degrade one card — it corrupts the living summary that is the product's differentiator.
> Quality loss compounds; this is not a pure cost decision.

## 3b. Provider routing

> DECISION (2026-09-10): **Our gateway is the only routing seam. Pydantic AI is the typed
> call layer inside it. No third-party aggregator in front of Claude.**
>
> Layering:
> ```
> pipeline stage  ──asks for a capability tier, never a vendor──▶
>   gateway (ours: config, metering, retry, provider choice)
>     ├─ Pydantic AI ──▶ Anthropic / open-weight providers   (typed, structured calls)
>     └─ Anthropic SDK direct ──▶ Batch API                  (summarisation only, if needed)
> ```
>
> Rationale: the cost model depends on Anthropic's **Batch API 50% discount** (§2a), and
> it is not established that this survives an aggregator. Routing Claude through a
> third-party gateway to save integration effort could silently double our Claude spend —
> the exact opposite of the intent.

**On LiteLLM** — proposed 2026-09-10, deferred rather than rejected.

> DECISION (2026-09-10): **Do not adopt LiteLLM now.** Two reasons:
> 1. **Redundant layer.** Pydantic AI already abstracts 20+ providers. Stacking LiteLLM
>    beneath it means two abstraction layers, two cost-tracking paths, and two places to
>    debug a bad call — for provider-agnosticism we would already have.
> 2. **Unverified batch support.** A search on 2026-09-10 could not confirm LiteLLM
>    carries Anthropic Batch API pricing. Until verified, it cannot sit on the
>    summarisation path.
>
> Note a correction to an earlier framing here: LiteLLM ships **both** an SDK (a library,
> no extra deployable) and a proxy (a separate service). The earlier rejection targeted
> the *proxy*; the SDK is a fairer proposition and is what would be revisited.
>
> **Revisit when** we run open-weight inference across several providers at once and want
> unified fallback/budget rules — that is the problem LiteLLM actually solves well, and we
> do not have it yet.

> ⚠️ VERIFY BEFORE BUILDING THE SUMMARISATION STAGE — two questions, both load-bearing
> for §2a:
> 1. Does **Pydantic AI** expose native Anthropic Batch API access? If not, that stage
>    uses the Anthropic SDK directly behind the gateway (`architecture.md`).
> 2. Does **any** aggregator (LiteLLM, OpenRouter) preserve the batch discount? If one
>    does, the routing decision above is worth revisiting.

## 4. Implementation requirements

- **One LLM gateway module.** Every call goes through it (`architecture.md`). It owns:
  model selection from config, **provider routing (§3b)**, batch routing, retry, and
  **per-call cost metering written to a `llm_usage` table**. Cost observability is a build
  requirement, not an afterthought. Provider choice must never leak into a pipeline stage —
  a stage asks for a *capability tier*, not a vendor.
- **Batch the enrichment pipeline.** Overnight article summarisation has no latency
  requirement; 50% off is free money. Poll `batches.retrieve` until `ended`, key results
  by `custom_id` (results arrive in any order).
- **Cache aggressively on synthesis.** Stable prefix = system prompt + living summary +
  research direction. Volatile suffix = new items. Put the breakpoint at the end of the
  stable portion. Verify with `usage.cache_read_input_tokens` — if it's zero across runs,
  something is invalidating the prefix.
- **No `datetime.now()` / UUIDs in prompt prefixes.** Classic silent cache invalidator;
  also breaks test determinism.
- **Structured outputs for anything parsed.** Use `output_config.format` with a JSON
  schema — not prose parsing, not assistant prefill (prefill returns 400 on Opus 4.8).
- **Set `thinking: {type: "adaptive"}` explicitly** where thinking is wanted; omitting it
  on Opus 4.8 runs without thinking.
- **Handle `stop_reason`** — check it before reading `content`; handle `refusal` and
  `max_tokens` distinctly.

## 5. Open questions
> OPEN: Full-rewrite vs incremental-append for Dossier re-synthesis. Full rewrite is
> higher quality and simpler; append is far cheaper as the dossier grows. Likely answer:
> append the delta each run, full rewrite on a slower cadence or on direction change.
> OPEN: When a user edits `research_direction`, do we re-frame the existing summary (one
> expensive full rewrite) or only steer future synthesis? (Also open in `../product/view-primitives.md`.)
> OPEN: BYO-API-key for power users — sidesteps the heavy-user cost tail entirely.
> Carried over from `../product/market-analysis.md`.

## Related
- `architecture.md` — the shared-enrichment decision this costs out
- `ingestion.md` — determines article volume, the input to §2a
- `../product/market-analysis.md` — why this doc reports ranges instead of hitting a target
