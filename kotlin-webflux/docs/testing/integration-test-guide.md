# Integration Test Guide

> **트리거**: 통합 테스트, WebTestClient, Testcontainers, Repository Adapter 테스트를 작성할 때 이 문서를 참조한다.

---

## 통합 테스트 대상

| 대상 | 검증 내용 | 도구 |
|------|----------|------|
| Handler + Router | HTTP 요청/응답, 상태 코드, JSON 직렬화 | `WebTestClient` |
| Repository Adapter | DB CRUD, 쿼리 정확성, 매핑 | `@DataR2dbcTest` + Testcontainers |
| SSE Endpoint | 스트리밍 응답, 이벤트 형식 | `WebTestClient` |
| WebSocket | 양방향 메시지 교환 | `WebSocketClient` |
| 외부 API 연동 | WebClient 호출, 응답 매핑 | `MockWebServer` (WireMock) |

---

## Handler 통합 테스트

```kotlin
@SpringBootTest
@AutoConfigureWebTestClient
class OrderHandlerIntegrationTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @MockBean
    private lateinit var createOrderUseCase: CreateOrderUseCase

    @Test
    fun `POST 주문 생성 - 201 응답`() = runTest {
        val order = OrderFixture.draftOrder()
        coEvery { createOrderUseCase.execute(any()) } returns order

        webTestClient.post()
            .uri("/api/v1/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""
                {
                    "customerId": "c1",
                    "items": [{"productId": "p1", "quantity": 2}]
                }
            """.trimIndent())
            .exchange()
            .expectStatus().isCreated
            .expectBody()
            .jsonPath("$.id").isNotEmpty
            .jsonPath("$.customerId").isEqualTo("c1")
            .jsonPath("$.status").isEqualTo("DRAFT")
    }

    @Test
    fun `GET 존재하지 않는 주문 - 404 응답`() = runTest {
        coEvery { getOrderUseCase.execute(any()) } returns null

        webTestClient.get()
            .uri("/api/v1/orders/nonexistent")
            .exchange()
            .expectStatus().isNotFound
    }

    @Test
    fun `POST 잘못된 요청 본문 - 400 응답`() {
        webTestClient.post()
            .uri("/api/v1/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"customerId": ""}""")
            .exchange()
            .expectStatus().isBadRequest
            .expectBody()
            .jsonPath("$.code").isEqualTo("INVALID_REQUEST")
    }
}
```

---

## Repository Adapter 통합 테스트 (Testcontainers)

```kotlin
@DataR2dbcTest
@Testcontainers
@Import(OrderPersistenceAdapter::class, OrderEntityMapper::class)
class OrderPersistenceAdapterTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:15-alpine")
            .withDatabaseName("testdb")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.r2dbc.url") {
                "r2dbc:postgresql://${postgres.host}:${postgres.firstMappedPort}/${postgres.databaseName}"
            }
            registry.add("spring.r2dbc.username") { postgres.username }
            registry.add("spring.r2dbc.password") { postgres.password }
        }
    }

    @Autowired
    private lateinit var adapter: OrderPersistenceAdapter

    @Test
    fun `주문 저장 후 조회`() = runTest {
        val order = OrderFixture.draftOrderWithItems()

        val saved = adapter.save(order)
        val found = adapter.findById(saved.id.value)

        assertNotNull(found)
        assertEquals(order.customerId, found!!.customerId)
        assertEquals(order.items.size, found.items.size)
        assertEquals(order.totalAmount, found.totalAmount)
    }

    @Test
    fun `고객 ID로 주문 목록 조회`() = runTest {
        adapter.save(OrderFixture.draftOrder(customerId = "c1"))
        adapter.save(OrderFixture.draftOrder(customerId = "c1"))
        adapter.save(OrderFixture.draftOrder(customerId = "c2"))

        val orders = adapter.findByCustomerId("c1").toList()

        assertEquals(2, orders.size)
    }
}
```

---

## SSE 통합 테스트

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class NotificationSseTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @Autowired
    private lateinit var eventBroadcaster: EventBroadcaster

    @Test
    fun `SSE 스트림에서 실시간 알림 수신`() = runTest {
        // 비동기로 이벤트 발행
        launch {
            delay(100)
            eventBroadcaster.publish(NotificationEvent(
                userId = UserId("u1"),
                message = "새 주문이 접수되었습니다",
            ))
        }

        webTestClient.get()
            .uri("/api/v1/sse/notifications/u1")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk
            .returnResult(ServerSentEvent::class.java)
            .responseBody
            .take(1)
            .doOnNext { event ->
                assertEquals("notification", event.event())
            }
            .blockFirst(Duration.ofSeconds(5))
    }
}
```

---

## 외부 API 연동 테스트 (MockWebServer)

```kotlin
class PaymentApiClientTest {

    private lateinit var mockWebServer: MockWebServer
    private lateinit var paymentClient: PaymentApiClient

    @BeforeEach
    fun setup() {
        mockWebServer = MockWebServer()
        mockWebServer.start()
        val webClient = WebClient.builder()
            .baseUrl(mockWebServer.url("/").toString())
            .build()
        paymentClient = PaymentApiClient(webClient)
    }

    @AfterEach
    fun teardown() {
        mockWebServer.shutdown()
    }

    @Test
    fun `결제 처리 성공`() = runTest {
        mockWebServer.enqueue(MockResponse()
            .setBody("""{"transactionId":"tx1","status":"APPROVED"}""")
            .addHeader("Content-Type", "application/json"))

        val result = paymentClient.processPayment("order1", Money(50000))

        assertEquals("tx1", result.transactionId)
        assertEquals(PaymentStatus.APPROVED, result.status)
    }

    @Test
    fun `결제 서비스 500 에러 시 예외`() = runTest {
        mockWebServer.enqueue(MockResponse().setResponseCode(500))

        assertThrows<ExternalServiceException> {
            paymentClient.processPayment("order1", Money(50000))
        }
    }
}
```

---

## 테스트 설정

### 테스트 프로파일

```yaml
# src/test/resources/application-test.yml
spring:
  r2dbc:
    url: r2dbc:h2:mem:///testdb
    username: sa
    password:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

### 공통 테스트 베이스 클래스

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
abstract class IntegrationTestBase {

    @Autowired
    protected lateinit var webTestClient: WebTestClient
}
```

---

## 테스트 격리

| 규칙 | 설명 |
|------|------|
| DB 초기화 | 각 테스트 전에 데이터 정리 (`@BeforeEach`에서 delete) |
| 포트 랜덤 | `RANDOM_PORT`로 포트 충돌 방지 |
| Testcontainers | 실제 DB와 동일한 환경에서 테스트 |
| MockWebServer | 외부 API를 격리하여 테스트 |

---

## 변경 시 체크리스트

- [ ] 새 Handler에 대응하는 통합 테스트가 있는가?
- [ ] Repository Adapter 테스트가 Testcontainers를 사용하는가?
- [ ] 외부 API 연동이 MockWebServer로 격리되어 있는가?
- [ ] SSE/WebSocket 테스트가 실시간 이벤트를 검증하는가?
- [ ] 테스트 간 데이터 격리가 보장되는가?
- [ ] suspend 함수 테스트에 `runTest`를 사용하는가?
