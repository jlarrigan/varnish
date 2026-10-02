# Stack module: Expo / React Native
_Signals: `expo` or `react-native`, `app.json`/`app.config.*`, `eas.json`. Adds to the core passes by number, and defines passes 17 and 19._

- **Client-exposed prefix:** `EXPO_PUBLIC_`. And more broadly: **anything in the app bundle is public.** A key compiled into the app can be extracted from the binary, prefix or not (passes 10, 20; launch 1).
- **Several app versions are live at once.** App-store lag and OTA updates mean old builds keep calling the API for weeks. Every API or schema change must stay compatible with the oldest supported build (passes 7, 16; feature-audit Evolvability).

### 17. Local-First Device-Migration Safety — *the on-device database is a second schema you have to migrate*
> **Buckets:** delivery
- **What:** for local-first apps (an on-device SQLite replica + offline outbox). If the repo has server-side sync rules (additive-only synced tables, tombstones, last-write-wins), don't re-audit those; this covers the DEVICE side they usually miss.
- **Look for (any ship touching synced tables):** an OTA update assuming a SQLite column the installed DB doesn't have; un-synced outbox writes lost across a schema change; tombstone/`deleted_at` filtering missing on the client; no on-device `schema_version` and migration runner; no server-driven force-resync for a corrupted local DB.
- **Fix direction:** an idempotent on-device migration per synced change, outbox drained before upgrade, tombstones filtered client-side, a tested force-resync path. Pair with pass 19.
- **Tools:** `expo-sqlite` migrations / Drizzle or Kysely on-device; a `schema_version` row; an upgrade-with-non-empty-outbox test.

### 19. OTA / runtimeVersion Governance — *did a JS-only OTA assume native code old binaries don't have?*
> **Buckets:** delivery
- **What:** the dangerous default is `runtimeVersion` pinned as a literal instead of a fingerprint or appVersion policy. An OTA that needs a new native module (or a new on-device schema) pushed under the same runtimeVersion crashes on launch for clients that can't satisfy it.
- **Look for (this ship):** native-module or on-device-schema changes shipped as OTA instead of a new build + runtimeVersion bump; OTA to the wrong channel; the minimum-supported-version gate not raised when old clients must be forced off; no tested OTA rollback for production.
- **Fix direction:** native changes get a store build and runtimeVersion bump; correct channel; raise the gate when needed. Report only — never publish or roll back an update during an audit.
- **Tools (read-only):** `eas update:list`, `eas.json` channels, the `runtimeVersion` policy in app config, `expo-updates` config.

### Other passes
- **12:** `@sentry/react-native` (not the retired `sentry-expo`); source maps uploaded on EAS builds and OTA updates.
- **18:** the OTA channel is its own deploy surface — check which update production clients are actually on (read-only).
- **20:** `eas env` / EAS secrets vs `.env.example`.
- **27:** touchables without `accessibilityLabel`/`accessibilityRole`; icon-only buttons; touch targets under 44pt; `maxFontSizeMultiplier` clipping Dynamic Type; test with VoiceOver and TalkBack.
- **28:** tracking SDKs before App Tracking Transparency consent (`expo-tracking-transparency`); App Store Privacy / Play Data Safety declarations vs the actual SDK list.
- **Launch 12:** Apple requires in-app account deletion for any app with account creation.
