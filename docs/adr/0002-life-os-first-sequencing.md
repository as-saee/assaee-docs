# ADR 0002: Life OS first: sequencing, access and sync decisions

- **Status:** Accepted, 2026-11-10.
- **Deciders:** the author.
- **Amends:** [ADR 0001](0001-platform-and-product-decisions.md) (order of work). Everything in 0001 not mentioned
  here stands.
- **Related:** [`As-Saee-Master-Plan.md`](../../As-Saee-Master-Plan.md) (the vision; this record and the roadmap stay the authority).

## Context

The master plan consolidates the roadmap with a longer-term vision. Reviewing it raised questions the plan left open:
when new data concepts arrive, how the phone reaches a not-yet-authenticated API, what the author actually uses
between M1 and M5, and which piece of the Planner depends on a later milestone. The first goal is a Life OS the author
uses daily. The platform remains the career evidence.

## Decisions

| Area | Decision | Reason |
|---|---|---|
| Document roles | The master plan is the vision. The roadmap tracks status, ADRs record decisions, code and tests show what is real. A plan change that touches a milestone becomes a roadmap edit or an ADR. | One source of truth per question. |
| Pursuit and checkpoint | No pursuit, link, event or checkpoint tables before M5. The first piece is a one-line "where I left off" note on a Reader passage. A separate pursuit record arrives when a second module needs to link to it. A client-generated `event_id` on touches is added in M1. | Tables no screen uses are guesses about shape; the Postgres move in M3 is the real work. `event_id` costs one column and makes retries safe. |
| Access before login | Tailscale moves from M4 to M1. The API and ntfy are reachable only over the tailnet; only `/health` is public. | Simplest safe interim, and nothing to throw away when Keycloak arrives in M4. |
| Thin mobile app | A thin Flutter app (thread list, Touch, home widget) arrives around M2–M3. Touches queue offline with `event_id`s and the server ignores duplicates. Full sync of editable data waits for M5. | Gives a daily-use surface early and starts mobile CI before M5. Touches are events that cannot conflict. |
| Nudge sender | ntfy is the only sender through M5. The handoff to local notifications is decided in the Planner milestone. | Server push survives a killed app; local alarms matter only for exact prayer-anchored items. |
| Order after M5 | M6 Planner, M7 Fitness, M8 Deen. The AWS replica becomes side track S1, time-boxed to about 2–3 weeks, runnable any time after M4, never blocking M5–M8. | Daily-use modules first without dropping AWS evidence. |
| Prayer times | The prayer-time and window calculation (on-device `adhan_dart`, MWL default, location, mosque offsets) is built with the Planner in M6. Deen adds qibla, Quran, audio, hifz and adhkar in M8. | The Planner's Today view needs prayer windows. |
| Threads and habits | They stay separate. Finishing a workout or reading session touches a thread the author linked; nothing is linked by guessing from names. | Thread creation stays name-only. |
| Lived-in check | After M1, M5 and M6: about four weeks of real use, with private notes on what was used, skipped and annoying. It runs alongside the next module's research. | Tests that it works for the author, not only that the software works. |
| Phone data protection | Rely on Android storage encryption and a screen lock. Lock-screen notifications use neutral wording. Add app-level encryption for sensitive data (reflections, prayer logs, health) in M7 and M8. | Matches the real risk; stronger protection when sensitive data exists. |
| Reader format | PDFs only in M5. EPUB is decided after the lived-in check. | M5 is the proof that the platform serves a real app; keep it narrow. |
| Capture inbox | Saves the link, the time and an optional one-line note. No page fetching. | Social links are often unfetchable, and fetching adds outside requests. |
| Devices | One device in year one. Adding a second device is a trigger to redesign sync (last-edit-wins is enough for one device). | Keeps M5 sync simple. |
| Intellectual Companion | A committed destination, not an optional extra, but resource-gated with no date. RAG and fine-tuning are methods, not milestones. The gate criteria are not yet defined. | Separates "will we" from "when and how". |

## Consequences

- **Good:** a daily-use app arrives before the Reader; Planner and prayer windows are no longer blocked by Deen; no
  unused data tables to migrate in M3.
- **Costs:** a few days for the thin app in M2–M3; Tailscale on the phone is required for Done to work; milestone
  numbers after M5 changed (old M7 to M6, old M8 to M7, old M9 to M8, old M6 to S1).
- **Revisit when:** a second device is added; the thin app shows nudges missing because Android stops the background
  app; most of the author's reading turns out to be EPUB; or the lived-in check shows the heartbeat rules are noise.
- **Open:** what resource gate must be met before Companion work starts, and when it is reviewed.
