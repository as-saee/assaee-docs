# ADR 0001: Platform and product decisions for As-Sa'ee

- **Status:** Accepted, 2026-10-06. Amended 2026-10-09 (see [Amendments](#amendments)).
- **Deciders:** the author.
- **Records:** the decisions from the planning session that produced [the roadmap](../roadmap.md). The research behind
  the product decisions is in [`../research/`](../research/).

## Context

As-Sa'ee is a personal life OS (reading, planning, fitness, deen; later content, Spanish and a self-trained AI), and
at the same time a public portfolio for DevOps / cloud-architect roles in Spain. Both goals shape every choice:

- The infrastructure is a deliverable in its own right, so it should be the kind real teams run and job ads ask for
  (Kubernetes, Terraform, GitOps, observability), not the cheapest way to host an app.
- The features must be ones the author uses every day, so the platform carries real traffic, metrics and incidents
  to write about.
- Budget is effectively €0. Anything that costs money has to be temporary, small, or deferred.
- Single user for now, but with real accounts, and data kept in the EU.

## Decisions

### Product
| Area | Decision | Reason |
|---|---|---|
| Client | Flutter; Android first, iOS later via cloud macOS CI | One codebase; the existing reader carries over. Home-screen widgets are native code in any framework. |
| Offline | Offline-first: local drift database, last-edit-wins sync | Used while reading, at the gym, with no signal. One user, so conflicts are rare. |
| Data | PostgreSQL, one schema per module, UUIDv7 ids, soft deletes, server-stamped `updated_at`, full JSON export; pgvector later | Sync needs these anyway. Clean, exportable data becomes the future AI training set, and export is a good GDPR habit. |
| Files | PDFs and media in Cloudflare R2, not in the database | Free 10 GB, S3-compatible. |
| Design principles | See [roadmap](../roadmap.md#design-principles-for-every-module): no streaks, points or guilt notifications; "no failed days, only returns"; every practice has a minimum form that counts | Taken from the research (planner, fitness, Islamic sources, psychology). The app succeeds when the user closes it. |

### Platform
| Area | Decision | Reason |
|---|---|---|
| Career target | AWS + Kubernetes + Terraform; certifications in the order Terraform Associate → CKA → AWS SAA | Most common in DevOps/cloud job ads; each certification lines up with a milestone. |
| Backend | Python FastAPI modular monolith plus a worker; AI service later | Python is the AI/ML language. Services are split only where scaling needs differ, which is a better story than many microservices for one user. |
| Hosting | Oracle Always Free ARM, k3s, Madrid region, free-only account | €0 and data stays in Spain. The risk of reclaimed capacity is accepted because everything is rebuildable from Terraform/Ansible plus backups. |
| AWS | Temporary Terraform-built replica, then `terraform destroy` | Shows AWS skills without a monthly bill (EKS alone is about $70/month). |
| Repos | Polyrepo under the `as-saee` GitHub organisation, all public: `assaee-mobile`, `assaee-api`, `assaee-infra`, `assaee-gitops`, `assaee-docs`, `readflow-web` (archived) | The API publishes OpenAPI and the mobile app generates its client from it, which stops drift between repos. |
| Environments | Local (Compose) → staging (auto from `main`) → prod (tagged release); no preview environments yet | Free cluster memory is limited. |
| CI/CD | GitHub Actions: lint, test, build, Trivy scan, cosign signing, push to GHCR, GitOps commit, ArgoCD sync; Renovate; APK via GitHub Releases | Nobody runs `kubectl apply` by hand. ArgoCD is the most common GitOps tool in job ads. The Play Store fee can wait. |
| Secrets | SOPS + age now; External Secrets + AWS Secrets Manager after any move to AWS | Free now; the later migration is itself something to write about. |
| Ingress | Cloudflare DNS/TLS → Oracle free load balancer → Traefik with Gateway API and cert-manager | `ingress-nginx` was retired by the Kubernetes project in 2026, so Gateway API is the future-proof skill. |
| Access and login | Admin tools only over Tailscale; Keycloak for app login and single sign-on | Keycloak is common in Spanish enterprises and banks. |
| Infrastructure as code | Terraform (Oracle, Cloudflare, AWS, GitHub) plus Ansible (node setup, k3s); state in Oracle object storage | Ansible appears next to Terraform in Spanish job ads. Staying on Terraform rather than OpenTofu keeps the certification path simple. |
| Observability | Prometheus, Grafana, Loki, Tempo, OpenTelemetry, SLOs, an external uptime check, alerts to Discord | In-cluster monitoring dies with the cluster, so an outside check is required. Discord keeps full history for free. |
| Backups | 3-2-1: database to Oracle object storage plus a copy on R2; PDFs on R2 plus a copy on Oracle; monthly automated restore test | "I tested my disaster recovery" is the story. |
| Public writing | ADRs, runbooks and postmortems in the repos (English); a Hugo blog on the cluster cross-posted to LinkedIn and dev.to | What is explained counts as much as what is built. Spanish summaries come as the author learns Spanish. |
| Name | As-Sa'ee (السعي); technical name `assaee`; domain `as-saee.com`; Android id `com.assaee.app` | Apostrophes and hyphens are not allowed in package ids. |

### Order of work
Walking skeleton first (M1): a small service goes through the whole path (CI → signed image → ArgoCD → k3s → HTTPS →
alert) within weeks, and every later layer is added to something already running. This is how platform teams work.
The modules (reader, planner, fitness, deen) each get their own research round before they are built.

## Consequences

- **Good:** a complete, realistic delivery path is public from early on; nothing depends on paid services; the data
  model is ready for sync, export and later AI use.
- **Costs:** a lot of platform work before most features exist, so progress is checked against M5 (the reader running
  on the platform). Polyrepo means changes that span API and app need two pull requests. Exam fees (about $70
  Terraform, $150 SAA, $445 CKA) are outside the €0 budget and wait until money allows.
- **Risks:** Oracle free ARM capacity is often out of stock and idle free-tier VMs can be reclaimed; most Quran
  translations, tafsir and hadith translations are copyrighted, so each source is checked before use and public
  repos hold import scripts and checksums, not the texts.
- **Revisit when:** the free cluster runs out of memory; the author moves hosting to AWS; the app gets a second user;
  or the platform work is starving features (the M5 checkpoint).

## Amendments

**2026-10-09: the first real feature is the "heartbeat".**
Context: the problem the app exists for is things going quiet. The author starts something, switches to something
else, and the first thing vanishes for days. Decision: M1's feature is the heartbeat (every thread has a rhythm and a
last-touched time, and comes back to attention when it has been quiet too long) instead of a plain hello endpoint.
It still goes through the full delivery path. Rhythms are learned from the author's real behaviour, with no required
setup, snooze and pause, and a daily cap on notifications. Notifications are delivered by a self-hosted **ntfy**
server (free, open source) with a "Done" action, until the mobile app can send its own. Storage in M1 is SQLite on a
persistent volume, moving to CloudNativePG in M3. Not chosen: running as an Android system app, which needs root and
buys nothing a normal app with exact-alarm permission cannot do.

**Not recorded here:** alternatives that were considered and dropped during planning are only captured where the
reason is stated above; the others were not written down at the time.
