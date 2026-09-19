# Runbook — Database Restore

**Last drilled:** not yet. The first timed drill is a [M9](../../product/milestones/M9.md)
deliverable, and the RTO and RPO below are **targets, not measured facts**, until then.

**Target RTO:** under 60 minutes. **Target RPO:** under 5 minutes, via continuous WAL archiving.

---

## When to use this

- Data is missing or wrong in production and you can't explain it
- A data migration corrupted rows
- Someone ran a destructive statement against production
- The database won't start, or reports corruption
- You need to inspect historical data to establish the scope of an incident

## When NOT to use this

- **A bad deploy with no data problem.** → [`api-rollback.md`](api-rollback.md). Restoring a
  database to fix a code bug is a dramatic way to cause a real outage.
- **A schema problem with a clean downgrade.** `alembic downgrade -1` is far cheaper.
- **Local SQLite.** Just delete it:
  ```bash
  rm goldseats.db* && alembic upgrade head && python -m app.cli.seed_theatre --all
  ```

---

## Before you touch anything

**Stop. Read this.**

A restore is the most destructive operation in this document set. Done carelessly it turns a
partial data problem into total data loss, plus an outage.

Three rules:

1. **Snapshot the current broken state first.** Whatever is wrong, the current data is still
   evidence and may contain writes that the backup doesn't. Never restore over it.
2. **Restore into a scratch database first, always.** Verify there, then decide. A restore
   directly over production is only correct when production is already unusable.
3. **Understand the blast radius before acting.** Which tables? How many rows? Since when? A
   surgical fix on three rows is almost always better than a full restore.

---

## What matters and what doesn't

Knowing what's actually irreplaceable changes the decision substantially.

| Data | Reproducible? | Priority |
| --- | --- | --- |
| `users` | **No.** Accounts, hashed passwords, OAuth links. | **Critical** |
| `follows`, `watchlist_entries` | **No.** A user's curated lists. | **Critical** |
| `chat_conversations`, `chat_messages` | No, but low value if lost | Medium |
| `recommendations` | No, but each is reproducible on demand. The valuable part is `selected_seats_json` — the training signal. | Medium |
| `theatres`, `auditoriums`, `seat_layouts`, `seats` | **Yes** — re-seed from committed JSON | Low |
| `showtimes`, `showtime_formats`, `booking_links` | **Yes** — re-seed | Low |
| `films`, `film_sources`, `film_releases`, `film_timeline_events` | **Yes** — re-ingest from TMDB | Low |
| `news_items` | Mostly — re-ingest recent, lose old | Low |
| `seat_availability_snapshots` | Yes — regenerate | Low |

**The implication: if only seeded or ingested data is damaged, don't restore. Re-seed.** It's
faster, safer, and doesn't risk the user data that's actually irreplaceable.

```bash
python -m app.cli.seed_theatre --all
python -m app.ingest.tmdb --window 90d
```

Only user data justifies a restore.

---

## Prerequisites

- Production database credentials — founder-only access
- Access to the managed Postgres provider's dashboard
- `psql` and `pg_dump` installed and version-compatible with production
- The incident declared. **Data loss is a Sev-1.** →
  [`../incident-response.md`](../incident-response.md)

---

## Step 1 — Assess

Do not skip to restoring. Five minutes here saves an hour.

```bash
export PROD_DB="$PRODUCTION_DATABASE_URL"

# Which tables are affected, and how bad?
psql "$PROD_DB" -c "
  SELECT 'users' t, count(*) FROM users
  UNION ALL SELECT 'follows', count(*) FROM follows
  UNION ALL SELECT 'recommendations', count(*) FROM recommendations
  UNION ALL SELECT 'films', count(*) FROM films
  UNION ALL SELECT 'theatres', count(*) FROM theatres;"

# When did it start? created_at/updated_at are on every table.
psql "$PROD_DB" -c "
  SELECT date_trunc('hour', created_at) h, count(*)
  FROM users WHERE created_at > now() - interval '7 days'
  GROUP BY 1 ORDER BY 1;"

# Schema consistent?
psql "$PROD_DB" -c "\dt"
alembic current
```

Write down: which tables, how many rows, the approximate time it started, and a known-good
timestamp to target.

## Step 2 — Snapshot the current state

**Non-negotiable.** Do this even when the data looks worthless.

```bash
export TS=$(date -u +%Y%m%dT%H%M%SZ)
pg_dump "$PROD_DB" -Fc -f "goldseats-broken-$TS.dump"
ls -lh "goldseats-broken-$TS.dump"
```

Store it somewhere durable and note the path in the incident. If the restore goes wrong, this is
the only copy of whatever was still correct.

## Step 3 — Decide the approach

**Decision point. Pick one.**

### 3a — Surgical fix

Choose when: a small, identified set of rows is wrong, and the correct values are known or
derivable.

Safest by a wide margin. Always prefer this.

```bash
psql "$PROD_DB"
BEGIN;
-- your correction
SELECT ...;  -- verify inside the transaction, before committing
COMMIT;      -- or ROLLBACK
```

Always in an explicit transaction. Always `SELECT` to verify before `COMMIT`.

### 3b — Re-seed or re-ingest

Choose when: only seeded or TMDB-derived data is damaged. See the table above.

```bash
python -m app.cli.seed_theatre --all --dry-run   # look first
python -m app.cli.seed_theatre --all
python -m app.ingest.tmdb --window 90d
```

Both are idempotent and upsert on natural keys, which is exactly why they're safe to re-run.

### 3c — Partial restore of specific tables

Choose when: user data in a few tables is damaged but the rest of the database is fine.

Restore into a scratch database (Step 4), then copy the good tables across. Mind the foreign
keys — `follows.user_id` needs `users` present first.

### 3d — Full point-in-time restore

Choose when: the database is unusable, corruption is widespread, or the damage timeline is
unknown.

**Highest risk.** It discards every write after the target timestamp, including legitimate ones.
Continue to Step 4.

## Step 4 — Restore into a scratch database

**Never restore directly over production** unless production is already unusable and you've
accepted the loss.

Use the provider's point-in-time restore to create a **new** instance at the target timestamp:

```bash
# Provider-specific. The shape is always the same:
# create a new instance from a PITR target, then get its connection string.
export SCRATCH_DB="postgresql://...scratch..."
```

Pick the target timestamp as **just before the damage began**, from Step 1.

## Step 5 — Verify the scratch database

Prove the restore is actually good before you rely on it.

```bash
psql "$SCRATCH_DB" -c "
  SELECT 'users' t, count(*) FROM users
  UNION ALL SELECT 'follows', count(*) FROM follows
  UNION ALL SELECT 'recommendations', count(*) FROM recommendations;"

# Is the damage absent here?
psql "$SCRATCH_DB" -c "<the query that showed the problem in step 1>"

# Schema at the expected revision?
DATABASE_URL="$SCRATCH_DB" alembic current

# Referential integrity
psql "$SCRATCH_DB" -c "
  SELECT count(*) FROM follows f
  LEFT JOIN users u ON u.id = f.user_id WHERE u.id IS NULL;"   -- expect 0

# Does the app actually work against it?
DATABASE_URL="$SCRATCH_DB" uvicorn app.main:app --port 8001 &
curl -s localhost:8001/health/ready | jq .
curl -s 'localhost:8001/v1/films/now-playing?limit=1' | jq -e '.data | length > 0'
```

**If the scratch database is also damaged**, the damage is older than you thought. Restore to an
earlier timestamp and repeat.

## Step 6 — Cut over

**Decision point.** How much data is between the restore target and now, and are you accepting
losing it?

### Option A — Promote the scratch instance (preferred)

Faster and reversible, because the broken original still exists.

1. Put the API in maintenance mode, or scale it to zero
2. Point `DATABASE_URL` at the scratch instance
3. Restart the API
4. Verify (Step 7)
5. **Keep the original instance for at least 7 days.** Do not delete it.

### Option B — Restore over production

Only when the original is unusable.

1. Maintenance mode
2. `pg_restore` the verified dump over production
3. Restart
4. Verify

```bash
pg_restore --clean --if-exists -d "$PROD_DB" "goldseats-verified-$TS.dump"
```

`--clean` drops objects before recreating them. It is destructive. You have the Step 2 snapshot.

## Step 7 — Verify production

```bash
curl -s https://api.goldseats.app/health/ready | jq '{status, version, checks}'
curl -fsS 'https://api.goldseats.app/v1/films/now-playing?limit=1' | jq -e '.data | length > 0'
curl -fsS 'https://api.goldseats.app/v1/theatres?limit=1' | jq -e '.data | length > 0'
curl -fsS -X POST 'https://api.goldseats.app/v1/recommendations' \
  -H 'content-type: application/json' \
  -d "{\"showtime_id\":\"$SMOKE_SHOWTIME_ID\",\"party_size\":2}" | jq -e '.options | length > 0'
```

Then:

- Sign in as a test account and confirm follows and watchlist are intact
- Check `alembic current` matches the deployed code's expected head
- Watch the dashboard for 15 minutes
- Re-seed anything reproducible that the restore rolled back:
  ```bash
  python -m app.cli.seed_theatre --all
  python -m app.ingest.tmdb --window 30d
  ```

## Step 8 — Close out

1. Record in the incident: restore target timestamp, approach taken, **and exactly what data was
   lost**
2. If any user data was lost, work out who was affected. If it's material, tell them. If it might
   be a privacy matter, see
   [`../incident-response.md`](../incident-response.md#security-incidents) — PIPEDA obligations
   may apply and a lawyer should be involved.
3. Keep the broken snapshot and the pre-restore instance for 7 days minimum
4. **Update this runbook with the real RTO and RPO you just measured.** The numbers at the top are
   targets; replace them with facts.
5. Post-incident review within 48 hours, with action items on prevention

---

## Backup configuration

| | Production | Staging | Local |
| --- | --- | --- | --- |
| Base backups | Daily, automated | Daily | None |
| WAL archiving | Continuous | None | None |
| Retention | 30 days | 7 days | — |
| Encryption | At rest | At rest | — |
| Restore test | Quarterly drill | Ad hoc | — |

Local SQLite is deliberately not backed up. It's disposable and reproducible from a committed
seed.

---

## Quarterly drill

The drill is the whole point. Undrilled restore procedures are fiction.

1. Create a scratch instance from a PITR target
2. Verify with the Step 5 queries
3. Boot the app against it and run the smoke tests
4. **Time every step** and record it
5. Destroy the scratch instance
6. Update this runbook with what was wrong in it

**If a drill fails, that's a Sev-2 finding**, not an inconvenience. It means we currently cannot
recover from data loss, and fixing it takes priority over feature work.

---

## Escalation

Stop and get help when:

- The scratch restore is also damaged
- Data loss would affect real users
- You're about to run something with `--clean` or `DROP` against production
- You can't determine when the damage started
- A privacy obligation might be triggered

Contacts: [`../on-call.md`](../on-call.md).

---

## Related

- [`../incident-response.md`](../incident-response.md)
- [`api-rollback.md`](api-rollback.md) — when it's code, not data
- [`../../engineering/database-conventions.md`](../../engineering/database-conventions.md) —
  migrations, cascades, and the deletion rules
- [`../../architecture/data-model.md`](../../architecture/data-model.md) — what's in each table
