# Branching and Releases

How code gets from a branch to production, and how we label what's out there. The
day-to-day git mechanics are in [`git-workflow.md`](git-workflow.md); the deploy
mechanics are in [`../operations/release-process.md`](../operations/release-process.md).
This document is the branching and versioning policy that sits between them.

---

## Branch inventory

There are exactly two kinds of branch. Anything else is a mistake.

| Branch | Lifetime | Protected | Deploys to |
| --- | --- | --- | --- |
| `main` | Forever | Yes | Staging on every merge, production on every tag |
| `feat/*`, `fix/*`, `chore/*`, `docs/*`, `refactor/*`, `perf/*` | < 48 hours | No | Ephemeral preview (web only) |

We do **not** have:

- a `develop` branch — `main` is the integration branch
- `release/*` branches — we cut tags from `main`
- `hotfix/*` branches — a hotfix is a `fix/` branch that gets expedited review
- long-lived feature branches — long features hide behind flags, not branches

---

## Feature flags instead of branches

If a feature needs more than two days of work, it merges to `main` incomplete but
inert, behind a flag. This is how [M6](../product/milestones/M6.md)'s 3D view and
[M7](../product/milestones/M7.md)'s chatbot land without a month-long branch.

API side, flags are plain settings so they're visible in one file:

```python
# app/config.py
class Settings(BaseSettings):
    feature_chat_enabled: bool = False
    feature_3d_seat_view_enabled: bool = False
    feature_in_person_reservation_enabled: bool = False
```

```python
# app/api/v1/chat.py
@router.post("/chat/messages")
async def post_message(...) -> ChatReply:
    if not settings.feature_chat_enabled:
        raise NotFoundError("endpoint", "/v1/chat/messages")
    ...
```

Web side, flags are build-time public env vars, surfaced once and threaded down as props
so components stay testable:

```ts
// src/lib/flags.ts
export const flags = {
  chat: process.env.NEXT_PUBLIC_FEATURE_CHAT === 'true',
  seatView3D: process.env.NEXT_PUBLIC_FEATURE_3D === 'true',
} as const;
```

Rules for flags:

- Every flag is registered in `.env.example` with a comment naming the milestone that
  owns it.
- A flag defaults to **off** and ships off. You turn it on in staging first.
- A flag is deleted in the same milestone it goes fully live. Stale flags are worse than
  no flags — they're untested code paths. The M-closing checklist includes flag cleanup.
- Flags are for incomplete work, not for configuration. Anything a user or operator
  should control long-term is a setting, not a flag.

---

## Versioning

We version the two deployable components independently, both with **SemVer-shaped tags
driven by Conventional Commits**.

```
goldseats-api   v0.4.1
goldseats-web   v0.6.0
```

Because the product is pre-launch, both stay in `0.y.z`. That's deliberate: it signals
that the public API contract is not yet frozen. The `1.0.0` of each is cut at
[M9](../product/milestones/M9.md) launch.

| Commit types in the range | Bump |
| --- | --- |
| `fix:`, `perf:`, `refactor:`, `chore:`, `docs:` | patch — `0.4.1` → `0.4.2` |
| any `feat:` | minor — `0.4.1` → `0.5.0` |
| any `BREAKING CHANGE:` footer | minor while in `0.y.z`, major after `1.0.0` |

`goldseats-docs` is **not versioned**. It's a living document set; `main` is the only
truth. Historical states are recoverable from git history.

### The HTTP API version is not the release version

`/v1/` in the URL is the *contract* version and it changes almost never. `v0.4.1` is the
*deployment* version and changes several times a week. Don't conflate them. Contract
evolution rules are in [`api-design-guidelines.md`](api-design-guidelines.md).

---

## Cutting a release

Releases are cut from `main`, never from a branch, and only when `main` is green.

```bash
# In goldseats-api, on a fresh main.
git checkout main && git pull --ff-only

# Sanity: what's going out?
git log --oneline $(git describe --tags --abbrev=0)..HEAD

# Everything below is what `npm run release` / `make release` wraps.
# 1. Update CHANGELOG.md from Conventional Commits.
# 2. Tag.
git tag -a v0.5.0 -m "release: v0.5.0 — film catalog filters, TMDB backfill"
git push origin v0.5.0
```

Pushing the tag is the trigger. The `release` GitHub Actions workflow then:

1. Re-runs the full check suite against the tagged commit
2. Builds and publishes the container image / static bundle
3. Deploys to staging and runs smoke tests
4. Waits for a manual approval gate
5. Deploys to production
6. Creates the GitHub Release with generated notes

Steps 3–6 are specified in
[`../operations/release-process.md`](../operations/release-process.md).

Tags are **immutable**. Never move or delete one. If a tag was cut wrong, cut the next
patch version.

---

## Changelogs

`CHANGELOG.md` in each code repo, [Keep a Changelog](https://keepachangelog.com) format,
generated from Conventional Commits and then edited by a human for readability. The
generated version is a starting point, not the deliverable — a changelog nobody can read
isn't serving anyone.

```markdown
## [0.5.0] - 2026-04-14

### Added
- `GET /v1/films/now-playing` and `/v1/films/upcoming` with title, release-date, and
  language filters (#42, #47)
- TMDB ingestion job backfilling `films`, `film_releases`, and language metadata (#39)

### Fixed
- Showtime start times are normalised to UTC using the theatre's IANA timezone, fixing
  evening "now playing" results for Toronto theatres (#87)

### Changed
- `GET /v1/films/{id}` now embeds `releases` by default; pass `?embed=` to opt out (#51)
```

Write entries for the person reading them — usually future-you debugging a regression,
occasionally a user wondering what changed. "Bump deps" is not an entry; group the
dependency noise under a single line or leave it out.

---

## Release cadence

| Environment | Cadence | Gate |
| --- | --- | --- |
| Preview (web PRs) | Every push to a topic branch | CI green |
| Staging | Every merge to `main`, automatic | CI green |
| Production | On demand, tag-triggered. Expect 1–3 per week. | Manual approval + green staging smoke tests |

We don't ship to production on a Friday afternoon unless it's fixing something that's
already broken. Not superstition — it's about who's around to notice the graphs.

---

## Hotfixes

A production-breaking bug is still a normal `fix/` branch. What changes is the urgency,
not the process:

1. Declare the incident if users are affected —
   [`../operations/incident-response.md`](../operations/incident-response.md).
2. **Consider rolling back first.** Re-deploying the previous tag is usually faster and
   always safer than a forward fix written under pressure. See
   [`../operations/runbooks/api-rollback.md`](../operations/runbooks/api-rollback.md).
3. If rolling forward: branch `fix/` off `main`, minimal diff, CI green, expedited review
   (a reviewer should drop what they're doing), squash merge.
4. Tag a patch release immediately and push it.
5. Write the post-incident note within 48 hours.

We never patch a previous tag. There is no `v0.4.x` maintenance line — production always
runs the tip of `main`'s latest tag, and the fix goes forward.

---

## Migrations and release ordering

Schema changes constrain deploy order, and getting this wrong is the most likely way to
cause a self-inflicted outage.

Every migration must be **backwards compatible with the currently deployed code**,
because during a deploy both versions run at once and because a rollback must not require
a database restore.

The expand/contract pattern, across two releases:

| Release | Migration | Code |
| --- | --- | --- |
| `v0.6.0` — expand | Add nullable `showtimes.language_code`; backfill | Writes both old and new columns; reads old |
| `v0.7.0` — switch | none | Reads new column |
| `v0.8.0` — contract | Drop the old column | Only knows the new column |

What this rules out in a single release: renaming a column, dropping a column still read
by deployed code, adding a `NOT NULL` column with no default, or tightening a constraint
that existing rows violate.

Full rules, including the Alembic downgrade requirement and SQLite/Postgres parity, are in
[`database-conventions.md`](database-conventions.md).

---

## Rollback

Every production release must be rollback-able without a database restore. That's a
[definition-of-done](definition-of-done.md) item, not an aspiration, and the documented
rollback path is a [M9](../product/milestones/M9.md) launch gate.

- **API** — redeploy the previous tag's image. Migrations were expand-only, so the schema
  still satisfies it.
- **Web** — the host's instant rollback to the previous deploy.
- **Both** — if a migration genuinely has to be reversed,
  `alembic downgrade -1` against a *tested* downgrade, which is why every migration ships
  one. If there's data loss on the table,
  [`../operations/runbooks/database-restore.md`](../operations/runbooks/database-restore.md)
  takes over.
