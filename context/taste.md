---
description: "Crib domain taste — accumulated preferences for the memory system"
events: [PreToolUse, UserPromptSubmit]
always: true
last_reviewed: "2026-03-14"
review_interval: 30
---

## Crib domain taste

Accumulated preferences for work within the crib (memory) repository.

### Architecture
- Three-channel retrieval: fact triples (SQL JOINs), full-text (FTS5), vector (sqlite-vec).
- Single SQLite database file. No external services except API calls for embeddings.
- Triple extraction on write via Anthropic API.
- Stale facts are superseded, not appended.

### Interface
- `crib write` reads from stdin, writes to database, extracts triples.
- `crib retrieve` reads query from stdin, returns formatted results on stdout.
- `crib maintain` handles correction linking and decay.
- `crib doctor` reports health as JSON.

### Quality
- Memory entries have types: decision, correction, error, note.
- Triples follow (subject, predicate, object) structure.
- Vector embeddings via Voyage AI API.
