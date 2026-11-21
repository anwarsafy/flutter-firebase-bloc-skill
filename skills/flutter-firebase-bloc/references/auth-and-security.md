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
@LazySingleton(as: AuthRemoteDatasource)
class AuthRemoteDatasourceImpl implements AuthRemoteDatasource {
  const AuthRemoteDatasourceImpl(this._auth, this._firestore);
  final FirebaseAuth _auth;
  final FirebaseFirestore _firestore;

  @override
  Stream<UserModel?> watchAuthState() => _auth.authStateChanges().asyncExpand((user) {
        if (user == null) return Stream.value(null);
        return _firestore.collection('users').doc(user.uid).snapshots().map(
              (doc) => doc.exists ? UserModel.fromFirestore(doc) : UserModel.fromFirebaseUser(user),
            );
      });

  @override
  Future<UserModel> signInWithGoogle() async {
    final googleUser = await GoogleSignIn().signIn();
    if (googleUser == null) throw const CancelledException();
    final ga = await googleUser.authentication;
    final cred = await _auth.signInWithCredential(
      GoogleAuthProvider.credential(accessToken: ga.accessToken, idToken: ga.idToken),
    );
    return _upsertUser(cred.user!);
  }

  @override
  Future<UserModel> signInWithApple() async {
    final rawNonce = generateNonce();
    final sha256Nonce = sha256.convert(utf8.encode(rawNonce)).toString();
    final appleCred = await SignInWithApple.getAppleIDCredential(
      scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
      nonce: sha256Nonce,
    );
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

  @override
  Future<void> deleteAccount() async {
    final user = _auth.currentUser;
    if (user == null) return;
    // Note: if FirebaseAuthException('requires-recent-login') is thrown, prompt re-auth first.
    await user.delete(); // Cloud Function `onUserDeleted` cleans up server documents.
  }

  @override
  Future<void> signOut() async {
    await GoogleSignIn().signOut();
    await _auth.signOut();
    await _firestore.terminate();
    await _firestore.clearPersistence();
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
        _repo.watchAuthState(),
        onData: (user) {
          unawaited(_reporter.setUser(user?.id));
          if (user != null) unawaited(_fcm.initialize(user.id));
          return user == null
              ? const AuthState(status: AuthStatus.unauthenticated)
              : AuthState(status: AuthStatus.authenticated, user: user);
        },
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
    if (uid != null) await _fcm.removeToken(uid);
    _fcm.dispose();
    await _repo.signOut();
  }

  Future<void> _onDeleteAccount(DeleteAccountRequested _, Emitter<AuthState> emit) async {
    emit(state.copyWith(status: AuthStatus.loading));
    final uid = state.user?.id;
    if (uid != null) await _fcm.removeToken(uid);
    _fcm.dispose();
    final res = await _repo.deleteAccount();
    res.fold(
      (f) => emit(state.copyWith(status: AuthStatus.failure, failure: () => f)),
      (_) {},
    );
  }
}
```

## 4. App Check (`lib/core/firebase/app_check_setup.dart`)
```dart
Future<void> activateAppCheck() => FirebaseAppCheck.instance.activate(
      androidProvider: AppConfig.instance.isDev ? AndroidProvider.debug : AndroidProvider.playIntegrity,
      appleProvider: AppConfig.instance.isDev ? AppleProvider.debug : AppleProvider.appAttest,
      webProvider: ReCaptchaEnterpriseProvider(AppConfig.instance.recaptchaSiteKey),
    );
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
    function unchanged(fields) { return !request.resource.data.diff(resource.data).affectedKeys().hasAny(fields); }
    function onlyChanges(fields) { return request.resource.data.diff(resource.data).affectedKeys().hasOnly(fields); }
    function validBoardFields(d) {
      return d.keys().hasOnly(['title', 'color', 'ownerId', 'memberIds', 'createdAt', 'updatedAt'])
        && d.title is string && d.title.size() >= 1 && d.title.size() <= 120
        && d.color is string && d.color.size() <= 16
        && d.memberIds is list && d.memberIds.size() <= 50 && d.ownerId in d.memberIds;
    }

    match /users/{userId} {
      allow read: if isOwner(userId);
      allow create: if isOwner(userId) && !request.resource.data.keys().hasAny(['isPro', 'role', 'boardCount']);
      allow update: if isOwner(userId) && unchanged(['id', 'isPro', 'role', 'boardCount']);
      allow delete: if false;
    }

    match /publicProfiles/{userId} {
      allow get: if isSignedIn();
      allow list: if false;
      allow write: if isOwner(userId) && request.resource.data.keys().hasOnly(['name', 'photoUrl', 'username']);
    }

    match /boards/{boardId} {
      allow read: if isMember(resource.data);
      allow create: if isSignedIn() && request.resource.data.ownerId == request.auth.uid
        && request.resource.data.memberIds == [request.auth.uid] && validBoardFields(request.resource.data);
      allow update: if isMember(resource.data) && unchanged(['ownerId', 'createdAt'])
        && (isOwner(resource.data.ownerId) || unchanged(['memberIds']))
        && validBoardFields(request.resource.data);
      allow delete: if isOwner(resource.data.ownerId);

      match /cards/{cardId} {
        allow read: if isMember(get(/databases/$(database)/documents/boards/$(boardId)).data);
        allow create, update: if isMember(get(/databases/$(database)/documents/boards/$(boardId)).data)
          && request.resource.data.keys().hasOnly(['title', 'description', 'position', 'assigneeId', 'updatedAt', 'createdAt'])
          && request.resource.data.title is string && request.resource.data.title.size() <= 200;
        allow delete: if isMember(get(/databases/$(database)/documents/boards/$(boardId)).data);
      }
    }

    match /notifications/{notifId} {
      allow read: if isOwner(resource.data.userId);
      allow update: if isOwner(resource.data.userId) && onlyChanges(['isRead']);
      allow create, delete: if false;
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
        && request.resource.contentType.matches('image/.*|application/pdf');
      allow delete: if isBoardMember(boardId);
    }
    match /users/{userId}/avatar.jpg {
      allow read: if request.auth != null;
      allow create, update: if request.auth != null && request.auth.uid == userId
        && request.resource.size < 5 * 1024 * 1024
        && request.resource.contentType.matches('image/.*');
      allow delete: if request.auth != null && request.auth.uid == userId;
    }
    match /{allPaths=**} { allow read, write: if false; }
  }
}
```

## 6. Security Rules Automated Test (`firebase/tests/rules.test.ts`)
```typescript
import { assertFails, assertSucceeds, initializeTestEnvironment } from '@firebase/rules-unit-testing';
import * as fs from 'fs';

describe('Firestore Security Rules', () => {
  it('blocks privilege escalation on users/{uid} (isPro/role)', async () => {
    const env = await initializeTestEnvironment({
      projectId: 'demo-test',
      firestore: { rules: fs.readFileSync('firebase/firestore.rules', 'utf8') },
    });
    const alice = env.authenticatedContext('alice').firestore();
    await assertFails(alice.collection('users').doc('alice').set({ isPro: true }));
    await assertFails(alice.collection('publicProfiles').get()); // Anti-scraping
    await assertFails(alice.collection('notifications').add({ userId: 'alice', title: 'fake' }));
    await assertSucceeds(alice.collection('users').doc('alice').set({ name: 'Alice' }));
    await env.cleanup();
  });
});
```

## 7. High-Sensitivity Apps (Fintech / Enterprise)
- Store local sensitive data only via `FlutterSecureStorage` (never `SharedPreferences`).
- Add RASP / Root / Jailbreak / Frida detection (`freerasp`) + `FLAG_SECURE` on sensitive screens.
- Enforce **App Check** in Firebase Console on Auth, Firestore, Storage, AND Functions, and enable **Email Enumeration Protection**.
