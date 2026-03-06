# R2DBC Guide

> **트리거**: R2DBC DB 접근, 쿼리 작성, 트랜잭션 관리를 할 때 이 문서를 참조한다.

---

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **DAO** | `CoroutineCrudRepository`를 상속하여 정의. Spring Data가 구현체 자동 생성 |
| **Persistence Adapter** | `@Repository` 어노테이션. DAO를 감싸서 Outbound Port를 구현 |
| **Entity 격리** | R2DBC Entity는 `adapter/out/persistence/entity/`에만 존재. 도메인에 노출 금지 |
| **매퍼 필수** | Entity <-> Domain 변환은 반드시 매퍼를 통해 |

---

## Entity

```kotlin
// adapter/out/persistence/entity/OrderEntity.kt
@Table("orders")
data class OrderEntity(
    @Id val id: String? = null,
    val customerId: String,
    val status: String,
    val totalAmount: Long,
    @CreatedDate val createdAt: LocalDateTime? = null,
    @LastModifiedDate val updatedAt: LocalDateTime? = null,
)

@Table("order_items")
data class OrderItemEntity(
    @Id val id: Long? = null,
    val orderId: String,
    val productId: String,
    val quantity: Int,
    val unitPrice: Long,
)
```

---

## Spring Data R2DBC Coroutine Repository

```kotlin
// adapter/out/persistence/repository/OrderCoroutineRepository.kt
interface OrderCoroutineRepository : CoroutineCrudRepository<OrderEntity, String> {
    fun findByCustomerId(customerId: String): Flow<OrderEntity>
    fun findByStatus(status: String): Flow<OrderEntity>
    fun findByCustomerIdAndStatus(customerId: String, status: String): Flow<OrderEntity>
    suspend fun countByCustomerId(customerId: String): Long

    @Query("SELECT * FROM orders WHERE created_at > :since ORDER BY created_at DESC")
    fun findRecentOrders(since: LocalDateTime): Flow<OrderEntity>

    @Query("SELECT * FROM orders WHERE status = :status LIMIT :limit OFFSET :offset")
    fun findByStatusPaged(status: String, limit: Int, offset: Int): Flow<OrderEntity>
}
```

---

## Persistence Adapter (Port 구현)

```kotlin
// adapter/out/persistence/repository/OrderPersistenceAdapter.kt
@Repository
class OrderPersistenceAdapter(
    private val orderRepository: OrderCoroutineRepository,
    private val orderItemRepository: OrderItemCoroutineRepository,
    private val mapper: OrderEntityMapper,
) : OrderRepository {

    override suspend fun save(order: Order): Order {
        val entity = mapper.toEntity(order)
        val savedEntity = orderRepository.save(entity)
        val items = order.items.map { mapper.toItemEntity(it, savedEntity.id!!) }
        val savedItems = orderItemRepository.saveAll(items).toList()
        return mapper.toDomain(savedEntity, savedItems)
    }

    override suspend fun findById(id: String): Order? {
        val entity = orderRepository.findById(id) ?: return null
        val items = orderItemRepository.findByOrderId(entity.id!!).toList()
        return mapper.toDomain(entity, items)
    }

    override fun findByCustomerId(customerId: String): Flow<Order> =
        orderRepository.findByCustomerId(customerId)
            .map { entity ->
                val items = orderItemRepository.findByOrderId(entity.id!!).toList()
                mapper.toDomain(entity, items)
            }

    override suspend fun existsById(id: String): Boolean =
        orderRepository.existsById(id)

    override suspend fun deleteById(id: String) {
        orderItemRepository.deleteByOrderId(id)
        orderRepository.deleteById(id)
    }
}
```

---

## Entity Mapper

```kotlin
// adapter/out/persistence/mapper/OrderEntityMapper.kt
@Component
class OrderEntityMapper {

    fun toEntity(order: Order) = OrderEntity(
        id = order.id.value,
        customerId = order.customerId.value,
        status = order.status.name,
        totalAmount = order.totalAmount.amount,
    )

    fun toItemEntity(item: OrderItem, orderId: String) = OrderItemEntity(
        orderId = orderId,
        productId = item.productId.value,
        quantity = item.quantity.value,
        unitPrice = item.unitPrice.amount,
    )

    fun toDomain(entity: OrderEntity, items: List<OrderItemEntity>) = Order.reconstitute(
        id = OrderId(entity.id!!),
        customerId = CustomerId(entity.customerId),
        items = items.map { toDomainItem(it) },
        status = OrderStatus.valueOf(entity.status),
        createdAt = entity.createdAt!!,
    )

    private fun toDomainItem(entity: OrderItemEntity) = OrderItem(
        productId = ProductId(entity.productId),
        quantity = Quantity(entity.quantity),
        unitPrice = Money(entity.unitPrice),
    )
}
```

---

## 트랜잭션

```kotlin
// application/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
) : CreateOrderUseCase {

    // @Transactional은 코루틴 suspend 함수에서 동작
    @Transactional
    override suspend fun execute(command: CreateOrderCommand): Order {
        val order = Order.create(CustomerId(command.customerId))
        // ... 항목 추가
        return orderRepository.save(order)
    }

    // 또는 프로그래매틱 트랜잭션
    override suspend fun execute(command: TransferOrderCommand): Order {
        return transactionalOperator.executeAndAwait {
            val order = orderRepository.findById(command.orderId)
                ?: throw OrderNotFoundException(OrderId(command.orderId))
            order.confirm()
            orderRepository.save(order)
        }
    }
}
```

| 방식 | 사용 시점 |
|------|----------|
| `@Transactional` | 단순 트랜잭션 (suspend 함수에서 동작) |
| `TransactionalOperator.executeAndAwait` | 프로그래매틱 제어가 필요할 때 |

---

## DatabaseClient (커스텀 쿼리)

복잡한 쿼리는 `DatabaseClient`와 코루틴 확장을 사용한다.

```kotlin
@Repository
class OrderCustomRepository(
    private val databaseClient: DatabaseClient,
    private val mapper: OrderEntityMapper,
) {
    suspend fun findWithFilters(
        status: OrderStatus?,
        minAmount: Long?,
        fromDate: LocalDateTime?,
    ): List<Order> {
        var sql = "SELECT * FROM orders WHERE 1=1"
        val binds = mutableMapOf<String, Any>()

        status?.let {
            sql += " AND status = :status"
            binds["status"] = it.name
        }
        minAmount?.let {
            sql += " AND total_amount >= :minAmount"
            binds["minAmount"] = it
        }
        fromDate?.let {
            sql += " AND created_at >= :fromDate"
            binds["fromDate"] = it
        }

        sql += " ORDER BY created_at DESC"

        var spec = databaseClient.sql(sql)
        binds.forEach { (key, value) -> spec = spec.bind(key, value) }

        return spec.map { row, _ ->
            OrderEntity(
                id = row.get("id", String::class.java),
                customerId = row.get("customer_id", String::class.java)!!,
                status = row.get("status", String::class.java)!!,
                totalAmount = row.get("total_amount", java.lang.Long::class.java)!!.toLong(),
                createdAt = row.get("created_at", LocalDateTime::class.java),
            )
        }.flow()                                      // Flow<OrderEntity>
        .map { mapper.toDomain(it, emptyList()) }
        .toList()
    }
}
```

---

## 스키마 관리

Flyway 또는 Liquibase로 스키마를 관리한다.

| 파일 | 위치 | 설명 |
|------|------|------|
| `V1__create_orders_table.sql` | `src/main/resources/db/migration/` | 주문 테이블 생성 |
| `V2__create_order_items_table.sql` | `src/main/resources/db/migration/` | 주문 항목 테이블 생성 |
| `V3__add_index_orders_customer_id.sql` | `src/main/resources/db/migration/` | 고객 ID 인덱스 추가 |

> 네이밍 형식: `V{번호}__{설명}.sql`

---

## 변경 시 체크리스트

- [ ] `CoroutineCrudRepository`를 사용하는가? (`ReactiveCrudRepository` 아님)
- [ ] Entity가 `adapter/out/persistence/entity/`에만 존재하는가?
- [ ] Entity <-> Domain 변환에 매퍼를 사용하는가?
- [ ] Spring Data Repository를 직접 노출하지 않고 Adapter로 감쌌는가?
- [ ] 트랜잭션이 Application Service 레벨에서 관리되는가?
- [ ] 스키마 변경이 마이그레이션 파일로 관리되는가?
