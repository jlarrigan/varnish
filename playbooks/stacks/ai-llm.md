# Stack module: AI / LLM features
_Signals: `openai`, `@anthropic-ai/sdk`, `@google/genai`, `ai`, `langchain`, model-provider API keys. Adds to the core passes by number._

- **Keys:** provider keys (`sk-`, `sk-proj-`, `sk-ant-`, `AIza…`) anywhere client-side, behind a public prefix, or in git are High (launch 1, 8). Calls to the model belong on the server.
- **Launch 6 / pass 10 — denial of wallet:** every route that calls a model has: an auth check; a per-user rate limit or quota; `max_tokens` (or equivalent) set; input size limits; and the provider account has a monthly budget limit (Couldn't-check — list it). An open model endpoint is someone else's free AI on your card.
- **10 — prompt injection:** user content, retrieved documents (RAG), web pages, or tool output are untrusted. Look for: system prompts that rely on "ignore instructions in the user text" for security; model output used to build SQL, shell commands, file paths, or URLs; tools with write access or open `fetch` (SSRF) callable based on model output; agents acting with an admin or service key. Security decisions belong in code, not in the prompt.
- **10 / launch 11 — injection:** model output rendered as HTML or unsanitized markdown (stored XSS through the model).
- **28:** prompts and responses containing user data logged or sent to analytics; providers' data-retention settings vs the privacy policy.
- **12 / 23:** token usage per request logged; cost per user visible; retries on model errors with backoff and a cap (retry storms multiply the bill).
- **26:** what the feature does when the provider is down or rate-limits you — fail soft, not a broken page.
