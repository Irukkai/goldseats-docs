<!--
Install as .github/pull_request_template.md in goldseats-web, goldseats-api, and goldseats-docs.

PR TITLE must be a Conventional Commit — it becomes the squash commit on main.
  feat(recommendations): add POST /v1/recommendations
  fix(showtimes): treat naive datetimes as theatre-local
Delete sections that genuinely don't apply. Don't delete them to avoid answering.
-->

## What

<!-- One or two sentences. What does this change do, from a user's or caller's perspective? -->

Closes #

## Why

<!-- Why is this the right change? Link the issue, and add anything the issue doesn't say.
     If this implements an ADR or PRD, link it. -->

## How

<!-- Only if the approach isn't obvious from the diff. Call out anything a reviewer would
     otherwise have to reverse-engineer: a non-obvious trade-off, a deliberate omission,
     a thing you tried first that didn't work. -->

## How I verified this

<!-- Not "CI is green". What did you actually do? -->

- [ ] Ran the full check suite locally
- [ ] Manually exercised the change
- [ ] Tested the failure path, not just the happy path

## Screenshots / recording

<!-- Required for any UI change. Both mobile (375px) and desktop (1440px). -->

| Before | After |
| --- | --- |
|  |  |

---

## Definition of done

Full checklist: [`engineering/definition-of-done.md`](https://github.com/Irukkai/goldseats-docs/blob/main/engineering/definition-of-done.md)

- [ ] Acceptance criteria on the issue are met — all of them
- [ ] `lint`, `typecheck`, `test`, `build` pass **locally**
- [ ] Tests added that would fail without this change
- [ ] Docs updated in this PR or a linked companion PR
- [ ] No secrets in the diff. No new `os.environ` / `process.env` outside `app/config.py` / `src/lib/env.ts`
- [ ] `.env.example` updated if a setting was added
- [ ] Self-reviewed the diff on GitHub before requesting review
- [ ] Diff is under ~400 lines, or I've explained why not below

---

## Backend

<!-- Delete if this is web-only. -->

- [ ] Follows [api-design-guidelines](https://github.com/Irukkai/goldseats-docs/blob/main/engineering/api-design-guidelines.md): plural nouns, `/v1/`, envelope shape, problem-detail errors
- [ ] OpenAPI entry has a summary, description, and response models for every status code
- [ ] Pydantic schemas for request and response — no raw dicts returned
- [ ] List endpoints paginated with a deterministic `order_by` ending in `id`
- [ ] Everything serialised is eager-loaded; query-count test proves no N+1
- [ ] Auth and authorisation correct; a test proves user A can't read user B's data
- [ ] Domain exceptions raised from `domain/`, never `HTTPException`
- [ ] CPU-bound work off the event loop via `anyio.to_thread.run_sync`
- [ ] Structured logs at the boundaries — no PII, no tokens

### Schema change

<!-- Delete if no migration. -->

- [ ] Alembic revision reviewed **by hand** — autogenerate output is a draft
- [ ] `downgrade()` implemented; round-trip test passes
- [ ] Backwards compatible with deployed code. Expand/contract step: **expand / switch / contract**
- [ ] Runs on **both** SQLite and Postgres; parity CI job green
- [ ] [`architecture/data-model.md`](https://github.com/Irukkai/goldseats-docs/blob/main/architecture/data-model.md) updated, including the ER diagram
- [ ] Indexes considered for every new FK and every filtered column
- [ ] One migration in this PR, not two

### Scoring change

<!-- Delete unless you touched app/ml/scoring.py. -->

**Before/after score distribution on the golden fixtures:**

```
<paste it. A weight change without this gets request-changes automatically.>
```

- [ ] Golden snapshots regenerated **and** the change justified in words above
- [ ] Each of the five factors still independently tested and normalised 0–1
- [ ] `SCORING_WEIGHTS` still sums to 1.0 and is still the single source
- [ ] `SeatScore.factors` still populated — the chatbot and seat map depend on it
- [ ] Group scenarios still return contiguous same-row blocks
- [ ] `SCORING_VERSION` bumped
- [ ] Benchmark still under 50ms on the largest IMAX layout

---

## Frontend

<!-- Delete if this is api-only. -->

- [ ] Server component by default; every `'use client'` is justified
- [ ] Explicit cache policy on every `fetch`. **Seat availability and recommendations are `no-store`.**
- [ ] `loading.tsx` and `error.tsx` on every route that awaits data
- [ ] Verified at 375px, 768px, 1440px — screenshots above
- [ ] Tailwind tokens only. No raw hex values.
- [ ] Keyboard navigable with a visible focus ring
- [ ] No information conveyed by colour alone
- [ ] axe clean; meets [accessibility](https://github.com/Irukkai/goldseats-docs/blob/main/engineering/accessibility.md)
- [ ] Within [performance budgets](https://github.com/Irukkai/goldseats-docs/blob/main/engineering/performance-budgets.md); Lighthouse ≥ 90
- [ ] API types generated, not hand-written
- [ ] Empty states designed
- [ ] Error copy written for the user, with the `request_id` quietly surfaced

---

## Risk

**Breaking change for the other repo?**
- [ ] No
- [ ] Yes — `BREAKING CHANGE:` footer present, and the companion PR is linked: #

**Rollback:** <!-- Redeploy previous tag / flip flag `FEATURE_X` / revert this PR -->

**Feature flag:** <!-- name and default, or "none" -->

**What could this break that isn't obvious?**

<!-- The most useful thing you can write for a reviewer. What worried you while building it? -->

---

## New dependency

<!-- Delete if none. -->

| Package | Version | Size | Why not write it ourselves? |
| --- | --- | --- | --- |
|  |  |  |  |

---

## New data source

<!-- Delete if none. STOP and read legal/data-sources-policy.md before filling this in.
     We do not scrape. A PR adding a scraper is closed, not reviewed. -->

- [ ] Terms of service read; the relevant clause quoted below
- [ ] `robots.txt` checked and respected
- [ ] Licence confirmed
- [ ] Attribution implemented **in this PR**
- [ ] Added to the source register in [`legal/data-sources-policy.md`](https://github.com/Irukkai/goldseats-docs/blob/main/legal/data-sources-policy.md)
- [ ] ADR written if material

---

## Notes for the reviewer

<!-- Where should they look hardest? What are you unsure about? Anything you'd like a
     second opinion on rather than an approval? -->
