# ADR 001 — Parsing via tree-sitter

**Status:** accepted  
**Phase:** 0

---

## Context

semdup needs to extract function-like units from source files. The options are:

- **Language-specific parsers** (Roslyn for C#, Javac for Java, etc.): highest semantic fidelity — type resolution, symbol binding, cross-file analysis. But each parser is a separate dependency, each has its own API, and several have licensing or platform constraints (Roslyn is .NET-only).
- **tree-sitter:** a uniform incremental parsing library with grammars for 100+ languages. All grammars are MIT or Apache 2.0 licensed. Runs on all platforms via a C library with Python bindings (`py-tree-sitter`). Does not provide semantic resolution — only syntax trees.
- **Regex/heuristic extraction:** fast and simple but fragile. Not acceptable for a tool that claims precision.

## Decision

Use tree-sitter for all language parsing. Define a `LanguageAdapter` protocol so each language is a thin adapter over the tree-sitter grammar, and all downstream code works with the uniform `CodeUnit` schema.

## Consequences

**Positive:**
- Single parsing dependency, one API, one way to add new languages.
- All grammar licenses are permissive — no licensing risk.
- Runs on Linux, macOS, Windows without language runtimes.
- Incremental parsing is available if we ever need on-save performance (Phase 9 future optimization).

**Negative:**
- No type resolution. We cannot determine if two functions have the same signature at the type level — only at the textual level. This is acceptable for duplication detection, which operates on source text and AST structure.
- Grammar quality varies. For obscure languages, the grammar may be incomplete. Validate each grammar before shipping a language adapter.
- The C# tree-sitter grammar may lag behind the latest C# version. Pin the grammar version and test against the C# features we care about in Phase 1 fixtures.
