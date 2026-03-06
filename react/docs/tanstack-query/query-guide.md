# TanStack Query Guide

## QueryClient 설정

```tsx
// app/providers.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,       // 5분
      gcTime: 1000 * 60 * 30,          // 30분
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

export const Providers = ({ children }: React.PropsWithChildren) => {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
};
```

## Query Key 관리

Key Factory 패턴으로 관리한다. 자세한 내용은 [key-guide.md](./key-guide.md) 참조.

```typescript
// Good
useQuery({ queryKey: itemKeys.list(params), ... });

// Bad
useQuery({ queryKey: ['items', 'list', params], ... });
```

## useQuery 사용

### 기본 패턴

```typescript
// pages/{domain}/api/{domain}Queries.ts
export const useItems = (params: ListParams) => {
  return useQuery({
    queryKey: itemKeys.list(params),
    queryFn: () => fetchItems(params),
  });
};

export const useItemDetail = (id: string) => {
  return useQuery({
    queryKey: itemKeys.detail(id),
    queryFn: () => fetchItemDetail(id),
    enabled: !!id,
  });
};
```

### 주요 옵션

| 옵션 | 용도 | 예시 |
|---|---|---|
| `enabled` | 조건부 쿼리 실행 | `enabled: !!id` |
| `select` | 응답 데이터 변환 | `select: (data) => data.items.map(i => i.name)` |
| `staleTime` | 데이터 신선도 유지 시간 | `staleTime: 1000 * 60 * 10` |

### 로딩/에러 상태

| 상태 | 설명 | 사용 |
|---|---|---|
| `isLoading` | 최초 로딩 (데이터 없음) | 스켈레톤 UI 표시 |
| `isFetching` | 백그라운드 리패치 포함 | 리패치 인디케이터 표시 |
| `isError` | 요청 실패 | 에러 메시지 표시 |

## API 함수 분리

API 호출 함수와 Query 훅은 별도 파일로 분리한다.

| 파일 | 역할 | 내용 |
|---|---|---|
| `pages/{domain}/api/*Api.ts` | 순수 API 호출 | axios 호출, 응답 반환 |
| `pages/{domain}/api/*Queries.ts` | TanStack Query 훅 | useQuery, useMutation 래핑 |

```typescript
// pages/{domain}/api/itemApi.ts
export const fetchItems = async (params: ListParams): Promise<ListResponse> => {
  const { data } = await apiClient.get('/items', { params });
  return data;
};

// pages/{domain}/api/itemQueries.ts
export const useItems = (params: ListParams) => {
  return useQuery({
    queryKey: itemKeys.list(params),
    queryFn: () => fetchItems(params),
  });
};
```
