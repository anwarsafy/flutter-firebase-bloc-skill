# Architecture, Offline-First Data & Dependency Injection (Complete Code)

## 1. `pubspec.yaml`
```yaml
name: my_app
description: Flutter + Firebase + BLoC Production App
publish_to: none
version: 1.0.0+1

environment:
  sdk: ">=3.5.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # Firebase
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.4
  firebase_messaging: ^15.1.3
  firebase_storage: ^12.3.2
  firebase_analytics: ^11.3.3
  firebase_crashlytics: ^4.1.3
  firebase_performance: ^0.10.0
  firebase_remote_config: ^5.1.0
  firebase_app_check: ^0.3.1
  google_sign_in: ^6.2.1
  sign_in_with_apple: ^6.1.1

  # State & DI
  flutter_bloc: ^9.0.0
  equatable: ^2.0.5
  get_it: ^8.0.2
  injectable: ^2.4.4

  # Navigation & Functional
  go_router: ^14.2.7
  dartz: ^0.10.1

  # Notifications & UI
  flutter_local_notifications: ^18.0.0
  app_badge_plus: ^1.1.0
  cached_network_image: ^3.4.1

  # Network, Storage & Utils
  connectivity_plus: ^6.0.5
  shared_preferences: ^2.3.2
  flutter_secure_storage: ^9.2.2
  dio: ^5.7.0
  intl: ^0.19.0
  crypto: ^3.0.5
  package_info_plus: ^8.0.2
  pub_semver: ^2.1.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  very_good_analysis: ^6.0.0
  bloc_test: ^10.0.0
  mocktail: ^1.0.4
  fake_cloud_firestore: ^3.0.2
  build_runner: ^2.4.13
  injectable_generator: ^2.6.2

flutter:
  generate: true
  uses-material-design: true
```

## 2. UseCase Base (`lib/core/usecases/usecase.dart`)
```dart
import 'package:dartz/dartz.dart';
import 'package:equatable/equatable.dart';
import '../errors/failures.dart';

abstract interface class UseCase<Type, Params> {
  Future<Either<AppFailure, Type>> call(Params params);
}

abstract interface class StreamUseCase<Type, Params> {
  Stream<Either<AppFailure, Type>> call(Params params);
}

final class NoParams extends Equatable {
  const NoParams();
  @override
  List<Object?> get props => [];
}
```

## 3. Dependency Injection (`lib/injection_container.dart`)
```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:firebase_performance/firebase_performance.dart';
import 'package:firebase_remote_config/firebase_remote_config.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'package:shared_preferences/shared_preferences.dart';

import 'injection_container.config.dart';

final GetIt sl = GetIt.instance;

@InjectableInit()
Future<void> configureDependencies() => sl.init();

@module
abstract class FirebaseModule {
  @lazySingleton FirebaseFirestore get firestore => FirebaseFirestore.instance;
  @lazySingleton FirebaseAuth get auth => FirebaseAuth.instance;
  @lazySingleton FirebaseMessaging get messaging => FirebaseMessaging.instance;
  @lazySingleton FirebaseStorage get storage => FirebaseStorage.instance;
  @lazySingleton FirebaseRemoteConfig get remoteConfig => FirebaseRemoteConfig.instance;
  @lazySingleton FirebasePerformance get performance => FirebasePerformance.instance;
  @lazySingleton FirebaseAnalytics get analytics => FirebaseAnalytics.instance;
  @lazySingleton Connectivity get connectivity => Connectivity();
  @lazySingleton FlutterSecureStorage get secureStorage => const FlutterSecureStorage();
  @preResolve Future<SharedPreferences> get prefs => SharedPreferences.getInstance();
  @preResolve Future<PackageInfo> get packageInfo => PackageInfo.fromPlatform();
}
```

## 4. Offline-First Firestore + Connectivity (`core/firebase/firestore_setup.dart`, `core/network/*`, `core/widgets/offline_banner.dart`)
```dart
// lib/core/firebase/firestore_setup.dart
import 'package:cloud_firestore/cloud_firestore.dart';

void configureFirestore() {
  FirebaseFirestore.instance.settings = const Settings(
    persistenceEnabled: true,
    cacheSizeBytes: 100 * 1024 * 1024, // 100 MB
  );
}

// lib/core/network/network_info.dart
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:injectable/injectable.dart';

abstract interface class NetworkInfo {
  Future<bool> get isConnected;
  Stream<bool> get onStatusChange;
}

@LazySingleton(as: NetworkInfo)
class NetworkInfoImpl implements NetworkInfo {
  NetworkInfoImpl(this._connectivity);
  final Connectivity _connectivity;

  @override
  Future<bool> get isConnected async => _hasNetwork(await _connectivity.checkConnectivity());

  @override
  Stream<bool> get onStatusChange => _connectivity.onConnectivityChanged.map(_hasNetwork).distinct();

  bool _hasNetwork(List<ConnectivityResult> results) =>
      results.any((r) => r != ConnectivityResult.none);
}

// lib/core/network/connectivity_cubit.dart
@lazySingleton
class ConnectivityCubit extends Cubit<bool> {
  ConnectivityCubit(this._network) : super(true) {
    _sub = _network.onStatusChange.listen(emit);
    unawaited(_network.isConnected.then((online) {
      if (!isClosed) emit(online);
    }));
  }
  final NetworkInfo _network;
  late final StreamSubscription<bool> _sub;

  @override
  Future<void> close() async {
    await _sub.cancel();
    return super.close();
  }
}

// lib/core/widgets/offline_banner.dart
class OfflineBanner extends StatelessWidget {
  const OfflineBanner({super.key});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final l = AppLocalizations.of(context);
    return BlocBuilder<ConnectivityCubit, bool>(
      builder: (context, online) => AnimatedSize(
        duration: const Duration(milliseconds: 250),
        child: online
            ? const SizedBox(width: double.infinity)
            : Material(
                color: cs.tertiaryContainer,
                child: SafeArea(
                  bottom: false,
                  child: Padding(
                    padding: const EdgeInsetsDirectional.symmetric(horizontal: 16, vertical: 8),
                    child: Row(children: [
                      Icon(Icons.cloud_off_rounded, size: 18, color: cs.onTertiaryContainer),
                      const SizedBox(width: 8),
                      Expanded(
                        child: Text(l.offlineBanner, style: TextStyle(color: cs.onTertiaryContainer)),
                      ),
                    ]),
                  ),
                ),
              ),
      ),
    );
  }
}
```

## 5. Safe Model, Offline Datasource & Repository Impl
```dart
// features/boards/data/models/board_model.dart
class BoardModel {
  const BoardModel({
    required this.id,
    required this.title,
    required this.color,
    required this.ownerId,
    required this.memberIds,
    required this.updatedAt,
    this.createdAt,
    this.hasPendingWrites = false,
  });

  final String id, title, color, ownerId;
  final List<String> memberIds;
  final DateTime? createdAt;
  final DateTime updatedAt;
  final bool hasPendingWrites;

  factory BoardModel.fromFirestore(DocumentSnapshot<Map<String, dynamic>> doc) {
    final data = doc.data() ?? <String, dynamic>{};
    return BoardModel(
      id: doc.id,
      title: (data['title'] as String?) ?? '',
      color: (data['color'] as String?) ?? '#6366F1',
      ownerId: (data['ownerId'] as String?) ?? '',
      memberIds: List<String>.from((data['memberIds'] as List?) ?? const []),
      createdAt: (data['createdAt'] as Timestamp?)?.toDate(),
      updatedAt: (data['updatedAt'] as Timestamp?)?.toDate() ?? DateTime.now(),
      hasPendingWrites: doc.metadata.hasPendingWrites,
    );
  }

  Map<String, dynamic> toMap() => {
        'title': title,
        'color': color,
        'ownerId': ownerId,
        'memberIds': memberIds,
        'updatedAt': FieldValue.serverTimestamp(),
      };

  BoardEntity toEntity() => BoardEntity(
        id: id,
        title: title,
        color: color,
        ownerId: ownerId,
        memberIds: memberIds,
        createdAt: createdAt,
        updatedAt: updatedAt,
        hasPendingWrites: hasPendingWrites,
      );
}

// features/boards/data/datasources/boards_remote_datasource.dart
abstract interface class BoardsRemoteDatasource {
  Stream<List<BoardModel>> watchBoards(String userId);
  Future<BoardModel> createBoard(CreateBoardParams params);
  Future<void> updateBoard(String boardId, Map<String, dynamic> data);
  Future<void> deleteBoard(String boardId);
}

@LazySingleton(as: BoardsRemoteDatasource)
class BoardsRemoteDatasourceImpl implements BoardsRemoteDatasource {
  const BoardsRemoteDatasourceImpl(this._firestore);
  final FirebaseFirestore _firestore;

  CollectionReference<Map<String, dynamic>> get _raw => _firestore.collection('boards');
  CollectionReference<BoardModel> get _boards => _raw.withConverter<BoardModel>(
        fromFirestore: (snap, _) => BoardModel.fromFirestore(snap),
        toFirestore: (model, _) => model.toMap(),
      );

  @override
  Stream<List<BoardModel>> watchBoards(String userId) => _boards
      .where('memberIds', arrayContains: userId)
      .orderBy('updatedAt', descending: true)
      .limit(50)
      .snapshots(includeMetadataChanges: true)
      .map((s) => s.docs.map((d) => d.data()).toList());

  @override
  Future<BoardModel> createBoard(CreateBoardParams params) async {
    final ref = _raw.doc();
    final model = BoardModel(
      id: ref.id,
      title: params.title,
      color: params.color,
      ownerId: params.userId,
      memberIds: [params.userId],
      updatedAt: DateTime.now(),
      hasPendingWrites: true,
    );
    _fireAndLog(
      ref.set({...model.toMap(), 'createdAt': FieldValue.serverTimestamp()}),
      'createBoard',
    );
    return model;
  }

  @override
  Future<void> updateBoard(String boardId, Map<String, dynamic> data) async {
    _fireAndLog(
      _raw.doc(boardId).update({...data, 'updatedAt': FieldValue.serverTimestamp()}),
      'updateBoard',
    );
  }

  @override
  Future<void> deleteBoard(String boardId) async {
    final boardRef = _raw.doc(boardId);
    final cards = await boardRef.collection('cards').get();
    final refs = [...cards.docs.map((d) => d.reference), boardRef];
    const chunk = 400;
    for (var i = 0; i < refs.length; i += chunk) {
      final batch = _firestore.batch();
      for (final ref in refs.skip(i).take(chunk)) {
        batch.delete(ref);
      }
      _fireAndLog(batch.commit(), 'deleteBoard');
    }
  }

  void _fireAndLog(Future<void> write, String op) {
    unawaited(write.catchError((Object e, StackTrace s) {
      AppLogger.error('$op rejected by server', error: e, stack: s);
    }));
  }
}

// features/boards/data/repositories/boards_repository_impl.dart
@LazySingleton(as: BoardsRepository)
class BoardsRepositoryImpl implements BoardsRepository {
  const BoardsRepositoryImpl(this._remote);
  final BoardsRemoteDatasource _remote;

  @override
  Stream<Either<AppFailure, List<BoardEntity>>> watchBoards(String userId) async* {
    try {
      yield* _remote.watchBoards(userId).map<Either<AppFailure, List<BoardEntity>>>(
            (models) => Right(models.map((m) => m.toEntity()).toList()),
          );
    } on FirebaseException catch (e) {
      yield Left(e.toFailure());
    }
  }

  @override
  Future<Either<AppFailure, BoardEntity>> createBoard(CreateBoardParams params) async {
    try {
      final model = await _remote.createBoard(params);
      return Right(model.toEntity());
    } on FirebaseException catch (e) {
      return Left(e.toFailure());
    } catch (e, s) {
      AppLogger.error('createBoard failed', error: e, stack: s);
      return Left(UnknownFailure(e.toString()));
    }
  }

  @override
  Future<Either<AppFailure, Unit>> deleteBoard(String boardId) async {
    try {
      await _remote.deleteBoard(boardId);
      return const Right(unit);
    } on FirebaseException catch (e) {
      return Left(e.toFailure());
    } catch (e, s) {
      AppLogger.error('deleteBoard failed', error: e, stack: s);
      return Left(UnknownFailure(e.toString()));
    }
  }
}
```

## 6. Cursor Pagination, Storage Service, DraftStore, Fractional Ordering & Sign-Out Wipe
```dart
// Cursor pagination
class PaginatedResult<T> {
  const PaginatedResult({required this.items, required this.hasMore, this.cursor});
  final List<T> items;
  final bool hasMore;
  final Object? cursor;
}

Future<PaginatedResult<ActivityModel>> fetchActivityPage({
  required String boardId,
  Object? cursor,
  int pageSize = 25,
}) async {
  var q = _firestore
      .collection('boards').doc(boardId).collection('activity')
      .orderBy('createdAt', descending: true)
      .limit(pageSize);
  if (cursor is DocumentSnapshot) q = q.startAfterDocument(cursor);
  final snap = await q.get();
  return PaginatedResult(
    items: snap.docs.map(ActivityModel.fromFirestore).toList(),
    hasMore: snap.docs.length == pageSize,
    cursor: snap.docs.isEmpty ? cursor : snap.docs.last,
  );
}

// lib/core/storage/storage_service.dart
@lazySingleton
class StorageService {
  StorageService(this._storage, this._perf);
  final FirebaseStorage _storage;
  final PerfService _perf;

  Future<String> uploadBytes({
    required String path,
    required Uint8List bytes,
    required String contentType,
    void Function(double progress)? onProgress,
  }) =>
      _perf.trace('storage_upload', () async {
        final ref = _storage.ref(path);
        final task = ref.putData(bytes, SettableMetadata(contentType: contentType));
        final sub = task.snapshotEvents.listen((s) {
          if (s.totalBytes > 0) onProgress?.call(s.bytesTransferred / s.totalBytes);
        });
        try {
          await task;
          return await ref.getDownloadURL();
        } finally {
          await sub.cancel();
        }
      }, attributes: {'size_bucket': bytes.length < 1048576 ? '<1MB' : '>=1MB'});

  Future<void> delete(String path) => _storage.ref(path).delete();
}

// lib/core/storage/draft_store.dart (non-sensitive UI drafts only; use FlutterSecureStorage for secrets)
@lazySingleton
class DraftStore {
  DraftStore(this._prefs);
  final SharedPreferences _prefs;
  String? read(String cardId) => _prefs.getString('draft_$cardId');
  Future<void> save(String cardId, String text) => _prefs.setString('draft_$cardId', text);
  Future<void> clear(String cardId) => _prefs.remove('draft_$cardId');
}

// Fractional ordering (drag-and-drop updates ONE document)
double positionBetween(double? before, double? after) {
  if (before == null && after == null) return 1000;
  if (before == null) return after! - 1000;
  if (after == null) return before + 1000;
  return (before + after) / 2;
}

// Sign-out local persistence wipe
Future<void> signOutAndWipeCache(FirebaseAuth auth) async {
  await auth.signOut();
  await FirebaseFirestore.instance.terminate();
  await FirebaseFirestore.instance.clearPersistence();
}
```
