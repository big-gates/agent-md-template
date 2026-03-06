# Component Guide

## 컴포넌트 분류

| 분류 | 위치 | 역할 | export 방식 |
|---|---|---|---|
| Page 컴포넌트 | `pages/{domain}/*Page.tsx` | 라우트 진입점. 데이터 패칭과 도메인 컴포넌트 조합 | `export default` |
| 도메인 컴포넌트 | `pages/{domain}/components/` | 특정 도메인 로직을 포함하는 컴포넌트 | named export |
| Shared 컴포넌트 | `shared/components/` | 도메인 비종속 재사용 UI 컴포넌트 | named export |

## 컴포넌트 작성 원칙

### 함수 컴포넌트 사용

모든 컴포넌트는 함수 컴포넌트로 작성한다.

```tsx
// Good
const ItemCard = ({ item }: ItemCardProps) => {
  return <div>{item.name}</div>;
};

// Bad - 클래스 컴포넌트
class ItemCard extends React.Component { ... }
```

### Props 타입 정의

컴포넌트 Props는 `interface`로 정의하고, 컴포넌트명 + `Props` 접미사를 사용한다.

```tsx
interface ItemCardProps {
  item: Item;
  onEdit: (id: string) => void;
  onDelete: (id: string) => void;
}

const ItemCard = ({ item, onEdit, onDelete }: ItemCardProps) => {
  // ...
};
```

### children이 있는 컴포넌트

`React.PropsWithChildren`을 사용한다.

```tsx
interface LayoutProps {
  title: string;
}

const Layout = ({ title, children }: React.PropsWithChildren<LayoutProps>) => {
  return (
    <div>
      <h1>{title}</h1>
      {children}
    </div>
  );
};
```

## Page 컴포넌트 패턴

```tsx
// pages/{domain}/ItemListPage.tsx
const ItemListPage = () => {
  const { data, isLoading } = useItems();

  if (isLoading) return <Loading />;

  return (
    <PageLayout title="목록">
      <ItemFilter />
      <ItemList items={data?.items ?? []} />
      <Pagination total={data?.totalPages ?? 0} />
    </PageLayout>
  );
};

export default ItemListPage;
```

## Shared 컴포넌트 패턴

```tsx
// shared/components/Button.tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
}

const Button = ({ variant = 'primary', size = 'md', loading, children, ...rest }: ButtonProps) => {
  return (
    <button className={`btn btn-${variant} btn-${size}`} disabled={loading} {...rest}>
      {loading ? <Spinner /> : children}
    </button>
  );
};
```

## 조건부 렌더링

| 패턴 | 사용 시점 | 예시 |
|---|---|---|
| 삼항 연산자 | 간단한 분기 | `{isAdmin ? <AdminPanel /> : <UserPanel />}` |
| `&&` 연산자 | 단일 조건 | `{hasPermission && <DeleteButton />}` |
| 얼리 리턴 | 로딩/에러 상태 | `if (isLoading) return <Loading />;` |

## 이벤트 핸들러

`handle` 접두사를 사용한다.

```tsx
const ItemForm = () => {
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // ...
  };

  const handleNameChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // ...
  };

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleNameChange} />
    </form>
  );
};
```
