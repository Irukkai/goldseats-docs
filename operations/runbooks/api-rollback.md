# Runbook — API Rollback

**Last drilled:** not yet. First drill is a [M9](../../product/milestones/M9.md) deliverable.

**Target time:** under 10 minutes from decision to verified healthy.

---

## When to use this

- Error rate spiked after a deploy
- p95 latency jumped after a deploy
- A core flow broke after a deploy — recommendations failing, seat maps empty, booking links
  wrong
- Sentry is full of a new issue tagged with the current release
- `booking_handoff_total` dropped to zero after a deploy

**The heuristic: if a deploy correlates with the start of the problem, roll back. Don't debug
first.** A revert takes minutes and is well understood. A fix written under pressure is a second
incident waiting to happen, and you can investigate calmly once users are fine.

## When NOT to use this

- **The problem predates the last deploy.** Rolling back won't help and will waste ten minutes.
- **The problem is data, not code.** → [`database-restore.md`](database-restore.md)
- **The problem is DNS or TLS.** → [`domain-cutover.md`](domain-cutover.md)
- **A dependency is down** and its designed degradation is working. → Sev-3, see
  [`README.md`](README.md#designed-degradations)
- **The bad release contained a non-reversible migration.** Stop and read
  [Step 4](#step-4-check-the-migration-question) before doing anything.

---

## Prerequisites

- GitHub access with permission to run workflows in `goldseats/goldseats-api`
- `gh` authenticated
- Access to the service health dashboard and Sentry
- The incident declared — [`../incident-response.md`](../incident-response.md)

---

## Step 1 — Confirm what's running

```bash
curl -s https://api.goldseats.app/health/ready | jq '{status, version, checks}'
```

Note the version. That's what you're rolling back *from*.

## Step 2 — Confirm the deploy correlation

```bash
gh run list --repo goldseats/goldseats-api --workflow deploy.yml --limit 5
```

Compare the deploy time against when errors started on the dashboard.

**Decision point:**

- Errors started **within ~5 minutes after** a deploy → continue, roll back
- Errors started **before** the deploy → **stop.** This is not a deploy problem. Investigate.

## Step 3 — Identify the target version

```bash
cd goldseats-api
git fetch --tags
git tag --sort=-creatordate | head -5
```

The target is the tag before the current one. Confirm it was healthy in production — if the
previous release was also bad, go back one further.

```bash
export CURRENT=v0.5.3
export TARGET=v0.5.2
git log --oneline $TARGET..$CURRENT
```

Read that log. It tells you what you're removing, and occasionally it makes the cause obvious.

## Step 4 — Check the migration question

**Do not skip this.** This is the step that turns a clean rollback into a two-hour outage.

```bash
git diff $TARGET..$CURRENT -- alembic/versions/
```

**Decision point:**

**No migrations in the range** → safe. Go to Step 5.

**Migrations present, but expand-only** — added a nullable column, added a table, added an index,
added a `CHECK` that existing rows satisfy → safe, because the old code doesn't know about the
new column and doesn't care that it exists. **Leave the migration applied.** Go to Step 5.

**Migrations that dropped, renamed, or tightened something** → the old code will break against
the new schema. This should be impossible under our expand/contract rule
([`../../engineering/branching-and-releases.md`](../../engineering/branching-and-releases.md#migrations-and-release-ordering)),
so if you're here, the rule was violated.

Then:

1. Check whether the migration's `downgrade()` is safe. Every migration has one and it's
   round-trip tested, but a downgrade that drops a column loses the data written since the
   upgrade.
2. If the downgrade is lossless → downgrade, then roll back the code:
   ```bash
   # From a maintenance context with the production DATABASE_URL
   alembic downgrade -1
   ```
3. If the downgrade **loses data** → do not downgrade. **Fix forward instead**, with a minimal
   diff. Escalate; this is a decision to make with a second person if one exists.
4. Write this up in the post-incident review. An expand/contract violation reaching production is
   a process finding, not a code finding.

## Step 5 — Roll back

```bash
gh workflow run deploy.yml \
  --repo goldseats/goldseats-api \
  -f version=$TARGET \
  -f environment=production

gh run watch --repo goldseats/goldseats-api
```

If the workflow is unavailable, use the host dashboard's rollback-to-previous-deploy. Note which
route you took in the incident log — it matters for the review.

## Step 6 — Verify

```bash
# Version is back
curl -s https://api.goldseats.app/health/ready | jq '{status, version}'

# Core paths work
curl -fsS 'https://api.goldseats.app/v1/films/now-playing?limit=1' | jq -e '.data | length > 0'
curl -fsS 'https://api.goldseats.app/v1/theatres?limit=1' | jq -e '.data | length > 0'

# The one that matters — exercises db, layout, availability, and scoring in one request
curl -fsS -X POST 'https://api.goldseats.app/v1/recommendations' \
  -H 'content-type: application/json' \
  -d "{\"showtime_id\":\"$SMOKE_SHOWTIME_ID\",\"party_size\":2}" \
  | jq -e '.options | length > 0'
```

Then on the dashboard:

- Error rate back to baseline
- p95 latency back to baseline
- **`booking_handoff_total` moving again.** A silent drop to zero is the failure no error rate
  will show you.
- No new Sentry issues on the rolled-back release

**Watch for 15 minutes before declaring resolved.** A rollback that looks clean for two minutes
sometimes isn't.

## Step 7 — Web app, if needed

If the web app is also affected, roll it back independently — the two deploy separately.

Use the host dashboard's instant rollback to the previous deploy. Then:

```bash
curl -fsS https://goldseats.app/ | grep -q 'GoldSeats'
```

Check the browser console for errors and load the home page on a real device.

## Step 8 — Close out

1. Update the incident: rolled back to `$TARGET`, users unaffected as of `<time>`
2. Update the status page
3. Open an issue for the root cause, linking the incident
4. **Do not re-deploy the bad version.** Whatever comes next is a new patch release.
5. Post-incident review within 48 hours —
   [`../incident-response.md`](../incident-response.md#post-incident-review-template)

---

## If the rollback makes things worse

1. The previous version may also have been bad. Go back one more tag.
2. If two consecutive rollbacks fail, the problem is probably **not the code**. Check:
   - Database — connection pool exhausted? Disk full? →
     [`database-restore.md`](database-restore.md)
   - Redis — is the degradation actually working, or is something erroring on a cache miss?
   - Migrations — did a partial migration leave the schema in an inconsistent state?
     `alembic current` vs `alembic heads`
   - Configuration — did a secret rotate or expire? An expired credential looks exactly like a
     bad deploy.
   - Host — platform incident? Check the provider's status page.
3. Escalate. Stop rolling versions and start diagnosing.

---

## Escalation

Stop and get help when:

- Two rollbacks haven't helped
- A migration needs reversing and the downgrade loses data
- The database is involved at all
- You've been at it 30 minutes with no improvement on a Sev-1
- You need to make a call about data loss

Contacts: [`../on-call.md`](../on-call.md).

---

## Related

- [`../release-process.md`](../release-process.md) — how deploys work
- [`../incident-response.md`](../incident-response.md) — severities and comms
- [`../../engineering/branching-and-releases.md`](../../engineering/branching-and-releases.md) —
  versioning and the expand/contract rule
- [`database-restore.md`](database-restore.md) — when data is the problem
