# Hooks Guide

## 훅 위치 규칙

| 훅 종류 | 위치 | 예시 |
|---|---|---|
| 도메인 전용 훅 | `pages/{domain}/hooks/` | `useTableParams.ts` |
| TanStack Query 훅 | `pages/{domain}/api/*Queries.ts` | `itemQueries.ts` |
| 공통 유틸 훅 | `shared/hooks/` | `useDebounce.ts` |

## 커스텀 훅 작성 원칙

### 네이밍

모든 커스텀 훅은 `use` 접두사를 사용한다.

```typescript
// Good
const useItems = () => { ... };
const useAuth = () => { ... };
const useDebounce = (value: string, delay: number) => { ... };

// Bad
const getItems = () => { ... };
const itemHook = () => { ... };
```

### 단일 책임

하나의 훅은 하나의 관심사만 다룬다.

```typescript
// Good - 각각 단일 책임
const useItems = (params: ListParams) => {
  return useQuery({ ... });
};

const useCreateItem = () => {
  return useMutation({ ... });
};

// Bad - 여러 관심사 혼합
const useItemEverything = () => {
  const query = useQuery({ ... });
  const mutation = useMutation({ ... });
  const [filter, setFilter] = useState({ ... });
  // ...
};
```

## 자주 사용하는 커스텀 훅 패턴

### useDebounce

```typescript
const useDebounce = <T>(value: T, delay: number): T => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
};
```

### useLocalStorage

```typescript
const useLocalStorage = <T>(key: string, initialValue: T) => {
  const [storedValue, setStoredValue] = useState<T>(() => {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : initialValue;
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
};
```

### useDisclosure (모달/드로어 제어)

```typescript
const useDisclosure = (initialState = false) => {
  const [isOpen, setIsOpen] = useState(initialState);

  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen((prev) => !prev), []);

  return { isOpen, open, close, toggle };
};
```
