# core / ui

> **트리거**: `core/ui/` 하위 파일을 수정할 때, 또는 2개 이상 feature에서 공유하는 UI 컴포넌트를 승격할 때 이 문서를 참조한다.

## 역할
- 2개 이상 feature에서 재사용되는 공용 UI 컴포넌트, 에러 화면 등을 제공한다.
- feature 전용 → 공용으로 **승격**되는 컴포넌트의 수용 모듈이다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `core/ui/src/commonMain/kotlin/{package}/component/` | 공용 Composable 컴포넌트 |
| `core/ui/src/commonMain/kotlin/{package}/error/` | 공용 에러/빈 상태 화면 |

## 의존/연동
- `feature:*`에서 사용한다.
- 이 모듈의 컴포넌트는 `core:designsystem` 토큰/컴포넌트를 기반으로 구현한다.

## 에이전트 판단 규칙

### 이 컴포넌트는 어디에 있어야 하는가?

| 질문 | Yes | No |
|------|-----|-----|
| 앱 전체의 기초 위젯인가? (Button, Card, Theme) | `core:designsystem` | 다음 질문으로 |
| 2개 이상 feature에서 사용하는가? | `core:ui` (승격) | `feature/{name}/component/` |

### 승격 판단 기준

| 조건 | 판단 |
|------|------|
| feature A에서만 사용 | **승격하지 않음** — feature에 유지 |
| feature A, B에서 동일하게 사용 | **승격** — `core:ui`로 이동 |
| 승격 후 feature 특화 분기가 필요 | `core:ui`에 범용 버전, feature에 래핑 버전 |

### 승격 절차

| 단계 | 수행 |
|------|------|
| 1 | feature 모듈에서 컴포넌트를 `core:ui`로 이동 |
| 2 | feature 특화 로직을 제거하고 범용 파라미터로 교체 |
| 3 | `core:designsystem` 토큰 기반으로 스타일 정리 |
| 4 | 기존 feature 모듈의 import 경로를 갱신 |

## DO / DON'T

```kotlin
// DO — 범용 파라미터, designsystem 토큰 사용
@Composable
fun EmptyStateView(
    title: String,
    description: String,
    icon: ImageVector = Icons.Default.Info,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        Icon(icon, contentDescription = null)
        Text(title, style = MaterialTheme.typography.titleMedium)
        Text(description, style = MaterialTheme.typography.bodyMedium)
    }
}

// DON'T — feature 특화 비즈니스 로직 유입
@Composable
fun EmptyStateView(
    itemType: ItemType,   // 금지: 특정 도메인에 종속
) {
    val message = when (itemType) {
        ItemType.FOOD -> "식재료가 없어요"   // 금지: 하드코딩 + 도메인 의존
        ItemType.DRINK -> "음료가 없어요"
    }
}

// DON'T — designsystem 토큰 없이 하드코딩
@Composable
fun ErrorBanner(...) {
    Text(color = Color.Red, fontSize = 14.sp)   // 금지
}
```

## 변경 시 체크리스트
- [ ] `core:designsystem` 토큰/컴포넌트를 기반으로 구현했는가?
- [ ] feature 특화 비즈니스 로직이 유입되지 않았는가?
- [ ] `modifier: Modifier = Modifier` 파라미터가 포함되어 있는가?
- [ ] 공용 UI 변경이 사용 중인 모든 화면에 미치는 영향을 확인했는가?
- [ ] 텍스트가 `composeResources`로 로컬라이징되어 있는가?
