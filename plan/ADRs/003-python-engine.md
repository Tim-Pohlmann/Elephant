# ADR 003 — Engine in Python

**Status:** accepted  
**Phase:** 0

---

## Context

The core semdup engine needs a host language. Candidates:

- **Python:** the ML/data ecosystem (FAISS, ONNX Runtime, tree-sitter bindings, tokenizers, pygls) is Python-native or has strong Python bindings. Fast to iterate. PyInstaller can produce standalone binaries, though they are large (~400MB).
- **Rust:** small binaries, fast, safe. But ONNX Runtime bindings are less mature, tree-sitter has Rust bindings but the ecosystem is thinner, and the ML interop story is harder. Significant additional development effort.
- **Go:** similar trade-offs to Rust. ML ecosystem even thinner.

## Decision

Python 3.11+. Standalone binaries via PyInstaller. Rewriting in Rust is a v2 consideration if adoption justifies it — the `Embedder` and `LanguageAdapter` protocols are designed to be portable interfaces.

## Consequences

**Positive:**
- All dependencies (FAISS, ONNX Runtime, tree-sitter, pygls) have first-class Python support.
- Fast iteration speed.
- PyInstaller produces usable binaries despite their size.

**Negative:**
- Binaries are ~400MB (model not included). This is unusual but not unprecedented (e.g., PyTorch-based tools are larger).
- Python startup time is ~1–2 seconds for a cold binary, which is slightly slow for a CLI but acceptable for a batch tool.
- A Rust rewrite, if ever needed, would touch all layers simultaneously — the protocols help, but it's still a large project.
