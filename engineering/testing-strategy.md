# Testing Strategy

What we test, with what, and to what standard. `pytest` for the API, `Vitest` and
`Playwright` for the web app. All four run in CI on every PR and gate merge.

---

## Principles

1. **Test behaviour, not implementation.** A test that breaks when you rename a private
   function, without any behaviour changing, is a liability.
2. **The pyramid is a budget, not a law.** Lots of fast unit tests, a decent layer of
   integration tests, a handful of end-to-end tests covering the flows that make money.
3. **Fixtures are real.** Seat scoring is tested against actual seeded auditorium
   geometry, not `range(10)` grids. The geometry *is* the thing under test.
4. **A bug fix ships with a regression test** that fails before the fix.
5. **Flaky tests get fixed or deleted the day they're noticed.** A suite people learn to
   re-run is a suite nobody trusts.
6. **Coverage is a smell detector, not a target.** We don't gate on a percentage. We do
   read the coverage diff on a PR and ask why an untested branch is untested.

---

## `goldseats-api` — pytest

```bash
pytest                              # everything
pytest tests/unit -q                # fast loop, ~2s
pytest tests/api/test_films.py -k now_playing
pytest --cov=app --cov-report=term-missing
```

### Layout

```
tests/
  conftest.py              # shared fixtures: settings override, engine, client
  fixtures/
    layouts/               # real seeded seat_layout JSON, committed
      cineplex-scotiabank-toronto-auditorium-3.json
      cineplex-yonge-dundas-imax.json
    tmdb/                  # recorded TMDB responses, keys stripped
    golden/                # expected score distributions for scoring regression
  unit/
    ml/test_scoring.py         # the most important test file in the repo
    ml/test_factors.py
    domain/test_formats.py
    domain/test_seating.py
  api/
    test_films.py
    test_showtimes.py
    test_recommendations.py
    test_auth.py
  db/
    test_migrations.py
  ingest/
    test_tmdb.py
```

### Unit tests — `app/ml/` and `app/domain/`

Pure functions, no database, no network, no clock. This is where most of our tests live
because this is where the product's actual intelligence lives.

Every one of the five scoring factors gets its own tests:

```python
# tests/unit/ml/test_factors.py
import pytest
from app.ml.scoring import horizontal_centre_factor, view_angle_factor

def test_horizontal_centre_peaks_at_centreline() -> None:
    assert horizontal_centre_factor(x=0.0, half_width_m=8.0) == pytest.approx(1.0)

@pytest.mark.parametrize("x", [-8.0, 8.0])
def test_horizontal_centre_bottoms_out_at_the_walls(x: float) -> None:
    assert horizontal_centre_factor(x=x, half_width_m=8.0) < 0.2

def test_view_angle_peaks_at_thx_target() -> None:
    # THX puts the ideal horizontal viewing angle near 36 degrees.
    best = view_angle_factor(distance_m=11.0, screen_width_m=16.0)
    too_close = view_angle_factor(distance_m=4.0, screen_width_m=16.0)
    too_far = view_angle_factor(distance_m=30.0, screen_width_m=16.0)
    assert best > too_close
    assert best > too_far
```

And the composed behaviour, against real fixtures:

```python
# tests/unit/ml/test_scoring.py
def test_weights_sum_to_one() -> None:
    assert sum(SCORING_WEIGHTS.values()) == pytest.approx(1.0)

def test_factors_are_always_reported_for_explainability() -> None:
    layout = load_layout("cineplex-scotiabank-toronto-auditorium-3")
    scores = score_layout(layout.seats, scenario="solo", screen_width_m=16.0)
    assert set(scores[0].factors) == set(SCORING_WEIGHTS)

def test_group_scenarios_return_contiguous_same_row_blocks() -> None:
    layout = load_layout("cineplex-scotiabank-toronto-auditorium-3")
    occupy(layout, row="H", seats=[10, 11])
    scores = score_layout(layout.seats, scenario="small_group", screen_width_m=16.0)
    best_row = scores[0].row_index
    block = [s for s in scores[:3] if s.row_index == best_row]
    assert [s.seat_index for s in block] == list(
        range(block[0].seat_index, block[0].seat_index + 3)
    )

def test_returns_empty_when_party_cannot_be_seated_together() -> None:
    layout = load_layout("cineplex-scotiabank-toronto-auditorium-3")
    occupy_all_but(layout, free_seats=[("D", 4), ("K", 19)])
    assert score_layout(layout.seats, scenario="pair", screen_width_m=16.0) == []

def test_front_row_never_outranks_mid_auditorium_for_solo() -> None:
    layout = load_layout("cineplex-scotiabank-toronto-auditorium-3")
    scores = score_layout(layout.seats, scenario="solo", screen_width_m=16.0)
    assert scores[0].row_index > 1
```

### Scoring regression tests

Scoring can degrade without any test failing, because "better seat" is a judgement. So we
pin the distribution and force a conscious decision when it moves:

```python
# tests/unit/ml/test_scoring.py
GOLDEN_LAYOUTS = [
    "cineplex-scotiabank-toronto-auditorium-3",
    "cineplex-yonge-dundas-imax",
    "cineplex-vip-varsity-auditorium-1",
]

@pytest.mark.parametrize("layout_id", GOLDEN_LAYOUTS)
@pytest.mark.parametrize("scenario", ["solo", "pair", "small_group", "large_group", "family"])
def test_scores_match_golden_snapshot(layout_id: str, scenario: str) -> None:
    """Fails on any scoring drift. If intentional, regenerate and justify in the PR.

    Regenerate with: pytest tests/unit/ml/test_scoring.py --snapshot-update
    """
    scores = score_layout(load_layout(layout_id).seats, scenario=scenario, screen_width_m=16.0)
    assert summarise(scores) == load_golden(layout_id, scenario)
```

Updating a golden file is allowed. Updating it without the before/after distribution in
the PR description is not — see [`code-review.md`](code-review.md).

### API tests

FastAPI `TestClient` against an in-memory SQLite database, per-test transaction rollback.
Real HTTP semantics, no mocked routers.

```python
# tests/conftest.py
@pytest.fixture
async def client(session: AsyncSession) -> AsyncIterator[AsyncClient]:
    app.dependency_overrides[get_session] = lambda: session
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://t") as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# tests/api/test_films.py
async def test_now_playing_filters_by_language(client: AsyncClient) -> None:
    await seed_film(title="Dune: Part Three", original_language="en", playing=True)
    await seed_film(title="Les Chambres Rouges", original_language="fr", playing=True)

    res = await client.get("/v1/films/now-playing", params={"language": "fr"})

    assert res.status_code == 200
    titles = [f["title"] for f in res.json()["data"]]
    assert titles == ["Les Chambres Rouges"]

async def test_unknown_film_returns_problem_detail(client: AsyncClient) -> None:
    res = await client.get(f"/v1/films/{uuid4()}")
    assert res.status_code == 404
    body = res.json()
    assert body["title"] == "Not found"
    assert "request_id" in body
```

Every route needs, at minimum: happy path, a 404, a validation 422, and — if it's
authenticated — a 401 and a cross-user 403.

### Migration tests

Every Alembic revision must round-trip. This is what makes rollback safe.

```python
# tests/db/test_migrations.py
def test_every_revision_upgrades_and_downgrades(alembic_config: Config) -> None:
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")
    command.upgrade(alembic_config, "head")

def test_models_match_migrations(alembic_config: Config) -> None:
    """No model change may land without a migration."""
    command.upgrade(alembic_config, "head")
    assert compare_metadata(migration_context(), Base.metadata) == []
```

### External services

Never hit a real external service in a test.

- **TMDB** — recorded responses in `tests/fixtures/tmdb/`, replayed via `respx`. Refresh
  them deliberately, in their own PR, with the API key stripped.
- **LLM** — a fake tool-calling transport that returns scripted tool calls. What we're
  testing is that the model's tool calls are correctly dispatched and that a reply
  containing a seat not present in the tool result is rejected — not the model's prose.
- **Redis** — `fakeredis`, plus one integration test against a real Redis in CI.

---

## `goldseats-web` — Vitest

```bash
npm run test           # vitest run
npm run test:watch
npm run test:coverage
```

Component tests with Testing Library. Query the way a user would.

```tsx
// src/components/films/FilmFilters.test.tsx
it('pushes the language filter into the URL without losing the title filter', async () => {
  const user = userEvent.setup();
  const router = mockRouter({ query: { title: 'dune' } });
  render(<FilmFilters />);

  await user.selectOptions(screen.getByRole('combobox', { name: 'Language' }), 'fr');

  expect(router.push).toHaveBeenCalledWith('/?title=dune&language=fr');
});
```

```tsx
// src/components/seats/SeatMap2D.test.tsx
it('is navigable by keyboard across a row', async () => {
  const user = userEvent.setup();
  render(<SeatMap2D layout={layoutFixture} recommendations={recommendationsFixture} />);

  await user.tab();
  expect(screen.getByRole('button', { name: /Row A, seat 1/ })).toHaveFocus();
  await user.keyboard('{ArrowRight}');
  expect(screen.getByRole('button', { name: /Row A, seat 2/ })).toHaveFocus();
});

it('announces seat quality in text, not only colour', () => {
  render(<SeatMap2D layout={layoutFixture} recommendations={recommendationsFixture} />);
  expect(
    screen.getByRole('button', { name: /Row H, seat 12.*score 98.*recommended/i }),
  ).toBeInTheDocument();
});
```

Pure logic in `src/lib/` gets plain unit tests — date formatting, language labels, format
ranking display order, URL builders.

**We do not unit-test the 3D scene.** Asserting on a WebGL canvas is expensive and tells
you nothing. Instead: unit-test the pure camera-placement maths that turns a seat's
geometry into an eye position, and cover the rest with a Playwright screenshot.

---

## `goldseats-web` — Playwright

End-to-end, against a locally running web app and API with a seeded database. Few tests,
each covering a flow that matters.

```bash
npx playwright test
npx playwright test --ui
```

The suite, and it stays roughly this small:

1. **Browse and filter** — home page loads, filter by title, filter by language, filter by
   release date, results change correctly. ([M3](../product/milestones/M3.md))
2. **Film detail** — open a film, the aggregated timeline expands, nearby showtimes are
   grouped by format, the "best way to experience" panel ranks formats.
   ([M4](../product/milestones/M4.md))
3. **Seat recommendation to booking handoff** — the money path. Pick a showtime, request
   two seats, golden seats are highlighted with scores, choose "book online", assert we
   navigate to the theatre's domain. ([M5](../product/milestones/M5.md))
4. **Reserve in person** — same up to the fork, choose "reserve in person", assert the
   pick is saved and visible on reload.
5. **3D seat view** — select a seat, the canvas renders, screenshot comparison within
   tolerance. ([M6](../product/milestones/M6.md))
6. **Chatbot grounding** — "two seats, not too close, aisle preferred" returns seats that
   all exist in the layout and match the seats the scoring endpoint returned.
   ([M7](../product/milestones/M7.md))

```ts
// e2e/booking-handoff.spec.ts
test('recommends seats and hands off to the theatre booking page', async ({ page }) => {
  await page.goto('/films/dune-part-three');
  await page.getByRole('link', { name: /7:30 PM.*IMAX/ }).click();
  await page.getByLabel('Party size').fill('2');
  await page.getByRole('button', { name: 'Find our golden seats' }).click();

  const top = page.getByRole('button', { name: /recommended/ }).first();
  await expect(top).toBeVisible();
  await expect(top).toContainText(/\d{2}/); // the score, visible as text

  await top.click();
  await page.getByRole('button', { name: 'Book online' }).click();
  await expect(page).toHaveURL(/cineplex\.com/);
});
```

Rules:

- No `waitForTimeout`. Use web-first assertions and role-based locators.
- Seeded, deterministic data. Never depend on what's actually playing this week.
- If a Playwright test goes flaky, it gets fixed or deleted that day.

---

## Accessibility testing

Automated checks catch maybe a third of real problems, so both layers are required:

- `@axe-core/playwright` runs on the home page, film detail, and seat map in the E2E
  suite. Any violation fails the build.
- Manual keyboard-only and screen-reader passes on the seat map and the booking fork
  before each milestone closes.

See [`accessibility.md`](accessibility.md).

---

## Performance testing

- **Lighthouse CI** on every web PR against the budgets in
  [`performance-budgets.md`](performance-budgets.md). Regressions fail the build.
- **Load testing** with k6 against staging, a [M9](../product/milestones/M9.md)
  deliverable. Target: `POST /v1/recommendations` p95 under 300ms at 50 requests/second.
- **A scoring microbenchmark** in `pytest-benchmark` on the largest seeded auditorium.
  Scoring a full IMAX layout must stay under 50ms.

---

## CI

Both code repos run four required checks on every PR, and `main` is protected on all four:

| Check | `goldseats-api` | `goldseats-web` |
| --- | --- | --- |
| `lint` | `ruff check . && ruff format --check .` | `npm run lint` |
| `typecheck` | `mypy app` | `npm run typecheck` |
| `test` | `pytest --cov=app` | `npm run test` + `npx playwright test` |
| `build` | `docker build` / package build | `npm run build` |

The API test job runs twice: once on SQLite and once against a real Postgres service
container. That parity run is not optional — it's the only thing that catches a query that
works on SQLite and breaks in production. See
[`database-conventions.md`](database-conventions.md).
