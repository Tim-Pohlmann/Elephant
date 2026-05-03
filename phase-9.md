# Phase 9 — LSP server for inline diagnostics

**Dependencies:** Phase 8 done.

---

## Scope

An LSP server that the Claude Code plugin declares in its `lspServers` block. Duplication warnings appear as inline diagnostics (the same way type errors do) on file save.

---

## Before starting this phase

**Re-read the current LSP plugin docs at `code.claude.com/docs`.** The `lspServers` manifest schema has been evolving. The schema below is correct as of the plan's writing — but the docs win if they differ. Note any discrepancies in a new ADR.

---

## How Claude Code LSP plugins work

The plugin's `plugin.json` (or a sibling `.lsp.json`) declares:
```json
{
  "lspServers": {
    "semdup": {
      "command": "semdup-lsp",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".cs": "csharp",
        ".py": "python",
        ".ts": "typescript",
        ".java": "java"
      }
    }
  }
}
```

Claude Code starts `semdup-lsp` as a subprocess and communicates via standard LSP over stdio. It consumes `textDocument/publishDiagnostics` messages and shows them as inline squiggles/warnings.

---

## LSP capabilities to implement

| Capability | Notes |
|-----------|-------|
| `initialize` | Warm-load the `.semdup/` index from the workspace root. If no index exists, respond with a hint diagnostic on the first opened file. |
| `textDocument/didOpen` | No-op (index is pre-built). |
| `textDocument/didSave` | Re-extract + re-embed the saved file, query for neighbors, publish diagnostics. |
| `textDocument/didClose` | Clear diagnostics for the file. |
| `textDocument/publishDiagnostics` | One diagnostic per clone cluster involving a unit from the saved file. Severity: Warning. Message: "Possible duplicate of `qualified_name` in `file:line` (score: 0.87, Type 3)". |
| `textDocument/codeAction` | Offer one code action per diagnostic: "Show semdup cluster details" — invokes `/semdup explain <cluster-id>` in the Claude Code terminal. |

**Graceful degradation:** if the workspace has no `.semdup/` index, the server should not crash or spam diagnostics. It should publish a single informational diagnostic on any opened file: "semdup: no index found. Run `semdup index <path>` to enable duplicate detection." The diagnostic clears when an index appears.

**Performance:** `didSave` processing must complete in under 10 seconds on a file with up to 50 functions, on a 500-unit indexed codebase, on a laptop CPU. The re-embed step is the bottleneck — only re-embed the saved file, don't re-run the full index.

---

## Implementation notes

- Use `pygls` (MIT) for LSP scaffolding. It handles the JSON-RPC protocol and threading.
- `semdup.lsp.server` is the entry point. It imports `semdup.index` and `semdup.embed` as libraries — no subprocess, no duplication of logic.
- The server is single-process. Embedding runs in a threadpool executor to avoid blocking the LSP event loop.
- Diagnostics are keyed by `(file, cluster_id)` so they can be cleared cleanly on `didClose` or when a re-save produces no duplicates.

---

## Binary

- `semdup-lsp` is a second PyInstaller entry point, built alongside `semdup` in the existing release workflow.
- The plugin manifest updated to include the `lspServers` block pointing to `semdup-lsp`.

---

## Deliverables

- `semdup/lsp/server.py` — `pygls`-based LSP server.
- `semdup/lsp/__init__.py`.
- Second PyInstaller entry point + updated `semdup.spec`.
- Updated `packages/plugin/plugin.json` with `lspServers` entry.
- `tests/integration/test_lsp.py` — fires up the server over stdio, sends a sequence of LSP messages (initialize → didOpen → didSave), asserts diagnostics are published for a known-duplicate file and empty for a non-duplicate file.

---

## Verification

1. `pytest tests/integration/test_lsp.py` passes.
2. Install the updated plugin in a real Claude Code session.
3. Open a C# repo with a known duplicate function pair. Save one of the files. Within 10 seconds, a warning diagnostic appears inline on the duplicate unit. The message correctly identifies the other file and line.
4. Fix the duplication (delete one of the functions). Save. Diagnostic clears.
