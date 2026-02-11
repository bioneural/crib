<h1 align="center">
  c r i b
  <br>
  <sub>automatic memory for Claude Code agents</sub>
</h1>

Agents forget between sessions. Decisions made yesterday are relitigated today. Context that shaped a choice evaporates before the next prompt arrives. Crib makes memory structural — one SQLite database with three query channels, automatic retrieval on every prompt, and consolidation-on-write so the archive stays clean.

A single script. No gems. No external services. No API keys.

---

## How it works

Crib stores memory in an SQLite database with three query channels:

| Channel | Mechanism | Use case |
|---------|-----------|----------|
| Fact triples | SQL JOINs on `(subject, predicate, object)` | "What are ClientA's payment terms?" |
| Full-text | FTS5 with Porter stemming | "What did we decide about caching?" |
| Vector | sqlite-vec nearest neighbors | Semantic similarity when keywords fail |

**Write path** — text arrives on stdin. Crib stores it as a full-text entry, extracts fact triples via ollama, and performs consolidation-on-write: if a new fact supersedes an existing one, the old relation is marked `valid_until` rather than duplicated. The archive stays clean without manual curation.

**Read path** — a prompt arrives on stdin. Crib extracts keywords, queries all three channels, merges and deduplicates results, reranks with ollama if over budget, and wraps the output in `<memory>` tags. Empty output means nothing relevant was found. Fail-open — a broken retrieval never blocks the agent.

---

## Usage

```sh
# Store a memory
echo "Chose SQLite for storage. Zero dependencies." | bin/crib write

# Store with a type prefix
echo "type=decision Chose SQLite for storage." | bin/crib write

# Retrieve relevant context for a prompt
echo "what database did we choose?" | bin/crib retrieve
# <memory>
# ## Known facts
# - SQLite → storage_backend → crib
# ## Memory entries
# [decision] Chose SQLite for storage. Zero dependencies.
# </memory>

# Initialize the database explicitly (happens automatically on first write)
bin/crib init

# Print the schema
bin/crib schema
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
| `CRIB_DB` | `memory/crib.db` | Path to the SQLite database |
| `CRIB_EXTRACTION_MODEL` | `gemma3:1b` | Ollama model for triple extraction |
| `CRIB_RERANK_MODEL` | `gemma3:1b` | Ollama model for reranking |
| `CRIB_EMBEDDING_MODEL` | `nomic-embed-text` | Ollama model for embeddings |

---

## Integration with hooker

Crib is designed as a [hooker](https://github.com/bioneural/hooker) context command target. A policy wires retrieval to every prompt:

```ruby
policy "Memory" do
  on :UserPromptSubmit
  inject command: "bin/crib retrieve"
end
```

Hooker pipes the user's prompt to stdin and injects stdout as additional context. Empty stdout means nothing is injected. Write calls happen from the agent via shell:

```sh
echo "type=decision Switched from Postgres to SQLite" | bin/crib write
```

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
```

Triples use bi-temporal validity — `valid_from` / `valid_until` on relations. A query for current facts filters on `valid_until IS NULL`. Old facts are superseded, not deleted.

---

## Prerequisites

- Ruby 3.x (ships with macOS)
- SQLite 3.x (ships with macOS)
- [Ollama](https://ollama.com) with `gemma3:1b` pulled (for triple extraction and reranking)

Vector embeddings require `nomic-embed-text` pulled and sqlite-vec loaded. Without sqlite-vec, the vector channel is skipped and the other two channels still function.

---

## License

MIT
