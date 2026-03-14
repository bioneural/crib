# Crib Evolution Plan

Informed by a comparison with [Google's always-on memory agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent) and the requirements of [Prophet](https://github.com/bioneural/prophet) — an autonomous multi-agent operating system.

---

## 1. Background Consolidation Daemon

**Problem**: Crib consolidates on write — superseding triples with matching subject+predicate. It does not perform generative consolidation: reviewing accumulated memories to discover cross-cutting patterns and synthesize new understanding.

**Design**:
- New subcommand: `crib consolidate`
- Queries unconsolidated entries (add `consolidated` boolean column to `entries`)
- Sends batch to LLM with prompt: identify connections between memories, generate insight summaries
- Stores consolidations in a new `consolidations` table: `id`, `source_ids` (JSON), `summary`, `insight`, `created_at`
- Updates source entries: marks as consolidated, adds bidirectional `connections` JSON field linking related entries
- Can be run on a cron or by an external scheduler (prophet's coordinator)

**Migration**:
- Add `consolidated` (boolean, default false) column to `entries`
- Add `connections` (text/JSON, default '[]') column to `entries`
- Create `consolidations` table

**Acceptance criteria**:
- `crib consolidate` reviews entries where `consolidated = 0`, groups >= 2
- Produces consolidation records with source IDs and synthesized insight
- Marks source entries as consolidated
- Bidirectional connection links written to both sides of each discovered relationship
- Idempotent — running twice with no new entries produces no new consolidations
- Fail-open — if the LLM is unavailable, exits 0 with no changes

---

## 2. Startup Summary Generator

**Problem**: Cold-start agents have no context. They must either explore the memory store (slow, inconsistent) or receive a query-triggered retrieval (reactive, not proactive). Neither gives a new agent a comprehensive briefing at startup.

**Design**:
- New subcommand: `crib briefing`
- Reads all active entries, all consolidation insights, all operator-sourced directives
- Sends to LLM with prompt: produce a compact summary document covering active facts, standing directives, recent insights, and key decisions
- Output: markdown document suitable for injection into an agent's system prompt
- Optional `--role` flag for role-scoped briefings (e.g., `--role planner`, `--role worker`)
- Cached: store generated briefing with timestamp, regenerate only when entries have changed since last generation

**Storage**:
- New `briefings` table: `id`, `role` (nullable), `content`, `entry_count`, `created_at`
- Or simpler: write to a well-known file path (e.g., `$CRIB_DIR/briefing.md`)

**Acceptance criteria**:
- `crib briefing` produces a concise markdown document covering the current memory state
- Document fits within ~2000 tokens (8000 chars) — same budget as retrieval output
- Directives (operator-sourced) are always included
- Consolidation insights are prioritized over raw entries
- `--role` flag filters content appropriately (details TBD per role)
- Stale briefings are detected and regenerated

---

## 3. Multimodal Ingestion

**Problem**: Crib is text-only. Prophet needs to ingest images, PDFs, audio, and video — research papers, screenshots, invoices, meeting recordings.

**Design**:
- Extend `crib write` to accept file paths in addition to stdin text
- New flag or prefix: `file=/path/to/document.pdf`
- For non-text files: detect MIME type, send raw bytes to a multimodal LLM (Anthropic Claude with vision, or similar), extract text description
- Store the LLM-generated text description as the entry content, with `source_file` metadata
- Triple extraction and embedding proceed on the text description as usual

**Acceptance criteria**:
- `crib write file=/path/to/image.png` extracts text description and stores as entry
- Supported formats: PNG, JPG, PDF (at minimum)
- Original file path stored in metadata for provenance
- Triple extraction and embedding work on the extracted text
- Fail-open — if multimodal extraction fails, log warning and skip (don't block)

---

## 4. Concurrent Write Safety

**Problem**: Prophet runs multiple agents concurrently. Multiple agents may write to crib simultaneously. Current dedup and triple consolidation logic assumes single-writer.

**Design**:
- Audit all write-path operations for race conditions under concurrent SQLite writers
- Wrap dedup-check + insert in a transaction
- Wrap triple-supersession (check existing + update `valid_until` + insert new) in a transaction
- Consider `BEGIN IMMEDIATE` for write transactions to avoid `SQLITE_BUSY` under contention
- Add retry logic with exponential backoff for `SQLITE_BUSY` errors
- Test with parallel writes from multiple processes

**Acceptance criteria**:
- Two concurrent `crib write` processes with identical content: exactly one entry created
- Two concurrent `crib write` processes with same-subject-same-predicate triples: exactly one supersession, no data loss
- No `SQLITE_BUSY` errors propagated to callers under normal concurrent load
- Fail-open preserved — write contention never blocks the calling agent indefinitely

---

## 5. README/Code Alignment

**Problem**: README documents an earlier version (Ollama-based extraction, nomic-embed-text 768-dim embeddings). Code uses Anthropic Claude Haiku for extraction/reranking and Voyage AI voyage-4-large 1024-dim embeddings.

**Design**:
- Update README to reflect current implementation
- Document all environment variables and their current defaults
- Document the consolidation and briefing subcommands once implemented

**Acceptance criteria**:
- README accurately describes the current code
- All subcommands documented with usage examples
- Environment variables listed with defaults and descriptions

---

## Sequencing

1. **Background consolidation** — foundation for briefings (briefings prioritize consolidation insights)
2. **Startup summary generator** — depends on consolidation being in place
3. **Concurrent write safety** — needed before Prophet runs multiple agents
4. **Multimodal ingestion** — independent, can be done in parallel with 3
5. **README alignment** — last, once all features are implemented

---

## Non-goals

- **Replacing SQLite.** The current storage layer is sufficient. Vector search, FTS, and relational queries in a single embedded database is a strength, not a limitation to overcome.
- **Adding authentication.** Crib runs as a local tool invoked by agents on the same machine. Multi-tenant access control is a Prophet-level concern, not a crib-level concern.
- **Granular reranking scores.** The [binary reranking investigation](/posts/your-reranker-is-a-no-op) showed that binary discrimination (1.0 or 0.0) is sufficient for a small personal memory store. Pursuing continuous relevance scores is not worth the complexity.
- **Session persistence.** Each invocation is stateless by design. Memory lives in SQLite, not in conversation history. This is correct for a tool that multiple agents invoke concurrently.
