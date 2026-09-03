---
name: modular-architecture
description: Layer definitions, the dependency matrix, and how a module is composed. Consult when adding a module, changing dependencies between modules, deciding which layer something belongs in, or diagnosing a layer violation. Describes the architecture itself, not the build system that implements it.
---

# Modular Architecture

Examples here are Swift, but the layering and dependency rules are language-agnostic —
the same structure holds for a Kotlin or TypeScript app.

## At a glance

```
App (AppMobile, AppWatch, AppNotificationHandler …)
 │
 ├─→ Feature ──→ Domain ──→ Infra
 │      ↘
 │      FeatureExtra   (localization, assets, design system)
 │
 └─→ AppDIContainer (composition root)

every layer ──→ Shared   (type extensions and utilities, Foundation only)
```

## Layers

| Layer | Responsibility | UI frameworks |
|---|---|---|
| **App** | Runnable targets and the composition root. Assembly happens only in `AppDIContainer` | allowed |
| **Feature** | One screen. View + ViewModel + ViewFactory | allowed |
| **FeatureExtra** | Shared UI resources for features — localization, assets, design system | allowed |
| **Domain** | Entities, repository protocols, and their implementations | **forbidden** |
| **Infra** | Infrastructure: networking, databases, keychain | **forbidden** |
| **Shared** | Type extensions and utilities | **forbidden** |

Each layer below says what it holds, what it keeps out, and the call that most often gets argued about.

### App

**Holds** the app lifecycle, deep link routing, the wiring between features (passing one feature's view factory into another as a closure), and `AppDIContainer`.

**Keeps out** screens and business logic. The app owns *which* screen appears, never *what* it shows.

**When it's unclear —** global state such as the signed-in session belongs to a `Domain` repository, not the app. The app may read it to decide the root screen, but it does not own it. Otherwise every layer ends up reaching back up for state.

### Feature

**Holds** views, view models, display models (a formatted date, a localized label), and screen state: loading, error, selection, scroll position.

**Keeps out** other features, network calls, persistence, and domain rules. A feature talks to `Domain` interfaces and nothing else.

**When it's unclear —** pagination splits. The screen owns "the user reached the bottom, load more"; the repository owns the cursor and what page comes next. If the view model is tracking offsets, that logic belongs one layer down.

### FeatureExtra

**Holds** design tokens (colour, type, spacing), reusable components, icons, and string catalogs.

**Keeps out** anything that knows a domain concept, and anything only one feature uses — that lives with its feature.

**When it's unclear —** a component wants to take a domain entity, say `UserCard(user:)`. Don't. It takes primitives: `UserCard(name:avatarURL:)`. The moment a component knows `User`, the design system depends on `Domain` and stops being reusable across projects.

### Domain

**Holds** entities, repository protocols and their implementations, DTO-to-entity mapping, domain rules and validation, and domain error types.

**Keeps out** UI formatting (a date *string* belongs to the feature showing it) and transport details such as status codes or SQL.

**When it's unclear —** there are no use case types here by default; a repository absorbs that role. Introduce one only when a single operation genuinely composes several repositories. A `UseCase` that forwards one call to one repository is indirection with no reader.

### Infra

**Holds** HTTP clients, database and keychain access, file IO, and DTOs — the shapes data arrives in. Retry, timeout, and caching policy live here too.

**Keeps out** entities and domain rules. `Infra` does not know `Domain` exists, which is why `Domain` owns the mapping between them.

**When it's unclear —** permission prompts (camera, location, notifications) need UI frameworks, so they cannot live here. Put them in a dedicated module the feature layer can reach, and let `Infra` assume access was already granted.

### Shared

**Holds** extensions on standard types, pure utilities, and contracts every layer needs — logging is the usual one.

**Keeps out** domain concepts, and anything only one module uses.

**When it's unclear —** ask whether two or more layers genuinely use it. If not, put it where it is used. `Shared` is the layer that quietly turns into a junk drawer, and because everything depends on it, everything rebuilds when it changes.


## Dependency rules

Allowed directions (row → column). The single source of truth for this table is the `ALLOWED` dictionary at the top of `Scripts/lint-deps.py`, and `make lint-deps` enforces it.

| from \ to | App | Feature | FeatureExtra | Domain | Infra | Shared |
|---|---|---|---|---|---|---|
| **App** | ✗ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Feature** | ✗ | ✗ | ✅ | ✅ | ✗ | ✅ |
| **FeatureExtra** | ✗ | ✗ | ✗ | ✗ | ✗ | ✅ |
| **Domain** | ✗ | ✗ | ✗ | ✗ | ✅ | ✅ |
| **Infra** | ✗ | ✗ | ✗ | ✗ | ✗ | ✅ |
| **Shared** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

Further rules:

1. **Modules in the same layer must not depend on each other.** Navigation between features is wired by the app, which injects closures into view factories.
2. **A module depends only on the Interface of the layers below it.** The single exception is `AppDIContainer` — a composition root has to see concrete types in order to assemble them.
3. **Interface targets may depend only on external packages and Shared Interfaces.** That narrow allowance exists so shared error and ID types don't have to be redefined per module. (The App layer is exempt: `AppDIContainerInterface` needs Feature Interfaces to expose view factory contracts.)
4. **Testing products are for test targets and Example apps only.**
5. **Domain, Infra, and Shared must not import UI frameworks** (SwiftUI, UIKit, Compose, and so on).

### Why features never depend on each other

A feature exposes a view factory contract rather than a concrete view:

```swift
@MainActor
public protocol SampleViewFactory: Sendable {
    func makeSampleView() -> AnyView
}
```

When one screen has to push another, the app passes a closure into the factory. Features stay unaware of each other, so any one of them can be built and demoed alone.

## How a module is composed

A module is made of up to five build targets, each one a folder:

| Folder | Target | Role |
|---|---|---|
| `Interface/` | library | Public protocols and entities. Depends on no other internal module |
| `Sources/` | library | The implementation |
| `Testing/` | library | Public mocks that other modules, demos, and previews import |
| `Tests/` | test bundle | Unit tests |
| `Example/` | app | A standalone demo of this module alone |

**Targets may be omitted when a layer doesn't need them.** `Sources` and `Tests` are required; the rest are optional.

- `FeatureExtraDesignSystem` has no `Interface` or `Testing` — a resource module has no contract to abstract
- `AppDIContainer` has no `Testing` — it exists only to assemble the app
- `Example` is for the Feature layer only; other layers are verified through their tests

Splitting `Interface` from `Sources` is what makes rule 2 enforceable: a module that depends on an interface cannot reach the implementation behind it, so changing an implementation never ripples upward.

### Mock conventions

Mocks in `Testing/` are actors that guard their state and expose `stubbed*` and `*CallCount`.

```swift
public actor MockSampleRepository: SampleRepository {
    public var stubbedItems: Result<[SampleItem], SampleRepositoryError>
    public private(set) var itemsCallCount = 0
}
```

Because `Testing` is a real target rather than test-only code, an Example app can inject the same mocks a unit test uses and run the module without the rest of the app.

### Errors

Each layer defines its own error type and translates at the boundary. An `Infra` failure becomes a `Domain` error before it crosses into `Domain`, so callers never have to know what lives two layers down.

### Resources

Modules that ship assets or string catalogs keep them under `Sources/Resources/`, and read them from the module's own bundle rather than the app bundle.

---

This document describes the architecture. How a given project builds it — which parts are packages, which are IDE targets, and where each thing is declared — belongs to that project. In this repository that is the **build-system-split** skill.
