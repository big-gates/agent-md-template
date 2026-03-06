# core / domain

> **트리거**: `core/domain/` 하위 파일을 수정할 때, 또는 도메인 모델·UseCase·Repository 인터페이스를 추가·변경할 때 이 문서를 참조한다.

## 역할
- **도메인 모델**, **UseCase**, **Repository 인터페이스**를 정의한다.
- 프레임워크·라이브러리에 의존하지 않는 **순수 Kotlin** 레이어이다.
- 클린 아키텍처에서 가장 안쪽 원이며, 어떤 외부 레이어에도 의존하지 않는다 (`core:common` 제외).

## 표준 경로

| 경로 | 설명 | 예시 |
|------|------|------|
| `core/domain/src/commonMain/kotlin/{package}/model/` | 도메인 모델 (비즈니스 엔티티) | `User.kt`, `Item.kt`, `Dashboard.kt` |
| `core/domain/src/commonMain/kotlin/{package}/repository/` | Repository 인터페이스 | `UserRepository.kt`, `ItemRepository.kt` |
| `core/domain/src/commonMain/kotlin/{package}/usecase/` | UseCase 클래스 | `GetUserUseCase.kt`, `SyncItemsUseCase.kt` |

## 의존/연동

| from | to | 관계 |
|------|----|------|
| `feature:*` | `core:domain` | 의존 (UseCase, Repository 인터페이스 호출) |
| `core:data` | `core:domain` | 의존 (Repository 인터페이스 구현, **의존성 역전**) |
| `core:domain` | `core:common` | 의존 (유틸, 확장 함수만 허용) |

- `core:common`만 의존할 수 있다 (유틸, 확장 함수 등).
- `core:data`가 Repository 인터페이스를 **구현**한다 (의존성 역전).
- `feature:*`가 UseCase 또는 Repository 인터페이스를 호출한다.
- `core:datasource`, `core:data`에 의존하지 않는다. 의존 방향은 항상 **data -> domain** (역전).

---

## 도메인 모델

네트워크 DTO, DB 엔티티와 분리된 순수 비즈니스 모델이다.

```kotlin
// 순수 data class — 프레임워크 어노테이션 없음
data class Item(
    val id: Long,
    val name: String,
    val expirationDate: LocalDate?,
    val isActive: Boolean,
)

data class User(
    val id: Long,
    val name: String,
    val email: String,
)
```

| 규칙 | 설명 |
|------|------|
| `val` 전용 | 불변 프로퍼티만 사용 |
| 프레임워크 무관 | Room `@Entity`, Kotlinx `@Serializable` 등 금지 |
| DTO/Entity와 분리 | 매핑은 `core:data`의 mapper에서 수행 |

### 모델 계층 분리

| 계층 | 타입 | 위치 | 용도 |
|------|------|------|------|
| Domain Model | `Item` | `core:domain/model/` | 비즈니스 로직, UI 전달 |
| DB Entity | `ItemEntity` | `core:datasource/local/entity/` | DB 테이블 매핑 |
| Network DTO | `ItemDto` | `core:datasource/remote/model/` | API 요청/응답 직렬화 |
| Mapper | `ItemMapper` | `core:data/mapper/` | 계층 간 모델 변환 |

### 변경 시 주의
- 모델 변경은 매퍼, DTO, Entity에 연쇄 영향이 있다.
- 직렬화 호환성(Kotlinx Serialization 등)은 DTO 쪽에서 관리한다.
- 필드 추가/삭제 시 `core:data`의 mapper를 함께 갱신한다.

---

## Repository 인터페이스

```kotlin
// domain 레이어에 인터페이스 정의 (순수 Kotlin)
interface ItemRepository {
    fun observeItems(): Flow<List<Item>>         // 반응형 조회
    suspend fun getItemById(id: Long): Item?     // 일회성 조회
    suspend fun addItem(item: Item)              // 변경
    suspend fun deleteItem(id: Long)             // 변경
    suspend fun sync()                           // 동기화
}

// data 레이어에서 구현 (프레임워크 의존 허용)
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource,
) : ItemRepository {
    // 구현...
}
```

| 규칙 | 설명 |
|------|------|
| 인터페이스는 `core:domain`에 | 순수 Kotlin, 프레임워크 무관 |
| 구현은 `core:data`에 | DB, 네트워크 등 외부 의존 허용 |
| 반환 타입은 도메인 모델 | DTO, Entity가 아닌 `model/`의 타입 |

---

## UseCase

### 기본 패턴

```kotlin
// 단일 책임: 하나의 비즈니스 동작만 수행
class GetActiveItemsUseCase(
    private val itemRepository: ItemRepository,  // 인터페이스에 의존
) {
    operator fun invoke(): Flow<List<Item>> =
        itemRepository.observeItems()
            .map { items -> items.filter { it.isActive } }
}
```

### UseCase 작성 기준

| 규칙 | 설명 |
|------|------|
| `operator fun invoke()` | 호출 시 `useCase()` 형태로 간결하게 사용 |
| 단일 책임 | 하나의 비즈니스 동작만 수행 |
| 순수 로직 | Android/iOS 프레임워크에 의존하지 않음 |
| 생성자 주입 | Repository 인터페이스를 DI로 주입 |

### UseCase가 필요한 경우 vs 불필요한 경우

| 필요 | 불필요 |
|------|--------|
| 여러 Repository를 조합하는 비즈니스 로직 | Repository 메서드를 단순 위임하는 경우 |
| 데이터 변환/필터링 등 도메인 규칙이 있는 경우 | ViewModel이 Repository를 직접 호출해도 충분한 경우 |
| 같은 로직을 여러 ViewModel에서 재사용하는 경우 | 한 화면에서만 사용하는 단순 CRUD |

```kotlin
// 필요 — 여러 Repository 조합 + 비즈니스 규칙
class GetDashboardUseCase(
    private val userRepo: UserRepository,
    private val itemRepo: ItemRepository,
    private val statsRepo: StatsRepository,
) {
    operator fun invoke(): Flow<Dashboard> = combine(
        userRepo.observeUser(),
        itemRepo.observeItems(),
        statsRepo.observeStats(),
    ) { user, items, stats ->
        Dashboard(
            userName = user.name,
            activeItems = items.filter { it.isActive },
            expiringCount = stats.expiringCount,
        )
    }
}

// 불필요 — 단순 위임은 ViewModel에서 Repository 직접 호출
class GetItemsUseCase(private val repo: ItemRepository) {
    operator fun invoke() = repo.observeItems()   // 의미 없는 래핑
}
```

---

## 에이전트 판단 규칙

### 이 모듈에 넣어야 하는가?

| 질문 | Yes → core:domain | No → 다른 곳 |
|------|-------------------|-------------|
| 순수 비즈니스 모델인가? | `model/`에 배치 | DTO → `core:datasource/remote/model/`, Entity → `core:datasource/local/entity/` |
| 여러 Repository를 조합하는 로직인가? | `usecase/`에 배치 | 단순 CRUD → ViewModel에서 Repository 직접 호출 |
| Repository의 계약(인터페이스)인가? | `repository/`에 배치 | 구현 → `core:data` |
| 프레임워크 의존이 없는가? | O → domain에 적합 | Room/Ktor/Serialization 의존 → `core:datasource` |

### 새 도메인 엔티티를 추가할 때

| 단계 | 수행 |
|------|------|
| 1 | `model/`에 data class 생성 (val 전용, 프레임워크 어노테이션 없음) |
| 2 | 필요하면 `repository/`에 Repository 인터페이스 추가 |
| 3 | `core:data`에 Repository 구현 + mapper 추가 |
| 4 | `core:datasource/local/entity/`에 Entity, `core:datasource/remote/model/`에 DTO 추가 |
| 5 | DI 모듈에 바인딩 등록 |

## DO / DON'T 요약

```kotlin
// DO — 순수 Kotlin data class
data class Item(val id: Long, val name: String, val isActive: Boolean)

// DO — Repository 인터페이스는 도메인 모델만 반환
interface ItemRepository {
    fun observeItems(): Flow<List<Item>>   // Item (도메인), not ItemDto/ItemEntity
}

// DO — UseCase는 operator fun invoke
class GetActiveItemsUseCase(private val repo: ItemRepository) {
    operator fun invoke(): Flow<List<Item>> =
        repo.observeItems().map { it.filter { item -> item.isActive } }
}

// DON'T — 프레임워크 어노테이션
@Entity data class Item(...)         // 금지: Room 의존
@Serializable data class Item(...)   // 금지: Serialization 의존

// DON'T — Repository 인터페이스가 DTO/Entity 반환
interface ItemRepository {
    fun observeItems(): Flow<List<ItemDto>>   // 금지: 인프라 타입 노출
}
```

## 변경 시 체크리스트
- [ ] 도메인 모델에 프레임워크 어노테이션이 유입되지 않았는가?
- [ ] Repository 인터페이스가 도메인 모델만 반환하는가? (DTO/Entity 아님)
- [ ] UseCase가 단일 책임을 유지하는가?
- [ ] 도메인 모델 변경 시 `core:data`의 mapper를 함께 갱신했는가?
- [ ] Repository 인터페이스 변경 시 `core:data`의 구현을 함께 갱신했는가?
- [ ] UseCase/Repository 추가 시 DI 모듈에 등록했는가?
