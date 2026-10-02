# Stack module: Vite single-page app (incl. Lovable, Bolt, v0)
_Signals: `vite` with React/Vue/Svelte and no server framework; builder scaffolding. Adds to the core passes by number._

- **The whole app runs in the browser.** There is no server to hide anything on. Every key in the bundle is public, and every rule the app "enforces" in components can be skipped by anyone with dev tools. The only real security is in the backend (database rules, edge/cloud functions). Weight passes 10 and launch checks 1–4 accordingly.
- **Client-exposed prefix:** `VITE_`. Every `VITE_*` value ships to the browser. A `VITE_` secret (service-role, Stripe secret, AI key) is High (passes 10, 20; launch 1). If `dist/` exists, grep `dist/assets/*.js` for key patterns.
- **Paywalls and roles:** "is admin" / "is Pro" checks in React state or `localStorage` are decoration (launch 4). The backend must check.
- **Builder specifics:** Lovable apps usually use Supabase client-direct and keep migrations in `supabase/migrations/` only when synced; if they're missing, RLS state is Couldn't-check — recommend Lovable's security scan plus Supabase's Security Advisor. Fix prompts should be phrased for the builder's chat, not git.
- **29:** builder-generated READMEs are usually template text; flag only if someone relies on them.
