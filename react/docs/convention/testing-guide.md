# Testing Guide

## 테스트 도구

| 도구 | 역할 |
|---|---|
| Vitest | 테스트 러너 + 단언(assertion) |
| React Testing Library | 컴포넌트 렌더링 + DOM 쿼리 |
| MSW | API 모킹 |

## 테스트 파일 위치

테스트 대상 파일과 같은 디렉토리에 `.test.ts(x)` 파일을 생성한다.

| 대상 파일 | 테스트 파일 |
|---|---|
| `pages/{domain}/components/ItemCard.tsx` | `pages/{domain}/components/ItemCard.test.tsx` |
| `pages/{domain}/hooks/useItems.ts` | `pages/{domain}/hooks/useItems.test.ts` |
| `pages/{domain}/utils/validation.ts` | `pages/{domain}/utils/validation.test.ts` |
| `shared/utils/formatDate.ts` | `shared/utils/formatDate.test.ts` |

## 컴포넌트 테스트

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ItemCard } from './ItemCard';

const mockItem = {
  id: '1',
  name: '테스트 항목',
  price: 10000,
  status: 'ACTIVE' as const,
};

describe('ItemCard', () => {
  it('이름과 가격을 표시한다', () => {
    render(<ItemCard item={mockItem} />);

    expect(screen.getByText('테스트 항목')).toBeInTheDocument();
    expect(screen.getByText('10,000원')).toBeInTheDocument();
  });

  it('수정 버튼 클릭 시 onEdit을 호출한다', async () => {
    const handleEdit = vi.fn();
    render(<ItemCard item={mockItem} onEdit={handleEdit} />);

    await userEvent.click(screen.getByRole('button', { name: '수정' }));

    expect(handleEdit).toHaveBeenCalledWith('1');
  });
});
```

## 커스텀 훅 테스트

TanStack Query 훅은 `QueryClientProvider`로 감싸서 테스트한다.

```tsx
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return ({ children }: React.PropsWithChildren) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('useItems', () => {
  it('목록을 반환한다', async () => {
    const { result } = renderHook(
      () => useItems({ page: 1, size: 20 }),
      { wrapper: createWrapper() },
    );

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data?.items).toHaveLength(2);
  });
});
```

## 유틸 함수 테스트

```typescript
import { formatPrice } from './formatPrice';

describe('formatPrice', () => {
  it('숫자를 원화 형식으로 변환한다', () => {
    expect(formatPrice(10000)).toBe('10,000원');
    expect(formatPrice(0)).toBe('0원');
  });
});
```

## 테스트 네이밍

| 블록 | 규칙 | 예시 |
|---|---|---|
| `describe` | 테스트 대상 (컴포넌트명, 함수명) | `describe('ItemCard', ...)` |
| `it` | 한글로 기대 동작 서술 | `it('이름을 표시한다', ...)` |
