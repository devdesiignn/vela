# Receipt Intelligence Platform

Since September 2025 I've been photographing paper receipts from shopping trips around Ilorin, Nigeria. This is a small platform that turns that growing pile of photos into a structured, queryable record of what I've bought, where, and when — built as a set of focused services that share one core data layer.

## The repos

| #   | Repo                                                                  | What it does                                                                                                                                                   |
| --- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| —   | [`receipt-core`](https://github.com/devdesiignn/receipt-core)         | Shared schema, migrations, and data model — the contract every other service depends on                                                                        |
| 1   | [`receipt-etl`](https://github.com/devdesiignn/receipt-etl)           | Turns receipt photos into structured, validated data: extraction, confidence scoring, and a manual-review path for anything the pipeline isn't confident about |
| 2   | [`receipt-api`](https://github.com/devdesiignn/receipt-api)           | REST API and dashboard for browsing and querying the data                                                                                                      |
| 3   | [`receipt-search`](https://github.com/devdesiignn/receipt-search)     | Semantic search over purchases — find things by meaning, not just exact wording                                                                                |
| 4   | [`receipt-agent`](https://github.com/devdesiignn/receipt-agent)       | Conversational interface for asking questions about spending; routes between structured queries and semantic search                                            |
| 5   | [`receipt-forecast`](https://github.com/devdesiignn/receipt-forecast) | Purchase forecasting and spending anomaly detection                                                                                                            |
| 6   | [`receipt-infra`](https://github.com/devdesiignn/receipt-infra)       | Containerization, CI/CD, and monitoring for the whole set                                                                                                      |

_**(Repo links go live as each project is built — this table is the map, not a promise everything's already up.)**_

## Architecture

```text
                         ┌─────────────────────────┐
                         │      receipt-core       │
                         │  (shared data entities)  │
                         │                          │
                         │  stores · receipts       │
                         │  line_items              │
                         │  extraction_reviews      │
                         └────────────┬─────────────┘
                                      │
        ┌───────────────┬────────────┼────────────┬───────────────┐
        │               │            │             │               │
   ┌────▼────┐    ┌─────▼─────┐ ┌────▼─────┐ ┌─────▼──────┐  ┌─────▼─────┐
   │ receipt-│    │ receipt-  │ │ receipt- │ │ receipt-   │  │ receipt-  │
   │  etl    │    │  api      │ │  search  │ │  agent     │  │  forecast │
   └─────────┘    └───────────┘ └──────────┘ └────────────┘  └───────────┘

                    receipt-infra — wraps all of the above
              (Docker, CI/CD, monitoring, async queue)
```

`receipt-etl` writes into the shared data entities; `receipt-api`, `receipt-search`, `receipt-agent`, and `receipt-forecast` all read from them (directly or via `receipt-api`). `receipt-agent` also calls into `receipt-api` and `receipt-search` rather than touching the database directly.

## Scope, on purpose

- **v1 covers printed receipts only.** POS screenshots and handwritten market-list receipts exist in the source photos but are explicitly out of scope for v1. `receipt-etl` uses an extractor pattern so new source types can be added later without changing the shared schema.
- Every repo states what it does **not** do, as clearly as what it does.
- Each repo documents the concrete messy-input problem it ran into and how it was handled.

## Data and privacy

The database is private and not shared or published. The photos it's built from contain real purchase history and store details, but the stored data is limited by schema design:

- `receipt-core`'s schema defines required, common, and rare fields based on analysis of real sample receipts. `receipt-etl` extracts only what the schema defines — nothing else is captured or retained. Rare, store-specific fields go into a structured `extras` field rather than open-ended storage.
- Raw source photos are referenced by ID from the database but stored separately, outside `receipt-core`.
- Sample/seed data in each public repo is synthetic — generated with Faker to match the schema's shape and statistical patterns, not derived from or redacted from real receipts.

## Timeline

Sept 2026 – Dec 2026, roughly:

| Weeks | Focus                                                                                        |
| ----- | -------------------------------------------------------------------------------------------- |
| 1–4   | `receipt-core` schema (done) + `receipt-etl` (extraction, confidence scoring, manual review) |
| 5–7   | `receipt-api` (backend + dashboard)                                                          |
| 8–9   | `receipt-search` (semantic search)                                                           |
| 10–12 | `receipt-agent` (conversational agent)                                                       |
| 13–14 | `receipt-forecast` (forecasting/anomaly detection)                                           |
| 15–16 | `receipt-infra` + polish across all repos                                                    |

## Full write-up

The complete plan — architecture decisions, tradeoffs, and reasoning behind the repo split — lives in [`docs/master-plan.md`](docs/master-plan.md) in this repo.
