# Git Commit Convention

## 커밋 메시지 형식

```
<type>(<scope>): <subject>

<body>
```

## Type

| Type | 설명 |
|---|---|
| feat | 새로운 기능 추가 |
| fix | 버그 수정 |
| refactor | 코드 리팩토링 (기능 변경 없음) |
| style | 코드 포맷팅, 세미콜론 누락 등 (기능 변경 없음) |
| docs | 문서 수정 |
| test | 테스트 코드 추가/수정 |
| chore | 빌드, 설정 파일 변경 (프로덕션 코드 변경 없음) |
| perf | 성능 개선 |

## Scope (선택)

변경 영역을 명시한다. 페이지 도메인명 또는 shared를 사용한다.

```
feat(auth): 로그인 폼 유효성 검사 추가
fix(shared): API 인터셉터 토큰 갱신 오류 수정
refactor(home): 대시보드 컴포넌트 분리
```

## Subject 규칙

| 규칙 | 설명 |
|---|---|
| 언어 | 한글로 작성 |
| 길이 | 50자 이내 |
| 마침표 | 붙이지 않음 |
| 어조 | 명령형 ("추가", "수정", "제거") |

```
# Good
feat(auth): 로그인 폼 유효성 검사 추가
fix(shared): API 인터셉터 토큰 갱신 오류 수정

# Bad
feat(auth): 로그인 폼 유효성 검사를 추가했습니다.
fix: 버그 수정
```

## Body (선택)

- 변경 이유와 이전 동작과의 차이를 설명한다.
- 72자마다 줄바꿈한다.
