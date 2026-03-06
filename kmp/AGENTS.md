# AGENTS.md

## 목적
- 이 파일은 KMP 프로젝트의 전반 지침만 관리한다.
- 모듈/기능별 세부 지침은 `docs/architecture` 하위 문서에서 관리한다.
- 모듈 경로(`core/*`, `feature/*`, `androidapp`, `iosApp`, `shared`)에 `AGENTS.md`를 분산 생성하지 않는다.

---

## 에이전트 문서 라우팅

> 작업 대상 파일 경로를 보고, 아래 테이블에서 **반드시 읽어야 할 문서**를 먼저 참조한다.

| 작업 대상 경로 패턴 | 읽어야 할 문서 |
|-------------------|--------------|
| `androidapp/**` | [app/androidapp.md](docs/architecture/app/androidapp.md) |
| `iosApp/**` | [app/iosapp.md](docs/architecture/app/iosapp.md) |
| `shared/**` | [app/shared.md](docs/architecture/app/shared.md) |
| `feature/*/` (새 생성) | [feature/README.md](docs/architecture/feature/README.md) |
| `feature/{name}/**` | `docs/architecture/feature/{name}.md` + [feature/README.md](docs/architecture/feature/README.md) |
| `core/domain/**` | [core/domain.md](docs/architecture/core/domain.md) |
| `core/data/**` | [core/data.md](docs/architecture/core/data.md) |
| `core/datasource/**` | [core/datasource.md](docs/architecture/core/datasource.md) |
| `**/entity/**`, `**/dao/**`, `**/migration/**` | [core/datasource.md](docs/architecture/core/datasource.md) (Local DataSource 섹션) |
| `**/dto/**`, `**/service/**`, `**/client/**` | [core/datasource.md](docs/architecture/core/datasource.md) (Remote DataSource 섹션) |
| `**/mapper/**`, `**/*Mapper*.kt` | [core/data.md](docs/architecture/core/data.md) (매퍼 섹션) |
| `core/designsystem/**` | [core/designsystem.md](docs/architecture/core/designsystem.md) |
| `core/ui/**` | [core/ui.md](docs/architecture/core/ui.md) |
| `core/common/**` | [core/common.md](docs/architecture/core/common.md) |
| `**/strings.xml`, `**/composeResources/**` | [localization.md](docs/convention/localization.md) |
| `**/*Route.kt`, `**/*Screen.kt`, `**/*Content.kt` | [compose-guide.md](docs/compose/compose-guide.md), [navigation-guide.md](docs/compose/navigation-guide.md), [udf-guide.md](docs/reactive/udf-guide.md) |
| `**/*Navigation.kt` | [navigation-guide.md](docs/compose/navigation-guide.md) |
| `**/*ViewModel.kt` | [coroutine-guide.md](docs/coroutine/coroutine-guide.md), [udf-guide.md](docs/reactive/udf-guide.md), [stream-guide.md](docs/reactive/stream-guide.md) |
| `**/*Repository*.kt` | [core/domain.md](docs/architecture/core/domain.md), [core/data.md](docs/architecture/core/data.md) |
| `**/*DataSource*.kt` | [core/datasource.md](docs/architecture/core/datasource.md) |
| `**/*UseCase*.kt` | [core/domain.md](docs/architecture/core/domain.md) |
| `**/*Test.kt`, `**/*Test*.kt` | [testing-guide.md](docs/convention/testing-guide.md) |
| `**/di/**`, `**/*Module.kt` | [di-guide.md](docs/convention/di-guide.md) |

---

## 프로젝트 전반 원칙

### 아키텍처
- 클린 아키텍처를 반드시 고수한다. 레이어 경계를 넘는 직접 참조/의존은 금지한다.
- 의존 방향 (모듈 Gradle 의존):

| from | to | 관계 |
|------|----|------|
| `app` | `shared` | 의존 |
| `shared` | `feature:*`, `core:*` | DI 조합용 의존 |
| `feature:*` | `core:domain` | 의존 (UseCase, Repository 인터페이스) |
| `feature:*` | `core:ui`, `core:designsystem` | 의존 (UI 컴포넌트) |
| `core:data` | `core:domain` | 의존 (인터페이스 구현, **의존성 역전**) |
| `core:data` | `core:datasource` | 의존 (DataSource 사용) |
- `core:domain`이 Repository 인터페이스를 정의하고, `core:data`가 구현한다 (의존성 역전).
- **`core:domain`은 어떤 하위 레이어에도 의존하지 않는다** (`core:common` 제외).
- KMP 소스셋(`commonMain`, `androidMain`, `iosMain`)을 유지한다.
- 계층 간 데이터 이동은 반드시 **매퍼(Mapper)** 를 통해 수행한다. Entity/DTO를 직접 상위 레이어로 전달하지 않는다.
- `core:domain` 모델 변경 시 `core:data`의 매퍼/소비 모듈 호환성을 함께 검토한다.
- `core:datasource` 인프라(Entity, DTO, 스키마) 변경 시 매퍼를 함께 갱신한다.
- 플랫폼 전용 변경(Android, iOS)은 공용 모듈(`shared`)과의 초기화/라이프사이클 연동 영향을 같이 확인한다.

### TDD
- **TDD(Test-Driven Development)** 를 준수한다.
- 새 기능/버그 수정 시 테스트를 먼저 작성한다 (Red -> Green -> Refactor).
- Fake 기반 테스트를 선호한다. 인터페이스 분리가 Fake 교체를 가능하게 한다.
- 테스트 작성 규칙은 [testing-guide.md](docs/convention/testing-guide.md)를 참조한다.

### 데이터
- 동기화/저장소 정책은 Offline First(로컬 우선, 서버 동기화)를 반드시 유지한다.
- 오프라인 큐 전략은 `docs/architecture/core/data.md`를 기준으로 관리한다.

### UI
- 모든 UI 컴포넌트는 `core:designsystem` 모듈을 반드시 사용한다.
- `core:designsystem`은 Material Design 3 가이드 원칙을 반드시 따른다.
- 모든 UI 텍스트는 로컬라이제이션을 반드시 적용한다. 하드코딩된 문자열 대신 `composeResources`를 사용하여 다국어를 지원한다.
- UI 컴포넌트는 해당 기능의 `feature/*` 모듈에서 우선 생성한다.
- 두 개 이상 feature에서 재사용되는 공통 컴포넌트는 `core/ui`로 반드시 승격한다.

---

## 에이전트 판단 규칙

### 새 파일/클래스를 어디에 만들어야 하는가?

| 만들려는 것 | 위치 | 근거 |
|-----------|------|------|
| 화면 Composable | `feature/{name}/` | 화면 단위 캡슐화 |
| ViewModel | `feature/{name}/` | 화면에 1:1 대응 |
| 도메인 모델 (data class) | `core/domain/model/` | 순수 비즈니스 모델 |
| UseCase | `core/domain/usecase/` | 비즈니스 로직 단위 |
| Repository 인터페이스 | `core/domain/repository/` | 의존성 역전 |
| Repository 구현 | `core/data/repository/` | 프레임워크 의존 허용 |
| 매퍼 (Mapper) | `core/data/mapper/` | 계층 간 모델 변환 |
| Local/Remote DataSource | `core/datasource/` | 데이터 원천 접근 |
| DB Entity / DAO | `core/datasource/local/entity/`, `local/dao/` | 로컬 DB 인프라 |
| API Service / DTO | `core/datasource/remote/service/`, `remote/model/` | 네트워크 인프라 |
| 공용 UI (2+ feature 사용) | `core/ui/` | 승격 대상 |
| 테마/토큰/기본 위젯 | `core/designsystem/` | 앱 전역 표준 |
| Logger / 유틸 / Dispatcher | `core/common/` | 횡단 관심사 |
| Fake (테스트용) | `{module}/src/commonTest/` | 인터페이스의 테스트 구현체 |

### 의존을 추가해도 되는가?

| from -> to | 허용? |
|----------|------|
| `feature` -> `core:domain` | O |
| `feature` -> `core:ui` | O |
| `feature` -> `core:designsystem` | O |
| `feature` -> `core:data` | **X** (domain 인터페이스를 통해 접근) |
| `feature` -> `core:datasource` | **X** |
| `feature` -> 다른 `feature` | **X** |
| `core:data` -> `core:domain` | O (인터페이스 구현, 의존성 역전) |
| `core:data` -> `core:datasource` | O |
| `core:domain` -> `core:data` | **X** (역방향 금지, 의존성 역전 위반) |
| `core:domain` -> `core:datasource` | **X** |
| 모든 모듈 -> `core:common` | O (횡단) |

---

## 세부 지침 문서 위치

| 문서 | 읽어야 할 때 |
|------|-------------|
| [docs/architecture/README.md](docs/architecture/README.md) | 모듈 인덱스 확인, 새 모듈 문서 등록 시 |
| [docs/architecture/app/](docs/architecture/app/) | 플랫폼 앱(Android/iOS) 또는 shared 모듈 변경 시 |
| [docs/architecture/core/](docs/architecture/core/) | core 레이어 모듈 변경 시 |
| [docs/architecture/feature/README.md](docs/architecture/feature/README.md) | feature 모듈 생성·구조·규칙 확인 시 |
| [docs/architecture/feature/_template.md](docs/architecture/feature/_template.md) | 새 feature 문서 작성 시 |
| [docs/convention/localization.md](docs/convention/localization.md) | 다국어 리소스, 톤앤매너 규칙 확인 시 |
| [docs/kotlin/kotlin-guide.md](docs/kotlin/kotlin-guide.md) | Kotlin 네이밍, 타입, 이디엄 규칙 확인 시 |
| [docs/compose/compose-guide.md](docs/compose/compose-guide.md) | Composable 작성 규칙 확인 시 |
| [docs/compose/navigation-guide.md](docs/compose/navigation-guide.md) | Route/Screen/Content 계층, 네비게이션 그래프, 인자 전달, 이벤트 기반 네비게이션 확인 시 |
| [docs/coroutine/coroutine-guide.md](docs/coroutine/coroutine-guide.md) | 코루틴/Flow 패턴 확인 시 |
| [docs/reactive/reactive-guide.md](docs/reactive/reactive-guide.md) | 리액티브 프로그래밍 원칙, SOT, 불변 상태, 선언적 변환 확인 시 |
| [docs/reactive/udf-guide.md](docs/reactive/udf-guide.md) | UDF/MVI 패턴, State vs Event 분리, Action 디스패치 확인 시 |
| [docs/reactive/stream-guide.md](docs/reactive/stream-guide.md) | 레이어 간 Flow 연결, 스트림 합성(combine/flatMapLatest), 백프레셔 확인 시 |
| [docs/convention/testing-guide.md](docs/convention/testing-guide.md) | TDD 워크플로우, Fake/Mock, 코루틴 테스트, 테스트 DI 확인 시 |
| [docs/convention/di-guide.md](docs/convention/di-guide.md) | DI 모듈 등록, 스코프 선택, 테스트 DI 오버라이드, Koin 설정 확인 시 |

## 문서 운영 규칙
- 모듈별 지침을 추가/수정할 때는 `docs/architecture` 하위 문서를 갱신한다.
- 새 모듈이 생기면 `docs/architecture/<영역>/<모듈>.md`를 추가하고 `docs/architecture/README.md`에 등록한다.
- 새 DataSource 인프라 모듈을 추가하면 `docs/architecture/core/datasource-{tech}.md` 문서를 작성한다 ([datasource.md](docs/architecture/core/datasource.md) > 인프라 모듈 추가 가이드 참조).
- 오프라인 큐 타입/재시도 정책을 바꿀 때는 `docs/architecture/core/data.md`를 함께 갱신한다.
- 새 feature 모듈을 추가하면 `docs/architecture/feature/{name}.md` 문서를 작성하고, `feature/README.md`와 `docs/architecture/README.md`에 등록한다.
- 매퍼 추가/변경 시 매퍼 테스트를 반드시 함께 작성한다.
