_The deep, run-rarely **judgment call**: "is this the right system, built the right way — and should we rebuild any of it before we keep paying to extend it?" A **Review**, not an Audit — it ends in a *decision* (the rebuild-vs-refactor ledger), not a uniform findings list. Run rarely (new system, inherited codebase, post-rewrite, or when the routine Codebase Audit keeps finding the same rot). Companion to `codebase-audit.md`. Part of Varnish Tools._

# Architecture Review

_The heavy, run-rarely audit you do when a system is NEW, just had a major rewrite, was INHERITED, or has quietly outgrown the shape it was built in. The routine `codebase-audit.md` asks "are we still following the rules and is the code clean." This one asks the question that one never does: **is this the right system, built the right way — and should we rebuild any of it before we keep paying to extend it?**_

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say:** _"Read this file and run the architecture review on this repo."_

It's **read-only** and **evidence-led**: the agent gathers evidence (import graphs, git churn, schema dumps, query plans, Sentry density) and drafts the ledger — it does **not** change code or make the final calls. ("Start from evidence, not vibes" *is* the verification here: every verdict cites a graph/plan/metric, not a feeling.) Write the result to `audits/<YYYY-MM-DD>-architecture-review-<HHMM>.md`.

**Right-size to the system.** The passes are grounded in the product-app stack (Supabase/RLS, a Hono API, RN/OTA, RevenueCat). On a simpler system — a static site, a CLI, a library — most of Parts 2–4 (data architecture, distributed consistency, idempotency, reliability posture, scalability ceiling) will legitimately be **N/A**. Mark them N/A *with a reason*; do **not** invent architecture a system this size doesn't need. The depth scales to the system, not the other way around.

**This produces *decisions*, not a fix-list.** Bring the ledger to a human to make the keep / refactor / rebuild calls; they carry real cost and trade-offs (and timing matters — don't kick off a rebuild right before a launch). Nothing gets rebuilt off the back of this audit without an explicit decision.

## When to run it
- A new system / first real audit, before it ossifies.
- After (or instead of) a major rewrite, to sanity-check the new shape.
- When you inherit a codebase and need a debt map before committing a roadmap.
- When the routine audit keeps re-flagging the same subsystem — that's the system telling you it's a design problem, not a hygiene problem.
- ~Once or twice a year per system, max. This is expensive and produces decisions, not diffs.

## How this differs from the routine audit
| | Codebase Audit (routine) | Architecture Review (this doc) |
|---|---|---|
| Question | Did this ship add entropy / break the rules? | Are the rules/boundaries themselves right? |
| Output | A reviewable diff, pass/fail checks | A ranked **decision ledger** needing the owner's sign-off |
| Scope | The code WITHIN the design | The DESIGN itself |
| Cadence | Every ship | Rarely |
| Reversibility | Cheap, bisectable | Can mean throwing code away |
| Agent role | Runs the checks | Gathers evidence; human makes the calls |

**Rule of thumb:** if the output is a pass/fail or a small diff, it belongs in the routine audit. If the output is a recommendation that could mean a rewrite, it belongs here.

## The deliverable
One artifact: a **Rebuild-vs-Refactor Ledger** — a ranked table of subsystems, each with a verdict (keep / refactor-in-place / strangle-and-replace / rip-out), the evidence behind it, rough cost, and the seam you'd hide the work behind. Everything below feeds that ledger. Plus a handful of reference artifacts the routine audit then guards (target boundary map, source-of-truth map, SLOs, perf budgets, dependency-failure table).

## How to run it
- **Human-led, agent-assisted.** The agent pulls the evidence (import graphs, git churn, Sentry defect density, schema dumps, EXPLAIN plans, advisors). The human makes the judgment calls. Fan out the evidence-gathering passes in parallel; converge on the ledger together.
- **Start from evidence, not vibes.** Every verdict cites churn × defect density, a query plan, an import graph, or a constraint dump — not "this feels gross."

---

## Part 1 — System shape (is it drawn right?)

### A1. Module-boundary correctness — *are the seams in the right place, not just clean?*
The routine audit checks layers don't reach past neighbors and there are no cycles. It never asks whether the boundaries are drawn correctly.
- **Look for:** features that always change together but live in different modules (false separation) and unrelated concerns crammed together (false cohesion); a Hono route owning validation + business logic + Supabase queries + RevenueCat calls (no service seam); a `shared`/`utils`/`core` package that's become a dependency magnet everything imports (the thing you can never change); domain concepts smeared across UI, API, and DB with no home; barrel files leaking internals. **The tell is change-amplification:** a one-line product change forces edits in 4+ packages.
- **Decide:** map the actual import graph, overlay it on the INTENDED bounded contexts. For each boundary ask: single owner/reason to change? cohesion-inside > coupling-outside? narrow public interface? Redraw seams where graph and intent disagree. **Write down the target boundary map** — this becomes the reference the routine audit lints against (new cross-boundary imports, new entries into the magnet package, direct Supabase-client imports bypassing the data layer).
- **Tools:** `dependency-cruiser` (codify allowed-deps as config), `madge` (visual graph + cycle detection), `eslint-plugin-boundaries`; a git-log co-change analysis to surface files that always change together but live apart.

### A2. Right-sizing abstractions — *the inverse of the DRY pass: too MUCH abstraction*
The routine DRY pass only hunts too-little abstraction (duplication). The opposite is just as expensive and invisible to it.
- **Look for:** generic framework-y layers built for one or two callers (config-driven engines, plugin systems, base-class hierarchies) whose flexibility is never used; indirection you trace through 4 files to answer "what does this do"; premature interfaces with exactly one implementation; a wrapper around Supabase/Sentry/RevenueCat that adds a leaky layer without buying portability or testability; "extensible" extension points empty for months; abstractions whose seams don't match where change actually happens (so every feature modifies the abstraction itself — the surest sign it's wrong).
- **Decide:** does each abstraction have ≥3 real users, do its seams line up with the axis the code actually varies along, does it lower or raise the cost of the next likely change? Collapse one-caller indirection inline (rule of three in reverse), delete empty extension points, reshape wrong-axis abstractions. Bias toward the simplest thing still honest about the domain.
- **Tools:** import graph (`dependency-cruiser`/`madge`) to count real consumers; `ts-prune`/`knip` for single-implementation interfaces; coverage to find never-exercised extension points.

---

## Part 2 — Data architecture (the most expensive thing to get wrong)

### A3. Data model integrity, ownership & source-of-truth
On Supabase/Postgres+RLS this is the highest-leverage, hardest-to-change part of the system — and the routine audit only validates shapes at the edge (Zod), never the model.
- **Look for:** the same fact in two places that can disagree (RevenueCat entitlement vs a cached `is_premium` column vs a JWT claim — which wins?); missing/wrong FKs and no `ON DELETE`, so orphans accumulate; missing UNIQUE/CHECK so invariants live only in app code one Claude forgot to call; nullable-but-actually-required columns; enums as free text; money as float; timestamps without timezone; JSONB hiding what should be relational; no clear aggregate root (which table owns a write, what must be transactionally consistent with it); RLS policies that ARE the authz model but were never designed as one.
- **Decide:** for each core entity name the single source of truth; make every other copy a derived projection with a defined refresh/reconciliation path. Push invariants into the DB as constraints (FK, UNIQUE, CHECK, NOT NULL, generated columns) so illegal states are unrepresentable at storage, not just in TS. Treat RLS as a designed authorization model — enumerate every table's policies, confirm they match intended access. **Write down the source-of-truth map** (per fact: authoritative system + derived copies). Make TS types derive FROM the schema, not drift alongside it.
- **Tools:** `mcp__plugin_supabase_supabase__list_tables` + `execute_sql` against `information_schema`/`pg_constraint` to dump the real constraint set; `get_advisors` (security + perf); `generate_typescript_types`; the `supabase:supabase-postgres-best-practices` skill.

### A4. Query performance ceiling & the RLS multiplier
- **Look for:** N+1 patterns baked into the design (a fan-out cron doing in-memory `for` loops over all users, list screens firing a query per row); FK and RLS-predicate columns with no index (Postgres does NOT auto-index FKs); RLS policies with bare `auth.uid()` re-evaluated per row turning a point lookup into a seq scan on every request; `select('*')` over wide rows; unbounded queries on tables that grow.
- **Decide:** `EXPLAIN` the hot queries (use `ANALYZE` only on a non-production copy — it executes the statement); establish the **index strategy as a deliberate design**, not a missing-index patch; wrap `auth.uid()` as `(select auth.uid())` in policies; set the pagination convention. Record the index list + budgets as the baseline the routine audit diffs against.
- **Tools:** `get_advisors`, `execute_sql` + `EXPLAIN` (read-only; `ANALYZE` on a non-prod copy only), `pg_stat_statements`; Sentry performance traces for the real slow endpoints.

---

## Part 3 — Failure & evolution (a quietly-distributed system)

### A5. Consistency model & failure semantics across services
The DB + payments provider + push + email + telemetry each hold pieces of one user's truth — a distributed system that was never designed as one.
- **Look for:** a write that updates the DB then must reflect in the payments provider (or vice versa) with no reconciliation when they diverge; entitlement/paywall logic trusting one source with no fallback when the provider is unreachable; derived columns with no invalidation or staleness bound; no "which system is authoritative" rule per fact; no answer to "what's the UX when dependency X is down" (hard-fail / degrade / queue); fire-and-forget side effects (push, email, webhook fan-out) that silently vanish on failure with no dead-letter.
- **Decide:** per cross-service interaction, choose the consistency target — strong where money/entitlements demand it (transaction or verified webhook), eventual where delay is acceptable (with a max-staleness bound + a reconciliation/backfill job), best-effort where loss is tolerable (but make it an explicit choice). Name the authoritative source per fact; define degraded-mode UX per dependency. **Deliverable: a one-page "who's authoritative + what happens when X is down" map**, plus reconciliation jobs (e.g. a cron re-syncing payment entitlements to the DB and reporting drift).
- **Tools:** scheduled crons for reconciliation; DB RPC/transactions for must-be-strong writes; a dead-letter table for fire-and-forget; Sentry alerts on reconciliation drift.

### A6. Idempotency & write-safety doctrine on retried serverless
The platform retries, mobile clients retry, payment webhooks WILL redeliver. The classic tells: a payment webhook asserting idempotency in a comment with no primitive behind it, and no `AbortController`/retry anywhere in the API against the function timeout.
- **Look for:** the structural classes — write paths with no dedupe key; webhooks not signature-verified / event-id not recorded-once; multi-step writes (DB → payments → push) with no transaction or compensation; read-modify-write races; reliance on module-global state across stateless invocations.
- **Decide:** classify every write path by required guarantee and design the mechanism — natural or client-supplied `Idempotency-Key` + UNIQUE constraint; webhook event-id table with `INSERT … ON CONFLICT DO NOTHING`; a DB transaction or single RPC for atomic multi-row writes; atomic SQL or version columns for counters; sagas/compensation where one transaction can't span the work. These patterns, once established, become the routine audit's mechanical checklist.
- **Tools:** Postgres UNIQUE + `ON CONFLICT`; DB RPC; webhook signature verification + an events table; `AbortSignal.timeout`.

### A7. Evolvability under N live client versions
App-store lag + OTA updates mean several app versions run in prod simultaneously against one API + one schema. The routine audit only checks the CURRENT client matches the server.
- **Look for:** API/schema changes silently breaking for old clients (removed/renamed field, newly-required param, narrowed response, tightened enum); migrations applied expand-AND-contract in one step instead of expand → migrate → contract; OTA bundles assuming native capability an older binary lacks; no minimum-supported-version / forced-upgrade gate as the escape hatch for unavoidable breaks; no contract/version negotiation. **Note (local-first apps):** the device-side SQLite replica is a second schema with its own migration story — repo sync rules usually cover the server side; the DEVICE migration runner, outbox-drain-on-upgrade, and force-resync escape hatch are the usual foundational gaps.
- **Decide:** set the compatibility policy — support window (how many versions back), expand-then-contract as law for every schema/API change, and build the forced-upgrade/min-version gate (a config-served minimum-supported version is the usual seam) as the deliberate escape hatch. Decide what's versioned and how, including the on-device schema-version + migration runner.
- **Tools:** versioned shared Zod schemas (old vs new contract diffable); contract/snapshot tests asserting old-client shapes still validate; OTA channels + runtimeVersion (fingerprint policy, not a hardcoded literal); a server-side min-version check.

---

## Part 4 — Reliability & delivery posture (does the machinery exist at all?)

### A8. Reliability posture — SLOs, health, observability, durability
The foundational question is "do we have the reliability machinery at all?" (the routine audit asks "did it keep up with this ship?").
- **Establish:** define 2-4 SLIs on the journeys that matter (the core write, the main read/feed, payment/entitlement resolution) with explicit SLO targets + error budgets; build a real `/health` dependency probe (replace the static `{ok:true}`) returning per-dependency status; wire server-side error capture + a global `app.onError` into the API (mobile-only telemetry is a common gap); stand up alerting (error rules + cron/deploy-failure notifications to a real channel); confirm + drill backups/PITR and set RPO/RTO; choose the region/pooling/capacity model (single- vs multi-region, pooler mode for serverless, the O(users) cron scaling model before it hits the timeout ceiling).
- **Write the dependency-failure / blast-radius table:** per external dep (DB, payments, email, push, auth) — what breaks, blast radius (one user / one route / whole app), intended degraded behavior. Move best-effort work (push, email) off the critical request path.
- **Tools:** error-tracker releases/alerts; platform function metrics + cron notifications; DB advisors/logs; a pooler + a restore drill; `dependency-cruiser` for blast-radius mapping.

### A9. Delivery & operability architecture — *is the delivery MACHINE built right?*
- **Establish:** the **CI gate set** as required status checks (typecheck, tests, build, dead-code, dependency audit, secret scan, migration-lint — the common gap: CI gating only typecheck/test/build); the **migration doctrine** (expand/migrate/contract, an in-repo apply path tied to deploy ordering, a reversal note per migration — the dangerous default is forward-only with no down-path, applied out-of-band); the **rollback runbook** per surface (platform instant rollback, OTA republish/rollback, and the fact that migrations have no rollback — which is WHY they must be expand/contract; and verify auto-promotion actually promoted, it can silently pin prod to a previous build); the **env-parity model** + a single typed Zod env schema parsed at boot; **flag governance** (config-table flags get owner + type + expiry); a **cost baseline** (function invocations/duration, DB egress, telemetry/payment tiers, any high-frequency crons) with rough budgets.
- **Tools:** CI branch protection; `squawk`; a migration CLI wrapped in an ordered process; platform rollback CLI; env tooling; platform + DB usage dashboards.

### A10. Foundational ergonomics — onboarding & DX
Mostly a one-time paving job; the routine audit only re-checks "does setup still work / has CI gotten slower."
- **Establish:** time a from-zero setup; add `.env.example` + a seed/local-Supabase script + a one-command bootstrap; make CI fast and deterministic; add pre-commit hooks (lint-staged + typecheck) so cheap checks fail locally, not in CI. Stand up the doc set: README, ARCHITECTURE/ADRs, and the `CLAUDE.md`/`.claude/rules` agent contract — and reconcile them against reality (these drift everywhere).
- **Tools:** `.env.example`; Supabase local + seed; Husky + lint-staged; Turborepo caching; MADR ADR template.

---

## Part 5 — The verdict

### A11. Rebuild-vs-refactor ledger — *the question this whole audit exists to answer*
The routine audit assumes "refactor in place, small safe steps" — correct for it, wrong as a default here. Nothing else tells you when a subsystem is past the point where incremental cleanup pays off.
- **Look for the rebuild signals:** a module where every change has high blast radius and bugs keep reappearing in the same place (churn × defect hotspot); an abstraction bent so far from its purpose that new requirements consistently fight it; a data model so wrong half the code works around it; a dependency the product has outgrown; "we're scared to touch this" files with no tests and no owner.
- **Decide — force an explicit verdict per heavy subsystem:** keep / refactor-in-place / strangle-and-replace / rip-out, backed by evidence (git churn + Sentry defect density to rank hotspots, coverage to mark untested-and-scary zones, carrying-cost vs replacement-cost+risk, and whether a clean seam to replace behind even exists — if not, the first task is creating the seam, not rewriting). **Default to strangler-fig over big-bang:** stand the new implementation up behind the existing interface, migrate callers incrementally, delete the old path.
- **Deliverable:** the ranked ledger — `subsystem | verdict | evidence | rough cost | seam to hide the work behind`. This is the core artifact; everything above is the evidence that justifies each row.
- **Tools:** git churn analysis (`git log --numstat` aggregation, or code-forensics for churn-vs-complexity hotspot maps); Sentry issue grouping by file/module for defect density; coverage for the scary zones; `dependency-cruiser` to confirm a replaceable seam exists.

---
_Run this rarely, decide deliberately, then let the routine `codebase-audit.md` guard what you decided. Part of Varnish Tools._
