# External Call Manifest

> One manifest per (target + data-scope), NOT per call. Calls sharing the same
> target and data scope reuse this manifest. A broader data scope needs a new
> manifest. Filename: `manifests/<target>__<data-scope-slug>.md` (readable slug,
> no hashes). Schema: `schemas/external-call-manifest.schema.json`

## Target
<!-- Gemini | Codex | Other -->

## Data Scope Name
<!-- Short human-readable slug describing what data is allowed to leave,
     e.g. public-docs-no-code, sanitized-error-trace, sanitized-diff-summary -->

## Purpose
<!-- Why this external call is needed. -->

## Data Allowed To Leave
<!-- Exactly what may be sent. Sanitized technical summaries only. -->
-

## Data Explicitly Excluded
<!-- What must NOT be sent. -->
-

## Redaction Method
<!-- How sensitive material was removed or summarized before sending. -->

## Sensitive Data Check
<!-- Result of scanning the outbound payload. -->

## User / Customer Data Present?
<!-- Yes / No -->

## Secrets Present?
<!-- Yes / No -->

## Approved For Calls
<!-- call-001, call-002, ... that this manifest covers. -->
-

## Reviewer Notes
<!-- Any caveats for future reuse. -->
