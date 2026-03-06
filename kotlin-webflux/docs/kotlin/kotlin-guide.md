# Kotlin Guide

> **트리거**: Kotlin 이디엄, 타입 안전성, 코드 스타일을 확인할 때 이 문서를 참조한다.

---

## 불변성

```kotlin
// DO -- val 사용 (불변)
val items: List<Item> = listOf(item1, item2)
val config = AppConfig(timeout = 30)

// DON'T -- var 사용 (가변) -- 정당한 이유 없이
var items: MutableList<Item> = mutableListOf()

// DO -- 불변 컬렉션
val list: List<String> = listOf("a", "b")
val map: Map<String, Int> = mapOf("a" to 1)

// DON'T -- 가변 컬렉션 외부 노출
fun getItems(): MutableList<Item>  // 금지: 내부 상태 변경 가능
```

| 규칙 | 설명 |
|------|------|
| `val` 기본 | 변경이 필요하지 않으면 항상 `val` |
| 불변 컬렉션 반환 | 외부에는 `List`, `Map`, `Set` 반환 |
| `data class` | 값 객체에 사용. `val` 프로퍼티만 |
| `copy()` | 수정 시 `copy()`로 새 인스턴스 생성 |

---

## Null 안전성

```kotlin
// DO -- nullable 최소화
suspend fun findById(id: String): Order?               // null로 없음 표현
fun calculateTotal(items: List<Item>): Money            // 빈 리스트로 없음 표현

// DON'T -- 불필요한 nullable
fun getItems(): List<Item>?                             // 빈 리스트로 대체

// DO -- 안전 호출 체이닝
val city = user?.address?.city ?: "Unknown"

// DON'T -- 강제 언래핑
val city = user!!.address!!.city                        // 금지
```

| 규칙 | 설명 |
|------|------|
| Nullable 최소화 | suspend 함수에서는 `null` 반환으로 없음 표현 |
| `!!` 금지 | `requireNotNull()` 또는 `?: throw` 사용 |
| `?.let {}` | null 안전한 변환 |
| Elvis `?:` | 기본값 제공 |

---

## Scope Functions

```kotlin
// let -- null 안전 변환
user?.let { sendEmail(it) }

// apply -- 객체 초기화
val config = WebClientConfig().apply {
    baseUrl = "https://api.example.com"
    timeout = Duration.ofSeconds(5)
}

// run -- 객체 컨텍스트에서 실행 + 결과 반환
val result = order.run {
    validate()
    calculate()
}

// also -- 부수 효과 (로깅 등)
order.also { log.info("주문 생성: ${it.id}") }

// with -- 여러 메서드 호출
with(response) {
    statusCode = HttpStatus.OK
    headers.contentType = MediaType.APPLICATION_JSON
}
```

---

## 확장 함수

```kotlin
// DO -- 변환 로직에 확장 함수 사용
fun Order.toResponse() = OrderResponse(
    id = id.value,
    status = status.name,
    totalAmount = totalAmount.amount,
)

fun CreateOrderRequest.toCommand() = CreateOrderCommand(
    customerId = customerId,
    items = items.map { it.toCommand() },
)

// DON'T -- 비즈니스 로직을 확장 함수에
fun Order.confirm() { ... }  // 금지: 도메인 모델 메서드로 정의
```

| 규칙 | 설명 |
|------|------|
| 매퍼에 활용 | DTO <-> Domain 변환에 확장 함수 사용 |
| 유틸리티 성격 | 변환, 포맷팅 등 유틸 성격일 때 |
| 비즈니스 로직 금지 | 도메인 로직은 도메인 모델 내부에 정의 |

---

## Sealed Class / Interface

```kotlin
// 제한된 계층 구조 표현
sealed interface DomainEvent {
    val occurredAt: LocalDateTime
}

data class OrderCreatedEvent(
    val orderId: OrderId,
    override val occurredAt: LocalDateTime = LocalDateTime.now(),
) : DomainEvent

data class OrderCancelledEvent(
    val orderId: OrderId,
    val reason: String,
    override val occurredAt: LocalDateTime = LocalDateTime.now(),
) : DomainEvent

// when 사용 시 컴파일 타임 완전성 보장
fun handle(event: DomainEvent) = when (event) {
    is OrderCreatedEvent -> processCreation(event)
    is OrderCancelledEvent -> processCancellation(event)
    // 새 타입 추가 시 컴파일 에러 → 누락 방지
}
```

---

## Value Class (Inline Class)

원시 타입에 의미를 부여하면서 런타임 오버헤드가 없다.

```kotlin
@JvmInline
value class OrderId(val value: String)

@JvmInline
value class CustomerId(val value: String)

@JvmInline
value class Quantity(val value: Int) {
    init { require(value > 0) { "수량은 1 이상" } }
}

// 컴파일 타임에 타입 혼동 방지
suspend fun findOrder(orderId: OrderId): Order?          // OrderId와 CustomerId 혼동 방지
suspend fun findCustomer(customerId: CustomerId): Customer?
```

---

## 기타 이디엄

```kotlin
// 구조 분해
val (orderId, customerId, amount) = triple

// 범위 함수 체이닝 -- 리스트 처리
orders
    .filter { it.status == OrderStatus.CONFIRMED }
    .sortedByDescending { it.createdAt }
    .take(10)
    .map { it.toResponse() }

// require / check
fun addItem(quantity: Int) {
    require(quantity > 0) { "수량은 1 이상이어야 합니다" }  // 입력 검증
    check(status == OrderStatus.DRAFT) { "DRAFT 상태에서만 가능" }  // 상태 검증
}

// when 표현식
val discount = when (grade) {
    MemberGrade.VIP -> 0.10
    MemberGrade.GOLD -> 0.05
    MemberGrade.NORMAL -> 0.0
}
```

---

## 변경 시 체크리스트

- [ ] `var` 대신 `val`을 사용했는가?
- [ ] `!!` 없이 null을 안전하게 처리했는가?
- [ ] 원시 타입에 `value class`를 적용했는가? (ID, 금액 등)
- [ ] 확장 함수에 비즈니스 로직이 없는가?
- [ ] `sealed class/interface`로 제한된 타입을 표현했는가?
