# Apex — Daily Submission Portal

A single-file web app for tracking daily task updates across a team, with a
one-click daily report. Galaxy-themed UI, no build step, no server to run.

## Files
- `index.html` — the entire app (UI + logic). Just open it in a browser.

## Two modes

### 1. Local mode (default — works instantly)
Open `index.html` by double-clicking it. Everything works, but data is stored
only in **that browser on that device**. Good for trying the UI. A yellow banner
reminds you you're in local mode.

### 2. Shared mode (everyone sees the same live data) — recommended
Connect a free [Supabase](https://supabase.com) database. ~5 minutes, no server,
no credit card.

#### Step 1 — Create the project
1. Go to https://supabase.com and sign up (free tier).
2. Click **New project**. Pick any name/password/region. Wait ~2 min for it to spin up.

#### Step 2 — Create the tables
1. In your project, open the **SQL Editor** (left sidebar).
2. Paste the SQL below and click **Run**.

```sql
create table users (
  id uuid primary key default gen_random_uuid(),
  name text unique not null,
  created_at timestamptz default now()
);

create table tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  descr text,
  status text default 'ongoing',
  created_by text,
  created_at timestamptz default now()
);

create table updates (
  id uuid primary key default gen_random_uuid(),
  task_id uuid references tasks(id) on delete cascade,
  author text,
  type text default 'update',
  body text,
  created_at timestamptz default now()
);

-- Row Level Security: allow this trusted group full access via the public key.
alter table users   enable row level security;
alter table tasks   enable row level security;
alter table updates enable row level security;

create policy "open users"   on users   for all using (true) with check (true);
create policy "open tasks"   on tasks   for all using (true) with check (true);
create policy "open updates" on updates for all using (true) with check (true);
```

**Editing updates:** members can edit their own text updates in place. To show an
"edited" marker, add this optional column (the app still works without it):

```sql
alter table updates add column if not exists edited_at timestamptz;
```

### Migration for v2 features (PINs, due dates, assignees)
If you already created the tables above, run this once in the SQL Editor to add
the columns used by PIN login, due dates, and assignees:

```sql
alter table users add column if not exists pin text;
alter table tasks add column if not exists due_date date;
alter table tasks add column if not exists assignees jsonb default '[]'::jsonb;
```

(If you're creating the tables fresh, you can instead add `pin text` to `users`,
`due_date date` and `assignees jsonb default '[]'::jsonb` to `tasks` above.)

**Forgotten PIN?** PINs are set by each member on first login. If someone forgets
theirs, clear it in Supabase (Table editor → `users` → set that row's `pin` to
empty/null); they'll be prompted to set a new one next time.

#### Step 3 — Paste your keys into the app
1. In Supabase, go to **Project Settings → API**.
2. Copy the **Project URL** and the **anon / public** key.
3. Open `index.html` in a text editor, find the `SUPABASE CONFIG` block near the
   top of the `<script>`, and fill them in:

```js
var SUPABASE_URL = "https://YOURPROJECT.supabase.co";
var SUPABASE_ANON_KEY = "eyJhbGciOi...your-anon-key...";
```

4. Save. Reopen `index.html` — the top bar now shows a **LIVE • SHARED** pill.
   The 9 default members are seeded automatically on first run.

#### Step 4 — Let everyone use it
The 9 people need to open the *same* `index.html` (with your keys in it). Easiest
ways to share it:
- **Netlify Drop** — drag the `Apex!` folder onto https://app.netlify.com/drop, get a URL.
- **GitHub Pages** — push the folder to a repo, enable Pages.
- Or just send everyone the `index.html` file (with keys already filled in).

Each person picks their name on the login screen; that identity is remembered on
their device.

## Features
- **PIN login** — each member sets a 4-6 digit PIN on first sign-in; required after.
- **Identity** — 9 preset members + add new members.
- **Mission board** — add tasks, mark done / reopen, filter Ongoing / Completed / All,
  and **search** by title, person, or assignee.
- **Due dates & assignees** — optional per task; overdue/soon badges and assignee
  avatars on each card.
- **Edit / delete tasks** — the creator (or admin) can edit a task's details or delete it.
- **Task threads** — every update is logged with author + timestamp; a
  "Log No Updates" option records an explicit no-update entry. Members can edit or
  delete their own updates ("edited" marker shown on edits).
- **@mentions** — tag members in an update (chips below the box, or type `@`);
  mentions are highlighted in the thread.
- **Activity feed** — one chronological stream of every update across all tasks.
- **Weekly summary** — per-person breakdown of the last 7 days, with a silent-this-week
  flag for anyone with zero updates.
- **Daily report** (Ponnam only) — scrapes the last **30 hours** of updates per
  ongoing task into a copyable text block. Tasks with no activity in the window
  show **No Response**; explicit no-update entries show **No Updates**.

## Notes
- The anon key is safe to expose in client-side code — the open RLS policies above
  intentionally allow this private group to read/write. If you ever need stricter
  access, tighten the policies in Supabase.
- Shared mode polls for new data every 12 seconds.
- Settings to tweak at the top of the script: `ADMIN` (who can generate reports),
  `REPORT_WINDOW_HOURS` (default 30), `POLL_MS`.
