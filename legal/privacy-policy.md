# Privacy Policy

**Last updated: 2026-02-09**
**Effective: on launch of `goldseats.app`**

> ⚠️ **This is a substantive draft and it is not legal advice.** It must be reviewed by a lawyer
> qualified in Canadian privacy law (PIPEDA, and Quebec's Law 25 if we serve Quebec users) before
> it is published. That review is a required deliverable of
> [M9](../product/milestones/M9.md). Sections needing specific legal attention are marked
> **[LEGAL REVIEW]**.

---

## The short version

- We hold your email, your password hash, what you follow, your watchlist, your saved seat picks,
  and your chat conversations with us.
- **We never see your payment information.** Buying a ticket happens entirely on the theatre's own
  website.
- We don't sell your data. We don't run advertising.
- Your conversations with our seat chatbot stay on our own servers — we don't send them to any AI
  company.
- You can export everything and delete everything, whenever you want.

The rest of this document is the detail.

---

## 1. Who we are

GoldSeats operates `goldseats.app`, a service that recommends cinema seats and links you to the
theatre's own booking page.

**Contact:** `privacy@goldseats.app`
**Security issues:** `security@goldseats.app` — see [`../SECURITY.md`](../SECURITY.md)

**[LEGAL REVIEW]** We need to state our legal entity name, jurisdiction of incorporation, and
registered address, and name a Privacy Officer as PIPEDA requires.

---

## 2. What we collect

### You give us

| Data | When | Why |
| --- | --- | --- |
| Email address | Signup | Account identity, notifications, password reset |
| Password | Signup | Authentication. Stored only as an Argon2id hash — we cannot read it. |
| Display name | Optional | Personalising the interface |
| Preferred language | Optional | Defaulting the language filter |
| Preferred city | Optional | Defaulting "playing nearby" |
| OAuth identity | If you sign in with Google | Authentication. We receive a subject ID and email, not your password. |

### You create by using GoldSeats

| Data | Why |
| --- | --- |
| Films, theatres, and people you follow | Building your feed and sending notifications you asked for |
| Watchlist and "seen" markers | The feature you're using |
| Your own private ratings | Your reference only. **Never aggregated, never shown to anyone else.** |
| Seat recommendations you requested — showtime, party size, preferences, seats we suggested | Showing you the result, and letting you return to a saved pick |
| Seats you actually chose | Improving our scoring. Where our recommendation and your choice differ is the most useful signal we get. |
| Chat conversations with our seat assistant | Conversation continuity and debugging |
| Push notification subscription | Sending the notifications you enabled |

### We collect automatically

| Data | Retention | Why |
| --- | --- | --- |
| IP address | 30 days in logs | Security, abuse prevention, rate limiting |
| Browser and device type | 30 days | Debugging, and choosing the 3D quality tier |
| Pages visited, timestamps | 30 days | Debugging and aggregate usage |
| Request IDs and error traces | 90 days | Debugging. Tied to a user ID, never an email. |
| Coarse location | Not stored | **Only if you grant permission.** Used in the request to find nearby showtimes, then discarded. |

### What we deliberately don't collect

- **Payment card data.** Never. Not once. Booking is a link to the theatre's own site, and the
  transaction happens entirely on their infrastructure. We are out of PCI scope entirely.
- **Precise location history.** We don't build a movement profile.
- **Contacts, photos, or anything else from your device.**
- **Cross-site tracking identifiers.** No advertising pixels, no third-party trackers.
- **Anything about children.** See section 10.

---

## 3. What we don't do with it

Stated plainly because these are the questions people actually have:

- **We don't sell your data.** Not to anyone, for any amount.
- **We don't run advertising** and we don't share data with ad networks.
- **We don't share your data with theatres.** When you click through to book, they see a normal web
  visitor arriving with a showtime and seat selection in the URL. They don't get your GoldSeats
  account, your email, or your history.
- **We don't send your conversations to an AI company.** Our seat assistant runs on a model we host
  ourselves. Your messages never leave our infrastructure. This was a deliberate architectural
  choice — see [ADR-0004](../architecture/adr/0004-self-hosted-llm.md).
- **We don't use dark patterns** on notification consent or deletion. Off is as easy as on.

---

## 4. Legal basis and consent

Under PIPEDA we rely on your consent, which is:

- **Express** for creating an account, enabling notifications, and granting location access
- **Implied** for the minimum processing needed to serve a page you requested

You can withdraw consent at any time by turning off a feature or deleting your account. Some
withdrawals mean a feature stops working — turning off email means no release notifications.

**[LEGAL REVIEW]** If we serve users in Quebec, Law 25 adds requirements: a designated privacy
officer, privacy impact assessments, and specific consent mechanics. Confirm applicability and
scope.

**[LEGAL REVIEW]** GDPR applies if we have EU users. Our primary market is Canada, but the site is
publicly reachable. Decide whether to geo-restrict, or to comply. This affects the lawful-basis
framing throughout this document.

---

## 5. Cookies and local storage

We use as little as we can get away with.

| What | Type | Purpose | Duration |
| --- | --- | --- | --- |
| Session token | Essential | Keeping you signed in | 15 min access, 30 day refresh |
| CSRF token | Essential | Security | Session |
| Language and theme preference | Essential | Remembering your choices | 1 year |
| 3D quality tier | Functional | Not re-detecting your device every visit | 30 days |

**No analytics cookies. No advertising cookies. No third-party cookies.**

We don't currently need a consent banner because everything above is strictly necessary or a
preference you set. **If that ever changes, the banner comes before the cookie.**

We self-host our fonts rather than loading them from Google, which removes a third-party
connection on every page load.

---

## 6. Who we share with

Only service providers doing work for us, under contract, with no right to use your data for their
own purposes.

| Provider | What they process | Where |
| --- | --- | --- |
| Web hosting | Page requests, IP addresses | **[LEGAL REVIEW]** — confirm region once the host is chosen |
| API and database hosting | All stored data | **[LEGAL REVIEW]** — confirm region |
| Email delivery | Your email address and message content | **[LEGAL REVIEW]** — name the provider |
| Error monitoring (Sentry) | Error traces with a user ID. **Configured with `send_default_pii=False`** — no emails, no IPs. | **[LEGAL REVIEW]** |
| TMDB | Nothing about you. We fetch film data; they don't see you. | — |

**[LEGAL REVIEW]** Data residency matters under PIPEDA and more so under Law 25. Ideally
Canadian-region hosting. If any provider stores data in the US, this section must say so
explicitly, and Quebec users may need specific notice.

We will also disclose data if legally required — a court order or lawful request. If we receive
one, we'll notify you unless legally prohibited.

---

## 7. How long we keep things

| Data | Retention |
| --- | --- |
| Account, follows, watchlist | Until you delete your account |
| Chat conversations | Until you delete them, or your account |
| Seat recommendations tied to you | Until you delete your account. Then **detached, not deleted** — the row survives with no user attached. |
| Server logs | 30 days |
| Error traces | 90 days |
| Database backups | 30 days |
| Inactive accounts | **[LEGAL REVIEW]** — propose: notice after 3 years of inactivity, deletion after 3 years 6 months |

Detaching rather than deleting recommendations deserves an explanation: a recommendation row with
no user attached contains a showtime, a party size, some seat scores, and which seats were chosen.
That's useful for improving the scoring model and contains nothing that identifies you. If you'd
rather it were deleted outright, email us and we will.

---

## 8. Your rights

Under PIPEDA you can:

| Right | How |
| --- | --- |
| **Know** what we hold | This document, and the export below |
| **Access** it | Settings → Privacy → Export my data. JSON, immediately. |
| **Correct** it | Edit it in Settings, or email us |
| **Withdraw consent** | Turn the feature off |
| **Delete** it | Settings → Delete my account |
| **Complain** | `privacy@goldseats.app`. If we don't resolve it, the Office of the Privacy Commissioner of Canada. |

We respond to any request within **30 days**, as PIPEDA requires. Usually much faster — export and
deletion are self-service and immediate.

### What deletion actually does

Not a flag. Real deletion, in one transaction:

- Your `users` row is **hard deleted.** No tombstone holding your email forever.
- Follows, watchlist entries, notification records, and push subscriptions are deleted.
- Chat conversations and messages are deleted.
- Seat recommendations are detached — `user_id` set to null.
- Backups still contain your data until they age out, within 30 days. We can't surgically edit a
  backup, and we're telling you rather than pretending otherwise.

There's a test in our codebase that deletes an account and asserts nothing identifying survives.
It runs on every change.

---

## 9. Security

- Passwords hashed with **Argon2id**. We cannot read your password.
- HTTPS everywhere, HSTS enabled.
- Short-lived access tokens (15 minutes) with rotating refresh tokens.
- Database encrypted at rest, backups encrypted.
- **No payment data exists in our systems**, so it cannot be breached.
- Logs deliberately exclude emails, tokens, passwords, and precise location. A scrubbing filter
  runs as a backstop.
- Access to production data is restricted and logged.
- Security review before launch; vulnerability reporting per [`../SECURITY.md`](../SECURITY.md).

No system is perfectly secure. If a breach creates a real risk of significant harm, PIPEDA requires
us to notify you and the Privacy Commissioner, and we will — see
[`../operations/incident-response.md`](../operations/incident-response.md).

---

## 10. Children

GoldSeats is not directed at children under 13, and we don't knowingly collect their data. If you
believe a child has created an account, email `privacy@goldseats.app` and we'll delete it.

**[LEGAL REVIEW]** Cinema seat selection is plausibly of interest to teenagers, and our "family"
party scenario acknowledges families use it. Confirm the appropriate age threshold and whether any
additional obligations apply.

---

## 11. Third-party links

Booking takes you to the theatre's own website. **Once you leave GoldSeats, their privacy policy
applies, not ours.** We don't control what they collect and we'd encourage you to read their
policy — especially since that's where you'll enter payment details.

Film news items link to publishers' sites. Same applies.

---

## 12. Changes

We'll update this page and change the "last updated" date. For material changes affecting how we
use your data, we'll email you at least 14 days before it takes effect and — where consent is
required — ask again rather than assuming.

Version history is in this repository's git log, so you can see exactly what changed and when.

---

## 13. Contact

**Privacy questions and requests:** `privacy@goldseats.app`
**Security vulnerabilities:** `security@goldseats.app`
**Conduct concerns:** `conduct@goldseats.app`

**[LEGAL REVIEW]** Named Privacy Officer and registered mailing address required.

If you're unsatisfied with our response, you can contact the **Office of the Privacy Commissioner
of Canada** at `priv.gc.ca`.

---

## Notes for the team

Not part of the published policy — engineering obligations that follow from it:

- **This document must describe reality.** Before launch, walk the schema in
  [`../architecture/data-model.md`](../architecture/data-model.md) table by table and confirm every
  category of personal data appears here. A [M9](../product/milestones/M9.md) task.
- **Every new data category needs a PR here**, in the same change that starts collecting it. It's
  on the [definition of done](../engineering/definition-of-done.md).
- The deletion test is not optional. If it starts failing, we're out of compliance, not just out of
  test coverage.
- Sentry's `send_default_pii=False` must never be flipped on.
- The self-hosted LLM is a privacy commitment now, not just an architectural preference. Moving to
  a hosted API would require changing this document and re-obtaining consent — factor that into any
  future ADR that reopens [ADR-0004](../architecture/adr/0004-self-hosted-llm.md).
