# Controlled Coding Fusion Skill (`ccfusion`)

**English** | [繁體中文](README.zh-TW.md)

> Inspired by [OpenRouter Fusion](https://openrouter.ai/).

**A panel of AIs reviewing your plan beats one model charging in on its first guess — yes, even a top model like Fable 5.**

There's a Chinese saying: three humble cobblers outsmart the lone genius. `ccfusion`
runs on that idea — the same spirit as OpenRouter Fusion, which combines several
models into one better answer. Instead of letting one AI dive in and edit your code
on a hunch, it first puts the plan in front of a panel of AIs: one checks the
architecture, one the tests, one security, Codex plays devil's advocate, and Gemini
looks things up when current facts matter. A "judge" weighs every opinion and only
signs off once the serious problems are resolved. **Only then does a single AI
actually write the code.**

It won't promise the code is perfect. It just makes a dumb plan far less likely —
and shows you exactly how the plan got decided.

> Plan first. Let many review. Let one write.

```text
Many AIs may inspect and critique.
Only one — the Claude Code main session — may write.
```

### Why bother

- **Kill a bad plan early.** Problems get caught before a single file is touched —
  not after you've wasted an hour building the wrong thing.
- **Many reviewers, one writer.** Lots of AIs can poke holes; only one is ever
  allowed to touch your files. No surprise edits.
- **You can see the receipts.** Every run leaves a full, never-rewritten record of
  how the plan was reached.
- **It can't run away.** Every retry loop has a hard limit and stops for a human;
  secrets never get sent out; dangerous commands are blocked.

**Status: v0.1.0.** On purpose, it does exactly one thing: vet a coding plan before
anyone writes code. Nothing fancier yet.

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

## Design sentence

> Controlled Coding Fusion v0.1 is not a framework.
> It is one narrow Plan Gate that makes Claude Code think with help before it writes.

## License

Apache-2.0. See `LICENSE`.
