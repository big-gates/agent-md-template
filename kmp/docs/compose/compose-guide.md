# Compose Guide

> Compose UI 코드 작성 시 참조할 인덱스. 세부 규칙은 하위 문서를 참조한다.

---

## 하위 문서

| 문서 | 읽어야 할 때 |
|------|-------------|
| [상태 관리](state-guide.md) | 상태 호이스팅, collectAsState, remember, derivedStateOf, 안정적 타입 |
| [컴포넌트 작성](component-guide.md) | Modifier, 이벤트 처리, 네이밍, 파일 구성, 다이얼로그 |
| [Effect / 성능](effect-guide.md) | LaunchedEffect, DisposableEffect, 리컴포지션 최적화, LazyColumn |
| [네비게이션](navigation-guide.md) | Route/Screen/Content 계층, 네스트 그래프, 인자 전달, 이벤트 기반 네비게이션 |

---

## 금지 패턴

| 금지 패턴 | 대안 |
|----------|------|
| Composable 내부에서 ViewModel 생성 (`val vm = FooViewModel()`) | `koinInject()` 또는 파라미터 |
| remember 내부에서 side-effect (`remember { api.fetch() }`) | `LaunchedEffect` |
| Composable에서 직접 코루틴 실행 (`scope.launch { }`) | `LaunchedEffect` 또는 이벤트 콜백 |
| 하위 컴포넌트에 ViewModel 전달 (`ChildView(viewModel)`) | 필요한 상태/콜백만 전달 |
| mutableStateOf를 remember 없이 사용 | `remember { mutableStateOf(false) }` |
| Composable 파라미터에 var 상태 (`fun Foo(var text: String)`) | 상태 + 콜백 쌍 `(text, onTextChange)` |

---

## 에이전트용 빠른 참조

| 상황 | 패턴 |
|------|------|
| Screen에서 상태 구독 | `val x by vm.uiState.collectAsStateWithLifecycle()` |
| 하위 컴포넌트 작성 | 상태 + 콜백 파라미터, `modifier: Modifier = Modifier` 포함 |
| 일회성 초기화 | `LaunchedEffect(Unit) { vm.init() }` |
| 키 변경 시 재실행 | `LaunchedEffect(key) { vm.load(key) }` |
| 비용 큰 계산 캐싱 | `remember(deps) { compute() }` |
| 리스트 아이템 | `items(list, key = { it.id }) { }` |
| 다이얼로그 표시 | `var show by remember { mutableStateOf(false) }` + 조건부 렌더링 |
| 컴포넌트 분리 | Screen > 300줄이면 `component/`로 분리 |
