# AGENTS.md — Flutter Firebase Hackathon Project

> 📎 Always read this file first.
> For architecture rules → see `ARCHITECTURE.md`
> For coding patterns and conventions → see `CONVENTIONS.md`

---

## Current Focus

> 🟡 **Pre-hackathon setup — not started yet.**
> Update this section at the start of the hackathon.

```
Working on   : [PLACEHOLDER — e.g., auth feature / login page]
Current task : [PLACEHOLDER — e.g., implementing sign in with email/password]
Next up      : [PLACEHOLDER — e.g., home feed skeleton]
Blocked by   : [PLACEHOLDER — or remove if nothing is blocking]
```

---

## Stack

- **Flutter** — UI framework
- **Firebase** — Firestore (database) + Auth (authentication)
- **BLoC** — state management (`flutter_bloc`)
- **go_router** — navigation and routing
- **Clean Architecture** — feature-first folder structure

---

## Packages

### Currently Installed

| Package | Version | Purpose |
|---------|---------|---------|
| `firebase_core` | ^4.5.0 | Firebase initialization |
| `firebase_auth` | ^6.2.0 | Authentication |
| `cupertino_icons` | ^1.0.8 | iOS-style icons |

### To Be Added Before Development

| Package | Purpose |
|---------|---------|
| `cloud_firestore` | Firestore database |
| `flutter_bloc` | BLoC state management |
| `go_router` | Navigation and routing |
| `dartz` | Functional programming (`Either`, `Option`) |
| `equatable` | Value equality for entities and states |

> ⚠️ Run `flutter pub add <package>` for each before writing feature code.
> After adding all packages, run `flutter pub get`.

---

## Current Features

| Feature | Data | Domain | Presentation | Notes |
|---------|------|--------|--------------|-------|
| `auth` | 🚧 | 🚧 | 🚧 | Firebase Auth |
| `home` | ⬜ | ⬜ | ⬜ | Not started |

> Legend: ✅ Done · 🚧 In progress · ⬜ Not started
> Update this table as features are completed.

---

## Hackathon Reminders

- Run `flutter clean && flutter pub get` if you get strange build errors
- Never regenerate or overwrite `firebase_options.dart` unless explicitly asked
- Commit often: `feat: add login page`, `fix: auth state not emitting`, `chore: add user entity`
- If two people are working on the same feature, split by layer (one takes domain+data, one takes presentation)
- Keep BLoCs small — if a BLoC is doing too much, split it
- Do not leave Firestore in test mode (open read/write) before presenting