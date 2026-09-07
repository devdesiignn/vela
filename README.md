# Receipt Intelligence Platform

**One dataset. Six engineering projects. One shared data layer.**

Since September 2025, I've been photographing shopping receipts from stores around Ilorin, Nigeria. This repo is the front door to a series of standalone projects built on that dataset — each one demonstrating a distinct engineering skill, from data pipelines to applied ML, all sharing a single underlying data platform.

Built by [Muiz Haruna](https://github.com/devdesiignn), September–December 2026 ("engineering season").

## Why this exists

This is structured the way real platforms are: one core data service, several applications built on top of it. Each repo below is independently reviewable, and together they share a single data layer instead of duplicating it six times.

## The Repos

| # | Repo | What it demonstrates |
|---|---|---|
| — | [`receipt-core`](https://github.com/devdesiignn/receipt-core) | Shared schema, migrations, and data-model ownership — the contract every other project depends on |
| 1 | [`receipt-etl`](https://github.com/devdesiignn/receipt-etl) | Image → structured data: extraction, validation, confidence scoring, manual-review path for messy real input |
| 2 | [`receipt-api`](https://github.com/devdesiignn/receipt-api) | REST API + dashboard on top of the core data |
| 3 | [`receipt-search`](https://github.com/devdesiignn/receipt-search) | Embeddings + semantic search over purchases |
| 4 | [`receipt-agent`](https://github.com/devdesiignn/receipt-agent) | Conversational AI agent that routes between structured and semantic queries |
| 5 | [`receipt-forecast`](https://github.com/devdesiignn/receipt-forecast) | Applied ML — purchase forecasting and spending anomaly detection |
| 6 | [`receipt-infra`](https://github.com/devdesiignn/receipt-infra) | Containerization, CI/CD, and monitoring across the whole series |

*(Repo links go live as each project is built — this table is the map, not a promise everything's already up.)*

## Architecture

```
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

## Scope, deliberately

- **v1 covers printed receipts only.** POS screenshots and handwritten market-list receipts are real source types in the data but are explicitly out of scope for v1 — noted as a v2 backlog item, not built. `receipt-etl`'s architecture uses an extractor pattern so new source types can be added later without changing the shared schema.
- Every project repo states what it does **not** do, as clearly as what it does. Deliberate scoping is treated as a feature of this series, not a shortcut.
- Each project repo documents the concrete real-input challenge it faced and how it handled it — not idealized demo data.

## Timeline

Sept 2026 – Dec 2026, roughly:

| Weeks | Focus |
|---|---|
| 1–4 | `receipt-core` schema + `receipt-etl` (extraction, confidence scoring, manual review) |
| 5–7 | `receipt-api` (backend + dashboard) |
| 8–9 | `receipt-search` (semantic search) |
| 10–12 | `receipt-agent` (AI agent) |
| 13–14 | `receipt-forecast` (ML) |
| 15–16 | `receipt-infra` + polish across all repos |

## Full write-up

The complete master plan — architecture decisions, tradeoffs, and reasoning behind the repo split — lives in [`docs/master-plan.md`](docs/master-plan.md) in this repo.
