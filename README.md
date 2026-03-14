<h1 align="center">
  c r i b
  <br>
  <sub>memory storage and retrieval for AI agents</sub>
</h1>

Agents forget between sessions. Decisions made yesterday are relitigated today. Context that shaped a choice evaporates before the next prompt arrives. Crib is the storage layer that solves this — one SQLite database with three query channels, generative consolidation that synthesizes cross-cutting insights, bidirectional connection graphs that discover latent relationships, and a retrieval command that surfaces what's relevant for a given prompt.

A single script. Ruby stdlib plus the sqlite3 gem. Anthropic and Voyage APIs for extraction, reranking, and embeddings.

---

## How it works

Crib stores memory in an SQLite database with three query channels:

| Channel | Mechanism | Use case |
|---------|-----------|----------|
| Fact triples | SQL JOINs on `(subject, predicate, object)` | "What are ClientA's payment terms?" |
| Full-text | FTS5 with Porter stemming | "What did we decide about caching?" |
| Vector | sqlite-vec nearest neighbors | Semantic similarity when keywords fail |

### Fact triples

A fact triple represents a single relationship between two things: `(subject, predicate, object)`. The sentence "We chose SQLite for storage" becomes the triple `(SQLite, storage_backend, project)`. The sentence "ClientA pays Net30" becomes `(ClientA, payment_terms, Net30)`.

Triples are precise where full-text search is fuzzy. Ask "what are ClientA's terms?" and a keyword search has to hope the right document surfaces. A triple query walks directly from the entity `ClientA` through the predicate `payment_terms` to the answer `Net30` — one SQL JOIN, no ranking, no ambiguity.

Crib extracts triples automatically on write via the Anthropic API. You write natural text; crib stores both the text (for full-text search) and the structured facts (for precise recall). When a new fact contradicts an old one, the old triple is superseded — marked with a `valid_until` timestamp rather than deleted. History is preserved but current queries return only what's true now.

**Write path** — text arrives on stdin. Crib stores it as a full-text entry, generates a vector embedding via the Voyage API, extracts fact triples via the Anthropic API, and performs consolidation-on-write: if a new fact supersedes an existing one, the old relation is marked `valid_until` rather than duplicated. The archive stays clean without manual curation.

**Read path** — a prompt arrives on stdin. Crib extracts keywords and generates an embedding of the prompt, then queries all three channels: triples via SQL JOINs, full-text via FTS5, and similar entries via sqlite-vec nearest neighbors. FTS and vector results are merged using Reciprocal Rank Fusion (RRF) — entries found by both channels score higher than single-channel results. A vector distance threshold filters noise before fusion. The fused results are then reranked using a cross-encoder: the Anthropic API scores each query-document pair with a yes/no relevance judgment, and entries below a rerank threshold are discarded. After reranking, a single-hop graph expansion queries the connections table for entries connected to the top results, appending them as "Connected memories." The surviving results are wrapped in `<system-reminder>` tags. Each fact triple includes its `valid_from` date; each memory entry includes its `created_at` date. The consuming agent can reason about recency. Empty output means nothing relevant was found. Fail-open — a broken retrieval never blocks the agent.

### Generative consolidation

During `maintain`, crib batches unconsolidated entries and calls the Anthropic API to synthesize cross-cutting insights. Each insight is stored as an entry with `type=insight` and `source=consolidation`, so it automatically participates in all three retrieval channels. The `consolidations` table tracks provenance — which source entries contributed to each insight. Entries are marked `consolidated_at` after processing and are not reprocessed on subsequent runs.

### Connection graphs

The same consolidation API call also identifies pairs of meaningfully connected entries. These are stored in a `connections` table with canonical ordering (`entry_id_a < entry_id_b`) and a `strength` counter that increments when the same connection is rediscovered across runs. During retrieval, a single-hop graph expansion surfaces entries connected to the top results, ordered by strength. Connections represent relationships between *memories* (entry IDs), distinct from triples which represent relationships between *entities*.

### Hints

`crib hints` returns lightweight topic-level nudges — not full memory content, just enough for the agent to know relevant memories exist. Hooker wraps these in `<system-reminder>` tags and injects them. The agent sees the nudges and can call `crib retrieve` deliberately for full content.

```sh
echo "what database did we choose?" | bin/crib hints
# You have 3 stored memories about sqlite.
# You have a standing directive about storage.
```

Hints use FTS5 and triples only (no API calls) for speed. Output is capped at 5 lines. Empty output on any error. Exit 0 always.

---

## Usage

```sh
# Store a memory
echo "Chose SQLite for storage. Zero dependencies." | bin/crib write

# Store with a type prefix
echo "type=decision Chose SQLite for storage." | bin/crib write

# Retrieve relevant context for a prompt
echo "what database did we choose?" | bin/crib retrieve
# <memory context_time="2026-02-11T14:32:00Z">
# ## Known facts
# - SQLite → storage_backend → project (since 2026-02-09)
# ## Memory entries
# - [2026-02-09 decision] Chose SQLite for storage. Zero dependencies.
# </memory>

# Initialize the database explicitly (happens automatically on first write)
bin/crib init

# Check all prerequisites
bin/crib doctor
# { "ruby": { "version": "3.x", "ok": true }, ... "ok": true }

# Get lightweight topic hints for a prompt (no API calls)
echo "what database did we choose?" | bin/crib hints
# You have 3 stored memories about sqlite.

# Run memory maintenance — contradiction detection, correction linking, consolidation
bin/crib maintain
# {"contradictions_found":1,"corrections_linked":0,"stale_count":3,"supersessions_applied":1,"consolidations_generated":2,"connections_discovered":3}
```

### Entry types

The optional `type=` prefix on write categorizes entries:

| Type | Purpose |
|------|---------|
| `note` | General knowledge or context (default) |
| `decision` | A choice made and why |
| `correction` | Something previously believed that turned out wrong |
| `error` | A failure that occurred and what was learned |
| `interest` | A topic the agent is tracking for external intelligence |

### Environment variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Required for triple extraction, reranking, and consolidation |
| `VOYAGE_API_KEY` | Required for vector embeddings |

---

## Claude Code integration

Crib integrates with Claude Code via [hooker](https://github.com/bioneural/hooker), a policy engine for Claude Code hooks.

**Retrieve** is automated. A hooker inject policy fires on every prompt, pipes the prompt text to `bin/crib retrieve` via stdin, and injects whatever crib returns on stdout as additional context. If crib finds nothing relevant, stdout is empty and nothing is injected.

```ruby
policy "Memory" do
  on :UserPromptSubmit
  inject command: "bin/crib retrieve"
end
```

**Write** is agent-initiated. The agent decides what's worth persisting — a decision it made, an error it encountered, a correction to something it previously believed — and calls `bin/crib write` directly:

```sh
echo "type=decision Switched from Postgres to SQLite" | bin/crib write
```

There is no automatic write policy. What to store is a judgment call that belongs to the agent, not to hook plumbing.

### WAL mode

crib uses SQLite WAL (Write-Ahead Logging) mode for concurrent write safety. Multiple formation channels — escort extraction, blog reflection, operator feedback, diagnose findings — may write to crib concurrently. WAL allows concurrent readers during writes without blocking.

### Write-time dedup

Before storing a new entry, crib checks for identical content written in the last hour via FTS5. If an exact duplicate is found, the write is skipped with a log message. This prevents duplicate entries from overlapping formation channels (escort and trick) writing the same knowledge.

---

Crib's interface is stdin/stdout. Any agent framework that can pipe text to a command can use it — hooker is one integration path, not the only one.

---

## Schema

```sql
-- Fact triples (structured recall)
CREATE TABLE entities (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  type TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE relations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  subject_id INTEGER NOT NULL REFERENCES entities(id),
  predicate TEXT NOT NULL,
  object_id INTEGER NOT NULL REFERENCES entities(id),
  valid_from TEXT DEFAULT (datetime('now')),
  valid_until TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE sources (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  relation_id INTEGER NOT NULL REFERENCES relations(id),
  raw_text TEXT NOT NULL,
  timestamp TEXT DEFAULT (datetime('now'))
);

-- Full-text entries (unstructured recall)
CREATE TABLE entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  type TEXT NOT NULL,
  content TEXT NOT NULL,
  embedding BLOB,
  created_at TEXT DEFAULT (datetime('now')),
  last_retrieved_at TEXT,
  superseded_by INTEGER,
  consolidated_at TEXT
);

CREATE VIRTUAL TABLE entries_fts USING fts5(
  content,
  content_rowid='id',
  tokenize='porter unicode61'
);

-- Vector embeddings (semantic similarity)
CREATE VIRTUAL TABLE entries_vec USING vec0(
  embedding float[1024]
);

-- Consolidation provenance
CREATE TABLE consolidations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  content TEXT NOT NULL,
  source_entry_ids TEXT NOT NULL,   -- JSON array of entry IDs
  created_at TEXT DEFAULT (datetime('now')),
  superseded_by INTEGER
);

-- Bidirectional connection graph
CREATE TABLE connections (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  entry_id_a INTEGER NOT NULL,
  entry_id_b INTEGER NOT NULL,
  relationship TEXT NOT NULL,
  strength REAL DEFAULT 1.0,        -- reinforced when rediscovered
  created_at TEXT DEFAULT (datetime('now')),
  UNIQUE(entry_id_a, entry_id_b)    -- canonical order: a < b
);
```

Triples use bi-temporal validity — `valid_from` / `valid_until` on relations. A query for current facts filters on `valid_until IS NULL`. Old facts are superseded, not deleted.

Vector embeddings are stored in a sqlite-vec `vec0` virtual table. Each entry's embedding is a 1024-dimensional float vector (voyage-4-large). Retrieval generates an embedding of the prompt and finds nearest neighbors.

Consolidations track provenance — which source entries contributed to each synthesized insight. Insights are stored as regular entries (`type=insight`, `source=consolidation`) so they participate in all retrieval channels automatically.

Connections record discovered relationships between memory entries. The `strength` counter increments when the same pair is rediscovered across maintain runs, helping retrieval prioritize well-established connections over one-off hallucinations.

---

## Smoke tests

```
$ test/smoke-test
46 passed, 3 skipped
```

Tests cover structure, syntax, doctor, init, write, retrieve, preference reranking, maintain, decay, write-time dedup, provenance, correct, migration, consolidation schema, consolidation idempotency, connection discovery, connection retrieval, hints, and sync. Tests run against a temporary database that is created and destroyed on each run. No artifacts are left behind. The 3 skipped tests require API keys — run with `--full` to include them.

---

## Prerequisites

- Ruby 3.x
- Ruby sqlite3 gem (handles extension loading natively — no Homebrew sqlite3 binary needed)
- [sqlite-vec](https://github.com/asg017/sqlite-vec) (`pip3 install sqlite-vec`)
- `ANTHROPIC_API_KEY` — for triple extraction, reranking, and consolidation
- `VOYAGE_API_KEY` — for vector embeddings

Default models can be overridden via environment variables:

| Purpose | Default | Override |
|---------|---------|----------|
| Triple extraction / reranking / consolidation | `claude-haiku-4-5-20251001` | `CRIB_ANTHROPIC_MODEL` |
| Embeddings | `voyage-4-large` | `CRIB_VOYAGE_MODEL` |

Run `bin/crib doctor` to verify all prerequisites are installed and report status as JSON.

---

## License

MIT
