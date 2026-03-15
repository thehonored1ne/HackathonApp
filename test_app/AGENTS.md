# AGENTS.md — Flutter Firebase Hackathon Project

## Stack
- **Flutter** — UI framework
- **Firebase** — Firestore (database) + Auth (authentication)
- **BLoC** — state management (`flutter_bloc`)
- **go_router** — navigation and routing
- **Clean Architecture** — feature-first folder structure

---

## Project Structure

```
lib/
├── core/
│   ├── constants/        # App-wide constants (strings, keys, asset paths)
│   ├── errors/           # Failure and Exception base classes
│   ├── network/          # Network info helpers
│   ├── router/
│   │   └── app_router.dart   # All go_router route definitions live here
│   ├── theme/            # AppTheme, text styles, color scheme
│   ├── usecases/         # ONLY the abstract UseCase base class lives here
│   ├── utils/            # Shared utility/helper functions
│   └── widgets/          # Truly reusable widgets used across 2+ features
├── features/
│   └── <feature_name>/
│       ├── data/
│       │   ├── datasources/   # Firebase calls (Auth, Firestore)
│       │   ├── models/        # Data models with fromFirestore/toFirestore
│       │   └── repositories/  # Repository implementations
│       ├── domain/
│       │   ├── entities/      # Pure Dart classes, no Flutter/Firebase imports
│       │   ├── repositories/  # Abstract repository interfaces
│       │   └── usecases/      # Feature use cases (one class per use case)
│       └── presentation/
│           ├── bloc/          # BLoC, Event, State files
│           ├── pages/         # Full screens
│           └── widgets/       # Widgets used only within this feature
├── firebase_options.dart
└── main.dart
```

---

## Architecture Rules — READ BEFORE WRITING ANY CODE

### Dependency Direction (never break this)
```
presentation → domain ← data
                ↑
              core
```

- `presentation/` depends on `domain/` only
- `data/` depends on `domain/` only
- `domain/` depends on **nothing** — pure Dart, no Flutter, no Firebase
- `core/` is shared infrastructure — no feature imports allowed inside `core/`
- Features are **fully isolated** — one feature must never import from another feature

### Feature-First Rule
Every piece of code belongs to a feature first. Ask: *"which feature owns this?"*
- If it belongs to one feature → put it in `features/<feature>/`
- If it is used by 2 or more features → move it to `core/`
- If you are unsure → default to the feature, promote to `core/` later

### `core/usecases/` Rule
`core/usecases/` contains **only** the abstract base class:
```dart
abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type>> call(Params params);
}

class NoParams extends Equatable {
  @override List<Object?> get props => [];
}
```
All actual use case implementations go in `features/<feature>/domain/usecases/`.

### `core/widgets/` vs feature `widgets/` Rule
- Widget used in **one feature only** → `features/<feature>/presentation/widgets/`
- Widget used in **two or more features** → `core/widgets/`
- Never move a widget to `core/widgets/` preemptively — wait until it's actually reused

---

## BLoC Conventions

Each feature has exactly one BLoC (or more if the feature is large), always inside `presentation/bloc/`.

### File structure per BLoC
```
presentation/bloc/
├── <feature>_bloc.dart
├── <feature>_event.dart
└── <feature>_state.dart
```

### State structure
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

### BLoC rules
- BLoC only calls **use cases** — never calls Firebase or repositories directly
- BLoC never imports `firebase_auth`, `cloud_firestore`, or any `data/` layer class
- Never use `BuildContext` inside a BLoC
- Use `BlocProvider` at the page level, not at the app root (unless truly global e.g. auth)
- Use the right consumer:
    - `BlocBuilder` → rebuild UI based on state
    - `BlocListener` → side effects (navigation, snackbars, dialogs)
    - `BlocConsumer` → both at once

---

## Firebase Conventions

- Firebase is initialized in `main.dart` via `firebase_options.dart` — do not re-initialize anywhere else
- All Firebase calls live in `data/datasources/` — this is the **only** place allowed to touch `FirebaseAuth` or `FirebaseFirestore`
- Firestore document IDs must match the entity's `id` field
- Always wrap Firebase calls in try/catch and throw a typed exception, never a raw string

```dart
// data/datasources/auth_remote_datasource.dart
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
      throw ServerException(e.message ?? 'Authentication failed');
    }
  }
}
```

### Firestore model pattern
```dart
// data/models/user_model.dart
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

### Navigation rules
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

## main.dart Responsibilities

`main.dart` should only do these things:
1. `WidgetsFlutterBinding.ensureInitialized()`
2. `await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform)`
3. `runApp(MyApp())`

`MyApp` sets up `MaterialApp.router` with `appRouter`. Global `BlocProvider`s (e.g. `AuthBloc`) go here. Nothing else.

---

## Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Files | `snake_case.dart` | `sign_in_bloc.dart` |
| Classes | `PascalCase` | `SignInBloc` |
| Variables/methods | `camelCase` | `fetchUser()` |
| BLoC events | Past tense verb | `SignInSubmitted`, `UserFetched` |
| BLoC states | Descriptive noun/adjective | `AuthLoading`, `AuthAuthenticated` |
| Firestore collections | `camelCase` plural | `users`, `posts` |
| Route paths | `kebab-case` | `/sign-in`, `/home` |

---

## Adding a New Feature — Step by Step

1. Create `features/<feature_name>/` with all subdirectories
2. **Domain first:** define entity → repository interface → use cases
3. **Data layer:** implement model → datasource → repository
4. **Presentation:** create BLoC (event, state, bloc) → pages → widgets
5. Register the route in `core/router/app_router.dart`
6. Add `BlocProvider` where the BLoC is needed
7. Update the feature table below

---

## What NOT to Do

- ❌ Do not call Firebase from a BLoC, page, or widget
- ❌ Do not put business logic in pages or widgets
- ❌ Do not import one feature from another feature
- ❌ Do not put feature-specific code in `core/`
- ❌ Do not use `Navigator.push` — use `context.go()` or `context.push()`
- ❌ Do not define routes outside `core/router/app_router.dart`
- ❌ Do not use `BuildContext` inside a BLoC
- ❌ Do not create barrel `index.dart` files

---

## Current Features

| Feature | Data | Domain | Presentation | Notes |
|---------|------|--------|--------------|-------|
| `auth` | 🚧 | 🚧 | 🚧 | Firebase Auth |
| `home` | ⬜ | ⬜ | ⬜ | Not started |

> Legend: ✅ Done · 🚧 In progress · ⬜ Not started  
> Update this table as you build.

---

## Agent Rules

### Response Rules
- Always state the full file path before generating any code block
    - Example: `// features/auth/data/datasources/auth_remote_datasource.dart`
- Always identify which layer a file belongs to before writing it (data / domain / presentation / core)
- If unsure where something belongs, ask — do not guess and generate
- Keep responses focused: one file or one problem at a time unless asked otherwise
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

### Hackathon Constraints
- Prioritize working code, but never break layer rules to get there
- If a feature is not listed in the Current Features table, confirm with the team before creating files for it
- If asked to do something that violates these rules, explain why and offer a compliant alternative instead
- When in doubt between two approaches, pick the simpler one — as long as it respects the architecture

---

## Hackathon Reminders

- Run `flutter clean && flutter pub get` if you get strange build errors
- Commit often: `feat: add login page`, `fix: auth state not emitting`, `chore: add user entity`
- If two people are working on the same feature, split by layer (one takes domain+data, one takes presentation)
- Keep BLoCs small — if a BLoC is doing too much, split it