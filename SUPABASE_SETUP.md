# Cloud Database Setup (Supabase) — Driver OT Module

The app now stores all shared data (drivers, admin password, punch entries, open
punch-ins) in a **Supabase** cloud database, so changes made on one device show up
on every other device. Session login and the month-end auto-send toggle stay on the
device where they were set (that is by design — they are per-browser).

Do this once. Total time ~5 minutes.

---

## Step 1 — Create a free Supabase project

1. Go to **https://supabase.com** and sign in (a free account is fine — you can use
   your Google account).
2. Click **New project**.
3. Give it a name (e.g. `driver-ot`), set a database password (save it somewhere),
   pick the region closest to you (e.g. *South Asia (Mumbai)*), and click
   **Create new project**. Wait ~1 minute for it to finish provisioning.

## Step 2 — Create the tables

1. In the left sidebar open **SQL Editor** → **New query**.
2. Paste the entire block below and click **Run**. You should see *Success. No rows returned*.

```sql
-- ── Tables ────────────────────────────────────────────────
create table if not exists ot_users (
  username text primary key,
  data     jsonb not null
);

create table if not exists ot_admin (
  id   int  primary key default 1,
  data jsonb not null
);

create table if not exists ot_entries (
  id       bigint primary key,
  username text,
  month    text,
  data     jsonb not null
);

create table if not exists ot_active_punch (
  username text primary key,
  data     jsonb not null
);

-- ── Row Level Security ────────────────────────────────────
alter table ot_users        enable row level security;
alter table ot_admin        enable row level security;
alter table ot_entries      enable row level security;
alter table ot_active_punch enable row level security;

-- ── Access policies (the app uses the public "anon" key) ──
create policy "anon full ot_users"        on ot_users        for all to anon using (true) with check (true);
create policy "anon full ot_admin"        on ot_admin        for all to anon using (true) with check (true);
create policy "anon full ot_entries"      on ot_entries      for all to anon using (true) with check (true);
create policy "anon full ot_active_punch" on ot_active_punch for all to anon using (true) with check (true);
```

## Step 3 — Copy your project keys into the app

1. In the sidebar open **Project Settings** (gear icon) → **API**.
2. Copy two values:
   - **Project URL** — looks like `https://abcdefghijklmno.supabase.co`
   - **anon public** key (under *Project API keys*) — a long string starting with `eyJ…`
3. Open **index.html**, find this block near the top of the `<script>` (around line 590):

   ```js
   const SUPABASE_URL      = 'https://YOUR-PROJECT-ref.supabase.co';
   const SUPABASE_ANON_KEY = 'YOUR-ANON-PUBLIC-KEY';
   ```

   Replace the two placeholder strings with your real values. Keep the quotes.

4. Save the file, commit, and push so Render redeploys (or re-upload it to Render).

## Step 4 — Verify

1. Open the deployed page. On the **admin** device log in with the default
   `admin / admin@123`, create a driver, and change the admin password.
2. On a **second device / phone**, log in as that driver and Punch In / Punch Out.
3. Back on the admin device the new entry appears within ~15 seconds (it also
   refreshes instantly when you switch back to the tab). That confirms cross-device
   sync is working.

### First-run migration
The first time the app loads with real keys, if it finds drivers/entries already
saved in **this browser** (from before), it asks once whether to upload them to the
cloud. Say **OK** on the device that has your real data; say **Cancel** everywhere else.

---

## Good to know

- **Free tier** is plenty for this app (hundreds of drivers × years of daily entries
  is well within the free row/storage limits).
- **Local-only fallback:** if the keys are left as placeholders, or the Supabase
  library can't load, the app keeps working using the old on-device storage and shows
  a small notice — so it never breaks.
- **Auto-send toggle & "already sent" log** stay per-device on purpose: month-end
  auto-send can only fire from a browser that is actually open that day.

## Security note (please read)

This is a browser-only app, so the **anon key lives inside index.html** and anyone
who can open the page can read/write the database with it — including the driver and
admin passwords, which are stored as plain text (this was already the case with the
old on-device storage). For an internal tool shared within Rodic that is usually
acceptable, and it matches how the app worked before.

If you later want to tighten this, the standard upgrade is to move login onto
**Supabase Auth** (real hashed passwords + per-user row policies) so the anon key can
no longer read everyone's data. That is a larger change — happy to do it as a
follow-up if you want it.
