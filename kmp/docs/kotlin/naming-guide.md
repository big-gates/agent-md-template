# Kotlin 네이밍 Guide

> 클래스, 함수, 변수, 상수, 패키지 네이밍 규칙이다.

---

## 기본 컨벤션

| 대상 | 스타일 | 예시 |
|------|--------|------|
| 클래스 / 인터페이스 | PascalCase | `UserRepository`, `HomeViewModel` |
| 함수 | camelCase | `getUserById`, `toFormattedString` |
| 프로퍼티 / 변수 | camelCase | `userName`, `isLoading` |
| 상수 (`const val`, `object` 내) | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `BASE_URL` |
| 패키지 | lowercase, 점 구분 | `com.example.core.data` |
| 파일명 | PascalCase (클래스명과 동일) | `UserRepository.kt` |
| 최상위 함수/확장만 있는 파일 | PascalCase 복수형 또는 동사 | `StringExtensions.kt`, `Mappers.kt` |

---

## 클래스 네이밍

| 유형 | 접미사 | 예시 |
|------|--------|------|
| ViewModel | `ViewModel` | `HomeViewModel` |
| Repository 인터페이스 | `Repository` | `UserRepository` |
| Repository 구현 | `RepositoryImpl` | `UserRepositoryImpl` |
| DataSource | `DataSource` | `UserLocalDataSource` |
| API 서비스 | `Service` | `UserService` |
| DTO (네트워크) | `Dto` | `UserDto`, `UserResponseDto` |
| DB 엔티티 | `Entity` | `UserEntity` |
| UI 상태 | `UiState` | `HomeUiState` |
| UI 액션/이벤트 | `Action` 또는 `Event` | `HomeAction`, `HomeEvent` |
| Mapper 확장 함수 | `to{Target}` | `UserDto.toDomain()`, `User.toEntity()` |
| Use Case | 동사 + 명사 | `GetUserUseCase`, `SyncProductsUseCase` |

---

## 함수 네이밍

| 유형 | 패턴 | 예시 |
|------|------|------|
| 데이터 조회 | `get{Name}`, `find{Name}` | `getUserById()`, `findByEmail()` |
| 데이터 관찰 | `observe{Name}` | `observeUsers()` → `Flow<List<User>>` |
| Boolean 반환 | `is{State}`, `has{Property}`, `can{Action}` | `isValid()`, `hasPermission()` |
| 변환 | `to{Target}` | `toUiState()`, `toEntity()` |
| 팩토리 | `create{Name}`, `{Name}.of()` | `createHttpClient()` |
| 이벤트 콜백 | `on{Event}` | `onClick()`, `onDismiss()` |
| 라이프사이클 | `init`, `start`, `stop`, `clear` | `initSync()`, `clearCache()` |

---

## Boolean 프로퍼티

```kotlin
// DO — is/has/can 접두사로 의미 명확화
val isLoading: Boolean
val hasNetworkConnection: Boolean
val canSubmit: Boolean

// DON'T — 동사/명사 혼동
val loading: Boolean     // 형용사? 동명사?
val network: Boolean     // 무엇이 Boolean인지 불명확
val submit: Boolean      // 동사? 상태?
```

---

## 약어 규칙

| 규칙 | 예시 |
|------|------|
| 2글자 약어는 모두 대문자 | `IOStream`, `ID` |
| 3글자 이상 약어는 PascalCase | `HttpClient`, `JsonParser`, `ApiService` |
| 변수에서는 camelCase 적용 | `val userId`, `val httpClient` |
