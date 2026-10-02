# Stack module: Vercel / serverless (also Netlify, AWS Lambda, edge functions)
_Signals: `vercel.json`, `.vercel/`, `netlify.toml`, function folders, Lambda handlers. Adds to the core passes by number._

- **Stateless and retried.** Each invocation may be a fresh instance; module-level variables don't persist reliably, and platforms retry. Module-global mutable state used as a cache or lock is a finding (pass 15).
- **Timeouts come from config, not a constant.** Read the actual limit from `vercel.json` / route config (`maxDuration`) or the platform default; with Vercel Fluid compute the defaults are far higher than the old 10–60s. Every external call needs a timeout well under that limit (passes 12, 15, 23).
- **12 / 23 — logs and cold starts:** structured logs are the only backend trace you get; check function duration and cold-start cost (heavy top-level imports, eager client construction).
- **14:** cron jobs (`vercel.json` `crons`) with no failure notification.
- **18 — deploy truth:** `vercel ls` / the Vercel MCP `list_deployments` (read-only, with permission) to confirm production serves the audited commit. Never `vercel promote` or roll back during an audit.
- **20:** `vercel env ls` (read-only, with permission) — preview vs production parity; server-only vars accidentally exposed via a framework prefix.
- **Launch 6 / 12 — the bill:** Spend Management configured with a hard limit (Couldn't-check from the repo; list it). Functions that call paid APIs with no rate limit.
