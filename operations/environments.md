# Environments

Three environments: local, staging, production. They differ in exactly the ways they have to
and are identical everywhere else.

---

## Summary

| | Local | Staging | Production |
| --- | --- | --- | --- |
| **Web** | `localhost:3000` | `staging.goldseats.app` | `goldseats.app` |
| **API** | `localhost:8000` | `api-staging.goldseats.app` | `api.goldseats.app` |
| **Database** | SQLite file | Managed Postgres 16 (small) | Managed Postgres 16 |
| **Redis** | Optional | Managed, small | Managed |
| **LLM** | Ollama, optional | vLLM, shared | vLLM, dedicated |
| **Deploy trigger** | You | Merge to `main`, automatic | Git tag + manual approval |
| **`ENVIRONMENT`** | `local` | `staging` | `production` |
| **Logs** | Console, pretty | JSON, aggregated | JSON, aggregated |
| **Tracing** | Off | 100% | 10% success / 100% error |
| **Sentry** | Off | On | On |
| **Data** | Seeded | Seeded, fake users | Real users |
| **Backups** | None | Daily | Daily + continuous WAL |

---

## Local

The design constraint: **`git clone` to running app with no services and no container
runtime.** Docker is not installed on the founder's machine and requiring it would put a
several-hundred-megabyte download and a class of "why won't my container start" problems
between a contributor and their first `uvicorn`. See
[ADR-0002](../architecture/adr/0002-stack-selection.md).

Exact commands: [`../onboarding/local-setup.md`](../onboarding/local-setup.md).

```bash
# API — http://localhost:8000, docs at /docs
cd goldseats-api
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
alembic upgrade head
uvicorn app.main:app --reload

# Web — http://localhost:3000
cd goldseats-web
npm install
npm run dev
```

What's different from production, and why it's acceptable:

| Difference | Why it's fine | Where the risk is caught |
| --- | --- | --- |
| SQLite, not Postgres | Zero-setup requirement | The CI parity job runs the suite against real Postgres |
| Redis optional | Caching degrades rather than fails | A staging run with Redis present |
| No Ollama unless you're working on chat | `FEATURE_CHAT_ENABLED=false` by default | [M7](../product/milestones/M7.md) evaluation suite |
| Sentry and tracing off | No noise from a dev machine | Staging |
| Email logged, not sent | Nobody wants to configure SMTP to test signup | Staging with a real transport |

Resetting is meant to be trivial:

```bash
rm goldseats.db*
alembic upgrade head
python -m app.cli.seed_theatre --all
python -m app.ingest.tmdb --window 30d
```

Configuration is your gitignored `.env`, seeded from the committed `.env.example`. See
[`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).

---

## Staging

A full, production-shaped environment with fake data. It exists so that "deployed to staging
and manually exercised there" can be a real line on the
[definition of done](../engineering/definition-of-done.md).

- **Deploys automatically on every merge to `main`.** No approval gate. If `main` is broken,
  staging is broken, and we'd rather know.
- Managed Postgres, managed Redis, a real (shared, smaller) vLLM instance.
- Seeded theatre and film data. **Fake users only** — never a copy of production user data. A
  staging dump on a laptop is a breach waiting to happen, and we don't need real accounts to
  test a seat map.
- Sentry on, with `environment=staging` so staging noise doesn't pollute production error
  triage.
- Tracing at 100%, because volume is low and debugging value is high.
- `robots.txt` disallows everything. Staging must never appear in search results.
- HTTP basic auth or an IP allowlist in front of it.
- **Secrets are entirely distinct from production.** In particular a different `JWT_SECRET` — if
  they were shared, a staging token would authenticate against production.
- Daily backups, short retention. Losing staging costs a re-seed.

Staging is also where we verify the things that can only be verified in a deployed environment:
alert rules actually firing ([M9](../product/milestones/M9.md)), migrations against real
Postgres, k6 load tests, and rollback.

---

## Production

- **Deploys on a git tag with a manual approval gate.** See
  [`release-process.md`](release-process.md).
- Managed Postgres 16 with automated daily base backups and continuous WAL archiving, 30-day
  retention. Restores are drilled quarterly —
  [`runbooks/database-restore.md`](runbooks/database-restore.md).
- Managed Redis. A Redis outage must degrade latency, never cause an error.
- Dedicated vLLM instance. If it's unavailable, the chatbot degrades to the deterministic seat
  map — never blocking seat selection.
- `LOG_LEVEL=INFO`, JSON logs, aggregated. `LOG_LEVEL` is an env var so it can be raised during
  an incident.
- Sentry with `release` set to the git tag, so an error spike is attributable to a specific
  deploy.
- Rate limiting on.
- Sized from [M9](../product/milestones/M9.md)'s load test results. Fixed capacity; autoscaling
  is a later decision.

### The production configuration guard

It is genuinely easy to deploy with the SQLite default still in place, so the application
refuses to boot rather than doing something embarrassing:

```python
@model_validator(mode="after")
def production_requires_real_infrastructure(self) -> "Settings":
    if self.environment == "production":
        if self.database_url.startswith("sqlite"):
            raise ValueError("SQLite is not permitted in production")
        if not self.redis_url:
            raise ValueError("REDIS_URL is required in production")
        if not self.sentry_dsn:
            raise ValueError("SENTRY_DSN is required in production")
    return self
```

Failing at boot is the point. A missing `SENTRY_DSN` discovered during an incident is worse
than a failed deploy.

---

## DNS

| Record | Points to |
| --- | --- |
| `goldseats.app` (apex) | Web host |
| `www.goldseats.app` | Web host |
| `api.goldseats.app` | API host |
| `staging.goldseats.app` | Web host, staging |
| `api-staging.goldseats.app` | API host, staging |

### The current situation

DNS today has:

```
CNAME www.goldseats.app -> stellar-hotteok-63503a.netlify.app
```

**The Netlify account that owns that site has been lost.** We cannot modify, redeploy, or delete
it. The landing page currently served at `www.goldseats.app` is therefore frozen and
unmanageable.

This is survivable because we control the **registrar**, which is what actually matters. DNS can
be repointed at a host we own regardless of who owns the old Netlify site; the orphaned deploy
simply stops receiving traffic.

Resolving it is a [M9](../product/milestones/M9.md) deliverable with a full procedure in
[`runbooks/domain-cutover.md`](runbooks/domain-cutover.md).

---

## Hosting — an open decision

**Not yet decided.** Both options are viable and the choice belongs to the founder, at
[M9](../product/milestones/M9.md) at the latest.

| | Netlify (new, owned account) | Vercel |
| --- | --- | --- |
| Next.js 15 App Router support | Good, via the adapter | First-party, built by the same team |
| Continuity with existing DNS | Already a Netlify `CNAME` | New target |
| Instant rollback | Yes | Yes |
| Preview deploys per PR | Yes | Yes |
| Free tier adequate pre-launch | Yes | Yes |
| Risk | Adapter lag on new Next.js features | Vendor concentration on the framework author |

The API needs a separate container host regardless — neither is a good fit for a long-running
FastAPI process with a GPU dependency nearby.

**Recommendation, for the founder to accept or reject:** Vercel for the web app, because App
Router caching behaviour is subtle and the failure mode we most fear — a cached seat map showing
a sold seat — is exactly the kind of thing that's least surprising on first-party
infrastructure. Netlify's only real advantage is DNS continuity, and we're changing the DNS
target anyway.

---

## Configuration per environment

Everything is environment variables. Full inventory in `.env.example` and
[`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).

| Variable | Local | Staging | Production |
| --- | --- | --- | --- |
| `ENVIRONMENT` | `local` | `staging` | `production` |
| `DATABASE_URL` | `sqlite+aiosqlite:///./goldseats.db` | Postgres | Postgres |
| `REDIS_URL` | unset | Redis | Redis |
| `LOG_LEVEL` | `DEBUG` | `INFO` | `INFO` |
| `LOG_FORMAT` | `console` | `json` | `json` |
| `LLM_BASE_URL` | `http://localhost:11434/v1` | vLLM | vLLM |
| `SENTRY_DSN` | empty | set | set |
| `RELEASE_VERSION` | `local` | commit SHA | git tag |
| `FEATURE_*` | mostly off | on early | on when shipped |

Secrets never overlap between environments. Rotation schedule is in
[`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).

---

## Promotion path

```
local  →  PR preview (web only)  →  staging  →  production
```

- A change must pass through staging. There is no path from a laptop to production.
- Migrations run automatically as part of the deploy, and must be backwards compatible with the
  currently running code — see
  [`../engineering/branching-and-releases.md`](../engineering/branching-and-releases.md#migrations-and-release-ordering).
- Smoke tests run after every deploy in both environments.
- Rollback is redeploying the previous tag. Verified, not assumed —
  [`runbooks/api-rollback.md`](runbooks/api-rollback.md).

---

## Access

| Environment | Who | How |
| --- | --- | --- |
| Local | Anyone with a clone | No credentials needed |
| Staging | Contributors | Host dashboard; basic auth on the web app |
| Production | Founder, and on-call | Host dashboard, MFA required |
| Production database | Founder only | Direct access is for incidents. Every session gets a note in the incident log. |

Production database access is deliberately awkward. Routine work should never need it; if it
does, that's a missing tool or a missing dashboard, and the fix is building that rather than
normalising a psql prompt against real user data.
