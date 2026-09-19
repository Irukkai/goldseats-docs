# GoldSeats Documentation

GoldSeats is an AI-powered movie-theatre seat recommendation product. You tell us the
showtime and who you're going with; we tell you which seats are actually worth sitting
in, show you the view from them in 3D, and hand you off to the theatre's own booking
page to pay.

Domain: `goldseats.app`. Primary market: Canada (Cineplex).

This repository (`goldseats-docs`) is the single source of truth for how we build
GoldSeats: engineering conventions, architecture decisions, the product roadmap,
operational runbooks, and legal policy. Code lives elsewhere; nothing here ships to
users.

---

## What GoldSeats does

| Capability | Where it's specified |
| --- | --- |
| Score every available seat in an auditorium and rank the best ones | [Seat scoring in `architecture/overview.md`](architecture/overview.md#seat-scoring-engine), [M5](product/milestones/M5.md) |
| Conversational seat picking grounded in real data via tool calls | [ADR-0004](architecture/adr/0004-self-hosted-llm.md), [M7](product/milestones/M7.md) |
| 3D "view from this seat" preview | [M6](product/milestones/M6.md) |
| Deep-link handoff to the theatre's booking page (we never take payments) | [M5](product/milestones/M5.md), [`legal/data-sources-policy.md`](legal/data-sources-policy.md) |
| Buy online now, or reserve the pick and pay in person | [M5](product/milestones/M5.md) |
| Follow films, unified news feed, release notifications | [M8](product/milestones/M8.md) |
| Home page of currently playing / upcoming films with filters and language selector | [M3](product/milestones/M3.md) |
| Film detail page: aggregated timeline, nearby showtimes, best format, best seats | [M4](product/milestones/M4.md) |

---

## Repo map

GoldSeats is three public repositories in one GitHub org. They are public so that
branch protection rulesets and GitHub Actions minutes are free. No secrets are ever
committed; only `.env.example` files are.

| Repo | Contents | Stack |
| --- | --- | --- |
| `goldseats-web` | Marketing page, home page, film detail, seat map, 3D seat view, chat UI | Next.js 16 (App Router), TypeScript, Tailwind CSS, react-three-fiber |
| `goldseats-api` | REST API, seat scoring engine, TMDB ingestion, theatre seeding CLI, chatbot tool server | Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, Postgres/SQLite, Redis |
| `goldseats-docs` | This repository | Markdown |

Clone all three as siblings:

```bash
cd ~/Startup/goldseats
git clone https://github.com/Irukkai/goldseats-web.git
git clone https://github.com/Irukkai/goldseats-api.git
git clone https://github.com/Irukkai/goldseats-docs.git
```

The GitHub organization is [`Irukkai`](https://github.com/Irukkai) — Tamil for
"seat". All three repositories are public.

---

## How to navigate these docs

**If you are new here, read these three, in order:**

1. [`onboarding/day-one.md`](onboarding/day-one.md) — what to read, what to install, what to ship on day one.
2. [`onboarding/local-setup.md`](onboarding/local-setup.md) — exact commands to get the API and web app running locally.
3. [`product/roadmap.md`](product/roadmap.md) — the M0–M9 plan and what we're working on right now.

**Then reference as needed:**

### `engineering/`
How we write, review, test, and ship code.

- [`code-conventions-typescript.md`](engineering/code-conventions-typescript.md) — TypeScript/React/Next.js rules
- [`code-conventions-python.md`](engineering/code-conventions-python.md) — Python/FastAPI/SQLAlchemy rules
- [`git-workflow.md`](engineering/git-workflow.md) — trunk-based flow, Conventional Commits, squash merge
- [`branching-and-releases.md`](engineering/branching-and-releases.md) — branch naming, tagging, versioning
- [`code-review.md`](engineering/code-review.md) — what reviewers must check, SLAs
- [`testing-strategy.md`](engineering/testing-strategy.md) — the test pyramid across pytest, Vitest, Playwright
- [`definition-of-done.md`](engineering/definition-of-done.md) — the checklist every issue must satisfy
- [`api-design-guidelines.md`](engineering/api-design-guidelines.md) — URL shape, pagination, errors, versioning
- [`database-conventions.md`](engineering/database-conventions.md) — naming, migrations, SQLite/Postgres parity
- [`observability.md`](engineering/observability.md) — structured logging, OpenTelemetry, Sentry, SLOs
- [`secrets-and-config.md`](engineering/secrets-and-config.md) — env var handling, `.env.example` discipline
- [`accessibility.md`](engineering/accessibility.md) — WCAG 2.2 AA targets, seat map a11y
- [`performance-budgets.md`](engineering/performance-budgets.md) — Lighthouse, API latency, 3D frame budgets

### `architecture/`
- [`overview.md`](architecture/overview.md) — system diagram and component responsibilities
- [`data-model.md`](architecture/data-model.md) — full table-by-table schema with ER diagram
- [`adr/`](architecture/adr/) — architecture decision records, starting with [ADR-0001](architecture/adr/0001-record-architecture-decisions.md)

### `product/`
- [`roadmap.md`](product/roadmap.md) — M0–M9, sequencing, parallelism
- [`glossary.md`](product/glossary.md) — every GoldSeats term, defined once
- [`milestones/`](product/milestones/) — one file per milestone with scope, deliverables, issue checklist, risks
- [`prd/`](product/prd/README.md) — product requirement docs and the template for new ones

### `operations/`
- [`environments.md`](operations/environments.md) — local, staging, production
- [`release-process.md`](operations/release-process.md) — how a merge becomes a deploy
- [`incident-response.md`](operations/incident-response.md) — severities, roles, comms
- [`on-call.md`](operations/on-call.md) — rotation and expectations
- [`runbooks/`](operations/runbooks/README.md) — [database restore](operations/runbooks/database-restore.md), [API rollback](operations/runbooks/api-rollback.md), [domain cutover](operations/runbooks/domain-cutover.md)

### `legal/`
- [`data-sources-policy.md`](legal/data-sources-policy.md) — **read before adding any data source.** No scraping, TMDB attribution, deep-link-only booking.
- [`privacy-policy.md`](legal/privacy-policy.md)
- [`terms.md`](legal/terms.md)

### `templates/`
Copy-paste starting points: [RFC](templates/rfc-template.md), [pull request](templates/pull-request-template.md),
[bug report](templates/issue-bug-report.md), [feature request](templates/issue-feature-request.md),
[CODEOWNERS](templates/CODEOWNERS.example).

---

## Locked technical decisions

These are settled. Do not re-open them in a pull request; open an ADR that supersedes
the existing one instead (see [ADR-0001](architecture/adr/0001-record-architecture-decisions.md)).

- **Three public repos** in one org — free branch protection and CI.
- **Next.js 16 App Router + TypeScript + Tailwind** for web. npm, not pnpm or yarn.
- **Python 3.12 + FastAPI + SQLAlchemy 2.0 + Alembic** for the API. pip + `requirements.txt`.
- **SQLite by default locally, Postgres in production**, switched purely by `DATABASE_URL`. Docker is not installed on the founder's machine, so a zero-container local setup is a hard requirement. See [ADR-0002](architecture/adr/0002-stack-selection.md).
- **Manually seeded theatre data. No scraping.** See [ADR-0003](architecture/adr/0003-manual-seed-theatre-data.md).
- **Self-hosted open-weight LLM** — Ollama locally, vLLM in production — with tool calls into our own scoring engine so it can never invent a seat. See [ADR-0004](architecture/adr/0004-self-hosted-llm.md).
- **Redis** for caching, **react-three-fiber** for the 3D seat view, **TMDB** for film metadata.
- **Trunk-based development** off `main` with short-lived `feat/`, `fix/`, `chore/` branches, Conventional Commits, squash merge, required CI plus one approval.

---

## Contributing

Read [`CONTRIBUTING.md`](CONTRIBUTING.md). In short: branch off `main`, keep the
branch alive for less than two days, write a Conventional Commit, open a PR using
[the template](templates/pull-request-template.md), get CI green and one approval,
squash merge.

Behaviour expectations are in [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).
Vulnerability reporting is in [`SECURITY.md`](SECURITY.md).

---

## Open decisions

Tracked here so they don't get lost:

- **Hosting provider for `goldseats-web`.** [`operations/environments.md`](operations/environments.md)
  assumes Netlify continuity under a *new, owned* account; Vercel is the obvious
  alternative. Decided at [M9](product/milestones/M9.md) latest.
- **Launch scope.** Whether v1 requires M6 through M8, or whether M5 plus M9 is a
  legitimate launch. [`product/roadmap.md`](product/roadmap.md) deliberately does not
  force an answer.
- **Which auditoriums to seed first** in [M2](product/milestones/M2.md).
- **Which instruct model** (Llama vs Mistral) the chatbot serves. Benchmarked during
  [M7](product/milestones/M7.md), not before.

---

## Licence

Documentation and code in this repository are MIT licensed. See [`LICENSE`](LICENSE).

Film metadata is provided by TMDB, which is not endorsed or certified by TMDB. See
[`legal/data-sources-policy.md`](legal/data-sources-policy.md) for full attribution
obligations.
