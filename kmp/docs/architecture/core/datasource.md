# core / datasource

> **트리거**: `core/datasource/` 하위 파일을 수정할 때, Local/Remote/Memory DataSource를 추가·변경할 때, 또는 인프라(DB, 네트워크) 관련 코드를 변경할 때 이 문서를 참조한다.

## 역할
- **Local/Remote/Memory DataSource 인터페이스, Entity, DTO 정의**를 제공한다.
- 인프라 기술(Room, SQLDelight, Ktor, Retrofit 등)을 **추상화**하여 교체 가능하게 한다.
- 구현은 별도 인프라 모듈(`core:database`, `core:network`)에서 제공한다.
- `core:data`(Repository)가 DataSource 인터페이스를 조합하여 비즈니스 데이터를 제공한다.

---

## 표준 경로

### 분리 모듈 구성

DataSource 인터페이스와 인프라 구현을 별도 Gradle 모듈로 분리한다. `core:datasource`는 인터페이스 + Entity/DTO 정의만 제공하고, 구현은 인프라별 모듈에서 제공한다.

| 모듈 | 역할 | Gradle 관계 |
|------|------|------------|
| `core/datasource` | DataSource 인터페이스 + Entity/DTO 정의 (구현 없음) | — |
| `core/database` | Room 기반 Local 구현 | `api(core:datasource)` |
| `core/network` | Ktor 기반 Remote 구현 | `api(core:datasource)` |

#### `core/datasource` (인터페이스 모듈)

| 경로 (`core/datasource/src/commonMain/kotlin/{package}/`) | 설명 | 예시 |
|----------------------------------------------------------|------|------|
| `local/{Name}LocalDataSource.kt` | 로컬 DataSource 인터페이스 | `ItemLocalDataSource.kt` |
| `local/TransactionRunner.kt` | 트랜잭션 추상화 | — |
| `local/entity/` | DB 엔티티 (`@Entity` 어노테이션 포함) | `ItemEntity.kt` |
| `local/dao/` | DAO 인터페이스 (`@Dao` 어노테이션 포함) | `ItemDao.kt` |
| `local/relation/` | Room Relation 클래스 | `ItemWithTags.kt` |
| `remote/model/req/` | API 요청 DTO | `CreateItemReq.kt` |
| `remote/model/res/` | API 응답 DTO | `ItemRes.kt` |
| `remote/service/` | API 서비스 인터페이스 | `ItemApiService.kt` |
| `remote/client/` | 클라이언트 인터페이스 (AccessTokenStore 등) | `AccessTokenStore.kt` |
| `memory/` | 메모리 캐시 인터페이스 (선택) | `TokenCacheDataSource.kt` |

#### `core/database` (Room 구현 모듈)

| 경로 (`core/database/src/commonMain/kotlin/{package}/`) | 설명 | 예시 |
|--------------------------------------------------------|------|------|
| `impl/` | LocalDataSource 구현 | `ItemLocalDataSourceImpl.kt` |
| `converter/` | Room TypeConverter | `IntervalConverter.kt` |
| `{App}Database.kt` | @Database 정의 | — |
| `DatabaseModule.kt` | DI 모듈 (DAO + LocalDataSource 바인딩) | — |
| `Migrations.kt` | 마이그레이션 정의 | — |

#### `core/network` (Ktor 구현 모듈)

| 경로 (`core/network/src/commonMain/kotlin/{package}/`) | 설명 | 예시 |
|-------------------------------------------------------|------|------|
| `{Name}ApiServiceImpl.kt` | API 서비스 구현 | `ItemApiServiceImpl.kt` |
| `client/CreateHttpClient.kt` | Ktor HttpClient 설정 | — |
| `client/{Name}AccessTokenStore.kt` | AccessTokenStore 구현 | `InMemoryAccessTokenStore.kt` |
| `di/NetworkModule.kt` | DI 모듈 (서비스 + 클라이언트 바인딩) | — |

---

## 의존/연동

| from | to | 관계 | 설명 |
|------|----|------|------|
| `core:data` | `core:datasource` | `implementation` | DataSource 인터페이스 사용 |
| `core:database` | `core:datasource` | `api` | 인터페이스 구현 + Entity/DAO 전이 노출 |
| `core:network` | `core:datasource` | `api` | 인터페이스 구현 + DTO 전이 노출 |
| `shared` | `core:database` | `implementation` | DI 모듈 조합용 |
| `shared` | `core:network` | `implementation` | DI 모듈 조합용 |

| 규칙 | 설명 |
|------|------|
| `core:data`는 인터페이스만 사용 | `core:datasource`의 인터페이스에만 의존. 구현(Room, Ktor)을 모름 |
| `feature:*`는 DataSource 접근 금지 | `core:domain` 인터페이스를 통해서만 데이터 접근 |
| 인프라 기술은 구현 모듈에 캡슐화 | Room은 `core:database`, Ktor는 `core:network`에만 존재 |
| DI 바인딩은 구현 모듈에서 | `core:database`가 LocalDataSource 바인딩, `core:network`가 ApiService 바인딩 제공 |

---

## DataSource 유형

| 유형 | 역할 | SOT 여부 | 반환 타입 |
|------|------|---------|----------|
| **Local** | 영속 데이터, Offline-First 대상 | O (주요 SOT) | `Entity` |
| **Remote** | 서버 데이터 fetch/push | X (동기화 소스) | `DTO` |
| **Memory** | 세션 토큰, 임시 필터, 캐시 | O (세션 한정) | 원시 타입 |

---

## Local DataSource (로컬 저장소)

### Entity 규칙

Entity는 DB 테이블에 매핑되는 **인프라 모델**이다. 비즈니스 로직을 포함하지 않는다.

```kotlin
// DO -- Entity는 DB 어노테이션만 포함
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val expirationDate: Long?,   // DB는 Long(timestamp), 변환은 mapper에서
)

// DON'T -- Entity에 비즈니스 로직
@Entity(tableName = "items")
data class ItemEntity(...) {
    fun isExpired(): Boolean = ...   // 금지: 로직은 domain 모델에 배치
}
```

### DAO 규칙

```kotlin
// DO -- DAO에서 Flow 반환 (변경 자동 감지)
@Dao
interface ItemDao {
    @Query("SELECT * FROM items ORDER BY name")
    fun observeAll(): Flow<List<ItemEntity>>

    @Query("SELECT * FROM items WHERE id = :id")
    suspend fun getById(id: Long): ItemEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(items: List<ItemEntity>)

    @Query("DELETE FROM items WHERE id = :id")
    suspend fun deleteById(id: Long)
}
```

### 마이그레이션 규칙

| 상황 | 수행 |
|------|------|
| 새 테이블 추가 | Entity + DAO 생성 -> `AppDatabase` entities 목록 추가 -> 마이그레이션 SQL -> DB 버전 증가 |
| 컬럼 추가/삭제 | Entity 수정 -> 마이그레이션 SQL -> DB 버전 증가 |
| 인덱스 추가 | `@Entity(indices = [...])` 또는 마이그레이션에서 `CREATE INDEX` |

```kotlin
// 마이그레이션 예시
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE items ADD COLUMN category TEXT")
    }
}
```

### 마이그레이션 테스트

마이그레이션은 데이터 무결성에 직결되므로 반드시 테스트한다.

```kotlin
@Test
fun `migration_1_2_컬럼 추가 후 기존 데이터 유지`() {
    // Given: v1 스키마에 데이터 삽입
    helper.createDatabase(TEST_DB, 1).apply {
        execSQL("INSERT INTO items (id, name) VALUES (1, 'Test')")
        close()
    }
    // When: v2로 마이그레이션
    val db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)
    // Then: 기존 데이터 유지 + 새 컬럼 존재
    val cursor = db.query("SELECT * FROM items WHERE id = 1")
    assertTrue(cursor.moveToFirst())
    assertEquals("Test", cursor.getString(cursor.getColumnIndex("name")))
}
```

---

## Remote DataSource (네트워크)

### DTO 규칙

DTO는 API 직렬화 전용 모델이다. 비즈니스 로직이나 매핑 함수를 포함하지 않는다.

```kotlin
// DO -- DTO는 직렬화 전용
@Serializable
data class ItemDto(
    val id: Long,
    val name: String,
    @SerialName("expiration_date")
    val expirationDate: String?,
)

// DON'T -- DTO에서 도메인 변환
@Serializable
data class ItemDto(...) {
    fun toDomain(): Item = ...   // 금지: 매핑은 core:data에서
}
```

### API Service 규칙

```kotlin
// DO -- Service는 DTO만 반환
interface ItemService {
    suspend fun getItems(): List<ItemDto>
    suspend fun getItem(id: Long): ItemDto
    suspend fun createItem(request: CreateItemRequest): ItemDto
}
```

### HTTP Client 설정

```kotlin
// DO -- 클라이언트 설정은 client/ 패키지에 집중
fun createHttpClient(): HttpClient = HttpClient {
    install(ContentNegotiation) { json() }
    install(Logging) { level = LogLevel.BODY }
    defaultRequest { url("https://api.example.com/") }
}
```

### 인증/인터셉터 패턴

```kotlin
// DO -- 인증 토큰 인터셉터
fun createHttpClient(tokenProvider: TokenProvider): HttpClient = HttpClient {
    install(Auth) {
        bearer {
            loadTokens { BearerTokens(tokenProvider.getAccessToken(), "") }
            refreshTokens { tokenProvider.refresh(); loadTokens() }
        }
    }
}
```

### 에러 처리

```kotlin
// DO -- 네트워크 에러를 도메인 예외로 변환 (DataSource 내부에서)
class ItemRemoteDataSourceImpl(
    private val service: ItemService,
    private val ioDispatcher: CoroutineDispatcher,
) : ItemRemoteDataSource {
    override suspend fun fetchItems(): List<ItemDto> = withContext(ioDispatcher) {
        try {
            service.getItems()
        } catch (e: ClientRequestException) {
            throw NetworkException.from(e)   // core:common의 공통 예외로 변환
        }
    }
}
```

---

## Memory DataSource (메모리 캐시)

```kotlin
// DO -- StateFlow 기반 세션 데이터
class TokenCacheDataSource {
    private val _token = MutableStateFlow<String?>(null)
    val token: StateFlow<String?> = _token.asStateFlow()
    fun update(token: String?) { _token.value = token }
    fun clear() { _token.value = null }
}
```

---

## DataSource 인터페이스 설계 규칙

| 규칙 | 설명 |
|------|------|
| 인터페이스 + 구현 분리 | Fake 교체로 테스트 가능 |
| Dispatcher 전환은 내부에서 | `withContext(ioDispatcher)` -- 호출자는 Dispatcher를 몰라도 됨 |
| 반환 타입 | Local -> `Entity`, Remote -> `DTO`, Memory -> 원시 타입 |
| 매핑 금지 | DataSource는 데이터를 그대로 반환. 변환은 `core:data`의 mapper |
| 비즈니스 로직 금지 | 데이터 접근만. 필터링/정렬/규칙은 Repository 또는 UseCase |

---

## 새 DataSource 추가 절차

| 단계 | 수행 |
|------|------|
| 1 | 인터페이스 정의 (`{Name}LocalDataSource` 또는 `{Name}RemoteDataSource`) |
| 2 | 구현 클래스 작성 (`{Name}LocalDataSourceImpl`) |
| 3 | 인프라 모델 작성: Local -> `entity/{Name}Entity.kt`, Remote -> `model/{Name}Dto.kt` |
| 4 | Local: DAO 추가 (`dao/{Name}Dao.kt`), Remote: Service 메서드 추가 |
| 5 | Dispatcher 전환을 구현 내부에서 `withContext(ioDispatcher)`로 처리 |
| 6 | DI 모듈에 바인딩 등록 ([di-guide.md](../../convention/di-guide.md) 참조) |
| 7 | `core:data`의 Repository에서 조합 + mapper 작성 ([data.md](data.md) 참조) |
| 8 | Fake 구현체 작성 + 테스트 ([testing-guide.md](../../convention/testing-guide.md) 참조) |

---

## DO / DON'T 종합

```kotlin
// DO -- 인터페이스 + 구현 분리, Dispatcher 내부 처리
interface ItemLocalDataSource {
    fun observeAll(): Flow<List<ItemEntity>>
    suspend fun getById(id: Long): ItemEntity?
    suspend fun insertAll(items: List<ItemEntity>)
    suspend fun deleteById(id: Long)
}

class ItemLocalDataSourceImpl(
    private val itemDao: ItemDao,
    private val ioDispatcher: CoroutineDispatcher,
) : ItemLocalDataSource {
    override fun observeAll(): Flow<List<ItemEntity>> = itemDao.observeAll()
    override suspend fun getById(id: Long) = withContext(ioDispatcher) { itemDao.getById(id) }
    override suspend fun insertAll(items: List<ItemEntity>) = withContext(ioDispatcher) { itemDao.insertAll(items) }
    override suspend fun deleteById(id: Long) = withContext(ioDispatcher) { itemDao.deleteById(id) }
}

// DO -- Remote DataSource
interface ItemRemoteDataSource {
    suspend fun fetchItems(): List<ItemDto>
    suspend fun fetchItem(id: Long): ItemDto
    suspend fun createItem(request: CreateItemRequest): ItemDto
}

class ItemRemoteDataSourceImpl(
    private val service: ItemService,
    private val ioDispatcher: CoroutineDispatcher,
) : ItemRemoteDataSource {
    override suspend fun fetchItems() = withContext(ioDispatcher) { service.getItems() }
    override suspend fun fetchItem(id: Long) = withContext(ioDispatcher) { service.getItem(id) }
    override suspend fun createItem(request: CreateItemRequest) = withContext(ioDispatcher) { service.createItem(request) }
}

// DON'T -- DataSource에서 비즈니스 로직
class ItemLocalDataSourceImpl(...) : ItemLocalDataSource {
    override suspend fun getActiveItems() =
        itemDao.getAll().filter { it.isActive }   // 금지: 필터링은 Repository/UseCase에서
}

// DON'T -- DataSource에서 모델 변환
class ItemRemoteDataSourceImpl(...) : ItemRemoteDataSource {
    override suspend fun fetchItems(): List<Item> =
        service.getItems().map { it.toDomain() }   // 금지: 매핑은 core:data에서
}

// DON'T -- feature에서 DataSource 직접 사용
class HomeViewModel(
    private val itemLocalDataSource: ItemLocalDataSource,   // 금지
) { ... }
```

---

## 인프라 모듈 추가 가이드

프로젝트에 새로운 인프라 기술(예: Room, SQLDelight, Ktor)을 도입하거나 분리 모듈로 구성할 때, 해당 모듈의 MD 문서를 작성한다.

### 문서 작성 절차

| 단계 | 수행 |
|------|------|
| 1 | `docs/architecture/core/datasource-{tech}.md` 파일 생성 (예: `datasource-room.md`) |
| 2 | 아래 템플릿을 기반으로 작성 |
| 3 | [docs/architecture/README.md](../README.md)의 모듈 인덱스에 등록 |
| 4 | [AGENTS.md](../../../AGENTS.md)의 라우팅 테이블에 경로 패턴 추가 |

### 문서 템플릿

```markdown
# core / datasource-{tech}

> **트리거**: `core/datasource-{tech}/` 하위 파일을 수정할 때 이 문서를 참조한다.

## 역할
- {기술명} 기반의 {Local/Remote} DataSource 구현을 제공한다.
- `core:datasource`의 DataSource 인터페이스를 구현한다.

## 인프라 기술
- **기술**: {Room / SQLDelight / Ktor / Retrofit 등}
- **버전**: {사용 중인 버전}
- **KMP 지원**: {commonMain / androidMain+iosMain 분리 등}

## 표준 경로
{모듈별 디렉터리 구조}

## 설정
{기술별 초기화/설정 코드 예시}

## 에이전트 판단 규칙
{기술 특화 판단 테이블}

## DO / DON'T
{기술 특화 예시}

## 변경 시 체크리스트
- [ ] DataSource 인터페이스를 구현하는가?
- [ ] 인프라 기술이 DataSource 내부에 캡슐화되어 있는가?
- [ ] DI 모듈에 등록했는가?
- [ ] 매핑은 core:data에서 수행하는가?
- [ ] Fake 구현체로 테스트가 가능한가?
```

---

## 계층 간 모델과 매퍼

DataSource 계층의 모델(Entity, DTO)은 도메인 모델과 분리된다. 변환은 `core:data`의 mapper에서 수행한다.

| 변환 방향 | from | to | 매퍼 위치 |
|----------|------|----|----------|
| Entity -> Domain | `core:datasource/local/entity/ItemEntity.kt` | `core:domain/model/Item.kt` | `core:data/mapper/ItemMapper.kt` |
| Domain -> Entity | `core:domain/model/Item.kt` | `core:datasource/local/entity/ItemEntity.kt` | `core:data/mapper/ItemMapper.kt` |
| DTO -> Domain | `core:datasource/remote/model/ItemDto.kt` | `core:domain/model/Item.kt` | `core:data/mapper/ItemMapper.kt` |
| DTO -> Entity (Sync) | `core:datasource/remote/model/ItemDto.kt` | `core:datasource/local/entity/ItemEntity.kt` | `core:data/mapper/ItemMapper.kt` |

| 계층 | 모델 타입 | 위치 | 용도 |
|------|----------|------|------|
| Domain | `Item` | `core:domain/model/` | 비즈니스 로직, UI 전달 |
| Local (DB) | `ItemEntity` | `core:datasource/local/entity/` | DB 테이블 매핑 |
| Remote (API) | `ItemDto` | `core:datasource/remote/model/` | API 직렬화 |

매퍼 작성 규칙은 [core/data.md](data.md)를 참조한다.

---

## 변경 시 체크리스트

### DataSource 공통
- [ ] 인터페이스와 구현이 분리되어 있는가?
- [ ] `withContext(ioDispatcher)`로 Dispatcher 전환을 내부에서 처리하는가?
- [ ] 비즈니스 로직(필터, 정렬, 검증)이 유입되지 않았는가?
- [ ] 모델 변환(매핑)이 유입되지 않았는가? (Entity/DTO를 그대로 반환)
- [ ] DI 모듈에 바인딩을 등록했는가?
- [ ] `feature:*`에서 직접 참조하지 않는가?
- [ ] Fake 구현체를 작성했는가?

### Local (DB) 변경 시
- [ ] 스키마 변경 시 마이그레이션 SQL을 작성했는가?
- [ ] DB 버전 번호를 증가시켰는가?
- [ ] 새 Entity를 `AppDatabase` entities 목록에 추가했는가?
- [ ] 마이그레이션 테스트를 작성했는가?
- [ ] Entity에 비즈니스 로직이 유입되지 않았는가?

### Remote (API) 변경 시
- [ ] API 스펙 변경 시 DTO를 함께 수정했는가?
- [ ] DTO 변경 시 `core:data`의 mapper를 갱신했는가?
- [ ] 인증 토큰/인터셉터 변경의 전역 영향을 확인했는가?
- [ ] 에러 처리가 적절한가? (네트워크 예외 -> 도메인 예외 변환)
- [ ] DTO에 비즈니스 로직이나 매핑 함수가 유입되지 않았는가?
