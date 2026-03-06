# Kotlin 타입 Guide

> Null safety, sealed/enum/value class, data class, 불변성 규칙이다.

---

## Null Safety

### 규칙: `!!` 사용 금지

```kotlin
// DO — 안전한 대안들
val name = user?.name ?: "Unknown"          // Elvis
val user = getUser() ?: return              // 조기 리턴
val user = requireNotNull(getUser()) { "User must exist" }  // 명시적 예외
val length = text?.length ?: 0              // 기본값

// DO — 스마트 캐스트 활용
val user = getUser()
if (user != null) {
    // 이 블록에서 user는 non-null
    println(user.name)
}

// DON'T
val name = user!!.name   // 금지: NPE 위험
```

### Nullable 설계 원칙

| 규칙 | 설명 |
|------|------|
| 함수 반환 | 없을 수 있는 값은 `T?` 반환, 반드시 있어야 하면 `T` + 예외 |
| 함수 파라미터 | nullable 파라미터보다 오버로드 또는 기본값 선호 |
| 컬렉션 | `null` 컬렉션보다 빈 컬렉션 반환 (`emptyList()`) |
| 플랫폼 타입 | Java 호출 결과는 즉시 nullable 여부를 명시 |

```kotlin
// DO — 빈 컬렉션 반환
fun getUsers(): List<User> = remoteUsers ?: emptyList()

// DON'T — null 컬렉션
fun getUsers(): List<User>? = remoteUsers   // 호출자가 null 처리 필요
```

---

## Data Class

```kotlin
// DO — 불변 프로퍼티, 의미 있는 구성
data class User(
    val id: Long,
    val name: String,
    val email: String,
)

// DON'T — var 사용
data class User(
    var id: Long,        // 금지: 불변성 파괴
    var name: String,
)
```

| 규칙 | 설명 |
|------|------|
| `val` 전용 | 모든 프로퍼티는 `val`로 선언 |
| 변경 시 `copy()` | `user.copy(name = "New")` |
| body 없는 것이 이상적 | 커스텀 로직은 확장 함수로 분리 |
| `equals`/`hashCode` 주의 | 생성자 파라미터만 비교 대상 |

---

## Sealed Class / Sealed Interface

분기가 고정된 타입 계층에 사용한다. `when`에서 `else` 없이 모든 분기를 컴파일 타임에 강제한다.

```kotlin
// DO — sealed로 상태 모델링
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

// when에서 else 없이 사용 — 새 분기 추가 시 컴파일 오류로 감지
fun <T> render(state: UiState<T>) = when (state) {
    is UiState.Loading -> showLoading()
    is UiState.Success -> showData(state.data)
    is UiState.Error -> showError(state.message)
}
```

| sealed class | sealed interface |
|-------------|-----------------|
| 상태를 가질 때 | 상태 없이 타입 분기만 필요할 때 |
| 단일 상속만 가능 | 다중 구현 가능 |

---

## Enum Class

고정된 상수 집합에 사용한다.

```kotlin
enum class SortOrder(val displayName: String) {
    NEWEST("최신순"),
    OLDEST("오래된순"),
    NAME_ASC("이름순"),
}
```

| enum vs sealed | 기준 |
|---------------|------|
| `enum` | 모든 인스턴스가 동일 구조, 데이터 없음/고정 |
| `sealed` | 분기별 데이터 구조가 다름 |

---

## Value Class

래핑 비용 없이 타입 안전성을 확보한다.

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class Email(val value: String)

// 컴파일 타임에 UserId와 Email 혼용 방지
fun getUser(id: UserId): User
```

| 사용 시점 | 예시 |
|----------|------|
| 원시 타입에 도메인 의미 부여 | `UserId`, `ProductId` |
| 단위 혼동 방지 | `Milliseconds`, `Seconds` |
| 문자열 타입 구분 | `Email`, `PhoneNumber` |

---

## 불변성 원칙

| 규칙 | 설명 |
|------|------|
| `val` 기본 | `var`는 꼭 필요한 경우에만 사용 |
| 읽기전용 컬렉션 노출 | `List`, `Set`, `Map` 타입으로 외부 노출 |
| `copy()` 활용 | data class 변경 시 새 인스턴스 생성 |
| Mutable 내부 한정 | `MutableList`는 내부에서만 사용, 외부에는 `List`로 노출 |

```kotlin
// DO
class UserRepository {
    private val _users = mutableListOf<User>()
    val users: List<User> get() = _users   // 읽기전용 노출
}

// DON'T
class UserRepository {
    val users = mutableListOf<User>()   // 외부에서 수정 가능
}
```
