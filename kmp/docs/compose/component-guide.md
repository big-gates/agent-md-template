# Compose 컴포넌트 작성 Guide

> Modifier, 이벤트 처리, 네이밍, 파일 구성, 다이얼로그 패턴 규칙이다.

---

## Modifier 파라미터

모든 공개 Composable에 `modifier: Modifier = Modifier`를 포함한다.

| # | 규칙 |
|---|------|
| 1 | `modifier`는 첫 번째 **선택적** 파라미터 (필수 파라미터 뒤) |
| 2 | 기본값은 `Modifier` (빈 Modifier) |
| 3 | 전달받은 `modifier`는 **최상위 레이아웃**에만 적용 |
| 4 | 내부 고정 스타일은 별도 Modifier 체인으로 작성 |

```kotlin
// DO
@Composable
fun InfoRow(label: String, value: String, modifier: Modifier = Modifier) {
    Row(modifier = modifier.fillMaxWidth().padding(8.dp)) {
        Text(label, modifier = Modifier.weight(1f))  // 내부 modifier 별도
        Text(value)
    }
}
```

---

## 이벤트 처리

이벤트는 **람다 콜백**으로 전달한다. ViewModel을 하위 컴포넌트에 전달하지 않는다.

```kotlin
// DO — 이벤트 콜백 람다
@Composable
fun ApprovalCard(
    approval: ApprovalUi,
    onApprove: (Long) -> Unit,
    onReject: (Long) -> Unit,
    modifier: Modifier = Modifier,
) { /* ... */ }

// DON'T — ViewModel 직접 전달
@Composable
fun ApprovalCard(approval: ApprovalUi, viewModel: ApprovalViewModel) { /* 금지 */ }
```

---

## Composable 네이밍

| 유형 | 네이밍 | 역할 | 예시 |
|------|--------|------|------|
| Route 진입점 | `{Name}Route` | ViewModel 주입, 상태 수집, 이벤트 처리, 네비게이션 콜백 | `HomeRoute`, `LoginRoute` |
| Screen | `{Name}Screen` | UI 레이아웃. Route에서 받은 상태/콜백으로 화면 구성 | `HomeScreen`, `LoginScreen` |
| Content | `{Name}Content` | Screen의 하위 섹션. 상태 분기별 UI 또는 스크롤 가능한 본문 | `HomeContent`, `ErrorContent` |
| 위젯/카드 | `{Domain}{Type}` | 재사용 단위 컴포넌트 | `AttendanceStatsCard`, `NoticeWidget` |
| 다이얼로그 | `{Name}Dialog` | 모달 다이얼로그 | `ConfirmDeleteDialog` |
| 테이블 행 | `{Domain}TableRow` | 리스트/테이블 행 | `NoticeTableRow` |
| 값 반환 Composable | camelCase 동사 | 상태/값을 반환하는 Composable | `rememberCalendarState()` |

> Route → Screen → Content 계층 상세는 [navigation-guide.md](navigation-guide.md)를 참조한다.

| 규칙 | 설명 |
|------|------|
| PascalCase 명사 | `@Composable`이 Unit 반환 시 |
| camelCase 동사 | `@Composable`이 값 반환 시 |
| 도메인 접두사 | 소속을 명확히 구분 |

---

## 파일 구성

### 파일 내부 순서

| 순서 | 내용 |
|------|------|
| 1 | Screen 진입점 (파일 최상단) |
| 2 | 주요 콘텐츠 Composable |
| 3 | 하위 컴포넌트 |
| 4 | Preview (있을 경우) |

### 가시성 규칙

| 대상 | 가시성 | 이유 |
|------|--------|------|
| Screen 진입점 | `internal` 또는 패키지 수준 | NavHost에서 참조 |
| `component/` 내 Composable | `internal` 또는 패키지 수준 | 같은 feature 내 사용 |
| Screen 내 보조 Composable | `private` | 외부 노출 불필요 |
| `ui/component/` 공용 Composable | `public` | 모든 feature에서 사용 |

---

## 다이얼로그 패턴

| # | 규칙 |
|---|------|
| 1 | 다이얼로그 표시 상태(`showDialog`)는 Screen에서 관리한다 |
| 2 | 다이얼로그 Composable은 `onConfirm`/`onDismiss` 콜백을 받는다 |
| 3 | 다이얼로그 내부에서 ViewModel을 직접 참조하지 않는다 |

```kotlin
// 다이얼로그 정의
@Composable
fun ConfirmDeleteDialog(
    targetName: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("삭제 확인") },
        text = { Text("${targetName}을(를) 삭제하시겠습니까?") },
        confirmButton = { TextButton(onClick = onConfirm) { Text("삭제") } },
        dismissButton = { TextButton(onClick = onDismiss) { Text("취소") } },
    )
}

// Screen에서 사용
var showDeleteDialog by remember { mutableStateOf(false) }
if (showDeleteDialog) {
    ConfirmDeleteDialog(
        targetName = target.name,
        onConfirm = { viewModel.delete(target.id); showDeleteDialog = false },
        onDismiss = { showDeleteDialog = false },
    )
}
```
