# Error Handling Guide

> **트리거**: 에러 처리, 글로벌 예외 핸들러, 코루틴 에러 복구 패턴을 작성할 때 이 문서를 참조한다.

---

## 에러 처리 계층

| 단계 | 레이어 | 설명 |
|------|--------|------|
| 1 | Domain Exception | 비즈니스 규칙 위반 발생 |
| 2 | Application Service | 오케스트레이션 에러 전파 |
| 3 | Global Error Handler | HTTP 응답으로 변환 |
| 4 | HTTP Response | ErrorResponse DTO 반환 |

---

## Domain Exception

```kotlin
// domain/exception/DomainException.kt
abstract class DomainException(
    val code: String,
    override val message: String,
) : RuntimeException(message)

// domain/exception/OrderNotFoundException.kt
class OrderNotFoundException(orderId: OrderId) :
    DomainException("ORDER_NOT_FOUND", "주문을 찾을 수 없습니다: ${orderId.value}")

// domain/exception/InsufficientStockException.kt
class InsufficientStockException(productId: ProductId, requested: Quantity) :
    DomainException("INSUFFICIENT_STOCK", "재고 부족: 상품 ${productId.value}, 요청 ${requested.value}")

// domain/exception/InvalidOrderStateException.kt
class InvalidOrderStateException(current: OrderStatus, expected: OrderStatus) :
    DomainException("INVALID_ORDER_STATE", "주문 상태 전환 불가: $current -> $expected")
```

| 규칙 | 설명 |
|------|------|
| 코드 포함 | 클라이언트가 에러를 구분할 수 있는 고유 코드 |
| 도메인 언어 | 기술 용어가 아닌 비즈니스 용어로 메시지 작성 |
| 계층 분리 | 도메인 예외는 HTTP 상태 코드를 모른다 |

---

## 글로벌 에러 핸들러

```kotlin
// config/GlobalErrorHandler.kt
@Component
@Order(-2)
class GlobalErrorHandler(
    private val objectMapper: ObjectMapper,
) : ErrorWebExceptionHandler {

    override fun handle(exchange: ServerWebExchange, ex: Throwable): Mono<Void> {
        val (status, response) = mapException(ex)

        exchange.response.statusCode = status
        exchange.response.headers.contentType = MediaType.APPLICATION_JSON

        val body = objectMapper.writeValueAsBytes(response)
        val buffer = exchange.response.bufferFactory().wrap(body)
        return exchange.response.writeWith(Mono.just(buffer))
    }

    private fun mapException(ex: Throwable): Pair<HttpStatus, ErrorResponse> = when (ex) {
        is OrderNotFoundException -> HttpStatus.NOT_FOUND to ErrorResponse(
            code = ex.code, message = ex.message,
        )
        is InsufficientStockException -> HttpStatus.CONFLICT to ErrorResponse(
            code = ex.code, message = ex.message,
        )
        is InvalidOrderStateException -> HttpStatus.UNPROCESSABLE_ENTITY to ErrorResponse(
            code = ex.code, message = ex.message,
        )
        is IllegalArgumentException -> HttpStatus.BAD_REQUEST to ErrorResponse(
            code = "INVALID_REQUEST", message = ex.message ?: "잘못된 요청",
        )
        is ServerWebInputException -> HttpStatus.BAD_REQUEST to ErrorResponse(
            code = "INVALID_INPUT", message = "요청 본문을 파싱할 수 없습니다",
        )
        else -> {
            log.error("예상치 못한 에러", ex)
            HttpStatus.INTERNAL_SERVER_ERROR to ErrorResponse(
                code = "INTERNAL_ERROR", message = "서버 내부 오류가 발생했습니다",
            )
        }
    }

    companion object {
        private val log = LoggerFactory.getLogger(GlobalErrorHandler::class.java)
    }
}
```

> 글로벌 에러 핸들러는 Spring WebFlux의 `ErrorWebExceptionHandler`로 구현한다.
> 이 인터페이스는 `Mono` 기반이므로 여기서만 `Mono`를 사용한다.
> 애플리케이션 코드에서는 코루틴을 사용한다.

### ErrorResponse DTO

```kotlin
data class ErrorResponse(
    val code: String,
    val message: String,
    val timestamp: String = LocalDateTime.now().toString(),
)
```

---

## 코루틴 에러 처리 패턴

### null 체크 + throw (빈 결과 → 에러)

```kotlin
val order = orderRepository.findById(orderId)
    ?: throw OrderNotFoundException(OrderId(orderId))
```

### try-catch (에러 복구)

```kotlin
suspend fun processPayment(order: Order): PaymentResult =
    try {
        paymentPort.processPayment(order.id.value, order.totalAmount)
    } catch (e: TimeoutException) {
        log.warn("결제 서비스 타임아웃, 재시도")
        retryPayment(order)
    }
```

### withTimeout (시간 초과)

```kotlin
suspend fun callExternalService(): Result =
    try {
        withTimeout(5000) {
            externalService.call()
        }
    } catch (e: TimeoutCancellationException) {
        throw ExternalServiceTimeoutException()
    }
```

### 재시도 패턴

```kotlin
suspend fun <T> retry(
    times: Int = 3,
    initialDelay: Long = 500,
    factor: Double = 2.0,
    block: suspend () -> T,
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            log.warn("재시도 ${attempt + 1}/$times", e)
            delay(currentDelay)
            currentDelay = (currentDelay * factor).toLong()
        }
    }
    return block()  // 마지막 시도 (실패 시 예외 전파)
}

// 사용
val result = retry(times = 3) {
    externalApiClient.call()
}
```

---

## Handler 레벨 에러 처리

글로벌 핸들러 외에 Handler에서 세밀한 에러 처리가 필요한 경우:

```kotlin
suspend fun create(request: ServerRequest): ServerResponse =
    try {
        val body = request.awaitBody<CreateOrderRequest>()
        val command = body.toCommand()
        val order = createOrderUseCase.execute(command)
        ServerResponse.status(HttpStatus.CREATED)
            .bodyValueAndAwait(order.toResponse())
    } catch (e: DomainException) {
        val status = when (e) {
            is OrderNotFoundException -> HttpStatus.NOT_FOUND
            is InsufficientStockException -> HttpStatus.CONFLICT
            else -> HttpStatus.UNPROCESSABLE_ENTITY
        }
        ServerResponse.status(status)
            .bodyValueAndAwait(ErrorResponse(e.code, e.message))
    }
```

---

## Flow 에러 처리

스트리밍(SSE/WebSocket)의 에러는 `catch`로 처리한다.

```kotlin
// Flow에서 catch로 에러 처리
val eventFlow = notificationStream
    .catch { e ->
        log.error("스트림 에러", e)
        emit(Notification.error(e.message))
    }

// Flow retry
val resilientFlow = dataFlow
    .retry(3) { it is IOException }

// WebSocket 메시지 처리 에러 격리
session.receive().asFlow()
    .map { parseMessage(it) }
    .onEach { message ->
        try {
            chatService.handleMessage(userId, message)
        } catch (e: Exception) {
            chatService.sendError(userId, e.message ?: "처리 실패")
        }
    }
```

---

## DO / DON'T

```kotlin
// DO -- 코루틴 에러 처리
val order = repository.findById(id)
    ?: throw NotFoundException()

// DON'T -- Reactor 스타일 에러 처리를 코루틴에서
repository.findById(id)
    .switchIfEmpty(Mono.error(NotFoundException()))  // 금지: 코루틴에서는 null 체크

// DO -- 도메인 예외 사용
throw OrderNotFoundException(orderId)

// DON'T -- 일반 예외에 HTTP 코드 혼합
throw ResponseStatusException(HttpStatus.NOT_FOUND)  // 금지: 도메인에서 HTTP 인지

// DO -- 글로벌 핸들러에서 HTTP 매핑
mapException(ex) -> Pair(HttpStatus, ErrorResponse)

// DON'T -- 에러 무시
try { riskyCall() } catch (_: Exception) { }  // 금지: 에러 삼킴
```

---

## 변경 시 체크리스트

- [ ] 도메인 예외가 HTTP 상태 코드를 모르는가?
- [ ] 글로벌 에러 핸들러에 새 예외 매핑이 등록되었는가?
- [ ] suspend 함수에서 try-catch / null 체크로 에러를 처리하는가?
- [ ] 에러를 삼키지 않는가? (로깅 또는 적절한 처리)
- [ ] Flow에서 `catch` 연산자로 스트림 에러를 처리하는가?
- [ ] 외부 API 호출에 timeout과 retry가 설정되어 있는가?
