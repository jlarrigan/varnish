# Stack module: Payments (Stripe, RevenueCat, Lemon Squeezy, Paddle)
_Signals: `stripe`, `@stripe/*`, `react-native-purchases`, `@lemonsqueezy/*`, `paddle`. Adds to the core passes by number._

- **Keys:** Stripe `pk_live_`/`pk_test_` are publishable (not a finding). `sk_live_`/`sk_test_` and restricted keys (`rk_`) are secrets — High anywhere client-side or in git (launch 1, 8). Webhook signing secrets (`whsec_`) are secrets too.
- **15 / launch 5 — webhooks:**
  - Signature verified with the provider's helper (`stripe.webhooks.constructEvent`) **against the raw request body** — a JSON body parser that runs first breaks verification, and code that "fixes" that by skipping verification is the classic hole.
  - The event id is recorded once (a unique constraint) so redelivery is a no-op. Providers WILL redeliver.
  - Cancellations, refunds, failed payments, and subscription updates are handled, not only `checkout.session.completed`.
- **Launch 4 / 5 — entitlement truth:** access is granted from the webhook (or a server-side check with the provider), never from a success redirect (`/success?paid=true`) or client state. One source of truth for "is Pro" — the provider's entitlement vs a cached column vs a JWT claim must not disagree (Architecture Review data-model pass).
- **RevenueCat:** entitlements checked through the SDK/REST API server-side for anything valuable; the webhook authorization header verified.
- **Test vs live:** live keys in development `.env` files; test keys in production config (Couldn't-check — list it).
- **12 / 14:** payment-path errors captured and alerted; a failed webhook is a revenue bug, not a log line.
