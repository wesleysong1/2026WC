# 전주신흥중학교 승부예측 이벤트 (2026 월드컵 한국 vs 남아공)

`index.html` 한 파일만 있으면 됩니다. 빌드 도구 없이 GitHub Pages에 바로 올릴 수 있고, 데이터 저장은 **Supabase**를 사용합니다.

---

## 1단계 · GitHub Pages에 올리기

1. GitHub 로그인 → 우측 상단 **+** → **New repository**
2. 이름 입력 (예: `worldcup-quiz`), **Public** 선택 → **Create repository**
3. 저장소에서 **Add file → Upload files** → `index.html` 업로드 → **Commit changes**
4. **Settings → Pages** 이동
5. Source를 **Deploy from a branch**, Branch를 **main / (root)** → **Save**
6. 잠시 후 `https://<내아이디>.github.io/<저장소이름>/` 주소가 생깁니다. 이 링크를 학생들에게 공유하세요.

> Supabase를 설정하지 않으면 **각 기기에만 따로 저장**되어 학급 전체 집계가 모이지 않습니다. 여러 학생의 제출을 한곳에 모으려면 아래 2단계를 진행하세요.

---

## 2단계 · Supabase 연결 (학급 전체 집계용, 무료)

1. https://supabase.com 에서 프로젝트 생성 (Region은 `Northeast Asia (Seoul)` 권장)
2. 좌측 **SQL Editor**에서 아래 SQL을 실행해 테이블과 접근 정책을 만듭니다.

```sql
-- 제출 테이블 (학번+이름 복합 기본키 → 학번 9999 처럼 같은 학번 여러 명도 정상 처리)
create table if not exists predictions (
  student_id   text not null,
  name         text not null,
  score_kr     int  not null,
  score_sa     int  not null,
  outcome      text not null,
  scorer       text not null,
  submitted_at timestamptz default now(),
  primary key (student_id, name)
);

-- 접근 권한 (RLS)
alter table predictions enable row level security;

-- 학생: 제출(insert)과 현황 조회(select) 허용
create policy "anyone can read"   on predictions for select using (true);
create policy "anyone can insert" on predictions for insert with check (true);
```

> **이미 학번 단독 기본키로 테이블을 만드셨다면**, 아래 SQL로 복합키로 변경하세요(데이터가 없을 때 권장):
>
> ```sql
> alter table predictions drop constraint predictions_pkey;
> alter table predictions add primary key (student_id, name);
> ```

> 누구나 읽고 쓸 수 있는 개방 정책입니다. 가벼운 학급 이벤트에 적합합니다.
> 같은 학번으로는 두 번 제출되지 않습니다 (`student_id`가 기본키라 중복이 차단됨).
> 수정·삭제 권한은 부여하지 않았으므로 학생이 남의 기록을 바꿀 수 없습니다. 이벤트가 끝나면 위 정책을 삭제해 잠그세요.

> **정답자 확인 기능을 쓰려면** 아래 테이블도 함께 만드세요 (경기 결과 1행 저장용):
>
> ```sql
> create table if not exists answer_key (
>   id text primary key default 'main',
>   score_kr int, score_sa int, scorer text,
>   updated_at timestamptz default now()
> );
> alter table answer_key enable row level security;
> create policy "answer read"   on answer_key for select using (true);
> create policy "answer insert" on answer_key for insert with check (true);
> create policy "answer update" on answer_key for update using (true) with check (true);
> ```

> **관리자 수동 추가 기능을 쓰려면** 아래 테이블도 만드세요 (명단에 없던 참여자를 관리자모드에서 추가):
>
> ```sql
> create table if not exists roster_extra (
>   student_id text not null,
>   name text not null,
>   primary key (student_id, name)
> );
> alter table roster_extra enable row level security;
> create policy "roster read"   on roster_extra for select using (true);
> create policy "roster insert" on roster_extra for insert with check (true);
> ```

3. 좌측 **Project Settings → API** 에서 두 값을 복사합니다.
   - **Project URL** (예: `https://abcdefgh.supabase.co`)
   - **anon public** key
4. `index.html` 상단을 열어 교체합니다.

```js
const SUPABASE_URL = "PASTE_PROJECT_URL";    // ← Project URL
const SUPABASE_ANON_KEY = "PASTE_ANON_KEY";  // ← anon public key
```

5. 다시 GitHub에 업로드(덮어쓰기)하면 끝. 화면 상단의 "localStorage 모드" 안내가 사라지면 정상 연결된 것입니다.

---

## 사용 방법

- **📋 경기 정보 & 진출 경우의 수** 탭: A조 1·2차전 결과와 순위표, 두 팀 정보, 한국의 토너먼트 진출 시나리오
- **⚽ 승부예측 참여하기** 탭: 학번·이름 입력 → 스코어 예측 → 첫 득점 선수 선택 → 제출
  - 명단(학번·이름)에 있는 학생만 참여할 수 있고, 학번과 이름이 일치하지 않으면 경고가 뜹니다.
  - 같은 학번+이름으로는 한 번만 참여할 수 있습니다 (중복·수정 불가).
  - 참여 마감 시각(2026-06-25 09:00 KST)이 지나면 참여 버튼이 자동으로 닫힙니다. 마감 시각은 `index.html` 상단의 `DEADLINE` 에서 변경할 수 있고, 명단은 `ROSTER` 에서 수정합니다.
- **🏆 정답자 확인 탭**: 경기 종료 후 관리자가 실제 스코어와 첫 득점 선수를 입력하면, 전체 제출을 자동 채점해 (1) 스코어+첫득점 모두 정답, (2) 정확한 스코어 정답, (3) 첫 득점 선수 정답 명단을 보여줍니다. 결과 입력은 같은 관리자 비밀번호로 보호됩니다.
- **참여자 수동 추가**: 승부예측 화면의 🔒 관리자 영역에서 학번·이름을 입력해 명단에 없던 학생/교사를 즉시 추가할 수 있습니다 (모든 기기에 반영).
- **관리자**: 참여 화면 맨 아래 **🔒 관리자** → 비밀번호 입력 → 전체 제출 현황 확인 및 **엑셀 다운로드**
  - 비밀번호는 `index.html` 상단 `ADMIN_PASSCODE = "admin2026"` 에서 변경하세요.
  - Supabase 대시보드의 Table Editor에서도 데이터를 직접 보거나 CSV로 내려받을 수 있습니다.

---

## 자주 묻는 것

- **화면이 잠깐 비어 보여요** — 처음 열 때 CDN에서 라이브러리를 받느라 1~2초 걸릴 수 있습니다.
- **제출이 안 돼요 / 콘솔에 오류** — SQL의 RLS 정책(insert/select)이 적용됐는지, URL·anon key가 정확한지 확인하세요.
- **엑셀이 안 받아져요** — 제출 데이터가 0건이면 버튼이 비활성화됩니다. 새로고침 후 다시 시도하세요.
- **localStorage 모드 안내가 떠요** — Supabase 설정 전이라는 뜻입니다. 2단계를 완료하면 사라집니다.
