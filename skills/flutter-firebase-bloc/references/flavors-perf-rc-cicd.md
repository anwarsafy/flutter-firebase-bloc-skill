# Flavors, Performance, Remote Config, CI/CD & Testing (Complete Code)

## 1. Flavors, `AppConfig` & `bootstrap.dart`
```dart
// lib/app_config.dart
enum AppFlavor { dev, staging, prod }

class AppConfig {
  const AppConfig._(this.flavor, this.appName, this.firebaseOptions, this.webVapidKey, this.recaptchaSiteKey,
      [this.googleServerClientId]);
  final AppFlavor flavor;
  final String appName, webVapidKey, recaptchaSiteKey; // public identifiers, NOT secrets
  final FirebaseOptions firebaseOptions;
  /// Web OAuth client ID for google_sign_in 7.x. Null on Android = read from google-services.json.
  final String? googleServerClientId;

  static late final AppConfig instance;

  static void init(AppFlavor flavor) => instance = switch (flavor) {
        AppFlavor.dev => AppConfig._(flavor, 'MyApp DEV', dev.DefaultFirebaseOptions.currentPlatform, 'DEV_VAPID_KEY', 'DEV_RECAPTCHA_SITE_KEY'),
        AppFlavor.staging => AppConfig._(flavor, 'MyApp STG', staging.DefaultFirebaseOptions.currentPlatform, 'STG_VAPID_KEY', 'STG_RECAPTCHA_SITE_KEY'),
        AppFlavor.prod => AppConfig._(flavor, 'MyApp', prod.DefaultFirebaseOptions.currentPlatform, 'PROD_VAPID_KEY', 'PROD_RECAPTCHA_SITE_KEY'),
      };

  bool get isDev => flavor == AppFlavor.dev;
  bool get isStaging => flavor == AppFlavor.staging;
  bool get isProd => flavor == AppFlavor.prod;
}

// lib/main_dev.dart / main_staging.dart / main_prod.dart
void main() { AppConfig.init(AppFlavor.dev); unawaited(bootstrap()); }

// lib/bootstrap.dart
Future<void> bootstrap() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: AppConfig.instance.firebaseOptions);
  await activateAppCheck();

  final ErrorReporter reporter = kIsWeb ? WebErrorReporter() : CrashlyticsErrorReporter();
  sl.registerSingleton<ErrorReporter>(reporter);
  await installGlobalErrorHandlers(reporter);

  await configureFirestore(); // runs a pending sign-out cache wipe, then sets persistence
  await configureDependencies();
  await sl<RemoteConfigService>().init();
  await sl<PerfService>().init();

  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  await FirebaseMessaging.instance.setForegroundNotificationPresentationOptions(alert: true, badge: true, sound: true);

  Bloc.observer = AppBlocObserver();
  sl<AuthBloc>().add(const AuthWatchStarted());
  runApp(const App());
}
```

```kotlin
// android/app/build.gradle.kts (flavors + CI release signing)
import java.util.Properties

val keystoreProps = Properties().apply {
    val f = rootProject.file("key.properties")
    if (f.exists()) f.inputStream().use { load(it) }
}

android {
    signingConfigs {
        create("release") {
            if (keystoreProps.isNotEmpty()) {
                storeFile = file(keystoreProps.getProperty("storeFile"))
                storePassword = keystoreProps.getProperty("storePassword")
                keyAlias = keystoreProps.getProperty("keyAlias")
                keyPassword = keystoreProps.getProperty("keyPassword")
            }
        }
    }
    buildTypes {
        release {
            signingConfig = if (keystoreProps.isNotEmpty()) signingConfigs.getByName("release") else signingConfigs.getByName("debug")
        }
    }
    flavorDimensions += "env"
    productFlavors {
        create("dev") { dimension = "env"; applicationIdSuffix = ".dev"; versionNameSuffix = "-dev"; resValue("string", "app_name", "MyApp DEV") }
        create("staging") { dimension = "env"; applicationIdSuffix = ".stg"; versionNameSuffix = "-stg"; resValue("string", "app_name", "MyApp STG") }
        create("prod") { dimension = "env"; resValue("string", "app_name", "MyApp") }
    }
}
```

### iOS Xcode Setup (6 Steps)
| Step | Action |
|---|---|
| 1. Build configurations | Duplicate Debug/Release/Profile as `Debug-dev`, `Release-dev`, `Profile-dev` (same for `-staging`, `-prod`). Flutter requires the `-flavor` suffix. |
| 2. Schemes | Create shared schemes named `dev`, `staging`, `prod` pointing to their matching configurations. |
| 3. Bundle ID | Set `PRODUCT_BUNDLE_IDENTIFIER` per config (`com.example.myapp.dev`, `.stg`, `com.example.myapp`). |
| 4. Display name | Add `FLAVOR_APP_NAME` user-defined setting and set `CFBundleDisplayName = $(FLAVOR_APP_NAME)` in `Info.plist`. |
| 5. `GoogleService-Info.plist` | Let `flutterfire configure --ios-build-config` install the per-flavor copy script. |
| 6. Push (APNs) | Upload an APNs `.p8` key to each Firebase project (`dev`, `staging`, `prod`). |

### `flutterfire configure`, Launcher Icons & `.vscode/launch.json`
```bash
# Run once per flavor & Xcode build config (repeat for Release-dev, Profile-dev, staging, prod)
flutterfire configure \
  --project=your-app-dev \
  --out=lib/firebase_options_dev.dart \
  --platforms=android,ios,web \
  --android-package-name=com.example.myapp.dev \
  --android-out=android/app/src/dev/google-services.json \
  --ios-bundle-id=com.example.myapp.dev \
  --ios-build-config=Debug-dev \
  --ios-out=ios/flavors/dev/GoogleService-Info.plist

# Run / Release commands
flutter run --flavor dev -t lib/main_dev.dart
flutter run -d chrome -t lib/main_dev.dart # Web uses -t without --flavor
flutter build appbundle --flavor prod -t lib/main_prod.dart --obfuscate --split-debug-info=build/symbols
flutter build ipa --flavor prod -t lib/main_prod.dart --obfuscate --split-debug-info=build/symbols
flutter build web -t lib/main_prod.dart --release
```

```yaml
# flutter_launcher_icons-dev.yaml (repeat for -staging.yaml and -prod.yaml)
flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/icons/icon_dev.png"
# Generate all: dart run flutter_launcher_icons
```

```json
// .vscode/launch.json
{
  "configurations": [
    { "name": "dev",     "request": "launch", "type": "dart", "program": "lib/main_dev.dart",     "args": ["--flavor", "dev"] },
    { "name": "staging", "request": "launch", "type": "dart", "program": "lib/main_staging.dart", "args": ["--flavor", "staging"] },
    { "name": "prod",    "request": "launch", "type": "dart", "program": "lib/main_prod.dart",    "args": ["--flavor", "prod"] },
    { "name": "web dev", "request": "launch", "type": "dart", "program": "lib/main_dev.dart",     "deviceId": "chrome" }
  ]
}
```

## 2. Performance (`PerfService` + `PerfInterceptor`)
```dart
// lib/core/performance/perf_service.dart
@lazySingleton
class PerfService {
  PerfService(this._perf);
  final FirebasePerformance _perf;

  Future<void> init() => _perf.setPerformanceCollectionEnabled(AppConfig.instance.isProd);

  Future<T> trace<T>(String name, Future<T> Function() op, {Map<String, String> attributes = const {}}) async {
    final t = _perf.newTrace(name);
    attributes.forEach(t.putAttribute);
    await t.start();
    try {
      final result = await op();
      t.putAttribute('result', 'ok');
      return result;
    } catch (_) {
      t.putAttribute('result', 'error');
      rethrow;
    } finally {
      await t.stop();
    }
  }

  Trace start(String name) => _perf.newTrace(name)..start();
}

// lib/core/network/perf_interceptor.dart
class PerfInterceptor extends Interceptor {
  final _metrics = <RequestOptions, HttpMetric>{};

  @override
  Future<void> onRequest(RequestOptions o, RequestInterceptorHandler h) async {
    if (!kIsWeb) {
      final url = '${o.uri.origin}${o.uri.path}'; // never include query string / PII
      final m = FirebasePerformance.instance.newHttpMetric(url, _method(o.method));
      _metrics[o] = m;
      await m.start();
    }
    h.next(o);
  }

  @override
  Future<void> onResponse(Response<dynamic> r, ResponseInterceptorHandler h) async {
    await _finish(r.requestOptions, r.statusCode, r.headers);
    h.next(r);
  }

  @override
  Future<void> onError(DioException e, ErrorInterceptorHandler h) async {
    await _finish(e.requestOptions, e.response?.statusCode, e.response?.headers);
    h.next(e);
  }

  Future<void> _finish(RequestOptions o, int? code, Headers? headers) async {
    final m = _metrics.remove(o);
    if (m == null) return;
    m.httpResponseCode = code;
    m.responseContentType = headers?.value(Headers.contentTypeHeader);
    final len = headers?.value(Headers.contentLengthHeader);
    if (len != null) m.responsePayloadSize = int.tryParse(len);
    await m.stop();
  }

  HttpMethod _method(String m) => HttpMethod.values.firstWhere(
        (v) => v.name.toUpperCase() == m.toUpperCase(),
        orElse: () => HttpMethod.Get,
      );
}
```

## 3. Remote Config (`RcKeys`, `RemoteConfigService`, `FeatureFlagsCubit`, `VersionGate`)
```dart
// lib/core/remote_config/rc_keys.dart
abstract final class RcKeys {
  static const featureNewHome = 'feature_new_home_enabled';
  static const maxItemsFree = 'limit_max_items_free';
  static const maintenanceMode = 'ops_maintenance_mode';
  static const minSupportedVersion = 'ops_min_supported_version';
  static const onboardingVariant = 'exp_onboarding_variant';
}

// lib/core/remote_config/remote_config_service.dart
@lazySingleton
class RemoteConfigService {
  RemoteConfigService(this._rc);
  final FirebaseRemoteConfig _rc;

  static const _defaults = <String, dynamic>{
    RcKeys.featureNewHome: false,
    RcKeys.maxItemsFree: 3,
    RcKeys.maintenanceMode: false,
    RcKeys.minSupportedVersion: '1.0.0',
    RcKeys.onboardingVariant: 'control',
  };

  Future<void> init() async {
    await _rc.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: const Duration(seconds: 10),
      minimumFetchInterval: AppConfig.instance.isProd ? const Duration(hours: 1) : const Duration(minutes: 1),
    ));
    await _rc.setDefaults(_defaults);
    await _rc.activate();
    unawaited(_rc.fetch().catchError((Object e) => AppLogger.warning('RC fetch failed: $e')));
  }

  Stream<Set<String>> get onUpdated => kIsWeb
      ? const Stream.empty()
      : _rc.onConfigUpdated.asyncMap((u) async { await _rc.activate(); return u.updatedKeys; });

  FeatureFlags snapshot() => FeatureFlags(
        featureNewHome: _rc.getBool(RcKeys.featureNewHome),
        maxItemsFree: _rc.getInt(RcKeys.maxItemsFree),
        maintenanceMode: _rc.getBool(RcKeys.maintenanceMode),
        minSupportedVersion: _rc.getString(RcKeys.minSupportedVersion),
        onboardingVariant: _rc.getString(RcKeys.onboardingVariant),
      );
}

// lib/core/remote_config/feature_flags_cubit.dart
@lazySingleton
class FeatureFlagsCubit extends Cubit<FeatureFlags> {
  FeatureFlagsCubit(this._rc) : super(_rc.snapshot()) {
    _sub = _rc.onUpdated.listen((keys) {
      if (keys.any((k) => k.startsWith('ops_'))) emit(_rc.snapshot());
    });
  }
  final RemoteConfigService _rc;
  late final StreamSubscription<Set<String>> _sub;

  @override
  Future<void> close() async {
    await _sub.cancel();
    return super.close();
  }
}

// lib/core/remote_config/version_gate.dart
bool isBelowMinVersion(String minSupported) {
  final current = Version.parse(sl<PackageInfo>().version);
  return current < Version.parse(minSupported);
}

// A/B Experiment + Goal Metric + PDPL Consent
Widget buildOnboarding(BuildContext context) {
  final variant = context.select((FeatureFlagsCubit c) => c.state.onboardingVariant);
  return switch (variant) {
    'variant_a' => const OnboardingFlowA(),
    _ => const OnboardingFlowControl(),
  };
}

Future<void> logOnboardingGoal(Duration elapsed) => FirebaseAnalytics.instance.logEvent(
      name: 'onboarding_completed',
      parameters: {'seconds': elapsed.inSeconds},
    );

// Honour user consent (e.g., KSA PDPL / GDPR) before analytics collection
Future<void> applyAnalyticsConsent({required bool granted}) async {
  await FirebaseAnalytics.instance.setAnalyticsCollectionEnabled(granted);
  await FirebaseAnalytics.instance.setConsent(
    analyticsStorageConsentGranted: granted,
    adStorageConsentGranted: false,
  );
}
```
- **RC Security Rule:** Never treat Remote Config flags (`limit_max_items_free`) as security — enforce limits server-side in Cloud Functions / Security Rules. Only live-update `ops_*` keys; apply `feature_*` on next cold launch.

## 4. Complete GitHub Actions CI/CD Workflows
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request: {}
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
permissions: { contents: read }
jobs:
  flutter:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - uses: subosito/flutter-action@v2
        with: { channel: stable, flutter-version-file: .fvmrc, cache: true }
      - run: flutter pub get
      - run: flutter gen-l10n && dart run build_runner build --delete-conflicting-outputs
      - run: dart format --output=none --set-exit-if-changed lib test
      - run: flutter analyze
      - run: flutter test --coverage
      - run: flutter build web -t lib/main_dev.dart
  rules:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6 # Firestore emulator needs a JRE
        with: { distribution: temurin, java-version: "21" }
      - uses: actions/setup-node@v7
        with: { node-version: 22, cache: npm, cache-dependency-path: firebase/tests/package-lock.json }
      - run: npm ci --prefix firebase/tests
      - run: npx firebase-tools@latest emulators:exec --only firestore --project demo-rules "npx --prefix firebase/tests vitest run firebase/tests"
  functions:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    defaults: { run: { working-directory: firebase/functions } }
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 22, cache: npm, cache-dependency-path: firebase/functions/package-lock.json }
      - run: npm ci && npm run build

# .github/workflows/deploy-web.yml
# PR previews build the STAGING flavor and deploy to the STAGING Firebase project, so a preview
# never reads or writes production data. Only pushes to main ship the prod build to prod.
name: Deploy Web
on:
  pull_request: {}
  push: { branches: [main] }
permissions: { contents: read, pull-requests: write, checks: write }
jobs:
  hosting:
    # Fork PRs get no secrets: skip them instead of failing.
    if: github.event_name == 'push' || github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    environment: ${{ github.event_name == 'push' && 'production' || 'staging' }} # each env has its own SA secret
    env:
      SA: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
      TARGET: ${{ github.event_name == 'push' && 'lib/main_prod.dart' || 'lib/main_staging.dart' }}
      PROJECT: ${{ github.event_name == 'push' && 'your-prod-project-id' || 'your-staging-project-id' }}
    steps:
      - if: env.SA == ''
        run: echo "::notice::FIREBASE_SERVICE_ACCOUNT not set — skipping"
      - uses: actions/checkout@v7
        if: env.SA != ''
      - uses: subosito/flutter-action@v2
        if: env.SA != ''
        with: { channel: stable, flutter-version-file: .fvmrc, cache: true }
      - if: env.SA != ''
        run: flutter pub get && flutter gen-l10n && dart run build_runner build --delete-conflicting-outputs && flutter build web --release -t "$TARGET"
      - uses: FirebaseExtended/action-hosting-deploy@v0
        if: env.SA != ''
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          projectId: ${{ env.PROJECT }}
          channelId: ${{ github.event_name == 'push' && 'live' || '' }}
          expires: 14d

# .github/workflows/deploy-firebase.yml
name: Deploy Firebase Backend
on:
  push: { branches: [main], paths: ["firebase/**", "firebase.json"] }
permissions: { contents: read, id-token: write }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with: { node-version: 22, cache: npm, cache-dependency-path: firebase/functions/package-lock.json }
      - run: npm ci && npm run build
        working-directory: firebase/functions
      - uses: google-github-actions/auth@v3
        with: { workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}, service_account: ${{ vars.GCP_DEPLOY_SA }} }
      - run: npx firebase-tools@latest deploy --only firestore,storage,functions --project your-project-id --force --non-interactive

# .github/workflows/release-android.yml
name: Release Android
on: { push: { tags: ["v*.*.*"] } }
permissions: { contents: read }
jobs:
  android:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-java@v6
        with: { distribution: temurin, java-version: "21" }
      - uses: subosito/flutter-action@v2
        with: { channel: stable, flutter-version-file: .fvmrc, cache: true }
      - run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > android/app/upload-keystore.jks
          cat > android/key.properties <<EOF
          storeFile=upload-keystore.jks
          storePassword=${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          EOF
      - run: |
          flutter pub get && flutter gen-l10n && dart run build_runner build --delete-conflicting-outputs
          flutter build appbundle --release --flavor prod -t lib/main_prod.dart \
            --build-name "${GITHUB_REF_NAME#v}" --build-number "${{ github.run_number }}" \
            --obfuscate --split-debug-info=build/symbols
      - if: vars.FIREBASE_ANDROID_APP_ID != ''
        run: npx firebase-tools@latest crashlytics:symbols:upload --app=${{ vars.FIREBASE_ANDROID_APP_ID }} build/symbols
      - uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.example.myapp
          releaseFiles: build/app/outputs/bundle/prodRelease/app-prod-release.aab
          tracks: internal
          status: completed
          mappingFile: build/app/outputs/mapping/prodRelease/mapping.txt

# .github/workflows/release-ios.yml
name: Release iOS
on: { push: { tags: ["v*.*.*"] } }
permissions: { contents: read }
jobs:
  ios:
    runs-on: macos-latest
    environment: production
    env: { ASC_KEY_ID: ${{ vars.APPSTORE_API_KEY_ID }}, ASC_ISSUER: ${{ vars.APPSTORE_ISSUER_ID }} }
    steps:
      - uses: actions/checkout@v7
      - uses: subosito/flutter-action@v2
        with: { channel: stable, flutter-version-file: .fvmrc, cache: true }
      - run: |
          mkdir -p ~/.appstoreconnect
          echo "${{ secrets.APPSTORE_API_PRIVATE_KEY }}" > ~/.appstoreconnect/AuthKey.p8
      - run: |
          flutter pub get && flutter gen-l10n && dart run build_runner build --delete-conflicting-outputs
          flutter build ios --release --no-codesign --flavor prod -t lib/main_prod.dart \
            --build-name "${GITHUB_REF_NAME#v}" --build-number "${{ github.run_number }}" \
            --obfuscate --split-debug-info=build/symbols
      - run: |
          AUTH="-allowProvisioningUpdates -authenticationKeyPath $HOME/.appstoreconnect/AuthKey.p8 -authenticationKeyID $ASC_KEY_ID -authenticationKeyIssuerID $ASC_ISSUER"
          xcodebuild -workspace ios/Runner.xcworkspace -scheme prod -configuration Release-prod -destination 'generic/platform=iOS' -archivePath build/Runner.xcarchive archive $AUTH
          xcodebuild -exportArchive -archivePath build/Runner.xcarchive -exportOptionsPlist ios/ExportOptions.plist -exportPath build/ipa $AUTH
      - if: vars.FIREBASE_IOS_APP_ID != ''
        run: npx firebase-tools@latest crashlytics:symbols:upload --app=${{ vars.FIREBASE_IOS_APP_ID }} build/symbols
      - uses: apple-actions/upload-testflight-build@v5
        with:
          app-path: build/ipa/Runner.ipa
          issuer-id: ${{ vars.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ vars.APPSTORE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_API_PRIVATE_KEY }}
```

### Required GitHub Environment Secrets & Variables (`production` / `staging`)
| Secret / Variable | Used By | Creation |
|---|---|---|
| `FIREBASE_SERVICE_ACCOUNT` (secret, **per environment**) | `deploy-web.yml` | `production` env → prod project SA, `staging` env → staging project SA (`firebase init hosting:github`) |
| `GCP_WIF_PROVIDER`, `GCP_DEPLOY_SA` (vars) | `deploy-firebase.yml` | GCP Workload Identity Federation bound to this GitHub repo (keyless) |
| `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` | `release-android.yml` | `keytool` + `base64` |
| `PLAY_SERVICE_ACCOUNT_JSON` (secret) | `release-android.yml` | Play Console Release Manager service account |
| `APPSTORE_API_PRIVATE_KEY` (secret), `APPSTORE_API_KEY_ID`, `APPSTORE_ISSUER_ID` (vars) | `release-ios.yml` | App Store Connect API Key (cloud-managed signing via `templates/ExportOptions.plist` — zero `.p12` certs in secrets) |
| `FIREBASE_ANDROID_APP_ID`, `FIREBASE_IOS_APP_ID` (vars) | `release-*.yml` | Used by `firebase crashlytics:symbols:upload` after `--split-debug-info` |

## 5. Complete Test Templates (Unit, BLoC, Widget, Integration)
```dart
// test/unit/boards/create_board_test.dart
class MockBoardsRepository extends Mock implements BoardsRepository {}

void main() {
  late CreateBoard usecase;
  late MockBoardsRepository mockRepo;

  setUp(() {
    mockRepo = MockBoardsRepository();
    usecase = CreateBoard(mockRepo);
    registerFallbackValue(const CreateBoardParams(title: 't', color: '#000', userId: 'u1'));
  });

  const params = CreateBoardParams(title: 'My Board', color: '#6366F1', userId: 'user-1');

  test('returns BoardEntity on success', () async {
    when(() => mockRepo.createBoard(params)).thenAnswer((_) async => const Right(tBoardEntity));
    final result = await usecase(params);
    expect(result, const Right(tBoardEntity));
    verify(() => mockRepo.createBoard(params)).called(1);
  });

  test('returns failure unchanged', () async {
    when(() => mockRepo.createBoard(params)).thenAnswer((_) async => const Left(PermissionFailure()));
    expect(await usecase(params), const Left(PermissionFailure()));
  });
}

// test/unit/boards/boards_bloc_test.dart
blocTest<BoardsBloc, BoardsState>(
  'emits loading then success when WatchBoardsStarted succeeds',
  build: () {
    when(() => mockWatchBoards(any())).thenAnswer((_) => Stream.value(Right([tBoardEntity])));
    return BoardsBloc(mockWatchBoards, mockCreateBoard, mockDeleteBoard);
  },
  act: (b) => b.add(const WatchBoardsStarted('user-1')),
  expect: () => [
    const BoardsState(status: BoardsStatus.loading),
    BoardsState(status: BoardsStatus.success, boards: [tBoardEntity]),
  ],
);

blocTest<BoardsBloc, BoardsState>(
  'keeps existing boards and exposes the failure when the stream fails',
  build: () {
    when(() => mockWatchBoards(any())).thenAnswer(
      (_) => Stream<Either<AppFailure, List<BoardEntity>>>.fromIterable(
        [Right([tBoardEntity]), const Left(PermissionFailure())],
      ),
    );
    return BoardsBloc(mockWatchBoards, mockCreateBoard, mockDeleteBoard);
  },
  act: (b) => b.add(const WatchBoardsStarted('user-1')),
  expect: () => [
    const BoardsState(status: BoardsStatus.loading),
    BoardsState(status: BoardsStatus.success, boards: [tBoardEntity]),
    BoardsState(status: BoardsStatus.failure, boards: [tBoardEntity], failure: const PermissionFailure()),
  ],
);

blocTest<BoardsBloc, BoardsState>(
  'refresh re-subscribes exactly once (restartable)',
  build: () {
    when(() => mockWatchBoards(any())).thenAnswer((_) => Stream.value(Right([tBoardEntity])));
    return BoardsBloc(mockWatchBoards, mockCreateBoard, mockDeleteBoard);
  },
  act: (b) async {
    b.add(const WatchBoardsStarted('user-1'));
    await Future<void>.delayed(Duration.zero);
    b.add(const BoardsRefreshed());
  },
  verify: (_) => verify(() => mockWatchBoards('user-1')).called(2),
);

// test/widget/boards/home_page_test.dart
class MockBoardsBloc extends MockBloc<BoardsEvent, BoardsState> implements BoardsBloc {}

Future<void> pumpHome(WidgetTester tester, MockBoardsBloc bloc, BoardsState state) async {
  whenListen(bloc, const Stream<BoardsState>.empty(), initialState: state);
  await tester.pumpWidget(
    BlocProvider<BoardsBloc>.value(
      value: bloc,
      child: const MaterialApp(
        localizationsDelegates: AppLocalizations.localizationsDelegates,
        supportedLocales: AppLocalizations.supportedLocales,
        home: HomePage(),
      ),
    ),
  );
}

// integration_test/boards_integration_test.dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  setUpAll(() async {
    AppConfig.init(AppFlavor.dev);
    await Firebase.initializeApp(options: AppConfig.instance.firebaseOptions);
    FirebaseFirestore.instance.useFirestoreEmulator('localhost', 8080);
    await FirebaseAuth.instance.useAuthEmulator('localhost', 9099);
    sl.registerSingleton<ErrorReporter>(WebErrorReporter());
    await configureDependencies();
  });

  testWidgets('create board appears in home list', (tester) async {
    await FirebaseAuth.instance.createUserWithEmailAndPassword(email: 'test@app.com', password: 'Password123!');
    sl<AuthBloc>().add(const AuthWatchStarted());
    await tester.pumpWidget(const App());
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(const Key('create_board_fab')));
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(const Key('board_title_field')), 'Sprint 1');
    await tester.tap(find.byKey(const Key('submit_btn')));
    await tester.pumpAndSettle();
    expect(find.text('Sprint 1'), findsOneWidget);
  });
}
```
