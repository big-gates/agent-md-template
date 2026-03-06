# Architecture Guide

> 이 문서는 Kotlin WebFlux 기반 백엔드 프로젝트의 아키텍처 개요이다.
> Hexagonal Architecture + Clean Architecture + DDD를 결합한 구조를 따른다.

---

## 아키텍처 원칙

| 원칙 | 설명 |
|------|------|
| **Hexagonal (Ports & Adapters)** | 비즈니스 로직을 외부 인프라로부터 격리. Port(인터페이스)와 Adapter(구현)로 분리 |
| **Clean Architecture** | 의존성은 항상 안쪽(도메인)을 향한다. 도메인은 어떤 외부 레이어에도 의존하지 않는다 |
| **DDD (Domain-Driven Design)** | 도메인 모델 중심 설계. Aggregate, Entity, Value Object, Domain Event 활용 |
| **Coroutine-First** | Spring WebFlux + Kotlin 코루틴 기반. 모든 I/O는 `suspend` 함수, 스트림은 `Flow` 사용 |
| **TDD** | 모든 비즈니스 로직은 테스트 먼저 작성 (Red-Green-Refactor) |

---

## 레이어 구조

| 레이어 | 구성요소 | 패키지 |
|--------|---------|--------|
| **Adapter (Inbound)** | Router, Handler, WebSocket, SSE | `adapter/in/` |
| **Port (Inbound)** | UseCase Interface | `application/port/in/` |
| **Application** | UseCase 구현, Application Service | `application/service/` |
| **Domain** | Entity, VO, Aggregate, Domain Event, Domain Service | `domain/` |
| **Port (Outbound)** | Repository Interface, External Port | `application/port/out/` |
| **Adapter (Outbound)** | R2DBC, WebClient, Redis, Kafka | `adapter/out/` |

| 흐름 | from | to | 설명 |
|------|------|----|------|
| Inbound | `Adapter(In)` | `Port(In)` | 외부 요청 진입 |
| Inbound | `Port(In)` | `Application` | UseCase 호출 |
| Inbound | `Application` | `Domain` | 도메인 로직 실행 |
| Outbound | `Application` | `Port(Out)` | 인프라 접근 요청 |
| Outbound | `Port(Out)` | `Adapter(Out)` | 실제 인프라 구현 |

---

## 패키지 구조

> 상세 구조는 [package-structure.md](package-structure.md) 참조.
> Bounded Context 단위로 최상위 패키지를 분리한다. `{domain}`은 BC 이름 (예: `order`, `product`, `user`).

| 패키지 | 설명 |
|--------|------|
| `{domain}/adapter/in/web/` | REST Handler, Router, DTO, Mapper |
| `{domain}/adapter/in/websocket/` | WebSocket Handler, 메시지 타입 |
| `{domain}/adapter/in/sse/` | SSE Endpoint |
| `{domain}/adapter/out/persistence/` | R2DBC Entity, Repository Adapter, Mapper |
| `{domain}/adapter/out/messaging/` | Kafka/RabbitMQ Producer/Consumer |
| `{domain}/adapter/out/external/` | 외부 API WebClient Adapter |
| `{domain}/adapter/out/cache/` | Redis Adapter |
| `{domain}/application/port/in/` | Inbound Port (UseCase 인터페이스), Command/Query |
| `{domain}/application/port/out/` | Outbound Port (Repository 인터페이스) |
| `{domain}/application/service/` | UseCase 구현 (Application Service) |
| `{domain}/domain/model/` | Aggregate Root, Entity, Value Object |
| `{domain}/domain/event/` | Domain Event |
| `{domain}/domain/exception/` | Domain Exception |
| `{domain}/domain/service/` | Domain Service |
| `config/` | 공통 Spring 설정 (Bean, Security, R2DBC 등) |

---

## 의존 방향

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
| **프레임워크 격리** | Domain, Application 레이어에 Spring 어노테이션 금지 (UseCase 구현의 `@Service` 제외) |
| **DTO 격리** | Request/Response DTO는 `adapter.in` 에만 존재. 도메인 레이어에 전달 금지 |
| **Entity 격리** | DB Entity는 `adapter.out.persistence`에만 존재. 도메인 레이어에 전달 금지 |
| **매퍼 필수** | 계층 간 데이터 이동은 반드시 매퍼를 통해 수행 |

---

## 계층 간 모델 매핑

| 모델 | 위치 | 용도 |
|------|------|------|
| Domain Model (`Order`) | `domain/model/` | 비즈니스 로직 |
| DB Entity (`OrderEntity`) | `adapter/out/persistence/entity/` | DB 테이블 매핑 |
| Request DTO (`CreateOrderRequest`) | `adapter/in/web/dto/` | API 요청 역직렬화 |
| Response DTO (`OrderResponse`) | `adapter/in/web/dto/` | API 응답 직렬화 |
| Inbound Mapper | `adapter/in/web/mapper/` | DTO <-> Domain 변환 |
| Outbound Mapper | `adapter/out/persistence/mapper/` | Entity <-> Domain 변환 |

---

## 문서 인덱스

### architecture
| 문서 | 읽어야 할 때 |
|------|-------------|
| [hexagonal-guide.md](hexagonal-guide.md) | Ports & Adapters 패턴, 의존 방향, 새 Port/Adapter 추가 시 |
| [package-structure.md](package-structure.md) | 패키지 배치, 새 모듈/클래스 추가 시 |

### domain
| 문서 | 읽어야 할 때 |
|------|-------------|
| [DDD Guide](../domain/ddd-guide.md) | Aggregate, Entity, VO, Domain Event 설계 시 |
| [UseCase Guide](../domain/usecase-guide.md) | Application Service / UseCase 작성 시 |

### webflux
| 문서 | 읽어야 할 때 |
|------|-------------|
| [WebFlux Guide](../webflux/webflux-guide.md) | Handler, Router, 함수형 엔드포인트 작성 시 |
| [Coroutine & Flow Guide](../webflux/coroutine-flow-guide.md) | suspend/Flow 패턴, 스트림 합성, 동시성 확인 시 |
| [SSE Guide](../webflux/sse-guide.md) | Server-Sent Events 실시간 스트리밍 구현 시 |
| [WebSocket Guide](../webflux/websocket-guide.md) | WebSocket 양방향 실시간 통신 구현 시 |
| [R2DBC Guide](../webflux/r2dbc-guide.md) | R2DBC DB 접근, 쿼리, 트랜잭션 작성 시 |
| [Error Handling Guide](../webflux/error-handling-guide.md) | 에러 처리, 글로벌 예외 핸들러 작성 시 |

### testing
| 문서 | 읽어야 할 때 |
|------|-------------|
| [TDD Guide](../testing/tdd-guide.md) | TDD 워크플로우, Red-Green-Refactor 사이클 확인 시 |
| [Unit Test Guide](../testing/unit-test-guide.md) | 단위 테스트, runTest, 코루틴 테스트 작성 시 |
| [Integration Test Guide](../testing/integration-test-guide.md) | 통합 테스트, WebTestClient, Testcontainers 사용 시 |
| [Fixture Guide](../testing/fixture-guide.md) | 테스트 픽스처, Fake, 테스트 데이터 빌더 작성 시 |

### kotlin
| 문서 | 읽어야 할 때 |
|------|-------------|
| [Kotlin Guide](../kotlin/kotlin-guide.md) | Kotlin 이디엄, 타입 안전성, 관례 확인 시 |
| [Naming Guide](../kotlin/naming-guide.md) | 네이밍 컨벤션, 패키지/클래스/함수 명명 규칙 확인 시 |

### convention
| 문서 | 읽어야 할 때 |
|------|-------------|
| [Git Commit Template](../convention/git-commit-template.md) | 커밋 메시지 작성 규칙 확인 시 |
| [PR Template](../convention/pull-request-template.md) | Pull Request 작성 시 |
| [API Design Guide](../convention/api-design-guide.md) | REST API, 스트리밍 API 엔드포인트 설계 시 |

---

## 운영 규칙

- 아키텍처 문서는 `docs/` 하위에서만 관리한다.
- 새 Adapter 추가 시 해당 영역 문서를 갱신하고 이 인덱스에 등록한다.
- Hexagonal + Clean Architecture + DDD + TDD 원칙은 모든 코드에 적용한다.
- 도메인 레이어는 순수 Kotlin만 허용. Spring 의존 금지.
