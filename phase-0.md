# Phase 0 — Repo skeleton and decisions

**Dependencies:** none.

---

## Scope

Establish the monorepo structure, license, CI, pre-commit hooks, and the six foundational ADRs. Nothing is implemented — only scaffolding and documentation.

---

## Deliverables

- Directory layout matching `PLAN.md` repo layout (empty packages are fine; the shape must be correct).
- `LICENSE` — MIT.
- `packages/core/pyproject.toml` with dev dependencies: `pytest`, `pytest-cov`, `ruff`, `mypy`, `pre-commit`.
- `.github/workflows/ci.yml` — matrix: Linux/macOS/Windows × Python 3.11/3.12. Steps: `ruff check`, `mypy --strict`, `pytest`.
- `.pre-commit-config.yaml` — runs `ruff format` and `ruff check` on commit.
- `docs/decisions/` — one ADR per the six decisions in `PLAN.md`. Each follows standard ADR format: **Context**, **Decision**, **Consequences**. File names must match the index in `PLAN.md`.
- `packages/core/tests/unit/test_smoke.py` — a single trivially passing test so CI has something to run.
- `STATUS.md` initialized (already exists; update it in step 3 of the workflow).

---

## Verification

1. On a fresh clone: `pip install -e packages/core[dev]` succeeds, `pytest` passes.
2. GitHub Actions CI is green on the initial commit (all three OS, both Python versions).
3. Pre-commit hooks run cleanly on a test commit.
4. All six ADR files exist and are non-empty.

---

## Notes

- Do not implement any `semdup` logic in this phase. If you find yourself writing anything beyond scaffolding and docs, stop.
- The ADRs in `docs/decisions/` are the canonical record of *why* we made each architectural choice. Write them with enough context that someone reading in six months understands the trade-offs. They are not one-liners.
- If you have a strong reason to prefer Apache 2.0 over MIT, document it in the ADR for decision 006 and raise it before committing the `LICENSE` file.
