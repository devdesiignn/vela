# What is this?

**Name:** Vela (pronounced VEL-uh).

**What it is:** A small personal data platform — the "Receipt Intelligence Platform," in plain language. Since September 2025 paper receipts from shopping trips around Ilorin, Nigeria have been photographed. Vela turns that growing pile of photos into a structured, queryable record of what was bought, where, and when.

**Why it exists:** To go from a pile of receipt photos to answerable questions ("how much did I spend on X last month?", "when did I last buy Y?", "is this month's spending unusual?") via a real, end-to-end system — not a single script, but six independently-scoped services covering data engineering, API design, search, a conversational agent, forecasting, and infra.

**How it's structured:** One shared dataset feeds six independent, separately-repo'd services built around a common core data layer (`vela-core`). Each repo solves a distinct problem; `vela-api` is the only service that talks to the database directly, everything else goes through it.

**This repo (`vela`):** The docs-only hub. No application code lives here — just the system-level plan, architecture diagram, and links to every service repo. See [README.md](README.md) for the repo map and diagram, and [docs/MASTER-PLAN.md](docs/MASTER-PLAN.md) for the full write-up (principles, timeline, evaluation approach).

**Naming:** On GitHub, every repo carries the `vela-` prefix (`vela-core`, `vela-etl`, `vela-api`, `vela-search`, `vela-agent`, `vela-forecast`, `vela-infra`); the hub repo is just `vela`. Locally, folders drop the prefix since they sit inside a `vela/` parent directory (`core`, `etl`, `api`, `search`, `agent`, `forecast`, `infra`, `docs`).

**Status:** In development, Sept 2026 – Dec 2026. See the timeline in [README.md](README.md).
