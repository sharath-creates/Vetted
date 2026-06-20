# Trending Meme Content Engine — Spec & PRD

**Version:** 0.1 (MVP)
**Date:** 2026-06-20
**Owner:** Sharath
**Status:** Draft for scoping

---

## 1. Summary

An agentic workflow that finds which meme formats are trending right now, picks the three that best fit a product and its audience, and produces meme mockups that weave the product's use-case into each format's narrative. The mockups land in a review board for a human to approve before anything goes further.

The MVP is personal, single-operator, and runs at near-zero cost. It generates mockups for review only. Publishing, scheduling, and multi-client self-serve come later.

---

## 2. Problem Statement

Producing on-trend social content is slow and timing-sensitive. Meme formats peak and die within days, so by the time a person spots a format, briefs it, and renders it, the moment has passed. A solo operator wanting to test meme-driven product marketing has no cheap, repeatable way to catch formats on the upswing and turn them into product-relevant creative the same day.

This engine compresses that loop from days to one daily run, and keeps a human in the seat for the judgment calls a model gets wrong.

---

## 3. Goals

- Surface the top 3 *rising* meme formats each day, ranked by momentum rather than raw popularity, so creative rides formats before they peak.
- Produce 3 product-relevant meme mockups per brief, each embedding a real product use-case in the format's narrative.
- Keep recurring cost at or near $0 for the MVP, with the only spend being optional LLM and render calls.
- Get from brief to reviewable mockups in a single automated run, with one human approval step.
- Build the trend engine so a second content type (short video) and a publishing layer can attach later without a rebuild.

## 4. Non-Goals

- **No auto-publishing in the MVP.** The workflow stops at mockups on a review board. Publishing and scheduling via ClickUp and Blotato are Phase 2, once mockup quality is trusted.
- **No paying clients yet.** The MVP is single-operator and personal. Multi-tenant client intake, billing, and any paid data tiers are out of scope until the creative loop is proven.
- **No video generation in v1.** Memes are cheaper to generate and faster to judge. Video is a later content type once the format-trend and review loop is proven.
- **No TikTok data source in the MVP.** TikTok has no free official API, is banned or restricted in some markets, and scraping it adds maintenance cost. Excluded to protect the zero-cost, low-upkeep goal.
- **No direct Instagram trend scraping.** Instagram's Graph API exposes no general trending feed, and scraping breaks Meta's ToS and rots constantly. The engine infers trends from Imgflip and Giphy instead.
- **No Reddit source.** New personal Reddit API/OAuth access closed around Nov 2025, so the MVP doesn't depend on it. An optional, best-effort Reddit RSS branch may be tested but isn't relied on.

---

## 5. Users & Personas

**The Operator (primary, MVP).** You. Submits a brief, reviews the daily trend picks and the generated mockups, approves or rejects, and tunes the prompts. Needs the output to be fast to judge and easy to correct.

**The Brand brief (input artifact).** A one-page document describing the company, the consumer persona, the target geography/market, and the product to promote. Drives both which formats get picked (tone, language, region fit) and how the caption embeds the product.

**The Client (future).** A company that submits a brief and reviews results, with no access to the internals. Out of scope for the MVP; the design should not block it.

---

## 6. Trend Engine Specification

### 6.1 What "trending" means here

Two distinct signals, fetched from different places:

- **Format trend** — the reusable template everyone is captioning right now. This is what the engine reuses. Sources: Imgflip and Giphy.
- **Moment trend** — the joke, topic, or velocity behind a format. Used only to validate that a format is rising, not dying. Source: Google Trends.

The engine ranks by *acceleration*, not raw popularity. A template parked at rank 5 for three months is popular but stale. A template that jumped from rank 60 to rank 12 in 48 hours is trending. The score rewards the second and ignores the first.

### 6.2 Source stack (all free for personal/MVP use)

| Source | Role | Access | Cost | Notes |
|---|---|---|---|---|
| Imgflip `get_memes` | Spine: top ~100 templates by recent usage | Public API, no auth | $0 | Lags brand-new formats by a few days |
| Giphy `gifs/trending` | Leading indicator: rising reaction-meme content before it hits Imgflip | API key, no OAuth | $0 | Free key, hourly rate limit, fine for a daily pull. Tenor is an equivalent free backup |
| Reddit RSS (`/r/MemeTemplates/top/.rss`) *(optional)* | Best-effort extra signal | RSS, no auth | $0 | New Reddit API access is closed; RSS may still work for personal use. Not relied on |
| Google Trends (via pytrends) | Validator: is interest rising or crashing, and where | Unofficial, no auth | $0 | Confirm top movers before they reach a brief |
| Know Your Meme | Safety check: what a format actually means | Manual / no clean API | $0 | Avoid putting a product on an offensive or off-topic format |

TikTok Creative Center is deliberately excluded for the MVP (no free API, regional bans, scraping upkeep). Revisit if the service goes regional or commercial.

### 6.3 Velocity scoring

Each daily run stores a snapshot and compares against yesterday's. State is load-bearing: without yesterday's numbers there is no acceleration to compute. Cheapest store is a Google Sheet or a small SQLite/Postgres table.

Score per format combines:

- **Imgflip rank movement** — positive when a template climbs the top-100 (e.g. 60 → 12).
- **Giphy trending movement** — change in a format's trending position day over day.
- **Freshness weight** — decays fast, so a format flat for a week sinks even if it sits high.

Top movers then pass two filters: a **used/stale list** (don't repeat a format you've already used or that's past peak) and a **Google Trends check** (interest rising, not crashing). Survivors are sorted; the top 3 feed the creative stage.

### 6.4 Geography gate

The brief's market field routes sources. The MVP is English/global-first via Imgflip and Giphy. For a specific market, the engine filters the pool by language/region and drops formats whose humor or text won't translate. Because TikTok is excluded, the gate is simple in the MVP, but the branch exists so regional sources can attach later. Maintain an allowed-markets list and confirm it against current platform availability when building.

### 6.5 End-to-end workflow (n8n)

1. **Intake** — Operator submits the one-page brief (company, persona, geography, product) via a ClickUp Form. Creates a task.
2. **Schedule trigger** — Daily run fires the trend pull.
3. **Parallel fetch** — HTTP nodes hit Imgflip `get_memes` and Giphy `gifs/trending` in parallel.
4. **Normalize** — Function node maps both into a common shape: `{format_id, name, rank_or_score, image_url, timestamp}`.
5. **Score** — Read yesterday's snapshot, compute velocity per format, write today's snapshot back.
6. **Filter** — Drop used/stale formats; optionally validate top movers against Google Trends.
7. **Select** — Sort, keep top 3 for the active brief.
8. **Caption** — LLM writes a meme caption per format that embeds the product's use-case in that format's narrative, tuned to the persona and language.
9. **Render** — Generate the meme image (text-on-template). Imgflip `caption_image` covers simple top/bottom-text memes for free; Blotato or a richer renderer for more complex layouts.
10. **Deliver** — Attach the 3 mockups back to the ClickUp task.
11. **Review** — Operator approves or rejects via task status. (Publishing is Phase 2.)

The only cost in this chain is the caption LLM (step 8) and any non-free render (step 9). Everything else is free.

---

## 7. Requirements

### Must-Have (P0)

**P0-1 — Brief intake.** Operator submits a one-page brief capturing company, consumer persona, target geography, and product.
- Given a ClickUp Form, when the operator submits company + persona + geography + product, then a task is created holding all four fields.
- Reject/flag a submission missing any of the four required fields.

**P0-2 — Daily trend pull from free sources.** The engine fetches Imgflip and Giphy on a daily schedule.
- Given the daily trigger fires, when both sources respond, then a normalized candidate list of formats exists with id, name, rank/score, image url, and timestamp.
- If one source fails, the run continues on the other and logs the failure.

**P0-3 — Velocity scoring with persisted state.** Rank formats by acceleration using yesterday's snapshot.
- Given a prior snapshot exists, when scoring runs, then each format has a velocity score combining rank movement, upvote acceleration, and a freshness decay.
- Given no prior snapshot (first run), then the engine seeds today's snapshot and ranks on available same-day signal.

**P0-4 — Top-3 selection with stale-format filter.** Return the 3 highest-velocity formats not on the used/stale list.
- Given scored formats, when selection runs, then exactly 3 formats are returned, none previously used or flagged past-peak.

**P0-5 — Product-aware caption generation.** For each of the 3 formats, generate a caption embedding the product use-case, tuned to the persona.
- Given a format and a brief, when captioning runs, then the caption fits the format's narrative and references the product's use-case in the persona's language/tone.

**P0-6 — Meme render.** Produce a viewable image per caption.
- Given a captioned format, when rendering runs, then a meme image is produced and is legible at typical feed size.

**P0-7 — Mockups to review board.** Deliver the 3 mockups for human approval.
- Given 3 rendered mockups, when delivery runs, then all 3 attach to the brief's ClickUp task and the operator can approve/reject by changing status.

### Nice-to-Have (P1)

- **P1-1 — Google Trends validation** of top movers before they reach a brief.
- **P1-2 — Know Your Meme safety screen** surfaced to the operator alongside each pick (context note + risk flag).
- **P1-3 — Caption variants:** generate 2 caption options per format so the operator picks the stronger one.
- **P1-4 — Geography filtering** of the candidate pool by language/region.

### Future Considerations (P2)

- **P2-1 — Publishing & scheduling** of approved mockups via ClickUp calendar + Blotato.
- **P2-2 — Short-video content type** reusing the same trend engine.
- **P2-3 — Client-facing intake & portal** (multi-tenant), with billing and any paid trend-source tier.
- **P2-4 — Regional trend sources** (e.g. TikTok Creative Center) behind the geography gate, for available markets only.
- **P2-5 — Performance feedback loop:** feed post engagement back into format scoring.

---

## 8. Success Metrics

**Leading (days to weeks)**
- **Trend freshness:** ≥ 60% of selected formats are rising (positive velocity) at time of selection. Stretch: 80%.
- **Mockup approval rate:** ≥ 50% of generated mockups approved without edit in the first month. Stretch: 70%.
- **Cycle time:** brief-to-reviewable-mockups in a single daily run, under 1 hour of compute/wait. Target met when 90% of runs clear it.
- **Cost per brief:** ≤ $0.10 (LLM + render only). Stretch: ≤ $0.03.

**Lagging (weeks to months)**
- **Operator time saved** vs. manual meme production, self-reported.
- **Format hit rate:** of published memes (Phase 2), share that beat the operator's baseline engagement.
- **Reuse without rework:** declining edit rate on captions over time as prompts tune.

Measurement is manual in the MVP (a tab in the same Sheet that stores snapshots). No analytics tooling required yet.

---

## 9. Open Questions

- **(Render) Does Blotato render text-on-template memes, or is it the publish layer only?** Determines whether Imgflip `caption_image` is the MVP renderer and Blotato moves entirely to Phase 2. *Blocking the render step.*
- **(Data) Google Sheet vs. SQLite/Postgres for snapshot state?** Sheet is zero-setup; a DB scales better. *Non-blocking; Sheet is fine to start.*
- **(LLM) Which model for captioning under the near-zero-cost goal?** Azure AI Foundry, a cheap hosted model, or a small local model. *Non-blocking; affects cost line only.*
- **(Data) Is the optional Reddit RSS feed worth wiring?** Test whether `/r/MemeTemplates/top/.rss` still returns data for personal use; skip if it doesn't. *Non-blocking; Imgflip + Giphy stand alone.*
- **(Scope) Geography for the first briefs?** English/global keeps the gate trivial; non-Latin or RTL scripts add render work. *Non-blocking; assume English-first.*

---

## 10. Timeline & Phasing

**Phase 1 — Trend engine (build first).** Daily Imgflip + Giphy pull, normalize, snapshot, velocity score, top-3 with stale filter. Verifiable on its own before any creative work: eyeball whether the top 3 are genuinely rising.

**Phase 2 — Creative loop.** Brief intake, persona-aware captioning, render, mockups to the ClickUp board, human approval.

**Phase 3 — Publish & scale.** Approved-mockup scheduling via ClickUp + Blotato, Google Trends + KYM screens, caption variants.

**Phase 4 — Productize.** Client intake/portal, paid/regional trend sources, short-video content type, performance feedback loop.

Dependencies: Phase 3 publishing depends on confirming Blotato's role (Open Question 1). Phase 2 delivery depends on choosing a review sink (ClickUp API token vs. Google Sheet/email).

---

## 11. Cost & Constraints Summary

- **Reddit is out.** New personal Reddit API/OAuth access closed around Nov 2025. Giphy replaces it as the early-trend signal (free key, no OAuth). An optional Reddit RSS branch is best-effort only.
- **Imgflip and Google Trends are free.** Imgflip `caption_image` covers simple meme rendering at no cost.
- **TikTok excluded** (no free API, regional bans, scraping upkeep).
- **Only real MVP spend:** caption LLM calls and any non-free render. Both small and per-brief.
- Confirm Giphy, Imgflip, and any render-tool terms at build time; platform terms shift.
