---
name: python-library-choices
description: Use when deciding which third-party library to reach for in a Python project — settings/structured data, web backends, HTTP clients, CLIs, databases, logging, retries, or testing. Gives the default pick per use case and flags the cases that still need a decision.
---

# Library choices

Defaults for common Python use cases. Use the default without asking
unless the project already has an established alternative in place — in
that case, follow the existing convention instead of introducing a second
library for the same job.

| Use case | Default |
|---|---|
| Settings & structured/validated data | `pydantic` + `pydantic-settings` |
| Web backend | `fastapi` |
| HTTP client | `httpx` |
| CLI | `typer` |
| Database / ORM | `sqlmodel` |
| Logging | `structlog` |
| Retries / resilience | `tenacity` |
| Testing | `pytest` + `pytest-asyncio` + `pytest-cov` |

## Notes per use case

- **Settings & structured data** — `pydantic` models for any structured
  data crossing a boundary (API payloads, parsed files, function configs);
  `pydantic-settings` for env-var/`.env`-driven app configuration.
- **Web backend** — `fastapi`, with `pydantic` models for request/response
  schemas. Use `uvicorn` as the ASGI server unless deployment constraints
  say otherwise.
- **HTTP client** — `httpx` for both sync and async call sites; don't mix
  in `requests` for the sync case.
- **CLI** — `typer` for anything beyond a couple of flags; plain
  `argparse` is acceptable only for a single trivial entry point.
- **Database / ORM** — `sqlmodel` on top of SQLAlchemy Core, with
  `alembic` for migrations.
- **Logging** — `structlog`, configured to emit JSON in production and
  human-readable output in development.
- **Retries / resilience** — `tenacity` decorators around flaky I/O
  (network calls, external APIs), with explicit backoff/jitter and a max
  attempt count — never an unbounded retry loop.
- **Testing** — `pytest` with `pytest-asyncio` for async test functions and
  `pytest-cov` for coverage reporting.

## Open decisions — ask before picking

These use cases don't have a settled default. Ask the user which option to
use the first time one comes up in a project, then record the answer in
that project's own docs (e.g. CLAUDE.md) so it doesn't need to be re-asked:

- **Async PostgreSQL driver** underneath SQLModel/SQLAlchemy — `psycopg`
  (v3, sync+async), `asyncpg` (async-only, fastest, different API shape),
  or `psycopg2` (sync-only, legacy but common).
- **Background jobs / scheduling** outside a request-response cycle —
  `APScheduler` (in-process, no extra infra), `arq` (async, Redis-backed,
  lighter than Celery), or `Celery` (full distributed task queue, for
  heavier multi-worker workloads).

When asking, name the trade-off in one line per option (sync vs async fit,
extra infra required, maturity) rather than just listing names.
