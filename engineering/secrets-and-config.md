# Secrets and Configuration

All three GoldSeats repos are **public**. A committed secret is a published secret. This
document is therefore short on philosophy and long on rules.

---

## The one rule

**No secret ever enters git. Only `.env.example` is committed.**

Not in code. Not in a test fixture. Not in a comment. Not in a docs example. Not in a
commit that you plan to amend before pushing. Not in a branch you'll delete.

---

## Where each kind of value lives

| Value | Local | CI | Staging / Production |
| --- | --- | --- | --- |
| Non-secret config (log level, feature flags, API base URL) | `.env` | Workflow `env:` | Host env vars |
| Secrets (DB password, JWT secret, TMDB key, Sentry DSN) | `.env`, gitignored | GitHub Actions **secrets** | Host secret store |
| Public web config (`NEXT_PUBLIC_*`) | `.env.local` | Workflow `env:` | Host build env |
| Placeholders and documentation of every var | `.env.example`, **committed** | — | — |

Gitignored in both code repos, and verify this is true before your first commit:

```gitignore
.env
.env.local
.env.*.local
*.pem
*.key
goldseats.db
goldseats.db-wal
goldseats.db-shm
.venv/
node_modules/
```

`NEXT_PUBLIC_*` variables are **compiled into the browser bundle**. They are public by
definition. Never put anything sensitive behind that prefix — the prefix is not a
protection mechanism, it's a declaration.

---

## `.env.example` is a contract

It's the only documentation of what the app needs to run, so it has to be complete and
honest. Every variable, with a comment, a safe placeholder, and a note if it's required.

`goldseats-api/.env.example`:

```bash
# ---- Core ----
# SQLite by default: Docker is not installed on dev machines, so local setup must
# require zero services. See architecture/adr/0002-stack-selection.md
DATABASE_URL=sqlite+aiosqlite:///./goldseats.db
# Production example:
# DATABASE_URL=postgresql+asyncpg://goldseats:CHANGE_ME@db.internal:5432/goldseats

# Optional locally — caching degrades gracefully if unset. Required in production.
REDIS_URL=redis://localhost:6379/0

ENVIRONMENT=local            # local | staging | production
LOG_LEVEL=INFO               # DEBUG | INFO | WARNING | ERROR
LOG_FORMAT=console           # console locally, json in staging/production

# ---- Auth (M1) ----
# REQUIRED. Min 32 chars. Generate: python -c "import secrets; print(secrets.token_urlsafe(48))"
JWT_SECRET=replace-with-a-long-random-string-at-least-32-chars
ACCESS_TOKEN_TTL_MINUTES=15
REFRESH_TOKEN_TTL_DAYS=30
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=

# ---- Film metadata (M1) ----
# Free key from https://www.themoviedb.org/settings/api
# Attribution is mandatory — see legal/data-sources-policy.md
TMDB_API_KEY=
TMDB_LANGUAGE=en-CA

# ---- Chatbot (M7) ----
# Ollama locally, vLLM in production. OpenAI-compatible endpoint either way.
LLM_BASE_URL=http://localhost:11434/v1
LLM_MODEL=llama3.1:8b-instruct-q4_K_M
LLM_API_KEY=                 # empty for local Ollama; set for a protected vLLM
LLM_TIMEOUT_SECONDS=30

# ---- Observability (M9) ----
SENTRY_DSN=
OTEL_EXPORTER_OTLP_ENDPOINT=
RELEASE_VERSION=local        # set to the git tag by CI

# ---- Feature flags ----
FEATURE_CHAT_ENABLED=false                    # M7
FEATURE_3D_SEAT_VIEW_ENABLED=false            # M6
FEATURE_IN_PERSON_RESERVATION_ENABLED=false   # M5
```

`goldseats-web/.env.example`:

```bash
# Server-side only. Not exposed to the browser.
API_BASE_URL=http://localhost:8000

# Compiled into the client bundle. PUBLIC. Never put a secret here.
NEXT_PUBLIC_SITE_URL=http://localhost:3000
NEXT_PUBLIC_FEATURE_CHAT=false
NEXT_PUBLIC_FEATURE_3D=false
NEXT_PUBLIC_SENTRY_DSN=
```

Adding a setting without adding it here is an incomplete change — it's on the
[definition-of-done](definition-of-done.md) checklist and reviewers check for it.

---

## Reading configuration

### API: exactly one file touches the environment

`app/config.py` is the only file in the repo allowed to read `os.environ`. Everything else
takes a `Settings` object. This means the full configuration surface is one file you can
read in thirty seconds, and it means tests can override settings without monkeypatching the
environment.

```python
# app/config.py
from functools import lru_cache
from pydantic import Field, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="forbid",   # a typo'd env var fails at boot, loudly
    )

    database_url: str = "sqlite+aiosqlite:///./goldseats.db"
    redis_url: str | None = None
    environment: Literal["local", "staging", "production"] = "local"
    log_level: str = "INFO"

    jwt_secret: str = Field(min_length=32)
    tmdb_api_key: str = ""
    llm_base_url: str = "http://localhost:11434/v1"
    sentry_dsn: str = ""

    feature_chat_enabled: bool = False
    feature_3d_seat_view_enabled: bool = False

    @field_validator("jwt_secret")
    @classmethod
    def reject_the_placeholder(cls, v: str) -> str:
        if "replace-with" in v:
            raise ValueError("JWT_SECRET is still the .env.example placeholder")
        return v

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

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Three things that matter more than they look:

- **`extra="forbid"`** — `TMBD_API_KEY` (transposed) crashes at boot instead of silently
  running with no key and an empty catalog.
- **Validation at boot, not at first use.** A missing `JWT_SECRET` should kill the process
  on startup, not produce a 500 the first time someone logs in.
- **The production guard.** It is genuinely easy to deploy with the SQLite default still
  in place. The check makes that impossible rather than embarrassing.

`os.environ` outside `app/config.py` fails review.

### Web: one module, validated at build

```ts
// src/lib/env.ts
import { z } from 'zod';

const serverSchema = z.object({
  API_BASE_URL: z.string().url(),
});

const clientSchema = z.object({
  NEXT_PUBLIC_SITE_URL: z.string().url(),
  NEXT_PUBLIC_FEATURE_CHAT: z.enum(['true', 'false']).default('false'),
  NEXT_PUBLIC_FEATURE_3D: z.enum(['true', 'false']).default('false'),
});

export const serverEnv = serverSchema.parse(process.env);
export const clientEnv = clientSchema.parse({
  NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
  NEXT_PUBLIC_FEATURE_CHAT: process.env.NEXT_PUBLIC_FEATURE_CHAT,
  NEXT_PUBLIC_FEATURE_3D: process.env.NEXT_PUBLIC_FEATURE_3D,
});
```

`process.env` elsewhere fails review. Importing `serverEnv` into a client component is a
build error, which is exactly what you want — it means a server-only value can't
accidentally reach the browser.

---

## Generating secrets

```bash
# JWT signing secret
python3 -c "import secrets; print(secrets.token_urlsafe(48))"

# Database password
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Never reuse a secret across environments. Staging and production must not share a
`JWT_SECRET` — if they did, a staging token would authenticate against production.

---

## CI and deploy secrets

GitHub Actions secrets, set per repo, referenced never echoed:

```yaml
- name: Run tests against Postgres
  env:
    DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test
    JWT_SECRET: ${{ secrets.CI_JWT_SECRET }}
  run: pytest
```

Rules:

- CI uses **throwaway** secrets, never production ones. CI has no reason to hold a
  credential that can touch real user data.
- No secret in a workflow's `env:` block as a literal.
- Never `echo` a secret, and never `set -x` in a step that handles one. GitHub masks known
  secret values in logs, but masking is a backstop, not a strategy.
- Workflows triggered by `pull_request` from a fork do not receive secrets. That's a
  feature; don't work around it.
- Scope deploy tokens to the minimum they need, and rotate them on a schedule.

---

## Secrets in staging and production

Provider secret stores, not files on disk. Injected as environment variables at process
start.

- Secrets are set through the host's dashboard or CLI, never committed to any repo,
  including a private one.
- Changing a secret requires a restart. Plan it like a deploy.
- Every secret has a named owner and a rotation interval in an offline inventory. That
  inventory is not in git.

| Secret | Rotation |
| --- | --- |
| `JWT_SECRET` | Annually, or immediately on suspicion. Rotation invalidates all sessions — support two keys during the overlap. |
| Database password | Annually, or on contributor offboarding |
| `TMDB_API_KEY` | On suspicion |
| OAuth client secrets | Annually |
| Deploy tokens | Quarterly |
| `SENTRY_DSN` | Low sensitivity; rotate if abused |

---

## If you commit a secret

Assume it is compromised the instant it hits a public repo. Bots scan GitHub's event
firehose within seconds, and a TMDB key or a database password will be used.

**Order matters. Do not start with the git history.**

1. **Rotate the credential.** Right now, before anything else. Revoke the old value at the
   provider. A rewritten history with a live leaked key is worse than nothing, because it
   feels solved.
2. **Assess the damage.** What could the credential reach? Check provider access logs and
   [`../operations/incident-response.md`](../operations/incident-response.md) — a leaked
   production database credential is a Sev-1.
3. **Remove it from history**, with `git filter-repo`, and force-push:

   ```bash
   git filter-repo --path .env --invert-paths
   git push --force --all
   git push --force --tags
   ```

   Then contact GitHub Support to purge cached views of the old commits — forks and the
   commit cache survive a force-push.
4. **Tell everyone** who has a clone; they need to re-clone, not pull.
5. **Fix the cause.** Usually a missing gitignore entry or a `git add -A`. Fix that too, or
   it happens again.

---

## Prevention

- **`gitleaks` runs in CI** on every PR in all three repos, and it's a required check.
- Install the pre-commit hook locally so you find it before you push:

  ```bash
  pip install pre-commit && pre-commit install
  ```

  ```yaml
  # .pre-commit-config.yaml
  repos:
    - repo: https://github.com/gitleaks/gitleaks
      rev: v8.18.4
      hooks: [{ id: gitleaks }]
    - repo: https://github.com/pre-commit/pre-commit-hooks
      rev: v4.6.0
      hooks:
        - id: detect-private-key
        - id: check-added-large-files
          args: ['--maxkb=500']
  ```
- GitHub secret scanning and push protection enabled on all three repos — free for public
  repos, and it will block the push outright.
- Dependabot enabled for pip, npm, and GitHub Actions.
- `git diff --staged` before every commit. It takes three seconds and catches most of this.

---

## What is safe to commit

For the avoidance of doubt:

- ✅ `.env.example` with placeholders
- ✅ Public TMDB image URLs and poster paths
- ✅ Theatre names, addresses, and seat layout geometry — public information, hand-authored
- ✅ Outbound booking URL templates for theatre chains
- ✅ The scoring weights in `app/ml/scoring.py`. These are product decisions and belong in
  review, not in a secret store.
- ✅ OAuth **client IDs** (public by design). Not client secrets.
- ✅ Sentry DSNs for the web app — they're designed to be public. Still injected as env
  vars for environment separation.
- ❌ Anything else that looks like a credential
