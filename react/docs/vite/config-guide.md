# Vite Configuration Guide

## 기본 설정

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
});
```

## 환경 변수

### 파일 구조

| 파일 | 역할 | git 추적 |
|---|---|---|
| `.env` | 공통 환경 변수 | O |
| `.env.local` | 로컬 오버라이드 | X |
| `.env.development` | 개발 환경 | O |
| `.env.production` | 운영 환경 | O |

### 네이밍 규칙

Vite에서 클라이언트에 노출하려면 `VITE_` 접두사가 필요하다.

```
# .env
VITE_API_BASE_URL=http://localhost:8080/api
VITE_APP_TITLE=Admin
```

### 타입 정의

```typescript
// src/vite-env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_TITLE: string;
}
```

## 프록시 설정

개발 서버에서 CORS 우회를 위해 프록시를 설정한다.

```typescript
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
      // rewrite: (path) => path.replace(/^\/api/, ''),  // 필요 시
    },
  },
},
```

## 빌드 최적화

```typescript
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        vendor: ['react', 'react-dom'],
        query: ['@tanstack/react-query'],
        router: ['react-router-dom'],
      },
    },
  },
},
```
