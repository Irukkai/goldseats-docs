# Git Workflow

Trunk-based development. One long-lived branch (`main`), short-lived topic branches,
squash merge, linear history. This is a locked decision — see
[ADR-0002](../architecture/adr/0002-stack-selection.md) and
[`../README.md`](../README.md#locked-technical-decisions).

Applies identically to `goldseats-web`, `goldseats-api`, and `goldseats-docs`.

---

## Why trunk-based

We are a very small team shipping a pre-launch product. Long-lived `develop` and
`release/*` branches buy you coordination benefits we don't need and cost merge pain we
can't afford. `main` is always deployable; anything unfinished hides behind a feature
flag rather than behind a branch.

---

## The model

```
main   ──●────●────●────●────●────●──>   always green, always deployable
          \        /      \    /
           ●──●──●         ●──●
        feat/film-filters   fix/tz-showtimes
```

- `main` is protected. Nobody pushes to it directly, including the founder.
- Topic branches live **less than 48 hours**. If yours is older, it's too big — split it.
- Squash merge only. One issue → one commit on `main`.
- History is linear. Merge commits and rebase-merge are disabled in repo settings.

---

## Branch naming

```
<type>/<short-kebab-description>
```

Types, and they match the Conventional Commit types:

| Prefix | Use for |
| --- | --- |
| `feat/` | New user-visible behaviour |
| `fix/` | Bug fix |
| `chore/` | Dependencies, config, CI, tooling, cleanup |
| `docs/` | Documentation only (most PRs in `goldseats-docs`) |
| `refactor/` | Internal change with no behaviour change |
| `perf/` | Performance work with a measured before/after |

Good:

```
feat/films-now-playing-endpoint
feat/seatmap-2d-golden-highlight
fix/showtime-timezone-offset
chore/bump-next-15-4
docs/adr-0005-format-ranking
```

Bad: `barun-work`, `new-feature`, `fix`, `temp`, `feat/stuff`.

Include the issue number if it helps you: `feat/42-film-filters` is fine.

---

## Day-to-day commands

```bash
# Always branch from a fresh main.
git checkout main
git pull --ff-only
git checkout -b feat/films-now-playing-endpoint

# ... work ...

git add app/api/v1/films.py tests/api/test_films.py
git commit -m "feat(films): add /v1/films/now-playing with title and language filters"

# Keep up with main by rebasing, not merging.
git fetch origin
git rebase origin/main

# Push (force-with-lease after a rebase — never plain --force).
git push -u origin feat/films-now-playing-endpoint
git push --force-with-lease

# Open the PR.
gh pr create --fill

# After merge, clean up.
git checkout main
git pull --ff-only
git branch -d feat/films-now-playing-endpoint
```

**Rebase, don't merge, to update a topic branch.** Merging `main` into your branch makes
the diff unreadable for the reviewer. Since we squash on merge, your intermediate history
doesn't survive anyway — optimise it for review, not for posterity.

---

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/), validated in CI
on the PR title (which becomes the squashed commit).

```
<type>(<scope>): <subject>

<body — why, not what. Wrap at 72.>

<footer — Refs #12 / Closes #12 / BREAKING CHANGE: ...>
```

Rules:

- Subject in imperative mood, lowercase, no trailing period, under 72 characters.
  "add", not "added" or "adds".
- Scope is the area touched. Established scopes: `films`, `theatres`, `seats`, `scoring`,
  `recommendations`, `showtimes`, `booking`, `auth`, `chat`, `news`, `follows`, `ingest`,
  `db`, `web`, `home`, `film-detail`, `seatmap`, `three`, `ui`, `ci`, `docs`.
- `BREAKING CHANGE:` in the footer for any change that breaks the API contract the web
  app depends on. See [`api-design-guidelines.md`](api-design-guidelines.md) for what
  counts.

A real example with a body:

```
fix(showtimes): interpret naive start times as theatre-local

TMDB and our seeding CSVs both hand us naive datetimes. We were storing them
as UTC, which shifted every Toronto showtime by four or five hours depending
on DST and made "now playing" wrong in the evening.

Showtimes are now normalised to UTC at ingest using the theatre's IANA
timezone, and the migration backfills existing rows.

Closes #87
```

Commits do not need to be individually green — CI runs on the PR head, and the squash
means only the final state lands on `main`.

---

## Pull requests

1. Open the PR the **same day** you branch, even if it's a draft. A draft PR is how the
   rest of the team sees what's in flight.
2. Fill in [the template](../templates/pull-request-template.md).
3. Link the issue. Every PR has one.
4. Run lint, typecheck, test, and build locally before marking ready. Using CI as your
   first test run wastes everybody's minutes.
5. Get CI green and one approval.
6. **Squash merge.** Check the squash commit message before confirming — GitHub
   sometimes appends the whole commit list into the body. Trim it to the PR description.
7. Delete the branch. Repo settings do this automatically.

### Protection ruleset on `main`

Configured identically in all three repos as part of [M0](../product/milestones/M0.md):

- Require a pull request before merging
- Require 1 approving review
- Dismiss stale approvals on new commits
- Require status checks to pass: `lint`, `typecheck`, `test`, `build`
- Require branches to be up to date before merging
- Require conversation resolution
- Require linear history
- Block force pushes
- Restrict deletions
- **No bypass list.** Admins included.

The repos are public specifically so these rulesets and Actions minutes cost nothing.

---

## Reviewing

Details in [`code-review.md`](code-review.md). Mechanically:

```bash
gh pr checkout 42          # get the branch locally
gh pr diff 42              # read the diff in the terminal
gh pr review 42 --approve  # or --request-changes / --comment
```

Review SLA is one business day. If you can't get to it, say so on the PR so the author
can find someone else.

---

## Reverting

`main` broke. Revert first, understand second.

```bash
gh pr list --state merged --limit 5    # find the offending PR
git revert -m 1 <squash-commit-sha>    # squash commits are single-parent; -m 1 is harmless
git checkout -b fix/revert-seat-scoring-weights
git commit -m "revert: seat scoring weight change (#91)"
gh pr create --fill
```

A revert PR is the one case where you may self-approve and merge with only CI green —
restoring a working `main` beats waiting for a review. Then open a follow-up issue
explaining what went wrong.

For a production incident, the revert is only half the job. Follow
[`../operations/runbooks/api-rollback.md`](../operations/runbooks/api-rollback.md) and
[`../operations/incident-response.md`](../operations/incident-response.md).

---

## Things not to do

- **Don't commit secrets.** The repos are public. If you do: rotate the credential
  *first*, then rewrite history. See
  [`secrets-and-config.md`](secrets-and-config.md).
- **Don't commit generated or large binary files.** No `.venv/`, no `node_modules/`,
  no `dist/`, no `.next/`, no `goldseats.db`, no generated dataset PNGs. The dataset
  generator's `output/` directory is gitignored — it's reproducible from a seed.
- **Don't force-push to a branch someone else is reviewing** without saying so. Use
  `--force-with-lease` and leave a comment.
- **Don't mix concerns.** A migration, a new endpoint, and a Tailwind refactor are three
  PRs.
- **Don't let a branch go stale.** Rebase daily or close it.
- **Don't `git add -A` blindly.** Run `git status` and `git diff --staged` first.

---

## Useful aliases

```bash
git config --global alias.st  "status -sb"
git config --global alias.lg  "log --oneline --graph --decorate -20"
git config --global alias.up  "pull --ff-only"
git config --global alias.pf  "push --force-with-lease"
git config --global pull.ff   only            # never accidentally create a merge commit
git config --global rebase.autosquash true
```

---

## One-time setup

`gh` must be authenticated before any push works:

```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
gh auth status
```

Full machine setup is in [`../onboarding/local-setup.md`](../onboarding/local-setup.md)
and [`../onboarding/tooling.md`](../onboarding/tooling.md).
