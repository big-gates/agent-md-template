# AGENTS.md

## 목적
- 이 파일은 React 어드민 프로젝트의 전반 지침을 관리한다.
- 모듈/기능별 세부 지침은 `docs/` 하위 문서에서 관리한다.

---

## 에이전트 문서 라우팅

> 작업 대상 파일 경로를 보고, 아래 테이블에서 **반드시 읽어야 할 문서**를 먼저 참조한다.

| 작업 대상 경로 패턴 | 읽어야 할 문서 |
|---|---|
| `src/app/**` | [architecture/README.md](docs/architecture/README.md) |
| `src/pages/*/` | [architecture/README.md](docs/architecture/README.md) + [folder-structure.md](docs/architecture/folder-structure.md) |
| `src/shared/**` | [architecture/README.md](docs/architecture/README.md) |
| `src/shared/components/**` | [component-guide.md](docs/react/component-guide.md) |
| `src/shared/hooks/**` | [hooks-guide.md](docs/react/hooks-guide.md) |
| `**/*Page.tsx` | [component-guide.md](docs/react/component-guide.md), [routing-guide.md](docs/react/routing-guide.md) |
| `**/components/**` | [component-guide.md](docs/react/component-guide.md) |
| `**/hooks/use*.ts` | [hooks-guide.md](docs/react/hooks-guide.md) |
| `**/api/*Api.ts` | [query-guide.md](docs/tanstack-query/query-guide.md) |
| `**/api/*Queries.ts` | [query-guide.md](docs/tanstack-query/query-guide.md), [mutation-guide.md](docs/tanstack-query/mutation-guide.md), [key-guide.md](docs/tanstack-query/key-guide.md) |
| `**/types/*.types.ts` | [type-guide.md](docs/typescript/type-guide.md) |
| `**/*.test.tsx`, `**/*.test.ts` | [testing-guide.md](docs/convention/testing-guide.md) |
| `vite.config.ts` | [config-guide.md](docs/vite/config-guide.md) |
| `.env*` | [config-guide.md](docs/vite/config-guide.md) |
| `**/*.module.css` | [code-style-guide.md](docs/convention/code-style-guide.md) |

---

## 프로젝트 전반 원칙

### 기술 스택

| 기술 | 버전 | 역할 |
|---|---|---|
| React | 18 | UI 프레임워크 |
| Vite | - | 빌드 도구 + 개발 서버 (:5173) |
| TypeScript | - | 타입 안전성 |
| TanStack Query | - | 서버 상태 관리 (React Query) |

### 아키텍처

- 페이지 단위 도메인 모듈 구조를 따른다. 각 `pages/{domain}/` 디렉토리가 해당 도메인의 컴포넌트, 훅, API, 타입을 자체 포함한다.
- 의존 방향:

| from | to | 허용? |
|---|---|---|
| `pages/{domain}` | `shared/*` | O |
| `pages/{domain}` | 다른 `pages/{domain}` | **X** (shared로 승격) |
| `shared` | `pages/*` | **X** |
| `shared` | 외부 라이브러리 | O |

- 두 개 이상의 페이지에서 사용되는 컴포넌트/훅/타입은 반드시 `shared/`로 승격한다.
- 모든 페이지 모듈은 `index.ts` 배럴 파일을 통해 public API만 노출한다.

### 상태 관리

- 서버 상태는 반드시 TanStack Query로 관리한다. `useState`로 API 데이터를 관리하지 않는다.
- 클라이언트 전역 상태는 React Context를 사용한다.
- 목록 필터/정렬/페이지네이션은 URL searchParams로 관리한다.
- 상세 규칙은 [state-guide.md](docs/react/state-guide.md)를 참조한다.

### TypeScript

- `any` 사용을 금지한다. `unknown` + 타입 가드를 사용한다.
- `enum` 대신 유니온 타입을 사용한다.
- Non-null assertion (`!`) 대신 옵셔널 체이닝 / nullish coalescing을 사용한다.
- 상세 규칙은 [type-guide.md](docs/typescript/type-guide.md), [naming-guide.md](docs/typescript/naming-guide.md)를 참조한다.

### 컴포넌트

- 모든 컴포넌트는 함수 컴포넌트로 작성한다.
- Page 컴포넌트는 `export default`, 그 외는 named export를 사용한다.
- 이벤트 핸들러는 `handle` 접두사를 사용한다.
- 상세 규칙은 [component-guide.md](docs/react/component-guide.md)를 참조한다.

### TanStack Query

- Query Key는 반드시 Key Factory 패턴을 사용한다. 문자열 직접 사용을 금지한다.
- API 호출 함수(`*Api.ts`)와 Query 훅(`*Queries.ts`)을 분리한다.
- mutation 성공 후 관련 쿼리를 무효화한다.
- 상세 규칙은 [query-guide.md](docs/tanstack-query/query-guide.md), [mutation-guide.md](docs/tanstack-query/mutation-guide.md), [key-guide.md](docs/tanstack-query/key-guide.md)를 참조한다.

---

## 에이전트 판단 규칙

### 새 파일/컴포넌트를 어디에 만들어야 하는가?

| 만들려는 것 | 위치 | 근거 |
|---|---|---|
| 페이지 컴포넌트 | `pages/{domain}/` | 라우트 진입점 |
| 도메인 전용 컴포넌트 | `pages/{domain}/components/` | 해당 도메인에서만 사용 |
| 도메인 전용 훅 | `pages/{domain}/hooks/` | 해당 도메인 로직 |
| API 호출 함수 | `pages/{domain}/api/*Api.ts` | 순수 API 함수 |
| TanStack Query 훅 | `pages/{domain}/api/*Queries.ts` | Query/Mutation 훅 |
| 도메인 전용 타입 | `pages/{domain}/types/` | 해당 도메인 타입 |
| 도메인 전용 유틸 | `pages/{domain}/utils/` | 해당 도메인 유틸 |
| 공통 UI 컴포넌트 (2+ 페이지 사용) | `shared/components/` | 도메인 비종속 |
| 공통 훅 | `shared/hooks/` | 범용 훅 |
| 공통 타입 | `shared/types/` | API 응답, 페이지네이션 등 |
| 공통 유틸 | `shared/utils/` | 포맷터, 밸리데이터 등 |
| API 클라이언트 / 인터셉터 | `shared/api/` | 전역 설정 |
| 상수 | `shared/constants/` | 라우트, 엔드포인트 등 |
| Context Provider | `shared/context/` | 전역 상태 (인증, 테마) |
| 라우터 / 프로바이더 | `app/` | 앱 초기 구성 |

### 의존을 추가해도 되는가?

| from → to | 허용? |
|---|---|
| `pages/{domain}` → `shared/*` | O |
| `pages/{domain}` → 다른 `pages/{domain}` | **X** |
| `shared` → `pages/*` | **X** |
| `app` → `pages/*`, `shared/*` | O |

---

## 세부 지침 문서 위치

| 문서 | 읽어야 할 때 |
|---|---|
| [docs/architecture/README.md](docs/architecture/README.md) | 전체 구조, 레이어 규칙 확인 시 |
| [docs/architecture/folder-structure.md](docs/architecture/folder-structure.md) | 폴더 구조, 네이밍, 경로 별칭 확인 시 |
| [docs/architecture/module/](docs/architecture/module/) | 각 도메인 모듈 상세 확인 시 |
| [docs/react/component-guide.md](docs/react/component-guide.md) | 컴포넌트 작성 규칙 확인 시 |
| [docs/react/hooks-guide.md](docs/react/hooks-guide.md) | 커스텀 훅 작성 규칙 확인 시 |
| [docs/react/state-guide.md](docs/react/state-guide.md) | 상태 관리 전략 확인 시 |
| [docs/react/routing-guide.md](docs/react/routing-guide.md) | 라우팅, lazy loading, Protected Route 확인 시 |
| [docs/typescript/type-guide.md](docs/typescript/type-guide.md) | 타입 작성 규칙 확인 시 |
| [docs/typescript/naming-guide.md](docs/typescript/naming-guide.md) | 네이밍 컨벤션 확인 시 |
| [docs/tanstack-query/query-guide.md](docs/tanstack-query/query-guide.md) | useQuery, QueryClient 설정 확인 시 |
| [docs/tanstack-query/mutation-guide.md](docs/tanstack-query/mutation-guide.md) | useMutation, 캐시 무효화 확인 시 |
| [docs/tanstack-query/key-guide.md](docs/tanstack-query/key-guide.md) | Query Key 패턴 확인 시 |
| [docs/convention/git-commit-template.md](docs/convention/git-commit-template.md) | 커밋 메시지 규칙 확인 시 |
| [docs/convention/pull-request-template.md](docs/convention/pull-request-template.md) | PR 작성 규칙 확인 시 |
| [docs/convention/testing-guide.md](docs/convention/testing-guide.md) | 테스트 작성 규칙 확인 시 |
| [docs/convention/code-style-guide.md](docs/convention/code-style-guide.md) | 코드 스타일, import 순서 확인 시 |
| [docs/vite/config-guide.md](docs/vite/config-guide.md) | Vite 설정, 환경 변수, 프록시 확인 시 |

## 문서 운영 규칙
- 새 페이지 도메인이 생기면 `docs/architecture/module/{name}.md`를 추가하고 이 파일의 라우팅 테이블에 등록한다.
- 공통 모듈 변경 시 `docs/architecture/README.md`를 함께 갱신한다.
- 두 개 이상 페이지에서 재사용되는 컴포넌트/훅을 발견하면 `shared/`로 승격하고 관련 문서를 갱신한다.
