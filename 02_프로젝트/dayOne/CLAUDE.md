# dayOne — CLAUDE.md

> 항상 이 파일을 먼저 읽고 시작한다.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트명 | dayOne |
| 목적 | 일일 기록 관리 (사진, 메모, iCal 생성) |
| 타겟 | 개인 서버, 클라이언트 |
| 버전 | v0.1.0 |
| 상태 | **배포 완료** (2026-06-01) |

### 인프라 현황

| 항목 | 상태 |
|---|---|
| Vercel 배포 | ✅ 완료 — GitHub 푸시 시 자동 배포 |
| Supabase DB | ✅ 연결 완료 — episodes 테이블 + RLS |
| Supabase Auth | ✅ 연결 완료 |
| Supabase Storage | ✅ 연결 완료 |
| 환경변수 (Vercel) | ✅ 설정 완료 |

---

## 2. 기술 스택

```
Frontend   : Next.js 16 (App Router) + TypeScript
Styling    : Tailwind CSS 4 + shadcn/ui
DB/Auth    : Supabase PostgreSQL + Supabase Auth
Storage    : Supabase Storage (이미지)
배포       : Vercel
```

---

## 3. 프로젝트 구조

```
dayOne/
├── CLAUDE.md                         ← 이 파일
├── src/
│   ├── proxy.ts                      ← 인증 게이트웨이 (로그인 리다이렉트)
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   ├── page.tsx                  ← 오늘의 기록
│   │   ├── login/page.tsx            ← 로그인 (이메일+비밀번호)
│   │   ├── history/page.tsx          ← 과거 기록 조회·수정
│   │   └── api/
│   │       ├── episodes/route.ts     ← 에피소드 목록 조회·생성 (GET, POST)
│   │       └── episodes/[id]/route.ts ← 에피소드 단건 조회·수정·삭제 (GET, PUT, DELETE)
│   ├── components/
│   │   ├── DateForm.tsx              ← 에피소드/활동 입력 폼 (EXIF 자동 입력, isEpisode 토글)
│   │   ├── EpisodeCard.tsx           ← 에피소드 카드
│   │   ├── EpisodeEditForm.tsx       ← 에피소드 수정 폼 + ICS 재다운로드
│   │   ├── IcsGenerator.tsx          ← ICS 다운로드 (에피소드는 Supabase 저장 포함)
│   │   └── PhotoPicker.tsx           ← 사진 선택 + EXIF 파싱
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts             ← 브라우저용
│   │   │   └── server.ts             ← 서버용
│   │   ├── exif.ts
│   │   ├── geocode.ts
│   │   ├── ics.ts
│   │   └── storage.ts
│   └── types/
│       └── episode.ts
│
├── supabase/
│   ├── migrations/
│   │   ├── 001_initial.sql           ← episodes 테이블 + RLS
│   │   └── 002_add_episode_columns.sql ← ep, area, place, trip 등 컬럼 추가
│   └── seed.sql                      ← ep 1~134 시드 데이터
│
├── .env.example
├── .env.local
└── package.json
```

---

## 4. DB 테이블

| 테이블 | 용도 |
|---|---|
| `episodes` | 에피소드 기록 (ep, date, area, place, trip, start_time, end_time, memo, location, lat, lng, photos) — 활동 기록은 Supabase 미저장 |

모든 테이블: RLS 활성화, `auth.uid() = user_id` 정책.

---

## 5. 환경변수

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_APP_VERSION=0.1.0
NEXT_PUBLIC_APP_NAME=dayOne
```

---

## 6. 핵심 패턴

### 클라이언트에서 Supabase 사용
```typescript
import { createClient } from '@/lib/supabase/client';

const supabase = createClient();
const { data: { user } } = await supabase.auth.getUser();
```

### 서버에서 Supabase 사용
```typescript
import { createClient } from '@/lib/supabase/server';

const supabase = await createClient();
const { data: { user } } = await supabase.auth.getUser();
```

---

## 7. 배포

### Vercel ✅
- GitHub 푸시 시 자동 배포 (main 브랜치)
- 환경변수 3개 설정 완료: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`

### Supabase ✅
- Production 데이터베이스 연결 완료
- `supabase/migrations/001_initial.sql` 적용 완료
- `supabase/migrations/002_add_episode_columns.sql` 적용 필요 (ep, area 등 컬럼 추가)
- RLS 정책 4개 활성화 (select / insert / update / delete)
- 인덱스: `episodes_user_id_ep_idx` (unique)

---

## 8. 다음 단계 (Next Steps)

- [x] 이메일+비밀번호 로그인 / proxy.ts 인증 게이트웨이
- [x] 에피소드 Supabase 저장·조회·수정 (CRUD)
- [x] 히스토리에서 수정 후 ICS 재다운로드
- [x] ep 1~134 시드 데이터 (supabase/seed.sql)
- [x] 에피소드/활동 구분 기록 (isEpisode 토글, 활동은 시간 지정 ICS만)
- [ ] supabase/migrations/002 및 seed.sql 실행 (Supabase SQL 에디터)
- [ ] Supabase Storage 버킷 생성 및 정책 설정 (사진 업로드)
- [ ] 커스텀 도메인 연결 (선택)
