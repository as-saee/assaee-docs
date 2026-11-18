# ADR 0003: Dig: guided research from a capture-inbox item

- **Status:** Accepted, 2026-11-18. Build is deferred until after the first lived-in check.
- **Deciders:** the author.
- **Related:** [ADR 0002](0002-life-os-first-sequencing.md) (capture inbox, lived-in check, Companion gate),
  [`As-Saee-Master-Plan.md`](../../As-Saee-Master-Plan.md) (the "Discover" surface).

## Context

A reel or post on a topic (a Quran theme, a period of history, any subject) catches attention, and today it is
forgotten or lost in a feed. The capture inbox saves the link, but saving is not understanding. The author wants to
be taken deeper into the topic, step by step, until they feel they have dug enough. Social links cannot be fetched
(ADR 0002), so the app cannot know what a reel is about unless the author says so.

## Decisions

| Area | Decision | Reason |
|---|---|---|
| Topic | Comes from the author's one-line note on the inbox item. The app never tries to read the reel. | Social links are unfetchable; a short note is cheap and exact. |
| Entry point | A **Dig** button on an inbox item. | The inbox is the front door; one extra tap, no new screen to remember. |
| AI | The core has no AI. The design keeps a plug-in slot (summaries, next-step suggestions, topic guess) for the later Companion work. | Works offline and for free; AI later fits the Companion plan without a rewrite. |
| Sources | A list the author curates per topic type (for example Quran and tafsir for deen, encyclopedia and scholarship for history). The app only offers sources from the list. | Predictable and safe; religious topics always start from verified sources. A "search wider" button may come later. |
| Where the list lives | In the phone app, editable there. May move to the API later. | App comes first and works with the server off (ADR 0002 order). |
| Finding articles | 1) Search inside a listed source that has a free search. 2) Otherwise open that source's own search page in a reader view. 3) The author can also store direct starting links per topic. | Clean results where possible, a fallback everywhere else. |
| Reading | A reader view that strips the page to text. No summarising. | The author reads the sources; no AI needed. |
| Depth levels | Three fixed levels: **Overview**, **Sources**, **Primary**. The author tags each source with its level when curating. Custom ladders per topic type may come later. | Gives a sense of "deeper" without a score; the level describes the sources reached, not the person. |
| Done digging | A button that ends the dig and keeps the links opened, an optional "where I left off" line, and the level reached. Nothing is required. | Matches the "optional checkpoint" idea in the master plan; the author decides when it is enough. |
| Religious content | Dig never generates Quran or Hadith text. Deen topics start from verified Quran and tafsir sources, with exact attribution. | Hard rule from ADR 0001 and the master plan. |
| Order | Not built before the first lived-in check. About four weeks of real use of the bare inbox first, then decide how much of Dig to build. | Real use shows whether notes pile up unread or get dug into. |

## Consequences

- **Good:** the main daily friction (a topic seen once, then lost) has a designed answer that needs no AI and no
  paid service. It adds practical platform work later (search calls, caching, rate limits).
- **Costs:** the author must maintain the source list. Sources without a free search only get the fallback.
  Fetching and cleaning outside pages adds outside requests the inbox deliberately avoided; Dig keeps that
  separate from the inbox.
- **Guardrail:** the depth level must never become a score, streak or reward.
- **Revisit when:** the lived-in check shows the inbox is not used; the source list becomes a chore; or the Companion
  gate is met and the AI slot is wanted.
- **Open:** how the reader view is built (which library); which free search interfaces the first sources offer; the
  exact fields saved on a finished dig.
