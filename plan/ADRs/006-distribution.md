# ADR 006 — Three distribution artifacts from one core

**Status:** accepted  
**Phase:** 0

---

## Context

semdup needs to reach different user types through different channels:
- **Developers** building pipelines or integrating into existing tooling.
- **End users** who want a binary they can run without installing Python.
- **Claude Code users** who want in-editor integration.

## Decision

Three artifacts, one core:

1. **Python package (`semdup-core` on PyPI):** for developers. `pip install semdup-core`.
2. **Standalone binaries on GitHub Releases:** one per platform, produced by PyInstaller in CI on every version tag. For end users who don't want Python.
3. **Claude Code plugin (`packages/plugin/`):** npm package that bundles/downloads the binary and exposes slash commands and an LSP server. For in-editor use.

The core engine (`packages/core`) is a library. CLI, binary, plugin, and LSP server are all frontends over that library. No logic lives in the frontends — only wiring.

## Consequences

**Positive:**
- One place to fix bugs and add features.
- The LSP server reuses the index and embed modules directly — no IPC overhead, no duplication.
- Binary users get the same quality as Python package users.

**Negative:**
- Three release artifacts to maintain. CI is more complex.
- PyInstaller binaries are large (~400MB). This is a known trade-off (see ADR 003).
- The plugin's binary download mechanism must work cross-platform. This is the most fragile piece of the distribution story and requires careful implementation in Phase 8.
