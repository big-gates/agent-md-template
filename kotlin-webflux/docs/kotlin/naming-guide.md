# Naming Guide

> **트리거**: 패키지, 클래스, 함수, 변수 네이밍 규칙을 확인할 때 이 문서를 참조한다.

---

## 패키지

| 패키지 | 설명 |
|--------|------|
| `com.example.project` | 기본 패키지 (도트 소문자) |
| `com.example.project.adapter.in.web` | 계층별 분리 |
| `com.example.project.domain.model` | 도메인 모델 |

| 규칙 | 설명 |
|------|------|
| 소문자만 | 언더스코어, 대문자 금지 |
| 의미 단위 | 기능/레이어별 분리 |

---

## 클래스 / 인터페이스

| 구분 | 패턴 | 예시 |
|------|------|------|
| Domain Model | `{명사}` | `Order`, `Money`, `Quantity` |
| Aggregate Root | `{명사}` | `Order` (items를 포함하는 루트) |
| Value Object | `{명사}` | `Money`, `OrderId`, `CustomerId` |
| Domain Event | `{도메인}{과거분사}Event` | `OrderCreatedEvent`, `OrderConfirmedEvent` |
| Domain Exception | `{설명}Exception` | `OrderNotFoundException`, `InsufficientStockException` |
| Domain Service | `{도메인}DomainService` | `PricingDomainService` |
| UseCase (Port) | `{동사}{도메인}UseCase` | `CreateOrderUseCase`, `GetOrderUseCase` |
| Command | `{동사}{도메인}Command` | `CreateOrderCommand`, `ConfirmOrderCommand` |
| Query | `{동사}{도메인}Query` | `GetOrderQuery`, `GetOrdersStreamQuery` |
| Application Service | `{도메인}Service` | `OrderService` |
| Handler | `{도메인}Handler` | `OrderHandler` |
| Router | `{도메인}Router` | `OrderRouter` |
| Repository Port | `{도메인}Repository` | `OrderRepository` |
| Persistence Adapter | `{도메인}PersistenceAdapter` | `OrderPersistenceAdapter` |
| DB Entity | `{도메인}Entity` | `OrderEntity` |
| Request DTO | `{동사}{도메인}Request` | `CreateOrderRequest` |
| Response DTO | `{도메인}Response` | `OrderResponse` |
| Mapper | `{도메인}{계층}Mapper` | `OrderDtoMapper`, `OrderEntityMapper` |
| Fake (테스트) | `Fake{인터페이스}` | `FakeOrderRepository` |
| Fixture (테스트) | `{도메인}Fixture` | `OrderFixture` |
| Config | `{기능}Config` | `WebFluxConfig`, `R2dbcConfig` |

---

## 함수 / 메서드

| 구분 | 패턴 | 예시 |
|------|------|------|
| 조회 | `find{By조건}`, `get{대상}` | `findById()`, `getOrder()` |
| 존재 확인 | `exists{By조건}` | `existsById()` |
| 생성 | `create`, `add` | `create()`, `addItem()` |
| 수정 | `update`, `change`, `modify` | `updateStatus()`, `changeAddress()` |
| 삭제 | `delete`, `remove` | `deleteById()`, `removeItem()` |
| 변환 | `to{대상}`, `from{소스}` | `toResponse()`, `toEntity()`, `toDomain()` |
| 스트림 | `stream{대상}`, `observe{대상}` | `streamOrders()`, `observeNotifications()` |
| 검증 | `validate`, `check`, `require` | `validate()`, `checkStock()` |
| UseCase 실행 | `execute` | `execute(command)` |

### 함수 네이밍 규칙

```kotlin
// DO -- 동사로 시작
suspend fun createOrder(command: CreateOrderCommand): Order
suspend fun findById(id: String): Order?
fun toResponse(): OrderResponse

// DON'T -- 명사로 시작하는 함수명
suspend fun order(command: CreateOrderCommand): Order  // 모호함
```

---

## 변수 / 프로퍼티

| 규칙 | 설명 | 예시 |
|------|------|------|
| camelCase | 소문자 시작, 단어 대문자 | `orderId`, `totalAmount` |
| 의미 명확 | 축약 지양 | `customerId` (o), `custId` (x) |
| Boolean | `is`, `has`, `should` 접두사 | `isActive`, `hasItems`, `shouldRetry` |
| 컬렉션 | 복수형 | `items`, `orders`, `notifications` |

```kotlin
// DO
val orderId: OrderId
val isActive: Boolean
val totalAmount: Money
val items: List<OrderItem>

// DON'T
val oid: String       // 축약 금지
val flag: Boolean     // 의미 불분명
val data: Any         // 타입 불분명
```

---

## 상수

```kotlin
companion object {
    const val MAX_RETRY_COUNT = 3
    const val DEFAULT_PAGE_SIZE = 20
    val DEFAULT_TIMEOUT: Duration = Duration.ofSeconds(5)
}
```

| 규칙 | 설명 |
|------|------|
| SCREAMING_SNAKE_CASE | 컴파일 타임 상수 |
| `const val` | 원시 타입 + String |
| `val` | 객체 타입 상수 |

---

## 테스트 메서드

```kotlin
@Test
fun `주어진 조건_실행할 동작_기대 결과`() { }

// 예시
@Test
fun `DRAFT 상태에서_항목 추가_아이템 목록에 포함`() { }

@Test
fun `존재하지 않는 주문 조회_OrderNotFoundException 발생`() { }

@Test
fun `VIP 등급_할인 계산_10퍼센트 할인 적용`() { }
```

| 규칙 | 설명 |
|------|------|
| 백틱 한글 | `fun \`한글 설명\`()` 형식 |
| Given_When_Then | `조건_동작_결과` 구조 |
| 구체적 | "성공" → "주문 생성 후 DRAFT 상태 반환" |

---

## 변경 시 체크리스트

- [ ] 네이밍 패턴이 이 가이드를 따르는가?
- [ ] 축약 없이 의미가 명확한가?
- [ ] Boolean 프로퍼티에 `is`/`has`/`should` 접두사가 있는가?
- [ ] 테스트 메서드가 `조건_동작_결과` 형식인가?
- [ ] 상수가 SCREAMING_SNAKE_CASE인가?
