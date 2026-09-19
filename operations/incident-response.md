# Incident Response

An incident is any unplanned event that degrades the service for users, or any confirmed
security vulnerability in a production system.

This document is deliberately written for a very small team. It scales down honestly — some
roles collapse into one person — but the **structure** matters even then, because the point of a
process is that you don't have to invent one while things are on fire.

---

## Severity levels

Assign a severity in the first two minutes. Getting it roughly right immediately beats getting
it exactly right in twenty.

### Sev-1 — Critical

**Users cannot use GoldSeats, or user data is at risk.**

- API down, or 5xx on the majority of requests
- Database unreachable or corrupted
- `goldseats.app` not resolving or serving TLS errors
- Data loss, or exposure of user data
- A confirmed exploited security vulnerability

**Response:** immediate, drop everything, any hour. Acknowledge within 15 minutes.

### Sev-2 — Major

**A core feature is broken, but the product is partially usable.**

- `POST /v1/recommendations` failing — **the product's core function**
- Seat maps rendering wrong or empty for seeded auditoriums
- Booking handoff producing broken or wrong URLs
- Auth broken; users can't sign in
- Error rate above 2% sustained
- p95 latency more than 3× budget

**Response:** within 30 minutes during working hours, within 1 hour otherwise.

### Sev-3 — Minor

**Degraded, with a workaround, or affecting a non-core feature.**

- Chatbot down — degrades to the seat map, which is a complete feature
- 3D seat view failing — 2D map still works
- TMDB ingestion failing; the catalog goes stale
- Notifications delayed
- Elevated but tolerable latency
- `chat_ungrounded_reply_blocked_total` non-zero

**Response:** next business day.

### Sev-4 — Low

**Cosmetic, or affecting one user.**

**Response:** file an issue. Not an incident.

### Severity is a live judgement

Upgrade freely. A Sev-3 that turns out to be a symptom of a database problem is a Sev-1 the
moment you realise that. Nobody is ever criticised for over-escalating and then standing down.

Downgrade only once you actually understand the impact — not because the graph looked better for
five minutes.

---

## Roles

In a one- or two-person team one person holds all of these, which is fine. Naming them still
helps, because it reminds you which hat you've forgotten to put on — usually Communications.

| Role | Responsibility |
| --- | --- |
| **Incident Commander (IC)** | Owns the incident. Decides severity, assigns work, makes the call on rollback. **Does not debug.** The IC's job is coordination, and an IC head-down in a stack trace is an incident with nobody steering. |
| **Operations Lead** | Does the technical work: investigates, mitigates, fixes. |
| **Communications Lead** | Updates the status page and users. Keeps the timeline. |
| **Scribe** | Timestamps everything in the incident channel. Feeds the post-incident review. |

When one person holds all four: be the IC first. Set a five-minute timer, and each time it
fires, step back and ask "what's the next decision, and have I told anyone?"

---

## The process

### 1. Detect

- An alert fires — see the table in
  [`../engineering/observability.md`](../engineering/observability.md)
- A user reports something
- You notice it

### 2. Declare — within 2 minutes

Say it out loud, in writing, somewhere durable. For a solo operator that's an issue titled
`[Sev-2] POST /v1/recommendations returning 500s`, opened immediately.

```
[Sev-2] POST /v1/recommendations returning 500s
Declared: 2026-07-14 19:42 EDT
IC: Barun
Impact: Users on a seat map get an error instead of recommendations.
        Browsing and film pages unaffected.
Status: Investigating
```

Declaring early costs nothing. An incident that turns out to be minor gets closed in ten
minutes; an incident nobody declared gets debugged for an hour with no record of what was tried.

### 3. Assess

- What's the user-visible impact, in plain words?
- How many users? All of them, or just one flow?
- When did it start? Correlate with the last deploy — **the answer is a recent deploy far more
  often than anything else.**
- Is it getting worse?

```bash
# What's deployed?
curl -s https://api.goldseats.app/health/ready | jq .version

# When did it last deploy?
gh run list --repo goldseats/goldseats-api --workflow deploy.yml --limit 3

# What's the error?
# → Sentry, filtered by release. → Service health dashboard. → Logs by request_id.
```

### 4. Mitigate before you fix

**Stop the bleeding first. Understanding can wait.**

In order of preference:

1. **Roll back.** If a deploy correlates with the start, roll it back. Two minutes, well
   understood, almost always right. → [`runbooks/api-rollback.md`](runbooks/api-rollback.md)
2. **Turn off the feature.** If it's flag-gated, flip the flag.
3. **Scale or restart.** For resource exhaustion. Buys time, doesn't fix anything.
4. **Degrade deliberately.** Disable the chatbot, disable the 3D view — both have designed
   fallbacks, and shedding them is cheap.
5. **Fix forward.** Only when rollback isn't possible, and only with a minimal diff.

Resist the urge to understand first. The post-incident review is where understanding happens,
and it's much more comfortable there.

### 5. Communicate

| Severity | Who | How often |
| --- | --- | --- |
| Sev-1 | Users, via the status page and the app | On declaration, every 30 min, on resolution |
| Sev-2 | Users, if a core flow is affected | On declaration, every hour, on resolution |
| Sev-3 | Internal only, unless a user asked | On resolution |

What a good update looks like:

```
19:42 — Investigating. Seat recommendations are failing for some users.
        Browsing films and showtimes is unaffected.
19:51 — Cause identified: a change in the last release. Rolling back.
19:58 — Rolled back. Recommendations are working again. Monitoring.
20:15 — Resolved. Full details to follow.
```

Rules for incident comms:

- **Say what users can and can't do.** That's the only thing they need from you.
- Don't speculate about cause in public until you know.
- Don't say "we're aware of the issue" with nothing else. That's noise.
- Don't blame a vendor, even when it's a vendor. It reads as deflection.
- Never promise a time you can't hold.

### 6. Resolve

Resolved means users are fine again, not that the root cause is understood.

```
Resolved: 2026-07-14 20:15 EDT
Duration: 33 minutes
Impact: ~40 minutes of failed recommendation requests. No data loss.
Fix: Rolled back to v0.5.2.
Follow-up: #147 (root cause), #148 (regression test)
```

### 7. Review

Within 48 hours for Sev-1 and Sev-2. Blameless, written down, and it must produce action items
with owners.

---

## Post-incident review template

```markdown
# Incident 2026-07-14 — Recommendations returning 500s

- **Severity:** Sev-2
- **Duration:** 33 minutes (19:42–20:15 EDT)
- **IC:** Barun
- **User impact:** Seat recommendations failed for all users. Browsing unaffected.
  No data loss. Estimated ~120 failed requests.

## Timeline
| Time | Event |
| --- | --- |
| 19:38 | v0.5.3 deployed to production |
| 19:42 | Alert: error rate above 2%. Incident declared. |
| 19:47 | Sentry shows KeyError in scoring factor lookup, only on IMAX layouts |
| 19:51 | Correlated with v0.5.3's weight-key rename. Decision: roll back. |
| 19:58 | v0.5.2 deployed. Errors stop. |
| 20:15 | 15 minutes clean. Resolved. |

## What happened
A weight key was renamed in `SCORING_WEIGHTS` but one call site still referenced the old
key. The path only executes for auditoriums with a premium centre section, so no seeded
standard auditorium exercised it and the test suite didn't cover the IMAX fixture for that
code path.

## Why it wasn't caught
- Golden snapshot tests cover IMAX layouts but not this specific preference combination.
- mypy didn't catch it because the lookup is a `dict[str, float]` access.

## What went well
- The alert fired within 4 minutes.
- Rollback took 7 minutes from decision to clean.
- The `request_id` in the error response made log correlation immediate.

## What didn't
- The weight rename should have been a `Literal` type, not a string key.
- No canary or staged rollout, so it hit 100% of users at once.

## Action items
| Action | Owner | Issue |
| --- | --- | --- |
| Make factor keys a `StrEnum` so mypy catches this class of bug | Barun | #147 |
| Add the IMAX × premium-centre preference combination to golden fixtures | Barun | #148 |
| Add a scoring smoke test to the production deploy smoke suite | Barun | #149 |

## Not doing
Canary deploys. Traffic is low enough that instant rollback is a better investment. Revisit
at higher volume.
```

### Blameless means blameless

The question is always "what about the system allowed this?", never "who did this?" A human
typing the wrong dictionary key is not a finding — a type system that permitted it, and a test
suite that didn't exercise the path, are findings.

If a review produces no action items, either it wasn't an incident or the review wasn't honest.

---

## Security incidents

A confirmed vulnerability in production is at minimum a Sev-2. Exposure of user data is a Sev-1.

Additional steps beyond the normal process:

1. **Don't discuss it in a public channel.** All three repos are public; a public issue is a
   disclosure.
2. **Preserve evidence** before you remediate. Snapshot logs and database state — you'll need
   them to establish scope, and remediation destroys them.
3. **Rotate any credential that might be affected.** First, before investigating further. See
   [`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).
4. **Establish scope**: what data, whose, for how long, and did anyone actually access it?
5. **PIPEDA breach notification.** Canadian law requires notifying the Privacy Commissioner and
   affected individuals for a breach creating a real risk of significant harm. **Involve a
   lawyer.** Do not make this call alone.
6. Coordinate disclosure per [`../SECURITY.md`](../SECURITY.md) — 48h acknowledgement, 90-day
   disclosure.
7. The post-incident review is private until disclosure.

A reminder that genuinely narrows the scope of most security incidents: **we hold no payment
data.** Booking is a deep-link handoff and no card data exists in our systems. What we do hold
is emails, hashed passwords, session tokens, and activity — see
[`../legal/privacy-policy.md`](../legal/privacy-policy.md).

---

## Escalation

Rotation and contacts are in [`on-call.md`](on-call.md).

Escalate when:

- A Sev-1 isn't mitigated within 30 minutes
- You don't know what to do next
- You need a decision you can't make alone — customer comms, legal, spending money
- **You've been at it for two hours.** Fatigue causes second incidents, and handing off is a
  technical decision, not an admission of anything.

---

## Runbooks

Follow these rather than improvising:

| Situation | Runbook |
| --- | --- |
| Bad deploy, API errors | [`runbooks/api-rollback.md`](runbooks/api-rollback.md) |
| Data loss or corruption | [`runbooks/database-restore.md`](runbooks/database-restore.md) |
| DNS, TLS, or domain problems | [`runbooks/domain-cutover.md`](runbooks/domain-cutover.md) |
| All runbooks | [`runbooks/README.md`](runbooks/README.md) |

If you hit a situation with no runbook, **write one during or immediately after the incident.**
That's when you know the most and it costs the least.

---

## Practice

Untested incident response is a document, not a capability. Quarterly, in staging:

- Roll back a release and time it
- Restore the database from a backup and time it
- Deliberately trigger every alert and confirm it routes
- Read a runbook start to finish and fix whatever's now wrong in it

The first full round is a [M9](../product/milestones/M9.md) deliverable.
