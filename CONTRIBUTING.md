# Contributing

Thanks for your interest. `ccfusion` v0.1.0 is **frozen**. Changes beyond bug
fixes belong to v0.2 and later.

## Scope discipline

The single validated use is coding-related plan gating. Before generalizing,
read the frozen spec's acceptance criteria. The most important rules:

1. Exactly one skill (`ccfusion`), one preset (`plan-gate`), one execution
   profile (`strict-single-writer`). No router.
2. **No `presets/` directory.** Do not create a preset directory before the
   preset is implemented. Roadmap items live in `README.md` only and must never
   be materialized as directories or implied to exist.
3. Pre-gate primary-input validation is the *only* place
   `REWRITE_PRIMARY_ARTIFACT_REQUIRED` may appear. It must not enter the panel
   decision model.
4. External-call manifests are per **(target + data-scope)** with readable slugs
   — no hashes.
5. Artifacts under `.fusion/runs/<run_id>/` are versioned `attempt-NN` and never
   overwritten. The run directory is never stashed, moved, or deleted.
6. All bounded-loop overflow → `BLOCKED_NEEDS_HUMAN`. Claude Code main session is
   the only executor.

A change that violates any acceptance criterion is out of scope for v0.1.

## Skill changes follow TDD

This skill was authored with the superpowers `writing-skills` methodology
(RED → GREEN → REFACTOR). Editing `SKILL.md` is editing behavior:

- Write a failing pressure scenario first (baseline: how does an agent behave
  *without* your change?).
- Make the minimal skill change that fixes the observed behavior.
- Re-test until the loophole is closed.

No untested skill changes — not even "just adding a section".

## Generalization gate

The general *Normalize → Panel → Judge → Decision* kernel is extracted only after
both Plan Gate **and** Review Gate have run on real repositories. Until then,
keep it narrow.

## License

By contributing you agree your contributions are licensed under Apache-2.0.
