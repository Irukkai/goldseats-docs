# Contributing to GoldSeats

Thanks for working on GoldSeats. This document is the short version; the long version
lives in [`engineering/`](engineering/). If the two ever disagree, `engineering/` wins
and this file needs a fix.

GoldSeats is currently a very small team. That does **not** mean we skip review — it
means review is fast, and the process exists so that future-you (and future hires) can
understand why the code looks the way it does.

---

## Before you start

1. Read [`onboarding/day-one.md`](onboarding/day-one.md) and get your machine set up
   with [`onboarding/local-setup.md`](onboarding/local-setup.md).
2. Find or file an issue. **Every PR references an issue.** No drive-by refactors that
   nobody can trace back to a reason.
3. Check the issue is on the current milestone. If your change belongs to a later
   milestone, say so on the issue and move on — see [`product/roadmap.md`](product/roadmap.md).
4. If the change is architecturally significant (new dependency, new service, schema
   change that breaks a contract, a new data source), write an
   [ADR](architecture/adr/template.md) or an [RFC](templates/rfc-template.md) *first*.

**New data source?** Stop and read [`legal/data-sources-policy.md`](legal/data-sources-policy.md).
We do not scrape. There are no exceptions to that and a PR that adds a scraper will be
closed.

---

## The loop

```bash
# 1. Start from a fresh main
git checkout main
git pull --ff-only

# 2. Branch. Prefix must be feat/, fix/, or chore/.
git checkout -b feat/seat-scoring-endpoint

# 3. Work. Commit in Conventional Commit format.
git add app/api/routes/recommendations.py tests/api/test_recommendations.py
git commit -m "feat(recommendations): add POST /recommendations endpoint"

# 4. Run the same checks CI will run, locally, before pushing.
#    API:
ruff check . && ruff format --check . && mypy app && pytest
#    Web:
npm run lint && npm run typecheck && npm run test && npm run build

# 5. Push and open a PR.
git push -u origin feat/seat-scoring-endpoint
gh pr create --fill
```

Branches are **short-lived**: open the PR the same day you branch, and aim to merge
within 48 hours. If a branch is going to live longer than that, the change is too big —
split it. Details in [`engineering/git-workflow.md`](engineering/git-workflow.md).

---

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/), enforced in CI.

```
<type>(<scope>): <subject in imperative mood, lowercase, no trailing period>

<optional body explaining why, not what>

<optional footer: Refs #123 / Closes #123 / BREAKING CHANGE: ...>
```

Allowed types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`, `revert`.

Scopes we actually use: `films`, `theatres`, `seats`, `scoring`, `recommendations`,
`showtimes`, `booking`, `auth`, `chat`, `news`, `follows`, `ingest`, `db`, `web`,
`home`, `film-detail`, `seatmap`, `three`, `ui`, `ci`, `docs`.

Good:

```
feat(scoring): weight neighbour openness by party size
fix(showtimes): treat naive datetimes as theatre-local, not UTC
chore(ci): cache pip wheels between runs
```

Bad:

```
update stuff
fix bug
WIP
feat: changes to the scoring and also the seatmap and a migration
```

The PR **title** must also be a Conventional Commit, because we squash merge and the PR
title becomes the commit on `main`.

---

## Pull requests

Use [`templates/pull-request-template.md`](templates/pull-request-template.md) (it is
installed as the default template in both code repos).

Requirements to merge, all enforced by a branch protection ruleset on `main`:

- ✅ CI green: lint, typecheck, test, build
- ✅ One approving review
- ✅ Branch up to date with `main`
- ✅ Conversation resolved
- ✅ Linear history (squash merge only — merge commits and rebase merge are disabled)

Keep PRs under ~400 changed lines where you can. A 1,200-line PR does not get a real
review, it gets a rubber stamp, and we'd rather not pretend otherwise.

Every PR must satisfy [`engineering/definition-of-done.md`](engineering/definition-of-done.md).
Reviewers work from [`engineering/code-review.md`](engineering/code-review.md).

---

## Tests

No new behaviour without a test. Specifics live in
[`engineering/testing-strategy.md`](engineering/testing-strategy.md), but the baseline:

- **API** — `pytest`. Unit tests for `app/ml/scoring.py` and domain logic; FastAPI
  `TestClient` tests for every route; Alembic upgrade/downgrade round-trip test for
  every migration.
- **Web** — `Vitest` for components and pure logic, `Playwright` for the critical paths
  (browse → film detail → showtime → seat recommendation → booking handoff).
- **Scoring changes are special.** If you touch the weights or the geometry in
  `app/ml/scoring.py`, you must include the before/after score distribution on the
  golden fixture set in the PR description. Silent scoring drift is the single easiest
  way to make this product worse without anyone noticing.

---

## Documentation

Docs change in the same PR as the code, or in a companion PR linked from it — never
"later".

- Schema change → update [`architecture/data-model.md`](architecture/data-model.md).
- New endpoint → the OpenAPI schema is generated, but check the response shape follows
  [`engineering/api-design-guidelines.md`](engineering/api-design-guidelines.md).
- New env var → add it to `.env.example` **and**
  [`engineering/secrets-and-config.md`](engineering/secrets-and-config.md).
- Milestone scope change → update the file in [`product/milestones/`](product/milestones/).
- Anything an on-call person would need at 3am → a runbook in
  [`operations/runbooks/`](operations/runbooks/).

---

## Secrets

Never commit a secret. Not in code, not in a test fixture, not in a comment, not in a
docs example.

- Real values go in your local `.env`, which is gitignored.
- Placeholders go in `.env.example`, which is committed.
- CI and deploy secrets go in GitHub Actions secrets or the host's env config.
- If you do commit one: rotate it first, then rewrite history. Rotating first is not
  optional — the repos are public.

Full rules: [`engineering/secrets-and-config.md`](engineering/secrets-and-config.md).

---

## Reporting problems

- **Bug** → [`templates/issue-bug-report.md`](templates/issue-bug-report.md)
- **Feature idea** → [`templates/issue-feature-request.md`](templates/issue-feature-request.md)
- **Security vulnerability** → do *not* open an issue. Follow [`SECURITY.md`](SECURITY.md).
- **Conduct concern** → `conduct@goldseats.app`, see [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

---

## Licence and ownership

By contributing you agree your contribution is licensed under the MIT Licence in
[`LICENSE`](LICENSE). Don't paste in code you don't have the right to relicense, and
don't paste in code from an AI assistant that you haven't read and understood well
enough to defend in review.
