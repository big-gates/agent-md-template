# core / common

> **트리거**: `core/common/` 하위 파일을 수정할 때, 또는 로깅·유틸·Dispatcher 등 횡단 관심사를 추가할 때 이 문서를 참조한다.

## 역할
- **횡단 관심사(Cross-Cutting Concern)** 를 제공하는 모듈이다.
- 로깅, 유틸리티, 플랫폼 헬퍼(expect/actual), 확장 함수, Dispatcher 제공 등 **모든 레이어에서 사용하는 기반 기능**을 담당한다.
- 어떤 모듈에서든 의존할 수 있는 유일한 횡단 모듈이다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `core/common/src/commonMain/kotlin/{package}/log/` | Logger 인터페이스 및 공통 구현 |
| `core/common/src/commonMain/kotlin/{package}/result/` | 공통 Result/Error 래퍼 (선택) |
| `core/common/src/commonMain/kotlin/{package}/ext/` | Kotlin 확장 함수 (String, Date 등) |
| `core/common/src/commonMain/kotlin/{package}/dispatcher/` | CoroutineDispatcher 제공자 (테스트 교체용) |
| `core/common/src/commonMain/kotlin/{package}/util/` | 포맷터, 검증, 기타 유틸리티 |
| `core/common/src/androidMain/kotlin/{package}/` | Android expect 구현 |
| `core/common/src/iosMain/kotlin/{package}/` | iOS expect 구현 |

## 에이전트 판단 규칙

### 이 모듈에 넣어야 하는가?

| 질문 | Yes → core:common | No → 다른 곳 |
|------|-------------------|-------------|
| 2개 이상 레이어에서 사용하는가? | O | 해당 레이어 모듈에 배치 |
| 프레임워크 무관한 순수 유틸인가? | O | 프레임워크 의존이면 해당 모듈 |
| 비즈니스 로직이 아닌가? | O | 비즈니스 로직이면 `core:domain` |
| UI 관련이 아닌가? | O | UI면 `core:designsystem` 또는 `core:ui` |

### 무엇이 들어가는가?

| 횡단 관심사 | 예시 | 위치 |
|------------|------|------|
| 로깅 | `AppLogger` (expect/actual) | `log/` |
| Dispatcher 제공 | `AppDispatchers(io, default, main)` | `dispatcher/` |
| 공통 에러 타입 | `AppException`, `AppResult<T>` | `result/` |
| 확장 함수 | `String.toFormattedDate()` | `ext/` |
| 플랫폼 헬퍼 | `expect fun platformName(): String` | 루트 |
| 상수/설정 | `const val MAX_RETRY = 3` | `util/` |

### 무엇이 들어가면 안 되는가?

| 금지 | 이유 | 올바른 위치 |
|------|------|-----------|
| 도메인 모델 | 비즈니스 계층 | `core:domain/model/` |
| Repository/UseCase | 비즈니스 로직 | `core:domain` |
| DTO/Entity | 인프라 의존 | `core:datasource` |
| Mapper | 데이터 계층 | `core:data/mapper/` |
| UI 컴포넌트 | 프레젠테이션 계층 | `core:designsystem`, `core:ui` |
| API 클라이언트 설정 | 네트워크 인프라 | `core:datasource/remote/client/` |

## DO / DON'T

```kotlin
// DO — expect/actual로 플랫폼별 로깅
// commonMain
expect object AppLogger {
    fun d(tag: String, message: String)
    fun e(tag: String, message: String, throwable: Throwable? = null)
}

// DO — 테스트 교체 가능한 Dispatcher 제공
data class AppDispatchers(
    val io: CoroutineDispatcher = Dispatchers.IO,
    val default: CoroutineDispatcher = Dispatchers.Default,
    val main: CoroutineDispatcher = Dispatchers.Main,
)

// DON'T — 특정 도메인에 종속된 로직
fun calculateExpirationDate(item: Item): LocalDate {
    // 금지: 비즈니스 로직은 core:domain에 배치
}

// DON'T — 외부 모듈 의존
import io.ktor.client.*   // 금지: core:common은 자체 완결
```

## 변경 시 체크리스트
- [ ] 공통 API 변경 시 영향받는 모듈의 컴파일 오류를 확인했는가?
- [ ] `expect`/`actual` 추가 시 `androidMain`과 `iosMain` 양쪽에 구현이 있는가?
- [ ] 새 유틸 추가 시 기존 유틸과 중복이 없는가?
- [ ] 비즈니스 로직이나 프레임워크 의존이 유입되지 않았는가?
