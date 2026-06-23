# Frontend — plz-job 웹 클라이언트

개발자 취업 준비생용 **공고·지원 기록 관리 + 공공 채용 데이터 분석** 서비스의 프론트엔드.
React 19 + Vite로 만들고, Spring Boot 백엔드(`localhost:8080`)의 REST API를 소비한다.

```
컴포넌트/페이지 → features/<도메인>/hooks(TanStack Query) → api.js
   → lib/api/client.js(axios, 응답 봉투 언래핑) → 백엔드 /api/*
   (개발 중 백엔드 없이 돌릴 땐 MSW 목으로 대체)
```

## 기술 스택

| 분류                        | 사용                                                    |
| --------------------------- | ------------------------------------------------------- |
| 코어                        | React 19, Vite 8 (JS, TS 아님)                          |
| 서버 상태 / 클라이언트 상태 | TanStack Query v5 / Zustand v5                          |
| 라우팅                      | react-router-dom v7                                     |
| 폼·검증                     | react-hook-form + yup                                   |
| 스타일                      | Tailwind CSS v4 (`@tailwindcss/vite`, config 파일 없음) |
| HTTP / 목킹                 | axios / MSW                                             |
| 차트·일정·날짜              | recharts · react-big-calendar · date-fns                |
| 테스트                      | Playwright                                              |

## 구조

```
src/
├─ pages/              # 라우트 단위 화면 (<이름>Page.jsx, default export)
├─ features/<도메인>/   # auth · jobPostings · schedules · documents · retrospectives · ai · dashboard
│  ├─ api.js           #   axios 호출 래퍼 (apiClient만 사용, named export)
│  └─ hooks.js         #   api.js를 감싼 TanStack Query 훅
├─ components/common/  # 재사용 UI (AsyncBoundary, LoadingSkeleton, ErrorState, EmptyState, StageBadge …)
├─ components/layout/  # AppLayout · PageShell · Sidebar · Header
├─ lib/api/client.js   # axios 인스턴스 + 응답 봉투 언래핑 인터셉터 (HTTP 단일 진입점)
├─ lib/query/          # QueryClient 기본 옵션
├─ store/              # Zustand (authStore, filterStore) — 클라이언트 상태 전용
├─ constants/          # stageCodes.js — 단계/타입 코드↔한글 라벨 단일 출처
├─ routes/             # AppRouter(라우트 정의), ProtectedRoute(인증 가드)
└─ mocks/              # MSW handlers.js, browser.js
```

## 실행

```powershell
npm install
npm run dev        # Vite dev 서버(HMR) → http://localhost:5173
npm run build      # 프로덕션 빌드 → dist/
npm run preview    # 빌드 결과 미리보기
npm run lint       # ESLint (코드 수정 후 항상 실행)
npx playwright test            # E2E 테스트 (package.json에 test 스크립트 없음)
npx playwright test <파일경로>  # 단일 테스트만 실행
```

실서버 연동으로 띄우려면 백엔드 인프라를 먼저 기동한다:

```bash
docker compose up -d            # 루트에서 Oracle + Ollama
cd ../backend && ./gradlew bootRun
```

## 환경변수 / 모킹

`.env.example`를 복사해 `.env` 생성. `VITE_` 접두사여야 클라이언트에 노출된다.

- `VITE_API_BASE_URL` — axios baseURL / OAuth 리다이렉트 접두사 (기본 `/api`)
- `VITE_API_MOCKING` — `enabled`면 MSW 브라우저 목킹 ON, 비우면 OFF

> **현재는 MSW를 끄고 실서버에 직접 연결**한다(`.env`·`.env.development` 모두 `VITE_API_MOCKING=` 비어 있음).
> dev는 `http://localhost:8080/api`로 direct 호출, build/preview는 `/api`(vite proxy → 8080).
> `src/mocks/handlers.js`는 백엔드 미구현 API를 메우는 용도이며, 목 응답도 봉투 포맷(`ok()` 헬퍼)을 지킨다.

## 모듈 간 계약 (깨면 안 되는 것)

- **응답 봉투**: 모든 응답은 `{ success, data, error, timestamp }`(명세서 §11). `apiClient` 인터셉터가
  `data`만 벗겨 반환하므로 컴포넌트에서 `res.data.data`로 **수동 언래핑 금지**.
- **인증**: 소셜 로그인(Kakao/Google) → 백엔드가 JWT를 **HttpOnly 쿠키**로 발급.
  토큰을 JS/localStorage에 저장하지 않고 `withCredentials`로 전송, 인증 여부는 `GET /users/me`로 판단.
- **단계/타입 코드**: API엔 영문 키(`APPLIED`, `INTERVIEW` …)를 보내고 화면엔 한글 라벨만.
  `constants/stageCodes.js`가 단일 출처 — 라벨을 새로 하드코딩하지 않는다.
- **분석 데이터(단방향)**: `etl/`가 적재한 `analytics_*`를 백엔드가 조회만 한다.
  대시보드/시장분석(DASH-04/05/06)은 그 조회 API를 소비하며 "전체"는 센티넬 `'ALL'`.

## 데이터 흐름과 소비처

- **시장 분석**: `/api/market/*` 요청 → 백엔드 → Oracle `ANALYTICS_*`. 이 테이블은 [../etl](../etl)가
  사람인 크롤링으로 채운다([../etl/README.md](../etl/README.md) 참고).
- **AI(예상 면접 질문·분석 리포트)**: 백엔드가 로컬 LLM(Ollama)으로 생성. 프론트는 결과만 표시하고
  AI 산출물에는 `AiDisclaimerBadge`를 단다.

## 핵심 규칙 (코드만 봐선 안 보이는 것)

- 컴포넌트에서 **axios·fetch 직접 호출 금지**. `features/<도메인>/api.js` → `hooks.js`(Query/Mutation) → 컴포넌트 순으로 감싼다.
- 서버에서 온 데이터는 전부 TanStack Query로 관리하고 Zustand에 복사하지 않는다.
- 스타일은 Tailwind 유틸리티만(CSS Modules·styled-components·인라인 style 금지). 다크모드 `dark:`, 색은 `zinc` 팔레트.
- UI 텍스트는 한국어. 로딩/에러/빈 상태는 `components/common/`의 컴포넌트를 재사용한다.
- 새 코드 주석에도 명세서 절(`§11`)·요구사항 ID(`AUTH-05`, `JOB-07`, `UI-03`)를 단다.
