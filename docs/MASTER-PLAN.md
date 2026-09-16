# Master Plan

## 1. What this is

One dataset — receipt photos collected since September 2025 — feeds six independent services that share a single core data layer. One data platform, several applications built on top of it. Each repo is self-contained, but together they cover the full path from raw photo to answerable question: data engineering, backend/API design, search, a conversational agent, applied forecasting, and infrastructure.

## 2. Guiding principles

- **Each project solves a different problem.** P1 is data engineering, P2 is API/backend design, P3 is information retrieval, P4 is agent/routing logic, P5 is applied forecasting, P6 is infra/ops. If two projects end up doing the same kind of work, one of them is scoped wrong.
- **Scope things out deliberately, and say so.** Every project has a "what this does NOT do" section. Someone should be able to tell in 30 seconds what a project's boundary was on purpose, versus what just didn't get built. This applies to all six — e.g. P4's agent won't handle every possible query type, P5's model isn't chasing state-of-the-art accuracy, P6 won't cover every failure mode.
- **Build for real inputs, wherever "real input" applies to that project.** For P1 that's messy receipt photos. For P2 it's realistic API edge cases (pagination, bad filters). For P4 it's users asking ambiguous or out-of-scope questions. For P5 it's the fact that only ~13 months of data exist yet, which caps what forecasting can honestly claim. Each project names its own version of this.
- **`vela-api` is the only writer to `vela-core`.** No other service — including `vela-etl` — connects to the database directly. Every write (new extracted receipts, review resolutions) and every read goes through `vela-api`'s contract. `vela-api` is the only service with a direct connection to `vela-core`; every other repo reaches the data by calling `vela-api`, never the database directly. (Open question on read access for batch/bulk workloads — see Section 5a.)
- **Interfaces between projects are contracts.** `vela-api`'s OpenAPI spec is the contract every other project is built against, the same way `vela-core`'s schema is the contract `vela-api` is built against. This is what lets the build order stay `vela-etl` first: `vela-etl` is written once against the OpenAPI spec and points at a mock server implementing that spec until `vela-api`'s real implementation exists, then switches to the real thing with no code change on `vela-etl`'s side.
- **Each repo stands alone.** Someone should be able to open any one of the six repos, without reading the other five, and understand what it does and why. This document is the connective tissue; it isn't required reading for any single repo to make sense.
- **Privacy is data minimization, not redaction.** The source data is real personal spending history, but the database is private and never published. `vela-core`'s schema defines exactly what gets persisted (required/common/rare tiers, with rare fields in a structured `extras` field); `vela-etl` extracts only what the schema defines, so there's nothing extra to anonymize afterward. Public repos get synthetic (Faker-generated) sample data, not redacted real data.
- **Report accuracy as a number.** `vela-etl` tracks and surfaces its own extraction accuracy (percent auto-extracted cleanly vs. flagged for manual review). `vela-agent` and `vela-forecast` are checked against a small evaluation set / backtest — see Section 8.

## 3. Shared architecture

```text
                         ┌─────────────────────────┐
                         │        vela-core         │
                         │  (Postgres + migrations)  │
                         │                          │
                         │  stores                  │
                         │  receipts                │
                         │  line_items              │
                         │  extraction_reviews       │
                         └────────────┬─────────────┘
                                      │  (only writer/reader)
                         ┌────────────▼─────────────┐
                         │      P2: vela-api         │
                         │ (write contract + reads)  │
                         └────────────┬─────────────┘
                                      │
                 ┌───────────────┬────┴───────┬───────────────┐
                 │               │            │               │
            ┌────▼────┐    ┌─────▼─────┐ ┌────▼─────┐  ┌─────▼─────┐
            │ P1: ETL │    │ P3: Search│ │ P4: Agent │  │ P5:       │
            │Pipeline │    │& Retrieval│ │           │  │ Forecast  │
            └─────────┘    └───────────┘ └───────────┘  └───────────┘

                         P6: Infra/Ops — wraps all of the above
                    (Docker, CI/CD, monitoring, async queue)
```

**Shared data entities:** `stores`, `receipts`, `line_items`, `extraction_reviews`, finalized in `vela-core`.

**Why `vela-api` sits between everything and `vela-core`:** a single write path keeps the "who's allowed to change the data" question answered once, in one place, instead of every service needing its own write logic and its own opinion about validation. It also means `vela-etl`'s only real job is extraction — it hands structured data to `vela-api` and is done, rather than also owning a piece of the data-access layer.

## 4. Repo structure

- `vela` — docs-only hub: this plan, links to all other repos, system diagram.
- `vela-core` — schema, migrations, seed/synthetic data, shared types. Not called directly by anything except `vela-api`.
- `vela-etl` (P1) — ingestion + extraction pipeline. Built against `vela-api`'s OpenAPI spec (calling a mock server implementing that spec until `vela-api` is real, then the real service — no code change either way); does not connect to `vela-core` directly.
- `vela-api` (P2) — REST API + dashboard frontend. The only service with a direct connection to `vela-core`. Owns all writes (new receipts/line_items from `vela-etl`, review resolutions) and all reads for every other service.
- `vela-search` (P3) — embeddings + semantic search service. Read access pattern (via `vela-api` vs. direct to `vela-core` for bulk indexing) is an open question — see Section 5a.
- `vela-agent` (P4) — conversational agent, calls into `vela-api` and `vela-search`.
- `vela-forecast` (P5) — forecasting/anomaly detection, reads via `vela-api`.
- `vela-infra` (P6) — docker-compose/CI-CD/monitoring configs referencing all of the above.

Each repo gets its own README with a problem statement, architecture, and a link back to `vela`, so a reader can land on any single repo and still get the bigger picture.

## 5. Timeline (Sept 7 – Dec 31, ~16 weeks)

| Weeks | Focus                                                                                                | Why this order                                                                                                                     |
| ----- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 1–4   | `vela-core` schema (done) + P1 (`vela-etl`, built against `vela-api`'s OpenAPI spec + a mock server) | Foundation — nothing else can start without structured data landing somewhere, and the mock lets this happen without waiting on P2 |
| 5–7   | P2: `vela-api` (real implementation, matching the spec `vela-etl` already built against)             | `vela-etl` switches from the mock to the real service here, with no changes to its own code                                        |
| 8–9   | P3: Search & Retrieval                                                                               | Embeddings/vector search — read-access pattern still open, see Section 5a                                                          |
| 10–12 | P4: AI Agent                                                                                         | The most complex piece — benefits from P2 and P3 already existing to call into                                                     |
| 13–14 | P5: Forecasting/Anomaly Detection                                                                    | Smaller, focused project, good pace change after the agent                                                                         |
| 15–16 | P6: Infra/Ops layer applied across all repos + writeups                                              | Dockerize, add CI/CD and monitoring retroactively across the set; polish READMEs                                                   |

Not rigid — P3/P4/P5 can reorder or run partially in parallel if one stalls. `vela-core`, then `vela-etl` (against the OpenAPI mock), then `vela-api` for real, are the hard prerequisites, in that order — same order as before the write-path rule was added.

## 5a. Open questions

- **`vela-search`'s bulk read path.** Building a semantic-search index means reading every receipt/line_item, likely repeatedly as the dataset grows. Going through `vela-api` for this means re-fetching everything via REST calls, which is the normal pattern for request-scoped reads but can get slow/expensive for a batch indexing job. The alternative is `vela-search` reading `vela-core` directly for bulk/batch access, as an explicit exception to the "api is the only reader/writer" rule. Unresolved — revisit when `vela-search`'s build phase starts (weeks 8–9).

## 6. Success criteria

- Each repo runs independently with its own README, setup instructions, and a short "Tradeoffs" section.
- The shared schema and cross-repo architecture is documented once (here) and linked from every repo, not re-explained six times.
- Each repo documents the concrete real-input challenge it faced (per its own version, as named in Section 2) and how it was handled.
- By December, all six repos are deployed or at minimum runnable via Docker Compose from `vela-infra`.

## 7. The six projects

1. **ETL Pipeline** — image → validated, structured, confidence-scored data, written via `vela-api`.
2. **API + Dashboard** — the write/read contract for `vela-core`, plus a frontend on top of the data.
3. **Search & Retrieval** — semantic search over purchases.
4. **AI Agent** — natural-language Q&A over spending, routes between structured queries and semantic search.
5. **Forecasting/Anomaly Detection** — predict next purchase, flag spend anomalies.
6. **Infra/Ops** — containerization, CI/CD, monitoring across the whole set.

## 8. Evaluation

Two of the six projects get a concrete evaluation check:

- **P4 (Agent):** a fixed set of ~30 sample questions with expected answers (mix of structured lookups, semantic queries, and ambiguous/out-of-scope questions), run against the agent as a basic regression check whenever the routing logic changes.
- **P5 (Forecast):** a backtest methodology — holding out recent months and checking forecast/anomaly accuracy against them.

`vela-etl`'s extraction accuracy (percent cleanly auto-extracted vs. flagged for manual review) is tracked the same way, as a reported number.
