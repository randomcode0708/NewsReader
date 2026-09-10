# Source Discovery

Status: DRAFT — 2026-09-10

Source discovery is a **core subsystem, not a setting**. The user should never have to
hand-assemble RSS lists (Feedly's toil). They declare intent; the system proposes where
to read from; the user confirms and can add their own. This is a key part of what makes
the product feel like a researcher working for you.

## The flow

```
1. User enters intent (plain language)
        │
        ▼
2. System proposes DOMAINS/TOPICS  ──►  user selects the relevant ones
   e.g. intent "EU AI regulation" → [AI policy] [EU institutions] [Big Tech] [Legal/compliance]
        │
        ▼
3. For selected domains, system proposes concrete SOURCES (sites/feeds/newsletters)
   ranked by relevance + credibility  ──►  user toggles which to include
        │
        ▼
4. User can ADD SOURCES DIRECTLY at any time — by URL *or* by name/keyword search
   (full manual entry, Feedly-style — see "Manual source entry" below)
        │
        ▼
5. Source set is bound to the intent; feeds the query + view (intent-model.md)
```

Steps 2–3 are the "we figure out sources for you" promise. Step 4 preserves full
power-user control. The set is **editable anytime** and the system keeps suggesting new
sources as the topic evolves (a background job proposes additions; user approves).

## Manual source entry

Discovery is the default path, not the only path. A user must always be able to add a
source themselves, as richly as a dedicated reader allows. Two entry modes:

**1. By URL** — paste anything and we resolve it:
- an RSS/Atom feed URL → use directly
- a website/blog homepage → auto-discover its feed(s); if none, fall back to sitemap or
  fair-use extraction (`design/ingestion.md`)
- a specific section URL (e.g. a publication's AI section) → scope to that section

**2. By name / keyword search** — the user types a publication name, topic, or keyword and
we return matching sources to follow, with enough metadata to choose confidently
(what it is, how often it publishes, a sample of recent headlines, credibility signal).
This is the Feedly "add content" affordance and it is table stakes for power users.

> Requirement: name-search implies a **searchable source index** we maintain or query.
> Build-vs-buy and how it interacts with LLM-proposed sources: `design/ingestion.md`.

**Source types to support** (target set; sequence in design phase):
RSS/Atom · websites without feeds · newsletters (email-in) · subreddits · X/Bluesky
accounts · YouTube channels · podcasts · keyword/alert-style standing searches.

> DECISION (2026-09-10): Discovery and manual entry are **peers**, not fallback-and-primary.
> We never *block* a user on choosing sources — an intent works immediately from proposals —
> but we never *limit* them either. Users who want Feedly-grade control get it.

## What the system must do

- **Domain proposal**: from intent text, generate candidate topic facets. (LLM-assisted.)
- **Source proposal**: for each domain, produce concrete, resolvable sources with:
  - a fetch method (RSS, sitemap, API, or fair-use scrape — see `design/ingestion.md`)
  - a credibility/relevance signal so ranking isn't naive
  - dedup against sources the user already has
- **Manual entry**: accept a raw URL and auto-detect its feed/fetch method; and resolve a
  name/keyword search against a source index. See "Manual source entry" above.
- **Continuous discovery**: periodically surface *new* candidate sources for an existing
  intent (e.g., a newly relevant publication), gated by user approval.

## Product principles

- **Confirm, don't configure.** Defaults should be good enough to accept in one tap.
- **Transparent provenance.** Every item traces to its source; users see and can prune
  where things come from. This also builds trust in the synthesis.
- **Bias/credibility aware.** Surface source diversity; avoid silently narrowing to one
  viewpoint (nod to Ground News's blindspot value — see `market-analysis.md`).
- **Ethical sourcing.** Prefer RSS/API and summary-with-linkback over full-text scraping
  (Particle model). Details and legal posture: `design/ingestion.md`.

## Open questions
> OPEN: Cold-start quality — how good are proposals for an obscure intent? Fallback to
> web search + snippet extraction?
> OPEN: How much to lean on a general web-search API vs. a curated source registry we
> maintain? Cost/quality tradeoff — resolve in `design/ingestion.md`.
> OPEN: Trust model for user-added sources of unknown credibility.

## Related
- `intent-model.md` — source set is one output of intent compilation
- `design/ingestion.md` — how sources are actually fetched, parsed, deduped (Phase 1.5)
