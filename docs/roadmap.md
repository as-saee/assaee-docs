# Roadmap — As-Sa'ee (السعي, "the striving")

Where the project is going and why. Decisions are recorded in [`adr/`](adr/); the research behind each module is in
[`research/`](research/). Last full revision 2026-10-06; heartbeat decisions added 2026-10-09.

**Priority: career first.** The DevOps/cloud platform (M0–M4, M6) is the main focus. Each app module gets its own
intensive research round *before* it is built; the module sections below are v1 sketches, not specs.

## Progress
- [ ] **M0 — Setup**
  - [x] Author: create GitHub org `as-saee` (done 2026-10-09; Free plan; display name "As-Saee", GitHub URLs are case-insensitive)
  - [ ] Author: register `as-saee.com` (Cloudflare Registrar; ~$10/yr, only needed before `api.as-saee.com` goes live)
  - [x] Author: Oracle Cloud account (free-only, Madrid home region; done 2026-10-09; home-region check pending)
  - [ ] Author: other accounts, each when its milestone needs it: Cloudflare (before the domain), Discord (end of M1), Tailscale (M4), AWS + billing alarms $1/$5 (M6)
  - [ ] Author: before M9, request a sunnah.com API key (GitHub issue on `sunnah-com/api`) and create a Quran Foundation developer app (both take approval time)
  - [x] Commit the Flutter baseline in this repo (staged files + the four files with unstaged edits)
  - [x] Create the six empty public repos in the org (done 2026-10-09: assaee-api/infra/gitops/docs/mobile, readflow-web)
  - [ ] Split repos: the original Flutter app and React reader move into `assaee-mobile` and `readflow-web` with history kept; `readflow-web` is archived
  - [ ] `assaee-docs`: roadmap, research and ADR-001 (this repo)
- [ ] **M1 — Walking skeleton** (study: Terraform Associate)
  - [ ] Terraform: Oracle network + ARM VM(s), Cloudflare DNS; state in Oracle object storage with locking
  - [ ] Ansible: OS hardening + k3s install
  - [ ] `assaee-api`: FastAPI with `/health` + the **heartbeat** feature (see "Heartbeat" below), tests
  - [ ] Heartbeat storage: SQLite on a persistent volume (moves to CloudNativePG in M3)
  - [ ] Adaptive heartbeat v1: zero-setup threads, learned rhythm, snooze/pause, daily nudge budget (see "Heartbeat")
  - [ ] Self-hosted **ntfy** on the cluster → phone notifications with a "Done" action button (alarm-level priority for what matters)
  - [ ] Capture inbox: share a link (e.g. a reel) from the phone into a "Research inbox" thread
  - [ ] CI: lint → test → build → Trivy → cosign → GHCR
  - [ ] ArgoCD watching `assaee-gitops`; Traefik + Gateway API + cert-manager → `https://api.as-saee.com/health`
  - [ ] External uptime check → Discord
  - [ ] Blog post #1 "Day 1 of building As-Sa'ee" (cross-post LinkedIn, dev.to)
- [ ] **M2 — Delivery done right**
  - [ ] Hugo blog at `blog.as-saee.com`, deployed by the pipeline
  - [ ] Staging (auto from `main`) and prod (tagged release) namespaces; promotion flow
  - [ ] SOPS + age secrets; Renovate on all repos; branch protection
- [ ] **M3 — Data + observability** (study: CKA)
  - [ ] CloudNativePG Postgres; continuous backups to Oracle object storage, copy to Cloudflare R2
  - [ ] Monthly automated restore test → result posted to Discord; restore runbook
  - [ ] Prometheus, Grafana, Loki, Tempo; OpenTelemetry in the API; Alertmanager → Discord
  - [ ] SLOs (e.g. 99.5% availability, p95 latency) with error-budget alerts
- [ ] **M4 — Identity + access**
  - [ ] Keycloak; API auth (OIDC); Grafana + ArgoCD single sign-on
  - [ ] Admin surfaces reachable only via Tailscale
- [ ] **M5 — Reader module on the platform** (research round first)
  - [ ] Reader schema, PDFs in R2 (copied to Oracle), offline-first sync (drift), data export endpoint
  - [ ] Mobile CI: signed APK → GitHub Releases → Obtainium; Dart client generated from the API's OpenAPI spec
  - [ ] Widget fixes + reading features from the original reader roadmap (kept in `assaee-mobile`)
- [ ] **M6 — AWS replica** (study: AWS Solutions Architect Associate)
  - [ ] Same platform built on AWS with Terraform, tested, documented, destroyed
- [ ] **M7 — Planner v1** (research round first)
- [ ] **M8 — Fitness v1** (research round first)
- [ ] **M9 — Deen v1** (research round first)
- [ ] **Later:** hadith search · Spanish module · content creation (2027–28) · RAG over own data (~2027) · fine-tuned own model (~2028) · iPhone build

## The idea
As-Sa'ee is the author's personal "life OS" — reading, planning, fitness, deen, later content and Spanish — and,
just as importantly, a **public DevOps/cloud-architect portfolio** for jobs in **Spain**. Single user for now, with
real accounts. The infrastructure is a deliverable in its own right; features must be ones the author actually uses
daily, so the platform carries real traffic, real metrics and real incidents to write about.

## Platform decisions and why
| Area | Decision | Why |
|---|---|---|
| Career target | AWS + Kubernetes + Terraform (certs: Terraform Assoc → CKA → AWS SAA) | Most common in DevOps/cloud job ads; certs line up with milestones |
| Client | Flutter (Android now, iOS later via cloud macOS CI) | One codebase for Android/iOS/web; existing reader carries over. Home-screen widgets are native code in any framework |
| Backend | Python FastAPI modular monolith + worker; AI service later | Python is the AI/ML language (RAG, fine-tuning). Split services only where scaling needs differ — a better interview story than many microservices for one user |
| Offline | Offline-first: local drift DB, last-edit-wins sync | Reading/gym/no-signal use. Single user, so conflicts are rare |
| Data | PostgreSQL (CloudNativePG), one schema per module, UUIDv7 IDs, soft deletes, server-stamped `updated_at`, full JSON export; pgvector later | Sync needs these anyway; clean exportable data is the future AI training set; export is a good GDPR habit in the EU |
| Files | PDFs and media in Cloudflare R2, not the database | Free 10 GB, S3-compatible |
| Hosting | Oracle Always Free ARM k3s (Madrid), **free-only account** | €0; data stays in Spain. Reclaim risk accepted: all infra is rebuildable from Terraform/Ansible + backups |
| AWS | Temporary Terraform-built replica, then `terraform destroy` | Proves AWS skills without a monthly bill (EKS alone ≈ $70/mo) |
| Repos | Polyrepo under GitHub org `as-saee`, all public: `assaee-mobile`, `assaee-api` (API + worker), `assaee-infra`, `assaee-gitops`, `assaee-docs`, `readflow-web` (archived) | Author's choice. API publishes OpenAPI; mobile generates its client from it to stop drift |
| Environments | Local (Compose) → staging (auto from `main`) → prod (tagged release). No preview envs yet | Free cluster memory is limited |
| CI/CD | GitHub Actions: lint, test, build, Trivy, cosign, GHCR → GitOps commit → ArgoCD. Renovate. APK via GitHub Releases + Obtainium | ArgoCD is most common in job ads; nobody `kubectl apply`s by hand. Play Store ($25) can wait |
| Secrets | SOPS + age now; External Secrets + AWS Secrets Manager after moving to AWS | Free now; the migration is itself a portfolio story |
| Ingress | Cloudflare DNS/TLS → Oracle free LB → Traefik with Gateway API + cert-manager | ingress-nginx was retired by the Kubernetes project in 2026; Gateway API is the future-proof skill |
| Access / login | Admin tools only via Tailscale; Keycloak for app login and SSO | Keycloak is common in Spanish enterprises/banks |
| IaC | Terraform (Oracle, Cloudflare, AWS, GitHub) + Ansible (node config, k3s); state in Oracle object storage | Ansible appears alongside Terraform in Spanish job ads; stay on Terraform (not OpenTofu) for the cert |
| Observability | Prometheus, Grafana, Loki, Tempo, OpenTelemetry, SLOs, external uptime check, alerts to **Discord** | In-cluster monitoring dies with the cluster, so an outside check is required. Discord keeps full history free |
| Backups | 3-2-1: DB → Oracle object storage + copy on R2; PDFs on R2 + copy on Oracle; monthly automated restore test | "I tested my disaster recovery" |
| Public writing | ADRs, runbooks, postmortems (English) in repos; Hugo blog on the cluster, cross-posted to LinkedIn/dev.to | What you explain counts as much as what you build. Spanish summaries later as the author learns Spanish |
| Name | As-Sa'ee; technical `assaee`; domain `as-saee.com` (verified unregistered 2026-10-06; `assaee.com` is taken); Android ID `com.assaee.app` | Apostrophes/hyphens not allowed in package IDs |

**Why walking skeleton first (M1):** a hello service going through the *entire* path (CI → signed image → ArgoCD →
k3s → HTTPS → alert) gets something live and blog-worthy within weeks, and every later layer is added to something
already running. This is how real platform teams work.

## Heartbeat — the first real feature (decided 2026-10-09)
**The problem the whole app exists for:** the author starts something (e.g. a book that needs a focused mind), switches
to something else, and the first thing just vanishes for days. It comes back only when the author happens to remember
it, and then the cycle repeats. Data being spread across apps is not the real issue. The real issue is "out of sight,
out of mind": once you stop, nothing pulls the thread back.

**The feature:** every thread (a book, a project, gym, Quran, Spanish…) has a **rhythm** set by the author (daily,
3×/week, weekly) and a **last-touched** time. When a thread has been quiet longer than its rhythm allows, it
**comes back to the author's attention** before it dies, e.g. "Book: last read 2 days ago". Touches are logged with one
tap in v1. Automatic touches (GitHub commits, VS Code project opened, gym log) come later.

**Why it goes in M1 instead of a hello endpoint:**
- It was the original problem. On the old order it would arrive at M7, after months of platform work, so the app
  would become one more delayed project, which is exactly the failure it is meant to fix.
- It is small (threads, rhythm, last-touched, a "touched" action, a list of quiet threads), so the walking skeleton
  still goes live within weeks.
- The author uses it every day from week one, so the platform carries real traffic, metrics and incidents from the
  start (see "The idea").
- The career plan doesn't change: it still goes through the full CI → signed image → ArgoCD → k3s → HTTPS → alert path.

**How it fits the design principles:** resurfacing must be a gentle return, not a guilt notification (principle 6).
That means no streaks, no "you failed", no red counts, only "last touched N days ago" (principle 3, "no failed days,
only returns"). The Planner (M7) grows out of the heartbeat; it does not replace it.

**Storage in M1:** SQLite on a persistent volume, because Postgres (CloudNativePG) arrives in M3. The move to Postgres
in M3 becomes a real data-migration story for the blog.

**Smart, adaptive, configurable (decided 2026-10-09).** The author has no single fixed routine, so the app must not
demand set-up answers up front. Every rule below has a default and can be overridden per thread in the app; nothing
is required except a name.
- **Zero-setup threads:** create one with just a name. It starts with a default rhythm for its kind (book, project,
  gym, research…) and the author can change it any time.
- **Learned rhythm:** after a few touches the app uses the author's *real* gap between touches (a running typical
  interval), not a number typed in once. A thread "goes quiet" when the current gap is clearly longer than usual.
  Routines that change week to week are handled automatically.
- **Learns from replies:** a nudge answered with "Done" keeps its timing; nudges that are repeatedly snoozed or
  ignored back off for that thread. Snooze options: later today / tomorrow / pause until a date (travel, Ramadan,
  exams). A paused thread never nags.
- **Context:** active days per thread (work projects on workdays only), quiet hours, and later prayer windows as
  anchors (principle 1) so nudges arrive at natural breaks, not mid-meeting.
- **Nudge budget:** a daily cap; when several threads are quiet, the most overdue / most important come first and the
  rest are grouped into one message. No notification storms (principle 6).
- **How it reaches the author:** phone notification via self-hosted ntfy (free, open source) with a "Done" button so a
  touch needs no app; top priority acts like an alarm. The Flutter app later takes over with local notifications and
  real alarms (exact-alarm permission). Not a system app — no root needed.
- **Kinds of thread:** *heartbeat* (book, projects, gym), *inbox* (research ideas captured from reels/links; the inbox
  itself resurfaces when items pile up), and later *fixed-time* (prayers, M9). Follow-ups with people fit as inbox items
  with a person attached.
- **"Smart" means plain statistics on the author's own data in v1, no paid AI APIs.** The own-AI plans (RAG ~2027)
  can build on this history later.

## Design principles for every module
From the research in [`research/`](research/) (evidence strength is in those files). Each practice shown in the app is labelled
with its source and, where it exists, what modern research says (author's choice).
1. Anchor the day to the sun and the prayers, not the clock (Spain's clock is ~1h ahead of solar time; prayer anchors follow the sun).
2. Every practice has a small minimum form that counts fully ("small but consistent", Bukhari 6464; Lally 2010).
3. No failed days, only returns. Show consistency over a window (e.g. 40 days), weekly targets with repair, never a red 0.
4. Plans as "after X, I do Y", tied to prayers or routines (implementation intentions, strongest evidence found).
5. Niyyah at Fajr, short muhasaba after Isha ending in a resolve.
6. No points, badges, feeds, leaderboards or guilt notifications (overjustification effect; ikhlas).
7. Mastery levels earned by demonstrated skill, like an ijazah.
8. Self-testing over rereading (retrieval practice).
9. Health as Ibn Sina's balanced regimen, not optimisation.
10. The app succeeds when the user closes it.

## Module v1 sketches (each gets its own research round before building)
- **Reader (M5):** PDFs synced and offline; progress, bookmarks, highlights/notes as a commonplace book; reading goals feed the planner; widget fixes from the original reader roadmap. EPUB later.
- **Planner (M7, the hub):** grows out of the M1 heartbeat (threads + rhythms stay the core). Today view by prayer windows + Anytime bucket; tasks with do-date, deadline, prayer anchor, RRULE recurrence; habits with weekly targets; day templates (al-Ghazali's awrad); evening muhasaba; fresh-start prompts on Jumu'ah / new Hijri month / Ramadan; configurable day rollover (summer Isha in Madrid ≈ 23:30). Single cross-module `plan_item` feed. See [`research-planner.md`](research/research-planner.md).
- **Fitness (M8):** fast set logger, rest timer, PRs; calisthenics progressions, timed holds, weighted sets; football sessions; mobility routines; body weight; steps + sleep from Health Connect on app open; minimum version of every routine; starter packs Pehlwani + Baduanjin; fasting-day flag. Seed exercises from free-exercise-db (public domain). See [`research-fitness.md`](research/research-fitness.md).
- **Deen (M9):** prayer times — windows from Sahih Muslim 612, Standard Asr (shadow = 1×), Muslim World League angles by default + "match my mosque" offsets, qibla; madhab setting "no madhab — show the evidence"; Quran = Tanzil Arabic + Saheeh International (EN) + Isa García (ES) via QuranEnc terms; adhkar each traced to a cited hadith; hifz by mushaf page with FSRS, recite-then-reveal, lawh mode. Hard rules: never generated Quran/hadith text, AI answers only with exact citations, no fatwas, hadith grades stored with their grader. Public repos commit import scripts + checksums, not copyrighted texts. See [`research-islamic.md`](research/research-islamic.md).
- **Later:** Spanish (v1 = habit tracking in the planner only), content creation (2027–28), RAG (~2027), fine-tuning (~2028).

## Risks
- Oracle free ARM capacity is often "out of stock" — retry provisioning; free-only accounts can have idle VMs reclaimed.
- Platform work can starve features forever; M5 (reader live) is the checkpoint that the platform serves a real app.
- Exam fees (~$70 Terraform, ~$150 SAA, ~$445 CKA) are outside the €0 budget; book when funds allow.
- Licensing: most Quran translations, tafsir, hadith translations and recitation audio are copyrighted — check each source before use.
