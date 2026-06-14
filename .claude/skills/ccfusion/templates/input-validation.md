# Input Validation (Pre-Gate)

> Runs BEFORE any Gemini, Codex, or subagent call. Validates the primary input
> (Context Pack). This is the line between "the input is unusable" and "the plan
> needs revision". Schema: `schemas/input-validation.schema.json`

## Decision
<!-- One of: PASS | REWRITE_PRIMARY_ARTIFACT_REQUIRED | BLOCKED_NEEDS_HUMAN -->

## Validated Artifact
<!-- Path to the primary artifact under review, e.g. input/context-pack.md -->

## Missing Or Invalid Fields
<!-- Specific fields that are absent, ambiguous, or contradictory. Empty if PASS. -->
-

## Findings
<!-- What was checked and what was found. -->
-

## Rewrite Instructions
<!-- Only if REWRITE_PRIMARY_ARTIFACT_REQUIRED: exactly what to fix. -->

## Counter State
<!-- primary_artifact_rewrite_count after this validation, and the limit. -->
- primary_artifact_rewrite_count:
- max_primary_artifact_rewrites: 2

## Rationale
<!-- Why this decision. If BLOCKED_NEEDS_HUMAN, state what a human must resolve. -->
