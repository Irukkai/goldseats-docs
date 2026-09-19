# Product Requirement Documents

A PRD describes **what** we're building and **why**, for the user. It's the counterpart to an
[ADR](../../architecture/adr/), which describes *how* we build it and why we picked that way.

A PRD is a feature's brief. A milestone file is a delivery plan. They're different documents
and conflating them produces a bad version of both.

---

## When you need one

Write a PRD when:

- A feature has real product ambiguity — several defensible user experiences, and the choice
  matters
- The feature spans both repos and needs a shared understanding before either side starts
- Success is measurable and we should agree on the measure before we build
- The feature touches a trust boundary: what we tell users about data quality, privacy, or
  what "reserve" means

Skip the PRD when:

- The milestone file already specifies it adequately. Most of M0–M4 is like this — a home
  page listing films with filters doesn't need a brief.
- It's a bug fix, a refactor, or a technical improvement. Those are issues.
- The decision is technical rather than product. That's an ADR.
- It's a single endpoint following existing conventions.

**Rule of thumb:** if you can write the acceptance criteria straight onto a GitHub issue
without arguing with yourself, you don't need a PRD.

---

## Existing PRDs

| PRD | Status | Milestone |
| --- | --- | --- |
| *None yet.* M0–M4 are adequately specified by their milestone files. | | |

Expected candidates, in likely order:

| Candidate | Why it needs one | Milestone |
| --- | --- | --- |
| **Booking handoff and in-person reservation** | The fork between "buy online" and "reserve in person" is the product's most delicate moment. "Reserve" must not imply a held seat, we need to know what we do when the chain can't accept a seat preselection, and the conversion measurement matters. | [M5](../milestones/M5.md) |
| **Seat map interaction model** | How scores are surfaced without overwhelming the map, how a group block is represented, what happens when no contiguous block exists, and how simulated availability is disclosed. Genuinely several defensible designs. | [M5](../milestones/M5.md) |
| **Chatbot conversation design** | What the bot can and can't do, how it declines, how it discloses that it's grounded in our scoring, what happens when the model is down. The trust surface is the whole feature. | [M7](../milestones/M7.md) |
| **Notification policy** | What we notify about, how often, defaults, and the unsubscribe path. Easy to make annoying, and annoying notifications lose users permanently. | [M8](../milestones/M8.md) |
| **Coverage and expectation setting** | What we show for a theatre we haven't seeded. This is the product's headline limitation and handling it well is a design problem, not an error state. | [M2](../milestones/M2.md)/[M3](../milestones/M3.md) |

---

## Process

1. Copy [`template.md`](template.md) to `product/prd/<kebab-case-name>.md`.
2. Open it as a **draft PR** early. A PRD's value is in the argument, and a finished document
   nobody argued with has skipped the useful part.
3. Fill in the problem and the success metrics **before** the solution. If you can't state
   the problem without describing your solution, you don't understand the problem yet.
4. Get one review. Reviewers should push hardest on the "Not doing" section — a PRD with no
   scope boundaries will produce a feature with no scope boundaries.
5. Merge when accepted. Add it to the table above.
6. Create GitHub issues from the requirements, on the right milestone.
7. **Update the PRD when reality diverges.** A PRD claiming we shipped something we cut is
   worse than no PRD, because someone will trust it.

---

## What makes a PRD good here

- **It names the user and the moment.** "Someone standing outside a Cineplex at 7:15pm
  deciding whether to buy for the 7:30 showing" is a design constraint. "Users" is not.
- **It's honest about our data.** Availability is simulated until a partner feed exists. A PRD
  that assumes real-time availability is specifying a product we can't build yet.
- **It states what we won't do.** The "Not doing" section is the most useful part and the
  first thing to get skipped.
- **It has a measurable success criterion**, agreed before building. "Users love it" is not a
  criterion; "60% of sessions that reach a seat map produce a booking handoff" is.
- **It respects the locked decisions.** No payments, no scraping, no held seats. A PRD that
  needs one of those needs an ADR first.
- **It's short.** Two pages of specifics beats eight pages of context. Link to
  [`../glossary.md`](../glossary.md) instead of redefining terms.

---

## Related

- [`template.md`](template.md) — the PRD template
- [`../roadmap.md`](../roadmap.md) — M0–M9 and sequencing
- [`../milestones/`](../milestones/) — delivery plans per milestone
- [`../../templates/rfc-template.md`](../../templates/rfc-template.md) — for exploring a
  technical question
- [`../../architecture/adr/`](../../architecture/adr/) — for recording a technical decision
