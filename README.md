# Supabase Wakeup

Supabase 무료 플랜 프로젝트가 **비활성으로 일시정지되는 것을 막기 위해**, 매일 아침 6시(KST)에
GitHub Actions가 `wakeup` 테이블의 `id = 1` 행의 `created_at`을 현재 시각으로 갱신합니다.

GitHub Pages(`index.html`)에는 아날로그 시계와 **마지막 wakeup 시각**이 표시됩니다.

```
.github/workflows/wakeup.yml         매일 06:00 KST — Supabase 갱신 + last-wakeup.json 커밋
.github/workflows/cleanup-runs.yml   매주 1회 — 워크플로별 실행 기록 최신 5개만 남기고 삭제
index.html                           아날로그 시계 + 마지막 wakeup 표시
last-wakeup.json                     워크플로가 자동 생성/갱신 (직접 만들 필요 없음)
```

---

## 1. Supabase 테이블 준비

SQL Editor에서 실행:

```sql
create table if not exists public.wakeup (
  id         bigint primary key,
  created_at timestamptz not null default now()
);

-- 갱신 대상 행
insert into public.wakeup (id, created_at)
values (1, now())
on conflict (id) do nothing;

-- RLS 를 켤 경우: service_role 키는 RLS 를 우회하므로 별도 정책 불필요
alter table public.wakeup enable row level security;
```

> 워크플로는 **service_role** 키를 사용하므로 RLS 정책 없이도 업데이트됩니다.
> 이 키는 절대 `index.html` 등 클라이언트에 넣지 마세요.

## 2. GitHub Secrets 등록

리포지토리 → **Settings → Secrets and variables → Actions → New repository secret**

| 이름 | 값 |
|---|---|
| `SUPABASE_URL` | `https://xxxxxxxx.supabase.co` (끝에 `/` 없이) |
| `SUPABASE_SERVICE_ROLE_KEY` | Settings → API → `service_role` secret |

## 3. Actions 권한

**Settings → Actions → General → Workflow permissions**
→ **Read and write permissions** 선택 (커밋 푸시 및 실행 기록 삭제에 필요)

## 4. GitHub Pages 활성화

**Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**
→ `https://<사용자명>.github.io/<리포지토리명>/` 에서 시계 확인

## 5. 동작 확인

**Actions → Supabase Wakeup → Run workflow** 로 수동 실행해 보세요.

---

## 스케줄 메모

GitHub cron은 **UTC 기준**입니다.

| 워크플로 | cron (UTC) | 실제 시각 (KST) |
|---|---|---|
| wakeup | `0 21 * * *` | 매일 06:00 |
| cleanup-runs | `0 20 * * 0` | 매주 월요일 05:00 |

- 다른 시간대를 쓰려면 `KST - 9시간 = UTC`로 계산해 `cron` 값을 바꾸세요.
- GitHub 스케줄은 부하에 따라 **수 분~수십 분 지연**될 수 있습니다 (정시 보장 아님).
- 스케줄 워크플로는 **60일간 리포지토리 활동이 없으면 자동 비활성화**되지만,
  wakeup이 매일 `last-wakeup.json`을 커밋하므로 이 문제는 발생하지 않습니다.

## 실행 기록 정리

`cleanup-runs.yml`은 **워크플로 파일별로** 완료된 실행 중 최신 5개만 남기고 삭제합니다.
개수를 바꾸려면 수동 실행 시 `keep` 입력값을 지정하거나, cron 스케줄 실행의 기본값
(`github.event.inputs.keep || '5'`)을 수정하세요.
