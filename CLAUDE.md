# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`receipt-intelligence-platform` is a **docs-only hub repo** — it contains no application code. It holds the system-level plan and architecture diagram for a multi-repo platform; the actual services live in separate sibling repos (not present here).

There is nothing to build or test in this repo. Work here is limited to editing `README.md` and `docs/master-plan.md`, plus the markdown tooling described below.

## Markdown tooling

This repo has a small Node toolchain scoped to linting/formatting markdown only:

- `npm install` — installs Prettier, markdownlint-cli, husky, lint-staged.
- `npm run format` / `npm run format:check` — Prettier over `*.md` and `docs/*.md`. Prettier owns table formatting (padded pipes) and general markdown style.
- `npm run lint` — markdownlint over the same files, for structural/style issues Prettier doesn't fix (e.g. missing code-fence language, heading structure). Config is `.markdownlint.json`; MD060 (table pipe style) is disabled there since it conflicts with Prettier's table output — don't re-enable it without also matching Prettier's style.
- A husky pre-commit hook runs `lint-staged` (markdownlint --fix + prettier --write on staged `.md` files), so contributors get the same checks locally before a commit lands.
- `node_modules/` is gitignored — never commit it.

## The platform this repo describes

One shared dataset (photographed paper receipts) feeds six independent, separately-repo'd services built around a common core data layer:

| Repo                    | Role                                                                                                                                                        |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `receipt-core`          | Shared schema, migrations, data model — the contract every other service depends on                                                                         |
| `receipt-etl` (P1)      | Photo → structured, validated, confidence-scored, anonymized data; manual-review path for low-confidence extractions                                        |
| `receipt-api` (P2)      | REST API + dashboard for browsing/querying the data                                                                                                         |
| `receipt-search` (P3)   | Semantic search over purchases                                                                                                                              |
| `receipt-agent` (P4)    | Conversational Q&A; routes between structured queries (via `receipt-api`) and semantic search (via `receipt-search`) — does not touch the database directly |
| `receipt-forecast` (P5) | Purchase forecasting and spend anomaly detection                                                                                                            |
| `receipt-infra` (P6)    | Docker Compose, CI/CD, monitoring wrapping all of the above                                                                                                 |

Data flow: `receipt-etl` writes into the shared entities (`stores`, `receipts`, `line_items`, `extraction_reviews`); `receipt-api`, `receipt-search`, `receipt-agent`, and `receipt-forecast` all read from them, either directly or via `receipt-api`.

## Design principles that shape any related work

- **v1 is printed receipts only.** POS screenshots and handwritten receipts are out of scope; `receipt-etl` uses an extractor pattern so other source types can be added later without changing the shared schema.
- **Each of the six repos solves a distinct problem** (data engineering / API design / IR / agent routing / forecasting / infra) — if two end up doing similar work, one is scoped wrong.
- **Every repo documents what it does NOT do**, as explicitly as what it does.
- **The shared schema is a contract**: since P2–P6 all depend on what `receipt-core` produces, changes to it are cross-cutting.
- **Privacy is part of the design, not an afterthought**: source photos are the author's real purchase history. `receipt-etl` redacts personal details (names, card numbers, exact addresses) before data reaches storage; all sample/seed data across repos is synthetic or redacted.
- **Accuracy is reported as a number**: `receipt-etl` tracks percent auto-extracted vs. flagged for manual review; `receipt-agent` is checked against a ~30-question eval set; `receipt-forecast` uses a backtest against held-out recent months.

## Working in this repo specifically

- Keep `README.md` and `docs/master-plan.md` in sync — the README is the short public-facing summary; the master plan is the full write-up (principles, timeline, evaluation approach). Don't duplicate detailed content into the README beyond what's already summarized there.
- The architecture diagram appears in both files (ASCII art) — update both if the shared entity list or repo topology changes.
- This repo is the hub every other service repo links back to: per the master plan (Section 4), each of the six repos gets its own README with a problem statement and architecture section plus a link back to `receipt-intelligence-platform`. The shared schema and cross-repo architecture are meant to be documented once, here, and linked from every repo rather than re-explained in each one (Section 6).
- Do not add a `Co-Authored-By: Claude` trailer to commits in this repo.
