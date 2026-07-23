# Codebase Audit

_The cleanup-and-hardening pass you run after shipping a big feature — or a pile of small ones. The deliberate paying-down of the entropy a ship sprint leaves behind._

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say:** _"Read this file and run the `cleancode` audit on this repo."_ (Swap in any bucket below, or `all`.)

Claude will: take the passes tagged with that bucket → confirm the repo is green → run them **read-only** (find, don't fix) → **adversarially verify every finding** (a fresh skeptic greps the repo to confirm each claim and right-size severity) → write the *verified* findings to `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` → stop and show you the report. You then run `remediation-plan.md` on the report to decide solutions, and fix in chunks. _(Run it in the target repo's own session — the fixing happens there.)_

**Buckets** — every pass below carries a `> Buckets:` tag; a bucket just runs the passes tagged with it. Pass 0 + the Finish step always run.

| Bucket | What it checks |
|---|---|
| `cleancode` | **hygiene only** — lint, dead code, unused deps, DRY, type-hygiene, naming/clarity, test-hygiene, comment-truth, magic-numbers _(the biggest line-count wins live here)_ |
| `architecture` | layering, separation of concerns, contracts, idempotency, boundaries, testability/coverage |
| `security` | authz, secrets, PII/privacy, dep vulns |
| `reliability` | errors, observability, SLO/health, alerting, durability, blast radius _(SRE)_ |
| `scalability` | perf, cost, N+1s, caching, cold-starts |
| `delivery` | deploy/rollback, migrations, OTA, CI, flags, env |
| `accessibility` | a11y regressions |
| `docs` | docs / agent-instruction currency |
| `all` | every pass |
| `compliance` | _(different mode)_ run the technical compliance precheck → see `compliance-review.md`. Reuses the security/reliability/delivery checks, maps them to CIS v8 → SOC2/ISO/PCI/HIPAA, and renders a crosswalk matrix instead of a fix-list. |
| `seo` | _(different target)_ the **marketing-site visibility** audit → see `seo-audit.md`. Classic SEO + GEO (AI-answer citation readiness) with evidence-tiered findings and a cargo-cult ledger. Run it on the site, not the app repo. |

For the deep "is this built right / rebuild vs refactor?" audit, use `architecture-review.md` instead.

**Two ground rules:** the examples below are grounded in one real production stack (React Native app · serverless API · Postgres) — treat them as *principles* and apply them to THIS repo's stack. If a pass doesn't apply, mark it **N/A** in the report (don't force a finding) — wide coverage is the point.

## Findings are adversarially verified (built in, not optional)
Don't trust a raw finding. After the find phase, **every finding is re-checked by a fresh verifier whose job is to *disprove* it** — grep the repo to confirm an "unused / duplicate / dead" claim is actually true, check the cited line really says what the finding claims, and right-size the severity. Drop anything that doesn't survive; note what got adjusted. *A report of 30 verified findings is worth ten times a report of 50 plausible ones.*
- `cleancode` → the verifier proves claims by grep/inspection ("is this export really unreferenced?").
- Other buckets adapt the same skeptic stance: `security` → "can this actually be exploited?"; `reliability` → "can this failure really happen, and what's the blast radius?".

## Report format
Write the report in the shared **Audit shape** — see [`_report-shape.md`](./_report-shape.md). In brief: **findings-first** (the report detects, the remediation plan decides), every finding tagged **`[Severity · Cost-of-doing-nothing]`**, findings **grouped by area** so clusters show, and a closing **→ handoff to `remediation-plan.md`**. No prescribed fix-order, no per-finding effort — that's the remediation plan's job (it's also where the one upstream fix that kills a whole cluster gets spotted).

**Filename:** `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` — date + **bucket** + the run's **HHMM** (from the shell, so same-day re-runs don't collide). e.g. `audits/2026-06-25-cleancode-1455.md`.

**On top of the shared shape, the Codebase Audit adds:**
- **Header** carries the **bucket** + **playbook**.
- **Pass 0 baseline metrics** — LOC, tests, typecheck, bundle size, captured up top. (Measurement, not a decision — it belongs.)
- **Findings grouped by pass** — each pass marked `*Applicable.*` or `*N/A — why*`; each finding gets `[Severity · Cost-of-doing-nothing]` + `file:line` + what-and-why, plus a ✅ *Clean:* line for what passed.
- **Finish** — once fixes land (later), re-measure the deltas against Pass 0.

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
1. **Start green, end green.** Never audit on a red build. Confirm tests + typecheck + build pass *before* you start, so any regression you introduce is obviously yours.
2. **One pass = one (or a few) commits.** Keep each pass independently committable and verifiable so you can always bisect what broke. Small safe steps beat one giant "cleanup" commit.

## Execution: order & fan-out
- **Solo:** work the bucket's passes top-to-bottom (the order matters).
- **Agent fan-out (how I actually do it):** spin up one agent per pass in parallel, each returning a findings list; aggregate into the report, then review/apply/verify. This playbook is written to be handed to an agent directly.

**Order matters:** format/lint first (clears cosmetic noise), then dead-code (shrinks the surface area everything else has to scan), then the structural/correctness passes, then a full verification pass last. (This orders the *passes you run*, not the *findings you fix* — the report stays grouped, not pre-sequenced; fix-order is the remediation plan's call.)

---

## The passes

### 0. Baseline / safety net
> **Buckets:** always (runs with every audit)
- Branch off `main`. Run the full suite + typecheck + build — **everything green before you touch anything.**
- Snapshot the metrics you'll want to brag about later: line count, bundle/app size, test count, type-error count.

### 1. Format & Lint
> **Buckets:** cleancode
- **What:** consistent style, zero lint noise.
- **Do:** run the formatter and linter with autofix *first* — it clears cosmetic churn so later passes show real diffs.
- **Tools:** Prettier, ESLint (`--fix`). Enforce in CI so it never drifts again.

### 2. Dead Code Elimination — *biggest line-count win*
> **Buckets:** cleancode
- **Look for:** unused exports, files, and components; installed-but-unused (or imported-but-uninstalled) deps; unreachable branches; commented-out blocks; stale feature flags + their dead branches; orphaned assets; abandoned experiments.
- **Do:** delete it. Git remembers — "just in case" code is what history is for.
- **Tools:** `knip`, `ts-prune` (unused exports), `depcheck` (unused deps), `unimported` (orphan files), coverage reports (never-hit code).

### 3. Dependency Hygiene
> **Buckets:** cleancode · security · delivery
- **Look for:** unused deps, duplicate/multiple versions, known vulns, heavyweight deps you could drop.
- **Do:** remove unused, dedupe, update the safe ones, replace heavy deps where a few lines would do.
- **Tools:** `depcheck`, `pnpm dedupe`, `pnpm audit`, a bundle analyzer to see real weight.

### 4. DRY / Duplication
> **Buckets:** cleancode
- **Look for:** copy-paste logic, near-duplicate components, the same validation/format/util written three times.
- **Do:** extract shared utilities/components — but respect the **rule of three** (don't abstract until the third copy; premature DRY is its own mess).
- **Tools:** `jscpd` (copy-paste detector) + your own eyes on the report.

### 5. Separation of Concerns & Layering
> **Buckets:** architecture
- **Look for:** business logic leaking into UI; data-access calls inside components; layers reaching past their neighbors; circular dependencies; god-files doing five jobs.
- **Testability & coverage:** pure logic (formatting, merge, decision) stranded where it can't be unit-tested — buried in a component or provider — plus risky paths with no coverage. Untestable usually *means* mis-layered. → extract to a testable module/core layer and add coverage. _(Overlaps `cleancode`'s test-hygiene — fine.)_
- **Do:** push logic to the right layer. Enforce the boundary rules (e.g. *frontends never hit the DB directly, only through the API*). Break circular deps.
- **Tools:** `madge` / `dependency-cruiser` (circular deps + layer rules), ESLint boundary rules.

### 6. Type Hygiene
> **Buckets:** cleancode
- **Look for:** `any`, unsafe `as` casts, non-null assertions hiding bugs, non-exhaustive switches, implicitly-`any` params.
- **Do:** turn on `strict`. Replace `any` with real types. Add exhaustive `switch` checks. Make illegal states unrepresentable where it's cheap.
- **Tools:** `tsc --strict`, `typescript-eslint` strict rules.
- _Runtime validation of untrusted/bad data at boundaries lives in pass 8 (reliability) — that's the crash/brick class, not type hygiene._

### 7. API & Contract Hygiene
> **Buckets:** architecture
- **Look for:** inconsistent endpoint shapes, untyped or drifting client↔server contracts, missing request/response validation, accidental breaking changes, inconsistent error envelopes.
- **Do:** share typed contracts between client and server (single source of truth). Validate requests *and* responses. Keep one consistent success/error shape. Version when you must break.
- **Tools:** shared `zod` schemas, typed API clients, contract tests.

### 8. Error Handling & Resilience
> **Buckets:** reliability
- **Look for:** swallowed errors (`catch {}`), inconsistent error patterns, missing timeouts/retries on I/O, no graceful degradation, missing user-facing error states, observability gaps.
- **Boundary validation (the bad-data / brick class):** untrusted data crossing a boundary (API response, DB/SQLite row, env var, external payload) that's `as`-cast but never *validated* — one corrupt value can crash or **permanently brick** the app downstream. → parse-don't-assume at every boundary, quarantine/skip a bad row instead of throwing, guard required env at load.
- **Do:** handle or propagate — never silently swallow. Standardize the error pattern. Add timeouts/retries on network calls. Make failures surface to the user *and* to Sentry.
- **Tools:** Sentry (confirm it's actually capturing), a lint rule against empty catch blocks, `zod` (or similar) at every boundary.

### 9. Naming & Clarity
> **Buckets:** cleancode
- **Look for:** inconsistent naming, mystery abbreviations, the same concept under two names (user-facing vs internal), misleading names.
- **Do:** rename for clarity; align user-facing and internal terms.
- _God-files / oversized files / mixing concerns → pass 5 (Separation of Concerns & Layering, architecture), not here._

### 10. Security pass
> **Buckets:** security
- **Look for:** committed secrets; authorization enforced only on the frontend (the classic trap — gate it server-side too); missing input validation; dependency vulns; over-broad data access (does your RLS / row-level policy *actually* restrict?).
- **Do:** move gating to the server, validate inputs, rotate any leaked secrets, tighten access rules.
- **Tools:** a secret scanner, `pnpm audit`, a manual authz review.

### 11. Test Hygiene
> **Buckets:** cleancode
- **Look for:** dead/skipped tests, stray `.only`, tautological or flaky tests, test files that don't match their source (1:1 naming).
- **Do:** delete dead tests, remove `.only`, fix or quarantine flakes, rename mismatched test files.
- _Coverage gaps + testability (pure logic stranded where it can't be tested) → pass 5 (Separation of Concerns, architecture). They'll overlap a bit — fine._

---

## More passes — systems design, SRE, ops & data
_Added 2026-06-24 via a 5-lens deep-dive (systems design · SRE · platform/ops · data/perf · completeness critic). These are **diff-scoped** — they check what THIS batch of ships changed against the standard, not the whole system. Deduped across lenses, and grounded in a real local-first mobile stack. Where a pass sharpens an earlier one (8 Error Handling, 10 Security) it says so._

### 12. Observability Completeness — *"every new route emits the four signals," not just "Sentry exists"*
> **Buckets:** reliability
- **What:** the existing pass 8 only asks "is Sentry capturing." That bar is too low. The classic shape: Sentry wired in the mobile app but **not in the API at all**, and no global `app.onError` (only per-route handlers on a few routes), so an unhandled error on any other route returns the framework's default 500 with zero capture.
- **Look for:** new Hono routes added this ship with no error capture path to Sentry; a missing global `app.onError` at the API entrypoint; `console.log` instead of structured JSON logs (Vercel function logs are the only backend trace you get); errors with no request context (route, `userId`/auth subject, request id); new crons whose success/failure isn't an emitted signal anyone can query — only a returned value nobody reads; mobile source maps not uploaded on the EAS/OTA build (minified stack traces). For 3-5 critical routes confirm the four golden signals are queryable: latency (function duration p95/p99 vs the 30s `maxDuration` ceiling), traffic (invocations), errors (rate), saturation (timeout/OOM rate).
- **Do:** treat "a new route with no telemetry" as a finding of the same severity as a swallowed error. Wire `@sentry/node` (or OTel) into the API and add a single global `app.onError(captureException + structured 500 envelope)`. Standardize a request-scoped JSON logger (route, requestId, userId). Confirm new crons emit a visible success/failure signal.
- **Tools:** `@sentry/node` + `@sentry/nextjs` + `sentry-expo`; Vercel function logs / OTel; Sentry CLI for source-map upload; Hono `onError` + a logger middleware; `mcp__plugin_vercel_vercel__get_runtime_logs`.

### 13. SLO Attainment & Health-Check Honesty — *did we stay inside the budget, and does /health actually probe?*
> **Buckets:** reliability
- **What:** routine re-measurement against the SLOs the Architecture Review set. Also a standing check on the health endpoint, which is usually hollow: a `/health` returning a static `{ ok: true }` proves only that the function cold-starts — it reports green while the database is down, a service key is rotated, or a critical dependency is unreachable.
- **Look for:** SLI attainment since last ship (e.g. the core write path's success+latency, the main read path, payment/entitlement resolution) and how much error budget burned; whether `/health` still returns a flat `{ok:true}` rather than per-dependency status; new env/keys this ship added that `/health` doesn't assert exist at boot.
- **Do:** report budget burned since last ship; if it's exhausted, flag "stop shipping features, harden" as a finding. Make `/health` a real dependency probe (cheap `SELECT 1` + assert required env/keys present) returning per-dependency status. (Defining the SLOs themselves and building the probe is the Architecture Review's job; this pass only re-measures and flags regression.)
- **Tools:** Sentry releases/issue-rate or Vercel Analytics for SLI data; a small `checkHealth()` doing `SELECT 1` + env assertions; Zod to validate required env at boot; `mcp__plugin_supabase_supabase__get_logs`.

### 14. Alerting & On-Call Surface — *does anything page a human, and did this ship add a silent failure?*
> **Buckets:** reliability
- **What:** captured signal is worthless if nothing routes it to a person. The common shape: crons with no on-failure alert, and a static `/health` cron that can't alert because it never goes unhealthy.
- **Look for:** new error paths with no alert rule; new or changed crons with no cron-failure notification; alert thresholds not tied to the SLOs (error-rate spike, p99 breach, function timeout/OOM, 5xx burst); new on-call surface (a route, cron, dependency, env var) with no owner; alert fatigue (alerts so noisy they're muted).
- **Do:** wire Sentry alert rules (new-issue, error-rate spike, regression) and Vercel cron-failure/deploy-failure notifications to a real channel (Slack, or email via a provider already in the stack). Set thresholds from SLOs, not arbitrary numbers. Every alert is actionable or deleted.
- **Tools:** Sentry alert rules + metric alerts; Vercel cron/deployment notifications; Slack webhook or Resend; optionally Better Stack/Checkly external uptime monitor.

### 15. Idempotency & Write-Safety on New Endpoints — *Vercel retries, mobile retries, RevenueCat WILL redeliver*
> **Buckets:** architecture · reliability
- **What:** sharpens pass 8's resilience, which only covers the CALLING side (timeouts/retries). This covers the RECEIVING side: is every new write safe to run twice? The classic tells: a payment-webhook handler that asserts idempotency in a **comment only** with no `ON CONFLICT`/upsert primitive backing it, and an API where a grep for `AbortController|retry|backoff|circuit` returns nothing while the function timeout is 30s.
- **Look for (diff-scoped to new write/webhook endpoints):** POST/mutation handlers and webhook receivers with no dedupe key; webhooks not signature-verified or whose event id isn't recorded-once; multi-step writes (update the DB → call the payments provider → send push) with no transaction and no compensating action; read-modify-write races (incrementing a counter, flipping a flag) with no atomic SQL or version column; reliance on module-global mutable state to "remember" across requests (invalid on stateless serverless); external calls (DB client, email, push, payments) with no `AbortController` timeout, so a hung dependency burns the full 30s and the function is killed mid-write.
- **Do:** run each new write/webhook endpoint through a short checklist — has a dedupe key (natural or client-supplied `Idempotency-Key` persisted with a UNIQUE constraint; webhook event-id table with `INSERT … ON CONFLICT DO NOTHING`); is signature-verified; multi-step writes are wrapped in a Postgres transaction or single Supabase RPC; no module-global mutable state. Wrap every external call in an `AbortController` timeout well under `maxDuration` (5-8s) to fail fast. Add a test that double-delivery of the payment webhook is a no-op.
- **Tools:** Postgres UNIQUE + `INSERT ON CONFLICT`; DB RPC/database functions; webhook signature verification + an events table; `AbortSignal.timeout`, `p-retry` (idempotent ops only); grep for module-level mutable singletons and for `catch` around the second step of a multi-write; `mcp__plugin_supabase_supabase__execute_sql`.

### 16. Migration Safety & Reversibility (this batch) — *forward-only SQL, no down-path, applied out-of-band*
> **Buckets:** delivery · scalability
- **What:** the highest-stakes operability check for a managed-Postgres stack. The dangerous default: forward-only numbered SQL with **no down-migrations** and **no migration step in CI** — schema reaches prod out-of-band via a dashboard or tool call, un-versioned-against-deploy and un-reversible. And where OTA-updated mobile apps mean weeks-old builds are live, a non-additive migration silently breaks in-flight old clients.
- **Look for (only migrations added since the last release tag):** any `DROP`/`RENAME`/retype of a column; `NOT NULL` added without a default; a backfill in the same statement/transaction as a schema change (lock risk); anything that isn't strictly expand-then-contract; whether the migration was applied in the correct order relative to the code deploy that reads the new shape; whether anyone can state "how do we undo migration N." Re-run Supabase advisors so schema security/perf regressions surface every ship.
- **Do:** flag any non-additive migration as a release blocker until split expand → backfill → contract across ≥2 deploys. Require a stated reversal note per migration (even "forward-fix only, here's the compensating migration"). Confirm migration applied before the code that depends on it.
- **Tools:** `squawk` (Postgres migration linter for lock/destructive ops); Supabase CLI shadow-DB diff; `mcp__plugin_supabase_supabase__get_advisors`, `list_migrations`, `execute_sql` (inspect `pg_constraint`); a migration-lint CI step for `DROP`/`ALTER…TYPE`.

### 17. Local-First Device-Migration Safety — *the on-device SQLite replica is a SECOND schema you now have to migrate*
> **Buckets:** delivery
- **What:** for local-first apps only. If the repo has sync rules governing the SERVER side (additive-only synced tables, frozen API contracts, tombstones, LWW) — don't re-audit those. This pass covers the DEVICE side such rules usually miss: the on-device SQLite replica + offline outbox + conflict policy.
- **Look for (any ship touching synced tables or shared compute contracts):** an OTA JS update that assumes a SQLite column the installed on-device DB doesn't have; un-synced outbox writes that could be lost when a user updates across a schema change; tombstone/`deleted_at` filtering missing on the client side; no on-device `schema_version` and no forward migration runner; no server-driven "force full re-sync / wipe-and-rehydrate" escape hatch for a corrupted local DB. The eventual cutover from server-authoritative reads to local reads is a data-migration event for every user, not a code change.
- **Do:** for any synced-data change, verify the on-device migration exists and is idempotent, the outbox survives the upgrade (drain-before-upgrade), tombstones are filtered client-side too, and there's a tested force-resync path. Pair with pass 19 — a replica-schema change may require a native build + runtimeVersion bump, not an OTA.
- **Tools:** `expo-sqlite` migration runner / Drizzle or Kysely on-device migrations; a `schema_version` row in the local DB; integration test simulating upgrade-across-schema with a non-empty outbox; a server-side `force_resync` config flag.

### 18. Deploy & Rollback Verification — *is prod actually serving the commit we shipped?*
> **Buckets:** delivery
- **What:** git-driven auto-promotion has a **real, documented failure shape**: two branches merge within seconds, the platform's build dedup skips the prod build, and prod stays pinned to the previous build while everyone believes the ship landed.
- **Look for:** confirmation that the prod deployment SHA == the audited commit, across all three independent surfaces — Vercel (web+api), Supabase migrations (no rollback at all), and EAS/OTA (rollback = republish previous update, NOT a git revert); whether the previous-good build/OTA is identified and one action away; whether any migration this batch made the code irreversible (you can't roll back code that depends on a now-applied non-reversible migration — which is why pass 16 matters).
- **Do:** as the final step of the audit, assert prod serves the audited commit on web, api, and the OTA channel; confirm the prior-good build/OTA is one action away on each surface; confirm no migration this batch blocked code rollback.
- **Tools:** Vercel CLI (`vercel ls`, `vercel promote`, instant rollback) / `mcp__plugin_vercel_vercel__list_deployments`, `get_deployment`; EAS `eas update --rollback`/republish; the `/api/health` cron as a smoke gate; a SHA-match assertion.

### 19. OTA / runtimeVersion Governance — *did a JS-only OTA assume native code old binaries don't have?*
> **Buckets:** delivery
- **What:** the dangerous default: `runtimeVersion` pinned as a hardcoded literal (not a fingerprint/appVersion policy that auto-bumps on native change), and an `updates` block with no `fallbackToCacheTimeout`/`checkAutomatically`. The risk: an OTA JS update referencing a new native module — or a new on-device-replica expectation — pushed under the same runtimeVersion to clients that can't satisfy it → crash-on-launch with no native fallback. Even where a force-update gate exists (an API route serving a minimum-supported version from config), often nothing ties runtimeVersion, OTA channel, and that gate together.
- **Look for (this ship):** any native-module or on-device-schema change that went out as an OTA instead of a new build + runtimeVersion bump; OTA pushed to the wrong channel (`development`/`preview`/`production`); the minimum-supported gate not raised when old clients must be forced off; no tested OTA rollback (republish prior update) for the production channel.
- **Do:** confirm native/replica-shape changes got a runtimeVersion bump and a store build, not an OTA; confirm correct channel; raise the minimum-supported gate if needed.
- **Tools:** EAS Update (channels, `eas update --rollback`); Expo runtimeVersion fingerprint policy; `expo-updates` config; a config-driven mobile-version gate.

### 20. Environment & Secrets Drift — *did a new var land in code without parity / .env.example?*
> **Buckets:** security · delivery
- **What:** even where secrets are correctly gitignored and live in platform env vars, the common gap is **no runtime env-schema validation**, so a missing/renamed prod var fails at request time, not at boot. And `.env.example` drifts: it ends up listing a fraction of the vars the code actually references.
- **Look for (diff-scoped):** any `process.env` var added in code this batch with no corresponding `.env.example` entry; a server-only secret slipped behind a `NEXT_PUBLIC_` prefix; a secret logged to Sentry; parity gaps between preview/prod env sets across platforms.
- **Do:** diff `.env.example` against env vars actually referenced in code and against the platform env sets; flag every new var missing from `.env.example`. (Introducing the single typed Zod env schema parsed at boot is the Architecture Review's job; this pass enforces the contract per ship.)
- **Tools:** Zod env schema (`env.ts` parsed at startup); `mcp__plugin_vercel_vercel__list_projects` / `vercel:env` skill / `vercel env ls`; `eas env`; a grep of `process.env` references vs `.env.example`; secret scanner in CI.

### 21. Feature-Flag Lifecycle — *every new flag gets a type and an expiry, not just a dead-flag sweep*
> **Buckets:** delivery
- **What:** sharpens pass 2, which finds stale flags only AFTER they rot. Where flags live as rows in a config table and toggle via SQL with no redeploy, a flag can be "on" in prod with its code branch already deleted, or vice versa — and nothing reconciles config rows against the code that reads them.
- **Look for:** flag keys read in code with no matching config row (and the inverse — DB rows no code reads); flags added this batch with no owner, type (release / experiment / ops-gate / kill-switch), or expiry; release/experiment flags past their removal date.
- **Do:** require every new flag to carry an owner + type + expiry (or "permanent" marker); reconcile code ↔ config; sweep expired release flags and collapse their branches (this feeds pass 2 rather than duplicating it).
- **Tools:** a reconciliation script (grep flag keys in code vs a `SELECT key FROM` the config table); a checked-in flag registry/manifest; `mcp__plugin_supabase_supabase__execute_sql`.

### 22. CI-as-Gate Drift — *are the audit's own tools actually blocking merge?*
> **Buckets:** delivery
- **What:** this playbook has you RUN dead-code, supply-chain, and secret scans during the audit — but the common state is a CI that only does typecheck, tests, and a build, with those tools living as local scripts that never block a PR. Every gain this audit makes can silently erode next ship because nothing fails the PR.
- **Look for:** which audit checks block merge vs. honor-system; any check added then disabled, `continue-on-error` crept in, or a workspace escaping the test/typecheck matrix this ship; whether `--frozen-lockfile` is the only lockfile/supply-chain guard.
- **Do:** confirm this audit's findings are now enforced in CI so they can't come back; flag any silently-disabled check. (Deciding the full required-gate set is the Architecture Review's job; this pass guards against regression.)
- **Tools:** GitHub Actions required-checks/branch protection; `gitleaks`/`trufflehog`; `pnpm audit --audit-level=high` / Socket.dev / Snyk; promote `knip` to a CI job; a `squawk` migration-lint step.

### 23. Performance & Cost Regression — *N+1s, missing indexes, cold-starts, and the bill*
> **Buckets:** scalability
- **What:** the draft treats bundle size as a vanity metric only and never audits runtime data cost. This is diff-scoped: did THIS ship regress against the perf budgets and index baseline the Architecture Review recorded?
- **Look for:** new N+1 patterns (a `.map()` firing one query per row — common in fan-out crons and list screens); queries filtering/joining on unindexed columns, especially new FK columns (Postgres does NOT auto-index FKs) and any column in an RLS `USING` clause; RLS policies with bare `auth.uid()` re-evaluated per row instead of `(select auth.uid())`; `select('*')` over-fetching wide rows; unbounded list queries with no `LIMIT`/pagination on growing tables; new top-level imports / eager client construction in Hono routes inflating cold-start; O(users) cron work in one invocation against the timeout ceiling (an in-memory `for` loop over all users with their settings and recent rows is a time bomb as the table grows); new or more-frequent crons, raised `maxDuration`, or a new noisy Sentry error path driving cost.
- **Do:** `EXPLAIN (ANALYZE, BUFFERS)` the hot queries this ship touched; add indexes on new FKs and RLS-predicate columns; wrap `auth.uid()` as `(select auth.uid())` in policies; collapse N+1 into one query/RPC; add pagination; snapshot the cost-driving deltas (cron frequency, maxDuration, fan-out queries, Sentry volume) next to the existing before/after metrics and right-size anything that grew without justification.
- **Tools:** `mcp__plugin_supabase_supabase__get_advisors` (flags unindexed FKs + RLS perf directly), `execute_sql` with `EXPLAIN ANALYZE`, `pg_stat_statements`; the `supabase:supabase-postgres-best-practices` skill; `mcp__plugin_vercel_vercel__get_runtime_logs` for cold-start/duration; `@next/bundle-analyzer` / `expo-atlas`; Sentry performance traces.

### 24. Data Durability & Migration Backout (this batch) — *can we get the data back if this migration was wrong?*
> **Buckets:** reliability
- **What:** the security pass covers RLS access but never asks "can we recover the data." Postgres holds all user state; a bad migration is a data-loss event, not a code bug.
- **Look for (this batch):** destructive DDL (`DROP`, type narrowing, in-place backfills) shipped with no verified recovery path or backout note; whether Supabase PITR/backups are even enabled on the project tier; secrets/keys (service-role, push, payments) touched this ship with no documented rotation procedure.
- **Do:** review each migration in the ship for reversibility and a backout note; flag any unguarded destructive DDL as a release blocker. (Confirming backups, setting RPO/RTO, and running a restore drill is the Architecture Review's job; this pass guards the per-ship diff.)
- **Tools:** Supabase backups/PITR + branch-restore; `mcp__plugin_supabase_supabase__list_migrations`; expand-contract pattern; migration-lint for `DROP`/`ALTER…TYPE`.

### 25. Runbook & On-Call-Surface Currency — *did this ship add a failure mode with no runbook?*
> **Buckets:** reliability
- **What:** the audit produces a cleanup report but no operational docs. When something breaks at 2am, what does the responder read?
- **Look for:** new failure surface this ship introduced (a new cron, dependency, env var, webhook) with no runbook entry or rollback note; missing runbooks for the obvious incidents ("push stopped", "entitlements wrong", "DB unreachable", "auth failing", "cron didn't run"); no documented deploy rollback or OTA emergency-republish path.
- **Do:** for each new on-call surface, add a thin runbook entry (symptom, where to look — specific Sentry/Vercel/Supabase view, likely cause, fix/mitigation incl. rollback command). A new on-call surface with no runbook is a finding.
- **Tools:** Markdown runbooks in the repo alongside this playbook; Vercel instant rollback + EAS channels/rollback; Sentry saved searches + Supabase log queries linked from each runbook.

### 26. Failure-Mode & Blast-Radius Drift — *did a new dependency join the critical path without a degrade decision?*
> **Buckets:** reliability
- **What:** pass 8 mentions "graceful degradation" in one line but never maps WHAT degrades when WHICH dependency fails. Best-effort deps (email, push) should degrade, not 500 a user's write.
- **Look for (this ship):** new routes/deps added with no degradation decision; best-effort work (push, email) on the critical request path so an email/push outage can fail a user write; a single failing dependency taking down unrelated routes (shared-fate); a fan-out cron whose per-item `catch` has no behavior for "whole batch fails" and no isolation between items.
- **Do:** verify the dependency-failure/blast-radius table (an Architecture Review artifact) still matches the code after this ship; move best-effort work off the critical path; flag any new dep with no fail-soft-vs-fail-hard decision. (Authoring the table is the Architecture Review's job; this pass checks drift.)
- **Tools:** `dependency-cruiser` to see what imports each external client (blast-radius map); Hono route grouping for isolation; explicit try/catch with documented fail-soft vs fail-hard.

### 27. Accessibility Regression (mobile + web) — *did new screens keep their labels and font scaling?*
> **Buckets:** accessibility
- **What:** no other routine pass looks at a11y; store review and a real slice of users are affected. If the repo has its own a11y rules doc, check new code against it; otherwise use the checklist below.
- **Look for (new/changed screens this ship):** RN touchables with no `accessibilityLabel`/`accessibilityRole`; icon-only buttons with no label; touch targets under 44pt; fixed font sizes/heights that clip OS Dynamic Type scaling (missing `maxFontSizeMultiplier`); color-only state signaling, contrast below WCAG AA; Next.js: missing `alt`, divs-as-buttons, modal focus traps, inputs with no associated label, keyboard-unreachable controls.
- **Do:** regression-check that new screens kept labels and didn't break font scaling; run automated baseline + spot-check VoiceOver/TalkBack on changed flows.
- **Tools:** `eslint-plugin-jsx-a11y`, `@axe-core/react` / Lighthouse a11y, Expo a11y inspector, VoiceOver/TalkBack manual sweep.

### 28. Privacy & PII-Egress Regression — *did this ship leak PII into Sentry/logs/analytics?* (distinct from the authz security pass)
> **Buckets:** security
- **What:** pass 10 covers authz/RLS; this covers PII leaving the system through telemetry, which pass 10 never looks at.
- **Look for (this ship):** PII in Sentry breadcrumbs/event payloads (emails, tokens, user content); analytics events with raw PII; `console.log` of sensitive data in production builds; full request bodies in Vercel/serverless logs; new tracking SDK firing before ATT/consent gating; App Store Privacy / Play Data Safety declarations drifting from the actual SDK/collection set; a working GDPR account/data-deletion path that respects RLS.
- **Do:** verify Sentry `beforeSend` scrubbing/denylist still covers new event shapes; strip PII from new analytics events; confirm consent gating fires before tracking init; reconcile store privacy declarations against any new SDK.
- **Tools:** Sentry `beforeSend`/data-scrubbing config; a PII grep over new log+analytics call sites; `expo-tracking-transparency`; App Store Privacy & Play Data Safety checklists.

### 29. Docs & Agent-Instruction Currency — *a wrong AGENTS.md is a correctness bug for the next Claude*
> **Buckets:** docs
- **What:** the "many Claudes touched it" tax. Stale `CLAUDE.md`/`AGENTS.md`/`README` actively steer the next agent into the ditch — and they drift in nearly every actively-developed repo. In an agent-runnable codebase this is the highest-leverage routine doc check.
- **Look for (diff against what changed this ship):** `CLAUDE.md`/`.claude/rules/*` describing architecture or conventions this ship changed; README setup steps or env-var docs now wrong; inline comments/JSDoc that now lie (e.g. an "idempotent" comment on a handler that still lacks the primitive); load-bearing decisions made this cycle with no short ADR.
- **Do:** reconcile `CLAUDE.md`/`.claude/rules` against what changed; do a clean-clone README smoke test if setup touched; delete or fix lying comments; capture any non-obvious decision as a one-paragraph ADR. Treat agent-facing docs as code under test.
- **Tools:** fresh-clone README smoke test; ADR template (MADR); `CLAUDE.md`/rules reconciliation; JSDoc/comment review; a CHANGELOG.

---

### 30. Comment & Code Truth
> **Buckets:** cleancode
- **Look for:** comments / JSDoc that *lie* — describe behavior the code no longer has, point at moved or renamed things, or claim a guarantee the code doesn't make (e.g. a comment saying "merges LWW here" above a blind `INSERT OR REPLACE`).
- **Do:** fix the comment to match the code, or delete it. A wrong comment is worse than none — the next human or agent trusts it and "optimizes" the wrong thing.
- **Tools:** grep for stale references; read each comment against the code it sits on.

### 31. Magic Numbers & Hardcoded Values
> **Buckets:** cleancode
- **Look for:** unexplained literals — timeouts, thresholds, retry counts, sizes, durations, limits, magic strings — scattered inline with no named constant.
- **Do:** lift them to named constants (with the unit/why in the name). Especially anything *tuned* (a 5s timeout, a 0.7 confidence cutoff) that someone will need to find and change later.
- **Tools:** eyes + grep for bare numeric/string literals in logic.

---

## Finish: verify and measure
- Run the **full** suite + typecheck + lint + build. All green.
- Re-measure: lines removed, bundle/app-size delta, type-errors killed, deps dropped. **Write the win down** — it's motivating and it justifies the time.
- Commit in logical chunks (per pass), then ship the cleanup.

## What the audit produces
- A cleaner, smaller, safer repo.
- A **structured findings report** written to `audits/<YYYY-MM-DD>-<bucket>-<HHMM>.md` — findings grouped by area, each tagged `[Severity · Cost-of-doing-nothing]` + location, plus Pass-0/Finish metrics. **This file is the handoff:** run `remediation-plan.md` on it to decide solutions (and upstream fixes), rather than trying to find-and-decide in one overloaded pass. (See [`_report-shape.md`](./_report-shape.md) and the README's _audit → report → action loop_.)
- Updated CI guards so the gains don't erode (lint, typecheck, and a dead-code check in CI).

---
_Living draft — refine as the process sharpens. Part of Varnish Tools._
