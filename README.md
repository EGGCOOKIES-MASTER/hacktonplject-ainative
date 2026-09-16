# 마음곁 · 취준생 멘탈케어 AI 동반자 (MVP)

취업 준비 과정에서 겪는 만성적인 불안감과 고립감을 완화하기 위한 **24시간 AI 동반자(챗봇)** 서비스입니다.
대화 내용은 사용자에게 별도의 작업을 요구하지 않고 **백그라운드에서 자동 분석**되어 구조화된 `difficulty` 신호로 변환되며, 이 신호를 기반으로 공공기관 및 정책 정보를 규칙 기반으로 추천합니다.

전체 기획 사양, 아키텍처 결정사항(AD-1~AD-8), 구현 승인 기준(AC-1~AC-15)은 `.omc/plans/plan-jobseeker-mental-care.md`에서 확인할 수 있습니다.

**MVP 범위**: 가명 기반 계정, 스트리밍 AI 채팅, 입력 단계 위기 감지 및 국내 상담전화 안내, 백그라운드 어려움 추출, 규칙 기반 공공기관 추천, 동의 이력 관리, PWA 기능을 포함합니다. 실제 기관 연계, 상담원 실시간 연결 및 실명 전환, 취업 매칭 기능은 2단계 개발 범위로 제외했습니다.

## 기술 스택

Next.js 16 (App Router, TypeScript) · Tailwind v4 · Supabase (Postgres, Auth, RLS, Edge Functions) · Anthropic Claude (채팅 + 위기 분류 + 백그라운드 분석) · Vitest (단위 테스트) · Playwright (E2E) · pgTAP (DB 레벨 테스트)

## 설치 및 환경 설정

### 1. 환경 변수 설정

`.env.local.example` 파일을 `.env.local`로 복사한 뒤 아래 값을 입력합니다.

| 변수 | 설명 |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | 브라우저에서 사용 가능한 값입니다. RLS로 보호됩니다. |
| `SUPABASE_SERVICE_ROLE_KEY` | **서버 전용** 키이며 RLS를 우회합니다. `lib/supabase/admin.ts`의 관리자 통계 기능과 데이터 추출 Worker에서 사용합니다. 클라이언트에 절대 노출하면 안 됩니다. |
| `ANTHROPIC_API_KEY` | 서버 전용 키입니다. `/api/chat`의 AI 동반자 및 위기 분류 기능과 `extract-difficulty` Edge Function에서 사용합니다. |
| `ADMIN_EMAILS` | 집계 정보만 보여주는 `/admin/metrics` 대시보드 접근 허용 이메일 목록입니다. 쉼표로 구분하며 대소문자를 구분하지 않습니다. (`lib/metrics/admin-auth.ts`) 이는 `auth.uid()` 기반 RLS 정책이 아니라 애플리케이션 레벨 검사입니다. |

### 2. Supabase 구성

```bash
supabase login
supabase link --project-ref <your-project-ref>   # 로컬 개발은 `supabase start` 사용

# 마이그레이션을 순서대로 적용합니다.
# 스키마 + RLS + AD-1 기본 구조 + AD-3 동의 이력
# + AD-4 real_name 보호 트리거 + AD-7 기관 초기 데이터
supabase db push        # 호스팅된 Supabase 프로젝트

# 로컬 개발 환경에서는 아래 명령을 사용합니다.
supabase db reset        # 0001_init.sql -> 0002_extraction_trigger.sql -> 0003_institutions_seed.sql 순서로 적용
```

각 마이그레이션에는 대응되는 `*.down.sql` 롤백 스크립트가 포함되어 있습니다.

Supabase Auth 설정에서 이메일/비밀번호 로그인을 활성화합니다. 필요하다면 Google OAuth도 활성화할 수 있으며, `signInWithOAuthAction`에 이미 `google` provider가 연결되어 있습니다.

### 3. 백그라운드 데이터 추출 Worker 배포 (AD-1)

`messages` 테이블의 INSERT 트리거(`0002_extraction_trigger.sql`)는 Edge Function에 비동기 HTTP 요청을 전송합니다.
다만 아래 Edge Function을 실제로 배포하기 전까지는 이 기능이 동작하지 않는 것이 정상입니다.

```bash
supabase functions deploy extract-difficulty
supabase secrets set ANTHROPIC_API_KEY=sk-ant-... SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=...
```

이후 트리거가 사용할 DB 설정값을 입력합니다.

```sql
update extraction_config set value = 'https://<project-ref>.functions.supabase.co/extract-difficulty'
  where key = 'edge_function_url';
update extraction_config set value = '<function-invoke-secret>'
  where key = 'service_role_key';
```

필요하다면 `0002_extraction_trigger.sql` 하단에 주석 처리되어 있는 재시도 작업도 예약할 수 있습니다.

```sql
create extension if not exists pg_cron;
select cron.schedule('extraction-retry-sweep', '*/5 * * * *',
  $$select reenqueue_stale_extractions();$$);
```

Edge Function을 배포하지 않아도 채팅 기능 자체는 정상적으로 동작합니다. 다만 추출 및 추천에 사용할 데이터가 없으므로 `/recommendations`는 `lib/match/curate.ts`의 fallback 규칙에 따라 모든 공공기관을 점수 없이 표시합니다.

## 실행 방법

```bash
npm install
npm run dev          # 개발 서버 실행: http://localhost:3000
npm run build        # 프로덕션 빌드
npm run start        # 프로덕션 빌드 실행
npm run typecheck    # tsc --noEmit
npm run lint
```

## 테스트

기획 문서의 Expanded Test Plan에 맞춰 테스트를 3단계로 구성했습니다.

### 단위 테스트 (Vitest) — 외부 서비스 없이 순수 로직 검증

```bash
npm test              # vitest run
npm run test:watch
npm run test:coverage
```

안전 관련 핵심 로직과 주요 품질 지표에 사용되는 프레임워크 독립 모듈을 검증합니다.

- `lib/safety/crisis-core.ts` — 위기 관련 키워드, 유사 표현 및 완곡 표현 탐지, 공백 변화 대응, 위기 판단 우선순위를 테스트합니다. 분류기가 실패해도 일반 응답으로 넘어가지 않고 안전 안내를 우선하도록 구성했습니다.
- `lib/extract/parse.ts` + `lib/extract/taxonomy.ts` — 모델이 반환한 JSON을 검증된 `difficulty_data` 행으로 변환하는 로직을 테스트합니다. 어려움이 없는 경우의 `null` 처리와 taxonomy/intensity 경계값도 포함합니다.
- `lib/match/curate.ts` — 추천 순위, 신호가 없을 때 모든 기관을 점수 없이 보여주는 fallback, 카테고리 중복 제외 로직을 테스트합니다.
- `lib/metrics/quality.ts` — AD-7에서 정의한 `quality = 0.4*coverage + 0.3*richness + 0.3*consistency` 공식, 하위 지표, `QUALITY_TARGET`(0.6) 경계값을 검증합니다.

### 데이터베이스 테스트 (pgTAP) — Supabase 환경 필요

테스트 파일은 `supabase/tests/database/*.sql`에 있으며 `npm test`에는 포함되지 않습니다.

다음 항목을 검증합니다.

- RLS 사용자 데이터 격리
- `conversations`를 경유하는 `messages` 접근 경로
- AD-4 실명 저장 방지 테스트
- AD-3 / AC-14 동의 철회
- 계정 삭제 시 연관 데이터 삭제 및 익명화 동작

정확한 `supabase test db` / `pg_prove` 실행 방법은 `supabase/tests/README.md`를 참고하세요. 실행 전 `supabase start`와 `supabase db reset`이 필요합니다.

### E2E Smoke Test (Playwright) — Supabase 프로젝트 및 Anthropic API Key 필요

```bash
npm run test:e2e
```

`e2e/smoke.spec.ts`는 다음 사용자 흐름을 검증합니다.

`연령 확인 → ai_processing 동의 → 채팅 → 추천`

또한 미성년자 차단 및 동의 여부에 따른 접근 제한도 확인합니다.

테스트 대상 환경이 설정되어 있지 않으면 `test.skip()`으로 명확하게 건너뜁니다. 실제 성공으로 위장하지 않습니다.

설정 방법은 `e2e/README.md`를 참고하세요. 이메일 확인을 비활성화한 전용 테스트 Supabase 프로젝트와 실제 `ANTHROPIC_API_KEY`가 필요합니다.

### 아직 자동화되지 않은 테스트

현재 아래 항목은 자동화하지 않았으며, 누락된 테스트로 명시적으로 관리하고 있습니다.

- AC-2 응답 지연 기준(첫 토큰 < 1.5초, p95 < 4초)은 `/api/chat`을 대상으로 한 k6 또는 Artillery 기반 부하 테스트가 필요합니다.
- AD-1 데이터 추출 재시도 및 멱등성(`reenqueue_stale_extractions`, dead-letter 처리)은 아직 자동화 테스트가 없습니다.

## PWA (AC-5)

- `public/manifest.json` — 앱 이름, 아이콘(192/512, `any maskable`), standalone 표시 방식, 테마 및 배경색을 정의합니다.
- `public/icons/icon-192.png` / `icon-512.png` — `node scripts/generate-pwa-icons.mjs`로 생성합니다. 별도 의존성 없이 PNG를 생성하며, 브랜드 인디고 배경과 중앙 흰색 원을 maskable 아이콘 안전 영역 안에 배치합니다. 디자인이 변경되면 스크립트를 다시 실행하면 됩니다.
- `public/sw.js` — 앱 셸 오프라인 fallback을 제공합니다. `/api/*`는 의도적으로 캐시하지 않습니다. 채팅, 인증 및 민감한 응답은 항상 네트워크 요청을 사용합니다.
- `app/layout.tsx` — manifest 링크, viewport, `appleWebApp` 메타데이터를 설정합니다.
- `app/service-worker-register.tsx` — 클라이언트에서 `/sw.js`를 등록합니다.
- 설치 가능 여부는 Chrome DevTools의 Lighthouse → PWA 항목에서 확인할 수 있습니다. `next build && next start`로 실행한 프로덕션 서버를 기준으로 검사해야 하며, Service Worker는 HTTPS 또는 `localhost` 환경이 필요합니다.

### 반응형 UI 검증

다음 모든 경로를 375px 이하(iPhone SE 수준)와 데스크톱 환경에서 확인했습니다.

`/`, `/sign-in`, `/sign-up`, `/onboarding`, `/consent`, `/chat`, `/recommendations`, `/admin/metrics`

모든 페이지는 모바일 우선 Tailwind 구성을 사용합니다.

- `max-w-*` + `mx-auto` 컨테이너
- `px-*` 패딩
- `flex-col sm:flex-row`
- `grid-cols-1 sm:grid-cols-*`
- `min-h-dvh` / `h-dvh`

고정 픽셀 너비나 가로 스크롤을 유발하는 패턴은 발견되지 않았습니다. `playwright.config.ts`에는 E2E Smoke Test에서 지속적으로 확인할 수 있도록 375px 전용 프로젝트가 포함되어 있습니다.

## 안전 및 개인정보 보호 설계

> 아래 항목은 서비스의 핵심 안전 설계입니다. 변경할 경우 관련 기획 및 설계 문서도 함께 수정해야 합니다.

- **위기 감지 (AD-2 / AC-12)**: 모든 사용자 메시지는 AI 동반자가 응답하기 전에 빠른 분류기와 키워드 fallback(`lib/safety/crisis-core.ts`)을 통해 검사합니다. 분류기가 오류 또는 시간 초과가 발생하면 일반 응답을 보여주지 않고 안전 안내 및 상담전화 정보를 우선 표시하는 fail-safe 방식으로 동작합니다. MVP에는 실제 상담원이 실시간 개입하는 기능이 없으며, 상담전화 안내가 최종 escalation 단계입니다.
- **국외 데이터 이전 동의 (AC-13)**: 사용자가 `ai_processing` 동의를 명시적으로 허용하기 전까지 `app/onboarding`에서 채팅을 차단합니다. 해당 동의 화면에서는 메시지가 미국의 Anthropic으로 전송된다는 점을 안내하며, 첫 번째 메시지를 보내기 전에 반드시 동의해야 합니다.
- **동의 이력 관리 (AD-3)**: `consent_events`는 append-only 구조입니다. 현재 동의 상태는 `current_consents` View에서 계산합니다. 동의 철회 시 기존 이력을 수정하지 않고 `revoke` 행을 추가합니다.
- **실명 저장 보호 (AD-4)**: `profiles.real_name`은 활성화된 `institution_sharing` 동의가 존재할 때만 저장할 수 있습니다. `SECURITY DEFINER` Postgres 트리거인 `guard_realname`으로 DB 레벨에서 강제하므로 직접 DB 접근이나 service-role 요청에도 적용됩니다.
- **삭제 및 보존 정책 (AC-14 / RC-2)**: 계정 삭제 시 `conversations`, `messages`, `difficulty_data`, `emotional_states`, `routines`, `recommendation_events` 데이터를 완전히 삭제합니다. `consent_events`와 `crisis_events`는 예외적으로 법적 동의 증빙 및 위기 대응 기록을 보존하기 위해 완전 삭제 대신 `user_id` / `message_id`를 `NULL`로 변경해 익명화합니다.
- **성인 사용자 제한 (AD-8 / AC-15)**: 온보딩 단계에서 사용자가 직접 성인 여부를 확인하도록 합니다. MVP에서는 별도 신분증 인증을 하지 않으며, 이는 기획 문서에 잔여 위험으로 기록되어 있습니다. 성인이 아니라고 응답하면 `age_verified`가 설정되지 않고 채팅 대신 상담전화 정보를 포함한 차단 화면으로 이동합니다.
- **RLS (AD-5)**: 사용자 범위 데이터가 있는 모든 테이블은 `user_id = auth.uid()`를 적용합니다. `user_id` 컬럼이 없는 `messages` 테이블은 부모 `conversations` 행을 경유한 `EXISTS` 조건으로 접근을 제한합니다. `extraction_status`는 RLS가 활성화되어 있지만 사용자 정책은 없으며 service-role만 접근할 수 있도록 기본 거부 방식으로 구성합니다.
- **암호화 (AD-6)**: Supabase의 디스크/볼륨 수준 저장 데이터 암호화에 의존합니다. 컬럼 단위 암호화는 사용하지 않았습니다. 제품의 핵심 품질 지표인 집계형 quality/quantity 쿼리를 방해하기 때문입니다.
- **관리자 대시보드 (AD-5)**: `/admin/metrics`에는 집계 데이터만 표시합니다. 사용자별 상세 조회 및 원문 메시지/컨텍스트는 제공하지 않습니다. `lib/metrics/admin-auth.ts`의 `ADMIN_EMAILS` allowlist를 사용하며 RLS가 아닌 애플리케이션 레벨 인증입니다.

## T8 단계에서 확인된 알려진 이슈

- `app/page.tsx`의 `내 기록 보기` 버튼은 `/journal`로 연결되지만 현재 코드베이스에는 `app/journal` 경로가 존재하지 않습니다.
- AC-3의 "대화 및 루틴 저장 후 `/journal`에서 시간순 조회" 기능은 랜딩 페이지에서 참조하고 있으나 T1~T7 단계에서 실제 페이지가 구현되지 않았습니다.
- 따라서 현재 `/journal`은 dead link 상태입니다.
- T8의 범위는 테스트/QA/PWA이며 신규 기능 페이지 구현이 아니므로, 해당 문제는 별도의 후속 작업으로 관리하는 것이 적절합니다.
