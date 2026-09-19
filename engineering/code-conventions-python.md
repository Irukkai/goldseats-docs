# Python Conventions (`goldseats-api`)

Applies to everything in `goldseats-api`: Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic,
Pydantic v2, and the seat scoring engine in `app/ml/`. Enforced by Ruff and mypy in CI.

---

## Tooling

- **Python 3.12.** Pinned in `.python-version` and in CI. Use 3.12 syntax freely —
  `X | None`, `match`, `type` aliases.
- **pip + `requirements.txt`.** Not Poetry, not uv. `requirements.txt` for runtime,
  `requirements-dev.txt` for tooling. Pin exact versions in both.
- **Ruff** for lint *and* format. It replaces Black, isort, flake8, and pyupgrade.
- **mypy** in strict mode on `app/`.

```bash
ruff check .            # lint
ruff format --check .   # formatting
mypy app                # types
pytest                  # tests
```

Run all four before pushing. CI runs exactly these.

`pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "C4", "SIM", "TCH", "RUF", "ASYNC", "S"]
ignore = ["S101"]  # assert is fine in tests

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
```

---

## Project layout

```
app/
  main.py               # FastAPI app factory, middleware, router registration
  config.py             # pydantic-settings Settings, the ONLY place os.environ is read
  api/
    deps.py             # shared dependencies: db session, current user, pagination
    v1/
      films.py          # /v1/films/*
      theatres.py
      showtimes.py
      recommendations.py
      chat.py
      follows.py
  domain/               # business logic. No FastAPI imports, no SQLAlchemy sessions created here.
    films.py
    seating.py
    formats.py          # "best way to experience" format ranking (M4)
  ml/
    scoring.py          # the seat scoring engine, promoted from the dataset generator
    dataset/            # migrated synthetic dataset generator
  db/
    base.py             # DeclarativeBase
    session.py          # engine + sessionmaker, chooses SQLite vs Postgres
    models/             # one module per table group
    repositories/       # query objects; all SQL lives here
  schemas/              # Pydantic request/response models
  ingest/
    tmdb.py             # TMDB film metadata ingestion (M1)
    news.py             # news aggregation (M8)
  cli/
    seed_theatre.py     # admin seeding CLI (M2)
  observability/
    logging.py
    tracing.py
alembic/versions/
tests/
```

The dependency rule, and it's one-directional:

```
api  →  domain  →  db.repositories  →  db.models
 ↓         ↓
schemas   ml
```

`domain/` and `ml/` must not import from `api/`. `ml/scoring.py` must not import
SQLAlchemy at all — it takes plain dataclasses in and returns plain dataclasses out, which
is what makes it testable against the synthetic dataset fixtures.

---

## Naming

| Thing | Convention | Example |
| --- | --- | --- |
| Module | `snake_case.py`, singular | `showtime.py` |
| Class | `PascalCase` | `SeatAvailabilitySnapshot` |
| Function / variable | `snake_case` | `score_seat_block` |
| Constant | `SCREAMING_SNAKE_CASE` | `THX_TARGET_VIEW_ANGLE_DEG` |
| Private | single leading underscore | `_resolve_scenario` |
| SQLAlchemy model | singular class, plural table | `class Showtime` → `showtimes` |
| Pydantic schema | suffix by role | `FilmRead`, `FilmCreate`, `RecommendationRequest` |

Table and column naming is fixed by [`database-conventions.md`](database-conventions.md).
Domain vocabulary is fixed by [`../product/glossary.md`](../product/glossary.md).

---

## Typing

Annotate everything. mypy strict enforces it, so this is mostly a reminder of style:

```python
from collections.abc import Sequence
from datetime import datetime
from uuid import UUID

def rank_formats(
    showtimes: Sequence[Showtime],
    *,
    prefer_premium: bool = True,
) -> list[FormatRanking]:
    ...
```

- Built-in generics: `list[str]`, `dict[str, int]`. Never `typing.List`.
- `X | None`, never `Optional[X]`.
- Accept `Sequence`/`Iterable`, return concrete `list`/`dict`.
- Keyword-only arguments for anything optional or boolean. `score_seats(layout, party_size=2, aisle_preferred=True)` reads; `score_seats(layout, 2, True)` does not.
- Import typing-only symbols under `if TYPE_CHECKING:` (Ruff's `TCH` rules enforce it).

Use frozen dataclasses for the scoring engine's interface — no ORM objects cross that
boundary:

```python
# app/ml/scoring.py
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class SeatGeometry:
    row_index: int
    seat_index: int
    x: float           # metres from auditorium centreline, negative = left
    y: float           # metres from screen
    elevation: float   # metres above screen-floor datum
    is_available: bool

@dataclass(frozen=True, slots=True)
class SeatScore:
    row_index: int
    seat_index: int
    score: float                     # 0–100
    factors: dict[str, float]        # per-factor contributions, for explainability
```

---

## The scoring engine

`app/ml/scoring.py` is the most important file in the repo and gets extra rules. The
weights are product decisions, not implementation details:

```python
# app/ml/scoring.py
SCORING_WEIGHTS: Final[dict[str, float]] = {
    "horizontal_centre": 0.25,
    "vertical_position": 0.25,   # peaks around 50% auditorium depth
    "view_angle": 0.20,          # THX-inspired ~36 degree horizontal target
    "neighbour_openness": 0.15,
    "row_quality": 0.15,
}
THX_TARGET_VIEW_ANGLE_DEG: Final[float] = 36.0
```

- Weights live in exactly one named constant. Never inline a magic number in a factor
  function.
- Every factor function returns a normalised `0.0–1.0` and is independently unit tested.
- Always populate `SeatScore.factors`. The chatbot in [M7](../product/milestones/M7.md)
  explains its recommendations from these, and the 2D map in
  [M5](../product/milestones/M5.md) shows them on hover. A score with no breakdown is
  unusable to both.
- Changing a weight requires the before/after distribution on the golden fixtures in the
  PR body — see [`code-review.md`](code-review.md).
- The engine is pure and synchronous. No I/O, no `async`, no database, no clock. Pass
  time in if you need it.

---

## Async and blocking

Routes are `async def`. Everything they await must genuinely be non-blocking.

```python
@router.get("/now-playing", response_model=Page[FilmSummary])
async def now_playing(
    filters: Annotated[FilmFilters, Query()],
    page: Annotated[PageParams, Depends()],
    session: Annotated[AsyncSession, Depends(get_session)],
) -> Page[FilmSummary]:
    films = await film_repository.list_now_playing(session, filters, page)
    return Page.of(films, page)
```

- Use `AsyncSession` and `httpx2.AsyncClient` throughout. Never `requests` in request
  handling.
- CPU-bound work — which includes seat scoring for a large auditorium — goes through
  `anyio.to_thread.run_sync` so it doesn't stall the event loop:

  ```python
  scores = await anyio.to_thread.run_sync(
      partial(score_layout, geometry, scenario=scenario)
  )
  ```
- If a function isn't awaiting anything, make it `def`. A pointless `async def` just
  moves the blocking somewhere harder to see.

---

## Database access

All queries live in `app/db/repositories/`. Routes never build a `select()`.

```python
# app/db/repositories/films.py
async def list_now_playing(
    session: AsyncSession,
    filters: FilmFilters,
    page: PageParams,
) -> list[Film]:
    stmt = (
        select(Film)
        .join(Showtime, Showtime.film_id == Film.id)
        .where(Showtime.starts_at >= func.now())
        .options(selectinload(Film.releases))
        .order_by(Film.popularity.desc())
        .limit(page.limit)
        .offset(page.offset)
        .distinct()
    )
    if filters.title:
        stmt = stmt.where(Film.title.ilike(f"%{filters.title}%"))
    if filters.language:
        stmt = stmt.where(Film.original_language == filters.language)

    return list((await session.scalars(stmt)).all())
```

- **Never** build SQL by string interpolation. Bind parameters, always.
- Always eager-load what the response serialises (`selectinload` / `joinedload`). An
  N+1 on `/v1/films/now-playing` is a guaranteed Lighthouse failure.
- Every list endpoint is paginated and has a deterministic `order_by`.
- The session is per-request, provided by `Depends(get_session)`, and commits once at the
  end of a successful request.

Read [`database-conventions.md`](database-conventions.md) for SQLite/Postgres parity
rules — several convenient Postgres features are off-limits precisely because local dev
runs on SQLite.

---

## Configuration

`app/config.py` is the only file allowed to read the environment.

```python
# app/config.py
from functools import lru_cache
from pydantic import Field, PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="forbid")

    # SQLite by default: Docker is not installed on dev machines. See ADR-0002.
    database_url: str = "sqlite+aiosqlite:///./goldseats.db"
    redis_url: str = "redis://localhost:6379/0"
    tmdb_api_key: str = ""
    llm_base_url: str = "http://localhost:11434/v1"  # Ollama locally, vLLM in prod
    jwt_secret: str = Field(min_length=32)
    log_level: str = "INFO"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`os.environ` anywhere outside this file fails review. Full rules in
[`secrets-and-config.md`](secrets-and-config.md).

---

## Errors

Domain code raises domain exceptions. A single handler maps them to HTTP.

```python
# app/domain/errors.py
class GoldSeatsError(Exception):
    """Base for all expected domain failures."""

class NotFoundError(GoldSeatsError):
    def __init__(self, entity: str, identifier: object) -> None:
        self.entity, self.identifier = entity, identifier
        super().__init__(f"{entity} {identifier!r} not found")

class NoAvailableSeatsError(GoldSeatsError):
    """Party cannot be seated contiguously in this auditorium."""
```

```python
# app/main.py
@app.exception_handler(NotFoundError)
async def handle_not_found(request: Request, exc: NotFoundError) -> JSONResponse:
    return problem_response(request, status=404, title="Not found", detail=str(exc))
```

- Never raise `HTTPException` from `domain/` or `ml/`. That couples business logic to the
  transport and makes the scoring engine untestable outside a web request.
- Never swallow an exception. No bare `except:`, no `except Exception: pass`.
- Error response bodies follow the problem-detail shape in
  [`api-design-guidelines.md`](api-design-guidelines.md).

---

## Logging

Structured, JSON, with context. Never `print`.

```python
import structlog

log = structlog.get_logger(__name__)

log.info(
    "recommendation.generated",
    showtime_id=str(showtime.id),
    party_size=request.party_size,
    scenario=scenario.value,
    top_score=round(scores[0].score, 1),
    candidates=len(scores),
    duration_ms=round(elapsed * 1000, 1),
)
```

Event names are `noun.verb_past_tense`. No f-strings in log messages — pass keyword
fields so they're queryable. Never log a JWT, a password, an email address, or a full
`seat_layout` blob. Details in [`observability.md`](observability.md).

---

## Docstrings and comments

Docstrings on every public module, class, and function. Google style, imperative mood,
one-line summary first.

```python
def score_layout(
    seats: Sequence[SeatGeometry],
    *,
    scenario: PartyScenario,
    screen_width_m: float,
) -> list[SeatScore]:
    """Score every available seat and return them ranked best-first.

    Combines five weighted factors defined in ``SCORING_WEIGHTS``. For group
    scenarios, only contiguous same-row blocks large enough for the party are
    scored, so a returned block is always bookable as-is.

    Args:
        seats: Full auditorium geometry, including unavailable seats — occupied
            neighbours still affect the openness factor.
        scenario: Resolved party scenario, which selects the block size.
        screen_width_m: Used for the view-angle factor.

    Returns:
        Scores sorted by ``score`` descending. Empty if the party cannot be
        seated contiguously.
    """
```

Comments explain *why*, never *what*. Good: `# THX puts the ideal horizontal viewing
angle near 36 degrees; we penalise deviation in both directions.` Bad:
`# loop over seats`.

---

## Testing

See [`testing-strategy.md`](testing-strategy.md). Shape of tests here:

```python
def test_group_block_is_contiguous_in_one_row() -> None:
    layout = fixture_layout("cineplex-scotiabank-toronto-auditorium-3")
    occupy(layout, row="H", seats=[10, 11])

    scores = score_layout(layout.seats, scenario="small_group", screen_width_m=16.0)

    best = scores[0]
    block = [s for s in scores[:3] if s.row_index == best.row_index]
    assert len(block) == 3
    assert [s.seat_index for s in block] == list(
        range(block[0].seat_index, block[0].seat_index + 3)
    )
```

Use real seeded auditorium fixtures, not `range(10)` grids — the geometry is the thing
under test.
