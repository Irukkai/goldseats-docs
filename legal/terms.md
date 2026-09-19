# Terms of Service

**Last updated: 2026-02-09**
**Effective: on launch of `goldseats.app`**

> ⚠️ **This is a substantive draft and it is not legal advice.** It must be reviewed by a lawyer
> qualified in Ontario contract and consumer-protection law before publication. That review is a
> required deliverable of [M9](../product/milestones/M9.md). Sections needing specific legal
> attention are marked **[LEGAL REVIEW]**.

---

## The short version

- GoldSeats recommends cinema seats. **We don't sell tickets** — we send you to the theatre's own
  website to buy.
- Our recommendations are informed opinions based on auditorium geometry. They're not guarantees.
- Our theatre and availability data is limited, and some of it is currently **simulated**. We tell
  you when it is.
- **"Reserve to pay in person" saves your seat choice. It does not hold the seat.** Nobody has
  reserved anything for you.
- Don't abuse the service, don't scrape us, don't try to break it.
- Your account, your data, your right to delete it — see [`privacy-policy.md`](privacy-policy.md).

The rest is the detail.

---

## 1. Acceptance

By using `goldseats.app` you agree to these terms. If you don't agree, don't use it.

If you're using GoldSeats on behalf of an organisation, you're confirming you have the authority to
accept these terms for it.

**[LEGAL REVIEW]** Confirm entity name, jurisdiction, and registered address.

---

## 2. What GoldSeats is

GoldSeats analyses cinema auditorium layouts and recommends seats. We:

- Score seats using a weighted model based on auditorium geometry and seat availability
- Show you the view from a seat in 3D
- Let you ask for seats conversationally
- Show films that are playing and coming, with news and release information
- **Link you to the theatre's own booking page to buy your ticket**

### What GoldSeats is not

Read this part. It's the section that matters most.

- **We are not a ticket seller.** We have no ability to sell, hold, reserve, or cancel a cinema
  ticket. Every transaction happens on the theatre's own website, under their terms.
- **We are not affiliated with any theatre chain.** Not Cineplex, not any other. We link to their
  public booking pages the same way any website links to another.
- **We don't take payments.** We never see your card details. If any site claiming to be GoldSeats
  asks for payment information, it isn't us.
- **We are not TMDB.** We use their API for film metadata. *This product uses the TMDB API but is
  not endorsed or certified by TMDB.*
- **We are not THX.** Our view-angle scoring is informed by published THX recommendations. We have
  no affiliation with or certification from THX.

---

## 3. Seat recommendations — what they are and aren't

This is the heart of our service, so we want to be precise about what we're claiming.

**What a recommendation is:** an informed opinion, computed from an auditorium's geometry and the
seat availability we have, using a weighted model — horizontal position, depth into the
auditorium, the angle the screen subtends, whether neighbouring seats are free, and row
characteristics.

**What it isn't:**

- **A guarantee.** Seat quality is partly subjective. You may prefer a seat we scored lower, and
  you're not wrong.
- **A promise of availability.** Availability changes constantly and we're not the seller. A seat we
  recommend may already be sold by the time you reach the theatre's page.
- **A promise about the physical auditorium.** Layouts change. Seats break. Screens get replaced. We
  work from published information and it can be out of date.
- **A claim about audio quality, projection quality, or print condition.** We score geometry, not
  the venue's maintenance.

### Our data has real limits, and here they are

We'd rather tell you than have you discover it:

- **Coverage is limited.** We have seat data for a small number of auditoriums, hand-entered from
  published seating charts. Most cinemas aren't covered.
- **Some auditorium dimensions are estimated**, not measured. Where a screen dimension is an
  estimate, we say so.
- **Seat availability is currently simulated**, not live. Where availability is simulated, we
  display that clearly. **Do not rely on simulated availability** to decide whether a showtime is
  full.
- **Showtimes may be out of date.** They're entered manually and cinemas change their schedules.
  Always confirm on the theatre's own site.

Our policy on where data comes from — including that we never scrape — is public:
[`data-sources-policy.md`](data-sources-policy.md).

---

## 4. "Reserve to pay in person" — read this carefully

When you choose to reserve rather than buy online, GoldSeats **saves your seat selection to your
account so you can look it up later.**

**It does not hold the seat. It does not reserve anything. It does not communicate with the
theatre in any way.**

We cannot hold a seat, because we don't sell seats. Someone else may buy the seats you picked at
any moment, including thirty seconds after you save them. To actually secure a seat, buy a ticket —
online through the theatre, or at their box office.

We've written this section bluntly on purpose. A feature called "reserve" invites a reasonable
misunderstanding, and we'd rather be clear than technically accurate.

---

## 5. The seat assistant

Our conversational assistant is a self-hosted AI model. Every seat it mentions comes from our
scoring engine and is verified before it reaches you — it cannot invent a seat that doesn't exist.

That said:

- It can misunderstand you.
- Its explanations are generated text. They reflect our scoring factors, and they can be phrased
  imprecisely.
- It's limited to seats, showtimes, and auditorium layouts. It will decline anything else.
- It may be unavailable. When it is, the seat map still works.
- **Don't send it personal information.** It doesn't need any, and conversations are stored on your
  account.

Your conversations are stored on our own servers and are not sent to any third-party AI provider.
See [`privacy-policy.md`](privacy-policy.md).

---

## 6. Your account

- You must be **13 or older** to create an account. **[LEGAL REVIEW]** — confirm threshold.
- Give us accurate information, and keep your email current so password reset works.
- Keep your password secure. You're responsible for activity under your account.
- Tell us at `security@goldseats.app` if you think your account has been compromised.
- One account per person. Don't share it.
- You can delete your account at any time, from Settings.

We may suspend or terminate an account that violates these terms. Where we reasonably can, we'll
tell you why first.

---

## 7. Acceptable use

**Do:**

- Use GoldSeats to find good seats. That's what it's for.
- Tell us when something's wrong — a layout that doesn't match reality is a bug we want to hear
  about.
- Report security issues responsibly, per [`../SECURITY.md`](../SECURITY.md).

**Don't:**

- **Scrape GoldSeats**, or access it by automated means beyond normal browsing. We hold ourselves to
  this standard with other people's sites, and we ask the same. If you want our data, email us — we
  may well say yes.
- Attempt to access another user's account, data, or saved picks.
- Circumvent rate limits, authentication, or any security control.
- Overload the service, or run load tests against production.
- Reverse engineer or attempt to extract our scoring model.
- Use the seat assistant to attempt prompt injection, extract our system prompt, or generate content
  unrelated to seats.
- Misrepresent yourself as GoldSeats, or imply we endorse you.
- Use GoldSeats for anything illegal.

We may rate-limit, suspend, or block access for any of the above.

---

## 8. Our content and yours

### Ours

The GoldSeats name, logo, design, seat scoring model, and the software are ours or our licensors'.

Our **source code is MIT licensed** and public — see [`../LICENSE`](../LICENSE). That licence covers
the code, not the GoldSeats name or logo.

### Third parties'

- Film metadata, posters, and images are from TMDB and belong to their respective owners. *This
  product uses the TMDB API but is not endorsed or certified by TMDB.*
- News headlines and links belong to their publishers. We show a headline, a link, attribution, and
  a short summary — never the full article.
- Theatre names and trademarks belong to those companies.

If you're a rights holder and believe we've used something improperly, email
`legal@goldseats.app` and we'll respond promptly. **[LEGAL REVIEW]** — confirm the notice-and-
takedown process appropriate to Canadian law.

### Yours

Your watchlist, follows, ratings, and chat messages are yours. You grant us only the licence needed
to operate the service for you — storing it, showing it back to you, and using it to generate your
feed and notifications.

We use aggregated, de-identified information about which seats people pick to improve our scoring
model. That never identifies you.

---

## 9. Availability

We aim to keep GoldSeats running and we're not promising perfection.

- No uptime SLA. We're a small team; we'll be honest rather than aspirational.
- We may change, suspend, or discontinue features. For a significant change we'll give reasonable
  notice.
- We may perform maintenance, occasionally without notice.
- We depend on third parties — hosting, TMDB, theatre booking sites. Their outages affect us.

---

## 10. Disclaimers

**[LEGAL REVIEW]** — the whole section, with attention to Ontario's *Consumer Protection Act*,
which limits what can be disclaimed for consumers.

GoldSeats is provided **"as is" and "as available"**, without warranties of any kind, express or
implied, including merchantability, fitness for a particular purpose, and non-infringement.

Specifically, we don't warrant that:

- Seat recommendations will match your preferences
- Seat availability, showtimes, or auditorium layouts are accurate or current
- A seat we recommend will be available when you try to buy it
- The service will be uninterrupted or error-free
- The 3D preview precisely represents what you'll see

Nothing here limits rights you have under applicable consumer protection law that cannot be
limited.

---

## 11. Limitation of liability

**[LEGAL REVIEW]** — enforceability and appropriate cap.

To the fullest extent permitted by law, GoldSeats is not liable for indirect, incidental, special,
consequential, or punitive damages, or for lost profits, data, or goodwill.

We are specifically not liable for:

- A ticket purchase you made on a theatre's website
- A seat that turned out to be worse than you expected
- A seat being unavailable when you tried to buy it
- A showtime that was cancelled, moved, or never existed
- Anything that happens on a third-party site we linked to

Our total liability for any claim is limited to **CAD $50**, or the amount you paid us in the
preceding 12 months, whichever is greater. GoldSeats is currently free, so in practice that's CAD
$50.

Nothing here excludes liability that cannot be excluded by law, including for fraud or for death or
personal injury caused by negligence.

---

## 12. Indemnity

You agree to indemnify GoldSeats against claims arising from your misuse of the service or your
violation of these terms.

**[LEGAL REVIEW]** — confirm appropriateness and scope for a consumer service.

---

## 13. Governing law

These terms are governed by the laws of the Province of Ontario and the federal laws of Canada.
Disputes are subject to the exclusive jurisdiction of the courts of Ontario.

**[LEGAL REVIEW]** — confirm venue, and whether arbitration or a class-action waiver is appropriate
or enforceable here.

---

## 14. Changes

We'll update this page and change the "last updated" date. For material changes we'll give at least
**14 days' notice** by email to account holders. Continuing to use GoldSeats after a change means
you accept it; if you don't, delete your account.

Version history is in this repository's git log.

---

## 15. Other

- **Severability.** If a provision is unenforceable, the rest stands.
- **No waiver.** Not enforcing a provision doesn't waive it.
- **Entire agreement.** These terms plus [`privacy-policy.md`](privacy-policy.md) are the whole
  agreement between us.
- **Assignment.** You can't assign these terms. We may, in connection with a merger or acquisition,
  with notice.
- **Language.** **[LEGAL REVIEW]** — Quebec's *Charter of the French Language* may require a French
  version. Our primary market is Canada and we serve French-language film data; assess whether a
  French translation is legally required and, separately, whether we want one regardless.

---

## 16. Contact

**General and legal:** `legal@goldseats.app`
**Privacy:** `privacy@goldseats.app`
**Security:** `security@goldseats.app`
**Conduct:** `conduct@goldseats.app`

**[LEGAL REVIEW]** — registered mailing address required.

---

## Notes for the team

Not part of the published terms:

- **Section 4 is the highest-risk copy in the product.** The UI wording must match it exactly:
  "We've saved your pick. This doesn't hold the seats — buy at the box office to secure them." Any
  UI copy that softens this is a trust bug and a legal exposure, and it's called out in
  [M5](../product/milestones/M5.md)'s deliverables.
- **Section 3's data limits must stay true.** When we get real availability from a partner, this
  section changes in the same release. When coverage expands, so does the claim.
- **Section 7's anti-scraping clause** should read consistently with how we behave toward others —
  see [`data-sources-policy.md`](data-sources-policy.md). Asking others not to scrape us while
  scraping them would be indefensible, which is one more reason [ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md)
  matters.
- Both this and the privacy policy get published as real web routes and linked from the footer, a
  [M9](../product/milestones/M9.md) deliverable.
