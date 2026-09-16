# Vela

Since September 2025 I've been photographing paper receipts from shopping trips around Ilorin, Nigeria. Vela is a small platform that turns that growing pile of photos into a structured, queryable record of what I've bought, where, and when — built as a set of focused services that share one core data layer.

See [WHAT-IS-THIS.md](WHAT-IS-THIS.md) for a quick orientation, or [docs/MASTER-PLAN.md](docs/MASTER-PLAN.md) for the full write-up.

## The repos

| #   | Repo                                                            | What it does                                                                                                                                                   |
| --- | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| —   | [`vela-core`](https://github.com/devdesiignn/vela-core)         | Shared schema, migrations, and data model — the contract every other service depends on                                                                        |
| 1   | [`vela-etl`](https://github.com/devdesiignn/vela-etl)           | Turns receipt photos into structured, validated data: extraction, confidence scoring, and a manual-review path for anything the pipeline isn't confident about |
| 2   | [`vela-api`](https://github.com/devdesiignn/vela-api)           | REST API and dashboard for browsing and querying the data — the only service with a direct connection to `vela-core`                                           |
| 3   | [`vela-search`](https://github.com/devdesiignn/vela-search)     | Semantic search over purchases — find things by meaning, not just exact wording                                                                                |
| 4   | [`vela-agent`](https://github.com/devdesiignn/vela-agent)       | Conversational interface for asking questions about spending; routes between structured queries and semantic search                                            |
| 5   | [`vela-forecast`](https://github.com/devdesiignn/vela-forecast) | Purchase forecasting and spending anomaly detection                                                                                                            |
| 6   | [`vela-infra`](https://github.com/devdesiignn/vela-infra)       | Containerization, CI/CD, and monitoring for the whole set                                                                                                      |

_**(Repo links go live as each project is built — this table is the map, not a promise everything's already up.)**_

## Architecture

```text
                         ┌─────────────────────────┐
                         │        vela-core        │
                         │  (shared data entities)  │
                         │                          │
                         │  stores · receipts       │
                         │  line_items              │
                         │  extraction_reviews      │
                         └────────────┬─────────────┘
                                      │
                         ┌────────────▼─────────────┐
                         │         vela-api          │
                         │  (write contract + reads) │
                         └────────────┬─────────────┘
                                      │
                 ┌───────────────┬────┴───────┬───────────────┐
                 │               │            │               │
            ┌────▼────┐    ┌─────▼─────┐ ┌────▼─────┐  ┌─────▼─────┐
            │  vela-  │    │   vela-   │ │  vela-   │  │  vela-    │
            │   etl   │    │  search   │ │  agent   │  │  forecast │
            └─────────┘    └───────────┘ └──────────┘  └───────────┘

                       vela-infra — wraps all of the above
              (Docker, CI/CD, monitoring, async queue)
```

`vela-api` is the only service with a direct connection to `vela-core`. Everything else reaches the data by calling `vela-api`: `vela-etl` calls it to write newly extracted receipts and to resolve flagged reviews, `vela-search`, `vela-agent`, and `vela-forecast` call it to read. `vela-agent` also calls into `vela-search` directly for semantic queries.

`vela-etl` is built first (against `vela-api`'s OpenAPI spec and a mock server standing in for it), with `vela-api`'s real implementation following — full reasoning in `docs/MASTER-PLAN.md`.

## Scope, on purpose

- **v1 covers printed receipts only.** POS screenshots and handwritten market-list receipts exist in the source photos but are explicitly out of scope for v1. `vela-etl` uses an extractor pattern so new source types can be added later without changing the shared schema.
- Every repo states what it does **not** do, as clearly as what it does.
- Each repo documents the concrete messy-input problem it ran into and how it was handled.

## Data and privacy

The database is private and not shared or published. The photos it's built from contain real purchase history and store details, but the stored data is limited by schema design:

- `vela-core`'s schema defines required, common, and rare fields based on analysis of real sample receipts. `vela-etl` extracts only what the schema defines — nothing else is captured or retained. Rare, store-specific fields go into a structured `extras` field rather than open-ended storage.
- Raw source photos are referenced by ID from the database but stored separately, outside `vela-core`.
- Sample/seed data in each public repo is synthetic — generated with Faker to match the schema's shape and statistical patterns, not derived from or redacted from real receipts.

## Timeline

Sept 2026 – Dec 2026, roughly:

| Weeks | Focus                                                                                  |
| ----- | -------------------------------------------------------------------------------------- |
| 1–4   | `vela-core` schema (done) + `vela-etl` (extraction, confidence scoring, manual review) |
| 5–7   | `vela-api` (backend + dashboard)                                                       |
| 8–9   | `vela-search` (semantic search)                                                        |
| 10–12 | `vela-agent` (conversational agent)                                                    |
| 13–14 | `vela-forecast` (forecasting/anomaly detection)                                        |
| 15–16 | `vela-infra` + polish across all repos                                                 |

## Full write-up

The complete plan — architecture decisions, tradeoffs, and reasoning behind the repo split — lives in [`docs/MASTER-PLAN.md`](docs/MASTER-PLAN.md) in this repo.
