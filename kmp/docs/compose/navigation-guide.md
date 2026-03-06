# Compose Navigation Guide

> **트리거**: `*Navigation.kt`, `*Route*`, 네비게이션 그래프를 추가·수정할 때, 또는 화면 간 이동 로직을 변경할 때 이 문서를 참조한다.

---

## Composable 계층 규칙

화면은 반드시 **Route → Screen → Content/Component** 3단 계층으로 구성한다.

| 계층 | 네이밍 | 역할 | ViewModel 접근 |
|------|--------|------|---------------|
| **Route** | `{Name}Route` | 진입점. ViewModel 주입, 상태 수집, 이벤트 처리, 네비게이션 콜백 연결 | O (여기서만) |
| **Screen** | `{Name}Screen` | UI 레이아웃. Route에서 받은 상태와 콜백으로 화면 구성 | X |
| **Content** | `{Name}Content` | Screen의 하위 섹션. 특정 상태 조합에 따른 UI 분기 | X |
| **Component** | `{Name}Card`, `{Name}Row` 등 | 재사용 가능한 단위 위젯 | X |

```kotlin
// Route — ViewModel 주입, 상태 수집, 이벤트 처리
@Composable
fun HomeRoute(
    viewModel: HomeViewModel = koinViewModel(),
    navigateToDetail: (Long) -> Unit,
    onShowSnackbar: suspend (String, String?) -> Boolean,
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is HomeEvent.NavigateToDetail -> navigateToDetail(event.id)
                is HomeEvent.ShowError -> onShowSnackbar(event.message, null)
            }
        }
    }

    HomeScreen(
        state = uiState,
        onAction = viewModel::onAction,
    )
}

// Screen — 순수 UI, ViewModel 참조 없음
@Composable
private fun HomeScreen(
    state: HomeUiState,
    onAction: (HomeAction) -> Unit,
) {
    when {
        state.isLoading -> LoadingContent()
        state.error != null -> ErrorContent(
            message = state.error,
            onRetry = { onAction(HomeAction.Retry) },
        )
        else -> HomeContent(
            items = state.items,
            onItemClick = { onAction(HomeAction.SelectItem(it)) },
            onDelete = { onAction(HomeAction.Delete(it)) },
        )
    }
}

// Content — Screen의 하위 섹션
@Composable
private fun HomeContent(
    items: List<ItemUi>,
    onItemClick: (Long) -> Unit,
    onDelete: (Long) -> Unit,
    modifier: Modifier = Modifier,
) {
    LazyColumn(modifier = modifier) {
        items(items, key = { it.id }) { item ->
            ItemRow(item = item, onClick = { onItemClick(item.id) })
        }
    }
}
```

### 계층 간 규칙

| 규칙 | 설명 |
|------|------|
| ViewModel은 Route에서만 주입 | Screen, Content, Component에 ViewModel을 전달하지 않는다 |
| Screen은 상태 + 콜백만 수신 | `state: UiState`, `onAction: (Action) -> Unit` 패턴 |
| Content는 필요한 데이터만 수신 | 전체 UiState가 아닌 해당 섹션에 필요한 데이터만 파라미터로 전달 |
| 네비게이션 콜백은 Route에서 처리 | Screen/Content는 네비게이션을 알지 못한다 |

---

## Route 정의

### Route 클래스

모든 Route는 `@Serializable`로 선언한다. `kotlinx.serialization`을 사용하여 타입 안전 네비게이션을 구현한다.

```kotlin
// 인자 없는 Route
@Serializable
data object HomeRoute

// 인자 있는 Route
@Serializable
data class DetailRoute(val productId: Long)

// 복합 객체 인자 Route
@Serializable
data class AddProductRoute(val product: ManagedProduct)
```

| Route 유형 | 패턴 | 예시 |
|-----------|------|------|
| 인자 없음 | `data object` | `HomeRoute`, `LoginRoute`, `SettingsRoute` |
| 단순 인자 | `data class` + 원시 타입 | `DetailRoute(val id: Long)` |
| 복합 인자 | `data class` + `CustomNavType` | `AddProductRoute(val product: ManagedProduct)` |
| 네스트 그래프 루트 | `data object` (Base) | `RegisterBaseRoute`, `SettingsBaseRoute` |

### 네스트 그래프용 Base Route

여러 화면을 하나의 흐름으로 묶을 때 Base Route를 사용한다.

```kotlin
@Serializable data object RegisterBaseRoute       // 그래프 루트
@Serializable data object RegisterChatRoute        // 시작 화면
@Serializable data object RegisterEmailRoute       // 단계별 화면
@Serializable data object RegisterPasswordRoute
```

---

## Navigation 파일 구조

각 feature는 `navigation/` 디렉토리에 네비게이션 파일을 둔다.

| 경로 | 설명 |
|------|------|
| `feature/{name}/src/commonMain/kotlin/{package}/navigation/{Name}Navigation.kt` | Route 정의 + NavGraphBuilder + NavController 확장 |
| `feature/{name}/src/commonMain/kotlin/{package}/{Name}Screen.kt` | Route + Screen Composable |
| `feature/{name}/src/commonMain/kotlin/{package}/component/` | 하위 컴포넌트 |

### Navigation 파일 구성

하나의 `{Name}Navigation.kt` 파일에 다음을 포함한다:

```kotlin
// 1. Route 클래스 정의
@Serializable data object HomeRoute

// 2. NavController 확장 함수 (navigate*)
fun NavController.navigateToHome(navOptions: NavOptions) {
    navigate(route = HomeRoute, navOptions = navOptions)
}

// 3. NavGraphBuilder 확장 함수 (그래프 등록)
fun NavGraphBuilder.home(
    navigateToDetail: (Long) -> Unit,
    onShowSnackbar: suspend (String, String?) -> Boolean,
) {
    composable<HomeRoute> {
        HomeRoute(
            navigateToDetail = navigateToDetail,
            onShowSnackbar = onShowSnackbar,
        )
    }
}
```

---

## 네스트 그래프

여러 화면이 하나의 흐름을 구성할 때(회원가입 단계, 설정 > 약관 등) 네스트 그래프를 사용한다.

```kotlin
fun NavGraphBuilder.registerGraph(
    navigateToHome: () -> Unit,
    navController: NavHostController,
    // ... 네비게이션 콜백들
) {
    navigation<RegisterBaseRoute>(startDestination = RegisterChatRoute) {
        composable<RegisterChatRoute> { /* ... */ }
        composable<RegisterEmailRoute> { /* ... */ }
        composable<RegisterPasswordRoute> { /* ... */ }
        // ...
    }
}
```

| 규칙 | 설명 |
|------|------|
| `navigation<BaseRoute>(startDestination = ...)` | Base Route로 그래프를 감싼다 |
| 외부에서는 Base Route로만 진입 | 내부 Route에 직접 navigate하지 않는다 (그래프 내부에서만) |
| `startDestination`이 첫 화면 | 그래프 진입 시 표시되는 화면 |

---

## 복합 객체 인자 전달

### CustomNavType

원시 타입이 아닌 객체를 Route 인자로 전달할 때 `CustomNavType`을 사용한다.

```kotlin
class CustomNavType<T : Any>(
    private val serializer: KSerializer<T>,
    override val isNullableAllowed: Boolean = false,
) : NavType<T>(isNullableAllowed) {
    override fun parseValue(value: String): T =
        Json.decodeFromString(serializer, value)
    override fun serializeAsValue(value: T): String =
        Json.encodeToString(serializer, value)
    override fun put(bundle: SavedState, key: String, value: T) {
        bundle.write { putString(key, serializeAsValue(value)) }
    }
    override fun get(bundle: SavedState, key: String): T? =
        bundle.read { parseValue(getString(key)) }
    override val name: String = serializer.descriptor.serialName
}

inline fun <reified T : Any> navigationCustomArgument(
    isNullable: Boolean = false,
): Pair<KType, CustomNavType<T>> {
    return typeOf<T>() to CustomNavType(serializer(), isNullable)
}
```

### 사용 패턴

```kotlin
// Route 정의
@Serializable
data class AddProductRoute(val product: ManagedProduct)

// 그래프 등록 시 typeMap 지정
composable<AddProductRoute>(
    typeMap = mapOf(navigationCustomArgument<ManagedProduct>())
) {
    AddProductRoute(/* ... */)
}

// navigate 시 URL 인코딩 (URL 포함 필드가 있는 경우)
fun NavController.navigateToAddProduct(
    product: ManagedProduct,
    navOptions: NavOptionsBuilder.() -> Unit = {},
) {
    val encoded = product.copy(
        productUrl = UrlEncoder.encode(product.productUrl),
        imageUrl = product.imageUrl?.let { UrlEncoder.encode(it) },
    )
    navigate(route = AddProductRoute(encoded)) { navOptions() }
}
```

| 규칙 | 설명 |
|------|------|
| `@Serializable` 모델만 전달 가능 | CustomNavType이 JSON 직렬화를 사용 |
| URL 포함 필드는 인코딩 | `UrlEncoder.encode()`로 URL 문자열을 인코딩한 뒤 전달 |
| ViewModel에서 `SavedStateHandle`로 수신 | Route 인자는 ViewModel의 SavedStateHandle에 자동 주입 |

---

## 공유 ViewModel 스코프

다단계 흐름(회원가입 등)에서 여러 화면이 하나의 ViewModel을 공유할 때, 네스트 그래프의 backstack entry에 스코프한다.

```kotlin
composable<RegisterEmailRoute> {
    // 네스트 그래프(RegisterBaseRoute)의 backstack entry에 ViewModel 스코프
    val registerBackStackEntry = remember {
        navController.getBackStackEntry(RegisterBaseRoute::class)
    }
    val viewModel: RegisterViewModel = koinViewModel(
        viewModelStoreOwner = registerBackStackEntry,
    )
    RegisterEmailRoute(viewModel = viewModel, /* ... */)
}
```

| 규칙 | 설명 |
|------|------|
| 네스트 그래프 전체에 걸친 상태 공유 | 회원가입처럼 여러 단계의 데이터를 누적하는 경우 |
| Base Route의 backstack entry 사용 | `navController.getBackStackEntry(BaseRoute::class)` |
| 그래프 이탈 시 자동 정리 | 네스트 그래프를 나가면 ViewModel도 소멸 |
| 단일 화면은 개별 ViewModel | 공유가 불필요한 화면은 각자 ViewModel 사용 |

---

## 이벤트 기반 네비게이션

ViewModel의 일회성 이벤트(Event)로 네비게이션을 트리거한다. Route에서 `LaunchedEffect`로 이벤트를 수집한다.

```kotlin
@Composable
fun LoginRoute(
    viewModel: LoginViewModel = koinViewModel(),
    navigateToHome: () -> Unit,
    onShowSnackbar: suspend (String, String?) -> Boolean,
) {
    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is LoginEvent.Success -> navigateToHome()
                is LoginEvent.Error -> onShowSnackbar(event.message, null)
            }
        }
    }

    LoginScreen(onLoginClick = { viewModel.onAction(LoginAction.Login) })
}
```

| 규칙 | 설명 |
|------|------|
| 네비게이션은 Event로 트리거 | State가 아닌 Event(Channel)로 일회성 네비게이션 |
| `LaunchedEffect(Unit)`로 수집 | Route의 생명주기에 맞춰 이벤트를 수집 |
| Screen은 네비게이션을 모름 | Screen은 `onAction` 콜백만 호출, 네비게이션은 Route가 처리 |

---

## TopLevelDestination

앱 하단 네비게이션의 최상위 목적지를 정의한다.

```kotlin
sealed class TopLevelDestination(
    val route: KClass<*>,
    val baseRoute: KClass<*> = route,
) {
    data object Home : TopLevelDestination(route = HomeRoute::class)
    data object Settings : TopLevelDestination(
        route = SettingsRoute::class,
        baseRoute = SettingsBaseRoute::class,   // 네스트 그래프인 경우
    )
}
```

### 최상위 이동 NavOptions

```kotlin
// 최상위 목적지 간 이동 시 — 상태 저장/복원
navOptions {
    popUpTo(navController.graph.findStartDestination().id) {
        saveState = true
    }
    launchSingleTop = true
    restoreState = true
}

// 인증 완료 후 전체 스택 정리
navOptions {
    popUpTo(navController.graph.id) {
        inclusive = true     // 전체 백스택 제거
    }
    launchSingleTop = true
}
```

| NavOptions | 용도 |
|-----------|------|
| `saveState` + `restoreState` | 탭 간 이동 시 상태 유지 (하단 네비게이션) |
| `popUpTo(graph.id) { inclusive = true }` | 인증 후 전체 백스택 정리 (로그인 → 홈) |
| `launchSingleTop = true` | 동일 목적지 중복 방지 |

---

## DO / DON'T

```kotlin
// DO — Route에서만 ViewModel 주입
@Composable
fun ItemRoute(viewModel: ItemViewModel = koinViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    ItemScreen(state = state, onAction = viewModel::onAction)
}

// DO — Screen은 상태 + 콜백만 수신
@Composable
private fun ItemScreen(state: ItemUiState, onAction: (ItemAction) -> Unit) {
    // 순수 UI
}

// DO — Route 클래스는 @Serializable
@Serializable data object ItemRoute

// DON'T — Screen에서 ViewModel 직접 주입
@Composable
fun ItemScreen(viewModel: ItemViewModel = koinViewModel()) {  // 금지
    // Screen이 ViewModel에 의존
}

// DON'T — Screen에서 네비게이션 직접 호출
@Composable
fun ItemScreen(navController: NavController) {  // 금지
    Button(onClick = { navController.navigate(DetailRoute(id)) })
}

// DON'T — Content/Component에 ViewModel 전달
@Composable
fun ItemCard(viewModel: ItemViewModel) {  // 금지
    // 하위 컴포넌트에 ViewModel 전달
}

// DON'T — navigate 함수를 직접 호출 (NavController 확장 함수 사용)
navController.navigate(route = HomeRoute)        // 지양
navController.navigateToHome(navOptions)          // 선호
```

---

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 화면 추가 | Route 클래스 정의 → Navigation 파일에 NavGraphBuilder 확장 추가 → Route Composable + Screen Composable 작성 |
| 여러 화면이 하나의 흐름 | 네스트 그래프(Base Route) 사용, 공유 ViewModel은 backstack entry 스코프 |
| 복합 객체를 다음 화면에 전달 | `CustomNavType` + `@Serializable` 모델 + `typeMap` 등록 |
| 화면 이동 후 백스택 정리 | `popUpTo(graph.id) { inclusive = true }` |
| 하단 탭 간 이동 | `saveState`/`restoreState` + `launchSingleTop` |
| ViewModel에서 네비게이션 트리거 | Event(Channel) 발행 → Route의 `LaunchedEffect`에서 수집 → navigate 호출 |

---

## 변경 시 체크리스트

- [ ] Route 클래스에 `@Serializable`이 있는가?
- [ ] Navigation 파일에 Route 정의, NavController 확장, NavGraphBuilder 확장이 모두 있는가?
- [ ] Route Composable에서만 ViewModel을 주입하는가? (Screen/Content에 전달 금지)
- [ ] Screen Composable은 상태 + 콜백만 수신하는가?
- [ ] 복합 객체 인자에 `CustomNavType` + `typeMap`을 등록했는가?
- [ ] URL 포함 필드를 인코딩했는가?
- [ ] 다단계 흐름에서 공유 ViewModel을 Base Route의 backstack entry에 스코프했는가?
- [ ] 네비게이션이 Event(Channel)로 트리거되는가? (State로 트리거 금지)
- [ ] `shared`의 NavHost에 새 feature 그래프를 등록했는가?
- [ ] NavController 확장 함수(navigate*)를 작성했는가?
