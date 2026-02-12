<h1 align="center">
  c r i b
  <br>
  <sub>memory storage and retrieval for AI agents</sub>
</h1>

Agents forget between sessions. Decisions made yesterday are relitigated today. Context that shaped a choice evaporates before the next prompt arrives. Crib is the storage layer that solves this — one SQLite database with three query channels, consolidation-on-write so the archive stays clean, and a retrieval command that surfaces what's relevant for a given prompt.

A single script. No gems. No API keys. Ollama runs locally for embeddings and extraction.

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

Crib extracts triples automatically on write using ollama. You write natural text; crib stores both the text (for full-text search) and the structured facts (for precise recall). When a new fact contradicts an old one, the old triple is superseded — marked with a `valid_until` timestamp rather than deleted. History is preserved but current queries return only what's true now.

**Write path** — text arrives on stdin. Crib stores it as a full-text entry, generates a vector embedding via ollama, extracts fact triples, and performs consolidation-on-write: if a new fact supersedes an existing one, the old relation is marked `valid_until` rather than duplicated. The archive stays clean without manual curation.

**Read path** — a prompt arrives on stdin. Crib extracts keywords and generates an embedding of the prompt, then queries all three channels: triples via SQL JOINs, full-text via FTS5, and similar entries via sqlite-vec nearest neighbors. Results are merged, deduplicated, sorted newest-first, reranked with ollama if over budget, and wrapped in `<memory>` tags with a `context_time` timestamp. Each fact triple includes its `valid_from` date; each memory entry includes its `created_at` date. The consuming agent can reason about recency — whether a memory is older or newer than what's already in the conversation. Empty output means nothing relevant was found. Fail-open — a broken retrieval never blocks the agent.

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
```

### Entry types

The optional `type=` prefix on write categorizes entries:

| Type | Purpose |
|------|---------|
| `note` | General knowledge or context (default) |
| `decision` | A choice made and why |
| `correction` | Something previously believed that turned out wrong |
| `error` | A failure that occurred and what was learned |

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CRIB_DB` | `.state/crib/crib.db` | Path to the SQLite database |
| `CRIB_SQLITE3` | auto-detected | Path to a sqlite3 binary with extension loading |
| `CRIB_VEC_EXTENSION` | auto-detected | Path to the sqlite-vec `vec0` extension |

Model overrides are documented in [Prerequisites](#prerequisites).

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
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE VIRTUAL TABLE entries_fts USING fts5(
  content,
  content_rowid='id',
  tokenize='porter unicode61'
);

-- Vector embeddings (semantic similarity)
CREATE VIRTUAL TABLE entries_vec USING vec0(
  embedding float[768]
);
```

Triples use bi-temporal validity — `valid_from` / `valid_until` on relations. A query for current facts filters on `valid_until IS NULL`. Old facts are superseded, not deleted.

Vector embeddings are stored in a sqlite-vec `vec0` virtual table. Each entry's embedding is a 768-dimensional float vector (nomic-embed-text default). Retrieval generates an embedding of the prompt and finds nearest neighbors.

---

## Smoke tests

```
$ bin/smoke-test

Doctor
  ✓ Doctor reports all prerequisites OK
  ✓ Doctor output is valid JSON

Init
  ✓ Init creates database
  ✓ Database file exists after init
  ✓ Database has all expected tables
  ✓ Vec0 virtual table exists

Write
  ✓ Write stores entry and reports success
  ✓ Write respects type= prefix
  ✓ Write extracts triples
  ✓ Write rejects empty input

Retrieve
  ✓ Retrieve finds entries via FTS5
  ✓ Retrieve finds fact triples
  ✓ Retrieve finds entries via vector similarity
  ✓ Retrieve wraps output in <memory> tags
  ✓ Retrieve includes context_time on <memory> tag
  ✓ Retrieve entries include date prefix
  ✓ Retrieve triples include since date
  ✓ Retrieve orders entries newest-first
  ✓ Retrieve with empty input exits cleanly
  ✓ Retrieve with missing database exits cleanly

Consolidation
  ✓ Consolidation-on-write runs without error

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
21 passed
```

Tests run against a temporary database that is created and destroyed on each run. No artifacts are left behind.

---

## Prerequisites

- Ruby 3.x
- SQLite 3.x with extension loading enabled (Homebrew's `sqlite` — the macOS system sqlite3 has extension loading disabled)
- [sqlite-vec](https://github.com/asg017/sqlite-vec) (`pip3 install sqlite-vec`)
- [Ollama](https://ollama.com) with the default models pulled (see below)

Default models can be overridden via environment variables:

| Purpose | Default | Override |
|---------|---------|----------|
| Triple extraction | `gemma3:1b` | `CRIB_EXTRACTION_MODEL` |
| Reranking | `gemma3:1b` | `CRIB_RERANK_MODEL` |
| Embeddings | `nomic-embed-text` | `CRIB_EMBEDDING_MODEL` |

Run `bin/crib doctor` to verify all prerequisites are installed and report status as JSON.

---

## License

MIT
