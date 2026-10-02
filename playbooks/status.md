# Status

_Progress across every Varnish run in this repo: what got fixed, what came back, what was accepted, and what's still open. Reads what's already in `audits/` plus a quick look at the code. It isn't a new audit, it doesn't change anything, and it's cheap. Part of Varnish._

## ▶️ How to run this
`/varnish status` (alias: `progress`) in the target repo. Optional scope: a date (`since 2026-09-01`) or a type (`security`).

The Audit-mode rules in `skills/varnish/SKILL.md` apply: look-only. Status writes **nothing** — not even to `audits/` — it prints to the chat. (If the user asks to save it, write `audits/<YYYY-MM-DD>-status-<HHMM>.md`; plans and audits ignore `*-status-*` files.)

## Step 1 — Inventory
Read every file in `audits/`:
- **Finding reports** — codebase buckets, `feature`, `seo`, `launch`. Note each report's date, type, scope, and HEAD SHA from its header.
- **Plans** (`*-plan-*`) — dispositions (`fix / accept / defer / disputed`), upstream fixes, work order, Accepted-Risk Ledger, Deferred triggers.
- **Ignore** Review deliverables (architecture-review, compliance) except to mention when each last ran.

No `audits/` folder, or nothing in it → say so in one line and recommend a first run (`launch` for a new or AI-built app, otherwise the bucket that matches their worry).

## Step 2 — Track each finding across runs
A finding's identity is **its class + where it lives** (file and function/table/route, not the line number — lines move). The same problem in two reports is one finding with a history. **If a plan exists, its IDs are canonical:** map later reports (including `launch` reports, which number their own findings) onto the plan's findings by class + location. Any report type that covers a finding's area counts as a later look at it (a `launch` run re-checks security findings; a `cleancode` run doesn't).

Give every finding exactly one status:

| Status | Meaning | How you know |
|---|---|---|
| **🔁 Came back** | It was fixed (absent from a later report of the same type, or its plan item was done) and it's in the newest report again — or a new instance of an already-fixed class turned up somewhere else. | Compare reports in date order. |
| **🔴 Open** | In the newest report that covered its area, with a `fix` verdict or no plan yet. | Newest relevant report. |
| **⏳ Decided, not done** | A plan said `fix` and it's still there — fully or partly. Partial fixes say what's left. Med/Low items that weren't spot-checked say "(not checked)". The single most useful thing this view surfaces. | Plan work order vs newer reports and the spot-check. |
| **✅ Fixed** | Raised before; a later report of the same type covering that area didn't raise it. | Report comparison. |
| **👀 Looks fixed** | No later report covers it, but the spot-check (Step 3) shows **nothing cited remains** (any part still visible → ⏳). **Unconfirmed** until the audit runs again. Fixes that live outside the code — rotating a key, a dashboard setting — can't be seen from the repo; say so instead of marking them fixed. | Spot-check. |
| **🆕 Unreviewed new code** | **New** files since the newest report, in an area where a class of problem was found before (new API routes after an auth finding, new migrations after an RLS finding). Changed files that reports already cite are covered by the spot-check instead. Not a confirmed finding — a place nobody has audited. If a glance plainly shows the old problem again, say what you saw on the 🆕 line ("looks like F04 again: no auth check"); it still stays 🆕, not 🔁, until a re-run confirms it. 🔁 is reserved for what a report has confirmed. | Step 3's change scan. |
| **🟰 Accepted** / **⏸ Deferred** / **❓ Disputed** | Per the newest plan's ledger. Flag any accept whose recorded assumptions no longer hold, or any defer whose trigger has fired. | Plan ledgers + repo signals. |
| **? Unknown** | Can't tell without re-running the audit. | — |

## Step 3 — Spot-check (keep it small)
**Change scan first:** `git diff --name-status <newest report's SHA>..HEAD`. List new and heavily changed files; flag the ones in areas where earlier reports found a class of problem as 🆕 (new `app/api/**` or `functions/**` after an auth or IDOR finding, new `supabase/migrations/**` after an RLS finding, new client files after a secrets finding). New instances of an old class are how fixes quietly come back, and only a re-run can confirm them — so 🆕 decides the Next run.

Then, for every **High** finding that's Open, Decided-not-done, or not covered by a later report: open the cited file at HEAD and look. If a cited file was deleted, every finding cited only in that file — any severity — becomes 👀 unless its problem moved somewhere visible. Did the code change? Is the problem still plainly there? That's a glance, not a re-audit — no verification subagent, no tools beyond reading. Mark `👀 Looks fixed` or leave it Open. Cap it at about fifteen findings; say how many you skipped.

Also give the reader a staleness sense: commits and changed files since the newest report.

## Step 4 — Report (in the chat)

```
Varnish status · <repo> · <today>
Since <first run date>: 23 found · 11 fixed · 4 look fixed · 2 came back · 5 decided-not-done (2 High) · 3 accepted · 1 deferred · 2 open
Last runs: launch 09-28 · security 09-12 · plan 09-29   (47 commits, 31 files changed since the newest)
🆕 3 new API routes since the last security run — not reviewed yet
```

Then, in this order, skipping empty sections:
1. **🔁 Came back** — first, always. A fix that didn't hold usually means a band-aid where an upstream fix was needed; say so, and point at `varnish plan` to find it.
2. **🆕 Unreviewed new code** — if any, right after Came back: "3 new API routes since the last security run. Nothing has checked them yet." Never let a clean-looking status hide this.
3. **⏳ Decided, not done** — plan items that never shipped, with the plan date.
4. **🔴 Open, High** — one line each with the finding ID and report.
5. **✅ Fixed since last time** — name them. Progress is fuel; show it.
6. **Ledger check** — accepts whose assumptions broke, defers whose triggers fired.
7. **👀 Unconfirmed fixes** — how many, and which audit to re-run to confirm them.
8. **Next run** — one recommendation, not a menu: the single run that would tell them the most. 🆕 code in a security-relevant area wins; otherwise the type with the most unconfirmed or stale findings, or `launch` again before inviting more users.

Plain English throughout. If `learn` was passed, add one lesson for any finding that came back: why that class tends to regrow.

Close with the author footer from `_report-shape.md`.

---
_v1.1, 2026-10-02 (change scan + 🆕, partial fixes, canonical IDs, outside-the-code fixes) after a test on a simulated week of work. Part of Varnish._
