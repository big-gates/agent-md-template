# Dependency Injection Guide

> **트리거**: DI 모듈을 추가·수정할 때, 새 의존성을 등록할 때, 또는 ViewModel/Repository/UseCase를 DI에 연결할 때 이 문서를 참조한다.

---

## DI 프레임워크

KMP 프로젝트에서는 **Koin**을 사용한다. Koin은 Kotlin Multiplatform을 네이티브로 지원하며, 어노테이션 프로세서 없이 DSL로 의존성을 정의한다.

---

## 모듈 구조

| 경로 | 설명 |
|------|------|
| `shared/src/commonMain/kotlin/{package}/di/AppModule.kt` | 최상위 모듈 (모든 하위 모듈 포함) |
| `shared/src/commonMain/kotlin/{package}/di/DomainModule.kt` | UseCase 등록 |
| `shared/src/commonMain/kotlin/{package}/di/DataModule.kt` | Repository 구현체 바인딩 + Mapper 등록 |
| `shared/src/commonMain/kotlin/{package}/di/DataSourceModule.kt` | DataSource + DAO + API Service 등록 |
| `shared/src/commonMain/kotlin/{package}/di/ViewModelModule.kt` | ViewModel 등록 |

> 이전의 `NetworkModule`, `DatabaseModule`은 `DataSourceModule`에 통합되었다. 인프라(DB, 네트워크)는 DataSource의 구현 상세이므로 하나의 모듈에서 관리한다.

---

## 모듈 정의 패턴

### Repository 바인딩 (인터페이스 -> 구현체)

```kotlin
val dataModule = module {
    // DO -- 인터페이스에 구현체를 바인딩
    singleOf(::ItemRepositoryImpl) bind ItemRepository::class
    singleOf(::UserRepositoryImpl) bind UserRepository::class
}
```

### UseCase 등록

```kotlin
val domainModule = module {
    // DO -- UseCase는 factory로 등록 (상태 없음, 매번 새 인스턴스)
    factoryOf(::GetActiveItemsUseCase)
    factoryOf(::SyncItemsUseCase)
}
```

### ViewModel 등록

```kotlin
val viewModelModule = module {
    // DO -- KMP Koin ViewModel 등록
    viewModelOf(::HomeViewModel)
    viewModelOf(::SettingsViewModel)
}
```

### DataSource 통합 등록

```kotlin
val dataSourceModule = module {
    // -- 인프라: DB --
    single { createDatabase() }                        // 플랫폼별 expect/actual
    single { get<AppDatabase>().itemDao() }
    single { get<AppDatabase>().userDao() }

    // -- 인프라: Network --
    single { createHttpClient() }                      // HttpClient 싱글턴
    singleOf(::ItemService)

    // -- DataSource --
    singleOf(::ItemLocalDataSourceImpl) bind ItemLocalDataSource::class
    singleOf(::ItemRemoteDataSourceImpl) bind ItemRemoteDataSource::class
    singleOf(::TokenCacheDataSource)
}
```

---

## 스코프 선택 기준

| 스코프 | 용도 | 예시 |
|--------|------|------|
| `single` / `singleOf` | 앱 전체에서 하나의 인스턴스 | Repository, DataSource, HttpClient, Database |
| `factory` / `factoryOf` | 매번 새 인스턴스 생성 | UseCase (상태 없음) |
| `viewModelOf` | ViewModel 생명주기 관리 | ViewModel |

```kotlin
// DO -- 상태 없는 UseCase는 factory
factoryOf(::GetItemsUseCase)

// DO -- 상태 있는 Repository는 single
singleOf(::ItemRepositoryImpl) bind ItemRepository::class

// DON'T -- UseCase를 single로 (불필요한 메모리 유지)
singleOf(::GetItemsUseCase)  // 지양

// DON'T -- Repository를 factory로 (매번 새 인스턴스 -> 캐시 무효화)
factoryOf(::ItemRepositoryImpl)  // 금지
```

---

## 최상위 모듈 조합

```kotlin
val appModule = module {
    includes(
        // -- core 모듈 (하위 → 상위 순서) --
        dataSourceModule,    // 인프라 + DataSource (가장 먼저)
        dataModule,          // Repository 구현
        domainModule,        // UseCase

        // -- feature 모듈 (각 feature의 DI 모듈) --
        // {name}ViewModelModule,  // feature:{name}
    )
}
```

### feature DI 모듈 통합 규칙

각 feature 모듈은 자체 DI 모듈 파일을 가지며, `shared`의 `appModule`에서 `includes`로 통합한다.

| feature | DI 모듈 파일 | 모듈 변수명 |
|---------|-------------|------------|
| `feature:{name}` | `{name}/di/{Name}ViewModelModule.kt` | `{name}ViewModelModule` |

| 규칙 | 설명 |
|------|------|
| feature DI 모듈은 ViewModel만 등록 | Repository, DataSource 등은 core 모듈에서 등록 |
| `viewModelOf()` 사용 | feature ViewModel은 Koin `viewModelOf()`로 등록 |
| 새 feature 추가 시 | feature DI 모듈 생성 → `appModule`의 `includes`에 추가 |
| core 의존성은 자동 해결 | core 모듈이 먼저 등록되므로 feature ViewModel의 생성자 의존성은 Koin이 자동 주입 |

---

## 플랫폼별 초기화

### Android

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApplication)
            modules(appModule, androidPlatformModule)
        }
    }
}
```

### iOS

```kotlin
fun initKoin() {
    startKoin {
        modules(appModule, iosPlatformModule)
    }
}
```

### 플랫폼 모듈 (expect/actual)

```kotlin
// commonMain
expect val platformModule: Module

// androidMain
actual val platformModule = module {
    single { AndroidDatabaseBuilder(get()) }
}

// iosMain
actual val platformModule = module {
    single { IosDatabaseBuilder() }
}
```

---

## 의존성 방향과 DI

| DI 모듈 | 의존하는 모듈 | 등록 순서 (선행 필요) |
|---------|-------------|---------------------|
| `DataSourceModule` | 없음 (최하위) | 1번째 |
| `DataModule` | `DataSourceModule` | 2번째 |
| `DomainModule` | `DataModule` (간접) | 3번째 |
| `ViewModelModule` | `DomainModule`, `DataModule` | 4번째 (최상위) |

DI 모듈 정의 순서는 의존 방향과 일치해야 한다. 하위 모듈이 먼저 등록되어야 상위 모듈에서 `get()`으로 참조할 수 있다.

---

## 테스트 DI 오버라이드

테스트에서 실제 구현을 Fake로 교체하여 격리된 테스트를 수행한다.

### 단위 테스트 (DI 없이)

단위 테스트에서는 Koin을 사용하지 않고 생성자 주입으로 Fake를 직접 전달한다.

```kotlin
// DO -- 생성자 주입으로 Fake 전달 (가장 선호)
class ItemViewModelTest {
    private val fakeRepository = FakeItemRepository()
    private val viewModel = ItemViewModel(fakeRepository)

    @Test
    fun `아이템 로드 성공`() = runTest { ... }
}
```

### 통합 테스트 (Koin 오버라이드)

통합 테스트에서 Koin 모듈을 오버라이드하여 Fake를 주입한다.

```kotlin
// 테스트 전용 모듈
val testDataSourceModule = module {
    single<ItemLocalDataSource> { FakeItemLocalDataSource() }
    single<ItemRemoteDataSource> { FakeItemRemoteDataSource() }
}

val testAppModule = module {
    includes(
        testDataSourceModule,   // Fake DataSource
        dataModule,             // 실제 Repository (DataSource만 Fake)
        domainModule,
        viewModelModule,
    )
}

class IntegrationTest : KoinTest {
    @BeforeTest
    fun setup() {
        startKoin { modules(testAppModule) }
    }

    @AfterTest
    fun teardown() {
        stopKoin()
    }

    @Test
    fun `Repository가 Fake DataSource를 통해 정상 동작`() = runTest {
        val repository: ItemRepository = get()
        // ...
    }
}
```

### 순환 의존 방지

| 규칙 | 설명 |
|------|------|
| 단방향 의존만 허용 | `DataSourceModule -> DataModule -> DomainModule -> ViewModelModule` |
| 같은 레이어 내 상호 참조 금지 | Repository A가 Repository B를 직접 참조하지 않음 |
| 순환 감지 시 | UseCase로 조합 레이어를 추가하여 의존 방향을 단일화 |

---

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 Repository 추가 | `DataModule`에 `singleOf(::FooRepositoryImpl) bind FooRepository::class` 추가 |
| 새 UseCase 추가 | `DomainModule`에 `factoryOf(::FooUseCase)` 추가 |
| 새 ViewModel 추가 | `ViewModelModule`에 `viewModelOf(::FooViewModel)` 추가 |
| 새 DataSource 추가 | `DataSourceModule`에 `singleOf(::FooLocalDataSourceImpl) bind FooLocalDataSource::class` 추가 |
| 새 DAO 추가 | `DataSourceModule`에 `single { get<AppDatabase>().fooDao() }` 추가 |
| 새 API Service 추가 | `DataSourceModule`에 `singleOf(::FooService)` 추가 |
| 플랫폼 전용 의존성 | `platformModule`의 `actual`에 추가 |
| 테스트 Fake 필요 | 단위 테스트: 생성자 주입. 통합 테스트: testModule에 오버라이드 |

---

## DO / DON'T

```kotlin
// DO -- 인터페이스에 바인딩하여 의존성 역전
singleOf(::ItemRepositoryImpl) bind ItemRepository::class

// DO -- ViewModel에서 생성자 주입
class HomeViewModel(
    private val getItemsUseCase: GetActiveItemsUseCase,
    private val itemRepository: ItemRepository,
) : ViewModel()

// DO -- Screen에서 koinInject로 ViewModel 주입
@Composable
fun HomeScreen(viewModel: HomeViewModel = koinInject()) { }

// DON'T -- 서비스 로케이터 패턴 (직접 get() 호출)
class HomeViewModel : ViewModel() {
    private val repo = KoinJavaComponent.get<ItemRepository>()  // 금지
}

// DON'T -- Composable 하위 컴포넌트에서 직접 DI 주입
@Composable
fun ItemCard(viewModel: HomeViewModel = koinInject()) { }  // 금지: Screen에서만 주입
```

---

## 변경 시 체크리스트

- [ ] 새 클래스를 적절한 DI 모듈에 등록했는가?
- [ ] Repository는 `single`로, UseCase는 `factory`로 등록했는가?
- [ ] 인터페이스와 구현체를 `bind`로 연결했는가?
- [ ] DataSource/DAO/Service를 `DataSourceModule`에 등록했는가?
- [ ] 플랫폼 전용 의존성은 `platformModule`에 분리했는가?
- [ ] ViewModel은 Screen 진입점에서만 주입하는가?
- [ ] 순환 의존이 발생하지 않는가?
- [ ] 새 인터페이스에 대응하는 Fake 구현체가 테스트에 준비되어 있는가?
