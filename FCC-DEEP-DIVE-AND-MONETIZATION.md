# freeCodeCamp 전수조사 · 활용법 · 수익화 전략 총정리

> Claude Code(카리나 페르소나)와 진행한 2차 심화 분석 대화 기록입니다.
> 작성일: 2026-09-30 / 작업 브랜치: `claude/eloquent-ptolemy-0fveh4`
> 1차 분석 문서: [`FCC-ANALYSIS-SUMMARY.md`](./FCC-ANALYSIS-SUMMARY.md)

---

## 🔗 관련 깃허브 · 웹 주소 모음

| 구분                      | 주소                                                                                    |
| ------------------------- | --------------------------------------------------------------------------------------- |
| **내 포크 (작업 저장소)** | https://github.com/bmshin94/freeCodeCamp                                                |
| **작업 브랜치**           | https://github.com/bmshin94/freeCodeCamp/tree/claude/eloquent-ptolemy-0fveh4            |
| **원본 업스트림**         | https://github.com/freeCodeCamp/freeCodeCamp                                            |
| 공식 서비스               | https://www.freecodecamp.org                                                            |
| 기여 가이드               | https://contribute.freecodecamp.org                                                     |
| 이슈 트래커               | https://github.com/freeCodeCamp/freeCodeCamp/issues                                     |
| 포럼                      | https://forum.freecodecamp.org                                                          |
| 유튜브 채널               | https://youtube.com/freecodecamp                                                        |
| 기술 블로그(News)         | https://www.freecodecamp.org/news                                                       |
| 서브모듈 – i18n 커리큘럼  | https://github.com/freeCodeCamp/i18n-curriculum                                         |
| 서브모듈 – 챌린지 에디터  | https://github.com/freeCodeCamp/challenge-editor                                        |
| RAG/MCP 커리큘럼          | https://github.com/bmshin94/freeCodeCamp/tree/main/curriculum/challenges/english/blocks |

---

## 1. 전수조사 결과 — 이게 뭐하는 물건인가

### 결론

라이브러리도, 플러그인도, 스킬도, MCP도 아니다.
**freeCodeCamp.org 학습 플랫폼 전체를 돌리는 실제 프로덕션 소스코드(모노레포)** 이다.

| 항목                      | 측정값                                                                     |
| ------------------------- | -------------------------------------------------------------------------- |
| 추적 파일 수              | **19,481개**                                                               |
| 저장소 용량               | **128MB** (.git 제외)                                                      |
| 커리큘럼 마크다운         | **16,679개**                                                               |
| 블록(챕터)                | **1,043개**                                                                |
| 슈퍼블록(자격증 과정)     | **99개**                                                                   |
| 챌린지 유형               | **34종 (0~33번)**                                                          |
| Prisma 모델               | **12개**                                                                   |
| Playwright E2E 스펙       | 약 **80개**                                                                |
| GitHub Actions 워크플로우 | **20개**                                                                   |
| 라이선스                  | 코드 **BSD-3-Clause** / 커리큘럼 © freeCodeCamp / TOP 부분 CC BY-NC-SA 4.0 |
| 패키지 매니저             | **pnpm 10.33.3** + **Turborepo 2.10**                                      |
| Node 요구 버전            | **>= 24** (`.nvmrc` = 24)                                                  |

### 폴더별 해부

#### `client/` — 프론트엔드

- **Gatsby 5.16 + React 18.3** (정적 사이트 생성 기반)
- **Redux Toolkit 2.11 + redux-saga 1.4** (+ 레거시 redux 4)
- **Tailwind CSS** + 자체 디자인 시스템 `@freecodecamp/ui`
- **Monaco Editor 0.55** (VS Code 엔진) · **Sandpack** · **xterm.js 6** (웹 터미널)
- **Algolia** 검색, **Stripe / PayPal / Patreon** 결제 위젯
- **i18next 25** 다국어 (Crowdin 연동)
- `src/templates/Challenges/` 하위에 문제 유형별 화면 분리
  (classic / projects / quiz / exam / lab / fill-in-the-blank / generic / codeally / ms-trophy …)

#### `api/` — 백엔드

- **Fastify 5.8 + TypeScript**
- **Prisma 6.19 + MongoDB 8.2** — `api/prisma/schema.prisma`
  (`user`, `AccessToken`, `AuthToken`, `Donation`, `SocratesUsage`, `UserToken`,
  `sessions`, `MsUsername`, `Exam`, `Survey`, `DailyCodingChallenges`, `DripCampaign`)
- **Auth0 OAuth2** 로그인 / JWT / 쿠키 / **CSRF 보호**
- **Stripe 16** 결제, nodemailer(로컬 Mailpit) / AWS SES(운영) 메일
- **Sentry** 에러추적, **GrowthBook** 기능플래그·A/B, **Swagger UI** 자동 API 문서
- 라우트 권한 분리: `public/` `protected/` `apps/` `helpers/`

#### ⭐ 핵심 발견: Socrates (AI 힌트 기능)

`api/src/routes/protected/socrates.ts`

```ts
const DAILY_LIMITS = { donor: 10, nonDonor: 3 } as const;

const response = await fetch(`${SOCRATES_ENDPOINT}/hint`, {
  headers: { 'x-api-key': SOCRATES_API_KEY },
  body: JSON.stringify({ description, userInput, seed, hints })
});
```

| 실무 과제          | Socrates의 해법                                                             |
| ------------------ | --------------------------------------------------------------------------- |
| AI 비용 폭발 방지  | 일일 쿼터 (후원자 10회 / 일반 3회)                                          |
| 사용량 집계        | `SocratesUsage` 컬렉션 + **UTC 날짜 기준** 카운트                           |
| 권한 차단          | `req.user.socrates === false` → 403                                         |
| 네트워크 에러 분류 | `ENOTFOUND`/`ECONNREFUSED`/`ECONNRESET`/`ETIMEDOUT`/`EAI_AGAIN`/`UND_ERR_*` |
| 입력 검증          | TypeBox 스키마, validation 실패 시 400                                      |
| 아키텍처 격리      | AI를 **별도 마이크로서비스**로 분리 (`SOCRATES_ENDPOINT`)                   |
| 유저 설정          | `client/src/components/settings/socrates.tsx` 토글                          |
| 프론트 비동기      | `templates/Challenges/redux/ask-socrates-saga.js`                           |
| 테스트             | `socrates.test.ts`, `socrates-reducer.test.js`                              |

→ **LLM 연동 + 레이트리밋 + 에러핸들링의 프로덕션 교본.**

#### `curriculum/` — 학습 콘텐츠 (레포의 진짜 자산)

마크다운 1장 = 문제 1개. 커스텀 섹션 문법:

```markdown
---
id: 5c6c06847491271903d37cfd
title: Use the value attribute with Radio Buttons and Checkboxes
challengeType: 0
dashedName: use-the-value-attribute-with-radio-buttons-and-checkboxes
---

# --description-- 설명

# --instructions-- 지시사항

# --hints-- 채점 테스트 (실행 가능한 JS assert 코드)

# --seed-- 시작 코드
```

즉 **"자연어 문제 + 실행 가능한 정답 판정 코드"가 한 파일에 세트**로 들어있다.

**주목: AI 정규 커리큘럼 존재** — `curriculum/structure/superblocks/learn-rag-mcp-fundamentals.json`

```json
{
  "blocks": [
    "understanding-rag",
    "retrieval-engine-internals",
    "designing-reliable-rag-systems",
    "mcp-ecosystem-and-tooling"
  ]
}
```

#### `packages/` · `tools/` · `e2e/` · `docker/` · `.github/`

- `packages/`: `shared`(클라·서버 공용), `challenge-builder`, `challenge-linter`, `eslint-config`
- `tools/challenge-parser`: **마크다운 → 챌린지 JSON 변환기**
- `tools/challenge-helper-scripts`: 문제/프로젝트/퀴즈 생성 CLI
- `tools/client-plugins/gatsby-source-challenges`: 커리큘럼을 Gatsby 데이터로 주입
- `e2e/`: Playwright 스펙 약 80개 (결제, 시험, 에디터, 핫키, 격리 프리뷰 등)
- `docker/`: MongoDB 레플리카셋 + Mailpit 원클릭 구성
- `.devcontainer/`: GitHub Codespaces 완전 대응 (자동 시딩)
- `.github/`: 워크플로우 20개 + `copilot-instructions.md`(AI 코드리뷰 지시문)

### 언제 쓰는가 / 나에게 무슨 도움인가

| 상황                 | 활용                                               |
| -------------------- | -------------------------------------------------- |
| 오픈소스 기여        | first-timers-only 친화, 커리큘럼 오타수정부터 가능 |
| 대규모 아키텍처 학습 | 모노레포·Turborepo·Fastify·Prisma 실전 구성        |
| 교육/LMS 서비스 개발 | 자동채점·진도관리·자격증 발급 레퍼런스             |
| AI 기능 개발         | Socrates = LLM 연동 + 쿼터 관리 완성본             |
| 포트폴리오           | 세계적 오픈소스 컨트리뷰터 이력                    |

---

## 2. 쉬운 비유 정리

> **"코딩 학원 건물 전체 설계도 + 교재 창고를 통째로 복사해온 것"**

| 폴더               | 학원 비유                                       |
| ------------------ | ----------------------------------------------- |
| `client/`          | 강의실·로비 (학생이 보는 화면)                  |
| `api/`             | 교무실·행정실 (로그인·진도저장·수료증·결제)     |
| `curriculum/`      | 교재 창고 (문제 16,679개 + 채점표)              |
| `tools`·`packages` | 관리실·공구실 (교재 제작 틀)                    |
| `e2e/`             | 건물 점검 로봇 (가짜 사용자가 자동 클릭 테스트) |
| `docker/`          | 건물 조립 키트 (`docker compose up` 한 줄)      |

핵심 포인트 3가지

1. **Gatsby** = 주문 받고 요리하는 게 아니라 **미리 도시락 다 싸두는 방식** → 그래서 빠름
2. **채점은 사람이 아니라 코드가** — 마크다운 안의 `assert(...)` 검사 코드가 즉시 O/X 판정
3. **Socrates** = AI 힌트 선생님. AI 호출은 비싸니까 **하루 3번 제한**을 코드로 구현해 둠

---

## 3. 질문 7개 답변

### Q1. 설치 및 사용법

준비물: Node **24+**, pnpm **10+**, Docker, Git, 메모리 8GB 권장(빌드 시 `--max-old-space-size=7168`)

```bash
git clone https://github.com/bmshin94/freeCodeCamp.git
cd freeCodeCamp
cp sample.env .env
docker compose -f docker/docker-compose.yml -f docker/docker-compose.ports.yml up -d
pnpm install
pnpm seed
pnpm develop
```

| 주소                                | 용도                  |
| ----------------------------------- | --------------------- |
| http://localhost:8000               | 사이트 본체           |
| http://localhost:3000               | API 서버              |
| http://localhost:3000/documentation | Swagger API 문서      |
| http://localhost:8025               | Mailpit (가짜 메일함) |

`FCC_ENABLE_DEV_LOGIN_MODE=true` → Auth0 없이 로그인 테스트 가능.
더 쉬운 방법: **GitHub Codespaces** (`.devcontainer/`가 Mongo 세팅·시딩 자동 처리)

자주 쓰는 명령어

```bash
pnpm develop / develop:client / develop:api
pnpm test / lint / format
pnpm playwright:run
pnpm create-new-project / create-new-quiz
pnpm challenge-editor
pnpm clean-and-develop      # 막힐 때 만능키
```

### Q2. 플러그인? 스킬? MCP?

**셋 다 아니다. 완제품 웹 애플리케이션이다.**

| 구분                | freeCodeCamp |
| ------------------- | ------------ |
| 플러그인            | ❌           |
| Claude Skill        | ❌           |
| MCP 서버            | ❌           |
| **웹 애플리케이션** | ✅           |

단, 헷갈릴 만한 요소는 실제로 있다.

- `tools/client-plugins/` → Gatsby **자체 플러그인** 3개
- `api/src/plugins/` → Fastify **플러그인** (Auth, CSRF, 세션)
- 루트 `CLAUDE.md` → Claude에게 주는 **지침서**(카리나 페르소나 설정)
- `curriculum/.../mcp-ecosystem-and-tooling` → MCP를 **가르치는 교육 콘텐츠**

### Q3. API 토큰을 사용해야 되는가

**로컬 개발은 토큰 없이 가능하다** (`sample.env` 복사만으로 동작).

| 서비스              | 환경변수                                        | 없으면                      |
| ------------------- | ----------------------------------------------- | --------------------------- |
| MongoDB             | `MONGOHQ_URL`                                   | ❌ 필수 (Docker가 제공)     |
| 세션/쿠키/JWT       | `SESSION_SECRET`, `COOKIE_SECRET`, `JWT_SECRET` | ❌ 필수 (임의 문자열 OK)    |
| Auth0               | `AUTH0_CLIENT_ID/SECRET/DOMAIN`                 | 🟡 개발모드 로그인으로 우회 |
| Stripe              | `STRIPE_SECRET_KEY`                             | 🟡 결제만 비활성            |
| Algolia             | `ALGOLIA_APP_ID/API_KEY`                        | 🟡 검색만 비활성            |
| **Socrates(AI)**    | `SOCRATES_API_KEY`, `SOCRATES_ENDPOINT`         | 🟡 AI 힌트만 비활성         |
| Sentry / GrowthBook | `SENTRY_DSN` 등                                 | 🟢 없어도 무해              |
| PayPal / Patreon    | `PAYPAL_CLIENT_ID` 등                           | 🟡 해당 결제만              |

**핵심**: `SOCRATES_ENDPOINT`는 외부 마이크로서비스라서, `POST /hint` 하나만 받는 작은 서버를
직접 만들어 Claude API에 연결하면 **AI 힌트 기능을 자체 구현으로 되살릴 수 있다.**

⚠️ `.env`는 `.gitignore` 대상. 실제 토큰은 절대 커밋하지 말 것.

### Q4. AI 에이전트 구축에 도움이 되는가 → **매우 크게 도움된다**

1. **Socrates = 프로덕션급 LLM 연동 완성 코드** (쿼터, 권한, 에러분류, 스키마 검증, 서비스 분리, 테스트)
2. **RAG/MCP 정규 커리큘럼** — 에이전트의 양대 축(지식 주입 + 도구 연결) 학습자료
3. **구조화 데이터 16,679개** = `문제 설명 + 실행 가능한 판정 코드`
   → 코딩 에이전트 **벤치마크**, **RAG 지식베이스**, instruction 데이터셋
4. **AI 협업 인프라 레퍼런스** — `copilot-instructions.md`, `CLAUDE.md`, `.github/instructions/`
5. **에이전트가 자체 검증 가능한 환경** — Turborepo 캐싱 + 명확한 lint/test + Playwright E2E

### Q5. 수익화 아이디어 → 4장 참조

### Q6. React나 PHP로 만들 수 있는가

**React → 이미 React다.** `client/`가 React 18.3 + Gatsby 5.16이므로 바로 읽고 수정 가능.
(컴포넌트 설계 · 템플릿 패턴 · RTK+saga · Monaco 통합 · i18n · Gatsby 커스텀 플러그인 학습 가능)
Next.js로 재구성하는 것도 좋은 학습 프로젝트 겸 콘텐츠 소재.

**PHP → 가능하지만 "이 코드를 쓰는 것"이 아니라 "다시 만드는 것".**

| 부분               | PHP 가능성 | 방법                                                          |
| ------------------ | ---------- | ------------------------------------------------------------- |
| 백엔드 API         | ✅ 쉬움    | Laravel로 유저/진도/제출 API 재작성                           |
| 커리큘럼 파싱      | ✅ 가능    | 마크다운 파서로 fCC 포맷 읽기                                 |
| 관리자 페이지      | ✅ 강점    | Laravel Nova / Filament                                       |
| **코드 채점 실행** | ⚠️ 제약    | `hints`가 **JS 코드** → 브라우저 실행 또는 Node 샌드박스 필요 |
| 프론트 에디터      | ❌         | Monaco는 JS 라이브러리                                        |

추천 하이브리드

```
[Laravel/PHP]   유저·결제·관리자·진도 저장
      ↕ REST
[React]         문제 화면 + Monaco 에디터 (채점은 브라우저에서)
      ↕
[Node 소형서버] AI 힌트 /hint → Claude API   (Socrates 대체)
```

### Q7. 유튜브 강의 영상 제작 가능한가 → **가능. 단 라이선스 준수 필수**

| 대상                             | 라이선스        | 영상 활용                            |
| -------------------------------- | --------------- | ------------------------------------ |
| 코드 (`client/` `api/` `tools/`) | BSD-3-Clause    | ✅ 자유 (저작권 표시)                |
| 커리큘럼 (`curriculum/`)         | © freeCodeCamp  | ⚠️ 원문 전재 금지 / 해설·리뷰는 가능 |
| The Odin Project 부분            | CC BY-NC-SA 4.0 | ⚠️ 비영리 + 동일조건                 |
| 로고·브랜드                      | 상표권          | ⚠️ 공식 제휴 오인 금지               |

**안전**: 코드 읽기·구조 분석·따라 만들기 / **위험**: 커리큘럼 원문 낭독·번역 영상화

시리즈 기획안

- **A. "GitHub 스타 40만 프로젝트 해부하기" (10편)** — 구조 파악법 / Fastify 선택 이유 /
  Monaco 해부 / 마크다운이 채점기가 되는 원리 / Socrates AI 코드 / 모노레포 / Prisma /
  Playwright / Docker / GitHub Actions
- **B. "AI 힌트 기능 직접 만들기" (5편)** — Socrates 분석 → Claude API `/hint` 서버 →
  일일 쿼터 구현 → redux-saga 연결 → RAG로 품질 향상
- **C. "한국어 fCC 만들기" 클론코딩 (8편)** — 문제 데이터 설계 → 에디터 → 채점기 →
  진도저장 → 수료증 → 배포

팁: 첫 3초 훅 / 코드+실화면 병렬 배치 / 썸네일에 숫자(16,679 · 34종 · 40만 스타) /
국내 대형 오픈소스 코드리딩 콘텐츠 공백 = 선점 기회

---

## 4. 수익화 전략 (상세)

### 전체 지도

```
[0원]   콘텐츠 계열 → 유튜브·블로그·전자책·강의        (지금 당장)
[소액]  SaaS 계열   → AI 힌트 API·채점엔진·플랫폼      (3~6개월)
[고단가] B2B 계열   → 기업교육·코테솔루션·구축대행     (6개월+)
[간접]  커리어 계열 → 오픈소스 이력·이직·프리랜싱      (병행)
```

### 아이디어 1. 오픈소스 코드리딩 콘텐츠 (난이도 ⭐ / 추천 ⭐⭐⭐⭐⭐)

투자금 0원, 라이선스 안전(BSD-3), 국내 경쟁 희박, 다른 수익화의 씨앗.

| 채널                | 모델              | 예상 월수익            |
| ------------------- | ----------------- | ---------------------- |
| 유튜브              | 애드센스 + 멤버십 | 구독 1만 기준 50~200만 |
| 블로그              | 애드센스 + 제휴   | 10~50만                |
| 전자책(크몽/부크크) | 1.5~3만원 × 판매  | 50~300만               |
| 인프런/유데미       | 5~10만원 × 수강   | 100~500만              |

로드맵: 1~2주 코드 정독+대본 10편 → 3~4주 영상 3편 → 2개월 블로그 연재 →
3개월 전자책 → 6개월 강의 오픈

### 아이디어 2. AI 힌트 API SaaS (난이도 ⭐⭐⭐ / 추천 ⭐⭐⭐⭐⭐) ★최우선 제품

fCC가 인터페이스를 이미 정의해 둠:

```
POST {SOCRATES_ENDPOINT}/hint
headers: { 'x-api-key': ... }
body:    { description, userInput, seed, hints }
→ { hint }
```

**차별점 = "정답을 알려주지 않는 AI"**

- 소크라테스식 유도 질문 / 정답 코드 출력 가드레일
- `hints`(채점 기준) 참고 → 틀린 지점만 정확히 지적
- 단계별 힌트(방향 → 개념 → 위치)

| 플랜       | 월 가격   | 내용                          |
| ---------- | --------- | ----------------------------- |
| Free       | $0        | 100 힌트                      |
| Starter    | **$29**   | 5,000 힌트                    |
| Pro        | **$99**   | 30,000 힌트 + 커스텀 프롬프트 |
| Enterprise | **$499+** | 무제한 + 온프레미스 + SLA     |

타깃: 코딩 부트캠프 / 온라인 강의 플랫폼 / 대학 SW교육센터 / 사내 교육팀 / 인디 개발자

손익 개략

```
비용: Claude Haiku 기준 힌트 1건 ≈ $0.001~0.003 + 서버 $20/월
수익: Starter 10곳 = $290/월 (원가 ≈ $70, 마진 70%+)
      + Pro 5곳 = 합계 $785/월 → 순익 약 $550/월 (≈75만원)
→ 고객 15곳으로 월 70만원대 반복수익(구독)
```

스택: Fastify + TypeScript → Claude API(haiku 계열) → Redis 레이트리밋 → Stripe 구독
(**모든 참고 구현이 이 레포 안에 있음**)

### 아이디어 3. 코딩테스트/자동채점 플랫폼 (난이도 ⭐⭐⭐⭐ / 추천 ⭐⭐⭐⭐)

구조: `문제(설명+판정코드) → 제출 → 격리 실행 → 즉시 O/X`

| 형태               | 가격               |
| ------------------ | ------------------ |
| 사내 코테 SaaS     | 월 30~100만원      |
| 학원·부트캠프 LMS  | 월 20~50만원       |
| 대학 실습 자동채점 | 건당 500~3,000만원 |

핵심 기술은 **코드 격리 실행**. fCC는 **브라우저 실행(Sandpack + 웹워커)** 으로 서버 위험과
인프라 비용을 동시에 줄였다 (`e2e/preview-isolation.spec.ts` 참고).

### 아이디어 4. 한국어 특화 학습 플랫폼 (난이도 ⭐⭐⭐⭐ / 추천 ⭐⭐⭐)

기회: fCC는 영어 중심, 한국어 커버리지 낮음 → 영어 장벽에서 이탈하는 수요.
⚠️ 코드(BSD-3)는 활용 가능하나 **커리큘럼 번역 상업 서비스는 불가**.
→ "엔진·구조는 참고, 콘텐츠는 직접 제작"이 정답.
모델: 구독 월 9,900~29,000원 / 수료증 3~5만원 / 기업 단체 월 50~300만원 / 취업연계 수수료

### 아이디어 5. B2B 기업교육 · 구축 대행 (난이도 ⭐⭐ / 추천 ⭐⭐⭐⭐⭐) ★최고단가

| 서비스                                 | 단가             |
| -------------------------------------- | ---------------- |
| 대형 오픈소스 아키텍처 분석 사내세미나 | 회당 100~300만원 |
| 모노레포 전환 컨설팅                   | 500~2,000만원    |
| 사내 교육플랫폼 구축                   | 2,000만~1억      |
| AI 학습도우미 기능 개발                | 1,000~5,000만원  |
| 기술서적 집필                          | 인세 + 브랜딩    |

신뢰도 축적 순서: 콘텐츠로 포지셔닝(0~3개월) → 컨퍼런스 발표(3~6개월) →
fCC 실제 PR 머지로 실력 증명(병행) → 기업 문의 유입(6개월+)

### 아이디어 6. AI 에이전트 데이터·벤치마크 (난이도 ⭐⭐⭐⭐⭐ / 추천 ⭐⭐⭐)

커리큘럼 16,679개 = "문제 + 자동검증 코드" → AI 코딩 능력 자동 평가에 최적.

| 사업                                    | 수익               |
| --------------------------------------- | ------------------ |
| AI 코딩 벤치마크 리포트 (모델별 정답률) | 리포트 판매 / 협찬 |
| 코딩 에이전트 평가 SaaS                 | 월 100만원+        |
| RAG 기반 코딩 Q&A 봇                    | 구독               |
| 교육 콘텐츠 연결 MCP 서버               | 라이선스           |

⚠️ 커리큘럼 **재배포 금지**. **평가 결과·점수 공개는 가능** (데이터가 아니라 결과를 판매).

### 최종 추천 로드맵

| 단계  | 기간     | 실행                                             | 목표 수익        |
| ----- | -------- | ------------------------------------------------ | ---------------- |
| 1단계 | 0~3개월  | 유튜브 10편 + AI 힌트 API MVP 동시 진행          | 월 30~100만원    |
| 2단계 | 3~6개월  | SaaS 유료화(Stripe) + 전자책 + 인프런 강의       | 월 100~300만원   |
| 3단계 | 6~12개월 | B2B 세미나·컨설팅·플랫폼 구축, PR 머지 실적 활용 | 월 300~1,000만원 |

이 순서의 이유

1. 투자금 0인 것부터 시작
2. 한 번의 코드 분석이 영상·블로그·전자책·강의·컨설팅으로 다중 재활용
3. 제품과 콘텐츠가 상호 홍보 (영상 → API 가입, API 개발 → 영상 소재)
4. 최종 목표는 단가가 10~100배인 B2B

### 라이선스 준수 체크리스트

| 하지 말 것                 | 대신 이렇게               |
| -------------------------- | ------------------------- |
| 커리큘럼 번역해서 판매     | 구조·기술을 해설          |
| fCC 로고 사용 / 공식 사칭  | "비공식 분석 콘텐츠" 명시 |
| 커리큘럼 데이터 재배포     | 분석 결과·점수만 공개     |
| 코드 무표기 복제 후 서비스 | BSD-3 저작권 표시 후 활용 |

---

## 5. 한 장 요약

- **정체**: freeCodeCamp.org 프로덕션 소스 전체(모노레포). 플러그인/스킬/MCP 아님
- **규모**: 파일 19,481 / 128MB / 커리큘럼 16,679 / 슈퍼블록 99 / 챌린지 유형 34
- **스택**: Gatsby+React / Fastify+Prisma+MongoDB / pnpm+Turborepo / Playwright / Docker
- **숨은 보물**: `socrates.ts`(LLM 연동+쿼터 완성코드), `learn-rag-mcp-fundamentals`(RAG·MCP 커리큘럼)
- **설치**: Node 24 + pnpm 10 + Docker → `cp sample.env .env` → `pnpm install` → `pnpm seed` → `pnpm develop`
- **토큰**: 로컬은 불필요. AI 힌트만 자체 엔드포인트 구현 시 활성화
- **수익화 1순위**: 코드리딩 콘텐츠(0원) + AI 힌트 API SaaS(반복수익) → B2B 고단가
- **주의**: 코드는 BSD-3로 자유, **커리큘럼은 저작권 보호** → 전재·번역판매 금지

---

_작성: Claude Code (카리나 페르소나) · 저장소: https://github.com/bmshin94/freeCodeCamp_
