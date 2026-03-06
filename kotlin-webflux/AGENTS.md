# AGENTS.md

## 목적

- 이 파일은 Kotlin WebFlux 백엔드 프로젝트의 전반 지침을 관리한다.
- Hexagonal Architecture + Clean Architecture + DDD + TDD를 따른다.
- 세부 가이드는 `docs/` 하위 문서에서 관리한다.

---

## 에이전트 문서 라우팅

> 작업 대상 파일 경로를 보고, 아래 테이블에서 **반드시 읽어야 할 문서**를 먼저 참조한다.

| 작업 대상 경로 패턴 | 읽어야 할 문서 |
|-------------------|--------------|
| `domain/model/**` | [ddd-guide.md](docs/domain/ddd-guide.md) |
| `domain/event/**` | [ddd-guide.md](docs/domain/ddd-guide.md) |
| `domain/exception/**` | [ddd-guide.md](docs/domain/ddd-guide.md), [error-handling-guide.md](docs/webflux/error-handling-guide.md) |
| `domain/service/**` | [ddd-guide.md](docs/domain/ddd-guide.md) |
| `application/port/in/**` | [usecase-guide.md](docs/domain/usecase-guide.md), [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `application/port/out/**` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md), [r2dbc-guide.md](docs/webflux/r2dbc-guide.md) |
| `application/service/**` | [usecase-guide.md](docs/domain/usecase-guide.md) |
| `adapter/in/web/controller/**` | [webflux-guide.md](docs/webflux/webflux-guide.md), [hexagonal-guide.md](docs/architecture/hexagonal-guide.md), [api-spec-guide.md](docs/convention/api-spec-guide.md) |
| `adapter/in/web/dto/**` | [webflux-guide.md](docs/webflux/webflux-guide.md), [api-design-guide.md](docs/convention/api-design-guide.md), [api-spec-guide.md](docs/convention/api-spec-guide.md) |
| `adapter/in/web/mapper/**` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `adapter/in/sse/**` | [sse-guide.md](docs/webflux/sse-guide.md) |
| `adapter/in/websocket/**` | [websocket-guide.md](docs/webflux/websocket-guide.md) |
| `adapter/out/persistence/**` | [r2dbc-guide.md](docs/webflux/r2dbc-guide.md), [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `adapter/out/external/**` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `adapter/out/messaging/**` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `adapter/out/cache/**` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) |
| `config/**` | [webflux-guide.md](docs/webflux/webflux-guide.md) |
| `**/mapper/**`, `**/*Mapper*.kt` | [hexagonal-guide.md](docs/architecture/hexagonal-guide.md), [r2dbc-guide.md](docs/webflux/r2dbc-guide.md) |
| `**/*Test.kt` | [tdd-guide.md](docs/testing/tdd-guide.md), [unit-test-guide.md](docs/testing/unit-test-guide.md) |
| `**/fixture/**`, `**/Fake*.kt` | [fixture-guide.md](docs/testing/fixture-guide.md) |
| `**/integration/**` | [integration-test-guide.md](docs/testing/integration-test-guide.md) |
| `db/migration/**` | [r2dbc-guide.md](docs/webflux/r2dbc-guide.md) |
| `docs/api-specs/**` | [api-spec-guide.md](docs/convention/api-spec-guide.md), [api-design-guide.md](docs/convention/api-design-guide.md) |

---

## 프로젝트 전반 원칙

### 아키텍처

- Hexagonal Architecture + Clean Architecture + DDD를 준수한다.
- 의존 방향은 항상 안쪽(도메인)을 향한다.

| from | to | 관계 |
|------|----|------|
| `adapter.in.web` | `application.port.in` | UseCase 인터페이스 호출 |
| `application.service` | `domain.model` | 도메인 모델 사용 |
| `application.service` | `application.port.out` | Outbound Port 호출 |
| `adapter.out.persistence` | `application.port.out` | Outbound Port 구현 (**의존성 역전**) |
| `domain` | 없음 | 최내부 레이어. 외부 의존 없음 |

| 규칙 | 설명 |
|------|------|
| **역방향 금지** | 안쪽 레이어는 바깥 레이어를 참조할 수 없다 |
| **의존성 역전** | Domain이 Repository 인터페이스(Port)를 정의하고, Adapter가 구현한다 |
| **프레임워크 격리** | Domain, Application 레이어에 Spring 어노테이션 금지 (`@Service` 제외) |
| **DTO 격리** | Request/Response DTO는 `adapter.in`에만 존재 |
| **Entity 격리** | DB Entity는 `adapter.out.persistence`에만 존재 |
| **매퍼 필수** | 계층 간 데이터 이동은 반드시 매퍼를 통해 수행 |

### Coroutine-First

| 규칙 | 설명 |
|------|------|
| `suspend fun` | 모든 단일 결과 I/O에 사용 |
| `Flow<T>` | 모든 스트림에 사용 |
| `Mono`/`Flux` 금지 | Spring SSE 인터페이스, GlobalErrorHandler 제외 |
| `runBlocking` 금지 | 이벤트 루프 차단 방지 |
| `GlobalScope` 금지 | 구조화된 동시성 사용 |

### TDD

| 규칙 | 설명 |
|------|------|
| Red → Green → Refactor | 모든 새 기능/버그 수정에 적용 |
| Fake 선호 | Mock보다 Fake를 우선 사용 |
| `runTest` 필수 | suspend 함수 테스트에 사용. `runBlocking` 금지 |
| Turbine | Flow 테스트에 `.test { awaitItem() }` 패턴 사용 |

---

## 에이전트 판단 규칙

### 새 파일/클래스를 어디에 만들어야 하는가?

> `{domain}`은 Bounded Context 이름 (예: `order`, `product`, `user`).

| 만들려는 것 | 위치 | 근거 |
|-----------|------|------|
| Aggregate Root / Entity / VO | `{domain}/domain/model/` | 순수 도메인 모델 |
| Domain Event | `{domain}/domain/event/` | 비즈니스 사건 |
| Domain Exception | `{domain}/domain/exception/` | 도메인 예외 |
| Domain Service | `{domain}/domain/service/` | 상태 없는 도메인 로직 |
| UseCase 인터페이스 | `{domain}/application/port/in/` | Inbound Port |
| Command / Query | `{domain}/application/port/in/command/` | UseCase 입력 객체 |
| Repository 인터페이스 | `{domain}/application/port/out/` | Outbound Port (의존성 역전) |
| Application Service | `{domain}/application/service/` | UseCase 구현 |
| Controller | `{domain}/adapter/in/web/controller/` | `@RestController` HTTP 요청 처리 (suspend) |
| Request/Response DTO | `{domain}/adapter/in/web/dto/` | API 직렬화/역직렬화 |
| DTO Mapper | `{domain}/adapter/in/web/mapper/` | DTO <-> Domain 변환 |
| SSE Handler | `{domain}/adapter/in/sse/handler/` | SSE 스트리밍 |
| WebSocket Handler | `{domain}/adapter/in/websocket/handler/` | WebSocket 양방향 통신 |
| DB Entity | `{domain}/adapter/out/persistence/entity/` | R2DBC 테이블 매핑 |
| Repository 구현 | `{domain}/adapter/out/persistence/repository/` | Outbound Port 구현 |
| Entity Mapper | `{domain}/adapter/out/persistence/mapper/` | Entity <-> Domain 변환 |
| 외부 API Client | `{domain}/adapter/out/external/client/` | WebClient Adapter |
| Spring Config | `config/` | 공통 Bean, Security, R2DBC 설정 |
| Fake (테스트) | `test/{domain}/fixture/` | 인터페이스의 테스트 구현체 |
| Fixture (테스트) | `test/{domain}/fixture/` | 테스트 데이터 팩토리 |

### 의존을 추가해도 되는가?

| from → to | 허용 |
|----------|------|
| `adapter.in` → `application.port.in` | O |
| `application.service` → `domain` | O |
| `application.service` → `application.port.out` | O |
| `adapter.out` → `application.port.out` | O (Port 구현) |
| `adapter.in` → `domain` | **X** (UseCase를 통해 접근) |
| `adapter.in` → `adapter.out` | **X** |
| `domain` → `application` | **X** (역방향 금지) |
| `domain` → `adapter` | **X** (역방향 금지) |
| `application` → `adapter` | **X** (Port를 통해 접근) |

---

## 세부 지침 문서 위치

### architecture

| 문서 | 읽어야 할 때 |
|------|-------------|
| [README.md](docs/architecture/README.md) | 아키텍처 개요, 레이어 구조, 문서 인덱스 확인 시 |
| [hexagonal-guide.md](docs/architecture/hexagonal-guide.md) | Ports & Adapters 패턴, 의존 방향, 새 Port/Adapter 추가 시 |
| [package-structure.md](docs/architecture/package-structure.md) | 패키지 배치, 새 클래스 추가 시 |

### domain

| 문서 | 읽어야 할 때 |
|------|-------------|
| [ddd-guide.md](docs/domain/ddd-guide.md) | Aggregate, Entity, VO, Domain Event 설계 시 |
| [usecase-guide.md](docs/domain/usecase-guide.md) | UseCase / Application Service 작성 시 |

### webflux

| 문서 | 읽어야 할 때 |
|------|-------------|
| [webflux-guide.md](docs/webflux/webflux-guide.md) | Controller, 엔드포인트 작성 시 |
| [coroutine-flow-guide.md](docs/webflux/coroutine-flow-guide.md) | suspend/Flow 패턴, 스트림 합성, 동시성 확인 시 |
| [sse-guide.md](docs/webflux/sse-guide.md) | SSE 실시간 스트리밍 구현 시 |
| [websocket-guide.md](docs/webflux/websocket-guide.md) | WebSocket 양방향 통신 구현 시 |
| [r2dbc-guide.md](docs/webflux/r2dbc-guide.md) | R2DBC DB 접근, 쿼리, 트랜잭션 작성 시 |
| [error-handling-guide.md](docs/webflux/error-handling-guide.md) | 에러 처리, 글로벌 예외 핸들러 작성 시 |

### testing

| 문서 | 읽어야 할 때 |
|------|-------------|
| [tdd-guide.md](docs/testing/tdd-guide.md) | TDD 워크플로우, Red-Green-Refactor 확인 시 |
| [unit-test-guide.md](docs/testing/unit-test-guide.md) | 단위 테스트, runTest, 코루틴 테스트 작성 시 |
| [integration-test-guide.md](docs/testing/integration-test-guide.md) | 통합 테스트, WebTestClient, Testcontainers 사용 시 |
| [fixture-guide.md](docs/testing/fixture-guide.md) | Fake, Fixture, 테스트 데이터 작성 시 |

### kotlin

| 문서 | 읽어야 할 때 |
|------|-------------|
| [kotlin-guide.md](docs/kotlin/kotlin-guide.md) | Kotlin 이디엄, 타입 안전성 확인 시 |
| [naming-guide.md](docs/kotlin/naming-guide.md) | 네이밍 컨벤션 확인 시 |

### convention

| 문서 | 읽어야 할 때 |
|------|-------------|
| [git-commit-template.md](docs/convention/git-commit-template.md) | 커밋 메시지 작성 시 |
| [pull-request-template.md](docs/convention/pull-request-template.md) | PR 작성 시 |
| [api-design-guide.md](docs/convention/api-design-guide.md) | REST/스트리밍 API 설계 시 |
| [api-spec-guide.md](docs/convention/api-spec-guide.md) | API Spec 문서 작성, 프론트엔드 공유용 API 명세 생성 시 |

---

## 문서 운영 규칙

- 새 Adapter 추가 시 해당 영역 문서를 갱신하고 `docs/architecture/README.md` 인덱스에 등록한다.
- 새 엔드포인트 추가 시 `docs/api-specs/{domain}.md`에 API Spec을 작성한다. 작성 규칙은 [api-spec-guide.md](docs/convention/api-spec-guide.md)를 따른다.
- Hexagonal + Clean Architecture + DDD + TDD 원칙은 모든 코드에 적용한다.
- 도메인 레이어는 순수 Kotlin만 허용. Spring 의존 금지.
- 매퍼 추가/변경 시 매퍼 테스트를 반드시 함께 작성한다.
