# Mutation Guide

## useMutation 기본 패턴

```typescript
// pages/{domain}/api/{domain}Queries.ts
export const useCreateItem = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateRequest) => createItem(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: itemKeys.lists() });
    },
  });
};

export const useUpdateItem = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdateRequest }) =>
      updateItem(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: itemKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: itemKeys.lists() });
    },
  });
};

export const useDeleteItem = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => deleteItem(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: itemKeys.lists() });
    },
  });
};
```

## 컴포넌트에서 사용

```tsx
const ItemForm = () => {
  const navigate = useNavigate();
  const createItem = useCreateItem();

  const handleSubmit = (formData: CreateRequest) => {
    createItem.mutate(formData, {
      onSuccess: () => {
        toast.success('등록되었습니다.');
        navigate(ROUTES.ITEM_LIST);
      },
      onError: (error) => {
        toast.error(error.message);
      },
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* ... */}
      <Button type="submit" loading={createItem.isPending}>
        등록
      </Button>
    </form>
  );
};
```

## 캐시 무효화 전략

| 전략 | 메서드 | 사용 시점 |
|---|---|---|
| 무효화 (리패치) | `invalidateQueries` | 생성/수정/삭제 성공 후 |
| 직접 캐시 수정 | `setQueryData` | 낙관적 업데이트 |
| 쿼리 취소 | `cancelQueries` | 낙관적 업데이트 전 충돌 방지 |

### 낙관적 업데이트 패턴

```typescript
export const useToggleStatus = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, status }: { id: string; status: Status }) =>
      updateStatus(id, status),

    onMutate: async ({ id, status }) => {
      await queryClient.cancelQueries({ queryKey: itemKeys.detail(id) });
      const previous = queryClient.getQueryData(itemKeys.detail(id));

      queryClient.setQueryData(itemKeys.detail(id), (old: Item) => ({
        ...old,
        status,
      }));

      return { previous };
    },

    onError: (_, { id }, context) => {
      queryClient.setQueryData(itemKeys.detail(id), context?.previous);
    },

    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: itemKeys.detail(id) });
    },
  });
};
```

## Mutation 상태 활용

| 상태 | 설명 | UI 대응 |
|---|---|---|
| `isPending` | 요청 진행 중 | 버튼 비활성화, 로딩 표시 |
| `isError` | 요청 실패 | 에러 메시지 표시 |
| `isSuccess` | 요청 성공 | 성공 피드백 |
