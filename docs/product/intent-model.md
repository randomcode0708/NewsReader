# Intent Model

Status: DRAFT — 2026-09-10

The intent model is the product's spine. A user expresses what they want in **plain
language**; the system **compiles** that into a structured, editable object that drives
source discovery, retrieval, ranking, and the view. This is the mechanism behind the
"declare intent, get a researcher" promise (`vision.md`).

## Lifecycle

```
NL intent (free text)
   │  1. COMPILE (LLM → structured Intent object; user can edit any field)
   ▼
Structured Intent  ──►  2. SOURCE DISCOVERY (source-discovery.md)
   │
   │  3. RETRIEVE items from bound sources
   ▼
   4. RANK per the intent's policy
   │
   ▼
   5. RENDER as chosen primitive (view-primitives.md): Edition | Stream | Dossier
```

The user always sees and can edit the compiled structure — **NL is the input, not a
black box.** This is the trust difference vs. algorithmic feeds (Particle/Google News).

## The Intent object (draft schema)

```yaml
intent:
  id: string
  raw_prompt: string            # the user's original words (preserved verbatim)
  title: string                 # short label, LLM-suggested, user-editable
  topics: [string]              # compiled facets (also drives source discovery domains)
  must_include: [string]        # keywords/entities that must be present
  must_exclude: [string]        # noise to filter out
  scope:
    recency: enum(breaking | daily | rolling)   # how time-sensitive
    depth: enum(headlines | summaries | deep)    # how much synthesis
    geography: [string]?         # optional regional focus
  ranking_policy:
    weights: {importance, relevance, freshness, source_credibility, diversity}
    dedupe: bool                 # cluster same-story-across-outlets
  view:
    primitive: enum(edition | stream | dossier)
    schedule: cron?              # for edition (e.g., daily 6am)
    dossier_mode:                 # REQUIRED if primitive == dossier
      research_direction: string  # REQUIRED — the user's angle/question/purpose.
                                  # Steers what counts as significant. See view-primitives.md.
      synthesis_style: enum(brief | analytical | exhaustive)
      delta_tracking: bool
  sources:                       # bound by source-discovery.md; user-editable
    included: [source_ref]
    user_added: [source_ref]
  created_at: date
  updated_at: date
```

> This schema is a **product-level draft** to align on the concept. The authoritative,
> implementable version lives in `design/data-model.md` (Phase 1.5).

## Compilation principles

- **Accept anything, then ask for what's missing.** The user writes whatever they want,
  in whatever form — a word, a sentence, a paragraph. We compile immediately, then ask
  targeted follow-up questions **only for fields that are genuinely needed and absent**.
  Never a fixed wizard, never a question we could have inferred.
- **Suggest a primitive, don't force one.** From the phrasing, infer whether they want an
  Edition ("my morning brief on…"), Stream ("keep me on top of…"), or Dossier ("help me
  understand…" / "track everything about…") — but let the user switch.
- **Editable everywhere.** Every compiled field is adjustable; edits re-run downstream steps.
- **Transparent.** Show *why* items appear (which intent facet, which source) so users
  trust and can tune it.
- **One prompt → possibly multiple intents.** "Give me a tech front page and also track
  the OpenAI antitrust case" = one Edition intent + one Dossier intent. The compiler may
  propose a split.

## Examples

| Raw prompt | primitive | notable compiled fields |
|---|---|---|
| "My morning tech front page" | edition | schedule: daily 6am; depth: summaries; topics: [AI, startups, big tech] |
| "Keep me on top of NBA trades" | stream | recency: rolling; dedupe: true; must_include: [trade, signing] |
| "Help me understand the EU AI Act and its fallout" | dossier | depth: deep; delta_tracking: true; topics: [AI policy, EU institutions, compliance]; **asks for research_direction** |
| "Track everything about Company X" | dossier | recency: breaking alerts; topics auto-expanded from entity; **asks for research_direction** |

## Gap-driven clarification

> DECISION (2026-09-10): The user may write **whatever they want** — free-form, any
> length, any level of detail. We compile it immediately and then ask follow-up questions
> **driven by gaps, not by confidence**: we ask only for fields that are required and
> that we cannot reasonably infer. No fixed onboarding wizard.

What can and cannot be inferred:

| Field | If absent |
|---|---|
| `topics`, `must_include/exclude` | Infer from prompt. Never ask. |
| `view.primitive` | Infer from phrasing; present as a changeable choice, don't ask. |
| `scope.recency`, `scope.depth` | Infer sensible default from primitive. Don't ask. |
| `sources` | Never *block* on this — run source discovery and have the user confirm proposals. But always expose full manual add (by URL or by name search) as a peer path (`source-discovery.md`). |
| `view.schedule` (Edition) | Default (daily, morning). Confirm inline, don't block. |
| **`dossier_mode.research_direction`** | **Must ask.** Cannot be inferred — it is the user's own angle. This is the one genuinely blocking question. |

So in practice: Edition and Stream intents usually compile with **zero questions**;
Dossier intents ask **one** — "what angle are you pursuing here?" — because the direction
is required (`view-primitives.md`) and only the user has it.

Ask sparingly and always in plain language. A follow-up question should feel like a
researcher clarifying a brief, not a form.

## Open questions
> OPEN: Do we let users write raw structured intents (power users), or NL-only? Leaning:
> NL-first, structured-editable.
> OPEN: Ranking policy defaults per primitive — needs tuning against real content (eval
> harness in `factory/quality-gates.md`).

## Related
- `source-discovery.md` — consumes topics to propose sources
- `view-primitives.md` — the render targets
- `design/data-model.md` — the authoritative schema (Phase 1.5)
- `factory/quality-gates.md` — how we evaluate compilation & ranking quality
