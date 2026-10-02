---
name: varnish
description: Varnish the current repo — run an engineering audit, review, or remediation plan using the Varnish playbooks. A finishing coat for shipped code. Types — launch (is this app safe to put in front of real users? plain-language go/no-go, built for people new to shipping), codebase audits by bucket (cleancode, architecture, security, reliability, scalability, delivery, accessibility, docs, all), feature (vertical per-feature audit), seo (marketing-site SEO + AI-search visibility), plan (turn findings reports into a remediation plan), architecture-review (the deep rebuild-vs-refactor review — distinct from the routine architecture bucket), compliance (SOC2/ISO/PCI/HIPAA precheck), status (progress across past runs: fixed, regressed, accepted, still open). Add `learn` to any run for a short lesson with each finding.
argument-hint: <type> [scope] [learn]
---

# Varnish — the finishing coat

_You varnish woodwork after you build it: a protective coat that makes the work last. Same idea here — the build is done; these passes protect it._

The user invoked `varnish $ARGUMENTS` in the current repo. Route to the matching playbook, read it fully, then execute it against this repo as the playbook instructs, **under the Audit-mode rules below, which override anything a playbook says.**

## Routing

Parse `$ARGUMENTS` as `<type> [scope...]` and dispatch:

| `<type>` | Playbook to read and run | Notes |
|---|---|---|
| `launch` (alias: `ready`, `safe`) | `${CLAUDE_PLUGIN_ROOT}/playbooks/launch-review.md` | **Is this safe to put in front of real users?** Plain-language go/no-go. The right first run for anyone new to shipping. |
| `cleancode` · `architecture` · `security` · `reliability` · `scalability` · `delivery` · `accessibility` · `docs` · `all` (alias: `everything`) | `${CLAUDE_PLUGIN_ROOT}/playbooks/codebase-audit.md` | The bucket IS the type. Run the passes tagged with that bucket. `codebase <bucket>` also accepted. |
| `feature` | `${CLAUDE_PLUGIN_ROOT}/playbooks/feature-audit.md` | Scope = the feature name/paths, or diff against the base branch by default. |
| `seo` | `${CLAUDE_PLUGIN_ROOT}/playbooks/seo-audit.md` | For a marketing site. Scope = URL or site dir if given. |
| `plan` · `remediate` | `${CLAUDE_PLUGIN_ROOT}/playbooks/remediation-plan.md` | Scope = a report file in `audits/`, or "everything since <date>". Default: all reports newer than the newest existing plan; if no plan exists yet, all reports in `audits/`. |
| `architecture-review` | `${CLAUDE_PLUGIN_ROOT}/playbooks/architecture-review.md` | The deep, run-rarely judgment call. NOT the routine `architecture` bucket — confirm with the user if ambiguous (see below). |
| `compliance` | `${CLAUDE_PLUGIN_ROOT}/playbooks/compliance-review.md` | CIS v8 → SOC2/ISO/PCI/HIPAA precheck. |
| `status` (alias: `progress`) | `${CLAUDE_PLUGIN_ROOT}/playbooks/status.md` | Progress across past runs in `audits/`: fixed, came back, accepted, still open. Cheap; reads reports, spot-checks the code. |

**Also load** `${CLAUDE_PLUGIN_ROOT}/playbooks/_report-shape.md` whenever running any Audit type (codebase buckets, `feature`, `seo`) — it is the shared report contract every Audit writes — and on `launch` (for the severity scale, the contain-today rules, and the footer). A `compliance` run additionally loads `codebase-audit.md` (its reuse rows run passes from there); a `plan` run may load `_report-shape.md` (the contract of the reports it consumes). _(If not installed as a plugin, resolve these paths relative to this file's repo: `playbooks/` sits two levels up.)_

**Stack modules.** Before running any Audit, Review, or `launch`, detect the stack and load the matching modules from `${CLAUDE_PLUGIN_ROOT}/playbooks/stacks/` — the signals table is in `stacks/_index.md`. Load only the modules that match; record them in the report header. (`plan` and `status` don't need them.)

**Learn mode.** If `learn` appears anywhere in `$ARGUMENTS` (e.g. `varnish security learn`), strip it from the scope and run the playbook with Learn mode on, as defined in `_report-shape.md`. `launch` runs with Learn mode on by default.

**Ambiguity rule:** `varnish architecture` means the routine codebase-audit *bucket*. If the user's phrasing suggests the deep review ("should we rebuild", "review the architecture", "is the design right"), ask one clarifying question before running — the two differ in cost by an order of magnitude.

**No/unknown type:** show the table above in one compact list with a one-line description each, and ask which to run. Do not guess between Audits and Reviews. **If the user seems new to shipping** (no tests, no CI, built with Lovable/Bolt/Replit/v0, asks "is my app safe?"), recommend `launch` first.

**Cost check:** `all` / `everything` and `architecture-review` are long, expensive runs. Before starting one, say so in one line and confirm. If the repo looks like a first-time or AI-built app, suggest `launch` instead — it answers the question they actually have, faster.

## ⛔ Audit mode — hard rules (override every playbook)

Every Varnish run except the later fix phase is **look-only**. Playbooks describe fixes as *direction for the plan*; they are never instructions to act during the run. When a playbook line says "Do:", "Fix direction:", "delete", "add", "rotate", "commit", or "ship", read it as **what the eventual fix looks like**, and write it into the report — don't do it.

1. **The only files you may create or change are under `audits/`** in the target repo (create the folder if missing). No edits anywhere else, including config, lockfiles, `.gitignore`, and docs.
2. **No git writes.** No branch, commit, stash, checkout, reset, push, or tag. Reading (`git log`, `git diff`, `git show`, `git ls-files`) is fine.
3. **No installs.** Never run `npm/pnpm/yarn/bun install`, `npx`/`pnpm dlx` of a package that isn't already installed, `pip install`, or similar. An audit may be looking at a malicious or typosquatted dependency; installing it runs its code. If a tool or `node_modules` isn't present, skip that check, inspect statically instead, and note it.
4. **No autofix.** Run linters/formatters only in check mode (no `--fix`, `--write`).
5. **No deploys or production changes.** Never run promote, rollback, redeploy, OTA publish/republish, env-var edits, or anything that changes a live service.
6. **Databases are read-only.** SQL is `SELECT` (and catalog reads) only. Use plain `EXPLAIN`, never `EXPLAIN ANALYZE` (it executes the statement, writes included). Never run migrations, seeds, or `apply_migration`.
7. **Ask before touching any live account.** Before the first call to a connected service (Supabase, Vercel, Stripe, Firebase, a cloud console, any MCP tool that reaches a real project), say which project you're about to read and get a yes. If it looks like production, say so.
8. **If the build or tests are red, or can't run:** record that as a finding in the baseline, keep going with the static checks, and skip the ones that need a working build. Never try to make it green.

The fixing happens after the remediation plan, in normal sessions, where the user is in charge.

## Other ground rules

1. **Run in the target repo.** All output paths (`audits/…`) are relative to the current repo root.
2. **Adversarially verify findings** before writing the report, exactly as the playbook specifies, and report the tally (N raw → M kept, K dropped, J adjusted, A added by the verifier). (Reviews and plans use their own verification models — a plan explicitly does NOT re-verify.)
3. **Respect prior decisions** (Audits and `launch`): before writing findings, check `audits/*-plan-*.md` for Accepted-Risk Ledgers and Disputed sections, per `_report-shape.md`.
4. **After any Audit completes**, the report's own Handoff section carries the next step (don't add a second one to the file); in the chat, offer: _"Run `varnish plan` to turn this report into a remediation plan."_ (`launch` uses its own next-step text, which already includes this.)
5. **Version stamps** use the `version` in `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json`.
6. The playbooks' code examples are grounded in one example stack (React Native · serverless API · Postgres). Treat them as **principles** and translate to THIS repo's stack; mark inapplicable passes N/A with a reason, never force a finding. **Client-direct backends (Supabase, Firebase) are a valid architecture** — see pass 5 in `codebase-audit.md` for how to judge them.
7. **Every report and plan ends with the author footer** defined in `_report-shape.md`.
