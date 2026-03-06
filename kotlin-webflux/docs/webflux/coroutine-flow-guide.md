# Coroutine & Flow Guide

> **트리거**: suspend/Flow 패턴, 스트림 합성, 동시성, 코루틴 컨텍스트를 확인할 때 이 문서를 참조한다.

---

## 핵심 타입

| 타입 | 설명 | 용도 |
|------|------|------|
| `suspend fun` | 단일 비동기 결과 | 조회, 저장, 삭제, 외부 API 호출 |
| `Flow<T>` | 0..N개 비동기 스트림 | 목록 조회, 실시간 스트리밍, SSE |

---

## suspend 패턴

### 순차 호출

```kotlin
suspend fun processOrder(orderId: String): Order {
    val order = orderRepository.findById(orderId)
        ?: throw OrderNotFoundException(OrderId(orderId))
    val payment = paymentPort.processPayment(order.id.value, order.totalAmount)
    notificationPort.sendConfirmation(order)
    return order
}
```

### 병렬 호출 (coroutineScope + async)

```kotlin
suspend fun getDashboard(customerId: String): Dashboard = coroutineScope {
    val orders = async { orderRepository.countByCustomerId(customerId) }
    val amount = async { orderRepository.sumAmountByCustomerId(customerId) }
    val favorites = async { productRepository.countFavorites(customerId) }
    Dashboard(orders.await(), amount.await(), favorites.await())
}
```

### nullable 처리

```kotlin
// suspend 함수에서 null은 직관적으로 처리
suspend fun getOrder(id: String): Order {
    return orderRepository.findById(id)
        ?: throw OrderNotFoundException(OrderId(id))
}

// 없을 수 있는 경우 nullable 반환
suspend fun findOrder(id: String): Order? =
    orderRepository.findById(id)
```

---

## Flow 패턴

### 생성

```kotlin
// Repository에서 Flow 반환
fun findByCustomerId(customerId: String): Flow<Order>

// 직접 생성
val tickerFlow = flow {
    while (true) {
        emit(fetchLatestData())
        delay(5000)  // 5초마다
    }
}

// 컬렉션 -> Flow
listOf(1, 2, 3).asFlow()

// channelFlow (여러 코루틴에서 emit)
val merged = channelFlow {
    launch { source1.collect { send(it) } }
    launch { source2.collect { send(it) } }
}
```

### 변환

```kotlin
flow.map { it.toResponse() }                    // 동기 변환
flow.filter { it.isActive }                      // 필터링
flow.take(10)                                    // 처음 N개
flow.drop(5)                                     // 처음 N개 건너뛰기
flow.distinctUntilChanged()                      // 연속 중복 제거
flow.debounce(300)                               // 디바운스 (ms)
flow.sample(1000)                                // 샘플링 (ms)
```

### 집계

```kotlin
flow.toList()                                    // List로 수집
flow.first()                                     // 첫 번째 값
flow.firstOrNull()                               // 첫 번째 값 또는 null
flow.count()                                     // 개수
flow.fold(0) { acc, value -> acc + value }       // 축약
flow.reduce { acc, value -> acc + value }        // 축약 (초기값 없음)
```

---

## 스트림 합성

### merge (도착 순서대로)

```kotlin
val merged = merge(
    notificationSource1.stream(),
    notificationSource2.stream(),
)  // 두 소스의 이벤트를 도착 순서대로 합침
```

### combine (최신 값 조합)

```kotlin
val combined = combine(priceFlow, exchangeRateFlow) { price, rate ->
    price * rate
}
```

### zip (1:1 대응)

```kotlin
val zipped = flow1.zip(flow2) { a, b -> Pair(a, b) }
```

### flatMapConcat / flatMapMerge / flatMapLatest

```kotlin
// 순차적 (이전 Flow 완료 후 다음)
flow.flatMapConcat { id -> orderRepository.findById(id).asFlow() }

// 병렬 (동시 실행)
flow.flatMapMerge { id -> orderRepository.findById(id).asFlow() }

// 최신만 (이전 Flow 취소)
searchQueryFlow.flatMapLatest { query -> searchRepository.search(query) }
```

---

## SharedFlow / StateFlow (이벤트 브로드캐스트)

### MutableSharedFlow (이벤트 발행)

```kotlin
@Component
class EventBroadcaster {
    private val _events = MutableSharedFlow<DomainEvent>(
        extraBufferCapacity = 256,     // 버퍼 크기
    )

    suspend fun publish(event: DomainEvent) {
        _events.emit(event)
    }

    fun subscribe(): Flow<DomainEvent> = _events.asSharedFlow()

    fun subscribe(eventType: String): Flow<DomainEvent> =
        _events.asSharedFlow().filter { it.type == eventType }
}
```

### MutableStateFlow (상태 보관)

```kotlin
@Component
class DashboardState {
    private val _state = MutableStateFlow(Dashboard.empty())

    val state: StateFlow<Dashboard> = _state.asStateFlow()

    suspend fun update(dashboard: Dashboard) {
        _state.value = dashboard
    }
}
```

| 타입 | 설명 | 용도 |
|------|------|------|
| `MutableSharedFlow` | 여러 구독자, 값 보관 없음 | 이벤트 브로드캐스트, 알림 |
| `MutableStateFlow` | 여러 구독자, 최신 값 보관 | 상태 공유, 캐시 |

### SharedFlow 설정

```kotlin
// replay: 새 구독자에게 재전송할 이전 값 개수
MutableSharedFlow<Event>(replay = 0)                       // 새 이벤트만
MutableSharedFlow<Event>(replay = 10)                      // 최근 10개 재전송

// extraBufferCapacity: emit 시 버퍼
MutableSharedFlow<Event>(extraBufferCapacity = 256)        // 256개 버퍼

// onBufferOverflow: 버퍼 초과 시 정책
MutableSharedFlow<Event>(
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_OLDEST,         // 오래된 것 버림
)
```

---

## 에러 처리

```kotlin
// catch -- Flow 에러 처리
flow.catch { e ->
    log.error("스트림 에러", e)
    emit(fallbackValue)
}

// retry -- 재시도
flow.retry(3) { e ->
    e is IOException
}

// retryWhen -- 조건부 재시도
flow.retryWhen { cause, attempt ->
    if (attempt < 3 && cause is IOException) {
        delay(1000 * (attempt + 1))  // 지수 백오프
        true
    } else {
        false
    }
}
```

---

## 컨텍스트 & 디스패처

```kotlin
// 디스패처 전환
flow.flowOn(Dispatchers.IO)                     // 상류를 IO 디스패처에서 실행

// 블로킹 호출 감싸기
withContext(Dispatchers.IO) {
    blockingJdbcCall()
}

// 타임아웃
withTimeout(5000) {
    externalService.call()
}

// 타임아웃 (null 반환)
withTimeoutOrNull(5000) {
    externalService.call()
}
```

| 디스패처 | 용도 |
|---------|------|
| `Dispatchers.Default` | CPU 바운드 작업 |
| `Dispatchers.IO` | 블로킹 I/O 감싸기 |
| `Dispatchers.Unconfined` | 테스트용 (사용 주의) |

---

## DO / DON'T

```kotlin
// DO -- suspend로 순차 비동기 호출
suspend fun process(id: String): Order {
    val order = repository.findById(id) ?: throw NotFoundException()
    return repository.save(order)
}

// DON'T -- runBlocking으로 동기 변환
fun process(id: String): Order = runBlocking {  // 금지: 이벤트 루프 차단
    repository.findById(id)!!
}

// DO -- coroutineScope + async로 병렬 호출
suspend fun dashboard() = coroutineScope {
    val a = async { repoA.count() }
    val b = async { repoB.count() }
    Result(a.await(), b.await())
}

// DON'T -- GlobalScope 사용
GlobalScope.launch { ... }  // 금지: 수명 관리 불가

// DO -- Flow에서 catch로 에러 처리
flow.catch { e -> emit(fallback) }

// DON'T -- Flow 안에서 try-catch로 emit 감싸기
flow {
    try { emit(compute()) } catch (e: Exception) { }  // 금지
}

// DO -- flowOn으로 디스패처 전환
flow.flowOn(Dispatchers.IO)

// DON'T -- withContext로 Flow 내부에서 디스패처 전환
flow { withContext(Dispatchers.IO) { emit(value) } }  // 금지: 예외 발생
```

---

## 변경 시 체크리스트

- [ ] 단일 결과에 `suspend`, 스트림에 `Flow`를 사용하는가?
- [ ] `runBlocking` / `GlobalScope` 없이 구조화된 동시성을 사용하는가?
- [ ] 병렬 호출에 `coroutineScope` + `async`를 사용하는가?
- [ ] Flow 에러 처리에 `catch` 연산자를 사용하는가?
- [ ] 블로킹 호출이 `withContext(Dispatchers.IO)`로 감싸져 있는가?
- [ ] `flowOn`으로 디스패처를 전환하는가? (Flow 내부에서 `withContext` 아님)
