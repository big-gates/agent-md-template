# Coroutine Guide

> 코루틴/Flow 코드 작성·수정·리뷰 시 참조할 인덱스. 모든 코루틴 코드는 **구조적 동시성**을 기반으로 작성한다.

---

## 하위 문서

| 문서 | 읽어야 할 때 |
|------|-------------|
| [스코프 가이드](scope-guide.md) | 스코프 선택, Job 계층, coroutineScope vs supervisorScope, Dispatcher |
| [에러 처리 가이드](error-guide.md) | 예외 전파, CancellationException, try-catch, runCatching |
| [Flow 가이드](flow-guide.md) | StateFlow 노출, SOT 구독, stateIn, collect/first |
| [callbackFlow 가이드](callback-flow-guide.md) | callbackFlow, 콜백/리스너 래핑, SSE, 리소스 정리 |

---

## 구조적 동시성 (Structured Concurrency)

모든 코루틴은 반드시 **부모 스코프에 연결**되어야 한다. 부모가 취소되면 자식도 취소되고, 자식이 모두 완료되어야 부모가 완료된다.

| 스코프 | 코루틴 | 취소 시점 |
|--------|--------|----------|
| `viewModelScope` (부모) | `launch { loadUser() }` | ViewModel 소멸 시 자동 취소 |
| `viewModelScope` (부모) | `launch { loadPosts() }` | ViewModel 소멸 시 자동 취소 |

### 3대 원칙

| 원칙 | 설명 |
|------|------|
| **부모-자식 연결** | 모든 코루틴은 부모 Job에 연결된다. 고아 코루틴(`GlobalScope`, `CoroutineScope(Job())`)은 금지한다. |
| **취소 전파** | 부모가 취소되면 모든 자식이 취소된다. 자식이 실패하면 부모와 형제에게 전파된다. |
| **완료 대기** | 부모는 모든 자식이 완료될 때까지 완료되지 않는다. 리소스 누수를 방지한다. |

```kotlin
// DO — 구조적 동시성 유지
viewModelScope.launch {
    val user = async { userRepo.getUser() }
    val posts = async { postRepo.getPosts() }
    _uiState.value = UiState(user.await(), posts.await())
}
// viewModelScope 취소 → 두 async 모두 자동 취소

// DON'T — 구조적 동시성 파괴
val scope = CoroutineScope(Job())   // 고아 스코프: 누가 취소하는가?
scope.launch { userRepo.getUser() } // 누수 위험
```

---

## 스코프 선택 기준

| 위치 | 사용할 스코프 | 이유 |
|------|-------------|------|
| ViewModel | `viewModelScope` | ViewModel 소멸 시 자동 취소 |
| suspend 함수 내 병렬 작업 | `coroutineScope { }` | 부모에 연결, 하나 실패 시 나머지 취소 |
| suspend 함수 내 독립 병렬 작업 | `supervisorScope { }` | 부모에 연결, 하나 실패해도 나머지 계속 |
| Repository 데이터 반환 | `flow { emit() }` | 수집자의 스코프에 연결 |
| 콜백/리스너 래핑 | `callbackFlow { }` | 리소스 정리(`awaitClose`) 보장 |
| Dispatcher 전환 | `withContext(Dispatchers.IO) { }` | 현재 스코프 유지, 스레드만 전환 |
| 앱 전역 싱글턴 | `CoroutineScope(SupervisorJob() + Dispatcher)` | 앱 전체 수명, 자식 실패 격리 |

---

## Dispatcher 규칙

| Dispatcher | 용도 | 예시 |
|-----------|------|------|
| `Main` | UI 갱신, 경량 작업 | StateFlow 업데이트, 네비게이션 |
| `IO` | 네트워크, 디스크, DB | API 호출, 파일 읽기, Room 쿼리 |
| `Default` | CPU 집약 연산 | JSON 파싱, 대량 리스트 정렬, 이미지 처리 |
| `Main.immediate` | Main이지만 재디스패치 없이 즉시 실행 | 이미 Main에 있을 때 불필요한 딜레이 방지 |

```kotlin
// DO — IO 작업은 withContext로 Dispatcher 전환
suspend fun getUsers(): List<User> = withContext(Dispatchers.IO) {
    userService.fetchUsers()
}

// DO — CPU 집약 작업
suspend fun parseJson(raw: String): Data = withContext(Dispatchers.Default) {
    json.decodeFromString(raw)
}

// DON'T — launch에서 Dispatcher 지정 (스코프의 Dispatcher를 무시)
viewModelScope.launch(Dispatchers.IO) {   // 지양
    val users = userRepo.getUsers()
    _uiState.value = users                // IO에서 UI 갱신 → 위험
}
```

**원칙**: Dispatcher 전환 책임은 **호출받는 쪽**(Repository, DataSource)에 있다. ViewModel은 Dispatcher를 신경 쓰지 않는다.

---

## 금지 패턴 — 발견 시 즉시 수정

| 금지 패턴 | 코드 예시 | 대안 |
|----------|----------|------|
| GlobalScope | `GlobalScope.launch { }` | 생명주기에 연결된 스코프 사용 |
| 메인 스레드 runBlocking | `runBlocking { repo.getData().first() }` | `viewModelScope.launch` |
| 고아 스코프 | `CoroutineScope(Job()).launch { }` | `viewModelScope` 또는 DI로 관리되는 스코프 |
| CancellationException 삼킴 | `catch (e: Exception) { log(e) }` | `catch (e: CancellationException) { throw e }` 추가 |
| Repository 내부 launch | `fun getData() { scope.launch { } }` | `flow { emit() }` 반환 |
| launch 내부 Dispatcher.IO + UI 갱신 | `launch(IO) { _state.value = x }` | `launch { withContext(IO) { } }` |
| Job() 직접 사용 | `CoroutineScope(Job())` | `SupervisorJob()` + 생명주기 관리 |
| delay로 동기화 | `delay(1000); checkResult()` | `Flow`, `Channel`, 또는 콜백 패턴 |
| CoroutineExceptionHandler로 에러 처리 | `launch(handler) { repo.getData() }` | `launch` 내부 try-catch. Handler는 전역 로깅 용도로만 허용 |

---

## 에이전트용 빠른 참조

| 상황 | 패턴 |
|------|------|
| ViewModel에서 API 호출 | `viewModelScope.launch { repo.foo().first() }` |
| Repository에서 데이터 반환 | `fun foo(): Flow<T> = flow { emit(...) }` |
| UI에서 상태 구독 | `val state by vm.state.collectAsStateWithLifecycle()` |
| 여러 화면에서 같은 데이터 | DataSource(SOT) → Repository 노출 → 각 ViewModel 독립 구독 |
| 병렬 호출 (전부 성공 필요) | `coroutineScope { async {} + async {} }` |
| 병렬 호출 (부분 실패 허용) | `supervisorScope { async {} + async {} }` |
| IO 작업 | `withContext(Dispatchers.IO) { }` |
| 에러 처리 | `runCatching` + CancellationException 재throw |
| 콜백 → Flow 변환 | `callbackFlow { trySend(); awaitClose { } }` |
