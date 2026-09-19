# ADR-NNNN — Short title in the imperative

- **Status:** Proposed | Accepted | Rejected | Deprecated | Superseded by `ADR-NNNN` (link it)
- **Date:** 2026-MM-DD
- **Deciders:** names
- **Milestone:** the milestone this belongs to, linked — e.g. [M5](../../product/milestones/M5.md)
- **Supersedes:** `ADR-NNNN`, or "none"

---

## Context

What's the situation that forces a decision? Facts and constraints, not opinions.

Include the things that are true regardless of what we choose: team size, existing code,
what's installed on the machines we develop on, deadlines, what the product needs to do,
what the law requires. If a constraint is the reason a tempting option is off the table,
say so here rather than in the rejection.

Write this so that someone who joins in two years, who has never met you, understands why
this was a real question.

## Decision

The decision, in the present tense and the active voice. "We use X." Not "we will use X"
and not "X seems better".

Be specific enough to be actionable: versions, package names, file paths, which env var
controls it.

## Options considered

### Option A — the one we chose

- **Pros:** …
- **Cons:** …

### Option B

- **Pros:** …
- **Cons:** …
- **Why not:** …

### Option C

- **Pros:** …
- **Cons:** …
- **Why not:** …

Include the option someone will inevitably propose later, even if it was never seriously in
contention. Half the value of an ADR is being able to point at it when the question comes
back.

## Consequences

### What this makes easier

### What this makes harder

Be honest here. An ADR with no downsides is an ADR nobody will trust, and the downsides are
the part that's genuinely useful later.

### What this commits us to

Follow-on work, constraints on future decisions, things we now can't do without revisiting
this.

## Revisit when

Concrete triggers that should make us re-open this — a scale threshold, a team-size change,
a dependency going unmaintained, a cost crossing a number. Vague "when it becomes a problem"
is not a trigger.

## References

- Links to related ADRs, RFCs, issues, external documentation
