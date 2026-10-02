_The **deciding** step — the second half of the audit → report → action loop. Audits detect and hand back findings; **this playbook turns one or more findings reports into a decision-ready plan**: what's worth fixing, what to consciously accept, and where one upstream change collapses a whole cluster of findings instead of N band-aids. Part of Varnish Tools._

# Remediation Plan

_Every Audit report closes by handing off here (see `_report-shape.md` §4). This is that handoff's process, written down so it runs the same way every time. Its job is the two questions a findings pile never answers by itself: **(1) which of these are actually worth doing,** and **(2) of the ones worth doing, which are symptoms of something one or two steps up** — where the right move is a re-architecture, a shared abstraction, or a guardrail that makes the whole class of finding impossible, not another patch._

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say one of:**
- _"Read this file and build a remediation plan from `audits/<report>.md`."_ — one report
- _"Read this file and build a remediation plan from everything in `audits/` since `<date>`."_ — several reports at once (**better** — cross-report clusters are where the upstream fixes hide)

Claude will: **load the report(s) + project context + prior plans** (Step 0) → disposition every finding (Pass 1) → **hunt upstream fixes across ALL findings' clusters, then feed back** (Pass 2) → assign effort *to chosen solutions* and sequence (Pass 3) → write the plan to `audits/<YYYY-MM-DD>-plan[-<scope>]-<HHMM>.md` → stop and show you. Implementation happens after, off the plan.

**This playbook decides; it does not fix.** No code changes. It also does not re-verify findings — that was the audit's job. If a finding looks wrong, mark it `disputed` (it has an exit route — see Pass 1) and move on.

**Multi-report hygiene:** consume only Audit reports (the `_report-shape.md` contract). Exclude prior `*-plan-*` files (those are inputs to Step 0, not findings), `*-status-*` snapshots, and Review deliverables. `launch` reports are consumed like Audit reports. The same finding appearing in two reports is **one disposition row noting both sources** — that repetition is Pass-2 regrowth signal, not a duplicate.

## When to run it
- **Right after an audit lands** — the default second half of every audit run.
- After **several** audits pile up unplanned — batch them; cross-report clusters strengthen the upstream signal.
- When a fix-list has been sitting untouched — usually a sign findings were never turned into decisions (an undecided list costs willpower every time you look at it; a dispositioned one doesn't).

## Step 0 — Load the context (decisions need it)
An audit needs almost no context — a finding is true or it isn't (its one context read is the prior Accepted-Risk Ledger). **Disposition needs all of it**: "worth doing" depends on what this project is and what stage it's at. Before touching a finding:

1. **The report(s)** — every finding, every `[Severity · Cost-of-doing-nothing]` tag, the area grouping (grouped by subsystem precisely so clusters are visible here).
2. **The project's own context** — `PRODUCT.md` / `CLAUDE.md` / `README`: what is this, who uses it, what stage (prototype / pre-revenue / production / regulated), what's the current goal. A five-nines reliability finding disposes differently on a pre-revenue solo app than on a client's production system. **If no context docs exist:** infer stage from repo signals (deploy config, CI, tests, user-facing artifacts), ask the user **one** question ("what stage is this project at, and what's the current goal?"), and record the assumption in the plan header so it's correctable.
3. **Prior plans in `audits/*-plan-*.md`** — four reads, all cheap:
   - **Accepted-Risk Ledger:** a finding matching a prior accept is carried, not re-argued — *but check the accept's recorded context assumptions still hold* (an accept made "while pre-revenue" is void once there are paying users). Sweep the ledger as a whole, too: forty individually-rational accepts can sum to a quality ratchet no single decision saw — if the pile has grown, say so.
   - **Deferred:** a re-flagged finding whose defer trigger hasn't fired carries its existing defer (original date + trigger) unless its tags changed. If the trigger's status can't be determined from the repo, carry it and mark `trigger status unknown` for the user.
   - **Disputed:** any disputes the last plan logged should have been re-verified by the audit you're consuming (see `_report-shape.md`); make sure they resolved somewhere — surviving ones enter this plan as ordinary findings.
   - **Work order follow-through:** skim prior plans' work orders for items that never shipped, and list them in this plan's header as `inherited, still open`. "We decided this was worth doing and didn't do it" is the most valuable signal this loop produces — surface it, don't bury it.
4. **Prior audit *reports*** (skim headlines + area groupings only) — needed for Pass 2's regrowth check ("has this class been flagged before?").

## Pass 1 — Disposition: every finding gets a verdict
Walk **every** finding — none skipped, none left "we'll see":

- **`fix`** — worth doing. (Which *solution* is Pass 2's job; don't design here.)
- **`accept`** — real, but consciously not worth attention at this stage. Requires: a one-line reason, and **the context assumptions the accept depends on** ("while pre-revenue / solo-dev / no PII") — that's its mechanical invalidation condition, not an optional nicety. Accepts are **instance-scoped by default**: they cover *these* findings at *these* locations. New instances of the same class at new locations are new findings — and regrowth evidence. (A deliberately class-scoped accept must state a regrowth condition that re-opens it, e.g. "class accepted while ≤3 instances.") Accept is a first-class outcome, not a failure.
- **`defer`** — worth doing, not now. Requires a trigger. **The standing trigger "defer to the next `<bucket>` audit of this area" is legitimate** — it's concrete and self-checking (the next audit re-flags it if still true). A defer with no trigger at all is an accept wearing a costume — call it what it is.
- **`disputed`** — you think the audit got it wrong. Note why. **Exit route:** disputed findings are listed in the plan's Disputed section for the *next audit run to re-verify first*; survive re-verification → next plan treats them as ordinary findings; fail → closed. (If more than ~⅓ of a report ends up disputed, stop — tell the user the audit itself likely needs re-running; don't finish a plan on findings you don't believe.)

**Default verdicts from the report's 2×2** — the prior, not the answer; context may override any of them *with the reason stated*:
- **High/High → `fix`** now-ish.
- **High-severity / Low-cost → `defer`** with the standing next-audit trigger (this is the report-shape's "backlog" cell — real but stable).
- **Low / High-cost → `fix`** (sneaky compounder — the quadrant that punishes procrastination).
- **Low/Low → `accept`** with context assumptions recorded — unless the fix is nearly free.
- **`Med` on either axis → no default.** Judgment call, reason required. (Med is common; pretending the 2×2 covers it just makes the agent invent policy.)

**⚠️ The dormant-catastrophe carve-out:** cost-of-doing-nothing measures *trajectory*, not likelihood — so a latent injection hole, auth bypass, or data-loss path can sit "inert" and tag High/Low. **High-severity findings in the security / data-loss / irreversibility class never default below `fix` on trajectory alone.** Overriding one of those requires arguing low *exposure* (unreachable path, compensating control) — not low compounding.

## Pass 2 — The upstream hunt: fix causes, not instances
Cluster **all non-disputed findings — accepts included** — by root cause. (Classes hide precisely among the findings individually judged not worth fixing: each instance Low/Low, the class anything but.) A cluster must name its **concrete shared mechanism** — the same missing helper, the same copy-pasted shape, the same absent check. *A theme is not a cluster*: "insufficient validation" spanning half the codebase is a lens, not a mechanism, and licenses nothing.

Cluster sources: same subsystem, many findings (the report's area grouping shows this) · same mistake-class scattered across files · same missing guardrail · **the same class flagged in prior audit reports** (Step 0.4 — proven regrowth; a point fix is already known not to stick).

For each qualifying cluster, **pick the cheapest change that kills the class within acceptable blast radius** — the rungs are options to weigh, not a height to climb:

1. **Point fix** — patch each instance. Correct for true one-offs; the trap as a default.
2. **Shared abstraction** — one helper/module/pattern all instances route through, fixed once.
3. **Structural change** — re-architect the piece that keeps generating findings. *The "one or two steps up" move.*
4. **Guardrail** — make the class impossible or automatic *going forward*: a type that won't compile unvalidated, a lint rule, a CI check, RLS-by-default, a generator that starts new code correct.

**Rungs compose — and usually should.** Rung 4 alone fixes nothing that already exists; the most common winning shape is **rung 1 for the existing instances + rung 4 so the class never returns.** Say which rungs, for which halves.

**For each proposed upstream fix, state the honest comparison:**
> _N point fixes ≈ <effort>, and the class regrows. This <rungs> change ≈ <effort>, kills all N, prevents recurrence. Blast radius: <what else it touches>._

**Qualifying bar (anti-overreach — both failure modes are real):**
- A cluster of **≥3 same-mechanism findings**, or a class **flagged across ≥2 audits**, or a **High-severity class with ≥2 instances or a stated recurrence mechanism**. One High-sev one-off is, by rung-1's own language, just a fix.
- **Recurring *structural* findings escalate to the Architecture Review regardless of size** — recurrence there means the design is wrong, which is that Review's question, not this plan's (the Varnish Tools `architecture-review.md` playbook; README's escalation tell). Recurrence at rungs 2/4 stays in-plan. A structural proposal that outgrows a small bounded project likewise escalates rather than smuggling a rebuild into a fix plan.
- A genuine one-off is just a fix. A band-aid on a wound that will never reopen is called a repair.

**Feed back into Pass 1:** a chosen upstream fix changes the economics of everything it touches. **Re-open any `accept` or `defer` the chosen fix collapses** — "accepted at standalone cost" is void when the fix rides along free — and note the flip in the disposition table (`accept → fix (collapsed by U1)`).

## Pass 3 — Effort and sequence (now, and only now)
Effort was deliberately banned from the audit report — it belongs to *solutions*, and solutions now exist. Assign **effort per chosen solution** (`S / M / L`), then sequence:

1. **Quick wins** — S-effort `fix` point fixes **not collapsed by any chosen upstream fix** (don't patch Tuesday what Wednesday's change deletes). Momentum is fuel; a plan that opens with a three-day project opens unstarted.
2. **Upstream projects** — each with its collapsed-findings list attached (the payoff made visible), effort, blast radius.
3. **Escalations** — anything flagged for an Architecture Review, named as such.

An upstream fix enters the work order if it collapses **any** `fix`-verdict finding; an all-defer cluster lists under Deferred with the earliest trigger. Singleton defers get no solution design — just the trigger. The sequence is a **suggested order, not a schedule**.

## The plan (the written artifact)
**Filename:** `audits/<YYYY-MM-DD>-plan[-<scope>]-<HHMM>.md` — same folder as the reports; date/time from the shell. Scope: inherit the input report's scope for single-report plans; omit (or use `batch`) for multi-report plans. Sections:

1. **Header** — date · input report(s) · context sources read (+ any stage *assumption* made) · prior ledgers inherited (`none` is valid on first runs) · **inherited-still-open items** from prior work orders.
2. **Decision headline** — one line: N findings in → X fix (of which Y collapse into Z upstream fixes) · A accepted · B deferred · C disputed. X+A+B+C = N is the self-check; previously-accepted carryovers live in the ledger, not the headline.
3. **Disposition table** — every finding: ID (report's finding ID, else `file:line`) · `[Severity · Cost]` · verdict · one-line reason (including any Pass-2 flips). Compact — the audit report keeps the detail.
4. **Upstream fixes** — each: the change (one paragraph, rungs named), findings collapsed (IDs), effort, the honest-comparison sentence, blast radius.
5. **Work order** — quick wins → upstream projects → escalations.
6. **Accepted-Risk Ledger** — every accept: reason · **context assumptions it's valid under** · scope (instance/class) · optional revisit trigger — including inherited accepts (original dates). Durable: future audits and plans read this so decided stays decided *while its assumptions hold*.
7. **Deferred** — each with its trigger (carried defers keep original date + trigger).
8. **Disputed** — each: why, and the note that the next audit re-verifies these first.
9. **Version stamp** — `remediation-plan.md v<version> · run <date>`.
10. **Author footer** — the same closing line every Audit report carries (see `_report-shape.md`).

**Contain first.** If any input report has a 🚨 Contain-today section, those items go **above the work order as Phase 0, today**, in plain words, regardless of upstream fixes or quick-win rules. A plan that defers rotating a live key until after a refactor is wrong. Containment isn't the fix — the underlying finding still gets dispositioned normally.

**Learn mode** (if the user added `learn`, or the input report was in Learn mode): end the plan with "What to take from this" — three to five lessons, each tied to the fixes that teach it. Format per `_report-shape.md`.

**Write for the person who will act on it.** Open the plan with a one-screen **"Do these today"** summary in plain language (at most 5 items) before the tables. If the repo looks like a first-time or AI-built app, give each work item a short prompt the user could paste into their AI tool (Lovable, Cursor, Claude Code) to make the fix, plus how to check it worked.

**Degenerate cases:** zero findings → a two-line plan recording the clean result (still worth the ledger/follow-through reads). All-accept → the plan is the ledger; say so plainly.

**→ Handoff:** the plan goes to implementation — item by item in the repo's session, or via a planning skill (superpowers writing-plans) for the bigger upstream projects. Future audits check this plan's ledger and Disputed sections before writing findings (see `_report-shape.md`).

---
_v1.2, 2026-10-02 — contain-first Phase 0, plain-language summary, paste-able fix prompts, footer. v1.1, 2026-07-21 — v1 same day, hardened after a 3-critic adversarial review (defaults/trigger collision, accept-cluster blindness, dormant-catastrophe carve-out, ledger regrowth-muting, disputed exit, rung composition). Born from re-prompting the same two instructions after every audit: "decide what's worth doing" and "look for the fix one or two steps up." Part of Varnish Tools._
