# WebFlux Guide

> **트리거**: Controller, 엔드포인트를 작성·수정할 때 이 문서를 참조한다.

---

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **@RestController** | 어노테이션 기반 Controller 사용 |
| **코루틴 전용** | 모든 Controller 메서드는 `suspend` 함수. 블로킹 호출 금지 |
| **@RequestMapping** | 클래스 레벨에 공통 경로, 메서드 레벨에 세부 경로 매핑 |
| **Flow 스트리밍** | 실시간 스트림은 `Flow`로 처리 |

---

## Controller

Controller는 HTTP 요청을 처리하는 suspend 함수를 정의한다.

```kotlin
// adapter/in/web/controller/OrderController.kt
@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderUseCase: GetOrderUseCase,
    private val getOrdersStreamUseCase: GetOrdersStreamUseCase,
) {
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@RequestBody body: CreateOrderRequest): OrderResponse {
        require(body.customerId.isNotBlank()) { "customerId는 필수" }
        require(body.items.isNotEmpty()) { "주문 항목은 1개 이상 필요" }
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

    @GetMapping("/stream/customer/{customerId}", produces = [MediaType.APPLICATION_NDJSON_VALUE])
    fun streamByCustomer(@PathVariable customerId: String): Flow<OrderResponse> {
        return getOrdersStreamUseCase.execute(GetOrdersStreamQuery(customerId))
            .map { it.toResponse() }
    }
}
```

| 규칙 | 설명 |
|------|------|
| `@RestController` | Controller는 `@RestController`로 등록 |
| `suspend fun` | 단일 결과를 반환하는 모든 메서드는 suspend |
| `Flow<T>` 반환 | 스트리밍 응답은 `Flow`를 직접 반환 (suspend 아님) |
| UseCase(Port) 의존 | Repository 직접 호출 금지 |
| DTO 변환 | Controller에서 Request DTO -> Command, Domain -> Response DTO 변환 |
| 비즈니스 로직 금지 | Controller에 비즈니스 로직을 넣지 않는다 |

---

## Request / Response DTO

```kotlin
// adapter/in/web/dto/request/CreateOrderRequest.kt
data class CreateOrderRequest(
    val customerId: String,
    val items: List<OrderItemRequest>,
) {
    fun toCommand() = CreateOrderCommand(
        customerId = customerId,
        items = items.map { OrderItemCommand(it.productId, it.quantity) },
    )
}

data class OrderItemRequest(
    val productId: String,
    val quantity: Int,
)

// adapter/in/web/dto/response/OrderResponse.kt
data class OrderResponse(
    val id: String,
    val customerId: String,
    val status: String,
    val totalAmount: Long,
    val items: List<OrderItemResponse>,
    val createdAt: String,
)

// adapter/in/web/mapper/OrderDtoMapper.kt -- 확장 함수로 매핑
fun Order.toResponse() = OrderResponse(
    id = id.value,
    customerId = customerId.value,
    status = status.name,
    totalAmount = totalAmount.amount,
    items = items.map { it.toResponse() },
    createdAt = createdAt.toString(),
)
```

| 규칙 | 설명 |
|------|------|
| DTO는 Adapter에만 | `adapter/in/web/dto/` 에만 존재 |
| 매퍼는 확장 함수 | Domain -> Response 변환은 확장 함수로 |
| 도메인 모델에 직렬화 어노테이션 금지 | `@Serializable`, `@JsonProperty` 등은 DTO에만 |

---

## 요청 검증 (Validation)

| 검증 위치 | 검증 내용 |
|-----------|----------|
| Adapter (Controller) | 포맷 검증 (null, blank, 형식) |
| Command `init` | 비즈니스 입력 검증 (값 범위, 필수 필드) |
| Domain Model | 불변식 검증 (비즈니스 규칙) |

---

## Flow 스트리밍 응답

```kotlin
// NDJSON 스트리밍 - Flow를 직접 반환
@GetMapping("/stream", produces = [MediaType.APPLICATION_NDJSON_VALUE])
fun streamOrders(): Flow<OrderResponse> {
    return orderUseCase.executeStream()
        .map { it.toResponse() }
}

// SSE 스트리밍 (Flow -> Flux 변환 필요)
@GetMapping("/sse", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun streamSse(): Flux<ServerSentEvent<NotificationResponse>> {
    return notificationUseCase.execute(query)
        .map { ServerSentEvent.builder(it.toResponse()).event("notification").build() }
        .asFlux()  // Flow -> Flux 변환 (SSE에서는 Flux 필요)
}
```

---

## 변경 시 체크리스트

- [ ] Controller가 UseCase(Port)만 호출하는가?
- [ ] DTO가 Adapter 내부에만 존재하는가?
- [ ] Controller에 비즈니스 로직이 없는가?
- [ ] 모든 단일 응답 메서드가 `suspend`인가?
- [ ] 블로킹 호출이 없는가?
