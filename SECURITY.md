# Security Policy

GoldSeats runs a public codebase and a public web app at `goldseats.app`. We take
security reports seriously and we'd rather hear about a problem from you than from our
users.

---

## We hold no payment data

This matters for scoping any report, so it's first:

**GoldSeats never processes, transmits, or stores payment card data.** Booking is a
deep-link handoff — we send the user to the theatre's own booking and payment page with
the showtime and seat selection encoded in the URL. The transaction happens entirely on
the theatre's infrastructure. We are therefore **out of PCI DSS scope entirely**, and
there is no cardholder data in our database, logs, caches, or backups.

If you find something that looks like card data in a GoldSeats system, that is itself a
critical bug — report it immediately using the process below.

What we *do* hold, and what is therefore in scope for a report:

- Account identity: email address, hashed password (Argon2id), OAuth subject IDs
- Session material: JWT access and refresh tokens
- User activity: followed films and theatres, watchlist, "seen" markers, saved seat
  picks, chatbot conversation history
- Coarse location, only when the user grants it, to find nearby showtimes

See [`legal/privacy-policy.md`](legal/privacy-policy.md) for the full data inventory.

---

## Supported versions

GoldSeats is a continuously deployed web service, not distributed software. There is
exactly one supported version of each component: whatever is currently running in
production, deployed from `main`.

| Component | Supported | Notes |
| --- | --- | --- |
| `goldseats-api` — current production deploy from `main` | ✅ Supported | Fixed forward; see [`operations/release-process.md`](operations/release-process.md) |
| `goldseats-web` — current production deploy from `main` | ✅ Supported | Fixed forward |
| Staging environments | ✅ In scope for reports | Not user data, but still report it |
| Any tagged release older than current production | ❌ Not supported | Tags exist for rollback audit only |
| The legacy `barung1/goldseats` repo and its Netlify deploy | ❌ Not supported | Archived; superseded by the three-repo org. See [`operations/runbooks/domain-cutover.md`](operations/runbooks/domain-cutover.md) |

We do not backport fixes. A security fix lands on `main` and deploys.

---

## Reporting a vulnerability

**Do not open a public GitHub issue for a security vulnerability.** Our repos are
public, so an issue is a disclosure.

Use either of these, whichever you prefer:

1. **GitHub private vulnerability reporting** — on the affected repository, go to the
   **Security** tab → **Report a vulnerability**. This is enabled on all three repos and
   is our preferred channel, because it keeps the report, the discussion, the fix, and
   the eventual advisory in one place.
2. **Email `security@goldseats.app`** — if you'd rather not use GitHub, or the issue
   spans more than one repo, or it concerns infrastructure rather than code.

If you want to encrypt the email, ask for our current PGP key in a first contact
message with no sensitive detail in it.

### What to include

The more of this you can give us, the faster we can fix it:

- The component: `goldseats-api`, `goldseats-web`, or infrastructure
- Environment and URL, e.g. `https://api.goldseats.app/v1/recommendations` in production
  vs staging
- A description of the vulnerability and the impact you think it has
- Reproduction steps, ideally a minimal `curl` or a short script
- Any request/response pairs, log excerpts, or screenshots — **with your own
  credentials redacted**
- Whether you believe the issue has been exploited, and whether anyone else knows about it

---

## Our commitments

| Stage | Target |
| --- | --- |
| Acknowledge your report | **Within 48 hours** |
| Initial triage and severity assignment | Within 5 business days |
| Status updates while we work | At least every 7 days |
| Fix deployed for Critical and High severity | As fast as we can, target 7 days |
| Fix deployed for Medium and Low severity | Target 30 days |
| Public disclosure | **Within 90 days** of the report, or sooner once a fix is deployed — coordinated with you |

We use the severity definitions in [`operations/incident-response.md`](operations/incident-response.md).
A confirmed vulnerability affecting production user data is a Sev-1 and triggers that
process.

We will:

- Tell you honestly whether we consider the report valid, and why if we don't
- Credit you in the resulting GitHub Security Advisory unless you'd rather stay
  anonymous
- Not take legal action against you for research conducted in good faith under the
  rules below

We will not:

- Pay a bounty. We're pre-launch and have no bounty programme. If that changes, this
  file changes with it.
- Ask you to sign an NDA as a condition of us fixing the bug.

---

## Safe harbour and rules of engagement

Research conducted in good faith and consistent with this policy is authorised, and we
will not pursue or support any legal action related to it.

**Please do:**

- Test against your own accounts and your own data
- Test against staging where you can
- Stop as soon as you've confirmed a vulnerability, and report it
- Give us reasonable time to fix it before telling anyone else

**Please do not:**

- Access, modify, download, or delete data belonging to another user
- Run automated scanners, fuzzers, or brute-force tooling against production. Our
  rate limits will stop you and the noise makes real attacks harder to see.
- Perform denial-of-service or load testing of any kind against production
- Use social engineering, phishing, or physical attacks against GoldSeats, our
  contributors, or our vendors
- Attack third parties we integrate with. **TMDB and the theatre chains' booking sites
  are explicitly out of scope** — report issues there to those companies directly.
- Publicly disclose before the coordinated date

---

## Out of scope

Reports about the following will be closed as informative without a fix:

- Missing security headers with no demonstrated exploit path
- Clickjacking on pages with no state-changing action
- Self-XSS requiring the victim to paste code into their own console
- Email enumeration on signup or password reset that matches our documented,
  intentional behaviour
- Vulnerabilities in outdated browsers or in software we do not run
- Rate-limiting complaints on unauthenticated read endpoints that are intentionally
  cached and public, such as `/v1/films/now-playing`
- Findings from an automated tool pasted in with no validation and no impact analysis
- Anything in the archived legacy repo or its abandoned Netlify deploy

---

## Our side of the deal

Security practices we hold ourselves to, documented in full elsewhere:

- No secrets in git, ever. Only `.env.example` is committed — see
  [`engineering/secrets-and-config.md`](engineering/secrets-and-config.md).
- Dependabot enabled on all three repos; security updates are merged on sight.
- Branch protection on `main` in all repos: required CI, one approval, linear history.
- Parameterised queries only, via SQLAlchemy 2.0. No string-built SQL — see
  [`engineering/database-conventions.md`](engineering/database-conventions.md).
- The chatbot is confined to tool calls into our own scoring and lookup endpoints and
  cannot reach arbitrary URLs or execute code — see
  [ADR-0004](architecture/adr/0004-self-hosted-llm.md).
- Full security review is a required deliverable of [M9](product/milestones/M9.md)
  before launch.
