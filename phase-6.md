# Phase 6 — Multi-language expansion

**Dependencies:** Phase 5 done (benchmarks must be passing before adding languages).

---

## Scope

Add Python, TypeScript, and Java support. Each language follows the same pattern: a `LanguageAdapter` implementation, fixtures, golden tests, and a language-specific entry in the benchmark. Thresholds are tuned per language.

---

## Per-language work

For each language (Python, TypeScript, Java), in order:

1. **Pin the tree-sitter grammar** version in `pyproject.toml`.
2. **Implement `LanguageAdapter`** in `semdup/extract/<language>.py`. Register it in the language registry (`semdup/extract/__init__.py`).
3. **What to extract** — language-specific guidance below.
4. **Add fixtures** to `tests/fixtures/<language>/` with golden NDJSON output. Minimum 10 fixture files covering the same kinds of edge cases as Phase 1 (nested scopes, lambdas, short bodies that should be skipped, parse errors).
5. **Add golden tests** in `tests/unit/test_extract_<language>.py`.
6. **Add a language-specific clone set** to `benchmarks/` with at least 20 labeled pairs.
7. **Tune thresholds**: run the benchmark for this language. If the optimal threshold differs from the C# default, add a language-specific override to `semdup.toml` defaults.
8. **Commit** with `[phase-6-<language>]` prefix.

**Parallelizable:** after Python is done and its adapter is working (use it to validate the adapter pattern holds), TypeScript and Java can be done by subagents in parallel. Each subagent gets steps 1–7 for its language. Review before committing.

---

## What to extract per language

**Python:**
- Functions (`def`), async functions (`async def`).
- Methods, class methods, static methods, property getters/setters.
- Lambdas are too short to be useful — skip unless LOC ≥ min-loc threshold.
- Exclude: abstract methods (decorated with `@abstractmethod`), pass-only bodies.

**TypeScript:**
- Function declarations, function expressions, arrow functions (when assigned to a variable/property).
- Class methods (including async, static, abstract — but skip abstract with no body).
- Interface method signatures — skip (no body).
- Generic functions: include, the source text carries the generic params.

**Java:**
- Method declarations, constructor declarations.
- Lambda expressions assigned to a field.
- Skip: abstract methods, interface methods without default body.

---

## Qualified name conventions

| Language | Format |
|----------|--------|
| Python | `module.Class.method` (use the relative import path as module) |
| TypeScript | `file.Class.method` (use the relative file path without extension) |
| Java | `com.example.package.Class.method` (from package declaration) |

---

## Deliverables

- `semdup/extract/python.py`, `typescript.py`, `java.py`
- Fixtures + golden tests for each language
- `benchmarks/python-curated/`, `benchmarks/typescript-curated/`, `benchmarks/java-curated/`
- Updated `benchmarks/RESULTS.md` with per-language results
- Updated `semdup/config.py` with per-language threshold defaults where they differ from C#

---

## Verification

For each language:
1. Golden-file tests pass.
2. Benchmark precision ≥ 0.85 and recall ≥ 0.70 at the tuned threshold.
3. If UniXcoder performs poorly on a language (F1 < 0.75 even after threshold tuning), document it in `benchmarks/RESULTS.md` and raise with the user — do not proceed silently.
