# ADR 005 — Two-stage verifier: structural always-on, LLM optional

**Status:** accepted  
**Phase:** 0

---

## Context

Embeddings produce recall — they surface candidate pairs. Without a second stage, precision suffers: short functions cluster regardless of meaning, and thematically related but non-duplicate code scores high. A verifier is needed to turn candidate pairs into trusted results.

Options:
- **Structural only:** normalized AST comparison + tree edit distance. Fast, always available, no external deps. Catches Type 1–3 well; Type 4 (semantic-only) is inherently hard for structural comparison.
- **LLM only:** high quality, understands intent, but slow, costs money, requires an API key or local model. Not viable as a mandatory step.
- **Structural + optional LLM:** structural verifier runs on every candidate pair (fast, free, good recall filter). LLM verifier runs only on borderline pairs (slow path, improves precision where it matters most).

## Decision

Two-stage verifier:
1. `StructuralVerifier` — always runs. Uses AST fingerprint equality and tree edit distance.
2. `LLMVerifier` — optional. Activates only when `ANTHROPIC_API_KEY` is set or when running inside Claude Code. Runs only on pairs in the borderline similarity window. Budget-capped per run.

Both implement the `Verifier` protocol. The pipeline accepts either as a drop-in.

## Consequences

**Positive:**
- Tool is useful without any API key. The structural verifier alone is competitive.
- LLM verifier adds precision for users who want the best possible results and accept the API cost.
- The `Verifier` protocol makes it easy to add future verifiers (e.g., dataflow equivalence, a local LLM via llama.cpp).

**Negative:**
- Two verifiers means two code paths to maintain.
- The LLM verifier introduces a runtime dependency on the Anthropic API for best results. Users without access get slightly lower precision on Type 4 clones.
- Budget cap introduces a non-deterministic element: results vary if borderline pairs exceed the budget. Document this clearly.
