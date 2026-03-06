# Coroutine callbackFlow Guide

> 콜백/리스너 기반 API를 Flow로 변환하는 패턴, SSE(Server-Sent Events) 처리 규칙이다.

---

## callbackFlow란

콜백/리스너 기반 비동기 API를 **구조적 동시성을 유지하면서** Flow로 변환하는 빌더이다.

| 단계 | 흐름 | 설명 |
|------|------|------|
| 1 | 콜백 기반 API → `callbackFlow { }` | 콜백을 등록하고 `trySend`로 값 전달 |
| 2 | `callbackFlow { }` → `Flow<T>` | 구조적 동시성을 유지하는 Flow로 변환 |
| 3 | `Flow<T>` → `collect` | 수집자의 스코프에 연결 |
| 4 | collect 종료 시 → `awaitClose { }` | 리소스 정리 보장 (리스너 해제, 연결 닫기) |

---

## 규칙 1: 기본 패턴

```kotlin
fun observeLocationUpdates(): Flow<Location> = callbackFlow {
    // 1. 콜백 등록
    val callback = object : LocationCallback {
        override fun onLocationUpdate(location: Location) {
            trySend(location)   // Channel로 전송
        }
        override fun onError(error: Exception) {
            close(error)        // Flow 종료 + 에러 전파
        }
    }
    locationClient.registerCallback(callback)

    // 2. awaitClose — collect 종료 시 정리 (필수)
    awaitClose {
        locationClient.unregisterCallback(callback)
    }
}
```

| 요소 | 역할 |
|------|------|
| `trySend(value)` | 콜백에서 값을 Flow로 전송 (비-suspend, non-blocking) |
| `send(value)` | suspend 버전, 백프레셔 적용 (콜백 내에서는 `trySend` 사용) |
| `close()` | Flow를 정상 종료 |
| `close(cause)` | Flow를 에러와 함께 종료 |
| `awaitClose { }` | **필수** — collect 종료/취소 시 리소스 정리 |

---

## 규칙 2: awaitClose는 반드시 포함한다

`awaitClose`가 없으면 callbackFlow가 즉시 종료되어 값을 수신할 수 없다.

```kotlin
// DO — awaitClose로 리소스 정리
fun observeAuth(): Flow<AuthState> = callbackFlow {
    val listener = AuthListener { state -> trySend(state) }
    authManager.addListener(listener)
    awaitClose { authManager.removeListener(listener) }
}

// DON'T — awaitClose 누락
fun observeAuth(): Flow<AuthState> = callbackFlow {
    val listener = AuthListener { state -> trySend(state) }
    authManager.addListener(listener)
    // awaitClose 없음 → 즉시 종료, listener 누수
}
```

---

## 규칙 3: SSE (Server-Sent Events) 패턴

```kotlin
fun observeSSE(url: String): Flow<ServerEvent> = callbackFlow {
    val client = httpClient.newCall(Request.Builder().url(url).build())

    val eventSource = EventSource.Factory(client).create(
        request,
        object : EventSourceListener() {
            override fun onEvent(id: String?, type: String?, data: String) {
                val event = parseEvent(data)
                trySend(event)
            }
            override fun onClosed() {
                close()   // 서버가 연결을 닫음
            }
            override fun onFailure(t: Throwable, response: Response?) {
                close(t)  // 에러와 함께 종료
            }
        }
    )

    awaitClose {
        eventSource.cancel()   // collect 종료 시 연결 해제
    }
}
```

### SSE를 ViewModel에서 사용

```kotlin
class ChatViewModel(private val chatRepo: ChatRepository) : ViewModel() {
    private val _messages = MutableStateFlow<List<Message>>(emptyList())
    val messages: StateFlow<List<Message>> = _messages.asStateFlow()

    fun startListening(roomId: String) {
        viewModelScope.launch {
            chatRepo.observeMessages(roomId)
                .catch { e ->
                    if (e is CancellationException) throw e
                    _uiState.value = UiState.Error(e.message)
                }
                .collect { message ->
                    _messages.value = _messages.value + message
                }
        }
        // viewModelScope 취소 → collect 종료 → awaitClose → 연결 해제
    }
}
```

---

## 규칙 4: 재연결 패턴

SSE/WebSocket 등 장기 연결이 끊겼을 때 재연결한다.

```kotlin
fun observeWithRetry(roomId: String): Flow<Message> =
    chatRepo.observeMessages(roomId)
        .retryWhen { cause, attempt ->
            if (cause is CancellationException) return@retryWhen false
            val delayMs = minOf(1000L * (1 shl attempt.toInt()), 30_000L)
            delay(delayMs)   // 지수 백오프 (최대 30초)
            true             // 재시도
        }
```

| 파라미터 | 설명 |
|---------|------|
| `cause` | 발생한 예외 |
| `attempt` | 재시도 횟수 (0부터) |
| 반환 `true` | 재시도 |
| 반환 `false` | 재시도 중단, 예외 전파 |

**주의**: `CancellationException`에서는 반드시 `false`를 반환하여 취소 전파를 유지한다.

---

## 규칙 5: trySend vs send

| | `trySend` | `send` |
|--|----------|--------|
| suspend | 아니오 | 예 |
| 버퍼 가득 시 | `ChannelResult.failure` 반환 | suspend (대기) |
| 콜백 내부 사용 | **권장** | 사용 불가 (non-suspend 컨텍스트) |
| launch 내부 사용 | 가능 | **권장** (백프레셔 적용) |

```kotlin
// 콜백 내부 — trySend
val callback = Callback { data ->
    trySend(data)   // non-suspend 환경에서 사용
}

// launch 내부 — send
callbackFlow {
    launch {
        while (isActive) {
            val data = pollData()
            send(data)   // 버퍼 가득 시 대기 (백프레셔)
        }
    }
    awaitClose { }
}
```

---

## 에이전트 체크리스트

- [ ] callbackFlow에 `awaitClose { }` 블록이 있는가?
- [ ] `awaitClose`에서 리스너/콜백/연결을 정리하는가?
- [ ] 콜백 내부에서 `trySend`를 사용하는가? (`send` 아님)
- [ ] SSE/WebSocket에서 `close(cause)`로 에러를 전파하는가?
- [ ] `retryWhen`에서 `CancellationException` 시 `false`를 반환하는가?
- [ ] ViewModel에서 `collect` → `viewModelScope` 취소 → `awaitClose` 흐름이 보장되는가?
