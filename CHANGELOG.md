# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/), and this project adheres to
[Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-06-14

Frozen specification v0.1.0. First release. Intentionally narrow: one preset, one
execution profile.

### Added
- `ccfusion` skill (`.claude/skills/ccfusion/SKILL.md`) implementing the
  **Plan Gate** preset under the **Strict Single Writer** execution profile.
- Run directory model `.fusion/runs/<run_id>/` with deterministic `run_id`
  fallbacks (`no-commit`, `no-git`, `detached-or-unknown`) and never-overwritten
  `attempt-NN` artifacts.
- Pre-gate primary input validation (`PASS` /
  `REWRITE_PRIMARY_ARTIFACT_REQUIRED` / `BLOCKED_NEEDS_HUMAN`), kept strictly
  separate from panel decisions.
- External-call manifests per **(target + data-scope)** with readable slugs, no
  hashes.
- Read-only Claude subagent panel, read-only Codex adversarial review, and Gemini
  external research as an evidence layer (no gating decision).
- Plan Judge synthesis with most-severe-wins precedence
  (`BLOCKED_NEEDS_HUMAN > NEEDS_MORE_RESEARCH > REVISE_PLAN > APPROVE_PLAN`).
- Bounded plan-revision and research-reentry loops; all overflow →
  `BLOCKED_NEEDS_HUMAN`.
- Approved Plan artifact gating single-writer implementation; mid-implementation
  falsification stops and escalates (no auto re-plan in v0.1).
- 9 templates, 6 JSON schemas, 3 worked examples, plus `README.md`, `LICENSE`
  (Apache-2.0), `SECURITY.md`, `CONTRIBUTING.md`.
- "User Override" section in `SKILL.md` clarifying that an explicit user opt-out
  is honored (stop run cleanly, record, proceed outside the gate) while mere
  impatience/urgency is never an override. Added after subagent pressure-testing
  surfaced the gap; respects instruction priority without weakening the gate.
- "Candidate plan & plan-artifact lifecycle" note in `SKILL.md`: the panel and
  Plan Judge review the latest `approved-plan/attempt-NN.md`; an approval-time
  refinement (folding accepted non-blocking findings) writes a new attempt but
  does NOT increment `plan_revision_attempt_count`. Surfaced by an end-to-end
  dogfood run; a dedicated candidate-plan directory is deferred to v0.2.
- `.gitignore` ignoring `.fusion/` (per-run audit output is never committed),
  the local `/sandbox/` dogfood scaffold, and Python build artifacts.
- Validated end-to-end against a real toy repo: an APPROVE→implement run (real
  Codex read-only adversarial review), a research-path run (scoped manifest,
  reused across two calls; research pack consumed by the Judge — Gemini path
  exercised via a WebSearch stand-in, since the Gemini CLI was unavailable), and
  a bounded REVISE loop terminating in BLOCKED_NEEDS_HUMAN at the revision cap.

### Deliberately excluded (roadmap only; see `README.md`)
- Preset router / classification / multi-preset chaining.
- Review Gate, Research Fusion, Decision Fusion, Patch Fusion.
- Relaxed native execution profile; automatic mid-implementation re-planning;
  partial-patch preservation; parallel code modification.
- Codex/Gemini as executor; the general Normalize → Panel → Judge → Decision
  kernel.

There is intentionally no `presets/` directory.
