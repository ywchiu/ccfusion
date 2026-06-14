# Controlled Coding Fusion Skill (`ccfusion`)

**English** | [繁體中文](README.zh-TW.md)

> Inspired by [OpenRouter Fusion](https://openrouter.ai/).

A Claude Code Skill for safer AI-assisted coding **planning**.

> Plan first. Review in parallel. Execute with one writer.

```text
Many agents may inspect and critique.
Only Claude Code main session may execute.
```

`ccfusion` raises planning quality **before any file is touched** and leaves an
auditable trail of how the plan was reached. It does **not** guarantee
correctness — it gates the plan, then lets a single writer implement.

**Status: v0.1.0 (frozen).** Intentionally narrow: exactly one preset
(`plan-gate`) under exactly one execution profile (`strict-single-writer`). It is
not a general fusion framework, not a router of presets, and not a multi-agent
runtime.

---

## Install

This repository ships the skill at:

```text
.claude/skills/ccfusion/SKILL.md
```

Place the `ccfusion` skill directory under your Claude Code skills path (project
`.claude/skills/` or personal `~/.claude/skills/`). Then ask Claude Code to "use
controlled coding fusion" or invoke `/ccfusion`.

## What it does

```text
User task
  -> Create run directory + run-state.json
  -> Create Context Pack
  -> Pre-gate primary input validation        (PASS required to proceed)
  -> Optional Gemini external research          (scoped manifest required)
  -> Read-only internal repo review (Claude subagents)
  -> Codex adversarial review
  -> Plan Judge synthesis (most-severe-wins)
  -> APPROVE_PLAN -> Approved Plan -> Claude Code main session may implement
     REVISE_PLAN          -> plan revision loop (bounded)
     NEEDS_MORE_RESEARCH  -> research reentry loop (bounded)
     BLOCKED_NEEDS_HUMAN  -> stop, write final summary, escalate
```

Every loop is bounded; every overflow ends in `BLOCKED_NEEDS_HUMAN`. The Plan
Gate never edits files. Implementation begins only after an Approved Plan
artifact exists.

## Roles

| Role | Capability |
| ---- | ---------- |
| Claude Code main session | orchestrator, judge, planner, **and the only executor** |
| Claude Code read-only subagents | internal repository reviewers (no file modification) |
| Codex plugin | adversarial reviewer only (no file modification) |
| Gemini | external research only (no file modification, no gating decision) |
| Tests / CI | the objective validator, after implementation |

## Run directory

Each invocation creates one run directory — the audit trail, never stashed,
moved, or deleted:

```text
.fusion/runs/<run_id>/
```

`run_id` = `YYYYMMDDTHHMMSSZ-<repo_slug>-<git_ref>-<random6>`, with deterministic
`git_ref` fallbacks (`no-commit`, `no-git`, `detached-or-unknown`). Artifacts are
versioned and never overwritten; a regenerated artifact is a new `attempt-NN`.

## Layout

```text
.claude/skills/ccfusion/
  SKILL.md
  templates/   context-pack, input-validation, external-call-manifest,
               gemini-research-pack, reviewer-output, codex-adversarial-review,
               plan-judge, approved-plan, final-summary
  schemas/     context-pack, input-validation, external-call-manifest,
               reviewer-output, plan-judge, approved-plan  (JSON Schema)
  examples/    bug-fix, dependency-upgrade, refactor
```

There is intentionally **no `presets/` directory** in v0.1.

## Safety boundary

- Only the Claude Code main session may modify files.
- External calls are gated per **(target + data-scope)** manifest with a readable
  slug (no hashes); secrets and customer data never leave — a sanitized technical
  summary is sent instead. See `SECURITY.md`.
- Destructive commands and unscoped `git stash` are forbidden without explicit
  approval.

## Roadmap (NOT implemented in v0.1)

These are future directions only. **They do not exist yet** — no directories,
presets, or code for them are present in this repository, and `ccfusion` will not
behave as if they do.

```text
v0.2 Review Gate
  post-implementation Codex review; P0/P1/P2 classification; repair loop;
  add-tests loop; mid-implementation re-plan with path-scoped partial-patch
  preservation; review-gate total attempt backstop;
  dedicated candidate-plan artifact directory (separate from approved-plan/) so
  the candidate vs approved distinction is structural, not just by attempt number

v0.3 Research Fusion
  independent research preset; multi-source comparison; source quality scoring

v0.4 Decision Fusion
  cost / risk / time tradeoff panel; option comparison; decision memo

v0.5 Patch Fusion
  isolated git worktree execution; multiple candidate patches; test-based selection
```

**Roadmap rule:** do not create a preset directory before implementing the
preset. The general *Normalize → Panel → Judge → Decision* kernel, if it proves
real, is extracted only after Plan Gate and Review Gate have both run on real
repositories.

## Design sentence

> Controlled Coding Fusion v0.1 is not a framework.
> It is one narrow Plan Gate that makes Claude Code think with help before it writes.

## License

Apache-2.0. See `LICENSE`.
