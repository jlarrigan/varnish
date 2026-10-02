# Codebase Audit

_The cleanup-and-hardening pass you run after shipping a big feature — or a pile of small ones. The deliberate paying-down of the entropy a ship sprint leaves behind._

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say:** _"Read this file and run the `cleancode` audit on this repo."_ (Swap in any bucket below, or `all`.)

Claude will: take the passes tagged with that bucket → detect the stack and load its modules → record the baseline (what builds, what's tested) → run them **look-only** (find, don't fix; see the Audit-mode rules in `skills/varnish/SKILL.md`) → **adversarially verify every finding** (a fresh skeptic greps the repo to confirm each claim and right-size severity) → write the *verified* findings to `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` → stop and show you the report. You then run `remediation-plan.md` on the report to decide solutions, and fix in chunks. _(Run it in the target repo's own session — the fixing happens there.)_

**Buckets** — every pass below carries a `> Buckets:` tag; a bucket just runs the passes tagged with it. Pass 0 + the Finish step always run. The Passes column is a shortcut so you can read just those sections; the tags are the source of truth.

| Bucket | What it checks | Passes |
|---|---|---|
| `cleancode` | **hygiene only** — lint, dead code, unused deps, DRY, type-hygiene, naming/clarity, test-hygiene, comment-truth, magic-numbers _(the biggest line-count wins live here)_ | 1, 2, 3, 4, 6, 9, 11, 30, 31 |
| `architecture` | layering, separation of concerns, contracts, idempotency, boundaries, testability/coverage | 5, 7, 15 |
| `security` | who can call what, secrets in code and in the browser, database rules (RLS), rate limits and AI spend, webhooks, PII/privacy, dependency vulns and fake packages | 3, 10, 15, 20, 28 |
| `reliability` | errors, observability, SLO/health, alerting, durability, blast radius _(SRE)_ | 8, 12, 13, 14, 15, 24, 25, 26 |
| `scalability` | perf, cost, N+1s, caching, cold-starts | 16, 23 |
| `delivery` | deploy/rollback, migrations, OTA, CI, flags, env | 3, 16, 17, 18, 19, 20, 21, 22 |
| `accessibility` | a11y regressions | 27 |
| `docs` | docs / agent-instruction currency | 29 |
| `all` | every pass | 0–31 |
| `compliance` | _(different mode)_ run the technical compliance precheck → see `compliance-review.md`. Reuses the security/reliability/delivery checks, maps them to CIS v8 → SOC2/ISO/PCI/HIPAA, and renders a crosswalk matrix instead of a fix-list. |
| `seo` | _(different target)_ the **marketing-site visibility** audit → see `seo-audit.md`. Classic SEO + GEO (AI-answer citation readiness) with evidence-tiered findings and a cargo-cult ledger. Run it on the site, not the app repo. |

For the deep "is this built right / rebuild vs refactor?" audit, use `architecture-review.md` instead.

**Scope.** If there's a meaningful base to diff against (the last release tag, else the merge-base with the default branch), the "this ship" passes look at what changed since then; otherwise — no tags, one long-lived branch, AI-builder commit history — **audit the whole repo**. Record the base ref and HEAD SHA (or "whole repo") in the report header either way.

**Fix lines are direction, not actions.** Each pass's `Fix direction:` line describes what the eventual fix looks like, so the report can point the right way. Nothing gets changed during the audit — the fixing happens after `remediation-plan.md`, in a normal session.

**Stack modules.** The passes are written stack-neutral. Detect the stack and load the matching `stacks/*.md` modules (signals in `stacks/_index.md`); each adds the exact files, settings, and traps for its platform, keyed by pass number. Passes 17 and 19 exist only inside the modules that define them.

**Ground rule:** if a pass doesn't apply, mark it **N/A** in the report (don't force a finding) — wide coverage is the point.

## Findings are adversarially verified (built in, not optional)
Don't trust a raw finding. After the find phase, **every finding is re-checked by a separate verifier whose job is to *disprove* it.** Concretely:
1. Spawn a **fresh subagent** (not the context that found them). Give it only the findings list — ID, claim, `file:line` — not the finder's reasoning.
2. For each finding it returns `confirmed`, `adjusted` (severity or claim corrected), or `dropped`, **with the evidence**: the quoted line or the command output that settles it.
3. Drop what doesn't survive; apply the adjustments.
4. The report header states the tally: **"N raw → M kept (K dropped, J adjusted, A added by the verifier)."** No tally means it wasn't verified — don't write "adversarially verified" without one.

If subagents aren't available, do the verification as a separate, explicit second pass and say so in the header. *A report of 30 verified findings is worth ten times a report of 50 plausible ones.*
- `cleancode` → the verifier proves claims by grep/inspection ("is this export really unreferenced?").
- Other buckets adapt the same skeptic stance: `security` → "can this actually be exploited?"; `reliability` → "can this failure really happen, and what's the blast radius?".

## Report format
Write the report in the shared **Audit shape** — see [`_report-shape.md`](./_report-shape.md). In brief: **findings-first** (the report detects, the remediation plan decides), every finding tagged **`[Severity · Cost-of-doing-nothing]`**, findings **grouped by area** so clusters show, and a closing **→ handoff to `remediation-plan.md`**. No prescribed fix-order, no per-finding effort — that's the remediation plan's job (it's also where the one upstream fix that kills a whole cluster gets spotted).

**Filename:** `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` — date + **bucket** + the run's **HHMM** (from the shell, so same-day re-runs don't collide). e.g. `audits/2026-06-25-cleancode-1455.md`.

**On top of the shared shape, the Codebase Audit adds:**
- **Header** carries the **bucket** + **playbook**.
- **Pass 0 baseline metrics** — LOC, tests, typecheck, bundle size, captured up top. (Measurement, not a decision — it belongs.)
- **Findings grouped by area** (per `_report-shape.md`), each tagged with its pass; a short pass list up top marks each pass `Applicable` or `N/A — why`; each finding gets `[Severity · Cost-of-doing-nothing]` + `file:line` + what-and-why, plus a ✅ *Clean:* line for what passed.
- **Finish** — on the audit run, nothing more than the report. Re-measuring deltas against Pass 0 happens after fixes land, in the fix phase.

The two axes are the triage signal: **Severity** (how bad if it bites) × **Cost of doing nothing** (what leaving it costs) tells you backlog-vs-queue without guessing at a fix.

### Codebase health scorecard — on an `all` run
On an **`all`** run, top the report with a scorecard grading each of the **8 buckets** A–D — the repo-scale twin of the Feature Audit's 8-dimension scorecard, and the at-a-glance "is this repo in good shape?" read. **Single-bucket runs skip it** — the findings headline already carries one bucket.

It's a **rough health dashboard**, not a precise judgment: a whole bucket across a whole repo has no single "core contract" the way one feature's slice does, so the grade is coarser by nature. The real precision lives in the findings beneath it. Grade from the *verified* findings:
- **A** — clean (no findings, or only trivial ones).
- **B** — minor: some Low/Med findings, nothing High.
- **C** — at least one High finding, *or* a systemic pile of Meds (the same issue repeated repo-wide).
- **D** — multiple Highs, a repo-wide High, or the bucket is broadly broken.
- **N/A** — bucket not run / not applicable to this repo.

Example (`all` run):
```
CODEBASE: <repo>   scope: whole repo   (all buckets)

Headline: 41 findings (4 High). Worst: reliability (D — no API error capture +
silent-failing cron) and security (C). cleancode strong (A).

Bucket          Grade  Headline
cleancode         A    lint clean; ~30 dead exports (trivial); no duplicated logic
architecture      B    one god-provider; otherwise layered cleanly
security          C    one client-only authz check; the rest server-gated
reliability       D    no error capture in the API; a cron fails silently
scalability       B    two unindexed FKs on growing tables
delivery          B    migrations forward-only (no down-path)
accessibility     A    new screens pass the WCAG checks
docs              C    CLAUDE.md drifted from the new env setup

(findings, grouped by pass, follow…)
```

## When to run it
- Right after a big feature lands, or after a stretch of rapid small ships.
- Before a release you want to be proud of.
- When the repo "feels heavy" — slow builds, fear of touching things, copy-paste creeping in.
- On a cadence (end of a milestone) so entropy never compounds.

## The mindset
When you ship fast, you trade cleanliness for speed *on purpose*: duplication, dead paths, loosened types, swallowed errors, half-abandoned experiments. That's fine in the moment — but it compounds. This audit is the scheduled repayment. Run it as a **dedicated pass, separate from feature work**, so the diffs are reviewable and the risk stays contained.

**Real example:** run on a shipped production React Native app, this removed ~20k lines and cut the app size roughly in half. Dead-code and duplication do most of the heavy lifting.

## Two rules that keep it safe
1. **The audit only looks.** No edits outside `audits/`, no git writes, no installs, no autofix, no deploys, read-only SQL (full list: the Audit-mode rules in `skills/varnish/SKILL.md`). If the build is red or can't run, that's a baseline finding — note it, run the static checks, skip the rest. Don't try to make it green.
2. **When you fix later: start green, end green, one pass = one (or a few) commits.** Confirm tests + typecheck + build pass before touching anything, so any regression is obviously yours, and keep each change independently verifiable so you can bisect what broke.

## Execution: order & fan-out
- **Solo:** work the bucket's passes top-to-bottom (the order matters).
- **Agent fan-out:** spin up **one agent per bucket** (not per pass — passes shared across buckets would run twice), each returning a findings list with `pass`, `file:line`, claim, and suggested `[Severity · Cost]`. The main context dedupes by location and class, runs the verifier, and writes the report. Every agent works under the Audit-mode rules.

**Order matters:** format/lint first (clears cosmetic noise), then dead-code (shrinks the surface area everything else has to scan), then the structural/correctness passes, then a full verification pass last. (This orders the *passes you run*, not the *findings you fix* — the report stays grouped, not pre-sequenced; fix-order is the remediation plan's call.)

---

## The passes

### 0. Baseline / safety net
> **Buckets:** always (runs with every audit)
- **Record what you find; change nothing.** If dependencies are already installed, run the existing test / typecheck / build / lint scripts in check mode and record pass or fail. If they aren't installed, **don't install them** — note "not run (deps not installed)" and continue statically.
- **No tests, no CI, or a red build is a finding, not a blocker.** Write it up and keep going.
- Snapshot the metrics: line count, bundle/app size if a build exists, test count, type-error count, the base ref and HEAD SHA.

### 1. Format & Lint
> **Buckets:** cleancode
- **What:** consistent style, zero lint noise.
- **Fix direction:** run the formatter and linter (check mode during the audit; autofix belongs to the fix phase, where it goes *first* so later changes show real diffs).
- **Tools:** Prettier, ESLint (`--fix`). Enforce in CI so it never drifts again.

### 2. Dead Code Elimination — *biggest line-count win*
> **Buckets:** cleancode
- **Look for:** unused exports, files, and components; installed-but-unused (or imported-but-uninstalled) deps; unreachable branches; commented-out blocks; stale feature flags + their dead branches; orphaned assets; abandoned experiments.
- **Fix direction:** delete it. Git remembers — "just in case" code is what history is for.
- **Tools:** `knip` (unused files, exports, and deps), `depcheck`, coverage reports (never-hit code).

### 3. Dependency Hygiene
> **Buckets:** cleancode · security · delivery
- **Look for:** unused deps, duplicate/multiple versions, known vulns, heavyweight deps you could drop. **And the AI-era supply-chain check:** does every dependency actually exist and come from a credible publisher? Flag names one typo off a popular package (`lodahs`, `reqeusts`), very new or very-low-download packages, and `postinstall` scripts. AI tools invent package names, and attackers register them ("slopsquatting"). **Never install to check** — read `package.json` and the lockfile.
- **Fix direction:** remove unused, dedupe, update the safe ones, replace heavy deps where a few lines would do.
- **Tools:** `depcheck`, `pnpm audit` / `npm audit` (only if the lockfile is present; no install), Socket.dev for malicious-package signals, a bundle analyzer to see real weight.

### 4. DRY / Duplication
> **Buckets:** cleancode
- **Look for:** copy-paste logic, near-duplicate components, the same validation/format/util written three times.
- **Fix direction:** extract shared utilities/components — but respect the **rule of three** (don't abstract until the third copy; premature DRY is its own mess).
- **Tools:** `jscpd` (copy-paste detector) + your own eyes on the report.

### 5. Separation of Concerns & Layering
> **Buckets:** architecture
- **Look for:** business logic leaking into UI; data-access calls inside components; layers reaching past their neighbors; circular dependencies; god-files doing five jobs.
- **Testability & coverage:** pure logic (formatting, merge, decision) stranded where it can't be unit-tested — buried in a component or provider — plus risky paths with no coverage. Untestable usually *means* mis-layered. → extract to a testable module/core layer and add coverage. _(Overlaps `cleancode`'s test-hygiene — fine.)_
- **Client-direct backends are a valid architecture — judge them as one.** Supabase and Firebase are *designed* for the frontend to talk to the database directly, with row-level security (RLS) or security rules as the gatekeeper. Don't report that as a layering violation. Instead:
  - **The security rules ARE the API.** Audit every table's RLS policies (or Firebase rules) as the authorization layer — that's a `security` finding when they're missing or permissive (see pass 10).
  - **Note the tradeoff as an architecture observation** (Low/Med, not a defect): for most products that will grow, **an API layer in the middle is the more scalable default.** It keeps business rules in one place instead of spread across clients and policies, gives you somewhere to put rate limits, validation, and secrets, lets you change the database without shipping new clients (old mobile builds stay live for weeks), and makes the security model reviewable as code instead of policy SQL. Client-direct is fine for prototypes, small apps, and read-heavy features; flag it when the app is gaining paying users, multiple clients, or complex business rules.
- **Fix direction:** push logic to the right layer; pick one boundary rule and enforce it (either "frontends only go through the API" or "client-direct, and every table has a tested RLS policy"). Break circular deps.
- **Tools:** `madge` / `dependency-cruiser` (circular deps + layer rules), ESLint boundary rules.

### 6. Type Hygiene
> **Buckets:** cleancode
- **Look for:** `any`, unsafe `as` casts, non-null assertions hiding bugs, non-exhaustive switches, implicitly-`any` params.
- **Fix direction:** turn on `strict`. Replace `any` with real types. Add exhaustive `switch` checks. Make illegal states unrepresentable where it's cheap.
- **Tools:** `tsc --strict`, `typescript-eslint` strict rules.
- _Runtime validation of untrusted/bad data at boundaries lives in pass 8 (reliability) — that's the crash/brick class, not type hygiene._

### 7. API & Contract Hygiene
> **Buckets:** architecture
- **Look for:** inconsistent endpoint shapes, untyped or drifting client↔server contracts, missing request/response validation, accidental breaking changes, inconsistent error envelopes.
- **Fix direction:** share typed contracts between client and server (single source of truth). Validate requests *and* responses. Keep one consistent success/error shape. Version when you must break.
- **Tools:** shared `zod` schemas, typed API clients, contract tests.

### 8. Error Handling & Resilience
> **Buckets:** reliability
- **Look for:** swallowed errors (`catch {}`), inconsistent error patterns, missing timeouts/retries on I/O, no graceful degradation, missing user-facing error states, observability gaps.
- **Boundary validation (the bad-data / brick class):** untrusted data crossing a boundary (API response, DB/SQLite row, env var, external payload) that's `as`-cast but never *validated* — one corrupt value can crash or **permanently brick** the app downstream. → parse-don't-assume at every boundary, quarantine/skip a bad row instead of throwing, guard required env at load.
- **Fix direction:** handle or propagate — never silently swallow. Standardize the error pattern. Add timeouts/retries on network calls. Make failures surface to the user *and* to Sentry.
- **Tools:** Sentry (confirm it's actually capturing), a lint rule against empty catch blocks, `zod` (or similar) at every boundary.

### 9. Naming & Clarity
> **Buckets:** cleancode
- **Look for:** inconsistent naming, mystery abbreviations, the same concept under two names (user-facing vs internal), misleading names.
- **Fix direction:** rename for clarity; align user-facing and internal terms.
- _God-files / oversized files / mixing concerns → pass 5 (Separation of Concerns & Layering, architecture), not here._

### 10. Security pass
> **Buckets:** security
- **Look for:** committed secrets (including a committed `.env` — anything ever committed counts as leaked, even if deleted later); authorization enforced only on the frontend (the classic trap — gate it server-side too); missing input validation; dependency vulns; over-broad data access (does your RLS / row-level policy *actually* restrict?).
- **Who can call this?** List every API route, server action, and edge/cloud function and answer that question for each. Routes with no auth check at all; IDs taken from the URL or body with no ownership check (IDOR); Next.js Server Actions (they're public POST endpoints); auth enforced only in middleware; `getSession()` trusted on the server instead of a verified `getUser()`/`getClaims()`; JWTs decoded without verifying.
- **Exposure counts as exploitable.** A secret behind a public prefix, in client code, or in git history is High even if no bundle ships it today — the key is already exposed or one refactor away from it.
- **Secrets in the browser:** server-only keys behind a client-exposed prefix (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`, `REACT_APP_`, `PUBLIC_`), a Supabase `service_role` / `sb_secret_` key or a payments/AI provider secret imported into client code. Anything in a mobile app bundle is public too. If a build output exists, grep it for key patterns — that's the proof.
- **Database rules (Supabase/Firebase):** tables in exposed schemas with RLS off; `USING (true)` policies; INSERT/UPDATE with no `WITH CHECK`; grants to `anon`; `SECURITY DEFINER` functions callable by `anon`; views that bypass RLS; public storage buckets; Firebase rules with `if true`.
- **Abuse and the bill:** no rate limit on login, signup, password reset, OTP/SMS/email sends, or any route that calls a paid API (AI especially — an open AI endpoint is someone else's free API on your card). AI routes with no auth, no per-user cap, no `max_tokens`.
- **Webhooks:** payment/provider webhooks not signature-verified (and verified against the raw body).
- **Injection:** SQL built by string concatenation, `dangerouslySetInnerHTML` / rendering AI or user content as HTML.
- **Fix direction:** move gating to the server, validate inputs, rotate any leaked secret (rotation first, then remove it from code and history), tighten access rules, add rate limits and spend caps.
- **Tools:** a secret scanner (`gitleaks`, `trufflehog`), `git log -p` for committed `.env` files, Supabase `get_advisors` (security) with permission, a manual authz review. For a full pre-launch sweep, `launch-review.md` has the long-form checklist.

### 11. Test Hygiene
> **Buckets:** cleancode
- **Look for:** dead/skipped tests, stray `.only`, tautological or flaky tests, test files that don't match their source (1:1 naming).
- **Fix direction:** delete dead tests, remove `.only`, fix or quarantine flakes, rename mismatched test files.
- _Coverage gaps + testability (pure logic stranded where it can't be tested) → pass 5 (Separation of Concerns, architecture). They'll overlap a bit — fine._

---

## More passes — systems design, SRE, ops & data
_These are **diff-scoped** when there's a base to diff against — they check what THIS batch of ships changed against the standard. With no usable base, apply them to the whole repo (see Scope, above). They're written stack-neutral; the **stack modules** you loaded (`stacks/*.md`) add the exact files, settings, tools, and traps for this repo's platforms, keyed by pass number. Where a pass sharpens an earlier one (8 Error Handling, 10 Security) it says so._

### 12. Observability Completeness — *"every new route emits the four signals," not just "an error tracker exists"*
> **Buckets:** reliability
- **What:** pass 8 only asks "is the error tracker capturing." That bar is too low. The classic shape: error tracking wired into the client but **not the API at all**, and no global error handler (only per-route handlers on a few routes), so an unhandled error on any other route returns a default 500 with zero capture.
- **Look for:** new routes with no error-capture path; no global error handler at the API entrypoint; `console.log` instead of structured logs; errors with no request context (route, user, request id); new scheduled jobs whose success/failure isn't an emitted signal anyone can query; client source maps not uploaded (minified stack traces). For 3–5 critical routes, confirm the four golden signals are queryable: latency (p95/p99 against the platform's timeout), traffic, errors, saturation (timeouts / out-of-memory).
- **Fix direction:** treat "a new route with no telemetry" as seriously as a swallowed error. One global error handler that captures and returns a consistent error shape; a request-scoped structured logger; jobs that emit a visible success/failure.
- **Tools:** the stack's error tracker (Sentry or similar) and its source-map upload; platform logs; OpenTelemetry.

### 13. SLO Attainment & Health-Check Honesty — *did we stay inside the budget, and does /health actually probe?*
> **Buckets:** reliability
- **What:** routine re-measurement against the SLOs the Architecture Review set (if any exist — if not, that's the finding). Also a standing check on the health endpoint, which is usually hollow: a `/health` returning a static `{ ok: true }` proves only that the process starts; it reports green while the database is down or a key is rotated.
- **Look for:** SLI attainment since last ship (core write path success + latency, main read path, payment/entitlement resolution) and error budget burned; `/health` that doesn't probe its dependencies; new env vars/keys `/health` doesn't assert at boot.
- **Fix direction:** if the budget is exhausted, flag "stop shipping features, harden." Make `/health` a real dependency probe (cheap DB round-trip + required config present) with per-dependency status. Defining SLOs is the Architecture Review's job.
- **Tools:** error-tracker release health or platform analytics for SLI data; a `checkHealth()` with a DB ping + env assertions.

### 14. Alerting & On-Call Surface — *does anything page a human, and did this ship add a silent failure?*
> **Buckets:** reliability
- **What:** captured signal is worthless if nothing routes it to a person. Common shape: scheduled jobs with no on-failure alert, and a static health check that can't alert because it never goes unhealthy.
- **Look for:** new error paths with no alert rule; new or changed jobs with no failure notification; thresholds not tied to SLOs (error-rate spike, p99 breach, timeouts, 5xx bursts); new on-call surface (route, job, dependency, env var) with no owner; alerts so noisy they're muted.
- **Fix direction:** error-tracker alert rules and platform job/deploy-failure notifications to a real channel; thresholds from SLOs; every alert actionable or deleted.
- **Tools:** error-tracker alerts; platform notifications; an external uptime monitor.

### 15. Idempotency & Write-Safety on New Endpoints — *platforms retry, clients retry, payment providers WILL redeliver*
> **Buckets:** architecture · reliability · security
- **What:** sharpens pass 8, which covers the CALLING side (timeouts/retries). This is the RECEIVING side: is every new write safe to run twice? Classic tells: a payment webhook that claims idempotency in a **comment only** with no unique constraint or upsert behind it; no timeouts anywhere in an API whose platform kills functions mid-write.
- **Look for (new write/webhook endpoints):** mutations and webhook receivers with no dedupe key; webhooks not signature-verified, or whose event id isn't recorded once; multi-step writes (update DB → call payments → send notification) with no transaction and no compensating action; read-modify-write races with no atomic update or version column; module-global mutable state used to "remember" across requests (invalid on serverless and multi-instance servers); external calls with no timeout.
- **Fix direction:** each new write/webhook gets a dedupe key (a persisted `Idempotency-Key` or an event-id table with a unique constraint), signature verification, a transaction or single database function for multi-step writes, and a timeout well under the platform limit on every external call. A test that double delivery is a no-op.
- **Tools:** unique constraints + upsert/`ON CONFLICT`; database functions; signature verification; `AbortSignal.timeout`; grep for module-level mutable singletons.

### 16. Migration Safety & Reversibility (this batch) — *forward-only SQL, no down-path, applied out-of-band*
> **Buckets:** delivery · scalability
- **What:** the highest-stakes operability check on any relational database. The dangerous default: forward-only SQL with **no down-migrations** and **no migration step in CI**, so schema reaches prod out-of-band, unversioned against the deploy and unreversible. Where old clients stay live (mobile apps, cached SPAs), a non-additive migration silently breaks them.
- **Look for (migrations added since the base):** any `DROP`/`RENAME`/retype; `NOT NULL` added without a default; a backfill in the same transaction as a schema change (lock risk); anything that isn't expand-then-contract; whether the migration ran before the code that reads the new shape; whether anyone can state "how do we undo migration N."
- **Fix direction:** any non-additive migration is a release blocker until split expand → backfill → contract across ≥2 deploys; a reversal note per migration; migration applied before dependent code.
- **Tools:** `squawk` (Postgres migration linter); the platform's migration CLI and diff tooling; read-only catalog queries.

### 17. Local-First Device-Migration Safety
> **Buckets:** delivery
- **Runs only when a stack module defines it** (today: `stacks/expo-react-native.md`, for apps with an on-device database replica). Otherwise N/A — "no on-device replica."

### 18. Deploy & Rollback Verification — *is prod actually serving the commit we shipped?*
> **Buckets:** delivery
- **What:** git-driven auto-deploys have a documented failure shape: two merges land seconds apart, the platform dedupes the build, and prod stays on the previous build while everyone believes the ship landed.
- **Look for:** whether prod's deployed commit matches the audited commit on every independent surface (web, API, database migrations, mobile updates); whether the previous-good build is identified and one action away; whether any migration this batch made code rollback impossible (see pass 16).
- **Fix direction:** check (read-only, with permission) that each surface serves the audited commit and that rollback is one action away. Report it; never promote or roll back during an audit.
- **Tools (read-only):** the platform's deployment list / CLI; the health endpoint; a SHA comparison.

### 19. Client Update Governance — *can an update reach clients that can't run it?*
> **Buckets:** delivery
- **Runs only when a stack module defines it** (today: `stacks/expo-react-native.md`, for over-the-air updates against native binaries). Otherwise N/A — "no OTA updates."

### 20. Environment & Secrets Drift — *did a new var land in code without parity / .env.example?*
> **Buckets:** security · delivery
- **What:** even with secrets in platform env vars, the common gap is **no runtime env validation**, so a missing or renamed prod var fails at request time, not boot. And `.env.example` drifts until it lists a fraction of what the code reads.
- **Look for:** env vars read in code with no `.env.example` entry; a server-only secret behind a client-exposed prefix (the stack module lists the prefixes); a secret logged to the error tracker; preview/prod env parity gaps.
- **Fix direction:** diff `.env.example` against env vars referenced in code and the platform env sets; flag every gap. (A typed env schema parsed at boot is the Architecture Review's job; this pass enforces the contract per ship.)
- **Tools:** an env schema (`zod` or similar) parsed at startup; the platform's env listing (read-only, with permission); grep of env references vs `.env.example`; a secret scanner in CI.

### 21. Feature-Flag Lifecycle — *every new flag gets a type and an expiry, not just a dead-flag sweep*
> **Buckets:** delivery
- **What:** sharpens pass 2, which finds stale flags only after they rot. Where flags live in a config table or flag service, a flag can be "on" in prod with its code deleted, or vice versa, and nothing reconciles them.
- **Look for:** flag keys in code with no config entry (and the inverse); new flags with no owner, type (release / experiment / ops / kill-switch), or expiry; flags past their removal date.
- **Fix direction:** owner + type + expiry on every flag; reconcile code ↔ config; sweep expired flags into pass 2.
- **Tools:** a reconciliation grep against the flag store (read-only); a checked-in flag registry.

### 22. CI-as-Gate Drift — *are the audit's own tools actually blocking merge?*
> **Buckets:** delivery
- **What:** this playbook runs dead-code, supply-chain, and secret scans — but the common state is CI that only typechecks, tests, and builds, so every gain can silently erode next ship. **No CI at all is the finding**, not N/A.
- **Look for:** which checks block merge vs. honor-system; checks added then disabled, `continue-on-error` creeping in, packages escaping the test matrix; a frozen lockfile as the only supply-chain guard.
- **Fix direction:** enforce this audit's findings in CI so they can't come back; flag silently disabled checks.
- **Tools:** required status checks / branch protection; `gitleaks`/`trufflehog`; dependency audit at high severity; Socket.dev; `knip` as a CI job; a migration-lint step.

### 23. Performance & Cost Regression — *N+1s, missing indexes, cold-starts, and the bill*
> **Buckets:** scalability
- **What:** bundle size is easy to measure; runtime data cost is what actually bills you. Did THIS ship regress against the perf budgets and index baseline?
- **Look for:** N+1 patterns (one query per row in lists and jobs); filters/joins on unindexed columns, especially new foreign keys (Postgres doesn't auto-index them) and columns used in access-policy predicates; `select *` over-fetching; unbounded queries with no limit/pagination; heavy top-level imports inflating cold starts; jobs that loop over every user in one invocation against the timeout; new or more frequent jobs, raised timeouts, noisy error paths, or new paid-API calls driving cost.
- **Fix direction:** `EXPLAIN` the hot queries (plain `EXPLAIN` during the audit — `ANALYZE` executes the query; use it on a non-production database in the fix phase); index new FKs and policy columns; collapse N+1s; paginate; snapshot the cost drivers.
- **Tools:** the database's advisor/stats views (read-only, with permission); `pg_stat_statements`; platform duration logs; a bundle analyzer; tracing.

### 24. Data Durability & Migration Backout (this batch) — *can we get the data back if this migration was wrong?*
> **Buckets:** reliability
- **What:** the security pass covers access but never asks "can we recover the data." A bad migration is a data-loss event, not a code bug.
- **Look for:** destructive DDL shipped with no recovery path or backout note; whether backups / point-in-time recovery are even enabled (usually Couldn't-check from the repo — list it); dev and prod sharing one database; seed/reset scripts or agent tools that can reach prod; secrets touched this ship with no rotation procedure.
- **Fix direction:** reversibility and a backout note per migration; unguarded destructive DDL is a release blocker; separate dev and prod. (Setting RPO/RTO and running a restore drill is the Architecture Review's job.)
- **Tools:** the platform's backup/restore settings (read-only, with permission); migration history; migration lint.

### 25. Runbook & On-Call-Surface Currency — *did this ship add a failure mode with no runbook?*
> **Buckets:** reliability
- **What:** when something breaks at 2am, what does the responder read?
- **Look for:** new failure surface (job, dependency, env var, webhook) with no runbook entry; missing runbooks for the obvious incidents ("payments wrong", "DB unreachable", "auth failing", "job didn't run", "a key leaked"); no documented rollback path.
- **Fix direction:** a thin runbook per new surface — symptom, where to look, likely cause, mitigation including rollback.
- **Tools:** markdown runbooks in the repo, linked to the exact log/error views.

### 26. Failure-Mode & Blast-Radius Drift — *did a new dependency join the critical path without a degrade decision?*
> **Buckets:** reliability
- **What:** pass 8 mentions graceful degradation in one line but never maps WHAT degrades when WHICH dependency fails. Best-effort work (email, push, analytics) should degrade, not fail a user's write.
- **Look for:** new deps with no fail-soft/fail-hard decision; best-effort work on the critical request path; one failing dependency taking down unrelated routes; batch jobs whose per-item failure handling is missing.
- **Fix direction:** move best-effort work off the critical path; record a degrade decision per new dependency; check the blast-radius table (an Architecture Review artifact) still matches.
- **Tools:** `dependency-cruiser` to see what imports each external client; explicit, documented try/catch policies.

### 27. Accessibility Regression — *did new screens keep their labels, focus, and text scaling?*
> **Buckets:** accessibility
- **What:** no other routine pass looks at accessibility, and a real slice of users (and, in the EU since June 2025, the European Accessibility Act) depend on it. If the repo has its own a11y rules, check against them; otherwise the checklist below plus the stack module's platform specifics.
- **Look for (new/changed screens):** images without `alt`; icon-only buttons with no accessible name; `div`s used as buttons; inputs without labels; modals without focus management; keyboard-unreachable controls; color-only state; contrast below WCAG AA; fixed sizes that break when text is enlarged.
- **Fix direction:** regression-check new screens; automated baseline plus a manual screen-reader spot check on changed flows.
- **Tools:** `eslint-plugin-jsx-a11y`; axe DevTools / `@axe-core/playwright`; Lighthouse; VoiceOver/TalkBack/NVDA.

### 28. Privacy & PII-Egress Regression — *did this ship leak PII into logs, error tracking, or analytics?*
> **Buckets:** security
- **What:** pass 10 covers who can access data; this covers PII leaving through telemetry, which pass 10 never looks at.
- **Look for:** PII in error-tracker payloads (emails, tokens, user content); raw PII in analytics events; logging of sensitive data or full request bodies; tracking that fires before consent; prompts and AI responses containing user data sent to logs or third parties; a privacy policy or store privacy declaration that no longer matches what's collected; no working account/data-deletion path.
- **Fix direction:** scrubbing in the error tracker's send hook; strip PII from analytics; consent before tracking; reconcile declarations; a real deletion path.
- **Tools:** the error tracker's data-scrubbing config; a PII grep over log/analytics call sites; the platform's privacy declaration checklist.

### 29. Docs & Agent-Instruction Currency — *a wrong AGENTS.md is a correctness bug for the next Claude*
> **Buckets:** docs
- **What:** the "many agents touched it" tax. Stale `CLAUDE.md`/`AGENTS.md`/`README` steer the next agent into the ditch, and they drift in nearly every actively developed repo.
- **Look for:** agent instructions describing architecture or conventions that changed; README setup or env docs now wrong; comments that now lie (e.g. "idempotent" on a handler with no idempotency); load-bearing decisions with no short ADR.
- **Fix direction:** reconcile agent docs against reality; clean-clone README smoke test if setup changed; fix or delete lying comments; one-paragraph ADRs for non-obvious decisions.
- **Tools:** fresh-clone smoke test; MADR template; a CHANGELOG.

---

### 30. Comment & Code Truth
> **Buckets:** cleancode
- **Look for:** comments / JSDoc that *lie* — describe behavior the code no longer has, point at moved or renamed things, or claim a guarantee the code doesn't make (e.g. a comment saying "merges LWW here" above a blind `INSERT OR REPLACE`).
- **Fix direction:** fix the comment to match the code, or delete it. A wrong comment is worse than none — the next human or agent trusts it and "optimizes" the wrong thing.
- **Tools:** grep for stale references; read each comment against the code it sits on.

### 31. Magic Numbers & Hardcoded Values
> **Buckets:** cleancode
- **Look for:** unexplained literals — timeouts, thresholds, retry counts, sizes, durations, limits, magic strings — scattered inline with no named constant.
- **Fix direction:** lift them to named constants (with the unit/why in the name). Especially anything *tuned* (a 5s timeout, a 0.7 confidence cutoff) that someone will need to find and change later.
- **Tools:** eyes + grep for bare numeric/string literals in logic.

---

## Finish
**On the audit run:** write the verified report and stop. That's the whole finish.

**Later, after fixes land (fix phase, not this run):** run the full suite + typecheck + lint + build, all green; re-measure against Pass 0 (lines removed, bundle/app-size delta, type-errors killed, deps dropped) and **write the win down**; commit in logical chunks.

## What the audit produces
- A **structured findings report** written to `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` — findings grouped by area, each tagged `[Severity · Cost-of-doing-nothing]` + location, plus Pass-0/Finish metrics. **This file is the handoff:** run `remediation-plan.md` on it to decide solutions (and upstream fixes), rather than trying to find-and-decide in one overloaded pass. (See [`_report-shape.md`](./_report-shape.md) and the README's _audit → report → action loop_.)
- Once the fixes land: a cleaner, smaller, safer repo, and CI guards so the gains don't erode.

---
_v1.1, 2026-10-02 — audits made look-only, real verifier, whole-repo scope fallback, client-direct backends, expanded security pass. Part of Varnish._
