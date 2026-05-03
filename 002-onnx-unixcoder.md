# ADR 002 — Embedding via UniXcoder exported to ONNX

**Status:** accepted (provisional — subject to revision after Phase 5 benchmark)  
**Phase:** 0

---

## Context

semdup needs to produce embeddings for code units so semantically similar code maps to nearby vectors. The options are:

- **Hosted embedding APIs** (OpenAI, Voyage, Cohere): highest quality, but require network access, cost money per call, and send user code to a third party. Ruled out by the "no proprietary dependencies" constraint.
- **GraphCodeBERT** (MIT): pretrained on Python, Java, JavaScript, PHP, Go, Ruby. C# is not in the training corpus. Would generalize to C# via Java similarity, but quality is uncertain.
- **UniXcoder** (MIT): pretrained on a broader set including more language types. Marginally better cross-language generalization than GraphCodeBERT in published benchmarks.
- **CodeT5+** (Apache 2.0): larger, higher quality, but the base model is 220M params vs UniXcoder's 125M — slower CPU inference, harder to bundle.
- **Fine-tuned small model:** best quality for a specific language set, but requires a labeled training set we don't have yet. Possible v2 option after Phase 5 generates labeled data.

**ONNX vs PyTorch at runtime:**
- PyTorch adds ~1.5GB to the dependency footprint and makes standalone binaries impractical.
- ONNX Runtime (MIT) runs exported models with no PyTorch dep, ~150MB footprint, acceptable CPU performance.

## Decision

Use `microsoft/unixcoder-base` exported to ONNX. Tokenize with the `tokenizers` library (MIT, no PyTorch). Run inference with `onnxruntime`.

This decision is **provisional**: Phase 5 benchmarks may show that UniXcoder's C# quality is insufficient. If F1 on the C# curated set is below 0.75, we will evaluate CodeT5+ or a fine-tuned variant before proceeding to Phase 6.

## Consequences

**Positive:**
- No PyTorch at runtime. Standalone binaries are viable.
- MIT license. No licensing risk.
- `Embedder` protocol means the model is swappable without changing downstream code.
- CPU inference is acceptable (~50–200 units/sec depending on hardware).

**Negative:**
- C# is not in UniXcoder's pretraining corpus. Embedding quality for C# is extrapolated, not validated. Phase 5 must validate.
- ONNX export adds a one-time maintenance step when the model is updated.
- 125M parameters means the model file is ~500MB — not small. Users must download it on first run.
