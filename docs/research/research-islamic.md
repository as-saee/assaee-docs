# As-Sa'ee: Islamic scholar module, research on sources and libraries (Oct 2026)

Key licensing rule for a PUBLIC repo: the code can be public, but most translations, tafsir, hadith
translations and audio are copyrighted. Commit fetch/import scripts and checksums, not the datasets.
The app downloads data at install or first run into local storage (Flutter) and Postgres. That counts
as personal use, not redistribution. Exceptions you can commit: Tanzil Arabic text (verbatim, with its
notice), and MIT/Unlicense code libraries.

## 1. Quran text, translations, tafsir, audio
- **Tanzil Arabic text**: "Creative Commons Attribution 3.0", but "copy and distribute verbatim copies
  ... CHANGING IT IS NOT ALLOWED". You must name Tanzil, link to tanzil.net, and keep the copyright
  notice in every file that holds it. In practice it works like a no-derivatives license: ship the file
  unmodified, and keep any normalized search index (no diacritics) as a separate derived file that
  also carries the notice. Variants: Uthmani, Simple, and others. https://tanzil.net/docs/text_license
- **Tanzil translations**: "for non-commercial purposes only", copyright stays with each translator, and
  you must link back if you use more than three. Fine for display, but don't commit them to the public
  repo. https://tanzil.net/trans/
- **Quran Foundation (quran.com) API v4**: OAuth2 client_credentials with scope `content`. You create an
  app in the Developer Console (https://dev-console.quran.foundation). New apps start in **pre-live**,
  which only has Surahs 1-2, and **production needs approval**. Headers: `x-auth-token` and
  `x-client-id`. Tokens last 1 h. The client_secret must stay on the server, so it goes through
  FastAPI, never into Flutter. Terms: **no caching of QF content beyond 1 week** unless it goes through
  their Content Sync APIs (re-sync at least every 7 days). No redistribution as a dataset. No training
  ML models without written consent. RAG inside the app is allowed, but the storage limits still apply.
  Attribution text: "Quran data provided by Quran Foundation", plus a credit for each edition. That
  makes it a poor fit as the offline base layer and fine as an optional online add-on.
  https://api-docs.quran.foundation/docs/quickstart/ ,
  https://api-docs.quran.foundation/legal/developer-terms/
- **fawazahmed0/quran-api**: the code is Unlicense, with 440+ translations on the jsDelivr CDN. The
  Unlicense only covers the repo's own work, not the translators' copyrights, so treat it as a
  convenient mirror, not a license grant. https://github.com/fawazahmed0/quran-api
- **QuranEnc.com (best licensing for translations)**: published terms allow downloading and
  re-publishing, including in apps and commercially, if you: make no changes; credit QuranEnc and the
  publisher; show the version number; keep the info inside the file; update to the latest version; and
  show no inappropriate ads. SQLite/JSON downloads plus an API (`/api/v1/translations/list/{lang}`).
  Checked live today:
  - EN: `english_saheeh` (Saheeh International, via Noor International, v1.1.2), `english_rwwad`
    (Rowwad, v1.0.19), `english_hilali_khan`.
  - ES: `spanish_garcia` (**Isa García**, v1.0.2), `spanish_montada_eu` (Noor International, Spain
    Spanish), `spanish_montada_latin`.
  https://quranenc.com/en/home/api/
- **Copyright status of specific translations**:
  - **Clear Quran (Khattab)**: © Al-Furqaan Foundation, all rights reserved, trademarked. Use it only
    through the QF API under QF terms. Never bundle it.
  - **Isa García**: printed editions say "all rights reserved ... free distribution". QuranEnc's terms
    are the cleanest route.
  - **Sahih International**: copyrighted (Abul-Qasim / Al-Muntada). Use the QuranEnc edition.
  - **Julio Cortés and Raúl González Bórnez** (both on Tanzil): copyrighted.
- **Tafsir**:
  - Classical Arabic tafsir (Ibn Kathir, al-Sa'di, al-Tabari) is public-domain text. Digital editions
    may still claim rights.
  - **English Ibn Kathir** is the abridged Darussalam translation (Mubarakpuri et al.), copyrighted.
  - `spa5k/tafsir_api` is MIT for code, with 122 tafsirs and corrections to the English Ibn Kathir.
    The MIT license does not cover the tafsir texts.
  - Advice: fetch tafsir at runtime or install, show it read-only, and don't commit it.
  https://github.com/spa5k/tafsir_api
- **Audio**:
  - EveryAyah: per-ayah MP3s by reciter (good for hifz loops and ayah-by-ayah repeat). Reported as
    CC-BY-NC. Timing files need a link back. Free direct-link hosting. Fine for personal use and local
    caching, but don't re-host or commit MP3s. https://everyayah.com/data/timings_files/000_disclaimer.txt
  - QF audio API: has word and ayah timings (useful for highlighting), but QF terms and the 7-day cache
    limit apply.

## 2. Hadith
- **sunnah.com API**: you need an API key, requested by opening a GitHub issue on `sunnah-com/api` (say
  the project, use case, expected volume, and how you will store the key). There is a large backlog of
  requests, and the offline dump is still "not available yet". Docs:
  https://sunnah.stoplight.io/docs/api/ , https://github.com/sunnah-com/api/issues ,
  https://sunnah.com/developers . The About page has a "Reproduction, Copying, Scraping" section (it
  returned 403 when fetched, so read it in a browser before relying on it).
- **fawazahmed0/hadith-api**: Unlicense, jsDelivr CDN, multiple languages. Grades come as a list of
  `{grader, grade}` (for example al-Albani, Zubair Ali Zai), with an `/info` endpoint for references.
  Best practical source. The same caveat applies: the English translations (mostly Darussalam / Muhsin
  Khan) remain copyrighted. https://github.com/fawazahmed0/hadith-api
- **Other datasets**: AhmedBaset/hadith-json, Hadith-JSON-Engine (about 50.9k hadith across 17
  collections, with named graders), and HF datasets such as `meeAtif/hadith_datasets` and
  `gurgutan/sunnah_ar_en_dataset`. Most of these are **scrapes of sunnah.com relabelled MIT**, which
  amounts to license laundering. Use them for local experiments only and don't commit them.
  https://github.com/TheAbubakrAbu/Hadith-JSON-Engine
- **Grading model**: store each grade as (grader, grade, source). Never collapse them into one "sahih"
  flag. Show the grader's name. Bukhari and Muslim have no per-hadith grades; their status comes from
  the collection.
- **Spanish hadith**: thin coverage. Plan on Arabic + English with no Spanish hadith in v1.

## 3. Prayer times and qibla
- **Dart `adhan_dart`** (farend.net): v2.0.1, May 2026, MIT, no runtime dependencies.
  - Methods: MWL, Egyptian, Karachi, Umm al-Qura, Dubai, Qatar, Kuwait, Moonsighting Committee,
    **Morocco**, Russia, Singapore, Tunisia, Türkiye, Tehran, North America, plus fully custom angles.
  - Madhab option: Shafi (1x shadow, also used by Maliki and Hanbali) or Hanafi (2x shadow).
  - Also includes: high-latitude rules, a **Qibla** function, middle and last third of the night, and
    per-prayer minute adjustments.
  - Use this one. The older `adhan` package on pub is the alternative.
  https://pub.dev/packages/adhan_dart
- **Python `adhanpy`**: MIT, port of adhan-java, last release **1.0.5 in Jan 2023**, so stale but stable
  (pure math). Officially supports Python 3.9-3.12. Pin it, or vendor about 500 lines.
  - Better plan: compute times on the device only, and have the backend reuse the same parameters only
    if it ever needs times (reminders).
  - Write a cross-check test: Dart vs Python for the same date, coordinates and method, within ±1 min.
  https://pypi.org/project/adhanpy/
- **Spain**:
  - No single national method found from the Comisión Islámica de España or UCIDE.
  - Spanish timetable sites mostly default to **MWL (Fajr 18°, Isha 17°)**.
  - Moroccan-majority communities often follow Morocco Habous practice (**Fajr 19°, Isha 17°**). I could
    not verify this for specific Spanish mosques.
  - Large mosques publish their own yearly or monthly timetables, for example the Centro Cultural
    Islámico de Madrid (M-30) at https://centro-islamico.com/mezquita/ .
  - **Recommendation**: default to MWL, keep Morocco as a one-tap option, and add per-prayer minute
    offsets with a "match my mosque" helper. The user compares against the local mosque timetable for a
    week and sets the offsets.
  - Asr: most Spanish Muslims are Maliki (Moroccan), which uses the 1x shadow ("Shafi"/standard
    setting). Make Hanafi selectable.
  - High latitude: not needed for Spain (36-43.8°N, twilight angles always reached). Set a rule anyway
    for travel.
  - Spain uses CET/CEST, so let tz handle the DST switches.
- **Qibla**: great-circle initial bearing to the Kaaba (21.4225, 39.8262). `adhan_dart` includes it.
  Madrid is about 100° (ESE). Compass UI: `flutter_compass` or `sensors_plus` magnetometer plus
  magnetic declination (about 0-1° in Spain, so negligible), and a calibration hint.

## 4. Adhkar
- **Seen-Arabic/Morning-And-Evening-Adhkar-DB**: MIT. AR + EN in JSON, CSV and SQLite. Each item has the
  text, translation, transliteration, repeat count, virtue (fadl), source reference, morning/evening
  flag, and audio. Sources: Hisn al-Muslim, al-Muqaddam, hisnmuslim.com, sunnah.com. Only 19 commits,
  so check each item. https://github.com/Seen-Arabic/Morning-And-Evening-Adhkar-DB
- **Hisn al-Muslim** (al-Qahtani): the compilation and its translations are copyrighted. Many editions
  say "free distribution", which is not an open license.
- The Arabic wording of the adhkar is itself Quran or hadith text (public domain). For a public repo:
  - Commit only a small curated list of IDs + counts + hadith references (for example
    "Muslim 2723", "Bukhari 6306").
  - Pull the Arabic text from your hadith or Quran store at import time.
  - Write English and Spanish glosses yourself, or link to the source.
  - This also meets the "never generated" rule: every dhikr traces to a cited hadith.

## 5. Hifz tracking: methods, apps, algorithms
- **Sabaq / sabqi / manzil** (South Asian madrasa practice):
  - sabaq: new lesson.
  - sabqi: about the last 7-30 days, revised daily.
  - manzil: everything older, rotated (for example 1/7 of the Quran a week, or 1 juz a day).
  - It is effectively hand-tuned spaced repetition with expanding intervals.
- **FSRS**:
  - `fsrs` on pub.dev (open-spaced-repetition/dart-fsrs, ported from py-fsrs).
  - `fsrs` on PyPI (py-fsrs, official).
  - Both expose Card, ReviewLog and Scheduler. Desired retention is about 0.90-0.95 for Quran.
  https://github.com/open-spaced-repetition/dart-fsrs , https://pypi.org/project/fsrs/
- **Unit choice matters**: schedule per page (Madani 604-page mushaf) or per half-page, not per ayah. A
  card per ayah gives about 6.2k cards, which is too granular and loses the linking between ayahs.
- Track **mutashabihat** (similar verses) as extra linked cards.
- **Apps to learn from**:
  - Hifz (OmarHalaseh/Hifz): offline-first, page-by-page, daily plan across sabaq/sabqi/manzil with an
    SRS. https://github.com/OmarHalaseh/Hifz
  - hifz-trainer: SM-2 or FSRS, desired retention adjustable from 70-98%.
    https://github.com/Mohammed-Abdelaziem/hifz-trainer
  - biraye: spaced review, scaffolded recall, mutashabihat drills.
  - Sadr, QuranH and Hafiz Quran: FSRS + sabaq/sabqi/manzil.
  - Tarteel: speech recognition for mistake detection (closed source, a model for later).
  - Quran Companion: gamified streaks.
- **Hybrid design**: FSRS drives sabqi and manzil due-dates. The sabaq quota is a fixed daily setting
  (lines or pages). A daily review cap stops the backlog snowballing after missed days.

## 6. Learning psychology and classical Muslim learning methods
Format: what it is / evidence / possible app feature.

- **Spacing effect** (Cepeda et al. 2006 meta-analysis: 839 assessments, 317 experiments).
  - Evidence: spreading study over time beats massed study. The best gap grows as the retention
    interval grows.
  - Feature: FSRS sabqi/manzil scheduling, and a short daily session rather than a long weekly one.
  https://augmentingcognition.com/assets/Cepeda2006.pdf
- **Retrieval practice / testing effect** (Roediger & Karpicke 2006).
  - Evidence: one week later, testing gave about 61% recall vs 40% for re-reading. Re-reading only
    wins at 5 minutes.
  - Feature: recite from memory first, then reveal and grade. Example modes: first-word cue, hidden
    page, "next ayah?". Don't make re-reading the default.
  https://journals.sagepub.com/doi/10.1111/j.1467-9280.2006.01693.x
- **Interleaving** (Brunmair & Richter 2019 meta-analysis, g=0.42).
  - Evidence: it helps mostly with **similar, confusable** items. It hurts or is neutral for dissimilar
    text and words.
  - Feature: interleave only for mutashabihat drills (contrast similar verses side by side). Keep
    normal manzil in blocks, in mushaf order.
  https://www.researchgate.net/publication/335004545
- **Sleep consolidation**.
  - Evidence: hippocampal replay during slow-wave sleep strengthens declarative memory, according to
    several reviews. One 2022 review argues the effect is overrated.
  - Feature: a sabaq review prompt after Isha or before sleep, and a re-test after Fajr the next
    morning. That lines up the traditional Fajr hifz slot with consolidation.
  https://www.cell.com/neuron/fulltext/S0896-6273(23)00201-5
- **Small but consistent**. The hadith: "the most beloved deed to Allah is the most regular and
  constant even if it were little" (Bukhari 6464; quote it from your hadith store).
  - Evidence: Lally et al. 2010 found a median of 66 days to automaticity (range 18-254), and that
    **missing a single day did not materially derail habit formation**.
  - Feature: a minimum-viable daily wird (for example 1 page revision + morning adhkar). Track
    consistency rather than "streak-or-nothing", and use a forgiving streak (one missed day doesn't
    reset it). Anchor each habit to a prayer time ("after Fajr"), which is habit-cue stacking.
  https://www.researchgate.net/publication/32898894
- **Kuttab/maktab**: the elementary Quran school.
  - Method: children learn letters, then recite and write short surahs. Chorus recitation, daily
    recitation to the teacher, and graded progress (often from Juz 'Amma upwards).
  - Feature: a "progression path" starting from the short surahs, plus a session where you recite
    aloud to yourself or a recording.
- **Lawh** (wooden slate, Morocco, Mauritania, West Africa; Mauritanian mahdara).
  - Method: the student writes the day's portion on a slate by hand (often dictated), repeats it
    hundreds of times (300-500 is reported, counted on beads), recites it to the shaykh, then washes
    the slate and writes the next portion.
  - Evidence: writing adds generation and encoding effects, and the volume of repetition is very high.
    Claims of "nearly indestructible" memory are anecdotal.
  - Feature: a "write it out" mode (handwriting or typing the ayah from memory with error check),
    a repetition counter (tasbih-style tap, target N), and a daily slate that clears after a
    successful recital.
  https://en.wikipedia.org/wiki/Qur%27an_Boards ,
  https://www.researchgate.net/publication/389640627
- **Sabaq/sabqi/manzil**: see section 5. The feature is a three-lane daily plan.
- **'Ard / recitation to a teacher, ijazah and sanad**.
  - Method: the Quran was passed on orally by presentation ('ard) and listening. An ijazah is a
    teacher's written authorization after a full, precise recitation, with the sanad (chain of
    teachers) going back to the Prophet.
  - Feature: you can't replicate an ijazah, and the app must not pretend to grant one. Instead: a
    "teacher log" (date, portion recited, teacher's notes on mistakes), a record-yourself-and-send
    export, and later optional ASR mistake hints (Tarteel-style) clearly marked as non-authoritative.
  https://merathalquran.com/en/blog/ijazah-and-sanad/
- **al-Zarnuji, *Ta'lim al-Muta'allim*** (12th-13th c.).
  - Method:
    - Start with a lesson small enough to master in about 2 repetitions, and grow it gradually.
    - Repeat yesterday's lesson 5x, the day before 4x, the one before that 3x (a decaying review
      ladder, reported via secondary sources).
    - Best times are pre-dawn (sahar) and between Maghrib and Isha.
  - Evidence: this is a proto-spacing schedule.
  - Feature: a "Zarnuji ladder" preset as an alternative to FSRS for new material, and a sabaq size
    auto-tuned so it takes about 2 clean recalls.
  https://en.wikipedia.org/wiki/Al-Zarnuji , https://www.researchgate.net/publication/321047279
- **Ibn Jama'ah, *Tadhkirat al-Sami' wa'l-Mutakallim*** (14th c.).
  - Method: adab of the student (sincere intention, respect for the teacher, choosing the time, steady
    progress, revising with peers).
  - Feature: an intention (niyyah) prompt at the start of a session, and short adab reminders from
    the text (quoted with citation).
  - I could not verify its specific memorization-time advice online.
  https://www.researchgate.net/publication/399035945
- **Wird / awrad**: a fixed daily portion of Quran or dhikr, kept steadily.
  - Feature: user-defined wird items tied to prayer anchors, with a daily checklist.
- **Al-Ghazali, *Ihya* Book 38: musharata, muraqaba, muhasaba** (al-Muhasibi before him).
  - Method: set the conditions with yourself in the morning, stay vigilant through the day, and do an
    evening self-accounting. That is followed by renewed striving, not self-punishment.
  - Feature: a morning intention plus a 2-minute evening muhasaba journal (what was kept or missed,
    one adjustment for tomorrow). Keep it private and local, and never score it.
  https://en.wikipedia.org/wiki/Muhasaba ,
  https://www.cambridgemuslimcollege.ac.uk/wp-content/uploads/2023/04/Tools-for-Transformation.pdf
- **Dhikr / muraqaba** (Sufi practice).
  - Method: counted dhikr and contemplative sitting. These are tariqa-specific practices.
  - Feature: a general tasbih counter and a quiet timer only. Prescribed tariqa awrad belong to the
    user's own teacher, so the app doesn't supply them.
- **Fajr memorization**: a widespread traditional practice. The rested brain after sleep fits it, but
  there is no direct study.
  - Feature: the default sabaq slot is after Fajr, with the time taken from the prayer-times engine.

## 7. Pitfalls
1. Committing copyrighted translations, hadith English, tafsir, adhkar books or MP3s to the public repo.
   Commit the import scripts and the expected SHA-256 instead.
2. Changing the Tanzil text (normalizing, stripping tashkeel in place, "fixing" glyphs). This breaks
   the license and the integrity rule. Store it verbatim and derive the search fields.
3. Mixing Quran scripts (Uthmani vs simple vs IndoPak) with mushaf page maps. Hifz pages need the
   Madani 604-page layout to match the text edition.
4. Putting the QF client_secret in the Flutter app, or keeping QF content offline for more than 7 days
   without Content Sync.
5. Treating "MIT"-labelled sunnah.com scrapes as licensed.
6. Showing one grade as a verdict instead of grader-attributed grades.
7. Hadith numbering differs between editions (sunnah.com, Darussalam, Fuad Abd al-Baqi). Store the
   numbering scheme alongside each reference in citations.
8. Prayer-time mismatch with the local mosque. Offsets are needed, and it is the main user complaint
   in Spain.
9. `adhanpy` being stale. Pin the version, and run the Dart-vs-Python test.
10. The AI phase:
    - Retrieve only from the stored Quran/hadith/tafsir records.
    - Show exact references.
    - Refuse fatwa-style rulings and point to a scholar.
    - QF content can't be used for model training, though RAG inside the app is allowed.

## 8. Suggested v1 sources
| Need | Source | Ships in repo? |
|---|---|---|
| Arabic Quran | Tanzil Uthmani (verbatim + notice) | Yes |
| EN translation | QuranEnc `english_saheeh` (or `english_rwwad`) | No, import script |
| ES translation | QuranEnc `spanish_garcia` (+ `spanish_montada_eu`) | No, import script |
| Tafsir (v1.5) | spa5k tafsir_api: Arabic Ibn Kathir / al-Sa'di; EN Ibn Kathir display-only | No |
| Audio | EveryAyah per-ayah MP3 (Alafasy / Husary muallim for hifz), stream + local cache | No |
| Prayer/qibla | `adhan_dart` 2.x on device; MWL default, Morocco option, offsets; Maliki=standard Asr | Code |
| Adhkar | Curated ID list + hadith refs; text from hadith store; Seen-Arabic DB as checklist | IDs only |
| Hifz | `fsrs` (Dart) per page + sabaq/sabqi/manzil lanes; py-fsrs on server if needed | Code |
| Hadith (v2) | fawazahmed0 hadith-api import (Arabic + EN + grades); request a sunnah.com key now | No |
| QF API | Optional online extras (Clear Quran, word timings), via FastAPI proxy, 7-day cache | No |

Action now: open the sunnah.com API key issue and create the QF Developer Console app. Both have
approval lead time.
