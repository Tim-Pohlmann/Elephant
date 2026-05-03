# semdup — Project Plan

**Project:** A semantic code duplication detector that finds Type 4 (functionally similar) clones across languages, distributed as a standalone binary and a Claude Code plugin with an embedded LSP server for inline diagnostics.

**License posture:** MIT. All runtime dependencies must be MIT, Apache 2.0, or BSD. No GPL/LGPL/commercial deps.

**How work is run:** see `PROCESS.md`. Read it every session before acting.
**Current progress:** see `STATUS.md`. It is the single source of truth for which phase is active.
**Phase details:** see `docs/phases/phase-N.md` for the active phase only.
**Decisions:** see `docs/decisions/` when a decision is challenged or a new one is needed.

---

## Goals

- Detect Type 1–4 code clones with higher precision and recall than token-based tools (SonarQube CPD, PMD CPD).
- Target all mainstream languages eventually; ship C# first.
- Run on Linux, macOS, and Windows.
- Ship as: a Python package, a single-file standalone binary, and a Claude Code plugin with LSP server.
- Zero proprietary dependencies at build or runtime.

## Non-goals (v1)

- IDE plugins beyond Claude Code.
- Real-time on-keystroke checking (diagnostics fire on save).
- A hosted service.
- Refactoring auto-apply.

---

## Repository layout

```
semdup/
  packages/
    core/
      src/semdup/
        extract/       # tree-sitter extraction, per-language adapters
        embed/         # ONNX embedder
        index/         # FAISS + SQLite
        verify/        # structural + LLM verifiers
        report/        # clustering, ranking, formatting
        lsp/           # LSP server (Phase 9)
        cli.py
      tests/
        fixtures/
        unit/
        integration/
      pyproject.toml
    plugin/
      plugin.json      # Claude Code plugin manifest
      commands/
      README.md
  benchmarks/
    bigclonebench/
    csharp-curated/
    run.py
    RESULTS.md
  docs/
    decisions/         # ADRs — one file per decision
    phases/            # Phase detail files — read only the active one
    usage.md
    contributing.md
  tools/
    export_model.py    # UniXcoder -> ONNX exporter (run once)
  .github/workflows/
  PLAN.md              # this file — goals, layout, decision and phase indexes
  PROCESS.md           # how we work — read every session
  STATUS.md            # progress tracker — updated every phase
  README.md
  LICENSE
```

---

## Architectural decisions index

Full rationale in `docs/decisions/`. Summary here for orientation.

| # | Decision | File |
|---|----------|------|
| 1 | Parsing via tree-sitter (not language-specific parsers) | `docs/decisions/001-tree-sitter.md` |
| 2 | Embedding via UniXcoder exported to ONNX | `docs/decisions/002-onnx-unixcoder.md` |
| 3 | Engine in Python | `docs/decisions/003-python-engine.md` |
| 4 | FAISS + SQLite for index and metadata | `docs/decisions/004-faiss-sqlite.md` |
| 5 | Two-stage verifier: structural always-on, LLM optional | `docs/decisions/005-verifier.md` |
| 6 | Three distribution artifacts from one core | `docs/decisions/006-distribution.md` |

---

## Phase index

| Phase | Title | Detail |
|-------|-------|--------|
| 0 | Repo skeleton and decisions | `docs/phases/phase-0.md` |
| 1 | Tree-sitter extraction for C# | `docs/phases/phase-1.md` |
| 2 | Embedding with ONNX Runtime | `docs/phases/phase-2.md` |
| 3 | Indexing and incremental updates | `docs/phases/phase-3.md` |
| 4 | Verification, ranking, reporting | `docs/phases/phase-4.md` |
| 5 | Benchmark harness | `docs/phases/phase-5.md` |
| 6 | Multi-language expansion | `docs/phases/phase-6.md` |
| 7 | Standalone binary + LLM verifier | `docs/phases/phase-7.md` |
| 8 | Claude Code plugin (slash commands) | `docs/phases/phase-8.md` |
| 9 | LSP server for inline diagnostics | `docs/phases/phase-9.md` |
