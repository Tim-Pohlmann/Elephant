# PROCESS.md — How we work on semdup

This file is the operating manual for every Claude Code session on this project. **Read it fully every session before acting.**

---

## What to read each session

| File | When to read |
|------|-------------|
| `PROCESS.md` | Every session, fully (you are reading it now) |
| `STATUS.md` | Every session, fully (it's short) |
| `PLAN.md` | Every session, fully (it's a slim index — goals, layout, decision and phase tables) |
| `docs/phases/phase-N.md` | Only the active phase. Check STATUS.md for N. |
| `docs/phases/phase-M.md` | Only if the active phase explicitly lists phase M as a dependency to verify |
| `docs/decisions/NNN-*.md` | Only when an existing decision is challenged, or you need to add a new ADR |

Do not load phase files speculatively. Do not load ADRs speculatively. Only load what the above table directs.

---

## Core rules

1. **Never work on multiple phases in one session.** The active phase is whichever phase in `STATUS.md` is `active`. If none is `active`, it is the first `not_started` phase.
2. **Never skip the propose step.** Before writing code, post the plan-of-attack and wait for user approval. A one-word "go" is enough approval.
3. **Never deviate from the active phase's spec silently.** If you find a reason to change scope, contract, or a dependency, stop and ask. Record the outcome as a new ADR.
4. **Update `STATUS.md` as the final commit of every phase.**
5. **Commit in logical chunks.** One commit per module, one per substantial test suite, one per docs update. Every commit message starts with the phase tag: `[phase-1] add CSharpAdapter`.
6. **No phase is done without passing tests.** `ruff`, `mypy --strict`, and `pytest` must all pass before declaring verification complete.
7. **Keep dependencies minimal and permissively licensed.** Any new dependency requires a note in the phase's ADR or commit message, including its license.
8. **When verification fails, stop and report.** Do not paper over a regression or a flaky test. Report findings, propose options, wait.

---

## Phase workflow

Every phase runs these six steps in order.

### Step 1 — Orient

Read the files listed in the "What to read" table above. Then post a brief **orientation message** covering:
- Which phase is active and what its verification criteria are.
- Whether all dependency phases are marked `done` in `STATUS.md`. If not, stop and ask.

### Step 2 — Propose

Post a **proposal** covering:
- Module skeletons to create (filename + one-line purpose).
- New dependencies and their licenses.
- Test plan: what each test verifies, what fixtures are needed.
- Any ambiguity in the phase spec, with a proposed resolution.
- Whether the phase has parallelizable work for subagents (see below), and if so, the split.

Wait for user approval before writing code.

### Step 3 — Mark active

In `STATUS.md`, mark the phase `active` with today's date. Commit: `[phase-N] mark active`.

### Step 4 — Implement

Implement the proposal. Commit logically as you go. Keep CI green at every commit.

If something mid-phase changes the scope, stop and raise it — don't silently re-plan.

### Step 5 — Verify

Run the verification criteria from the phase file exactly. Report results in one message:
- Pass/fail per criterion.
- Numbers where the phase asks for numbers.
- Any known gaps being deferred.

If verification fails, stop. Do not proceed to step 6.

### Step 6 — Close

Update `STATUS.md`:
- Flip the phase to `done`.
- Append a one-paragraph summary: what shipped, what was learned, what was deferred.
- Add any follow-up debts under the `Debts` section.

Final commit: `[phase-N] close`. Stop the session. Do not start the next phase.

---

## Subagent guidance

Default: don't use them. The coordination overhead exceeds the benefit for most tasks.

**Use subagents when the active phase file explicitly flags parallelizable work.** Currently that's Phase 1 (fixtures), Phase 5 (clone pair curation), and Phase 6 (per-language adapters after the first).

**When using a subagent:**
- Give it a crisp deliverable: exact input, output, acceptance criteria.
- Tell it what it must NOT do: don't commit, don't modify PLAN.md or PROCESS.md, don't touch code outside its scope.
- Review its output before committing. Subagents don't commit directly.

---

## Communication norms

- Short and direct. No filler.
- Numbers over adjectives ("142 units/sec" not "good throughput").
- Flag risks in step 1 or 2, not at verification.
- One question at a time when blocked.
