# SEO & AI-Search (GEO) Audit

_The visibility pass for a marketing site: can search engines and AI answer engines find you, extract you, and cite you — and is anything you're doing (or paying for) cargo cult? Classic SEO and GEO (generative engine optimization) are audited together because the platforms say they're the same discipline: Google's own AI-optimization guide calls AEO/GEO "still SEO."_

## ▶️ How to run this (read me first)
**In the target repo's own Claude session, say:** _"Read this file and run the SEO audit on this site."_ Point it at the marketing site's repo + production URL. Works best with GSC/GA4 access; runs degraded without (Pass 0 records what's missing).

Claude will: run the passes **read-only** (find, don't fix) → **adversarially verify every finding** (a fresh skeptic re-checks each claim — re-fetch the URL, re-run the command, confirm the cited doc actually says that) → write the *verified* findings to `audits/<YYYY-MM-DD>-seo-<HHMM>.md` → stop and show you the report. Then run `remediation-plan.md` on the report to decide solutions (disposition + upstream fixes).

Report format: the shared **Audit shape** — see [`_report-shape.md`](./_report-shape.md). Findings-first, `[Severity · Cost-of-doing-nothing]` tags, grouped by pass so clusters show, no effort estimates, closing handoff to `remediation-plan.md`.

## ⚖️ The evidence rule (what makes this audit different)
**Every finding must carry its evidence tier**, so when someone asks "why should I care?", the answer is in the report:

- **[P] Platform-confirmed** — Google/OpenAI/Anthropic/Microsoft official docs or on-record statements.
- **[S] Large-scale study** — Ahrefs / Semrush / Vercel / Pew / Seer-class data (name the study + year + n).
- **[R] Peer-reviewed** — published research (the KDD 2024 GEO paper is the anchor).
- **[C] Expert consensus** — widely-agreed practice with mechanism but no controlled data. Say so.

Two hard rules that fall out of this:
1. **Never present [C] as [P].** "Google says" is reserved for things Google actually says.
2. **Flagging cargo cult is a finding.** If the site (or a vendor invoice) is spending effort on a dead practice, that's a `[Med · Med]` finding — wasted spend compounds. The Cargo-Cult Ledger below is the checklist.

## 🧭 Right-sizing (the mindset — where audits lose credibility)
The evidence says visibility problems are lexically boring: **indexation, raw-HTML availability, and extraction beat everything else.** Weight accordingly:
- **Severity caps:** CWV failures are `Med` max unless catastrophic (LCP > 6s / INP > 500ms) — Google: CWV is "not [a] giant factor," relevance dominates. HTTPS was "<1% of queries" at launch. Never rank a perf tweak above an indexation or rendering finding.
- **Suppress crawl-budget advice** below ~10k indexable URLs — Google's own docs scope crawl budget to 1M+ page sites (small sites "don't need to read this guide").
- **No "page experience score," no "duplicate content penalty," no "E-E-A-T score"** — none exist per Google. Frame duplicates as consolidation issues, E-E-A-T as rater-rubric evidence, page experience as individual small signals.
- **site: counts are not indexation data** (Mueller: incomplete, filtered). Zero results = smoke-test alarm; anything else needs GSC's Page Indexing report.
- **CWV verdicts come from CrUX field data (p75), never Lighthouse scores** — lab data is diagnostic only per Google's PSI docs. No CrUX data (low-traffic site)? Report "not assessable by Google's criteria," don't substitute lab numbers.

---

## Pass 0 — Baseline & access
Record before finding: production URL(s) · indexable URL count (crawl or sitemap) · CMS/framework + rendering mode (SSR/SSG/CSR) · GSC available? GA4 available? CrUX data exists? · current GSC 16-month clicks/impressions trend (API: `searchanalytics.query`, branded vs non-branded split via brand-term regex) · GSC **Generative AI performance report** numbers if present (launched June 2026 — AI Overviews/AI Mode impressions; note it blends surfaces) · CrUX p75 LCP/INP/CLS for origin + top templates.
**Why:** the GSC Search Analytics API is the highest-value programmatic source (16-month window) [P: Google API docs]; the gen-AI report is the only first-party AI-visibility number that exists [P: Search Central blog, June 2026].

## Pass 1 — Retrievability: crawl & index
The load-bearing pass. Checks:
- **robots.txt** — fetch + parse; accidental `Disallow: /`; the **robots/noindex trap**: a page blocked in robots.txt can't have its `noindex` seen, so it can still index. Flag any URL both disallowed AND noindexed, and disallowed URLs appearing in Google. [P: Google "Block Search Indexing" doc]
- **Sitemap** — referenced from robots.txt; valid XML; every URL 200 + canonical + indexable; `lastmod` plausible (all-identical dates = ignored). **Don't tune `priority`/`changefreq` — Google ignores both.** [P: sitemap docs]
- **Canonicals** — exactly one per page, absolute, self-referencing on canonical pages, target is 200 + not-noindexed. Canonical is a *hint* not a directive — flag canonical chains, canonical-to-redirect, canonical+noindex contradictions. [P]
- **Redirects** — flag chains ≥2 hops, loops, meta-refresh/JS redirects where a 301 belongs, internal links pointing at redirecting URLs. 301s pass full signal (no PageRank loss since 2016). [P]
- **Soft 404s** — fetch a guaranteed-nonexistent URL; assert 404/410 (not 200, not 302-to-home). SPAs that render an empty shell are the classic cause. [P]
- **Indexation reality** — GSC Page Indexing report: indexed vs excluded with reasons; "Discovered — currently not indexed" ratio. [P]
- **Mobile parity** — mobile-first indexing is 100% complete (Oct 2023): desktop-only content effectively doesn't exist. Diff smartphone-UA render vs desktop for content/meta/schema parity. [P]
- **HTTPS hygiene** — all pages https, one-hop http→https, valid cert, no mixed content. (Lightweight signal — baseline hygiene, not a headline finding.) [P]
- **Click depth & orphans** — crawl from home; flag money pages at depth ≥4, orphan pages (in sitemap, zero inlinks). Click depth matters, URL folder depth doesn't (Mueller: "we don't count slashes"). [P]

## Pass 2 — Raw-HTML availability (the GEO keystone)
**The single highest-leverage 2026 check, because it serves both SEO and GEO:** major AI crawlers — GPTBot, ClaudeBot, PerplexityBot — **do not execute JavaScript.** They read the initial HTML response only. Google is the lone renderer. Client-side-rendered content is invisible to every AI answer engine. [S: Vercel/MERJ study, Dec 2024, 500M+ GPTBot fetches — zero evidence of JS execution]
- `curl` each key template raw (and with AI-bot UAs); render the same page headless; **diff the visible text.** Any content, pricing, product facts, or nav links present only post-JS = finding, severity `High` if it's money content.
- Verify title/meta/canonical/robots exist in the raw HTML, links are real `<a href>` (not onclick), no client-side-injected canonicals.
- Framework check: Next/Astro/Nuxt SSR/SSG = pass; empty `<div id="root">` SPA = fail.
- Google-only nuance: Googlebot renders on a delayed queue — SSR still recommended for content-critical pages. [P: JS SEO basics doc]

## Pass 3 — AI crawler access matrix
Each vendor runs **separate bots for training vs. search vs. live-fetch, with different consequences.** Parse robots.txt + test actual responses (CDN/WAF can 403 what robots.txt allows — curl with each bot's UA):

| Token | Blocking it means |
|---|---|
| `OAI-SearchBot` | invisible in ChatGPT search answers [P: OpenAI bot docs] |
| `GPTBot` | opted out of OpenAI *training* only — does NOT affect ChatGPT search citation [P] |
| `ChatGPT-User` | live user-initiated fetches fail (no JS execution either) [P] |
| `ClaudeBot` / `Claude-SearchBot` / `Claude-User` | training / search / live-fetch, independently [P: Anthropic docs] |
| `PerplexityBot` | invisible in Perplexity [P] |
| `Google-Extended` | Gemini training/grounding only — **does NOT remove you from AI Overviews/AI Mode.** AIO inclusion = ordinary Search controls (`nosnippet`, `data-nosnippet`, `max-snippet`, `noindex`) [P: Google AI-features doc] |
| `Bingbot` | invisible to Bing AND weaker ChatGPT citation odds — ChatGPT search citations overlapped Bing top results ~87% [S: Seer]. Check Bing indexation + consider IndexNow. |

- Scan page HTML for `nosnippet`/`max-snippet:0` — these silently exclude from AI Overviews. [P]
- Eligibility ground truth: **AIO/AI Mode requires only indexed + snippet-eligible — no extra technical requirements exist.** [P: Google AI-features doc]
- If the goal is *blocking* AI training: robots.txt is a request, not enforcement (Perplexity documented crawling via stealth UAs; Bytespider ignores robots) — WAF-level enforcement is the real lever. [S: Cloudflare, Aug 2025]

## Pass 4 — Extraction & on-page
What engines (and LLM retrieval) can lift off the page:
- **Title/H1 alignment** — Google rewrites ~61-76% of titles; sources include H1/og:title/anchors; title↔H1 match is the strongest rewrite protection. Truncation is pixel-width, not a character law — treat ~60 chars as a heuristic, front-load what matters, never auto-fail on length. [P: title-link doc + S: Zyppy n=80,959]
- **Meta descriptions** — not a ranking factor, rewritten ~63-71% of the time. Audit as a targeted-CTR play on high-value pages only; **don't** flag sitewide completeness. [P + S: Ahrefs n=20,000]
- **Headings** — topic labels, not a weighted hierarchy (multiple H1s fine per Google). Check headings *describe their sections* and key sections open with a direct 40–75-word answer (passage-level retrieval: Google confirmed AI Mode "query fan-out" retrieves passages; RAG literature confirms chunk-level scoring). Per key page, enumerate the fan-out sub-questions (what/cost/vs/how) and check each has a self-contained section. [P: Google I/O 2025 + C]
- **GEO content levers** — the one peer-reviewed result: adding **cited sources, quotations, and statistics** raised generative-engine visibility 30–40%; keyword stuffing tested ~10% *worse* than baseline. Score key pages for attributed stats, quotable statements, and citations. [R: Aggarwal et al., KDD 2024, n=10,000 queries]
- **Keyword placement, not density** — primary concept once in title, H1, opening section; then stop counting. No density target has ever existed. [P: Mueller]
- **Images** — alt text is the top image signal (keep it descriptive, no stuffing); filenames a "very light" signal; meaningful images not as CSS backgrounds. [P: image SEO doc]
- **Internal anchors** — descriptive and concise; flag targets reached mostly by "learn more"/"click here." [P: link best practices]
- **Cannibalization, narrowly** — multiple pages ranking for one query is NOT inherently bad (Mueller, Sept 2025). Only flag: GSC shows URL flip-flopping AND the pages serve the same intent with overlapping content → consolidate. [P]

## Pass 5 — Structured data & entity
- **Living types only** (Google's search gallery, 2026): Organization, Article/BlogPosting (author + dates), Product/Review, LocalBusiness, Breadcrumb. Validate (Rich Results Test / schema.org validator); markup must match visible text. [P]
- **Organization schema on the home/about page** with `@id`, legalName, logo, `sameAs` to real profiles (LinkedIn, Wikidata/Wikipedia if they exist) — the on-site entity-disambiguation lever. [P type + C practice]
- **Dead markup ledger:** FAQ rich results fully removed (May 2026 — doc deleted); HowTo dead since 2023; sitelinks searchbox (Oct 2024) and seven more types (June 2025) retired. Existing markup is harmless; **any active effort or vendor deliverable still selling these is a finding.** [P]
- **No AI-specific schema exists.** Google: "no special schema.org markup you need to add" for AI features. [P]

## Pass 6 — Content quality & E-E-A-T
- **Run Google's own people-first rubric** (published verbatim in their docs) as an LLM rubric over top + mid-tail pages: original info/analysis? substantial completeness? clear Who (author) / How (process, incl. AI-use disclosure) / Why (for people or for rankings)? [P: helpful-content doc — the system is site-wide and inside core ranking since March 2024]
- **Information gain** — for each key page, fetch the top 5 for its primary query and compute the delta: what does this page have that they don't (original data, first-hand testing, tools, contrarian analysis)? Zero-delta summarizing content is the post-HCU loser profile. [C: patent US 10,521,479 exists but unconfirmed; S: HCU-recovery analyses — recovered pages averaged *fewer* words but more originality]
- **First-hand experience evidence** on review/how-to content: original images, stated methodology, use-over-time specifics. [P: QRG + S: post-update winner analyses]
- **Author verifiability** — bylines/bios are NOT ranking inputs (Google dropped authorship in 2014) but are rater/user trust evidence; flag "Admin" bylines and unverifiable personas on YMYL topics. [P]
- **AI content policy is quality-not-provenance** — don't run AI-detection as pass/fail. Audit for the actual violation: low-effort pages adding nothing (Lowest rater rating since Jan 2025 QRG), **scaled content abuse** patterns (template cohorts, publish-velocity spikes, keyword-permutation pages — enforcement wave May 2024 deindexed sites), **site reputation abuse** (off-purpose sections: coupons/betting/loans). [P]
- **Freshness where the query deserves it** — AI engines skew hard recent: ~65% of AI bot hits target content <1 year old; AI-cited content ~26% fresher than organic-ranked. Check visible dates + `dateModified` accuracy (flag date-bumping without substantive diff — compare Wayback). [S: Seer n=5,000+; Ahrefs n=17M citations]

## Pass 7 — Off-site & authority
- **For AI visibility, brand mentions beat backlinks:** across 75,000 brands, branded web mentions correlate 0.66 with AI-answer visibility vs 0.22 for backlinks; YouTube mentions ~0.74. Audit the mention footprint: unlinked mentions, "best X" listicle presence, YouTube coverage, Wikipedia/Wikidata entity, branded search trend. A strong-site/weak-mention profile is the classic GEO gap. [S: Ahrefs]
- **Reddit/UGC reality:** Reddit is the #1-cited domain across the AI platforms (and a top-visible site in Google since 2024). Check brand presence/sentiment in the threads that rank/get cited for its money queries; recommend *legitimate* participation only. [S: Semrush n=30M citations]
- **Backlinks, right-sized:** still real, no longer top-3 (Illyes: "very few links" needed). Check referring-domain trend vs competitors, topical relevance, anchor distribution (flag >20-30% exact-match commercial), paid-link fingerprints (link farms, "write for us" clusters — SpamBrain neutralizes or worse). [P + S]
- **DR/DA are not Google signals** — competitor benchmark only, labeled as third-party estimates. Any goal framed "raise DA to X" is a vanity-target finding. [P: Illyes/Mueller denials]

## Pass 8 — Measurement & AI-visibility baseline
End the audit with a re-measurable baseline (data, not decisions):
- **GSC:** 16-month organic trend (branded/non-branded), gen-AI report numbers, CTR-vs-expected-by-position outliers (snippet/title problems).
- **CWV:** CrUX p75 per template (field only).
- **AIO exposure:** top 50–100 GSC queries → AIO trigger rate + own-domain citation rate + competitors cited where you're not (SERP API / rank tracker). Context for expectations: users click a result on ~8% of searches with an AI summary vs ~15% without [S: Pew, July 2025, n=68,879 searches] — but AI referrals convert well above average where they land [S: Semrush clickstream].
- **Prompt panel (GEO):** 15–30 category prompts × 2–3 engines × **≥5 runs each** — LLM answers are non-deterministic; single-run "ChatGPT doesn't mention us" checks are methodologically invalid. Report mention rate + citation rate with sample sizes. [R: 2026 statistical-framework preprint + C]
- **GA4:** AI-referral channel group exists? (`chatgpt.com|perplexity|copilot|gemini` source regex — ChatGPT appends `utm_source=chatgpt.com`; counts are systematically understated.) If absent, that's a finding with a concrete fix. [P: OpenAI UTM behavior]
- **KPI table for the report:** non-branded organic clicks · organic conversions · gen-AI impressions (GSC) · AIO citation rate · prompt-panel mention rate · AI-referral sessions/conversions — each with source + re-measure cadence.

---

## 🚫 The Cargo-Cult Ledger (flagging these IS the audit working)
Practices that are dead, mythical, or platform-denied. Finding them in the site, its content briefs, or a vendor's invoice = `[Med · Med]` finding ("wasted spend, compounds"):

| Practice | Verdict | Evidence |
|---|---|---|
| **llms.txt** as a visibility lever | No platform consumes it. Mueller: like the keywords meta tag; log studies: ~0.1% of AI-bot requests touch it. Present = harmless; *recommending* it = finding | [P+S] |
| "AI schema" / markdown mirrors / ai.txt | Google: "you don't need to create new machine readable files, AI text files, markup, or Markdown… Google Search ignores them" | [P] |
| FAQ / HowTo schema for rich results | HowTo dead 2023; FAQ removed May 2026 | [P] |
| Meta keywords tag | Ignored since 2009; populated = competitive-intel leakage | [P] |
| Keyword density targets / LSI keywords | Never existed / no evidence | [P] |
| Word-count minimums | Not a factor; HCU-recovered pages averaged *fewer* words | [P+S] |
| Crawl-budget work on a small site | Google scopes it to 1M+ page sites | [P] |
| "Increase DA/DR to X" as a goal | Not Google signals | [P] |
| `priority`/`changefreq` tuning | Google ignores both | [P] |
| AMP for ranking | Requirement removed June 2021 | [P] |
| Buying links / PBNs / guest-post farms | Spam policy + SpamBrain neutralization | [P] |
| Date-bumping without substantive updates | Rater guidelines target it; Wayback-diffable | [P] |
| Blocking Google-Extended to "control AIO" | Doesn't touch AIO — wrong lever | [P] |
| Single-run AI-visibility checks | Non-deterministic engines; need ≥5 runs/prompt | [R] |
| Duplicate-content "penalty" fixes | No such penalty — consolidation problem | [P] |

## Report additions (on top of the shared shape)
- **Header** carries: playbook = seo-audit · site URL · repo · GSC/GA4 access level.
- Every finding carries its **evidence tier tag** `[P]/[S]/[R]/[C]` next to `[Severity · Cost-of-doing-nothing]`.
- **Scorecard on a full run** — grade A–D per pass (0–8), same rubric as the Codebase Audit's.
- **The KPI baseline table** (Pass 8) closes the report before the remediation-plan handoff.
- N/A rules honored: no GSC → mark GSC-dependent checks N/A-with-reason; <10k URLs → crawl-budget suppressed by design.

## Sources (the load-bearing ones)
Google: AI-features doc + AI-optimization guide (Nov 2025/June 2026) · SEO Starter Guide · helpful-content doc · structured-data gallery · CWV docs · crawl-budget doc. OpenAI/Anthropic/Perplexity bot docs. Vercel/MERJ AI-crawler study (Dec 2024). Aggarwal et al., *GEO*, KDD 2024 (arXiv:2311.09735). Ahrefs: AI-citation + 75k-brand correlation studies. Semrush: AIO studies + 30M-citation domain study. Pew Research zero-click study (July 2025). Seer Interactive recency + Bing-overlap studies. Cloudflare stealth-crawler report (Aug 2025). Full citations live with each claim above.

---
_v1, 2026-07-17. Built from a 6-lane research sweep (90 tiered findings). Evidence tiers are the contract: re-verify [S]/[C] claims ~quarterly — this field moves fast; the [P] claims cite living Google/OpenAI docs that should be re-fetched on each run. Part of Varnish Tools._
