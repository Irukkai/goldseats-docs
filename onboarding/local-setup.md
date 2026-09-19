# Local Setup

From nothing to both apps running. **No Docker, no database server, no services.** That's a hard
design requirement, not a convenience — see
[ADR-0002](../architecture/adr/0002-stack-selection.md).

Target: under 15 minutes.

---

## Prerequisites

Verify these first. Full tooling detail in [`tooling.md`](tooling.md).

```bash
node --version      # v20.19.0 or later
npm --version       # 10.8.2 or later
python3 --version   # 3.12.4 or later
git --version
gh --version        # 2.100.0 or later
```

If `python3` isn't 3.12+, install it before continuing. The API pins 3.12 and uses 3.12 syntax.

### Authenticate `gh` — do this first

Nothing pushes until this is done:

```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
gh auth status
```

---

## Clone the repos

All three as siblings, so relative paths in these docs work:

```bash
mkdir -p ~/Startup/goldseats && cd ~/Startup/goldseats
git clone https://github.com/Irukkai/goldseats-api.git
git clone https://github.com/Irukkai/goldseats-web.git
git clone https://github.com/Irukkai/goldseats-docs.git
```

Replace `goldseats` in those URLs with the actual org name if it differs.

You end up with:

```
~/Startup/goldseats/
  goldseats-api/
  goldseats-web/
  goldseats-docs/
```

---

## API — `goldseats-api`

### 1. Virtual environment

```bash
cd ~/Startup/goldseats/goldseats-api
python3 -m venv .venv
source .venv/bin/activate
```

Your prompt should now show `(.venv)`. **You need to run `source .venv/bin/activate` in every new
terminal** — forgetting it is the single most common cause of "it worked yesterday".

### 2. Dependencies

```bash
pip install -r requirements.txt
pip install -r requirements-dev.txt    # ruff, mypy, pytest
```

### 3. Configuration

```bash
cp .env.example .env
```

Then edit `.env`. To just get it running you need one thing:

```bash
# Generate a JWT secret
python3 -c "import secrets; print(secrets.token_urlsafe(48))"
```

Paste that into `JWT_SECRET`. Everything else has a working default:

| Setting | Default | Note |
| --- | --- | --- |
| `DATABASE_URL` | `sqlite+aiosqlite:///./goldseats.db` | **Leave it.** This is the point of the setup. |
| `REDIS_URL` | unset | Optional. Caching degrades gracefully. |
| `TMDB_API_KEY` | empty | Needed only to ingest films. See below. |
| `LLM_BASE_URL` | Ollama | Needed only for chat work, which is flag-off by default. |
| `LOG_FORMAT` | `console` | Readable logs instead of JSON. |

`.env` is gitignored. **Never commit it** — the repos are public. See
[`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md).

### 4. Create the database

```bash
alembic upgrade head
```

This creates `goldseats.db` in the current directory. It's a file, it's gitignored, and it's
disposable.

### 5. Run it

```bash
uvicorn app.main:app --reload
```

- API: **http://localhost:8000**
- Interactive docs: **http://localhost:8000/docs**
- ReDoc: **http://localhost:8000/redoc**
- Health: **http://localhost:8000/health/ready**

```bash
curl -s localhost:8000/health/ready | jq .
```

### 6. Get some data in

An empty database means an empty home page. Two commands:

```bash
# Theatres, auditoriums, seat layouts, showtimes, booking links (M2)
python -m app.cli.seed_theatre --all

# Films from TMDB (M1) — needs TMDB_API_KEY
python -m app.ingest.tmdb --window 90d
```

A free TMDB key comes from your TMDB account's API settings. Attribution obligations that come
with it are in [`../legal/data-sources-policy.md`](../legal/data-sources-policy.md).

Without a key, seed theatre data still works — you just have no films.

```bash
curl -s 'localhost:8000/v1/films/now-playing?limit=3' | jq '.data[].title'
curl -s 'localhost:8000/v1/theatres?limit=3'          | jq '.data[].name'
```

---

## Web — `goldseats-web`

In a second terminal:

### 1. Dependencies

```bash
cd ~/Startup/goldseats/goldseats-web
npm install
```

### 2. Configuration

```bash
cp .env.example .env.local
```

The default `API_BASE_URL=http://localhost:8000` is correct if the API is running.

### 3. Generate API types

With the API running:

```bash
npm run generate:api-types
```

This writes `src/types/api.ts` from the API's OpenAPI schema. **Never hand-write those types**,
and re-run this whenever the API changes.

### 4. Run it

```bash
npm run dev
```

- Web app: **http://localhost:3000**
- Marketing page: **http://localhost:3000** (marketing route group)

---

## Verify it all works

```bash
# API
curl -s localhost:8000/health/ready | jq -e '.status == "ready"'
curl -s 'localhost:8000/v1/films/now-playing?limit=1' | jq -e '.data | length > 0'

# Web
curl -s localhost:3000 | grep -q 'GoldSeats' && echo web ok
```

Then in a browser: load http://localhost:3000, filter by title, filter by language, open a film,
open a showtime, request seat recommendations.

---

## The checks CI runs

Run these before every push. CI runs exactly these, so a local failure is a guaranteed CI
failure and a wasted round trip.

```bash
# API — with .venv active
cd goldseats-api
ruff check .
ruff format --check .
mypy app
pytest

# Web
cd goldseats-web
npm run lint
npm run typecheck
npm run test
npm run build
```

Install the pre-commit hooks so secrets never reach a public repo:

```bash
pip install pre-commit && pre-commit install
```

---

## Resetting

The database is disposable. When something is weird, reset it:

```bash
cd goldseats-api
rm goldseats.db goldseats.db-wal goldseats.db-shm
alembic upgrade head
python -m app.cli.seed_theatre --all
python -m app.ingest.tmdb --window 30d
```

Full reset, including the environment:

```bash
# API
deactivate 2>/dev/null
rm -rf .venv goldseats.db*
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
alembic upgrade head

# Web
rm -rf node_modules .next
npm install
```

---

## Optional extras

### Redis

Not needed. Caching falls through to the database when `REDIS_URL` is unset.

If you're working on caching or rate limiting:

```bash
brew install redis
brew services start redis
# then set REDIS_URL=redis://localhost:6379/0 in .env
```

### Ollama — only for chatbot work

Only needed if you're working on [M7](../product/milestones/M7.md). `FEATURE_CHAT_ENABLED`
defaults to `false`, so everything else works without it.

```bash
brew install ollama
ollama serve                              # in its own terminal
ollama pull llama3.1:8b-instruct-q4_K_M
```

Then set `FEATURE_CHAT_ENABLED=true` in `.env`. Expect a few GB of download and real memory use.

### Playwright browsers

```bash
cd goldseats-web
npx playwright install --with-deps
npx playwright test
```

---

## Troubleshooting

**`ModuleNotFoundError` on a package you just installed**
The virtualenv isn't active. `source .venv/bin/activate`. This is the most common problem by a
wide margin.

**`ValidationError` about `JWT_SECRET` at startup**
It's missing, under 32 characters, or still the `.env.example` placeholder. Generate a real one.
Failing at boot is intentional.

**`Extra inputs are not permitted` at startup**
`Settings` uses `extra="forbid"`, so a typo'd variable in `.env` crashes the app rather than being
silently ignored. Check the spelling against `.env.example`. `TMBD_API_KEY` is the classic.

**`alembic: command not found`**
Virtualenv not active, or `requirements-dev.txt` not installed.

**`Target database is not up to date`**
```bash
alembic current   # where you are
alembic heads     # where you should be
alembic upgrade head
```

**Web app shows no films**
The API has no data. Run the seed and ingest commands. Check `curl
localhost:8000/v1/films/now-playing`.

**Type errors on `src/types/api.ts`**
Stale generated types. Re-run `npm run generate:api-types` with the API running.

**`EADDRINUSE` on 3000 or 8000**
```bash
lsof -ti:8000 | xargs kill
lsof -ti:3000 | xargs kill
```

**Tests pass locally but fail in CI**
Almost always the SQLite/Postgres parity job. Some Postgres features are off-limits for exactly
this reason — read
[`../engineering/database-conventions.md`](../engineering/database-conventions.md). `ILIKE` is the
usual culprit.

**`gh: could not authenticate`**
```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
```

---

## Next

- [`day-one.md`](day-one.md) — what to read and what to ship first
- [`tooling.md`](tooling.md) — editor setup and every command you'll need
- [`../CONTRIBUTING.md`](../CONTRIBUTING.md) — the contribution loop
