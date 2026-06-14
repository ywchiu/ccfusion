# Example: Refactor Plan Gate (ends in BLOCKED_NEEDS_HUMAN)

A worked Plan Gate run that exhausts a bounded loop and escalates. Toy data.
Shows the non-happy path: the gate stops cleanly rather than writing code.

## Task

> "Refactor the billing module to remove the God object `BillingManager`."

Refactor → suggested routing: **architecture + test**.

## Run directory

```text
.fusion/runs/20260614T140000Z-billing-svc-no-commit-h7t1pa/
```

Note the `no-commit` git_ref: this repo had no commits when the run started, so
the deterministic fallback was used.

## 1. Context Pack → PASS

- Acceptance Criteria: behavior preserved; `BillingManager` split into ≥3
  cohesive units; characterization tests green.
- External Freshness Needed?: No.

## 2. Reviewers (read-only)

- `reviews/architecture-reviewer.md` → REVISE_PLAN: the proposed seams cut across
  a transaction boundary; splitting there risks partial writes.
- `reviews/test-reviewer.md` → BLOCKED_NEEDS_HUMAN: there are **no**
  characterization tests, and the module's current behavior is undocumented, so
  "behavior preserved" cannot be verified by the team's stated criteria.

## 3. Codex adversarial review → REVISE_PLAN

Data-loss risk at the transaction boundary; simpler alternative: extract
read-only query helpers first, defer the transactional split.

## 4. Plan Judge → `judge/attempt-01.md`

Precedence: BLOCKED_NEEDS_HUMAN > NEEDS_MORE_RESEARCH > REVISE_PLAN > APPROVE_PLAN.
One reviewer raised a credible human-blocking risk (no way to verify behavior
preservation). Per rule 1, it must be resolved or explicitly rejected with
rationale. The Judge cannot reject it — characterization coverage is a human
decision about acceptable risk.

```text
Decision: BLOCKED_NEEDS_HUMAN
```

## 5. Stop and escalate

- No plan is approved. No `approved-plan/` attempt is written.
- `run-state.status` → `blocked`.
- `final/final-summary.md` written:
  - Plan Gate Decision: BLOCKED_NEEDS_HUMAN
  - Approved Plan Location: none
  - Files Changed: none — **No implementation performed.**
  - Recommended Next Step: a human must decide whether to invest in
    characterization tests before the refactor, or accept the risk explicitly.

## Why this is the correct outcome

v0.1 deliberately stops and escalates rather than guessing. The Plan Gate never
edits files, and a credible un-rejectable blocking risk outranks every other
signal. The run directory remains as the audit trail and is never stashed.
