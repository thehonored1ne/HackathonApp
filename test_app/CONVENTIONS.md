# CONVENTIONS.md — Coding Patterns & Agent Rules

> Read this file when: writing BLoC, Firebase datasources, navigation, or any feature code.

---

## BLoC Conventions

Each feature has exactly one BLoC (or more if the feature is large), always inside `presentation/bloc/`.

### File Structure per BLoC

```
presentation/bloc/
├── <feature>_bloc.dart
├── <feature>_event.dart
└── <feature>_state.dart
```

### State Structure

Always cover these states at minimum:

```dart
abstract class AuthState extends Equatable {}

class AuthInitial extends AuthState {
  @override List<Object?> get props => [];
}
class AuthLoading extends AuthState {
  @override List<Object?> get props => [];
}
class AuthAuthenticated extends AuthState {
  final UserEntity user;
  const AuthAuthenticated(this.user);
  @override List<Object?> get props => [user];
}
class AuthUnauthenticated extends AuthState {
  @override List<Object?> get props => [];
}
class AuthError extends AuthState {
  final String message;
  const AuthError(this.message);
  @override List<Object?> get props => [message];
}
```

### BLoC Rules

- BLoC only calls **use cases** — never calls Firebase or repositories directly
- BLoC never imports `firebase_auth`, `cloud_firestore`, or any `data/` layer class
- Never use `BuildContext` inside a BLoC
- Use `BlocProvider` at the page level, not at the app root (unless truly global e.g. auth)
- Use the right consumer:
  - `BlocBuilder` → rebuild UI based on state
  - `BlocListener` → side effects (navigation, snackbars, dialogs)
  - `BlocConsumer` → both at once

---

## Loading & Error UI Standard

Every `BlocBuilder` or `BlocConsumer` must handle all states — no exceptions:

- ❌ Never leave a state unhandled with just an empty `Container()`
- ✅ Always handle at minimum: `initial`, `loading`, `error`, and `success`

```dart
BlocBuilder<AuthBloc, AuthState>(
  builder: (context, state) {
    if (state is AuthInitial) return const SizedBox.shrink();
    if (state is AuthLoading) return const CircularProgressIndicator();
    if (state is AuthError) return Text(state.message);
    if (state is AuthAuthenticated) return const HomeView();
    return const SizedBox.shrink(); // fallback
  },
)
```

---

## Firebase Conventions

- Firebase is initialized in `main.dart` via `firebase_options.dart` — do not re-initialize anywhere else
- All Firebase calls live in `data/datasources/` — this is the **only** place allowed to touch `FirebaseAuth` or `FirebaseFirestore`
- Firestore document IDs must match the entity's `id` field
- Always wrap Firebase calls in try/catch and throw a typed exception, never a raw string

```dart
// features/auth/data/datasources/auth_remote_datasource.dart
class AuthRemoteDataSource {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  Future<UserModel> signInWithEmail(String email, String password) async {
    try {
      final credential = await _auth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
      return UserModel.fromFirebase(credential.user!);
    } on FirebaseAuthException catch (e) {
      debugPrint('Firebase Auth error: ${e.message}');
      throw ServerException(e.message ?? 'Authentication failed');
    }
  }
}
```

### Firestore Model Pattern

```dart
// features/auth/data/models/user_model.dart
class UserModel extends UserEntity {
  const UserModel({required super.id, required super.name, required super.email});

  factory UserModel.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return UserModel(
      id: doc.id,
      name: data['name'] ?? '',
      email: data['email'] ?? '',
    );
  }

  Map<String, dynamic> toFirestore() => {
    'name': name,
    'email': email,
  };
}
```

---

## Firestore Index Rule

Whenever you write a Firestore query that uses **multiple `where` clauses**, an **`orderBy`**, or a **combination of `where` + `orderBy`**, you **must** include a Firestore index notice after the code block in this exact format:

```
⚠️ Firestore Index Required
Collection : <collection_name>
Fields     : <field_1> (asc/desc), <field_2> (asc/desc), ...
How to add : Firebase Console → Firestore → Indexes → Add Index
             OR click the auto-generated link in the Flutter debug console when the query fails.
```

- If the query is simple (single `where` on one field, no `orderBy`) — no index notice needed
- Never assume the index already exists — always show the notice when in doubt
- Always `debugPrint` the raw `FirebaseException` message **before** rethrowing so the index URL is never swallowed:

```dart
} on FirebaseException catch (e) {
  debugPrint('Firestore error: ${e.message}'); // surfaces the index URL in debug console
  throw ServerException(e.message ?? 'Firestore error');
}
```

> ⚠️ The index URL only appears in **debug mode** against **real Firestore** (not the emulator).
> If the URL still doesn't show, check that the query has actually executed and the BLoC event has been triggered.

---

## Navigation with go_router

All routes are defined in `core/router/app_router.dart`. No `Navigator.push` anywhere in the app.

```dart
// core/router/app_router.dart
final appRouter = GoRouter(
  initialLocation: '/login',
  redirect: (context, state) {
    // Check auth state here if needed
    return null;
  },
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => const LoginPage(),
    ),
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomePage(),
    ),
  ],
);
```

### Navigation Rules

- All route paths are defined as constants — no hardcoded strings in widgets
- Navigate using `context.go('/path')` for replacing stack, `context.push('/path')` for pushing
- Pass data via `extra` or query params — never via global state just for navigation
- Auth redirects are handled inside `GoRouter`'s `redirect` callback, not in widgets

---

## Error Handling

```
core/errors/
├── exceptions.dart   # ServerException, CacheException, etc.
└── failures.dart     # ServerFailure, CacheFailure (extend Failure base class)
```

- `datasources/` throw **Exceptions**
- `repositories/` catch Exceptions and return `Either<Failure, T>` (using `dartz`)
- `BLoC` maps `Left(failure)` → error state, `Right(value)` → success state
- Never let a raw Firebase exception bubble up past the `data/` layer

---

## Agent Rules

### Response Rules

- Always state the full file path before generating any code block
  - Example: `// features/auth/data/datasources/auth_remote_datasource.dart`
- Always identify which layer a file belongs to before writing it (data / domain / presentation / core)
- If unsure where something belongs, ask — do not guess and generate
- When generating a full layer (e.g., "implement auth data layer"), generate all files for that layer together
- But never generate files spanning multiple layers unless explicitly asked
- When generating a BLoC, always generate all three files together: `_bloc.dart`, `_event.dart`, `_state.dart`

### Hard Constraints

- ❌ Never suggest changing or simplifying the architecture to "save time"
- ❌ Never merge BLoC event, state, and bloc classes into a single file
- ❌ Never use `Navigator.push` or `Navigator.pop` — always use `context.go()` or `context.push()`
- ❌ Never call Firebase from a BLoC, page, or widget — datasources only
- ❌ Never skip the domain layer (no datasource → BLoC directly)
- ❌ Never create a file in `core/` for something only one feature uses
- ❌ Never use `setState` for feature-level logic — use BLoC
- ❌ Never introduce a package or pattern not in the defined stack (no Provider, no GetX, no Riverpod)
- ❌ Never define a route outside `core/router/app_router.dart`
- ❌ Never import one feature from another feature

### Hackathon Mode — Scope Control

Hackathon mode is active. On every first pass, follow these rules unless explicitly told otherwise:

- **One happy path only** — implement the working flow first, edge cases later
- **No tests** — skip unit/widget test generation unless explicitly requested
- **No caching, pagination, or retry logic** — unless the feature directly requires it to function
- **Generic error messages are acceptable** — `'Something went wrong'` is fine for a demo
- **Skip input validation on the first pass** — add it only after the core feature works end-to-end
- **No over-abstraction** — do not generate factories, strategies, or extra base classes beyond what the architecture already requires
- A feature is **done** when it works end-to-end, not when it's perfect
- If asked to do something that violates these rules, explain why and offer a compliant alternative instead