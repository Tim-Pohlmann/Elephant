# Phase 3 — Indexing and incremental updates

**Dependencies:** Phases 1 and 2 done.

---

## Scope

Persist embeddings in a FAISS index and unit metadata in SQLite. Support incremental re-indexing so unchanged files are not re-embedded on subsequent runs. Wire up the full CLI.

---

## Storage layout

Index directory: `.semdup/` in the scanned repo root.

```
.semdup/
  index.faiss      # FAISS IndexFlatIP, L2-normalized vectors
  metadata.db      # SQLite — see schema below
```

**SQLite schema:**

```sql
CREATE TABLE files (
    path        TEXT PRIMARY KEY,
    sha256      TEXT NOT NULL,
    mtime       REAL NOT NULL,
    last_indexed TEXT NOT NULL   -- ISO 8601 timestamp
);

CREATE TABLE units (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path       TEXT NOT NULL REFERENCES files(path),
    qualified_name  TEXT NOT NULL,
    start_line      INTEGER NOT NULL,
    end_line        INTEGER NOT NULL,
    loc             INTEGER NOT NULL,
    ast_fingerprint TEXT NOT NULL,
    vector_id       INTEGER NOT NULL  -- row index in FAISS index
);

CREATE INDEX idx_units_file ON units(file_path);
CREATE INDEX idx_units_fingerprint ON units(ast_fingerprint);
```

---

## Incremental update logic

On each `semdup index` run:

1. Walk the target directory, collect all source files for indexed languages.
2. For each file: compute SHA-256 + mtime.
3. Compare against the `files` table:
   - **New file:** extract + embed + insert into `units` + add vectors to FAISS + insert into `files`.
   - **Changed file** (sha256 differs): delete existing `units` rows for this file, remove their vectors from FAISS (mark as deleted — see FAISS removal note), re-extract + re-embed + reinsert.
   - **Deleted file:** delete `units` rows, mark vectors deleted, delete from `files`.
   - **Unchanged file** (sha256 matches): skip entirely. Zero embedding calls.
4. After processing all files, compact the FAISS index if the deletion ratio exceeds 10%.

**FAISS removal note:** `IndexFlatIP` doesn't support in-place removal. Maintain a `deleted_ids` set in memory (persisted as a column or separate table). Filter deleted IDs from query results. Compact by rebuilding the index from scratch during the compaction step.

---

## CLI

```
semdup index <path>            # index or re-index a directory
semdup query <path>            # run a duplicate query on an already-indexed directory
  [--file F]                   # restrict query to units from file F
  [--threshold T]              # cosine similarity threshold (default 0.80)
  [--top-k K]                  # neighbors per unit (default 10)
  [--min-loc N]                # minimum LOC filter (default 20)
  [--format {json,text}]       # output format (default json)
semdup run <path>              # index then query in one step (same flags as query)
```

---

## Deliverables

- `semdup/index/store.py` — `IndexStore` class wrapping FAISS + SQLite.
- `semdup/index/incremental.py` — file diffing, change detection logic.
- `semdup/index/__init__.py` — clean public API.
- Full CLI wiring in `semdup/cli.py`.
- `semdup.toml` schema stub in `semdup/config.py` (thresholds, min-loc, batch size, language list).
- `tests/unit/test_index.py` — see test plan below.

**Test plan:**
- Index a fixture corpus of 20 units. Query each unit, assert self is top-1 result.
- Incremental no-op: index once, record embedding call count (0 calls expected on second run with no file changes).
- Incremental partial update: modify one fixture file, re-index, assert only that file's units were re-embedded.
- Deletion: remove a fixture file, re-index, assert its units are gone from query results.
- Compaction: force a compaction, assert index integrity is maintained.

---

## Verification

1. `pytest tests/unit/test_index.py` passes.
2. Index a real C# repo (~500 files). Re-run with no changes → completes in under 5 seconds with zero embedding calls. Confirm by logging embedding call count.
3. Modify a single file in the repo, re-index → only that file's units re-embedded. Confirm via logs.
