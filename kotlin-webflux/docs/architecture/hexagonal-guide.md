# Hexagonal Architecture Guide (Ports & Adapters)

> **트리거**: Port/Adapter를 추가·변경할 때, 의존 방향을 확인할 때 이 문서를 참조한다.

---

## 핵심 개념

Hexagonal Architecture(Ports & Adapters)는 비즈니스 로직을 외부 인프라로부터 완전히 격리하는 패턴이다.

| 흐름 | Adapter | Port | Core |
|------|---------|------|------|
| **Inbound** (외부 → 내부) | Controller, SSE, WebSocket | UseCase Interface (Inbound Port) | Application Service → Domain |
| **Outbound** (내부 → 외부) | R2DBC, WebClient, Redis | Repository Interface (Outbound Port) | Application Service ← Domain |

| 구성요소 | 역할 | 위치 |
|---------|------|------|
| **Inbound Port** | 외부에서 애플리케이션으로 진입하는 인터페이스 | `application/port/in/` |
| **Inbound Adapter** | Inbound Port를 호출하는 구현체 (Web Handler, CLI 등) | `adapter/in/` |
| **Outbound Port** | 애플리케이션이 외부 인프라를 사용하기 위한 인터페이스 | `application/port/out/` |
| **Outbound Adapter** | Outbound Port를 구현하는 인프라 코드 (DB, API 등) | `adapter/out/` |

---

## Inbound Port

UseCase 인터페이스로 정의한다. 외부 Adapter가 호출할 수 있는 계약이다.

```kotlin
// application/port/in/CreateOrderUseCase.kt
interface CreateOrderUseCase {
    suspend fun execute(command: CreateOrderCommand): Order
}

// application/port/in/command/CreateOrderCommand.kt
data class CreateOrderCommand(
    val customerId: String,
    val items: List<OrderItemCommand>,
)

data class OrderItemCommand(
    val productId: String,
    val quantity: Int,
)
```

| 규칙 | 설명 |
|------|------|
| 인터페이스 정의 | UseCase별 단일 인터페이스 |
| Command/Query 분리 | 입력은 Command(변경) 또는 Query(조회) 객체로 전달 |
| `suspend` 함수 | 단일 결과는 `suspend fun`, 스트림은 `Flow<T>` 반환 |
| Spring 무관 | Port 인터페이스에 Spring 어노테이션 금지 |

---

## Outbound Port

Repository 또는 외부 시스템 연동 인터페이스로 정의한다.

```kotlin
// application/port/out/OrderRepository.kt
interface OrderRepository {
    suspend fun save(order: Order): Order
    suspend fun findById(id: String): Order?
    fun findByCustomerId(customerId: String): Flow<Order>
    suspend fun existsById(id: String): Boolean
    suspend fun deleteById(id: String)
}

// application/port/out/PaymentPort.kt
interface PaymentPort {
    suspend fun processPayment(orderId: String, amount: Money): PaymentResult
}

// application/port/out/NotificationPort.kt
interface NotificationPort {
    suspend fun sendOrderConfirmation(order: Order)
}
```

| 규칙 | 설명 |
|------|------|
| 도메인 관점 네이밍 | DB 용어가 아닌 비즈니스 용어 사용 (`save`, `findBy` 등) |
| 도메인 모델 사용 | Entity/DTO가 아닌 도메인 모델을 파라미터/반환 타입으로 사용 |
| `suspend` / `Flow` | 단일 결과는 `suspend`, 스트림은 `Flow<T>` 반환 |
| 단일 책임 | Repository는 Aggregate Root 단위로 정의 |

---

## Inbound Adapter

`@RestController`가 Inbound Port(UseCase)를 호출한다.

```kotlin
// adapter/in/web/controller/OrderController.kt
@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderUseCase: GetOrderUseCase,
) {
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@RequestBody body: CreateOrderRequest): OrderResponse {
        val command = body.toCommand()
        val order = createOrderUseCase.execute(command)
        return order.toResponse()
    }

    @GetMapping("/{id}")
    suspend fun getById(@PathVariable id: String): OrderResponse {
        val order = getOrderUseCase.execute(GetOrderQuery(id))
            ?: throw OrderNotFoundException(OrderId(id))
        return order.toResponse()
    }
}
```

| 규칙 | 설명 |
|------|------|
| `@RestController` | 어노테이션 기반 Controller 사용 |
| DTO 변환 | Adapter에서 DTO <-> Command/Domain 변환 |
| UseCase만 호출 | Controller는 UseCase(Port) 인터페이스만 의존 |
| 비즈니스 로직 금지 | Controller에 비즈니스 로직을 넣지 않는다 |

---

## Outbound Adapter

Outbound Port를 구현하여 실제 인프라에 접근한다.

```kotlin
// adapter/out/persistence/repository/OrderPersistenceAdapter.kt
@Repository
class OrderPersistenceAdapter(
    private val orderCoroutineRepository: OrderCoroutineRepository,
    private val orderMapper: OrderPersistenceMapper,
) : OrderRepository {

    override suspend fun save(order: Order): Order {
        val saved = orderCoroutineRepository.save(orderMapper.toEntity(order))
        return orderMapper.toDomain(saved)
    }

    override suspend fun findById(id: String): Order? =
        orderCoroutineRepository.findById(id)?.let { orderMapper.toDomain(it) }

    override fun findByCustomerId(customerId: String): Flow<Order> =
        orderCoroutineRepository.findByCustomerId(customerId)
            .map { orderMapper.toDomain(it) }

    override suspend fun existsById(id: String): Boolean =
        orderCoroutineRepository.existsById(id)

    override suspend fun deleteById(id: String) =
        orderCoroutineRepository.deleteById(id)
}

// adapter/out/persistence/entity/OrderEntity.kt
@Table("orders")
data class OrderEntity(
    @Id val id: String? = null,
    val customerId: String,
    val status: String,
    val totalAmount: Long,
    val createdAt: LocalDateTime,
)
```

| 규칙 | 설명 |
|------|------|
| Port 구현 | Outbound Port 인터페이스를 구현 |
| 매퍼 사용 | Entity <-> Domain 변환은 반드시 매퍼를 통해 |
| 인프라 캡슐화 | R2DBC, WebClient 등 인프라 상세는 Adapter 내부에 은닉 |
| DAO | `CoroutineCrudRepository`를 상속하여 정의 |

### Outbound Adapter 어노테이션 규칙

| Adapter 종류 | 어노테이션 | 이유 |
|-------------|-----------|------|
| DB Persistence Adapter | `@Repository` | Spring persistence 예외 자동 변환 적용 |
| 외부 API Client | `@Component` | DB 예외 변환이 불필요 |
| Redis Cache Adapter | `@Component` | DB 예외 변환이 불필요 |
| Message Producer | `@Component` | DB 예외 변환이 불필요 |

---

## 새 기능 추가 절차

| 단계 | 위치 | 수행 |
|------|------|------|
| 1 | `domain/model/` | 도메인 모델 설계 (Entity, VO, Aggregate) |
| 2 | `application/port/in/` | Inbound Port (UseCase 인터페이스) 정의 |
| 3 | `application/port/out/` | Outbound Port (Repository 인터페이스) 정의 |
| 4 | `application/service/` | UseCase 구현 (Application Service) |
| 5 | `adapter/out/` | Outbound Adapter 구현 (DB, 외부 API 등) |
| 6 | `adapter/in/` | Inbound Adapter 구현 (Controller) |
| 7 | 각 단계마다 | TDD: 테스트를 먼저 작성하고 구현 |

---

## DO / DON'T

```kotlin
// DO -- Port는 도메인 관점의 suspend 인터페이스
interface OrderRepository {
    fun findByCustomerId(customerId: String): Flow<Order>
}

// DON'T -- 인프라 기술이 Port에 노출
interface OrderRepository {
    fun findByCustomerId(customerId: String): Flow<OrderEntity>  // Entity 노출 금지
}

// DO -- Adapter에서 매퍼 사용
class OrderPersistenceAdapter(...) : OrderRepository {
    override suspend fun save(order: Order): Order {
        val saved = repository.save(mapper.toEntity(order))
        return mapper.toDomain(saved)
    }
}

// DON'T -- 도메인 모델에 인프라 어노테이션
@Table("orders")
data class Order(...)  // 금지: 도메인에 Spring Data 의존

// DO -- Controller에서 UseCase(Port) 호출
class OrderController(private val createOrderUseCase: CreateOrderUseCase)

// DON'T -- Controller에서 Repository 직접 호출
class OrderController(private val orderRepository: OrderRepository)  // 금지: 계층 건너뜀
```

---

## 변경 시 체크리스트

- [ ] 의존 방향이 안쪽(도메인)을 향하는가?
- [ ] Port 인터페이스가 도메인 모델만 사용하는가?
- [ ] Adapter가 Port 인터페이스를 구현하는가?
- [ ] Controller가 UseCase(Port)만 호출하는가? (Repository 직접 호출 아님)
- [ ] 도메인 레이어에 Spring/인프라 어노테이션이 유입되지 않았는가?
- [ ] 계층 간 데이터 이동에 매퍼를 사용하는가?
- [ ] 새 Port/Adapter 추가 시 DI 설정을 갱신했는가?
