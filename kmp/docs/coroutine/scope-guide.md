# Coroutine Scope Guide

> 스코프 선택, Job 계층, coroutineScope vs supervisorScope, Dispatcher 전환 규칙이다.

---

## 규칙 1: ViewModel에서는 viewModelScope만 사용한다

```kotlin
// DO
class FooViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch {
            val result = repository.getFoo().first()
            _uiState.value = result.toUiState()
        }
    }
}

// DON'T — 자체 스코프 생성 금지
class FooViewModel : ViewModel() {
    private val scope = CoroutineScope(Job())   // 금지: 누가 취소하는가?
    fun load() {
        scope.launch { /* ... */ }              // 누수 위험
    }
}
```

**이유**: `viewModelScope`는 ViewModel의 `onCleared()`에서 자동 취소된다. 자체 스코프를 만들면 취소 책임이 불명확해지고, 화면 이탈 후에도 작업이 계속 실행된다.

---

## 규칙 2: Repository는 flow { } 로 반환한다

```kotlin
// DO — 수집자의 스코프에 자동 연결
override fun getFoos(): Flow<Result<List<Foo>>> = flow {
    val response = withContext(Dispatchers.IO) {
        fooService.getFoos()
    }
    emit(Result.success(response.toDomain()))
}

// DON'T — Repository 내부에서 launch
override fun getFoos(): Flow<...> = flow {
    CoroutineScope(Job()).launch { /* ... */ }   // 금지: 고아 코루틴
}

// DON'T — Repository가 suspend fun으로 직접 반환하면서 내부 launch
suspend fun sync() {
    CoroutineScope(Job()).launch { upload() }    // 금지: fire-and-forget
}
```

**이유**: `flow { }` 빌더는 수집자(collect 호출자)의 스코프에 연결된다. ViewModel의 `viewModelScope`에서 collect하면, ViewModel 소멸 시 flow도 자동 취소된다.

---

## 규칙 3: coroutineScope — 병렬 작업, 전부 성공 필요

하나가 실패하면 나머지도 취소된다 (all-or-nothing).

```kotlin
// DO — 두 작업이 모두 필요한 경우
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user = async { userRepo.getUser() }
    val stats = async { statsRepo.getStats() }
    Dashboard(user.await(), stats.await())
}
// user 실패 → stats 자동 취소 → coroutineScope이 예외를 던짐
```

```kotlin
// DON'T — 최상위에서 async 사용 (예외가 전파되지 않음)
suspend fun loadDashboard(): Dashboard {
    val user = viewModelScope.async { userRepo.getUser() }   // 금지
    val stats = viewModelScope.async { statsRepo.getStats() } // 금지
    return Dashboard(user.await(), stats.await())
    // await()에서 예외 발생 시 다른 async는 취소되지 않음
}
```

---

## 규칙 4: supervisorScope — 병렬 작업, 부분 실패 허용

하나가 실패해도 나머지는 계속 실행된다.

```kotlin
// DO — 일부 실패해도 나머지 결과는 사용
suspend fun loadWidgets(): List<Widget> = supervisorScope {
    val weather = async { weatherRepo.get() }
    val news = async { newsRepo.get() }
    val calendar = async { calendarRepo.get() }

    buildList {
        runCatching { weather.await() }
            .onSuccess { add(it.toWidget()) }
        runCatching { news.await() }
            .onSuccess { add(it.toWidget()) }
        runCatching { calendar.await() }
            .onSuccess { add(it.toWidget()) }
    }
}
```

### coroutineScope vs supervisorScope

| | `coroutineScope` | `supervisorScope` |
|--|-----------------|-------------------|
| 자식 실패 시 | **모든 형제 취소** | 실패한 자식만 취소, 나머지 계속 |
| 용도 | 모든 결과가 필요할 때 | 부분 실패 허용할 때 |
| 예외 전파 | 자식 → 부모 → 형제 전파 | 자식 예외가 부모로 전파되지 않음 |

---

## 규칙 5: withContext — Dispatcher 전환

스레드를 전환하되 현재 스코프의 구조적 동시성을 유지한다.

```kotlin
// DO — IO 작업은 호출받는 쪽에서 Dispatcher 전환
class UserRepositoryImpl(
    private val userService: UserService,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : UserRepository {
    override suspend fun getUser(id: Long): User =
        withContext(ioDispatcher) {
            userService.fetchUser(id).toDomain()
        }
}

// ViewModel은 Dispatcher를 몰라도 된다
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    fun load(id: Long) {
        viewModelScope.launch {           // Main dispatcher
            val user = repo.getUser(id)   // 내부에서 IO로 전환
            _uiState.value = user.toUi()  // 다시 Main
        }
    }
}
```

### withContext vs launch(Dispatcher)

| | `withContext` | `launch(Dispatcher)` |
|--|-------------|---------------------|
| 용도 | 스레드 전환 후 결과 반환 | 새 코루틴 시작 (결과 반환 안 함) |
| 순차/동시 | **순차** (suspend, 결과 기다림) | **동시** (새 코루틴) |
| 일반 사용 | Repository/DataSource 내부 | 거의 사용하지 않음 |

---

## 규칙 6: 앱 전역 스코프 (DI 관리)

앱 전체 수명 동안 필요한 작업(동기화, 로깅 등)에만 사용한다. 반드시 DI로 주입하고 생명주기를 관리한다.

```kotlin
// DO — DI에서 관리되는 앱 스코프
class AppModule {
    val appScope = CoroutineScope(
        SupervisorJob() + Dispatchers.Main.immediate
    )
}

// DO — 사용처에서 DI 주입
class SyncManager(private val appScope: CoroutineScope) {
    fun scheduleSync() {
        appScope.launch {
            // 앱 전체 수명 동안 유지되는 작업
        }
    }
}

// DON'T — GlobalScope
GlobalScope.launch { syncData() }   // 금지: 테스트 불가, 취소 불가
```

| 규칙 | 설명 |
|------|------|
| `SupervisorJob()` 사용 | 자식 하나 실패해도 다른 자식에 영향 없음 |
| DI로 주입 | 테스트에서 교체 가능, 생명주기 명시적 관리 |
| `GlobalScope` 금지 | 취소 불가, 테스트 불가, 누수 원인 |

---

## 에이전트 체크리스트

- [ ] ViewModel에서 `CoroutineScope(Job())`을 사용하지 않는가?
- [ ] Repository에서 `launch { }` 대신 `flow { emit() }`를 사용하는가?
- [ ] 병렬 작업에 `coroutineScope { async { } }`를 사용하는가?
- [ ] 부분 실패 허용 시 `supervisorScope`를 사용하는가?
- [ ] IO 작업에 `withContext(Dispatchers.IO)`를 사용하는가?
- [ ] Dispatcher 전환 책임이 호출받는 쪽(Repository/DataSource)에 있는가?
- [ ] 앱 전역 스코프가 `SupervisorJob()` + DI 관리인가?
- [ ] `GlobalScope` 사용이 없는가?
