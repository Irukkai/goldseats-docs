# RFC-NNNN — <Title>

- **Status:** Draft | In review | Accepted | Rejected | Withdrawn | Implemented
- **Author:** name
- **Date:** 2026-MM-DD
- **Reviewers:** names
- **Milestone:** the milestone this belongs to, linked — e.g. [M5](../product/milestones/M5.md)
- **Resulting ADR:** link it once written, or "none yet"

---

## When to write an RFC instead of something else

| Document | Purpose |
| --- | --- |
| **RFC** | Explore a technical question **before** it's decided. The answer isn't known and the design needs discussion. |
| **[ADR](../architecture/adr/template.md)** | Record a decision **after** it's made, with the options rejected. |
| **[PRD](../product/prd/template.md)** | Specify what to build for users, and why. |
| **Issue** | A task. |

Many decisions skip the RFC and go straight to an ADR — if you already know the answer and just
need to record it, write the ADR. An RFC is for when you genuinely don't know, or when the design
is complex enough that writing it down is how you'll find the problems.

Every RFC that concludes should end with either a link to the resulting ADR, or a note saying we
decided not to do it.

---

## Summary

Three sentences. What's the problem, and what are you proposing? Someone should be able to read
just this and know whether to keep going.

## Motivation

What's wrong today. Be concrete — a specific incident, a measured number, a user complaint, a
piece of code that's genuinely painful to change.

Include the cost of doing nothing. Sometimes that's the right answer and it should be visible.

## Constraints

What can't change, and why. For GoldSeats, the ones that most often bind:

- **SQLite parity** — local development must work with no Docker and no services. Rules out
  `JSONB` querying, `ARRAY`, Postgres `ENUM`, `ILIKE`, `tsvector`. See
  [ADR-0002](../architecture/adr/0002-stack-selection.md).
- **No scraping** — [ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md).
- **No payment processing** — keeps us out of PCI scope permanently.
- **The scoring engine stays pure** — no I/O, no SQLAlchemy, testable against fixtures.
- **The chatbot can't reach data except through tool calls** —
  [ADR-0004](../architecture/adr/0004-self-hosted-llm.md).
- **One engineer.** Operational complexity is expensive in a way that doesn't show up in a design
  doc.
- **Performance budgets** — [`../engineering/performance-budgets.md`](../engineering/performance-budgets.md).

If your proposal needs one of these to change, say so explicitly and early. That's a much bigger
conversation than the rest of the RFC.

## Proposal

The design. Be specific enough to implement from: module names, file paths, function signatures,
endpoint shapes, table definitions.

### Interfaces

```python
# or TypeScript. Real signatures, not pseudocode.
```

### Data model changes

Reference [`../architecture/data-model.md`](../architecture/data-model.md) and note the
expand/contract plan if a schema change is involved.

### Diagram

```mermaid
flowchart LR
  a["Component A"] --> b["Component B"]
```

## Alternatives considered

At least two, and at least one of them should be a serious contender rather than a strawman.

### Alternative 1 — <name>

- **Pros:**
- **Cons:**
- **Why not:**

### Alternative 2 — do nothing

Always include this one. What breaks if we don't? Sometimes nothing, and finding that out is the
RFC doing its job.

## Trade-offs

What gets worse. An RFC with no downsides hasn't been thought through, and reviewers will
(correctly) distrust it.

| | Better | Worse |
| --- | --- | --- |
| Performance | | |
| Complexity | | |
| Operational load | | |
| Testability | | |
| Time to build | | |

## Impact

- **Repos affected:** `goldseats-api` / `goldseats-web` / `goldseats-docs`
- **Breaking API change?** If yes, the deprecation plan and the coordinated web PR
- **Migration needed?** Expand/contract steps
- **Performance budgets touched?** Which ones, and the plan to stay inside them
- **New dependency?** Name it, size it, and say why not build it
- **New personal data?** Then [`../legal/privacy-policy.md`](../legal/privacy-policy.md) changes too
- **New external source?** Then [`../legal/data-sources-policy.md`](../legal/data-sources-policy.md)
  applies and this probably needs an ADR

## Rollout

- Feature flag name and default
- Staged rollout, or all at once
- How we'd verify it's working in staging, then production
- **How we'd turn it off** if it's wrong

## Testing

How we'd know it works, mapped to [`../engineering/testing-strategy.md`](../engineering/testing-strategy.md).
If this touches scoring, say how the golden fixtures are affected.

## Effort

A rough size, honestly. Small (< 1 day), Medium (2–5 days), Large (> 1 week). If it's Large, say
what the first shippable slice would be — an RFC that can only be implemented as a big bang is
usually an RFC that should be split.

## Open questions

What you don't know, and who could answer it. An RFC with open questions is fine — that's often
the point of circulating it.

## Unresolved objections

If a reviewer disagreed and you proceeded anyway, record the disagreement and the reasoning here.
Future-you will want to know that the argument happened, and what it was.

## References

Prior art, other companies' write-ups, relevant ADRs, issues, benchmarks.
