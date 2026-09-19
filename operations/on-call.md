# On-Call

Who responds when something breaks, and what's reasonable to expect of them.

Right now GoldSeats is one engineer. This document describes that honestly rather than
pretending there's a rotation, and it specifies what changes when a second person joins —
because the moment to agree those terms is *before* someone is woken up, not after.

---

## Current state

**Rotation:** none. The founder is the sole responder.

**What that actually means, stated plainly:**

- There is no 24/7 coverage and we should not claim any.
- Alerts route to one phone. If that phone is off, nobody is responding.
- Sev-1 response is "when the founder sees it", which overnight could be eight hours.

**What we do instead of coverage we don't have:**

1. **Degrade rather than fail.** Redis down is slower, not broken. The chatbot down falls back to
   the seat map. The 3D view down leaves the 2D map. Each of those is a designed fallback, and
   collectively they convert most would-be Sev-2s into Sev-3s that can wait for morning.
2. **Automate the response.** Health checks restart unhealthy instances. Failed deploys don't
   promote. Migrations are expand-only so a rollback never needs a human with a psql prompt.
3. **Keep the blast radius small.** No payment processing. Seat data is seeded and reproducible.
   The only genuinely irreplaceable data is user accounts and their follows, which is why
   backups and the restore drill matter more than uptime.
4. **Set honest expectations.** No SLA is published while there's one responder. The status page
   says what's happening; it doesn't promise a response time we can't hold.

This is an acceptable posture for a pre-launch product with no paying customers. It stops being
acceptable the day someone depends on us, which is the trigger for the rotation below.

---

## Expectations while on call

For whoever holds the pager, now or later.

**You are expected to:**

- Have your phone on, charged, with alerts un-silenced
- Acknowledge within the severity's target — 15 min Sev-1, 30 min Sev-2 in hours
- Be able to reach a laptop and the internet within 30 minutes
- Know where the runbooks are: [`runbooks/README.md`](runbooks/README.md)
- Have working access to the host dashboard, Sentry, the database, and DNS **before** your shift
  starts, verified, not assumed
- Declare incidents early and escalate freely

**You are not expected to:**

- Fix everything yourself. Rolling back and going back to bed is a correct, complete response.
- Understand the root cause during the incident. Mitigate; understand in the review.
- Work more than two hours straight on an incident. Hand off or stand down — fatigue causes
  second incidents.
- Do feature work while on call. If it's a quiet shift, do the reliability backlog.
- Be available during a documented handover gap. Say when you're unreachable and arrange cover.

**Compensation and time back:** anyone woken up outside working hours takes equivalent time off,
no discussion. A rotation that quietly costs people their evenings stops being staffed honestly,
and then the on-call doc is fiction.

---

## Alert routing

Alerts, conditions, and severities are in
[`../engineering/observability.md`](../engineering/observability.md#alerts). Routing:

| Severity | Channel | Hours |
| --- | --- | --- |
| Sev-1 | Push notification + phone call | Any hour |
| Sev-2 | Push notification | 08:00–23:00 local; queued overnight |
| Sev-3 | Email + a daily digest | Business hours |
| Sev-4 | GitHub issue only | — |

**Every alert must be actionable and must name its runbook.** If an alert fires and the correct
response is "yeah, that happens", it gets deleted or its threshold gets fixed — in a PR, that
week.

Alert fatigue is the failure mode that kills on-call. One ignorable alert teaches you to ignore
alerts, and the next one you ignore is the real one.

---

## What warrants waking someone up

| Wake someone | Wait until morning |
| --- | --- |
| API down | Chatbot down — the seat map still works |
| Database unreachable | 3D view broken — the 2D map still works |
| Data loss or corruption | TMDB ingestion failed — the catalog goes stale, slowly |
| User data exposed | Notifications delayed |
| `goldseats.app` not resolving or TLS broken | Elevated but tolerable latency |
| Error rate above 20% | A single user reporting a problem |
| Recommendations failing for everyone | `chat_ungrounded_reply_blocked_total` non-zero |

The test: **can a user still find a film, see showtimes, and get a seat recommendation?** If yes,
it waits.

---

## Handover

When a rotation exists, each shift change is a five-minute written handover:

```markdown
## Handover 2026-08-11

- **Open incidents:** none
- **Recent deploys:** api v0.7.2 (Fri 16:20), web v0.9.0 (Fri 16:35). Both clean.
- **Watching:** `recommendation_empty_total` up ~3% since v0.7.2 — likely the new
  contiguity check being stricter. Not user-impacting. Issue #203.
- **Known noise:** TMDB ingest warns about rate limiting around 03:00 daily. Expected;
  the job retries and completes.
- **Planned work:** none this weekend.
- **Unreachable:** Sat 14:00–17:00, cover arranged with <name>.
```

---

## Access checklist

Verify **before** your shift, not during an incident. Discovering an expired credential at 2am is
its own small disaster.

- [ ] Web host dashboard — MFA working
- [ ] API host dashboard
- [ ] Production database credentials, and a connection actually tested
- [ ] Redis access
- [ ] Sentry
- [ ] The metrics and dashboard tool
- [ ] DNS registrar — **this is the one people forget, and it's the one needed for the worst
      outage**
- [ ] GitHub org, with permission to run workflows
- [ ] Email provider
- [ ] Status page

---

## The rotation, when there is one

At two engineers:

- Weekly, Monday 10:00 to Monday 10:00
- Primary and secondary. Secondary is called only if primary doesn't acknowledge in 15 minutes.
- Swaps are fine, arranged directly, written in the handover.
- Nobody on call two weeks consecutively.

At three or more:

- Weekly, no more often than one week in three
- Follow-the-sun if anyone is in a distant timezone
- A published SLA, because now we can actually hold one

**Trigger for starting the rotation:** the first of these to happen —

1. Someone outside the founder depends on GoldSeats for something that matters
2. A second engineer joins
3. We publish an SLA
4. Revenue starts

---

## Reducing the load

The best on-call shift is one where nothing pages you, and getting there is engineering work, not
willpower.

Every incident's review must ask: **what would have made this not page a human?** Concretely, for
our system:

- More designed degradations. Every component that can fall back instead of failing converts a
  Sev-2 into a Sev-3.
- Better health checks and automatic restarts.
- Tighter alert thresholds — alerting on the symptom, not the cause.
- Expand-only migrations, always, so rollback never needs database surgery.
- Runbooks that are actually followed, which means they must be accurate, which means they get
  read and fixed quarterly.

If the same alert fires twice in a month, fixing the underlying cause takes priority over
whatever feature work was planned. That's a rule, not a preference — recurring pages are how a
team ends up with an unstaffable rotation.

---

## Related

- [`incident-response.md`](incident-response.md) — severities, roles, the process
- [`runbooks/README.md`](runbooks/README.md) — the runbook index
- [`../engineering/observability.md`](../engineering/observability.md) — alerts, dashboards, SLOs
- [`environments.md`](environments.md) — what runs where
- [`../SECURITY.md`](../SECURITY.md) — vulnerability reporting
