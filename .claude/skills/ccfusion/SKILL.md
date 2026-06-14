---
name: ccfusion
description: Use when the user asks to use controlled coding fusion / ccfusion, or wants an AI-assisted coding change planned and adversarially reviewed by multiple read-only agents before any file is edited. Plan-gating only — one writer, bounded loops, auditable run directory. Not for routine edits the user wants done immediately.
---

# Controlled Coding Fusion (ccfusion)

## Overview

ccfusion is a Plan Gate for safer AI-assisted coding. It raises planning quality
**before any file is touched** and leaves an auditable trail of how the plan was
reached. It does not guarantee correctness.

```text
Many agents may inspect and critique.
Only Claude Code main session may execute.
```

**Tagline:** Plan first. Review in parallel. Execute with one writer.

This is v0.1 — intentionally narrow. Exactly **one preset** (`plan-gate`) under
exactly **one execution profile** (`strict-single-writer`). There is no router,
no preset classification, no `presets/` directory. Roadmap items live in
`README.md` only and must not be materialized as directories or implied to exist.

## When to Use

- The user says "use controlled coding fusion", "ccfusion", "plan-gate this", or
  asks to plan/review a change with multiple agents before writing code.
- The change is risky enough to want adversarial review first: dependency
  upgrades, refactors, security-sensitive code, ambiguous bug fixes.

**When NOT to use:**
- The user wants a routine edit done immediately. ccfusion adds a planning gate;
  don't impose it on simple, low-risk tasks.
- Post-implementation diff review (that is the v0.2 Review Gate — not built).

## The One Rule

**Only the Claude Code main session may modify files.** Claude subagents, Codex,
and Gemini are read-only. The Plan Gate itself **never edits files**.
Implementation begins only after an Approved Plan artifact exists.

## User Override

User instructions outrank this skill — direct requests (and CLAUDE.md/AGENTS.md)
are the highest priority. If the user **explicitly** tells you to skip, stop, or
not run ccfusion for a task, honor it: stop the run cleanly (`status` =
`stopped`), record the opt-out in `final/final-summary.md`, then proceed outside
the gate by their direct instruction. The "no edit before an Approved Plan" rule
constrains a *live gated run*; once the user terminates the run, there is no
gated run to violate.

**This is not a bypass for pressure.** Impatience, urgency, a "trivial" change,
or "the user is waiting" is **not** an override. Absent an explicit opt-out, stay
in the gate and finish the bounded loop fast rather than skipping it. Never
silently edit without acknowledging either the skill or the override.

## Execution Profile: strict-single-writer

These constraints hold for the entire run:

1. Only Claude Code main session may modify files.
2. Claude subagents are read-only.
3. Codex is read-only adversarial review only.
4. Gemini is external research only.
5. No other agent may write to the working tree.
6. No agent may run destructive commands (see Forbidden Commands).
7. No production deployment.
8. No database migration unless the user explicitly approves it.
9. Tests and CI are the final objective validator.

**Allowed Codex commands:** `/codex:adversarial-review`, `/codex:review`,
`/codex:status`, `/codex:result`.
**Forbidden by default:** `/codex:rescue` (belongs to a future worktree-isolated
profile, not v0.1).

## Behavior Summary (the flow)

Run these in order. Every loop is bounded and every overflow ends in
`BLOCKED_NEEDS_HUMAN`.

1. Create a run directory and `run-state.json`.
2. Create a Context Pack (`input/context-pack.md`).
3. Validate the Context Pack **before any panel call** (`input/input-validation.md`).
4. If invalid, rewrite it (bounded) or `BLOCKED_NEEDS_HUMAN`.
5. If external freshness matters, call Gemini **only with a scoped manifest**.
6. Run read-only Claude reviewers selected by task type.
7. Run Codex adversarial review if available.
8. Synthesize through the Plan Judge with most-severe-wins precedence.
9. `REVISE_PLAN` / `NEEDS_MORE_RESEARCH` → bounded loops → else `BLOCKED_NEEDS_HUMAN`.
10. On `APPROVE_PLAN`, write the Approved Plan.
11. Only then implement, as the single writer, from a clean working tree.
12. If the plan is falsified mid-implementation, stop and escalate (no auto re-plan).
13. No other agent may modify files.
14. Write the Final Summary.

## Step Detail

### 1. Run directory & run_id

Create one run directory per invocation: `.fusion/runs/<run_id>/`

`run_id` format: `YYYYMMDDTHHMMSSZ-<repo_slug>-<git_ref>-<random6>`

Resolve `git_ref` deterministically:

```text
git rev-parse --short HEAD succeeds  -> short commit sha
repo has no commits                  -> no-commit
not a git repository                 -> no-git
```

`branch` is optional metadata, never used for uniqueness. If branch cannot be
resolved, record `detached-or-unknown`. The run directory is the audit trail:
**never stash, move, or delete it.**

Create the full layout up front:

```text
.fusion/runs/<run_id>/
  run-state.json
  index.md
  input/        context-pack.md, input-validation.md
  manifests/    gemini__<data-scope-slug>.md, codex__<data-scope-slug>.md
  calls/        gemini/call-NNN.md, codex/call-NNN.md
  research/     gemini-research-pack.md
  reviews/      <reviewer-name>.md, codex-adversarial-review.md
  judge/        attempt-NN.md
  approved-plan/ attempt-NN.md
  final/        final-summary.md
```

**Artifacts must never be overwritten.** A regenerated artifact is a new
`attempt-NN` file. `index.md` may point to the currently selected artifact but
must never erase attempt history.

### 2. run-state.json

Initialize with status `running`. Counters are **per-run global** and bounded:

```json
{
  "run_id": "...",
  "preset": "plan-gate",
  "execution_profile": "strict-single-writer",
  "status": "running",
  "created_at": "<UTC ISO8601>",
  "repo": { "slug": "...", "git_ref": "...", "branch": "..." },
  "counters": {
    "primary_artifact_rewrite_count": 0,
    "secondary_artifact_rewrite_count": {},
    "research_reentry_count": 0,
    "plan_revision_attempt_count": 0
  },
  "limits": {
    "max_primary_artifact_rewrites": 2,
    "max_secondary_artifact_rewrites_per_artifact": 2,
    "max_research_reentry": 1,
    "max_plan_revision_attempts": 2
  }
}
```

`status` values: `running`, `approved`, `implemented`, `blocked`, `stopped`.
Update `status` and counters as the run progresses.

### 3-4. Context Pack & pre-gate validation

The Context Pack is the **primary input artifact**: task summary, acceptance
criteria, known constraints, repo context. Write it from
`templates/context-pack.md` (schema: `schemas/context-pack.schema.json`).

Before **any** Gemini, Codex, or subagent call, validate the primary input and
write `input/input-validation.md` (schema: `schemas/input-validation.schema.json`).
Allowed validator decisions:

```text
PASS
REWRITE_PRIMARY_ARTIFACT_REQUIRED
BLOCKED_NEEDS_HUMAN
```

On `REWRITE_PRIMARY_ARTIFACT_REQUIRED`: rewrite the Context Pack and increment
`primary_artifact_rewrite_count`. If it exceeds `max_primary_artifact_rewrites`,
the decision becomes `BLOCKED_NEEDS_HUMAN`.

**The panel cannot rewrite primary input.** Once primary input passes, no panel
participant (Claude reviewer, Codex, Gemini, Plan Judge) may emit
`REWRITE_PRIMARY_ARTIFACT_REQUIRED`. If they think context is incomplete they
must express it as `NEEDS_MORE_RESEARCH`, `REVISE_PLAN`, or `BLOCKED_NEEDS_HUMAN`.

### 5. External Call Manifest + Gemini research (optional)

External-call safety is handled **per target + data-scope**, not per call.
Identity = `target + data-scope-slug` (a short readable name, **no hashes**):

```text
manifests/gemini__public-docs-no-code.md
manifests/gemini__sanitized-error-trace.md
manifests/codex__sanitized-diff-summary.md
```

Calls sharing the same target and data scope **reuse** the same manifest. A call
needing a broader data scope requires a new manifest. Write manifests from
`templates/external-call-manifest.md` (schema:
`schemas/external-call-manifest.schema.json`) and record each call under
`calls/<target>/call-NNN.md`.

**Default — data that must not leave:** `.env`/keys/credentials, customer records
and contracts, personal data, raw production logs, database dumps, internal or
regulated financial data, proprietary datasets, defense-sensitive data. If
external research is needed, send a **sanitized technical summary**, never raw
material.

Gemini is the external research layer only — neither reviewer nor executor, and
it **emits no gating decision**. It returns `research/gemini-research-pack.md`
(template: `templates/gemini-research-pack.md`); the Plan Judge consumes it. Use
Gemini for: latest docs, release notes, GitHub issues, breaking API changes,
security advisories, dependency/cloud/browser behavior, unknown error messages.

### Candidate plan & plan-artifact lifecycle

The plan the panel and Plan Judge review lives in `approved-plan/attempt-NN.md`.
Write the initial candidate as `approved-plan/attempt-01.md` **before** the panel
runs; reviewers and the Judge always evaluate the **latest** attempt. On
`REVISE_PLAN`, write the next attempt and increment `plan_revision_attempt_count`.
On `APPROVE_PLAN`, the latest attempt is the Approved Plan; if you fold accepted
**non-blocking** findings in at approval time, write a new attempt for it — that
is an approval-time refinement, **not** a revision, so it does not increment
`plan_revision_attempt_count`. Never overwrite an earlier attempt.

### 6. Claude subagent panel (read-only)

Dispatch read-only reviewers selected by task type — **not all four every time**.
Available roles: `architecture-reviewer`, `test-reviewer`, `security-reviewer`,
`domain-reviewer`. Each writes `reviews/<reviewer-name>.md` from
`templates/reviewer-output.md` (schema: `schemas/reviewer-output.schema.json`).

Suggested routing (a default, not a rule):

| Task type           | Reviewers |
| ------------------- | --------- |
| bug fix             | test + architecture (if root cause unclear) |
| dependency upgrade  | migration + security + test + Gemini |
| refactor            | architecture + test |
| security-sensitive  | security + test + Codex |

### 7. Codex adversarial review

If Codex is available, run read-only adversarial review of the plan
(`/codex:adversarial-review`). Output: `reviews/codex-adversarial-review.md`
(template: `templates/codex-adversarial-review.md`). Focus: hidden assumptions,
simpler alternatives, edge cases, migration/security risks, test gaps, rollback
gaps, over-engineering, concurrency, data loss. Codex cannot modify files.

### 8. Plan Judge synthesis

Synthesize all reviewer outputs, the Codex review, and the Gemini research pack
into `judge/attempt-NN.md` (template: `templates/plan-judge.md`, schema:
`schemas/plan-judge.schema.json`). Decision is one of:

```text
APPROVE_PLAN  REVISE_PLAN  NEEDS_MORE_RESEARCH  BLOCKED_NEEDS_HUMAN
```

There is no `REPAIR`, `ADD_TESTS`, or `REWRITE_*` at the Judge layer. Rejected
findings must include reasons. Contradictions must be resolved or escalated.

**Decision precedence — most-severe-wins:**

```text
BLOCKED_NEEDS_HUMAN > NEEDS_MORE_RESEARCH > REVISE_PLAN > APPROVE_PLAN
```

1. Any credible human-blocking risk must be resolved or explicitly rejected with
   rationale.
2. If material external facts are missing, research happens before approval.
3. If plan changes are needed, the plan is revised before approval.
4. Approval requires no unresolved P0/P1 planning issues.

### 9. Bounded loops

- **`REVISE_PLAN`:** revise and write `approved-plan/attempt-NN.md`, then the
  Plan Judge may run again. Increment `plan_revision_attempt_count`. Limit 2. On
  exceed → `BLOCKED_NEEDS_HUMAN`.
- **`NEEDS_MORE_RESEARCH`:** run one additional Gemini pass. Increment
  `research_reentry_count`. Limit 1. On exceed → `BLOCKED_NEEDS_HUMAN`. Reentry
  reuses the same target + data-scope manifest if scope is unchanged.
- **`BLOCKED_NEEDS_HUMAN`:** stop, set `status` = `blocked`, write the Final
  Summary, escalate to a human.

### 10. Approved Plan

On `APPROVE_PLAN`: write `approved-plan/attempt-NN.md` from
`templates/approved-plan.md` (schema: `schemas/approved-plan.schema.json`) and
set `status` = `approved`. **Only after this file exists may Claude Code modify
code.**

### 11-12. Implementation (single writer) & falsification

After approval, Claude Code main session may implement, following the approved
plan, starting from a clean working tree. Codex, Gemini, and subagents still
cannot implement. Set `status` = `implemented` when done.

**If the plan is falsified mid-implementation (v0.1 — no auto re-plan):**

1. Stop implementation immediately. Do not continue against a known-bad plan.
2. Record in `final/final-summary.md`: what was attempted, which files were
   touched, and **why the plan failed** (the highest-value signal in the run).
3. Set `status` = `blocked`.
4. Report `BLOCKED_NEEDS_HUMAN`.

Do not stash or discard the user's working tree. If uncommitted work exists,
leave it in place and describe it in the summary. Partial-patch preservation and
automatic return to the Plan Gate are a v0.2 feature.

### Git & stash safety

Unscoped stash is **forbidden** (`git stash`, `git stash push`). Stash is allowed
only with an explicit source path scope:

```bash
git stash push -m "fusion:<run_id>:<reason>" -- path/to/source/file
```

The run directory `.fusion/runs/<run_id>/` is **never** stashed, moved, or
deleted. Before any stash, state why it is needed, which paths are included, and
how it will be restored.

### Forbidden commands without explicit approval

```text
rm -rf            git push --force   git reset --hard   git clean -fd
drop database     truncate table     delete from        terraform apply
kubectl delete    kubectl apply      docker system prune
production deploy scripts            database migration scripts
```

No production branch push by default.

### 14. Final Summary

At the end of **every** run, write `final/final-summary.md` (template:
`templates/final-summary.md`). If no implementation was performed, state:
`No implementation performed.`

## Decision Emitter Matrix

| Emitter         | PASS | REWRITE_PRIMARY | APPROVE_PLAN | REVISE_PLAN | NEEDS_MORE_RESEARCH | BLOCKED_NEEDS_HUMAN |
| --------------- | :--: | :-------------: | :----------: | :---------: | :-----------------: | :-----------------: |
| Input Validator | Yes  |       Yes       |      No      |     No      |         No          |         Yes         |
| Gemini Research |  No  |       No        |      No      |     No      |     advisory        |     advisory        |
| Claude Reviewer |  No  |       No        |     Yes      |     Yes     |         Yes         |         Yes         |
| Codex Reviewer  |  No  |       No        |     Yes      |     Yes     |         Yes         |         Yes         |
| Plan Judge      |  No  |       No        |     Yes      |     Yes     |         Yes         |         Yes         |

Gemini returns a research pack, not a gating decision — it may *flag* that more
research or a human is needed, but never approves or revises. The Plan Judge owns
the decision. `REWRITE_PRIMARY_ARTIFACT_REQUIRED` exists **only** in pre-gate
validation; it never competes with `REVISE_PLAN` or `NEEDS_MORE_RESEARCH`.

## Counters & Limits

| Counter | Limit | On overflow |
| ------- | :---: | ----------- |
| `primary_artifact_rewrite_count` | 2 | `BLOCKED_NEEDS_HUMAN` |
| `secondary_artifact_rewrite_count[artifact]` | 2 per artifact | `BLOCKED_NEEDS_HUMAN` |
| `research_reentry_count` | 1 | `BLOCKED_NEEDS_HUMAN` |
| `plan_revision_attempt_count` | 2 | `BLOCKED_NEEDS_HUMAN` |

Secondary artifacts (research pack, reviewer outputs, Codex review, judge
attempts, approved-plan attempts) may be rewritten **only inside the phase that
owns them**, and that rewrite never competes with a plan decision.

## Red Flags — STOP

- About to edit a file before an Approved Plan exists → stop, finish the gate first.
- Letting Codex / Gemini / a subagent modify files → forbidden, single writer only.
- Sending raw `.env`, keys, customer data, or production logs to Gemini/Codex →
  sanitize first, gate through a manifest.
- Running `git stash` unscoped, or stashing the run directory → forbidden.
- Overwriting an existing `attempt-NN` artifact → write a new attempt instead.
- A loop exceeding its limit but continuing → it must become `BLOCKED_NEEDS_HUMAN`.
- Auto re-planning mid-implementation → not in v0.1; stop and escalate.
- Materializing a roadmap preset directory (`presets/`, review-gate, etc.) → README only.

## Common Mistakes

| Mistake | Correct behavior |
| ------- | ---------------- |
| Panel emits `REWRITE_PRIMARY_ARTIFACT_REQUIRED` | Only the pre-gate validator can; panel uses REVISE/RESEARCH/BLOCKED. |
| One manifest per call | One manifest per (target + data-scope); reuse across same-scope calls. |
| Hashing the data-scope slug | Use a short readable name, no hashes. |
| Skipping the run directory | Every invocation creates one; it is the audit trail. |
| Implementing on `REVISE_PLAN` | Only `APPROVE_PLAN` + an Approved Plan file unlock implementation. |
