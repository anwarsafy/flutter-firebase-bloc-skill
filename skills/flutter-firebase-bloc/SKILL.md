---
name: flutter-firebase-bloc
description: >
  Production architecture, security, and implementation skill for Flutter + Firebase + BLoC apps.
  Use when scaffolding, extending, auditing, or refactoring any Flutter app backed by Firebase
  (Auth, Firestore, Storage, Messaging, Cloud Functions, Crashlytics, Performance, Remote Config,
  App Check). Enforces Clean Architecture, offline-first writes, single-state BLoC, sealed
  AppFailure localization, GoRouter auth/ops guards, flavors, RTL l10n, strict lints, anti-hacking
  rules, and emulator testing. Do NOT use for quick single-file prototypes without persistence.
---

# Flutter + Firebase + BLoC — #1 Production Skill

> **Universal Skill:** Compatible with Claude Code, Cursor, Windsurf, GitHub Copilot, Codex CLI, Gemini CLI / Antigravity, and standalone `AGENTS.md` / `.cursorrules`.
> Load files from `references/` **on demand** for the specific domain you are implementing; copy base configs directly from `templates/`.

---

## 1. Reference Map (Progressive Disclosure)

| Need | Read This File |
|---|---|
| Domain, Data, Offline-First, Pagination, Storage, DI (`GetIt` + `Injectable`) | `references/architecture-and-di.md` |
| Full Auth (Google, Apple Nonce, OTP, Email, DeleteAccount), Security Rules, App Check, RASP, Rules Unit Tests | `references/auth-and-security.md` |
| Single-State BLoC, Sealed `AppFailure`, Static `AppLogger`, `ErrorReporter`, Theme, `AdaptiveScaffold`, EN/AR RTL `l10n` | `references/bloc-errors-l10n.md` |
| Leak-Free `FcmService` (Mobile + Web VAPID/SW), Local Notifications, Cloud Functions v2, GoRouter + Universal/App Links | `references/notifications-and-links.md` |
| Flavors (`dev`/`staging`/`prod`), `bootstrap()`, Performance Traces, Remote Config + A/B, GitHub Actions CI/CD + Symbols, Tests | `references/flavors-perf-rc-cicd.md` |
| Starter Configs & Templates | `templates/`: `analysis_options.yaml`, `l10n.yaml`, `firebase.json`, `pubspec.yaml`, `app_en.arb`, `app_ar.arb`, `firestore.indexes.json`, `.firebaserc`, `dependabot.yml`, `ExportOptions.plist` |

---

## 2. Decision Matrix (Choose Once, Never Mix)

| Decision | Default Choice | When to Use Alternative |
|---|---|---|
| State equality & `copyWith` | **Equatable** + manual `copyWith` | **Freezed** only if used consistently across ALL models & states |
| Firestore writes | **Offline-first unawaited writes** (client IDs + `batch` <= 400 ops) | **`onCall` Cloud Function** for billing, quotas, multi-user invariants, or >500-doc deletes |
| Connectivity check (`NetworkInfo`) | **Never gate** normal Firestore CRUD | **Always gate** `onCall` Functions, Storage uploads, and `deleteAccount` |
| Failure handling | **Sealed `AppFailure`** -> `exception_mapper.dart` -> `failure.localize(context)` | Never pass raw `e.toString()` or English strings into UI state |
| Local persistence | **`SharedPreferences`** ONLY for non-sensitive UI prefs (theme, locale, drafts) | **`flutter_secure_storage`** for any tokens, secrets, or sensitive local data |
| Navigation | **GoRouter** with `RouteNames` + `AuthStatus.unknown` splash guard | Never call `Navigator.of(context).push` for app routes |
| Logging & crashes | Static **`AppLogger`** -> `ErrorReporter` (`Crashlytics` on mobile, console/Sentry on web) | Never call `print()` or invoke Crashlytics directly on Web |

---

## 3. The 10 Ironclad Laws (Anti-Patterns)

1. **`domain/` is pure Dart.** NEVER import `flutter/*`, `cloud_firestore`, `firebase_auth`, or `firebase_messaging` in `domain/`.
2. **Every repository method returns `Future<Either<AppFailure, T>>` or `Stream<Either<AppFailure, T>>`.** Catch `FirebaseAuthException` before `FirebaseException` before `Object`.
3. **One state class per feature BLoC.** Use `enum <Feature>Status { initial, loading, success, failure }` + `copyWith` + `AppFailure? failure`. Failed refreshes MUST preserve existing data in state.
4. **NEVER leak `StreamSubscription`s.** Use `emit.forEach` / `emit.onEach` inside BLoC handlers. In singleton services (`FcmService`, `ConnectivityCubit`, `FeatureFlagsCubit`), store and cancel all subscriptions in `dispose()` / `close()`.
5. **Offline-first means NEVER `await` Firestore write futures** (`set`, `update`, `delete`, `batch.commit`) in normal CRUD. Fire via `_fireAndLog(...)` and listen via `snapshots(includeMetadataChanges: true)`.
6. **Transactions fail offline.** Never call `runTransaction` on the client for user actions; move transactions to Cloud Functions.
7. **Always `withConverter` for Firestore reads**, and always write `updatedAt: FieldValue.serverTimestamp()`. Handle nullable `Timestamp?` locally (`?? DateTime.now()`).
8. **Zero hardcoded colors, strings, or directional padding.** Use `Theme.of(context).colorScheme.*`, `AppLocalizations.of(context)`, and `EdgeInsetsDirectional` / `AlignmentDirectional` for RTL.
9. **Security Rules deny by default & block tampering:**
   - Never allow clients to write privileged fields (`isPro`, `role`, `plan`, `boardCount`) on `users/{uid}` — enforce `unchanged([...])` + `hasOnly([...])`; update plans/roles ONLY via Admin SDK / Custom Claims.
   - Prevent scraping (`allow get: if isSignedIn(); allow list: if false;` on `publicProfiles`).
   - Validate `keys().hasOnly(...)` and string/array `size()` on BOTH `create` and `update`.
   - Notifications are created ONLY by Cloud Functions; Storage rules MUST check `firestore.exists(...)` before `firestore.get(...)`.
10. **Zero secrets in client code or `--dart-define`.** Protection = Security Rules + App Check (`Play Integrity` / `App Attest` / `reCAPTCHA Enterprise` enforced on Auth, Firestore, Storage, and `onCall` Functions) + Hosting CSP/`X-Frame-Options` headers.

---

## 4. Canonical Project Layout

```text
project_root/
├── analysis_options.yaml               # Copy from templates/analysis_options.yaml
├── l10n.yaml                           # Copy from templates/l10n.yaml
├── firebase.json                       # Copy from templates/firebase.json
├── .firebaserc                         # Copy from templates/.firebaserc
├── .fvmrc / pubspec.yaml               # Copy starter from templates/pubspec.yaml
├── .github/
│   ├── dependabot.yml                  # Copy from templates/dependabot.yml
│   └── workflows/                      # ci.yml, deploy-web.yml, deploy-firebase.yml, release-*.yml
├── ios/ExportOptions.plist             # Copy from templates/ExportOptions.plist
├── firebase/
│   ├── firestore.rules / storage.rules
│   ├── firestore.indexes.json          # Copy from templates/firestore.indexes.json
│   ├── tests/                          # @firebase/rules-unit-testing
│   └── functions/src/                  # index.ts, notifications/, triggers/
├── web/firebase-messaging-sw.js
├── lib/
│   ├── app.dart / app_config.dart / bootstrap.dart
│   ├── main_dev.dart / main_staging.dart / main_prod.dart
│   ├── injection_container.dart
│   ├── l10n/ (app_en.arb, app_ar.arb)
│   ├── core/ (constants, errors, firebase, l10n, network, notifications, observers, performance, remote_config, router, storage, theme, usecases, utils, widgets)
│   ├── common/ (layouts, widgets shared across >=2 features)
│   └── features/ (auth, notifications, <feature>/{domain,data,presentation})
├── test/ (unit, widget, helpers)
└── integration_test/
```

---

## 5. Exact Startup Sequence (`bootstrap.dart`)

Every flavor entrypoint (`main_dev.dart`, `main_staging.dart`, `main_prod.dart`) calls `AppConfig.init(...)` then `bootstrap()`:

1. `WidgetsFlutterBinding.ensureInitialized()`
2. `await Firebase.initializeApp(options: AppConfig.instance.firebaseOptions)`
3. `await activateAppCheck()` (debug in dev; PlayIntegrity / AppAttest / ReCaptchaEnterprise in staging/prod)
4. Register `ErrorReporter` (`WebErrorReporter` on web, `CrashlyticsErrorReporter` on mobile) + `await installGlobalErrorHandlers(reporter)`
5. `configureFirestore()` (persistence enabled + 100 MB cache — MUST precede any Firestore call)
6. `await configureDependencies()` (`sl.init()` with `@preResolve` for `SharedPreferences` & `PackageInfo`)
7. `await sl<RemoteConfigService>().init()` (`activate()` cached + unawaited background `fetch()`)
8. `await sl<PerfService>().init()` (enabled only in `prod`)
9. Register `FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler)` + `setForegroundNotificationPresentationOptions(...)`
10. `Bloc.observer = AppBlocObserver()`
11. `sl<AuthBloc>().add(const AuthWatchStarted())`
12. `runApp(const App())`

---

## 6. Feature Scaffold Order (Inside-Out)

1. **`domain/`**: Entity (`Equatable`) -> Repository interface (`Either<AppFailure, T>`) -> UseCases (`UseCase` / `StreamUseCase` + typed `Params`).
2. **`data/`**: Model (null-safe `fromFirestore`, `?? DateTime.now()`, `hasPendingWrites`, `toMap()`, `toEntity()`) -> Datasource (`withConverter`, `snapshots(includeMetadataChanges: true)`, unawaited `_fireAndLog`, cursor pagination) -> RepositoryImpl (maps `FirebaseAuthException` -> `FirebaseException` -> `AppLogger.error`).
3. **`presentation/`**: Single State + enum + `copyWith` -> Sealed Events -> `@injectable` BLoC (`emit.forEach`) -> `BlocConsumer` UI (`ErrorView` only when list empty, otherwise `AppSnackBar.error`).
4. **Routing & Security**: `RouteNames` + `createAppRouter` -> `firebase/firestore.rules` + `firebase/firestore.indexes.json` -> Cloud Functions if needed.
5. **Tests**: Unit tests (UseCases + Repository) -> `blocTest` -> Widget tests (wrapped with `AppLocalizations` delegates) -> Emulator integration & Rules unit tests.

---

## 7. Mandatory Verification Gate (Run Before Finishing)

```bash
flutter gen-l10n
dart run build_runner build --delete-conflicting-outputs
dart format --output=none --set-exit-if-changed lib test
flutter analyze
flutter test
# If firebase/functions changed:
cd firebase/functions && npm run build
# If rules or composite queries changed:
firebase emulators:exec --only auth,firestore "flutter test integration_test/"
```
