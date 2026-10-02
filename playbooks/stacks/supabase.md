# Stack module: Supabase
_Signals: `@supabase/*`, a `supabase/` folder, `SUPABASE_` env vars. Adds to the core passes by number._

**Live checks need permission.** The Supabase MCP tools reach a real project. Ask which project before the first call, and say so if it looks like production. Read-only only: `list_tables`, `list_migrations`, `get_advisors`, `query_logs`, and `SELECT`-only `execute_sql`. Never `apply_migration`, never `EXPLAIN ANALYZE`.

- **Keys:** the anon key and `sb_publishable_…` keys are public by design (not a finding). The `service_role` key and `sb_secret_…` keys bypass all RLS — in client code, behind a public prefix, or in git history they're High (pass 10; launch 1, 8).
- **5 — client-direct is the design.** With the browser talking to the database, **RLS policies are the API.** Audit them as the authorization layer.
- **10 / launch 2 — RLS:** every table in an exposed schema (usually `public`) has RLS enabled (`alter table … enable row level security` in `supabase/migrations/`); no `USING (true)` on private data; INSERT/UPDATE policies have `WITH CHECK`; per-user data restricted to `auth.uid() = user_id`, not just "authenticated"; no `grant all … to anon`; no `SECURITY DEFINER` functions executable by `anon`/`authenticated` that skip checks; views with `security_invoker` off bypass RLS; policies trusting `user_metadata` (users can edit it — use `app_metadata`). Storage buckets public only when the files are meant to be.
- **10 — the advisor:** with permission, `get_advisors` (type `security`) catches RLS-off tables and risky functions directly. If migrations aren't in the repo, this is the only way to see real RLS state. If the user says no (or there's no Supabase connection), list "RLS state in the live project" under Couldn't-check.
- **10 / launch 3:** Edge Functions with `verify_jwt = false` in `supabase/config.toml` and no auth check of their own.
- **16:** forward-only migrations; schema changed in the dashboard instead of a migration file (compare `list_migrations` with `supabase/migrations/`); `squawk` on new SQL.
- **23:** `get_advisors` (type `performance`) for unindexed foreign keys; RLS predicates using bare `auth.uid()` (re-evaluated per row) instead of `(select auth.uid())`; indexes on policy columns.
- **24 / launch 9:** point-in-time recovery is a paid add-on — Couldn't-check from the repo unless confirmed; list it. Branching or a separate project for dev; seed scripts pointing at the production URL.
- **13 / 25:** `query_logs` (with permission) for auth, API, and Postgres errors.
