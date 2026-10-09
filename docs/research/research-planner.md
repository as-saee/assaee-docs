# As-Sa'ee — Daily Planner module research (Oct 2026)

## 1. What existing apps get right and wrong

**What drives daily use**
- **A short "Today" list you choose yourself.** Things 3's Today/This Evening split and Sunsama's guided morning planning ("what will you do today, what can wait?") keep the day realistic. Sunsama's weakness: if you skip the ritual for a few days, the tool stops working. Keep the ritual optional and under 60 seconds. [Things](https://culturedcode.com/things/features/), [Sunsama alternatives (Ellie)](https://ellieplanner.com/comparisons/sunsama-alternative), [sortedbrain](https://sortedbrain.com/blog/day-to-day-planner-app)
- **A separate "do" date and "due" date.** Todoist added Deadlines in Jan 2025 so that the date you plan to work on something is separate from the date it must be done. The deadline stays fixed while the planned date moves. [Todoist help](https://www.todoist.com/help/articles/add-deadlines-to-tasks-jan-7-WFbv37eEw)
- **A visual timeline of the day** (Structured). Its weakness: an empty, unhelpful timeline on days you didn't plan. [routine.co](https://routine.co/blog/posts/apps-like-structured)
- **Fast capture and fast check-off.** Streaks caps you at 24 habits on purpose. The top complaints about it are a check-off that takes too many taps and no way to log after midnight for people who log at bedtime. [Streaks review](https://makeheadway.com/blog/streaks-app-review/), [App Store reviews](https://apps.apple.com/us/app/streaks/id963034692?see-all=reviews)
- **Bloat that rarely gets daily use:** gamification (Habitica's avatar damage drives guilt spirals and people quit), Pomodoro timers, Eisenhower matrices, tags plus filters plus priorities all at once, and team features. [Habitica review](https://just-habits.com/blog/habitica-review/), [habithuddle](https://habithuddle.com/blog/habitica-alternatives)

**Muslim-focused planners**
- **Zarrah** shows Fajr, Sunrise, Dhuhr, Asr, Maghrib, Isha, Islamic midnight and the last third of the night. Tasks can be pinned to a fixed time or *relative to a prayer* ("after Fajr", "before Maghrib"). [Play](https://play.google.com/store/apps/details?id=com.getzarrat.app&hl=en_US)
- **Solah** (launched Sep 2026) splits the day into the windows between prayers instead of hours. It has three steps: intention (niyyah), work planned by window (ʿamal), and a nightly review (muḥāsabah). [solah.app](https://solah.app/)
- **PrayerCal** only pushes prayer times into a calendar. [prayercal.com](https://prayercal.com/)
- **Takeaway:** prayer windows as "buckets" work better than exact time slots. Times relative to a prayer are the feature that sets these apps apart. A short evening review is the habit loop.

**What the evidence says about habits and streaks**
- Strict all-or-nothing streaks are the most cited reason people abandon habit apps ("streak anxiety").
- Lally et al. (2010): missing a single opportunity did not measurably harm habit formation (see §5a).
- **"Never miss twice"** is the widely recommended rule. [tracebyme](https://tracebyme.com/blog/habit-tracker-streak-anxiety-two-day-rule/), [habitdoom](https://habitdoom.com/blog/streak-anxiety-habit-trackers)
- Loop Habit Tracker uses a **habit-strength score** (an exponentially weighted average). A few misses after a long run don't wipe out progress, and the score handles "N times per week" habits. [uhabits](https://github.com/isoron/uhabits)
- Strik **earns freezes** (one per 7 consistent days) and applies them automatically. [Play](https://play.google.com/store/apps/details?id=com.hugocodes.strik&hl=en)
- Caveat: blog statistics like "63% more likely to quit" and "37% longer" have no primary sources. Treat them as directional only.
- **Recommendation:**
  - Make strength or consistency % the main number and show the streak second.
  - Use **per-period targets** (e.g. 3 times per week). The streak counts periods met, not days.
  - Allow a miss without breaking the streak, and show a "don't miss twice" nudge after a miss.
  - Prayers are not streak-gamified. Log them neutrally, with qada/late states. This is a design choice, not something the research covers.

## 2. Data model (SQLite with drift, mirrored in Postgres)

**Sync columns on every synced table**
- `id` UUIDv7 generated on the client. Dart: `uuid` package. Postgres 18 has `uuidv7()` and Python 3.14 has `uuid.uuid7()`. [PG docs](https://www.postgresql.org/docs/current/functions-uuid.html), [paulox](https://www.paulox.net/2025/11/14/how-to-use-uuidv7-in-python-django-and-postgresql/)
- `created_at`, `updated_at` (UTC, set by the client).
- `deleted_at` for soft deletes (tombstones, never hard-delete).
- `server_version` or `synced_at` so the client can pull only rows changed since its last sync.

**Last-edit-wins pitfalls**
- Device clocks drift. Have the server stamp `server_updated_at` and use it as the pull cursor.
- LWW at the row level loses concurrent edits to different fields. That's acceptable for one user. Per-field timestamps are optional later.
- Never reuse ids. Never cascade hard deletes.

**`task`**
- `title`, `notes`, `parent_id` (subtasks; one level is enough), `project_id`/`list_id`, `priority` (optional).
- `status` (open/done/cancelled), `completed_at`.
- Dates:
  - `scheduled_date` (the "do" date, a local date with no time).
  - Optional `scheduled_time` *or* `anchor` (see below).
  - `deadline_date` (the "due" date).
  - `estimate_min`.
- Recurrence:
  - `rrule` TEXT (an RFC 5545 RRULE string), `dtstart_local` and `tz` (IANA name, e.g. `Europe/Madrid`).
  - `recurrence_mode`: `fixed` (repeats on the schedule) or `after_completion` (Todoist's "every!" vs "every"). Plain RRULE can't express after-completion.
- Recurring tasks: keep **one template row and create the next instance when the current one is completed**. Don't expand every future instance into rows; it causes sync churn and duplicates.
- Exceptions: an `occurrence_override(task_id, original_date, new_date|skipped)` table, the RFC 5545 EXDATE/RECURRENCE-ID idea.

**Prayer anchor (stored on task/plan item)**
- `anchor_prayer` (fajr|sunrise|dhuhr|asr|maghrib|isha|null), `anchor_rel` (before|after|between), `anchor_offset_min`.
- Store the anchor, **not** the computed time. Times are recomputed daily from location, calculation method and madhab.
- Prayer times are *derived* data. Compute them on the device and don't sync them. Do sync the settings (location, method, madhab, offsets).

**`habit`**
- `title`, `kind` (yes/no | count | duration), `target_value` (e.g. 20 pages, 30 min).
- `period` (day|week|month), `target_per_period` (e.g. 3 per week), optional `days_mask` (allowed weekdays).
- Optional prayer anchor, `reminder` settings, `archived_at`.

**`habit_checkin`**
- `habit_id`, `local_date` (the user's logical day, not a UTC date), `value`, `note`, `source` (manual|module).
- Unique on (habit_id, local_date) for yes/no habits.
- Streaks and strength are **computed** from check-ins and never stored. That avoids sync conflicts on counters.

**Cross-module feed: `plan_item`**
- Columns:
  - `source_module` (reader|fitness|...), `source_ref` (that module's entity id), `kind` (task|habit_checkin|event)
  - `title`, `local_date`, an optional anchor or time, `estimate_min`, `status`
  - `payload` JSON (e.g. `{book_id, pages: 20}`), `deep_link` (route to open in the source module)
- Unique on `(source_module, source_ref, local_date)` so pushing the same item again is idempotent.
- Modules write through a single `PlannerFeed` API, e.g. `upsertItem` / `completeItem`.
- Completing an item in the planner raises an event the source module listens to. For example, the reader logs pages; the planner never reaches into reader tables.
- If the source deletes its goal, the source soft-deletes its items too.

## 3. Flutter packages (checked Oct 2026)

| Need | Package | Status |
|---|---|---|
| Local DB | `drift` 2.35.x plus `drift_flutter` 0.3.x. Reactive streams, migrations, isolates. Sponsored by PowerSync and Stream. | Very active. [pub](https://pub.dev/packages/drift), [2026 DB guide](https://luci-studio.com/blog/the-flutter-local-database-landscape-in-2026-a-maintenance-first-guide-fe6d267c/) |
| Recurrence | `rrule` 0.2.18 (wanke.dev). Parses and expands RFC 5545 strings, ~126k downloads/month. Alternative: `teno_rrule` (young, 0.0.x). | Maintained but slow. Pin the version and wrap it behind your own interface. [pub](https://pub.dev/packages/rrule) |
| Prayer times | `adhan_dart` 2.x, a port of Adhan JS (MIT). Supports methods, madhab, high-latitude rules and adjustments. | Active. [pub](https://pub.dev/packages/adhan_dart) |
| Notifications | `flutter_local_notifications` 22.3.x (v23 in dev). Use `zonedSchedule` with the `timezone` package. | Very active. [pub](https://pub.dev/packages/flutter_local_notifications) |
| Day view | `kalender` (day, multi-day and schedule views, drag-to-resize), `calendar_view` (Simform), `calendar_day_view`. Syncfusion needs a commercial licence. A **custom prayer-window list** is likely simpler than a calendar grid for v1. | [fluttergems calendar list](https://fluttergems.dev/calendar/) |
| Home widget | `home_widget` 0.10.0 (Sep 2026). Native UI layouts, data from Dart, interactive callbacks. Fits the existing Kotlin `ReadingWidgetProvider`. | Active. [pub](https://pub.dev/packages/home_widget) |
| IDs | `uuid` (supports v7) | Active |

**Exact alarms on Android 14+**
- `SCHEDULE_EXACT_ALARM` is **denied by default** on new installs, so the user has to grant "Alarms & reminders". [Android docs](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms)
- `USE_EXACT_ALARM` is granted automatically, but Play only allows it for apps whose *core* feature is alarms, timers or calendar event notifications. A planner is borderline; prayer and adhan apps often use it. [Android alarms](https://developer.android.com/develop/background-work/services/alarms)
- **Plan:**
  - Task and habit reminders use `inexactAllowWhileIdle`. A few minutes late is fine.
  - Prayer-anchored reminders use exact scheduling only if the user grants it, with an inexact fallback.
  - Check `canScheduleExactNotifications()`.
- Limits:
  - Samsung allows at most ~500 pending alarms and iOS keeps 64. **Schedule only a rolling window of 1–2 days** and refresh it on app open, after a reboot (`RECEIVE_BOOT_COMPLETED`), and on timezone or location change.
  - OEM battery killers (Xiaomi, Samsung) also delay alarms. Prompt the user to exempt the app from battery optimisation. [FLN #2185](https://github.com/MaikuB/flutter_local_notifications/issues/2185), [FLN #2339](https://github.com/MaikuB/flutter_local_notifications/issues/2339)

## 4. Python / FastAPI side

**Recurrence**
- Use `dateutil.rrule` / `rrulestr` only on the server: validation, previews and future Google Calendar sync.
- The client is the source of truth for creating instances. Don't let both sides create the next instance, or you get duplicates. If the server ever does, give the instance a deterministic id, e.g. uuid5(task_id + date).

**dateutil pitfalls**
- If DTSTART is timezone-aware, UNTIL must be in UTC. [dateutil #851](https://github.com/dateutil/dateutil/issues/851)
- Mixing naive and aware datetimes raises a TypeError.
- pytz gives wrong offsets across DST changes. Use `zoneinfo` (stdlib) or `dateutil.tz`, never pytz. [dateutil #641](https://github.com/dateutil/dateutil/issues/641)
- Safest approach: expand using **naive local wall-clock time** plus the IANA tz, then attach the zone to each occurrence. That way "08:00 every day" stays at 08:00 across DST.

**Time storage rules**
- Instants (completed_at, updated_at): `timestamptz` in UTC.
- "Do on date" values: a `date` column with no time and no zone.
- Wall-clock times: local time plus IANA tz. Never store a precomputed UTC instant for a recurring local event.
- Prayer anchors are local and depend on location. Moving from Pakistan to Spain changes every computed time, but no stored anchor needs to change.

**Keycloak:** validate JWTs in FastAPI (JWKS) and scope every row by `user_id = sub`, even with a single user.

## 5a. Behavioural psychology turned into features

Evidence ratings:
- **Strong:** meta-analysis or several replications.
- **Moderate:** solid but limited studies.
- **Weak:** mostly theory, or practitioner claims.
- **Contested:** replication failures.

**Implementation intentions** ("If it's after Asr, I will read 10 pages"). **Strong.**
- Gollwitzer & Sheeran 2006: 94 studies, about 8k people, d≈0.65 on reaching the goal. [KOPS](https://kops.uni-konstanz.de/handle/123456789/10973)
- **Feature:** every habit and plan item has an optional "when/where" cue. The prayer anchor *is* the cue. The editor phrases it as an if-then sentence.

**Habit loop: context cues beat reminders.** **Moderate.**
- Stawarz et al. (CHI 2015) reviewed 115 apps and ran a 4-week study. Reminders raise compliance but slow down automaticity. Cues tied to an event (an existing routine) build habit. [UCL](https://discovery.ucl.ac.uk/1468224/)
- Habit formation takes a median of 59–66 days, with a range of 4–335 days. Morning habits and self-chosen habits form more strongly. Lally 2010 found that missing one opportunity didn't materially hurt formation. [Singh 2024](https://www.mdpi.com/2227-9032/12/23/2488), [Surrey/Lally](https://www.surrey.ac.uk/news/does-it-really-take-66-days-form-habit-we-asked-expert-dr-pippa-lally)
- **Features:**
  - Anchor habits to salah, which is an existing 5×/day routine: habit stacking with an unusually strong cue.
  - Let the user turn a reminder off once the habit feels automatic.
  - Show "~2 months, varies widely" instead of promising 21 days.

**Habit stacking / Tiny Habits (Fogg).** **Weak-to-moderate.**
- The mechanism is the same as implementation intentions. The Tiny Habits method itself has little peer-reviewed evidence. [productive.fish summary](https://productive.fish/blog/habit-stacking/)
- **Feature:** a "starter version" field per habit (e.g. "1 page"), offered when two misses happen in a row.

**Self-determination theory: autonomy, competence, relatedness.** **Strong.**
- Deci, Koestner & Ryan 1999 (128 experiments): expected tangible rewards undermine intrinsic motivation, while informational feedback supports it. Streak punishment works through guilt ("introjected" motivation), which doesn't last. [meta-analysis PDF](https://depts.washington.edu/techdocs/papers/deciExtrinsicRewardsAndIntrinsicMotivation99.pdf)
- **Features:**
  - No points or XP.
  - The user picks the habit and its target.
  - Feedback reads as information ("4 of 5 this week, strongest after Fajr"), not judgement.
  - A "why" (niyyah) field shown at check-in.

**Loss aversion and streaks.** **Moderate.**
- Silverman & Barasch (JCR 2023, 7 studies): showing an intact streak increases engagement. Showing a broken one reduces it, independent of actual behaviour, and more so when people blame themselves. **Letting users "repair" a streak reduces the damage.** [INSEAD](https://www.insead.edu/faculty-research/publications/journal-articles/or-track-how-broken-streaks-affect-consumer)
- **Features:**
  - Display streaks as weeks of target met, not consecutive days.
  - One repair per period.
  - Excused days (travel, illness, menstruation for prayer and fasting habits) don't break the streak.
  - Never show a red "0".

**Fresh start effect.** **Moderate-strong.**
- Dai, Milkman & Riis 2014: goal-seeking spikes after landmarks such as a new week, month or birthday. [Mgmt Sci](https://pubsonline.informs.org/doi/10.1287/mnsc.2014.1901)
- **Feature:** offer a gentle "start fresh" prompt on Jumu'ah (Friday), the new Hijri month, Ramadan, and after a lapse. Archive old misses instead of showing them.

**Planning fallacy.** **Strong.**
- Buehler, Griffin & Ross 1994: actual completion (55.5 days) took longer than even the worst-case estimate (48.6). [JPSP PDF](https://web.mit.edu/curhan/www/docs/Articles/biases/67_J_Personality_and_Social_Psychology_366,_1994.pdf)
- Breaking a task into steps reduces the bias. [Kruger & Evans 2004](https://www.sciencedirect.com/science/article/abs/pii/S002210310300177X), [Forsyth & Burt 2008](https://pubmed.ncbi.nlm.nih.gov/18604961/)
- **Features:**
  - Optional estimate per task.
  - Each prayer window shows "planned 140 min / available 95 min".
  - Record actual vs estimated time and show the user's personal overrun ratio (the "outside view").
  - Prompt for subtasks on big tasks.

**Zeigarnik effect.** The memory claim is **contested**: a 2025 meta-analysis found no recall advantage once Zeigarnik's own data is excluded. The "open loops intrude" claim holds up better. Masicampo & Baumeister 2011: *making a specific plan* removes the intrusive thoughts from unfinished goals. [JPSP PDF](https://users.wfu.edu/masicaej/MasicampoBaumeister2011JPSP.pdf), [meta-analysis](https://www.researchgate.net/publication/393261073_Interruption_recall_and_resumption_a_meta-analysis_of_the_Zeigarnik_and_Ovsiankina_effects)
- **Feature:** in the evening review, every undone item must be *placed* (tomorrow, a date, someday, or dropped). That closes the loop and supports a clear mind for sleep.

**Decision fatigue / ego depletion.** **Contested.**
- The 23-lab replication (Hagger et al. 2016) found about zero effect. The parole-judges study is confounded by how cases were ordered. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4971805/)
- Task-switching and cognitive-load costs are still real.
- **Feature:** decide the plan in one place, so the next prayer window shows "the next 1–3 things". That's justified by switching costs, not by the "willpower battery" idea.

**Why people abandon planner apps** (practitioner evidence, **weak-moderate**):
- Setup takes too long, and the overdue list piles up into guilt.
- A daily ritual that's required (Sunsama) or punishment (Habitica).
- Logging is slow; the bedtime-logging-after-midnight case.
- **Features:**
  - Auto-roll overdue items into a quiet "carry-over" tray, not a red count.
  - First run: 3 habits maximum and pre-filled prayer anchors.
  - Check-in in one tap from the widget or notification.

## 5b. How older traditions structured the day, turned into features

**al-Ghazali, *Iḥyāʾ* Book 10, *Tartīb al-Awrād*.**
- Splits the day into **seven awrād** (portions), e.g. dawn→sunrise, then two between sunrise and zawal, and so on. The night has four: two from Maghrib to sleep and two from the second half of the night to Fajr.
- Each portion has a fitting activity: dhikr and Qur'an after Fajr, earning a living and learning in the forenoon, and so on. It is meant to suit workers, students and families. [ghazali.org](https://www.ghazali.org/books/ghazali_worship.htm), [Fons Vitae Book 10](https://fonsvitae.com/product/al-ghazali-the-arrangement-of-the-litanies-and-the-exposition-of-the-night-vigil-book-10/)
- **Features:**
  - The day view is built from **windows (awrād), not hours**.
  - Each window gets a default *character* (e.g. Fajr→sunrise = Qur'an, adhkar and deep work; post-Dhuhr = light admin and qaylula).
  - The user assigns tasks to a window, and an exact time is optional.

**Salah as the anchor of the day** in classical Muslim towns, where markets and schedules were set by the adhan.
- The Prophet's prayer "O Allah, bless my Ummah in its early mornings" (Abu Dawud, Tirmidhi) and dispatching expeditions and trade at dawn. [hadeethenc](https://hadeethenc.com/en/browse/hadith/5941)
- **Feature:** a "Bakūr" (early-morning) slot after Fajr, suggested for the hardest task of the day.

**Muḥāsaba (self-accounting).**
- ʿUmar: "Take account of yourselves before you are taken to account." Al-Muḥāsibī's version looks *back* (audit the day) and *forward* (check tomorrow's intentions). [Wikipedia](https://en.wikipedia.org/wiki/Muhasaba), [al-Muhasibi](https://hamzathehistorian.substack.com/p/islamic-self-psychology-muhasabat)
- **Feature:** a 2-minute **evening muhasaba** after Isha, in two parts:
  - Looking back: what got done, what was missed and why, a one-line gratitude or regret.
  - Looking forward: place tomorrow's items into windows and set one niyyah.
- This is the planner's main daily ritual, and the weekly review later grows out of it.

**Stoic evening review.**
- Seneca, *De Ira* 3.36, quoting Sextius: "What bad habit have you cured today? What fault resisted? In what are you better?" Seneca says it brings "tranquil and deep" sleep. [Perseus](http://www.perseus.tufts.edu/hopper/text?doc=Sen.+Ira+3.36&lang=original)
- **Feature:** the same evening review, with an optional rotating question ("one fault resisted today?").

**Benedictine horarium** (Rule ch. 48).
- Fixed Hours: Vigils, Lauds, Prime, Terce, Sext, None, Vespers, Compline.
- Set blocks of manual work and *lectio* (holy reading) between them. Work and prayer are one rhythm (*ora et labora*). [Wikipedia](https://en.wikipedia.org/wiki/Rule_of_Saint_Benedict), [NLM](https://www.newliturgicalmovement.org/2025/02/the-rhythms-of-day-and-night-in-rule-of.html)
- **Feature:** **day templates**, i.e. a repeating default shape per weekday (e.g. "workday", "Jumu'ah", "rest day"). Daily planning then only *adjusts* a template and never starts from blank. This also fixes Structured's empty-timeline problem.
- A daily reading block (lectio) maps to the Reader feed.

**Agrarian and seasonal rhythms.**
- Hesiod's *Works and Days* timed work to the Pleiades. Medieval Books of Hours gave each month its "labour". [Labours of the Months](https://en.wikipedia.org/wiki/Labours_of_the_Months)
- In Spain, summer Isha is around 23:30 and winter Fajr is late, so the day's shape really does change with the season.
- **Features:**
  - **Seasonal modes:** Ramadan (suhoor/iftar windows, taraweeh, lighter targets), summer and winter, travel (shortened/combined prayers, paused habits).
  - Habit targets can vary by mode.

**Qaylula and segmented sleep.**
- The midday nap (qaylula) is Sunnah. [SeekersGuidance](https://seekersguidance.org/answers/general-counsel/the-sunna-of-taking-a-midday-nap/)
- Spain's siesta culture fits it well.
- Ekirch's "first and second sleep", with a night wake (cf. tahajjud in the last third), is **contested** as a universal pattern. [Ekirch](https://harpers.org/archive/2013/08/segmented-sleep/), [Medical History critique](https://www.cambridge.org/core/journals/medical-history/article/have-we-lost-sleep-a-reconsideration-of-segmented-sleep-in-early-modern-england/B70D0BFF8E77CFB81A839E9B72240CF2)
- **Features:**
  - Optional rest blocks: qaylula before or after Dhuhr, and a "last third of the night" block for tahajjud.
  - The available-time calculation subtracts sleep and rest, so the capacity check is honest.

## 5. Pitfalls

1. **Where the day ends.** Isha in Madrid in summer is around 23:30, and night prayers can run past midnight. Use a configurable "day rollover" (e.g. 03:00, or at Fajr) for `local_date`, so check-ins made at 00:30 still count for "today". This fixes the main Streaks complaint.
2. **Prayer times change daily and with location.** Recompute each day. Watch for an anchored task whose window disappears (e.g. "between Asr and Maghrib" while travelling). Tasks that don't fit fall into an "Anytime" bucket.
3. Late-night summer and high-latitude edge cases: pick the method explicitly (e.g. Muslim World League, or a Spain-specific option), set the madhab for Asr, and set a high-latitude rule.
4. Pre-expanding recurring instances causes sync duplicates. Generate them lazily.
5. Storing streak counters creates LWW conflicts. Derive streaks instead.
6. Notification over-scheduling, exact-alarm denial and OEM battery killers all cause silent failures.
7. Feature creep in the cross-module feed. Use one contract (`plan_item`). Modules never touch planner tables directly.
8. Timezone travel. Prefer the device's current tz for display, but keep each task's own tz for recurrence.

## v1 feature list
- [ ] Today view grouped by prayer windows (Fajr→Sunrise, Sunrise→Dhuhr, Dhuhr→Asr, Asr→Maghrib, Maghrib→Isha, After Isha) plus an "Anytime" bucket.
- [ ] Quick-add task with do date, optional deadline, optional prayer anchor (before/after X ± min) or fixed time.
- [ ] Subtasks (one level), recurrence (daily / weekly on days / monthly / every N; fixed vs after-completion) stored as RRULE.
- [ ] Inbox, Upcoming (7 days) and Overdue → "move to today" in one tap.
- [ ] Habits: yes/no or count, N per week targets, check-in from the Today view and widget, strength %, forgiving streak with a "don't miss twice" nudge.
- [ ] Prayer logging (on time / late / qada / missed), neutral with no streak shame.
- [ ] Feed items from Reader (pages goal) and Fitness (workout), with deep links and completion written back.
- [ ] Reminders: inexact by default, optional exact for prayer-anchored items, rolling 48 h scheduling.
- [ ] Home widget: today's next items in the current prayer window.
- [ ] Offline-first drift DB plus LWW sync (UUIDv7, soft delete, server cursor).
- [ ] 2-minute evening muhasaba after Isha (looking back, then placing every undone item and setting one niyyah for tomorrow). This is the core ritual (§5b).
- [ ] An if-then cue per habit and plan item (the prayer anchor), a niyyah/"why" field, and a "starter version" offered after two misses in a row (§5a).
- [ ] Day templates per weekday (workday / Jumu'ah / rest), so planning only adjusts a template.
- [ ] Forgiving streaks: per-week targets, one repair per period, excused days, a "start fresh" prompt on Friday and the new Hijri month.
- [ ] Capacity hint per prayer window (estimates vs available minutes).

## Not in v1
- Drag-and-drop time-blocking on a calendar grid (v1 only shows a capacity hint per window)
- Google Calendar two-way sync (later via server, RRULE already compatible)
- Weekly review screen, stats dashboards, heatmaps beyond a simple habit calendar
- Gamification (XP, avatars), Pomodoro, Eisenhower, labels/filters query language
- Natural-language date parsing (nice-to-have; v1.1)
- Seasonal modes (Ramadan/travel) and estimate-vs-actual tracking: v1.1. Leave space in the schema for them now (`mode` on habit targets, `actual_min` on task).
- iOS (keep the 64-notification limit and the WidgetKit split in mind in the design)
- Shared/team lists, attachments, location-based reminders
