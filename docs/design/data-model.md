# Data Model

Status: DRAFT — 2026-09-10
Authoritative schema. Supersedes the product-level sketch in `../product/intent-model.md`.

Postgres + `pgvector`, via SQLAlchemy 2.x + Alembic. Shown as SQL-ish DDL for clarity;
the SQLAlchemy models are the implementation and Alembic migrations are the change record.

> Note: this document survived the 2026-09-10 stack revision (TypeScript → Python) almost
> unchanged — Postgres was never the part in question. Only the ORM differs.

## Shape of the model

The schema is deliberately split along the **shared vs per-user** line established in
`architecture.md`. Get this boundary wrong and the cost model in `llm-and-cost.md`
collapses.

```
 ── SHARED (global, amortised) ──────────────────────────────────
 sources ──< articles ──< article_summaries
                 └──< article_embeddings
 stories ──< story_articles                (same event, many outlets)

 ── PER-USER ────────────────────────────────────────────────────
 users ──< intents ──< intent_sources  >── sources
                 ├──< view_items        (Edition / Stream)
                 └──< dossiers ──< dossier_sections
                                └──< dossier_deltas
```

## Shared tables

```sql
-- A place we read from. Discovered or user-added.
CREATE TABLE sources (
  id            uuid PRIMARY KEY,
  kind          text NOT NULL,      -- rss | atom | site | newsletter | subreddit | youtube | search
  feed_url      text UNIQUE,        -- resolved fetch target
  site_url      text,
  name          text NOT NULL,
  topics        text[],             -- for discovery matching
  credibility   real,               -- 0..1 signal; see ingestion.md
  poll_interval interval NOT NULL DEFAULT '30 minutes',
  last_polled_at timestamptz,
  last_error    text,
  failure_count int NOT NULL DEFAULT 0,
  created_at    timestamptz NOT NULL
);

-- One row per article, globally. NOT per user.
CREATE TABLE articles (
  id            uuid PRIMARY KEY,
  source_id     uuid NOT NULL REFERENCES sources(id),
  url           text NOT NULL,
  url_canonical text NOT NULL,      -- tracking params stripped; dedupe key
  content_hash  text NOT NULL,      -- dedupe across syndication
  title         text NOT NULL,
  author        text,
  published_at  timestamptz,
  fetched_at    timestamptz NOT NULL,
  extracted_text text,              -- fair-use working copy; see ingestion.md
  extract_status text NOT NULL,     -- pending | ok | paywalled | failed
  UNIQUE (url_canonical),
  UNIQUE (content_hash)
);

-- SHARED enrichment. One summary per article, reused by every user.
-- Nothing user-specific or intent-specific may appear here. See architecture.md.
CREATE TABLE article_summaries (
  article_id    uuid PRIMARY KEY REFERENCES articles(id),
  summary       text NOT NULL,
  key_points    jsonb,
  entities      jsonb,
  model_id      text NOT NULL,      -- provenance: which model produced this
  prompt_version text NOT NULL,     -- lets us re-enrich selectively on prompt change
  cost_usd      numeric(10,6),
  created_at    timestamptz NOT NULL
);

-- Self-hosted sentence-transformers embeddings (architecture.md).
-- The dimension is fixed by the chosen model. Changing the model means re-embedding
-- the ENTIRE corpus — so model_id is stored per row to make a staged migration possible.
CREATE TABLE article_embeddings (
  article_id    uuid PRIMARY KEY REFERENCES articles(id),
  embedding     vector(768) NOT NULL,   -- dimension follows the chosen model; see config
  model_id      text NOT NULL,
  created_at    timestamptz NOT NULL
);
CREATE INDEX ON article_embeddings USING hnsw (embedding vector_cosine_ops);

-- A real-world event, covered by N articles. Powers dedupe in Stream/Edition.
CREATE TABLE stories (
  id            uuid PRIMARY KEY,
  canonical_title text NOT NULL,
  centroid      vector(1536),
  first_seen_at timestamptz NOT NULL,
  last_seen_at  timestamptz NOT NULL,
  article_count int NOT NULL DEFAULT 0
);

CREATE TABLE story_articles (
  story_id      uuid REFERENCES stories(id),
  article_id    uuid REFERENCES articles(id),
  PRIMARY KEY (story_id, article_id)
);
```

## Per-user tables

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY, email text UNIQUE NOT NULL, created_at timestamptz NOT NULL
);

-- The compiled Intent. NL in, structure out, always user-editable.
-- Product-level rationale: ../product/intent-model.md
CREATE TABLE intents (
  id            uuid PRIMARY KEY,
  user_id       uuid NOT NULL REFERENCES users(id),
  raw_prompt    text NOT NULL,          -- user's original words, preserved verbatim
  title         text NOT NULL,
  primitive     text NOT NULL,          -- edition | stream | dossier
  topics        text[] NOT NULL,
  must_include  text[],
  must_exclude  text[],
  recency       text NOT NULL,          -- breaking | daily | rolling
  depth         text NOT NULL,          -- headlines | summaries | deep
  geography     text[],
  ranking_policy jsonb NOT NULL,        -- weights + dedupe flag
  schedule      text,                   -- cron, for edition
  compiled_by   text,                   -- model provenance
  created_at    timestamptz NOT NULL,
  updated_at    timestamptz NOT NULL
);

CREATE TABLE intent_sources (
  intent_id     uuid REFERENCES intents(id) ON DELETE CASCADE,
  source_id     uuid REFERENCES sources(id),
  origin        text NOT NULL,          -- discovered | user_added
  enabled       boolean NOT NULL DEFAULT true,
  PRIMARY KEY (intent_id, source_id)
);

-- Materialised, ranked items for Edition and Stream.
CREATE TABLE view_items (
  id            uuid PRIMARY KEY,
  intent_id     uuid NOT NULL REFERENCES intents(id) ON DELETE CASCADE,
  story_id      uuid NOT NULL REFERENCES stories(id),
  score         real NOT NULL,
  score_reasons jsonb,                  -- WHY this appeared — transparency requirement
  edition_date  date,                   -- set for edition; NULL for stream
  state         text NOT NULL DEFAULT 'unread',  -- unread | read | saved | dismissed
  read_at       timestamptz,
  created_at    timestamptz NOT NULL,
  UNIQUE (intent_id, story_id, edition_date)
);
CREATE INDEX ON view_items (intent_id, state, score DESC);
```

## Dossier — the differentiator

Modelled per `../product/view-primitives.md`: a Dossier is **never just a topic**; it
always carries a user-supplied `research_direction` that steers synthesis.

```sql
CREATE TABLE dossiers (
  id            uuid PRIMARY KEY,
  intent_id     uuid NOT NULL UNIQUE REFERENCES intents(id) ON DELETE CASCADE,

  -- REQUIRED, NOT NULL. The user's angle/question/purpose. Cannot be inferred —
  -- it is the one thing intent compilation must ask for. See ../product/intent-model.md.
  research_direction text NOT NULL,

  living_summary text,                  -- current state of understanding
  synthesis_style text NOT NULL DEFAULT 'analytical',
  synthesis_frequency interval NOT NULL DEFAULT '1 day',  -- runtime-tunable cost knob
  last_synthesised_at timestamptz,
  direction_changed_at timestamptz,     -- triggers re-frame decision
  created_at    timestamptz NOT NULL
);

-- Auto-organised sub-topics that grow over time.
CREATE TABLE dossier_sections (
  id            uuid PRIMARY KEY,
  dossier_id    uuid NOT NULL REFERENCES dossiers(id) ON DELETE CASCADE,
  heading       text NOT NULL,
  body          text NOT NULL,
  sort_order    int NOT NULL,
  updated_at    timestamptz NOT NULL
);

-- "What changed" / "since you last read". Append-only timeline.
CREATE TABLE dossier_deltas (
  id            uuid PRIMARY KEY,
  dossier_id    uuid NOT NULL REFERENCES dossiers(id) ON DELETE CASCADE,
  body          text NOT NULL,
  covers_from   timestamptz NOT NULL,
  covers_to     timestamptz NOT NULL,
  created_at    timestamptz NOT NULL
);

-- Provenance: every claim traces to source articles. Trust requirement from
-- ../product/source-discovery.md — users must see where synthesis came from.
CREATE TABLE dossier_citations (
  id            uuid PRIMARY KEY,
  dossier_id    uuid NOT NULL REFERENCES dossiers(id) ON DELETE CASCADE,
  section_id    uuid REFERENCES dossier_sections(id) ON DELETE CASCADE,
  delta_id      uuid REFERENCES dossier_deltas(id) ON DELETE CASCADE,
  article_id    uuid NOT NULL REFERENCES articles(id),
  quoted_span   text
);
```

## Infrastructure tables

```sql
-- The queue. Every long job decomposes into rows here. See architecture.md.
CREATE TABLE jobs (
  id            uuid PRIMARY KEY,
  kind          text NOT NULL,          -- poll_source | extract | summarise | cluster | rank | synthesise
  payload       jsonb NOT NULL,
  state         text NOT NULL DEFAULT 'queued',  -- queued | claimed | done | failed
  attempts      int NOT NULL DEFAULT 0,
  claimed_at    timestamptz,
  claim_expires_at timestamptz,         -- lets a dead worker's job be reclaimed
  run_after     timestamptz NOT NULL DEFAULT now(),
  last_error    text,
  idempotency_key text UNIQUE,          -- makes re-enqueue safe
  created_at    timestamptz NOT NULL
);
CREATE INDEX ON jobs (state, run_after) WHERE state = 'queued';

-- Cost observability is a build requirement, not an afterthought. See llm-and-cost.md.
CREATE TABLE llm_usage (
  id            uuid PRIMARY KEY,
  stage         text NOT NULL,
  model_id      text NOT NULL,
  user_id       uuid REFERENCES users(id),   -- NULL for shared/platform work
  input_tokens  int NOT NULL,
  output_tokens int NOT NULL,
  cache_read_tokens int NOT NULL DEFAULT 0,
  cache_write_tokens int NOT NULL DEFAULT 0,
  batch         boolean NOT NULL DEFAULT false,
  cost_usd      numeric(10,6) NOT NULL,
  created_at    timestamptz NOT NULL
);

-- Runtime-tunable knobs, per ../product/market-analysis.md. No hardcoded model IDs.
CREATE TABLE config (key text PRIMARY KEY, value jsonb NOT NULL, updated_at timestamptz);
```

## Invariants the factory must enforce

These are testable assertions, and they are how we keep agent-written code from eroding
the design:

1. **No user or intent identifier may appear in `article_summaries` or
   `article_embeddings`.** Violating this breaks shared enrichment and the cost model.
2. **`dossiers.research_direction` is `NOT NULL`.** No code path creates a Dossier
   without a user-supplied direction.
3. **Every `view_items` row has non-null `score_reasons`.** Transparency is a product
   promise (`../product/intent-model.md`), so it's a schema-level obligation.
4. **Every LLM call writes an `llm_usage` row.** Enforced by routing all calls through
   the gateway module.
5. **Every job handler is idempotent**, keyed by `idempotency_key`. Re-running a job must
   not double-charge or duplicate rows.
6. **No hardcoded model ID outside the gateway + `config`.** Greppable, so it's a lint rule.
7. **`article_embeddings.model_id` matches the configured embedding model.** A mismatch
   means rows were written under a different model and are not comparable — clustering
   against them is silently wrong. Assert on write; alert on drift.

> RESOLVED (2026-09-10): Embeddings are **self-hosted via `sentence-transformers`**
> (`architecture.md`). `vector(768)` above assumes a 768-dim model; the specific model
> choice is still to be made and **fixes the dimension permanently** for the corpus.
> Record it in `config` before the first embedding is written.

> OPEN: Which sentence-transformers model. Constraints: must run acceptably on Fly.io CPU
> (or justify a GPU machine), and clustering quality on real multi-outlet coverage is the
> metric that decides it — not a generic benchmark. Belongs in the eval harness.
> OPEN: Retention. `articles.extracted_text` is the largest column and a legal-posture
> concern (`ingestion.md`) — likely TTL it once `article_summaries` exists.

## Related
- `architecture.md` — the shared/per-user split this encodes
- `llm-and-cost.md` — why `article_summaries` must stay user-agnostic
- `ingestion.md` — populates `sources` and `articles`
- `../product/intent-model.md` — product-level intent sketch this supersedes
