<div align="center">

**English** · [العربية](README.ar.md)

# Flutter Firebase BLoC

### Ship production Flutter with agents, from the first screen down to Firestore rules.

[![Skill check](https://img.shields.io/github/actions/workflow/status/anwarsafy/flutter-firebase-bloc-skill/validate-skill.yml?style=flat-square&label=skill%20check)](https://github.com/anwarsafy/flutter-firebase-bloc-skill/actions)
[![License: MIT](https://img.shields.io/badge/license-MIT-f4c430?style=flat-square)](LICENSE)
[![Agent skill](https://img.shields.io/badge/agent%20skill-standard-5c4ee5?style=flat-square)](https://skills.sh)

**[Website](https://anwarsafy.github.io/flutter-firebase-bloc-skill/)** · **[Blueprint](https://anwarsafy.github.io/flutter-firebase-bloc-skill/Flutter_Firebase_BLoC_Blueprint_ULTIMATE.html)** · **[Install guide](skills/flutter-firebase-bloc/INSTALL.md)**

<code>npx skills add anwarsafy/flutter-firebase-bloc-skill</code>

</div>

> [!IMPORTANT]
> Install from [anwarsafy/flutter-firebase-bloc-skill](https://github.com/anwarsafy/flutter-firebase-bloc-skill) or the [website](https://anwarsafy.github.io/flutter-firebase-bloc-skill/). The skill makes Cursor, Claude Code, Copilot, and the rest hold one bar: Clean Architecture, single-state BLoC, offline-first Firestore, hardened rules, and CI.

---

## ⚡ One-Command Install (Works Everywhere)

Install into **Claude Code, Cursor, Windsurf, GitHub Copilot, Codex CLI, Gemini CLI, Cline, Roo Code**, and 35+ AI coding tools in seconds:

```bash
npx skills add anwarsafy/flutter-firebase-bloc-skill
```

<details>
<summary><strong>Alternative Install Methods (Claude Code Marketplace, Manual Copy, Chat UIs)</strong></summary>

### Claude Code Plugin Marketplace
```text
/plugin marketplace add anwarsafy/flutter-firebase-bloc-skill
/plugin install flutter-firebase-bloc@flutter-firebase-bloc-marketplace
```

### Git Clone (Global Install)
```bash
# Claude Code
git clone https://github.com/anwarsafy/flutter-firebase-bloc-skill.git /tmp/ffb-skill && \
  mkdir -p ~/.claude/skills && cp -r /tmp/ffb-skill/skills/flutter-firebase-bloc ~/.claude/skills/ && rm -rf /tmp/ffb-skill

# Codex CLI
git clone https://github.com/anwarsafy/flutter-firebase-bloc-skill.git /tmp/ffb-skill && \
  mkdir -p ~/.codex/skills && cp -r /tmp/ffb-skill/skills/flutter-firebase-bloc ~/.codex/skills/ && rm -rf /tmp/ffb-skill

# Gemini CLI / Antigravity
git clone https://github.com/anwarsafy/flutter-firebase-bloc-skill.git /tmp/ffb-skill && \
  mkdir -p ~/.gemini/skills && cp -r /tmp/ffb-skill/skills/flutter-firebase-bloc ~/.gemini/skills/ && rm -rf /tmp/ffb-skill
```

### ChatGPT / Claude.ai / Gemini Web
Upload `skills/flutter-firebase-bloc/SKILL.md` and `skills/flutter-firebase-bloc/references/*.md` as Knowledge files and paste `SKILL.md` into Custom Instructions.

</details>

---

## 🚨 Why Most AI-Generated Flutter + Firebase Code Fails in Production

| Problem Without This Skill | What This Skill Enforces Automatically |
|---|---|
| `await firestore.set(...)` hangs forever offline | **True Offline-First:** Unawaited writes via `_fireAndLog` + `snapshots(includeMetadataChanges: true)` + `hasPendingWrites` sync UI |
| Clients can self-upgrade `isPro: true` / `role: "admin"` / any new field | **Anti-Hacking Rules:** Allow-list of client fields on `users/{uid}` (`hasOnly` on create, `affectedKeys().hasOnly` on update), enforced server timestamps, `allow list: if false` on `publicProfiles` — every rule backed by an emulator test |
| Token/Secret leakage & replay attacks | **Defense in Depth:** `App Check` (`Play Integrity` / `App Attest` / `reCAPTCHA Enterprise`), `flutter_secure_storage`, and Hosting security headers |
| Login screen flashes for 500ms on app cold start | **`AuthStatus.unknown` Splash Guard:** Clean `GoRouter` redirect priority (`Ops` → `Unknown` → `Auth` → `Home`) |
| `FcmService` leaks listeners & crashes on Web | **Leak-Free Messaging:** Cancelled `StreamSubscription`s on `dispose()`, Web VAPID + Service Worker, `NotificationNavigator` payload safety |
| BLoC stuck on a spinner after a Firestore error, or a silent rollback of an offline write | **Mapped stream errors:** `catchFailures()` on every repository stream, `restartable()` watchers, and `WriteErrorBus` for server-rejected offline writes |
| Raw English `e.toString()` shown to users | **Sealed `AppFailure` + EN/AR `l10n`:** Maps `FirebaseAuthException` → `FirebaseException` → localized ARB strings with full RTL layout support |
| Crashlytics reports unreadable obfuscated stack traces | **Production CI/CD:** Automated `--obfuscate --split-debug-info` + `firebase crashlytics:symbols:upload` + keyless WIF deploys |

---

## 📦 What's Inside This Skill

### 1. Core Manifest & Progressive Disclosure (`SKILL.md`)
- **Decision Matrix:** Enforces one clear choice per concern (`Equatable` vs `Freezed`, offline writes vs `onCall` Functions, `SharedPreferences` vs `flutter_secure_storage`).
- **The 10 Laws:** Zero Firebase/Flutter imports in `domain/`, single-state BLoCs (`enum status` + `copyWith`), zero `print()`, zero hardcoded colors/strings/directional padding.
- **Exact `bootstrap()` Startup Sequence** & **Mandatory Verification Gate** (`gen-l10n` → `build_runner` → `dart format` → `flutter analyze` → `flutter test` → Emulator tests).

### 2. Full Unabridged Code References (`skills/flutter-firebase-bloc/references/`)
| File | Coverage |
|---|---|
| [`architecture-and-di.md`](skills/flutter-firebase-bloc/references/architecture-and-di.md) | `pubspec.yaml`, `UseCase`/`StreamUseCase`, `GetIt` + `Injectable` (`@preResolve`), Firestore `100 MB` cache, `ConnectivityCubit`, `OfflineBanner`, null-safe `Model`, Cursor Pagination, `StorageService`, `DraftStore` |
| [`auth-and-security.md`](skills/flutter-firebase-bloc/references/auth-and-security.md) | Google (`google_sign_in` 7.x), Apple (SHA-256 Nonce), Phone OTP, Email, `deleteAccount` + server-side data erasure (App Store 5.1.1(v)), `AuthBloc`, `App Check`, hardened `firestore.rules` & `storage.rules`, `@firebase/rules-unit-testing`, Fintech RASP |
| [`bloc-errors-l10n.md`](skills/flutter-firebase-bloc/references/bloc-errors-l10n.md) | Single-state `BoardsBloc`, `AppBlocObserver`, 9 sealed `AppFailure` classes, static `AppLogger`, `ErrorReporter` (Mobile + Web), `installGlobalErrorHandlers`, `AppTheme`, `AdaptiveScaffold`, EN/AR RTL `l10n` |
| [`notifications-and-links.md`](skills/flutter-firebase-bloc/references/notifications-and-links.md) | `AndroidManifest`, Notification Channels, `LocalNotificationService`, `fcm_background.dart`, `App` widget, leak-free `FcmService`, `web/firebase-messaging-sw.js`, Cloud Functions v2, `GoRouter` + Deep Links |
| [`flavors-perf-rc-cicd.md`](skills/flutter-firebase-bloc/references/flavors-perf-rc-cicd.md) | `dev`/`staging`/`prod` Flavors, Xcode 6-step setup, `flutterfire configure`, `PerfService` + `PerfInterceptor`, `RemoteConfigService` + A/B + KSA PDPL consent, 5 GitHub Actions workflows + Crashlytics symbols, Unit/BLoC/Widget/Integration tests |

### 3. 10 Drop-In Templates (`skills/flutter-firebase-bloc/templates/`)
Copy-paste-ready starter files: `pubspec.yaml`, `analysis_options.yaml`, `l10n.yaml`, `app_en.arb`, `app_ar.arb`, `firebase.json` (with CSP, HSTS and other Hosting security headers), `firestore.indexes.json`, `.firebaserc`, `dependabot.yml`, and `ExportOptions.plist`.

### 4. Interactive Visual Blueprint (`docs/`)
[Open the blueprint](https://anwarsafy.github.io/flutter-firebase-bloc-skill/) — a searchable visual reference with one-click copy buttons, generated from the skill's Markdown by `scripts/build_blueprint.py` (CI fails if it drifts).

See [CHANGELOG.md](CHANGELOG.md) for what changed in each release.

---

## 💬 Example Prompts to Try Right Away

Once installed, prompt your AI assistant naturally:

- *"Use the flutter-firebase-bloc skill to scaffold a complete `orders` feature (domain, offline-first datasource, single-state BLoC, GoRouter route, EN/AR ARB keys, Firestore rules & indexes, and blocTests)."*
- *"Audit my `firebase/firestore.rules` and `storage.rules` against the skill's anti-hacking checklist and add `@firebase/rules-unit-testing` coverage."*
- *"Refactor my `main.dart` and `FcmService` to follow the skill's `bootstrap()`, `AuthStatus.unknown` splash guard, and zero-leak subscription pattern."*

---

## 🤝 Contributing & Support

Found a security edge case or an improvement? Pull requests and issues are very welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

If this skill saved you hours of debugging or prevented a security bug in production, **please drop a ⭐ Star on the repo** so more Flutter engineers can discover it!

---

<div align="center">
Released under the <a href="LICENSE">MIT License</a>.
</div>
