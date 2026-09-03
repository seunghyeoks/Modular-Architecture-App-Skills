---
name: modular-architecture
description: Layer structure and dependency rules for this repository. Consult when adding a module, changing dependencies, adding an app target (AppWatch, extensions), or diagnosing a layer violation. Covers the SPM-modules + XcodeGen-app-shell setup, the five module targets, resource placement, and host test branching.
---

# Modular Architecture

## At a glance

```
App (AppMobile, AppWatch, AppNotificationHandler …)     ← .xcodeproj (XcodeGen)
 │
 ├─→ Feature ──→ Domain ──→ Infra                       ← everything below is a local Swift Package
 │      ↘
 │      FeatureExtra   (localization, assets, design system)
 │
 └─→ AppDIContainer (composition root)

every layer ──→ Shared   (type extensions and utilities, Foundation only)
```

**One module is one local Swift Package.** Only runnable app targets — the app itself and per-module Example apps — live in the XcodeGen-generated `.xcodeproj`. That split exists because SwiftPM cannot produce an iOS app target.

The point of this structure is to **eliminate project regeneration from everyday work**. Adding files, editing modules, and changing dependencies all stay inside SwiftPM, so Xcode never has to reindex.

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
2. **A module depends only on the Interface products of the layers below it.** The single exception is `AppDIContainer` — a composition root has to see concrete types in order to assemble them.
3. **Interface targets may depend only on external SPM packages and Shared Interface products.** That narrow allowance exists so shared error and ID types don't have to be redefined per module. (The App layer is exempt: `AppDIContainerInterface` needs Feature Interfaces to expose view factory contracts.)
4. **Testing products are for test targets and Example apps only.**
5. **Domain, Infra, and Shared must not import SwiftUI or UIKit.**

## Module layout

```
Projects/<Layer>/<Module>/
├── Package.swift
├── Interface/     Public protocols and entities. Depends on no other internal module
├── Sources/       Implementation
├── Testing/       Public mocks that other modules import
├── Tests/         swift-testing
└── Example/       Feature layer only — a standalone demo app
    ├── project.yml    That app target's spec, with paths relative to itself
    └── Sources/
```

**Targets may be omitted when a layer doesn't need them.** `Sources` and `Tests` are required; the rest are optional.

- `FeatureExtraDesignSystem` has no `Interface` or `Testing` — there is no contract to abstract.
- `AppDIContainer` has no `Testing` — it exists only to assemble the app.

### Mock conventions

Mocks in `Testing/` are actors that guard their state and expose `stubbed*` and `*CallCount`.

```swift
public actor MockSampleRepository: SampleRepository {
    public var stubbedItems: Result<[SampleItem], SampleRepositoryError>
    public private(set) var itemsCallCount = 0
}
```

## Resources

Modules that ship resources put them in `Sources/Resources/` and declare them in `Package.swift`.

```swift
.target(name: "FeatureExtraDesignSystem", path: "Sources", resources: [.process("Resources")])
```

SwiftPM generates the resource bundle and the `Bundle.module` accessor. Assets and string catalogs are both read with `bundle: .module`.

> Argument order in `.target(...)` is `name → dependencies → path → exclude → sources → resources`. Putting `resources:` before `path:` is a compile error.

## Testing

`make check <Module>` picks one of two paths automatically.

- If `platforms` in `Package.swift` includes `.macOS` → `swift test` (**no simulator, considerably faster**)
- Otherwise → `xcodebuild test` against an iOS simulator

Enabling macOS is **opt-in per module**. Modules that avoid UI frameworks — Domain, Infra, Shared — can turn it on, but if an iOS-only third-party dependency lands, just switch that one module back off.

## Adding an app target (AppWatch, AppNotificationHandler, …)

**App targets carry their own settings in their own folder.** `Projects/App/AppMobile/project.yml` is the example; paths inside it are relative to that file (`Sources`, `Tests`).

1. Declare the target in `Projects/App/<AppName>/project.yml`
   - watchOS app: `type: application.watchapp2`, `platform: watchOS`
   - Extension: `type: app-extension` — the host app pulls it in with `embed: true`. Embedding within the App layer is the exception to the "no same-layer dependencies" rule.
2. Add one line to `include:` in the root `project.yml`
3. Add the target platform to `platforms` in **every module** that target uses
   ```bash
   make module NAME=FeatureGlance LAYER=Feature PLATFORMS=iOS,watchOS
   ```

Example apps follow the same convention: each one keeps its spec in `Example/project.yml`, and `Scripts/gen-modules.sh` picks them all up. A demo that needs camera usage descriptions or an extra mock package declares that in its own file.

## Adding a module

```bash
make module NAME=FeatureHome LAYER=Feature
```

Two things are left to do afterwards.

1. Add lower-layer modules to `dependencies` in `Package.swift` (the path depth is always `../../<Layer>/<Module>`)
2. If the app uses it, add it to `Projects/App/AppDIContainer/Package.swift` and wire it up

`packages:` registration and Example app includes are generated by `Scripts/gen-modules.sh`, so the root `project.yml` stays untouched. The only reason to edit it is adding an app target.
