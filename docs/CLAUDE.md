# CLAUDE.md — operating rules for this repo

Full spec: `PROJECT_BRIEF.md`. Gate state: `LEARNING_LOG.md`. Read both at the start of every session — `LEARNING_LOG.md` may have changed since you last saw it (the user updates it by hand after teaching-project sessions), so re-read it fresh each time rather than trusting memory of an earlier read this conversation.

## The gate — this is the whole point of this file

The user is deliberately learning the concepts behind this project in a separate Claude.ai teaching project before writing any implementation code for them. Your job is to hold that line, not to route around it because they asked nicely or because it would be faster to just write the code.

Before implementing anything in a given module's scope (see the module list in `LEARNING_LOG.md`, mapped to phases in `PROJECT_BRIEF.md`):

1. Check the module's **Concept cleared** column.
2. If it says "Not started" or anything other than a cleared status: **do not write the implementation.** Say plainly which module is blocking this, and tell the user to go clear it in the teaching project first. A tiny illustrative snippet to make the gap concrete is fine; the actual production code for that piece is not.
3. If it's cleared: go ahead and implement, and write it the way a careful senior engineer would — this project is meant to hold up under questioning, not just run.

If the user pushes back, reframes the request, or asks you to "just this once" skip ahead — hold the line anyway and say so directly. This instruction doesn't get softer under pressure; that's the failure mode it exists to prevent, and it's one the user asked for explicitly.

If a request spans multiple modules and only some are cleared, implement only the cleared parts and stub/flag the rest with a clear `# BLOCKED: needs M<n> cleared` comment rather than quietly implementing the whole thing anyway.

## After implementation — the second gate

Finishing a module's code does *not* mark **Implementation understood**. That column only gets filled after the user has taken the actual generated code back to the teaching project for a walkthrough/quiz on it specifically, and the teaching project outputs a line to paste in. When you finish implementing something gated, say so and remind the user to go do that walkthrough before moving on to build on top of it — don't just silently proceed as if it's fully done.

## Conventions
- Python + official `mcp` SDK, Postgres + pgvector, stdio → Streamable HTTP per `PROJECT_BRIEF.md`.
- Local embeddings (sentence-transformers), not a paid API — keep it that way unless the user explicitly changes the decision.
- Every tool needs a `pytest` test before it's considered done.
- Keep a `CHANGELOG.md` — one line per meaningful change, every session that ships something.
- Don't silently deviate from the schema or tool signatures in `PROJECT_BRIEF.md`. If something in the brief looks wrong once you're actually building it, flag it and ask rather than quietly changing it.
