# feature 모듈 가이드

> **트리거**: `feature/` 하위 파일을 생성·수정할 때, 또는 새 feature 모듈을 만들 때 이 문서를 참조한다.
> 이 문서는 모든 feature에 적용되는 **공통 규칙**을 정의한다. 개별 feature의 화면·ViewModel·네비게이션 상세는 각 feature 문서를 참조한다.

## 개별 feature 문서

> 새 feature를 추가할 때 [_template.md](_template.md)를 복사하여 `{name}.md`로 작성한 뒤 아래 테이블에 등록한다.

| feature | 문서 |
|---------|------|
| _(예시)_ `feature/sample` | [sample.md](sample.md) |

---

## feature 모듈이란

하나의 사용자 기능(화면 단위)을 캡슐화하는 모듈이다.

| 경로 (`feature/{name}/src/commonMain/kotlin/{package}/`) | 설명 |
|---------------------------------------------------------|------|
| `{Name}Screen.kt` | Route + Screen Composable |
| `{Name}ViewModel.kt` | UI 상태 관리, 비즈니스 로직 호출 |
| `{Name}UiState.kt` | UI 상태 data class + Action sealed interface |
| `component/` | 이 feature 전용 하위 컴포넌트 |
| `navigation/{Name}Navigation.kt` | Route 정의 + NavGraphBuilder + NavController 확장 |

---

## 의존 규칙

| feature 모듈이 의존하는 대상 | 용도 |
|---------------------------|------|
| `core:domain` | 도메인 모델, UseCase, Repository 인터페이스 |
| `core:ui` | 공용 컴포넌트 |
| `core:designsystem` | 테마, 위젯 |

| 허용 | 금지 |
|------|------|
| `core:domain` (모델 / UseCase / Repository 인터페이스) | 다른 `feature:*` 직접 참조 |
| `core:ui` (공용 컴포넌트) | `core:datasource` 직접 접근 |
| `core:designsystem` (테마·위젯) | `core:data` 직접 접근 |

**원칙**: feature는 `core:domain`의 UseCase 또는 Repository 인터페이스를 통해서만 데이터에 접근한다. 구현 계층(data, datasource)을 직접 참조하지 않는다.

---

## UDF (Unidirectional Data Flow) 패턴

feature 모듈은 UDF 패턴을 따른다. 상세 규칙은 [udf-guide.md](../../reactive/udf-guide.md)를 참조한다.

| 단계 | 흐름 | 설명 |
|------|------|------|
| 1 | UI -> Action | 사용자 이벤트를 `sealed interface Action`으로 변환 |
| 2 | Action -> ViewModel | `onAction()` 디스패처가 Action을 수신 |
| 3 | ViewModel -> State | 비즈니스 로직 수행 후 `StateFlow` 갱신 |
| 4 | State -> UI | `collectAsStateWithLifecycle()`로 UI에 반영 |

### UiState + Action 정의

```kotlin
// {Name}UiState.kt
data class {Name}UiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
)

sealed interface {Name}Action {
    data object Load : {Name}Action
    data class Delete(val id: Long) : {Name}Action
    data object Retry : {Name}Action
}
```

---

## ViewModel 규칙

```kotlin
class {Name}ViewModel(
    private val getFoosUseCase: GetFoosUseCase,     // UseCase 주입
    private val fooRepository: FooRepository,        // 또는 Repository 인터페이스 주입
) : ViewModel() {

    private val _uiState = MutableStateFlow({Name}UiState())
    val uiState: StateFlow<{Name}UiState> = _uiState.asStateFlow()

    // Event (일회성 소비): 네비게이션, 스낵바 등
    private val _event = Channel<{Name}Event>()
    val event: Flow<{Name}Event> = _event.receiveAsFlow()

    fun onAction(action: {Name}Action) {
        when (action) {
            is {Name}Action.Load -> load()
            is {Name}Action.Delete -> delete(action.id)
            is {Name}Action.Retry -> load()
        }
    }

    private fun load() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                getFoosUseCase().collect { items ->
                    _uiState.update { it.copy(items = items, isLoading = false) }
                }
            } catch (e: CancellationException) {
                throw e   // 반드시 재throw
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
}
```

| 규칙 | 설명 |
|------|------|
| `viewModelScope`만 사용 | 자체 스코프 생성 금지 |
| `CancellationException` 재throw | 코루틴 취소 전파 보장 |
| `onAction` 디스패처 패턴 | 모든 UI 이벤트를 `sealed interface Action`으로 수신 |
| State와 Event 분리 | 지속 상태는 `StateFlow`, 일회성은 `Channel` |

---

## 에러 처리 패턴

```kotlin
// UiState에 에러 상태 포함
data class {Name}UiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,        // null이면 에러 없음
)

// Screen에서 에러 UI 표시
@Composable
fun {Name}Content(
    state: {Name}UiState,
    onAction: ({Name}Action) -> Unit,
) {
    when {
        state.isLoading -> LoadingIndicator()
        state.error != null -> ErrorView(
            message = state.error,
            onRetry = { onAction({Name}Action.Retry) },
        )
        else -> {Name}List(items = state.items, onAction = onAction)
    }
}
```

---

## Route -> Screen -> Content 계층

> 상세 규칙은 [navigation-guide.md](../../compose/navigation-guide.md)를 참조한다.

| 계층 | 역할 | ViewModel 참조 |
|------|------|---------------|
| `{Name}Route` | ViewModel 주입, 상태 수집, 이벤트 처리, 네비게이션 콜백 | O |
| `{Name}Screen` | UI 레이아웃. 상태/콜백만 받아 화면 구성 | X |
| `{Name}Content` | Screen 하위 섹션. 상태 분기별 UI 또는 스크롤 본문 | X |

```kotlin
// Route -- ViewModel 주입, 이벤트 처리
@Composable
fun {Name}Route(
    viewModel: {Name}ViewModel = koinViewModel(),
    navigateBack: () -> Unit,
    onShowSnackbar: suspend (String, String?) -> Boolean,
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is {Name}Event.NavigateBack -> navigateBack()
                is {Name}Event.ShowError -> onShowSnackbar(event.message, null)
            }
        }
    }

    {Name}Screen(state = uiState, onAction = viewModel::onAction)
}

// Screen -- 순수 UI, ViewModel 참조 없음
@Composable
private fun {Name}Screen(
    state: {Name}UiState,
    onAction: ({Name}Action) -> Unit,
) {
    when {
        state.isLoading -> LoadingContent()
        state.error != null -> ErrorContent(
            message = state.error,
            onRetry = { onAction({Name}Action.Retry) },
        )
        else -> {Name}Content(items = state.items, onAction = onAction)
    }
}
```

| 규칙 | 설명 |
|------|------|
| Route에서만 ViewModel 주입 | `koinViewModel()` 호출은 Route에서만 |
| Screen은 상태/콜백만 수신 | ViewModel 직접 참조 금지 |
| 하위 컴포넌트에 ViewModel 전달 금지 | 필요한 상태/콜백만 파라미터로 전달 |

---

## UI 규칙

1. **`core:designsystem` 컴포넌트를 반드시 사용한다.**
2. feature 전용 UI 컴포넌트는 이 모듈의 `component/`에 우선 생성한다.
3. 2개 이상 feature에서 재사용이 확인되면 `core:ui`로 승격한다.
4. 모든 UI 텍스트는 `composeResources`를 사용한다 (하드코딩 금지).

---

## 새 feature 추가 체크리스트

- [ ] `feature/{name}` Gradle 모듈 생성
- [ ] `build.gradle.kts`에 `core:domain`, `core:designsystem` 의존 추가
- [ ] `{Name}Route`, `{Name}Screen`, `{Name}ViewModel`, `{Name}UiState`, `{Name}Action` 생성
- [ ] ViewModel 에러 처리에서 `CancellationException` 재throw 확인
- [ ] feature DI 모듈 생성 (`{name}/di/{Name}ViewModelModule.kt`에 `viewModelOf()` 등록)
- [ ] `shared`의 `appModule`에 feature DI 모듈을 `includes`에 추가 ([di-guide.md](../../convention/di-guide.md) 참조)
- [ ] 개별 feature 문서 생성 (`docs/architecture/feature/{name}.md`) 후 README.md와 architecture/README.md에 등록
- [ ] Navigation 그래프에 화면 등록
- [ ] 문자열 리소스를 `composeResources`에 추가
- [ ] ViewModel 유닛 테스트 작성 (Fake Repository/UseCase 사용)
- [ ] UDF 패턴 준수 확인 (Action -> ViewModel -> State -> UI)
