---
name: modular-architecture
description: Layer definitions, the dependency matrix, and how a module is composed. Consult when adding a module, changing dependencies between modules, deciding which layer something belongs in, or diagnosing a layer violation. Describes the architecture itself, not the build system that implements it.
---

# Modular Architecture

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
5. **Domain, Infra, and Shared must not import SwiftUI or UIKit.**

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

This document describes the architecture. For how it is actually built — which parts are Swift Packages, which are Xcode targets, and where each thing is declared — see the **build-system-split** skill.
