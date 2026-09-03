---
name: modular-architecture
description: 이 저장소의 레이어 구조와 의존 규칙. 모듈을 추가하거나 의존성을 바꾸거나 앱 타겟(AppWatch, 익스텐션)을 늘릴 때, 그리고 레이어 위반을 진단할 때 참고한다. SPM 모듈 + XcodeGen 앱 셸 구성, TMA 5타겟, 리소스 배치, 호스트 테스트 분기를 다룬다.
---

# 모듈러 아키텍처

## 한눈에

```
App (AppMobile, AppWatch, AppNotificationHandler …)     ← .xcodeproj (XcodeGen)
 │
 ├─→ Feature ──→ Domain ──→ Infra                       ← 이하 전부 로컬 Swift Package
 │      ↘
 │      FeatureExtra   (Localization, Assets, DesignSystem)
 │
 └─→ AppDIContainer (Composition Root)

모든 레이어 ──→ Shared   (타입 오버라이드·확장·유틸, Foundation only)
```

**모듈 하나 = 로컬 Swift Package 하나.** 실행 가능한 앱 타겟(App, Example)만 XcodeGen 이 만드는 `.xcodeproj` 에 있다. SwiftPM 이 iOS 앱 타겟을 만들 수 없기 때문에 생긴 분담이다.

이 구조의 목적은 **일상 작업에서 프로젝트 재생성을 없애는 것**이다. 파일 추가, 모듈 수정, 의존성 변경이 모두 SwiftPM 영역에서 끝나므로 Xcode 재인덱싱이 일어나지 않는다.

## 레이어

| 레이어 | 역할 | UI 프레임워크 |
|---|---|---|
| **App** | 실행 타겟과 Composition Root. 앱 조립은 `AppDIContainer` 한 곳에서만 | 허용 |
| **Feature** | 화면 단위. View + ViewModel + ViewFactory | 허용 |
| **FeatureExtra** | Feature 가 쓰는 UI 공용 자원 — 로컬라이제이션, 에셋, 디자인 시스템 | 허용 |
| **Domain** | Entity + Repository 프로토콜 + 구현 | **금지** |
| **Infra** | 네트워크·DB·Keychain 등 인프라 | **금지** |
| **Shared** | 타입 오버라이드, Swift 확장, 공용 유틸 | **금지** |

## 의존 규칙

허용 행렬 (행 → 열). 이 표의 단일 출처는 `Scripts/lint-deps.py` 상단의 `ALLOWED` 이며, `make lint-deps` 가 검사한다.

| from \ to | App | Feature | FeatureExtra | Domain | Infra | Shared |
|---|---|---|---|---|---|---|
| **App** | ✗ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Feature** | ✗ | ✗ | ✅ | ✅ | ✗ | ✅ |
| **FeatureExtra** | ✗ | ✗ | ✗ | ✗ | ✗ | ✅ |
| **Domain** | ✗ | ✗ | ✗ | ✗ | ✅ | ✅ |
| **Infra** | ✗ | ✗ | ✗ | ✗ | ✗ | ✅ |
| **Shared** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

추가 규칙:

1. **같은 레이어끼리 의존 금지.** Feature 간 화면 전환은 App 이 closure 를 ViewFactory 에 주입해 잇는다
2. **상위는 하위의 Interface product 만 의존한다.** 유일한 예외가 `AppDIContainer` — Composition Root 는 구현체를 알아야 조립할 수 있다
3. **Interface 타겟은 외부 SPM 과 Shared 레이어 Interface 만** 의존할 수 있다. 공용 에러·ID 타입을 쓰기 위한 최소 허용이다. (App 레이어는 예외 — `AppDIContainerInterface` 가 ViewFactory 계약을 노출하려면 Feature Interface 가 필요하다)
4. **Testing product 는 테스트 타겟과 Example 앱에서만** 쓴다
5. **Domain / Infra / Shared 는 SwiftUI·UIKit 을 import 하지 않는다**

## 모듈 구성

```
Projects/<Layer>/<Module>/
├── Package.swift
├── Interface/     public 프로토콜과 Entity. 다른 내부 모듈에 의존하지 않는다
├── Sources/       구현
├── Testing/       다른 모듈이 import 하는 public Mock
├── Tests/         swift-testing
└── Example/       Feature 레이어만. 앱 프로젝트가 앱 타겟으로 가져간다
```

**타겟은 레이어 성격에 따라 생략할 수 있다.** `Sources` 와 `Tests` 는 필수이고 나머지는 필요할 때만 둔다.

- `FeatureExtraDesignSystem` 은 추상화할 계약이 없어 `Interface` / `Testing` 이 없다
- `AppDIContainer` 는 조립 전용이라 `Testing` 이 없다

### Mock 관례

`Testing/` 의 Mock 은 `actor` 로 상태를 보호하고 `stubbed*` 와 `*CallCount` 를 노출한다.

```swift
public actor MockSampleRepository: SampleRepository {
    public var stubbedItems: Result<[SampleItem], SampleRepositoryError>
    public private(set) var itemsCallCount = 0
}
```

## 리소스

리소스를 가진 모듈은 `Sources/Resources/` 에 두고 `Package.swift` 에 선언한다.

```swift
.target(name: "FeatureExtraDesignSystem", path: "Sources", resources: [.process("Resources")])
```

SwiftPM 이 리소스 번들과 `Bundle.module` 접근자를 자동 생성한다. Assets, String Catalog 모두 `bundle: .module` 로 읽는다.

> `.target(...)` 의 인자 순서는 `name → dependencies → path → exclude → sources → resources` 다. `resources:` 를 `path:` 앞에 두면 컴파일 에러가 난다.

## 테스트

`make check <Module>` 이 두 경로를 자동으로 갈라 쓴다.

- `Package.swift` 의 `platforms` 에 `.macOS` 가 있으면 → `swift test` (**시뮬레이터 불필요, 훨씬 빠름**)
- 없으면 → `xcodebuild test` (iOS 시뮬레이터)

macOS 를 켜는 건 **모듈별 선택 사항**이다. UI 프레임워크를 쓰지 않는 Domain/Infra/Shared 는 켤 수 있지만, iOS 전용 서드파티가 붙으면 그 모듈만 도로 끄면 된다.

## 앱 타겟 추가 (AppWatch, AppNotificationHandler 등)

**앱 타겟은 자기 폴더에서 설정을 들고 있다.** `Projects/App/AppMobile/project.yml` 이 그 예이며, 경로는 그 파일 기준 상대경로(`Sources`, `Tests`)로 쓴다.

1. `Projects/App/<앱이름>/project.yml` 에 타겟 선언
   - watchOS 앱: `type: application.watchapp2`, `platform: watchOS`
   - 익스텐션: `type: app-extension` — 앱 타겟이 `embed: true` 로 물린다. App 레이어 안에서의 embed 는 "같은 레이어 금지" 규칙의 예외다
2. 루트 `project.yml` 의 `include:` 에 한 줄 추가
3. 그 타겟이 쓰는 **모든 하위 모듈**의 `platforms` 에 대상 플랫폼 추가
   ```bash
   make module NAME=FeatureGlance LAYER=Feature PLATFORMS=iOS,watchOS
   ```

Example 앱은 이와 달리 전부 같은 모양이라 `Scripts/gen-modules.sh` 가 자동 생성한다 — 별도 `project.yml` 을 두지 않는다.

## 새 모듈 추가

```bash
make module NAME=FeatureHome LAYER=Feature
```

생성 후 할 일은 두 가지뿐이다.

1. `Package.swift` 의 `dependencies` 에 하위 레이어 모듈 추가 (경로 깊이는 항상 `../../<Layer>/<Module>`)
2. 앱에서 쓴다면 `Projects/App/AppDIContainer/Package.swift` 에 추가하고 조립

`packages:` 등록과 Example 앱 타겟은 `Scripts/gen-modules.sh` 가 자동 생성하므로 `project.yml` 은 손대지 않는다. 루트 `project.yml` 을 건드리는 경우는 **앱 타겟을 추가할 때뿐**이다.
