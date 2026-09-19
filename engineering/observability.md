# Observability

You can't operate what you can't see. Structured logs from day one
([M1](../product/milestones/M1.md)), traces and error tracking wired up properly at
[M9](../product/milestones/M9.md).

The standard we hold ourselves to: **when something breaks at 3am, the logs and dashboards
should be enough to find it without adding new instrumentation first.**

---

## The three signals

| Signal | Tool | In place by |
| --- | --- | --- |
| **Logs** — structured JSON, one line per event | `structlog` → stdout → host log aggregation | M1 |
| **Traces** — distributed spans across web, API, database, LLM | OpenTelemetry → OTLP collector | M9 |
| **Errors** — exceptions with stack traces and release tagging | Sentry, both repos | M9 |
| **Metrics** — RED and USE, plus product counters | OpenTelemetry metrics → dashboards | M9 |

---

## Correlation: `request_id`

Every request gets an ID, and that ID appears in the response, the logs, the trace, and
the Sentry event. It is the thread that ties a user's bug report to the line of code that
failed.

```python
# app/observability/middleware.py
@app.middleware("http")
async def request_context(request: Request, call_next):
    request_id = request.headers.get("X-Request-Id") or str(ulid.new())
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        method=request.method,
        path=request.url.path,
    )
    start = time.perf_counter()
    try:
        response = await call_next(request)
    finally:
        structlog.contextvars.clear_contextvars()
    response.headers["X-Request-Id"] = request_id
    log.info(
        "http.request_handled",
        status=response.status_code,
        duration_ms=round((time.perf_counter() - start) * 1000, 1),
    )
    return response
```

- The web app forwards an inbound `X-Request-Id` if present and generates one if not.
- Every error response body includes `request_id` — see
  [`api-design-guidelines.md`](api-design-guidelines.md).
- Error UI surfaces it quietly, so a user can paste it into a bug report.
- When the trace layer exists, `trace_id` is bound alongside it in exactly the same way.

---

## Logging

### Rules

1. **JSON to stdout.** The platform collects it. Never write log files, never rotate logs
   in-process.
2. **Never `print`.** Ruff will catch it.
3. **Event names, not sentences.** `noun.verb_past_tense`, stable over time, greppable.
   `recommendation.generated`, not `f"Generated {n} recommendations"`.
4. **Data goes in keyword fields**, never interpolated into the message. A field is
   queryable; a sentence isn't.
5. **One line per meaningful event.** Not one per loop iteration.

```python
log.info(
    "recommendation.generated",
    showtime_id=str(showtime.id),
    auditorium_id=str(auditorium.id),
    seat_layout_version=layout.version,
    party_size=request.party_size,
    scenario=scenario.value,
    candidate_blocks=len(candidates),
    top_score=round(options[0].score, 1),
    duration_ms=round(elapsed_ms, 1),
)
```

### Levels

| Level | Use | Example |
| --- | --- | --- |
| `DEBUG` | Local only. Never enabled in production. | Per-factor scoring intermediates |
| `INFO` | Something meaningful happened | `recommendation.generated`, `tmdb.ingest_completed` |
| `WARNING` | Degraded but handled | `tmdb.rate_limited`, `llm.fallback_to_seatmap` |
| `ERROR` | A request failed, or a job failed | `recommendation.failed`, unhandled 500 |
| `CRITICAL` | The service can't do its job | `db.unreachable` |

Production runs at `INFO`. `LOG_LEVEL` is an env var so it can be raised temporarily during
an incident.

### Never log

Not in a message, not in a field, not in an exception string:

- Passwords, password hashes, JWTs, refresh tokens, OAuth codes, API keys
- Email addresses — log `user_id` instead, always
- Precise user coordinates. Round to ~10km if you need to log location at all.
- Full chatbot message content by default. Log message *length*, token counts, tool calls,
  and latency. Conversation bodies live in the database, where the privacy policy governs
  them and the user can delete them.
- Full `layout_json` blobs. Log `auditorium_id` and `seat_layout_version`.

A scrubbing processor is installed as a backstop, but it's a net, not a plan:

```python
SENSITIVE_KEYS = frozenset({"password", "token", "authorization", "api_key", "email", "jwt_secret"})

def scrub(_, __, event_dict: dict) -> dict:
    for key in list(event_dict):
        if any(s in key.lower() for s in SENSITIVE_KEYS):
            event_dict[key] = "[redacted]"
    return event_dict
```

### Web-side logging

Server components and route handlers log with the same JSON shape. In the browser we log
almost nothing — errors go to Sentry, and `console.log` in a merged PR is a review miss.

---

## Tracing

OpenTelemetry, auto-instrumented for FastAPI, SQLAlchemy, `httpx`, and Redis, plus manual
spans on the parts of the system we actually care about.

```python
tracer = trace.get_tracer(__name__)

async def generate_recommendation(...) -> Recommendation:
    with tracer.start_as_current_span("recommendation.generate") as span:
        span.set_attribute("goldseats.party_size", party_size)
        span.set_attribute("goldseats.scenario", scenario.value)

        with tracer.start_as_current_span("seat_availability.fetch"):
            availability = await repo.latest_availability(session, showtime_id)

        with tracer.start_as_current_span("scoring.score_layout") as scoring_span:
            scores = await anyio.to_thread.run_sync(partial(score_layout, geometry, ...))
            scoring_span.set_attribute("goldseats.seats_scored", len(geometry))
            scoring_span.set_attribute("goldseats.blocks_found", len(scores))

        return persist(scores)
```

Spans we want on the critical path, because these are the questions we'll actually ask:

- `recommendation.generate` and inside it `seat_availability.fetch` and
  `scoring.score_layout` — is a slow recommendation slow because of the database or the
  maths?
- `chat.turn` → `llm.completion` → `tool.score_seats` → `tool.lookup_showtime` — where did
  a 9-second chatbot reply go?
- `tmdb.fetch_film` inside the ingestion job.

Attribute names are prefixed `goldseats.`. Never put a user ID, an email, or free-text user
input in a span attribute — traces are retained longer and are more widely readable than
you think.

Sampling: 100% in staging. In production, 10% of successful requests, 100% of errors, and
100% of `POST /v1/recommendations` and chat turns, because those are low-volume and
high-value.

---

## Error tracking

Sentry in both repos.

```python
sentry_sdk.init(
    dsn=settings.sentry_dsn,
    environment=settings.environment,      # local | staging | production
    release=settings.release_version,      # the git tag, e.g. v0.5.0
    traces_sample_rate=0.1,
    send_default_pii=False,                # never flip this on
    before_send=scrub_event,
)
```

- `release` is set to the git tag so a spike in errors can be attributed to a specific
  deploy. Without it, error tracking is half useless.
- `send_default_pii=False`, always. We attach `user_id` and nothing else — no email, no IP.
- Expected domain exceptions (`NotFoundError`, validation failures) are **not** sent. If
  404s are in your error tracker, you'll stop reading your error tracker.
- Web-side Sentry captures unhandled React errors and includes the `request_id` from the
  failing API call as a tag.

---

## Metrics and SLOs

### RED, per endpoint group

- **Rate** — requests per second
- **Errors** — proportion of 5xx
- **Duration** — p50, p95, p99

### Product metrics

These are the ones that tell us whether GoldSeats is working as a *product*, not just as a
server:

| Metric | Why we care |
| --- | --- |
| `recommendations_generated_total` by scenario | Are people using it solo or in groups? Drives scoring priorities. |
| `recommendation_empty_total` by reason | How often we can't seat a party contiguously. If this is high, the group logic needs work. |
| `booking_handoff_total` by `{online, in_person}` | The conversion event, and the split answers a real product question. |
| `scoring_duration_ms` histogram by auditorium size | Guards the 50ms scoring budget. |
| `chat_turns_total`, `chat_tool_calls_total` | Cost and usefulness of the LLM. |
| `chat_ungrounded_reply_blocked_total` | **Watch this one.** Any non-zero value means the model tried to invent a seat. |
| `tmdb_ingest_films_upserted_total` | Is the catalog actually being refreshed? |

### SLOs, from M9

| SLO | Target |
| --- | --- |
| API availability (non-5xx on public reads) | 99.5% monthly |
| `GET /v1/films/now-playing` p95 | < 200ms |
| `POST /v1/recommendations` p95 | < 300ms |
| Web LCP p75 on home page | < 2.5s |
| Chat first token p95 | < 2s |

Error budget: at 99.5%, that's about 3.6 hours a month. Burning more than half the budget
in a week means the next milestone's first issue is reliability work, not features.

---

## Dashboards

Three, and no more — a dashboard nobody opens is worse than none.

1. **Service health** — RED per endpoint group, error budget burn-down, database connection
   pool saturation, Redis hit rate, container CPU and memory.
2. **Product funnel** — home page views → film detail → showtime selected → recommendation
   generated → booking handoff, split online vs in-person. Plus empty-recommendation rate.
3. **Chatbot** — turns per day, tool calls per turn, first-token and total latency, blocked
   ungrounded replies, token spend.

---

## Alerts

Alert on symptoms users feel, not on causes. Every alert must be **actionable** and must
name its runbook. If an alert fires and the answer is "yeah, that happens", delete it.

| Alert | Condition | Severity | Runbook |
| --- | --- | --- | --- |
| API down | `/health` failing 2 min | Sev-1 | [api-rollback](../operations/runbooks/api-rollback.md) |
| Error rate | 5xx > 2% over 5 min | Sev-2 | [api-rollback](../operations/runbooks/api-rollback.md) |
| Recommendations failing | `recommendation.failed` > 5% over 10 min | Sev-2 | [api-rollback](../operations/runbooks/api-rollback.md) |
| Database unreachable | Connection failures 1 min | Sev-1 | [database-restore](../operations/runbooks/database-restore.md) |
| Latency | p95 > 1s over 10 min | Sev-3 | — |
| Ungrounded chat reply | `chat_ungrounded_reply_blocked_total` > 0 | Sev-3 | Investigate prompt and guardrail |
| TMDB ingest failed | Job failed 2 consecutive runs | Sev-3 | — |
| Certificate expiry | < 14 days | Sev-3 | [domain-cutover](../operations/runbooks/domain-cutover.md) |
| Error budget burn | > 50% of monthly budget consumed | Sev-3 | Reliability work next milestone |

Routing and escalation are in [`../operations/on-call.md`](../operations/on-call.md).
Severity definitions are in
[`../operations/incident-response.md`](../operations/incident-response.md).

---

## Health endpoints

```
GET /health        # liveness. Process is up. No dependency checks. Always fast.
GET /health/ready  # readiness. DB reachable, Redis reachable, migrations at head.
```

`/health` must never touch the database. If it does, a slow database turns a degradation
into a restart loop, and a restart loop turns a Sev-3 into a Sev-1.

```json
{
  "status": "ready",
  "version": "v0.5.0",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "migrations": "at_head"
  }
}
```

---

## Local development

`LOG_FORMAT=console` gives you `structlog`'s pretty renderer instead of JSON. Same events,
same fields, readable in a terminal.

```bash
LOG_LEVEL=DEBUG LOG_FORMAT=console uvicorn app.main:app --reload
```

Sentry is disabled locally (empty DSN). Tracing can be enabled against a local Jaeger if
you're chasing something specific, but it is not part of the default setup — see the
Docker constraint in [`database-conventions.md`](database-conventions.md).
