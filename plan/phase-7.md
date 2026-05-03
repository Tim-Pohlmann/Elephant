# Phase 7 — Standalone binary + LLM verifier

**Dependencies:** Phase 6 done.

---

## Scope

Package semdup as single-file binaries for Linux, macOS, and Windows. Implement the optional LLM verifier that runs on borderline candidate pairs when an Anthropic API key is available.

---

## Standalone binary

**Tool:** PyInstaller, `--onefile` mode.

**Key concerns:**
- ONNX Runtime and tree-sitter grammars must be correctly bundled (both have native extensions). Add explicit `--collect-all` and `--hidden-import` directives as needed.
- Model file (`unixcoder.onnx`) is NOT bundled — it's too large. The binary downloads and caches it on first run, same as the library.
- `semdup.spec` lives in the repo root. All PyInstaller config goes there, not on the command line.

**CI build matrix:**
- Trigger: push a git tag matching `v*`.
- Matrix: `ubuntu-latest`, `macos-latest`, `windows-latest`.
- Each job: `pip install pyinstaller`, `pyinstaller semdup.spec`, upload artifact to the GitHub Release.
- Artifact names: `semdup-linux-x86_64`, `semdup-macos-arm64`, `semdup-windows-x86_64.exe`.

**Smoke test job** (runs after all three builds succeed):
- Downloads each binary on a fresh runner of the matching OS.
- Runs `semdup run tests/fixtures/csharp/` on a small fixture corpus.
- Asserts exit code 0 and non-empty JSON output.

---

## LLM verifier

Implements the `Verifier` protocol from Phase 4. Replaces the stub in `semdup/verify/llm.py`.

**When it runs:**
- Only on borderline pairs: `similarity_lower ≤ embedding_sim ≤ similarity_upper` (defaults: 0.75–0.90). High-confidence pairs skip it; the structural verifier is sufficient.
- Only when `ANTHROPIC_API_KEY` is set (or when running inside Claude Code — see Phase 8).
- Respects a `--llm-budget N` flag (max API calls per `semdup run`). Default: 50. Raises a warning when the budget is exhausted; remaining borderline pairs are classified by the structural verifier only.

**API contract:**
- Model: `claude-haiku-*` (cheapest, fast enough for this use case).
- Prompt: two code snippets + the question "Are these functionally equivalent or do they represent the same logical operation? Answer JSON: `{verdict: bool, clone_type: int, confidence: 'high'|'medium'|'low', reasoning: str}`".
- Parse the JSON response. If parsing fails, log a warning and fall back to structural result.
- Retry once on rate limit (429), then fall back.

**Tests:**
- Mock the Anthropic client. Test: verdict=true case, verdict=false case, JSON parse failure (fall back), rate limit retry, budget exhaustion.
- Integration test (marked `@pytest.mark.slow`, excluded from default CI): call the real API with a known clone pair and assert verdict=true.

---

## Deliverables

- `semdup.spec` — PyInstaller spec file.
- `.github/workflows/release.yml` — build + upload binary on tag.
- Smoke test jobs in the release workflow.
- `semdup/verify/llm.py` — full `LLMVerifier` implementation.
- `tests/unit/test_llm_verifier.py` — mocked tests.
- `docs/usage.md` updated with binary install instructions and LLM verifier cost/opt-out docs.

---

## Verification

1. `pytest tests/unit/test_llm_verifier.py` passes (mocked).
2. Tag a release. All three binaries appear as GitHub Release assets.
3. On a fresh runner with no Python installed, download and run the binary against `tests/fixtures/csharp/`. Exit code 0, non-empty output.
4. With `ANTHROPIC_API_KEY` set and `--llm-budget 5`: run against a fixture with 3 borderline pairs. Assert exactly 3 API calls made (≤ budget), results are more precise than structural-only.
