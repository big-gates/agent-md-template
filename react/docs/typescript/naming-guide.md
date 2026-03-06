# Naming Convention Guide

## 종류별 네이밍 규칙

| 대상 | 스타일 | 예시 |
|---|---|---|
| 컴포넌트 | PascalCase | `ItemCard`, `LoginForm` |
| 함수 / 변수 | camelCase | `formatDate`, `isLoading` |
| 커스텀 훅 | camelCase (use 접두사) | `useItems`, `useAuth` |
| 상수 | SCREAMING_SNAKE_CASE | `API_BASE_URL`, `MAX_FILE_SIZE` |
| 타입 / 인터페이스 | PascalCase | `Item`, `LoginRequest` |
| enum 대체 유니온 | SCREAMING_SNAKE_CASE 값 | `'ACTIVE' \| 'INACTIVE'` |
| 이벤트 핸들러 | handle 접두사 | `handleSubmit`, `handleClick` |
| boolean 변수 | is/has/can/should 접두사 | `isOpen`, `hasPermission` |

## 파일 네이밍

| 대상 | 스타일 | 예시 |
|---|---|---|
| 컴포넌트 파일 | PascalCase.tsx | `ItemCard.tsx` |
| 훅 파일 | camelCase.ts | `useItems.ts` |
| 유틸 파일 | camelCase.ts | `formatDate.ts` |
| 타입 파일 | camelCase.types.ts | `item.types.ts` |
| 테스트 파일 | 대상.test.tsx | `ItemCard.test.tsx` |
| 스타일 파일 | 컴포넌트.module.css | `ItemCard.module.css` |
| 상수 파일 | camelCase.ts | `routes.ts` |
| API 파일 | 도메인Api.ts | `itemApi.ts` |
| Query 파일 | 도메인Queries.ts | `itemQueries.ts` |

## 디렉토리 네이밍

디렉토리는 kebab-case를 사용한다.

| 예시 | 설명 |
|---|---|
| `pages/auth/` | 단일 단어 |
| `pages/manage-item/` | 복합 단어 |
| `shared/components/` | 공통 모듈 하위 |

## Props 네이밍

| 용도 | 접두사/패턴 | 예시 |
|---|---|---|
| 콜백 Props | `on` 접두사 | `onEdit`, `onClick`, `onStatusChange` |
| 렌더링 제어 Props | 형용사 / `is` 접두사 | `isOpen`, `closable`, `fullWidth` |

```typescript
interface ItemCardProps {
  onEdit: (id: string) => void;
  onClick: () => void;
  onStatusChange: (status: Status) => void;
}

interface ModalProps {
  isOpen: boolean;
  closable: boolean;
  fullWidth: boolean;
}
```

## Query Key 네이밍

```typescript
// {domain}Keys 패턴
const itemKeys = { ... };
const authKeys = { ... };
```
