# Compose Effect / 성능 Guide

> LaunchedEffect, DisposableEffect, 리컴포지션 최적화 규칙이다.

---

## LaunchedEffect

| 패턴 | 사용 시점 |
|------|----------|
| `LaunchedEffect(Unit) { }` | 최초 1회 초기화 |
| `LaunchedEffect(key) { }` | 키 변경 시 재실행 |
| 이벤트 콜백 (`onClick = { }`) | 리컴포지션마다 호출해야 하는 로직 |

```kotlin
// DO — 키가 변경될 때만 재실행
LaunchedEffect(userId) { viewModel.loadUser(userId) }

// DO — 최초 1회
LaunchedEffect(Unit) { viewModel.initialize() }

// DON'T — 의도 불명확
LaunchedEffect(true) { viewModel.load() }
```

| # | 규칙 |
|---|------|
| 1 | 키(key)는 effect 재실행 조건을 명확히 반영한다 |
| 2 | 일회성 초기화는 `LaunchedEffect(Unit)` |
| 3 | 리컴포지션마다 호출할 로직은 이벤트 콜백에 배치한다 |

---

## DisposableEffect

리소스 정리가 필요할 때 사용한다. 반드시 `onDispose { }` 블록을 포함한다.

```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> /* ... */ }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}
```

---

## 리컴포지션 최적화

### 람다 안정화

```kotlin
// DO — remember로 람다 안정화
val onItemClick = remember<(Long) -> Unit> {
    { id -> viewModel.selectItem(id) }
}
```

### LazyColumn / LazyRow

| # | 규칙 |
|---|------|
| 1 | `items()`에 항상 `key`를 지정한다 |
| 2 | 아이템 파라미터는 안정적 타입 사용 |
| 3 | 아이템 내부에서 인덱스 기반 상태 접근을 피한다 |

```kotlin
// DO — key 지정
LazyColumn {
    items(employees, key = { it.id }) { employee ->
        EmployeeRow(employee)
    }
}

// DON'T — key 없음 (위치 기반 비교)
LazyColumn {
    items(employees) { employee -> EmployeeRow(employee) }
}
```

### 무거운 컴포넌트

```kotlin
// DO — 조건부 렌더링
if (isExpanded) {
    HeavyDetailView(data = detailData)
}

// DON'T — AnimatedVisibility 내부에서 비용 큰 계산
AnimatedVisibility(visible = isExpanded) {
    val data = expensiveComputation()  // 금지
    HeavyDetailView(data = data)
}
```
