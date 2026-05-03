# Phase 8 — Claude Code plugin (slash commands)

**Dependencies:** Phase 7 done (binary must exist before the plugin can shell out to it).

---

## Scope

A Claude Code plugin that exposes three slash commands backed by the semdup binary. The plugin auto-wires the LLM verifier using Claude Code's ambient model context, so no API key is needed when running inside Claude Code.

---

## Before starting this phase

**Re-read the current Claude Code plugin docs at `code.claude.com/docs`.** The plugin manifest format is actively evolving. Do not rely on the schema described here if the docs say otherwise — the docs win. Note any discrepancies in a new ADR.

---

## Plugin structure

```
packages/plugin/
  plugin.json       # manifest
  commands/
    scan.md         # /semdup scan
    check.md        # /semdup check
    explain.md      # /semdup explain
  scripts/
    install.js      # post-install: download binary if not bundled
  README.md
  package.json
```

---

## Commands

**`/semdup scan`**
- Runs `semdup run <workspace_root> --format json`.
- Displays the top 10 clusters inline as a formatted summary.
- Shows total cluster count and a note to run with `--format json` for full output.

**`/semdup check <file>`**
- Runs `semdup query <workspace_root> --file <file> --format json`.
- Displays clusters involving units from the specified file only.

**`/semdup explain <cluster-id>`**
- Runs `semdup query` with a cluster filter and invokes the LLM verifier on the pairs in that cluster.
- Asks Claude to explain why the cluster is a duplication and whether it's worth extracting.
- This command uses the ambient Claude Code model context — no API key required.

---

## LLM verifier auto-wiring

When the plugin invokes the binary, it sets an environment variable (`SEMDUP_CLAUDE_CODE=1`) that tells the binary it is running inside Claude Code. In this mode:
- The `LLMVerifier` is enabled automatically (no `ANTHROPIC_API_KEY` check needed).
- The budget defaults to 100 instead of 50.
- The binary uses a subprocess call back to the Claude Code model for verification prompts (exact mechanism: use the `--mcp` flag or shell out to `claude` CLI — check current Claude Code docs for the correct pattern).

---

## Binary distribution

Options in order of preference:
1. **Download on install:** `scripts/install.js` runs after `npm install`, downloads the correct binary for the current platform from GitHub Releases, caches it in `~/.cache/semdup/bin/`.
2. **Require PATH:** if auto-download is too complex to do reliably cross-platform, fall back to requiring the user to install the binary separately and pointing to it via PATH. Document clearly.

Pick option 1. If it proves unreliable on any platform, fall back to option 2 and document why in an ADR.

---

## Deliverables

- `packages/plugin/` — full plugin package.
- Installation docs in `packages/plugin/README.md`.
- At least one end-to-end smoke test: run the plugin against the Phase 1 C# fixtures, assert `/semdup scan` produces non-empty output.

---

## Verification

1. Install the plugin in a real Claude Code session (`/plugin install` or equivalent).
2. Open a C# repo with known duplication.
3. Run `/semdup scan` — confirm non-empty, coherent output within 60 seconds.
4. Run `/semdup check <one file>` — confirm output is scoped to that file.
5. Run `/semdup explain <a cluster-id from the scan>` — confirm a coherent explanation is generated.
