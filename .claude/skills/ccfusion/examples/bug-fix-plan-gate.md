# Example: Bug Fix Plan Gate

A worked Plan Gate run for an ambiguous bug. Toy data; illustrates the flow, not
a real repository.

## Task

> "Users intermittently get logged out after ~5 minutes. Fix it."

Root cause is unclear, so this is a bug fix with an unclear root cause →
suggested routing: **test-reviewer + architecture-reviewer**.

## Run directory

```text
.fusion/runs/20260614T093012Z-example-api-a13f9c2-k8x4mz/
```

`run-state.json` starts with `status: "running"`, `preset: "plan-gate"`,
`execution_profile: "strict-single-writer"`.

## 1. Context Pack → `input/context-pack.md`

- Task Summary: sessions drop after ~5 min; reproduces under load.
- Acceptance Criteria: session survives ≥30 min of activity; a regression test
  fails before the fix and passes after.
- Known Constraints: cannot change the public auth API; Redis-backed sessions.
- External Freshness Needed?: No.

## 2. Pre-gate validation → `input/input-validation.md`

```text
Decision: PASS
```

Acceptance criteria are checkable and constraints are explicit, so the panel may
run. `primary_artifact_rewrite_count` stays 0.

## 3. No Gemini

External Freshness Needed = No → skip Gemini. No manifest created.

## 4. Reviewers (read-only)

- `reviews/test-reviewer.md` → **REVISE_PLAN**: no failing regression test exists
  yet; the plan must add one that reproduces the 5-minute drop.
- `reviews/architecture-reviewer.md` → **REVISE_PLAN**: the draft plan assumes a
  cookie bug, but session TTL is refreshed in two places; likely a TTL race.

## 5. Codex adversarial review → `reviews/codex-adversarial-review.md`

**Recommendation: REVISE_PLAN.** Blocking concern: the draft "increase TTL" fix
hides the race rather than fixing it; a data-loss-free fix must make TTL refresh
idempotent. Simpler alternative: single source of truth for refresh.

## 6. Plan Judge → `judge/attempt-01.md`

Most-severe-wins over {REVISE, REVISE, REVISE} → **REVISE_PLAN**.
Required plan changes: (a) add a failing repro test, (b) consolidate TTL refresh,
(c) drop the blunt TTL increase. `plan_revision_attempt_count` → 1 (limit 2).

## 7. Revision → `approved-plan/attempt-01.md`, re-judge → `judge/attempt-02.md`

Revised plan adds the repro test and consolidates refresh. Plan Judge:
**APPROVE_PLAN** — no unresolved P0/P1 issues.

## 8. Approved Plan → `approved-plan/attempt-02.md`

`run-state.status` → `approved`. Only now may Claude Code edit files.

## 9. Implementation (single writer)

Claude Code main session implements from a clean tree: adds the failing test,
consolidates refresh, confirms the test now passes. `status` → `implemented`.

## 10. Final Summary → `final/final-summary.md`

Decision APPROVE_PLAN; files changed listed; repro test red→green; remaining risk:
monitor session metrics post-deploy.
