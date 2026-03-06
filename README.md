# md-template

AI 코딩 에이전트(Claude, Gemini 등)를 위한 프로젝트별 마크다운 템플릿 모음입니다.
각 프로젝트 유형에 맞는 `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` 및 세부 가이드 문서를 제공합니다.

---

## 사용 방법

1. 프로젝트 유형에 맞는 폴더를 선택한다.
2. 해당 폴더의 파일들을 자신의 프로젝트 루트에 복사한다.
3. 프로젝트에 맞게 플레이스홀더(`{name}`, `{Name}`, `{package}` 등)를 수정한다.
4. feature 문서는 `docs/architecture/feature/_template.md`를 복사하여 각 feature별로 작성한다.

---

## 템플릿 목록

| 폴더 | 프로젝트 유형 | 설명 |
|------|-------------|------|
| [`kmp/`](kmp/) | Kotlin Multiplatform | 클린 아키텍처, Compose Multiplatform, Koin DI, Offline-First |
| [`react/`](react/) | React (Admin) | Vite, TypeScript, TanStack Query, 페이지 단위 도메인 모듈 구조 |

---

## 폴더 구조

### 루트

| 경로 | 설명 |
|------|------|
| `README.md` | 이 파일 |
| `kmp/` | KMP 프로젝트 템플릿 |
| `react/` | React 프로젝트 템플릿 |
| `kotlin-webflux/` | Kotlin WebFlux 백엔드 프로젝트 템플릿 |

### kmp/

| 경로 | 설명 |
|------|------|
| `AGENTS.md` | 에이전트 전반 지침 (Single Source of Truth) |
| `CLAUDE.md` | Claude용 진입점 (AGENTS.md 참조) |
| `GEMINI.md` | Gemini용 진입점 (AGENTS.md 참조) |
| `docs/architecture/` | 모듈별 아키텍처 가이드 (README, app, core, feature) |
| `docs/compose/` | Compose UI, 상태, Effect, 네비게이션 |
| `docs/convention/` | Git 커밋, PR, 테스트, DI, 로컬라이제이션 |
| `docs/coroutine/` | 코루틴, Flow, 스코프, 에러 처리 |
| `docs/kotlin/` | 네이밍, 타입, 이디엄 |
| `docs/reactive/` | 리액티브, UDF/MVI, 스트림 |

### react/

| 경로 | 설명 |
|------|------|
| `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` | 에이전트 지침 파일 |
| `docs/architecture/` | 아키텍처, 폴더 구조 |
| `docs/convention/` | Git 커밋, PR, 테스트, 코드 스타일 |
| `docs/react/` | 컴포넌트, 훅, 라우팅, 상태 |
| `docs/tanstack-query/` | Query, Mutation, Key 패턴 |
| `docs/typescript/` | 네이밍, 타입 |
| `docs/vite/` | Vite 설정 |

### kotlin-webflux/

| 경로 | 설명 |
|------|------|
| `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` | 에이전트 지침 파일 |
| `docs/architecture/` | Hexagonal Architecture, 패키지 구조 |
| `docs/domain/` | DDD, UseCase 가이드 |
| `docs/webflux/` | WebFlux, R2DBC, SSE, WebSocket, 에러 처리 |
| `docs/testing/` | TDD, 단위/통합 테스트, Fixture |
| `docs/kotlin/` | Kotlin 이디엄, 네이밍 |
| `docs/convention/` | Git 커밋, PR, API 설계 |

---

## 에이전트 파일 구조

각 템플릿은 동일한 패턴을 따릅니다:

| 파일 | 역할 |
|------|------|
| `AGENTS.md` | 프로젝트 전반 지침의 **Single Source of Truth**. 에이전트 문서 라우팅, 아키텍처 원칙, 판단 규칙 포함 |
| `CLAUDE.md` | Claude Code용 진입점. `AGENTS.md`를 참조 |
| `GEMINI.md` | Gemini용 진입점. `AGENTS.md`를 참조 |
| `docs/` | 세부 가이드 문서. `AGENTS.md`의 라우팅 테이블에서 참조 |

---

## 새 프로젝트 유형 추가

1. 루트에 새 폴더를 생성한다 (예: `flutter/`, `spring/`).
2. `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`를 작성한다.
3. `docs/` 하위에 아키텍처, 컨벤션, 기술별 가이드를 추가한다.
4. 이 README의 템플릿 목록에 등록한다.

---

## 기여

- 기존 템플릿 수정 시 범용성을 유지한다. 특정 프로젝트에 종속된 내용은 포함하지 않는다.
- 새 가이드 문서 추가 시 `AGENTS.md`의 라우팅 테이블과 `docs/architecture/README.md` 인덱스에 등록한다.
