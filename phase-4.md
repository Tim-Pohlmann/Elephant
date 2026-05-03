# Phase 4 — Verification, ranking, and reporting

**Dependencies:** Phase 3 done.

---

## Scope

Turn raw FAISS candidate pairs into a trusted, ranked report. This phase adds the structural verifier, the scoring pipeline, union-find clustering, and the two output formatters. Quality is the goal — this is where semdup earns the right to claim it beats SonarQube.

---

## Pipeline

For each `CodeUnit` in the index:

1. **Candidate retrieval:** query FAISS for top-k neighbors (default k=10) above the cosine threshold (default 0.80). Filter: cross-file only by default (`--include-same-file` to override), minimum LOC.
2. **Structural verification:** for each candidate pair, run `StructuralVerifier` (see below).
3. **Scoring:** combine embedding similarity + structural score into a final `clone_score` (0–1). See scoring formula below.
4. **Deduplication:** normalize pairs so (A, B) == (B, A). Remove self-pairs.
5. **Clustering:** union-find over the pair graph. Each cluster = a set of mutually similar units.
6. **Ranking:** sort clusters by `cluster_score` descending. `cluster_score` = mean `clone_score` of all pairs in the cluster, weighted by total LOC.

---

## StructuralVerifier

```python
class Verifier(Protocol):
    def verify(self, a: CodeUnit, b: CodeUnit, embedding_sim: float) -> VerificationResult: ...

@dataclass
class VerificationResult:
    clone_type: Literal[1, 2, 3, 4]  # best estimate
    structural_score: float           # 0–1, 1 = identical structure
    confidence: str                   # "high" | "medium" | "low"
    notes: str                        # human-readable explanation
```

`StructuralVerifier` logic:
- If `a.ast_fingerprint == b.ast_fingerprint`: Type 1 or 2, `structural_score = 1.0`, confidence = "high".
- Else: compute normalized tree edit distance (TED) between the two ASTs.
  - Use the APTED algorithm or Zhang-Shasha. Pick the one available as a permissively licensed Python package; document the choice in an ADR if one is clearly preferable.
  - `structural_score = 1.0 - (ted / max(size_a, size_b))` where size = node count.
  - If `structural_score > 0.7`: Type 3, confidence = "medium".
  - Else: Type 4 (semantic only), confidence = "low" (rely on embedding).
- Final `clone_score = 0.6 * embedding_sim + 0.4 * structural_score`.

**LLM verifier hook:** `semdup/verify/llm.py` must exist as a stub that implements the `Verifier` protocol and raises `NotImplementedError`. Phase 7 fills it in. The pipeline must already accept it as a drop-in replacement.

---

## Output formats

**`--format json`** (default): a JSON object with:
```json
{
  "semdup_version": "...",
  "scanned_at": "...",
  "threshold": 0.80,
  "clusters": [
    {
      "id": "cluster-001",
      "cluster_score": 0.92,
      "total_loc": 84,
      "units": [...],
      "pairs": [{"unit_a": "...", "unit_b": "...", "clone_score": 0.92, "clone_type": 3, "confidence": "medium"}]
    }
  ]
}
```

**`--format text`**: human-readable, grouped by cluster. Example:
```
Cluster 001  score=0.92  3 units  84 LOC
  src/Foo.cs:42  Namespace.Foo.CalculateTotal   (28 LOC)
  src/Bar.cs:17  Namespace.Bar.ComputeSum        (30 LOC)
  src/Baz.cs:99  Namespace.Baz.SumItems          (26 LOC)
  Type 3 clone · medium confidence
```

**`--format sarif`**: leave a `TODO` stub with the schema link. Do not implement yet.

---

## Deliverables

- `semdup/verify/structural.py` — `StructuralVerifier`.
- `semdup/verify/llm.py` — stub `LLMVerifier`.
- `semdup/verify/__init__.py` — `Verifier` protocol, `VerificationResult`.
- `semdup/report/cluster.py` — union-find, ranking.
- `semdup/report/format_json.py` and `format_text.py`.
- `semdup/report/__init__.py`.
- `tests/unit/test_verify.py` — structural verifier unit tests.
- `tests/unit/test_report.py` — clustering + formatting tests with golden outputs.
- `tests/integration/test_pipeline.py` — end-to-end on the Phase 1 C# fixtures, seeded with known clone pairs.

**Integration test clone pairs (C#):**
Write at least 5 Type 4 clone pairs as fixture files. Suggested pairs:
1. Iterative factorial vs recursive factorial
2. LINQ `Sum()` vs manual for-loop accumulator
3. Two different moving-average implementations (sliding window vs cumulative)
4. `Dictionary`-based frequency counter vs `List`-based sort approach
5. Manual string CSV formatter vs `string.Join`-based

The integration test asserts: all 5 pairs appear in the output above threshold; no unrelated function pairs appear.

---

## Verification

1. `pytest tests/unit/test_verify.py tests/unit/test_report.py` passes.
2. `pytest tests/integration/test_pipeline.py` passes — all 5 clone pairs detected, 0 false-positive pairs among the unrelated functions.
3. Run `semdup run` against a real C# repo with known duplication (pick one with a public SonarQube report if possible). Top 20 clusters should all be genuine duplication on eyeball review.
