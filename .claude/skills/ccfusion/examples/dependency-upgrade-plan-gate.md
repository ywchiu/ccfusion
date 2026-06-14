# Example: Dependency Upgrade Plan Gate

A worked Plan Gate run that uses Gemini research and a reused manifest. Toy data.

## Task

> "Upgrade our HTTP client library from v2 to v4."

Dependency upgrade → suggested routing: **migration + security + test + Gemini**.

## Run directory

```text
.fusion/runs/20260614T101500Z-example-api-a13f9c2-q9w2re/
```

## 1. Context Pack → `input/context-pack.md`

- Acceptance Criteria: all call sites migrated; test suite green; no behavior
  change for existing endpoints.
- Known Constraints: must keep Node 18 support.
- External Freshness Needed?: **Yes** — v3/v4 breaking changes and advisories.

## 2. Pre-gate validation → PASS

## 3. External Call Manifest (per target + data-scope)

`manifests/gemini__public-docs-no-code.md`

- Data Scope Name: `public-docs-no-code`
- Data Allowed To Leave: library name, current/target versions, public error
  strings.
- Data Excluded: repo source, `.env`, internal URLs.
- Secrets Present?: No. User/Customer Data Present?: No.
- Approved For Calls: call-001, call-002, call-003.

**Reuse:** all three Gemini calls share this one manifest because the data scope
is unchanged:

```text
calls/gemini/call-001.md  official v4 migration guide
calls/gemini/call-002.md  GitHub issues for v3→v4 breaking changes
calls/gemini/call-003.md  security advisories for v2
```

## 4. Gemini research → `research/gemini-research-pack.md`

Key claims: v4 drops the callback API (breaking); v2 has a known SSRF advisory;
Node 18 still supported in v4. Confidence: high. Emits **no** gating decision.

## 5. Reviewers (read-only)

- `reviews/migration-reviewer.md` → REVISE_PLAN: plan must enumerate every
  callback-style call site. (Recorded under the `domain-reviewer` role acting as
  migration reviewer.)
- `reviews/security-reviewer.md` → APPROVE_PLAN: upgrade closes the SSRF advisory.
- `reviews/test-reviewer.md` → NEEDS_MORE_RESEARCH initially; satisfied by the
  research pack → APPROVE_PLAN.

## 6. Codex adversarial review → REVISE_PLAN

Blocking: no rollback path if v4 changes timeout semantics. Require a documented
pin-back-to-v2 rollback.

## 7. Plan Judge → `judge/attempt-01.md`

Most-severe-wins over {REVISE, APPROVE, APPROVE, REVISE} → **REVISE_PLAN**.
Required: enumerate call sites + add rollback-to-v2 plan.
`plan_revision_attempt_count` → 1.

## 8. Revision + re-judge → `judge/attempt-02.md` → APPROVE_PLAN

## 9. Approved Plan → `approved-plan/attempt-01.md`

External Research Used: cites the research pack's breaking-change and advisory
claims. Rollback Plan: pin to v2, documented.

## 10. Implementation + Final Summary

Single writer migrates call sites, suite green. `status` → `implemented`.
Final Summary notes the closed SSRF advisory and the rollback pin.
