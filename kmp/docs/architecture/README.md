# Architecture Docs Index

> 이 문서는 KMP(Kotlin Multiplatform) 클린 아키텍처 프로젝트의 모듈별 지침 인덱스입니다.
> 어떤 KMP 프로젝트에든 복사하여 사용할 수 있도록 범용적으로 작성되었습니다.

---

## 레이어 구조

| 레이어 | 모듈 | 설명 |
|--------|------|------|
| **App** | `androidapp`, `iosApp` | 플랫폼 런처, 매니페스트 |
| **App** | `shared` | KMP 공용 엔트리, DI, 브릿지 |
| **Presentation** | `feature:*` | 화면별 UI + ViewModel |
| **Presentation** | `core:ui` | 2개 이상 feature 공용 UI |
| **Presentation** | `core:designsystem` | 테마, 토큰, 공통 위젯 |
| **Domain** | `core:domain` | 도메인 모델 + UseCase + Repository 인터페이스 |
| **Data** | `core:data` | Repository 구현, Mapper, Sync |
| **Data** | `core:datasource` | DataSource + 인프라(DB, API) |
| **Cross-Cutting** | `core:common` | 유틸리티, 플랫폼 헬퍼 (모든 레이어에서 의존 가능) |

---

## 의존 방향

| from | to | 관계 |
|------|----|------|
| `app` | `shared` | 의존 |
| `shared` | `feature:*`, `core:*` | DI 조합용 의존 |
| `feature:*` | `core:domain` | 의존 (UseCase, Repository 인터페이스) |
| `feature:*` | `core:ui`, `core:designsystem` | 의존 (UI 컴포넌트) |
| `core:data` | `core:domain` | 의존 (인터페이스 구현, **의존성 역전**) |
| `core:data` | `core:datasource` | 의존 (DataSource 사용) |
| `core:datasource` | 인프라 라이브러리 | 내부 의존 (Room, SQLDelight, Ktor 등) |

| 규칙 | 설명 |
|------|------|
| **역방향 금지** | 하위 레이어가 상위 레이어를 참조할 수 없다 |
| **의존성 역전** | `core:domain`이 Repository 인터페이스를 정의하고, `core:data`가 구현한다. **`core:domain`은 `core:data`에 의존하지 않는다** |
| **feature 격리** | feature 간 직접 참조 금지. 공유 데이터는 domain/data 레이어를 통한다 |
| **횡단 모듈** | `core:common`은 어떤 레이어에서든 의존할 수 있다 |
| **매퍼 필수** | 계층 간 데이터 이동은 `core:data`의 mapper를 통해 수행. Entity/DTO를 상위 레이어에 직접 전달 금지 |

### 횡단 모듈 (Cross-Cutting)

`core:common`은 레이어와 무관하게 모든 모듈이 의존할 수 있는 유일한 횡단 모듈이다.

`core:common`에 의존하는 모듈: `feature:*`, `core:domain`, `core:data`, `core:datasource`, `core:ui`

제공 기능: 로깅, Dispatcher, 유틸리티, 확장 함수, 플랫폼 헬퍼(expect/actual)

### 계층 간 모델 매핑

| 모델 | 위치 | 용도 |
|------|------|------|
| Domain Model (`Item`) | `core:domain/model/` | 비즈니스 로직, UI 전달 |
| Entity (`ItemEntity`) | `core:datasource/local/entity/` | DB 테이블 매핑 |
| DTO (`ItemDto`) | `core:datasource/remote/model/` | API 직렬화 |
| Mapper | `core:data/mapper/` | 계층 간 변환 |

---

## 모듈별 문서

### app
| 모듈 | 문서 | 읽어야 할 때 |
|------|------|-------------|
| `androidapp` | [app/androidapp.md](app/androidapp.md) | Android 앱 초기화, 매니페스트, 플랫폼 연동 변경 시 |
| `iosApp` | [app/iosapp.md](app/iosapp.md) | iOS 진입점, 라이프사이클, 브릿지 변경 시 |
| `shared` | [app/shared.md](app/shared.md) | 공용 엔트리, DI, 플랫폼 초기화 변경 시 |

### core
| 모듈 | 문서 | 읽어야 할 때 |
|------|------|-------------|
| `core/domain` | [core/domain.md](core/domain.md) | 도메인 모델, UseCase, Repository 인터페이스 변경 시 |
| `core/data` | [core/data.md](core/data.md) | Repository 구현, Mapper, Offline-First, 동기화 정책 변경 시 |
| `core/datasource` | [core/datasource.md](core/datasource.md) | DataSource, Entity, DAO, DTO, API Service, 인프라 변경 시 |
| `core/designsystem` | [core/designsystem.md](core/designsystem.md) | 디자인 시스템 컴포넌트, 테마 토큰 변경 시 |
| `core/ui` | [core/ui.md](core/ui.md) | 공용 UI 컴포넌트 승격·변경 시 |
| `core/common` | [core/common.md](core/common.md) | 공통 유틸, 플랫폼 헬퍼 변경 시 |

### feature
| 모듈 | 문서 | 읽어야 할 때 |
|------|------|-------------|
| (공통) | [feature/README.md](feature/README.md) | 새 feature 모듈 생성, feature 공통 규칙 확인 시 |
| (템플릿) | [feature/_template.md](feature/_template.md) | 새 feature 문서 작성 시 복사하여 사용 |

> 새 feature를 추가하면 `feature/{name}.md`를 작성하고 이 테이블에 등록한다.

---

## 관련 가이드

| 문서 | 읽어야 할 때 |
|------|-------------|
| [Kotlin 가이드](../kotlin/kotlin-guide.md) | 네이밍, 타입 안전성, 이디엄 규칙 확인 시 |
| [Compose 가이드](../compose/compose-guide.md) | Composable 작성, 상태·컴포넌트·Effect 규칙 확인 시 |
| [Navigation 가이드](../compose/navigation-guide.md) | Route/Screen/Content 계층, 네스트 그래프, 인자 전달, 이벤트 네비게이션 확인 시 |
| [Coroutine 가이드](../coroutine/coroutine-guide.md) | 스코프 선택, CancellationException, Flow/SSE 패턴 확인 시 |
| [Reactive 가이드](../reactive/reactive-guide.md) | 리액티브 원칙, UDF/MVI, 스트림 합성, 백프레셔 확인 시 |
| [Testing 가이드](../convention/testing-guide.md) | TDD 워크플로우, Fake 테스트, 테스트 DI 확인 시 |
| [DI 가이드](../convention/di-guide.md) | Koin 모듈 등록, 테스트 DI 오버라이드 확인 시 |
| [Localization 가이드](../convention/localization.md) | 다국어 리소스 키·톤앤매너 규칙 확인 시 |

---

## 운영 규칙
- 모듈별 상세 지침은 이 폴더(`docs/architecture/`)에서만 관리한다.
- 루트 `AGENTS.md`는 프로젝트 공통 원칙만 관리한다.
- 새 모듈이 생기면 해당 영역에 `.md` 파일을 추가하고 이 인덱스에 등록한다.
- 새 DataSource 인프라 모듈을 추가하면 `core/datasource-{tech}.md` 문서를 작성한다 ([datasource.md](core/datasource.md) > 인프라 모듈 추가 가이드 참조).
- 새 feature 모듈을 추가하면 `feature/{name}.md` 문서를 작성하고 이 인덱스에 등록한다. 공통 규칙은 `feature/README.md`를 따른다.
- 클린 아키텍처 고수, TDD 준수, `core:designsystem` 강제 사용, Material Design 3 준수 원칙은 루트 `AGENTS.md`를 기준으로 적용한다.
