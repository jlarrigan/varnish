# Varnish

_The finishing coat for shipped code. You varnish woodwork after you build it so the work lasts; same idea. (No relation to the HTTP cache.)_

Jordan Larrigan's engineering playbooks: the stuff that's usually in my head after 20 years of building products and running tech orgs, written down so I don't have to re-explain it every time, and so other people (and their AI agents) can use it too.

## Install

```
/plugin marketplace add jlarrigan/varnish
/plugin install varnish@varnish
```

That's it. `/varnish` is now available in every repo. _(No install? See [Other ways to use this](#other-ways-to-use-this) below.)_

## Use

Open Claude Code in the repo you want audited and pick a lane. **New to shipping, or built it with Lovable, Bolt, Cursor, or Replit? Start with `/varnish launch`.**

| Command | What it does |
|---|---|
| `/varnish launch` | **Start here.** "Is this safe to put in front of real users?" Plain-English go/no-go: open databases, keys the public can see, routes anyone can call, fakeable payments, AI endpoints billing you for strangers. Every finding comes with a fix prompt you can paste into your AI tool |
| `/varnish security` | One bucket: `cleancode · architecture · security · reliability · scalability · delivery · accessibility · docs` |
| `/varnish feature <name>` | Vertical audit of ONE feature, every dimension, graded A–D. Run it pre-merge |
| `/varnish seo` | Marketing-site SEO + AI-search visibility (every finding carries an evidence tier) |
| `/varnish plan` | **The second half:** turns your findings reports into a remediation plan |
| `/varnish architecture-review` | The deep, run-rarely judgment: rebuild vs refactor |
| `/varnish compliance` | Where you roughly stand vs SOC2 / ISO 27001 / PCI / HIPAA |
| `/varnish everything` | The full codebase audit, every bucket in one sweep. Long and token-heavy; Claude confirms before starting |

**Audits only look; they never change your code.** Every run is read-only except for writing its report to `audits/`: no edits, no commits, no installs, no deploys, read-only database queries, and Claude asks before touching any live account (Supabase, Vercel, Stripe…). Fixing happens afterwards, in a normal session, when you decide.

**Built your app in Lovable, Bolt, or v0?** Connect it to GitHub, clone the repo to your computer, open Claude Code in that folder, and run `/varnish launch`. The report's fix prompts can be pasted straight back into your builder.

## What you get

Every audit writes a **findings report** to `audits/` in your repo, adversarially verified (a skeptic re-checks every claim before it's written), every finding tagged with severity and the cost of ignoring it:

```
audits/2026-07-23-reliability-1042.md

Findings: 11 total · 3 High-severity · 4 High-cost-of-doing-nothing
Worst-hit: the API error path, cron jobs

── The API ────────────────────────────────────────────
F03 [High · High] api/entry.ts: no global error handler; an unhandled
    error on most routes returns a default 500 with zero capture.
F04 [Med · High]  api/webhooks/billing.ts:88: webhook handler not
    idempotent; provider redelivery double-writes entitlements.
...
```

Then `/varnish plan` turns the report(s) into **a decision-ready remediation plan**: every finding gets a verdict (`fix / accept / defer`), clusters get hunted for the **one upstream fix that replaces N band-aids**, and the work is sequenced quick-wins-first:

```
audits/2026-07-23-plan-1115.md

11 findings in → 7 fix (5 collapse into 2 upstream fixes) · 3 accepted · 1 deferred

Upstream fix U1: one global app.onError + request-scoped logger
  Collapses: F03, F07, F09   Effort: S
  7 point fixes ≈ a day and the class regrows. One handler kills all
  three and catches every future route. Blast radius: error responses.

Work order: 1. U1 (S)  2. F04 idempotency key (S)  3. U2 env schema (M) ...
```

Accepted risks go in a ledger future audits **respect**: consciously declined findings stay declined instead of being re-argued every run.

## Want a second set of eyes?
Varnish is how I'd review your project if you asked me. If you're stuck on a finding, not sure the fix is right, or want me to look at the whole thing, reach me at [varnishlabs.io](https://varnishlabs.io). Every report ends with the same pointer.

## A typical session

```
you>  /varnish reliability
      … audit runs read-only, verifies its findings, writes the report …
claude> Report written to audits/2026-07-23-reliability-1042.md
        11 findings: 3 High-severity. Worst-hit: API error path, crons.
        Run `varnish plan` to turn this into a remediation plan.

you>  /varnish plan
      … reads the report + your project context, dispositions every finding …
claude> Plan written to audits/2026-07-23-plan-1115.md
        7 fix (2 upstream fixes collapse 5 of them) · 3 accepted · 1 deferred.
        Quick wins first: the global error handler is S-effort and kills 3 findings.

you>  fix the quick wins
      … now it's a normal coding session, working off the plan …
```

Audits **find**; the plan **decides**; you (and your agent) **fix**. Detection, decision, and repair stay separate on purpose. That's what keeps each one honest.

## Other ways to use this

- **No plugin, just playbooks:** clone the repo, then in your target repo's session:
  ```
  Read <path-to-this-repo>/playbooks/codebase-audit.md and run the reliability audit on this repo.
  ```
  _(Claude will ask permission to read outside the repo; allow it.)_
- **As reading**: every playbook stands alone as a written process a human can follow.

## Why this exists
1. **For me**: stop retyping the same process every project. Externalize it once, reuse it forever.
2. **For others**: share how I think about building and shipping software.
3. **For AI agents** (Claude Code, or any agent that can read files): these are written to be *run*, not just read. Point an agent at a playbook and it can execute the process. A doc that just sits there is worth a fraction of one an agent can act on.

## Two kinds: Audits and Reviews
The playbooks split into two classes, and the split *is* the format:

- **Audits**: *findings-first, routine, repeatable.* They sweep for issues and hand back **findings** (severity + what it costs to ignore); **you and your AI decide** what to do. Every Audit shares the **same report shape**, so they feel identical to run and to read.
- **Reviews**: *judgment calls, run rarely.* Each produces a **decision or specialized deliverable** in **its own shape** (a rebuild-vs-refactor ledger; a compliance crosswalk matrix). Reviews are allowed to differ from each other; Audits are not.

Rule of thumb: output is "here's what's wrong, you decide" → **Audit**. Output is a considered verdict or a bespoke artifact → **Review**. And connecting them: the **Planner** (`remediation-plan.md`), the Audits' shared second half, which turns any findings report into decisions (it consumes the Audit shape; it isn't one).

## What's here
Each playbook maps a **stage of building a product or running a tech org**: what to think about at that stage, what to look for, and how to act.

**Audits** (shared findings shape):
- `playbooks/codebase-audit.md`: the **routine, whole-repo** cleanup & hardening pass, after shipping a big feature (or a pile of small ones). **Self-contained and runnable:** a bucket table up top, each pass tagged with its bucket(s). _"Read this file and run the `cleancode` audit on this repo."_ Buckets: `cleancode · architecture · security · reliability · scalability · delivery · accessibility · docs · all`.
- `playbooks/feature-audit.md`: the **vertical / per-feature** audit: point it at ONE freshly-built feature and grade how it was built across 8 dimensions (architecture & seam, performance, security & privacy, reliability, evolvability & data safety, fit, completeness, a11y). Run it the moment a feature is done, pre-merge. It's the Architecture Review's hard questions shrunk to one feature.
- `playbooks/seo-audit.md`: the **visibility** audit for a *marketing site*: classic SEO + **GEO** (AI Overviews / ChatGPT / Perplexity citation readiness) in one pass, because the platforms say they're the same discipline. Distinctive contract: **every finding carries an evidence tier** (`[P]` platform-confirmed / `[S]` large-scale study / `[R]` peer-reviewed / `[C]` consensus) so "why should I care?" is always answerable, plus a **Cargo-Cult Ledger** that flags dead practices (llms.txt, FAQ schema, keyword density…) as findings. Born from a manual pass on a real production marketing site (2026-07); research-swept 2026-07-17.

**The Planner** (the Audits' shared second half):
- `playbooks/remediation-plan.md`: the **deciding** step every Audit hands off to. Takes one or more findings reports and returns a plan: every finding dispositioned (`fix / accept / defer` + `disputed`, accept is first-class), fix-clusters hunted for **upstream solutions** (the ladder: point fix → shared abstraction → structural change → guardrail that kills the class), effort assigned *to chosen solutions*, quick-wins-first work order, and a durable **Accepted-Risk Ledger** future audits respect. Born from re-prompting the same two instructions after every audit (2026-07-21).

**Reviews** (own shape each):
- `playbooks/launch-review.md`: the **pre-launch go/no-go** for people shipping their first app, often built with AI. Twelve checks drawn from what actually sinks new apps (open databases, public secret keys, routes anyone can call, fakeable payments, AI endpoints billing you, no backups, fake packages), written in plain English with a paste-able fix prompt per finding, a required "couldn't check" list, and a 🛑 HOLD / 🟡 FIX THEN SHIP / 🟢 SHIP verdict.
- `playbooks/architecture-review.md`: the **deep, run-rarely** design judgment: "is this the right system, built right, and should we rebuild any of it before we keep paying to extend it?" Ends in a rebuild-vs-refactor ledger. Run rarely (new system, inherited codebase, post-rewrite).
- `playbooks/compliance-review.md`: a **technical compliance precheck**: per technical control, where you roughly stand across **SOC2 / ISO 27001 / PCI DSS / HIPAA** at once (backbone = CIS Controls v8). A gut-check to get *ahead* of an audit; explicitly **not** an attestation, repo-observable controls only.

- _More to come: project kickoff, feature build, release/ship, incident response, running the org, hiring, code review… (TBD)_

## Which one? Codebase Audit vs Architecture Review
They cover overlapping *topics* (architecture, reliability, …) but ask different *questions*:

| | **codebase-audit** (routine) | **architecture-review** |
|---|---|---|
| Question | "Did *this ship* drift? Still clean / following the rules?" | "Is the *design* right? Rebuild vs refactor?" |
| Scope | diff-scoped (what changed), whole repo when there's no usable diff | whole-system |
| Cadence | often (after ships, on a cadence) | rarely |
| Output | a findings report | a judgment / rebuild-vs-refactor ledger |
| Assumes | the architecture is roughly right | nothing; it questions the architecture |

**Default to the Codebase Audit.** Reach for the Architecture Review when you're starting fresh / inherited a codebase / just did a big rewrite, **or when the Codebase Audit keeps flagging the same structural thing.** That recurrence is the tell: routine treats symptoms; if the same symptom keeps regrowing, the *design* is wrong, and that's the Architecture Review's job.

**On the overlap:** intentional. Same topics, two altitudes (the Codebase Audit: "did we drift?"; the Architecture Review: "is the target right?"). The Audit hands you findings to act on; the Review hands you a decision. Same subject, different job.

## The skill layer
The playbooks are wrapped in a **skill** (`skills/varnish/SKILL.md`) you invoke by type: `varnish <bucket>` (e.g. `varnish security`, or `varnish everything`) / `varnish feature` / `varnish seo` / `varnish plan` (Audits + the Planner) and `varnish architecture-review` / `varnish compliance` (Reviews). It assembles just the passes for that type, then (optionally) fans them out as agents.

This dissolves the "one doc vs two docs" question: the **passes are the data**, tagged by which audit types they belong to; the skill is the router that selects them. How the files are arranged matters less than the tags.

**Audit types to route by:** the buckets are **inline in `codebase-audit.md`**: a bucket table at the top, and a `> Buckets:` tag on every pass (`cleancode`, `architecture`, `security`, `reliability`, `scalability`, `delivery`, `accessibility`, `docs`, plus `all`). A pass can belong to several buckets; the skill just runs the passes tagged with the requested one. **Wide coverage is intended**; a pass that doesn't apply gets marked N/A, not forced.

**Beyond the Codebase Audit's buckets**, each other playbook routes to its own file: `feature` (`feature-audit.md`, all 8 dimensions *vertically* on one feature's slice), `seo` (`seo-audit.md`, the marketing-site visibility audit), and the Reviews `launch` (`launch-review.md`), `architecture-review` (`architecture-review.md`), and `compliance` (`compliance-review.md`). So the skill's full type set is: `launch` · the Codebase Audit buckets · `feature` · `seo` · `plan` (`remediation-plan.md`, the loop's second half) · `architecture-review` · `compliance`.

**Convention for new passes:** add the pass to `codebase-audit.md` with a `> **Buckets:**` tag line. That tag *is* the routing; there's no separate table to keep in sync.

## The audit → report → action loop
Don't stop at "run the audit." Each audit **writes its findings to a structured report**: an `audits/` folder with a file per run named `<YYYY-MM-DD>-<bucket>-<HHMM>.md` (e.g. `audits/2026-06-25-cleancode-1455.md`: date + bucket + time so same-day re-runs and different buckets never collide): findings grouped by area, each tagged severity + cost-of-doing-nothing with its location, plus before/after metrics. That report is the **handoff**: `remediation-plan.md` turns the findings into decisions and solutions, including the upstream fixes that collapse a cluster at once. (A freehand brainstorm is the ancestry; the playbook is the mechanism.)

**Why it's the right instinct (not overthinking):**
- It **separates *finding* from *deciding***, two genuinely different jobs. Detecting + prioritizing + planning in one pass is a working-memory overload for humans and agents alike; a written report lets you detect, step away, and come back to plan with a fresh head.
- It leaves a **record**: what was found, when, what got fixed, the before/after numbers.
- The report is a **clean interface**: a brainstorm can consume it, a human can, a future agent can. Mirrors a detect→spec→plan flow.

**Where the overthinking risk actually is:** building a heavy status-tracking system on top of it. Don't. Keep it a plain dated markdown report. The skill just has to *write* the file; what consumes it can be as simple as "now brainstorm this."

**Full shape:** `varnish <type>` (skill) → runs the matching passes → writes `audits/<YYYY-MM-DD>-<type>[-<scope>]-<HHMM>.md` → **run `remediation-plan.md` on it** (disposition every finding, hunt upstream fixes, sequence) → implement off the plan. The deciding step used to be a freehand brainstorm re-prompted every time ("what's worth doing? what's the fix one or two steps up?"); it's now a playbook, so it runs the same way every time and leaves an Accepted-Risk Ledger the next audit respects.

## Conventions (rough; will evolve)
- One playbook per stage/process.
- Each is built to be **skimmable by a human and executable by an agent**: when to run it, the goal/mindset, then concrete passes/checklists with real tooling.
- **Versioned.** See `CHANGELOG.md`. Update with `/plugin marketplace update varnish`.

_Started 2026-06-24._
