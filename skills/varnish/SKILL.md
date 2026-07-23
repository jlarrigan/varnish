---
name: varnish
description: Varnish the current repo — run an engineering audit, review, or remediation plan using the Varnish playbooks. A finishing coat for shipped code. Types — codebase audits by bucket (cleancode, architecture, security, reliability, scalability, delivery, accessibility, docs, all), feature (vertical per-feature audit), seo (marketing-site SEO + AI-search visibility), plan (turn findings reports into a remediation plan), architecture-review (the deep rebuild-vs-refactor review — distinct from the routine architecture bucket), compliance (SOC2/ISO/PCI/HIPAA precheck).
argument-hint: <type> [scope]
---

# Varnish — the finishing coat

_You varnish woodwork after you build it: a protective coat that makes the work last. Same idea here — the build is done; these passes protect it._

The user invoked `varnish $ARGUMENTS` in the current repo. Route to the matching playbook, read it fully, then execute it against this repo exactly as the playbook instructs. Every playbook is written to be *run*, not just read — it defines the passes, the report contract, and where output files go.

## Routing

Parse `$ARGUMENTS` as `<type> [scope...]` and dispatch:

| `<type>` | Playbook to read and run | Notes |
|---|---|---|
| `cleancode` · `architecture` · `security` · `reliability` · `scalability` · `delivery` · `accessibility` · `docs` · `all` (alias: `everything`) | `${CLAUDE_PLUGIN_ROOT}/playbooks/codebase-audit.md` | The bucket IS the type. Run the passes tagged with that bucket. `codebase <bucket>` also accepted. |
| `feature` | `${CLAUDE_PLUGIN_ROOT}/playbooks/feature-audit.md` | Scope = the feature name/paths, or diff against the base branch by default. |
| `seo` | `${CLAUDE_PLUGIN_ROOT}/playbooks/seo-audit.md` | For a marketing site. Scope = URL or site dir if given. |
| `plan` · `remediate` | `${CLAUDE_PLUGIN_ROOT}/playbooks/remediation-plan.md` | Scope = a report file in `audits/`, or "everything since <date>". Default: all reports newer than the newest existing plan; if no plan exists yet, all reports in `audits/`. |
| `architecture-review` | `${CLAUDE_PLUGIN_ROOT}/playbooks/architecture-review.md` | The deep, run-rarely judgment call. NOT the routine `architecture` bucket — confirm with the user if ambiguous (see below). |
| `compliance` | `${CLAUDE_PLUGIN_ROOT}/playbooks/compliance-review.md` | CIS v8 → SOC2/ISO/PCI/HIPAA precheck. |

**Also load** `${CLAUDE_PLUGIN_ROOT}/playbooks/_report-shape.md` whenever running any Audit type (the first three rows) — it is the shared report contract every Audit writes. A `compliance` run additionally loads `codebase-audit.md` (its reuse rows run passes from there); a `plan` run may load `_report-shape.md` (the contract of the reports it consumes). _(If not installed as a plugin, resolve these paths relative to this file's repo: `playbooks/` sits two levels up.)_

**Ambiguity rule:** `varnish architecture` means the routine codebase-audit *bucket*. If the user's phrasing suggests the deep review ("should we rebuild", "review the architecture", "is the design right"), ask one clarifying question before running — the two differ in cost by an order of magnitude.

**No/unknown type:** show the table above in one compact list with a one-line description each, and ask which to run. Do not guess between Audits and Reviews.

## Ground rules (apply to every run)

1. **Run in the target repo.** All output paths (`audits/…`) are relative to the current repo root. Create `audits/` if missing.
2. **Audits are read-only** — find, don't fix. The fixing happens after the remediation plan, in normal sessions. (On audit runs, the codebase-audit's Finish step reduces to re-running the green checks and writing the report — the commit/ship/re-measure half belongs to the later fix phase.)
3. **On Audit runs, adversarially verify findings** before writing the report, exactly as the playbook specifies. (Reviews and plans use their own verification models — a plan explicitly does NOT re-verify.)
4. **Respect prior decisions:** before writing findings, check `audits/*-plan-*.md` for Accepted-Risk Ledgers and Disputed sections, per `_report-shape.md`.
5. **After any Audit completes**, close by offering the next step: _"Run `varnish plan` to turn this report into a remediation plan."_
6. The playbooks' code examples are grounded in one example stack (React Native · serverless API · Postgres). Treat them as **principles** and translate to THIS repo's stack; mark inapplicable passes N/A with a reason, never force a finding.
