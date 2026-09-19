# Database Conventions

Postgres 16 in staging and production. SQLite in local development. SQLAlchemy 2.0 ORM,
Alembic for migrations. The full schema is specified in
[`../architecture/data-model.md`](../architecture/data-model.md); this document is the
rules for changing it.

---

## The two-engine constraint

**This is the single most important thing on this page.** Docker is not installed on the
founder's development machine, and requiring it would put a container runtime between a
new contributor and their first `uvicorn`. So:

- **Local dev defaults to SQLite**, a file at `./goldseats.db`. `git clone`,
  `pip install`, `alembic upgrade head`, `uvicorn` — no services to start.
- **Production and staging run Postgres.**
- The switch is purely `DATABASE_URL`. No code branches on engine type outside
  `app/db/session.py`.

```python
# app/config.py
class Settings(BaseSettings):
    database_url: str = "sqlite+aiosqlite:///./goldseats.db"
```

```bash
# .env.example
# SQLite by default — no Docker required. See engineering/database-conventions.md
DATABASE_URL=sqlite+aiosqlite:///./goldseats.db
# Production:
# DATABASE_URL=postgresql+asyncpg://goldseats:password@localhost:5432/goldseats
```

The reasoning is recorded in [ADR-0002](../architecture/adr/0002-stack-selection.md).

### What this rules out

Do not use, anywhere in models, queries, or migrations:

| Off-limits | Use instead |
| --- | --- |
| `JSONB` operators (`->>`, `@>`, `jsonb_path_query`) | Store JSON as `TEXT`, parse in Python, extract anything queryable into a real column |
| `ARRAY` columns | A join table |
| Postgres `ENUM` types | `String` column + a Python `StrEnum` + a `CHECK` constraint |
| `ILIKE` | `func.lower(col).like(...)` — see below |
| `tsvector` full-text search | `LIKE` on a lowercased column for now; revisit at [M9](../product/milestones/M9.md) if search becomes a real feature |
| Window functions in hot paths | Compute in Python; our result sets are small |
| `CREATE INDEX CONCURRENTLY` in a migration | Plain `CREATE INDEX`; our tables are small enough that the lock is measured in milliseconds |
| Partial and expression indexes | Plain indexes |
| `NOW()` / `CURRENT_TIMESTAMP` defaults | Set timestamps in Python, so both engines agree on the value and the timezone |

Case-insensitive title search — the filter on the [M3](../product/milestones/M3.md) home
page — is the one that bites, because `ILIKE` works in Postgres and silently doesn't exist
in SQLite:

```python
# Works on both engines. Index func.lower(Film.title) in Postgres.
stmt = stmt.where(func.lower(Film.title).like(f"%{title.lower()}%"))
```

### Parity is enforced in CI

The API test job runs twice: once on SQLite, once against a real Postgres service
container. Both are required checks. That parity run is the only thing standing between us
and "worked locally, 500s in production", so it doesn't get skipped when it's
inconvenient.

Before merging anything schema-adjacent, run it locally against Postgres too. If you don't
have one, the CI run is your check — say so in the PR.

---

## Naming

| Thing | Convention | Example |
| --- | --- | --- |
| Table | `snake_case`, **plural** | `seat_availability_snapshots` |
| Column | `snake_case`, singular | `original_language` |
| Primary key | `id` | `id` |
| Foreign key | `<singular_table>_id` | `auditorium_id` |
| Boolean | `is_` / `has_` prefix | `is_available`, `has_recliners` |
| Timestamp | `_at` suffix, tz-aware UTC | `created_at`, `starts_at` |
| Date, no time | `_date` suffix | `release_date` |
| Duration | unit in the name | `runtime_minutes`, `hold_duration_seconds` |
| Measurement | unit in the name | `screen_width_m`, `distance_km` |
| Money | `_cents` + a `currency` column | `amount_cents` |
| Index | `ix_<table>_<cols>` | `ix_showtimes_starts_at` |
| Unique constraint | `uq_<table>_<cols>` | `uq_follows_user_id_target` |
| Check constraint | `ck_<table>_<description>` | `ck_seats_row_index_non_negative` |
| Foreign key constraint | `fk_<table>_<col>` | `fk_showtimes_film_id` |

Set the naming convention on the metadata so Alembic generates these automatically —
otherwise you get engine-dependent names and downgrades that don't work:

```python
# app/db/base.py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(table_name)s_%(column_0_N_name)s",
    "uq": "uq_%(table_name)s_%(column_0_N_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

Never use a SQL reserved word as a column name. `order`, `user`, `format`, `end` are all
traps. We have `showtime_formats.format_code`, not `format`.

---

## Primary keys

**UUIDv7 for every table**, stored as `CHAR(36)` on SQLite and `UUID` on Postgres via
SQLAlchemy's `Uuid` type.

Why UUIDv7 rather than a sequential integer:

- IDs are generated client-side of the database, so ingestion and the seeding CLI can
  build a whole object graph before the first `INSERT`.
- No ID enumeration on a public API.
- Time-ordered, so index locality is nearly as good as a serial — which is what makes it
  better than UUIDv4 here.

```python
# app/db/models/film.py
from uuid import UUID
from sqlalchemy import Uuid
from sqlalchemy.orm import Mapped, mapped_column
from app.db.ids import uuid7

class Film(Base):
    __tablename__ = "films"

    id: Mapped[UUID] = mapped_column(Uuid, primary_key=True, default=uuid7)
```

Join tables get a surrogate `id` too. It makes `DELETE /v1/me/follows/{follow_id}` trivial
and costs one column.

---

## Every table gets these

```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=utcnow,
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=utcnow, onupdate=utcnow,
    )
```

`utcnow` is ours and returns a tz-aware UTC datetime. Not `datetime.utcnow` (naive, and
deprecated), not `func.now()` (engine-dependent).

**All datetimes are stored in UTC and are tz-aware.** SQLite has no timezone type, so a
naive datetime will round-trip *looking* fine and be wrong by five hours in production.
This has already caused one real bug with Toronto evening showtimes.

For showtimes we store three things, deliberately redundantly:

```python
class Showtime(Base, TimestampMixin):
    __tablename__ = "showtimes"

    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    # IANA zone of the theatre, e.g. "America/Toronto". Needed to render "7:30 PM"
    # without the client guessing, and to answer "what's on tonight" correctly.
    timezone: Mapped[str] = mapped_column(String(64), nullable=False)
    starts_at_local: Mapped[datetime] = mapped_column(DateTime(timezone=False), nullable=False)
```

---

## Columns

- **`nullable=False` by default.** Nullable is a decision you justify, not a default you
  fall into. Every nullable column should have an obvious answer to "what does NULL mean
  here?"
- Always bound a `String`: `String(255)`. SQLite ignores the length, Postgres doesn't, and
  an unbounded `TEXT` for a country code is an invitation.
- Enum-like columns are `String` + a Python `StrEnum` + a `CHECK`:

  ```python
  class SeatType(StrEnum):
      REGULAR = "regular"
      RECLINER = "recliner"
      WHEELCHAIR = "wheelchair"
      COMPANION = "companion"

  seat_type: Mapped[str] = mapped_column(
      String(32),
      nullable=False,
      default=SeatType.REGULAR,
  )
  __table_args__ = (
      CheckConstraint(
          "seat_type IN ('regular', 'recliner', 'wheelchair', 'companion')",
          name="seat_type_valid",
      ),
  )
  ```

  A Postgres `ENUM` would be nicer and doesn't exist in SQLite. Adding a value to a
  `CHECK` is also a plain migration rather than a locking `ALTER TYPE`.
- Numbers: `Integer` for counts and indexes, `Float` for geometry in metres, `Numeric` for
  anything where rounding matters. Never `Float` for money.
- **JSON is a last resort.** `seat_layouts.layout_json` is the sanctioned use, because the
  geometry is genuinely a document and is read whole or not at all. Anything we filter or
  sort on gets extracted into a real column at write time.

---

## Indexes

Index, at minimum:

- Every foreign key. Postgres does not do this for you.
- Every column a documented filter targets.
- Any column in an `ORDER BY` on a paginated endpoint.

```python
__table_args__ = (
    Index("ix_showtimes_film_id_starts_at", "film_id", "starts_at"),
    Index("ix_showtimes_auditorium_id_starts_at", "auditorium_id", "starts_at"),
    UniqueConstraint("auditorium_id", "starts_at", name="auditorium_id_starts_at"),
)
```

Composite index column order matters: equality predicates first, then range, then sort.
`(film_id, starts_at)` serves "showtimes for this film, tonight, in time order".
`(starts_at, film_id)` doesn't.

Don't index speculatively. Every index costs write throughput and space. Add one when you
have a query that needs it, and say which query in the PR.

---

## Relationships

Explicit on both sides, with typed `Mapped` annotations and an explicit cascade:

```python
class Auditorium(Base, TimestampMixin):
    __tablename__ = "auditoriums"

    theatre_id: Mapped[UUID] = mapped_column(
        ForeignKey("theatres.id", ondelete="CASCADE"), nullable=False, index=True,
    )
    theatre: Mapped["Theatre"] = relationship(back_populates="auditoriums")
    seat_layouts: Mapped[list["SeatLayout"]] = relationship(
        back_populates="auditorium",
        cascade="all, delete-orphan",
        order_by="SeatLayout.version.desc()",
    )
```

`ondelete` rules for our schema:

| Relationship | `ondelete` | Why |
| --- | --- | --- |
| `auditoriums.theatre_id` | `CASCADE` | An auditorium has no meaning without its theatre |
| `seats.seat_layout_id` | `CASCADE` | Seats belong to a layout version |
| `showtimes.film_id` | `RESTRICT` | Deleting a film with showtimes is a data bug; make it fail loudly |
| `recommendations.showtime_id` | `CASCADE` | A recommendation is worthless without its showtime |
| `recommendations.user_id` | `SET NULL` | Anonymous recommendations are valid, and this is how account deletion works |
| `follows.user_id` | `CASCADE` | Deleting an account removes its follows |
| `booking_links.showtime_id` | `CASCADE` | |

**SQLite does not enforce foreign keys unless you ask it to.** Turn them on for every
connection or local dev silently permits orphans that production rejects:

```python
# app/db/session.py
if url.startswith("sqlite"):
    @event.listens_for(engine.sync_engine, "connect")
    def _sqlite_pragmas(dbapi_conn, _):
        cur = dbapi_conn.cursor()
        cur.execute("PRAGMA foreign_keys=ON")
        cur.execute("PRAGMA journal_mode=WAL")
        cur.close()
```

---

## Queries

All queries live in `app/db/repositories/`. Routes never build a `select()`.

- **Bind every parameter.** No f-strings, no `.format()`, no `%` in SQL. A string-built
  query is an automatic block in review.
- **Eager-load everything you serialise.** `selectinload` for collections, `joinedload`
  for many-to-one. An N+1 on `/v1/films/now-playing` will fail our Lighthouse budget.
- **Every list query is paginated and has a deterministic `order_by` ending in `id`.**
  Without the tiebreaker, page 2 can repeat a row from page 1.
- `select()` 2.0 style only. No legacy `session.query()`.
- Never `SELECT *` into Python and filter there. Filter in SQL.

```python
async def list_showtimes_for_film(
    session: AsyncSession, film_id: UUID, *, on_date: date, page: PageParams,
) -> list[Showtime]:
    stmt = (
        select(Showtime)
        .where(Showtime.film_id == film_id)
        .where(Showtime.starts_at >= start_of_day_utc(on_date))
        .where(Showtime.starts_at < start_of_day_utc(on_date + timedelta(days=1)))
        .options(
            joinedload(Showtime.auditorium).joinedload(Auditorium.theatre),
            selectinload(Showtime.formats),
            selectinload(Showtime.booking_links),
        )
        .order_by(Showtime.starts_at, Showtime.id)
        .limit(page.limit)
        .offset(page.offset)
    )
    return list((await session.scalars(stmt)).unique().all())
```

Sessions are per-request, injected with `Depends(get_session)`, and commit once at the end
of a successful request. Ingestion jobs and the seeding CLI manage their own sessions and
commit in batches.

---

## Migrations

Alembic. Every schema change is a migration; nothing is ever applied by hand.

```bash
alembic revision --autogenerate -m "add language metadata to showtimes"
alembic upgrade head
alembic downgrade -1
alembic current
alembic history --verbose
```

Rules:

1. **Autogenerate produces a draft, not a migration.** Read it. Alembic misses
   constraint renames, server defaults, and index changes, and it happily emits things
   SQLite can't run.
2. **`downgrade()` is mandatory and must actually work.** The round-trip test in
   `tests/db/test_migrations.py` enforces it. This is what makes rollback safe.
3. **Batch mode for any `ALTER` on SQLite.** SQLite can't drop a column or alter a type;
   Alembic's batch mode rebuilds the table:

   ```python
   def upgrade() -> None:
       with op.batch_alter_table("showtimes") as batch:
           batch.add_column(sa.Column("language_code", sa.String(8), nullable=True))
   ```
4. **Backwards compatible with deployed code.** Expand in one release, contract in a later
   one. See
   [`branching-and-releases.md`](branching-and-releases.md#migrations-and-release-ordering).
   No renaming a column and no dropping a column still read by production in a single
   release.
5. **A new `NOT NULL` column needs three steps**, not one: add nullable, backfill, then
   set not-null.
6. **Data migrations use `op.execute` with Core constructs, not ORM models.** Importing a
   model into a migration means the migration breaks the next time the model changes.
7. **One migration per PR.** Two revisions in one PR means two branch points and a merge
   conflict waiting to happen.
8. Migrations are immutable once merged. Fix a bad one with a new revision.
9. Every migration runs on both engines. CI checks it.

Migration message style matches Conventional Commit scopes: `add language metadata to
showtimes`, `create seat_layouts and seats`, `backfill showtime timezones`.

---

## Seed data

Two distinct things, and they don't mix:

- **`app/cli/seed_theatre.py`** — the real seeding CLI from
  [M2](../product/milestones/M2.md). Idempotent, keyed on a natural key so re-running
  updates rather than duplicating. Reads versioned `seat_layout` JSON files that are
  committed to the repo. This is production data creation and it's reviewable as code.
- **`tests/fixtures/`** — factory functions for tests. Never used outside tests.

```bash
python -m app.cli.seed_theatre --file seeds/theatres/cineplex-scotiabank-toronto.json
python -m app.cli.seed_theatre --all --dry-run
```

Theatre layouts are hand-authored from real auditoriums, reusing the geometry logic from
the dataset generator so seeded layouts match what the scoring model was tuned on. We do
not scrape them — see
[`../legal/data-sources-policy.md`](../legal/data-sources-policy.md) and
[ADR-0003](../architecture/adr/0003-manual-seed-theatre-data.md).

---

## Layout versioning

`seat_layouts` is append-only and versioned per auditorium. A theatre reconfiguring to
recliners creates version 2; version 1 stays, because recommendations already computed
against it must remain interpretable.

- `uq_seat_layouts_auditorium_id_version`
- `seat_layouts.is_current` marks the one to use for new recommendations, with a partial
  guarantee enforced in application code (a partial unique index would break SQLite).
- `recommendations.seat_layout_version` records which version a recommendation used, and
  the API returns it.
- `layout_json` carries its own `schema_version` so the parser can handle old documents.

---

## Deleting user data

Account deletion is a legal obligation under
[`../legal/privacy-policy.md`](../legal/privacy-policy.md), so it's a schema concern:

- `users` rows are **hard deleted**. No soft-delete tombstone holding an email address
  forever.
- `follows`, `watchlist_entries`, and chat conversations cascade.
- `recommendations.user_id` is `SET NULL`, leaving an anonymous row. The seat scores are
  useful aggregate data and contain nothing identifying once detached.
- Deletion is a single transaction and there's a test proving nothing identifying survives
  it.

---

## Backups

Production Postgres: automated daily base backups plus continuous WAL archiving, 30-day
retention. Restores are drilled, not assumed — see
[`../operations/runbooks/database-restore.md`](../operations/runbooks/database-restore.md).
The first successful timed restore drill is a [M9](../product/milestones/M9.md)
deliverable.

Local SQLite is not backed up. It's disposable: `rm goldseats.db && alembic upgrade head &&
python -m app.cli.seed_theatre --all`.
