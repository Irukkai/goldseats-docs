# Code Review

Every change to `main` goes through a pull request with one approving review. No
exceptions, including for the founder. The one narrow carve-out is a revert that restores
a broken `main` — see [`git-workflow.md`](git-workflow.md#reverting).

Review exists to catch the things tests and linters can't: wrong abstractions, missing
cases, decisions that will be expensive in three months, and code the next person won't
understand.

---

## SLAs

| Event | Target |
| --- | --- |
| First review on a ready PR | **1 business day** |
| Review on a PR labelled `urgent` or an incident hotfix | **Drop what you're doing** |
| Author response to review comments | 1 business day |
| Re-review after changes | Same day |

If you can't review within the SLA, say so on the PR immediately so the author can find
someone else. Silence is the worst outcome — it blocks work without telling anyone.

---

## Author responsibilities

A reviewable PR is the author's job, not the reviewer's problem.

**Before marking ready for review:**

- [ ] Lint, typecheck, test, and build all pass **locally**. CI is a safety net, not your
      first test run.
- [ ] The diff is under ~400 lines, or the description explains why it can't be.
- [ ] One concern per PR. A migration, a new endpoint, and a Tailwind refactor are three
      PRs.
- [ ] No unrelated formatting churn. If you need to reformat a file you're touching, do it
      in a separate `chore:` commit within the PR so the reviewer can skip it.
- [ ] Commented-out code, debug prints, and `console.log` removed.
- [ ] The [PR template](../templates/pull-request-template.md) is filled in — especially
      **why**, and **how you verified it**.
- [ ] Screenshots or a screen recording for any UI change. Both mobile and desktop widths.
- [ ] Self-reviewed. Read your own diff in GitHub before anyone else does; you'll catch a
      third of the comments yourself.

**Special requirements:**

- **Scoring changes** (`app/ml/scoring.py`): include the before/after score distribution
  across the golden fixture auditoriums. Weight changes without evidence get
  `request-changes` automatically. Scoring drift is the easiest way to quietly make the
  whole product worse.
- **Schema changes**: link the [`../architecture/data-model.md`](../architecture/data-model.md)
  update, and state the expand/contract step this release represents.
- **New dependency**: justify it in the description — what it does, its size, its
  maintenance status, and what writing it ourselves would cost.
- **New data source**: link the
  [`../legal/data-sources-policy.md`](../legal/data-sources-policy.md) compliance check.

**Responding to review:**

- Answer every comment. "Done" is a valid answer; silence isn't.
- Push fixes as new commits, don't force-push mid-review — the reviewer loses their place.
  Squash happens at merge anyway.
- Disagree freely, but explain. "I'd rather not, because X" is a fine outcome and often
  the right one.
- Don't resolve a reviewer's thread yourself unless it was purely informational. Let them
  resolve it.

---

## Reviewer responsibilities

### What to actually check

Roughly in order of how much damage a miss causes:

**1. Correctness**
- Does it do what the linked issue asked?
- Edge cases: empty result set, single seat left, party larger than any contiguous block,
  a showtime that has already started, a film with no releases in the user's language.
- Timezones. Showtimes are the most timezone-sensitive data we have and the bug is always
  the same: a naive datetime treated as UTC. Every datetime should be tz-aware.
- Off-by-one in seat and row indexing. Row `A` is index 0; seat numbering is
  1-based in labels and 0-based in geometry, and that seam has already bitten us.

**2. Security**
- Any secret in the diff? The repos are public. If there is one, say so immediately and
  privately — rotate before anything else.
- Parameterised queries only. Any string-built SQL is an automatic block.
- Is a new endpoint authenticated and authorised correctly? Can user A read user B's
  watchlist, saved picks, or chat history?
- Does user-controlled input reach the LLM prompt, a file path, or an outbound URL?
- Does a new outbound `booking_link` URL get validated against an allowlist of theatre
  domains? An open redirect in the booking handoff would be genuinely dangerous.

**3. Data and contract**
- Is the migration backwards compatible with deployed code?
  ([`branching-and-releases.md`](branching-and-releases.md#migrations-and-release-ordering))
- Does the migration have a working `downgrade()`?
- Does the endpoint follow [`api-design-guidelines.md`](api-design-guidelines.md) — naming,
  pagination, error shape? A sloppy response shape is permanent once the web app depends
  on it.
- Is this a breaking change for `goldseats-web`? If so, is the `BREAKING CHANGE:` footer
  there and is the web-side PR linked?

**4. Performance**
- N+1 queries. Is every serialised relationship eager-loaded?
- Is the list endpoint paginated with a deterministic `order_by`?
- Web: did three.js leak into the initial bundle? Did a component become a client
  component unnecessarily? Is cache policy explicit on every `fetch`?
  ([`performance-budgets.md`](performance-budgets.md))
- Is seat scoring for a large auditorium off the event loop?

**5. Accessibility** — for any UI change
- Keyboard reachable, visible focus, correct roles and accessible names.
- Seat quality never communicated by colour alone.
  ([`accessibility.md`](accessibility.md))

**6. Tests**
- Does a new behaviour have a test that would fail without the change? Check that by
  reading the test, not by trusting the coverage number.
- Is the test asserting behaviour or re-implementing the code?

**7. Observability**
- Would you be able to debug this at 3am from logs alone?
- Structured log fields, no PII, no tokens.
  ([`observability.md`](observability.md))

**8. Readability**
- Will someone understand this in six months with no context? That includes you.
- Do names match [`../product/glossary.md`](../product/glossary.md)?
- Do comments explain *why* rather than narrate *what*?

Then check it against [`definition-of-done.md`](definition-of-done.md).

### How to comment

Prefix every comment with its weight, so the author knows what's blocking:

| Prefix | Meaning | Blocks merge |
| --- | --- | --- |
| `blocking:` | Must change before merge | Yes |
| `question:` | I don't understand this yet | Until answered |
| `suggestion:` | I'd do it differently, your call | No |
| `nit:` | Trivial. Ignore freely. | No |
| `praise:` | This is good and I want you to know | No |

```
blocking: This builds the WHERE clause with an f-string on user input. Bind the
parameter instead — see database-conventions.md.

question: If the party is 5 and no row has 5 contiguous free seats, does this return
an empty list or raise? The test only covers the happy path.

suggestion: `_resolve_scenario` might read better as a dict lookup than a chain of ifs.
Fine either way.

nit: typo — "recomendation".

praise: The factors dict on SeatScore is going to make the chatbot's explanations much
easier in M7. Nice call.
```

Rules:

- **Review the code, not the person.** "This function does X" not "you always do X".
  Required by [`../CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md).
- Be specific. "This is confusing" is useless; "I can't tell from the name whether
  `score` is 0–1 or 0–100" is actionable.
- Say what you *want*, not just what's wrong.
- Don't design a different feature in review. If the approach is wrong at a level the PR
  can't fix, say so once, block, and move the conversation to an
  [ADR](../architecture/adr/template.md) or [RFC](../templates/rfc-template.md).
- **Don't bikeshed formatting.** Prettier and Ruff own formatting. If you're arguing about
  it, the tool config is wrong — fix the config in its own PR.
- If you learned something from the PR, say so. Review is the main way knowledge moves
  around a small team.

### Approving

Approve when the PR is correct, safe, tested, and understandable. Not when it's how *you*
would have written it.

- **Approve** — good to merge. Leftover `nit:` and `suggestion:` comments are fine to
  merge over.
- **Approve with blocking comments** — don't. Pick one.
- **Comment** — you looked, you have questions, you're not making a call yet.
- **Request changes** — there's a `blocking:` comment. Be explicit about what would make
  it approvable.

Never approve a diff you didn't read. A rubber stamp on a 1,200-line PR is worse than no
review, because it creates a false record that the change was checked.

---

## Merging

The author merges, not the reviewer — the author knows whether anything else needs to land
alongside it.

Squash merge only. Check the squash commit subject is a valid Conventional Commit before
confirming; GitHub likes to paste the full commit list into the body, so trim it down to
the PR description.

---

## Reviewing in a one-person team

Until there's a second engineer, the process degrades honestly rather than pretending:

- The PR, the template, the CI gates, and the checklists all still apply. They're doing
  most of the work.
- Self-review on the GitHub diff page, not in your editor. The change of context is the
  point, and it catches real bugs.
- Wait at least an hour between finishing the code and reviewing it. Same-minute
  self-review finds nothing.
- Self-approve and merge only after CI is green and the diff has actually been read.
- Anything architectural gets an [ADR](../architecture/adr/template.md) instead of a
  reviewer, so the reasoning survives even though nobody argued with it at the time.

The first thing a second engineer changes is this section.
