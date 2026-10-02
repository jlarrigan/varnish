# Stack modules

_The core playbooks are written stack-neutral. A stack module adds the exact files, settings, tools, and traps for one platform, keyed by the core pass it sharpens. Load only the modules whose signals you find, so a web app never reads about OTA updates and a mobile app never reads about server actions. Part of Varnish._

## How to use
1. **Detect** (a minute): read `package.json` (and `app.json`, `pubspec.yaml`, `requirements.txt` if present), config files, and top-level folders.
2. **Load** every module whose signal matches. Several usually do (a typical vibe-coded app is `nextjs` + `supabase` + `payments` + `ai-llm` + `vercel-serverless`).
3. **Apply** each module's sections alongside the core pass with the same number (or, in `launch`, the check with the same name). A module section adds to the core pass; it never replaces it.
4. **Record** the loaded modules in the report header (`stack: nextjs, supabase, payments, ai-llm`). If nothing matches, say so and run the core passes on principles alone.

## Signals → module

| Module | Load when you see |
|---|---|
| `nextjs.md` | `next` dependency, `next.config.*`, `app/` or `pages/` router |
| `vite-spa.md` | `vite` dependency with React/Vue/Svelte and no server framework; Lovable, Bolt, or v0 scaffolding |
| `expo-react-native.md` | `expo` or `react-native` dependency, `app.json`/`app.config.*`, `eas.json` |
| `node-api.md` | `hono`, `express`, `fastify`, `koa`, or `@nestjs/*` server code |
| `vercel-serverless.md` | `vercel.json`, `.vercel/`, `@vercel/*`, `netlify.toml`, `netlify/functions/`, `serverless.yml`, AWS Lambda handlers. (Next.js `app/api/` routes alone don't count — that's `nextjs.md`. Load this when there's evidence of where it deploys.) |
| `supabase.md` | `@supabase/*` dependency, a `supabase/` folder, `SUPABASE_` env vars |
| `firebase.md` | `firebase` / `firebase-admin` dependency, `firebase.json`, `*.rules` files |
| `payments.md` | `stripe`, `@stripe/*`, `react-native-purchases` (RevenueCat), `@lemonsqueezy/*`, `paddle` |
| `ai-llm.md` | `openai`, `@anthropic-ai/sdk`, `@google/genai`, `ai` (Vercel AI SDK), `langchain`, `@mistralai/*`, any `*_API_KEY` for a model provider |

Missing a stack you hit often? Add a module: same shape, sections keyed by pass number, signals added to this table.
