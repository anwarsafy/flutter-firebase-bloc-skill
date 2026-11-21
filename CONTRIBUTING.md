# Contributing to Flutter + Firebase + BLoC Production Skill

Thank you for helping make this the highest-quality Flutter + Firebase AI coding skill!

## Guidelines

1. **Keep Code Unabridged & Compilable:** Do not replace working implementations in `skills/flutter-firebase-bloc/references/` with pseudocode or `// TODO` stubs.
2. **Maintain Internal Consistency:** Every symbol referenced across files (e.g., `AppLogger`, `sl.init()`, `AuthStatus.unknown`, `NotificationNavigator.navigateFromPayload`, `AppLocalizations`) must match exact signatures across all references and templates.
3. **Security First:** Any Firestore, Storage, Auth, or Cloud Functions addition must include server-side authorization checks, input validation (`hasOnly` / `size()`), and a `@firebase/rules-unit-testing` test that proves the tampering attempt fails (cover `update`, not only `create`).
4. **Update Templates & Validation:** If you add a new localization key or config file, update `skills/flutter-firebase-bloc/templates/` as well and verify the `.github/workflows/validate-skill.yml` checks pass.

## Submitting a Pull Request

1. Fork the repository and create a branch from `main`.
2. Make your edits inside `skills/flutter-firebase-bloc/`. Never edit `docs/` by hand: regenerate it with `pip install markdown pygments && python3 scripts/build_blueprint.py`.
3. Pin package versions you have verified on pub.dev, add a `CHANGELOG.md` entry, and bump `version` in both `.claude-plugin/*.json` files for releases.
4. Open a PR describing the real-world production problem or security edge case solved.
