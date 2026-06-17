# weddingCard — CLAUDE.md

> 항상 이 파일을 먼저 읽고 시작한다.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트명 | weddingCard |
| 목적 | 성룡 & 주영의 감성 모바일 청첩장 |
| 타겟 | 결혼식 하객 (모바일 웹) |
| 버전 | v0.0.1 |
| 상태 | 개발 시작 전 |
| 본식 | 2026년 9월 19일 (토) |

---

## 2. 기술 스택

```
Frontend  : Next.js (App Router) + TypeScript
Styling   : Tailwind CSS + 커스텀 CSS (Keyframe 애니메이션)
DB        : Supabase PostgreSQL (방명록)
Storage   : Supabase Storage (게스트스냅)
배포      : Vercel
참석여부  : 구글 폼 외부 링크
```

---

## 3. 프로젝트 구조

```
weddingCard/
├── CLAUDE.md
├── app/
│   ├── page.tsx                  ← 메인 청첩장 (섹션 S01~S16 전체)
│   ├── guest-snap/page.tsx       ← 게스트스냅 서브페이지
│   ├── layout.tsx
│   ├── globals.css
│   └── api/
│       ├── guestbook/route.ts    ← GET·POST (공개), DELETE (service_role)
│       └── guest-snap/route.ts   ← 파일 업로드 → Supabase Storage
│
├── components/
│   ├── EntryPopup.tsx            ← 진입 팝업 (오늘하루 안보기 + 음악 시작)
│   ├── MusicButton.tsx           ← 우측 상단 음악 재생/정지
│   ├── FloatingActions.tsx       ← 우측 하단 위로가기 + 공유
│   └── sections/
│       ├── S01_HeroCover.tsx
│       ├── S02_Invitation.tsx
│       ├── S03_People.tsx
│       ├── S04_Interview.tsx     ← 바텀시트 모달
│       ├── S05_Ceremony.tsx      ← 9월 달력 (19일 골드 원)
│       ├── S06_DDay.tsx
│       ├── S07_OurStory.tsx      ← 지그재그 타임라인
│       ├── S08_Gallery.tsx       ← 라이트박스
│       ├── S09_GuestSnap.tsx
│       ├── S10_InfoTabs.tsx      ← 3탭
│       ├── S11_Location.tsx      ← 네이버지도 iframe
│       ├── S12_Attendance.tsx
│       ├── S13_Gift.tsx          ← 계좌 아코디언 + 복사
│       ├── S14_Guestbook.tsx
│       ├── S15_Together.tsx      ← 초 단위 카운터
│       └── S16_Ending.tsx
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   └── server.ts
│   └── constants.ts              ← 날짜, 이름, 장소, 계좌 등 모든 상수
│
└── public/
    ├── audio/bgm.mp3
    └── gallery/
```

---

## 4. DB 테이블

### wedding_guestbook
```sql
CREATE TABLE wedding_guestbook (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name       text NOT NULL,
  message    text NOT NULL,
  created_at timestamptz DEFAULT now()
);
-- RLS 비활성: 누구나 읽기·쓰기 가능
-- 삭제는 service_role key로만 처리 (API 라우트 서버 사이드)
```

### Supabase Storage
```
버킷명: guest-snap (공개)
파일명: {Date.now()}_{originalName}
```

---

## 5. 환경변수

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_WEDDING_DATE=2026-09-19
NEXT_PUBLIC_FIRST_MET_DATE=2022-12-24
NEXT_PUBLIC_ATTENDANCE_FORM_URL=    # 구글폼 URL
NEXT_PUBLIC_APP_NAME=weddingCard
```

---

## 6. 핵심 패턴

### 배경음악 — 팝업 닫기 시 재생
```tsx
// EntryPopup.tsx
const audioRef = useRef<HTMLAudioElement>(null)

const handleClose = () => {
  localStorage.setItem('popup-hidden-date', today)
  setVisible(false)
  audioRef.current?.play()  // 사용자 인터랙션 후 재생 → 브라우저 정책 통과
}

return (
  <>
    <audio ref={audioRef} src="/audio/bgm.mp3" loop />
    {visible && <div className="popup">...</div>}
  </>
)
```

### 오늘하루 안보기
```tsx
useEffect(() => {
  const today = new Date().toISOString().slice(0, 10)
  if (localStorage.getItem('popup-hidden-date') === today) setVisible(false)
}, [])
```

### 바텀시트 모달 (S04)
```tsx
const [open, setOpen] = useState(false)
<div className={`fixed bottom-0 inset-x-0 transition-transform duration-300
  ${open ? 'translate-y-0' : 'translate-y-full'}`}>
```

### 스크롤 fade-in
```tsx
const ref = useRef(null)
useEffect(() => {
  const observer = new IntersectionObserver(
    ([entry]) => { if (entry.isIntersecting) entry.target.classList.add('visible') },
    { threshold: 0.2 }
  )
  if (ref.current) observer.observe(ref.current)
  return () => observer.disconnect()
}, [])
```

### 초 단위 카운터 (S15)
```tsx
const [elapsed, setElapsed] = useState(0)
useEffect(() => {
  const start = new Date(process.env.NEXT_PUBLIC_FIRST_MET_DATE!).getTime()
  const timer = setInterval(() => setElapsed(Math.floor((Date.now() - start) / 1000)), 1000)
  return () => clearInterval(timer)
}, [])
```

### 계좌번호 복사
```tsx
navigator.clipboard.writeText(accountNumber).then(() => toast('복사되었습니다'))
```

### 게스트스냅 혼례일 전 비활성
```tsx
const isActive = new Date() >= new Date(process.env.NEXT_PUBLIC_WEDDING_DATE!)
if (!isActive) return <p>혼례일 당일부터 업로드 가능합니다</p>
```

### Supabase 클라이언트에서 사용
```typescript
import { createClient } from '@/lib/supabase/client'
const supabase = createClient()
```

### Supabase 서버에서 사용
```typescript
import { createClient } from '@/lib/supabase/server'
const supabase = await createClient()
```

---

## 7. 코딩 컨벤션

- 컴포넌트: PascalCase, `components/` 폴더
- 섹션 컴포넌트: `S[번호]_[이름].tsx` (예: `S01_HeroCover.tsx`)
- 상수(날짜·이름·장소·계좌): 모두 `lib/constants.ts` 에 모아서 관리
- API 핸들러: `app/api/[resource]/route.ts`
- Supabase: 서버 컴포넌트·API 라우트는 `server.ts`, 클라이언트는 `client.ts`
- 포인트 컬러: CSS 변수 `--gold: #C4A882` 사용

---

## 8. 자주 쓰는 명령어

```bash
pnpm dev    # 개발 서버
pnpm build  # 빌드
pnpm lint   # 린트
```

---

## 9. 다음 단계 (Next Steps)

- [ ] Next.js 프로젝트 초기화 (`pnpm create next-app`)
- [ ] Supabase `wedding_guestbook` 테이블 생성
- [ ] Supabase Storage `guest-snap` 버킷 생성 (공개)
- [ ] `lib/constants.ts` 에 날짜·이름·장소·계좌 채우기
- [ ] Vercel 환경변수 설정
- [ ] 참석여부 구글폼 생성 후 URL 등록
- [ ] 섹션 순서대로 구현 (S01 → S16)
- [ ] 진입 팝업 + 배경음악 연동 (마지막에 처리)
- [ ] 배경음악 파일 `public/audio/bgm.mp3` 등록
- [ ] 갤러리 사진 `public/gallery/` 등록
- [ ] 네이버 지도 장소 ID 확인 (S11)
- [ ] 카카오페이 송금 링크 확인 (S13)
