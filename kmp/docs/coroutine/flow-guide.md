# Coroutine Flow Guide

> StateFlow 노출, SOT 구독, stateIn 변환, collect/first 선택, Flow 연산자 규칙이다.

---

## 규칙 1: StateFlow 상태 노출 패턴

```kotlin
// DO — private Mutable + public 읽기전용
private val _uiState = MutableStateFlow(HomeUiState())
val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

// DON'T — MutableStateFlow를 직접 노출
val uiState = MutableStateFlow(HomeUiState())   // 외부에서 .value 수정 가능
```

**UI에서 구독:**
```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

---

## 규칙 2: 여러 화면에서 같은 데이터를 구독할 때

**DataSource가 SOT이고, 각 ViewModel이 독립 구독한다.**

```kotlin
// DataSource — StateFlow를 보유 (SOT)
class FooLocalDataSource {
    private val _foos = MutableStateFlow<List<Foo>>(emptyList())
    val foos: StateFlow<List<Foo>> = _foos.asStateFlow()
    fun update(data: List<Foo>) { _foos.value = data }
}

// Repository — DataSource를 노출
class FooRepositoryImpl(
    private val localDataSource: FooLocalDataSource,
    private val remoteService: FooService,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : FooRepository {
    override val foos: StateFlow<List<Foo>> = localDataSource.foos
    override suspend fun refresh() {
        val response = withContext(ioDispatcher) {
            remoteService.getFoos()
        }
        localDataSource.update(response.map { it.toDomain() })
    }
}

// 각 ViewModel에서 독립 구독
class ScreenAViewModel(repo: FooRepository) : ViewModel() {
    val foos = repo.foos
}
class ScreenBViewModel(repo: FooRepository) : ViewModel() {
    val foos = repo.foos
}

// DON'T — ViewModel을 공유하지 않는다
// DON'T — feature/common/에 데이터 캐시 Store를 만들지 않는다
```

---

## 규칙 3: stateIn 변환

Repository의 Flow를 ViewModel에서 StateFlow로 변환할 때:

```kotlin
val data: StateFlow<List<Item>> = repository.observeItems()
    .map { items -> items.filter { it.isActive } }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = emptyList()
    )
```

| 파라미터 | 값 | 이유 |
|---------|---|------|
| `scope` | `viewModelScope` | ViewModel 소멸 시 자동 정리 |
| `started` | `WhileSubscribed(5_000)` | 마지막 구독자 사라진 후 5초간 업스트림 유지 (화면 회전 대응) |
| `initialValue` | non-null 기본값 | 구독 시작 전 초기 상태 |

---

## 규칙 4: collect vs collectLatest vs first

| 연산자 | 사용 시점 | 예시 |
|--------|----------|------|
| `first()` | 일회성 호출 (가장 흔한 케이스) | `repository.getFoo().first()` |
| `collect` | 모든 값을 순서대로 처리 | 상태 변경 스트림 구독 |
| `collectLatest` | 최신 값만 중요, 이전 처리 취소 | 검색어 입력에 따른 결과 갱신 |

**판단 기준**: Repository가 `flow { emit(한번) }` 패턴이면 `first()`. 연속 emit이면 `collect`.

---

## 규칙 5: Flow 연산자 활용

### 변환 연산자

```kotlin
// map — 데이터 변환
val uiItems: Flow<List<ItemUi>> = repository.observeItems()
    .map { items -> items.map { it.toUi() } }

// filter — 조건 필터
val activeUsers: Flow<List<User>> = repository.observeUsers()
    .filter { it.isNotEmpty() }

// combine — 여러 Flow 결합
val uiState: StateFlow<HomeUiState> = combine(
    repository.observeItems(),
    repository.observeCategories(),
    searchQuery,
) { items, categories, query ->
    HomeUiState(
        items = items.filter { it.name.contains(query) },
        categories = categories,
    )
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = HomeUiState()
)
```

### 에러 처리 연산자

```kotlin
// catch — 업스트림 에러를 처리 (다운스트림은 정상 동작)
repository.observeItems()
    .catch { e ->
        if (e is CancellationException) throw e
        emit(emptyList())   // 에러 시 빈 리스트 emit
    }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

### 디바운스 / 중복 제거

```kotlin
// debounce — 검색 입력 시 (마지막 입력 후 일정 시간 대기)
searchQuery
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query -> repository.search(query) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

| 연산자 | 용도 |
|--------|------|
| `debounce(ms)` | 마지막 emit 후 일정 시간 대기 (검색) |
| `distinctUntilChanged()` | 연속 동일 값 무시 |
| `flatMapLatest` | 새 값이 오면 이전 Flow 취소 후 새 Flow 구독 |
| `conflate()` | 느린 수집자가 중간 값을 건너뜀 |

---

## 규칙 6: Flow는 cold, StateFlow/SharedFlow는 hot

| 특성 | Cold Flow (`flow { }`) | Hot Flow (`StateFlow`, `SharedFlow`) |
|------|----------------------|-------------------------------------|
| 시작 시점 | `collect` 호출 시 | 생성 즉시 (또는 첫 구독 시) |
| 구독자 수 | 각 구독자가 독립 실행 | 모든 구독자가 같은 값 공유 |
| 주 사용처 | Repository 반환 | ViewModel 상태 노출, DataSource SOT |

```kotlin
// Cold — 각 collect마다 API 호출이 실행됨
fun getUser(): Flow<User> = flow {
    emit(api.fetchUser())   // collect할 때마다 실행
}

// Hot — 하나의 상태를 여러 구독자가 공유
private val _user = MutableStateFlow<User?>(null)
val user: StateFlow<User?> = _user.asStateFlow()
```

---

## 에이전트 체크리스트

- [ ] MutableStateFlow를 외부에 직접 노출하지 않는가? (`asStateFlow()` 사용)
- [ ] 공유 데이터는 DataSource(SOT) → Repository → ViewModel 독립 구독 구조인가?
- [ ] `stateIn`에 `viewModelScope`, `WhileSubscribed(5_000)`, non-null 초기값을 사용하는가?
- [ ] 일회성 호출에 `first()`를 사용하는가?
- [ ] `combine`으로 여러 Flow를 합성할 때 `stateIn`으로 StateFlow 변환했는가?
- [ ] Flow `catch`에서 CancellationException을 재throw하는가?
