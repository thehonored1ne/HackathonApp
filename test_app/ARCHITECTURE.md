# ARCHITECTURE.md — Clean Architecture Rules

> Read this file when: scaffolding a new feature, unsure where a file belongs, or setting up a new layer.

---

## Dependency Direction (never break this)

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

### Example: Fully Implemented Feature (`auth`)

```
features/auth/
├── data/
│   ├── datasources/auth_remote_datasource.dart
│   ├── models/user_model.dart
│   └── repositories/auth_repository_impl.dart
├── domain/
│   ├── entities/user_entity.dart
│   ├── repositories/auth_repository.dart
│   └── usecases/sign_in_usecase.dart
└── presentation/
    ├── bloc/
    │   ├── auth_bloc.dart
    │   ├── auth_event.dart
    │   └── auth_state.dart
    ├── pages/login_page.dart
    └── widgets/login_form.dart
```

---

## Feature-First Rule

Every piece of code belongs to a feature first. Ask: *"which feature owns this?"*
- Belongs to one feature → `features/<feature>/`
- Used by 2 or more features → `core/`
- Unsure → default to the feature, promote to `core/` later

---

## `core/usecases/` Rule

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

---

## `core/widgets/` vs Feature `widgets/` Rule

- Widget used in **one feature only** → `features/<feature>/presentation/widgets/`
- Widget used in **two or more features** → `core/widgets/`
- Never move a widget to `core/widgets/` preemptively — wait until it's actually reused

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
7. Update the feature table in `AGENTS.md`

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