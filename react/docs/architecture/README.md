# Architecture Overview

## 기술 스택

| 기술 | 버전 | 역할 |
|---|---|---|
| React | 18 | UI 프레임워크 |
| Vite | - | 빌드 도구 + 개발 서버 (:5173) |
| TypeScript | - | 타입 안전성 |
| TanStack Query | - | 서버 상태 관리 (React Query) |

## 프로젝트 구조

| 경로 | 역할 |
|---|---|
| `src/app/` | 앱 진입점, 프로바이더, 라우터 |
| `src/app/App.tsx` | 루트 컴포넌트 |
| `src/app/providers.tsx` | QueryClient 등 프로바이더 구성 |
| `src/app/router.tsx` | 라우트 정의 |
| `src/pages/{domain}/` | 페이지 단위 도메인 모듈 (components, hooks, api, types, utils 포함) |
| `src/pages/{domain}/*Page.tsx` | 라우트 진입점 페이지 컴포넌트 |
| `src/pages/{domain}/index.ts` | 배럴 파일 (public API) |
| `src/shared/api/` | API 클라이언트, 인터셉터 |
| `src/shared/components/` | 공통 UI 컴포넌트 |
| `src/shared/hooks/` | 공통 커스텀 훅 |
| `src/shared/types/` | 공통 타입 정의 |
| `src/shared/utils/` | 유틸리티 함수 |
| `src/shared/constants/` | 상수 정의 |
| `src/shared/context/` | Context Provider |
| `src/assets/` | 정적 리소스 (이미지, 폰트 등) |

## 레이어 구조

| from | to | 허용? | 설명 |
|---|---|---|---|
| `pages/{domain}` | `shared/*` | O | 공통 모듈 사용 |
| `pages/{domain}` | 다른 `pages/{domain}` | **X** | shared로 승격 필요 |
| `shared` | `pages/*` | **X** | 역방향 금지 |
| `shared` | 외부 라이브러리 | O | - |
| `app` | `pages/*`, `shared/*` | O | 앱 초기 구성 |

- **pages**: 라우트에 매핑되는 도메인 모듈. 페이지 컴포넌트 + 도메인 전용 컴포넌트, 훅, API, 타입을 모두 포함한다.
- **shared**: 도메인에 종속되지 않는 공통 모듈.

## 페이지 모듈 내부 구조

각 페이지 디렉토리는 해당 도메인의 모든 기능을 자체적으로 포함한다.

| 하위 경로 | 역할 | 예시 |
|---|---|---|
| `components/` | 도메인 전용 컴포넌트 | `ItemCard.tsx`, `ItemFilter.tsx` |
| `hooks/` | 도메인 전용 커스텀 훅 | `useTableParams.ts` |
| `api/*Api.ts` | 순수 API 호출 함수 | `itemApi.ts` |
| `api/*Queries.ts` | TanStack Query 훅 (query + mutation) | `itemQueries.ts` |
| `types/*.types.ts` | 도메인 전용 타입 | `item.types.ts` |
| `utils/` | 도메인 전용 유틸리티 | `itemValidation.ts` |
| `*Page.tsx` | 페이지 컴포넌트 (라우트 진입점) | `ItemListPage.tsx` |
| `index.ts` | public API (배럴 파일) | - |

## 의존성 규칙

1. **pages**는 shared를 import할 수 있다.
2. **pages**는 다른 pages를 직접 import하지 않는다. 공유가 필요하면 shared로 승격한다.
3. **shared**는 외부 라이브러리와 자체 모듈만 참조한다.
4. 두 개 이상의 페이지에서 사용되는 컴포넌트/훅/타입은 shared로 승격한다.
5. 모든 페이지 모듈은 `index.ts`를 통해 public API만 노출한다.
