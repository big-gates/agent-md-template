# WebFlux Guide

> **트리거**: Handler, Router, Functional Endpoint를 작성·수정할 때 이 문서를 참조한다.

---

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **Functional Endpoint** | `@Controller` 대신 Handler + coRouter 조합 사용 |
| **코루틴 전용** | 모든 Handler는 `suspend` 함수. 블로킹 호출 금지 |
| **coRouter** | `coRouter {}` DSL로 경로 매핑 |
| **Flow 스트리밍** | 실시간 스트림은 `Flow`로 처리 |

---

## Handler

Handler는 HTTP 요청을 처리하는 suspend 함수를 정의한다.

```kotlin
// adapter/in/web/handler/OrderHandler.kt
@Component
class OrderHandler(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderUseCase: GetOrderUseCase,
    private val getOrdersStreamUseCase: GetOrdersStreamUseCase,
) {
    suspend fun create(request: ServerRequest): ServerResponse {
        val body = request.awaitBody<CreateOrderRequest>()
        validate(body)
        val command = body.toCommand()
        val order = createOrderUseCase.execute(command)
        return ServerResponse.status(HttpStatus.CREATED)
            .bodyValueAndAwait(order.toResponse())
    }

    suspend fun getById(request: ServerRequest): ServerResponse {
        val id = request.pathVariable("id")
        val order = getOrderUseCase.execute(GetOrderQuery(id))
        return order?.let {
            ServerResponse.ok().bodyValueAndAwait(it.toResponse())
        } ?: ServerResponse.notFound().buildAndAwait()
    }

    suspend fun streamByCustomer(request: ServerRequest): ServerResponse {
        val customerId = request.pathVariable("customerId")
        val ordersFlow = getOrdersStreamUseCase.execute(GetOrdersStreamQuery(customerId))
            .map { it.toResponse() }
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_NDJSON)
            .bodyAndAwait(ordersFlow)
    }

    private fun validate(request: CreateOrderRequest) {
        require(request.customerId.isNotBlank()) { "customerId는 필수" }
        require(request.items.isNotEmpty()) { "주문 항목은 1개 이상 필요" }
    }
}
```

| 규칙 | 설명 |
|------|------|
| `@Component` | Handler는 Spring Bean으로 등록 |
| `suspend fun` | 모든 Handler 함수는 suspend |
| UseCase(Port) 의존 | Repository 직접 호출 금지 |
| DTO 변환 | Handler에서 Request DTO -> Command, Domain -> Response DTO 변환 |
| 비즈니스 로직 금지 | Handler에 비즈니스 로직을 넣지 않는다 |

---

## Router

경로 매핑을 정의한다. Handler와 분리하여 관리한다.

```kotlin
// adapter/in/web/router/OrderRouter.kt
@Configuration
class OrderRouter {
    @Bean
    fun orderRoutes(handler: OrderHandler) = coRouter {
        "/api/v1/orders".nest {
            POST("", handler::create)
            GET("/{id}", handler::getById)
            GET("/stream/customer/{customerId}", handler::streamByCustomer)
        }
    }
}
```

| 규칙 | 설명 |
|------|------|
| `coRouter {}` | 코루틴 DSL로 경로 정의 |
| 도메인별 분리 | Router는 도메인(Bounded Context) 단위로 분리 |
| 버전 포함 | `/api/v1/` 접두사 |
| `nest` 사용 | 공통 경로 접두사를 `nest`로 그룹핑 |

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
| Adapter (Handler) | 포맷 검증 (null, blank, 형식) |
| Command `init` | 비즈니스 입력 검증 (값 범위, 필수 필드) |
| Domain Model | 불변식 검증 (비즈니스 규칙) |

---

## Flow 스트리밍 응답

```kotlin
// NDJSON 스트리밍
suspend fun streamOrders(request: ServerRequest): ServerResponse {
    val flow = orderUseCase.executeStream()
        .map { it.toResponse() }
    return ServerResponse.ok()
        .contentType(MediaType.APPLICATION_NDJSON)
        .bodyAndAwait(flow)
}

// SSE 스트리밍 (Flow -> Flux 변환 필요)
suspend fun streamSse(request: ServerRequest): ServerResponse {
    val sseFlux = notificationUseCase.execute(query)
        .map { ServerSentEvent.builder(it.toResponse()).event("notification").build() }
        .asFlux()  // Flow -> Flux 변환 (SSE에서는 Flux 필요)
    return ServerResponse.ok()
        .contentType(MediaType.TEXT_EVENT_STREAM)
        .bodyAndAwait(sseFlux)
}
```

---

## 변경 시 체크리스트

- [ ] Handler가 UseCase(Port)만 호출하는가?
- [ ] DTO가 Adapter 내부에만 존재하는가?
- [ ] Handler에 비즈니스 로직이 없는가?
- [ ] Router와 Handler가 분리되어 있는가?
- [ ] 모든 Handler 함수가 `suspend`인가?
- [ ] 블로킹 호출이 없는가?
