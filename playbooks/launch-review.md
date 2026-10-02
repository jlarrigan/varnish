# Launch Review

_"Is this safe to put in front of real users?" A go/no-go for apps built fast, often with AI, by people who may be shipping for the first time. It answers one question in plain language and ends with a verdict. Part of Varnish._

## ▶️ How to run this
**In the target repo's Claude session:** `/varnish launch` (or _"Read this file and run a launch review on this repo."_)

Claude will: detect the stack → run the checks below **look-only** (the Audit-mode rules in `skills/varnish/SKILL.md` apply in full: no edits outside `audits/`, no installs, no git writes, read-only SQL, ask before touching any live account) → verify every finding with a separate skeptic → write `audits/<YYYY-MM-DD>-launch-<HHMM>.md` → show the verdict.

**Who this is for.** Someone with a working app and a real question: can strangers see my users' data, run up my bills, or get my paid features for free? Write the whole report so that person can act on it. Every finding gets plain English and a fix they can hand to their AI tool. Jargon goes in parentheses after the plain version, never instead of it.

**What it isn't.** Not a full audit and not a security certification. It checks the dozen things that actually sink new apps — drawn from what has burned real vibe-coded apps in 2025-26: open databases, keys in the browser, paywalls enforced only in the browser, AI endpoints billing someone else's usage to you, apps with no backups. When the app is past launch and growing, run the `security` and `reliability` audits next.

## Step 1 — Detect the stack (two minutes)
Read `package.json`, config files, and folder names. Note: framework (Next.js, Vite/React, Expo/React Native, SvelteKit…), backend (Supabase, Firebase, own API, serverless functions, edge functions), payments (Stripe, RevenueCat, Lemon Squeezy…), AI providers (OpenAI, Anthropic, Gemini…), hosting (Vercel, Netlify, Lovable Cloud, Replit…), and whether it looks AI-built (Lovable/Bolt/v0 scaffolding, no tests, one-line commit messages). Then **load the matching stack modules** from `stacks/` (signals in `stacks/_index.md`) — they add each platform's exact files, key formats, and settings to the checks below. A check that doesn't fit the stack is **N/A with a reason**.

**Client-direct backends are normal here.** Lovable and most Supabase/Firebase apps let the browser talk to the database directly, protected by row-level security (RLS) or security rules. That's a valid design — it just means those rules are the only thing standing between strangers and your data, so check 2 matters most.

## Step 2 — The checks
Each check: what to look for, how to look (statically, no installs), and the plain-English stakes.

### 1. Secret keys the public can see
- **Look for:** server-only keys behind a client-exposed prefix (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`, `REACT_APP_`, `PUBLIC_`); a Supabase `service_role` / `sb_secret_` key, Stripe `sk_live_`/`sk_test_`, or an AI provider key (`sk-`, `sk-proj-`, `sk-ant-`) anywhere in client code or a mobile app; hardcoded keys in source. If a build output exists (`dist/`, `.next/`, `build/`), grep it — that's what the browser actually gets.
- **Severity:** a secret key that is (or ever was) behind a public prefix, in client code, or committed to git is **High even if nothing in the browser uses it today** — the key itself is exposed, and the next refactor ships it.
- **Not a finding:** keys that are *meant* to be public: Supabase anon / `sb_publishable_` keys, Stripe `pk_`, Firebase web config. Say so explicitly so the user isn't scared by them.
- **Stakes:** "Anyone who opens your site can copy this key and use it as you."

### 2. The database lets strangers in
- **Supabase:** every table in an exposed schema (usually `public`) has RLS enabled (look in `supabase/migrations/` for `enable row level security`); no policy that's just `USING (true)` on private data; INSERT/UPDATE policies have a `WITH CHECK`; no `grant all … to anon`; no `SECURITY DEFINER` functions callable by `anon`; storage buckets not public unless the files are meant to be. Policies must restrict to the owner (`auth.uid() = user_id`), not just "logged in" when the data is per-user.
- **Firebase:** no `allow read, write: if true` or expired test-mode rules in `firestore.rules` / `database.rules.json` / `storage.rules`.
- **If migrations aren't in the repo** (common with Lovable), the repo can't prove it. Put it in **Couldn't check**, and offer — with permission — to run Supabase's security advisor (`get_advisors`, type security) through a connected Supabase MCP.
- **Stakes:** "Anyone with your public key — which is in your website — can read or change this table."

### 3. Routes anyone can call
- **Look for:** list every API route, server action, and edge/cloud function. For each: does it check who's calling? Does it check that the caller owns the thing they're asking for (`/api/notes/123` → is note 123 theirs)? Next.js Server Actions are public endpoints even if no page links to them. Auth checked only in middleware, or only by hiding a button, doesn't count.
- **Stakes:** "Changing a number in the URL shows someone else's data." / "Anyone can call this directly, skipping your app."

### 4. Paid features enforced only in the browser
- **Look for:** "is Pro" / "has paid" / credit checks done in React components or local state, with the server or database never checking. Payment status a user could set themselves (a `plan` column with a permissive update policy).
- **Stakes:** "Anyone can unlock the paid features without paying."

### 5. Payments you can fake
- **Look for:** payment webhooks (Stripe, RevenueCat, Lemon Squeezy) that don't verify the signature, or that parse JSON before verifying (Stripe needs the raw body). Granting access from a redirect URL (`/success?paid=true`) instead of the webhook. No handling of cancellations/refunds.
- **Stakes:** "Anyone can send a fake 'payment succeeded' message and get your product free."

### 6. AI endpoints that bill you for strangers
- **Look for:** routes that call an AI provider with no login check, no rate limit, no per-user cap, no `max_tokens`. An AI key in client code (check 1). No spend limit set at the provider.
- **Stakes:** "Someone can use your AI endpoint as their own free AI, on your credit card. Leaked AI keys routinely run up thousands of dollars in a weekend."

### 7. No limits on the expensive or abusable stuff
- **Look for:** no rate limiting on signup, login, password reset, magic links, SMS/OTP, or email sends; no CAPTCHA/bot protection on public forms. SMS sends are the classic surprise bill.
- **Stakes:** "A bot can sign up 10,000 fake accounts or send thousands of texts on your account."

### 8. Secrets already leaked through git
- **Look for:** `git ls-files | grep -i env`, `.env` missing from `.gitignore`, `git log --all --oneline -- .env '*.env*'` — anything ever committed is leaked, even if it was deleted later, and doubly so in a public repo.
- **Stakes:** "These keys are in your project history. Deleting the file doesn't take them back — the keys have to be replaced."

### 9. Can you get your data back?
- **Look for:** evidence of backups / point-in-time recovery (Supabase paid tiers, Firebase exports), separate development and production databases (one `.env` pointing everywhere is the tell), seed or reset scripts that could run against production, an AI agent or MCP connected straight to the production database.
- **Usually Couldn't check** from the repo — list it for the dashboard.
- **Stakes:** "One bad command or one AI mistake could delete your users' data, with no way back."

### 10. Fake or risky packages
- **Look for:** dependency names one typo off a popular package (`lodahs`, `reqeusts`, `expresss`), packages that don't look real or are brand new, `postinstall` scripts. AI tools sometimes invent package names, and attackers register them. **Read `package.json` and the lockfile; never install to check.**
- **Stakes:** "A package your AI added may be malware that runs on your machine and your server."

### 11. Injection and unsafe rendering
- **Look for:** SQL built by gluing strings together with user input; `dangerouslySetInnerHTML` or rendering user/AI content as HTML; file uploads with no type or size limit; SVG uploads served back to users.
- **Stakes:** "Typed input can run commands against your database or your other users' browsers."

### 12. The launch basics outside the code
Mostly **Couldn't check** — list them as a short checklist: a privacy policy that matches what the app collects; a way for users to delete their account (required by Apple for apps with accounts); who owns the domain, hosting, database, and app store accounts, and whether they have 2FA on; spend alerts on every paid service; email sending set up properly (SPF/DKIM/DMARC) so password resets don't land in spam.

**Overlapping checks.** One problem often trips two checks (an open AI route is both check 3 and check 6). Write it once, as the more severe one, and name both checks.

## Step 3 — Verify
Hand the findings (ID, claim, `file:line`) to a **separate subagent** whose job is to disprove each one: is the key really a secret and not a publishable one? Does the route really skip auth, or is it checked upstream? Is RLS really off, or enabled in a later migration? It returns `confirmed / adjusted / dropped` with the quoted evidence, and may add a finding it trips over. Record the tally (N raw → M kept, K dropped, J adjusted, A added) on the second line of the report. Severity stays High / Med / Low. False alarms are worse here than in an expert audit — a scared beginner fixes the wrong thing.

## Step 4 — The verdict
- **🛑 HOLD** — any confirmed finding in checks 1-6 or 8 that's exploitable right now (real users' data readable/writable by strangers, a live secret key public, payments or paid features fakeable, an open AI endpoint). Don't launch, or if already live: contain today.
- **🟡 FIX THEN SHIP** — no HOLD items, but High findings that need fixing before you invite more users (no rate limits, no backups, a risky package).
- **🟢 SHIP** — nothing High. List what's left as "next" and what couldn't be checked.

The verdict is about the code that was checked. Never write "secure" or "safe" unqualified; write "no launch blockers found in what this review could check."

## The report
`audits/<YYYY-MM-DD>-launch-<HHMM>.md`, in this order:

1. **Verdict** — 🛑 / 🟡 / 🟢 in one line, then two or three sentences in plain English: what's wrong, and what it means for their users and their wallet. Directly under it, one small line: date · stack · HEAD SHA · verification tally · version stamp.
2. **🚨 Contain today** (on any HOLD) — the first step for each, in plain words. Leaked or public secret keys get rotated whether or not the app is live yet: the repo can't tell you who has already copied them. Usually: rotate the key in the provider's dashboard, switch on the access rule, or take the route offline. Rotation comes before code changes; a removed key that was public is still compromised.
3. **Fix before launch** — each finding:
   - `F01` · check name · `[Severity · Cost-of-doing-nothing]` · `file:line`
   - **What's wrong** (plain English, one or two sentences)
   - **What could happen** (to their users, or their bill)
   - **Fix prompt** — a short instruction they can paste into Lovable / Cursor / Claude Code, e.g. _"Enable row level security on the `notes` table and add policies so users can only select, insert, update, and delete rows where user_id equals their auth.uid(). Don't change any other tables."_
   - **How to check it worked** — one concrete test ("log in as a second account and try to open the first account's note URL — you should get an error").
4. **Next, after launch** — Med/Low findings, one line each.
5. **Couldn't check** — required, never empty. Everything that lives in a dashboard, not the repo: RLS actually enabled in production, backups, provider spend limits, live vs test keys, auth settings, domain/account ownership. A short checklist they can tick through themselves.
6. **What looked good** — a few lines. People new to this deserve to know what they got right.
7. **Scope and method** — scope (whole repo), what was skipped and why, version stamp `varnish <version> · launch-review · <date>`.
8. **Next step** — "Fix the Contain-today and Fix-before-launch items, then run `/varnish launch` again. Once you're live and growing, `/varnish security` and `/varnish reliability` go deeper. `/varnish plan` turns any report into an ordered plan."
9. **Author footer** (from `_report-shape.md`).

**Fix prompts are the point here.** This report deliberately breaks the Audit shape's "no solutioning" rule: its reader can't turn a finding into a fix on their own. Keep each prompt scoped to the one problem, in the user's tool's terms (prefer dashboard steps and plain instructions over git commands for Lovable/Bolt users).

**Tone.** Direct, calm, specific. No scare words without a concrete "what could happen." No blame — most of these come from tools' defaults, not carelessness.

---
_v1, 2026-10-02. Built for the people who build with AI and ship before they've had a senior engineer look. Part of Varnish._
