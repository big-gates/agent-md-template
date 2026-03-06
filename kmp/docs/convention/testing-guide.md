# Testing Guide

> **트리거**: 테스트 코드를 작성·수정할 때, 또는 새 기능/모듈에 테스트를 추가할 때 이 문서를 참조한다.

---

## TDD 워크플로우

TDD(Test-Driven Development)를 준수한다. 모든 새 기능/버그 수정은 다음 사이클을 따른다.

```
1. Red    — 실패하는 테스트를 먼저 작성
2. Green  — 테스트를 통과하는 최소한의 코드 작성
3. Refactor — 코드 정리 (테스트는 여전히 통과)
```

| 상황 | TDD 적용 |
|------|---------|
| 새 UseCase 추가 | UseCase 테스트 먼저 -> UseCase 구현 -> 리팩터 |
| 새 Repository 구현 | Repository 테스트 먼저 (Fake DataSource) -> 구현 -> 리팩터 |
| 새 Mapper 추가 | Mapper 테스트 먼저 (입출력 검증) -> Mapper 구현 |
| 새 ViewModel 추가 | ViewModel 테스트 먼저 (Fake Repository) -> 구현 -> 리팩터 |
| 버그 수정 | 버그를 재현하는 테스트 먼저 -> 수정 -> 테스트 통과 확인 |

---

## 테스트 피라미드

| 레벨 | 대상 | 비율 (권장) |
|------|------|-----------|
| Unit | UseCase, Repository 로직, ViewModel, Mapper, 유틸 | 70%+ |
| Integration | Repository + DataSource, DB 쿼리, API 직렬화 | 20% |
| E2E / UI | 주요 사용자 흐름 | 10% |

---

## 테스트 위치

```
{module}/src/commonTest/kotlin/{package}/    # KMP 공통 테스트
{module}/src/androidTest/kotlin/{package}/   # Android 전용 (Instrumented)
{module}/src/iosTest/kotlin/{package}/       # iOS 전용
```

---

## 네이밍 규칙

### 테스트 클래스
- `{대상클래스}Test` -- 예: `GetActiveItemsUseCaseTest`, `ItemRepositoryImplTest`

### 테스트 메서드
```kotlin
@Test
fun `주어진 조건_실행할 동작_기대 결과`() { }

// 예시
@Test
fun `빈 리스트일 때_observeItems 호출_빈 Flow 반환`() { }

@Test
fun `네트워크 실패 시_sync 호출_로컬 캐시 반환`() { }
```

---

## 레이어별 테스트 전략

### ViewModel 테스트

```kotlin
class ItemViewModelTest {
    private val fakeRepository = FakeItemRepository()
    private lateinit var viewModel: ItemViewModel

    @BeforeTest
    fun setup() {
        viewModel = ItemViewModel(fakeRepository)
    }

    @Test
    fun `아이템 로드 성공 시 UiState에 아이템 목록 반영`() = runTest {
        fakeRepository.emit(listOf(testItem))

        viewModel.onAction(ItemAction.Load)
        advanceUntilIdle()

        viewModel.uiState.test {
            val state = awaitItem()
            assertEquals(1, state.items.size)
            assertFalse(state.isLoading)
            assertNull(state.error)
        }
    }

    @Test
    fun `로드 실패 시 UiState에 에러 반영`() = runTest {
        fakeRepository.setShouldFail(true)

        viewModel.onAction(ItemAction.Load)
        advanceUntilIdle()

        viewModel.uiState.test {
            val state = awaitItem()
            assertNotNull(state.error)
            assertFalse(state.isLoading)
        }
    }
}
```

| 규칙 | 설명 |
|------|------|
| Fake 사용 | Repository/UseCase의 Fake 구현을 사용한다. Mock 프레임워크보다 Fake를 선호한다 |
| `runTest` 사용 | 코루틴 테스트에는 `kotlinx-coroutines-test`의 `runTest`를 사용한다 |
| Turbine 사용 | Flow 테스트에는 Turbine 라이브러리의 `.test { }` 블록을 사용한다 |
| Action 기반 테스트 | ViewModel의 `onAction()`을 통해 테스트 (내부 함수 직접 호출 금지) |

### UseCase 테스트

```kotlin
class GetActiveItemsUseCaseTest {
    private val fakeRepository = FakeItemRepository()
    private val useCase = GetActiveItemsUseCase(fakeRepository)

    @Test
    fun `활성 아이템만 필터링하여 반환`() = runTest {
        fakeRepository.emit(listOf(activeItem, inactiveItem))

        useCase().test {
            val result = awaitItem()
            assertEquals(1, result.size)
            assertTrue(result.all { it.isActive })
        }
    }
}
```

### Repository 테스트

```kotlin
class ItemRepositoryImplTest {
    private val fakeLocalDataSource = FakeItemLocalDataSource()
    private val fakeRemoteDataSource = FakeItemRemoteDataSource()
    private val repository = ItemRepositoryImpl(fakeLocalDataSource, fakeRemoteDataSource)

    @Test
    fun `sync 호출 시 원격 데이터를 로컬에 저장`() = runTest {
        fakeRemoteDataSource.setItems(listOf(remoteItemDto))

        repository.sync()

        val localItems = fakeLocalDataSource.getAll()
        assertEquals(1, localItems.size)
    }

    @Test
    fun `observeItems가 Entity를 Domain 모델로 변환하여 반환`() = runTest {
        fakeLocalDataSource.emit(listOf(testEntity))

        repository.observeItems().test {
            val items = awaitItem()
            assertEquals(1, items.size)
            assertIs<Item>(items.first())   // Domain 모델인지 확인
        }
    }

    @Test
    fun `sync 실패 시 로컬 데이터 유지`() = runTest {
        fakeLocalDataSource.emit(listOf(existingEntity))
        fakeRemoteDataSource.setShouldFail(true)

        try { repository.sync() } catch (_: Exception) {}

        val localItems = fakeLocalDataSource.getAll()
        assertEquals(1, localItems.size)   // 기존 데이터 유지
    }
}
```

### Mapper 테스트

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

### DataSource 테스트 (Integration)

```kotlin
// DB 마이그레이션 테스트 (Android Instrumented)
class MigrationTest {
    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java,
    )

    @Test
    fun `migration_1_2_컬럼 추가 후 기존 데이터 유지`() {
        helper.createDatabase(TEST_DB, 1).apply {
            execSQL("INSERT INTO items (id, name) VALUES (1, 'Test')")
            close()
        }

        val db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)

        val cursor = db.query("SELECT * FROM items WHERE id = 1")
        assertTrue(cursor.moveToFirst())
        assertEquals("Test", cursor.getString(cursor.getColumnIndex("name")))
    }
}
```

---

## Fake 작성 패턴

### Fake vs Mock

| 기준 | Fake | Mock |
|------|------|------|
| 정의 | 인터페이스의 간단한 인메모리 구현 | 프레임워크가 생성하는 동작 기록 객체 |
| 선호도 | **선호** | 필요 시에만 |
| 장점 | 읽기 쉽고, 리팩터링에 강하고, KMP 호환 | 설정이 간편 |
| 단점 | 구현 코드가 필요 | 테스트가 구현에 결합됨 |

### Fake Repository

```kotlin
class FakeItemRepository : ItemRepository {
    private val items = MutableStateFlow<List<Item>>(emptyList())
    private var shouldFail = false

    fun emit(newItems: List<Item>) { items.value = newItems }
    fun setShouldFail(fail: Boolean) { shouldFail = fail }

    override fun observeItems(): Flow<List<Item>> = items
    override suspend fun getItemById(id: Long): Item? = items.value.find { it.id == id }
    override suspend fun addItem(item: Item) { items.value = items.value + item }
    override suspend fun deleteItem(id: Long) { items.value = items.value.filter { it.id != id } }
    override suspend fun sync() {
        if (shouldFail) throw IOException("Fake network error")
    }
}
```

### Fake DataSource

```kotlin
class FakeItemLocalDataSource : ItemLocalDataSource {
    private val items = MutableStateFlow<List<ItemEntity>>(emptyList())

    fun emit(newItems: List<ItemEntity>) { items.value = newItems }
    fun getAll(): List<ItemEntity> = items.value

    override fun observeAll(): Flow<List<ItemEntity>> = items
    override suspend fun getById(id: Long) = items.value.find { it.id == id }
    override suspend fun insertAll(entities: List<ItemEntity>) {
        items.value = items.value + entities
    }
    override suspend fun deleteById(id: Long) {
        items.value = items.value.filter { it.id != id }
    }
}

class FakeItemRemoteDataSource : ItemRemoteDataSource {
    private var items: List<ItemDto> = emptyList()
    private var shouldFail = false

    fun setItems(newItems: List<ItemDto>) { items = newItems }
    fun setShouldFail(fail: Boolean) { shouldFail = fail }

    override suspend fun fetchItems(): List<ItemDto> {
        if (shouldFail) throw IOException("Fake network error")
        return items
    }
}
```

---

## 테스트 DI 오버라이드

테스트에서 실제 구현을 Fake로 교체하는 패턴. 상세는 [di-guide.md](di-guide.md) > 테스트 DI 오버라이드 섹션을 참조한다.

```kotlin
// 테스트 전용 Koin 모듈
val testModule = module {
    // 실제 구현을 Fake로 오버라이드
    single<ItemRepository> { FakeItemRepository() }
    single<ItemLocalDataSource> { FakeItemLocalDataSource() }
    single<ItemRemoteDataSource> { FakeItemRemoteDataSource() }
}

// 테스트 설정
@BeforeTest
fun setup() {
    startKoin {
        modules(testModule)
    }
}

@AfterTest
fun teardown() {
    stopKoin()
}
```

---

## 테스트 라이브러리

| 라이브러리 | 용도 | 제공 API |
|-----------|------|---------|
| `kotlinx-coroutines-test` | 코루틴 테스트 | `runTest`, `advanceUntilIdle()`, `TestDispatcher` |
| `app.cash.turbine:turbine` | Flow 테스트 | `.test { awaitItem(); awaitComplete() }` |
| `kotlin.test` | KMP 공통 assertion | `assertEquals`, `assertTrue`, `assertNull`, `assertIs` |

> 버전은 `gradle/libs.versions.toml`에서 관리한다. 새 테스트 라이브러리 추가 시 version catalog에 등록한다.

---

## 코루틴 테스트 규칙

| 규칙 | 설명 |
|------|------|
| `runTest` 필수 | 모든 suspend 함수 테스트에 사용. `runBlocking` 금지 |
| `TestDispatcher` | `runTest`가 자동 제공. 별도 생성 불필요 |
| `advanceUntilIdle()` | 모든 대기 중인 코루틴이 완료될 때까지 진행 |
| `Turbine` | Flow 테스트 시 `.test { awaitItem(); awaitComplete() }` 패턴 사용 |

```kotlin
// DO
@Test
fun `데이터 로드 후 상태 갱신`() = runTest {
    viewModel.onAction(ItemAction.Load)
    advanceUntilIdle()
    assertEquals(expected, viewModel.uiState.value)
}

// DON'T -- delay로 타이밍 맞추기
@Test
fun `데이터 로드 후 상태 갱신`() = runBlocking {
    viewModel.onAction(ItemAction.Load)
    delay(1000)  // 금지: 불안정하고 느림
    assertEquals(expected, viewModel.uiState.value)
}
```

---

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 UseCase 추가 | **테스트 먼저** 작성 (TDD). Fake Repository 사용 |
| 새 ViewModel 추가 | **테스트 먼저** 작성 (TDD). Fake Repository/UseCase 사용 |
| 새 Mapper 추가 | **테스트 먼저** 작성 (TDD). 각 변환 방향별 입출력 검증 |
| Repository 구현 추가 | **테스트 먼저** 작성 (TDD). Fake DataSource 사용 |
| 새 DataSource 추가 | Fake 구현체도 함께 작성 |
| 버그 수정 | 버그를 재현하는 **테스트를 먼저 작성** -> 수정 -> 테스트 통과 확인 |
| 기존 로직 변경 | 기존 테스트가 깨지지 않는지 확인. 필요하면 테스트도 갱신 |

---

## 변경 시 체크리스트

- [ ] TDD 사이클을 따랐는가? (테스트 먼저 -> 구현 -> 리팩터)
- [ ] 새로 추가한 로직에 대응하는 테스트가 있는가?
- [ ] Fake를 사용하고 있는가? (Mock보다 Fake 선호)
- [ ] `runTest`를 사용하고 있는가? (`runBlocking` 아님)
- [ ] 테스트 메서드 이름이 `주어진 조건_동작_기대 결과` 형식인가?
- [ ] Flow 테스트에 Turbine을 사용하고 있는가?
- [ ] 테스트가 외부 의존성(네트워크, DB) 없이 독립적으로 실행되는가?
- [ ] Mapper 테스트에서 null/경계값 케이스를 포함하는가?
- [ ] 새 DataSource에 대응하는 Fake 구현체가 있는가?
