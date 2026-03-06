# Pull Request Template

> **트리거**: Pull Request를 생성할 때 이 템플릿을 따른다.

---

## PR 템플릿

```markdown
## Summary
<!-- 변경 사항 1-3줄 요약 -->

## Changes
<!-- 주요 변경 사항 -->
- [ ] 도메인 모델 추가/변경
- [ ] UseCase 추가/변경
- [ ] Adapter 추가/변경
- [ ] 테스트 추가/변경

## Architecture
<!-- 아키텍처 관련 변경 -->
- 레이어: domain / application / adapter
- 패턴: Hexagonal / DDD / Coroutine

## Test
<!-- 테스트 관련 -->
- [ ] TDD로 테스트 먼저 작성
- [ ] 단위 테스트 통과
- [ ] 통합 테스트 통과
- [ ] runTest로 코루틴 테스트

## Checklist
- [ ] 의존 방향이 안쪽(도메인)을 향하는가?
- [ ] 도메인 레이어에 프레임워크 의존이 없는가?
- [ ] DTO/Entity가 도메인에 유입되지 않았는가?
- [ ] 매퍼로 계층 간 데이터를 변환하는가?
- [ ] 블로킹 호출이 없는가? (runBlocking, GlobalScope 금지)
- [ ] 테스트가 추가/갱신되었는가?

## Related Issues
<!-- Refs: #123 / Closes: #456 -->
```

---

## PR 규칙

| 규칙 | 설명 |
|------|------|
| 제목 70자 이내 | 간결한 제목 |
| 단일 책임 | 하나의 PR = 하나의 기능/수정 |
| 테스트 필수 | 테스트 없는 PR은 리뷰 거부 |
| 아키텍처 체크 | 의존 방향, 레이어 분리 확인 |
| 리뷰어 지정 | 최소 1명 이상 리뷰 |
