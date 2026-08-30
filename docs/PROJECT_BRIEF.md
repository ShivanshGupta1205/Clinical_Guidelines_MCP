# PROJECT_BRIEF.md — Drug Information MCP Server

Read this alongside `CLAUDE.md` (operating rules) and `LEARNING_LOG.md` (build/learning journal). This brief is the technical spec — the north star. `DECISIONS.md` records *why* the major choices were made.

> Repo/package name is still `Clinical-Guidelines-MCP` / `clinical_guidelines_mcp` for now (see DECISIONS.md D8). The *subject* has changed from CDC clinical guidelines to **FDA drug information via openFDA**.

## What this is
An **MCP server** that exposes public FDA drug information to an LLM client through a set of tools. It deliberately mixes two retrieval styles:
- **Structured / exact lookup** — facts read straight from structured data (ingredient strengths, adverse-event counts, recalls). Deterministic, no guessing. *This is the core.*
- **Semantic search** — fuzzy, meaning-based search over free-text label sections (warnings, indications). Useful but secondary.

Plus the MCP-specific feature that plain RAG cannot do: **elicitation** — the server asks the client a clarifying question (e.g. which drug product / population) before answering.

Portfolio project, meant to hold up under interview questioning. The emphasis (exact-lookup + elicitation first, semantic search second) is deliberate — it's what makes this an MCP project rather than a RAG demo. See DECISIONS.md D3/D4.

## Data source — openFDA
openFDA is the FDA's free, public data service. It returns clean JSON over HTTP — **no scraping, no PDF parsing**. Base URL: `https://api.fda.gov/`. Free API key is optional (raises daily rate limit to ~120k/day; ~1k/day without).

| Endpoint | What it gives | Retrieval style it feeds |
|---|---|---|
| `/drug/label.json` | Official drug label, split into named sections (`indications_and_usage`, `dosage_and_administration`, `warnings`, `boxed_warning`, `contraindications`, `adverse_reactions`, `drug_interactions`, `use_in_specific_populations`, `overdosage`, …). Section names are structured; text inside is free-form. | **Semantic search** (free-text sections) |
| `/drug/ndc.json` | National Drug Code directory. Truly structured fields: `active_ingredients` (name + strength), `dosage_form`, `route`, `product_type`, packaging. | **Exact lookup** (structured facts) |
| `/drug/event.json` | Adverse-event reports (FAERS). With `count=`, returns ranked **counts** of reactions per drug. | **Exact lookup** (real numbers) |
| `/drug/enforcement.json` | Drug recalls (what, why, severity). | **Exact lookup** (structured records) |

Harmonized identifiers across endpoints (so the same drug can be matched everywhere): `generic_name`, `brand_name`, `product_ndc`, `spl_set_id`, `route`, `dosage_form`, `product_type`, `rxcui`, `unii`.

**Corpus is curated/pinned, not crawled** (DECISIONS.md D2): a deliberately chosen set of ~15–30 drugs across a few therapeutic areas, so the eval harness runs against a fixed, reproducible set. "Deliberately scoped," never "comprehensive." Not medical advice — openFDA's own disclaimer is baked into the server prompt.

## Architecture
```
Ingestion (offline/batch)
  openFDA API (httpx) → normalize JSON → write structured facts + label sections
        ↓
Postgres + pgvector
  drugs | drug_facts | drug_label_sections | adverse_event_counts | recalls | tool_call_log
        ↓
MCP server (Python, official mcp SDK)
  tools / resources / prompts / elicitation
  stdio (local dev) → Streamable HTTP (remote)
        ↓
  Claude Desktop (manual)      Eval harness (scripted MCP client, CI)
```

## DB schema (draft — refine while building)
```sql
CREATE TABLE drugs (
  drug_id        TEXT PRIMARY KEY,        -- deterministic, from a stable openFDA id (e.g. spl_set_id or product_ndc)
  generic_name   TEXT,
  brand_name     TEXT,
  product_ndc    TEXT,
  spl_set_id     TEXT,
  product_type   TEXT,                    -- 'HUMAN PRESCRIPTION DRUG' | 'HUMAN OTC DRUG' | ...
  route          TEXT,
  dosage_form    TEXT,
  source_url     TEXT,
  ingested_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE drug_facts (                 -- the exact-lookup / "not RAG" table (from /drug/ndc)
  fact_id        SERIAL PRIMARY KEY,
  drug_id        TEXT REFERENCES drugs(drug_id) ON DELETE CASCADE,
  subject        TEXT NOT NULL,           -- e.g. active ingredient name
  value          TEXT NOT NULL,           -- e.g. strength value
  unit           TEXT,                    -- e.g. 'mg'
  kind           TEXT                     -- 'active_ingredient' | 'strength' | ...
);

CREATE TABLE drug_label_sections (        -- the semantic-search leg (from /drug/label)
  section_id     SERIAL PRIMARY KEY,
  drug_id        TEXT REFERENCES drugs(drug_id) ON DELETE CASCADE,
  section        TEXT NOT NULL,           -- e.g. 'warnings', 'indications_and_usage'
  section_text   TEXT NOT NULL,
  embedding      VECTOR(384)              -- nullable; filled in the embeddings phase
);

CREATE TABLE adverse_event_counts (       -- exact numbers (from /drug/event count=)
  id             SERIAL PRIMARY KEY,
  drug_id        TEXT REFERENCES drugs(drug_id) ON DELETE CASCADE,
  reaction       TEXT NOT NULL,
  report_count   INT NOT NULL
);

CREATE TABLE recalls (                     -- exact records (from /drug/enforcement)
  id             SERIAL PRIMARY KEY,
  drug_id        TEXT REFERENCES drugs(drug_id) ON DELETE CASCADE,
  reason         TEXT,
  classification TEXT,                    -- recall severity class
  status         TEXT,
  recall_date    DATE
);

CREATE TABLE tool_call_log (               -- telemetry
  id             SERIAL PRIMARY KEY,
  ts             TIMESTAMPTZ DEFAULT now(),
  tool_name      TEXT,
  params         JSONB,
  latency_ms     INT,
  result_count   INT,
  elicitation_triggered BOOLEAN DEFAULT false
);
```

Revision strategy: whole-drug replacement (delete the `drugs` row → cascades to facts/sections/events/recalls → re-ingest). `drug_id` is deterministic so re-ingestion targets the same row.

## MCP capabilities — build decisions
| Primitive | Build? | Notes |
|---|---|---|
| Tools | Yes, core | See signatures below |
| Elicitation | **Yes, core** (promoted from stretch) | The differentiator. Ask for the specific drug product / population before answering. |
| Resources | Yes, lean | `drug://ndc/{product_ndc}` and `drug://label/{drug_id}/{section}` — structured facts + cleaned section text |
| Prompts | Yes, 1–2 | `drug_query`, `drug_summary` (bakes in openFDA's "not medical advice" disclaimer) |
| Sampling | Stretch | Server-initiated disambiguation where a client supports it. Design for it, don't depend on it. |
| Roots | Skip, explicitly | No reason to browse client filesystem. State this decision in the README. |

### Tool signatures (exact-lookup + elicitation first; semantic second)
```python
# --- Core: exact lookup + elicitation (build first) ---
find_drug(name: str)                                   # resolve name -> product(s); elicits on ambiguity
get_dosing(drug: str, population: str | None = None)   # strength/form/route (structured) + dosing; elicits population
top_adverse_events(drug: str, top_k: int = 10)         # ranked counts from FAERS (pure exact numbers)
check_recalls(drug: str)                               # structured recall records

# --- Secondary: semantic search (build after embeddings) ---
search_label(drug: str, question: str, top_k: int = 5) # fuzzy search over free-text label sections

# --- Stretch ---
compare_drugs(drug_a: str, drug_b: str, aspect: str)   # e.g. compare warnings / adverse-event profiles
```

## Eval harness requirements
Scripted MCP client (not the Claude Desktop UI) runs a hand-written golden set (40–60 queries), scoring separately:
- tool selection accuracy (did it pick the right tool?)
- parameter extraction accuracy (did it fill the tool's inputs right?)
- answer correctness (exact match for structured facts; LLM-as-judge with a fixed faithfulness+relevance rubric for semantic answers)
- elicitation precision (triggers on the ambiguous subset, not on the rest)
- latency

Do not reuse a RAG-only eval framework as-is — this measures tool-call correctness, not just retrieval.

## Tech stack (pinned)
- Python + official `mcp` SDK
- Postgres + pgvector
- Embeddings: local sentence-transformers model (small BGE variant) — no paid API (DECISIONS.md D6)
- Ingestion: `httpx` against openFDA JSON endpoints
- Testing: `pytest` per tool; eval harness wired into GitHub Actions
- Transport: stdio → Streamable HTTP; auth is API key for MVP, OAuth 2.1 only if it becomes genuinely multi-user

## Phases (code-first; exact-lookup + MCP core prioritized)
1. **Ingestion** — fetch label + ndc (+ event, enforcement) for the curated drug set; normalize; write `drugs`, `drug_facts`, `drug_label_sections` (embedding NULL), `adverse_event_counts`, `recalls`.
2. **MCP core (exact + elicitation)** — `find_drug`, `get_dosing`, `top_adverse_events`, `check_recalls`; stdio server working in Claude Desktop; elicitation on ambiguous drug/population.
3. **Semantic leg** — generate embeddings; `search_label`; resources + one prompt.
4. **Evaluation** — golden set, harness, telemetry logging.
5. **Deployment** — Streamable HTTP, containerize, deploy.
6. **Stretch** — `compare_drugs`, caching/index tuning, versioning/monitoring.
