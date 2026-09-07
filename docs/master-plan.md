# Master Plan — Receipt Intelligence Platform
**Engineering Season: September – December 2026**

## 1. The Narrative

One dataset — receipt photos collected since September 2025 — powers six standalone projects that together tell a single story: **one core data service, multiple applications built on top of it.** Each repo is independently reviewable, but together they demonstrate range across the full stack: data engineering, backend/API design, search, applied AI, applied ML, and infrastructure.

## 2. Guiding Principles

- **Each project proves a distinct engineering muscle, not a variation on the same one.** P1 is data engineering, P2 is API/backend design, P3 is information retrieval, P4 is agent orchestration, P5 is applied ML/statistics, P6 is infra/ops. If two projects end up demonstrating the same skill, one of them is scoped wrong.
- **Scope deliberately, state it explicitly.** Every PRD has a "what this does NOT do" section. A reviewer should be able to tell in 30 seconds what each project's ceiling was on purpose, versus what just didn't get built. This applies to all six equally — e.g. P4's agent won't handle every possible query type, P5's model won't chase state-of-the-art accuracy, P6 won't cover every possible failure mode.
- **Build for real inputs, not idealized ones — wherever "real input" applies to that project.** For P1 that's messy receipt photos. For P2 it's realistic API failure/edge cases (pagination, bad filters). For P4 it's users asking ambiguous or out-of-scope questions. For P5 it's the fact that only ~13 months of data exist yet, which caps what forecasting can honestly claim. Each PRD names its own version of this.
- **Interfaces between projects are contracts, not afterthoughts.** Since P2–P6 all depend on the schema P1 produces, changes to that schema after P2 is built are exactly the kind of cost real teams manage — so the schema is scrutinized early, rather than patched repeatedly later.
- **Each repo stands alone.** Someone should be able to open any one of the six repos, without reading the other five, and understand what it does and why it's a good showcase of that specific skill. The master plan is the connective tissue; it's not required reading for any single project to make sense.

## 3. Shared Architecture

```
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
   │ P1: ETL │    │ P2: API + │ │ P3: Search│ │ P4: AI     │  │ P5: ML    │
   │ Pipeline│    │ Dashboard │ │ & Retrieval│ │ Agent      │  │ Forecasting│
   └─────────┘    └───────────┘ └───────────┘ └────────────┘  └───────────┘

                         P6: Infra/Ops — wraps all of the above
                    (Docker, CI/CD, monitoring, async queue)
```

**Shared data entities.** The projects share one data layer, starting with four entities: `stores`, `receipts`, `line_items`, and `extraction_reviews`. This is a starting point, not a final schema — it will evolve as Project 1 is built.

**Why one shared schema instead of six independent databases:** it's the architectural story ("one data platform, many products") and it forces a stable contract to be designed early — a real skill, since changing a shared schema after three projects depend on it is exactly the kind of pain production engineers manage.

## 4. Repo Structure

- `receipt-intelligence-platform` — docs-only hub: this master plan, links to all seven other repos, system diagram. The front door for anyone reviewing the series.
- `receipt-core` — schema, migrations, seed data, shared types/interfaces (the extractor pattern lives here as an interface even though only one implementation exists in v1).
- `receipt-etl` (Project 1) — ingestion + extraction pipeline, depends on `receipt-core`
- `receipt-api` (Project 2) — REST API + dashboard frontend, depends on `receipt-core`
- `receipt-search` (Project 3) — embeddings + semantic search service
- `receipt-agent` (Project 4) — conversational agent, calls into `receipt-api` and `receipt-search`
- `receipt-forecast` (Project 5) — forecasting/anomaly detection, reads from `receipt-core`
- `receipt-infra` (Project 6) — docker-compose/CI-CD/monitoring configs referencing all of the above

Each project repo gets its own README with problem statement, architecture, and a link back to `receipt-intelligence-platform` — so a reviewer can land on any single repo and still get the bigger picture.

## 5. Timeline (Sept 7 – Dec 31, ~16 weeks)

| Weeks | Focus | Why this order |
|---|---|---|
| 1–2 | `receipt-core` schema design + Project 1 (ETL/extraction) build | Foundation — nothing else can start without structured data landing somewhere |
| 3–4 | Project 1 hardening: confidence scoring, manual-review UI, currency/date normalization | This is where the real-data messiness gets handled |
| 5–7 | Project 2: API + Dashboard | Standard full-stack layer, also becomes the "ground truth" UI used to sanity-check later projects |
| 8–9 | Project 3: Search & Retrieval | Embeddings/vector search, self-contained, moderate lift |
| 10–12 | Project 4: AI Agent | The most technically dense piece — benefits from P2 and P3 already existing to call into |
| 13–14 | Project 5: Forecasting/Anomaly Detection | Smaller, focused ML project — good pace change after the agent |
| 15–16 | Project 6: Infra/Ops layer applied across all repos + writeups | Dockerize, add CI/CD and monitoring retroactively across the set; polish READMEs |

This isn't rigid — Projects 3/4/5 can reorder or run partially in parallel if one stalls. Projects 1 and 2 are the only hard prerequisites.

## 6. Success Criteria

- Each repo runs independently with its own README, setup instructions, and a short "Tradeoffs" section
- The shared schema and cross-repo architecture is documented once (here) and linked from every repo, not re-explained six times
- Each project's README documents the concrete real-input challenge it faced (per its own version, as named in Section 2) and how it was handled
- By December, all six repos are deployed or at minimum runnable via Docker Compose from `receipt-infra`

## 7. The Six Projects

1. **ETL Pipeline** — image → validated, structured, confidence-scored data
2. **API + Dashboard** — proper backend + frontend on top of the data
3. **Search & Retrieval** — semantic search over purchases
4. **AI Agent** — natural-language Q&A over spending, routes between structured and semantic queries
5. **Forecasting/Anomaly Detection** — real ML: predict next purchase, flag spend anomalies
6. **Infra/Ops** — containerization, CI/CD, monitoring across the whole set
