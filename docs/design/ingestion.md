# Ingestion & Source Discovery

Status: DRAFT — 2026-09-10
Implements `../product/source-discovery.md`. Feeds `articles` in `data-model.md`.

## Strategy

> DECISION (2026-09-10): **RSS-first, own crawler, fair-use summary with linkback.** We
> operate our own RSS/Atom crawler and article extractor; we do not buy a news API for the
> core corpus. Rationale: near-zero marginal data cost, no per-request vendor pricing on
> the highest-volume path, full control over source selection, and it matches the
> publisher-friendly posture that is now the market norm (`../product/market-analysis.md`).
>
> Rejected: paid news APIs ($25–90/mo entry, and most free/entry tiers return
> **no full article text**, which makes them useless for synthesis). We may still add one
> later purely for *discovery* breadth — that is a smaller, bounded decision.

## Legal and ethical posture

This is a product constraint, not just an engineering one.

| Rule | Why |
|---|---|
| **Always link back to the original article.** Every surfaced item carries its source and URL. | Publisher-friendly; drives traffic back (the Particle model) |
| **Summarise; never republish.** We show our own summary + a short quoted span at most, never the full article body. | Fair use posture |
| **`extracted_text` is a working copy, not a library.** Used to produce a summary, then TTL'd. | Minimises the size of what we hold |
| **Honour `robots.txt` and feed terms.** Respect `Crawl-delay`; identify with a real User-Agent and contact URL. | Basic good citizenship; also keeps us unblocked |
| **Rate-limit per host**, not just globally. | Never be the reason a small publisher's server falls over |
| **Prefer the feed.** Only fall back to fetching the page when the feed lacks usable content. | Fewer requests, clearer consent |

> OPEN: Paywalled content. Current stance: detect (`extract_status = 'paywalled'`), store
> the feed-provided excerpt only, never attempt to bypass. Needs an explicit owner sign-off
> before launch.

## Fetch pipeline

Each step is a separate queued job, run by a long-lived worker (`architecture.md`):

```
poll_source ──▶ extract ──▶ summarise ──▶ cluster
 (per source)   (per article)  (per article,   (per article,
                               batched)        embeddings)
```

**1. `poll_source`** — one job per source per tick.
- Conditional GET: send `If-None-Match` / `If-Modified-Since`; a `304` costs nothing.
- Parse RSS/Atom; for each entry compute `url_canonical` (strip `utm_*`, `fbclid`, etc.).
- Skip entries already present by `url_canonical`.
- **Adaptive polling**: increase `poll_interval` for quiet feeds, decrease for busy ones.
  On repeated failures, back off exponentially and mark `last_error`; after N failures,
  surface the dead source to its subscribers rather than failing silently.

**2. `extract`** — one job per new article. Uses **`trafilatura`**.
- If the feed carries full `content:encoded`, use it — no extra HTTP request at all.
- Otherwise fetch the page and extract (title, byline, body, publish date, lead image).
- Detect paywalls/truncation and set `extract_status` accordingly.
- Compute `content_hash` over normalised body text.

> Extraction quality was a primary driver of the Python stack decision
> (`architecture.md`). It is on the critical path: bad extraction → bad summary → corrupted
> Dossier synthesis. Treat extraction regressions as high-severity, and hold a fixture
> corpus of real pages with known-good expected output (`../factory/quality-gates.md`).

**3. `summarise`** — see `llm-and-cost.md`. **Runs once globally per article**, routed
through the Batch API (50% cheaper, no latency requirement).

**4. `cluster`** — embed with self-hosted `sentence-transformers`, then group into
`stories`. Runs in-process on the worker; no external API call, so no rate limit sits on
the ingestion critical path.

## Deduplication (three distinct problems)

| Level | Problem | Approach |
|---|---|---|
| **URL** | Same article, different tracking params | Canonicalise; unique index on `url_canonical` |
| **Content** | Syndicated verbatim (wire copy across outlets) | `content_hash` over normalised text; unique index |
| **Story** | Same event, 10 outlets, 10 different write-ups | **Embedding similarity + time window** → `stories` |

Story clustering is the one that visibly matters — it's the difference between a Stream
that shows one item and a Stream that shows the same news ten times. Approach: cosine
similarity against recent story centroids above a threshold, bounded to a rolling time
window (an event from 3 months ago should not absorb today's article). Cheap: pgvector
similarity, no LLM call.

> OPEN: Similarity threshold and window length need empirical tuning against real
> multi-outlet coverage. This belongs in the eval harness (`../factory/quality-gates.md`),
> not in a guess here.

## Source discovery

Implements the flow in `../product/source-discovery.md`: intent → domains → sources →
user confirms. Discovery and manual entry are **peers**, per that document.

**Domain proposal** — LLM call over the intent text, returning candidate topic facets
via structured output (`output_config.format`, JSON schema).

**Source proposal** — for each selected domain, produce concrete resolvable sources.
Two-layer approach:

1. **Registry first.** A `sources` table we grow over time — every source any user has
   ever added or we've ever verified, tagged with `topics` and `credibility`. Cheap,
   fast, gets better with use, and doubles as the index that powers name/keyword search.
2. **Web search fallback** for cold-start on obscure intents, using Claude's server-side
   `web_search` tool. Resolve results to feeds, verify, then **write them back into the
   registry** so the next user with a similar intent hits layer 1.

> DECISION (2026-09-10): **Build the registry; don't buy a source index.** The registry is
> a compounding asset (it improves with every user), it's the same table that serves
> name-search, and the web-search fallback bounds the cold-start problem without a
> subscription. This resolves the build-vs-buy question left open in
> `../product/source-discovery.md`.

**Manual entry** — the peer path:
- *By URL*: accept feed URL, homepage (auto-discover via `<link rel="alternate">`), or a
  section URL. Resolve, verify it parses, then register.
- *By name/keyword*: search the registry (`name`, `topics`, trigram match), returning
  metadata rich enough to choose confidently — what it is, cadence, recent headlines,
  credibility signal.

**Continuous discovery** — a low-frequency job proposes *new* candidate sources for
existing intents. Always gated by user approval; never silently added.

## Source credibility

Needed so ranking isn't naive and so `../product/source-discovery.md`'s
diversity/blindspot goal is achievable. Start with cheap signals, not a grand system:
domain age and stability, publish cadence consistency, whether other registry sources
cite it, extraction reliability, and explicit user signals (a source a user prunes is a
negative signal for that user; pruned by many users is a global negative).

> OPEN: Do we take an off-the-shelf bias/credibility dataset, or stay purely behavioural?
> Behavioural is defensible and free but slow to bootstrap.

## Failure modes to design for

These are the things that will actually break, so they need tests
(`../factory/quality-gates.md`):

| Failure | Handling |
|---|---|
| Feed returns malformed XML | Parse defensively; quarantine the source, don't crash the tick |
| Feed URL 404s / domain dies | Exponential backoff → mark dead → notify subscribers |
| Publisher blocks our crawler | Detect 403/429 pattern; back off; surface for manual review |
| Extraction returns navigation chrome instead of article | Heuristic quality check (length, text/markup ratio); mark `failed` rather than summarising garbage |
| Same story floods a Stream | Story clustering (above) |
| A source floods (100 posts/hour) | Per-source volume cap per tick |
| Timezone/`published_at` nonsense (future dates, missing) | Clamp to `fetched_at`; never trust publisher dates for ranking alone |

## Related
- `../product/source-discovery.md` — the product flow this implements
- `architecture.md` — the job-queue model these stages run in
- `llm-and-cost.md` — article volume here drives the dominant cost line
- `data-model.md` — `sources`, `articles`, `stories`
