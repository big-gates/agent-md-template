# SSE (Server-Sent Events) Guide

> **트리거**: 실시간 단방향 스트리밍 기능을 구현할 때 이 문서를 참조한다.
> 서버 → 클라이언트 단방향 실시간 데이터 푸시에 사용한다.

---

## SSE vs WebSocket

| 기준 | SSE | WebSocket |
|------|-----|-----------|
| 방향 | 서버 → 클라이언트 (단방향) | 양방향 |
| 프로토콜 | HTTP/1.1, HTTP/2 | WS/WSS |
| 자동 재연결 | 브라우저 기본 지원 | 직접 구현 필요 |
| 데이터 형식 | 텍스트 (UTF-8) | 텍스트 + 바이너리 |
| 적합한 경우 | 알림, 실시간 피드, 대시보드 갱신 | 채팅, 게임, 양방향 통신 |

---

## 기본 SSE Handler

SSE 응답은 `Flow`를 `Flux`로 변환하여 전송한다 (Spring SSE가 Flux 기반).

```kotlin
// adapter/in/sse/handler/NotificationSseHandler.kt
@Component
class NotificationSseHandler(
    private val notificationUseCase: GetNotificationStreamUseCase,
) {
    suspend fun stream(request: ServerRequest): ServerResponse {
        val userId = request.pathVariable("userId")
        val sseFlux = notificationUseCase.execute(NotificationStreamQuery(userId))
            .map { notification ->
                ServerSentEvent.builder<NotificationResponse>()
                    .id(notification.id.value)
                    .event(notification.type.name.lowercase())
                    .data(notification.toResponse())
                    .build()
            }
            .asFlux()  // Flow -> Flux 변환 (SSE에서 필요)

        return ServerResponse.ok()
            .contentType(MediaType.TEXT_EVENT_STREAM)
            .bodyAndAwait(sseFlux)
    }
}
```

### Router 설정

```kotlin
// adapter/in/web/router/SseRouter.kt
@Configuration
class SseRouter {
    @Bean
    fun sseRoutes(handler: NotificationSseHandler) = coRouter {
        "/api/v1/sse".nest {
            GET("/notifications/{userId}", handler::stream)
        }
    }
}
```

---

## SharedFlow 기반 이벤트 발행

여러 구독자에게 실시간으로 이벤트를 브로드캐스트하는 패턴이다.

```kotlin
// application/service/EventBroadcaster.kt
@Component
class EventBroadcaster {
    private val _events = MutableSharedFlow<DomainEvent>(
        extraBufferCapacity = 256,
    )

    suspend fun publish(event: DomainEvent) {
        _events.emit(event)
    }

    fun subscribe(): Flow<DomainEvent> = _events.asSharedFlow()

    fun subscribe(eventType: String): Flow<DomainEvent> =
        _events.asSharedFlow().filter { it.type == eventType }
}
```

### UseCase에서 SharedFlow 활용

```kotlin
// application/service/NotificationService.kt
@Service
class NotificationService(
    private val eventBroadcaster: EventBroadcaster,
    private val notificationRepository: NotificationRepository,
) : GetNotificationStreamUseCase {

    override fun execute(query: NotificationStreamQuery): Flow<Notification> {
        val historical = notificationRepository
            .findByUserId(query.userId)
            .take(20)

        val live = eventBroadcaster.subscribe()
            .filterIsInstance<NotificationEvent>()
            .filter { it.userId.value == query.userId }
            .map { it.toNotification() }

        return merge(historical, live)
    }
}
```

---

## Heartbeat (연결 유지)

SSE 연결이 프록시/로드밸런서에 의해 끊기지 않도록 주기적 heartbeat를 전송한다.

```kotlin
suspend fun streamWithHeartbeat(request: ServerRequest): ServerResponse {
    val userId = request.pathVariable("userId")

    val dataStream = notificationUseCase.execute(NotificationStreamQuery(userId))
        .map { notification ->
            ServerSentEvent.builder<String>()
                .event("notification")
                .data(objectMapper.writeValueAsString(notification.toResponse()))
                .build()
        }

    val heartbeat = flow {
        while (true) {
            emit(
                ServerSentEvent.builder<String>()
                    .event("heartbeat")
                    .data("")
                    .build()
            )
            delay(30_000)  // 30초마다
        }
    }

    val merged = merge(dataStream, heartbeat).asFlux()

    return ServerResponse.ok()
        .contentType(MediaType.TEXT_EVENT_STREAM)
        .bodyAndAwait(merged)
}
```

---

## 실시간 대시보드 패턴

```kotlin
// 주기적 폴링 + 이벤트 기반 갱신 혼합
@Service
class DashboardService(
    private val statsRepository: StatsRepository,
    private val eventBroadcaster: EventBroadcaster,
) : GetDashboardStreamUseCase {

    override fun execute(): Flow<Dashboard> {
        val periodic = flow {
            while (true) {
                emit(statsRepository.getDashboard())
                delay(5000)  // 5초마다
            }
        }

        val eventDriven = eventBroadcaster.subscribe("order")
            .map { statsRepository.getDashboard() }

        return merge(periodic, eventDriven)
            .distinctUntilChanged()  // 변경된 경우에만 전송
    }
}
```

---

## SharedFlow 전략

| SharedFlow 설정 | 설명 | 용도 |
|----------------|------|------|
| `MutableSharedFlow(replay = 0, extraBufferCapacity = 256)` | 여러 구독자, 버퍼링 | 알림 브로드캐스트 |
| `MutableSharedFlow(replay = 10)` | 새 구독자에게 최근 10개 재전송 | 최근 이벤트 캐시 |
| `MutableSharedFlow(onBufferOverflow = DROP_OLDEST)` | 느린 구독자 → 오래된 것 버림 | 실시간 메트릭 |

---

## 에러 처리 & 복원력

```kotlin
suspend fun stream(request: ServerRequest): ServerResponse {
    val eventStream = notificationUseCase.execute(query)
        .catch { e ->
            log.error("SSE 스트림 에러", e)
            emit(Notification.error(e.message))
        }
        .map { it.toSseEvent() }
        .onCompletion { log.info("SSE 스트림 종료") }
        .asFlux()

    return ServerResponse.ok()
        .contentType(MediaType.TEXT_EVENT_STREAM)
        .bodyAndAwait(eventStream)
}
```

---

## 변경 시 체크리스트

- [ ] `Content-Type`이 `text/event-stream`인가?
- [ ] Heartbeat가 설정되어 있는가? (프록시 타임아웃 방지)
- [ ] Flow를 Flux로 변환(`asFlux()`)하여 SSE 응답에 사용하는가?
- [ ] `onCompletion`으로 스트림 종료 처리가 있는가?
- [ ] 에러 발생 시 `catch`로 적절히 처리되는가?
- [ ] SharedFlow 설정(replay, buffer)이 요구사항에 맞는가?
