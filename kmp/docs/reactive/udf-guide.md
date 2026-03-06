# UDF (Unidirectional Data Flow) Guide

> 단방향 데이터 흐름, State와 Event 분리, MVI 패턴 규칙이다.

---

## 단방향 데이터 흐름 (UDF)

데이터는 한 방향으로만 흐른다. UI는 상태를 읽기만 하고, 사용자 액션은 ViewModel을 통해 상태를 변경한다.

| 단계 | from | to | 전달 |
|------|------|----|------|
| 1 | UI | ViewModel | Action (사용자 이벤트) |
| 2 | ViewModel | Repository | 데이터 요청/변경 |
| 3 | Repository | DataSource | 인프라 접근 |
| 4 | ViewModel | UI | State (상태 갱신) |
| 5 | UI | UI | 렌더링 |

| 방향 | 흐름 | 예시 |
|------|------|------|
| 위로 (UI → ViewModel) | 사용자 액션 | 버튼 클릭, 텍스트 입력, 스크롤 |
| 아래로 (ViewModel → UI) | 상태 | 로딩 여부, 데이터 리스트, 에러 메시지 |

---

## State vs Event

### State (상태) — 현재 화면의 스냅샷

```kotlin
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<ItemUi> = emptyList(),
    val error: String? = null,
)

// ViewModel에서 StateFlow로 노출
private val _uiState = MutableStateFlow(HomeUiState())
val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
```

| 특성 | 설명 |
|------|------|
| 항상 최신 값이 있음 | 구독 시 즉시 현재 상태를 받음 |
| 중복 무시 | 같은 값을 다시 emit해도 수집자에게 전달되지 않음 (`distinctUntilChanged`) |
| 화면 회전 후에도 유지 | StateFlow는 마지막 값을 보유 |
| `StateFlow`로 표현 | |

### Event (이벤트) — 일회성 발생, 한 번만 소비

```kotlin
sealed interface HomeEvent {
    data class ShowToast(val message: String) : HomeEvent
    data class NavigateTo(val route: String) : HomeEvent
    data object ScrollToTop : HomeEvent
}

// ViewModel에서 Channel로 노출
private val _event = Channel<HomeEvent>(Channel.BUFFERED)
val event: Flow<HomeEvent> = _event.receiveAsFlow()
```

| 특성 | 설명 |
|------|------|
| 한 번만 소비 | 토스트, 네비게이션, 스낵바 |
| 소비자가 없으면 버퍼에 대기 | 화면 재생성 시 소비 |
| `Channel` + `receiveAsFlow()`로 표현 | |

### State vs Event 판단 기준

| 질문 | State | Event |
|------|-------|-------|
| 화면 회전 후에도 보여야 하는가? | O | X |
| 여러 번 같은 값을 보내도 되는가? | O (중복 무시됨) | X (중복 전송 = 중복 실행) |
| 구독 시 마지막 값을 받아야 하는가? | O | X |

```kotlin
// State — 에러 메시지를 화면에 표시 (회전 후에도 유지)
_uiState.update { it.copy(error = "Network error") }

// Event — 토스트로 한 번만 표시
_event.send(HomeEvent.ShowToast("저장했어요"))
```

---

## MVI 패턴 (Model-View-Intent)

### 구조

| 단계 | from | to | 설명 |
|------|------|----|------|
| 1 | View | ViewModel | Action(Intent) 전달 |
| 2 | ViewModel | Model(Repository) | 데이터 요청/변경 |
| 3 | Model(Repository) | ViewModel | 결과 반환 |
| 4 | ViewModel | View | State 갱신 |
| 5 | ViewModel | View | Event 발행 (일회성) |

### Action 정의

```kotlin
sealed interface HomeAction {
    data object Load : HomeAction
    data object Refresh : HomeAction
    data class Search(val query: String) : HomeAction
    data class DeleteItem(val id: Long) : HomeAction
    data class ToggleFavorite(val id: Long) : HomeAction
}
```

### ViewModel 구현

```kotlin
class HomeViewModel(
    private val itemRepo: ItemRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    private val _event = Channel<HomeEvent>(Channel.BUFFERED)
    val event: Flow<HomeEvent> = _event.receiveAsFlow()

    // 단일 진입점: 모든 액션을 하나의 함수로 수신
    fun onAction(action: HomeAction) {
        when (action) {
            is HomeAction.Load -> load()
            is HomeAction.Refresh -> refresh()
            is HomeAction.Search -> search(action.query)
            is HomeAction.DeleteItem -> deleteItem(action.id)
            is HomeAction.ToggleFavorite -> toggleFavorite(action.id)
        }
    }

    private fun load() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val items = itemRepo.getItems().first()
                _uiState.update { it.copy(isLoading = false, items = items) }
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    private fun deleteItem(id: Long) {
        viewModelScope.launch {
            try {
                itemRepo.deleteItem(id)
                _event.send(HomeEvent.ShowToast("삭제했어요"))
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                _event.send(HomeEvent.ShowToast("삭제에 실패했어요"))
            }
        }
    }

    // ... 나머지 액션 처리
}
```

### Screen 구현

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = koinInject()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // Event 수집 (일회성)
    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is HomeEvent.ShowToast -> snackbarHostState.showSnackbar(event.message)
                is HomeEvent.NavigateTo -> navigator.navigate(event.route)
                is HomeEvent.ScrollToTop -> listState.animateScrollToItem(0)
            }
        }
    }

    // 초기 로드
    LaunchedEffect(Unit) {
        viewModel.onAction(HomeAction.Load)
    }

    // UI는 State만 읽고, Action만 전달
    HomeContent(
        state = uiState,
        onRefresh = { viewModel.onAction(HomeAction.Refresh) },
        onSearch = { viewModel.onAction(HomeAction.Search(it)) },
        onDeleteItem = { viewModel.onAction(HomeAction.DeleteItem(it)) },
        onToggleFavorite = { viewModel.onAction(HomeAction.ToggleFavorite(it)) },
    )
}
```

---

## UiState 설계 규칙

### 단일 UiState로 화면 전체를 표현

```kotlin
// DO — 하나의 data class로 화면 상태 전체 표현
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<ItemUi> = emptyList(),
    val searchQuery: String = "",
    val error: String? = null,
)

// DON'T — 상태를 여러 StateFlow로 분산
class HomeViewModel : ViewModel() {
    val isLoading = MutableStateFlow(false)   // 분산 → 정합성 깨짐 위험
    val items = MutableStateFlow(emptyList())
    val error = MutableStateFlow<String?>(null)
}
```

### 상태 갱신은 update로

```kotlin
// DO — 원자적 갱신
_uiState.update { current ->
    current.copy(isLoading = false, items = newItems, error = null)
}

// DON'T — 여러 번 나눠서 갱신 (중간 상태가 UI에 노출됨)
_uiState.value = _uiState.value.copy(isLoading = false)
_uiState.value = _uiState.value.copy(items = newItems)   // 두 번 리컴포지션
```

---

## 리액티브 UiState 파생

SOT의 Flow를 조합하여 UiState를 선언적으로 생성한다.

```kotlin
class HomeViewModel(
    private val itemRepo: ItemRepository,
    private val categoryRepo: CategoryRepository,
) : ViewModel() {

    private val searchQuery = MutableStateFlow("")

    // 리액티브 파생 — 소스가 변경되면 자동 갱신
    val uiState: StateFlow<HomeUiState> = combine(
        itemRepo.observeItems(),
        categoryRepo.observeCategories(),
        searchQuery,
    ) { items, categories, query ->
        HomeUiState(
            items = items
                .filter { it.name.contains(query, ignoreCase = true) }
                .map { it.toUi() },
            categories = categories.map { it.toUi() },
            searchQuery = query,
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = HomeUiState(),
    )

    fun onAction(action: HomeAction) {
        when (action) {
            is HomeAction.Search -> searchQuery.value = action.query
            is HomeAction.DeleteItem -> deleteItem(action.id)
            // ...
        }
    }
}
```

---

## 에이전트 체크리스트

- [ ] 데이터가 단방향으로 흐르는가? (Action 위로, State 아래로)
- [ ] 화면 상태가 단일 `data class`로 표현되는가?
- [ ] State(지속)와 Event(일회성)가 분리되어 있는가?
- [ ] Event에 `Channel` + `receiveAsFlow()`를 사용하는가?
- [ ] 상태 갱신에 `update { copy() }`를 사용하는가?
- [ ] ViewModel의 액션 처리가 `when`으로 분기되는가?
- [ ] 하위 Composable에 ViewModel을 전달하지 않는가? (State + 콜백만 전달)
