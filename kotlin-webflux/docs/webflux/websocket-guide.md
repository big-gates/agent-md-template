# WebSocket Guide

> **트리거**: 양방향 실시간 통신 기능을 구현할 때 이 문서를 참조한다.
> 클라이언트 ↔ 서버 양방향 메시지 교환이 필요한 경우 사용한다.

---

## WebSocket Handler

```kotlin
// adapter/in/websocket/handler/ChatWebSocketHandler.kt
@Component
class ChatWebSocketHandler(
    private val chatService: ChatService,
    private val objectMapper: ObjectMapper,
) : WebSocketHandler {

    override fun handle(session: WebSocketSession): Mono<Void> {
        val userId = extractUserId(session)

        // 인바운드: 클라이언트 → 서버 (코루틴으로 처리)
        val inbound = session.receive()
            .asFlow()
            .map { it.payloadAsText }
            .map { objectMapper.readValue(it, ChatMessage::class.java) }
            .onEach { message -> chatService.handleMessage(userId, message) }
            .catch { e -> chatService.sendError(userId, e.message ?: "처리 실패") }
            .onCompletion { chatService.disconnectUser(userId) }
            .asFlux()
            .then()

        // 아웃바운드: 서버 → 클라이언트
        val outbound = session.send(
            chatService.subscribe(userId)
                .map { objectMapper.writeValueAsString(it) }
                .map { session.textMessage(it) }
                .asFlux()
        )

        return Mono.zip(inbound, outbound).then()
    }

    private fun extractUserId(session: WebSocketSession): String =
        session.handshakeInfo.uri.query
            ?.split("&")
            ?.associate { it.split("=").let { (k, v) -> k to v } }
            ?.get("userId")
            ?: throw IllegalArgumentException("userId 파라미터 필수")
}
```

---

## WebSocket 설정

```kotlin
// config/WebSocketConfig.kt
@Configuration
class WebSocketConfig {

    @Bean
    fun webSocketHandlerMapping(chatHandler: ChatWebSocketHandler): HandlerMapping {
        val map = mapOf("/ws/chat" to chatHandler)
        return SimpleUrlHandlerMapping(map, -1)
    }

    @Bean
    fun webSocketHandlerAdapter(): WebSocketHandlerAdapter =
        WebSocketHandlerAdapter()
}
```

---

## 메시지 프로토콜

### 메시지 타입 정의

```kotlin
// adapter/in/websocket/message/ChatMessage.kt
data class ChatMessage(
    val type: MessageType,
    val roomId: String,
    val content: String? = null,
    val timestamp: Long = System.currentTimeMillis(),
)

enum class MessageType {
    JOIN, LEAVE, MESSAGE, TYPING,
}

data class ChatResponse(
    val type: ResponseType,
    val roomId: String,
    val senderId: String,
    val content: String? = null,
    val timestamp: Long,
)

enum class ResponseType {
    MESSAGE, USER_JOINED, USER_LEFT, TYPING_INDICATOR, ERROR,
}
```

---

## 세션 관리 (SharedFlow 기반)

```kotlin
// application/service/ChatService.kt
@Service
class ChatService(
    private val messageRepository: MessageRepository,
) {
    private val sessions = ConcurrentHashMap<String, MutableSharedFlow<ChatResponse>>()
    private val roomMembers = ConcurrentHashMap<String, MutableSet<String>>()

    suspend fun handleMessage(userId: String, message: ChatMessage) {
        when (message.type) {
            MessageType.JOIN -> joinRoom(userId, message.roomId)
            MessageType.LEAVE -> leaveRoom(userId, message.roomId)
            MessageType.MESSAGE -> sendMessage(userId, message)
            MessageType.TYPING -> notifyTyping(userId, message.roomId)
        }
    }

    fun subscribe(userId: String): Flow<ChatResponse> {
        val flow = MutableSharedFlow<ChatResponse>(extraBufferCapacity = 64)
        sessions[userId] = flow
        return flow.asSharedFlow()
            .onCompletion { sessions.remove(userId) }
    }

    private suspend fun joinRoom(userId: String, roomId: String) {
        roomMembers.getOrPut(roomId) { ConcurrentHashMap.newKeySet() }.add(userId)
        broadcastToRoom(roomId, ChatResponse(
            type = ResponseType.USER_JOINED,
            roomId = roomId,
            senderId = userId,
            timestamp = System.currentTimeMillis(),
        ))
    }

    private suspend fun sendMessage(userId: String, message: ChatMessage) {
        messageRepository.save(message.toDomain(userId))
        broadcastToRoom(message.roomId, ChatResponse(
            type = ResponseType.MESSAGE,
            roomId = message.roomId,
            senderId = userId,
            content = message.content,
            timestamp = System.currentTimeMillis(),
        ))
    }

    private suspend fun broadcastToRoom(roomId: String, response: ChatResponse) {
        val members = roomMembers[roomId] ?: return
        members.forEach { userId ->
            sessions[userId]?.emit(response)
        }
    }

    private suspend fun leaveRoom(userId: String, roomId: String) {
        roomMembers[roomId]?.remove(userId)
        broadcastToRoom(roomId, ChatResponse(
            type = ResponseType.USER_LEFT,
            roomId = roomId,
            senderId = userId,
            timestamp = System.currentTimeMillis(),
        ))
    }

    private suspend fun notifyTyping(userId: String, roomId: String) =
        broadcastToRoom(roomId, ChatResponse(
            type = ResponseType.TYPING_INDICATOR,
            roomId = roomId,
            senderId = userId,
            timestamp = System.currentTimeMillis(),
        ))

    suspend fun sendError(userId: String, message: String) {
        sessions[userId]?.emit(ChatResponse(
            type = ResponseType.ERROR,
            roomId = "",
            senderId = "system",
            content = message,
            timestamp = System.currentTimeMillis(),
        ))
    }

    fun disconnectUser(userId: String) {
        sessions.remove(userId)
        roomMembers.values.forEach { it.remove(userId) }
    }
}
```

---

## 분산 환경 (Redis Pub/Sub 연동)

```kotlin
// adapter/out/messaging/RedisWebSocketBridge.kt
@Component
class RedisWebSocketBridge(
    private val reactiveRedisTemplate: ReactiveRedisTemplate<String, String>,
    private val objectMapper: ObjectMapper,
) {
    suspend fun publish(channel: String, message: ChatResponse) {
        reactiveRedisTemplate.convertAndSend(channel, objectMapper.writeValueAsString(message))
            .awaitSingle()
    }

    fun subscribe(channel: String): Flow<ChatResponse> =
        reactiveRedisTemplate.listenToChannel(channel)
            .asFlow()
            .map { objectMapper.readValue(it.message, ChatResponse::class.java) }
}
```

---

## 연결 관리 & 복원력

| 규칙 | 설명 |
|------|------|
| 연결 해제 처리 | `onCompletion`에서 세션 정리 |
| 에러 격리 | `catch`로 메시지 처리 에러가 전체 연결을 끊지 않도록 |
| 인증 | 핸드셰이크 시 토큰/세션 검증 |
| Ping/Pong | 연결 유지 확인 (WebFlux 기본 지원) |

---

## 변경 시 체크리스트

- [ ] 인바운드/아웃바운드 스트림이 모두 처리되는가?
- [ ] 세션 연결 해제 시 `onCompletion`으로 정리하는가?
- [ ] 메시지 파싱 에러가 `catch`로 처리되어 연결을 끊지 않는가?
- [ ] 인증/인가가 핸드셰이크 단계에서 처리되는가?
- [ ] 분산 환경에서 Redis Pub/Sub 등으로 브로드캐스트하는가?
- [ ] 메시지 프로토콜이 명확히 정의되어 있는가?
