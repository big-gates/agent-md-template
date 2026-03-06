# Reactive Stream 설계 Guide

> 레이어 간 반응형 연결, 스트림 합성, 백프레셔, 테스트 규칙이다.

---

## 레이어 간 반응형 연결

각 레이어가 Flow로 연결되어 SOT 변경이 UI까지 자동 전파된다.

| 계층 | 출력 | 역할 |
|------|------|------|
| Room DAO | `Flow<List<Entity>>` | `observeAll()` — DB 변경 시 자동 emit |
| DataSource | `StateFlow<List<T>>` | SOT 보유, Entity 노출 |
| Repository | `Flow<T>` | Flow 변환/합성, 도메인 모델 반환 |
| ViewModel | `StateFlow<UiState>` | `stateIn`으로 변환, UI 상태 생성 |
| UI | 렌더링 | `collectAsStateWithLifecycle()`으로 수집 |

### 계층별 책임

| 계층 | Flow 생산 | Flow 소비 | 변환 |
|------|----------|----------|------|
| **DAO** | `Flow<List<Entity>>` (Room 자동) | - | - |
| **DataSource** | `StateFlow<T>` (SOT) | DAO Flow 또는 API | Entity → Domain |
| **Repository** | SOT 노출, Flow 합성 | DataSource | 비즈니스 규칙 적용 |
| **ViewModel** | `StateFlow<UiState>` | Repository Flow | Domain → UiModel |
| **UI** | - | ViewModel StateFlow | 렌더링 |

---

## 스트림 합성 패턴

### combine — 여러 소스를 하나의 상태로 병합

```kotlin
// 모든 소스 중 하나라도 변경되면 재계산
val uiState: StateFlow<DashboardUiState> = combine(
    userRepo.observeProfile(),
    itemRepo.observeItems(),
    notificationRepo.observeUnreadCount(),
) { profile, items, unreadCount ->
    DashboardUiState(
        userName = profile.name,
        itemCount = items.size,
        unreadBadge = unreadCount,
    )
}.stateIn(viewModelScope, WhileSubscribed(5_000), DashboardUiState())
```

### flatMapLatest — 키가 바뀌면 이전 스트림 취소

```kotlin
// 선택된 카테고리가 바뀌면 이전 아이템 Flow 취소 → 새 Flow 구독
val items: StateFlow<List<ItemUi>> = selectedCategory
    .flatMapLatest { category ->
        itemRepo.observeByCategory(category.id)
    }
    .map { items -> items.map { it.toUi() } }
    .stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())
```

### merge — 여러 소스를 하나의 스트림으로 합류

```kotlin
// 로컬 알림과 원격 알림을 하나의 스트림으로
val allNotifications: Flow<Notification> = merge(
    localNotificationFlow,
    remoteNotificationFlow,
)
```

### 합성 연산자 선택

| 연산자 | 용도 | 소스 변경 시 |
|--------|------|-------------|
| `combine` | N개 Flow를 하나의 값으로 병합 | 어떤 소스든 변경되면 재계산 |
| `flatMapLatest` | 키 기반 Flow 전환 | 이전 Flow 취소, 새 Flow 구독 |
| `merge` | N개 Flow를 하나의 스트림으로 합류 | 모든 소스의 값을 독립 emit |
| `zip` | N개 Flow를 쌍으로 묶음 | 모든 소스가 emit해야 다음 값 생성 |

---

## 반응형 CRUD 패턴

### 쓰기 → SOT 변경 → 읽기 자동 갱신

```kotlin
// Repository
class ItemRepositoryImpl(
    private val dao: ItemDao,
    private val service: ItemService,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : ItemRepository {

    // 읽기 — Room의 Flow를 직접 노출 (SOT)
    override fun observeItems(): Flow<List<Item>> =
        dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    // 쓰기 — DB에 쓰면 observeItems()가 자동으로 새 리스트를 emit
    override suspend fun addItem(item: Item) = withContext(ioDispatcher) {
        dao.insert(item.toEntity())
        // observeItems() 구독자에게 자동 전파 — 수동 새로고침 불필요
    }

    override suspend fun deleteItem(id: Long) = withContext(ioDispatcher) {
        dao.deleteById(id)
        // 마찬가지로 자동 전파
    }
}

// ViewModel — 상태가 자동으로 갱신됨
class ItemListViewModel(private val repo: ItemRepository) : ViewModel() {
    val items: StateFlow<List<ItemUi>> = repo.observeItems()
        .map { items -> items.map { it.toUi() } }
        .stateIn(viewModelScope, WhileSubscribed(5_000), emptyList())

    fun delete(id: Long) {
        viewModelScope.launch {
            repo.deleteItem(id)
            // items StateFlow가 자동으로 갱신됨 — 별도 갱신 코드 불필요
        }
    }
}
```

---

## 백프레셔 (Backpressure)

수집자가 emit 속도를 따라가지 못할 때의 처리 전략이다.

| 전략 | 연산자 | 동작 |
|------|--------|------|
| 최신값만 유지 | `conflate()` | 중간 값 무시, 항상 최신 |
| 버퍼링 | `buffer(capacity)` | 지정된 용량만큼 버퍼, 초과 시 suspend |
| 이전 처리 취소 | `collectLatest { }` | 새 값이 오면 이전 처리 취소 |
| 샘플링 | `sample(intervalMs)` | 일정 간격마다 최신 값만 수집 |

```kotlin
// 센서 데이터처럼 빠르게 emit되는 Flow
sensorFlow
    .conflate()              // 느린 UI가 최신 값만 받음
    .collect { updateUi(it) }

// 검색 — 이전 검색 취소, 최신 쿼리만 실행
searchQuery
    .debounce(300)
    .collectLatest { query ->
        val results = repo.search(query)   // 새 쿼리 → 이전 search 취소
        _results.value = results
    }
```

### 백프레셔 전략 선택

| 상황 | 전략 |
|------|------|
| UI 갱신 (최신 상태만 중요) | `conflate()` 또는 `collectLatest` |
| 모든 이벤트 처리 필요 | `buffer()` |
| 고빈도 데이터 + 일정 간격 샘플링 | `sample(intervalMs)` |
| 사용자 입력 (타이핑) | `debounce(ms)` + `collectLatest` |

---

## 스트림 생명주기 관리

### WhileSubscribed — 구독자 없으면 업스트림 정지

```kotlin
val data: StateFlow<T> = upstream
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        //       구독자 사라진 후 5초 대기 → 업스트림 취소
        //       화면 회전(~1초)에는 유지, 화면 이탈(>5초)에는 정지
        initialValue = emptyData,
    )
```

| `SharingStarted` | 동작 | 사용 시점 |
|-------------------|------|----------|
| `WhileSubscribed(5_000)` | 마지막 구독자 후 5초 대기 | **기본값** — 대부분의 ViewModel |
| `Eagerly` | 즉시 시작, 스코프 취소까지 유지 | 앱 전역 상태, 항상 최신 필요 |
| `Lazily` | 첫 구독자 시 시작, 스코프 취소까지 유지 | 초기화 비용이 큰 데이터 |

---

## 금지 패턴

| 금지 패턴 | 문제 | 대안 |
|----------|------|------|
| collect 내부에서 collect 중첩 | 내부 Flow가 외부와 독립 수명 | `combine`, `flatMapLatest` |
| Flow에서 mutable 외부 상태 변경 | 스레드 안전 위반, 예측 불가 | `scan`, `runningFold`, StateFlow |
| `SharedFlow`로 상태 표현 | 초기값 없음, 구독 시 값 없음 | `StateFlow` |
| 매 이벤트마다 새 Flow 생성 | 메모리 누수, 구독 누적 | 단일 Flow 유지, 파라미터로 키 전달 |
| `launchIn` 남용 | 취소 관리 어려움 | `launch { flow.collect { } }` |

---

## 에이전트 체크리스트

- [ ] SOT(DataSource) → Repository → ViewModel → UI 단방향 흐름인가?
- [ ] 쓰기 작업 후 수동 새로고침 없이 SOT Flow가 자동 전파하는가?
- [ ] 여러 Flow 합성에 `combine`/`flatMapLatest`를 사용하는가?
- [ ] `collect` 내부에서 다른 `collect`를 중첩하지 않는가?
- [ ] `stateIn`에 `WhileSubscribed(5_000)`을 기본으로 사용하는가?
- [ ] 빠른 emit에 대한 백프레셔 전략이 있는가?
