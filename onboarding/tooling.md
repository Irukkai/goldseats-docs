# Tooling

Everything you need installed, configured, and the commands you'll actually type.

Setup order is in [`local-setup.md`](local-setup.md). This is the reference.

---

## Required

| Tool | Version | Check | Install |
| --- | --- | --- | --- |
| Node | 20.19.0+ | `node --version` | `brew install node@20` |
| npm | 10.8.2+ | `npm --version` | ships with Node |
| Python | 3.12.4+ | `python3 --version` | `brew install python@3.12` |
| git | any recent | `git --version` | Xcode CLT or `brew install git` |
| GitHub CLI | 2.100.0+ | `gh --version` | `brew install gh` |

**Not required, deliberately:** Docker. Local development runs on SQLite with no services, because
requiring a container runtime puts a several-hundred-megabyte download and a class of
"why-won't-my-container-start" problems between a new contributor and their first `uvicorn`. See
[ADR-0002](../architecture/adr/0002-stack-selection.md).

### Optional

| Tool | When you need it | Install |
| --- | --- | --- |
| Redis | Working on caching or rate limiting | `brew install redis` |
| Ollama | Working on the chatbot ([M7](../product/milestones/M7.md)) | `brew install ollama` |
| Postgres client | Debugging a parity failure or a production issue | `brew install libpq` |
| `jq` | Reading API responses in the terminal | `brew install jq` |
| `k6` | Load testing ([M9](../product/milestones/M9.md)) | `brew install k6` |

---

## Authenticate `gh` first

Nothing pushes until this is done:

```bash
gh auth login --hostname github.com --git-protocol https --web
gh auth setup-git
gh auth status
```

---

## Python toolchain

All inside the repo's virtualenv. **Activate it in every terminal.**

```bash
cd goldseats-api
source .venv/bin/activate
```

| Tool | Role |
| --- | --- |
| **pip** | Package manager. `requirements.txt` for runtime, `requirements-dev.txt` for tooling. Not Poetry, not uv. |
| **Ruff** | Lint *and* format. Replaces Black, isort, flake8, pyupgrade. |
| **mypy** | Type checking, strict, on `app/`. |
| **pytest** | Tests. |
| **Alembic** | Migrations. |
| **uvicorn** | Dev server. |

```bash
ruff check .              # lint
ruff check . --fix        # autofix
ruff format .             # format
ruff format --check .     # what CI runs
mypy app
pytest
pytest tests/unit -q                      # fast loop, ~2s
pytest tests/unit/ml/test_scoring.py -k contiguous
pytest --cov=app --cov-report=term-missing
pytest --benchmark-only                   # scoring benchmarks
```

---

## Node toolchain

```bash
cd goldseats-web
```

| Tool | Role |
| --- | --- |
| **npm** | Package manager. `package-lock.json` committed, `npm ci` in CI. Not pnpm, not yarn. |
| **TypeScript** | Strict, plus `noUncheckedIndexedAccess`. |
| **ESLint** | `next/core-web-vitals`, type-aware `@typescript-eslint`, `jsx-a11y`. |
| **Prettier** | All formatting. No formatting arguments in review. |
| **Vitest** | Unit and component tests. |
| **Playwright** | End-to-end. |

```bash
npm run dev
npm run build
npm run start
npm run lint
npm run typecheck
npm run test
npm run test:watch
npm run generate:api-types       # from the API's OpenAPI schema — never hand-write these

npx playwright test
npx playwright test --ui
npx playwright install --with-deps
npx lhci autorun                 # Lighthouse
```

---

## The commands CI runs

Run these before every push. A local failure is a guaranteed CI failure and a wasted round trip.

```bash
# API
ruff check . && ruff format --check . && mypy app && pytest

# Web
npm run lint && npm run typecheck && npm run test && npm run build
```

---

## Pre-commit hooks

Install these. The repos are public and a committed secret is a published secret.

```bash
cd goldseats-api        # and again in goldseats-web
pip install pre-commit
pre-commit install
pre-commit run --all-files   # first run, to see what it catches
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
      - id: end-of-file-fixer
      - id: trailing-whitespace
```

---

## Editor — VS Code / Cursor

Extensions:

| Extension | Why |
| --- | --- |
| `charliermarsh.ruff` | Lint and format on save for Python |
| `ms-python.python` + `ms-python.vscode-pylance` | Interpreter and types |
| `ms-python.mypy-type-checker` | Inline type errors |
| `dbaeumer.vscode-eslint` | Inline lint |
| `esbenp.prettier-vscode` | Format on save |
| `bradlc.vscode-tailwindcss` | Tailwind class completion and token hints |
| `tamasfe.even-better-toml` | `pyproject.toml` |
| `ms-playwright.playwright` | Run E2E tests from the editor |

`.vscode/settings.json` — committed in both repos so everyone gets the same behaviour:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.organizeImports": "explicit" },
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.codeActionsOnSave": { "source.fixAll.ruff": "explicit" }
  },
  "[typescript]":      { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.testing.pytestEnabled": true,
  "files.exclude": { "**/__pycache__": true, "**/.pytest_cache": true, "**/.next": true }
}
```

Point your interpreter at `.venv/bin/python`. If Pylance can't find imports, that's almost always
why.

---

## git configuration

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"

git config --global pull.ff only            # never an accidental merge commit
git config --global rebase.autosquash true
git config --global init.defaultBranch main
git config --global push.autoSetupRemote true

git config --global alias.st "status -sb"
git config --global alias.lg "log --oneline --graph --decorate -20"
git config --global alias.up "pull --ff-only"
git config --global alias.pf "push --force-with-lease"
```

`pull.ff only` matters: we require linear history, and an accidental merge commit means a rebase
you didn't plan for.

---

## `gh` commands you'll use

```bash
gh pr create --fill
gh pr create --draft
gh pr list
gh pr checkout 42
gh pr diff 42
gh pr review 42 --approve
gh pr checks
gh pr merge 42 --squash --delete-branch

gh issue create --title "..." --body "..." --milestone M5
gh issue list --milestone M5

gh run list --limit 5
gh run watch
gh run view --log-failed      # the useful one when CI fails

gh release list
```

---

## Ollama — chatbot work only

Only needed for [M7](../product/milestones/M7.md). `FEATURE_CHAT_ENABLED` defaults to `false`, so
everything else works without it.

```bash
brew install ollama
ollama serve                              # its own terminal
ollama pull llama3.1:8b-instruct-q4_K_M
ollama list
ollama run llama3.1:8b-instruct-q4_K_M    # sanity check
```

Then set `LLM_BASE_URL=http://localhost:11434/v1`, `LLM_MODEL=llama3.1:8b-instruct-q4_K_M`, and
`FEATURE_CHAT_ENABLED=true` in `.env`.

Expect several GB of download and real memory use. Production serves the same interface via vLLM,
which is why switching is an env var — see
[ADR-0004](../architecture/adr/0004-self-hosted-llm.md).

---

## Dataset generator

The synthetic seat-map generator lives in `goldseats-api/ml/dataset/`. It produced the training
layouts the scoring model was tuned on, and it still generates test fixtures and the chatbot's
evaluation set.

```bash
cd goldseats-api/ml/dataset
python generate.py --count 10 --preview --seed 42
python visualize.py
```

`output/` is gitignored — it's reproducible from a seed, so there's no reason to commit PNGs.

---

## Useful one-liners

```bash
# Pretty-print an endpoint
curl -s 'localhost:8000/v1/films/now-playing?limit=3' | jq '.data[] | {title, original_language}'

# Ask for a recommendation
curl -s -X POST localhost:8000/v1/recommendations \
  -H 'content-type: application/json' \
  -d '{"showtime_id":"<uuid>","party_size":2,"preferences":{"aisle_preferred":true}}' \
  | jq '.options[0] | {rank, score, seats, factors}'

# Reset the local database
rm goldseats.db* && alembic upgrade head && python -m app.cli.seed_theatre --all

# Which migration am I on?
alembic current && alembic heads

# Free a stuck port
lsof -ti:8000 | xargs kill

# What's in the local database?
sqlite3 goldseats.db ".tables"
sqlite3 goldseats.db "SELECT count(*) FROM seats;"
```

---

## Related

- [`local-setup.md`](local-setup.md) — setup order and troubleshooting
- [`day-one.md`](day-one.md) — what to read and ship first
- [`../engineering/code-conventions-python.md`](../engineering/code-conventions-python.md)
- [`../engineering/code-conventions-typescript.md`](../engineering/code-conventions-typescript.md)
- [`../engineering/secrets-and-config.md`](../engineering/secrets-and-config.md)
