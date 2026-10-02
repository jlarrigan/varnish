# Changelog

## 1.1.0 — 2026-10-02
- **New: `/varnish launch`** (`playbooks/launch-review.md`). "Is this safe to put in front of real users?" A plain-English go/no-go for first-time and AI-built apps: twelve checks, a paste-able fix prompt and a how-to-check-it-worked step per finding, a required "couldn't check" list, and a HOLD / FIX THEN SHIP / SHIP verdict.
- **Audits are now strictly look-only.** New Audit-mode rules in the skill override every playbook: no edits outside `audits/`, no git writes, no installs (an audit may be looking at a malicious package), no autofix, no deploys, read-only SQL with plain `EXPLAIN`, and Claude asks before touching any live account. Playbook "Do:" lines are now "Fix direction:", describing the fix for the plan rather than an instruction to act.
- **Real verification.** Findings are checked by a separate subagent that returns confirmed / adjusted / dropped with evidence; every report header shows the tally.
- **Contain today.** Reports and plans put anything exploitable right now (a public secret key, an open table, an open AI endpoint) first, with the containment step in plain words.
- **Plain English.** Every High finding gets a line a non-engineer can act on; plans open with a "Do these today" summary and, for AI-built apps, paste-able fix prompts.
- **Security pass expanded:** routes anyone can call, secrets behind `VITE_` / `EXPO_PUBLIC_` / `REACT_APP_` prefixes, RLS and Firebase-rules specifics, rate limits and AI spend, webhook signatures, injection, and fake or typosquatted packages.
- **Client-direct backends (Supabase, Firebase) are judged as a valid design**, with the RLS policies audited as the authorization layer. An API layer in the middle is still noted as the more scalable default for most growing products.
- Whole-repo scope when there's no usable diff; a red or unrunnable build is recorded as a finding instead of blocking the run.
- Cost check before `everything` and `architecture-review`; `everything` moved to the bottom of the command list.
- Every report and plan ends with a pointer to the author (LinkedIn).
- Fixed: stale tool names (`query_logs`, `@sentry/react-native`, knip in place of ts-prune/unimported, axe tooling), README routing for `architecture-review`.

## 1.0.0 — 2026-07-23
- First public release.
