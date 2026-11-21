# Notifications, Cloud Functions & GoRouter Deep Links (Complete Code)

## 1. AndroidManifest & Channels (`NotificationChannels`, `LocalNotificationService`)
```xml
<!-- android/app/src/main/AndroidManifest.xml (inside <application>) -->
<meta-data android:name="com.google.firebase.messaging.default_notification_channel_id" android:value="high_importance_channel" />
<meta-data android:name="com.google.firebase.messaging.default_notification_icon" android:resource="@drawable/ic_notification" />
<meta-data android:name="com.google.firebase.messaging.default_notification_color" android:resource="@color/notification_color" />
<!-- Inside <manifest> -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

```dart
// lib/core/notifications/notification_channels.dart
class NotificationChannels {
  static const highImportance = AndroidNotificationChannel(
    'high_importance_channel',
    'High Importance Notifications',
    description: 'Board assignments, mentions & urgent tasks',
    importance: Importance.max,
    playSound: true,
    enableVibration: true,
  );
  static const low = AndroidNotificationChannel(
    'low_channel',
    'General Updates',
    description: 'Comments, board updates & reminders',
    importance: Importance.defaultImportance,
  );

  static Future<void> register(FlutterLocalNotificationsPlugin plugin) async {
    final android = plugin.resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>();
    await android?.createNotificationChannel(highImportance);
    await android?.createNotificationChannel(low);
  }
}

// lib/core/notifications/local_notification_service.dart
@lazySingleton
class LocalNotificationService {
  final _plugin = FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    const initSettings = InitializationSettings(
      android: AndroidInitializationSettings('@drawable/ic_notification'),
      iOS: DarwinInitializationSettings(
        requestAlertPermission: false,
        requestBadgePermission: false,
        requestSoundPermission: false,
      ),
    );
    // flutter_local_notifications >= 20: named parameters
    await _plugin.initialize(settings: initSettings, onDidReceiveNotificationResponse: _onTap);
    await NotificationChannels.register(_plugin);
  }

  Future<void> show({
    required int id,
    required String title,
    required String body,
    String? payload,
    bool highPriority = true,
  }) async {
    final channel = highPriority ? NotificationChannels.highImportance : NotificationChannels.low;
    final details = NotificationDetails(
      android: AndroidNotificationDetails(
        channel.id, channel.name,
        channelDescription: channel.description,
        importance: channel.importance,
        priority: Priority.high,
        icon: '@drawable/ic_notification',
        styleInformation: BigTextStyleInformation(body),
      ),
      iOS: const DarwinNotificationDetails(presentAlert: true, presentBadge: true, presentSound: true),
    );
    await _plugin.show(id: id, title: title, body: body, notificationDetails: details, payload: payload);
  }

  void _onTap(NotificationResponse response) {
    if (response.payload == null) return;
    NotificationNavigator.navigateFromPayload(response.payload!);
  }
}
```

## 2. Background Handler, `App` Widget & Complete `FcmService`
```dart
// lib/core/notifications/fcm_background.dart
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  debugPrint('BG message: ${message.messageId}');
}

// lib/app.dart
class App extends StatefulWidget {
  const App({super.key});
  @override
  State<App> createState() => _AppState();
}

class _AppState extends State<App> {
  final _messengerKey = GlobalKey<ScaffoldMessengerState>();
  late final StreamSubscription<AppFailure> _writeErrorSub;
  late final AuthBloc _authBloc = sl<AuthBloc>();
  late final FeatureFlagsCubit _flags = sl<FeatureFlagsCubit>();
  late final ThemeCubit _themeCubit = sl<ThemeCubit>();
  late final LocaleCubit _localeCubit = sl<LocaleCubit>();
  late final ConnectivityCubit _connectivity = sl<ConnectivityCubit>();
  late final GoRouter _router = createAppRouter(authBloc: _authBloc, flags: _flags);

  @override
  void initState() {
    super.initState();
    AppNavigator.router = _router;
    _writeErrorSub = sl<WriteErrorBus>().stream.listen(_showWriteError);
    unawaited(_openFromTerminatedNotification());
  }

  /// Offline-first writes rejected by the server (rules / App Check) surface here.
  void _showWriteError(AppFailure failure) {
    final messenger = _messengerKey.currentState;
    final ctx = _messengerKey.currentContext;
    if (messenger == null || ctx == null) return;
    final cs = Theme.of(ctx).colorScheme;
    messenger
      ..hideCurrentSnackBar()
      ..showSnackBar(SnackBar(
        content: Text(AppLocalizations.of(ctx).errorWriteRejected, style: TextStyle(color: cs.onError)),
        backgroundColor: cs.error,
        behavior: SnackBarBehavior.floating,
      ));
  }

  @override
  void dispose() {
    unawaited(_writeErrorSub.cancel());
    super.dispose();
  }

  Future<void> _openFromTerminatedNotification() async {
    final message = await FirebaseMessaging.instance.getInitialMessage();
    if (message == null || !mounted) return;
    WidgetsBinding.instance.addPostFrameCallback(
      (_) => NotificationNavigator.navigate(message.data),
    );
  }

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider.value(value: _authBloc),
        BlocProvider.value(value: _flags),
        BlocProvider.value(value: _themeCubit),
        BlocProvider.value(value: _localeCubit),
        BlocProvider.value(value: _connectivity),
      ],
      child: BlocBuilder<ThemeCubit, ThemeMode>(
        builder: (context, themeMode) => BlocBuilder<LocaleCubit, Locale?>(
          builder: (context, locale) => MaterialApp.router(
            title: AppConfig.instance.appName,
            theme: AppTheme.light,
            darkTheme: AppTheme.dark,
            themeMode: themeMode,
            locale: locale,
            localizationsDelegates: AppLocalizations.localizationsDelegates,
            supportedLocales: AppLocalizations.supportedLocales,
            routerConfig: _router,
            scaffoldMessengerKey: _messengerKey,
            builder: (context, child) {
              final app = Column(children: [const OfflineBanner(), Expanded(child: child!)]);
              return AppConfig.instance.isProd
                  ? app
                  : Banner(
                      message: AppConfig.instance.isDev ? 'DEV' : 'STG',
                      location: BannerLocation.topEnd,
                      color: AppConfig.instance.isDev ? Colors.orange : Colors.purple,
                      child: app,
                    );
            },
          ),
        ),
      ),
    );
  }
}

// lib/core/notifications/fcm_service.dart
@lazySingleton
class FcmService {
  FcmService(this._messaging, this._firestore, this._local);
  final FirebaseMessaging _messaging;
  final FirebaseFirestore _firestore;
  final LocalNotificationService _local;

  StreamSubscription<String>? _tokenSub;
  StreamSubscription<RemoteMessage>? _fgSub, _tapSub;
  String? _activeUserId;

  Future<void> initialize(String userId) async {
    if (_activeUserId == userId) return;
    dispose();
    _activeUserId = userId;

    if (!kIsWeb) await _local.init();
    final settings = await _messaging.requestPermission(provisional: true);
    AppLogger.info('FCM permission: ${settings.authorizationStatus}', tag: 'FCM');
    if (settings.authorizationStatus == AuthorizationStatus.denied) {
      await _firestore.collection('users').doc(userId).set(
        {'notificationsEnabled': false},
        SetOptions(merge: true),
      );
      return;
    }

    final token = await _messaging.getToken(
      vapidKey: kIsWeb ? AppConfig.instance.webVapidKey : null,
    );
    if (token != null) await _saveToken(userId, token);

    _tokenSub = _messaging.onTokenRefresh.listen(
      (t) => _saveToken(userId, t),
      onError: (Object e, StackTrace s) => AppLogger.error('Token refresh error', error: e, stack: s),
    );
    _fgSub = FirebaseMessaging.onMessage.listen(_handleForeground);
    _tapSub = FirebaseMessaging.onMessageOpenedApp.listen((m) => NotificationNavigator.navigate(m.data));
  }

  Future<void> _handleForeground(RemoteMessage m) async {
    final n = m.notification;
    if (n == null || kIsWeb) return;
    await _local.show(
      id: (m.messageId ?? '${m.sentTime}').hashCode & 0x7fffffff, // stable 31-bit id
      title: n.title ?? '',
      body: n.body ?? '',
      payload: jsonEncode(m.data),
      highPriority: m.data['priority'] == 'high',
    );
    if (await AppBadgePlus.isSupported()) {
      final current = await AppBadgePlus.getBadgeCount();
      await AppBadgePlus.updateBadge(current + 1);
    }
  }

  Future<void> _saveToken(String uid, String token) =>
      _firestore.collection('users').doc(uid).set({
        'fcmTokens': FieldValue.arrayUnion([token]),
        'notificationsEnabled': true,
        'lastTokenUpdatedAt': FieldValue.serverTimestamp(),
      }, SetOptions(merge: true));

  /// Call BEFORE signing out (the write needs auth). Best effort: never block sign-out.
  Future<void> removeToken(String uid) async {
    try {
      final t = await _messaging.getToken(vapidKey: kIsWeb ? AppConfig.instance.webVapidKey : null);
      if (t != null) {
        await _firestore.collection('users').doc(uid).update({
          'fcmTokens': FieldValue.arrayRemove([t]),
        }).timeout(const Duration(seconds: 5));
      }
    } catch (e, s) {
      AppLogger.error('removeToken failed', error: e, stack: s);
    }
    await deleteLocalToken();
  }

  /// After account deletion the user doc is gone; only invalidate the device token.
  Future<void> deleteLocalToken() async {
    try {
      await _messaging.deleteToken();
    } catch (e, s) {
      AppLogger.error('deleteToken failed', error: e, stack: s);
    }
  }

  Future<void> clearBadge() async {
    if (!kIsWeb && await AppBadgePlus.isSupported()) await AppBadgePlus.updateBadge(0);
  }

  void dispose() {
    unawaited(_tokenSub?.cancel());
    unawaited(_fgSub?.cancel());
    unawaited(_tapSub?.cancel());
    _tokenSub = _fgSub = _tapSub = null;
    _activeUserId = null;
  }
}

// lib/core/notifications/notification_navigator.dart
abstract final class NotificationNavigator {
  static void navigateFromPayload(String payload) {
    try {
      final decoded = jsonDecode(payload);
      if (decoded is Map<String, dynamic>) navigate(decoded);
    } catch (e, s) {
      AppLogger.error('Invalid notification payload', error: e, stack: s);
    }
  }

  static void navigate(Map<String, dynamic> data) {
    final type = data['type'] as String?;
    final id = data['id'] as String?;
    final boardId = data['boardId'] as String?;
    switch (type) {
      case 'board_invite':
        if (id != null) AppNavigator.go(RouteNames.boardPath(id));
      case 'card_assigned' || 'comment_added':
        if (boardId != null) AppNavigator.go(RouteNames.boardPath(boardId));
      default:
        AppNavigator.go(RouteNames.notifications);
    }
  }
}
```

## 3. `NotificationsBloc` & `UnreadBadge`
```dart
enum NotificationsStatus { initial, loading, success, failure }

class NotificationsState extends Equatable {
  const NotificationsState({
    this.status = NotificationsStatus.initial,
    this.notifications = const [],
    this.unreadCount = 0,
    this.failure,
  });
  final NotificationsStatus status;
  final List<NotificationEntity> notifications;
  final int unreadCount;
  final AppFailure? failure;

  NotificationsState copyWith({
    NotificationsStatus? status,
    List<NotificationEntity>? notifications,
    int? unreadCount,
    AppFailure? Function()? failure,
  }) =>
      NotificationsState(
        status: status ?? this.status,
        notifications: notifications ?? this.notifications,
        unreadCount: unreadCount ?? this.unreadCount,
        failure: failure != null ? failure() : this.failure,
      );

  @override
  List<Object?> get props => [status, notifications, unreadCount, failure];
}

// Badge UI
BlocBuilder<NotificationsBloc, NotificationsState>(
  buildWhen: (p, c) => p.unreadCount != c.unreadCount,
  builder: (_, s) => Badge(
    isLabelVisible: s.unreadCount > 0,
    label: Text(s.unreadCount > 99 ? '99+' : '${s.unreadCount}'),
    child: const Icon(Icons.notifications_outlined),
  ),
);
```

## 4. Cloud Functions v2 (`firebase/functions/src/*`) + Web Service Worker
```javascript
// web/firebase-messaging-sw.js (keep the SDK version in sync with Flutter web's Firebase JS SDK)
importScripts('https://www.gstatic.com/firebasejs/13.4.0/firebase-app-compat.js');
importScripts('https://www.gstatic.com/firebasejs/13.4.0/firebase-messaging-compat.js');
firebase.initializeApp({ apiKey: '...', projectId: '...', messagingSenderId: '...', appId: '...' });
firebase.messaging().onBackgroundMessage((p) => {
  if (p.notification?.title) self.registration.showNotification(p.notification.title, { body: p.notification.body, data: p.data });
});
```

```typescript
// firebase/functions/src/index.ts — firebase-functions ^7.4 (2nd gen) + firebase-admin ^14.5 (modular API)
import { initializeApp } from 'firebase-admin/app';
import { getFirestore, FieldValue } from 'firebase-admin/firestore';
import { getMessaging } from 'firebase-admin/messaging';
import { getStorage } from 'firebase-admin/storage';
import { onCall, HttpsError } from 'firebase-functions/https';
import { onDocumentWritten } from 'firebase-functions/firestore';
import { onUserDeleted } from 'firebase-functions/identity';
import { logger } from 'firebase-functions';

initializeApp();
const db = getFirestore();

// Called by BoardsRemoteDatasource.deleteBoard (client batches can't see uncached sub-docs offline).
export const deleteBoardRecursive = onCall({ enforceAppCheck: true }, async (req) => {
  const uid = req.auth?.uid;
  const boardId = req.data?.boardId;
  if (!uid) throw new HttpsError('unauthenticated', 'Sign in required');
  if (typeof boardId !== 'string' || boardId.length === 0 || boardId.length > 128) {
    throw new HttpsError('invalid-argument', 'boardId required');
  }
  const ref = db.collection('boards').doc(boardId);
  const snap = await ref.get();
  if (!snap.exists || snap.get('ownerId') !== uid) {
    throw new HttpsError('permission-denied', 'Owner only');
  }
  await db.recursiveDelete(ref);
  await getStorage().bucket().deleteFiles({ prefix: `boards/${boardId}/` });
  return { deleted: true };
});

// Account deletion (App Store 5.1.1(v) / Google Play data-deletion policy): erase everything
// the user owns when `user.delete()` succeeds on the client.
export const cleanupDeletedUser = onUserDeleted({}, async (event) => {
  const uid = event.data.uid;
  const bulk = db.bulkWriter();

  // 1. Boards the user owns: delete recursively (cards, activity, attachments).
  const owned = await db.collection('boards').where('ownerId', '==', uid).get();
  for (const board of owned.docs) {
    await db.recursiveDelete(board.ref, bulk);
    await getStorage().bucket().deleteFiles({ prefix: `boards/${board.id}/` });
  }
  // 2. Boards shared with the user: remove membership only.
  const shared = await db.collection('boards').where('memberIds', 'array-contains', uid).get();
  for (const board of shared.docs) {
    if (board.get('ownerId') !== uid) bulk.update(board.ref, { memberIds: FieldValue.arrayRemove(uid) });
  }
  // 3. Personal documents.
  const notifs = await db.collection('notifications').where('userId', '==', uid).get();
  notifs.docs.forEach((d) => bulk.delete(d.ref));
  bulk.delete(db.doc(`publicProfiles/${uid}`));
  bulk.delete(db.doc(`users/${uid}`));
  await bulk.close();
  await getStorage().bucket().deleteFiles({ prefix: `users/${uid}/` });
  logger.info('User data erased', { uid, ownedBoards: owned.size });
});

export const onCardAssigned = onDocumentWritten('boards/{boardId}/cards/{cardId}', async (event) => {
  const before = event.data?.before.data();
  const after = event.data?.after.data();
  if (!after || !after.assigneeId || before?.assigneeId === after.assigneeId) return;

  const assigneeId = after.assigneeId as string;
  const { boardId, cardId } = event.params;
  const title = 'New card assigned to you';
  const body = typeof after.title === 'string' ? after.title : 'You have a new task';
  const data = { type: 'card_assigned', id: cardId, boardId }; // FCM data values must be strings

  await db.collection('notifications').add({
    userId: assigneeId, title, body, data, isRead: false, createdAt: FieldValue.serverTimestamp(),
  });

  const userDoc = await db.collection('users').doc(assigneeId).get();
  const tokens: string[] = userDoc.get('fcmTokens') ?? [];
  if (!tokens.length) return;

  const res = await getMessaging().sendEachForMulticast({
    tokens,
    notification: { title, body },
    data,
    android: { priority: 'high', notification: { channelId: 'high_importance_channel', priority: 'max', sound: 'default' } },
    apns: { payload: { aps: { sound: 'default', badge: 1, contentAvailable: true } } },
  });

  const stale = res.responses
    .map((r, i) => ({ r, t: tokens[i] }))
    .filter(({ r }) => !r.success && ['messaging/registration-token-not-registered', 'messaging/invalid-registration-token'].includes(r.error?.code ?? ''))
    .map(({ t }) => t);
  if (stale.length) await userDoc.ref.update({ fcmTokens: FieldValue.arrayRemove(...stale) });
});

// Do NOT fan out private board updates through FCM topics: any client can subscribe to
// any topic name. Send to the members' tokens (as above) instead.
```

## 5. Complete `GoRouter` (`app_router.dart`, `route_names.dart`, `stream_listenable.dart`, `app_navigator.dart`) + Universal/App Links
```dart
// route_names.dart
abstract final class RouteNames {
  static const splash = '/splash';
  static const onboarding = '/onboarding';
  static const welcome = '/auth/welcome';
  static const phoneLogin = '/auth/phone';
  static const otp = '/auth/otp';
  static const emailLogin = '/auth/email';
  static const home = '/home';
  static const board = '/board/:id';
  static const notifications = '/notifications';
  static const maintenance = '/maintenance';
  static const updateRequired = '/update-required';
  static const authPrefix = '/auth';
  static String boardPath(String id) => '/board/$id';
}

// stream_listenable.dart & app_navigator.dart
class StreamListenable extends ChangeNotifier {
  StreamListenable(Stream<dynamic> stream) { _sub = stream.listen((_) => notifyListeners()); }
  late final StreamSubscription<dynamic> _sub;
  @override
  void dispose() { unawaited(_sub.cancel()); super.dispose(); }
}

abstract final class AppNavigator {
  static late GoRouter router;
  static void go(String location) => router.go(location);
}

// app_router.dart
GoRouter createAppRouter({required AuthBloc authBloc, required FeatureFlagsCubit flags}) {
  return GoRouter(
    initialLocation: RouteNames.splash,
    refreshListenable: Listenable.merge([StreamListenable(authBloc.stream), StreamListenable(flags.stream)]),
    redirect: (context, state) {
      final loc = state.matchedLocation;
      // Keep the original deep link (notification / universal link) through splash & login.
      String withFrom(String target) {
        final from = state.uri.queryParameters['from'] ??
            (_isPublic(loc) ? null : state.uri.toString());
        return from == null ? target : Uri(path: target, queryParameters: {'from': from}).toString();
      }

      final f = flags.state;
      if (f.maintenanceMode) return loc == RouteNames.maintenance ? null : RouteNames.maintenance;
      if (isBelowMinVersion(f.minSupportedVersion)) {
        return loc == RouteNames.updateRequired ? null : RouteNames.updateRequired;
      }
      if (loc == RouteNames.maintenance || loc == RouteNames.updateRequired) return RouteNames.splash;

      final authState = authBloc.state;
      if (authState.status == AuthStatus.unknown) {
        return loc == RouteNames.splash ? null : withFrom(RouteNames.splash);
      }
      // `user != null`, not `status == authenticated`: loading/failure states during
      // delete-account or re-auth keep the user, so they must not bounce to welcome.
      final isAuth = authState.user != null;
      final onAuthRoute = loc.startsWith(RouteNames.authPrefix);

      if (!isAuth) return onAuthRoute ? null : withFrom(RouteNames.welcome);
      if (authState.user!.needsOnboarding) {
        return loc == RouteNames.onboarding ? null : withFrom(RouteNames.onboarding);
      }
      if (loc == RouteNames.splash || onAuthRoute || loc == RouteNames.onboarding) {
        return _safeFrom(state.uri.queryParameters['from']) ?? RouteNames.home;
      }
      return null;
    },
    routes: [
      GoRoute(path: RouteNames.splash, builder: (_, __) => const SplashPage()),
      GoRoute(path: RouteNames.onboarding, builder: (_, __) => const OnboardingPage()),
      GoRoute(path: RouteNames.welcome, builder: (_, __) => const WelcomePage()),
      GoRoute(path: RouteNames.phoneLogin, builder: (_, __) => const PhoneLoginPage()),
      GoRoute(path: RouteNames.otp, builder: (_, __) => const OtpPage()),
      GoRoute(path: RouteNames.emailLogin, builder: (_, __) => const EmailLoginPage()),
      GoRoute(path: RouteNames.home, builder: (_, __) => const HomePage()),
      GoRoute(path: RouteNames.board, builder: (_, s) => BoardPage(boardId: s.pathParameters['id']!)),
      GoRoute(path: RouteNames.notifications, builder: (_, __) => const NotificationsPage()),
      GoRoute(path: RouteNames.maintenance, builder: (_, __) => const MaintenancePage()),
      GoRoute(path: RouteNames.updateRequired, builder: (_, __) => const UpdateRequiredPage()),
    ],
  );
}

bool _isPublic(String loc) =>
    loc == RouteNames.splash || loc == RouteNames.onboarding || loc.startsWith(RouteNames.authPrefix) ||
    loc == RouteNames.maintenance || loc == RouteNames.updateRequired;

/// Only same-app relative paths: blocks open redirects like `?from=https://evil.com`.
String? _safeFrom(String? from) {
  if (from == null || !from.startsWith('/') || from.startsWith('//')) return null;
  return _isPublic(Uri.parse(from).path) ? null : from;
}
```
- Universal Links / App Links: Place `public/.well-known/assetlinks.json` (Android SHA-256) and `public/.well-known/apple-app-site-association` (iOS Team ID + `/board/*`), and enable `FlutterDeepLinkingEnabled = true` in `ios/Runner/Info.plist`.
