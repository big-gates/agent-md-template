# TDD Guide

> **트리거**: 새 기능/버그 수정을 시작할 때, TDD 워크플로우를 확인할 때 이 문서를 참조한다.

---

## TDD 사이클

| 단계 | 이름 | 수행 |
|------|------|------|
| 1 | **Red** | 실패하는 테스트를 먼저 작성 |
| 2 | **Green** | 테스트를 통과하는 최소한의 코드 작성 |
| 3 | **Refactor** | 코드 정리 (테스트는 여전히 통과) |

모든 새 기능/버그 수정은 반드시 이 사이클을 따른다.

---

## 적용 순서

| 상황 | TDD 적용 |
|------|---------|
| 새 도메인 모델 추가 | 모델 테스트 먼저 (불변식 검증) → 모델 구현 |
| 새 UseCase 추가 | UseCase 테스트 먼저 (Fake Repository) → UseCase 구현 |
| 새 Repository Adapter | Adapter 통합 테스트 먼저 (Testcontainers) → Adapter 구현 |
| 새 Handler 추가 | Handler 통합 테스트 먼저 (WebTestClient) → Handler 구현 |
| 새 Domain Service | 단위 테스트 먼저 → Domain Service 구현 |
| 버그 수정 | 버그를 재현하는 테스트 먼저 → 수정 → 테스트 통과 확인 |

---

## 테스트 피라미드

| 레벨 | 대상 | 비율 |
|------|------|------|
| **Unit** | Domain Model, Domain Service, UseCase, Mapper | 70%+ |
| **Integration** | Repository Adapter, Handler, 외부 API 연동 | 25% |
| **E2E** | 주요 사용자 시나리오 | 5% |

---

## 레이어별 TDD 전략

### 1. Domain Model (Inside-Out)

가장 안쪽부터 시작한다.

```kotlin
// 1) Red -- 테스트 먼저
class OrderTest {
    @Test
    fun `주문 생성 시 DRAFT 상태이다`() {
        val order = Order.create(CustomerId("c1"))
        assertEquals(OrderStatus.DRAFT, order.status)
        assertTrue(order.items.isEmpty())
    }

    @Test
    fun `DRAFT 상태에서 항목을 추가할 수 있다`() {
        val order = Order.create(CustomerId("c1"))
        order.addItem(ProductId("p1"), Quantity(2), Money(10000))
        assertEquals(1, order.items.size)
    }

    @Test
    fun `CONFIRMED 상태에서 항목 추가 시 예외`() {
        val order = createConfirmedOrder()
        assertThrows<IllegalStateException> {
            order.addItem(ProductId("p2"), Quantity(1), Money(5000))
        }
    }
}

// 2) Green -- 최소 구현
// 3) Refactor -- 정리
```

### 2. UseCase

```kotlin
// 1) Red
class CreateOrderServiceTest {
    private val fakeOrderRepository = FakeOrderRepository()
    private val fakeProductRepository = FakeProductRepository()
    private val service = OrderService(fakeOrderRepository, fakeProductRepository, FakeEventPublisher())

    @Test
    fun `주문 생성 성공`() = runTest {
        fakeProductRepository.save(Product(ProductId("p1"), Money(10000)))

        val command = CreateOrderCommand(
            customerId = "c1",
            items = listOf(OrderItemCommand("p1", 2)),
        )

        val order = service.execute(command)

        assertEquals(OrderStatus.DRAFT, order.status)
        assertEquals(1, order.items.size)
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
}
```

### 3. Handler (Integration)

```kotlin
class OrderHandlerTest {
    @Test
    fun `POST 주문 생성 - 201 반환`() {
        val client = WebTestClient.bindToRouterFunction(
            OrderRouter().orderRoutes(OrderHandler(fakeOrderUseCase, ...))
        ).build()

        client.post()
            .uri("/api/v1/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"customerId":"c1","items":[{"productId":"p1","quantity":2}]}""")
            .exchange()
            .expectStatus().isCreated
            .expectBody()
            .jsonPath("$.status").isEqualTo("DRAFT")
    }
}
```

---

## TDD 원칙

| 원칙 | 설명 |
|------|------|
| **테스트 먼저** | 구현 코드보다 테스트를 먼저 작성 |
| **최소 구현** | 테스트를 통과하는 가장 간단한 코드만 작성 |
| **작은 단계** | 한 번에 하나의 테스트만 추가 |
| **삼각측량** | 여러 테스트 케이스로 일반화를 유도 |
| **리팩터 안전망** | 모든 리팩터링은 기존 테스트가 통과하는 상태에서 수행 |
| **Fake 선호** | Mock보다 Fake 사용 (리팩터링에 강함) |

---

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 기능 요청 | **테스트 먼저** 작성 후 구현 |
| 버그 수정 요청 | 버그를 재현하는 **테스트 먼저** 작성 |
| 리팩터링 요청 | 기존 테스트가 있는지 확인 → 없으면 먼저 추가 → 리팩터 |
| 기존 코드 변경 | 기존 테스트 확인 → 테스트 통과 유지 → 필요시 테스트 갱신 |

---

## 변경 시 체크리스트

- [ ] TDD 사이클(Red → Green → Refactor)을 따랐는가?
- [ ] 새 로직에 대응하는 테스트가 있는가?
- [ ] 테스트가 구현 세부사항이 아닌 동작(행위)을 검증하는가?
- [ ] 기존 테스트가 깨지지 않는가?
- [ ] 경계값/에러 케이스 테스트를 포함하는가?
