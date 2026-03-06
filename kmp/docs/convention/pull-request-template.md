# Pull Request Template

아래 템플릿을 복사해서 PR 본문에 사용한다.

```md
## 변경 목적
- 이 PR이 해결하는 문제/배경을 1~2줄로 작성

## 주요 변경
- 핵심 변경 3~5개

## 영향 범위
- 도메인/모듈/문서/운영 영향

## 테스트/검증
- [ ] 단위 테스트
- [ ] 통합 테스트
- [ ] 수동 검증
- 미실행 항목이 있으면 사유 기재

## 문서 동기화
- [ ] `AGENTS.md` 반영
- [ ] `docs/architecture` 반영
- [ ] `docs/convention` 반영
- [ ] 해당 없음 (사유: ...)

## 리스크 및 롤백
- 예상 리스크
- 롤백 방법(또는 되돌리는 커밋/파일)
```

## PR 제목 규칙 (권장)

`<type>: <summary>` 형식을 커밋 메시지와 동일하게 사용한다.

예시:

- `docs: add commit and PR templates for agent consistency`
- `feat: introduce new order cancellation orchestrator`
