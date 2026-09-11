# Master Plan — Receipt Intelligence Platform

## 1. What this is

One dataset — receipt photos collected since September 2025 — feeds six independent services that share a single core data layer. One data platform, several applications built on top of it. Each repo is self-contained, but together they cover the full path from raw photo to answerable question: data engineering, backend/API design, search, a conversational agent, applied forecasting, and infrastructure.

## 2. Guiding principles

- **Each project solves a different problem.** P1 is data engineering, P2 is API/backend design, P3 is information retrieval, P4 is agent/routing logic, P5 is applied forecasting, P6 is infra/ops. If two projects end up doing the same kind of work, one of them is scoped wrong.
- **Scope things out deliberately, and say so.** Every project has a "what this does NOT do" section. Someone should be able to tell in 30 seconds what a project's boundary was on purpose, versus what just didn't get built. This applies to all six — e.g. P4's agent won't handle every possible query type, P5's model isn't chasing state-of-the-art accuracy, P6 won't cover every failure mode.
- **Build for real inputs, wherever "real input" applies to that project.** For P1 that's messy receipt photos. For P2 it's realistic API edge cases (pagination, bad filters). For P4 it's users asking ambiguous or out-of-scope questions. For P5 it's the fact that only ~13 months of data exist yet, which caps what forecasting can honestly claim. Each project names its own version of this.
- **Interfaces between projects are contracts.** P2–P6 all depend on the schema P1 produces, so it gets scrutinized before those projects are built on top of it.
- **Each repo stands alone.** Someone should be able to open any one of the six repos, without reading the other five, and understand what it does and why. This document is the connective tissue; it isn't required reading for any single repo to make sense.
- **Privacy is part of the design.** The source data is real personal spending history. Anonymization/redaction and synthetic sample data are part of `receipt-etl` from the start.
- **Report accuracy as a number.** `receipt-etl` tracks and surfaces its own extraction accuracy (percent auto-extracted cleanly vs. flagged for manual review). `receipt-agent` and `receipt-forecast` are checked against a small evaluation set / backtest — see Section 8.

## 3. Shared architecture

```text
                         ┌─────────────────────────┐
                         │   Core Data Service      │
                         │  (Postgres + migrations) │
                         │                          │
                         │  stores                  │
                         │  receipts                │
                         │  line_items              │
                         │  extraction_reviews       │
                         └────────────┬─────────────┘
                                      │
        ┌───────────────┬────────────┼────────────┬───────────────┐
        │               │            │             │               │
   ┌────▼────┐    ┌─────▼─────┐ ┌────▼─────┐ ┌─────▼──────┐  ┌─────▼─────┐
   │ P1: ETL │    │ P2: API + │ │ P3: Search│ │ P4: Agent  │  │ P5:       │
   │ Pipeline│    │ Dashboard │ │ & Retrieval│ │            │  │ Forecast  │
   └─────────┘    └───────────┘ └───────────┘ └────────────┘  └───────────┘

                         P6: Infra/Ops — wraps all of the above
                    (Docker, CI/CD, monitoring, async queue)
```

**Shared data entities:** `stores`, `receipts`, `line_items`, `extraction_reviews`. This is a starting point, not a final schema — it will evolve as Project 1 is built.

**Why one shared schema instead of six independent databases:** it keeps the "one data platform, many applications" structure real, and it forces the contract to be designed early, before three projects depend on it.

## 4. Repo structure

- `receipt-intelligence-platform` — docs-only hub: this plan, links to all other repos, system diagram.
- `receipt-core` — schema, migrations, seed/synthetic data, shared types (the extractor pattern lives here as an interface even though only one implementation exists in v1).
- `receipt-etl` (P1) — ingestion + extraction pipeline, including anonymization of source data. Depends on `receipt-core`.
- `receipt-api` (P2) — REST API + dashboard frontend, depends on `receipt-core`.
- `receipt-search` (P3) — embeddings + semantic search service.
- `receipt-agent` (P4) — conversational agent, calls into `receipt-api` and `receipt-search`.
- `receipt-forecast` (P5) — forecasting/anomaly detection, reads from `receipt-core`.
- `receipt-infra` (P6) — docker-compose/CI-CD/monitoring configs referencing all of the above.

Each repo gets its own README with a problem statement, architecture, and a link back to `receipt-intelligence-platform`, so a reader can land on any single repo and still get the bigger picture.

## 5. Timeline (Sept 7 – Dec 31, ~16 weeks)

| Weeks | Focus                                                                                          | Why this order                                                                                    |
| ----- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1–2   | `receipt-core` schema design (done) + P1 (ETL/extraction) build                                | Foundation — nothing else can start without structured data landing somewhere                     |
| 3–4   | P1 hardening: confidence scoring, manual-review UI, currency/date normalization, anonymization | Where the real-data messiness and privacy handling both get resolved                              |
| 5–7   | P2: API + Dashboard                                                                            | Standard full-stack layer; also becomes the "ground truth" UI used to sanity-check later projects |
| 8–9   | P3: Search & Retrieval                                                                         | Embeddings/vector search, self-contained, moderate lift                                           |
| 10–12 | P4: AI Agent                                                                                   | The most complex piece — benefits from P2 and P3 already existing to call into                    |
| 13–14 | P5: Forecasting/Anomaly Detection                                                              | Smaller, focused project, good pace change after the agent                                        |
| 15–16 | P6: Infra/Ops layer applied across all repos + writeups                                        | Dockerize, add CI/CD and monitoring retroactively across the set; polish READMEs                  |

Not rigid — P3/P4/P5 can reorder or run partially in parallel if one stalls. P1 and P2 are the only hard prerequisites.

## 6. Success criteria

- Each repo runs independently with its own README, setup instructions, and a short "Tradeoffs" section.
- The shared schema and cross-repo architecture is documented once (here) and linked from every repo, not re-explained six times.
- Each repo documents the concrete real-input challenge it faced (per its own version, as named in Section 2) and how it was handled.
- By December, all six repos are deployed or at minimum runnable via Docker Compose from `receipt-infra`.

## 7. The six projects

1. **ETL Pipeline** — image → validated, structured, confidence-scored, anonymized data.
2. **API + Dashboard** — backend + frontend on top of the data.
3. **Search & Retrieval** — semantic search over purchases.
4. **AI Agent** — natural-language Q&A over spending, routes between structured queries and semantic search.
5. **Forecasting/Anomaly Detection** — predict next purchase, flag spend anomalies.
6. **Infra/Ops** — containerization, CI/CD, monitoring across the whole set.

## 8. Evaluation

Two of the six projects get a concrete evaluation check:

- **P4 (Agent):** a fixed set of ~30 sample questions with expected answers (mix of structured lookups, semantic queries, and ambiguous/out-of-scope questions), run against the agent as a basic regression check whenever the routing logic changes.
- **P5 (Forecast):** a backtest methodology — holding out recent months and checking forecast/anomaly accuracy against them.

`receipt-etl`'s extraction accuracy (percent cleanly auto-extracted vs. flagged for manual review) is tracked the same way, as a reported number.
