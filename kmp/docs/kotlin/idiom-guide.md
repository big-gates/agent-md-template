# Kotlin 이디엄 Guide

> 스코프 함수, 확장 함수, 컬렉션 연산, 시퀀스, 구조 분해 규칙이다.

---

## 스코프 함수

| 함수 | 수신 객체 | 반환값 | 주 용도 |
|------|----------|--------|---------|
| `let` | `it` | 람다 결과 | null 체크 후 변환 |
| `run` | `this` | 람다 결과 | 객체 설정 + 결과 계산 |
| `apply` | `this` | 수신 객체 | 객체 초기화/설정 |
| `also` | `it` | 수신 객체 | 부수 효과 (로깅 등) |
| `with` | `this` | 람다 결과 | 이미 있는 객체의 여러 멤버 호출 |

```kotlin
// let — null 안전 변환
val displayName = user?.let { "${it.firstName} ${it.lastName}" }

// apply — 객체 초기화
val request = HttpRequest().apply {
    url = "https://api.example.com"
    method = "GET"
    addHeader("Authorization", "Bearer $token")
}

// also — 부수 효과 (디버깅, 로깅)
val users = repository.getUsers()
    .also { logger.d("Fetched ${it.size} users") }

// run — 설정 + 결과 반환
val result = service.run {
    configure()
    execute()
}

// with — 여러 멤버 접근
with(binding) {
    titleText.text = item.title
    subtitleText.text = item.subtitle
}
```

### 스코프 함수 선택 규칙

| 상황 | 선택 |
|------|------|
| nullable 객체 변환 | `?.let { }` |
| 객체 생성 후 설정 | `.apply { }` |
| 로깅/디버깅 삽입 | `.also { }` |
| 객체 설정 + 결과 필요 | `.run { }` |
| 스코프 함수 2단 이상 중첩 | **금지** — 가독성 저하, 지역 변수로 분리 |

```kotlin
// DON'T — 스코프 함수 중첩
user?.let { u ->
    u.address?.let { a ->          // 중첩 금지
        formatAddress(a)
    }
}

// DO — 평탄화
val address = user?.address ?: return
formatAddress(address)
```

---

## 확장 함수

기존 클래스에 유틸리티를 추가할 때 사용한다.

```kotlin
// DO — 변환 매퍼
fun UserDto.toDomain(): User = User(
    id = this.id,
    name = this.name,
    email = this.email,
)

// DO — 도메인 유틸리티
fun String.isValidEmail(): Boolean =
    this.matches(Regex("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$"))

// DO — 플랫폼 헬퍼
fun Context.showToast(message: String) {
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
}
```

| 규칙 | 설명 |
|------|------|
| 변환 함수에 적극 활용 | `toEntity()`, `toDomain()`, `toUiModel()` |
| 수신 타입의 의미에 부합해야 함 | `String.isValidEmail()` O, `String.saveToDb()` X |
| 멤버 함수와 시그니처 중복 금지 | 멤버 함수가 항상 우선 호출됨 |
| 파일 위치 | 해당 모듈의 최상위 또는 `ext/` 패키지에 배치 |

---

## 컬렉션 연산

### 자주 쓰는 연산자

| 연산 | 용도 | 예시 |
|------|------|------|
| `map` | 변환 | `users.map { it.name }` |
| `filter` | 조건 필터 | `users.filter { it.isActive }` |
| `firstOrNull` | 조건에 맞는 첫 요소 | `users.firstOrNull { it.id == targetId }` |
| `groupBy` | 그룹핑 | `users.groupBy { it.department }` |
| `associate` | Map 변환 | `users.associate { it.id to it.name }` |
| `flatMap` | 중첩 리스트 평탄화 | `departments.flatMap { it.members }` |
| `sortedBy` | 정렬 | `users.sortedBy { it.name }` |
| `distinctBy` | 중복 제거 | `users.distinctBy { it.email }` |
| `any` / `none` / `all` | 조건 검사 | `users.any { it.isAdmin }` |

### 컬렉션 빌더

```kotlin
// DO — 조건부 요소 포함
val items = buildList {
    add(alwaysItem)
    if (showOptional) add(optionalItem)
    addAll(dynamicItems)
}

val config = buildMap {
    put("key", value)
    if (debug) put("debug", "true")
}
```

### 체이닝 규칙

```kotlin
// DO — 각 연산을 줄 바꿈
val activeAdmins = users
    .filter { it.isActive }
    .filter { it.role == Role.ADMIN }
    .sortedBy { it.name }
    .map { it.toUiModel() }

// DON'T — 한 줄에 모든 체이닝
val activeAdmins = users.filter { it.isActive }.filter { it.role == Role.ADMIN }.sortedBy { it.name }.map { it.toUiModel() }
```

---

## Sequence

대용량 컬렉션에서 중간 리스트 생성을 방지한다.

```kotlin
// DO — 대용량 데이터 처리 시 sequence
val result = largeList.asSequence()
    .filter { it.isValid }
    .map { it.transform() }
    .take(10)
    .toList()
```

| 사용 시점 | 설명 |
|----------|------|
| 원소 1000개 이상 + 체이닝 2단계 이상 | sequence 고려 |
| 원소 수가 적거나 체이닝 1단계 | 일반 컬렉션 사용 |
| `first()`, `take()` 등 단축 평가 | sequence 효과 큼 |

---

## 구조 분해 (Destructuring)

```kotlin
// DO — data class 구조 분해
val (id, name, email) = user

// DO — Map 순회
for ((key, value) in map) {
    println("$key = $value")
}

// DO — Pair/Triple 대신 data class
data class LoadResult(val data: List<Item>, val hasMore: Boolean)
val (data, hasMore) = loadItems()

// DON'T — Pair 남용 (의미 불명확)
fun loadItems(): Pair<List<Item>, Boolean>   // first? second?
```

---

## 기타 이디엄

### require / check

```kotlin
// 함수 진입 시 사전조건 검증
fun setAge(age: Int) {
    require(age >= 0) { "Age must be non-negative: $age" }
}

// 상태 검증
fun process() {
    check(isInitialized) { "Must call init() first" }
}
```

| 함수 | 용도 | 실패 시 |
|------|------|--------|
| `require` | 인자 검증 | `IllegalArgumentException` |
| `check` | 상태 검증 | `IllegalStateException` |
| `requireNotNull` | non-null 인자 검증 | `IllegalArgumentException` |
| `checkNotNull` | non-null 상태 검증 | `IllegalStateException` |

### sealed when 완전성

```kotlin
// DO — 모든 분기 명시, else 없음
fun handle(action: Action) = when (action) {
    is Action.Load -> load()
    is Action.Refresh -> refresh()
    is Action.Delete -> delete()
    // 새 Action 추가 시 컴파일 오류 → 누락 방지
}

// DON'T — else로 처리
fun handle(action: Action) = when (action) {
    is Action.Load -> load()
    else -> { }   // 새 Action이 무시됨
}
```

### 에이전트 체크리스트

- [ ] `!!` 사용이 없는가?
- [ ] `var` 대신 `val`을 사용했는가?
- [ ] mutable 컬렉션을 외부에 노출하지 않는가?
- [ ] sealed `when`에 `else`를 사용하지 않는가?
- [ ] 스코프 함수를 2단 이상 중첩하지 않는가?
- [ ] 확장 함수가 수신 타입의 의미에 부합하는가?
