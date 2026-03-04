# CLAUDE.md — CNEC 커머스 플랫폼 (크넥)

> **운영사:** 주식회사 하우파파 (HOWPAPA Inc.)
> **브랜드:** 크넥 (CNEC)
> **레포:** mktbiz-byte/cnec-shop → `frontend/kviewshop`

---

## 📋 프로젝트 컨텍스트

### 서비스 한 줄 정의

크리에이터가 자신만의 뷰티 셀렉트샵을 운영하고, 브랜드 상품 판매로 수익을 얻는 플랫폼

### 서비스 구조 (4자 에코시스템)

```
[브랜드] 상품 등록 + 캠페인 생성
    ↓
[크넥] 플랫폼 운영 + 결제(PG) + 정산
    ↓
[크리에이터] 개인 샵 운영 + SNS 홍보 (shop.cnec.kr/{shop_id})
    ↓
[팔로워] 크리에이터 샵에서 구매 → 브랜드 직접 배송
```

### 두 가지 상품 유형

|  | 🔥 공구 (GONGGU) | 💜 크리에이터픽 (PICK) |
|---|---|---|
| 성격 | 기간한정 특별가 (3-7일) | 상시 큐레이션 |
| 주도 | 브랜드 캠페인 생성 | 크리에이터 자유 선택 |
| 전환율 예상 | 5-10% | 1-3% |

### 핵심 보호 원칙

- **앙영작가** 크리에이터는 크넥 핵심 파트너 — 별도 보호 정책 적용
- 크리에이터 정산은 반드시 **익월 20일** 지급 보장
- **비회원 구매** 반드시 지원 (인스타 → 바로 구매 전환율 핵심)

### 참고 문서 (상세 기획/벤치마크)

- `docs/PRD_v1.0.md` — 개발 기획서 (전체 플로우, DB 설계, API 명세, UI 와이어프레임, 개발 일정)
- `docs/화해_벤치마크_UI.md` — 화해 편집샵센터 UI/기능 상세 분석
- `docs/화해_벤치마크_수익정산.md` — 화해 수수료 체계, 정산 정책, 어트리뷰션 규칙

---

## 🗂 프로젝트 구조

```
frontend/kviewshop/
├── src/
│   ├── app/
│   │   └── [locale]/
│   │       ├── (brand)/brand/*        # 브랜드 어드민 (brand.cnec.kr)
│   │       ├── (creator)/creator/*    # 크리에이터 센터 (creator.cnec.kr)
│   │       ├── (shop)/[username]/*    # 크리에이터 샵 (shop.cnec.kr) ← SSR 필수
│   │       ├── (buyer)/*              # 구매자 페이지
│   │       ├── (auth)/*               # 인증
│   │       ├── admin/*                # 크넥 운영 어드민
│   │       └── payment/*              # 결제 플로우
│   ├── components/
│   │   ├── ui/                        # shadcn/ui 전용 — 직접 수정 금지
│   │   ├── brand/
│   │   ├── creator/
│   │   ├── shop/
│   │   └── common/
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts              # 클라이언트 컴포넌트용
│   │   │   ├── server.ts              # 서버 컴포넌트/Route Handler용
│   │   │   └── middleware.ts          # 미들웨어용 — getUser() 사용 필수
│   │   ├── store/                     # zustand stores
│   │   └── i18n/config.ts
│   ├── types/database.ts              # 586줄 — Single Source of Truth
│   └── messages/                      # 11개 언어 JSON (ko, en, ja 우선)
├── supabase/migrations/               # SQL 마이그레이션 (번호 중복 주의)
├── docs/                              # PRD, 벤치마크 참고 문서
├── next.config.ts
├── netlify.toml
└── package.json
```

---

## ⚡ 핵심 명령어

```bash
cd frontend/kviewshop
npm run dev              # 개발 서버
npm run build            # 프로덕션 빌드 (배포 전 반드시)
npm run type-check       # 타입 체크
npm run lint             # 린트

npx supabase start
npx supabase db push
npx supabase migration new <migration_name>
```

---

## 🔐 환경 변수 (.env.local 필수)

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=           # 서버 전용. 클라이언트 절대 금지

# 결제 (포트원 V2)
PORTONE_API_SECRET=
PORTONE_STORE_ID=
PORTONE_CHANNEL_KEY=

# 외부 서비스
SWEET_TRACKER_API_KEY=
KAKAO_ALIMTALK_API_KEY=
RESEND_API_KEY=

# Toss (레거시 — 포트원 V2 전환 후 제거 예정)
TOSS_SECRET_KEY=                     # fallback 하드코딩 절대 금지
```

---

## 🚨 CRITICAL — 반드시 준수 (위반 시 보안 사고)

### 1. Supabase Auth: `getUser()` 강제

```typescript
// ❌ NEVER — JWT 위조 가능
const { data: { session } } = await supabase.auth.getSession()

// ✅ ALWAYS — 서버에서 토큰 검증
const { data: { user }, error } = await supabase.auth.getUser()
```

### 2. 포트원 웹훅 HMAC-SHA256 서명 검증 필수

```typescript
import crypto from 'crypto'

function verifyWebhookSignature(payload: string, signature: string, secret: string): boolean {
  const hmac = crypto.createHmac('sha256', secret)
  hmac.update(payload)
  const expected = hmac.digest('hex')
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))
}
// 검증 실패 → 400 반환, 절대 주문 처리 금지
```

### 3. `/api/seed` 프로덕션 노출 금지

```typescript
if (process.env.NODE_ENV === 'production') {
  return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
}
```

### 4. 환경 변수 fallback 하드코딩 금지

```typescript
// ❌
const SECRET = process.env.TOSS_SECRET_KEY || 'test_sk_zXLk...'

// ✅
const SECRET = process.env.TOSS_SECRET_KEY
//    if (!SECRET) throw new Error('TOSS_SECRET_KEY is required')
```

### 5. `SUPABASE_SERVICE_ROLE_KEY` 서버 전용

- `NEXT_PUBLIC_` 접두사 **절대 금지**
- Route Handler, Edge Function, Server Component에서만 사용

---

## 🏗 아키텍처 & 패턴

### Supabase 클라이언트 사용

```typescript
import { createClient } from '@/lib/supabase/client'      // 'use client'
import { createClient } from '@/lib/supabase/server'      // 서버/Route Handler
import { createServerClient } from '@supabase/ssr'         // 미들웨어 전용
```

### 재고 차감 — 원자적 처리 필수

```typescript
// ✅ Supabase RPC만 사용
const { data, error } = await supabase.rpc('decrement_stock', { product_id: id, quantity })
```

### 결제 플로우 (포트원 V2)

```
Client 구매 → POST /api/orders (PENDING) → PortOne SDK → 웹훅 → HMAC 검증 → PAID/취소
```

---

## 📐 코딩 컨벤션

- **서버 컴포넌트 기본** / 클라이언트는 인터랙션 필요 시만 `'use client'`
- **서버 상태:** React Query / **클라이언트 전역:** Zustand
- 모든 페이지에 `error.tsx` + `loading.tsx` 필수
- **다국어:** next-intl — 번역 추가 시 `ko.json` 먼저 (현재 ~60% 누락)

---

## 💾 데이터베이스

### 마이그레이션: 번호 중복 주의 (001~003 각 2개 존재)

```bash
npx supabase migration new feature_name  # 타임스탬프 자동
```

### RLS — 모든 테이블 필수

```sql
-- 크리에이터 자기 데이터만 / 브랜드 자기 상품만 / 샵은 퍼블릭 읽기
```

### 주문 상태 머신

```
PENDING → PAID → PREPARING → SHIPPING → DELIVERED → CONFIRMED
CANCELLED ← (PENDING/PAID/PREPARING에서만)
         → REFUNDED
```

---

## 🛒 비즈니스 로직

- **커미션:** 직접 전환 (브랜드 설정률) + 간접 전환 (일괄 3%, 24시간 쿠키)
- **산정 기준:** 실결제만 포함. 배송비/쿠폰/포인트/취소환불 제외
- **어트리뷰션:** 24시간 쿠키, 라스트 클릭, 배송완료+7일 확정
- **공구:** 진행 중 취소 불가 / 재고 소진 자동 SOLD_OUT / 마감 5분 유예
- **정산:** 월 1회 익월 20일 / ₩1,000 미만 이월 / 비사업자 3.3% / 사업자 VAT

---

## 📡 API & 도메인

### API 경로

```
✅ /api/payments/ /api/orders/ /api/brand/ /api/creator/ /api/shop/ /api/admin/ /api/upload/
❌ /api/payment/ (단수형 금지)
```

### 도메인 라우트

| 도메인 | 라우트 | 설명 |
|---|---|---|
| shop.cnec.kr | `/(shop)/[username]/*` | 크리에이터 샵 (SSR) |
| brand.cnec.kr | `/(brand)/brand/*` | 브랜드 어드민 |
| creator.cnec.kr | `/(creator)/creator/*` | 크리에이터 센터 |
| admin.cnec.kr | `/admin/*` | 운영 어드민 |

---

## ⚠️ TODO

| 등급 | 항목 | 상태 |
|---|---|---|
| 🔴 | Toss 키 fallback 제거 | 미완료 |
| 🔴 | 미들웨어 getUser() 변경 | 미완료 |
| 🔴 | 웹훅 HMAC 검증 구현 | 미완료 |
| 🔴 | /api/seed NODE_ENV 가드 | 미완료 |
| 🟠 | 재고 RPC 함수 | 미완료 |
| 🟠 | API 경로 통일 | 미완료 |
| 🟠 | error.tsx/loading.tsx 54p | 미완료 |
| 🟠 | .env.example | 미완료 |
| 🟡 | KO 번역 ~40% 보완 | 진행중 |
| 🟡 | 배송비 로직 (현재 0원) | 미완료 |
| 🟡 | Migration 번호 중복 | 미완료 |
| 🟡 | robots.txt, sitemap.xml | 미완료 |
| 🟡 | CSP 헤더 | 미완료 |

> **배포 최소:** 🔴 4건 + 🟠 재고 RPC + API 경로 통일

---

## 🚫 절대 하지 말 것

1. `components/ui/*` 직접 수정 → `npx shadcn-ui@latest add`
2. `types/database.ts` 수동 편집 → `npx supabase gen types typescript`
3. 클라이언트에서 `SUPABASE_SERVICE_ROLE_KEY`
4. 서버에서 `auth.getSession()` → `auth.getUser()`
5. 결제 클라이언트만 검증 → 서버 재검증 필수
6. 재고 단순 UPDATE → RPC
7. 환경 변수 fallback 하드코딩
8. 마이그레이션 수동 번호 → `supabase migration new`

---

## ✅ 새 기능 체크리스트

```
□ DB 마이그레이션 (타임스탬프)  □ RLS 정책  □ database.ts 재생성
□ Route Handler  □ error.tsx + loading.tsx  □ ko.json + en.json
□ OG 메타태그  □ .env.example 업데이트  □ npm run build
```

---

## 🧪 테스트 계정 (로컬 전용, /api/seed)

| 계정 | 비밀번호 |
|---|---|
| brand@test.cnec.kr | test1234! |
| creator@test.cnec.kr | test1234! |
| admin@test.cnec.kr | test1234! |
| buyer@test.cnec.kr | test1234! |
