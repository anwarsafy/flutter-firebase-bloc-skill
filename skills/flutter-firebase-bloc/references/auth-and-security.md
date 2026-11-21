# Authentication & Hardened Security (Complete Code)

## 1. `UserEntity`, `UserModel` & `AuthRepository`
```dart
// features/auth/domain/entities/user_entity.dart
class UserEntity extends Equatable {
  const UserEntity({
    required this.id,
    this.name,
    this.email,
    this.phone,
    this.photoUrl,
    this.username,
    this.isPro = false,
  });

  final String id;
  final String? name, email, phone, photoUrl, username;
  final bool isPro;

  bool get needsOnboarding => username == null || username!.isEmpty;

  @override
  List<Object?> get props => [id, name, email, phone, photoUrl, username, isPro];
}

// features/auth/data/models/user_model.dart
class UserModel {
  const UserModel({
    required this.id,
    this.name,
    this.email,
    this.phone,
    this.photoUrl,
    this.username,
    this.isPro = false,
  });

  final String id;
  final String? name, email, phone, photoUrl, username;
  final bool isPro;

  factory UserModel.fromFirestore(DocumentSnapshot<Map<String, dynamic>> doc) {
    final d = doc.data() ?? <String, dynamic>{};
    return UserModel(
      id: doc.id,
      name: d['name'] as String?,
      email: d['email'] as String?,
      phone: d['phone'] as String?,
      photoUrl: d['photoUrl'] as String?,
      username: d['username'] as String?,
      isPro: (d['isPro'] as bool?) ?? false,
    );
  }

  factory UserModel.fromFirebaseUser(User u) => UserModel(
        id: u.uid,
        name: u.displayName,
        email: u.email,
        phone: u.phoneNumber,
        photoUrl: u.photoURL,
      );

  UserEntity toEntity() => UserEntity(
        id: id,
        name: name,
        email: email,
        phone: phone,
        photoUrl: photoUrl,
        username: username,
        isPro: isPro,
      );
}
```

## 2. Complete `AuthRemoteDatasourceImpl` (All Auth Methods + Delete Account)
```dart
// imports: package:rxdart/rxdart.dart (switchMap, onErrorReturnWith),
// package:google_sign_in/google_sign_in.dart (v7 API), package:shared_preferences/shared_preferences.dart
@LazySingleton(as: AuthRemoteDatasource)
class AuthRemoteDatasourceImpl implements AuthRemoteDatasource {
  AuthRemoteDatasourceImpl(this._auth, this._firestore, this._prefs);
  final FirebaseAuth _auth;
  final FirebaseFirestore _firestore;
  final SharedPreferences _prefs;
  Future<void>? _googleInit;

  /// `switchMap` (NOT `asyncExpand`): `asyncExpand` waits for the inner
  /// `snapshots()` stream to finish before handling the next auth event, and
  /// that stream never finishes — so sign-out (`null`) would never arrive.
  /// `switchMap` cancels the previous user's listener on every auth change.
  @override
  Stream<UserModel?> watchAuthState() => _auth.authStateChanges().switchMap((user) {
        if (user == null) return Stream<UserModel?>.value(null);
        return _firestore
            .collection('users')
            .doc(user.uid)
            .snapshots()
            .map<UserModel?>(
              (doc) => doc.exists ? UserModel.fromFirestore(doc) : UserModel.fromFirebaseUser(user),
            )
            // Profile read failed (e.g. rules, offline first launch): stay signed in with Auth data.
            .onErrorReturnWith((Object e, StackTrace s) {
              AppLogger.error('watch user profile failed', error: e, stack: s);
              return UserModel.fromFirebaseUser(user);
            });
      });

  // google_sign_in 7.x: initialize() exactly once before any other call.
  Future<void> _ensureGoogleInit() => _googleInit ??= GoogleSignIn.instance.initialize(
        serverClientId: AppConfig.instance.googleServerClientId, // null on Android = read from google-services.json
      );

  @override
  Future<UserModel> signInWithGoogle() async {
    if (kIsWeb) {
      // GoogleSignIn.authenticate() is not supported on web; use the Firebase popup flow.
      final cred = await _auth.signInWithPopup(GoogleAuthProvider());
      return _upsertUser(cred.user!);
    }
    await _ensureGoogleInit();
    final GoogleSignInAccount account;
    try {
      account = await GoogleSignIn.instance.authenticate(scopeHint: const ['email']);
    } on GoogleSignInException catch (e) {
      if (e.code == GoogleSignInExceptionCode.canceled) throw const CancelledException();
      rethrow;
    }
    final idToken = account.authentication.idToken; // v7: ID token only, no access token
    final cred = await _auth.signInWithCredential(GoogleAuthProvider.credential(idToken: idToken));
    return _upsertUser(cred.user!);
  }

  @override
  Future<UserModel> signInWithApple() async {
    final rawNonce = generateNonce();
    final sha256Nonce = sha256.convert(utf8.encode(rawNonce)).toString();
    final AuthorizationCredentialAppleID appleCred;
    try {
      appleCred = await SignInWithApple.getAppleIDCredential(
        scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
        nonce: sha256Nonce,
      );
    } on SignInWithAppleAuthorizationException catch (e) {
      if (e.code == AuthorizationErrorCode.canceled) throw const CancelledException();
      rethrow;
    }
    final cred = await _auth.signInWithCredential(
      OAuthProvider('apple.com').credential(idToken: appleCred.identityToken, rawNonce: rawNonce),
    );
    final name = [appleCred.givenName, appleCred.familyName].whereType<String>().join(' ').trim();
    return _upsertUser(cred.user!, displayName: name.isEmpty ? null : name);
  }

  @override
  Future<String> sendOtp(String phoneNumber) {
    final completer = Completer<String>();
    unawaited(_auth.verifyPhoneNumber(
      phoneNumber: phoneNumber,
      timeout: const Duration(seconds: 60),
      verificationCompleted: (_) {},
      verificationFailed: (e) {
        if (!completer.isCompleted) completer.completeError(e);
      },
      codeSent: (id, _) {
        if (!completer.isCompleted) completer.complete(id);
      },
      codeAutoRetrievalTimeout: (_) {},
    ));
    return completer.future;
  }

  @override
  Future<UserModel> verifyOtp({required String verificationId, required String otp}) async {
    final cred = await _auth.signInWithCredential(
      PhoneAuthProvider.credential(verificationId: verificationId, smsCode: otp.trim()),
    );
    return _upsertUser(cred.user!);
  }

  @override
  Future<UserModel> signInWithEmail({required String email, required String password}) async {
    final cred = await _auth.signInWithEmailAndPassword(email: email.trim(), password: password);
    return _upsertUser(cred.user!);
  }

  @override
  Future<UserModel> signUpWithEmail({required String email, required String password}) async {
    final cred = await _auth.createUserWithEmailAndPassword(email: email.trim(), password: password);
    return _upsertUser(cred.user!);
  }

  @override
  Future<void> sendPasswordReset(String email) =>
      _auth.sendPasswordResetEmail(email: email.trim());

  /// Throws FirebaseAuthException('requires-recent-login') if the session is old:
  /// the UI must re-authenticate (Google/Apple/OTP again) and retry.
  /// Server data is erased by the `cleanupDeletedUser` Cloud Function (notifications-and-links.md §4).
  @override
  Future<void> deleteAccount() async {
    final user = _auth.currentUser;
    if (user == null) return;
    await user.delete();
    await _afterSessionEnd();
  }

  @override
  Future<void> signOut() async {
    await _auth.signOut();
    await _afterSessionEnd();
  }

  Future<void> _afterSessionEnd() async {
    // Never `terminate()` here: GetIt and every datasource hold this Firestore instance.
    await _prefs.setBool(kPendingCacheWipeKey, true); // cleared on next cold start
    if (!kIsWeb && _googleInit != null) {
      await GoogleSignIn.instance.signOut().catchError((Object _) {});
    }
  }

  Future<UserModel> _upsertUser(User user, {String? displayName}) async {
    final ref = _firestore.collection('users').doc(user.uid);
    // Note: Never send `isPro`, `role`, or `boardCount` from client — Security Rules block them.
    await ref.set({
      'id': user.uid,
      'name': displayName ?? user.displayName,
      'email': user.email,
      'phone': user.phoneNumber,
      'photoUrl': user.photoURL,
      'lastSignIn': FieldValue.serverTimestamp(),
    }, SetOptions(merge: true));
    return UserModel.fromFirestore(await ref.get());
  }
}
```

## 2b. `AuthRepositoryImpl` (Stream Errors Mapped, Network-Gated Delete)
```dart
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl(this._remote, this._network);
  final AuthRemoteDatasource _remote;
  final NetworkInfo _network;

  @override
  Stream<Either<AppFailure, UserEntity?>> watchAuthState() => _remote
      .watchAuthState()
      .map<Either<AppFailure, UserEntity?>>((m) => Right(m?.toEntity()))
      .catchFailures('watchAuthState');

  @override
  Future<Either<AppFailure, UserEntity>> signInWithGoogle() => _guard('signInWithGoogle', _remote.signInWithGoogle);

  @override
  Future<Either<AppFailure, UserEntity>> signInWithApple() => _guard('signInWithApple', _remote.signInWithApple);

  @override
  Future<Either<AppFailure, Unit>> deleteAccount() async {
    if (!await _network.isConnected) return const Left(NetworkFailure()); // Decision Matrix: always gate
    try {
      await _remote.deleteAccount();
      return const Right(unit);
    } catch (e, s) {
      return Left(mapException(e, s, 'deleteAccount')); // 'requires-recent-login' -> AuthFailure
    }
  }

  @override
  Future<Either<AppFailure, Unit>> signOut() async {
    try {
      await _remote.signOut();
      return const Right(unit);
    } catch (e, s) {
      return Left(mapException(e, s, 'signOut'));
    }
  }

  Future<Either<AppFailure, UserEntity>> _guard(String op, Future<UserModel> Function() call) async {
    try {
      return Right((await call()).toEntity());
    } catch (e, s) {
      return Left(mapException(e, s, op)); // CancelledException -> CancelledFailure (no SnackBar)
    }
  }
}
```

## 3. Complete `AuthBloc` (`AuthStatus.unknown` Splash Guard)
```dart
enum AuthStatus { unknown, authenticated, unauthenticated, loading, failure }

class AuthState extends Equatable {
  const AuthState({
    this.status = AuthStatus.unknown,
    this.user,
    this.verificationId,
    this.failure,
  });

  final AuthStatus status;
  final UserEntity? user;
  final String? verificationId;
  final AppFailure? failure;

  AuthState copyWith({
    AuthStatus? status,
    UserEntity? Function()? user,
    String? verificationId,
    AppFailure? Function()? failure,
  }) =>
      AuthState(
        status: status ?? this.status,
        user: user != null ? user() : this.user,
        verificationId: verificationId ?? this.verificationId,
        failure: failure != null ? failure() : this.failure,
      );

  @override
  List<Object?> get props => [status, user, verificationId, failure];
}

sealed class AuthEvent extends Equatable {
  const AuthEvent();
  @override
  List<Object?> get props => [];
}

class AuthWatchStarted extends AuthEvent { const AuthWatchStarted(); }
class GoogleSignInRequested extends AuthEvent { const GoogleSignInRequested(); }
class AppleSignInRequested extends AuthEvent { const AppleSignInRequested(); }
class SignOutRequested extends AuthEvent { const SignOutRequested(); }
class DeleteAccountRequested extends AuthEvent { const DeleteAccountRequested(); }

@lazySingleton
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc(this._repo, this._fcm, this._reporter) : super(const AuthState()) {
    on<AuthWatchStarted>(_onWatch);
    on<GoogleSignInRequested>(_onGoogle);
    on<AppleSignInRequested>(_onApple);
    on<SignOutRequested>(_onSignOut);
    on<DeleteAccountRequested>(_onDeleteAccount);
  }
  final AuthRepository _repo;
  final FcmService _fcm;
  final ErrorReporter _reporter;

  Future<void> _onWatch(AuthWatchStarted _, Emitter<AuthState> emit) => emit.forEach(
        _repo.watchAuthState(), // Stream<Either<AppFailure, UserEntity?>> mapped with catchFailures
        onData: (result) => result.fold(
          (f) => state.copyWith(status: AuthStatus.failure, failure: () => f),
          (user) {
            unawaited(_reporter.setUser(user?.id));
            if (user != null) unawaited(_fcm.initialize(user.id)); // idempotent per uid
            return user == null
                ? const AuthState(status: AuthStatus.unauthenticated)
                : AuthState(status: AuthStatus.authenticated, user: user);
          },
        ),
        onError: (e, s) => state.copyWith(
          status: AuthStatus.failure,
          failure: () => const UnknownFailure('auth-stream'),
        ),
      );

  Future<void> _onGoogle(GoogleSignInRequested _, Emitter<AuthState> emit) async {
    emit(state.copyWith(status: AuthStatus.loading, failure: () => null));
    final res = await _repo.signInWithGoogle();
    res.fold(
      (f) => emit(state.copyWith(status: AuthStatus.failure, failure: () => f)),
      (_) {},
    );
  }

  Future<void> _onApple(AppleSignInRequested _, Emitter<AuthState> emit) async {
    emit(state.copyWith(status: AuthStatus.loading, failure: () => null));
    final res = await _repo.signInWithApple();
    res.fold(
      (f) => emit(state.copyWith(status: AuthStatus.failure, failure: () => f)),
      (_) {},
    );
  }

  Future<void> _onSignOut(SignOutRequested _, Emitter<AuthState> emit) async {
    final uid = state.user?.id;
    if (uid != null) await _fcm.removeToken(uid); // needs auth: run BEFORE signOut
    _fcm.dispose();
    await _repo.signOut(); // auth stream emits null -> unauthenticated
  }

  Future<void> _onDeleteAccount(DeleteAccountRequested _, Emitter<AuthState> emit) async {
    // `user` is kept in state, so the router keeps the user on the current screen.
    emit(state.copyWith(status: AuthStatus.loading, failure: () => null));
    final res = await _repo.deleteAccount(); // requires network: repository gates on NetworkInfo
    await res.fold(
      // e.g. AuthFailure('requires-recent-login') -> UI re-authenticates and retries
      (f) async => emit(state.copyWith(status: AuthStatus.failure, failure: () => f)),
      (_) async {
        _fcm.dispose();
        await _fcm.deleteLocalToken(); // user doc + tokens are erased server-side
      },
    );
  }
}
```

## 4. App Check (`lib/core/firebase/app_check_setup.dart`)
```dart
// firebase_app_check 0.5.x: provider objects (the old androidProvider/appleProvider/
// webProvider enum parameters were removed in 0.5.0).
Future<void> activateAppCheck() {
  final dev = AppConfig.instance.isDev;
  return FirebaseAppCheck.instance.activate(
    providerAndroid: dev ? AndroidDebugProvider() : AndroidPlayIntegrityProvider(),
    providerApple: dev ? AppleDebugProvider() : AppleAppAttestWithDeviceCheckFallbackProvider(),
    providerWeb: ReCaptchaEnterpriseProvider(AppConfig.instance.recaptchaSiteKey),
  );
}
```

## 5. Hardened Firestore & Storage Rules (`firebase/firestore.rules`, `firebase/storage.rules`)
```javascript
// firebase/firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() { return request.auth != null; }
    function isOwner(userId) { return isSignedIn() && request.auth.uid == userId; }
    function isMember(data) { return isSignedIn() && request.auth.uid in data.memberIds; }
    function boardData(boardId) { return get(/databases/$(database)/documents/boards/$(boardId)).data; }
    function incoming() { return request.resource.data; }
    function changed() { return incoming().diff(resource.data).affectedKeys(); }
    function unchanged(fields) { return !changed().hasAny(fields); }
    function onlyChanges(fields) { return changed().hasOnly(fields); }
    function optString(d, k, max) { return !(k in d) || d[k] == null || (d[k] is string && d[k].size() <= max); }
    function stampedNow(field) { return incoming()[field] == request.time; } // FieldValue.serverTimestamp()

    // ---- users/{uid}: an ALLOW-LIST of client fields. Everything else (isPro, role, plan,
    // boardCount, credits, ...) can only be written by the Admin SDK / Cloud Functions.
    function userClientFields() {
      return ['id', 'name', 'email', 'phone', 'photoUrl', 'username', 'lastSignIn',
              'fcmTokens', 'notificationsEnabled', 'lastTokenUpdatedAt'];
    }
    function validUser(d) {
      return optString(d, 'name', 100) && optString(d, 'email', 254) && optString(d, 'phone', 20)
        && optString(d, 'photoUrl', 2048) && optString(d, 'username', 30)
        && (!('fcmTokens' in d) || (d.fcmTokens is list && d.fcmTokens.size() <= 10))
        && (!('notificationsEnabled' in d) || d.notificationsEnabled is bool);
    }
    match /users/{userId} {
      allow read: if isOwner(userId);
      allow create: if isOwner(userId) && incoming().id == userId
        && incoming().keys().hasOnly(userClientFields()) && validUser(incoming());
      // affectedKeys().hasOnly(): server-owned fields already in the doc stay untouched.
      allow update: if isOwner(userId) && onlyChanges(userClientFields())
        && unchanged(['id']) && validUser(incoming());
      allow delete: if false; // `cleanupDeletedUser` Cloud Function only
    }

    match /publicProfiles/{userId} {
      allow get: if isSignedIn();
      allow list: if false; // anti-scraping
      allow create, update: if isOwner(userId) && incoming().keys().hasOnly(['name', 'photoUrl', 'username'])
        && optString(incoming(), 'name', 100) && optString(incoming(), 'photoUrl', 2048)
        && optString(incoming(), 'username', 30);
      allow delete: if false;
    }

    function validBoard(d) {
      return d.keys().hasOnly(['title', 'color', 'ownerId', 'memberIds', 'createdAt', 'updatedAt'])
        && d.title is string && d.title.size() >= 1 && d.title.size() <= 120
        && d.color is string && d.color.size() <= 16
        && d.memberIds is list && d.memberIds.size() <= 50 && d.ownerId in d.memberIds
        && d.createdAt is timestamp && d.updatedAt is timestamp;
    }
    match /boards/{boardId} {
      allow read: if isMember(resource.data);
      allow create: if isSignedIn() && incoming().ownerId == request.auth.uid
        && incoming().memberIds == [request.auth.uid]
        && stampedNow('createdAt') && stampedNow('updatedAt') && validBoard(incoming());
      allow update: if isMember(resource.data) && unchanged(['ownerId', 'createdAt'])
        && (isOwner(resource.data.ownerId) || unchanged(['memberIds']))
        && stampedNow('updatedAt') && validBoard(incoming());
      allow delete: if false; // `deleteBoardRecursive` callable (Admin SDK) only

      function validCard(d) {
        return d.keys().hasOnly(['title', 'description', 'position', 'assigneeId', 'createdAt', 'updatedAt'])
          && d.title is string && d.title.size() >= 1 && d.title.size() <= 200
          && optString(d, 'description', 5000)
          && d.position is number
          && (!('assigneeId' in d) || d.assigneeId == null || d.assigneeId in boardData(boardId).memberIds);
      }
      match /cards/{cardId} {
        allow read: if isMember(boardData(boardId));
        allow create: if isMember(boardData(boardId)) && stampedNow('createdAt') && stampedNow('updatedAt')
          && validCard(incoming());
        allow update: if isMember(boardData(boardId)) && unchanged(['createdAt']) && stampedNow('updatedAt')
          && validCard(incoming());
        allow delete: if isMember(boardData(boardId));
      }
    }

    match /notifications/{notifId} {
      allow read: if isOwner(resource.data.userId); // list queries must filter where('userId', '==', uid)
      allow update: if isOwner(resource.data.userId) && onlyChanges(['isRead']) && incoming().isRead is bool;
      allow create, delete: if false; // Cloud Functions only
    }
  }
}

// firebase/storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    function isBoardMember(boardId) {
      return request.auth != null
        && firestore.exists(/databases/(default)/documents/boards/$(boardId))
        && request.auth.uid in firestore.get(/databases/(default)/documents/boards/$(boardId)).data.memberIds;
    }
    match /boards/{boardId}/attachments/{fileName} {
      allow read: if isBoardMember(boardId);
      allow create, update: if isBoardMember(boardId)
        && request.resource.size < 10 * 1024 * 1024
        && request.resource.contentType.matches('image/(png|jpeg|webp|gif)|application/pdf');
      allow delete: if isBoardMember(boardId);
    }
    match /users/{userId}/avatar.jpg {
      allow read: if request.auth != null;
      allow create, update: if request.auth != null && request.auth.uid == userId
        && request.resource.size < 5 * 1024 * 1024
        && request.resource.contentType.matches('image/(png|jpeg|webp)');
      allow delete: if request.auth != null && request.auth.uid == userId;
    }
    match /{allPaths=**} { allow read, write: if false; }
  }
}
```

## 6. Security Rules Automated Tests (`firebase/tests/rules.test.ts`)
Run with the emulator: `firebase emulators:exec --only firestore "npx vitest run firebase/tests"`.
Every rule that blocks tampering gets a test that proves it — especially `update` paths, where most privilege escalations happen.
```typescript
import { assertFails, assertSucceeds, initializeTestEnvironment, RulesTestEnvironment } from '@firebase/rules-unit-testing';
import { serverTimestamp, doc, setDoc, updateDoc, getDoc, getDocs, collection, addDoc, deleteDoc } from 'firebase/firestore';
import { readFileSync } from 'node:fs';
import { afterAll, beforeAll, beforeEach, describe, it } from 'vitest';

let env: RulesTestEnvironment;

beforeAll(async () => {
  env = await initializeTestEnvironment({
    projectId: 'demo-rules',
    firestore: { rules: readFileSync('firebase/firestore.rules', 'utf8') },
  });
});
afterAll(() => env.cleanup());
beforeEach(async () => {
  await env.clearFirestore();
  await env.withSecurityRulesDisabled(async (ctx) => {
    const db = ctx.firestore();
    await setDoc(doc(db, 'users/alice'), { id: 'alice', name: 'Alice', isPro: false });
    await setDoc(doc(db, 'boards/b1'), {
      title: 'Board', color: '#6366F1', ownerId: 'alice', memberIds: ['alice', 'bob'],
      createdAt: new Date(), updatedAt: new Date(),
    });
    await setDoc(doc(db, 'notifications/n1'), { userId: 'alice', title: 'Hi', isRead: false });
  });
});

const as = (uid: string) => env.authenticatedContext(uid).firestore();

describe('users/{uid}', () => {
  it('allows creating an own profile with client fields only', async () => {
    await assertSucceeds(setDoc(doc(as('carol'), 'users/carol'), { id: 'carol', name: 'Carol' }));
  });
  it('blocks privileged fields on create', async () => {
    await assertFails(setDoc(doc(as('carol'), 'users/carol'), { id: 'carol', isPro: true }));
    await assertFails(setDoc(doc(as('carol'), 'users/carol'), { id: 'carol', plan: 'enterprise' }));
  });
  it('blocks privilege escalation on update (isPro, role, plan, any unknown field)', async () => {
    const db = as('alice');
    for (const field of ['isPro', 'role', 'plan', 'boardCount', 'anythingElse']) {
      await assertFails(updateDoc(doc(db, 'users/alice'), { [field]: true }));
    }
  });
  it('allows updating client fields', async () => {
    await assertSucceeds(updateDoc(doc(as('alice'), 'users/alice'), { name: 'Alice B.' }));
  });
  it('blocks oversized values and other users', async () => {
    await assertFails(updateDoc(doc(as('alice'), 'users/alice'), { name: 'x'.repeat(101) }));
    await assertFails(getDoc(doc(as('bob'), 'users/alice')));
    await assertFails(deleteDoc(doc(as('alice'), 'users/alice')));
  });
});

describe('publicProfiles', () => {
  it('blocks listing (anti-scraping)', async () => {
    await assertFails(getDocs(collection(as('alice'), 'publicProfiles')));
  });
});

describe('boards', () => {
  const board = (uid: string, memberIds: string[]) => ({
    title: 'New', color: '#000000', ownerId: uid, memberIds,
    createdAt: serverTimestamp(), updatedAt: serverTimestamp(),
  });
  it('allows creating an own board', async () => {
    await assertSucceeds(setDoc(doc(as('alice'), 'boards/b2'), board('alice', ['alice'])));
  });
  it('blocks adding other members or a foreign owner on create', async () => {
    await assertFails(setDoc(doc(as('alice'), 'boards/b3'), board('alice', ['alice', 'mallory'])));
    await assertFails(setDoc(doc(as('mallory'), 'boards/b4'), board('alice', ['alice'])));
  });
  it('blocks non-members and ownership hijacking', async () => {
    await assertFails(getDoc(doc(as('mallory'), 'boards/b1')));
    await assertFails(updateDoc(doc(as('bob'), 'boards/b1'), { ownerId: 'bob', updatedAt: serverTimestamp() }));
    await assertFails(updateDoc(doc(as('bob'), 'boards/b1'), { memberIds: ['alice', 'bob', 'mallory'], updatedAt: serverTimestamp() }));
  });
  it('blocks client-side board deletes (callable only)', async () => {
    await assertFails(deleteDoc(doc(as('alice'), 'boards/b1')));
  });
});

describe('notifications', () => {
  it('blocks forged notifications and allows marking own as read', async () => {
    await assertFails(addDoc(collection(as('alice'), 'notifications'), { userId: 'alice', title: 'fake' }));
    await assertFails(updateDoc(doc(as('alice'), 'notifications/n1'), { title: 'edited' }));
    await assertSucceeds(updateDoc(doc(as('alice'), 'notifications/n1'), { isRead: true }));
  });
});
```
`firebase/tests/package.json` dev dependencies: `@firebase/rules-unit-testing`, `firebase`, `vitest`, `typescript`.

## 7. High-Sensitivity Apps (Fintech / Enterprise)
- Store local sensitive data only via `FlutterSecureStorage` (never `SharedPreferences`).
- Restart the app process right after sign-out so the pending Firestore cache wipe runs immediately (shared devices).
- Add RASP / Root / Jailbreak / Frida detection (`freerasp`) + `FLAG_SECURE` on sensitive screens.
- Enforce **App Check** in Firebase Console on Auth, Firestore, Storage, AND Functions, and enable **Email Enumeration Protection**.
