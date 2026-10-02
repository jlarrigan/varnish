# Stack module: Firebase
_Signals: `firebase` / `firebase-admin`, `firebase.json`, `*.rules`. Adds to the core passes by number._

- **Keys:** the Firebase web config (`apiKey`, `projectId`…) is public by design — not a finding. A service-account JSON (`"type": "service_account"`) or `firebase-admin` credentials in client code or git are High (launch 1, 8).
- **5 — client-direct is the design.** Security Rules are the API; audit them as the authorization layer.
- **10 / launch 2 — rules:** `firestore.rules`, `database.rules.json`, `storage.rules`: no `allow read, write: if true`; no test-mode rules with an expiry date (`request.time < timestamp.date(…)`) — they lock everyone out or, worse, were extended; per-user paths checked against `request.auth.uid`; writes validated (`request.resource.data`). Rules not in the repo → Couldn't-check, and recommend the Rules Playground.
- **10 / launch 7:** App Check enforced on Firestore, Storage, and callable functions; Cloud Functions (`onCall`, `onRequest`) checking `context.auth` themselves.
- **23:** missing composite indexes (`firestore.indexes.json`); reads per screen (Firestore bills per document read — a listener over a whole collection is a cost bug).
- **24 / launch 9:** scheduled Firestore exports or PITR enabled (Couldn't-check — list it).
- **Launch 6 / 12 — the bill:** Google Cloud budget alerts **alert but don't cap** spending; say so plainly.
