# core / data

> **트리거**: `core/data/` 하위 파일을 수정할 때, 또는 Repository 구현·Mapper·동기화·오프라인 큐 로직을 변경할 때 이 문서를 참조한다.

## 역할
- Repository **구현**, Mapper, Sync 로직 등 데이터 레이어의 중심이다.
- `core:domain`의 Repository 인터페이스를 구현한다 (의존성 역전).
- 계층 간 모델 변환(Mapper)의 거점이다.
- Offline-First 원칙의 구현 거점이다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `core/data/src/commonMain/kotlin/{package}/repository/` | Repository 구현 (core:domain 인터페이스 구현) |
| `core/data/src/commonMain/kotlin/{package}/mapper/` | 모델 간 매핑 (Entity <-> Domain, DTO <-> Domain) |
| `core/data/src/commonMain/kotlin/{package}/sync/` | 동기화 로직, 오프라인 큐 처리 |

## 의존/연동

| from | to | 관계 |
|------|----|------|
| `core:data` | `core:domain` | 의존 (Repository 인터페이스 구현, **의존성 역전**) |
| `core:data` | `core:datasource` | 의존 (DataSource 사용) |
| `core:data` | `core:domain/model/` | 매핑 대상 (도메인 모델 참조) |

- `core:domain`의 Repository 인터페이스를 **구현**한다 (의존성 역전).
- `core:datasource`를 통해 Local/Remote 데이터에 접근한다.
- 계층 간 모델 변환(Mapper)을 수행한다.
- `feature:*`는 이 모듈을 직접 참조하지 않는다. `core:domain`의 인터페이스를 통해 접근한다.

---

## 매퍼 (Mapper)

### 매퍼 원칙

계층 간 데이터 이동은 반드시 Mapper를 통해 수행한다. Entity/DTO를 상위 레이어에 직접 전달하지 않는다.

| 변환 방향 | from (입력) | to (출력) | 용도 |
|----------|------------|----------|------|
| Entity -> Domain | `ItemEntity` (`core:datasource`) | `Item` (`core:domain`) | 로컬 데이터 조회 |
| Domain -> Entity | `Item` (`core:domain`) | `ItemEntity` (`core:datasource`) | 로컬 데이터 저장 |
| DTO -> Domain | `ItemDto` (`core:datasource`) | `Item` (`core:domain`) | API 응답 변환 |
| DTO -> Entity | `ItemDto` (`core:datasource`) | `ItemEntity` (`core:datasource`) | Sync (서버 -> 로컬 DB) |

### 매퍼 작성 패턴

```kotlin
// 패턴 1: 확장 함수 (간단한 1:1 매핑에 적합)
// core/data/mapper/ItemMapper.kt

fun ItemEntity.toDomain(): Item = Item(
    id = id,
    name = name,
    expirationDate = expirationDate?.let {
        Instant.fromEpochMilliseconds(it).toLocalDate()
    },
    isActive = isActive,
)

fun Item.toEntity(): ItemEntity = ItemEntity(
    id = id,
    name = name,
    expirationDate = expirationDate?.toEpochMilliseconds(),
    isActive = isActive,
)

fun ItemDto.toDomain(): Item = Item(
    id = id,
    name = name,
    expirationDate = expirationDate?.let { LocalDate.parse(it) },
    isActive = true,   // 서버에서 내려온 건 기본 활성
)

fun ItemDto.toEntity(): ItemEntity = ItemEntity(
    id = id,
    name = name,
    expirationDate = expirationDate?.let { LocalDate.parse(it).toEpochMilliseconds() },
    isActive = true,
)
```

```kotlin
// 패턴 2: Mapper 클래스 (복잡한 매핑, 외부 의존이 필요한 경우)
class ItemMapper(
    private val dateFormatter: DateFormatter,   // core:common의 유틸 주입
) {
    fun toDomain(entity: ItemEntity): Item = Item(
        id = entity.id,
        name = entity.name,
        expirationDate = dateFormatter.fromEpoch(entity.expirationDate),
    )

    fun toEntity(domain: Item): ItemEntity = ItemEntity(
        id = domain.id,
        name = domain.name,
        expirationDate = dateFormatter.toEpoch(domain.expirationDate),
    )

    fun dtoToEntity(dto: ItemDto): ItemEntity = ItemEntity(
        id = dto.id,
        name = dto.name,
        expirationDate = dateFormatter.parseIso(dto.expirationDate),
    )
}
```

### 매퍼 규칙

| 규칙 | 설명 |
|------|------|
| 매퍼는 `core:data/mapper/`에만 위치 | Entity/DTO에 `toDomain()` 함수를 직접 작성하지 않음 |
| 도메인 모델을 반환 | Repository의 public API는 항상 도메인 모델을 반환 |
| Sync용 DTO→Entity 매퍼 | 서버 응답을 로컬 DB에 저장할 때 사용 |
| 매퍼는 반드시 테스트 | 각 변환 방향별 유닛 테스트 작성 |
| 매퍼에 비즈니스 로직 금지 | 순수 데이터 변환만 수행. 필터링/검증은 UseCase에서 |

### 매퍼 테스트

```kotlin
class ItemMapperTest {
    @Test
    fun `Entity를 Domain으로 정확히 변환`() {
        val entity = ItemEntity(id = 1, name = "Test", expirationDate = 1735689600000)
        val domain = entity.toDomain()
        assertEquals(1L, domain.id)
        assertEquals("Test", domain.name)
        assertEquals(LocalDate(2025, 12, 31), domain.expirationDate)
    }

    @Test
    fun `DTO를 Domain으로 정확히 변환`() {
        val dto = ItemDto(id = 1, name = "Test", expirationDate = "2025-12-31")
        val domain = dto.toDomain()
        assertEquals(1L, domain.id)
        assertEquals(LocalDate(2025, 12, 31), domain.expirationDate)
    }

    @Test
    fun `DTO를 Entity로 변환_Sync용`() {
        val dto = ItemDto(id = 1, name = "Test", expirationDate = "2025-12-31")
        val entity = dto.toEntity()
        assertEquals(1L, entity.id)
        assertNotNull(entity.expirationDate)
    }

    @Test
    fun `null 필드 변환 시 안전하게 처리`() {
        val dto = ItemDto(id = 1, name = "Test", expirationDate = null)
        val domain = dto.toDomain()
        assertNull(domain.expirationDate)
    }
}
```

---

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 Repository 구현 추가 | `core:domain`에 인터페이스가 먼저 있는지 확인. 없으면 인터페이스부터 생성 |
| DataSource 직접 호출 코드 발견 (feature에서) | **금지**. domain -> data -> datasource 경로를 따르도록 리팩터 |
| 매퍼 추가 | `mapper/`에 확장 함수 또는 Mapper 클래스로 작성. **반드시 테스트** 추가 |
| DTO→Entity 매퍼 필요 (Sync용) | `mapper/`에 `{Name}Dto.toEntity()` 확장 함수로 작성 |
| 오프라인 큐 타입 추가 | 아래 큐 패턴 참조 + 이 문서의 큐 운영 규칙 갱신 |
| Entity/DTO/Domain 모델 변경 | 매퍼를 함께 갱신 + 매퍼 테스트 갱신 |

## DO / DON'T

```kotlin
// DO -- domain 인터페이스 구현, DataSource 조합, Mapper 사용
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource,
) : ItemRepository {   // <- core:domain의 인터페이스

    override fun observeItems(): Flow<List<Item>> =
        localDataSource.observeAll()
            .map { entities -> entities.map { it.toDomain() } }   // mapper로 변환

    override suspend fun sync() {
        val dtos = remoteDataSource.fetchItems()
        val entities = dtos.map { it.toEntity() }   // DTO -> Entity (Sync용 매퍼)
        localDataSource.insertAll(entities)
    }
}

// DON'T -- DAO/Service를 직접 사용 (DataSource를 우회)
class ItemRepositoryImpl(
    private val itemDao: ItemDao,        // 금지: datasource를 통해 접근
    private val itemService: ItemService, // 금지: datasource를 통해 접근
) : ItemRepository { ... }

// DON'T -- Entity/DTO를 상위 레이어에 직접 전달
class ItemRepositoryImpl(...) : ItemRepository {
    override fun observeItems(): Flow<List<ItemEntity>> =   // 금지: 도메인 모델로 변환 필수
        localDataSource.observeAll()
}

// DON'T -- feature에서 직접 참조
class HomeViewModel(
    private val itemRepo: ItemRepositoryImpl,   // 금지: 구현이 아닌 인터페이스에 의존
) { ... }
```

## 변경 시 주의
- 동기화 정책 변경은 데이터 유실/중복과 직결된다.
- 로컬 소스 오브 트루스(SOT) 원칙을 유지한다.

---

## Offline-First 원칙

로컬 DB를 SOT로 사용하고, 서버는 동기화 대상으로 취급한다.

| 흐름 | 설명 |
|------|------|
| UI -> Repository -> LocalDataSource | 읽기: 항상 로컬(SOT)에서 조회 |
| Repository -> RemoteDataSource -> LocalDataSource | 쓰기(Sync): 서버에서 fetch -> 로컬에 저장 |
| Repository -> LocalDataSource -> RemoteDataSource | 쓰기(Push): 로컬 변경 -> 서버에 반영 |

### 오프라인 큐 패턴

통신 실패 시 변경 사항을 로컬 큐에 적재하고, 연결 복구 후 서버에 반영한다.

| 큐 전략 | 설명 | 적용 예시 |
|---------|------|----------|
| **엔티티별 최신 1건** | 동일 엔티티(`id` 기준) 최신 타임스탬프만 전송 | 아이템 CRUD |
| **타입별 최신 1건** | 해당 타입의 최신 요청만 큐에 유지 | 사용자 정보 업데이트, 약관 동의 |
| **성공 시 큐 정리** | 성공하면 같은 타입의 잔여 큐를 제거 | 기기 체크인, 헬스체크 |

### 충돌 해결 전략

| 전략 | 설명 | 적합한 경우 |
|------|------|-----------|
| **Last-Write-Wins** | 최신 타임스탬프의 변경이 우선 | 단일 사용자 앱, 빈번한 업데이트 |
| **서버 우선** | 충돌 시 서버 데이터를 채택, 로컬 변경 폐기 | 서버가 권위 있는 데이터 소스인 경우 |
| **로컬 우선 + 병합** | 로컬 변경을 서버에 push, 실패 시 재시도 | Offline-First가 핵심인 경우 |

### 큐 운영 규칙
- 큐 타입을 추가/변경할 때 이 문서와 `AGENTS.md`를 함께 갱신한다.
- 큐 재시도 정책(간격, 최대 횟수)은 프로젝트 요구사항에 맞게 정의한다.
- 큐 처리 순서는 적재 시각(FIFO) 기준을 기본으로 한다.
- Sync 실패 시 로컬 SOT는 유지하고, 큐에 재시도 항목을 남긴다.

## 변경 시 체크리스트
- [ ] `core:domain`에 Repository 인터페이스가 먼저 정의되어 있는가?
- [ ] DataSource를 통해서만 인프라에 접근하는가? (DAO/Service 직접 사용 금지)
- [ ] 매퍼가 Entity <-> Domain, DTO <-> Domain 변환을 모두 커버하는가?
- [ ] 매퍼 테스트를 작성했는가?
- [ ] Repository의 public API가 도메인 모델만 반환하는가? (Entity/DTO 노출 금지)
- [ ] 동기화 로직이 SOT(Local) 원칙을 유지하는가?
- [ ] 오프라인 큐 타입 추가 시 이 문서의 큐 패턴을 갱신했는가?
- [ ] Fake Repository를 작성/갱신했는가?
