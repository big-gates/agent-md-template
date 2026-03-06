# core / designsystem

> **트리거**: `core/designsystem/` 하위 파일을 수정할 때, 또는 테마·색상·타이포그래피·공통 위젯을 변경할 때 이 문서를 참조한다.

## 역할
- 테마, 색상, 타이포그래피, 공통 UI 컴포넌트를 제공한다.
- 프로젝트의 **모든** UI 컴포넌트는 이 모듈을 통해 구현한다.
- 앱 전반에서 공통 표준으로 사용되는 디자인 토큰의 단일 출처이다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `core/designsystem/src/commonMain/kotlin/{package}/theme/` | AppTheme, ColorScheme, Typography |
| `core/designsystem/src/commonMain/kotlin/{package}/component/` | 공통 위젯 (Button, Card, TextField 등) |
| `core/designsystem/src/commonMain/kotlin/{package}/icon/` | 아이콘 리소스 |

## 의존/연동
- `feature:*`, `core:ui`에서 사용한다.
- 신규/수정 UI는 이 모듈의 토큰과 컴포넌트를 반드시 사용해야 한다.

## 에이전트 판단 규칙

### 이 모듈 vs core:ui vs feature

| 질문 | 위치 |
|------|------|
| 앱 전체의 기초 위젯인가? (Button, Card, TextField) | `core:designsystem` |
| 2개 이상 feature에서 재사용하는 조합 컴포넌트인가? | `core:ui` |
| 한 feature에서만 사용하는 컴포넌트인가? | 해당 `feature` 모듈 |

### 새 컴포넌트를 추가할 때

| 단계 | 수행 |
|------|------|
| 1 | Material Design 3 가이드에 해당 컴포넌트가 있는지 확인 |
| 2 | 있으면 M3 컴포넌트를 래핑하여 앱 테마 적용 |
| 3 | 없으면 M3 토큰(색상, 타이포, 간격)을 사용하여 커스텀 생성 |
| 4 | `component/`에 배치, `modifier: Modifier = Modifier` 파라미터 포함 |

## DO / DON'T

```kotlin
// DO — M3 컴포넌트를 앱 테마로 래핑
@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
) {
    Button(
        onClick = onClick,
        modifier = modifier,
        enabled = enabled,
        colors = ButtonDefaults.buttonColors(
            containerColor = MaterialTheme.colorScheme.primary,
        ),
    ) {
        Text(text, style = MaterialTheme.typography.labelLarge)
    }
}

// DO — 테마 토큰 사용
Text(
    text = title,
    style = MaterialTheme.typography.titleMedium,
    color = MaterialTheme.colorScheme.onSurface,
)

// DON'T — 하드코딩된 색상/폰트
Text(
    text = title,
    fontSize = 18.sp,           // 금지: typography 토큰 사용
    color = Color(0xFF333333),  // 금지: colorScheme 토큰 사용
)

// DON'T — 비즈니스 로직 포함
@Composable
fun ItemCard(item: Item, viewModel: HomeViewModel) {
    // 금지: designsystem에 비즈니스 로직/ViewModel 유입
}
```

## 변경 시 체크리스트
- [ ] Material Design 3 가이드 원칙을 따르는가?
- [ ] 색상/폰트가 하드코딩되지 않고 테마 토큰을 사용하는가?
- [ ] `modifier: Modifier = Modifier` 파라미터가 포함되어 있는가?
- [ ] 비즈니스 로직이나 도메인 의존이 유입되지 않았는가?
- [ ] 변경이 전 화면에 미치는 영향을 확인했는가?
