# Day One

Welcome to GoldSeats. The goal for today: understand what we're building, get both apps running
locally, and ship one small thing.

Not "read all the documentation". There's a lot of it and reading it cold is the wrong way to
absorb it — it's reference material you'll return to, not a textbook.

---

## What GoldSeats is, in one paragraph

Movie theatres let you pick your seat, and almost nobody knows how to pick well. GoldSeats
analyses an auditorium's actual geometry and its current availability, scores every seat with a
weighted model, and tells you which seats are genuinely worth sitting in — for one person or for a
group who need to sit together. Then it shows you the view from that seat in 3D and hands you off
to the theatre's own booking page. We never take payments. Primary market is Canada, starting with
Cineplex.

The thing we sell is **being right about seats**. Every technical decision in these docs traces
back to protecting that.

---

## Read these three, in this order

About 45 minutes total. Don't read further than this today.

1. **[`../README.md`](../README.md)** — what GoldSeats is, the three repos, the locked decisions
2. **[`../product/roadmap.md`](../product/roadmap.md)** — the M0–M9 plan, and which milestone
   we're on right now
3. **[`../CONTRIBUTING.md`](../CONTRIBUTING.md)** — the contribution loop you'll use today

Then skim two more, just enough to know they exist and what's in them:

- **[`../architecture/overview.md`](../architecture/overview.md)** — look at the system diagram
  and the request-flow sequence diagram. Those two pictures are most of the architecture.
- **[`../product/glossary.md`](../product/glossary.md)** — skim the "Words we don't use" table at
  the bottom. It'll save you a review comment on your first PR.

---

## Get it running

Follow [`local-setup.md`](local-setup.md). Budget 15 minutes.

Two things worth knowing before you start:

- **No Docker, no database server, no services.** SQLite by default. This is a hard requirement,
  not laziness — see [ADR-0002](../architecture/adr/0002-stack-selection.md).
- **Activate the virtualenv in every terminal.** `source .venv/bin/activate`. Forgetting it causes
  most day-one confusion.

Then get editor setup out of the way: [`tooling.md`](tooling.md).

---

## Use the product for fifteen minutes

Before you write any code. Seriously — this is the highest-value fifteen minutes of your first day.

1. Load http://localhost:3000. Filter by title. Filter by language. Filter by release date.
2. Open a film. Expand the timeline. Look at how the formats are ranked.
3. Pick a showtime at a seeded auditorium.
4. Ask for two seats. Look at the scores. Hover a golden seat and read the factor breakdown.
5. Now ask for six seats and watch what happens to the contiguity requirement.
6. Click through the booking handoff and notice that you land on the theatre's own site.
7. Open http://localhost:8000/docs and read the endpoints behind what you just did.

Then do it once more, keyboard-only. The seat map is fully keyboard navigable, and understanding
why that matters will shape how you build everything else here. Press `G` on the seat map.

---

## The five things that matter most

If you remember nothing else from today:

### 1. The scoring engine is the product

`app/ml/scoring.py`. Five weighted factors: horizontal centre 25%, vertical position 25%, view
angle 20% (THX-inspired ~36° target), neighbour openness 15%, row quality 15%.

It's pure, synchronous, has no database access, and is tested against golden fixtures that fail
the build on any drift. If you change a weight, the PR needs the before/after distribution. Silent
scoring drift is the easiest way to make the whole product worse without anyone noticing.

### 2. We never scrape

Theatre data is hand-authored from public seating charts and committed to git. This is a legal and
commercial position, not a capacity limitation — our monetisation path runs through partnerships
with the chains we'd be scraping. A PR adding a scraper gets closed.
[ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md),
[`../legal/data-sources-policy.md`](../legal/data-sources-policy.md).

### 3. SQLite locally, Postgres in production

Which means a list of Postgres features are off-limits everywhere: `JSONB` operators, `ARRAY`,
Postgres `ENUM`, `ILIKE`, `tsvector`. A CI parity job runs the tests against both engines and it
never gets skipped.
[`../engineering/database-conventions.md`](../engineering/database-conventions.md).

### 4. The chatbot can't invent a seat

Not because we asked it nicely. The model has no database, no filesystem, and no outbound network
— only three tool calls into our own endpoints. And a verification step checks every seat in every
reply against the tool results before the user sees it. An unverifiable reply is blocked, not
corrected. [ADR-0004](../architecture/adr/0004-self-hosted-llm.md).

### 5. We're honest about data quality

Seat availability is currently **simulated**, stored with `confidence='low'`, and the UI says so.
Presenting simulated availability as real would destroy the only thing we have. If you ever find
yourself writing code that hides a data-quality caveat, stop.

---

## Ship something today

Find an issue labelled `good-first-issue`, or fix something you noticed in the last fifteen
minutes. A typo in a docs file counts — the point is to exercise the whole loop while it's small.

```bash
git checkout main && git pull --ff-only
git checkout -b fix/whatever-you-found

# ... change something ...

# Run what CI runs, before pushing
ruff check . && ruff format --check . && mypy app && pytest    # api
npm run lint && npm run typecheck && npm run test && npm run build   # web

git commit -m "fix(seatmap): correct row label on the legend"
git push -u origin fix/whatever-you-found
gh pr create --fill
```

Then squash merge it. You've now used branch protection, CI, Conventional Commits, and the PR
template — which is most of the process.

---

## Where to look things up

Don't memorise this. Just know the shape of it.

| Question | Where |
| --- | --- |
| How do I write TypeScript here? | [`../engineering/code-conventions-typescript.md`](../engineering/code-conventions-typescript.md) |
| How do I write Python here? | [`../engineering/code-conventions-python.md`](../engineering/code-conventions-python.md) |
| What shape should my endpoint be? | [`../engineering/api-design-guidelines.md`](../engineering/api-design-guidelines.md) |
| What's in the database? | [`../architecture/data-model.md`](../architecture/data-model.md) |
| How do I write a migration? | [`../engineering/database-conventions.md`](../engineering/database-conventions.md) |
| What do I test, and how? | [`../engineering/testing-strategy.md`](../engineering/testing-strategy.md) |
| When is my work done? | [`../engineering/definition-of-done.md`](../engineering/definition-of-done.md) |
| Why is it built this way? | [`../architecture/adr/`](../architecture/adr/) |
| What are we building next? | [`../product/milestones/`](../product/milestones/) |
| Something's on fire | [`../operations/runbooks/README.md`](../operations/runbooks/README.md) |
| Can we use this data source? | [`../legal/data-sources-policy.md`](../legal/data-sources-policy.md) |

---

## Things that will trip you up

Collected from real experience rather than imagined:

- **Row `A` is `row_index` 0.** Labels are 1-based and what's printed on the seat; geometry is
  0-based. Both are stored deliberately. This seam has already caused bugs.
- **Timezones.** Every datetime is tz-aware UTC. Showtimes store `starts_at` (UTC),
  `starts_at_local`, and `timezone` — redundantly, on purpose, because treating a naive datetime
  as UTC shifted every Toronto evening showtime by five hours once already.
- **`ILIKE` doesn't exist in SQLite.** Use `func.lower(col).like(...)`. CI will catch it; better
  if you don't need CI to.
- **Never cache a seat map.** Availability and recommendations are always `no-store`. The worst
  bug this product can have is showing someone a seat that's already sold.
- **`#FFD700` appears exactly once in the web repo**, in `tailwind.config.ts`. Use the token.
- **Gold-vs-grey is not enough** to distinguish a recommended seat. Every state carries two
  signals. [`../engineering/accessibility.md`](../engineering/accessibility.md).
- **"Reserve in person" does not hold the seat.** Copy that implies otherwise is a trust bug, not
  a wording preference.
- **`os.environ` lives only in `app/config.py`**, and `process.env` only in `src/lib/env.ts`.

---

## By end of day

- [ ] Read the three documents
- [ ] Both apps running locally with seeded data
- [ ] Used the product end to end, including keyboard-only
- [ ] Editor configured, hooks installed
- [ ] One PR opened, reviewed, and squash merged
- [ ] You know where to look things up

If any of that took much longer than expected because a document was wrong or a step was missing:
**fix it.** A new person's confusion is the most accurate signal we ever get about these docs, and
it decays within a week.
