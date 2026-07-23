# Compliance Review — technical precheck

_Run your tech audit, see roughly where you stand against the common compliance frameworks. A **technical precheck / gut-check** — to get ahead of a real audit, not to replace one. Part of Varnish Tools._

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say:** _"Read this file and run the compliance precheck on this repo."_

Read-only. It assesses the technical controls below (reusing the `codebase-audit.md` passes where noted, running the **NEW** ones inline), maps each to **CIS Controls v8**, and from CIS's crosswalk fills the SOC2 / ISO 27001 / PCI DSS / HIPAA columns. Output → `audits/<YYYY-MM-DD>-compliance-<HHMM>.md` as the crosswalk matrix.

## ⚠️ What this is — and isn't
- A **technical precheck**, NOT a SOC2 / PCI / ISO / HIPAA audit or attestation. Those require a licensed auditor *and* the org/process controls a repo can't see (HR, policies, training, vendor mgmt, physical security, incident-response *process*…).
- It covers only the **repo/infra-observable slice** of each framework. Report it as *"% of technically-assessable controls covered"* per framework — **never "you pass X."**
- Backbone = **CIS Controls v8** (engineer-grade, prioritized by Implementation Group; it ships official crosswalks to the other frameworks, so the columns come from CIS, not from guesses). The mappings below are **indicative / precheck-grade** — verify against CIS's official mapping before any external use. ISO = 27001:2022 Annex A; PCI = DSS 4.0; HIPAA = the Security Rule (§164.3xx).

## The controls (repo/infra-observable CIS v8 safeguards)

`Source`: **reuse** = an existing `codebase-audit.md` pass already checks this; **NEW** = a compliance-driven check not in the routine audit yet (should migrate into its bucket over time — see Notes).

| CIS v8 | What to check (technical) | Source | SOC2 | ISO 27001:2022 | PCI DSS 4.0 | HIPAA |
|---|---|---|---|---|---|---|
| 3.10 Encrypt in transit | TLS/HTTPS enforced everywhere; HSTS; no plaintext endpoints | NEW | CC6.7 | A.8.24 | 4.2 | §164.312(e)(1) |
| 3.11 Encrypt at rest | DB/storage encryption on; sensitive fields/secrets not plaintext | NEW | CC6.1 | A.8.24 | 3.5 | §164.312(a)(2)(iv) |
| 4.1 Secure configuration | No default creds; security headers; hardened framework/infra config | NEW | CC6.1 | A.8.9 | 2.2 | §164.312(a)(1) |
| 6.3 / 6.5 MFA (incl. admin) | MFA enforced for users and especially admin/console (auth-provider config) | NEW | CC6.1 | A.8.5 | 8.4 / 8.5 | §164.312(d) |
| 6.8 Least-privilege / RBAC | RLS / authz restricts to least privilege; no over-broad grants or anon-callable privileged RPCs | reuse (security) | CC6.3 | A.5.15 / A.8.2 | 7.2 | §164.312(a)(1) |
| 7.3 / 7.4 Automated patching | Dependabot/Renovate (or equiv) keeps deps current; no long-stale critical deps | NEW (+ reuse dep-hygiene) | CC7.1 | A.8.8 | 6.3.3 | §164.308(a)(1) |
| 7.5 / 7.6 Vuln scan & remediate | `pnpm/npm audit` (or SCA) run in CI; no unremediated high/critical vulns | reuse (dep hygiene) | CC7.1 | A.8.8 | 6.3.1 / 11.3 | §164.308(a)(1)(ii)(A) |
| 8.2 / 8.5 Audit logging | App + access/security-event logs captured, structured, retained | reuse (observability) + NEW (security-event log) | CC7.2 | A.8.15 | 10.2 | §164.312(b) |
| 8.11 Log review / alerting | Error/anomaly alerting wired to a human channel | reuse (reliability: alerting) | CC7.2 / CC7.3 | A.8.16 | 10.4 | §164.308(a)(1)(ii)(D) |
| 11.2 Automated backups | Backups/PITR configured for the durable store (e.g. Supabase PITR) | reuse (reliability: durability) | A1.2 | A.8.13 | 12.10.1 | §164.308(a)(7)(ii)(A) |
| 11.5 Test data recovery | A restore has been tested / is documented (not just "backups exist") | NEW | A1.3 | A.8.13 | — | §164.308(a)(7)(ii)(B) |
| 16.11 Secrets management | No hardcoded secrets; env/secret store used; keys rotatable | reuse (security: secrets) | CC6.1 | A.8.24 / A.5.33 | 8.3 / 6.x | §164.312(a)(2)(iv) |
| 16.12 Input validation | Untrusted input validated at every boundary (parse, don't assume) | reuse (pass 6/8 boundary) | CC8.1 | A.8.28 | 6.2.4 | §164.312(c)(1) |
| 16.x SAST / scanning in CI | SAST / secret-scan / dependency-scan run in the CI pipeline | NEW (+ reuse CI pass) | CC8.1 | A.8.25 / A.8.28 | 6.3.2 | §164.312(c)(1) |
| Change control (16/8) | Protected `main`, PR review, CI gates = auditable change management | reuse (delivery: CI-as-gate) | CC8.1 | A.8.32 | 6.5 | §164.308(a)(1) |
| 2.x Software inventory | Lockfile + dependency inventory; SBOM available | reuse (dep hygiene) + NEW (SBOM) | CC7.1 | A.8.8 | 6.3.2 | §164.310(d) |

_Out of repo scope (mark N-A with that reason, don't invent): HR on/offboarding, security training, vendor/risk management, physical security, incident-response **process**, formal policies — the org/process majority of every framework._

## Output: the crosswalk matrix
Write `audits/<date>-compliance-<HHMM>.md`:
- **Header** — date · "technical precheck, not an attestation" · CIS v8 backbone · which framework columns.
- **Per control** — the row above + a **status**: ✅ covered · ⚠️ gap (with the fix) · N-A (out of repo scope, with reason). One status drives all framework columns (it's the same evidence).
- **Per-framework summary** — *"X of Y technically-assessable controls covered (Z%)"* for SOC2 / ISO / PCI / HIPAA. Plain about the denominator: it's the repo-observable slice only.
- **Top gaps** — the ⚠️ rows, ranked, as the get-ahead-of-the-audit to-do list.

## Notes
- **Reuse vs. NEW:** the `reuse` rows are answered by running the named `codebase-audit.md` passes; the `NEW` rows are compliance-specific checks. For v1 they're defined here. Over time they migrate into their natural bucket in `codebase-audit.md` (MFA → `security`, audit-logging → `reliability`, restore-test → `reliability`…), each carrying a CIS tag — and this file becomes the pure lookup table.
- **It's conformance, not design.** This sits alongside the **Codebase Audit** (does the current code meet control X?), not the **Architecture Review** (is the design right / rebuild?).
- **Versions move.** Controls evolve and sometimes contradict (e.g. forced password rotation is now *discouraged* by NIST 800-63B). Re-check mappings when a framework version bumps.

---
_v1, 2026-06-27. Indicative precheck-grade mappings — anchor on the official CIS Controls v8 crosswalk before any external/sellable use._
