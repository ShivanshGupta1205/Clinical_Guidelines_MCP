# PROJECT_BRIEF.md — Clinical Guideline MCP Server

Read this alongside `CLAUDE.md` (operating rules for this session) and `LEARNING_LOG.md` (the gate — check it before implementing anything). This brief is the full technical spec; `CLAUDE.md` governs how you're allowed to act on it.

## What this is
An MCP server exposing public clinical guidelines (CDC/WHO/NIH) to an LLM client via two retrieval paths — structured lookup (dosages, thresholds — exact, deterministic) and semantic search (embedded chunks — fuzzy, explanatory). Portfolio project. Owner is learning the underlying concepts deliberately, not just shipping — see the gating rule in `CLAUDE.md`.

## Data sources
| Source | What | Access | License |
|---|---|---|---|
| CDC | MMWR "Recommendations and Reports" series — e.g. 2022 opioid prescribing guideline | HTML/PDF fetch, no API. Stable URLs under `cdc.gov/mmwr/rr/`. Well-structured (numbered recs, consistent sections) → deterministic parsing, not LLM extraction. | Public domain (17 U.S.C. §105) |
| WHO | Select guidelines via IRIS (`iris.who.int`) | No documented public API — manual curation: search, note handle URL, download PDF directly. 2–3 documents only, for cross-org comparison. | CC BY-NC-SA 3.0 IGO — non-commercial reuse w/ attribution OK |
| NIH/NLM | NCBI Bookshelf (`db=books`) via E-utilities | Real REST API (`eutils.ncbi.nlm.nih.gov`). ESearch → EFetch, XML. 3 req/s anonymous, 10 req/s with free API key (register one). | Public domain |

Target corpus: 5–8 source documents across 4–6 topics with clear structured elements (dosages, numeric thresholds) — e.g. opioid MME limits, adult immunization schedule, hypertension BP thresholds, antibiotic dosing. Do not attempt a comprehensive corpus. Do not touch MIMIC-III or any credentialed/PHI-adjacent dataset.

## Architecture
```
Ingestion pipeline (offline/batch)
  fetch → parse → chunk + embed → extract structured facts
        ↓
Postgres + pgvector
  guidelines | guideline_chunks | structured_facts | tool_call_log
        ↓
MCP server (Python, official mcp SDK)
  tools / resources / prompts / elicitation
  stdio (local dev) → Streamable HTTP (remote)
        ↓
  Claude Desktop (manual)      Eval harness (scripted MCP client, CI)
```

## DB schema
```sql
CREATE TABLE guidelines (
  guideline_id   TEXT PRIMARY KEY,
  source_org     TEXT NOT NULL,           -- 'cdc' | 'who' | 'nih'
  title          TEXT NOT NULL,
  topic          TEXT NOT NULL,
  publish_date   DATE,
  source_url     TEXT NOT NULL,
  superseded_by  TEXT REFERENCES guidelines(guideline_id),
  ingested_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE guideline_chunks (
  chunk_id       SERIAL PRIMARY KEY,
  guideline_id   TEXT REFERENCES guidelines(guideline_id),
  section        TEXT,
  recommendation_number INT,
  chunk_text     TEXT NOT NULL,
  embedding      VECTOR(384)              -- match embedding model dim
);

CREATE TABLE structured_facts (               -- the "not pure RAG" table
  fact_id        SERIAL PRIMARY KEY,
  guideline_id   TEXT REFERENCES guidelines(guideline_id),
  subject        TEXT NOT NULL,
  indication     TEXT,                    -- population/condition, nullable on purpose
  value          TEXT NOT NULL,
  unit           TEXT,
  source_section TEXT
);

CREATE TABLE tool_call_log (
  id             SERIAL PRIMARY KEY,
  ts             TIMESTAMPTZ DEFAULT now(),
  tool_name      TEXT,
  params         JSONB,
  latency_ms     INT,
  result_count   INT,
  elicitation_triggered BOOLEAN DEFAULT false
);
```

## MCP capabilities — build decisions
| Primitive | Build? | Notes |
|---|---|---|
| Tools | Yes, core | See signatures below |
| Resources | Yes, lean | `guideline://{org}/{guideline_id}` and `.../rec/{n}` — cleaned markdown + metadata |
| Prompts | Yes, 1–2 | `clinical_query`, `guideline_summary` (bakes in "not a substitute for clinical judgment") |
| Sampling | Stretch | Server-initiated disambiguation of ambiguous `lookup_dosage` matches. Client support inconsistent — design for it, don't depend on it. |
| Elicitation | Stretch, high value | Multi-round-trip, formalized in MCP 2026-07-28 spec. Use on `lookup_dosage` when `indication` is missing and dose depends on population. |
| Roots | Skip, explicitly | Not applicable — no reason to browse client filesystem. State this decision in the README, don't just omit it silently. |

### Tool signatures
```python
search_guidelines(query: str, source_org: str | None, topic: str | None, top_k: int = 5)
get_recommendation(guideline_id: str, recommendation_number: int)
lookup_dosage(drug_or_topic: str, indication: str | None = None)
compare_guidelines(topic: str, orgs: list[str])          # week-2 stretch
list_guidelines(topic: str | None = None)
```

## Eval harness requirements
Scripted MCP client (not Claude Desktop UI) runs a hand-written golden set (40–60 queries) through the server, scoring separately:
- tool selection accuracy
- parameter extraction accuracy
- answer correctness (exact match for structured facts; LLM-as-judge w/ fixed faithfulness+relevance rubric for semantic answers)
- elicitation precision (correctly triggers on the ambiguous subset, doesn't on the rest)
- latency

Do not reuse a RAG-only eval framework (Ragas etc.) as-is — this evaluates tool-call correctness, not just retrieval, and that harness is written custom on purpose.

## Tech stack (pinned)
- Python + official `mcp` SDK
- Postgres + pgvector
- Embeddings: local sentence-transformers model (small BGE variant) — no paid API, deliberate cost/tradeoff choice
- Ingestion: `httpx`/`requests` + `pdfplumber`; raw E-utilities calls for NCBI
- Testing: `pytest` for tools; eval harness wired into GitHub Actions on push
- Transport: stdio → Streamable HTTP; auth is API key for MVP, OAuth 2.1 only if it becomes genuinely multi-user

## Phases (each gated by LEARNING_LOG.md — see CLAUDE.md)
1. **Data & ingestion** (days 1–4) — fetch, parse, chunk, embed, extract structured facts
2. **MCP core** (days 5–7) — tools, resources, one prompt, stdio server working in Claude Desktop
3. **Evaluation** (days 8–10) — golden set, harness, telemetry logging
4. **Deployment** (days 11–12) — Streamable HTTP, containerize, deploy
5. **Scaling & ops** (days 13–14, stretch) — caching, index tuning, elicitation, versioning/monitoring

Days 1–7 are the real MVP. Everything after is what elevates it.
