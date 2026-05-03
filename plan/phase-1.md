# Phase 1 — Tree-sitter extraction for C#

**Dependencies:** Phase 0 done.

---

## Scope

Parse C# source files using tree-sitter, extract function-like units into a stable schema, and compute a normalized AST fingerprint for each unit. The fingerprint is used in Phase 4 for structural comparison.

---

## CodeUnit schema

```python
@dataclass
class CodeUnit:
    language: str           # e.g. "csharp"
    file: str               # absolute path
    qualified_name: str     # Namespace.Type.Member
    start_line: int
    end_line: int
    source: str             # full source text including signature
    loc: int                # non-blank, non-comment lines in body only
    ast_fingerprint: str    # SHA-256 of normalized AST node-kind sequence
```

This schema is the contract between extraction and every downstream phase. Do not change it without an ADR.

---

## What to extract (C#)

**Include:**
- Method declarations
- Constructor and destructor declarations
- Local function statements
- Property accessors with a body (block-bodied or expression-bodied `get`/`set`/`init`)
- Lambda expressions assigned to a field or property initializer (only if LOC ≥ min-loc threshold, default 5)

**Exclude:**
- Abstract method declarations (no body)
- Interface method declarations without a default body
- Partial method declarations without an implementing body
- Auto-properties with no accessor body

---

## Implementation notes

**Parsing:**
- Use `py-tree-sitter` and `tree-sitter-c-sharp`. Pin the grammar version in `pyproject.toml`.
- Define a `LanguageAdapter` protocol in `semdup/extract/base.py`. `CSharpAdapter` is the first implementation. New languages in Phase 6 implement the same protocol — design it with that in mind.

**Qualified name:**
- Walk ancestor nodes to build the chain: file-scoped namespace → namespace → type (→ nested type) → member.
- Use `.` as separator throughout, including nested types.
- For constructors use `Type..ctor`; for destructors use `Type..dtor`.

**AST fingerprint:**
- Walk the subtree for the function body only (not the signature).
- Emit only node *kinds* (no text content), depth-first, as a space-separated string.
- Hash the string with SHA-256, hex-encoded.
- This makes the fingerprint invariant to identifier renames and literal changes (Type 2 clones hash identically).

**LOC:**
- Count lines in the body only (not the signature line).
- Exclude lines that are blank or consist solely of `//` or `/* */` comments.

**Error handling:**
- Parse errors: log file path + error to stderr, skip the file, continue.
- Encoding errors: treat file as UTF-8; if decoding fails, skip + warn.

**CLI:**
- `semdup extract <path>` — walks the directory, emits one NDJSON object per unit to stdout.
- `--language` flag defaults to auto-detect by file extension; override with `--language csharp`.
- `--min-loc` flag (default 5) — skip units below this threshold.

---

## Deliverables

- `semdup/extract/base.py` — `LanguageAdapter` protocol, `CodeUnit` dataclass, `ExtractionError`.
- `semdup/extract/csharp.py` — `CSharpAdapter`.
- `semdup/extract/__init__.py` — exports `extract(path, language, min_loc) -> list[CodeUnit]`.
- CLI wiring in `semdup/cli.py` for the `extract` subcommand.
- `tests/fixtures/csharp/` — at minimum one file per edge case below. Golden NDJSON output committed alongside each fixture.
- `tests/unit/test_extract_csharp.py` — golden-file tests; one test per fixture.

**Required fixtures (C# edge cases):**
1. Simple method in a namespace + class
2. Nested namespace
3. Nested type (class within class)
4. Local function inside a method
5. Expression-bodied method
6. Property with block-bodied get/set
7. Property with expression-bodied accessor
8. Constructor and destructor
9. Lambda assigned to a field
10. File-scoped namespace (C# 10+)
11. Record with methods
12. Partial class (two files, one method each — both should be extracted)
13. Abstract method (should NOT appear in output)
14. Interface method without default body (should NOT appear in output)
15. File with a syntax error (should produce a warning, not a crash)

**Parallelizable:** fixture creation. Use subagents: assign each subagent 3–4 fixtures from the list above. Each subagent produces the `.cs` file and its expected `.ndjson` output. Review output before committing.

---

## Verification

1. Golden-file tests all pass (`pytest tests/unit/test_extract_csharp.py`).
2. Running `semdup extract <path>` on a clone of `dotnet/runtime/src/libraries/System.Text.Json` produces non-zero output with no crashes. Eyeball 10 random units — qualified names and source text should look correct.
3. Running the extractor twice on the same input produces byte-identical output.
4. A file with a deliberate syntax error produces a stderr warning and does not halt extraction of other files.
