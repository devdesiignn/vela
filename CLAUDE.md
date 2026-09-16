# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`vela` is a **docs-only hub repo** — it contains no application code. It holds the system-level plan and architecture diagram for a multi-repo platform (the "Receipt Intelligence Platform," in plain language); the actual services live in separate sibling repos (not present here).

There is nothing to build or test in this repo. Work here is limited to editing `README.md`, `WHAT-IS-THIS.md`, and `docs/MASTER-PLAN.md`, plus the markdown tooling described below.

## Naming

The project is named **Vela** (pronounced VEL-uh). On GitHub, every repo carries the `vela-` prefix: `vela-core`, `vela-etl`, `vela-api`, `vela-search`, `vela-agent`, `vela-forecast`, `vela-infra`. The hub repo is named `vela` (no suffix). Locally, folders drop the prefix since they already sit inside a `vela/` parent directory: `core`, `etl`, `api`, `search`, `agent`, `forecast`, `infra`, `docs`. "Receipt Intelligence Platform" is the plain-language description of what Vela is, not the project's name.

## Markdown tooling

This repo has a small Node toolchain scoped to linting/formatting markdown only:

- `npm install` — installs Prettier, markdownlint-cli, husky, lint-staged.
- `npm run format` / `npm run format:check` — Prettier over `*.md` and `docs/*.md`. Prettier owns table formatting (padded pipes) and general markdown style.
- `npm run lint` — markdownlint over the same files, for structural/style issues Prettier doesn't fix (e.g. missing code-fence language, heading structure). Config is `.markdownlint.json`; MD060 (table pipe style) is disabled there since it conflicts with Prettier's table output — don't re-enable it without also matching Prettier's style.
- A husky pre-commit hook runs `lint-staged` (markdownlint --fix + prettier --write on staged `.md` files), so contributors get the same checks locally before a commit lands.
- `node_modules/` is gitignored — never commit it.

## The platform this repo describes

One shared dataset (photographed paper receipts) feeds six independent, separately-repo'd services built around a common core data layer:

| Repo                 | Local folder | Role                                                                                                                                                          |
| -------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `vela-core`          | `core`       | Shared schema, migrations, data model — the contract every other service depends on                                                                           |
| `vela-etl` (P1)      | `etl`        | Photo → structured, validated, confidence-scored data; manual-review path for low-confidence extractions; writes via `vela-api`, not directly to the database |
| `vela-api` (P2)      | `api`        | REST API + dashboard for browsing/querying the data — the only service with a direct connection to `vela-core`                                                |
| `vela-search` (P3)   | `search`     | Semantic search over purchases                                                                                                                                |
| `vela-agent` (P4)    | `agent`      | Conversational Q&A; routes between structured queries (via `vela-api`) and semantic search (via `vela-search`) — does not touch the database directly         |
| `vela-forecast` (P5) | `forecast`   | Purchase forecasting and spend anomaly detection                                                                                                              |
| `vela-infra` (P6)    | `infra`      | Docker Compose, CI/CD, monitoring wrapping all of the above                                                                                                   |

Data flow: `vela-api` is the only service with a direct connection to `vela-core`. Every other service reaches the shared entities (`stores`, `receipts`, `line_items`, `extraction_reviews`) by calling `vela-api` — `vela-etl` calls it to write newly extracted data and resolve flagged reviews; `vela-search`, `vela-agent`, and `vela-forecast` call it to read. `vela-etl` is built first, against `vela-api`'s OpenAPI spec and a mock server standing in for it until `vela-api`'s real implementation exists (see `docs/MASTER-PLAN.md` Section 5).

## Design principles that shape any related work

- **v1 is printed receipts only.** POS screenshots and handwritten receipts are out of scope; `vela-etl` uses an extractor pattern so other source types can be added later without changing the shared schema.
- **Each of the six repos solves a distinct problem** (data engineering / API design / IR / agent routing / forecasting / infra) — if two end up doing similar work, one is scoped wrong.
- **Every repo documents what it does NOT do**, as explicitly as what it does.
- **The shared schema is a contract**: since P2–P6 all depend on what `vela-core` produces, changes to it are cross-cutting.
- **`vela-api` is the only writer to `vela-core`.** No other service, including `vela-etl`, connects to the database directly.
- **Privacy is data minimization, not redaction.** Source photos are the author's real purchase history, but the database is private and never published. `vela-core`'s schema defines exactly what gets persisted (required/common/rare tiers, rare fields in a structured `extras` field); `vela-etl` extracts only what the schema defines, so there's nothing extra to redact afterward. Public sample/seed data across repos is synthetic (Faker-generated), not redacted real data.
- **Accuracy is reported as a number**: `vela-etl` tracks percent auto-extracted vs. flagged for manual review; `vela-agent` is checked against a ~30-question eval set; `vela-forecast` uses a backtest against held-out recent months.

## Working in this repo specifically

- Keep `README.md`, `WHAT-IS-THIS.md`, and `docs/MASTER-PLAN.md` in sync — the README is the short public-facing summary; `WHAT-IS-THIS.md` is a quick orientation (name, what it is, why it exists); the master plan is the full write-up (principles, timeline, evaluation approach). Don't duplicate detailed content into the README beyond what's already summarized there.
- The architecture diagram appears in both `README.md` and `docs/MASTER-PLAN.md` (ASCII art) — update both if the shared entity list or repo topology changes.
- This repo is the hub every other service repo links back to: per the master plan (Section 4), each of the six repos gets its own README with a problem statement and architecture section plus a link back to `vela`. The shared schema and cross-repo architecture are meant to be documented once, here, and linked from every repo rather than re-explained in each one (Section 6).
- Do not add a `Co-Authored-By: Claude` trailer to commits in this repo.
