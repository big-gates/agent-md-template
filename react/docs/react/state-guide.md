# State Management Guide

## 상태 분류

| 분류 | 도구 | 예시 |
|---|---|---|
| 서버 상태 | TanStack Query | API 응답 데이터, 목록, 상세 |
| 클라이언트 전역 상태 | React Context | 인증 정보, 테마, 사이드바 상태 |
| 로컬 UI 상태 | useState | 모달 열림, 폼 입력값, 토글 |
| URL 상태 | URL params / searchParams | 페이지네이션, 필터, 정렬 |

## 상태 선택 기준

| 질문 | 선택 |
|---|---|
| API에서 오는 데이터인가? | **TanStack Query** |
| URL에 반영되어야 하는가? | **URL searchParams** |
| 여러 컴포넌트에서 공유하는가? | **Context** |
| 현재 컴포넌트에서만 쓰는가? | **useState** |

## 서버 상태 (TanStack Query)

API에서 가져오는 모든 데이터는 TanStack Query로 관리한다. `useState`로 API 데이터를 관리하지 않는다.

```typescript
// Good - TanStack Query 사용
const { data: items } = useQuery({
  queryKey: itemKeys.list(params),
  queryFn: () => fetchItems(params),
});

// Bad - useState로 서버 데이터 관리
const [items, setItems] = useState([]);
useEffect(() => {
  fetchItems(params).then(setItems);
}, [params]);
```

자세한 내용은 [TanStack Query Guide](../tanstack-query/query-guide.md)를 참고한다.

## 클라이언트 전역 상태 (Context)

앱 전반에서 공유되는 클라이언트 상태는 React Context를 사용한다.

```tsx
// shared/context/AuthContext.tsx
interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginRequest) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export const AuthProvider = ({ children }: React.PropsWithChildren) => {
  // ...인증 로직
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

## 로컬 UI 상태 (useState)

컴포넌트 내부에서만 사용되는 상태는 `useState`를 사용한다.

```tsx
const ItemFilter = () => {
  const [search, setSearch] = useState('');
  const [isFilterOpen, setIsFilterOpen] = useState(false);

  // ...
};
```

## URL 상태

목록 페이지의 필터, 정렬, 페이지네이션은 URL searchParams로 관리한다. 사용자가 URL을 공유하거나 새로고침해도 상태가 유지된다.

```typescript
const useTableParams = () => {
  const [searchParams, setSearchParams] = useSearchParams();

  const params = {
    page: Number(searchParams.get('page')) || 1,
    size: Number(searchParams.get('size')) || 20,
    search: searchParams.get('search') || '',
    sort: searchParams.get('sort') || 'createdAt',
    order: (searchParams.get('order') as 'asc' | 'desc') || 'desc',
  };

  const setParams = (newParams: Partial<typeof params>) => {
    setSearchParams((prev) => {
      Object.entries(newParams).forEach(([key, value]) => {
        if (value) prev.set(key, String(value));
        else prev.delete(key);
      });
      return prev;
    });
  };

  return { params, setParams };
};
```
