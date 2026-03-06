# Folder Structure Guide

## 루트 디렉토리

| 파일/디렉토리 | 역할 |
|---|---|
| `public/` | 정적 파일 (favicon, robots.txt 등) |
| `src/` | 소스 코드 |
| `docs/` | 프로젝트 문서 |
| `index.html` | Vite 진입 HTML |
| `vite.config.ts` | Vite 설정 |
| `tsconfig.json` | TypeScript 설정 |
| `tsconfig.node.json` | Node 환경 TypeScript 설정 |
| `package.json` | 패키지 매니저 설정 |
| `.env.*` | 환경 변수 |

## 파일 네이밍 규칙

| 대상 | 패턴 | 예시 |
|---|---|---|
| 페이지 컴포넌트 | PascalCase + Page | `ItemListPage.tsx` |
| 컴포넌트 | PascalCase | `ItemCard.tsx` |
| 훅 | camelCase (use 접두사) | `useItemList.ts` |
| 유틸리티 | camelCase | `formatDate.ts` |
| 타입 | camelCase.types | `item.types.ts` |
| 상수 | camelCase | `apiEndpoints.ts` |
| 스타일 | 컴포넌트명.module | `ItemCard.module.css` |
| 테스트 | 대상파일명.test | `ItemCard.test.tsx` |
| API 함수 | 도메인Api | `itemApi.ts` |
| Query 훅 | 도메인Queries | `itemQueries.ts` |

## 배럴 파일 (index.ts)

각 페이지 도메인 디렉토리는 `index.ts`로 public API를 노출한다.

```typescript
// pages/{domain}/index.ts
export { ItemList } from './components/ItemList';
export { useItems } from './hooks/useItems';
export type { Item } from './types/item.types';
```

외부에서는 반드시 배럴 파일을 통해 import한다.

```typescript
// Good
import { ItemList, useItems } from '@/pages/{domain}';

// Bad - 내부 경로 직접 접근
import { ItemList } from '@/pages/{domain}/components/ItemList';
```

## 경로 별칭 (Path Alias)

`tsconfig.json`과 `vite.config.ts`에서 `@/` 별칭을 설정한다.

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

```typescript
// vite.config.ts
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
});
```
