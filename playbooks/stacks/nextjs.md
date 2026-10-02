# Stack module: Next.js
_Signals: `next` dependency, `next.config.*`. Adds to the core passes by number._

- **Client-exposed prefix:** `NEXT_PUBLIC_`. Anything behind it is shipped to every browser (passes 10, 20; launch check 1). If `.next/` exists, grep `.next/static` for key patterns.
- **10 / launch 3 — who can call this:**
  - Every `app/**/route.ts` handler and every `pages/api/**` file is a public endpoint. Check each for an auth check and an ownership check.
  - **Server Actions (`"use server"`) are public POST endpoints** even if no page renders the form. Each one needs its own auth check.
  - **Auth enforced only in middleware is not enough.** Middleware (`middleware.ts`, renamed `proxy.ts` in Next 16) has been bypassed before (CVE-2025-29927). Check auth again where the data is read.
  - With Supabase: trust `supabase.auth.getUser()` / `getClaims()` on the server, never `getSession()` (it reads the cookie without verifying it).
- **3 — framework CVE currency:** Next.js and React Server Components have had critical advisories (the Dec-2025 RSC remote-code-execution bug, CVE-2025-55182). Compare the installed `next`/`react` versions against current advisories; an unpatched major is High.
- **10 / launch 11 — injection:** `dangerouslySetInnerHTML` with user or AI content; markdown rendered to HTML without sanitizing.
- **12:** `@sentry/nextjs` (or equivalent) wired for server, edge, and client; `instrumentation.ts` present; global error boundary (`app/global-error.tsx`).
- **23:** route segment config (`export const dynamic`, `revalidate`) forcing everything dynamic; `@next/bundle-analyzer`; large client components that could be server components.
- **27:** `next/image` without meaningful `alt`; `<Link>` wrapping non-text with no accessible name.
