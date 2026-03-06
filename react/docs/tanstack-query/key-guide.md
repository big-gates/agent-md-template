# Query Key Management Guide

## Key Factory 패턴

모든 도메인은 Query Key Factory를 정의한다. 문자열 직접 사용을 금지한다.

```typescript
// pages/{domain}/api/{domain}Queries.ts
export const {domain}Keys = {
  all: ['{domain}'] as const,
  lists: () => [...{domain}Keys.all, 'list'] as const,
  list: (params: ListParams) => [...{domain}Keys.lists(), params] as const,
  details: () => [...{domain}Keys.all, 'detail'] as const,
  detail: (id: string) => [...{domain}Keys.details(), id] as const,
};
```

## Key 계층 구조

| Key | 팩토리 메서드 | 용도 |
|---|---|---|
| `['{domain}']` | `all` | 도메인 전체 무효화 |
| `['{domain}', 'list']` | `lists()` | 모든 목록 무효화 |
| `['{domain}', 'list', { page, size, ... }]` | `list(params)` | 특정 파라미터 목록 |
| `['{domain}', 'detail']` | `details()` | 모든 상세 무효화 |
| `['{domain}', 'detail', 'abc123']` | `detail(id)` | 특정 상세 |

## 무효화 범위

| 범위 | 코드 | 설명 |
|---|---|---|
| 도메인 전체 | `invalidateQueries({ queryKey: keys.all })` | 목록 + 상세 전부 |
| 모든 목록 | `invalidateQueries({ queryKey: keys.lists() })` | 모든 파라미터 목록 |
| 특정 상세 | `invalidateQueries({ queryKey: keys.detail(id) })` | 단일 항목 |

## 규칙 요약

| 규칙 | 설명 |
|---|---|
| Key Factory 필수 | 문자열 직접 사용 금지 |
| 파일 위치 | 해당 도메인의 `*Queries.ts` 파일에 정의 |
| `as const` 필수 | 타입 안전성 확보 |
| 최소 범위 무효화 | 불필요한 리패치 방지 |
