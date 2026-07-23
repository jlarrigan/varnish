_The **vertical** audit: point it at ONE freshly-built feature and grade how it was built — every dimension at once — before it becomes load-bearing. Where `codebase-audit.md` sweeps one dimension across the **whole repo** and `architecture-review.md` judges the **whole system**, this traces a **single feature down through every layer** (UI → API → DB → external → jobs → config → tests) and hands back a graded scorecard + the findings. Part of Varnish Tools._

# Feature Audit

_You just built a feature. It works. But "it works" and "it's built right" are different questions — and the cheapest moment to fix the second one is **now**, while the feature is fresh, small, and not yet load-bearing. This audit takes one feature, grades how it was built across eight dimensions, and returns a **graded scorecard + the findings** — for you and your AI to decide what to do._

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say one of:**
- _"Read this file and audit the `<feature>` on this repo (diff against main)."_ — **diff mode** (default)
- _"Read this file and audit the `<feature>` feature — it lives in `<paths>`."_ — **named mode**

Claude will: **resolve the base branch and build the slice map** (every file/layer the feature touches) → run each dimension **read-only** against just that slice (find, don't fix) → **adversarially verify every finding** → **grade each dimension A–D from the surviving findings** → write the findings + scorecard to `audits/<YYYY-MM-DD>-feature-<slug>-<HHMM>.md` → stop and show you. You then run `remediation-plan.md` on the report to decide what to fix, and fix in the repo. _(Run it in the target repo's own session — the fixing happens there.)_

**It's read-only and right-sized.** A dimension that doesn't apply to this feature (a11y on a backend job; Evolvability on a feature that changes no schema/contract) gets marked **N/A with a reason** (shown, not graded) — never a forced finding. Depth scales to the feature.

## When to run it
- **Right after building a feature, pre-merge** — the sweet spot. The slice is exactly the diff, and nothing downstream depends on it yet, so fixes are cheap.
- Just after merge, before you build the *next* thing on top of it (before it's load-bearing).
- On any feature you're unsure about — "I rushed this, did I do it right?"
- On an inherited/old feature you're about to extend, to know what you're standing on.

**Feature Audit vs Codebase Audit — same diff, different cut.** Both can run against a branch diff, so the difference isn't scope, it's the *cut*. The **Codebase Audit** sweeps each **dimension across whatever changed** (one bucket at a time — breadth, hygiene). The **Feature Audit** takes the same diff but traces **one feature down through every layer and grades all dimensions at once** (depth, a graded read). Reach for the **Feature Audit when the diff is one coherent feature** and you want a graded read on how it was built; reach for the **Codebase Audit when the diff is a pile of mixed small ships** and you want hygiene breadth. (And the **Architecture Review** when the question is "is the whole system's design right.")

## How it differs from the other audits (the axis)
The other audits run **horizontally** — one dimension, swept thin across the repo. This runs **vertically** — every dimension, deep, on one feature's slice. Tracing a feature *end to end* surfaces what a thin horizontal sweep skates past (e.g. *this* feature's authz is client-only, *this* query has no index), because you followed the feature instead of sampling the repo.

| | **Codebase Audit** | **Architecture Review** | **Feature Audit** (this doc) |
|---|---|---|---|
| Cut | horizontal — a dimension × diff/repo | the whole system's design | **vertical — one feature × all dimensions** |
| Question | "did this ship drift / still clean?" | "is the design right? rebuild?" | **"is *this feature* built right?"** |
| Scope | diff or whole repo | whole system | **one feature's slice** |
| Cadence | every ship / on a cadence | rarely | **on demand, per feature** |
| Output | a findings report | a rebuild-vs-refactor ledger | **A–D grade per dimension + the findings (you decide ship/fix)** |

**Think of it as the Architecture Review's hard questions, shrunk to one feature and run on demand** — light enough to do the moment you finish building, instead of once or twice a year.

## Step 1 — Resolve the base & build the slice map (do this first; everything runs against it)
Before grading anything, establish **what the feature actually is** as a set of files and layers. This map *is* the audit surface, and it's held in context until it lands in the report's slice-map section at the end (the single write).

- **Resolve the base branch — don't assume `main`.** Detect the real integration branch (`git symbolic-ref refs/remotes/origin/HEAD`, else look for `main`/`master`/`develop`, else the branch this one was cut from). Record the chosen base in the report header. Use **three-dot** diff so the feature stays isolated even if the base has advanced: `git diff --stat <base>...HEAD` (and the full diff).
- **Named mode:** start from the named entry points and trace outward until the feature's boundary is clear.
- **Map the footprint across every layer** — `UI / API / DB + migration / external / jobs / config + flags + env / tests`. A whole layer that *should* be present but is missing (a user-facing write with **no** test; a DB write with **no** migration in the diff; a new env var with no schema/validation) is itself a finding — note it now.
- **Also note the feature's *consumers*** — what already imports or calls into this slice. The seam test ("could you rip it out") and the whole "before it's load-bearing" framing turn on who depends on it; you can't assess removability without knowing.

## The dimensions (the scorecard)
Grade each **A–D** (rubric + mechanical grading rule below). For each, the agent looks for the items **within the slice only**, cites `file:line`, and proposes a `→` fix. Examples are grounded in the product-app stack (Supabase/RLS, Hono API, RN/OTA, RevenueCat) — treat them as *principles* and apply them to THIS repo's stack. **If a named tool isn't installed here, fall back to static inspection / grep and mark the finding _unverified-by-tool_ (lower confidence) rather than skipping the check.**

**On reuse, stated plainly:** dimensions **2–5 and 8** check the same failure *classes* the Codebase Audit's scalability/security/reliability/delivery/a11y buckets do — the difference is they run **vertically on one feature's slice**, so they catch *this* feature's specific hole that a thin horizontal sweep misses. Dimensions **1 (the seam test), 6 (Fit), and 7 (Completeness)** are **feature-native** — they only exist at feature scope.

### 1. Architecture & seam
> Lead question (feature-native): **if this feature failed or had to be ripped out, how hard would it be?**
- **The seam/deletability test:** does the feature sit behind a narrow interface, or has it leaked its internals into shared state, the magnet `utils`/`core` package, and five unrelated screens? Use the *consumers* list from Step 1 — an experiment you can't cleanly delete is one that's already too expensive. This is the dimension's identity.
- **Plus the usual layering hygiene, scoped to the slice:** business logic crammed into a component/route instead of a testable service/core module; data-access reaching across layers (a component hitting Supabase directly, a frontend bypassing the API); a god-function doing fetch + transform + business-rule + render; the wrong abstraction (a generic "engine" built for one caller, or zero abstraction where the same shape repeats 3×). _(Same checks as the Codebase Audit's architecture bucket / Architecture Review A1–A2 — here, scoped to the slice.)_
- **Tools:** `madge`/`dependency-cruiser` on the slice (does it import across boundaries? does the rest of the app import its internals?).

### 2. Performance & scalability
> Built for *its own* growth — not the whole system's, just this feature's data and traffic. (The N+1 / index / RLS checks apply **even at low traffic** — don't mark N/A just because the feature isn't hot.)
- **Look for:** N+1 patterns the feature introduces (a list firing a query per row; an in-memory loop over all users); queries on this feature's tables with no index on the filter/FK/RLS-predicate column (Postgres does **not** auto-index FKs); RLS policies with bare `auth.uid()` re-evaluated per row; `select('*')` over wide rows; **unbounded** lists/queries with no pagination on a growing table; a cron/job whose cost is O(users) heading for the 30s ceiling.
- **Budget:** if the repo has a perf budget / index baseline (the Architecture Review sets one), check this feature against it — not just for anti-patterns.
- **Tools:** `EXPLAIN (ANALYZE, BUFFERS)` the hot query; `get_advisors` (perf).

### 3. Security & privacy
> The classic feature-shaped holes — authz, input, data scope, and PII egress.
- **Authz / input / access:** authorization enforced **only on the client** (the #1 trap — gate it server-side too); inputs from the user/client not validated at the API boundary (no Zod parse); the feature's data access not actually scoped (does RLS / the query *really* restrict to the right user/org, or does it leak?); committed secrets/keys; an external webhook it consumes that isn't signature-verified.
- **Privacy / PII-egress (its own failure class — the authz checks never look here):** PII into Sentry breadcrumbs / event payloads, analytics events, or logs (emails, tokens, user content, **full request bodies**); a new tracking SDK firing **before consent / ATT gating**; new data collection that drifts from the store privacy declaration; a missing/again-broken data-deletion path.
- **Tools:** trace every write/read for a server-side authz check; secret scan on the diff; a manual "can user A see user B's data through this feature?" pass; confirm RLS on any new table; check Sentry `beforeSend` scrubbing + grep the slice for PII in logs/analytics.

### 4. Reliability
> The SRE lens, scoped to the feature.
- **Look for:** swallowed errors (`catch {}`) on the feature's I/O; no timeout/retry on its network calls; **boundary/brick risk** — untrusted data (API response, DB/SQLite row, external payload) `as`-cast but never *validated*, where one bad value crashes or bricks the feature downstream; **a new route/job with NO error-capture path at all** (vs. a default 500, zero capture); the feature invisible to observability (no Sentry breadcrumb/trace, no log on the path that matters); **a new cron/background job or failure path with no on-failure alert to a human** (silent failure — captured signal is worthless if nothing pages someone); no user-facing error/empty/loading state; a hard dependency (RevenueCat, Resend, push) with no defined behavior when it's down; fire-and-forget side effects that vanish silently on failure.
- **Tools:** grep the slice for empty catches and boundary `as` casts; confirm Sentry actually captures this path; list the feature's external deps and ask "what happens when each is down, and who finds out."

### 5. Evolvability & data safety *(if the feature changes schema or a client contract; else N/A)*
> The highest-stakes "before it's load-bearing" question: **does this change break what's already deployed — existing data, or app versions already in the field?** A new feature is the single most likely thing to ship a destructive migration or break old clients.
- **Migration safety:** non-additive migration (DROP / RENAME / retype, `NOT NULL` without default, a backfill in the same statement as the DDL); missing FK / `ON DELETE`, `UNIQUE`, `CHECK`, `NOT NULL` so an invariant lives only in app code; nullable-but-actually-required columns; enum-as-free-text; money-as-float; timestamp-without-tz; no down-path / reversal note; migration applied out-of-band vs. deploy order; a new fact duplicating an existing source of truth with no reconciliation.
- **Contract / client-compat (N live versions):** a new/renamed/removed API field or newly-required param while old clients are still live; a narrowed response or tightened enum; an OTA-shipped JS change assuming native capability an older binary lacks; no min-version / forced-upgrade gate for an unavoidable break; for local-first, an on-device replica column the installed SQLite DB lacks (device migration story).
- **Rule of thumb:** every schema/API change should be **expand → migrate → contract**, additive and reversible. If it isn't, that's the finding.
- **Tools:** `squawk`; `list_migrations` / `pg_constraint` dump; `get_advisors`; diff the shared Zod contracts old-vs-new.

### 6. Fit *(feature-native)*
> Does it match how **this repo already does things**, or did it invent a parallel universe?
- **Look for:** a new data-fetching pattern when the repo has an established one; a hand-rolled component where a shared one exists; its own date/money/format util duplicating one in `core`; naming that diverges from the repo's domain language (the same concept under a new name); a new dependency added for something the repo already solves. **The tell:** a teammate reading this feature would think "we don't do it this way here."
- **Tools:** compare against a sibling feature; grep for existing utils/components/patterns it should have reused; check for net-new deps in the diff.

### 7. Completeness *(feature-native)*
> Is it actually *finished*, or finished-looking?
- **Look for:** half-wired stubs, `TODO`/`FIXME`/`HACK`, dead branches, commented-out scaffolding; a happy-path-only implementation (no error/empty/loading/offline state actually built); hardcoded placeholder values; a feature flag with no off-path; **no tests for the feature's own logic** (or tests that assert nothing).
- **Docs are part of "finished" (this is an agent-runnable codebase):** a convention/architecture this feature changed, or a new env var/setup step it added, **not reflected in `CLAUDE.md` / `.claude/rules` / README**; an inline comment that now lies about the new code. A feature that left the docs lying isn't done — it actively steers the next agent wrong.
- **Tools:** grep the slice for TODO/FIXME/console.log/placeholder; confirm the diff includes a test file for any non-trivial logic; click through the states in your head — empty / error / slow network.

### 8. Accessibility *(if the feature has UI; else N/A)*
> WCAG 2.2 AA basics — your standing bar — applied to the new surface, **plus the journey-level check only the vertical cut can make.**
- **Look for:** interactive elements with no accessible label / role; non-text contrast and color-only meaning; focus order and keyboard/screen-reader reachability on the new controls; touch-target size; motion without a reduce-motion respect; new images without alt text.
- **Journey check (feature-only):** is the feature's **whole new flow operable end-to-end by keyboard / screen-reader as a complete journey** — not just each control in isolation? (A per-control sweep is the Codebase Audit's job; an end-to-end-journey check is what only a feature-scoped audit sees.)
- **Tools:** the repo's a11y linter on the changed components; a screen-reader pass on the new flow. **Backend/no-UI feature → mark N/A.**

### Grading each dimension (mechanical — from the *verified* findings)
The grade is the dimension's at-a-glance **severity read**; map findings → letter so two runs land the same grade:
- **A** — no findings.
- **B** — only Low/Med findings.
- **C** — at least one High finding, but the dimension's **core contract still holds**.
- **D** — a High finding that **breaks the dimension's core contract** (authz bypass, data leak, destructive/irreversible migration, a design the feature constantly fights), or a pile of unresolved findings.
- **N/A** — doesn't apply (say *why*); shown, not graded.

The grade summarizes **severity** per dimension. Each finding *also* carries its own **`[Severity · Cost-of-doing-nothing]`** tag — that two-axis tag is what you triage individual findings by, and what the headline counts (see [`_report-shape.md`](./_report-shape.md)).

**Severity floor:** any unresolved **High security/privacy or High data-loss/brick** finding caps its dimension at **D** and is **named in the headline** — so a live exploit or data-loss bug can't hide behind otherwise-clean grades. (Surfacing it, not blocking the ship — that call is still yours.)

## Findings are adversarially verified (built in, not optional)
After the find phase, **every finding is re-checked by a fresh verifier whose job is to *disprove* it** — grep the repo to confirm a "client-only authz" / "no index" / "unused" / "dead" claim is actually true, check the cited line really says what's claimed, and right-size the severity. Drop anything that doesn't survive; note what got adjusted. *A scorecard of verified findings is worth ten times one of plausible ones.* Grading happens **after** this step, on the survivors. Per-dimension skeptic stances:
- `security` → "can this actually be exploited?"
- `reliability` → "can this failure really happen, and what's the blast radius?"
- `fit` *(feature-specific)* → "does the pattern it 'should have reused' actually exist in this repo?"

## The deliverable — scorecard + findings
Write it in the shared **Audit shape** (see [`_report-shape.md`](./_report-shape.md)) — findings-first, grouped by dimension, each tagged `[Severity · Cost-of-doing-nothing]`, closing with a **→ handoff to `remediation-plan.md`**. The Feature Audit's one addition to the shared shape is the **A–D scorecard** as its at-a-glance layer.

**Filename:** `audits/<YYYY-MM-DD>-feature-<slug>-<HHMM>.md` (date/time from the shell). **Slug:** lowercase the feature name, spaces/underscores → hyphens, strip non-alphanumerics (in diff mode, derive from the branch name if none given). e.g. `audits/2026-06-28-feature-streaks-1455.md`.

Shape:
- **Header** — date · feature name · base ref + scope (`diff vs <base>` / named) · mode (read-only) · playbook · method ("adversarially verified").
- **Headline** (up top, can't-miss) — a *read* of the scorecard, **not** a directive: the count of findings (and how many **High-severity** / **High-cost-of-doing-nothing**) and the lowest-graded dimensions by name. **You + your AI decide whether it ships** — the audit reports, it doesn't issue the go/no-go. **Design-concern flag:** if any dimension hits **D on Architecture & seam or Security & privacy**, say so plainly — that's a design-level problem, not a line-fix, and may warrant scoping an **Architecture Review** on this slice (the seam to the Review class).
- **Slice map** — the layers + key files the feature spans (from Step 1), with any missing-layer findings and the consumers list.
- **Scorecard** — the 8 dimensions as a table: `dimension | grade | one-line headline`. N/A rows shown with the reason.
- **Findings, grouped by dimension** — each dimension marked `*Applicable.*` / `*N/A — why*`; each finding tagged **`[Severity · Cost-of-doing-nothing]`** with `file:line` + what-and-why (the cost of leaving it), optionally a brief direction; plus a ✅ *Clean:* line for what passed. **No per-finding effort, no prescribed fix-order.**
- **→ Handoff** — *"Run `remediation-plan.md` on this report to decide what to fix and how — disposition every finding, and look for upstream fixes that resolve findings across dimensions at once. Effort and sequencing get decided there."*

Example:
```
FEATURE: streaks   base: develop   scope: diff vs develop

Headline: 7 findings (2 High-severity, both High cost-of-doing-nothing).
Lowest: Security (D), Evolvability (C). ⚠ Security D is design-level
(client-trusted identity) — consider a scoped Architecture Review on the auth path.

Dimension                Grade  Headline
Architecture & seam        B    streak logic in a testable service ✓; leaks into 3 screens
Performance & scalability  A    paginated, indexed, no N+1
Security & privacy         D    reset endpoint trusts client user_id
Reliability                C    no error state on the streak fetch; not in Sentry
Evolvability & data safety C    adds streak_count NOT NULL, no default → breaks old rows
Fit                        B    reuses core date utils ✓; new fetch hook vs repo's useQuery
Completeness               C    2 TODOs; offline state stubbed; no test for the reset path
Accessibility              A    labels + contrast + keyboard journey clean

Findings (grouped by dimension):
  Security & privacy
    [High · High]  api/streaks.ts:40  reset endpoint derives user from the request
                   body, not the session → any user can reset anyone's streak.
  Evolvability & data safety
    [High · High]  migrations/..streaks.sql  streak_count added NOT NULL, no default
                   → migration breaks every existing row.
  Reliability
    [Med · Med]   ui/StreakCard.tsx:88  no error state on the fetch; failure shows blank.
  Completeness
    [Low · Med]   streaks.service.ts  2 TODOs; reset path has no test.

→ Run remediation-plan.md: decide fixes + look for one upstream change (e.g. a shared
  authenticated-mutation helper) that closes several at once. Effort decided there.
```

## Execution: order & fan-out
- **Solo:** resolve base + build the slice map, then work the 8 dimensions against it top-to-bottom.
- **Agent fan-out (how I actually do it):** build the slice map once, then spin up one agent per dimension in parallel — each scoped to the slice, each returning a **findings list with `[Severity · Cost-of-doing-nothing]` tags only (no letter grade)**. Then the aggregator runs adversarial verification **first**, applies the grade-from-findings rule to the *survivors* to assign each letter, and writes the headline last. **Grading happens once, centrally, after verification** — so the letters always match the findings that survived.

---
_v1, 2026-06-28. Run it the moment a feature is done, fix what it flags while the fix is still cheap, then let `codebase-audit.md` guard it from there. Part of Varnish Tools._
