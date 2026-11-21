# Changelog

## 1.1.0 — 2026-11-21

### Fixed (critical)
- **Sign-out could hang:** `watchAuthState` used `asyncExpand`, which never processes the next auth event while the profile `snapshots()` stream is open. Now uses `rxdart` `switchMap` and falls back to Auth data if the profile read fails.
- **Firestore stream errors escaped the repository:** `try/catch` around `yield*` doesn't catch stream errors, so BLoCs stayed in `loading`. New `catchFailures()` stream extension + `mapException()`; every `emit.forEach` has an `onError`.
- **Duplicate listeners on refresh:** `BoardsRefreshed` called the watch handler directly. Watchers now use `restartable()` (`bloc_concurrency`) and refresh re-dispatches the event.
- **`users/{uid}` rules allowed arbitrary fields** (e.g. `plan`). Replaced the deny-list with an allow-list (`hasOnly` on create, `affectedKeys().hasOnly` on update) plus type/size validation.
- **Account deletion left server data behind:** added `cleanupDeletedUser` (`onUserDeleted`) Cloud Function that erases owned boards, memberships, notifications, profiles and Storage files.
- **Board deletion ran on the client** (orphaned cards offline). Now uses the `deleteBoardRecursive` callable, gated on connectivity; client `delete` is denied by rules.

### Fixed (medium)
- Server-rejected offline writes now reach the user through `WriteErrorBus` and a localized SnackBar.
- Deep links survive splash/login via a validated `?from=` parameter (no open redirects); users in `loading`/`failure` states are no longer bounced to welcome.
- Sign-out no longer calls `terminate()` on the shared Firestore instance; the cache wipe runs on the next cold start.
- `ValidationFailure` carries a code (localized), `FriendlyErrorWidget` uses theme colors, `firebase.json` adds CSP, HSTS and Permissions-Policy.
- PR web previews build the staging flavor and deploy to the staging project; fork PRs are skipped.
- Firestore rules enforce server timestamps; FCM topic fan-out removed (any client can subscribe to any topic).

### Changed
- All packages updated to versions verified on pub.dev (Nov 2026): `firebase_core` 5, `firebase_auth` 7, `cloud_firestore` 7, `google_sign_in` 7 (new `authenticate()` API), `firebase_app_check` 0.5 (provider objects), `go_router` 18, `flutter_local_notifications` 23 (named parameters), `injectable` 3, `get_it` 9. Minimum Dart 3.12 / Flutter 3.44.
- `dartz` → `fpdart`; added `rxdart`, `bloc_concurrency`, `cloud_functions`.
- Cloud Functions use `firebase-functions` 7 / `firebase-admin` 14 modular imports; web service worker uses Firebase JS SDK 13.4.0.
- Security rules test suite expanded from 4 to 23 assertions (create and update escalation, ownership hijacking, anti-scraping, forged notifications) and runs in CI.
- Interactive blueprint (`docs/`) is now generated from the skill's Markdown, so it can't drift from the code.
- Removed overclaims ("#1", "zero hallucinations") from the docs.

## 1.0.0 — 2026-11-21
- Initial release.
