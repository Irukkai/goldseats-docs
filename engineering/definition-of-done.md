# Definition of Done

An issue is not done when the code works. It's done when it's shipped, verified,
observable, documented, and safe to roll back.

This is the checklist. It's referenced by
[`../templates/pull-request-template.md`](../templates/pull-request-template.md) and by
every milestone file in [`../product/milestones/`](../product/milestones/).

---

## Universal — every issue

- [ ] The acceptance criteria on the issue are met. All of them, not most.
- [ ] Code follows [TypeScript](code-conventions-typescript.md) or
      [Python](code-conventions-python.md) conventions.
- [ ] Lint, typecheck, test, and build pass locally **and** in CI.
- [ ] Tests exist for the new behaviour, and they'd fail without the change.
- [ ] Reviewed and approved by one other person, per [`code-review.md`](code-review.md).
      (Or self-reviewed against the degraded solo process in that document.)
- [ ] Squash merged to `main` with a Conventional Commit title.
- [ ] Deployed to staging and manually exercised there. Not just "CI went green".
- [ ] No new secrets in git. No new `os.environ` / `process.env` reads outside
      `app/config.py` / `src/lib/env.ts`.
- [ ] Docs updated in the same PR or a linked companion PR.
- [ ] Branch deleted, issue closed, milestone correct.

---

## Backend — `goldseats-api`

- [ ] Endpoint follows [`api-design-guidelines.md`](api-design-guidelines.md): plural
      nouns, `/v1/` prefix, envelope shape, problem-detail errors.
- [ ] Appears correctly in the generated OpenAPI schema at `/docs`, with a summary,
      a description, and response models on every status code it returns.
- [ ] Pydantic schemas for request and response. No raw dicts returned.
- [ ] List endpoints paginated with a deterministic `order_by`.
- [ ] Every relationship the response serialises is eager-loaded. Verify by counting
      queries in the test — an N+1 that passes tests still breaks Lighthouse.
- [ ] Auth and authorisation correct. A test proves user A cannot read user B's data.
- [ ] Structured log lines at the boundaries, with no PII and no tokens.
      ([`observability.md`](observability.md))
- [ ] Errors raised as domain exceptions from `domain/`, never `HTTPException`.
- [ ] Anything CPU-bound is off the event loop via `anyio.to_thread.run_sync`.
- [ ] `.env.example` updated for any new setting, with a comment.

### Additionally, for a schema change

- [ ] Alembic revision generated, reviewed by hand, and **edited** — autogenerate output
      is a draft, not a migration.
- [ ] `downgrade()` implemented and tested. The round-trip test in
      `tests/db/test_migrations.py` passes.
- [ ] Backwards compatible with currently deployed code — expand now, contract in a later
      release. ([`branching-and-releases.md`](branching-and-releases.md#migrations-and-release-ordering))
- [ ] Runs on both SQLite and Postgres. The parity CI job passes.
      ([`database-conventions.md`](database-conventions.md))
- [ ] [`../architecture/data-model.md`](../architecture/data-model.md) updated: columns,
      types, keys, relationships, and the ER diagram.
- [ ] Indexes considered for every new foreign key and every column a filter targets.

### Additionally, for a scoring change

- [ ] Before/after score distribution across all golden fixture auditoriums is in the PR
      description.
- [ ] Golden snapshots regenerated **and** the change justified in words, not just
      committed.
- [ ] Each of the five factors still independently unit tested and normalised to 0–1.
- [ ] `SCORING_WEIGHTS` still sums to 1.0 and is still the single source of the weights.
- [ ] `SeatScore.factors` still populated — the chatbot and the seat map both depend on it.
- [ ] Group scenarios still guarantee contiguous same-row blocks.
- [ ] The scoring benchmark still completes a full IMAX layout in under 50ms.

---

## Frontend — `goldseats-web`

- [ ] Server component by default. Every `'use client'` is justified.
- [ ] Explicit cache policy on every `fetch`. Seat availability and anything user-specific
      is `no-store`.
- [ ] `loading.tsx` skeleton and `error.tsx` for every route that awaits data.
- [ ] Responsive and verified at 375px, 768px, and 1440px. Screenshots in the PR.
- [ ] Tailwind tokens only. No raw hex values — `#FFD700` lives in `tailwind.config.ts`
      and nowhere else.
- [ ] Keyboard navigable end to end, with a visible focus ring.
- [ ] Meets [`accessibility.md`](accessibility.md): correct roles and accessible names,
      no colour-only information, axe clean.
- [ ] Within [`performance-budgets.md`](performance-budgets.md): Lighthouse above 90,
      LCP under 2.5s, no three.js in the initial bundle.
- [ ] Types generated from the API's OpenAPI schema, not hand-written.
- [ ] Error states show human copy and quietly surface the `request_id`.
- [ ] Empty states designed. "No films match these filters" is a design problem, not a
      blank div.

### Additionally, for the 3D seat view

- [ ] Loaded via `next/dynamic` with `ssr: false`.
- [ ] 60fps on a mid-range desktop; measured, not assumed.
- [ ] Quality fallback verified on a real low-end mobile device.
- [ ] A non-3D path exists for users who can't or won't render it.
- [ ] Screen rendered at true relative size — the whole point is that the viewing angle is
      honest. A flattering-but-wrong preview is a product bug.

### Additionally, for the chatbot

- [ ] Every seat the model mentions is verified against the scoring endpoint's response
      before it reaches the user. A reply naming a seat we didn't return is blocked, not
      corrected.
- [ ] Prompt injection cases in the evaluation fixtures pass.
- [ ] Tool-call traces logged with the conversation for debugging.
- [ ] Graceful degradation when the model is unavailable — the user still gets the
      deterministic seat map.

---

## Milestone-level done

A milestone closes only when all of these hold, on top of every issue in it being done:

- [ ] The **done-when criterion** in the milestone file is demonstrably true, shown to
      someone, not asserted.
- [ ] All issues on the GitHub Milestone are closed or explicitly moved, with a note
      saying where they went.
- [ ] Feature flags introduced by the milestone are either fully on and deleted, or
      documented as intentionally still off with the reason.
- [ ] Docs reflect reality: architecture, data model, ADRs, and the milestone file itself.
- [ ] Any new operational surface has a runbook in
      [`../operations/runbooks/`](../operations/runbooks/README.md).
- [ ] New dashboards and alerts exist for anything a user now depends on.
      ([`observability.md`](observability.md))
- [ ] Rollback verified, not assumed. Actually roll staging back and forward once.
- [ ] [`../product/roadmap.md`](../product/roadmap.md) updated with what actually shipped
      versus what was planned, including the parts that were cut.

---

## What "done" explicitly does not require

Stated so nobody gold-plates and nobody feels guilty:

- 100% test coverage. See [`testing-strategy.md`](testing-strategy.md).
- Zero technical debt. It requires that debt is *written down* as an issue.
- Perfect design. It requires the design be consistent with the tokens and accessible.
- Solving the general case. Solve the case in the acceptance criteria; a comment noting a
  known limitation is fine and useful.
- Support for browsers we don't target. See
  [`performance-budgets.md`](performance-budgets.md) for the matrix.
