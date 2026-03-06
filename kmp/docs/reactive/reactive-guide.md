# Reactive Programming Guide

> 리액티브 프로그래밍 원칙과 Flow 기반 반응형 아키텍처 지침이다. 데이터가 흐르고, UI는 상태에 반응한다.

---

## 하위 문서

| 문서 | 읽어야 할 때 |
|------|-------------|
| [UDF 가이드](udf-guide.md) | 단방향 데이터 흐름, State/Event 분리, MVI 패턴 |
| [스트림 설계 가이드](stream-guide.md) | 레이어 간 반응형 연결, 스트림 합성, 백프레셔 |

---

## 리액티브 프로그래밍이란

**데이터의 변화를 선언적으로 전파**하는 프로그래밍 패러다임이다. 값을 직접 꺼내오는(pull) 대신, 값이 변할 때 자동으로 흘러오도록(push) 설계한다.

| 계층 | 역할 | 전달 |
|------|------|------|
| DataSource | 변경 발생 | `Flow` |
| Repository | 변환/병합 | `Flow` |
| ViewModel | 상태 생성 | `StateFlow` |
| UI | 렌더링 | `collectAsStateWithLifecycle()` |

### 명령형 vs 리액티브

```kotlin
// 명령형 — 직접 꺼내와서 갱신 (pull)
fun refresh() {
    val users = repo.getUsers()       // 한번 호출, 한번 결과
    _uiState.value = users.toUi()     // 수동 갱신
}

// 리액티브 — 변화에 자동 반응 (push)
val uiState: StateFlow<UiState> = repo.observeUsers()   // 변경 시 자동 emit
    .map { it.toUi() }                                    // 선언적 변환
    .stateIn(viewModelScope, WhileSubscribed(5_000), UiState())
// DB/서버 데이터가 변경되면 UI까지 자동 전파
```

---

## 핵심 원칙

### 1. 단일 진실 공급원 (Single Source of Truth)

모든 데이터는 **하나의 출처**에서만 생성되고 변경된다.

| 계층 | SOT 역할 |
|------|---------|
| `LocalDataSource` | DB/메모리의 현재 상태를 StateFlow로 보유 |
| `Repository` | SOT를 노출만, 자체 상태를 보유하지 않음 |
| `ViewModel` | UI 상태를 StateFlow로 보유, SOT에서 파생 |

### 2. 불변 상태 (Immutable State)

상태는 불변 객체로 표현한다. 상태를 변경하려면 새 객체를 생성한다.

```kotlin
// DO — data class copy로 새 상태 생성
_uiState.update { current ->
    current.copy(isLoading = false, items = newItems)
}

// DON'T — 상태 내부를 직접 변경
_uiState.value.items.add(newItem)   // 금지: MutableList 직접 변경
```

### 3. 선언적 변환 (Declarative Transformation)

데이터 파이프라인을 연산자 체인으로 **선언**한다. 로직이 아닌 **의도**를 표현한다.

```kotlin
// DO — 선언적: "무엇"을 원하는지 표현
val activeUsers: StateFlow<List<UserUi>> = userRepo.observeUsers()
    .map { users -> users.filter { it.isActive } }
    .map { users -> users.map { it.toUi() } }
    .stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())

// DON'T — 명령적: "어떻게" 하는지 나열
fun loadActiveUsers() {
    viewModelScope.launch {
        val users = userRepo.getUsers()
        val filtered = mutableListOf<UserUi>()
        for (user in users) {
            if (user.isActive) filtered.add(user.toUi())
        }
        _activeUsers.value = filtered
    }
}
```

### 4. 부수 효과 격리 (Side Effect Isolation)

순수한 데이터 변환 파이프라인과 부수 효과(API 호출, DB 쓰기, 네비게이션)를 분리한다.

```kotlin
// 순수 변환 파이프라인 (부수 효과 없음)
val filteredItems: StateFlow<List<Item>> = combine(
    repository.observeItems(),
    searchQuery,
) { items, query ->
    items.filter { it.name.contains(query, ignoreCase = true) }
}.stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())

// 부수 효과는 명시적인 함수로 분리
fun deleteItem(id: Long) {
    viewModelScope.launch {
        repository.deleteItem(id)   // 부수 효과: DB 삭제
        // SOT가 변경되면 filteredItems가 자동 갱신
    }
}
```

---

## 금지 패턴

| 금지 패턴 | 문제 | 대안 |
|----------|------|------|
| 수동 새로고침으로 상태 동기화 | 상태 불일치, 갱신 누락 | SOT에서 Flow로 자동 전파 |
| ViewModel 간 직접 참조 | 결합도 상승, 생명주기 문제 | 공유 데이터는 DataSource(SOT)에서 각 VM이 독립 구독 |
| MutableList/MutableMap으로 상태 관리 | 변경 감지 불가, 리컴포지션 미발생 | 불변 data class + `copy()` |
| collect 내부에서 다른 Flow collect | 중첩 구독, 누수 위험 | `combine`, `flatMapLatest` |
| 콜백으로 데이터 전달 | 리액티브 체인 파괴 | Flow로 변환 (`callbackFlow`) |
| 여러 곳에서 같은 상태를 독립 관리 | SOT 부재, 정합성 깨짐 | 단일 SOT에서 파생 |

---

## 에이전트용 빠른 참조

| 상황 | 패턴 |
|------|------|
| DB 변경 → UI 자동 반영 | `dao.observeAll(): Flow<List<Entity>>` → Repository → ViewModel `stateIn` |
| 여러 소스 결합 | `combine(flowA, flowB) { a, b -> merge(a, b) }` |
| 검색어 입력 | `searchQuery.debounce(300).distinctUntilChanged().flatMapLatest { }` |
| 일회성 명령 (삭제, 저장) | suspend fun → SOT 변경 → Flow가 자동 전파 |
| 화면 간 데이터 공유 | DataSource(SOT) → Repository 노출 → 각 ViewModel 독립 구독 |
| UI 이벤트 (토스트, 네비게이션) | `SharedFlow` 또는 `Channel` (일회성 소비) |
