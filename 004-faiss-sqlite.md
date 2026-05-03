# ADR 004 — FAISS + SQLite for index and metadata

**Status:** accepted  
**Phase:** 0

---

## Context

semdup needs to persist embeddings and unit metadata between runs, support nearest-neighbor queries, and support incremental updates (re-embed only changed files).

Options for vector storage:
- **FAISS (MIT):** battle-tested ANN library from Meta. `IndexFlatIP` is exact (not approximate), which is correct for moderate corpora (<1M vectors). HNSW is available as a drop-in for larger corpora. Python bindings are first-class.
- **HNSWlib (Apache 2.0):** faster than FAISS for HNSW specifically, simpler API. Less flexible — no flat/exact mode.
- **Chroma / Qdrant / Weaviate:** full vector database products. More features than we need, heavier dependencies, some have licensing complexity.

Options for metadata:
- **SQLite:** zero-dependency relational storage, excellent Python support via the standard library. Queryable. Persistent. Handles the file-hash tracking and unit metadata with ease.
- **JSON files:** simpler but not queryable. Gets unwieldy as corpus size grows.

## Decision

FAISS `IndexFlatIP` + SQLite. FAISS handles vector search; SQLite handles everything else (file tracking, unit metadata, incremental-update bookkeeping).

## Consequences

**Positive:**
- FAISS and SQLite are both MIT. No licensing risk.
- `IndexFlatIP` is exact — no recall loss from approximation at typical corpus sizes.
- SQLite is in the Python standard library (`sqlite3`). Zero extra dependency.
- Switching to HNSW later (larger corpora) is a one-line change in the FAISS wrapper.

**Negative:**
- FAISS does not support in-place deletion. We must maintain a deleted-IDs set and compact periodically. This adds implementation complexity in Phase 3.
- FAISS index files are not human-readable. Debugging requires tooling.
