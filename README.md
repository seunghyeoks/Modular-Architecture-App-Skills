# Swift Modular Architecture Skills

SPM 모듈 + XcodeGen 앱 셸로 구성한 iOS 모듈러 아키텍처의 규칙 문서.
사람이 읽는 문서이면서 코딩 에이전트가 참조하는 스킬이다.

프로젝트에 git submodule 로 붙여 쓴다.

```bash
git submodule add git@github.com:seunghyeoks/Swift-Modular-Architecture-Skills.git \
  .agents/skills/modular-architecture
```

여러 프로젝트가 같은 규칙을 공유하고, 규칙이 바뀌면 여기서 한 번만 고친다.

## 내용

- [SKILL.md](SKILL.md) — 레이어 정의와 의존 행렬, 모듈 타겟 구성, 리소스 배치,
  호스트 테스트 분기, 앱 타겟 추가 방법

## 규칙의 단일 출처

의존 행렬의 실제 집행자는 이 문서가 아니라 각 프로젝트의 `Scripts/lint-deps.py`
상단 `ALLOWED` 딕셔너리다. 둘이 어긋나면 스크립트가 옳다.
