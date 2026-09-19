# Data Sources Policy

**Read this before adding any data source to GoldSeats.** It's an engineering document with legal
consequences, and it's binding on every contributor.

The short version: **we do not scrape. Ever.** Every byte of external data comes from a documented
API, a published feed, or a human typing it in from a public source. Booking is a link, not an
integration.

---

## Why this is strict

Three reasons, in order of how much they'd cost us to get wrong:

1. **Our monetisation path runs through the companies we'd be scraping.** GoldSeats sends
   pre-qualified buyers into theatre chains' own booking flows and takes no payment data. That's a
   genuinely attractive affiliate pitch — and it only works if we're clean. "We've been scraping
   your seat picker, but we'd like to discuss a partnership" is not a conversation that goes
   anywhere good.
2. **Legal exposure.** Scraping a site whose terms prohibit automated access is a breach of
   contract. In Canada there's also the *Copyright Act* on database compilations. A pre-revenue
   company with one engineer should not be acquiring legal risk to save seeding effort.
3. **It wouldn't even solve our problem.** A seat picker renders seat IDs and availability states.
   It does not publish screen width in metres, first-row distance, stadium rise, or row curvature —
   which are the actual inputs to our scoring model. The hard part of our data problem isn't
   scrapeable.

Full reasoning and the alternatives considered:
[ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md).

---

## What's allowed

| Source | Status | Notes |
| --- | --- | --- |
| TMDB API | ✅ Allowed | With attribution. See below. |
| Published RSS/Atom news feeds | ✅ Allowed | Headline, link, attribution, short summary only |
| Hand-authored theatre data from public seating charts | ✅ Allowed | A human reading a published floor plan |
| Official APIs with a signed agreement | ✅ Allowed | The goal. Needs an ADR. |
| Licensed paid data feeds | ✅ Allowed | Needs an ADR and a budget |
| Public data the publisher explicitly offers for reuse | ✅ Allowed | Check the licence, record it |
| **Automated extraction from a web page's HTML** | ❌ **Prohibited** | This is scraping |
| **Automated access to an undocumented internal API** | ❌ **Prohibited** | Also scraping, with extra steps |
| **Headless-browser data collection** | ❌ **Prohibited** | Scraping with a browser attached |
| **Anti-bot circumvention of any kind** | ❌ **Prohibited** | If you're evading a defence, you know you're unwelcome |
| **Storing full article text** | ❌ **Prohibited** | Copyright |
| **Re-hosting another site's images** | ❌ **Prohibited** | Except CDN URLs a provider offers for the purpose |

---

## Adding a new source

Every new external data source requires **all** of the following before any code merges:

1. **Read the terms of service.** Not skim — read the automated-access and data-use sections.
   Quote the relevant clause in the PR.
2. **Check `robots.txt`.** Respect it, including `Crawl-delay`. If our access pattern isn't
   permitted, we don't do it.
3. **Confirm the licence** for the data and for any images.
4. **Record attribution requirements** and implement them in the same PR, not later.
5. **Write an ADR** if the source is material to the product.
6. **Add it to the table below**, with the date checked and by whom.
7. **Identify ourselves.** Every outbound request sends a real User-Agent with a contact address:
   ```
   User-Agent: GoldSeats/1.0 (+https://goldseats.app; contact@goldseats.app)
   ```
8. **Rate limit ourselves** below whatever the source permits, with exponential backoff on
   `429` and `5xx`.
9. **Cache aggressively.** Refetching data we already have is rude and pointless.

A PR that adds a source without these is incomplete. A PR that adds a scraper is closed, not
reviewed.

---

## Approved source register

| Source | Type | Data | Attribution | Last checked | Checked by |
| --- | --- | --- | --- | --- | --- |
| TMDB | Documented API, free tier | Film metadata, posters, releases, language, trailers | **Mandatory**, see below | 2026-02-09 | Founder |
| Theatre seating charts | Manual, human-authored | Auditorium geometry, seat layouts | Source noted in `seat_layouts.notes` | 2026-02-09 | Founder |
| Theatre booking pages | Manual, outbound link only | Booking URL structure | n/a | 2026-02-09 | Founder |
| Film news feeds | Published RSS/Atom | Headline, URL, publisher, timestamp, short summary | Publisher name and link on every item | pending [M8](../product/milestones/M8.md) | — |

---

## TMDB

Film metadata comes from TMDB. Their terms require attribution, and the wording matters.

**Required on every page displaying TMDB data, and in the footer site-wide:**

> This product uses the TMDB API but is not endorsed or certified by TMDB.

Plus the TMDB logo, linking to `https://www.themoviedb.org`.

Rules:

- **The disclaimer wording is not ours to improve.** Use it as specified.
- Never imply endorsement, partnership, or affiliation.
- Posters and backdrops are served from TMDB's CDN URLs — we store the URL, not a copy of the
  image.
- Respect their rate limits. Cache for 15 minutes at minimum; film metadata barely changes.
- The API key is a secret. It lives in `TMDB_API_KEY` and never in git. See
  [`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).
- If TMDB's terms change, that's a blocking issue, not a later thing. Re-check annually.

Where in the product:

- Footer of every page in `goldseats-web`
- Film detail pages
- Any timeline event whose `film_sources.source = 'tmdb'`
- `film_sources` records provenance per film, which makes attribution auditable rather than
  assumed

---

## Theatre data — hand-authored

Auditorium layouts, showtimes, and booking links are entered by a human from public sources.

**Permitted sources:**

- Theatre chains' published seating charts and floor plans, viewed as a normal user in a normal
  browser
- Venue specification pages, technical sheets, press materials
- In-person observation — counting rows, noting aisle positions, measuring where feasible
- Publicly published capacity figures

**Not permitted:**

- Automated extraction of a seat picker's HTML or its underlying JSON
- Collecting real-time availability by any automated means
- Any access pattern a reasonable operator would consider automated

**Provenance is recorded, not assumed.** Every `seat_layouts` row carries `seeded_by`,
`seeded_at`, `source_file`, and a `notes` field naming where each measurement came from and
whether it's measured or estimated:

```
"Rows and seat numbers from Cineplex's published seating chart, verified against
their seat picker rendering 2026-03-14. Screen width estimated at 16m from
auditorium kind and published capacity — NOT measured. First-row distance
estimated. Stadium rise from the standard geometry model."
```

That last part matters for the [M6](../product/milestones/M6.md) 3D view: if a screen dimension
is estimated, the preview says "approximate" rather than presenting a guess as a measurement.

---

## Seat availability — currently simulated

We have no permitted source of real-time seat availability. So:

- Availability is generated by the booking simulator from the dataset generator
- Stored with `source = 'simulated'` and `confidence = 'low'`
- **The API returns those fields and the UI is required to disclose them.** Not in a tooltip — in
  visible text.
- `seat_availability_snapshots.source` has a `CHECK` constraint permitting `manual`, `simulated`,
  and `partner_api`. **There is no `scraped` value.** The policy is expressed as a database
  constraint, so violating it requires a migration, a PR, and someone explaining themselves.

Presenting simulated availability as real would be the single most damaging thing this product
could do to its own credibility. Real availability requires a partnership, and that's the work —
not a workaround.

---

## Booking — deep link only

We link. We do not integrate.

- `booking_links.url_template` holds a public URL that a user could have reached by browsing the
  theatre's site themselves
- Clicking it navigates the user's own browser to the theatre's domain
- **No payment data ever touches GoldSeats**, which is why we have no PCI scope — see
  [`../SECURITY.md`](../SECURITY.md)
- We don't proxy, frame, iframe, or replicate any part of the theatre's booking flow
- We don't inject anything into their pages
- `allowed_host` is validated against a hardcoded allowlist **at render time**, not just at seed
  time, because unvalidated outbound navigation is an open redirect with our brand on it

If a chain asks us to stop linking to them, we stop. It's their traffic.

---

## News aggregation

For [M8](../product/milestones/M8.md). Feeds only.

**What we store:** headline, canonical URL, publisher name, author, publication timestamp, and
**our own short summary** or a licensed excerpt.

**What we never store:** the full article body, the publisher's images (unless the feed explicitly
offers them for syndication), or anything that would make our page a substitute for theirs.

The test: *does our feed item make someone less likely to click through to the publisher?* If yes,
we've taken too much. The point is to be a good index, not a replacement.

Additional rules:

- Only feeds the publisher publishes for consumption
- Honour `robots.txt` and any stated syndication terms
- Attribution visible on every item — publisher name and a link, not hidden behind a hover
- Honour a takedown request immediately, then discuss
- Every source in the register above, with its licence terms recorded

---

## User data

Covered properly in [`privacy-policy.md`](privacy-policy.md), but the principles that belong here:

- Collect only what a feature needs
- Never sell user data. Not now, not later.
- Never share it with a third party except a named subprocessor doing work for us
- **Chat conversations don't leave our infrastructure**, which is a direct consequence of
  self-hosting the model — see [ADR-0004](../architecture/adr/0004-self-hosted-llm.md)
- Location is coarse, opt-in, and used only to find nearby showtimes
- Users can export and delete everything

---

## If you think we should scrape something

You're not the first. The reasoning:

1. Read [ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md). It considers the option
   seriously and explains why it loses on every axis at once.
2. If you have new information the ADR didn't have, write a superseding ADR. That's the process and
   it's genuinely open.
3. **"It's technically easy" is not new information.** Neither is "a competitor does it".

What *would* be new information: a chain publishing terms that permit it, a chain granting us
access, or a court ruling that changes the Canadian legal position. Any of those is worth a
conversation.

---

## Enforcement

| Mechanism | What it catches |
| --- | --- |
| Code review | A PR adding automated extraction is closed, not reviewed |
| `CHECK` constraint on `seat_availability_snapshots.source` | No `scraped` value exists to write |
| This register | An undocumented source is an incomplete change |
| [Definition of done](../engineering/definition-of-done.md) | New source → policy check required |
| Annual review | Terms change; our register should notice |

---

## Annual review

Every source's terms get re-checked annually, and the register's "last checked" column updated.
Terms change quietly, and a policy nobody re-reads is a policy that drifts out of compliance
without anyone noticing.

Next review due: **2027-02**.

---

## A note on legal advice

This document reflects our understanding of our obligations and our deliberate policy choices. It
is not legal advice. The specific questions around database rights, syndication limits, and
Canadian scraping precedent would need a qualified lawyer to answer definitively.

Our approach to that uncertainty is to stay well inside any plausible boundary rather than to find
the edge of it. **A lawyer's review of this policy is a [M9](../product/milestones/M9.md)
deliverable**, alongside the privacy policy and terms.

---

## Related

- [ADR-0003 — Manually seed theatre data](../architecture/adr/0003-manual-seed-theatre-data.md)
- [ADR-0004 — Self-hosted LLM](../architecture/adr/0004-self-hosted-llm.md)
- [`privacy-policy.md`](privacy-policy.md)
- [`terms.md`](terms.md)
- [`../architecture/data-model.md`](../architecture/data-model.md) — `film_sources`,
  `seat_availability_snapshots`, `booking_links`, `news_items`
- [`../SECURITY.md`](../SECURITY.md) — why we hold no payment data
