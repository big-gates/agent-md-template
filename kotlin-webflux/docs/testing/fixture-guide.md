# Test Fixture Guide

> **트리거**: 테스트 픽스처, Fake 구현, 테스트 데이터 빌더를 작성할 때 이 문서를 참조한다.

---

## Fake vs Mock

| 기준 | Fake | Mock |
|------|------|------|
| 정의 | 인터페이스의 간단한 인메모리 구현 | 프레임워크가 생성하는 동작 기록 객체 |
| 선호도 | **선호** | 필요 시에만 |
| 장점 | 읽기 쉽고, 리팩터링에 강하고, 빠름 | 설정이 간편 |
| 단점 | 구현 코드가 필요 | 테스트가 구현에 결합됨 |
| 사용 시점 | Domain, UseCase 테스트 | 외부 라이브러리 의존 격리 |

---

## Fake Repository

```kotlin
// fixture/FakeOrderRepository.kt
class FakeOrderRepository : OrderRepository {
    private val store = ConcurrentHashMap<String, Order>()

    fun clear() = store.clear()

    override suspend fun save(order: Order): Order {
        store[order.id.value] = order
        return order
    }

    override suspend fun findById(id: String): Order? =
        store[id]

    override fun findByCustomerId(customerId: String): Flow<Order> =
        store.values
            .filter { it.customerId.value == customerId }
            .asFlow()

    override suspend fun existsById(id: String): Boolean =
        store.containsKey(id)

    override suspend fun deleteById(id: String) {
        store.remove(id)
    }
}
```

---

## Fake Event Publisher

```kotlin
// fixture/FakeEventPublisher.kt
class FakeEventPublisher : DomainEventPublisher {
    private val _events = mutableListOf<Any>()
    val events: List<Any> get() = _events.toList()

    fun clear() = _events.clear()

    override suspend fun publish(event: Any) {
        _events.add(event)
    }
}
```

---

## Fake External Port

```kotlin
// fixture/FakePaymentPort.kt
class FakePaymentPort : PaymentPort {
    private var shouldFail = false
    private val _processedPayments = mutableListOf<Pair<String, Money>>()
    val processedPayments: List<Pair<String, Money>> get() = _processedPayments.toList()

    fun setShouldFail(fail: Boolean) { shouldFail = fail }
    fun clear() {
        _processedPayments.clear()
        shouldFail = false
    }

    override suspend fun processPayment(orderId: String, amount: Money): PaymentResult =
        if (shouldFail) {
            throw PaymentFailedException("Fake payment error")
        } else {
            _processedPayments.add(orderId to amount)
            PaymentResult("tx-${orderId}", PaymentStatus.APPROVED)
        }
}
```

---

## Test Fixture (테스트 데이터 팩토리)

테스트에 필요한 도메인 객체를 생성하는 팩토리이다.

```kotlin
// fixture/OrderFixture.kt
object OrderFixture {

    fun draftOrder(
        customerId: String = "customer-1",
        id: String = UUID.randomUUID().toString(),
    ): Order = Order.reconstitute(
        id = OrderId(id),
        customerId = CustomerId(customerId),
        items = emptyList(),
        status = OrderStatus.DRAFT,
        createdAt = LocalDateTime.of(2025, 1, 1, 0, 0),
    )

    fun draftOrderWithItems(
        customerId: String = "customer-1",
        itemCount: Int = 2,
    ): Order {
        val order = draftOrder(customerId)
        repeat(itemCount) { i ->
            order.addItem(
                productId = ProductId("product-${i + 1}"),
                quantity = Quantity(1),
                unitPrice = Money(10000),
            )
        }
        return order
    }

    fun confirmedOrder(customerId: String = "customer-1"): Order {
        val order = draftOrderWithItems(customerId)
        order.confirm()
        return order
    }

    fun shippedOrder(customerId: String = "customer-1"): Order =
        Order.reconstitute(
            id = OrderId(UUID.randomUUID().toString()),
            customerId = CustomerId(customerId),
            items = listOf(
                OrderItem(ProductId("p1"), Quantity(1), Money(10000)),
            ),
            status = OrderStatus.SHIPPED,
            createdAt = LocalDateTime.of(2025, 1, 1, 0, 0),
        )

    fun orderWithAmount(amount: Long): Order {
        val order = draftOrder()
        order.addItem(ProductId("p1"), Quantity(1), Money(amount))
        return order
    }

    fun orderWithItemCount(count: Int, unitPrice: Money): Order {
        val order = draftOrder()
        repeat(count) { i ->
            order.addItem(ProductId("p${i + 1}"), Quantity(1), unitPrice)
        }
        return order
    }
}
```

```kotlin
// fixture/ProductFixture.kt
object ProductFixture {

    fun product(
        id: String = "product-1",
        price: Long = 10000,
    ) = Product(
        id = ProductId(id),
        name = "테스트 상품",
        price = Money(price),
        stock = Quantity(100),
    )
}
```

---

## Fake 작성 규칙

| 규칙 | 설명 |
|------|------|
| Port 인터페이스 구현 | 도메인의 Outbound Port를 구현 |
| 인메모리 저장소 | `ConcurrentHashMap`으로 간단한 저장소 구현 |
| `clear()` 메서드 | 각 테스트 전 초기화용 |
| `shouldFail` 플래그 | 에러 시나리오 테스트용 |
| 검증 가능한 상태 | `events`, `processedPayments` 등 행위 기록 노출 |
| 코루틴 반환 | `suspend fun`으로 단일 값 반환, `Flow`로 스트림 반환 |

---

## Fixture 작성 규칙

| 규칙 | 설명 |
|------|------|
| `object` 사용 | 싱글톤 팩토리 |
| 기본값 제공 | 모든 파라미터에 적절한 기본값 |
| 상태별 팩토리 | `draftOrder()`, `confirmedOrder()`, `shippedOrder()` |
| `reconstitute` 사용 | DB 재구성용 팩토리 메서드로 상태 직접 설정 |
| 최소 데이터 | 테스트에 필요한 최소한의 데이터만 설정 |

---

## 변경 시 체크리스트

- [ ] 새 Port/Repository에 대응하는 Fake가 있는가?
- [ ] Fake가 모든 인터페이스 메서드를 구현하는가?
- [ ] Fake에 `clear()` 메서드가 있는가?
- [ ] 에러 시나리오 테스트를 위한 `shouldFail` 플래그가 있는가?
- [ ] Fixture가 기본값을 제공하여 테스트에서 간결하게 사용 가능한가?
- [ ] Fixture가 다양한 상태(DRAFT, CONFIRMED 등)를 제공하는가?
