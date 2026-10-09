# As-Sa'ee (السعي) — docs

Roadmap, architecture decisions and research for **As-Sa'ee**, a personal life OS (reading, planning, fitness, deen)
built in public as a DevOps / cloud-architect portfolio.

The project starts from one problem: things go quiet. You begin something, switch to something else, and the first
thing disappears for days until you happen to remember it. The first feature is a "heartbeat" that notices when a
thread has gone quiet longer than your own rhythm and brings it back gently, with no streaks and no guilt.

## Where to start
- [Roadmap](docs/roadmap.md): milestones M0–M9, the reasoning, and current progress.
- [Architecture decisions](docs/adr/): what was decided and why, starting with [ADR 0001](docs/adr/0001-platform-and-product-decisions.md).
- [Research](docs/research/): the evidence behind each module (planner, fitness, Islamic sources, older approaches to
  habit and attention).

## Repositories
| Repo | Purpose |
|---|---|
| [assaee-api](https://github.com/as-saee/assaee-api) | FastAPI backend and worker |
| [assaee-infra](https://github.com/as-saee/assaee-infra) | Terraform and Ansible |
| [assaee-gitops](https://github.com/as-saee/assaee-gitops) | Kubernetes manifests watched by ArgoCD |
| [assaee-mobile](https://github.com/as-saee/assaee-mobile) | Flutter app |
| [readflow-web](https://github.com/as-saee/readflow-web) | The original React PDF reader (archived) |

## Status
Early planning and setup (M0). Nothing is deployed yet; the roadmap's progress list is the source of truth.
