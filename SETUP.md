# 세없결 실구동 세팅 가이드

## Supabase 프로젝트 정보
- **URL**: `https://yemxddkzgqzmmicnfakl.supabase.co`
- **Anon Key**: `.env.local` 파일 참고

---

## Step 1: DB 마이그레이션 실행

Supabase 대시보드 → SQL Editor → `saeobeoggyeol/supabase/migration.sql` 전체 붙여넣고 실행

---

## Step 2: Supabase 이메일 확인 비활성화 (개발 중)

Supabase 대시보드 → Authentication → Email → **"Confirm email"** 체크 해제

> 실서비스 전에는 다시 켜야 합니다

---

## Step 3: 관리자 계정 생성

Supabase 대시보드 → Authentication → Users → **Add user** 클릭

| 이메일 | 비밀번호 | 역할 |
|-------|---------|------|
| admin@seupkyul.com | (원하는 비밀번호) | super_admin |
| gangnam@seupkyul.com | (원하는 비밀번호) | branch_admin |

계정 생성 후 SQL Editor에서 역할 설정:
```sql
-- admin@seupkyul.com 의 UUID를 Supabase Auth > Users에서 확인 후 교체
UPDATE public.users SET role = 'super_admin', name = '관리자'
WHERE id = '여기에-UUID-붙여넣기';

UPDATE public.users SET role = 'branch_admin', name = '강남점주'
WHERE id = '강남점주-UUID-붙여넣기';
```

---

## Step 4: 지점 데이터 입력

SQL Editor에서 실제 지점 정보 입력:
```sql
INSERT INTO public.branches (name, address, region, phone, open_time, close_time)
VALUES
  ('강남점', '서울 강남구 테헤란로 123', '서울', '02-1234-5678', '10:00', '20:00'),
  ('홍대점', '서울 마포구 홍익로 45',    '서울', '02-2345-6789', '10:00', '21:00'),
  ('분당점', '경기 성남시 분당구 판교로 12', '경기', '031-1234-5678', '10:00', '20:00');
```

---

## Step 5: 직원 데이터 입력

```sql
-- branch_id는 위에서 생성한 branches.id UUID 사용
INSERT INTO public.staff (branch_id, name, role, phone, code)
VALUES
  ('강남점-UUID', '박지현', 'designer', '010-1234-5678', 'S-001'),
  ('강남점-UUID', '최유진', 'designer', '010-2345-6789', 'S-002');
```

---

## Step 6: GitHub Push → Vercel 자동 배포

```bash
git add .
git commit -m "feat: Supabase 실서버 연동 완료"
git push origin main
```

Vercel이 자동으로 재배포됩니다.

---

## 각 파일별 로그인 방법

| 파일 | 로그인 방식 | 비고 |
|-----|-----------|------|
| `b2c-home.html` | 전화번호 + 비밀번호 | `01012345678@m.seupkyul.co` 로 Supabase 인증 |
| `admin.html` | 이메일 + 비밀번호 | `admin@seupkyul.com` |
| `franchisee.html` (점주) | 이메일 + 비밀번호 | `gangnam@seupkyul.com` |
| `franchisee.html` (직원) | 전화번호 입력 | staff 테이블 phone 컬럼으로 조회 |
| `staff-pwa.html` | 지점 선택 → 이름 선택 | 비밀번호 없음 (단순 선택) |

---

## RLS (Row Level Security) 정책

`migration.sql`에 이미 포함됨:
- **공개 읽기**: branches, programs, products, notices, popups, reviews
- **관리자 전체 권한**: super_admin / branch_admin
- **고객 본인 데이터**: reservations, b2c_orders, inquiries

---

## Step 7: 추가 RLS 정책 (직원 앱용) — 중요!

`migration.sql`을 이미 실행한 경우, Supabase SQL Editor에서 아래를 추가로 실행하세요.
직원 로그인(staff-pwa.html, franchisee.html)이 Supabase Auth 세션 없이 동작하려면 반드시 필요합니다.

```sql
-- staff 테이블 공개 읽기 (직원 목록 선택, 전화번호 로그인)
CREATE POLICY "public_read_staff" ON public.staff
  FOR SELECT USING (is_active = true);

-- attendance 테이블 anon 출퇴근 기록
CREATE POLICY "anon_insert_attendance" ON public.attendance
  FOR INSERT WITH CHECK (true);

CREATE POLICY "anon_update_attendance" ON public.attendance
  FOR UPDATE USING (true);

CREATE POLICY "anon_read_attendance" ON public.attendance
  FOR SELECT USING (true);

-- reservations 테이블: 직원 자신의 예약 조회
CREATE POLICY "staff_own_reservations" ON public.reservations
  FOR SELECT USING (
    auth.uid() IS NOT NULL AND
    EXISTS (SELECT 1 FROM public.staff WHERE staff.id = reservations.staff_id)
  );
```

---

## Step 8: 점주 계정 branch_id 설정 (franchisee.html 지점 자동 연결)

`gangnam@seupkyul.com` 점주 계정에 강남점 UUID를 연결해야 franchisee.html 로그인 후 올바른 지점이 표시됩니다.

```sql
-- branches 테이블에서 강남점 UUID 확인
SELECT id, name FROM public.branches WHERE name = '강남점';

-- auth.users에 branch_id 메타데이터 설정 (위에서 확인한 UUID로 교체)
UPDATE auth.users
SET raw_user_meta_data = raw_user_meta_data || '{"branch_id":"<강남점-UUID>"}'::jsonb
WHERE email = 'gangnam@seupkyul.com';
```
