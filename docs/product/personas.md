# Personas

Status: DRAFT — 2026-09-10

This is a multi-persona product. We do **not** assume a fixed topic set or reading habit;
intent + automatic source discovery (`source-discovery.md`) exist precisely because
users' topics are unknown in advance. Below are the personas we design for. Each maps to
the view primitive (`view-primitives.md`) they lead with.

Ordering reflects strategic priority: the **Researcher** is the persona the product's
core value (continuous synthesis) is built for; the others are broader-reach entry points.

---

## P1 — The Researcher / Analyst  ★ core persona
- **Who**: knowledge worker who must *understand a topic deeply and stay current* — an
  analyst, founder, policy person, journalist, investor, or a curious generalist going
  deep on something.
- **Job**: "Build and maintain my understanding of X over time so I don't have to re-read
  everything." Wants connections across stories, what changed, what it means.
- **Leads with**: **Dossier**. Returns weekly+. This is where willingness-to-pay lives.
- **Success**: refers back to the Dossier to make a decision or brief someone.

## P2 — The Morning Reader
- **Who**: wants a calm, finite daily read to replace doomscrolling.
- **Job**: "Give me today's front page on what I care about, then let me be done."
- **Leads with**: **Edition**. Daily, time-boxed, ephemeral.
- **Success**: reads once each morning, feels caught up, closes the app.

## P3 — The News-Catcher / Power Reader
- **Who**: follows several topics, dips in through the day, hates missing things.
- **Job**: "Keep a running queue of what's worth reading; let me mark things read."
- **Leads with**: **Stream**. Persistent, ever-growing, read/unread state.
- **Success**: inbox-zero feeling; trusts nothing important slipped by.

## P4 — The Topic-Obsessive
- **Who**: intensely follows a narrow beat (a company, a technology, a conflict, a team).
- **Job**: "Track everything on this one thing and tell me when something meaningful moves."
- **Leads with**: **Dossier** + alerting on the beat.
- **Success**: finds out about meaningful developments here before elsewhere.

---

## Owner as design partner
The owner is the first test user and design partner (P1/P3-leaning), but the product is
**not** scoped to them. Any feature that only works if we know the user's topics ahead of
time is a design smell — route it through intent + source discovery instead.

## Related
- `view-primitives.md` — the shapes these personas lead with
- `source-discovery.md` — how any persona gets sources without curation toil
- `intent-model.md` — how each persona's plain-language ask becomes a query
