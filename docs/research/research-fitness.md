# As-Sa'ee fitness module: research (Oct 2026)

## 1. What strong workout loggers get right and wrong
**Right (copy these):**
- **Logging speed is the whole product.** Strong is built for someone resting between sets who needs to log fast. Each set row is prefilled from last time, and one tap on the checkmark logs it. Hevy prefills weight and reps from the previous session, set by set.
- **Previous performance shown inline.** A "Previous" column on each set row (e.g. `80 kg x 8`), filled from the last session of that exercise. Strong, Hevy and FitNotes all do this.
- **Rest timer starts when a set is checked off.** The default is set per exercise, with +/-15 s buttons, and it alerts with a notification and vibration when the app is in the background. Strong's timer is the more flexible of the two.
- **Templates/routines.** "Start workout from routine" copies the template into a live session. After the workout, offer "update template with today's values?" (Strong does this).
- **Automatic PR detection** with a small celebration: best weight, best estimated 1RM, best volume, and best reps at a given weight. Strong's PR animation gets positive mentions.
- **Exercise history and a chart per exercise:** estimated 1RM over time, plus best set per session.
- **Plate calculator and warmup set generator** (Strong). Cheap to build and well liked.
- **Boostcamp/Liftin'** do programs that progress on their own. **FitNotes** is liked for working fully offline, having no account, and its simple CSV export.

**Wrong (avoid these):**
- Free-tier limits on routines (Strong caps the number of templates and Hevy limits some features).
- Logging blocked by a social feed or account requirement (Hevy is social-first).
- **JEFIT** is cluttered and full of ads, with slow entry.
- Number entry that needs the system keyboard plus extra taps. Use a custom number pad or +/- steppers.
- Losing an in-progress workout when the app is killed. Write every set to SQLite the moment it is entered.

**Progression and 1RM:**
- **Double progression:** the routine sets a rep range (e.g. 3x8-12). Once every working set hits the top of the range, suggest adding weight (+2.5 kg upper body, +5 kg lower) and dropping back to the bottom of the range. Easy to implement as a suggestion shown on the set row.
- **Epley:** `1RM = w x (1 + r/30)`. **Brzycki:** `1RM = w x 36 / (37 - r)`. Both are only reasonable up to about 10 reps. Use Epley, and skip sets with more than 12 reps and warmup sets. If RIR is logged, an optional tweak is to use `r + RIR` as the effective reps.

**v1 must-haves:** prefilled sets with a Previous column, one-tap set completion, rest timer, routines, PR detection on estimated 1RM and best weight, exercise history and chart, crash-safe live session. **v1 nice-to-have:** a double-progression hint.

## 2. Open exercise datasets
| Dataset | License | Usable in a public open-source repo? |
|---|---|---|
| **yuhonas/free-exercise-db** (~870 exercises as JSON, with images, muscles, equipment, instructions) | **Unlicense (public domain)** | **Yes, with no conditions. Recommended seed.** |
| **wger** exercise data | Data is **CC-BY-SA** (3.0/4.0, with a license on each entry); code is AGPL-3.0 | Yes, but you must credit authors per entry and **share derived data under the same license**. Store `license` and `author` columns if you import it. Do not copy wger's code. |
| **ExerciseDB** (AscendAPI) | The free V1 dataset is **non-commercial only, with attribution required**. The GIFs and media are AscendAPI's property, and paid API access is a revocable subscription license. | **Avoid.** Do not bundle its GIFs or data in a public repo. |

Recommendation: seed from free-exercise-db, either compressing the images or loading them lazily from its GitHub-hosted URLs. Let the user add custom exercises. Keep a `source` field (`seed:free-exercise-db` / `user`) so the seed can be re-imported without clobbering the user's edits.

## 3. Data model (SQLite on device, mirrored in PostgreSQL)
Every synced table gets: `id UUIDv7 (text/uuid)`, `created_at`, `updated_at` (UTC ms), `deleted_at` (nullable, for soft deletes), and `dirty` (local only).
- **exercise**: name, primary_muscles[], secondary_muscles[], equipment, category (strength/cardio/stretch), `tracking_type` (weight_reps / reps_only / time / distance_time), source, `default_rest_s`, notes.
- **routine**: name, notes, sort_order. **routine_exercise**: routine_id, exercise_id, position, `superset_group` (nullable), rest_s, notes. **routine_set**: routine_exercise_id, position, set_type, target_reps_min/max, target_weight, target_rir.
- **workout_session**: started_at, ended_at, `tz_offset`, routine_id (nullable; what it was started from), name, notes, `planner_item_id` (nullable link to the daily planner), status (in_progress/finished).
- **session_exercise**: session_id, exercise_id, position, superset_group, notes.
- **set_entry**: session_exercise_id, position, `set_type` (warmup / working / drop / failure), `weight_kg` (REAL), reps, duration_s, distance_m, `rpe` REAL (6-10 in 0.5 steps) **or** `rir` INT (store both as nullable columns; RIR is roughly 10 - RPE), `completed_at`, `is_pr` (derived, can be recomputed).
- **body_measurement**: measured_at, `type` (weight / body_fat_pct / waist ...), value in SI units (kg, cm), source (manual / health_connect).
- **health_daily**, imported and read-only: date, steps (aggregated), `sleep_sessions` (start, end, stages JSON), source origins, plus `hc_record_id` for idempotent upserts.
- **settings**: `unit_system` (kg/lb), bar weight, available plates.

**Offline-sync implications:**
- **UUIDv7** gives client-generated IDs that are time-ordered, so they index well in Postgres. Dart has `uuid` (`Uuid().v7()`) and Postgres 18 has a native `uuidv7()`.
- **Always store kg internally** and convert only for display (1 lb = 0.45359237 kg). Round the display to plate steps (e.g. 2.5 lb, 1.25 kg). Store an `entered_unit` on each set so 225 lb round-trips exactly as 225 rather than 224.9.
- **Latest-edit-wins at row level is fine for one user**, with these caveats:
  - Sync the **session as one document**, or at least make child rows sync independently. A set edited on one device and a different set deleted on another should not clobber each other.
  - Never hard-delete. Use tombstones (`deleted_at`) and purge them after N days.
  - Use server-assigned `updated_at` or a hybrid logical clock. Phone clocks drift and that breaks latest-edit-wins.
- **Derived data** (PRs, estimated 1RM, volume) should be **computed, not synced**, or recomputed after a sync. Otherwise stale PR flags win the merge.
- **Health Connect data is a cache.** Health Connect is the source of truth, so key rows by HC record ID or by date and upsert idempotently.

## 4. Health Connect in 2026
**Flutter package:** `health` (carp.dk, a verified publisher). Current version is **13.3.2**, released about 50 days ago and actively maintained. It wraps both Health Connect and HealthKit.
- The `MainActivity` must extend **`FlutterFragmentActivity`**. This repo currently uses `MainActivity.kt`, so check it.
- Use `getTotalStepsInInterval()` for steps. This is the aggregated value and is deduplicated.
- Use `HealthDataType.SLEEP_SESSION` for sleep on Android. Stage types (SLEEP_DEEP / LIGHT / REM / AWAKE) only work if the writing app recorded stages.
- Android helpers: `isHealthDataInBackgroundAvailable()`, `requestHealthDataInBackgroundAuthorization()`, `isHealthDataHistoryAuthorized()`, `requestHealthDataHistoryAuthorization()`.
- Alternative: `health_connector` (it has `HealthPlatformFeature.readHealthDataHistory`).

**Manifest:**
- Permissions: `android.permission.health.READ_STEPS`, `android.permission.health.READ_SLEEP`, and optionally `READ_WEIGHT` and `WRITE_WEIGHT` so body weight syncs both ways.
- Add `READ_HEALTH_DATA_IN_BACKGROUND` and `READ_HEALTH_DATA_HISTORY` only if they are used.
- A privacy-policy rationale screen is required: an intent filter for `androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE`, plus an `activity-alias` named `ViewPermissionUsageActivity` with the `HEALTH_PERMISSIONS` category for Android 14+.
- Add a `<queries>` entry for the `com.google.android.apps.healthdata` package (Android 13 and lower, where Health Connect is a separate app).

**History limit:**
- By default an app can read only data from **30 days before permission was first granted**.
- On Android 14+ an app can read its own data with no limit, but other apps' data is still limited to 30 days. On Android 13 and lower the limit applies to all data.
- Reading anything older requires the runtime permission **`PERMISSION_READ_HEALTH_DATA_HISTORY`**, which must be declared and requested.
- Uninstalling and reinstalling resets the 30-day window.

**Background reads:**
- Check `FEATURE_READ_HEALTH_DATA_IN_BACKGROUND` is available, then declare and request `READ_HEALTH_DATA_IN_BACKGROUND`. Declaring it is not enough; it has to be requested at runtime.
- The documented pattern is a periodic WorkManager job, about every hour.
- Without this permission, reads only work while the app is in the foreground. For v1, just **sync on app open** plus a pull-to-refresh, which removes the need for the background permission.
- Rate limits: page through results (`pageSize` defaults to 1000; the old API cut off silently there), retry with backoff on `IllegalStateException` and `RemoteException`, and prefer `aggregate()` for steps.

**Incremental sync:**
- `getChanges(token)` returns only what was added, changed or deleted since the token was issued. **Tokens expire after 30 days.** On `ChangesTokenExpired`, fall back to a full time-range read and get a new token.
- The `health` package mostly exposes time-range reads, so a simple approach also works: re-read the last 7 days on each open and upsert by record ID.

**Deduplication:**
- `aggregate()` deduplicates steps and sleep using the user's source priority.
- `readRecords()` returns raw records **from every source**, so phone and watch steps would double-count. Never sum raw StepsRecords.
- Sleep: one night can be several `SleepSessionRecord`s that may overlap, so attribute each to a "sleep day" by its end time.

**Play Store (only if published there):**
- Required: the **Health apps declaration form** (Play Console > App content), a justification for each data type (Activity & fitness for steps, Sleep management for sleep, Body composition for weight), the Data Safety section, and a privacy policy both in the app and on the listing.
- Ask for the minimum data types only. Over-requesting is the main reason for rejection.
- Background and history permissions each need their own justification.
- **Sideloaded and debug builds are not gated by the declaration.** A personal sideloaded APK works without going through Play review, but manifest declarations are still required.

**iOS HealthKit later:**
- Needs the HealthKit capability/entitlement and the `NSHealthShareUsageDescription` and `NSHealthUpdateUsageDescription` strings.
- **No 30-day limit.** All history is readable once authorized.
- Read authorization status is **always hidden**. A denied read simply returns empty data, so the UI must treat "no data" as possibly "not permitted".
- Sleep is `HKCategoryTypeIdentifier.sleepAnalysis`, with stages (asleepCore, asleepDeep, asleepREM, awake, inBed).
- For steps use `HKStatisticsCollectionQuery` (cumulative sum, which handles phone/watch dedup).
- Background delivery needs an observer query plus the background-delivery entitlement, and the completion handler must be called promptly.
- Keep the app's data layer neutral (`health_daily`) so both platforms map into it.

## 5. Pitfalls
- Double-counted steps from summing raw records instead of using aggregate.
- Sleep stored as "one session per day" when it can be several.
- Timezones: store UTC plus the local offset, and decide which "day" a late-night workout or sleep belongs to (end time, local date).
- Rounding errors from kg/lb conversion.
- An in-progress session lost when the process is killed.
- A rest-timer notification that doesn't fire in the background. Use a scheduled local notification, not a Dart timer.
- PR flags going stale after edits or deletes, or after sync.
- Exercise duplicates after re-seeding (needs a stable seed ID).
- Health Connect not installed or outdated on Android 13 and lower. Check `getSdkStatus` and deep-link to the install or update page.
- Requesting permissions that aren't used.
- Changes tokens expiring after 30 days.
- Dropping latest-edit-wins straight onto nested set lists.

## Suggested v1
**In v1:**
- Exercise library (free-exercise-db seed plus custom exercises).
- Routines/templates.
- Live workout logging: prefilled sets, a Previous column, one tap to complete a set, set types (warmup/working/drop/failure), optional RIR.
- Rest timer with a background notification.
- Crash-safe live session.
- Finish summary: volume, duration, PRs.
- PR detection (best weight, estimated 1RM using Epley, reps at weight).
- Exercise history and estimated-1RM chart.
- Body-weight log and trend line (7-day moving average).
- kg/lb setting.
- Health Connect import: daily steps (aggregate) and sleep sessions. Read on app open, last 30 days at first, idempotent upsert. Show it in the planner/dashboard.
- Planner integration: routines can be scheduled on days, show up as planner items, and are marked done when the session finishes.
- CSV export.

**Not in v1:**
- Nutrition.
- Programs that progress on their own (Boostcamp-style).
- AI or auto-generated workouts.
- Social features and sharing.
- Watch app.
- Supersets UI (keep the column, skip the UI).
- Plate calculator and warmup generator (v1.1).
- Writing workouts back to Health Connect (`ExerciseSessionRecord`).
- Background Health Connect sync and the history-beyond-30-days permission (add both together later).
- Heart rate, HRV, calories.
- Body-measurement types beyond weight (schema is ready, no UI).
- Exercise videos/GIFs.
- iOS HealthKit (comes with the iOS build).

## 6. Adherence psychology (why people quit, what keeps them)
Evidence grades: **[Strong]** means a meta-analysis or several RCTs. **[Moderate]** means a single good longitudinal study or a small meta-analysis. **[Weak]** means a lab or correlational study, or an extrapolation.
- **Dropout is front-loaded.** About 50% of people who start an exercise programme quit within 6 months (Dishman 1988, widely replicated). In trials, the average dropout is about 20%, and half of that happens in the first 6 months. **[Strong, but old and mostly descriptive.]** *App:* treat weeks 1-12 as a protected "foundation phase" with lower volume, no shame for misses, and more prompts.
- **Habits take ~2 months, with huge spread.** Lally 2010 found a median of 66 days to reach automaticity, with a range of 18-254 days. A 2024 systematic review found medians of 59-66 days and means of 106-154 days. Missing a single day did not derail habit formation. **[Moderate]** *App:* show "habit strength" and not a fragile streak. Message the user that "one miss doesn't reset you".
- **Gym habit threshold.** Kaushal & Rhodes 2015 found that about 4 sessions/week for 6 weeks, in a **consistent context** (same time and place), predicted habit formation. **[Moderate, single cohort.]** *App:* each routine gets a fixed planner slot ("Mon/Wed/Fri after Fajr" or "18:00 gym"), and the app nudges for consistent time over more volume.
- **Implementation intentions** ("When X, I will Y"). In the Bélanger-Gravel 2013 meta-analysis, d was about 0.31 post-intervention and 0.24 at follow-up. The effect was bigger when the plan included **barrier plans** ("if tired, do the 10-minute version"). **[Strong, small-medium effect.]** *App:* when a routine is scheduled, ask for "when/where" plus one "if obstacle, then..." line.
- **Progress monitoring works, especially when recorded.** Harkin 2016 (138 RCTs, about 20,000 people) found monitoring goal progress improves attainment, with larger effects when progress is **physically recorded** or **made public**. **[Strong]** Self-monitoring is among the behaviour-change techniques most linked to physical-activity self-efficacy (Ashford 2010; Olander 2013). **[Strong]** *App:* this is the core loop: log, then see the trend. Show weekly volume, an e1RM chart and a body-weight trend, plus a weekly review card.
- **Self-efficacy** is best raised by **mastery experiences**: feedback on past performance, plus achievable graded tasks (Ashford 2010). **[Strong]** *App:* PR celebrations, "you lifted X more than 8 weeks ago" comparisons, and progression suggestions small enough to hit.
- **Identity.** Exercise identity is one of the strongest correlates of physical activity: r = 0.44 across 32 studies (Rhodes 2016), and r = 0.35 across 49 studies (2025). In a cross-lagged study, identity predicted later activity, but activity did not predict later identity. **[Moderate; correlational, and the causal direction is still debated.]** *App:* the user picks an identity statement ("I am someone who trains 3x/week", or "a strong believer"), and each logged session counts as "evidence for" it. Use light, non-preachy copy.
- **Streaks cut both ways.** Silverman & Barasch (JCR 2023) found that showing streaks increases continuation, but a **broken** streak demotivates disproportionately. The drop is **attenuated if the user can repair the streak**. **[Moderate, consumer-behaviour experiments.]** *App:* use a weekly target ("3 of 4 this week") and not a daily streak. Include a "repair" option (make up a session), and apply "never miss twice" (folk wisdom, **[Weak]**).
- **Fresh-start effect.** Gym visits spike after temporal landmarks: new week, new month, birthday (Dai, Milkman & Riis 2014). **[Moderate]** *App:* offer a restart on Monday, the 1st of the month, the start of Ramadan, or after Eid. Hijri-month landmarks fit the app's identity.
- **Affect during exercise predicts return.** How pleasant the session feels **during** the session predicts future activity, but affect after the session does not (Rhodes & Kates 2015 systematic review). **[Moderate]** *App:* keep beginner sessions sub-maximal (RIR 2-3), ask an optional 1-5 "how did it feel" rating, and suggest lighter sessions if ratings trend low.
- **Minimum viable workout and exercise snacks.** Short, frequent bouts improve fitness and strength in inactive adults. One review reported fitness gains with 4.5-67.5 minutes/week, and adherence was about 83%. Minimal-dose resistance training (1 set to near-failure, 1-2x/week) still builds strength (2024 overview). **[Moderate]** *App:* every routine has a "minimum version" (e.g. 1 set of each of the main lifts, or 10 minutes). Logging that counts as a full "show-up" for the weekly target.
- **Social accountability.** Carron 1996 meta-analysis (87 studies) found that social influence improves adherence, and task-cohesive groups beat standard classes. Partner support is a mediator in RCTs. **[Strong for groups, Moderate for partners.]** *App:* single user, so no social feed. Optionally let one accountability partner receive a weekly summary (share sheet or link). Defer to a later version.
- **Avoid:** guilt notifications, points/badges disconnected from the behaviour, and high-intensity starts. These are design guidance more than tested findings. **[Weak]**

## 7. Traditional and historical training systems: templates for the app
Note on evidence: almost none of these traditions have been studied directly as whole systems. Where they map onto a modern category (bodyweight resistance, club swinging, loaded walking, qigong), the evidence for that category is cited.

**Persian zurkhaneh (varzesh-e bastani / pahlavani)**
- *What it was:* a domed "house of strength" with a sunken pit (gowd). Training ran in a fixed sequence: warm-up calisthenics, **shena** (push-up variations, done to a drum rhythm), **sang** (heavy wooden shields pressed while lying on the back), **meel** (60-80 cm heavy wooden clubs swung around the head and shoulders), **charkh** (whirling), then wrestling. It combined spiritual verse (the morshed chants Ferdowsi and praise of Imam Ali) with ethics (javanmardi, chivalry). It is UNESCO-listed.
- *Evidence:* club swinging gave a 35% acute increase in shoulder flexibility versus control in a small study **[Weak]**. High-rep push-ups count as standard bodyweight resistance training **[Strong, for the category]**.
- *Feature:* a built-in **"Zurkhaneh" routine**: shena (tempo push-ups, logged as reps), meel (logged as time plus club weight), sang press (dumbbell or plate floor-press substitute), charkh (logged as time). Add a **rhythm/metronome cue** in place of the drum, and a cadence field on the set.

**Indian pehlwani / kushti**
- *What it was:* wrestlers trained in an earthen akhara. **Dand** (Hindu push-up: dive forward, then arch up) and **baithak** (Hindu squat: heels up, arms swinging) were done in the hundreds to thousands. **Gada/mugdar** (mace or club) was swung for shoulders and grip. Other work included rope climbing and **joris** (paired clubs), with a strict diet and sleep discipline. Gama's reported volumes (several thousand dand and baithak a day) are legend-level and partly unverifiable.
- *Evidence:* high-rep, low-load sets taken close to failure produce hypertrophy comparable to heavy loading (Schoenfeld 2017 meta-analysis) **[Strong]**. Mace and club swinging have no direct studies **[Weak]**. Very high-rep deep squats with heels up carry a risk of knee and tendon overuse **[Weak, expert opinion]**.
- *Feature:* a **"Pehlwani" routine**: dand and baithak in rep-ladder sets (e.g. 5x20, progressing by +5 reps per session), plus gada swings (logged as time and reps, with mace weight). Treat it as a **volume-progression** template with **total daily reps** as the PR metric, and give it a 12-week ramp, not Gama numbers.

**Greek palaestra / pankration**
- *What it was:* the gymnasion and palaestra were civic training spaces. Training included wrestling, boxing, pankration (all-in fighting), running, and jumping with **halteres** (2-9 kg stone or lead weights). Philostratus describes the **tetrad**, a 4-day cycle: preparation, hard ("toil") day, rest, moderate day. Milo of Croton carrying a growing calf is the founding myth of **progressive overload**.
- *Evidence:* the tetrad maps onto undulating periodization or a hard/easy split. Daily undulating periodization is roughly equivalent to linear periodization for strength in meta-analyses **[Moderate]**.
- *Feature:* a **"Tetrad" schedule template** that drops a 4-day prep/hard/rest/moderate cycle into the planner. The rotation is not tied to the 7-day week, and the planner must support N-day cycles. Add a **"Milo" progression rule**, a small fixed increment each session, as the app's progression engine name.

**Roman legion**
- *What it was:* in Vegetius' *De Re Militari*, recruits marched **20 Roman miles (~29.6 km) in 5 summer hours** at the regular step, and 24 Roman miles at the full step. They carried a load of about **60 Roman lb (~20 kg)**, and loaded marching was taught **before weapons**. Training also included swimming, running, jumping, and post (palus) drills with double-weight wooden weapons.
- *Evidence:* loaded walking (rucking) raises cardiorespiratory demand at the same speed **[Moderate]**. Steps and mortality: benefit plateaus around 6-8k steps/day for people over 60, and 8-10k for younger adults (Paluch 2022, 15 cohorts) **[Strong]**. Japanese interval walking (3 minutes fast, 3 minutes slow) improves fitness and strength versus continuous walking (Nose 2007; Masuki 2020) **[Moderate]**.
- *Feature:* a **"Legion march" activity type** (distance, duration, pack load in kg) with a progressive ruck plan: start at 10% of body weight and 5 km. Health Connect **steps feed a "march" goal**, e.g. a weekly target of 20 Roman miles.

**Shaolin and Chinese traditions**
- *What it was:* **Yijinjing** ("tendon-changing") and **Baduanjin** ("eight brocades") qigong, **ma bu** (horse-stance holds), and stone-lock lifting (shi suo), with daily, very consistent practice.
- *Evidence:* Baduanjin and Tai Chi meta-analyses (2025) show better balance, fewer falls, better grip strength and gait speed, and better sleep and quality of life, with no clear change in muscle mass **[Strong, mostly older adults]**.
- *Feature:* a **"Baduanjin" daily mobility routine**: 8 timed movements of about 12 minutes, logged as duration. It is ideal as the **minimum-viable day** or rest-day option. Add a **horse-stance hold** as an isometric exercise with time-based PRs.

**Prophetic-era (Sunnah) activities**
Hadith gradings vary; the user should verify against scholars and sunnah.com.
- **"The strong believer is better and more beloved to Allah than the weak believer, and in both is good"** (Sahih Muslim 2664). This is the anchor for an identity statement.
- **Archery.** Authentic hadith encourage it, e.g. "Whoever learns archery then abandons it..." (Muslim 1919).
- **Horse racing.** The Prophet held races (Bukhari).
- **Running.** He raced Aisha on foot (Abu Dawud 2578; graded sahih by al-Albani).
- **Wrestling.** He wrestled Rukana (Abu Dawud/Tirmidhi; the chain has weakness, and it is strengthened by other reports).
- **"Teach your children swimming, archery and horse riding"** is authentically **from Umar**, in a letter to Abu Ubaidah. Its attribution to the Prophet is weak.
- **Walking.** He walked briskly, "as if descending a slope" (Shama'il).
- *Feature:* **"Sunnah sports" activity types**: archery (arrows, distance, score), swimming (distance, time), riding (duration), running/racing, wrestling/grappling (rounds × minutes). Also a **"Brisk walk after Salah"** micro-habit tied to prayer times. Prayer times are natural **implementation-intention anchors** ("after Asr, walk 20 minutes") and the strongest consistent-context cue available to this user **[design inference]**.

**Fasting patterns and training**
- *Tradition:* fasting on Monday and Thursday (Tirmidhi 747: deeds are presented on those days), the White Days (13th-15th of the Hijri month), and Ramadan.
- *Evidence:*
  - A 2025-26 multilevel meta-analysis found that intermittent fasting plus resistance training gives strength and hypertrophy **similar** to resistance training alone **[Moderate-Strong]**.
  - Ramadan trials show strength is maintained; training **in a fed state (after iftar)** may be slightly better for performance, but hypertrophy and strength are not harmed either way **[Moderate]**.
  - Trained martial artists kept maximal strength and power during Ramadan (2026) **[Moderate]**.
  - A Monday-Thursday fasting trial (NCT07703865) studied metabolic outcomes, not training **[no training evidence]**.
- *Feature:*
  - A planner **"fasting day" flag** for Mon/Thu, the White Days and Ramadan, using a Hijri calendar.
  - On those days, suggest moving heavy sessions to **just before Maghrib or after iftar**, cap the volume, and switch to the minimum version.
  - A **Ramadan mode**: reschedule all routines around iftar/taraweeh, use a maintenance-volume template, and show a "maintain, not PR" message.
  - Fasting is nutrition-adjacent. Keep it a **schedule flag only** (no food logging), which stays inside the agreed scope.

**Cross-cutting design idea:** treat each tradition as a **"lineage pack"**: routines, custom exercise types, a short history card, and a progression rule. Packs are shipped as seed data, which free-exercise-db lacks (it has no dand, meel or Baduanjin), so the app writes and licenses these entries itself. The lineage view could double as identity reinforcement ("you trained in the Pehlwani tradition 14 times this season").

**Additions to the v1 list:**
- Weekly target with repair, instead of streaks.
- A "minimum version" per routine.
- A when/where + if-then prompt when scheduling.
- Fixed planner slots, including prayer-anchored ones.
- Time-based and cadence sets (needed by the traditional routines).
- Two lineage packs to start (Pehlwani bodyweight and Baduanjin mobility).
- Fasting-day flag.

**Later:**
- Remaining packs (Zurkhaneh, Tetrad cycle, Legion ruck, Sunnah sports).
- Ramadan mode.
- Accountability partner summary.
- Hijri fresh-start prompts.

## Sources
- Health Connect read data (30-day rule, background reads, pagination): https://developer.android.com/health-and-fitness/health-connect/read-data
- Sync data / changes tokens: https://developer.android.com/health-and-fitness/health-connect/sync-data
- Aggregate data and dedup: https://developer.android.com/health-and-fitness/health-connect/aggregate-data
- Sleep experiences: https://developer.android.com/health-and-fitness/health-connect/experiences/sleep
- Get started (manifest, rationale activity): https://developer.android.com/health-and-fitness/health-connect/get-started
- Publish on Play (declaration form): https://developer.android.com/health-and-fitness/health-connect/publish
- Play health permissions FAQ: https://support.google.com/googleplay/android-developer/answer/12991134
- Flutter `health` package: https://pub.dev/packages/health ; `health_connector`: https://pub.dev/packages/health_connector
- Android Authority on the history and background permissions: https://www.androidauthority.com/health-connect-historical-background-reads-3443726/
- free-exercise-db (Unlicense): https://github.com/yuhonas/free-exercise-db
- wger (AGPL code, CC-BY-SA data): https://github.com/wger-project/wger
- ExerciseDB license FAQ: https://exercisedb.io/faq ; repo: https://github.com/exercisedb/exercisedb-api
- Strong vs Hevy: https://repreturn.com/strong-app-vs-hevy/ ; https://prpath.app/blog/strong-vs-hevy-2026.html ; https://www.sensai.fit/blog/hevy-review-2026
- HealthKit hidden read authorization and background delivery: https://3nsofts.com/insights/healthkit-architecture-production-ios-apps
- Dropout and adherence (STRRIDE): https://doi.org/10.1249/TJX.0000000000000190 ; novice gym one-year follow-up: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8194699/
- Habit formation review (2024): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11641623/ ; Lally 2010: https://www.researchgate.net/publication/32898894
- Kaushal & Rhodes 2015: https://www.uvic.ca/research/labs/bmed/assets/docs/Kaushal,%20Rhodes,%202015.pdf
- Implementation intentions and physical activity (meta-analysis): https://www.researchgate.net/publication/233472164 ; https://journals.plos.org/plosone/article?id=10.1371%2Fjournal.pone.0206294
- Harkin 2016 progress monitoring: https://eprints.whiterose.ac.uk/id/eprint/87431/
- Ashford 2010 self-efficacy: https://bpspsychub.onlinelibrary.wiley.com/doi/10.1348/135910709X461752 ; Olander 2013: https://ijbnpa.biomedcentral.com/articles/10.1186/1479-5868-10-29
- Exercise identity (Rhodes 2016): https://www.uvic.ca/research/labs/bmed/assets/docs/Rhodes,%20Kaushal%20and%20Quinlan,%202016.pdf ; 2025 meta-analysis: https://www.tandfonline.com/doi/full/10.1080/1750984X.2025.2509218 ; cross-lagged: https://www.sciencedirect.com/science/article/abs/pii/S1469029224000529
- Broken streaks (Silverman & Barasch): https://www.researchgate.net/publication/361658509 ; https://www.colorado.edu/business/news/2023/04/20/research-streaks-marketing-tech-barasch
- Fresh-start effect: https://www.strategy-business.com/article/00266
- Affect predicts future exercise (Rhodes & Kates 2015): https://www.researchgate.net/publication/275662348
- Exercise snacks: https://bmjgroup.com/exercise-snacks-may-boost-cardiorespiratory-fitness-of-physically-inactive-adults/ ; minimal-dose resistance training: https://pmc.ncbi.nlm.nih.gov/articles/PMC11127831/
- Carron 1996 group cohesion: https://www.researchgate.net/publication/12417414
- Zurkhaneh: https://www.traditionalsports.org/images/sports/asia/zurkhaneh/Zoorkhaneh_and_Varzesh-E-Bastani.pdf ; club swinging and flexibility: https://digitalcommons.wku.edu/cgi/viewcontent.cgi?article=2436&context=ijesab
- Pehlwani: https://en.wikipedia.org/wiki/Hindu_squat ; https://en.wikipedia.org/wiki/Hindu_push-up ; Gama record: https://tagdaraho.us/blogs/news/gama-training-numbers
- Greek tetrad and halteres: https://www.researchgate.net/publication/332757496 ; https://en.wikipedia.org/wiki/Halteres_(ancient_Greece)
- Vegetius: https://www.roman-britain.co.uk/classical-references/vegetius/ ; https://en.wikipedia.org/wiki/Loaded_march
- Steps and mortality (Paluch 2022): https://www.sciencedaily.com/releases/2022/03/220303112207.htm ; interval walking: https://cdnsciencepub.com/doi/10.1139/apnm-2023-0595
- Baduanjin and Tai Chi: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12440739/ ; https://pubmed.ncbi.nlm.nih.gov/40177095/ ; https://peerj.com/articles/18512/
- Hadith: strong believer: https://sunnah.com/search?q=strong+believer+is+better ; "teach your children" (from Umar): https://islamqa.org/hanafi/hadithanswers/119604/ ; Rukana: https://sunnah.com/search?q=Rukanah ; race with Aisha: https://sunnah.com/search?q=race+aisha
- Fasting plus resistance training meta-analysis: https://pmc.ncbi.nlm.nih.gov/articles/PMC13341873/ ; Ramadan training timing: https://pubmed.ncbi.nlm.nih.gov/37068775/ ; Ramadan martial artists 2026: https://doi.org/10.1177/17479541261431033 ; Monday-Thursday trial: https://clinicaltrials.gov/study/NCT07703865
