# CLAUDE.md — CNEC 커머스 플랫폼 (크넥)

> 운영사: 주식회사 하우파파 (HOWPAPA Inc.)
> 브랜드: 크넥 (CNEC)
> 레포: `mktbiz-byte/cnec-shop` → `frontend/kviewshop`

---

## PART 1: 코딩 규칙 & 프로젝트 설정

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

| | 🔥 공구 (GONGGU) | 💜 크리에이터픽 (PICK) |
|---|---|---|
| 성격 | 기간한정 특별가 (3-7일) | 상시 큐레이션 |
| 주도 | 브랜드 캠페인 생성 | 크리에이터 자유 선택 |
| 전환율 | 예상 5-10% | 1-3% |

### 핵심 보호 원칙

- **앙영작가** 크리에이터는 크넥 핵심 파트너 — 별도 보호 정책 적용
- 크리에이터 정산은 반드시 **익월 20일 지급 보장**
- **비회원 구매 반드시 지원** (인스타 → 바로 구매 전환율 핵심)

---

### 프로젝트 구조

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
├── next.config.ts
├── netlify.toml
└── package.json
```

### 핵심 명령어

```bash
cd frontend/kviewshop
npm run dev              # 개발 서버
npm run build            # 프로덕션 빌드 (배포 전 반드시)
npm run type-check
npm run lint

npx supabase start
npx supabase db push
npx supabase migration new <migration_name>
```

### 환경 변수 (.env.local 필수)

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

### CRITICAL — 반드시 준수 (위반 시 보안 사고)

#### 1. Supabase Auth: `getUser()` 강제

```typescript
// ❌ NEVER — JWT 위조 가능
const { data: { session } } = await supabase.auth.getSession()

// ✅ ALWAYS — 서버에서 토큰 검증
const { data: { user }, error } = await supabase.auth.getUser()
```

#### 2. 포트원 웹훅 HMAC-SHA256 서명 검증 필수

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

#### 3. `/api/seed` 프로덕션 노출 금지

```typescript
if (process.env.NODE_ENV === 'production') {
  return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
}
```

#### 4. 환경 변수 fallback 하드코딩 금지

```typescript
// ❌ const SECRET = process.env.TOSS_SECRET_KEY || 'test_sk_zXLk...'
// ✅ const SECRET = process.env.TOSS_SECRET_KEY
//    if (!SECRET) throw new Error('TOSS_SECRET_KEY is required')
```

#### 5. `SUPABASE_SERVICE_ROLE_KEY` 서버 전용

- `NEXT_PUBLIC_` 접두사 **절대 금지**
- Route Handler, Edge Function, Server Component에서만 사용

---

### 아키텍처 & 패턴

#### Supabase 클라이언트 사용

```typescript
import { createClient } from '@/lib/supabase/client'      // 'use client'
import { createClient } from '@/lib/supabase/server'      // 서버/Route Handler
import { createServerClient } from '@supabase/ssr'         // 미들웨어 전용
```

#### 재고 차감 — 원자적 처리 필수

```typescript
// ✅ Supabase RPC만 사용
const { data, error } = await supabase.rpc('decrement_stock', { product_id: id, quantity })
```

```sql
CREATE OR REPLACE FUNCTION decrement_stock(product_id UUID, quantity INTEGER)
RETURNS JSONB AS $$
DECLARE current_stock INTEGER;
BEGIN
  SELECT stock INTO current_stock FROM products WHERE id = product_id FOR UPDATE;
  IF current_stock < quantity THEN
    RETURN '{"success": false, "reason": "insufficient_stock"}'::JSONB;
  END IF;
  UPDATE products SET stock = stock - quantity WHERE id = product_id;
  RETURN '{"success": true}'::JSONB;
END;
$$ LANGUAGE plpgsql;
```

#### 결제 플로우 (포트원 V2)

```
1. [Client] 구매하기 클릭
2. [Client → Server] POST /api/orders → 주문 생성 (PENDING)
3. [Client → PortOne] 결제 사전 등록 + SDK 호출
4. [PortOne → Server] 웹훅 POST /api/payments/webhook
5. [Server] HMAC 서명 검증 → 금액/주문번호/상태 일치 확인
6. [Server] 성공: PAID / 실패: 포트원 취소 API 호출
7. [Server] 브랜드·구매자 알림 발송
```

---

### 코딩 컨벤션

- **페이지**: `src/app/[locale]/(group)/path/page.tsx`
- **컴포넌트**: `src/components/[domain]/ComponentName.tsx`
- **훅**: `src/hooks/use-hook-name.ts`
- **유틸**: `src/lib/utils/function-name.ts`
- **타입**: `src/types/database.ts` (DB), `src/types/[domain].ts` (도메인)
- 서버 컴포넌트 기본 / 클라이언트는 인터랙션 필요 시만 `'use client'`
- 서버 상태: **React Query** / 클라이언트 전역: **Zustand**
- 모든 페이지에 `error.tsx` + `loading.tsx` 필수
- 다국어: **next-intl** — 번역 추가 시 `ko.json` 먼저 (현재 ~60% 누락)

```tsx
// error.tsx 표준
'use client'
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen gap-4">
      <h2 className="text-xl font-semibold">문제가 발생했습니다</h2>
      <p className="text-muted-foreground">{error.message}</p>
      <button onClick={reset} className="btn-primary">다시 시도</button>
    </div>
  )
}
```

### Git Workflow

- Write clear, descriptive commit messages focused on **why**, not just what
- Use conventional commit prefixes: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `style:`
- Keep commits atomic — one logical change per commit
- Do not force-push to shared branches

---

### 데이터베이스 규칙

#### 마이그레이션: 번호 중복 주의 (001~003 각 2개 존재)

```bash
npx supabase migration new feature_name  # 타임스탬프 자동
```

#### RLS — 모든 테이블 필수

```sql
CREATE POLICY "creator_own_data" ON creator_shop_items
  FOR ALL USING (creator_id = auth.uid());

CREATE POLICY "brand_own_products" ON products
  FOR ALL USING (brand_id = (SELECT id FROM brands WHERE user_id = auth.uid()));

CREATE POLICY "shop_public_read" ON creator_shop_items
  FOR SELECT USING (true);
```

#### 주문 상태 머신

```
PENDING → PAID → PREPARING → SHIPPING → DELIVERED → CONFIRMED
CANCELLED ← (PENDING/PAID/PREPARING에서만)
         → REFUNDED
```

---

### 비즈니스 로직

- **커미션**: 직접 전환 (브랜드 설정률) + 간접 전환 (일괄 3%, 24시간 쿠키)
- **산정 기준**: 실결제만 포함. 배송비/쿠폰/포인트/취소환불 제외
- **어트리뷰션**: 24시간 쿠키, 라스트 클릭, 배송완료+7일 확정
- **공구**: 진행 중 취소 불가 / 재고 소진 자동 SOLD_OUT / 마감 5분 유예
- **정산**: 월 1회 익월 20일 / ₩1,000 미만 이월 / 비사업자 3.3% / 사업자 VAT

### API & 도메인

```
✅ /api/payments/ /api/orders/ /api/brand/ /api/creator/ /api/shop/ /api/admin/ /api/upload/
❌ /api/payment/ (단수형 금지)
```

| 도메인 | 라우트 | 설명 |
|---|---|---|
| shop.cnec.kr | `/(shop)/[username]/*` | 크리에이터 샵 (SSR) |
| brand.cnec.kr | `/(brand)/brand/*` | 브랜드 어드민 |
| creator.cnec.kr | `/(creator)/creator/*` | 크리에이터 센터 |
| admin.cnec.kr | `/admin/*` | 운영 어드민 |

### 외부 서비스

#### 포트원 V2

```typescript
import PortOne from '@portone/browser-sdk/v2'
await portone.requestPayment({
  storeId, channelKey, paymentId: `CNEC-${Date.now()}`,
  orderName: '크넥 주문', totalAmount: amount, currency: 'KRW', payMethod: 'CARD',
})
```

#### Supabase Storage

```typescript
// 버킷: product-images, profile-images, cover-images
await supabase.storage.from('product-images')
  .upload(`${brandId}/${Date.now()}-${file.name}`, file)
```

#### Sweet Tracker (서버 전용)

```typescript
await fetch(`https://info.sweettracker.co.kr/api/v1/trackingInfo?t_key=${SWEET_TRACKER_API_KEY}&t_code=${courierCode}&t_invoice=${trackingNumber}`)
```

---

### TODO (보안/기능 우선순위)

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

> 배포 최소: 🔴 4건 + 🟠 재고 RPC + API 경로 통일

---

### 절대 하지 말 것

1. `components/ui/*` 직접 수정 → `npx shadcn-ui@latest add`
2. `types/database.ts` 수동 편집 → `npx supabase gen types typescript`
3. 클라이언트에서 `SUPABASE_SERVICE_ROLE_KEY`
4. 서버에서 `auth.getSession()` → `auth.getUser()`
5. 결제 클라이언트만 검증 → 서버 재검증 필수
6. 재고 단순 UPDATE → RPC
7. 환경 변수 fallback 하드코딩
8. 마이그레이션 수동 번호 → `supabase migration new`

### 새 기능 체크리스트

```
□ DB 마이그레이션 (타임스탬프)  □ RLS 정책  □ database.ts 재생성
□ Route Handler  □ error.tsx + loading.tsx  □ ko.json + en.json
□ OG 메타태그  □ .env.example 업데이트  □ npm run build
```

### 테스트 계정 (로컬 전용, /api/seed)

```
brand@test.cnec.kr / creator@test.cnec.kr / admin@test.cnec.kr / buyer@test.cnec.kr
공통: test1234!
```

---

## PART 2: 개발 기획서 (PRD v1.0)

### 1. 전체 플로우

**[브랜드]**
1. 크넥에 브랜드 등록
2. 상품 등록 (가격, 이미지, 상세)
3. 캠페인 생성 — 공구: 기간 + 특별가 + 커미션 / 상시: 커미션만
4. 모집 방식 선택 — 자유참여 / 승인제

**[크리에이터]**
1. 캠페인 탐색
2. 공구: 참여 신청 → (승인) → 내 샵에 자동 추가
3. 크리에이터픽: 상품 검색 → 내 샵에 직접 추가
4. SNS에 내 샵 링크 공유

**[팔로워]**
1. 크리에이터 샵 방문 (`shop.cnec.kr/beautyjin`)
2. 상품 선택 → 구매 (크넥 PG, 비회원 가능)

**[브랜드]** → 주문 알림 수신 → 배송 처리
**[크넥]** → 전환 추적 (직접 + 간접) → 커미션 계산 → 정산

### 2. 수익 구조

#### 커미션 체계

| 유형 | 조건 | 수수료율 |
|---|---|---|
| 직접 전환 | 크리에이터 샵에서 해당 상품 구매 | 브랜드가 설정 (5~40%) |
| 간접 전환 | 샵 방문 후 24시간 내 다른 상품 구매 | 일괄 3% |

#### 수익 예시

```
팔로워가 뷰티진 샵 방문
→ 하우파파 로션 구매 (₩24,700, 커미션 15%)
  → 직접: ₩3,705
→ 누씨오 토너패드 추가 구매 (₩18,000)
  → 간접: ₩540
→ 총 커미션: ₩4,245
```

#### 플랫폼 수수료

| 수수료 | 부담 | 비율 |
|---|---|---|
| PG 수수료 | 브랜드 | 약 3% |
| 플랫폼 수수료 | 브랜드 | 매출의 3-5% |

---

### 3. 브랜드 어드민

#### 메뉴 구조

```
브랜드 어드민 (brand.cnec.kr)
├── 대시보드
├── 상품 관리 ▼
│   ├── 전체 상품
│   └── 상품 등록
├── 캠페인 관리 ▼
│   ├── 공구 캠페인
│   ├── 상시 캠페인
│   └── 캠페인 생성
├── 주문 관리
├── 크리에이터 ▼
│   ├── 참여 현황
│   └── 승인 대기
├── 정산
└── 설정
```

#### 상품 등록 필드

**기본 정보**

| 필드 | 타입 | 필수 |
|---|---|---|
| 상품명 | Text (최대 100자) | Y |
| 카테고리 | Select (스킨케어/메이크업/헤어/바디) | Y |
| 정가 | Number | Y |
| 판매가 | Number | Y |
| 재고 | Number | Y |

**상세 정보**

| 필드 | 타입 | 필수 |
|---|---|---|
| 대표 이미지 | Image | Y |
| 추가 이미지 | Images | N |
| 상품 설명 | HTML Editor | Y |
| 용량/수량 | Text | Y |
| 성분 / 사용법 | Text | N |

**판매 설정**

| 필드 | 설명 |
|---|---|
| 판매 상태 | Toggle 판매중/판매중지 |
| 크리에이터픽 허용 | Toggle ON: 크리에이터가 자유 추가 |
| 기본 커미션율 | 크리에이터픽 시 적용 (%) |

#### 캠페인 생성 (공구) — 5단계

1. **기본 정보** — 캠페인명, 설명, 시작/종료일시
2. **상품/가격** — 대상 상품, 공구가, 할인율(자동계산), 한정 수량
3. **커미션** — 커미션율(%), 크리에이터당 판매 한도
4. **모집 방식** — 자유참여/승인제, 목표 인원, 조건
5. **홍보 키트** — 제품 이미지, 스토리 템플릿, 추천 멘트, 해시태그

#### 주문 관리

- **목록 컬럼**: 주문번호, 주문일시, 상품(상품명+수량), 구매자(이름+연락처), 크리에이터, 금액, 상태
- **상세**: 배송지 정보, 배송 처리(송장 입력), 주문 취소/환불 처리

#### 실시간 대시보드 (공구)

```
┌─────────────────────────────────────────┐
│ 🔥 하우파파 2월 공구                     │
│ ⏰ 남은 시간: 1일 05:32:15              │
│ 방문: 2,847 | 주문: 127 | 전환율: 4.5%  │
│ 매출: ₩3.1M | 커미션: ₩465K             │
│ [크리에이터별 판매 랭킹]                 │
│ 1. @beautyjin (32건, ₩790K)            │
│ 2. @skinhana (28건, ₩691K)             │
└─────────────────────────────────────────┘
```

---

### 4. 크리에이터 센터

#### 메뉴 구조

```
크리에이터 센터 (creator.cnec.kr)
├── [프로필] (이름 + 샵 링크)
├── 상품 관리 ▼
│   └── 전체 상품 (브랜드 상품 검색 + 내 샵 추가)
├── 내 셀렉트샵 ▼
│   ├── 샵 정보 (프로필, 커버, 설명)
│   ├── 컬렉션 관리 (테마별 상품 모음)
│   └── 뷰티 루틴 관리
├── 캠페인 ▼
│   ├── 공구 캠페인 (참여 가능한 공구)
│   └── 내 캠페인 (참여 중인 캠페인)
├── 판매 현황
├── 정산 관리
└── 설정
```

#### 상품 관리

- 검색/필터: 수수료 높은순, 인기순, 신상품순 / 브랜드 / 카테고리
- 수수료 금액 미리 표시: "15% → ₩3,705"
- 액션: [내 샵에 추가] 또는 [공구 참여 신청]

#### 샵 정보

| 항목 | 사양 | 필수 |
|---|---|---|
| 커버 이미지 | 1200×300px, 3MB 이하 | N |
| 프로필 이미지 | 160×160px, 1MB 이하 | N |
| 샵 이름 | 최대 20자 | Y |
| 샵 설명 | 최대 100자 | Y |
| 대표 채널 URL | 인스타, 유튜브 등 | Y |

#### 뷰티 프로필 태그

| 카테고리 | 선택 | 옵션 |
|---|---|---|
| 피부타입 | 1개 | 복합성, 건성, 지성, 중성, 수부지 |
| 퍼스널 컬러 | 1개 | 봄웜, 여름쿨, 가을웜, 겨울쿨 |
| 피부 고민 | 여러개 | 아토피, 여드름, 민감성, 미백/잡티, 피지/블랙헤드, 다크서클, 속건조, 주름/탄력, 모공, 홍조, 각질 |
| 두피 고민 | 여러개 | 탈모, 손상모, 두피트러블, 열감두피, 지성두피, 가려움, 비듬/각질 |

모바일 프리뷰: 편집 시 우측에 실시간 미리보기 표시

#### 컬렉션 관리

| 항목 | 사양 | 필수 |
|---|---|---|
| 내 샵에 노출 | 토글 ON/OFF | N |
| 컬렉션 이름 | 최대 20자 | Y |
| 컬렉션 설명 | 최대 100자 | N |

상품 추가: 검색 → 체크박스 선택 → 추가 / 순서 드래그 변경

#### 뷰티 루틴

| 항목 | 사양 | 필수 |
|---|---|---|
| 루틴 이름 | 최대 20자 | Y |
| 단계 이름 | 최대 20자 | Y |
| 단계 설명 | 최대 500자 | Y |
| 이미지 | 5MB 이하, 1장 | Y |
| 상품 태그 | 이미지 위에 상품 태그 | N |

단계 순서: 위/아래 화살표 / "단계 추가 +" 버튼 / 삭제 버튼

#### 캠페인 참여

- **승인제**: 참여 신청 → 승인 대기 → 승인 시 내 샵에 자동 추가
- **자유참여**: 바로 참여 → 내 샵에 즉시 추가

#### 판매 현황

- **대시보드**: 방문 수, 판매 건수, 전환율, 판매 금액, 정산 예정
- **상세**: 결제 일시, 상품명, 판매 금액, 전환 유형(직접/간접), 수수료율, 정산 예정

#### 정산 관리

- **탭**: 전체, 정산완료, 이월, 소멸
- **컬럼**: 정산 기간, 구매 수, 정산대상 금액(VAT포함), 수수료, 원천징수, 정산금액, 상태
- 엑셀 다운로드 지원

---

### 5. 크리에이터 샵 (구매자 화면)

#### 샵 페이지 구성

```
shop.cnec.kr/{creator_id}
┌─────────────────────────────────────────┐
│ [커버 이미지]                            │
│ [프로필] 뷰티진의 셀렉트샵              │
│ "제가 직접 써보고 추천해요"             │
│ 피부타입: 복합성 | 퍼스널: 봄웜톤        │
│ @beautyjin (인스타 링크)                │
│                                         │
│ ┌──────────────┐ ┌──────────────────┐   │
│ │ 🔥 공구       │ │ 💜 크리에이터픽   │   │
│ └──────────────┘ └──────────────────┘   │
│                                         │
│ [공구 탭]                               │
│ 🔥 하우파파 2월 공구  D-3 | 35% OFF     │
│ ₩38,000 → ₩24,700  [구매하기]          │
│                                         │
│ [크리에이터픽 탭]                       │
│ • 민감피부 추천 컬렉션 (5개)            │
│ • 뷰티 루틴                             │
│ [배너]                                  │
└─────────────────────────────────────────┘
```

#### 상품 상세 페이지

```
shop.cnec.kr/beautyjin/product/{product_id}

[상품 이미지 슬라이드]
브랜드명 / 상품명
₩38,000 → ₩24,700 (35% OFF)
🔥 공구 마감까지: 1일 05:32:15
[수량] - 1 +
[구매하기]
─────────────────
[상품 설명] / [배송 안내] / [교환/환불 안내]
```

#### 구매 플로우

1. [구매하기] → 로그인 (비회원 구매 가능) → 배송지 → 결제 → 주문 완료
2. 브랜드에 주문 전달 → 배송

#### 트래킹

```
[방문] shop.cnec.kr/beautyjin → creator_id 쿠키 (24시간) → 방문 로그
[구매] 결제 완료 → 쿠키에서 creator_id 확인 → 직접/간접 전환 → Conversion 기록
```

---

### 6. 정산 시스템

| 항목 | 내용 |
|---|---|
| 주기 | 월 1회, 익월 20일 |
| 비사업자 | 원천징수 3.3% |
| 사업자 | VAT 포함, 세금계산서 |

**프로세스**: 매월 1일 집계 → 5일 확정 (취소/환불 반영) → 10일 안내 → 사업자 세금계산서 → 20일 입금

**제외**: 취소/환불, 구매 미확정, 표시광고법 미준수, 콘텐츠 삭제/비공개

---

### 7. 데이터베이스 설계

#### User

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| email | VARCHAR UNIQUE | 이메일 |
| password_hash | VARCHAR | 비밀번호 |
| role | ENUM | BRAND, CREATOR, ADMIN |
| status | ENUM | PENDING, ACTIVE, SUSPENDED |
| name | VARCHAR | 이름 |
| phone | VARCHAR | 전화번호 |
| created_at | TIMESTAMP | 생성일시 |

#### Brand

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| user_id | UUID FK | User 연결 |
| brand_name | VARCHAR | 브랜드명 |
| logo_url | TEXT | 로고 |
| business_number | VARCHAR | 사업자번호 |
| bank_name / bank_account | VARCHAR | 정산 계좌 |
| platform_fee_rate | DECIMAL | 플랫폼 수수료율 |

#### Creator

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| user_id | UUID FK | User 연결 |
| shop_id | VARCHAR UNIQUE | 샵 ID (URL용) |
| display_name | VARCHAR | 샵 이름 |
| bio | TEXT | 샵 설명 |
| profile_image_url / cover_image_url | TEXT | 이미지 |
| instagram_handle | VARCHAR | 인스타 핸들 |
| skin_type | ENUM | 피부타입 |
| personal_color | ENUM | 퍼스널 컬러 |
| skin_concerns | TEXT[] | 피부 고민 |
| total_sales / total_earnings | DECIMAL | 누적 매출/수익 |
| bank_name / bank_account | VARCHAR | 정산 계좌 |
| is_business | BOOLEAN | 사업자 여부 |

#### Product

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| brand_id | UUID FK | 브랜드 |
| name / category / description | VARCHAR/TEXT | 상품 정보 |
| original_price / sale_price | DECIMAL | 가격 |
| stock | INTEGER | 재고 |
| images | TEXT[] | 이미지 URLs |
| status | ENUM | ACTIVE, INACTIVE |
| allow_creator_pick | BOOLEAN | 크리에이터픽 허용 |
| default_commission_rate | DECIMAL | 기본 커미션율 |

#### Campaign

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| brand_id | UUID FK | 브랜드 |
| type | ENUM | GONGGU, ALWAYS |
| title | VARCHAR | 캠페인명 |
| status | ENUM | DRAFT, RECRUITING, ACTIVE, ENDED |
| start_at / end_at | TIMESTAMP | 기간 |
| recruitment_type | ENUM | OPEN, APPROVAL |
| commission_rate | DECIMAL | 커미션율 |
| total_stock / sold_count | INTEGER | 수량/판매량 |

#### CampaignProduct

`campaign_id`, `product_id`, `campaign_price`, `per_creator_limit`

#### CampaignParticipation

`campaign_id`, `creator_id`, `status` (PENDING/APPROVED/REJECTED)

#### CreatorShopItem

`creator_id`, `product_id`, `campaign_id`, `type` (GONGGU/PICK), `collection_id`, `display_order`

#### Collection

`creator_id`, `name`, `description`, `is_visible`, `display_order`

#### Order

| 컬럼 | 타입 | 설명 |
|---|---|---|
| id | UUID PK | 고유 식별자 |
| order_number | VARCHAR UNIQUE | CNEC-YYYYMMDD-XXXXX |
| creator_id / brand_id | UUID FK | 관계 |
| buyer_name / buyer_phone / buyer_email | VARCHAR | 구매자 |
| shipping_address | TEXT | 배송지 |
| total_amount / product_amount / shipping_fee | DECIMAL | 금액 |
| status | ENUM | PENDING→PAID→PREPARING→SHIPPING→DELIVERED→CONFIRMED→CANCELLED→REFUNDED |

#### OrderItem

`order_id`, `product_id`, `campaign_id`, `quantity`, `unit_price`, `total_price`

#### Conversion

`order_id`, `creator_id`, `conversion_type` (DIRECT/INDIRECT), `commission_rate`, `commission_amount`, `status` (PENDING/CONFIRMED/CANCELLED)

#### Settlement

`creator_id`, `period`, `total_sales`, `direct_commission`, `indirect_commission`, `withholding_tax`, `net_amount`, `status`

#### ShopVisit

`creator_id`, `visitor_id`, `ip_address`, `user_agent`, `referer`, `visited_at`, `expires_at`

#### PromotionKit

`campaign_id`, `product_images`, `story_templates` (JSONB), `recommended_caption`, `hashtags` (TEXT[])

---

### 8. API 명세

#### 인증

| Endpoint | Method | 설명 |
|---|---|---|
| /auth/register | POST | 회원가입 |
| /auth/login | POST | 로그인 |
| /auth/refresh | POST | 토큰 갱신 |

#### 브랜드

| Endpoint | Method | 설명 |
|---|---|---|
| /brand/dashboard | GET | 대시보드 |
| /brand/products | GET/POST | 상품 목록/등록 |
| /brand/products/:id | GET/PUT/DELETE | 상품 상세 |
| /brand/campaigns | GET/POST | 캠페인 목록/생성 |
| /brand/campaigns/:id | GET/PUT | 캠페인 상세 |
| /brand/campaigns/:id/participants | GET | 참여자 |
| /brand/campaigns/:id/participants/:cid | PUT | 승인/거절 |
| /brand/orders | GET | 주문 목록 |
| /brand/orders/:id | GET/PUT | 주문 상세/배송처리 |
| /brand/settlements | GET | 정산 |

#### 크리에이터

| Endpoint | Method | 설명 |
|---|---|---|
| /creator/dashboard | GET | 대시보드 |
| /creator/shop | GET/PUT | 샵 정보 |
| /creator/shop/items | GET/POST/DELETE | 샵 상품 |
| /creator/collections | GET/POST | 컬렉션 |
| /creator/collections/:id | GET/PUT/DELETE | 컬렉션 상세 |
| /creator/products | GET | 전체 상품 검색 |
| /creator/campaigns | GET | 캠페인 탐색 |
| /creator/campaigns/:id/apply | POST | 캠페인 참여 |
| /creator/participations | GET | 내 캠페인 |
| /creator/sales | GET | 판매 현황 |
| /creator/settlements | GET | 정산 |

#### 샵 (Public)

| Endpoint | Method | 설명 |
|---|---|---|
| /shop/:shop_id | GET | 샵 정보 |
| /shop/:shop_id/items | GET | 샵 상품 (공구/픽) |
| /shop/:shop_id/collections | GET | 컬렉션 |
| /shop/:shop_id/products/:id | GET | 상품 상세 |

#### 주문

| Endpoint | Method | 설명 |
|---|---|---|
| /orders | POST | 주문 생성 |
| /orders/:id | GET | 주문 조회 |
| /orders/:id/pay | POST | 결제 |
| /payments/webhook | POST | PG 웹훅 |

---

### 9. 개발 일정 (총 12주)

| Phase | 주차 | 작업 |
|---|---|---|
| 1. 기반 | Week 1 | 프로젝트 셋업, DB, 인증 |
| | Week 2 | 브랜드 어드민 (상품 등록) |
| | Week 3 | 브랜드 어드민 (캠페인, 주문) |
| 2. 크리에이터 | Week 4 | 크리에이터 센터 (샵 정보) |
| | Week 5 | 크리에이터 센터 (컬렉션, 상품 추가) |
| | Week 6 | 크리에이터 센터 (캠페인 참여) |
| | Week 7 | 크리에이터 센터 (판매 현황, 정산) |
| 3. 샵+결제 | Week 8 | 크리에이터 샵 (메인, 상품 상세) |
| | Week 9 | 결제 시스템 (PG 연동) |
| | Week 10 | 주문 → 브랜드 전달 연동 |
| 4. 완성 | Week 11 | 트래킹, 커미션, 정산 |
| | Week 12 | QA, 파일럿 테스트 |

### 10. KPI

- **파일럿**: 브랜드 2개 (하우파파, 누씨오), 크리에이터 20명, 월 GMV ₩1,000만, 전환율 3%+
- **6개월**: 브랜드 20개+, 크리에이터 200명, 월 GMV ₩1억

---

## PART 3: 화해 벤치마크 — UI/기능 분석

### 편집샵센터 메뉴 구조

```
편집샵센터
├── [큐레이터 프로필] (이름 + 링크 아이콘)
├── 상품 관리 ▼
│   └── 전체 상품 (링크 생성/검색)
├── 내 편집샵 관리 ▼
│   ├── 편집샵 정보
│   ├── 컬렉션 관리
│   └── 뷰티 루틴 관리 [New]
├── 판매 현황
├── 정산 관리
└── 아이디어 [New]
```

### 상품 관리 — 전체 상품

- 검색/필터: 급상승 랭킹 · 누적 랭킹 · 실시간 판매 BEST / 브랜드 / 카테고리 / 기획전 / 수수료 높은순
- 상품 리스트: 썸네일 + 상품명 + 카테고리 + 할인율 + 가격 + 수수료율 + 수수료 금액(원) + [링크 생성/확인]
- 총 16,468개 상품. 수수료 금액 원 단위 미리 표시 (예: 10% → 1,990원)

**상품 유형별 수수료:**

| 유형 | 뱃지 | 수수료 |
|---|---|---|
| only화해 | 빨간 뱃지 | 10% |
| 마켓 상품 | 별도 뱃지 | 20% |
| 기타 | 없음 | 7% |

### 편집샵 정보

우측 실시간 모바일 프리뷰 표시.

| 항목 | 사양 |
|---|---|
| 커버 이미지 | 1200×300px, 3MB |
| 프로필 이미지 | 160×160px, 1MB |
| 편집샵 이름 | 최대 20자 (필수) |
| 편집샵 설명 | 최대 100자 (필수) |
| 대표 채널 URL | 드롭다운 (필수) |

뷰티 프로필 태그, 배너(가로/세로형), 랜딩 링크 설정 가능.

### 컬렉션 관리

이름(20자), 설명(100자), 노출 토글. 상품 추가 모달: 검색 + 체크박스 복수 선택. 상품별 수수료율+금액 표시.

### 뷰티 루틴 관리 [New]

루틴 이름(20자) + 단계별 구성 (이름 20자, 설명 500자, 이미지 5MB 1장, 상품 태그, 링크, 크롭). 순서 변경 가능. 실시간 프리뷰.

### 판매 현황

지표: 클릭 건수, 판매 건수, 전환율, 판매 금액, 정산 예정 금액. 세부: 결제일시, 상품명, 판매 금액(배송비/쿠폰/포인트 제외), 수수료, 정산 예정.

### 정산 관리

필터: 전전월/전월/당월/직접설정. 탭: 전체/정산완료/이월/소멸. 컬럼: 정산기간, 상품명, 구매수, 정산대상금액(VAT), 수수료율, 수수료, 원천징수, 정산금액, 상태. 엑셀 다운로드.

### 큐레이터 홈 (소비자 페이지)

```
hwahae.co.kr/curators-home?curator_id={ID}
```

구성: 커버 → 프로필(이름+설명+뷰티태그+채널) → 탭(컬렉션|뷰티루틴) → 컬렉션 목록(가로 스크롤 카드) → 배너 → CTA

### 크넥 vs 화해 차별화

| 영역 | 화해 | 크넥 목표 |
|---|---|---|
| 상품 소싱 | 기존 카탈로그 | 독점 구성/가격 |
| 수익 모델 | 어필리에이트 7~20% | 공구 + 어필리에이트 하이브리드 |
| 온보딩 | 구글 폼 수동 | 자동화 파이프라인 |
| 콘텐츠 | 컬렉션+루틴 (정적) | 영상/숏폼 동적 콘텐츠 |
| 데이터 | 기본 판매현황 | 실시간 대시보드 + 심화 분석 |

---

## PART 4: 화해 벤치마크 — 수익/정산 분석

### 수수료 체계

| 기여 유형 | 조건 | 수수료율 |
|---|---|---|
| 직접 기여 | 링크 클릭 후 24시간 내 해당 상품 구매 | 상품 유형별 차등 |
| 간접 기여 | 링크 클릭 후 24시간 내 다른 상품 구매 | 일괄 3% |

직접 기여: 마켓 상품 20%, only화해 10%, 기타 7%

### 어트리뷰션 규칙

- 쿠키 24시간 / 라스트 클릭 / 하위 버전 앱 추적 불가
- 정산 시점 콘텐츠 삭제/비공개 → 수익 미인정
- 산정: 실결제만 포함, 배송비/쿠폰/포인트/취소환불 제외

### 링크 생성 플로우

편집샵센터에서 검색 → [링크 생성] → 표시광고법 준수 확인 팝업 → 링크 발급

**표시광고법**: 경제적 이해관계 문구 필수. 게시글 제목/첫 부분 표기. 미기재 시 수익 미인정/정산 취소.

**금지**: 화해 로고 무단사용, 사칭, 성인/불법 매체, 허위과장광고

### 계정 체계

- 화해 앱과 별도 회원가입 (이메일+비밀번호)
- 사업자 없이 가입 가능
- 비사업자: 원천징수 후 계좌 입금
- 사업자: 화해 역발행 세금계산서 → 큐레이터 승인 (익월 5일까지)

### 정산 정책

- 월 1회, 익월 20일 지급 (공휴일 시 직전 영업일)
- 정산 대상자 개별 안내 (수동 프로세스)
- 기본 정보 설정/비밀번호 재설정 수동 (자동화 미흡)

### 화해 강점 (벤치마킹)

- 직접/간접 이원화 수수료 → 수익 기회 확대
- 표시광고법 준수를 링크 생성에 내장 (법적 리스크 관리)
- 차등 수수료로 전략 상품 프로모션
- 콘텐츠 아이디어 가이드 제공

### 화해 약점 (개선 기회)

- 기본 정보 설정/비밀번호 재설정 수동
- 정산 대상자 개별 안내 (자동화 필요)
- 24시간 쿠키 비교적 짧음
- 하위 앱 추적 불가
- 대시보드 기능 제한적

### 크넥 핵심 차별화

- **독점 상품**: 크리에이터 협상력 활용 독점 구성/가격
- **공동구매**: 단순 어필리에이트 넘어 가격 경쟁력
- **자동화 정산**: 실시간 대시보드 + 자동 정산
- **Cafe24 연동**: 기존 이커머스 인프라 활용

### 용어 매핑 (화해 → 크넥)

| 화해 | 크넥 | 비고 |
|---|---|---|
| 큐레이터 | 크리에이터 | 크리에이터 중심 용어 |
| 편집샵센터 | 크리에이터 센터 | 상품+샵+정산 통합 |
| 수익화 링크 | 크리에이터 샵 URL | 개인샵이 트래킹 겸함 |
| 직접 기여 | 직접 전환 | 링크 상품 구매 |
| 간접 기여 | 간접 전환 | 링크 외 상품 구매 |
| 판매현황 | 성과 대시보드 | 실시간 추적 목표 |

---

## Important Notes for AI Assistants

- Always read files before modifying them
- Do not introduce security vulnerabilities (SQL injection, XSS, command injection, etc.)
- Do not over-engineer — implement only what is requested
- Do not add comments, docstrings, or type annotations to unchanged code
- Avoid creating abstractions for one-time operations
- When unsure about a convention, check existing code for patterns before asking
- Do not create README or documentation files unless explicitly requested
- Keep the directory structure flat where possible; nest only when logical grouping is needed
