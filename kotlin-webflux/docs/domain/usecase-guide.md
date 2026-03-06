# UseCase (Application Service) Guide

> **트리거**: UseCase/Application Service를 추가·수정할 때, Command/Query 패턴을 적용할 때 이 문서를 참조한다.

---

## UseCase 구조

UseCase는 Inbound Port(인터페이스) + Application Service(구현)로 구성된다.

| 파일 | 위치 | 역할 |
|------|------|------|
| `CreateOrderUseCase.kt` | `application/port/in/` | Inbound Port (인터페이스) |
| `GetOrderUseCase.kt` | `application/port/in/` | Inbound Port (인터페이스) |
| `CreateOrderCommand.kt` | `application/port/in/command/` | Command 객체 |
| `GetOrderQuery.kt` | `application/port/in/command/` | Query 객체 |
| `OrderService.kt` | `application/service/` | UseCase 구현 |

---

## Inbound Port (UseCase 인터페이스)

```kotlin
// application/port/in/CreateOrderUseCase.kt
interface CreateOrderUseCase {
    suspend fun execute(command: CreateOrderCommand): Order
}

// application/port/in/GetOrderUseCase.kt
interface GetOrderUseCase {
    suspend fun execute(query: GetOrderQuery): Order?
}

// application/port/in/GetOrdersStreamUseCase.kt
interface GetOrdersStreamUseCase {
    fun execute(query: GetOrdersStreamQuery): Flow<Order>
}
```

| 규칙 | 설명 |
|------|------|
| 단일 메서드 | 하나의 UseCase = 하나의 `execute()` 메서드 |
| Command/Query 입력 | 입력은 Command(변경) 또는 Query(조회) 객체 |
| `suspend` / `Flow` | 단일 결과는 `suspend fun`, 스트림은 `Flow<T>` 반환 |
| Spring 무관 | Port 인터페이스에 Spring 어노테이션 금지 |

---

## Command & Query

### Command (변경 요청)

```kotlin
// application/port/in/command/CreateOrderCommand.kt
data class CreateOrderCommand(
    val customerId: String,
    val items: List<OrderItemCommand>,
) {
    init {
        require(customerId.isNotBlank()) { "customerId는 필수" }
        require(items.isNotEmpty()) { "주문 항목은 1개 이상 필요" }
    }
}

data class OrderItemCommand(
    val productId: String,
    val quantity: Int,
) {
    init {
        require(quantity > 0) { "수량은 1 이상" }
    }
}
```

### Query (조회 요청)

```kotlin
// application/port/in/command/GetOrderQuery.kt
data class GetOrderQuery(val orderId: String)

data class GetOrdersStreamQuery(
    val customerId: String,
    val status: OrderStatus? = null,
)
```

| 규칙 | 설명 |
|------|------|
| 자기 검증 | `init` 블록에서 `require()`로 입력 검증 |
| 불변 | `data class` + `val` 전용 |
| DTO와 분리 | Command/Query는 Request DTO가 아님. Adapter에서 변환 |

---

## Application Service (UseCase 구현)

```kotlin
// application/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productRepository: ProductRepository,
    private val eventPublisher: DomainEventPublisher,
) : CreateOrderUseCase, GetOrderUseCase, ConfirmOrderUseCase {

    override suspend fun execute(command: CreateOrderCommand): Order {
        val products = command.items.map { item ->
            productRepository.findById(item.productId)
                ?: throw ProductNotFoundException(ProductId(item.productId))
        }
        val order = Order.create(CustomerId(command.customerId))
        command.items.forEachIndexed { i, item ->
            order.addItem(
                productId = products[i].id,
                quantity = Quantity(item.quantity),
                unitPrice = products[i].price,
            )
        }
        return orderRepository.save(order)
    }

    override suspend fun execute(query: GetOrderQuery): Order? =
        orderRepository.findById(query.orderId)

    override suspend fun execute(command: ConfirmOrderCommand): Order {
        val order = orderRepository.findById(command.orderId)
            ?: throw OrderNotFoundException(OrderId(command.orderId))
        val event = order.confirm()
        val saved = orderRepository.save(order)
        eventPublisher.publish(event)
        return saved
    }
}
```

| 규칙 | 설명 |
|------|------|
| `@Service` 허용 | Application Service에는 Spring `@Service` 사용 가능 |
| Port 구현 | 하나의 Service가 여러 UseCase Port를 구현할 수 있음 |
| Outbound Port 의존 | Repository 등 Outbound Port 인터페이스에만 의존 |
| 오케스트레이션 | 도메인 로직은 도메인 모델에, Service는 흐름 조율만 |
| 트랜잭션 관리 | `@Transactional` 적용 시 Application Service에서 |

---

## UseCase 작성 기준

### 필요한 경우

| 상황 | 이유 |
|------|------|
| 여러 Aggregate/Repository를 조합 | 도메인 간 오케스트레이션이 필요 |
| Domain Event 발행 | 사이드 이펙트 처리 |
| 트랜잭션 경계 정의 | `@Transactional` 적용 |
| 입력 검증 + 도메인 호출 조합 | 여러 단계의 워크플로우 |

### 불필요한 경우

| 상황 | 대안 |
|------|------|
| 단순 CRUD 위임 | Handler에서 Repository Port 직접 호출 가능 (단, Port를 통해) |
| 단순 조회 | Query 전용 경량 Service 또는 Handler에서 직접 처리 |

---

## 코루틴 패턴

### 순차 호출

```kotlin
override suspend fun execute(command: UpdateOrderCommand): Order {
    val order = orderRepository.findById(command.orderId)
        ?: throw OrderNotFoundException(OrderId(command.orderId))
    order.updateStatus(command.status)
    val saved = orderRepository.save(order)
    log.info("주문 갱신: ${saved.id}")
    return saved
}
```

### Flow 스트리밍

```kotlin
override fun execute(query: GetOrdersStreamQuery): Flow<Order> =
    orderRepository.findByCustomerId(query.customerId)
        .filter { order ->
            query.status?.let { order.status == it } ?: true
        }
```

### 여러 Repository 병렬 조합

```kotlin
override suspend fun execute(query: GetDashboardQuery): Dashboard = coroutineScope {
    val orderCount = async { orderRepository.countByCustomerId(query.customerId) }
    val totalAmount = async { orderRepository.sumAmountByCustomerId(query.customerId) }
    val favoriteCount = async { productRepository.countFavorites(query.customerId) }
    Dashboard(orderCount.await(), totalAmount.await(), favoriteCount.await())
}
```

---

## DO / DON'T

```kotlin
// DO -- UseCase는 오케스트레이션만, 비즈니스 로직은 도메인 모델에
override suspend fun execute(command: ConfirmOrderCommand): Order {
    val order = orderRepository.findById(command.orderId)
        ?: throw OrderNotFoundException(OrderId(command.orderId))
    order.confirm()                    // 도메인 메서드 호출
    return orderRepository.save(order)
}

// DON'T -- UseCase에 비즈니스 로직
override suspend fun execute(command: ConfirmOrderCommand): Order {
    val order = orderRepository.findById(command.orderId)!!
    if (order.status != OrderStatus.DRAFT) throw IllegalStateException()  // 금지
    return orderRepository.save(order.copy(status = OrderStatus.CONFIRMED))
}

// DO -- Command로 입력 전달
data class CreateOrderCommand(val customerId: String, val items: List<OrderItemCommand>)

// DON'T -- Request DTO를 UseCase에 전달
suspend fun execute(request: CreateOrderRequest): Order  // 금지: Adapter 타입 유입

// DO -- Outbound Port 인터페이스에 의존
class OrderService(private val orderRepository: OrderRepository)

// DON'T -- 구현체에 직접 의존
class OrderService(private val orderPersistenceAdapter: OrderPersistenceAdapter)  // 금지
```

---

## 변경 시 체크리스트

- [ ] UseCase가 단일 책임을 유지하는가?
- [ ] 입력이 Command/Query 객체인가? (Request DTO 아님)
- [ ] 비즈니스 로직이 도메인 모델에 있는가? (Service가 아닌)
- [ ] Outbound Port 인터페이스에만 의존하는가?
- [ ] 단일 결과는 `suspend`, 스트림은 `Flow`를 사용하는가?
- [ ] 새 UseCase 추가 시 테스트를 먼저 작성했는가? (TDD)
