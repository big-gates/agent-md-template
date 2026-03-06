# Unit Test Guide

> **트리거**: 단위 테스트, runTest, 코루틴/Flow 테스트를 작성할 때 이 문서를 참조한다.

---

## 테스트 위치

| 테스트 위치 | 대상 | 예시 파일 |
|------------|------|----------|
| `domain/model/` | Domain Model 테스트 | `OrderTest.kt`, `MoneyTest.kt`, `QuantityTest.kt` |
| `domain/service/` | Domain Service 테스트 | `PricingServiceTest.kt` |
| `application/service/` | UseCase 테스트 | `OrderServiceTest.kt` |
| `fixture/` | 공유 픽스처 / Fake | `OrderFixture.kt`, `FakeOrderRepository.kt` |

---

## 네이밍 규칙

### 테스트 클래스

`{대상클래스}Test` -- 예: `OrderTest`, `OrderServiceTest`

### 테스트 메서드

```kotlin
@Test
fun `주어진 조건_실행할 동작_기대 결과`() { }

// 예시
@Test
fun `DRAFT 상태에서_항목 추가_아이템 목록에 포함`() { }

@Test
fun `존재하지 않는 주문 ID로_조회_OrderNotFoundException 발생`() { }
```

---

## Domain Model 테스트

도메인 모델은 순수 Kotlin이므로 코루틴 없이 직접 테스트한다.

```kotlin
class OrderTest {

    @Test
    fun `주문 생성 시 DRAFT 상태이다`() {
        val order = Order.create(CustomerId("c1"))

        assertEquals(OrderStatus.DRAFT, order.status)
        assertTrue(order.items.isEmpty())
        assertEquals(Money.ZERO, order.totalAmount)
    }

    @Test
    fun `항목 추가 시 총액이 갱신된다`() {
        val order = Order.create(CustomerId("c1"))

        order.addItem(ProductId("p1"), Quantity(2), Money(10000))

        assertEquals(Money(20000), order.totalAmount)
    }

    @Test
    fun `주문 확정 시 CONFIRMED 상태로 전환되고 이벤트가 반환된다`() {
        val order = OrderFixture.draftOrderWithItems()

        val event = order.confirm()

        assertEquals(OrderStatus.CONFIRMED, order.status)
        assertIs<OrderConfirmedEvent>(event)
        assertEquals(order.id, event.orderId)
    }

    @Test
    fun `빈 주문은 확정할 수 없다`() {
        val order = Order.create(CustomerId("c1"))

        assertThrows<IllegalStateException> {
            order.confirm()
        }
    }

    @Test
    fun `배송 시작 후 취소 불가`() {
        val order = OrderFixture.shippedOrder()

        assertThrows<IllegalStateException> {
            order.cancel()
        }
    }
}
```

---

## Value Object 테스트

```kotlin
class MoneyTest {

    @Test
    fun `음수 금액은 생성 불가`() {
        assertThrows<IllegalArgumentException> {
            Money(-1)
        }
    }

    @Test
    fun `같은 통화 덧셈`() {
        val result = Money(1000) + Money(2000)
        assertEquals(Money(3000), result)
    }

    @Test
    fun `다른 통화 덧셈 시 예외`() {
        assertThrows<IllegalArgumentException> {
            Money(1000, Currency.KRW) + Money(2000, Currency.USD)
        }
    }

    @Test
    fun `수량 곱셈`() {
        val result = Money(1000) * 3
        assertEquals(Money(3000), result)
    }
}
```

---

## UseCase 테스트 (runTest)

suspend 함수인 UseCase는 `runTest`로 테스트한다.

```kotlin
class OrderServiceTest {
    private val fakeOrderRepository = FakeOrderRepository()
    private val fakeProductRepository = FakeProductRepository()
    private val fakeEventPublisher = FakeEventPublisher()
    private val service = OrderService(fakeOrderRepository, fakeProductRepository, fakeEventPublisher)

    @BeforeEach
    fun setup() {
        fakeOrderRepository.clear()
        fakeProductRepository.clear()
        fakeEventPublisher.clear()
    }

    @Test
    fun `주문 생성 성공`() = runTest {
        fakeProductRepository.save(ProductFixture.product("p1", 10000))

        val command = CreateOrderCommand(
            customerId = "c1",
            items = listOf(OrderItemCommand("p1", 2)),
        )

        val order = service.execute(command)

        assertNotNull(order.id)
        assertEquals("c1", order.customerId.value)
        assertEquals(1, order.items.size)
        assertEquals(Money(20000), order.totalAmount)
    }

    @Test
    fun `존재하지 않는 상품이면 에러`() = runTest {
        val command = CreateOrderCommand(
            customerId = "c1",
            items = listOf(OrderItemCommand("nonexistent", 1)),
        )

        assertThrows<ProductNotFoundException> {
            service.execute(command)
        }
    }

    @Test
    fun `주문 확정 시 이벤트 발행`() = runTest {
        val order = OrderFixture.draftOrderWithItems()
        fakeOrderRepository.save(order)

        val confirmed = service.execute(ConfirmOrderCommand(order.id.value))

        assertEquals(OrderStatus.CONFIRMED, confirmed.status)
        assertEquals(1, fakeEventPublisher.events.size)
        assertIs<OrderConfirmedEvent>(fakeEventPublisher.events.first())
    }
}
```

---

## Flow 테스트 (Turbine)

Flow 스트림은 Turbine 라이브러리로 테스트한다.

```kotlin
class NotificationServiceTest {
    private val eventBroadcaster = EventBroadcaster()
    private val fakeRepository = FakeNotificationRepository()
    private val service = NotificationService(eventBroadcaster, fakeRepository)

    @Test
    fun `실시간 알림 스트림 수신`() = runTest {
        val query = NotificationStreamQuery("u1")

        service.execute(query).test {
            // 이벤트 발행
            eventBroadcaster.publish(NotificationEvent(
                userId = UserId("u1"),
                message = "새 주문",
            ))

            val notification = awaitItem()
            assertEquals("새 주문", notification.message)

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Turbine 패턴

```kotlin
// 단일 값 검증
flow.test {
    val item = awaitItem()
    assertEquals("expected", item.name)
    awaitComplete()
}

// 여러 값 검증
flow.test {
    assertEquals(item1, awaitItem())
    assertEquals(item2, awaitItem())
    awaitComplete()
}

// 에러 검증
flow.test {
    val error = awaitError()
    assertIs<NotFoundException>(error)
}

// 타임아웃 설정
flow.test(timeout = 5.seconds) {
    val item = awaitItem()
    cancelAndIgnoreRemainingEvents()
}

// 무한 Flow 테스트
flow.test {
    val first = awaitItem()
    val second = awaitItem()
    cancelAndIgnoreRemainingEvents()  // 취소 후 남은 이벤트 무시
}
```

---

## Domain Service 테스트

```kotlin
class PricingServiceTest {
    private val pricingService = PricingService()

    @Test
    fun `VIP 등급은 10% 할인`() {
        val order = OrderFixture.orderWithAmount(100000)

        val discount = pricingService.calculateDiscount(order, MemberGrade.VIP)

        assertEquals(Money(10000), discount)
    }

    @Test
    fun `10개 이상 구매 시 추가 3% 할인`() {
        val order = OrderFixture.orderWithItemCount(10, Money(10000))

        val discount = pricingService.calculateDiscount(order, MemberGrade.NORMAL)

        assertEquals(Money(3000), discount)
    }
}
```

---

## 테스트 라이브러리

| 라이브러리 | 용도 |
|-----------|------|
| `JUnit 5` | 테스트 프레임워크 |
| `kotlinx-coroutines-test` | `runTest` 제공 (suspend 함수 테스트) |
| `app.cash.turbine:turbine` | Flow 테스트 (`.test { awaitItem() }`) |
| `kotlin.test` / `AssertJ` | Assertion |
| `MockK` | Mock (Fake 선호, 필요시에만) |

---

## 코루틴 테스트 규칙

| 규칙 | 설명 |
|------|------|
| `runTest` 필수 | 모든 suspend 함수 테스트에 사용. `runBlocking` 금지 |
| `Turbine` | Flow 테스트 시 `.test { awaitItem() }` 패턴 사용 |
| 직접 assert | suspend 함수 결과는 직접 변수에 받아 assertion |
| `assertThrows` | 에러 테스트는 `assertThrows<ExceptionType>` 사용 |

```kotlin
// DO
@Test
fun `데이터 로드 후 결과 확인`() = runTest {
    val result = service.execute(query)
    assertEquals(expected, result)
}

// DON'T -- runBlocking 사용
@Test
fun `데이터 로드 후 결과 확인`() = runBlocking {  // 금지
    val result = service.execute(query)
    assertEquals(expected, result)
}
```

---

## 변경 시 체크리스트

- [ ] 테스트 메서드 이름이 `주어진 조건_동작_기대 결과` 형식인가?
- [ ] suspend 함수 테스트에 `runTest`를 사용하는가?
- [ ] Flow 테스트에 Turbine을 사용하는가?
- [ ] Fake를 사용하는가? (Mock보다 Fake 선호)
- [ ] 경계값/에러 케이스를 포함하는가?
- [ ] 테스트가 외부 의존성 없이 독립적으로 실행되는가?
