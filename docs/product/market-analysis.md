# Market Analysis & Validation

Status: DRAFT — 2026-09-10
Scope: validation-heavy (per owner decision 2026-09-10). Competitor teardown, market
sizing, positioning, and monetization hypothesis. Numbers are from public sources
(Sept 2026) and are directional, not audited.

## 1. Market sizing

Market-report figures vary widely by segment definition; treat as directional.

| Segment | Value (2026) | Forecast | CAGR | Source |
|---|---|---|---|---|
| Mobile News Apps | ~$16.9B | ~$24.1B by 2030 | ~9.3% | market.us / R&M |
| "News Apps" (narrow) | ~$6.8B | ~$15.3B by 2035 | ~9.4% | MarkWide |
| Digital Newspapers & Magazines (broad) | ~$56.8B | ~$121.5B by 2036 | ~7.9% | openPR |

**Interpretation for us:**
- **TAM** ≈ digital news consumption ($50B+, too broad to be meaningful).
- **SAM** ≈ paid personalized news-aggregation apps for power/knowledge users. The
  narrow "news apps" figure (~$6.8B) is the closest proxy.
- **SOM (Phase 2, realistic)** — a niche prosumer tool. At $5/mo, even 10k paying users
  = ~$600k ARR. This is a **lifestyle/indie-scale** opportunity by default, not a
  venture-scale one, unless the intent/Dossier mechanic proves to have broad pull.

> DECISION (2026-09-10): Frame the commercial case as **indie/prosumer SaaS first**.
> Venture framing is not justified by the sizing; do not over-build for scale in Phase 1.

## 2. Competitor teardown

| Product | Metaphor | AI | Price (2026) | Strength | Gap we exploit |
|---|---|---|---|---|---|
| **Feedly** | Source-based river (RSS, Reddit, newsletters, X) | "Leo" prioritizes, de-dupes, summarizes (Pro+ only) | Free / $72yr Pro / $99yr Pro+; Enterprise from $1,600/mo | Power-user source control, mature | No NL-intent; no time-boxed "edition"; no accumulating research view |
| **Particle** | Algorithmic personalized briefing | AI summaries w/ source links; learns from behavior | Free / $2.99mo / $29.99yr | Genuinely good personalization after ~1wk; publisher-friendly | Black-box feed; user can't *declare* intent; no persistent research thread |
| **Digg (relaunch)** | AI-scanned + editorial curation | AI scans thousands of sources | (relaunch, freemium) | Editorial layer on top of AI | Curated-for-everyone, not intent-for-you |
| **Ground News** | Bias/blindspot comparison | Bias labeling | Freemium | Best-in-class bias context | Narrow (bias lens); not a general reader |
| **Google News** | Algorithmic + topics | Personalization | Free | Ubiqudity, free | No control, no research accumulation, ad-driven |
| **Trace / Readless etc.** | AI daily digest, niche | Summaries | Various low $ | Focused digests | Single-shape output; no intent compiler |

**Anchors this establishes:**
- Consumer price ceiling for an individual is **~$3–8/mo** (Particle $2.99, Feedly Pro $6). Enterprise is a different market ($1,600+/mo) we are not targeting in Phase 1.
- "AI summaries" is **table stakes**, not a differentiator.
- Publisher-friendly sourcing (link back, don't just scrape) is now the expected ethical/legal posture (Particle's positioning) — relevant to our ingestion design.

## 3. Positioning

> **"Declare what you want to follow. Get it as a newspaper, a reading list, or a
> living research file — your choice, one intent."**

Two-axis positioning:
- **X: source-control ↔ algorithmic.** We sit center — user declares intent (more
  control than Particle) but doesn't hand-curate RSS lists (less toil than Feedly).
- **Y: ephemeral feed ↔ accumulating knowledge.** We are the only one that spans both,
  because of the **Dossier** primitive. This is the defensible corner.

## 4. Monetization — DEFERRED (illustrative only, not a plan)

> Read the DECISION at the end of this section before using any number here. Pricing is
> deferred until post-validation. The tiering below records *what competitors do*, to
> inform a later decision — it is not our pricing.

- **Free**: 1–2 intents, Edition + Stream, standard summaries.
- **Paid (~$5/mo)**: unlimited intents, Dossier (accumulating research), longer history,
  higher-quality/longer summaries.
- **Moat**: accumulated Dossiers = switching cost that source-based tools structurally
  lack. The longer you use it, the more your research files are worth.

> OPEN: BYO-LLM-key option to offload inference cost for power users? Revisit in
> `design/llm-and-cost.md`.

> DECISION (2026-09-10): **Pricing is deliberately deferred until after we build and
> validate.** We will not set a price on assumptions. Rationale (owner): the product's
> actual value — especially whether a steered Dossier does real research work — is not
> knowable from competitor tables. Pricing derived from assumed value would anchor the
> product to a guess.
>
> Therefore the tiering sketch above is **illustrative of what the market currently
> tolerates, not our plan.** Treat every number in §4 as a competitor observation.
>
> Note the unresolved tension for later: news-reader comps anchor $3–8/mo, but a research
> tool competes with analyst time ($20–50/mo). We resolve this with usage evidence, not
> analysis.
>
> **Consequence for Phase 1.5**: `design/llm-and-cost.md` must NOT design to a fixed price
> ceiling. Instead it must report **cost per user per month as a function of usage**
> (intents, items/day, Dossier re-synthesis frequency) so that once we do price, the
> economics are already characterized. Cost transparency and instrumentation over cost
> targets. Architecture should keep the expensive knobs (synthesis frequency, model tier,
> summary depth) **tunable at runtime** rather than baked in — that is what preserves
> pricing freedom later.

## 5. Risks / validation questions

1. **Cost**: per-article LLM summarization + continuous Dossier re-synthesis is the main
   unit-cost risk. Must be modeled before build — see `design/llm-and-cost.md`.
2. **Legal/sourcing**: scraping full text vs. RSS/summaries-with-linkback. Bias toward
   linkback + fair-use summary (Particle model).
3. **Will "intent" actually beat a good algorithmic feed?** Core product-risk. The
   personal Phase 1 build *is* the first validation of this.
4. **Is the market venture-scale?** Sizing says probably not; plan as indie SaaS.

## Sources

- [Feedly pricing/features (Capterra)](https://www.capterra.com/p/202497/Feedly/), [Readless: Feedly Pro pricing](https://www.readless.app/blog/feedly-pro-pricing-vs-readless-2026)
- [Best AI news aggregators 2026 (Readless)](https://www.readless.app/blog/best-ai-news-aggregators-2026), [daily.dev: best news aggregator apps](https://daily.dev/blog/best-news-aggregator-apps-tested-compared/)
- [Particle funding/business model (TechCrunch)](https://techcrunch.com/2024/11/12/particle-launches-an-ai-news-app-to-help-publishers-instead-of-just-stealing-their-work), [Particle Crunchbase](https://www.crunchbase.com/organization/particle-news)
- [Mobile News Apps market (market.us)](https://market.us/report/mobile-news-apps-market/), [Digital Newspapers market (openPR)](https://www.openpr.com/news/4600532/digital-newspapers-magazines-market-to-reach-usd-121-5-billion)
