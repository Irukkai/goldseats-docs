# Performance Budgets

Budgets, not aspirations. Each number below is enforced somewhere — Lighthouse CI, a
pytest benchmark, a k6 threshold, or a dashboard alert — and a regression fails the build
rather than getting noticed three months later.

Lighthouse above 90 is an explicit done-when criterion for
[M3](../product/milestones/M3.md), and 60fps on the 3D view is one for
[M6](../product/milestones/M6.md).

---

## Why these numbers

Our users are frequently on a phone, on transit or standing outside a theatre, on a
Canadian mobile network, deciding in the next ninety seconds whether to buy a ticket. If
the seat map takes four seconds we've lost to the theatre's own app — which is already
installed on their phone. Speed here is not polish; it's the whole competitive position.

---

## Web: Core Web Vitals

Measured at the **75th percentile** on mobile, throttled to Slow 4G and a 4x-slowed CPU.
Desktop numbers are strictly easier and we don't set separate targets for them.

| Metric | Budget | Fails build above |
| --- | --- | --- |
| LCP — Largest Contentful Paint | < 2.0s | 2.5s |
| INP — Interaction to Next Paint | < 150ms | 200ms |
| CLS — Cumulative Layout Shift | < 0.05 | 0.1 |
| TTFB | < 600ms | 800ms |
| TBT — Total Blocking Time | < 200ms | 300ms |

| Lighthouse category | Minimum |
| --- | --- |
| Performance | 90 |
| Accessibility | 95 |
| Best Practices | 90 |
| SEO | 90 |

CLS is the one we're structurally exposed to: poster images from TMDB arrive at varying
sizes. Every `<Image>` gets explicit `width` and `height` or an aspect-ratio container.
A poster grid that jumps as images load is an automatic block in review.

---

## Web: bundle budgets

| Route | First-load JS (gzipped) | Hard cap |
| --- | --- | --- |
| `/` (marketing) | < 90 KB | 120 KB |
| `/` (home: now-playing + upcoming) | < 120 KB | 160 KB |
| `/films/[filmId]` | < 140 KB | 180 KB |
| `/showtimes/[id]/seats` (2D map) | < 170 KB | 220 KB |
| 3D seat view chunk (lazy) | < 400 KB | 550 KB |
| Chat panel chunk (lazy) | < 60 KB | 90 KB |

The rule that matters most: **three.js and react-three-fiber must never appear in any
initial bundle.** That's roughly 400 KB of JavaScript that a user who never opens the 3D
view should never download.

```tsx
// src/components/seats/SeatViewPanel.tsx
const AuditoriumScene = dynamic(
  () => import('@/components/seats-3d/AuditoriumScene').then((m) => m.AuditoriumScene),
  { ssr: false, loading: () => <SeatViewSkeleton /> },
);
```

Checked in CI:

```bash
npm run build && npx next-bundle-analyzer --ci   # fails on budget breach
```

If a dependency pushes a route over budget, the PR justifies it or finds another way. "It's
only 40 KB" repeated eight times is how a 90 KB page becomes a 400 KB page.

---

## Web: asset budgets

| Asset | Budget | Notes |
| --- | --- | --- |
| Poster images | < 60 KB each | `next/image`, AVIF with WebP fallback, `w500` from TMDB — never the original |
| Hero background | < 100 KB | The animated seat grid is CSS and SVG, not an image |
| Fonts | < 90 KB total | Inter + Playfair Display, `woff2`, subset to Latin + Latin-Ext, `font-display: swap`, self-hosted |
| Icons | < 15 KB | Inline SVG only. **No Font Awesome.** |
| Total page weight, home | < 900 KB | Including images at mobile sizes |

Two specific migrations from the landing page, both deliberate:

- The landing page loads Google Fonts from a CDN. In `goldseats-web` we self-host via
  `next/font`, which removes a third-party connection, eliminates the flash of unstyled
  text, and is better for privacy under
  [`../legal/privacy-policy.md`](../legal/privacy-policy.md).
- The landing page loads the full Font Awesome kit for six social icons. That becomes six
  inline SVGs — a ~75 KB saving for identical output.

---

## Web: caching

Explicit cache policy on every `fetch`. An unspecified one is a review comment.

| Data | Next.js | Why |
| --- | --- | --- |
| Film catalog, film detail | `revalidate: 900` + tag | Changes hourly at most |
| Theatre, auditorium, seat layout | `revalidate: 86400` | Layouts change roughly never |
| Showtimes | `revalidate: 300` | Changes a few times a day |
| Seat availability | `no-store` | **Never cache a seat map** |
| Recommendations | `no-store` | Computed against live availability |
| `/v1/me/*` | `no-store` | User-specific |

These must match the API's `Cache-Control` headers in
[`api-design-guidelines.md`](api-design-guidelines.md). Two caches with different opinions
about staleness is a debugging nightmare, and the failure mode — showing someone a seat
that's been sold — is the worst bug this product can have.

Static assets: immutable, one year, content-hashed filenames.

---

## API latency

Measured server-side, excluding network. Enforced by k6 thresholds against staging
([M9](../product/milestones/M9.md)) and alerted on in production.

| Endpoint | p50 | p95 | p99 |
| --- | --- | --- | --- |
| `GET /health` | < 5ms | < 20ms | < 50ms |
| `GET /v1/films/now-playing` | < 60ms | < 200ms | < 400ms |
| `GET /v1/films/{id}` | < 50ms | < 150ms | < 300ms |
| `GET /v1/films/{id}/timeline` | < 80ms | < 250ms | < 500ms |
| `GET /v1/showtimes` | < 60ms | < 200ms | < 400ms |
| `GET /v1/auditoriums/{id}/seat-layout` | < 40ms | < 120ms | < 250ms |
| `POST /v1/recommendations` | < 120ms | < 300ms | < 600ms |
| `POST /v1/chat/.../messages` first token | < 900ms | < 2s | < 4s |

Load target for [M9](../product/milestones/M9.md): sustain 50 requests/second across the
catalog endpoints and 10 requests/second on `/v1/recommendations` while holding p95.

### Database budgets

| Rule | Budget |
| --- | --- |
| Queries per request | ≤ 5. More than that needs a comment justifying it. |
| Slow query log threshold | 100ms |
| Any single query in a request path | < 50ms |
| Connection pool | 10 base + 5 overflow per instance |

**N+1 queries are the number one performance risk in this codebase.** A film list that
lazy-loads releases per film turns one query into forty-one and blows the whole budget.
Every serialised relationship gets `selectinload` or `joinedload`, and a test counts
queries on the list endpoints:

```python
async def test_now_playing_is_a_bounded_number_of_queries(client, query_counter) -> None:
    await seed_films(count=40, playing=True)
    with query_counter as counter:
        res = await client.get("/v1/films/now-playing?limit=20")
    assert res.status_code == 200
    assert counter.count <= 3   # films + releases + count
```

### Caching

Redis in front of the expensive reads. TTLs match the `Cache-Control` table in
[`api-design-guidelines.md`](api-design-guidelines.md). Targets:

| Cache | Hit rate target |
| --- | --- |
| Film catalog | > 90% |
| Seat layouts | > 98% |
| Showtimes | > 80% |

Redis being down must degrade, never fail. A cache miss path that 500s is worse than no
cache.

---

## Seat scoring engine

`app/ml/scoring.py` runs on every recommendation request and every chatbot tool call, so it
gets its own budget. Enforced by `pytest-benchmark` in CI.

| Auditorium | Seats | Budget |
| --- | --- | --- |
| Small | ~80 | < 5ms |
| Standard | ~200 | < 15ms |
| Large | ~350 | < 30ms |
| IMAX (largest seeded) | ~500 | < 50ms |

```python
@pytest.mark.benchmark(group="scoring")
def test_imax_layout_scores_within_budget(benchmark) -> None:
    layout = load_layout("cineplex-yonge-dundas-imax")
    result = benchmark(score_layout, layout.seats, scenario="pair", screen_width_m=22.0)
    assert benchmark.stats["mean"] < 0.050
    assert result
```

Constraints that keep it there:

- Pure and synchronous. No I/O, no database, no clock inside the engine.
- Vectorise with numpy where it's clearly faster; keep it readable where it isn't. The
  scoring logic is the product's core IP and a clever unreadable version is a bad trade.
- Called via `anyio.to_thread.run_sync` so it never stalls the event loop.
- Score the same `(layout_version, availability_snapshot, scenario)` once and cache it in
  Redis for 60 seconds — during a busy evening, many users ask about the same showtime
  within the same minute.

---

## 3D seat view

[M6](../product/milestones/M6.md). "60fps on desktop, degrades gracefully on mobile" is the
done-when criterion, so it needs a definition.

| Target | Desktop (mid-range, integrated GPU) | Mobile (mid-range Android) |
| --- | --- | --- |
| Frame rate | 60fps sustained | ≥ 30fps sustained |
| Time to first frame | < 1.5s | < 3s |
| Draw calls | < 150 | < 80 |
| Triangles | < 250k | < 100k |
| Texture memory | < 64 MB | < 24 MB |
| Lazy chunk size | < 400 KB gzipped | same |

How we hold it:

- **Instanced meshes for seats.** 500 seats is one draw call, not 500. Non-negotiable.
- Static scene baked once per `(auditorium, seat_layout_version)` and reused across seat
  selections. Changing seat moves the camera; it does not rebuild the geometry.
- No real-time shadows. Baked ambient occlusion.
- Capability check on mount → quality tier:

  | Tier | Applied when | Changes |
  | --- | --- | --- |
  | `high` | Desktop, `deviceMemory` ≥ 8, no reduced-motion | Full geometry, AO, antialiasing, DPR up to 2 |
  | `medium` | Mid mobile, `deviceMemory` 4–8 | Simplified seat geometry, no AO, DPR 1.5 |
  | `low` | `deviceMemory` < 4, or measured < 24fps over 2s | Boxes for seats, DPR 1, static camera |
  | `none` | No WebGL, or `prefers-reduced-motion` | Static image + the text panel |

- `frameloop="demand"` — render on interaction, not continuously. A static preview
  redrawing 60 times a second for no reason is pure battery drain.
- Dispose geometries, materials, and textures on unmount. A leaked WebGL context after
  three seat selections will crash a phone.

The `none` tier is a first-class path, not a failure state — see
[`accessibility.md`](accessibility.md). The text panel (distance, viewing angle, comparison
to the THX 36-degree target) is arguably more useful than the render anyway.

---

## Chatbot

[M7](../product/milestones/M7.md). Self-hosted, so latency is our problem and our cost.

| Metric | Budget |
| --- | --- |
| Time to first token | p95 < 2s |
| Tokens per second | > 25 |
| Full turn, including tool calls | p95 < 6s |
| Tool calls per turn | ≤ 3 |
| Prompt tokens per turn | < 2,000 |

- Stream tokens. A 6-second wait with visible output beats a 3-second wait with a spinner.
- Tool calls run in parallel when independent — layout lookup and showtime lookup don't
  need to be sequential.
- Keep the system prompt tight. Seat data goes in via tool results, never pasted wholesale
  into the prompt; a full layout in the context window is both slow and expensive.
- Trim conversation history to the last 10 turns plus a summary.
- If the model is unavailable or exceeds `LLM_TIMEOUT_SECONDS`, fall back to the
  deterministic seat map immediately. A slow chatbot must never block someone from getting
  a seat.

---

## Browser support

| Browser | Support |
| --- | --- |
| Chrome / Edge | Last 2 versions |
| Safari (macOS and iOS) | Last 2 versions |
| Firefox | Last 2 versions |
| Samsung Internet | Last 2 versions |
| iOS Safari 15 | Degraded but functional |
| IE 11 | Not supported |

Roughly `>0.5%, last 2 versions, not dead`. We don't ship polyfills for browsers outside
that, and we don't break on them either: the 2D seat map works without WebGL, and the
filters work without JavaScript.

---

## Enforcement

| Budget | Enforced by | On breach |
| --- | --- | --- |
| Core Web Vitals, Lighthouse | Lighthouse CI on every web PR | Build fails |
| Bundle size | `next-bundle-analyzer --ci` | Build fails |
| Scoring latency | `pytest-benchmark` | Build fails |
| API query count | Query-counting tests | Build fails |
| API latency | k6 thresholds against staging | Release blocked |
| Production latency and frame rate | Dashboards and alerts | Sev-3 |

See [`observability.md`](observability.md) for the production side and
[`testing-strategy.md`](testing-strategy.md) for the test side.

---

## When a budget is wrong

Budgets are guesses informed by evidence, and sometimes the evidence changes. Raising one is
allowed, in its own PR, with:

1. The measurement showing the current number
2. Why the work causing the regression is worth it
3. What you tried first
4. A dated note here explaining the change

What's not allowed is quietly bumping the threshold in a config file inside a feature PR.
That's how budgets stop meaning anything.
