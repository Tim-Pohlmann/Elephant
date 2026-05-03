# Phase 5 — Benchmark harness

**Dependencies:** Phase 4 done.

**Do not skip this phase.** It is what lets you tune confidently and make defensible claims about quality.

---

## Scope

Build an automated benchmark that measures precision, recall, and F1 against labeled datasets. Tune default thresholds to meet the quality bar. Commit reproducible results.

---

## Datasets

**BigCloneBench (Java)**
- Standard academic benchmark for clone detection. Download from its public repository.
- We use it for cross-validation even though our primary target is C#. Java is close enough structurally, and it provides a large labeled set (millions of clone pairs).
- Load only a stratified sample for CI (full run is too slow): ~10,000 clone pairs + ~10,000 non-clone pairs, balanced across clone types.
- Loader: `benchmarks/bigclonebench/loader.py`.

**Hand-curated C# clone set**
- Target: ≥ 50 labeled clone pairs + ≥ 200 labeled non-clone pairs.
- Clone pairs sourced from real open-source C# repos. For each pair, label the clone type (1–4) and note the source repos.
- Non-clone pairs: randomly sampled functions from different repos that should not match.
- Committed to `benchmarks/csharp-curated/` as a JSON file with the schema below.
- **Parallelizable:** use subagents to source clone pairs from different repos. Assign each subagent 2–3 repos. Each subagent returns a JSON array of pairs. You review and merge.

**Curated dataset schema:**
```json
[
  {
    "id": "cs-001",
    "clone_type": 4,
    "unit_a": {"repo": "...", "file": "...", "qualified_name": "...", "source": "..."},
    "unit_b": {"repo": "...", "file": "...", "qualified_name": "...", "source": "..."},
    "is_clone": true,
    "notes": "optional human annotation"
  }
]
```

---

## Metrics

For each dataset, at cosine thresholds 0.75, 0.80, 0.85, 0.90:
- Precision, Recall, F1 overall.
- Precision, Recall, F1 broken down by clone type (1, 2, 3, 4).
- Index time (seconds per 1,000 units).
- Query time (milliseconds per unit).

Baseline comparison:
- Run SonarQube CPD on the same C# corpus if a Docker image is available and free. Record CPD's P/R/F1 for comparison. This is optional but valuable — skip with a note if the setup is too cumbersome.

---

## Deliverables

- `benchmarks/bigclonebench/loader.py` — downloads + caches dataset, returns labeled pairs.
- `benchmarks/csharp-curated/dataset.json` — the curated C# clone set.
- `benchmarks/run.py` — runs the full pipeline against each dataset, emits `benchmarks/results/YYYY-MM-DD.json`.
- `benchmarks/RESULTS.md` — human-readable summary of the latest run, committed.
- `benchmarks/README.md` — how to run the benchmark and how to add new datasets.

---

## Verification

1. `python benchmarks/run.py --dataset csharp-curated` completes without errors.
2. On the C# curated set at the default threshold (0.80): precision ≥ 0.85. If not, tune and re-run before closing this phase.
3. `benchmarks/RESULTS.md` is committed with numbers from a real run.
4. Results are reproducible: running the benchmark twice on the same data produces the same numbers.

**If precision is below 0.85 at any threshold:** do not close this phase. Instead:
1. Diagnose: which clone types are failing? What do false positives look like?
2. Options to try: adjust the scoring formula weights, adjust the structural verifier thresholds, adjust min-LOC, try a different ONNX model.
3. Document findings in `benchmarks/RESULTS.md` and raise with the user before making model-level changes (which would require a new ADR).
