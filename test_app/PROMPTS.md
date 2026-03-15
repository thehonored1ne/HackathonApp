# PROMPTS.md — Hackathon Prompt Sequence

> Copy-paste these prompts in order when building a feature.
> Fill in the [ ] placeholders before sending.
> Never skip a step — each one sets up the next.

---

## 🟢 Step 1 — Session Start
> Run this ONCE at the beginning of the hackathon.

```
Read AGENTS.md, ARCHITECTURE.md, and CONVENTIONS.md before doing anything.

App: [app name]
First feature: [feature name]
My task: [your layer — e.g., domain + data]
Teammate: [their layer — e.g., presentation]

Do not generate any code yet. Summarize:
1. Architecture layer rules
2. Stack and packages
3. Hackathon mode constraints

Wait for my go.
```

---

## 🏗️ Step 2 — Domain Layer
> Always build domain first — it has zero dependencies.

```
Implement the domain layer for [feature].

This includes:
- entity
- repository interface
- use cases: [list them, e.g. SignInUseCase, SignOutUseCase]

Follow ARCHITECTURE.md and CONVENTIONS.md.
One file at a time, state the full path before each file.
```

---

## 🗄️ Step 3 — Data Layer
> Build this after domain is confirmed.

```
Implement the data layer for [feature].

This includes:
- model (extend the entity, with fromFirestore/toFirestore)
- remote datasource (Firebase calls only, debugPrint before rethrow)
- repository implementation

Base it on the domain layer we just created.
State the full path before each file.
```

---

## 🎨 Step 4 — Presentation Layer
> Build this after data layer is confirmed.

```
Implement the presentation layer for [feature].

This includes:
- BLoC (generate all three files together: _bloc, _event, _state)
- page
- widgets

Rules:
- BLoC calls use cases only
- Every BlocBuilder must handle initial, loading, error, and success states
- No empty Container() for unhandled states

State the full path before each file.
```

---

## 🔗 Step 5 — Wire It Up
> Connect the feature to the rest of the app.

```
Wire up [feature] to the app.

This includes:
- Add the route to core/router/app_router.dart
- Add BlocProvider where needed
- Connect auth redirect in GoRouter if applicable

Do not modify any existing feature files outside of app_router.dart.
```

---

## ✅ Step 6 — Verify
> Catch mistakes before moving to the next feature.

```
Review what we just built for [feature].

Check:
1. No Firebase calls outside datasources
2. No cross-feature imports
3. All BlocBuilders handle all states
4. Route is registered in app_router.dart
5. No barrel index.dart files created

List any violations found. If none, confirm the feature is complete.
```

---

## 🔁 Step 7 — Next Feature
> Close the loop and move on.

```
Update AGENTS.md:
- Mark [feature] as ✅ Done in the feature table
- Set Current Focus to [next feature]

Then wait for my go before starting the next feature.
```

---

## 🐛 Bug Fix Prompt
> Use this anytime something breaks.

```
There is a bug in [feature / file path].

Observed behavior: [what is happening]
Expected behavior: [what should happen]
Error message (if any): [paste error here]

Do not refactor anything outside the broken file.
Do not change the architecture to fix this.
Identify the cause first, then fix it.
```

---

## ❓ Unsure Where a File Belongs
> Use this before creating any new file you're not sure about.

```
I need to add [describe what it does].

Which layer and folder does this belong to?
Reference ARCHITECTURE.md to explain your decision.
Do not generate the file yet — just tell me where it goes.
```

---

## 🔥 Firestore Query Prompt
> Use this when you need a Firestore query written.

```
Write a Firestore query for [describe what you need].

Rules:
- Place it in the correct datasource file
- debugPrint the FirebaseException message before rethrowing
- If the query needs a composite index, show the index notice in the required format
```

---

## 💡 Tips

- Always wait for the agent to confirm Step 1 before moving to Step 2
- If the agent starts generating code you didn't ask for — stop it and re-send the current step prompt
- If something breaks, use the Bug Fix prompt before asking the agent to continue
- One feature at a time — finish and verify before starting the next