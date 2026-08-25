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

Target corpus: **9 documents — 4 CDC, 3 WHO, 2 NIH/Bookshelf** — across topics with clear structured elements (dosages, numeric thresholds), chosen so at least 1–2 topics have real cross-org coverage (CDC + WHO/NIH) to support `compare_guidelines`. Expect ~400–800 chunks and ~70–130 structured-fact rows total. Do not attempt a comprehensive corpus. Do not touch MIMIC-III or any credentialed/PHI-adjacent dataset.

Ingestion is a **real idempotent pipeline** (fetch → parse → chunk/embed → structured-fact staging → reviewed promotion to `structured_facts`), not a one-time script — it needs to re-run cleanly when a document is added or a source guideline is updated (see `superseded_by`). The structured-fact promotion step is a mandatory human-review checkpoint regardless of how automated the rest of the pipeline is — never auto-promote extracted dosage/threshold candidates straight to `structured_facts`.

## Architecture
```
Ingestion pipeline (offline/batch)
  fetch → parse → chunk + embed → extract structured facts
        ↓
Postgres + pgvector
  guidelines | guideline_chunks | structured_facts_staging | structured_facts | tool_call_log
        ↓
MCP server (Python, official mcp SDK)
  tools / resources / prompts / elicitation
  stdio (local dev) → Streamable HTTP (remote)
        ↓
  Claude Desktop (manual)      Eval harness (scripted MCP client, CI)
```

## DB schema — finalized after critical review

Decisions made deliberately during review, worth keeping in mind while building:
- `topic` stays plain `TEXT`, assigned manually/consistently during curation (not a controlled vocab, not a tags table) — if manual consistency proves insufficient once `compare_guidelines` is actually being tested across orgs, the documented fallback is a semantic-matching layer, not a bigger schema.
- `value` in `structured_facts` stays `TEXT`, not split into numeric fields — a deliberate simplicity tradeoff; it means no range-querying/validation on dosage values, accepted because much of this data (e.g. "IV artesunate followed by oral therapy") doesn't reduce to a clean number anyway.
- `structured_facts_staging` exists specifically so nothing reaches `structured_facts` without human review — see the ingestion note above. Staging rows are NOT unique-constrained (duplicates across re-runs are fine, review handles it); the final table is.
- `guideline_chunks.embedding` needs `pgvector`'s `CREATE EXTENSION vector;` run once, before this DDL.

```sql
CREATE TYPE source_org_enum AS ENUM ('cdc', 'who', 'nih');

CREATE TABLE guidelines (
  guideline_id   TEXT PRIMARY KEY,                                -- human-readable ID, e.g. 'cdc-opioid-2022'
  source_org     source_org_enum NOT NULL,                        -- which org published it
  title          TEXT NOT NULL,                                   -- guideline title
  topic          TEXT NOT NULL,                                   -- topic label, assigned manually & consistently at curation time
  publish_date   DATE,                                            -- original publish date
  source_url     TEXT NOT NULL,                                   -- hub/landing page URL
  superseded_by  TEXT REFERENCES guidelines(guideline_id),        -- points to the newer version, if this one's outdated
  ingested_at    TIMESTAMPTZ DEFAULT now()                        -- when this row was ingested
  -- future: updated_at, reviewed_at — add once needed
);

CREATE TABLE guideline_chunks (
  chunk_id       SERIAL PRIMARY KEY,                                                  -- internal chunk ID (also preserves in-section order — see note below)
  guideline_id   TEXT NOT NULL REFERENCES guidelines(guideline_id) ON DELETE CASCADE, -- parent guideline
  section        TEXT,                                                                -- human-readable section label
  section_url    TEXT,                                                                -- exact sub-page URL this chunk came from
  chunk_text     TEXT NOT NULL,                                                       -- the chunked text
  embedding      VECTOR(384)                                                          -- embedding vector — dimension must match your model
);
-- chunk_id doubles as ordering within a (guideline_id, section) group, AS LONG AS
-- ingestion always inserts a section's chunks in reading order — true for a
-- single-threaded pipeline, which is the plan. No separate chunk_index column.

CREATE INDEX guideline_chunks_embedding_idx
  ON guideline_chunks USING hnsw (embedding vector_cosine_ops);   -- ANN index for semantic search, cosine distance

CREATE TABLE structured_facts_staging (
  staging_id     SERIAL PRIMARY KEY,                                                  -- staging row ID
  guideline_id   TEXT NOT NULL REFERENCES guidelines(guideline_id) ON DELETE CASCADE, -- parent guideline
  subject        TEXT NOT NULL,                                                       -- drug/topic the fact is about
  indication     TEXT,                                                                -- condition/use-case the value applies to
  population     TEXT,                                                                -- patient population it applies to (e.g. "pediatric")
  value          TEXT NOT NULL,                                                       -- extracted dose/threshold, as text
  unit           TEXT,                                                                -- unit for the value
  source_section TEXT,                                                                -- human-readable section label
  source_url     TEXT,                                                                -- exact sub-page URL the fact was pulled from
  status         TEXT NOT NULL DEFAULT 'pending',                                     -- 'pending' | 'approved' | 'rejected'
  reviewed_by    TEXT,                                                                -- who reviewed it
  reviewed_at    TIMESTAMPTZ                                                          -- when it was reviewed
);

CREATE TABLE structured_facts (               -- the "not pure RAG" table — promoted from staging only
  fact_id        SERIAL PRIMARY KEY,                                                  -- final fact ID
  guideline_id   TEXT NOT NULL REFERENCES guidelines(guideline_id) ON DELETE CASCADE, -- parent guideline
  subject        TEXT NOT NULL,                                                       -- drug/topic the fact is about
  indication     TEXT,                                                                -- condition/use-case the value applies to
  population     TEXT,                                                                -- patient population it applies to
  value          TEXT NOT NULL,                                                       -- dose/threshold, as text
  unit           TEXT,                                                                -- unit for the value
  source_section TEXT,                                                                -- human-readable section label
  source_url     TEXT,                                                                -- exact sub-page URL the fact was pulled from
  UNIQUE (guideline_id, subject, indication, population)                              -- blocks duplicate facts on re-ingestion
);

CREATE TABLE tool_call_log (
  id             SERIAL PRIMARY KEY,                -- log row ID
  ts             TIMESTAMPTZ DEFAULT now(),          -- when the call happened
  tool_name      TEXT,                               -- which MCP tool was called
  params         JSONB,                              -- the call's parameters
  latency_ms     INT,                                -- how long the call took
  result_count   INT,                                -- how many results were returned
  elicitation_triggered BOOLEAN DEFAULT false,        -- whether elicitation fired
  session_id     TEXT                                 -- MCP session ID — unused/null until Streamable HTTP, deferred
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
get_section(guideline_id: str, section: str)              # renamed from get_recommendation — recommendation_number was dropped from the schema; the CDC hub-page corpus never had numbered recs anyway
lookup_dosage(drug_or_topic: str, indication: str | None = None, population: str | None = None)
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
0. **DB connection & schema setup** (first step, day 1) — connect to Postgres, run the finalized DDL above. The relational tables (`guidelines`, `structured_facts_staging`, `structured_facts`, `tool_call_log`) don't need a learning-gate check — this is plain SQL/DDL, an existing skill, not new ground. `guideline_chunks.embedding` and its HNSW index specifically DO wait on M2 (pgvector/embeddings) clearing, since that's the actually-new part. Split the DDL into two migration steps if useful — one gate-free, one gated.
1. **Data & ingestion** (days 1–4) — fetch, parse, chunk, embed, extract structured facts
2. **MCP core** (days 5–7) — tools, resources, one prompt, stdio server working in Claude Desktop
3. **Evaluation** (days 8–10) — golden set, harness, telemetry logging
4. **Deployment** (days 11–12) — Streamable HTTP, containerize, deploy
5. **Scaling & ops** (days 13–14, stretch) — caching, index tuning, elicitation, versioning/monitoring

Days 1–7 are the real MVP. Everything after is what elevates it.
