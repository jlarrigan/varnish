_The shared report contract every **Audit** writes (Codebase Audit, Feature Audit, SEO Audit). The two **Reviews** (Architecture Review, Compliance Review) have their own deliverables and don't use this. The point of one shape: any Audit's report is a **predictable interface** — a human, a brainstorm, or a future skill can consume any of them the same way. Part of Varnish Tools._

# Audit Report Shape

## The one principle: the report *detects*, the brainstorm *decides*
An Audit's job is to find what's true and characterize it — **not** to decide what to do about it. Detecting, prioritizing, and planning all at once is the working-memory overload the whole audit → report → action loop exists to avoid. So the report **describes problems; it does not prescribe solutions, effort, or a work order.** That happens next, in a brainstorm — which is also where you spot the **upstream fix that collapses ten findings into one** (the band-aid-on-band-aid trap a per-finding fix-list walks you straight into).

## Every finding carries two axes (and only two)
- **Severity** — how bad is it *if it bites*? Impact magnitude. `High / Med / Low` — three levels only; if a verifier says "Critical", it's High (note the raise in the tally).
- **Cost of doing nothing** — what does *leaving it* cost? Its trajectory: does it sit inert, or compound / rot / spread / eventually bite? `High / Med / Low`.

These two — and *not* effort — are what let you triage each finding into **backlog it** vs **queue it now**:

| | Cost-of-doing-nothing **Low** | Cost-of-doing-nothing **High** |
|---|---|---|
| **Severity High** | real, but stable — **backlog** | **do it now** |
| **Severity Low** | cosmetic — whenever | sneaky-compounding — **queue it** |

**Why not Effort?** Effort is a property of the *solution*, not the *finding* — and you don't know it until you've decided *how* you're fixing it. Tagging effort per-finding silently assumes you'll fix each one individually, which is exactly the band-aid trap. **Effort gets assigned in the brainstorm, on the chosen solution** — which may be one upstream change that kills a whole cluster.

## The shape
**Filename:** `audits/<YYYY-MM-DD>-<type>[-<scope>]-<HHMM>.md`. Get the date/time from the shell (`date +%Y-%m-%d`, `date +%H%M`, local) so same-day re-runs don't collide. `<type>` = the bucket for the Codebase Audit (`cleancode`, …) or `feature` for the Feature Audit; `<scope>` = optional qualifier (a feature slug, a subsystem).

1. **Header** — date · audit type · target / scope · base ref + HEAD SHA (or "whole repo") · mode (look-only) · verification tally: **"N raw → M kept (K dropped, J adjusted, A added by the verifier)"**. No tally, no "verified" claim.
2. **🚨 Contain today (only if something is exposed right now)** — the one exception to "no prescriptions." If a finding means real users or real money are at risk *today* (a live secret key in the browser or in git history, a database table anyone can read or write, an open AI endpoint billing you, a payment path anyone can fake), list it here first, in plain words, with the first containment step (usually: rotate the key in the provider dashboard; switch on the access rule; take the route offline). One or two lines each. Everything else waits for the plan.
3. **Findings headline** — one can't-miss line reading the state: how many findings, how many **High-severity**, how many **High-cost-of-doing-nothing**, and the worst-hit areas by name. It's a *read of the findings*, not a directive. (An Audit may add its own at-a-glance layer here — e.g. the Feature Audit's per-dimension scorecard, or the Codebase Audit's per-bucket scorecard on an `all` run — as long as it stays a *characterization*, not a command.)
4. **Findings, grouped by area / subsystem** — *not* pre-sorted by severity. Grouping by where they live makes **clusters visible** ("five of these are all in the sync layer — maybe the sync layer is the problem"), which is what feeds the remediation plan's upstream-fix hunt. Each finding:
   - a **stable ID** — `F01, F02, …` numbered through the report. Remediation plans, ledgers, and future audits reference findings by `<report-file> F<nn>`, so the ID is the contract; `file:line` is the fallback for legacy reports.
   - the **`[Severity · Cost-of-doing-nothing]`** tag,
   - **location** (`file:line`),
   - **what** it is and **why it matters** (the cost of leaving it),
   - for every **High**: one **In plain English** line a non-engineer can act on — what a stranger could do, or what it costs you ("Anyone can read every user's notes by changing the number in the URL."),
   - optionally a brief **direction** — a hint, not a prescribed solution.
   - **No effort. No prescribed sequence.**
   Mark an area/pass/dimension **N/A with a reason** rather than forcing a finding.
5. **→ Handoff** — close every report with: *"Run `remediation-plan.md` on this report to decide solutions — disposition every finding, hunt the upstream fixes that collapse clusters. Effort and sequencing get decided there, on the chosen solutions."* (`playbooks/remediation-plan.md` is the written-down version of that brainstorm; it runs the same way every time.)
6. **Version stamp** — `varnish <plugin version> · <playbook> · <date>`.
7. **Author footer** — close every report with this line, verbatim:
   > _Varnish is Jordan Larrigan's playbooks, written down. Stuck on a finding, or want a second set of eyes on the fix? Reach Jordan on [LinkedIn](https://www.linkedin.com/in/jordanlarrigan)._

## Respect prior decisions (the Accepted-Risk Ledger)
Before writing findings, check `audits/*-plan-*.md` for **Accepted-Risk Ledgers** and **Disputed sections** from past remediation plans:
- **Accepts are instance-scoped:** a finding matching a consciously-accepted *instance* (same class, same location) is **not re-litigated** — one line under a "Previously accepted" note (item · original date · reason). But **new instances of an accepted class at new locations are full findings** — they're the regrowth evidence the escalation rules run on; suppressing them would mute exactly the recurrence signal that's supposed to trigger the upstream fix. Re-raise a matched accept as a full finding only if its severity/cost genuinely changed, its revisit trigger fired, or its recorded context assumptions no longer hold.
- **Disputed findings from the most recent plan get re-verified first** (adversarially, like any finding): survive → full finding again, noted "re-verified"; fail → note it closed. This is the disputed verdict's exit route.

Audits that re-argue decided risks every run train the reader to skim — which is how the real findings get missed.

## What's deliberately *not* here
- **No "recommended fix order" / "do this first" work plan.** A triage *signal* (the two axes + grouped findings) — yes. A prescribed *sequence* — no; that's the brainstorm's call.
- **No effort estimates per finding** (see above).
- **No solutioning.** The report can hint a direction; it doesn't design the fix. (The one exception is the Contain-today section, because waiting for a plan while a key is live costs real money.)

An Audit may layer **measurement** on top of this (e.g. the Codebase Audit's Pass-0 baseline metrics + before/after deltas) — that's data, not a decision, so it fits.

---
_v1.1, 2026-10-02 (contain-today, plain-English line, verification tally, footer). v1 2026-06-28. The Audit envelope — Reviews keep their own shapes. Part of Varnish._
