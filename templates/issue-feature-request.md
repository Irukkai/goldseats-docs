<!--
Install as .github/ISSUE_TEMPLATE/feature_request.yml (issue form) or
.github/ISSUE_TEMPLATE/feature_request.md in goldseats-web and goldseats-api.

A YAML issue-form version is at the bottom.

Before you write this, check product/roadmap.md. It may already be planned for a
milestone — and its "What we're explicitly not building" section may already answer it.
-->

---
name: Feature request
about: Suggest something GoldSeats should do
labels: enhancement, needs-triage
---

## The problem

<!-- Start here, not with your solution. What can't someone do today, or what's painful?

     Anchor it in a moment. "Someone standing outside a Cineplex at 7:15pm deciding
     whether to buy for the 7:30 showing, on a phone, on transit data" is something we
     can design against. "Users want more features" isn't. -->

## Who has this problem

- [ ] A solo moviegoer who cares about picture and sound quality
- [ ] A group who need to sit together
- [ ] Someone who needs accessible seating
- [ ] A film follower tracking releases
- [ ] A contributor or operator (developer experience, operations)
- [ ] Other:

## What you'd like

<!-- Now the solution. Be as concrete as you can. -->

## How you work around it today

<!-- Very useful. If there's no workaround, say so — that raises the priority. -->

## Alternatives you considered

<!-- Including "do nothing". Sometimes that's right, and saying so makes the case for the
     others stronger. -->

---

## Fit

Please check these before filing. They're the questions we'd ask anyway.

- [ ] I checked [`product/roadmap.md`](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md)
      and this isn't already planned
- [ ] I checked the "What we're explicitly not building" section and this isn't on it
- [ ] This is about seats, films, showtimes, or the experience of choosing where to sit —
      GoldSeats' actual remit
- [ ] This doesn't require us to process payments
- [ ] This doesn't require us to scrape anything

**If it's already on the roadmap**, comment on the milestone issue instead — that's more
useful than a duplicate.

### Things we've already decided against

Not to discourage you, but to save you writing an issue we'll close with a link:

| Not building | Why | Where it's recorded |
| --- | --- | --- |
| Payment processing | Would pull us into PCI scope and change the company's risk profile | [roadmap](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md) |
| Holding or reserving seats | We don't sell tickets, so we can't hold one | [terms](https://github.com/Irukkai/goldseats-docs/blob/main/legal/terms.md) §4 |
| Scraping theatre sites for availability | Legal and commercial position, not a capacity limit | [ADR-0003](https://github.com/Irukkai/goldseats-docs/blob/main/architecture/adr/0003-manual-seed-theatre-data.md) |
| Reviews, comments, ratings, following users | Letterboxd exists and is good at this | [roadmap](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md) |
| A native mobile app | Web app is mobile-first; native is a post-launch decision | [roadmap](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md) |
| Recommending *what* to watch | We recommend seats. That's the product. | [roadmap](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md) |
| Using a third-party LLM API | Cost predictability and conversation privacy | [ADR-0004](https://github.com/Irukkai/goldseats-docs/blob/main/architecture/adr/0004-self-hosted-llm.md) |

If you have **new information** that changes one of these, say what it is. Those decisions
are genuinely open to a superseding ADR — "it would be easier" isn't new information, but
"a chain has offered us API access" is.

---

## How we'd know it worked

<!-- If this ships, what changes that we could measure? See
     engineering/observability.md for what we already track. -->

## Rough size, if you have a view

- [ ] Small — a day or less
- [ ] Medium — a few days
- [ ] Large — a week or more, probably needs an RFC or a PRD
- [ ] No idea

## Anything else

<!-- Mockups, examples from other products, links. -->

---
---

<!-- ============================================================
     GitHub issue form — .github/ISSUE_TEMPLATE/feature_request.yml
     ============================================================ -->

```yaml
name: Feature request
description: Suggest something GoldSeats should do
labels: [enhancement, needs-triage]
body:
  - type: markdown
    attributes:
      value: |
        Check [the roadmap](https://github.com/Irukkai/goldseats-docs/blob/main/product/roadmap.md)
        first — this may already be planned, or already decided against.

        Already decided against: payment processing, holding seats, scraping,
        reviews/ratings/social, native apps, recommending what to watch, third-party LLM APIs.

  - type: textarea
    id: problem
    attributes:
      label: The problem
      description: What can't someone do today, or what's painful? Not your solution yet.
      placeholder: |
        Someone standing outside a Cineplex at 7:15pm, deciding whether to buy for 7:30, can't...
    validations:
      required: true

  - type: dropdown
    id: who
    attributes:
      label: Who has this problem
      multiple: true
      options:
        - Solo moviegoer who cares about quality
        - Group who need to sit together
        - Someone who needs accessible seating
        - Film follower tracking releases
        - Contributor or operator
        - Other
    validations:
      required: true

  - type: textarea
    id: proposal
    attributes:
      label: What you'd like
    validations:
      required: true

  - type: textarea
    id: workaround
    attributes:
      label: How you work around it today
      description: If there's no workaround, say so — that raises the priority.

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives you considered

  - type: dropdown
    id: area
    attributes:
      label: Area
      options:
        - Seat scoring / recommendations
        - Seat map
        - 3D seat view
        - Chatbot
        - Film catalog / filters
        - Film detail / timeline
        - Theatre coverage
        - Booking handoff
        - Follows / feed / notifications
        - Accessibility
        - Developer experience
        - Other
    validations:
      required: true

  - type: checkboxes
    id: fit
    attributes:
      label: Fit
      options:
        - label: I checked the roadmap and this isn't already planned or already declined
          required: true
        - label: This doesn't require us to process payments
          required: true
        - label: This doesn't require us to scrape anything
          required: true

  - type: textarea
    id: success
    attributes:
      label: How we'd know it worked

  - type: dropdown
    id: size
    attributes:
      label: Rough size
      options: [Small — a day or less, Medium — a few days, Large — a week or more, No idea]

  - type: textarea
    id: anything-else
    attributes:
      label: Anything else
```
