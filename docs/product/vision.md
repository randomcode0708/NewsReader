# Vision

Status: DRAFT — 2026-09-10

## The core reframe (read this first)

> **This is not a news reader. It is a personal researcher that wears a news reader's
> face.** The familiar shapes — a morning newspaper, a reading list — are the *entry
> points* that make it approachable. The heart of the product is **continuous
> sense-making**: the system reads on your behalf across many sources and builds an
> evolving understanding of the topics you declare, connecting today's story to last
> week's. The **Dossier** primitive (see `view-primitives.md`) is where that value
> lives; the Edition and Stream are how most users first meet the product.

## The problem

People consume news in three different shapes, and every existing tool forces one:

- **A morning newspaper** — a finite, time-boxed edition read once, then done.
- **A never-ending reading list** — items accumulate, get marked read, persist across days.
- **A research thread** — on a topic you care about, understanding should *accumulate and
  re-synthesize over time*, not reset each day.

Existing tools solve at most the first two. Feedly is a source-based river with AI
filtering. Particle/Digg are algorithmic personalized briefings. **None does the
research job**: reading across sources over time and maintaining a living synthesis.
And none lets the user simply **declare intent in natural language** — they all make
you either hand-curate sources or accept a black-box feed.

## Product thesis

> A user declares **intent in plain language** ("help me understand the EU AI Act and
> its fallout"). The platform:
> 1. **Discovers sources** for that intent — it proposes relevant *domains/topics*, then
>    proposes concrete *sites/feeds* per domain; the user selects, and can add their own.
>    (See `source-discovery.md`.)
> 2. **Compiles the intent** into a structured query + ranking policy (see `intent-model.md`).
> 3. **Renders it** as one of three **view primitives** (see `view-primitives.md`):
>    **Edition** (time-boxed newspaper), **Stream** (persistent reading list), or
>    **Dossier** (accumulating researcher).

The differentiators are (a) the **NL-intent → source-discovery → view-shape** pipeline
and (b) the **Dossier** research primitive. "AI summaries" alone are commodity in 2026;
*continuous synthesis over self-assembled sources* is not.

## Target users

**Multi-persona from day one** — this is not owner-specific. See `personas.md` for the
set (news-catcher, professional analyst/researcher, topic-obsessive, morning-reader).
The owner serves as the first design partner and test user, but the product must be
designed for users whose topics and habits we don't know in advance — which is exactly
why intent + automatic source discovery matter.

## What success looks like

**Product loop works when:**
- A user goes from a one-line intent to a working view in < 2 minutes, *without hand-
  curating sources* — source discovery does the heavy lifting and the user just confirms.
- The Dossier on a tracked topic becomes something users return to weekly because it
  saved them the work of reading 20 articles.
- Users trust the synthesis enough to act on it (share it, make a decision from it).

**Commercial hypothesis to validate — see `market-analysis.md`:**
- That the Dossier does enough real research work that people would pay for it. **We are
  not setting a price up front** — build, get usage evidence, then price. Do not treat any
  figure in `market-analysis.md` §4 as a plan.
- Retention driven by accumulated Dossiers (switching cost that source-based tools lack).

## Non-goals (Phase 1)

- Social features, sharing, comments.
- Being a source of record / breaking-news speed. We optimize for *sense-making*, not latency.
- Mobile-native apps. Start web-first (revisit in design phase).

> OPEN: Audio/TTS "listen to my edition" — deferred, revisit after core loop works.

## Related

- `market-analysis.md` — competitive landscape and monetization
- `personas.md` — who we build for (multi-persona)
- `source-discovery.md` — intent → suggested domains → suggested sites → user selection
- `view-primitives.md` — the three output shapes (Dossier is the heart)
- `intent-model.md` — the core differentiating mechanism
