# CLAUDE.md — 두리 (Duri) 웨딩 플래너

## 프로젝트 개요

성룡 & 주영의 결혼 준비를 함께 관리하는 커플 전용 웨딩 플래너 앱.
본식: **2026년 9월 19일 (토)**

핵심 가치: **두 사람이 같은 화면을 실시간으로 공유한다.**

---

## 기술 스택

- **Framework**: Next.js 14 App Router + TypeScript
- **Styling**: Tailwind CSS + shadcn/ui
- **DB / Auth**: Supabase (PostgreSQL + Row Level Security)
- **Realtime**: Supabase Realtime (Postgres Changes)
- **배포**: Vercel
- **패키지 매니저**: pnpm

---

## 프로젝트 구조

```
app/
├── (auth)/
│   └── login/page.tsx         ← 로그인 (이메일 + 구글)
├── (app)/
│   ├── layout.tsx             ← 하단 탭 네비게이션 포함
│   ├── page.tsx               ← 홈 (D-day 배너 + 요약 카드)
│   ├── calendar/page.tsx      ← 캘린더 탭
│   ├── timeline/page.tsx      ← 타임라인 탭
│   └── checklist/page.tsx     ← 체크리스트 탭
└── api/
    ├── tasks/route.ts         ← GET(목록) POST(추가)
    ├── tasks/[id]/route.ts    ← PATCH(수정/완료) DELETE(삭제)
    ├── events/route.ts        ← GET POST
    └── events/[id]/route.ts   ← PATCH DELETE

components/
├── DdayBanner.tsx             ← 상단 D-day 카운터
├── BottomNav.tsx              ← 하단 탭 바 (홈·캘린더·타임라인·체크리스트)
├── CalendarGrid.tsx           ← 월별 달력 그리드
├── EventPanel.tsx             ← 선택 날짜 일정 목록 + 추가 폼
├── UpcomingList.tsx           ← 다가오는 일정 카드
├── TimelineMonth.tsx          ← 월 섹션 (헤더 + 할 일 목록, 접기/펼치기)
├── TaskRow.tsx                ← 할 일 한 줄 (체크박스 + 제목 + 담당자)
├── ChecklistFilter.tsx        ← 필터 버튼 바
└── ProgressBar.tsx            ← 재사용 진행률 바

lib/
├── supabase/
│   ├── client.ts              ← createBrowserClient()
│   └── server.ts              ← createServerClient()
├── seed/
│   └── tasks.ts               ← 초기 할 일 48개 데이터 배열
│   └── events.ts              ← 초기 일정 7개 데이터 배열
├── hooks/
│   ├── useTasks.ts            ← tasks 조회 + Realtime 구독
│   └── useEvents.ts           ← events 조회 + Realtime 구독
└── types.ts                   ← Task, Event, Couple 타입 정의
```

---

## DB 스키마

### couples
```sql
CREATE TABLE couples (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user1_id      uuid REFERENCES auth.users NOT NULL,
  user2_id      uuid REFERENCES auth.users,
  wedding_date  date NOT NULL DEFAULT '2026-09-19',
  name1         text DEFAULT '성룡',
  name2         text DEFAULT '주영',
  created_at    timestamptz DEFAULT now()
);
ALTER TABLE couples ENABLE ROW LEVEL SECURITY;
CREATE POLICY "커플 멤버만 접근"
  ON couples FOR ALL
  USING (auth.uid() = user1_id OR auth.uid() = user2_id);
```

### tasks
```sql
CREATE TABLE tasks (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  couple_id     uuid REFERENCES couples(id) ON DELETE CASCADE NOT NULL,
  title         text NOT NULL,
  month         text CHECK (month IN ('6월','7월','8월','9월')) NOT NULL,
  priority      text CHECK (priority IN ('최우선','중요','진행')) NOT NULL,
  assignee      text CHECK (assignee IN ('성룡','주영','같이')) NOT NULL,
  detail        text,
  completed     boolean DEFAULT false NOT NULL,
  completed_at  timestamptz,
  sort_order    integer DEFAULT 0,
  created_at    timestamptz DEFAULT now()
);
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
CREATE POLICY "커플 멤버만 접근"
  ON tasks FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM couples
      WHERE couples.id = tasks.couple_id
        AND (couples.user1_id = auth.uid() OR couples.user2_id = auth.uid())
    )
  );
```

### events
```sql
CREATE TABLE events (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  couple_id     uuid REFERENCES couples(id) ON DELETE CASCADE NOT NULL,
  title         text NOT NULL,
  date          date NOT NULL,
  assignee      text CHECK (assignee IN ('성룡','주영','같이')) DEFAULT '같이',
  type          text CHECK (type IN ('예식장','촬영','피팅','미팅','예약','발송','기타')) DEFAULT '기타',
  memo          text,
  created_at    timestamptz DEFAULT now()
);
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY "커플 멤버만 접근"
  ON events FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM couples
      WHERE couples.id = events.couple_id
        AND (couples.user1_id = auth.uid() OR couples.user2_id = auth.uid())
    )
  );
```

---

## 초기 할 일 seed 데이터 (lib/seed/tasks.ts)

couple 생성 시 아래 48개를 일괄 insert.

```typescript
export const SEED_TASKS = [
  // ===== 6월 =====
  { title: '예식장 방문 일정 잡기',         month: '6월', priority: '최우선', assignee: '성룡', detail: '시식, 식사메뉴, 보증인원, 꽃장식, 야외세팅, 사회자, 정산 기준 확인' },
  { title: '최종 인원 통보 마감일 확인',     month: '6월', priority: '최우선', assignee: '성룡', detail: '하객명단·청첩장 수량·답례품 수량 산정의 기준' },
  { title: '본식 촬영 업체 예약',            month: '6월', priority: '최우선', assignee: '성룡', detail: '추천 업체 견적, 스냅/영상 포함 여부, 원본 제공, 보정 컷 수 확인' },
  { title: '하객 명단 재정리',               month: '6월', priority: '중요',   assignee: '같이', detail: '성룡측 / 주영측 / 양가 부모님측 분리' },
  { title: '청첩장 방식 결정',               month: '6월', priority: '중요',   assignee: '같이', detail: '손수건 100장 vs 종이 100장 vs 모바일 중심' },
  { title: '모바일 청첩장 기획',             month: '6월', priority: '중요',   assignee: '성룡', detail: '화면 구성, 사진, 문구, 약도, 계좌, 일정 정보 정리' },
  { title: '웨딩사진 셀렉 시작',             month: '6월', priority: '중요',   assignee: '같이', detail: '청첩장용 / 포토테이블용 / 보정용으로 분리' },
  { title: '드레스·예복 방향 논의',          month: '6월', priority: '진행',   assignee: '같이', detail: '대여/맞춤/기성복/간소화 여부 결정' },
  { title: '메이크업 예약 방식 확인',        month: '6월', priority: '진행',   assignee: '성룡', detail: '예식장 연계인지, 별도 예약인지 확인' },
  { title: '허니문 방향 1차 논의',           month: '6월', priority: '진행',   assignee: '같이', detail: '국내 vs 일본, 일정 길이, 예산 기준 결정' },
  // ===== 7월 =====
  { title: '상견례 일정·장소 확정',          month: '7월', priority: '최우선', assignee: '같이', detail: '7~8월 중 날짜 확정, 양가 참석자 확인' },
  { title: '하객 명단 1차 확정',             month: '7월', priority: '최우선', assignee: '같이', detail: '초대확정 / 고민 / 제외 3단계 분류' },
  { title: '양가 부모님 하객 명단 요청',     month: '7월', priority: '최우선', assignee: '같이', detail: '부모님 지인 수가 보증인원에 큰 영향' },
  { title: '청첩장 제작 착수',               month: '7월', priority: '최우선', assignee: '같이', detail: '제작업체 선정, 문구 확정, 수량 확정' },
  { title: '모바일 청첩장 개발',             month: '7월', priority: '최우선', assignee: '성룡', detail: '1차 버전 제작' },
  { title: '사진 보정 의뢰',                 month: '7월', priority: '중요',   assignee: '같이', detail: '최종 후보 10~20장 보정' },
  { title: '포토테이블 사진 후보 선정',      month: '7월', priority: '중요',   assignee: '같이', detail: '인화용 / 액자용 / 모바일용 구분' },
  { title: '드레스·예복 업체 후보 정리',     month: '7월', priority: '중요',   assignee: '같이', detail: '8월 피팅을 위해 후보·가격·일정 확인' },
  { title: '메이크업 업체 예약',             month: '7월', priority: '중요',   assignee: '같이', detail: '8월 확정 아닌 7월 예약 권장' },
  { title: '축가 후보 결정',                 month: '7월', priority: '진행',   assignee: '같이', detail: '지인 / 전문 / 생략 중 결정' },
  { title: '부모님 한복·예복 매장 후보 정리',month: '7월', priority: '진행',   assignee: '같이', detail: '8월 방문 전 후보 매장 선정' },
  { title: '허니문 방향 확정',               month: '7월', priority: '진행',   assignee: '같이', detail: '국내/일본 최종 선택, 기간·예산 결정' },
  // ===== 8월 =====
  { title: '청첩장 발송',                    month: '8월', priority: '최우선', assignee: '같이', detail: '8월 초~중 모바일 + 실물 발송' },
  { title: '하객 참석 여부 1차 체크',        month: '8월', priority: '최우선', assignee: '같이', detail: '답변 없음 / 참석 / 불참 분류' },
  { title: '드레스 피팅',                    month: '8월', priority: '최우선', assignee: '주영', detail: '액세서리, 헬퍼, 수선 일정 확인' },
  { title: '예복 피팅',                      month: '8월', priority: '최우선', assignee: '성룡', detail: '셔츠, 타이, 구두, 양말 포함 확정' },
  { title: '메이크업 최종 확정',             month: '8월', priority: '최우선', assignee: '같이', detail: '시작 시간, 장소, 이동 동선 확정' },
  { title: '본식 스냅·영상 촬영팀 최종 확인',month: '8월', priority: '중요',   assignee: '성룡', detail: '촬영 범위, 가족사진, 납품일 확인' },
  { title: '부모님 한복·예복 준비',          month: '8월', priority: '중요',   assignee: '같이', detail: '양가 일정 맞춰 피팅 진행' },
  { title: '부케 확정',                      month: '8월', priority: '중요',   assignee: '주영', detail: '색감, 형태, 부토니에 포함 여부' },
  { title: '꽃장식 추가 여부 확인',          month: '8월', priority: '중요',   assignee: '성룡', detail: '야외 공간, 포토테이블, 버진로드 기준' },
  { title: '축가 확정',                      month: '8월', priority: '진행',   assignee: '같이', detail: '곡명, MR, 리허설 여부' },
  { title: '허니문 예약',                    month: '8월', priority: '진행',   assignee: '성룡', detail: '항공, 숙소, 교통, 여행자보험' },
  { title: '답례품 후보 선정',               month: '8월', priority: '진행',   assignee: '같이', detail: '종류, 수량, 단가, 배송일 확인' },
  { title: '식전영상 제작 여부 결정',        month: '8월', priority: '진행',   assignee: '같이', detail: '제작한다면 사진·음악·문구 준비' },
  // ===== 9월 =====
  { title: '최종 하객수 확인 & 예식장 통보', month: '9월', priority: '최우선', assignee: '성룡', detail: '보증인원, 식권, 좌석 수 확정' },
  { title: '예식장 최종 미팅',               month: '9월', priority: '최우선', assignee: '같이', detail: '인원, 잔금, 주차, 당일 담당자 확인' },
  { title: '본식 타임라인 확정',             month: '9월', priority: '최우선', assignee: '같이', detail: '메이크업샵 → 웨딩홀 → 리허설 → 본식 → 식사인사' },
  { title: '당일 역할자 지정',               month: '9월', priority: '최우선', assignee: '같이', detail: '축의금, 식권, 귀중품, 부모님 케어 담당자 확정' },
  { title: '가족사진 순서표 작성',           month: '9월', priority: '중요',   assignee: '같이', detail: '양가 가족·친척 촬영 순서 정리' },
  { title: '방명록·예식 준비물 준비',        month: '9월', priority: '중요',   assignee: '같이', detail: '방명록, 펜, 봉투, 식권, 사례비' },
  { title: '답례품 확정·수령',               month: '9월', priority: '중요',   assignee: '같이', detail: '수량, 포장, 보관, 전달 방식' },
  { title: '촬영 요청 컷 리스트 전달 (D-10)',month: '9월', priority: '중요',   assignee: '같이', detail: '꼭 찍고 싶은 가족·친구·디테일 컷 목록 전달' },
  { title: '사례비 봉투 준비 (D-7)',         month: '9월', priority: '진행',   assignee: '성룡', detail: '축가, 헬퍼, 접수 도움 인원 등' },
  { title: '반지·예복·구두 최종 체크 (D-7)',  month: '9월', priority: '진행',   assignee: '성룡', detail: '반지, 셔츠, 타이, 양말, 구두' },
  { title: '신부 준비물 최종 체크 (D-7)',    month: '9월', priority: '진행',   assignee: '주영', detail: '드레스 속옷, 누브라, 스타킹, 보조신발 등' },
  { title: '허니문 짐 준비 (D-7)',           month: '9월', priority: '진행',   assignee: '같이', detail: '여권, 항공권, 환전, eSIM, 보험' },
  { title: '최종 리마인드 연락 (D-3)',       month: '9월', priority: '진행',   assignee: '같이', detail: '가족, 사회자, 축가, 접수 담당자 최종 확인' },
]
```

---

## 초기 일정 seed 데이터 (lib/seed/events.ts)

```typescript
export const SEED_EVENTS = [
  { title: '💍 본식',                 date: '2026-09-19', assignee: '같이', type: '예식장', memo: '본식 당일 — 2026년 9월 19일 (토)' },
  { title: '🏛️ 예식장 방문',          date: '2026-06-20', assignee: '성룡', type: '예식장', memo: '시식, 보증인원, 꽃장식, 정산 기준 확인' },
  { title: '📸 본식 촬영 업체 미팅',  date: '2026-06-30', assignee: '성룡', type: '촬영',   memo: '업체 견적 및 예약 확인' },
  { title: '🍽️ 상견례',               date: '2026-07-20', assignee: '같이', type: '미팅',   memo: '양가 부모님 상견례' },
  { title: '💌 청첩장 발송',          date: '2026-08-05', assignee: '같이', type: '발송',   memo: '모바일 + 실물 동시 발송' },
  { title: '👗 드레스 피팅',          date: '2026-08-15', assignee: '주영', type: '피팅',   memo: '액세서리, 헬퍼, 수선 일정 확인' },
  { title: '🤵 예복 피팅',            date: '2026-08-15', assignee: '성룡', type: '피팅',   memo: '셔츠, 타이, 구두, 양말 포함 확정' },
]
```

---

## UI 컴포넌트 동작 명세

### 홈 (/)
- 상단: D-day 배너 (본식까지 N일, 날짜 표시)
- 중단: 이번 달 진행률 카드 (완료 N / 전체 M)
- 하단: 이번 주 다가오는 일정 3개

### 캘린더 (/calendar)
- 상단: 월 네비게이션 (← 이전 / 다음 →)
- 중단: 7열 달력 그리드. 이벤트 있는 날에 컬러 dot 표시
- 날짜 클릭 시: 아래 패널에 해당 날짜 일정 + 추가 폼 슬라이드업
- 추가 폼: 제목 입력 + 담당자 select + 종류 select + 저장 버튼

### 타임라인 (/timeline)
- 완료된 항목 카드 (예식장 계약·반지·신혼집 등)
- 6월·7월·8월·9월 섹션 (기본 모두 펼침)
- 섹션 헤더: 월명 + 진행률 텍스트 + 진행률 바 + 화살표(접기)
- 섹션 내: 최우선·중요·진행 그룹 → TaskRow 목록

### 체크리스트 (/checklist)
- 상단: 전체 진행률 바 + 완료/전체 수
- 필터 바: 전체 / 성룡 / 주영 / 같이 / 미완료 / 완료
- 월 레이블 구분 후 TaskRow 나열

### TaskRow
- 체크박스 클릭 → PATCH /api/tasks/[id] → Supabase Realtime으로 상대방에게도 즉시 반영
- completed=true 시 줄긋기 + 흐리게

---

## Realtime 구독 패턴

```typescript
// hooks/useTasks.ts
useEffect(() => {
  const channel = supabase
    .channel('tasks-realtime')
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'tasks', filter: `couple_id=eq.${coupleId}` },
      (payload) => {
        // INSERT / UPDATE / DELETE 각각 처리
        if (payload.eventType === 'UPDATE') {
          setTasks(prev => prev.map(t => t.id === payload.new.id ? payload.new : t))
        }
      }
    )
    .subscribe()
  return () => { supabase.removeChannel(channel) }
}, [coupleId])
```

---

## 커플 연결 흐름

1. user1(성룡) 가입 → `/api/couples` POST → couples 생성 + tasks/events seed insert
2. 초대 링크 생성: `https://duri.app/invite?code={coupleId}`
3. user2(주영) 링크 클릭 → 가입 → couple.user2_id 업데이트
4. 이후 두 사람 모두 같은 couple_id 데이터에 접근

---

## 코딩 컨벤션

- 컴포넌트: PascalCase, `components/` 폴더
- API 핸들러: `app/api/[resource]/route.ts` 구조 준수
- 타입: `lib/types.ts`에 중앙 관리
- Supabase 클라이언트: 서버 컴포넌트는 `server.ts`, 클라이언트는 `client.ts` 엄격 구분
- 에러 처리: API 라우트는 항상 `try/catch` + 적절한 HTTP 상태코드 반환
- 색상: CSS 변수 `--primary`, `--primary-dark`, `--primary-light`, `--accent` 사용

---

## 자주 쓰는 명령어

```bash
pnpm dev          # 개발 서버
pnpm build        # 빌드
pnpm lint         # 린트
```

---

## 환경변수 (.env.local)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_WEDDING_DATE=2026-09-19
NEXT_PUBLIC_APP_VERSION=0.1.0
NEXT_PUBLIC_APP_NAME=두리
```
