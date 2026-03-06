# Compose 상태 관리 Guide

> 상태 호이스팅, Flow 구독, remember, 안정적 타입 규칙이다.

---

## 상태 호이스팅 (State Hoisting)

Composable은 **상태를 직접 보유하지 않고**, 호출자로부터 받는다.

```kotlin
// DO — 상태를 파라미터로 받는다
@Composable
fun EmployeeCard(
    employee: EmployeeUi,
    onEdit: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Card(modifier = modifier) {
        Text(employee.name)
        IconButton(onClick = onEdit) { Icon(Icons.Default.Edit, null) }
    }
}

// DON'T — Composable 내부에서 ViewModel 직접 접근
@Composable
fun EmployeeCard(viewModel: EmployeeViewModel) {
    val employee by viewModel.selected.collectAsState()  // 금지
}
```

### 예외: Screen 진입점

Screen 진입점에서만 ViewModel을 주입하고 UiState를 구독한다.

```kotlin
@Composable
fun HrHomeScreen(viewModel: HrHomeViewModel = koinInject()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // Screen에서 UiState를 읽고, 하위 컴포넌트에 필요한 값만 전달
    EmployeeList(
        employees = uiState.employees,
        onRefresh = viewModel::refresh,
    )
}
```

---

## collectAsStateWithLifecycle

StateFlow를 Compose에서 구독할 때 **반드시** `collectAsStateWithLifecycle()`을 사용한다.

| 패턴 | 사용 |
|------|------|
| `collectAsStateWithLifecycle()` | **기본** — 생명주기 인식 구독 |
| `collectAsState()` | Desktop에서 lifecycle 미지원 시에만 허용 |

---

## 안정적 타입 (Stable Types)

Compose 컴파일러가 스마트 리컴포지션을 수행하려면 파라미터가 **안정적**이어야 한다.

| 타입 | 안정성 |
|------|--------|
| 기본 타입 (`Int`, `String`, `Boolean` 등) | 안정적 |
| `data class` (모든 프로퍼티가 `val` + 안정적 타입) | 안정적 |
| `List<T>` (일반 kotlin List) | **불안정** |
| `ImmutableList<T>` (kotlinx.collections.immutable) | 안정적 |
| 함수 타입 (`() -> Unit`) | 안정적 |

**에이전트 판단**: 기본적으로 `List<T>`를 사용한다. 성능 이슈가 실측될 때만 `ImmutableList`를 도입한다.

---

## remember / derivedStateOf

| 상황 | 사용 |
|------|------|
| 비용 큰 계산/객체 생성 | `remember(keys) { }` |
| 상태 A → 상태 B 파생, B가 A보다 덜 자주 변경 | `derivedStateOf { }` |
| 단순 값 읽기 | 불필요 |

```kotlin
// remember — 비용 큰 계산 캐싱
val filteredList = remember(employees, searchQuery) {
    employees.filter { it.name.contains(searchQuery, ignoreCase = true) }
}

// derivedStateOf — 파생 상태
val isEmpty by remember {
    derivedStateOf { employees.isEmpty() }
}
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```
