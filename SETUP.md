# Supabase 設定（5 分鐘）

啟用跨裝置同步前需要先開一個 Supabase 專案。不設定也能用，只是只存在本機 localStorage。

## 1. 開 Supabase 帳號

1. 到 <https://supabase.com> 點 **Start your project**
2. 用 GitHub 或 Email 註冊（免費方案夠用）
3. 建立 New Project
   - Project name：隨意（例如 `fittinglog`）
   - Database password：自己設一個（之後幾乎不會用到，但要記著）
   - Region：選 `Northeast Asia (Tokyo)` 或 `Southeast Asia (Singapore)`
4. 等 1–2 分鐘讓專案 provision 好

## 2. 跑 SQL Schema

進入 Dashboard → **SQL Editor** → **New query**，貼上以下 SQL 後按 **Run**：

```sql
create table if not exists public.sessions (
  user_id uuid references auth.users on delete cascade not null default auth.uid(),
  date date not null,
  exercises jsonb not null default '[]'::jsonb,
  updated_at timestamptz not null default now(),
  primary key (user_id, date)
);

alter table public.sessions enable row level security;

create policy "users access own sessions"
  on public.sessions for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

create or replace function public.touch_updated_at()
returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end;
$$;

create trigger sessions_touch
  before update on public.sessions
  for each row execute function public.touch_updated_at();

-- Exercise catalog (metadata per exercise name, used for muscle-group
-- tagging and future menu/plan linking + trend analytics).
create table if not exists public.exercises (
  user_id uuid references auth.users on delete cascade not null default auth.uid(),
  name text not null,
  muscle_group text,
  equipment text,
  weight_recording text default 'single', -- 'single' (dumbbell) | 'total' (barbell)
  created_at timestamptz not null default now(),
  primary key (user_id, name)
);

alter table public.exercises enable row level security;

create policy "users access own exercises"
  on public.exercises for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- Explicit GRANT / REVOKE (belt-and-suspenders alongside RLS).
-- Anon traffic gets rejected at the API boundary; authenticated users
-- still pass through RLS, which then filters to their own rows.
-- See SETUP.md "升級舊專案" section if you need to apply this to an
-- already-running project.
revoke all on public.sessions  from anon;
revoke all on public.exercises from anon;
grant select, insert, update, delete on public.sessions  to authenticated;
grant select, insert, update, delete on public.exercises to authenticated;
```

說明：
- 一個訓練日一列，`(user_id, date)` 是主鍵
- `exercises` 用 JSONB 直接存（暫時對應前端結構，第 3 步會正規化）
- Row-Level Security 開著，每個使用者只看得到自己的列
- 顯式 `revoke from anon` + `grant to authenticated`：深度防禦。RLS 已經夠用，但這層讓未登入流量在 PostgREST API 邊界就被擋掉，不需要走到 row-level filter。同時不受 Supabase 之後改預設 grant 政策影響。

## 3. 設定 Auth Redirect URL

Dashboard → **Authentication** → **URL Configuration**：

- **Site URL**：填你部署 app 的網址。例如：
  - 本機開發：`http://localhost:8000`
  - GitHub Pages：`https://你的帳號.github.io/fittinglog/`
- **Redirect URLs**：把上面的網址也加進去

不設這個的話，Magic Link 點下去會跑去 `localhost:3000`（Supabase 預設）。

## 4. 拿 API Key 填進 index.html

Dashboard → **Settings** → **API**：

- 複製 **Project URL**（`https://xxxxx.supabase.co`）
- 複製 **anon public** key（很長的 JWT）

打開 `index.html`，找到開頭附近這兩行：

```js
const SUPABASE_URL = "";
const SUPABASE_ANON_KEY = "";
```

填進去：

```js
const SUPABASE_URL = "https://xxxxx.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGciOi...";
```

`anon` key 是設計來放在前端的（RLS policy 才是真的權限管控），不算機密。但還是別把它推到 public repo 比較好 — 部署時再填，或用 build-time 環境變數。

## 5. 測試

1. 重新打開 `index.html`
2. 看到登入畫面 → 輸入 email → 按「寄送登入連結」
3. 收信、點連結 → 自動登入回 app
4. 之前在本機的紀錄會自動推到雲端
5. 換手機開同一個 URL、用同一個 email 登入 → 看到一樣的紀錄

## 同步行為

- **登入時**：拉雲端全部紀錄、合併本地（衝突日期以雲端為準）、把本地獨有的日期推上去
- **每次改動**：本地立即存 localStorage，背景非同步 upsert 到 Supabase
- **離線**：本地仍可寫入，下次連上線會在下次操作時 sync（v1 沒有自動補推背景重試）
- **同步狀態**：header 右上角小圓點 — 黃色脈動 = 同步中、紅色 = 失敗

## 升級舊專案（已跑過舊版 schema 的人）

如果你的 Supabase 是在加入顯式 GRANT 那一行之前建的（兩張表已存在、跑得起來），補上深度防禦只要跑這段 patch SQL：

```sql
revoke all on public.sessions  from anon;
revoke all on public.exercises from anon;
grant select, insert, update, delete on public.sessions  to authenticated;
grant select, insert, update, delete on public.exercises to authenticated;
```

到 Dashboard → **SQL Editor** → **New query**，貼上按 Run。跑完什麼都不會壞 — RLS 仍是主要安全層，這層只是加一道前置的權限白名單。

跑完到左側 **Security Advisor** 確認 `public.sessions` 和 `public.exercises` 不再標 “anon role accessible” 之類的警告。

## 疑難排解

| 症狀 | 原因 | 解法 |
|------|------|------|
| 收不到登入信 | 看垃圾信匣；或 Supabase free tier 寄信速率限制 | 等 1 分鐘再試 |
| 點連結後跑到 localhost | Site URL 沒設對 | 回到第 3 步 |
| 「雲端讀取失敗」toast | RLS policy 沒建好、或網路問題 | 開 DevTools Console 看 error，檢查 SQL 是否跑完 |
| header 一直顯示「同步中…」 | 通常是 RLS 擋掉 upsert | 確認 step 2 的 policy 有建到 |
