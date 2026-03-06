# Git Commit Template

> **트리거**: 커밋 메시지를 작성할 때 이 규칙을 따른다.

---

## 커밋 메시지 형식

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 예시

```
feat(order): 주문 생성 UseCase 구현

- CreateOrderUseCase 인터페이스 및 OrderService 구현
- TDD로 단위 테스트 먼저 작성 후 구현
- FakeOrderRepository를 활용한 테스트

Refs: #123
```

---

## Type

| Type | 설명 |
|------|------|
| `feat` | 새로운 기능 |
| `fix` | 버그 수정 |
| `refactor` | 리팩터링 (기능 변경 없음) |
| `test` | 테스트 추가/수정 |
| `docs` | 문서 변경 |
| `chore` | 빌드, 설정 등 기타 |
| `style` | 코드 포맷팅 (로직 변경 없음) |
| `perf` | 성능 개선 |

---

## Scope

| Scope | 대상 |
|-------|------|
| `domain` | 도메인 모델, Domain Service, Domain Event |
| `application` | UseCase, Application Service, Port |
| `adapter` | Handler, Router, Repository Adapter |
| `infra` | 설정, 인프라, 빌드 |
| `test` | 테스트 코드 |
| 도메인명 | `order`, `product`, `user` 등 Bounded Context |

---

## 규칙

| 규칙 | 설명 |
|------|------|
| 제목 50자 이내 | 간결하게 작성 |
| 명령형 | "추가한다" → "추가" |
| 본문은 **왜** | 무엇을 했는지보다 왜 했는지 |
| 이슈 연결 | `Refs: #123`, `Closes: #456` |
| TDD 언급 | 테스트 먼저 작성한 경우 본문에 명시 |
