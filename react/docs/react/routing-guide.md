# Routing Guide

## 라우트 구조

React Router의 `createBrowserRouter`를 사용하여 라우트를 구성한다.

```tsx
// app/router.tsx
import { createBrowserRouter } from 'react-router-dom';
import { lazy } from 'react';

const SomePage = lazy(() => import('@/pages/{domain}/SomePage'));

export const router = createBrowserRouter([
  {
    path: '/login',
    element: <LoginPage />,
  },
  {
    path: '/',
    element: <ProtectedLayout />,
    children: [
      { index: true, element: <DashboardPage /> },
      { path: '{resource}', element: <ResourceListPage /> },
      { path: '{resource}/new', element: <ResourceCreatePage /> },
      { path: '{resource}/:id', element: <ResourceDetailPage /> },
    ],
  },
]);
```

## Lazy Loading

모든 Page 컴포넌트는 `React.lazy`로 코드 스플리팅한다. Suspense fallback으로 로딩 UI를 제공한다.

```tsx
// app/App.tsx
const App = () => {
  return (
    <Suspense fallback={<FullPageLoading />}>
      <RouterProvider router={router} />
    </Suspense>
  );
};
```

## Protected Route

인증이 필요한 라우트는 `ProtectedLayout`으로 감싼다.

```tsx
const ProtectedLayout = () => {
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  return (
    <AdminLayout>
      <Outlet />
    </AdminLayout>
  );
};
```

## 라우트 경로 상수

라우트 경로는 상수로 관리한다.

```typescript
// shared/constants/routes.ts
export const ROUTES = {
  LOGIN: '/login',
  HOME: '/',
  RESOURCE_LIST: '/resources',
  RESOURCE_NEW: '/resources/new',
  RESOURCE_DETAIL: (id: string) => `/resources/${id}`,
} as const;
```

사용:

```tsx
import { ROUTES } from '@/shared/constants/routes';

navigate(ROUTES.RESOURCE_DETAIL(item.id));
<Link to={ROUTES.RESOURCE_LIST}>목록</Link>
```

## 페이지 모듈 구조

| 경로 | 역할 |
|---|---|
| `pages/{domain}/components/` | 도메인 전용 컴포넌트 |
| `pages/{domain}/hooks/` | 도메인 전용 훅 |
| `pages/{domain}/api/` | API 함수 + Query 훅 |
| `pages/{domain}/types/` | 도메인 전용 타입 |
| `pages/{domain}/utils/` | 도메인 전용 유틸 (선택) |
| `pages/{domain}/*Page.tsx` | 라우트 진입점 |
| `pages/{domain}/index.ts` | 배럴 파일 |

## 페이지 파일 규칙

| 규칙 | 설명 |
|---|---|
| 파일명 | `*Page.tsx`로 끝난다 |
| export | `export default`를 사용한다 (lazy loading 대응) |
| 도메인 포함 | 각 페이지 디렉토리는 components, hooks, api, types를 자체 포함한다 |
