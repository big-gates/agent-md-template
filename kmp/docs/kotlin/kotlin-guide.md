# Kotlin Guide

> Kotlin 코드 작성 시 참조할 인덱스. 세부 규칙은 하위 문서를 참조한다.

---

## 하위 문서

| 문서 | 읽어야 할 때 |
|------|-------------|
| [네이밍 가이드](naming-guide.md) | 클래스, 함수, 변수, 패키지, 상수 네이밍 규칙 |
| [타입 가이드](type-guide.md) | Null safety, sealed/enum/value class, data class, 불변성 |
| [이디엄 가이드](idiom-guide.md) | 스코프 함수, 확장 함수, 컬렉션, 시퀀스, 구조 분해 |

---

## 금지 패턴

| 금지 패턴 | 대안 |
|----------|------|
| `!!` (non-null assertion) | `?.`, `?:`, `requireNotNull`, `checkNotNull` |
| 플랫폼 타입 방치 (`fun foo(): String!`) | 명시적 nullability 선언 (`String` 또는 `String?`) |
| `var` 남용 | `val` 기본, 꼭 필요한 경우에만 `var` |
| `when` 에서 `else` 로 sealed 분기 처리 | 모든 분기를 명시 (컴파일 타임 안전성) |
| `it` 중첩 (`list.map { it.filter { it > 0 } }`) | 명시적 파라미터명 (`list.map { item -> item.filter { value -> value > 0 } }`) |
| 빈 `catch` 블록 (`catch (e: Exception) { }`) | 최소한 로깅, 또는 의도 주석 |
| `String` 연결 반복 (`a + b + c`) | 문자열 템플릿 `"$a$b$c"` 또는 `buildString { }` |
| mutable 컬렉션 외부 노출 | `List`(읽기전용) 타입으로 노출 |
| `companion object` 에 비즈니스 로직 | 최상위 함수 또는 별도 클래스 |
| 무의미한 유틸 클래스 (`class StringUtils`) | 최상위 확장 함수 |

---

## 에이전트용 빠른 참조

| 상황 | 패턴 |
|------|------|
| null 기본값 제공 | `val name = user?.name ?: "Unknown"` |
| null 시 조기 리턴 | `val user = getUser() ?: return` |
| 타입 분기 | `when (result) { is Success -> ... is Error -> ... }` |
| 범위 체크 | `value in 1..100` |
| 리스트 변환 | `list.map { it.toUi() }` |
| 조건부 리스트 빌드 | `buildList { add(a); if (cond) add(b) }` |
| 리소스 정리 | `resource.use { it.read() }` |
| 싱글턴 | `object AppConfig { }` |
| 상수 | `private const val MAX_RETRY = 3` |
| 팩토리 함수 | 최상위 함수 `fun FooDto.toDomain(): Foo` |
