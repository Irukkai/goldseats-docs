# ADR-0001 — Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-02-09
- **Deciders:** Founder
- **Milestone:** [M0](../../product/milestones/M0.md)
- **Supersedes:** none

---

## Context

GoldSeats begins with an unusual amount of technical decision-making already done. Before a
line of production code exists, we have settled on: three public repos in a GitHub org,
Next.js 16 with Tailwind, FastAPI with SQLAlchemy 2.0, SQLite locally and Postgres in
production, manually seeded theatre data with no scraping, a self-hosted LLM with tool
calling, Redis, react-three-fiber, TMDB, and trunk-based development.

Each of those has a real reason behind it. Several have reasons that are non-obvious and
will look like mistakes to anyone who doesn't know the constraint. Two examples that will
definitely come up:

- SQLite locally looks like an amateur choice until you know that Docker is not installed on
  the development machine and that a local setup requiring a container runtime is a local
  setup that doesn't get run.
- Self-hosting an open-weight model looks like needless difficulty in 2026, when a hosted
  API would be one HTTP call.

Right now that reasoning lives in one person's head and in a handoff document. That fails in
three predictable ways:

1. **A new contributor re-litigates a settled decision** because the reasoning is invisible,
   and we spend a week re-deciding something we already decided.
2. **We change a decision without noticing we're changing it**, because nobody knew a
   constraint was load-bearing. Someone adds a `JSONB` query, it works in staging, and local
   development silently breaks for everyone.
3. **Future-us can't tell a deliberate choice from an accident.** Six months on, was
   `snake_case` in JSON responses a considered decision or did the first endpoint just do
   that? Without a record, everything looks accidental and everything feels changeable.

The alternative isn't "no documentation" — it's documentation in the wrong shape. Comments
rot, commit messages are unfindable, a wiki drifts from the code, and chat history is a
write-only medium.

## Decision

We record every architecturally significant decision as a numbered Architecture Decision
Record in `architecture/adr/`, using the format in [`template.md`](template.md).

Mechanics:

- Filename `NNNN-kebab-case-title.md`, monotonically numbered from `0001`.
- Every ADR carries a **Status**: Proposed, Accepted, Rejected, Deprecated, or Superseded.
- **ADRs are immutable once accepted.** We fix typos and broken links; we do not rewrite
  history. Changing a decision means writing a new ADR that supersedes the old one, and
  editing the old one's status to point at it.
- The index lives in [`../overview.md`](../overview.md#decision-records).
- ADRs are reviewed like code: a PR, a reviewer, a merge. The PR discussion is where the
  argument happens.

**What counts as architecturally significant** — write an ADR if the change:

- adds, removes, or replaces a technology, framework, or service
- changes how components communicate, or adds a component
- constrains future choices in a way that would be expensive to undo
- introduces a new external data source ([`../../legal/data-sources-policy.md`](../../legal/data-sources-policy.md) also applies)
- changes a cross-cutting convention: auth, caching, error handling, naming
- has a genuine trade-off where a reasonable engineer would pick differently

**What doesn't:** adding an endpoint that follows the existing conventions, a bug fix, a
dependency bump, a refactor with no interface change, a copy change. Those are PRs.

When it's ambiguous, prefer writing one. An unnecessary ADR costs thirty minutes; a missing
one costs a week of argument in six months.

### Relationship to RFCs

An [RFC](../../templates/rfc-template.md) is for exploring a question before it's decided,
when the answer isn't known and the design needs discussion. An ADR is for recording the
answer. Some decisions get an RFC then an ADR; most straightforward ones go straight to an
ADR. An RFC that concludes in a decision should end with a link to the resulting ADR.

## Options considered

### Option A — Numbered ADRs in the docs repo (chosen)

Michael Nygard's format, lightly extended with "Revisit when".

- **Pros:** Version-controlled alongside the docs they explain. Reviewable via PR.
  Greppable. No tooling. Numbered ordering gives a readable chronology of how our thinking
  developed. A very widely understood convention, so a new hire needs no explanation.
- **Cons:** Requires discipline — an unwritten ADR is the default failure mode. Lives in a
  different repo from the code it constrains.

### Option B — Decisions in code comments and docstrings

- **Pros:** Maximum proximity to the code. Impossible to miss while editing.
- **Cons:** No room for the options we rejected or the constraints behind the choice, which
  is the part with lasting value. Cross-cutting decisions have no single home — where does
  "trunk-based development" go? Deleting the code deletes the reasoning.
- **Why not:** A comment can say *what*, but an ADR's value is in *why not the alternatives*,
  and that doesn't fit in a comment.

### Option C — A GitHub wiki or Notion page

- **Pros:** Nice editing experience, easy linking, non-engineers can contribute.
- **Cons:** Not in git, so not reviewable via PR and not versioned with the code. Drifts
  immediately. Requires an account and a subscription. Outside the repo means outside the
  workflow, and outside the workflow means unmaintained.
- **Why not:** Docs that aren't in the repo don't get updated in the PR that changes the
  behaviour.

### Option D — GitHub Issues labelled `decision`

- **Pros:** Zero setup, discussion threading built in, already part of the workflow.
- **Cons:** Issues are for open work; a closed issue reads as finished, not as standing
  policy. Discoverability is poor — nobody browses closed issues to learn how the system
  works. No stable ordering.
- **Why not:** Wrong mental model. A decision isn't a task that got completed.

### Option E — Don't record decisions

- **Pros:** No overhead.
- **Cons:** Everything in the Context section above.
- **Why not:** The cost isn't hypothetical. Several of our decisions depend on constraints
  that are invisible from the code, and at least one (SQLite locally) will read as an error
  to anyone who doesn't know why.

## Consequences

### What this makes easier

- Onboarding. [`../../onboarding/day-one.md`](../../onboarding/day-one.md) can point at the
  ADR index instead of trying to explain everything in prose.
- Saying no. "That's ADR-0003, here's the reasoning, and here's what would change my mind"
  is a much better conversation than re-arguing from scratch.
- Changing our minds *deliberately*. A superseding ADR makes the reversal explicit and
  records what we learned, which is more useful than the original decision.
- Code review. A reviewer can check a change against a written decision instead of a
  remembered one.

### What this makes harder

- Every significant change now carries a writing task. That's real friction, and it is
  partly the point — a decision worth an ADR is worth thinking through in prose.
- Two places to keep consistent: the ADR and the docs it informs. When ADR-0002 is
  superseded, [`../overview.md`](../overview.md) and
  [`../../engineering/database-conventions.md`](../../engineering/database-conventions.md)
  need updating too. The superseding PR is responsible for that.

### What this commits us to

- Retroactively writing ADRs for the decisions already locked in. That's ADR-0002 through
  ADR-0004, all part of [M0](../../product/milestones/M0.md). They're dated to when they
  were actually decided, not when they were written down — pretending otherwise would make
  the chronology useless.
- Keeping the index in [`../overview.md`](../overview.md) current.
- Reviewing ADR PRs with the same seriousness as code PRs.

## Revisit when

Essentially never — this is a process decision with almost no cost and no scaling limit. We
would revisit only if the team grew large enough to need a heavier RFC process with formal
review stages, at which point ADRs would remain the record and the RFC process would change
around them.

## References

- Michael Nygard, ["Documenting Architecture Decisions"](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) (2011)
- [`template.md`](template.md)
- [`../../templates/rfc-template.md`](../../templates/rfc-template.md)
- [ADR-0002 — Stack selection](0002-stack-selection.md)
