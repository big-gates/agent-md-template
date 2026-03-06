# TypeScript Type Guide

## 타입 vs 인터페이스

| 키워드 | 사용 시점 | 예시 |
|---|---|---|
| `interface` | 객체 형태 (Props, API 응답, 도메인 모델) | `interface Item { id: string; }` |
| `type` | 유니온, 인터섹션, 유틸리티 타입 | `type Status = 'ACTIVE' \| 'INACTIVE'` |

```typescript
// interface - 객체 형태
interface Item {
  id: string;
  name: string;
  price: number;
}

// type - 유니온, 유틸리티
type ItemStatus = 'ACTIVE' | 'INACTIVE' | 'DRAFT';
type ItemFormData = Omit<Item, 'id' | 'createdAt' | 'updatedAt'>;
```

## 타입 파일 위치

| 위치 | 역할 | 예시 |
|---|---|---|
| `pages/{domain}/types/*.types.ts` | 도메인 타입 + API 요청/응답 타입 | `item.types.ts` |
| `shared/types/api.types.ts` | 공통 API 타입 (페이지네이션, 에러 등) | `ApiResponse<T>` |
| `shared/types/common.types.ts` | 공통 유틸리티 타입 | `SelectOption<T>` |

## 공통 API 타입

```typescript
// shared/types/api.types.ts
interface ApiResponse<T> {
  data: T;
  message: string;
  status: number;
}

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  size: number;
  totalPages: number;
}

interface ApiError {
  message: string;
  code: string;
  status: number;
}
```

## 타입 작성 규칙

| 규칙 | Good | Bad |
|---|---|---|
| `any` 사용 금지 | `unknown` + 타입 가드 | `any` |
| 열거형은 유니온 타입 | `type Status = 'ACTIVE' \| 'INACTIVE'` | `enum Status { ... }` |
| Non-null assertion 지양 | `user?.name ?? '알 수 없음'` | `user!.name` |

### 제네릭 활용

재사용 가능한 타입은 제네릭으로 작성한다.

```typescript
interface SelectOption<T = string> {
  label: string;
  value: T;
}

const statusOptions: SelectOption<ItemStatus>[] = [
  { label: '활성', value: 'ACTIVE' },
  { label: '비활성', value: 'INACTIVE' },
];
```

### as const 활용

상수 객체는 `as const`로 리터럴 타입을 유지한다.

```typescript
export const STATUS_LABEL = {
  ACTIVE: '활성',
  INACTIVE: '비활성',
  DRAFT: '임시저장',
} as const;

type StatusKey = keyof typeof STATUS_LABEL;
```
