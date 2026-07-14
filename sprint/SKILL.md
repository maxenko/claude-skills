---
name: sprint
description: "End-to-end workflow that takes a feature request (or any coding task) and drives it from framing to a machine-verified, adversarially-reviewed diff — pausing only at the two decisions that are the human's (approve the plan, accept the result). Use when the user says 'sprint', '/sprint', hands over a feature request, or wants the full plan-through-verified-change arc. The upgraded successor to devloop: the heavy adversarial pass runs on the DIFF in fresh context (not just the plan), critics are grounded in checkable failures, loops are bounded, and every confirmed finding becomes a permanent test/guard. Do NOT use for pure planning (use plan-feature), plan review only (use plan-eval), or reviewing existing code with no task attached (use code-scrutiny)."
argument-hint: "<feature request or any coding task>"
model: opus
---

# Sprint — request → verified, review-clean change

Drive `$ARGUMENTS` from framing to a machine-verified, adversarially-reviewed change.

**The reframe this encodes:** writing code is cheap; *trusting* it is the bottleneck. So
the rigor lives on the **diff** (the code that will run), not the **plan** (a prediction).
Governing rule: **the human decides *what*, the agent decides *how*, a machine-checked
signal decides *done*.**

This is the upgraded `devloop`. What changed, and why:

- **It does not end at "implemented."** The heavy adversarial pass runs on the DIFF, in
  **fresh context**, with the test output in hand — a reviewer that didn't write the code
  and has no sunk-cost attachment. That is the single highest-value pass.
- **Critics are grounded.** Every objection ties to something checkable (a test it proposes,
  a failure it demonstrates). Ungrounded self-critique degrades accuracy; grounded critique
  finds real defects regardless of which model found them.
- **Loops are bounded correctly.** Ungrounded loops (plan critique) cap at **≤2 rounds** —
  more rounds entrench errors, they don't fix them. Grounded loops (fix → re-run the check)
  run **until the signal is green**, because each iteration has new ground truth.
- **Findings become permanent guards.** Every confirmed defect is converted to a test or a
  demo harness *before* it's fixed, so the failure mode can never silently return.

Single-model Claude throughout — a concrete defect with a reproduction is verifiable on its
own merits; you do not need cross-model vote independence to find bugs, only fresh context,
divergent lenses, and a "show me the repro" bar.

---

## Phase 0 — Right-size (always first)

Before any machinery, judge the scope. **Skip the ceremony when the work is obvious.**

| Tier | What it is | What runs |
|------|-----------|-----------|
| **Trivial** | One-line / config / rename / obvious fix | Just do it → verify (Phase 5) → done. No plan, no panel. |
| **Small** | Single concern, one file or two | Light spec → implement → verify → quick diff-review. Skip the plan panel. |
| **Standard** | Multi-file feature or refactor | Full arc below. |
| **Large** | System-level / multi-stream | Full arc; plan and diff-review fan out to parallel subagents. |

State the tier in one line and proceed. When in doubt between two tiers, round down — you
can always escalate. Plan mode on a one-line fix is pure overhead.

---

## Phase 1 — Frame & name the check

Restate `$ARGUMENTS` as an **outcome**, not a task list. Then do the one thing that makes
everything after it work:

**Name the machine-checkable success criterion.** What command, test, or observable proves
this is done? (`cargo test` passes; a new test asserts X; the app drives through flow Y with
no console/network errors.) This is what lets the agent self-verify instead of stopping when
work merely *looks* done. **If you cannot name a check, that is the first thing to fix** —
either design one, or ask the user what "done" means here. Do not proceed without it.

If the request is ambiguous, interview the user briefly (2–3 questions) before planning.
Keep the spec lightweight — inline for Small, a short `plan/*.md` for Standard+.

---

## Phase 2 — Plan (Standard+ only)

Invoke **`plan-feature`** (or `plan-ultra` for large multi-stream work) to produce a phased,
vertically-sliced plan in `plan/*`. Small reviewable slices — review capacity is the real
bottleneck. Each slice states its own done-criterion (tie back to the Phase 1 check).

---

## Phase 3 — Plan critique + approval gate

Invoke **`plan-eval`** on the plan. **Bounded to ≤2 rounds** of refinement — grounded
critique converges fast; if two rounds don't land a SOLID verdict, **stop and surface the
disagreement to the user** rather than grinding a polluted context.

**■ HUMAN GATE — plan approval.** Present the plan and enter plan mode; do not write code
until the user approves. This gate is a strength of the old flow — keep it.

---

## Phase 4 — Implement in slices

Work the approved plan slice by slice. After **each** slice, run the Phase 1 check — don't
batch verification to the end. Honor the project's invariants (read its `CLAUDE.md`); a
generic implementer misses house-specific footguns unless they're in front of it.

---

## Phase 5 — Verify against the runnable check

Nothing is "done" until the check passes. Discover it (in priority order): the criterion
named in Phase 1 → a project `verify` skill → the obvious build/test command for the stack
(`cargo test`, `npm test`, …).

**For UI / runtime changes, drive the real app and look** — do not assert behavior from the
source or the DOM. Use the project's own verification path (read `CLAUDE.md`: e.g. this repo
drives the running window with `tools/ui_probe.ps1` + `AL_*` demo harnesses + in-page
`document::eval` dispatch — *not* Playwright, which cannot reach a native desktop WebView).
Pattern: **agent explores, code judges** — the agent exercises the running software; the
agent checks console + network errors; deterministic assertions decide pass/fail. **Demand
evidence** (command output, screenshots), not claims.

---

## Phase 6 — Diff-review in FRESH context (the center of gravity)

Spawn **fresh subagent(s)** — via `Agent` for Small, or a `Workflow` fan-out for Standard+
(≤10 in flight). They must NOT be the context that wrote the code. Give each subagent ONLY,
inline (a subagent sees only its dispatch prompt):

1. **The diff** — `git diff` of the change under review.
2. **The check output** — the passing (or failing) test/build result from Phase 5.
3. **The project's invariants** — the relevant `CLAUDE.md` rules as an explicit checklist
   (for this repo: theming/`var()` re-patch traps, the perf-scaling governance rules,
   layout invariants, FPL doc-sync). This is what turns a generic critic into one that finds
   *real* defects — the panel is smart because you feed it the domain.

Point your existing **`code-scrutiny`** / **`bug-hunt`** muscle at the diff, one per lens
(correctness & error paths; the project's known footguns; perf / resource scaling). Each
finding must carry a **concrete failing input/state → observable consequence** — no
style-only or speculative findings. "No issues found" is a valid, preferred outcome.

---

## Phase 7 — Regression capture

For every **confirmed** finding, add a permanent guard — a unit test, or a demo/probe
harness for UI — that fails on the defect **before** you fix it. The real endpoint of the
loop is not "review passed"; it's "the failure mode review caught can never silently
return." This is the highest-compounding habit in the whole flow.

---

## Phase 8 — Fix loop (grounded — run to green)

Fix the confirmed findings, then **re-run the Phase 5 check + the new Phase 7 guards**.
Repeat until green. This loop is **grounded** — each pass has a fresh, real signal — so it is
**not** capped at 2; run it until the check is clean. (The ≤2 cap applies only to *ungrounded*
rumination, e.g. re-critiquing the plan with no new signal.)

**■ HUMAN GATE — accept.** Report back, succinct and structured:
- what was built (against the outcome from Phase 1),
- the green-check evidence (output / screenshots),
- each confirmed finding, how it was resolved, and the guard now protecting it,
- anything deferred, and why.

The panel advises; **the human decides.** Commit only when asked (respect the repo's commit
conventions — branch off the default branch if needed).

---

## Loop discipline (the rule that governs every phase)

- **Ungrounded loop** = revising an answer with no new external signal (re-critiquing a plan,
  self-correcting prose). Converges in 1–2 rounds, then *entrenches* errors. **Cap at ≤2.**
  If it's not landing, `/clear` + re-prompt beats grinding a polluted context.
- **Grounded loop** = each iteration has new ground truth (a failing test, a new repro, real
  execution output). **Run to green.** Bounding this one is a mistake.

## What this reuses (don't reinvent — conduct)

| Phase | Delegates to |
|-------|-------------|
| Plan | `plan-feature` / `plan-ultra` |
| Plan critique | `plan-eval` |
| Diff-review | `code-scrutiny` / `bug-hunt` in fresh subagents |
| Verify (UI) | project `verify` skill / `tools/ui_probe.ps1` + demo harnesses |

## The five judgments that stay with the human (never automated)

1. Is this task big enough to bother with the machinery? (Phase 0)
2. What does "done" look like? (Phase 1 — you define the check; the flow enforces it)
3. How much do I trust it to run alone here? (autonomy between the two gates)
4. Is it stuck? (know when to `/clear` and restart)
5. Do I actually agree with the panel? (it advises; you decide — the accept gate)
