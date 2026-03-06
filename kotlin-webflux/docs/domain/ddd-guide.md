# DDD (Domain-Driven Design) Guide

> **트리거**: 도메인 모델을 설계·수정할 때, Aggregate/Entity/VO/Domain Event를 추가할 때 이 문서를 참조한다.

---

## DDD 전술 패턴 (Tactical Patterns)

| 패턴 | 설명 | 위치 |
|------|------|------|
| **Aggregate Root** | 일관성 경계의 루트 엔티티. 외부에서 Aggregate 내부에 직접 접근 금지 | `domain/model/` |
| **Entity** | 고유 식별자를 가진 도메인 객체 | `domain/model/` |
| **Value Object** | 식별자 없이 값으로만 동등성을 판단하는 불변 객체 | `domain/model/` |
| **Domain Event** | 도메인에서 발생한 비즈니스적으로 의미 있는 사건 | `domain/event/` |
| **Domain Service** | 특정 Entity/VO에 속하지 않는 상태 없는 도메인 로직 | `domain/service/` |
| **Repository** | Aggregate 단위의 저장소 인터페이스 | `application/port/out/` |

---

## Aggregate Root

Aggregate는 트랜잭션 일관성의 단위이다. Aggregate Root만 외부에서 접근 가능하다.

```kotlin
// domain/model/Order.kt -- Aggregate Root
class Order private constructor(
    val id: OrderId,
    val customerId: CustomerId,
    private val _items: MutableList<OrderItem>,
    private var _status: OrderStatus,
    val createdAt: LocalDateTime,
) {
    val items: List<OrderItem> get() = _items.toList()
    val status: OrderStatus get() = _status
    val totalAmount: Money get() = _items.fold(Money.ZERO) { acc, item -> acc + item.subtotal }

    fun addItem(productId: ProductId, quantity: Quantity, unitPrice: Money): OrderItem {
        check(_status == OrderStatus.DRAFT) { "주문이 DRAFT 상태에서만 항목 추가 가능" }
        val item = OrderItem(
            productId = productId,
            quantity = quantity,
            unitPrice = unitPrice,
        )
        _items.add(item)
        return item
    }

    fun removeItem(productId: ProductId) {
        check(_status == OrderStatus.DRAFT) { "주문이 DRAFT 상태에서만 항목 삭제 가능" }
        _items.removeAll { it.productId == productId }
    }

    fun confirm(): OrderConfirmedEvent {
        check(_status == OrderStatus.DRAFT) { "DRAFT 상태에서만 확정 가능" }
        check(_items.isNotEmpty()) { "주문 항목이 비어 있으면 확정 불가" }
        _status = OrderStatus.CONFIRMED
        return OrderConfirmedEvent(id, customerId, totalAmount)
    }

    fun cancel(): OrderCancelledEvent {
        check(_status != OrderStatus.SHIPPED) { "배송 시작 후 취소 불가" }
        _status = OrderStatus.CANCELLED
        return OrderCancelledEvent(id, customerId)
    }

    companion object {
        fun create(customerId: CustomerId): Order =
            Order(
                id = OrderId.generate(),
                customerId = customerId,
                _items = mutableListOf(),
                _status = OrderStatus.DRAFT,
                createdAt = LocalDateTime.now(),
            )
    }
}
```

| 규칙 | 설명 |
|------|------|
| **팩토리 메서드** | 생성은 `companion object`의 팩토리 메서드 또는 생성자를 통해. 불변식을 보장 |
| **비즈니스 메서드** | 상태 변경은 반드시 Aggregate Root의 메서드를 통해. 직접 필드 변경 금지 |
| **불변식 검증** | `check()`/`require()`로 비즈니스 규칙 위반을 즉시 차단 |
| **Domain Event 반환** | 상태 변경 시 Domain Event를 반환하여 사이드 이펙트 처리 |
| **내부 컬렉션 보호** | `MutableList`는 private, 외부에는 `List`로 노출 |

---

## Entity

고유 식별자로 동등성을 판단한다.

```kotlin
// domain/model/OrderItem.kt -- Aggregate 내부 Entity
class OrderItem(
    val productId: ProductId,
    val quantity: Quantity,
    val unitPrice: Money,
) {
    val subtotal: Money get() = unitPrice * quantity.value

    override fun equals(other: Any?): Boolean =
        other is OrderItem && productId == other.productId

    override fun hashCode(): Int = productId.hashCode()
}
```

| 규칙 | 설명 |
|------|------|
| 식별자 기반 동등성 | `equals`/`hashCode`는 식별자(ID)로 판단 |
| Aggregate 내부 Entity | Aggregate Root를 통해서만 접근 (외부 직접 참조 금지) |

---

## Value Object

식별자 없이 모든 속성 값으로 동등성을 판단하는 불변 객체이다.

```kotlin
// domain/model/Money.kt
data class Money(val amount: Long, val currency: Currency = Currency.KRW) {
    init {
        require(amount >= 0) { "금액은 0 이상이어야 한다: $amount" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "통화가 다르면 연산 불가" }
        return Money(amount + other.amount, currency)
    }

    operator fun times(multiplier: Int): Money =
        Money(amount * multiplier, currency)

    companion object {
        val ZERO = Money(0)
    }
}

// domain/model/OrderId.kt
@JvmInline
value class OrderId(val value: String) {
    companion object {
        fun generate(): OrderId = OrderId(UUID.randomUUID().toString())
    }
}

// domain/model/CustomerId.kt
@JvmInline
value class CustomerId(val value: String)

// domain/model/Quantity.kt
@JvmInline
value class Quantity(val value: Int) {
    init {
        require(value > 0) { "수량은 1 이상이어야 한다: $value" }
    }
}

// domain/model/OrderStatus.kt
enum class OrderStatus {
    DRAFT, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

| 규칙 | 설명 |
|------|------|
| **불변** | `val` 전용. 생성 후 변경 불가 |
| **값 동등성** | `data class` 또는 `value class` 사용. 모든 필드로 동등성 판단 |
| **자기 검증** | `init` 블록에서 `require()`로 유효성 검증 |
| **원시 타입 래핑** | ID, 금액, 수량 등 의미 있는 타입으로 래핑 (`value class` 활용) |
| **비즈니스 연산** | 해당 VO와 관련된 연산은 VO 내부에 정의 |

---

## Domain Event

비즈니스적으로 의미 있는 사건을 표현한다.

```kotlin
// domain/event/OrderConfirmedEvent.kt
data class OrderConfirmedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    val totalAmount: Money,
    val occurredAt: LocalDateTime = LocalDateTime.now(),
)

// domain/event/OrderCancelledEvent.kt
data class OrderCancelledEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    val occurredAt: LocalDateTime = LocalDateTime.now(),
)
```

| 규칙 | 설명 |
|------|------|
| 과거형 네이밍 | `{Domain}{Action}Event` — 이미 발생한 사실 (e.g., `OrderConfirmedEvent`) |
| 불변 | `data class` + `val` 전용 |
| 발생 시각 포함 | `occurredAt` 필드 포함 |
| Aggregate Root가 생성 | Domain Event는 Aggregate Root의 비즈니스 메서드가 반환 |

### Domain Event 발행

```kotlin
// application/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: DomainEventPublisher,
) : ConfirmOrderUseCase {

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

---

## Domain Service

특정 Entity/VO에 속하지 않는 도메인 로직을 담는다.

```kotlin
// domain/service/PricingService.kt
class PricingService {
    fun calculateDiscount(
        order: Order,
        memberGrade: MemberGrade,
    ): Money {
        val baseDiscount = when (memberGrade) {
            MemberGrade.VIP -> order.totalAmount * 0.1
            MemberGrade.GOLD -> order.totalAmount * 0.05
            else -> Money.ZERO
        }
        val bulkDiscount = if (order.items.size >= 10) order.totalAmount * 0.03 else Money.ZERO
        return baseDiscount + bulkDiscount
    }
}
```

| 규칙 | 설명 |
|------|------|
| 상태 없음 | 필드를 갖지 않는 순수 함수 집합 |
| 여러 Aggregate 조합 | 단일 Aggregate에 넣기 어려운 로직 |
| 도메인 전용 | Spring 의존 없는 순수 Kotlin |

---

## Domain Exception

```kotlin
// domain/exception/OrderNotFoundException.kt
class OrderNotFoundException(val orderId: OrderId) :
    RuntimeException("주문을 찾을 수 없습니다: ${orderId.value}")

// domain/exception/InsufficientStockException.kt
class InsufficientStockException(val productId: ProductId, val requested: Quantity) :
    RuntimeException("재고 부족: 상품 ${productId.value}, 요청 수량 ${requested.value}")
```

---

## Aggregate 설계 원칙

| 원칙 | 설명 |
|------|------|
| 작게 유지 | Aggregate는 가능한 한 작게. 필요한 Entity/VO만 포함 |
| ID로 참조 | Aggregate 간 참조는 ID(Value Object)로. 직접 객체 참조 금지 |
| 트랜잭션 단위 | 하나의 트랜잭션에서 하나의 Aggregate만 수정 |
| 일관성 경계 | Aggregate 내부는 즉시 일관성, Aggregate 간은 결과적 일관성 (Domain Event) |
| Repository = Aggregate | Repository는 Aggregate Root 단위로 정의. 내부 Entity용 별도 Repository 금지 |

---

## 변경 시 체크리스트

- [ ] Aggregate Root만 외부에서 접근 가능한가?
- [ ] 상태 변경이 Aggregate Root의 비즈니스 메서드를 통하는가?
- [ ] 불변식(비즈니스 규칙)이 `check()`/`require()`로 검증되는가?
- [ ] Value Object가 불변이며 자기 검증을 하는가?
- [ ] 원시 타입 대신 Value Object(`value class`)를 사용하는가?
- [ ] Domain Event가 과거형으로 명명되었는가?
- [ ] 도메인 모델에 프레임워크 어노테이션이 없는가?
- [ ] Aggregate 간 참조가 ID를 통하는가? (직접 객체 참조 아님)
