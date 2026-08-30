# CHANGELOG.md

One line per meaningful change, per docs/CLAUDE.md convention.

## 2026-08-30
- Pivoted subject from CDC clinical-guideline scraping to FDA drug information via the openFDA JSON API (see `docs/DECISIONS.md` D1).
- Switched to a code-first, learn-by-doing workflow; retired the teaching-project gate (D9).
- Repo hygiene: added `.gitignore`; untracked `.idea/` and `.DS_Store`.
- Trimmed `requirements.txt` (removed `pdfplumber`).
- Rewrote `docs/PROJECT_BRIEF.md` for the openFDA design (new schema, exact-lookup-first tools, elicitation core).
- Slimmed `docs/CLAUDE.md` to code-first operating rules.
- Repurposed `docs/LEARNING_LOG.md` from a gate table into a build/learning journal.
- Added `docs/DECISIONS.md` (decision log), seeded D1–D9.

## 2026-08-15
- Scaffolded empty repo structure (`ingestion/`, `db/`, `mcp_server/`, `eval/`, `tests/`) + pinned `requirements.txt`. No implementation — Phase 1 gate (M1–M3 in `docs/LEARNING_LOG.md`) not yet cleared.
- Restructured module folders under `src/clinical_guidelines_mcp/` (src layout) with `pyproject.toml` for package discovery; `tests/` kept at top level per convention.
- Added this `CHANGELOG.md`.
