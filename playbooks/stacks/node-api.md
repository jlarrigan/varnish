# Stack module: Node API server (Hono, Express, Fastify, Koa, Nest)
_Signals: server framework dependency and route files. Adds to the core passes by number._

- **8 / 12:** one global error handler — Hono `app.onError`, Express error middleware (4-arg `(err, req, res, next)` registered last), Fastify `setErrorHandler`, a Nest exception filter. Per-route try/catch on a few routes is the classic gap. A request-scoped structured logger (route, request id, user) as middleware.
- **10 / launch 3:** list every route from the router files and check each for auth middleware and an ownership check. Routes registered before the auth middleware, or on a separate router without it, are the usual misses. CORS set to `*` with credentials, or reflecting any origin.
- **10 / launch 7:** a rate limiter on auth and expensive routes (`hono-rate-limiter`, `express-rate-limit`, `@fastify/rate-limit`), backed by a shared store if there's more than one instance.
- **7:** request validation at the boundary (`zod` / `@hono/zod-validator`, `celebrate`, Fastify schemas) and one consistent error envelope.
- **15:** body parsers that consume the raw body before webhook signature verification (Express `express.json()` ahead of the Stripe route is the classic).
