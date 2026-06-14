# Security

`ccfusion` is a planning gate that coordinates external agents (Gemini for
research, Codex for adversarial review) and read-only Claude subagents. Its
security model is about **what data leaves the working tree** and **who may write
to it**.

## Single-writer guarantee

Only the Claude Code main session may modify files. Claude subagents, Codex, and
Gemini are read-only for the entire run. The Plan Gate itself never edits files;
implementation begins only after an Approved Plan artifact exists.

## External-call data boundary

External-call safety is reviewed **per (target + data-scope)** via a manifest
with a short human-readable slug (no hashes). Calls sharing the same target and
data scope reuse one manifest; a broader data scope requires a new manifest.

### Data that must NOT leave (default deny)

```text
.env files, API keys, private keys, credentials
customer records, customer contracts, personal data
raw production logs, database dumps
internal / regulated financial data
proprietary datasets
military or defense-sensitive data
```

If external research is needed, send a **sanitized technical summary**, never raw
material. Each manifest records `User / Customer Data Present?` and
`Secrets Present?` explicitly; both should be `No` before any call.

## Destructive-command boundary

Forbidden without explicit user approval:

```text
rm -rf            git push --force   git reset --hard   git clean -fd
drop database     truncate table     delete from        terraform apply
kubectl delete    kubectl apply      docker system prune
production deploy scripts            database migration scripts
```

Unscoped `git stash` / `git stash push` is forbidden. Stash is allowed only with
an explicit source path scope, and the run directory `.fusion/runs/<run_id>/` is
never stashed, moved, or deleted — it is the audit trail.

## Open-source boundary

This repository ships only the generic skill, templates, schemas, toy examples,
security guidance, and README. Company-specific workflows, real prompts, customer
rubrics, internal CI/deploy scripts, real customer logs, real production bugs, and
model-routing/cost strategy are **kept private** and must not be committed here.

## Reporting a vulnerability

Open a private report to the maintainers rather than a public issue if a finding
could expose secrets or customer data through the external-call path.
