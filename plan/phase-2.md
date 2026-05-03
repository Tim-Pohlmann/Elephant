# Phase 2 — Embedding with ONNX Runtime

**Dependencies:** Phase 1 done.

---

## Scope

Export UniXcoder to ONNX once, ship the model as a release artifact, load it via ONNX Runtime at runtime, and embed `CodeUnit` source text in batches. The output is a normalized float32 vector per unit.

---

## Model

- **Model:** `microsoft/unixcoder-base` (MIT license).
- **Why UniXcoder over GraphCodeBERT:** broader pretraining language coverage. C# is not in either model's pretraining corpus, but UniXcoder's coverage is wider overall. This is provisional — Phase 5 benchmarks will confirm or prompt a swap. See `docs/decisions/002-onnx-unixcoder.md`.
- **Export:** `tools/export_model.py` runs once by a maintainer, produces `unixcoder.onnx`. The file is published as a GitHub Release asset, not committed to git.
- **Runtime dep:** `onnxruntime` (MIT, no PyTorch). Tokenization via `tokenizers` (MIT, HuggingFace Rust crate, no PyTorch).

---

## Embedding procedure

1. Tokenize the unit's `source` field using the UniXcoder tokenizer. Truncate to 512 tokens with a warning logged if truncation occurs.
2. Run the ONNX model to get the last hidden state (shape: `[1, seq_len, hidden_dim]`).
3. Mean-pool over the sequence dimension → shape `[1, hidden_dim]`.
4. L2-normalize → unit vector.
5. Return as `float32 numpy array`, shape `[hidden_dim]`.

---

## Implementation notes

**Model download and cache:**
- On first run, download `unixcoder.onnx` from the GitHub Release URL (stored in `semdup/embed/config.py` as a constant alongside its expected SHA-256).
- Cache to `~/.cache/semdup/models/unixcoder.onnx`.
- Verify checksum after download. Raise a clear error if it fails.
- Never re-download if the cached file passes checksum.

**Embedder abstraction:**
```python
class Embedder(Protocol):
    def embed(self, sources: list[str]) -> np.ndarray: ...
    # returns shape [len(sources), hidden_dim], float32, L2-normalized
```
`UniXcoderEmbedder` is the only implementation. Future models (CodeT5+, fine-tuned variants) implement the same protocol.

**Batching:**
- Default batch size: 32. Configurable via `--batch-size`.
- Auto-detect CUDA via `onnxruntime`'s provider list; fall back to CPU silently.
- Show a `tqdm` progress bar when embedding more than 100 units.

**Export script (`tools/export_model.py`):**
- Requires PyTorch and `transformers` (only at export time — not runtime deps).
- Exports with `torch.onnx.export`, opset 14.
- Validates the exported model by comparing a sample output against the PyTorch output (max abs diff < 1e-4).
- Prints the SHA-256 of the output file (for updating `config.py`).

---

## Deliverables

- `tools/export_model.py` with a `README` note on how to re-export.
- `semdup/embed/config.py` — model URL, expected SHA-256, cache path.
- `semdup/embed/download.py` — download + checksum verification.
- `semdup/embed/embedder.py` — `Embedder` protocol + `UniXcoderEmbedder`.
- `semdup/embed/__init__.py` — exports `UniXcoderEmbedder`.
- `tests/unit/test_embed.py` — see test plan below.

**Test plan:**
- Determinism: embed the same string twice with a fixed ONNX session, assert bit-identical output.
- Batch/single equivalence: `embed([a, b])` == `[embed([a])[0], embed([b])[0]]`.
- L2 normalization: `np.linalg.norm(vector)` ≈ 1.0 for all outputs.
- Clone similarity: two hand-written C# methods that compute the same thing (e.g., iterative factorial vs recursive factorial) should score cosine similarity > 0.80.
- Non-clone distance: two unrelated methods should score < 0.70.
- Truncation warning: a source string longer than 512 tokens should emit a warning (capture via `pytest`'s log capture).
- Model mock: tests must not download the real model. Use a small random ONNX model as a fixture, injected via the `Embedder` protocol. Only the similarity tests require the real model and are marked `@pytest.mark.slow` — excluded from default CI runs.

---

## Verification

1. `pytest tests/unit/test_embed.py` passes (excluding slow tests).
2. With the real model: embed 1,000 C# units from the Phase 1 corpus. Measure throughput — target ≥ 50 units/sec on CPU. Report the actual number.
3. Spot-check cosine similarities: pick 3 known-similar pairs and 3 known-different pairs, report their scores.
