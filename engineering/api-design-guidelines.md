# API Design Guidelines

`goldseats-api` is a JSON REST API served by FastAPI. Its only consumer today is
`goldseats-web` plus the chatbot's tool calls, but it's a public repo on a public domain,
so we design it as if third parties will read it — because they will.

---

## Versioning

Every route lives under `/v1/`. There is no unversioned route except `/health` and
`/openapi.json`.

```
https://api.goldseats.app/v1/films/now-playing
```

`/v1/` is the *contract* version and we intend to never cut a `/v2/`. The deployment
version (`v0.5.0`) is a separate thing entirely — see
[`branching-and-releases.md`](branching-and-releases.md#versioning).

**Non-breaking** (ship freely): adding an endpoint, adding an optional query parameter,
adding a field to a response, adding a new enum value to a field the client treats as
opaque.

**Breaking** (needs a `BREAKING CHANGE:` footer, a linked `goldseats-web` PR, and a
deprecation window): removing or renaming a field, changing a field's type, making an
optional parameter required, tightening validation, changing pagination semantics, adding
a new enum value to a field the client switches on exhaustively.

Deprecate rather than delete. Mark it in OpenAPI, return a `Deprecation` header, keep it
for at least one full milestone, then remove it in its own PR.

```python
@router.get("/films/{film_id}/showtimes", deprecated=True, summary="Deprecated: use /v1/showtimes?film_id=")
```

---

## URLs

- Plural nouns for collections: `/v1/films`, `/v1/theatres`, `/v1/showtimes`.
- `kebab-case` for multi-word path segments: `/v1/films/now-playing`.
- `snake_case` for query parameters and JSON field names, matching the database and
  Python. The web client's generated types handle the mapping; we don't maintain two
  naming schemes.
- Path parameters identify, query parameters filter.
- Nest only one level deep, and only where the child genuinely cannot exist without the
  parent:

```
GET  /v1/films                                    # list + filter
GET  /v1/films/{film_id}                          # one film
GET  /v1/films/now-playing                        # curated collection
GET  /v1/films/upcoming
GET  /v1/films/{film_id}/timeline                 # film_timeline_events, aggregated
GET  /v1/theatres                                 # ?near=43.6532,-79.3832&radius_km=25
GET  /v1/theatres/{theatre_id}/auditoriums
GET  /v1/auditoriums/{auditorium_id}/seat-layout  # NOT /theatres/{id}/auditoriums/{id}/seat-layout
GET  /v1/showtimes                                # ?film_id=&theatre_id=&date=&format=
GET  /v1/showtimes/{showtime_id}
GET  /v1/showtimes/{showtime_id}/seat-availability
POST /v1/recommendations                          # the core of the product
GET  /v1/recommendations/{recommendation_id}
POST /v1/chat/conversations
POST /v1/chat/conversations/{conversation_id}/messages
GET  /v1/me/feed
POST /v1/me/follows
DELETE /v1/me/follows/{follow_id}
GET  /v1/me/watchlist
```

Never put a verb in a URL. The exception that proves the rule: `POST /v1/recommendations`
is a POST to a collection that *creates a recommendation record*, not
`POST /v1/recommend-seats`. That's not pedantry — the recommendation is persisted in the
`recommendations` table so the chatbot, the 2D map, and the in-person reservation can all
refer to the same one by ID.

IDs are UUIDv7 everywhere. Never expose a sequential integer primary key.

---

## Methods and status codes

| Method | Use | Success |
| --- | --- | --- |
| `GET` | Read. Always safe, always idempotent, never mutates. | `200` |
| `POST` | Create, or a non-idempotent action | `201` with `Location`, or `200` |
| `PATCH` | Partial update | `200` |
| `PUT` | Full replacement — we use this only for `seat_layouts` versions | `200` |
| `DELETE` | Remove | `204`, no body |

| Status | When |
| --- | --- |
| `200` | Fine |
| `201` | Created. Include a `Location` header. |
| `204` | Deleted, or accepted with nothing to say |
| `400` | Malformed request we can't even parse into a schema |
| `401` | No credentials, or expired ones |
| `403` | Valid credentials, not your resource |
| `404` | Doesn't exist, or you're not allowed to know it exists |
| `409` | Conflict — e.g. following a film you already follow |
| `422` | Schema validation failed. FastAPI's default, and we keep it. |
| `429` | Rate limited. Include `Retry-After`. |
| `500` | Our bug. Never expose a stack trace. |
| `502` / `503` | An upstream we depend on (LLM, TMDB) is down |

An empty list is `200` with `"data": []`. It is never a `404`. `GET /v1/films/now-playing`
on a quiet Tuesday returning 404 would be a bug in us, not information for the client.

---

## Request and response shape

### Collections

Every list response is enveloped, so we can add pagination metadata without breaking
clients:

```json
{
  "data": [
    {
      "id": "0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f",
      "title": "Dune: Part Three",
      "original_language": "en",
      "runtime_minutes": 166,
      "poster_url": "https://image.tmdb.org/t/p/w500/abc123.jpg",
      "primary_release_date": "2026-11-20"
    }
  ],
  "meta": {
    "total": 42,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

### Single resources

Returned bare, no envelope. The resource is the response.

```json
{
  "id": "0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f",
  "title": "Dune: Part Three",
  "original_language": "en",
  "runtime_minutes": 166,
  "synopsis": "...",
  "releases": [
    { "region": "CA", "release_date": "2026-11-20", "release_type": "theatrical", "certification": "PG" }
  ],
  "created_at": "2026-02-11T14:03:22Z",
  "updated_at": "2026-04-02T09:15:00Z"
}
```

### Rules

- `snake_case` field names.
- Timestamps are RFC 3339, UTC, with a `Z` suffix, named `*_at`. Dates with no time are
  `YYYY-MM-DD` and named `*_date`.
- **Showtimes carry both.** `starts_at` in UTC for correctness, plus `starts_at_local` and
  `timezone` (IANA, e.g. `America/Toronto`) so the client never has to guess how to render
  "7:30 PM". This has already caused one real bug; the redundancy is deliberate.
- Money is an integer of minor units plus a currency code: `{"amount_cents": 1699, "currency": "CAD"}`.
  Never a float.
- Enums are lowercase `snake_case` strings, never integers: `"imax"`, `"small_group"`,
  `"in_person"`.
- Omit nothing. A field that has no value is `null`, not absent — absent fields make
  clients write defensive code.
- No field named `data`, `object`, `type`, or `value`. Say what it is.

### Embedding

Default to a lean response and let the client ask for more, rather than shipping a
1,200-line film object because one screen needs it.

```
GET /v1/films/{film_id}?embed=timeline,showtimes
```

`embed` accepts a comma-separated allowlist per endpoint, documented in OpenAPI. Anything
not on the list is a `422`. Never let `embed` trigger an unbounded query.

---

## The recommendations endpoint

The core of the product, so it's specified exactly here rather than left to the code.

```http
POST /v1/recommendations
Content-Type: application/json

{
  "showtime_id": "0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f",
  "party_size": 2,
  "preferences": {
    "aisle_preferred": true,
    "avoid_front_rows": true,
    "max_walk_distance_rows": null,
    "accessible_seating_required": false
  }
}
```

```json
{
  "id": "0192f3b4-1e2d-7c3f-8a9b-0c1d2e3f4a5b",
  "showtime_id": "0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f",
  "scenario": "pair",
  "generated_at": "2026-04-14T19:02:11Z",
  "seat_layout_version": 3,
  "options": [
    {
      "rank": 1,
      "score": 98.2,
      "seats": [
        { "row_label": "H", "seat_number": 12, "seat_type": "regular" },
        { "row_label": "H", "seat_number": 13, "seat_type": "regular" }
      ],
      "factors": {
        "horizontal_centre": 0.97,
        "vertical_position": 0.99,
        "view_angle": 0.96,
        "neighbour_openness": 1.0,
        "row_quality": 0.95
      },
      "explanation": "Dead centre horizontally, just past the midpoint of the auditorium, and nobody booked either side."
    }
  ],
  "booking": {
    "online_url": "https://www.cineplex.com/showtime/...",
    "supports_in_person_reservation": true
  }
}
```

Non-negotiables on this endpoint:

- **`factors` is always present.** The chatbot's explanations and the seat map's hover
  state are both built from it. A score with no breakdown is unusable.
- **`seat_layout_version` is always returned.** A recommendation is only meaningful
  against the layout version it was computed on.
- **Scenario is derived server-side** from `party_size`, never sent by the client. One
  place decides what `small_group` means.
- **Never cached.** `Cache-Control: no-store`. Availability changes by the minute.
- Party that can't be seated contiguously → `200` with `"options": []` and a
  `"reason": "no_contiguous_block"`, not an error. That's a real answer to a valid
  question.

---

## Errors

Every error body is a single shape, modelled on RFC 9457 problem details:

```json
{
  "type": "https://goldseats.app/problems/showtime-not-found",
  "title": "Not found",
  "status": 404,
  "detail": "Showtime '0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f' not found",
  "instance": "/v1/showtimes/0192f3a1-7c4e-7a1b-9f2e-1a2b3c4d5e6f",
  "request_id": "01JQ8Z4X9K2M3N4P5Q6R7S8T9V"
}
```

Validation errors keep FastAPI's per-field detail inside the same envelope:

```json
{
  "type": "https://goldseats.app/problems/validation-error",
  "title": "Validation error",
  "status": 422,
  "detail": "Request body failed validation",
  "errors": [
    { "field": "party_size", "message": "must be between 1 and 12" }
  ],
  "request_id": "01JQ8Z4X9K2M3N4P5Q6R7S8T9V"
}
```

- `request_id` is on **every** response, success or failure, and also in the
  `X-Request-Id` header. The web app surfaces it in error UI so a bug report can be tied
  to a log line. See [`observability.md`](observability.md).
- `detail` is safe to show a developer. Never leak a stack trace, a SQL fragment, an
  internal hostname, or another user's data into it.
- User-facing copy is the client's job. The API returns facts; `goldseats-web` writes the
  sentence the human reads.

---

## Pagination, filtering, sorting

Limit/offset. Not cursors — our largest collection is a few thousand films and cursors
would be complexity with no payoff.

```
GET /v1/films?limit=20&offset=40&sort=-popularity
```

- `limit` defaults to 20, maximum 100. Over the max is a `422`, not a silent clamp.
- `sort` takes a field name, `-` prefix for descending, from a per-endpoint allowlist.
  Never interpolate it into SQL.
- Filters are explicit named parameters, documented in OpenAPI. The home page filters from
  [M3](../product/milestones/M3.md):

```
GET /v1/films/now-playing?title=dune&language=fr&released_after=2026-01-01&released_before=2026-12-31
```

- `title` is a case-insensitive substring match.
- `language` is ISO 639-1, matched against `films.original_language` and the spoken/subtitle
  language metadata on `showtimes`.
- Every paginated query has a deterministic tiebreaker in its `order_by`, always ending in
  `id`. Without it, page 2 can repeat a row from page 1.

---

## Auth

- JWT bearer tokens. Short-lived access token (15 min), long-lived rotating refresh token.
- `Authorization: Bearer <token>`.
- Email/password with Argon2id, plus OAuth providers. [M1](../product/milestones/M1.md).
- Anything under `/v1/me/` requires auth and is scoped to the token's subject. There is no
  `user_id` path parameter anywhere in the API — you cannot ask for someone else's feed
  because there's no way to express the request.
- The catalog is public and unauthenticated: films, theatres, showtimes, seat layouts,
  recommendations. Recommendations are associated with a user when one is present so they
  can be saved, but they work anonymously.

---

## Caching

| Endpoint group | `Cache-Control` | Redis TTL |
| --- | --- | --- |
| `/v1/films/*` | `public, max-age=300, stale-while-revalidate=900` | 15 min |
| `/v1/theatres/*`, `/v1/auditoriums/*/seat-layout` | `public, max-age=3600` | 24 h |
| `/v1/showtimes*` | `public, max-age=120` | 5 min |
| `/v1/showtimes/*/seat-availability` | `no-store` | none |
| `/v1/recommendations` | `no-store` | none |
| `/v1/me/*` | `private, no-store` | none |

`ETag` on cacheable `GET`s; honour `If-None-Match` with a `304`. Keep these aligned with
the client-side `revalidate` values in
[`code-conventions-typescript.md`](code-conventions-typescript.md) — two caches with
different opinions is a debugging nightmare.

---

## Rate limiting

Redis token bucket, keyed on user ID when authenticated and IP when not.
[M9](../product/milestones/M9.md) deliverable.

| Group | Limit |
| --- | --- |
| Public catalog reads | 120 req/min |
| `POST /v1/recommendations` | 30 req/min |
| `POST /v1/chat/.../messages` | 10 req/min — the LLM is the expensive resource |
| Auth endpoints | 5 req/min per IP |

`429` responses include `Retry-After` and `X-RateLimit-Remaining`.

---

## Documentation

OpenAPI is generated, but generated isn't the same as good:

- Every route has a `summary` and a `description`.
- Every route declares `responses` for every status code it can return, with models.
- Every Pydantic field has a description and, where it helps, an `examples` entry.
- Swagger UI at `/docs`, ReDoc at `/redoc`, schema at `/openapi.json`.
- `goldseats-web` generates its types from that schema — see
  [`code-conventions-typescript.md`](code-conventions-typescript.md). If the schema is
  vague, the client is untyped, so schema quality is not optional.

```python
class RecommendationRequest(BaseModel):
    showtime_id: UUID = Field(description="Showtime to recommend seats for.")
    party_size: int = Field(ge=1, le=12, description="Number of seats needed, 1–12.")
    preferences: SeatPreferences = Field(
        default_factory=SeatPreferences,
        description="Optional soft constraints. These bias scoring; they never filter seats out entirely.",
    )
```

---

## Health

```
GET /health        # liveness: process is up. No dependency checks. Always fast.
GET /health/ready  # readiness: database reachable, Redis reachable, migrations at head
```

`/health` must not touch the database. A slow database making the liveness probe fail
turns a degradation into a restart loop.
