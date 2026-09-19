# Glossary

One definition per term, used consistently in code, API fields, UI copy, and these docs. If
a term has a specific meaning at GoldSeats, it's here.

Naming consistency isn't pedantry: the same concept called `score` in Python, `rating` in
TypeScript, and "quality" in the UI is three concepts as far as anyone reading the code is
concerned.

---

## Product concepts

**Golden seat**
The highest-scoring available seat, or contiguous block of seats, for a given showtime and
party. The brand term, used in user-facing copy. In code and API responses the field is
`score` and rank 1 is the golden seat — we don't have a `is_golden` column, we have
`rank == 1`.

**Seat score**
A 0–100 number produced by the scoring engine for one seat or one block. Always `score`.
Never `rating`, `quality`, `grade`, or `stars`.

**Factor**
One of the five weighted components of a score: horizontal centre, vertical position, view
angle, neighbour openness, row quality. Each normalises to 0.0–1.0 before weighting. Always
returned in `SeatScore.factors` so a score can be explained.

**Scoring weights**
The five weights that combine factors into a score: 25% / 25% / 20% / 15% / 15%. They live
in one constant, `SCORING_WEIGHTS`, and sum to 1.0. Changing them is a product decision
requiring before/after distributions in the PR.

**Scenario**
The resolved party shape, derived **server-side** from `party_size`. One of `solo`, `pair`,
`small_group`, `large_group`, `family`. Clients never send it — one place decides what
`small_group` means.

**Contiguous block**
Two or more adjacent seats in the **same row** with no gap and no aisle between them. Group
scenarios only ever return contiguous blocks, so a recommended option is always bookable
as-is.

**Neighbour openness**
The factor measuring whether adjacent seats are free. An empty seat beside you is worth real
money, which is why it's 15% of the score. Requires unavailable seats to be passed into the
scoring engine, not filtered out beforehand.

**View angle**
The horizontal field of view the screen subtends from a seat, in degrees. Scored against a
THX-inspired target of about 36°, penalised in both directions — too close is as bad as too
far.

**Row quality**
Row-level attributes rolled into one factor: stadium elevation, recliner rows, aisle access,
premium centre sections.

**Booking handoff**
Sending the user to the theatre's own booking page via a deep link. The end of our flow.
**We never process payments**, which is why GoldSeats has no PCI scope.

**Reserve in person**
The alternative to booking online: we save the user's seat pick so they can buy at the box
office. **This does not hold the seat** — we can't reserve something we don't sell. UI copy
must never imply otherwise.

**Format**
A presentation format: `imax`, `dolby_cinema`, `ultraavx`, `standard`, and so on. A single
showtime can have several. Stored as `format_code` because `format` is a SQL reserved word.

**Best way to experience**
The film detail page panel ranking available formats by `quality_rank`. An editorial
judgement, stored as data so it's reviewable.

**Timeline**
The chronological, expandable, multi-source history of a film on its detail page, built from
`film_timeline_events`: announcement, casting, trailer, festival, review, release, award,
news.

**Feed**
A logged-in user's personalised stream, assembled from what they follow. Distinct from the
timeline, which is per-film and identical for everyone.

**Follow**
A user's subscription to a film, theatre, or person. Drives the feed and notifications.

**Watchlist**
Films a user wants to see, plus "seen" markers. Separate from follows: you can follow a film
you've already seen, and watchlist a film you don't want news about.

---

## Data model terms

Full specification: [`../architecture/data-model.md`](../architecture/data-model.md).

**Theatre**
A physical cinema building. `theatres`. Never "cinema" or "venue" in code.

**Auditorium**
A screening room inside a theatre. `auditoriums`. **Never** "screen", "room", "hall", or
"theatre" — "screen" in particular means the physical projection surface, and conflating the
two produces genuinely confusing code.

**Screen**
The physical projection surface. Has `screen_width_m`, `screen_height_m`,
`screen_bottom_height_m`. Not a synonym for auditorium.

**Auditorium kind**
One of `standard`, `large`, `small`, `imax`, `recliners` — the five theatre types from the
dataset generator. Seeded auditoriums use these so their geometry matches what the scoring
model was tuned on.

**Seat layout**
A versioned, immutable seating configuration for an auditorium. `seat_layouts`. Append-only:
a reconfiguration creates version 2 and version 1 survives, because recommendations computed
against it must stay interpretable.

**Layout version**
The integer `seat_layouts.version`, monotonic per auditorium. Returned with every
recommendation, because a recommendation is only meaningful against the layout it was computed
on.

**`layout_json`**
The canonical geometry document for a layout. The one sanctioned JSON blob in the schema,
because the geometry is genuinely a document read whole or not at all. Has its own
`schema_version`.

**Row label vs row index**
`row_label` is what's printed on the seat — `A`, `B`, `AA`. `row_index` is 0-based from the
screen, so **row A is index 0**. Both are stored, deliberately. Labels must match reality or
users sit in the wrong place; geometry must be 0-based or adjacency arithmetic is off by one.

**Seat number vs seat index**
`seat_number` is printed and 1-based. `seat_index` is 0-based within the row. Same reasoning.

**Seat type**
`regular`, `recliner`, `wheelchair`, `companion` — the four types from the dataset generator.

**Wheelchair space / companion seat**
An accessible seating position and its adjacent companion seat. Recommended as a contiguous
pair when `accessible_seating_required` is set. **Scored honestly** — an accessible space
with a poor view gets a low score, because telling someone their only option is great when it
isn't is worse than useless.

**Availability snapshot**
An immutable point-in-time observation of which seats are taken.
`seat_availability_snapshots`. Newest wins.

**Simulated availability**
Availability generated by the dataset generator's booking simulator rather than observed.
Stored with `source='simulated'` and `confidence='low'`, and **the UI says so**. The default
state until a partner data feed exists.

**Booking link**
The validated outbound URL for a showtime. `booking_links`. Has an `allowed_host` checked
against an allowlist at render time, because unvalidated outbound navigation is an open
redirect with our brand on it.

**Recommendation**
A persisted scoring result. `recommendations`. Persisted rather than computed-and-forgotten so
the 2D map, the 3D view, the chatbot, and an in-person reservation all refer to the same one
by ID.

**Scoring version**
`recommendations.scoring_version` — the version of the engine and weights that produced a
stored score. Without it, a score from March can't be compared to one from June.

---

## Engineering terms

**Done-when criterion**
The single observable fact that defines a milestone as complete. Demonstrable to another
person, not asserted. Every file in [`milestones/`](milestones/) has exactly one.

**Definition of done**
The per-issue checklist in
[`../engineering/definition-of-done.md`](../engineering/definition-of-done.md). Distinct from
the done-when criterion, which is per-milestone.

**Expand / contract**
The two-or-three-release pattern for schema changes that keeps every migration compatible with
currently deployed code, so a rollback never needs a database restore.

**Parity job**
The CI job that runs the API test suite against a real Postgres as well as SQLite. The only
thing standing between us and "worked locally, 500s in production", so it doesn't get skipped.

**Golden fixtures / golden snapshot**
Committed score distributions for real seeded auditoriums. Any scoring drift fails the build,
forcing drift to be a conscious decision.

**Grounded**
A chatbot reply in which every seat mentioned was actually returned by a tool call. Enforced
by verification between the model and the user; an ungrounded reply is blocked, not corrected.

**Tool call**
The LLM's only mechanism for obtaining facts: `score_seats`, `lookup_showtimes`,
`lookup_seat_layout`. The model has no database, no filesystem, and no outbound network. This
is what makes "can never invent a seat" structural.

**Quality tier**
The 3D view's capability level — `high`, `medium`, `low`, `none` — chosen from device
capability. `none` is a first-class path with a text panel, not a failure state.

**ADR**
Architecture Decision Record. A numbered, immutable record of a significant decision. See
[ADR-0001](../architecture/adr/0001-record-architecture-decisions.md).

**RFC**
A proposal exploring a question *before* it's decided. An ADR records the answer. See
[`../templates/rfc-template.md`](../templates/rfc-template.md).

**`request_id`**
The correlation ID on every request, present in the response body, the `X-Request-Id` header,
every log line, the trace, and the Sentry event. The thread tying a user's bug report to the
line of code that failed.

**Feature flag**
A setting gating incomplete work so it can merge to `main` inert rather than living on a
long-lived branch. Defaults off, deleted in the milestone that ships it.

---

## External terms

**TMDB**
The Movie Database. Our film metadata source. Free API, **mandatory attribution**, and not
endorsed by or affiliated with us. See
[`../legal/data-sources-policy.md`](../legal/data-sources-policy.md).

**Cineplex**
Canada's largest theatre chain and our primary target. No public API, which is why theatre
data is hand-seeded.

**THX**
The cinema certification standard whose recommended viewing angle (~36° horizontal) informs
our view-angle factor. Inspiration, not affiliation or certification.

**Ollama**
Local LLM server used in development. OpenAI-compatible endpoint.

**vLLM**
Production LLM inference server. Same OpenAI-compatible interface as Ollama, so switching is
an env var.

**react-three-fiber**
React renderer for three.js. Powers the 3D seat view. Must never appear in an initial bundle.

**UUIDv7**
Time-ordered UUID, our primary key everywhere. Client-generated, so no ID enumeration on a
public API and good index locality.

**Conventional Commits**
The commit message format we enforce. Also drives changelog generation and version bumps.

**AODA**
Accessibility for Ontarians with Disabilities Act. Relevant because our primary market is
Ontario; it legislates WCAG 2.0 AA, and we target 2.2 AA.

**PIPEDA**
Canada's federal privacy law. Governs how we handle personal information. See
[`../legal/privacy-policy.md`](../legal/privacy-policy.md).

**PCI DSS**
Payment card security standard. **We are entirely out of scope**, because booking is a
deep-link handoff and no card data touches us. Stated in
[`../SECURITY.md`](../SECURITY.md) so nobody has to work it out.

---

## Words we don't use

| Don't say | Say | Why |
| --- | --- | --- |
| "screen" meaning a room | auditorium | "Screen" is the projection surface |
| "rating" for a seat | score | "Rating" means a film's audience rating |
| "cinema", "venue" | theatre | One word per concept |
| "booking" for our flow | booking handoff | We don't book anything |
| "reserve" implying a held seat | reserve in person | We don't hold seats. This one is a copy-accuracy issue, not style. |
| "AI recommends" | scored / recommended | The scoring engine is a weighted heuristic, not a model. Say what it is. |
| "scrape" | seed, ingest | We don't scrape, and the vocabulary shouldn't suggest we might |
| "user" in UI copy | you | "User" is for code and docs |
