# 알뜰살뜰 (littleRecords) — CLAUDE.md

> 항상 이 파일을 먼저 읽고 시작한다.
> 도메인별 세부 규칙은 각 디렉토리의 CLAUDE.md를 참조한다.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트명 | 알뜰살뜰 (littleRecords) |
| 목적 | 구매내역 캡처 이미지 → AI 파싱 → 가계부 관리 |
| 타겟 | 부부 2인 (iOS, 개인 서버) |
| 개발자 | 42세 남성, 맥북 프로, 바이브 코딩 방식 |
| 버전 | v0.1.0 |

---

## 2. 기술 스택

```
Frontend   : Next.js 14 (App Router) + TypeScript
Styling    : Tailwind CSS + shadcn/ui
DB/Auth    : Supabase PostgreSQL + Supabase Auth
Storage    : Supabase Storage (이미지, 지식 문서)
이미지 AI  : Google Gemini 2.5 Flash
텍스트 AI  : Cerebras (llama3.1-8b, 무료)
배포       : Vercel
```

---

## 3. 프로젝트 구조

```
littleRecords/
├── CLAUDE.md                         ← 이 파일
├── app/
│   ├── layout.tsx                    ← 루트 레이아웃
│   ├── page.tsx                      ← /dashboard 리다이렉트
│   ├── globals.css
│   ├── (auth)/login/page.tsx         ← 로그인
│   ├── (app)/                        ← 인증 필요 영역 (공통 레이아웃)
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── upload/page.tsx
│   │   ├── purchases/page.tsx
│   │   ├── ai-chat/page.tsx
│   │   └── settings/page.tsx
│   └── api/
│       ├── analyze/route.ts          ← Gemini 이미지 분석
│       ├── blog/route.ts             ← GitHub 블로그 포스트 목록/내용
│       ├── chat/route.ts             ← Cerebras 텍스트 Q&A
│       ├── chat/history/route.ts     ← 채팅 기록 조회·삭제
│       ├── purchases/route.ts        ← 구매내역 CRUD
│       ├── purchases/[id]/route.ts
│       ├── purchases/export/route.ts ← CSV / JSON 내보내기
│       ├── knowledge/route.ts        ← 지식문서 CRUD
│       ├── knowledge/[id]/route.ts
│       ├── knowledge/import/route.ts ← Storage → DB 동기화
│       ├── knowledge/sync/route.ts   ← 고아 레코드 정리
│       ├── logs/route.ts             ← 로그 조회
│       ├── logs/summary/route.ts     ← 월별 사용량 집계
│       └── prompts/route.ts          ← 프롬프트 CRUD
│
├── components/
│   ├── layout/
│   │   ├── BottomNav.tsx
│   │   └── PageHeader.tsx            ← sticky 헤더 공통 컴포넌트
│   ├── dashboard/
│   │   ├── SummaryCards.tsx
│   │   ├── MonthlyChart.tsx
│   │   ├── CategoryChart.tsx
│   │   ├── ShopChart.tsx
│   │   └── RecentPurchases.tsx
│   ├── upload/
│   │   ├── ImageUploader.tsx
│   │   ├── AnalysisResult.tsx
│   │   ├── UploadProgress.tsx
│   │   └── KnowledgeManager.tsx      ← 지식문서 관리 UI
│   ├── purchases/
│   │   ├── PurchaseList.tsx
│   │   ├── PurchaseRow.tsx
│   │   ├── PurchaseDetail.tsx        ← 상세 모달
│   │   └── AddPurchaseModal.tsx
│   ├── ai-chat/
│   │   └── ChatInterface.tsx
│   ├── settings/
│   │   ├── SettingsPanel.tsx         ← 기본/고급 탭 구조
│   │   ├── AiUsageSummary.tsx        ← 월별 AI 사용량
│   │   ├── PromptEditor.tsx          ← 프롬프트 편집
│   │   ├── KnowledgeSync.tsx         ← Storage 동기화
│   │   ├── LogViewer.tsx             ← AI/에러 로그
│   │   └── BlogViewer.tsx            ← 블로그 탭 마크다운 뷰어
│   └── ui/                           ← shadcn/ui
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts                 ← 브라우저용
│   │   └── server.ts                 ← 서버용 (Route Handler, Server Component)
│   ├── gemini.ts                     ← analyzeImage(), summarizeDocument()
│   ├── logger.ts                     ← logAI(), logError() — supabase 인스턴스 필요
│   ├── get-prompt.ts                 ← DB 우선, 파일 fallback
│   ├── cerebras-models.ts            ← 모델 목록 + localStorage key
│   ├── rate-limit.ts                 ← checkRateLimit(), rateLimitResponse()
│   ├── utils.ts                      ← formatPrice() 등
│   ├── guest-context.tsx             ← GuestContext + GuestProvider + useGuest
│   └── prompts/                      ← 기본 프롬프트 (DB 없을 때 fallback)
│       ├── analyze-image.md
│       ├── summarize-doc.md          ← {{TITLE}}, {{CATEGORY}}, {{CONTENT}}
│       └── chat-system.md            ← {{PURCHASE_LINES}}, {{KNOWLEDGE_SECTION}}
│
├── types/index.ts
└── supabase/migrations/
    ├── 001_initial.sql               ← purchases
    ├── 002_logs.sql                  ← ai_logs, error_logs
    ├── 003_knowledge.sql             ← knowledge_docs
    ├── 004_storage_policies.sql      ← Storage RLS
    ├── 005_knowledge_input_type.sql  ← input_type 컬럼
    ├── 006_prompts.sql               ← prompts (사용자 커스텀)
    ├── 007_duplicate_prevention.sql  ← purchases UNIQUE 제약
    └── 008_chat_history.sql          ← chat_messages
```

---

## 4. 핵심 패턴

### 인증
```typescript
const supabase = await createClient();
const { data: { user } } = await supabase.auth.getUser();
if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

### 로깅 — supabase 인스턴스를 반드시 넘긴다
```typescript
// ❌ logAI 내부에서 createClient() 재생성 → 응답 후 컨텍스트 소멸로 RLS 실패
// ✅ 이미 인증된 인스턴스 전달
logAI({ supabase, userId, type: 'chat', model, status: 'success', ... });
logError({ supabase, userId, source: 'api/chat', message, statusCode: 500 });
```

### 프롬프트 로드
```typescript
// DB에 사용자 커스텀 프롬프트 있으면 우선, 없으면 lib/prompts/*.md fallback
const prompt = await getPrompt(supabase, 'chat-system', userId);
```

### AI 함수 시그니처
```typescript
analyzeImage(base64, mimeType, prompt)
summarizeDocument(title, category, content, promptTemplate)
```

---

## 5. DB 테이블 요약

| 테이블 | 용도 |
|---|---|
| `purchases` | 구매내역 (date, shop, item, price, discount 등) |
| `knowledge_docs` | 지식문서 (title, category, content, input_type) |
| `ai_logs` | AI 호출 로그 (type, model, status, input, output) |
| `error_logs` | 에러 로그 |
| `prompts` | 사용자 커스텀 프롬프트 (key, content) |
| `chat_messages` | AI 채팅 기록 (role, content, sources, reasoning) |

모든 테이블: RLS 활성화, `auth.uid() = user_id` 정책.

---

## 6. 환경변수

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
GEMINI_API_KEY=
CEREBRAS_API_KEY=
NEXT_PUBLIC_APP_VERSION=0.1.0
NEXT_PUBLIC_APP_NAME=알뜰살뜰
```

---

## 7. 카테고리

```typescript
// 구매 카테고리
["가전","가구","침실","주방·식기","욕실","인테리어","생활용품","수납·정리","식품","소품·기타"]

// 지식문서 카테고리
["카카오톡 대화","브랜드·제품 정보","예산 계획","쇼핑 목록","기타"]
```

---

## 8. 코딩 컨벤션

- TypeScript strict mode
- 함수형 컴포넌트 + React Hooks
- 모바일 퍼스트 (max-w-md, 하단 탭 고정)
- API 키는 Route Handler에서만 사용 (클라이언트 노출 금지)
- 금액: 정수(원 단위), 날짜: YYYY-MM-DD
- 주석 없이 명확한 네이밍으로

---

## 9. 도메인별 CLAUDE.md

| 파일 | 담당 영역 |
|---|---|
| `app/api/CLAUDE.md` | API 라우트 패턴, 에러 처리 |
| `components/CLAUDE.md` | UI 컴포넌트 패턴, shadcn 사용법 |
| `lib/CLAUDE.md` | 유틸 라이브러리, 프롬프트 시스템 |
| `supabase/CLAUDE.md` | DB 스키마 전문, 마이그레이션 규칙 |
