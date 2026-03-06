# app / shared

> **트리거**: `shared/` 하위 파일을 생성·수정할 때, 또는 DI 구성을 변경할 때 이 문서를 참조한다.

## 역할
- KMP 공용 모듈의 앱 레벨 엔트리, 브릿지, DI 설정, 앱 전역 상태를 관리한다.
- Android/iOS 양쪽에서 호출하는 공통 초기화 및 라이프사이클 조율 지점이다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `shared/src/commonMain/kotlin/{package}/` | 공용 로직, DI 모듈 정의 |
| `shared/src/androidMain/kotlin/{package}/` | Android 플랫폼 구현 |
| `shared/src/iosMain/kotlin/{package}/` | iOS 플랫폼 구현 |

## 의존/연동
- `core:data`, `core:datasource` 등 공용 레이어를 DI 조합용으로 사용한다.
- Android/iOS 플랫폼별 초기화 및 브릿지 코드를 포함한다.
- `feature:*` 모듈의 DI 등록을 조율할 수 있다.

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 모듈의 DI 등록 | `shared`의 Koin 모듈 정의에 추가한다 |
| 플랫폼 expect/actual 추가 | `commonMain`에 `expect`, `androidMain`/`iosMain`에 `actual` 구현 |
| 앱 초기화 순서 변경 | Android `Application.onCreate()`와 iOS `@main init` 양쪽 호출부 확인 |
| 새 feature 추가 | ViewModel을 DI에 등록하고 Navigation에 화면 추가 |

## DI 조합 구조

`shared`의 `appModule`이 모든 core/feature DI 모듈을 통합한다.

```kotlin
// shared의 AppModule.kt
val appModule = module {
    includes(
        // core 모듈 (하위 → 상위 순서)
        dataSourceModule,          // 인프라 + DataSource
        dataModule,                // Repository 구현
        domainModule,              // UseCase

        // feature 모듈 (각 feature의 ViewModel 등록)
        // {name}ViewModelModule,  // feature:{name}
    )
}
```

| 규칙 | 설명 |
|------|------|
| core 모듈 먼저, feature 모듈 나중에 | core의 의존성이 먼저 등록되어야 feature ViewModel이 주입받을 수 있음 |
| 새 feature 추가 시 | feature DI 모듈 → `appModule`의 `includes`에 추가 |
| DI 상세 규칙 | [di-guide.md](../../convention/di-guide.md) 참조 |

## DO / DON'T

```kotlin
// DO — DI 모듈에서 의존성 등록
val appModule = module {
    includes(dataSourceModule, dataModule, domainModule, ...)
}

// DON'T — shared에서 비즈니스 로직 직접 구현
fun processItems(items: List<Item>) {
    // 금지: 비즈니스 로직은 core:domain의 UseCase에 배치
}
```

## 변경 시 체크리스트
- [ ] 공용 API 변경 시 iOS/Android 양쪽 호출부를 함께 점검했는가?
- [ ] DI 모듈 변경 시 모든 의존성이 올바르게 해결되는가?
- [ ] 플랫폼별 동기화 트리거와 라이프사이클 영향을 확인했는가?
- [ ] `expect`/`actual` 추가 시 양쪽 플랫폼에 구현이 있는가?

## 빌드/실행
- 빌드·테스트는 상위 앱 모듈(`androidapp`, `iosApp`)을 통해 수행한다.
