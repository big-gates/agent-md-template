# Package Structure Guide

> **트리거**: 새 클래스/패키지를 추가할 때, 파일 배치를 결정할 때 이 문서를 참조한다.

---

## 패키지 구조

> 기준 경로: `src/main/kotlin/{base-package}/`
> Bounded Context 단위로 최상위 패키지를 분리한다.

| 패키지 경로 | 설명 |
|------------|------|
| `{domain}/adapter/in/web/handler/` | WebFlux Handler (Functional Endpoint) |
| `{domain}/adapter/in/web/router/` | Router 설정 |
| `{domain}/adapter/in/web/dto/request/` | Request DTO |
| `{domain}/adapter/in/web/dto/response/` | Response DTO |
| `{domain}/adapter/in/web/mapper/` | DTO <-> Domain 매퍼 |
| `{domain}/adapter/in/websocket/handler/` | WebSocket Handler |
| `{domain}/adapter/in/websocket/message/` | WebSocket 메시지 타입 |
| `{domain}/adapter/in/sse/handler/` | SSE Endpoint |
| `{domain}/adapter/out/persistence/entity/` | R2DBC Entity (DB 테이블 매핑) |
| `{domain}/adapter/out/persistence/repository/` | Repository Adapter (Port 구현) |
| `{domain}/adapter/out/persistence/mapper/` | Entity <-> Domain 매퍼 |
| `{domain}/adapter/out/persistence/config/` | R2DBC 설정 |
| `{domain}/adapter/out/messaging/producer/` | 메시지 발행 (Kafka, RabbitMQ) |
| `{domain}/adapter/out/messaging/consumer/` | 메시지 소비 |
| `{domain}/adapter/out/external/client/` | WebClient Adapter (외부 API 호출) |
| `{domain}/adapter/out/external/dto/` | 외부 API DTO |
| `{domain}/adapter/out/external/mapper/` | 외부 DTO <-> Domain 매퍼 |
| `{domain}/adapter/out/cache/adapter/` | 캐시 (Redis) |
| `{domain}/application/port/in/` | Inbound Port (UseCase 인터페이스) |
| `{domain}/application/port/in/command/` | Command 객체 (변경 요청) |
| `{domain}/application/port/out/` | Outbound Port (Repository 등 인터페이스) |
| `{domain}/application/service/` | UseCase 구현 (Application Service) |
| `{domain}/domain/model/` | Aggregate, Entity, Value Object |
| `{domain}/domain/event/` | Domain Event |
| `{domain}/domain/exception/` | Domain 예외 |
| `{domain}/domain/service/` | Domain Service (상태 없는 도메인 로직) |
| `config/` | 공통 인프라 설정 (WebFluxConfig, SecurityConfig, R2dbcConfig 등) |

> `{domain}`은 Bounded Context 이름 (예: `order`, `product`, `user`).
> 각 Bounded Context 내부는 동일한 `adapter/`, `application/`, `domain/` 구조를 유지한다.

### 예시

| Bounded Context | 패키지 |
|----------------|--------|
| 주문 | `order/adapter/`, `order/application/`, `order/domain/` |
| 상품 | `product/adapter/`, `product/application/`, `product/domain/` |
| 사용자 | `user/adapter/`, `user/application/`, `user/domain/` |
| 공통 설정 | `config/` |

---

## 배치 기준

### 이 파일은 어디에?

| 파일 종류 | 위치 | 예시 |
|-----------|------|------|
| REST Handler | `{domain}/adapter/in/web/handler/` | `OrderHandler.kt` |
| Router 설정 | `{domain}/adapter/in/web/router/` | `OrderRouter.kt` |
| Request DTO | `{domain}/adapter/in/web/dto/request/` | `CreateOrderRequest.kt` |
| Response DTO | `{domain}/adapter/in/web/dto/response/` | `OrderResponse.kt` |
| DTO Mapper | `{domain}/adapter/in/web/mapper/` | `OrderDtoMapper.kt` |
| WebSocket Handler | `{domain}/adapter/in/websocket/handler/` | `ChatWebSocketHandler.kt` |
| SSE Handler | `{domain}/adapter/in/sse/handler/` | `NotificationSseHandler.kt` |
| UseCase 인터페이스 | `{domain}/application/port/in/` | `CreateOrderUseCase.kt` |
| UseCase Command | `{domain}/application/port/in/command/` | `CreateOrderCommand.kt` |
| Repository 인터페이스 | `{domain}/application/port/out/` | `OrderRepository.kt` |
| UseCase 구현 | `{domain}/application/service/` | `OrderService.kt` |
| Domain Model | `{domain}/domain/model/` | `Order.kt`, `Money.kt` |
| Domain Event | `{domain}/domain/event/` | `OrderCreatedEvent.kt` |
| Domain Exception | `{domain}/domain/exception/` | `InsufficientStockException.kt` |
| Domain Service | `{domain}/domain/service/` | `PricingService.kt` |
| DB Entity | `{domain}/adapter/out/persistence/entity/` | `OrderEntity.kt` |
| Repository 구현 | `{domain}/adapter/out/persistence/repository/` | `OrderPersistenceAdapter.kt` |
| Entity Mapper | `{domain}/adapter/out/persistence/mapper/` | `OrderEntityMapper.kt` |
| WebClient Adapter | `{domain}/adapter/out/external/client/` | `PaymentApiClient.kt` |
| Spring Config | `config/` | `WebFluxConfig.kt` |

---

## 네이밍 규칙

| 구분 | 접미사 패턴 | 예시 |
|------|-----------|------|
| Handler | `{Domain}Handler` | `OrderHandler` |
| Router | `{Domain}Router` | `OrderRouter` |
| UseCase | `{Action}{Domain}UseCase` | `CreateOrderUseCase` |
| Application Service | `{Domain}Service` | `OrderService` |
| Domain Service | `{Domain}DomainService` | `PricingDomainService` |
| Repository Port | `{Domain}Repository` | `OrderRepository` |
| Persistence Adapter | `{Domain}PersistenceAdapter` | `OrderPersistenceAdapter` |
| DB Entity | `{Domain}Entity` | `OrderEntity` |
| Request DTO | `{Action}{Domain}Request` | `CreateOrderRequest` |
| Response DTO | `{Domain}Response` | `OrderResponse` |
| Mapper | `{Domain}{Layer}Mapper` | `OrderDtoMapper`, `OrderEntityMapper` |
| Domain Event | `{Domain}{Action}Event` | `OrderCreatedEvent` |
| Domain Exception | `{Description}Exception` | `InsufficientStockException` |

---

## 테스트 패키지 구조

> 기준 경로: `src/test/kotlin/{base-package}/`

| 테스트 위치 | 대상 | 예시 파일 |
|------------|------|----------|
| `{domain}/domain/model/` | Domain Model 단위 테스트 | `OrderTest.kt`, `MoneyTest.kt` |
| `{domain}/domain/service/` | Domain Service 단위 테스트 | `PricingServiceTest.kt` |
| `{domain}/application/service/` | UseCase 단위 테스트 | `OrderServiceTest.kt` |
| `{domain}/adapter/in/web/handler/` | Handler 통합 테스트 | `OrderHandlerIntegrationTest.kt` |
| `{domain}/adapter/out/persistence/` | Repository Adapter 통합 테스트 | `OrderPersistenceAdapterTest.kt` |
| `fixture/` | 공유 테스트 픽스처 / Fake | `OrderFixture.kt`, `FakeOrderRepository.kt` |

---

## 변경 시 체크리스트

- [ ] 새 클래스가 올바른 Bounded Context와 패키지에 배치되었는가?
- [ ] 의존 방향이 안쪽(도메인)을 향하는가?
- [ ] DTO/Entity가 도메인 레이어에 유입되지 않았는가?
- [ ] 네이밍 규칙을 따르는가?
- [ ] 대응하는 테스트 파일이 동일 패키지 구조에 있는가?
