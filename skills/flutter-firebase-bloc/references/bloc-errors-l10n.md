# BLoC, Errors, Theme, Adaptive UI & RTL Localization (Complete Code)

## 1. Single-State BLoC + Events + UI (`BoardsBloc`)
```dart
enum BoardsStatus { initial, loading, success, failure }

class BoardsState extends Equatable {
  const BoardsState({
    this.status = BoardsStatus.initial,
    this.boards = const [],
    this.failure,
  });
  final BoardsStatus status;
  final List<BoardEntity> boards;
  final AppFailure? failure;

  BoardsState copyWith({
    BoardsStatus? status,
    List<BoardEntity>? boards,
    AppFailure? Function()? failure,
  }) =>
      BoardsState(
        status: status ?? this.status,
        boards: boards ?? this.boards,
        failure: failure != null ? failure() : this.failure,
      );

  @override
  List<Object?> get props => [status, boards, failure];
}

sealed class BoardsEvent extends Equatable {
  const BoardsEvent();
  @override
  List<Object?> get props => [];
}
class WatchBoardsStarted extends BoardsEvent {
  const WatchBoardsStarted(this.userId);
  final String userId;
  @override
  List<Object?> get props => [userId];
}
class BoardsRefreshed extends BoardsEvent { const BoardsRefreshed(); }
class CreateBoardRequested extends BoardsEvent {
  const CreateBoardRequested({required this.title, required this.color, required this.userId});
  final String title, color, userId;
  @override
  List<Object?> get props => [title, color, userId];
}
class DeleteBoardRequested extends BoardsEvent {
  const DeleteBoardRequested(this.boardId);
  final String boardId;
  @override
  List<Object?> get props => [boardId];
}

@injectable
class BoardsBloc extends Bloc<BoardsEvent, BoardsState> {
  BoardsBloc(this._watchBoards, this._createBoard, this._deleteBoard) : super(const BoardsState()) {
    on<WatchBoardsStarted>(_onWatchStarted);
    on<BoardsRefreshed>(_onRefreshed);
    on<CreateBoardRequested>(_onCreateBoard);
    on<DeleteBoardRequested>(_onDeleteBoard);
  }
  final WatchBoards _watchBoards;
  final CreateBoard _createBoard;
  final DeleteBoard _deleteBoard;
  String? _userId;

  Future<void> _onWatchStarted(WatchBoardsStarted e, Emitter<BoardsState> emit) async {
    _userId = e.userId;
    emit(state.copyWith(status: BoardsStatus.loading));
    await emit.forEach(
      _watchBoards(e.userId),
      onData: (result) => result.fold(
        (f) => state.copyWith(status: BoardsStatus.failure, failure: () => f),
        (b) => state.copyWith(status: BoardsStatus.success, boards: b, failure: () => null),
      ),
    );
  }

  Future<void> _onRefreshed(BoardsRefreshed _, Emitter<BoardsState> emit) =>
      _userId == null ? Future.value() : _onWatchStarted(WatchBoardsStarted(_userId!), emit);

  Future<void> _onCreateBoard(CreateBoardRequested e, Emitter<BoardsState> emit) async {
    final res = await _createBoard(CreateBoardParams(title: e.title, color: e.color, userId: e.userId));
    res.fold((f) => emit(state.copyWith(status: BoardsStatus.failure, failure: () => f)), (_) {});
  }

  Future<void> _onDeleteBoard(DeleteBoardRequested e, Emitter<BoardsState> emit) async {
    final res = await _deleteBoard(e.boardId);
    res.fold((f) => emit(state.copyWith(status: BoardsStatus.failure, failure: () => f)), (_) {});
  }
}

// UI Usage (keep boards on screen if a refresh fails)
BlocConsumer<BoardsBloc, BoardsState>(
  listenWhen: (p, c) =>
      c.failure != null && p.failure != c.failure &&
      c.boards.isNotEmpty && c.failure is! CancelledFailure,
  listener: (context, s) => AppSnackBar.error(context, s.failure!.localize(context)),
  builder: (context, s) => switch (s) {
    BoardsState(status: BoardsStatus.loading, boards: []) => const AppShimmer(),
    BoardsState(status: BoardsStatus.failure, boards: [], :final failure?) => ErrorView(
        failure: failure,
        onRetry: () => context.read<BoardsBloc>().add(const BoardsRefreshed()),
      ),
    BoardsState(boards: []) => const AppEmptyState(),
    BoardsState(:final boards) => BoardsGrid(boards: boards),
  },
);
```

## 2. `AppBlocObserver` (`lib/core/observers/app_bloc_observer.dart`)
```dart
class AppBlocObserver extends BlocObserver {
  @override
  void onEvent(Bloc<dynamic, dynamic> bloc, Object? event) {
    super.onEvent(bloc, event);
    AppLogger.info('EVENT: $event', tag: '${bloc.runtimeType}');
  }

  @override
  void onTransition(Bloc<dynamic, dynamic> bloc, Transition<dynamic, dynamic> transition) {
    super.onTransition(bloc, transition);
    AppLogger.info(
      '${transition.currentState.runtimeType} -> ${transition.nextState.runtimeType}',
      tag: '${bloc.runtimeType}',
    );
  }

  @override
  void onError(BlocBase<dynamic> bloc, Object error, StackTrace stackTrace) {
    AppLogger.error('BLoC ${bloc.runtimeType}', error: error, stack: stackTrace);
    unawaited(sl<ErrorReporter>().record(error, stackTrace, reason: 'BLoC ${bloc.runtimeType}'));
    super.onError(bloc, error, stackTrace);
  }
}
```

## 3. Complete Error Handling (`failures.dart`, `exception_mapper.dart`, `failure_l10n.dart`, `error_reporter.dart`, `global_error_handlers.dart`, `app_logger.dart`, `error_view.dart`, `app_snack_bar.dart`, `friendly_error_widget.dart`)
```dart
// failures.dart
sealed class AppFailure extends Equatable {
  const AppFailure(this.message);
  final String message;
  @override
  List<Object?> get props => [message];
}
final class NetworkFailure extends AppFailure { const NetworkFailure() : super('network'); }
final class PermissionFailure extends AppFailure { const PermissionFailure() : super('permission-denied'); }
final class NotFoundFailure extends AppFailure { const NotFoundFailure() : super('not-found'); }
final class CancelledFailure extends AppFailure { const CancelledFailure() : super('cancelled'); }
final class CacheFailure extends AppFailure { const CacheFailure(super.message); }
final class FirebaseFailure extends AppFailure { const FirebaseFailure(super.message); }
final class ValidationFailure extends AppFailure { const ValidationFailure(super.message); }
final class AuthFailure extends AppFailure {
  const AuthFailure(this.code) : super('auth/$code');
  final String code;
}
final class UnknownFailure extends AppFailure { const UnknownFailure(super.message); }

// app_logger.dart
abstract final class AppLogger {
  static void info(String m, {String? tag}) { if (kDebugMode) debugPrint('[${tag ?? 'App'}] $m'); }
  static void warning(String m) { if (kDebugMode) debugPrint('[Warning] $m'); }
  static void error(String m, {Object? error, StackTrace? stack}) {
    if (kDebugMode) debugPrint('[Error] $m: $error');
    if (error != null) unawaited(sl<ErrorReporter>().record(error, stack, reason: m));
  }
}

// error_reporter.dart
abstract interface class ErrorReporter {
  Future<void> record(Object error, StackTrace? stack, {bool fatal = false, String? reason});
  Future<void> recordFlutterError(FlutterErrorDetails details);
  Future<void> setUser(String? uid);
  Future<void> setKey(String key, Object value);
}

class CrashlyticsErrorReporter implements ErrorReporter {
  final _c = FirebaseCrashlytics.instance;
  @override Future<void> record(Object e, StackTrace? s, {bool fatal = false, String? reason}) => _c.recordError(e, s, fatal: fatal, reason: reason);
  @override Future<void> recordFlutterError(FlutterErrorDetails d) => _c.recordFlutterFatalError(d);
  @override Future<void> setUser(String? uid) => _c.setUserIdentifier(uid ?? '');
  @override Future<void> setKey(String k, Object v) => _c.setCustomKey(k, v);
}

class WebErrorReporter implements ErrorReporter {
  @override Future<void> record(Object e, StackTrace? s, {bool fatal = false, String? reason}) async => debugPrint('$reason $e\n$s');
  @override Future<void> recordFlutterError(FlutterErrorDetails d) async => FlutterError.dumpErrorToConsole(d);
  @override Future<void> setUser(String? uid) async {}
  @override Future<void> setKey(String k, Object v) async {}
}

// global_error_handlers.dart
Future<void> installGlobalErrorHandlers(ErrorReporter reporter) async {
  if (!kIsWeb) {
    await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(kReleaseMode && !AppConfig.instance.isDev);
  }
  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    unawaited(reporter.recordFlutterError(details));
  };
  PlatformDispatcher.instance.onError = (error, stack) {
    unawaited(reporter.record(error, stack, fatal: true));
    return true;
  };
  if (!kIsWeb) {
    Isolate.current.addErrorListener(RawReceivePort((List<dynamic> pair) {
      unawaited(reporter.record(pair.first as Object, StackTrace.fromString(pair.last.toString()), fatal: true));
    }).sendPort);
  }
  ErrorWidget.builder = (details) =>
      kReleaseMode ? const FriendlyErrorWidget() : ErrorWidget(details.exception);
}

// exception_mapper.dart
extension FirebaseAuthExceptionX on FirebaseAuthException {
  AppFailure toFailure() => switch (code) {
        'invalid-credential' || 'wrong-password' || 'user-not-found' => const AuthFailure('invalid-credential'),
        'network-request-failed' => const NetworkFailure(),
        _ => AuthFailure(code),
      };
}
extension FirebaseExceptionX on FirebaseException {
  AppFailure toFailure() => switch (code) {
        'unavailable' || 'deadline-exceeded' => const NetworkFailure(),
        'permission-denied' => const PermissionFailure(),
        'not-found' => const NotFoundFailure(),
        _ => FirebaseFailure(message ?? code),
      };
}

// failure_l10n.dart
extension AppFailureL10n on AppFailure {
  String localize(BuildContext context) {
    final l = AppLocalizations.of(context);
    return switch (this) {
      NetworkFailure() => l.errorNoInternet,
      PermissionFailure() => l.errorNoPermission,
      NotFoundFailure() => l.errorNotFound,
      CancelledFailure() => '',
      ValidationFailure(:final message) => message,
      AuthFailure(:final code) => switch (code) {
          'invalid-credential' => l.authInvalidCredential,
          'email-already-in-use' => l.authEmailInUse,
          'weak-password' => l.authWeakPassword,
          'too-many-requests' => l.authTooManyRequests,
          'user-disabled' => l.authUserDisabled,
          _ => l.errorGeneric,
        },
      FirebaseFailure() || CacheFailure() || UnknownFailure() => l.errorGeneric,
    };
  }
}

// error_view.dart
class ErrorView extends StatelessWidget {
  const ErrorView({super.key, required this.failure, this.onRetry});
  final AppFailure failure;
  final VoidCallback? onRetry;

  IconData get _icon => switch (failure) {
        NetworkFailure() => Icons.wifi_off_rounded,
        PermissionFailure() => Icons.lock_outline_rounded,
        NotFoundFailure() => Icons.search_off_rounded,
        _ => Icons.error_outline_rounded,
      };

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final canRetry = onRetry != null && failure is! PermissionFailure && failure is! NotFoundFailure;
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(mainAxisSize: MainAxisSize.min, children: [
          Icon(_icon, size: 48, color: theme.colorScheme.onSurfaceVariant),
          const SizedBox(height: 16),
          Text(failure.localize(context), textAlign: TextAlign.center, style: theme.textTheme.bodyLarge),
          if (canRetry) ...[
            const SizedBox(height: 16),
            FilledButton.tonalIcon(
              onPressed: onRetry,
              icon: const Icon(Icons.refresh_rounded),
              label: Text(AppLocalizations.of(context).retry),
            ),
          ],
        ]),
      ),
    );
  }
}

// app_snack_bar.dart & friendly_error_widget.dart
abstract final class AppSnackBar {
  static void error(BuildContext context, String message) => _show(
        context, message, Theme.of(context).colorScheme.error, Theme.of(context).colorScheme.onError,
      );
  static void success(BuildContext context, String message) => _show(
        context, message, Theme.of(context).colorScheme.primary, Theme.of(context).colorScheme.onPrimary,
      );
  static void _show(BuildContext context, String msg, Color bg, Color fg) {
    if (msg.isEmpty) return;
    ScaffoldMessenger.of(context)
      ..hideCurrentSnackBar()
      ..showSnackBar(SnackBar(content: Text(msg, style: TextStyle(color: fg)), backgroundColor: bg, behavior: SnackBarBehavior.floating));
  }
}

class FriendlyErrorWidget extends StatelessWidget {
  const FriendlyErrorWidget({super.key});
  @override
  Widget build(BuildContext context) => const ColoredBox(
        color: Color(0xFFF5F5F7),
        child: Center(child: Icon(Icons.error_outline_rounded, size: 40, color: Color(0xFF9E9E9E))),
      );
}
```

## 4. Theme, Responsive Layouts & Localization (`app_theme.dart`, `theme_cubit.dart`, `adaptive_scaffold.dart`, `locale_cubit.dart`)
```dart
class AppTheme {
  static const _primary = Color(0xFF6366F1);
  static const _secondary = Color(0xFF818CF8);
  static const _error = Color(0xFFEF4444);

  static ThemeData get light => _build(const ColorScheme.light(
        primary: _primary, secondary: _secondary, surface: Color(0xFFFFFFFF), error: _error,
      ));
  static ThemeData get dark => _build(const ColorScheme.dark(
        primary: _primary, secondary: _secondary, surface: Color(0xFF1A1D2E), error: _error,
      ));

  static ThemeData _build(ColorScheme cs) => ThemeData(
        useMaterial3: true,
        colorScheme: cs,
        appBarTheme: const AppBarTheme(elevation: 0, centerTitle: false),
        cardTheme: CardThemeData(
          elevation: 0,
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
        ),
      );
}

@lazySingleton
class ThemeCubit extends Cubit<ThemeMode> {
  ThemeCubit(this._prefs) : super(ThemeMode.values.elementAt(_prefs.getInt(_key) ?? 0));
  static const _key = 'theme_mode';
  final SharedPreferences _prefs;
  Future<void> setMode(ThemeMode mode) async {
    await _prefs.setInt(_key, mode.index);
    emit(mode);
  }
}

@lazySingleton
class LocaleCubit extends Cubit<Locale?> {
  LocaleCubit(this._prefs) : super(_read(_prefs));
  static const _key = 'locale_code';
  final SharedPreferences _prefs;
  static Locale? _read(SharedPreferences p) {
    final code = p.getString(_key);
    return code == null ? null : Locale(code);
  }
  Future<void> setLocale(Locale? locale) async {
    if (locale == null) {
      await _prefs.remove(_key);
    } else {
      await _prefs.setString(_key, locale.languageCode);
    }
    emit(locale);
  }
}

class ResponsiveLayout extends StatelessWidget {
  const ResponsiveLayout({super.key, required this.mobile, this.tablet, required this.desktop});
  final Widget mobile, desktop;
  final Widget? tablet;
  @override
  Widget build(BuildContext context) => LayoutBuilder(builder: (_, c) {
        if (c.maxWidth >= 1024) return desktop;
        if (c.maxWidth >= 600) return tablet ?? desktop;
        return mobile;
      });
}

class AdaptiveScaffold extends StatelessWidget {
  const AdaptiveScaffold({
    super.key,
    required this.selectedIndex,
    required this.onSelected,
    required this.destinations,
    required this.body,
  });
  final int selectedIndex;
  final ValueChanged<int> onSelected;
  final List<NavigationDestination> destinations;
  final Widget body;

  @override
  Widget build(BuildContext context) {
    final wide = MediaQuery.sizeOf(context).width >= 600;
    return Scaffold(
      body: wide
          ? Row(children: [
              NavigationRail(
                selectedIndex: selectedIndex,
                onDestinationSelected: onSelected,
                destinations: [
                  for (final d in destinations)
                    NavigationRailDestination(icon: d.icon, selectedIcon: d.selectedIcon, label: Text(d.label)),
                ],
              ),
              const VerticalDivider(width: 1),
              Expanded(child: body),
            ])
          : body,
      bottomNavigationBar: wide
          ? null
          : NavigationBar(selectedIndex: selectedIndex, onDestinationSelected: onSelected, destinations: destinations),
    );
  }
}
```
- **RTL Rule:** ALWAYS use `EdgeInsetsDirectional`, `AlignmentDirectional`, `BorderRadiusDirectional`, and `Icons.arrow_back`. Never `EdgeInsets.only(left: ...)` or `right: ...`.
