# DECISIONS.md — decision log

Short records of *why* the major choices were made. Format per entry: **context → decision → why**. These exist so the reasoning survives, and so it can be explained in an interview instead of shrugged at.

Status values: `accepted` (in force), `superseded` (replaced by a later entry), `proposed` (not yet committed).

---

### D1 — Data source: openFDA, not CDC scraping — `accepted`
- **Context:** Original plan scraped CDC/WHO/NIH web pages and PDFs. CDC has no clean data feed; a week was lost fighting the website structure.
- **Decision:** Use the openFDA JSON API as the data source.
- **Why:** Clean, structured JSON over HTTP; no scraping or PDF parsing; reliable and reproducible. The MCP/retrieval skills are identical; only the source changes.

### D2 — Curated/pinned corpus, not a crawler — `accepted`
- **Context:** Could auto-discover drugs, or hand-pick a set.
- **Decision:** Pin a deliberately chosen set of ~15–30 drugs across a few therapeutic areas.
- **Why:** The eval harness needs a fixed, reproducible corpus. Production systems pin clinical sources too. Framed as "deliberately scoped," never "comprehensive."

### D3 — Exact-lookup tools are core; semantic search is secondary — `accepted`
- **Context:** Leading with semantic search makes the project look like plain RAG.
- **Decision:** Build the exact-lookup tools (structured facts, adverse-event counts, recalls) first; treat semantic search over label text as the secondary leg.
- **Why:** Exact lookups from structured data are what make this MCP-and-not-RAG, and they're safer for facts like dosages (no fuzzy guessing).

### D4 — Elicitation promoted from stretch to core — `accepted`
- **Context:** Elicitation (server asks the client a clarifying question) was originally a stretch goal.
- **Decision:** Make it a core feature (drug/product/population disambiguation).
- **Why:** It's an MCP capability with no RAG equivalent — the clearest proof the project is genuinely MCP.

### D5 — Structured facts are extracted from JSON, never hand-typed — `accepted`
- **Context:** A human hand-entering a dosage table is data entry, not engineering, and it would undercut the "not just RAG" claim.
- **Decision:** Ingest structured facts programmatically from openFDA JSON (`/drug/ndc`, `/drug/event`). Humans only QA a sample.
- **Why:** Demonstrates a real extraction pipeline; scales; keeps the structured leg honest.

### D6 — Local embeddings, not a paid API — `accepted`
- **Context:** Semantic search needs an embedding model.
- **Decision:** Use a local sentence-transformers model (small BGE variant).
- **Why:** No paid API, deliberate cost/tradeoff choice; keeps the project self-contained.

### D7 — Postgres + pgvector for storage — `accepted`
- **Context:** Need to store structured facts and vector embeddings together.
- **Decision:** Single Postgres database with the pgvector extension.
- **Why:** One store for both structured (exact) and vector (semantic) data; production-standard; avoids running a separate vector DB.

### D8 — Keep the repo/package name for now — `accepted`
- **Context:** Subject changed from clinical guidelines to FDA drug info; repo is `Clinical-Guidelines-MCP`.
- **Decision:** Keep `Clinical-Guidelines-MCP` / `clinical_guidelines_mcp` for now.
- **Why:** Drug labels are clinical information (defensible), and renaming later is cheap enough. Revisit before public launch.

### D9 — Code-first process; teaching-project gate retired — `accepted`
- **Context:** The strict "learn fully in a teaching project, prove it, then build" gate produced deep theory but zero code in a week.
- **Decision:** Switch to code-first / learn-by-doing. User implements first; Claude refines and teaches from the real code.
- **Why:** Building-then-learning is faster and sustains motivation; the old gate optimized understanding at the cost of ever shipping.
