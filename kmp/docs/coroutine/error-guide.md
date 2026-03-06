# Coroutine Error Guide

> 예외 전파, CancellationException 처리, 구조적 동시성에서의 에러 흐름 규칙이다.

---

## 구조적 동시성에서의 예외 전파

| 단계 | 대상 | 동작 |
|------|------|------|
| 1 | `launch (A)` | 예외 발생 |
| 2 | `viewModelScope` (부모) | A의 예외가 부모로 전파 |
| 3 | `launch (B)` (형제) | 부모가 B를 취소 |

| 빌더 | 예외 전파 방식 |
|------|-------------|
| `launch` | 즉시 부모로 전파 → 형제 취소 |
| `async` | `await()` 호출 시 전파 |
| `coroutineScope` | 자식 예외 → 모든 형제 취소 → 스코프가 예외를 던짐 |
| `supervisorScope` | 자식 예외가 부모/형제로 전파되지 않음 |

---

## 규칙 1: CancellationException은 반드시 재throw 한다

`CancellationException`은 코루틴 취소 신호이다. 이를 삼키면 구조적 동시성의 취소 전파가 깨진다.

```kotlin
// DO — 명시적 분리
try {
    suspendFunction()
} catch (e: CancellationException) {
    throw e   // 반드시 재throw
} catch (e: Exception) {
    handleError(e)
}
```

```kotlin
// DO — runCatching 사용 시
runCatching {
    repository.getData().first()
}.onFailure { e ->
    if (e is CancellationException) throw e   // 반드시 재throw
    handleError(e)
}
```

```kotlin
// DON'T — CancellationException이 삼켜짐
try {
    suspendFunction()
} catch (e: Exception) {   // CancellationException도 잡힘!
    log(e)                 // 취소 전파 파괴
}
```

**왜 위험한가**: ViewModel이 소멸되어 `viewModelScope`가 취소되면 `CancellationException`이 발생한다. 이를 삼키면 이미 파괴된 ViewModel에서 작업이 계속 실행되어 누수와 크래시가 발생한다.

---

## 규칙 2: launch 내부의 예외는 try-catch로 처리한다

```kotlin
// DO — launch 내부에서 try-catch
viewModelScope.launch {
    try {
        val data = repository.getData().first()
        _uiState.value = UiState.Success(data)
    } catch (e: CancellationException) {
        throw e
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message)
    }
}
```

```kotlin
// DON'T — CoroutineExceptionHandler로 에러 처리
// Handler는 전역 로깅 용도의 마지막 방어선이지, 정상적인 에러 처리 수단이 아니다
val handler = CoroutineExceptionHandler { _, e -> log(e) }
viewModelScope.launch(handler) {
    repository.getData().first()   // 에러 → UI에 반영 안 됨
}
```

### CoroutineExceptionHandler 허용 범위

| 용도 | 허용? | 설명 |
|------|------|------|
| ViewModel의 에러 처리 | **X** | `launch` 내부 try-catch로 처리. UI 상태에 에러를 반영해야 하므로 Handler로는 불가 |
| 앱 전역 크래시 로깅 | **O** | `Application`/`AppScope`에서 예기치 못한 예외를 로깅하는 최후 방어선으로만 허용 |
| Repository/DataSource | **X** | 에러를 호출자에게 전파해야 함. Handler로 삼키면 안 됨 |

---

## 규칙 3: async의 예외는 await에서 처리한다

```kotlin
// DO — await에서 try-catch
coroutineScope {
    val deferred = async { riskyOperation() }
    try {
        val result = deferred.await()
    } catch (e: CancellationException) {
        throw e
    } catch (e: Exception) {
        handleError(e)
    }
}
```

```kotlin
// 주의 — coroutineScope 내 async 예외는 자동으로 전파됨
coroutineScope {
    val a = async { riskyA() }   // 실패 시 b도 취소됨
    val b = async { riskyB() }
    Pair(a.await(), b.await())   // a 실패 → coroutineScope이 예외를 던짐
}
```

---

## 규칙 4: supervisorScope에서 개별 에러 처리

```kotlin
// DO — 각 async의 결과를 독립적으로 처리
supervisorScope {
    val userDeferred = async { userRepo.getUser() }
    val postsDeferred = async { postRepo.getPosts() }

    val user = runCatching { userDeferred.await() }
        .onFailure { if (it is CancellationException) throw it }
        .getOrNull()

    val posts = runCatching { postsDeferred.await() }
        .onFailure { if (it is CancellationException) throw it }
        .getOrDefault(emptyList())

    _uiState.value = DashboardState(user, posts)
}
```

---

## 규칙 5: withContext의 예외는 호출자에게 전파된다

```kotlin
// withContext 내부 예외는 호출 지점에서 catch 가능
viewModelScope.launch {
    try {
        val user = withContext(Dispatchers.IO) {
            userService.fetchUser()   // IOException 발생 가능
        }
        _uiState.value = UiState.Success(user)
    } catch (e: CancellationException) {
        throw e
    } catch (e: IOException) {
        _uiState.value = UiState.Error("Network error")
    }
}
```

---

## 에러 처리 패턴 선택

| 상황 | 패턴 |
|------|------|
| ViewModel에서 단일 작업 | `launch` + try-catch |
| 병렬 작업, 모두 필요 | `coroutineScope` + 외부 try-catch |
| 병렬 작업, 부분 실패 허용 | `supervisorScope` + 개별 `runCatching` |
| Repository에서 에러 래핑 | `flow { emit(Result.success(...)) }` + catch에서 `Result.failure` |
| suspend fun 에러 래핑 | `Result`/`sealed class`로 반환, 호출자가 분기 |

---

## 에이전트 리뷰 규칙

`catch (e: Exception)` 또는 `catch (e: Throwable)`을 발견하면, 그 위에 `catch (e: CancellationException) { throw e }`가 있는지 확인한다.

---

## 에이전트 체크리스트

- [ ] 모든 `catch (e: Exception)` 위에 `catch (e: CancellationException) { throw e }`가 있는가?
- [ ] `runCatching` + `onFailure`에서 CancellationException을 재throw하는가?
- [ ] `catch (e: Throwable)`을 사용하지 않는가? (사용 시 CancellationException 분리 필수)
- [ ] `launch` 내부에서 try-catch로 에러를 UI에 반영하는가?
- [ ] `CoroutineExceptionHandler`를 주된 에러 처리 수단으로 사용하지 않는가?
- [ ] `supervisorScope`에서 각 `async`의 실패를 개별 처리하는가?
