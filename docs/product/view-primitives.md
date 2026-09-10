# View Primitives

Status: DRAFT — 2026-09-10

Three output shapes. All three are rendered from the same underlying intent + source set
(`intent-model.md`, `source-discovery.md`); they differ in **lifecycle** and **what the
synthesis optimizes for**. One intent can drive more than one primitive.

The **Dossier is the heart of the product** (the "researcher"); Edition and Stream are
the approachable faces (the "news reader"). See `vision.md`.

---

## Comparison

| | **Edition** | **Stream** | **Dossier** |
|---|---|---|---|
| Metaphor | Morning newspaper | Reading list / inbox | Living research file |
| Lifecycle | Time-boxed, regenerated each period | Persistent, ever-growing | Persistent, continuously re-synthesized |
| Unit | Today's issue | Individual items | An evolving document + item trail |
| State | Ephemeral (archived after) | read / unread / saved per item | Sections that update; "what's new since you last looked" |
| Optimizes for | Feeling caught up, then done | Not missing anything | Understanding a topic over time |
| Primary persona | Morning Reader | News-Catcher | Researcher / Topic-Obsessive |
| AI job | Rank + summarize today's set | De-dupe, prioritize queue | **Synthesize across time**: connect, contrast, track changes |

---

## Edition
A finite issue generated on a schedule (e.g., daily 6am). Fixed at generation; reading it
does not change it. Has a front page (top stories), sections (by domain/facet), and a
clear end. Yesterday's editions are archived, browsable.
- Ranking: importance × relevance × freshness within the period.
- Explicitly **not** infinite scroll. The finiteness is the feature.

## Stream
A persistent, growing queue of items matching the intent. Items carry read/unread/saved
state; read items drop out (or move to history). De-duplicated so the same story from
five outlets appears once with sources grouped.
- Marking read: manual, and optionally auto (e.g., opened, or aged out).
- This is the closest analog to Feedly/Particle — table stakes, done cleanly.

## Dossier ★
The differentiator. For a tracked topic, the system maintains an **evolving synthesized
document**, not just a list:
- **Living summary**: the current state of understanding, updated as news arrives.
- **Timeline / what changed**: new developments linked to prior context ("this follows
  last week's X").
- **Threads / sub-topics**: auto-organized facets that grow over time.
- **"Since you last read"**: a delta so returning users see only what's new.
- **Source trail**: every claim traces to items and their sources (provenance = trust).
- **Ask-the-dossier** (candidate): query the accumulated corpus in natural language.

The Dossier is what "trigger the researcher" (owner's original phrasing) produces: a place
where summaries are *continuously added and re-synthesized to make sense*, not reset daily.

### Research direction is required
A Dossier is never just a topic — it always carries a **user-supplied direction**: the
angle, question, or purpose they're pursuing ("understand the EU AI Act *as it affects
small SaaS vendors*", "track Company X *for a possible investment*"). The direction is
what the synthesis is steered by: it decides what counts as significant, what gets
foregrounded in the living summary, and what the delta reports on.

Consequences:
- Direction is a **required field** on a Dossier intent (`intent-model.md`), not optional metadata.
- Two users tracking the same topic with different directions get genuinely different Dossiers.
- The direction is editable, and changing it re-steers future synthesis (and may re-frame
  the existing living summary).

> DECISION (2026-09-10): Dossiers are **never auto-created**. Edition/Stream items do not
> silently accumulate into research mode. Research mode is always explicitly entered by
> the user *with a direction they provide*. Rationale: unsteered accumulation produces a
> generic topic-summary — exactly the commodity output we're differentiating from. The
> user's direction is the input that makes synthesis valuable.
>
> Promotion path: from an Edition/Stream a user can "research this", which opens the
> Dossier setup asking for their direction. Convenient, but still explicit and still steered.

> DECISION (2026-09-10): Dossier is the primary value driver and the paid-tier anchor.
> Build Edition + Stream first for the approachable loop, but the architecture
> (`design/data-model.md`) must treat continuous synthesis as first-class from the start,
> not bolt it on later.

> OPEN: How much re-synthesis is full-rewrite vs. incremental append? Cost implications —
> see `design/llm-and-cost.md`.
> OPEN: When a user edits a Dossier's direction, do we re-frame the existing living summary
> or only steer future synthesis? Cost/UX tradeoff — see `design/llm-and-cost.md`.

## Related
- `intent-model.md` — how one intent selects/parameterizes a primitive
- `source-discovery.md` — where the content comes from
- `design/data-model.md`, `design/llm-and-cost.md` — Phase 1.5 realization
