# ADR-0002 — Stack selection

- **Status:** Accepted
- **Date:** 2026-02-09
- **Deciders:** Founder
- **Milestone:** [M0](../../product/milestones/M0.md)
- **Supersedes:** none

---

## Amendment, 2026-09-18

The decision below names Next.js 15 because that was current when it was written.
The web app now runs **Next.js 16**. This is not a superseding decision — the
choice being recorded here is Next.js with the App Router, and that stands. Only
the major version moved.

The bump was forced rather than elective: a high-severity PostCSS advisory
(arbitrary file read via an attacker-controlled `sourceMappingURL`) is reachable
through Next's build tooling on 15, and the fix ships in 16. Three things broke
in the upgrade — `eslint-config-next` 16 dropped the `FlatCompat` shim, Vitest 4
transforms with oxc instead of esbuild, and jsdom 30 needs Node 22 — all handled
in the web repository rather than here.

Read every "Next.js 15" below as "Next.js 16".

---

## Context

GoldSeats is being built by one engineer, pre-revenue, from an existing base of two working
artefacts:

- A finished single-file marketing landing page (`index.html`, 876 lines) — dark cinema
  theme, `#0a0a0f` background, `#FFD700` gold accents, Inter and Playfair Display, currently
  deployed on Netlify.
- A working synthetic seat-map dataset generator in Python, which already contains the seat
  scoring logic the entire product depends on: five weighted factors, five theatre geometry
  types, a booking simulator, and a Pillow renderer.

The product needs: a public catalog with filters, a film detail page with an aggregated
timeline, a 2D seat map, a 3D seat preview, a chatbot, and a deep-link booking handoff.

The constraints that actually determine the answer:

1. **The seat scoring engine is already written in Python.** It is the product's core
   intelligence and its tuning represents real work. Rewriting it in another language to
   satisfy a stack preference would be destroying value for no user benefit.
2. **Docker is not installed on the development machine**, and installing it is not a
   prerequisite we want between a contributor and their first running server. Verified
   environment: Node v20.19.0 / npm 10.8.2, Python 3.12.4 at `/usr/local/bin/python3`,
   `gh` 2.100.0.
3. **One engineer.** Anything requiring more than one deployable's worth of operational
   attention is a net loss.
4. **Zero budget for CI and hosting.** This is what drives public repos: branch protection
   rulesets and GitHub Actions minutes are free on public repositories, paid on private ones.
5. **The 3D seat view is a committed feature**, which means the web stack needs a mature
   WebGL story.
6. **SEO matters.** "Best seats at Scotiabank Theatre Toronto" is a search query we want to
   win, so film and theatre pages must server-render.

## Decision

### Repositories

Three **public** repos in one GitHub org: `goldseats-web`, `goldseats-api`,
`goldseats-docs`. Public for free branch protection and CI. No secrets in git; only
`.env.example` is committed — see
[`../../engineering/secrets-and-config.md`](../../engineering/secrets-and-config.md).

### Frontend — `goldseats-web`

- **Next.js 15, App Router.** Server components for the catalog (SEO, and a fast first paint
  on mobile), client components only where interaction requires them.
- **TypeScript**, strict, plus `noUncheckedIndexedAccess`.
- **Tailwind CSS**, with the landing page's palette extracted into design tokens.
- **react-three-fiber** for the 3D seat view ([M6](../../product/milestones/M6.md)).
- **npm.** Not pnpm, not yarn. It's installed, it works, `package-lock.json` is committed.
- **Vitest** for components and logic, **Playwright** for end-to-end.
- **ESLint + Prettier.**

### Backend — `goldseats-api`

- **Python 3.12 + FastAPI.** Async, generates OpenAPI for free, and shares a language with
  the scoring engine and the dataset generator.
- **SQLAlchemy 2.0** with typed `Mapped[]` declarative models, **Alembic** for migrations.
- **Pydantic v2** for request/response schemas and for settings.
- **Postgres 16** in staging and production; **SQLite** in local development. Switched purely
  by `DATABASE_URL`.
- **Redis** for caching, memoised scores, and rate limiting. Optional locally, required in
  production.
- **pip + `requirements.txt`.** Not Poetry, not uv.
- **Ruff** for lint and format, **mypy** strict, **pytest**.

### Git workflow

Trunk-based off `main`. Short-lived `feat/`, `fix/`, `chore/` branches. Conventional Commits.
Squash merge. Linear history. Required CI plus one approval, no admin bypass. Details in
[`../../engineering/git-workflow.md`](../../engineering/git-workflow.md).

### The SQLite decision, stated plainly

**SQLite by default locally is a hard requirement, not a convenience.** `git clone` →
`pip install -r requirements.txt` → `alembic upgrade head` → `uvicorn` must work with no
services running and no container runtime installed.

The cost is a list of Postgres features we cannot use anywhere in the codebase: `JSONB`
operators, `ARRAY` columns, Postgres `ENUM` types, `ILIKE`, `tsvector`,
`CREATE INDEX CONCURRENTLY`, partial indexes, and `NOW()` defaults. The full list, with
substitutes, is in
[`../../engineering/database-conventions.md`](../../engineering/database-conventions.md).

We pay for this with a CI job that runs the API test suite twice — once on SQLite, once
against a real Postgres service container. Both are required checks. That parity job is the
only thing standing between us and "worked locally, 500s in production", so it does not get
skipped.

## Options considered

### Frontend

#### Next.js 15 App Router (chosen)

- **Pros:** Server components give us SEO and a fast mobile first paint without a separate
  rendering layer. Mature ecosystem for three.js via react-three-fiber. Image optimisation
  handles TMDB posters, which is directly a Core Web Vitals win. One framework covers the
  marketing page and the app. Huge hiring pool.
- **Cons:** App Router has real complexity — the caching model in particular is easy to get
  subtly wrong. Framework churn between major versions has been significant.

#### Remix / React Router 7

- **Pros:** Simpler, more predictable data-loading model. Excellent progressive enhancement,
  which suits our filter forms.
- **Cons:** Smaller ecosystem. No equivalent of `next/image`, which we'd have to build.
- **Why not:** Close call. Next.js's image handling and the react-three-fiber ecosystem
  decided it. The filter forms work without JavaScript either way.

#### Astro with React islands

- **Pros:** Best-in-class for content pages. Minimal JavaScript shipped, which suits the
  marketing page and the catalog perfectly.
- **Cons:** The seat map, the 3D view, and the chatbot are three large interactive islands.
  At that point we're running a SPA inside a static site generator and getting the worst of
  both.
- **Why not:** Our app is more interactive than content-driven, once you get past the home
  page.

#### SvelteKit

- **Pros:** Smaller bundles, genuinely nicer authoring experience, excellent DX.
- **Cons:** Smaller three.js ecosystem (threlte is good but thinner than
  react-three-fiber). Smaller hiring pool.
- **Why not:** react-three-fiber is the most mature WebGL-in-a-framework option and
  [M6](../../product/milestones/M6.md) is a committed feature, not a maybe.

#### Plain React SPA with Vite

- **Pros:** Simplest mental model. No framework opinions to fight.
- **Cons:** No SSR, so no SEO on film and theatre pages — which is a primary acquisition
  channel. We'd end up building our own SSR.
- **Why not:** SEO is a requirement.

### Backend

#### Python 3.12 + FastAPI (chosen)

- **Pros:** The scoring engine is already Python — zero rewrite, and the dataset generator
  migrates into the same repo at `ml/dataset/`. Async by default. OpenAPI generated from
  Pydantic models, which is what lets the web app generate its types. If scoring ever
  becomes a real ML model, we're already in the ecosystem.
- **Cons:** Slower than a compiled language, though nothing in our workload is
  latency-critical enough to notice. Async Python has sharp edges — a sync call in an async
  handler blocks the event loop and it's easy to do accidentally.

#### Node + TypeScript (Nest or Fastify)

- **Pros:** One language across the whole stack. Shared types without codegen. Smaller
  cognitive load for a solo developer.
- **Cons:** **Would require rewriting the scoring engine in TypeScript.** That's the
  product's core IP, already tuned, with a synthetic dataset built around it. Numeric and
  ML tooling is far weaker than Python's.
- **Why not:** Rewriting the seat scorer to achieve language uniformity is the clearest
  example of optimising for the developer's comfort over the product's value. One language
  is genuinely nice; not as nice as not throwing away the thing that makes GoldSeats work.

#### Django + DRF

- **Pros:** Batteries included — admin, auth, ORM, migrations. The Django admin would
  genuinely be useful for hand-seeding theatre data in [M2](../../product/milestones/M2.md).
- **Cons:** Heavier and more opinionated than we need for a JSON API. Async support is
  bolted on. DRF's OpenAPI generation is markedly worse than FastAPI's.
- **Why not:** The admin is tempting, but a seeding CLI reading committed JSON files gives
  us reviewable, version-controlled theatre data — which is strictly better than clicking
  through an admin form. See [ADR-0003](0003-manual-seed-theatre-data.md).

#### Go

- **Pros:** Fast, single binary, excellent concurrency, trivial deployment.
- **Cons:** Rewrite the scoring engine. Weak numeric ecosystem. More boilerplate per
  endpoint.
- **Why not:** Same as Node, with less upside.

### Local database

#### SQLite by default, Postgres via `DATABASE_URL` (chosen)

- **Pros:** Zero-dependency local setup. No Docker. `rm goldseats.db` resets everything.
  Tests run in-memory and fast. Genuinely lowers the barrier for a future contributor.
- **Cons:** Two engines to support. A real list of Postgres features off-limits. A class of
  bug that only appears in production. Requires the CI parity job.

#### Postgres everywhere via Docker Compose

- **Pros:** Perfect dev/prod parity. Full Postgres feature set — `JSONB` querying would
  genuinely simplify `layout_json`. One engine.
- **Cons:** **Docker is not installed and we don't want to require it.** Adds a container
  runtime, a several-hundred-megabyte download, and a class of "why won't my container
  start" problems between a contributor and their first `uvicorn`.
- **Why not:** This is the constraint, stated in the Context. Worth noting that it's the
  option we'd pick if Docker were installed, which is exactly why it's written down here —
  if that constraint disappears, the trade-off changes.

#### Postgres everywhere via a local `brew` install

- **Pros:** Parity without Docker.
- **Cons:** A running service to manage, a version that drifts from production, and a setup
  step that fails differently on every machine.
- **Why not:** Roughly Docker's downsides with worse parity.

#### SQLite in production too

- **Pros:** One engine. Modern SQLite genuinely handles more load than people expect.
- **Cons:** No straightforward managed backups with point-in-time recovery, no concurrent
  writers, single-host. Makes [M9](../../product/milestones/M9.md)'s backup and restore
  drill substantially harder.
- **Why not:** Production data needs managed backups and WAL archiving. That's not
  negotiable for user accounts.

### Package managers

**npm** over pnpm/yarn, and **pip + `requirements.txt`** over Poetry/uv: both are already
installed and working, both are what every tutorial and CI example assumes, and neither is
a bottleneck at our scale. uv is genuinely faster and Poetry genuinely resolves better; we
would be trading a working setup for a marginal improvement plus a new thing to explain.
Revisit if install time becomes a real CI cost.

## Consequences

### What this makes easier

- The scoring engine migrates rather than gets rewritten. `generator/recommender.py` becomes
  `app/ml/scoring.py` in [M5](../../product/milestones/M5.md), and the synthetic dataset
  becomes the test fixtures and the chatbot's evaluation set.
- Local setup is three commands and needs nothing running. See
  [`../../onboarding/local-setup.md`](../../onboarding/local-setup.md).
- The landing page migrates directly: its palette becomes Tailwind tokens, its markup becomes
  the marketing route group.
- OpenAPI generation means `goldseats-web` never hand-writes an API type.
- Free CI with real branch protection, on all three repos.

### What this makes harder

- Two languages, so two toolchains, two lint configs, two CI pipelines, and context
  switching. This is the main cost of the decision and we're accepting it deliberately.
- The SQLite/Postgres split means a permanent discipline tax on every query and every
  migration, and a CI job that must never be skipped.
- Async Python requires care. Blocking calls in a handler, and CPU-bound scoring on the event
  loop, are both easy mistakes with production consequences. Hence the
  `anyio.to_thread.run_sync` rule in
  [`../../engineering/code-conventions-python.md`](../../engineering/code-conventions-python.md).
- Next.js App Router caching is subtle, and the failure mode — a cached seat map showing a
  seat that's been sold — is the worst bug this product can have. Hence explicit cache policy
  on every `fetch`.

### What this commits us to

- Maintaining SQLite compatibility for the life of the project, or writing an ADR to drop it.
- The CI parity job as a required check.
- Generating web types from the API's OpenAPI schema, which means schema quality is not
  optional.
- Keeping the scoring engine pure and free of SQLAlchemy, so it remains testable against the
  dataset fixtures.

## Revisit when

- **Docker becomes installed and normalised** on all development machines. Then Postgres
  everywhere becomes strictly better and this is worth a superseding ADR.
- **A second engineer joins and is stronger in Node than Python.** The two-language cost
  changes shape, though the scoring engine's gravity doesn't.
- **npm install time exceeds ~60 seconds in CI**, or pip resolution becomes a real bottleneck
  — then pnpm or uv earn their switching cost.
- **Scoring becomes a trained model rather than a weighted heuristic.** Python was already
  the right choice; this makes it unambiguous.
- **`tsvector`-grade search becomes a product feature** rather than a `LIKE` filter. That
  forces a decision on SQLite parity.

## References

- [`../overview.md`](../overview.md)
- [`../../engineering/database-conventions.md`](../../engineering/database-conventions.md)
- [`../../engineering/code-conventions-python.md`](../../engineering/code-conventions-python.md)
- [`../../engineering/code-conventions-typescript.md`](../../engineering/code-conventions-typescript.md)
- [ADR-0003 — Manually seed theatre data](0003-manual-seed-theatre-data.md)
- [ADR-0004 — Self-hosted LLM](0004-self-hosted-llm.md)
