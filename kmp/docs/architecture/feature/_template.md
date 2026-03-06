# feature / {name}

> **트리거**: `feature/{name}/` 하위 파일을 수정할 때, 또는 해당 기능의 화면/로직을 변경할 때 이 문서를 참조한다.
> 공통 feature 규칙은 [README.md](README.md)를 함께 참조한다.

---

## 역할

{기능에 대한 1~2줄 설명}

- **{주요 기능 1}**: {설명}
- **{주요 기능 2}**: {설명}

---

## 모듈 구조

| 경로 (`feature/{name}/src/commonMain/kotlin/{package}/`) | 설명 |
|---------------------------------------------------------|------|
| `{Name}Screen.kt` | Route + Screen Composable |
| `{Name}ViewModel.kt` | 상태 관리 |
| `{Name}UiState.kt` | UI 상태 + Action |
| `component/` | feature 전용 하위 컴포넌트 |
| `di/{Name}ViewModelModule.kt` | DI 등록 |
| `navigation/{Name}Navigation.kt` | 네비게이션 Route |

---

## 화면 구성

| 화면 | 파일 | 설명 |
|------|------|------|
| {Name}Route | `{Name}Screen.kt` | ViewModel 주입, 상태 수집, 이벤트 처리 |
| {Name}Screen | `{Name}Screen.kt` | 메인 UI 레이아웃 |
| {Name}Content | `{Name}Screen.kt` | 상태별 분기 UI |

---

## ViewModel

### {Name}ViewModel

| 항목 | 내용 |
|------|------|
| 의존성 | `{Foo}UseCase`, `{Bar}Repository` |
| UiState | `data class {Name}UiState(items, isLoading, error)` |
| Action | `sealed interface {Name}Action { Load, Delete(id), Retry }` |
| Event | `sealed interface {Name}Event { NavigateBack, ShowError(message) }` |

| 메서드 | 설명 |
|--------|------|
| `onAction(action)` | Action 디스패치 |
| `{비즈니스 메서드}` | {설명} |

---

## 네비게이션

| Route | 화면 | 비고 |
|-------|------|------|
| `{Name}Route` | {Name}Screen | 진입점 |

---

## DI 등록

```kotlin
// {Name}ViewModelModule.kt
val {name}ViewModelModule = module {
    viewModelOf(::{Name}ViewModel)
}
```

---

## 의존 모듈

| 모듈 | 용도 |
|------|------|
| `core:domain` | UseCase, Repository 인터페이스, 도메인 모델 |
| `core:designsystem` | 테마, 컴포넌트 |
| `core:ui` | 공용 UI 컴포넌트 |
| `core:common` | 유틸리티 |

---

## 변경 시 체크리스트

- [ ] UDF 패턴(Action -> ViewModel -> State -> UI)을 준수하는가?
- [ ] ViewModel에서 `CancellationException`을 재throw하는가?
- [ ] 모든 UI 텍스트가 `composeResources`를 사용하는가?
- [ ] DI 모듈에 새 ViewModel을 등록했는가?
- [ ] Navigation 그래프에 화면을 등록했는가?
- [ ] ViewModel 유닛 테스트를 작성했는가?
- [ ] {feature 고유 체크 항목}
