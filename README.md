# CarYard Manager

A real-time car yard inventory system. Track vehicles across lot locations with drag-and-drop, mark cars as sold, and sync changes live across all devices.

Built as a single HTML file — no build tools, no frameworks, no server. Just Supabase + GitHub Pages.

---

## Features

- **Dashboard** — Drag-and-drop cars between lot locations (columns)
- **Fleet Registry** — View all active vehicles grouped by location
- **Sold Cars** — Track sold inventory with restore/delete options
- **Real-time sync** — Changes appear on all open devices within seconds
- **Password lock** — SHA-256 hashed, never stored in plaintext
- **Export** — One-click JSON backup of all data

---

## Setup Guide

### 1. Create a Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign up / sign in (free tier works fine)
2. Click **New Project**
3. Give it a name (e.g. `caryard`), set a database password, pick a region close to your team
4. Wait ~2 minutes for it to provision

### 2. Create the Database Tables

Go to **SQL Editor** in the Supabase sidebar and run this entire block:

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Locations table (your lot areas)
CREATE TABLE locations (
  id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  name TEXT NOT NULL,
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Cars table
CREATE TABLE cars (
  id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  brand TEXT NOT NULL,
  model TEXT NOT NULL,
  plate TEXT NOT NULL,
  location_id UUID REFERENCES locations(id) ON DELETE SET NULL,
  sold BOOLEAN DEFAULT FALSE,
  sold_date TEXT,
  prev_location_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 3. Enable Row Level Security (RLS)

This is the most important security step. Without RLS, anyone who finds your publishable key could read or delete all your data.

Run this in the SQL Editor:

```sql
ALTER TABLE locations ENABLE ROW LEVEL SECURITY;
ALTER TABLE cars ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow all for anon" ON locations
  FOR ALL USING (true) WITH CHECK (true);

CREATE POLICY "Allow all for anon" ON cars
  FOR ALL USING (true) WITH CHECK (true);
```

> These policies allow full access via the publishable key. This is fine for a small team tool with password protection. For tighter security, see the "Going Further" section at the bottom.

### 4. Enable Realtime

This is what makes changes sync live across devices. Run this in the SQL Editor:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE cars;
ALTER PUBLICATION supabase_realtime ADD TABLE locations;
```

To confirm it worked, you can run:

```sql
SELECT * FROM pg_publication_tables WHERE pubname = 'supabase_realtime';
```

You should see both `cars` and `locations` listed. This is completely free on all Supabase plans.

> **Note:** Don't use the Database → Replication page in the Supabase dashboard — that's for paid read-replica features, not the realtime sync this app uses.

### 5. Get Your Supabase Credentials

Go to **Settings → API** in the Supabase dashboard. You need two values:

| Value | Where to find it | Looks like |
|-------|-----------------|------------|
| **Project URL** | Settings → API → Project URL | `https://abcdefghijk.supabase.co` |
| **Publishable key** | Settings → API → Project API keys → `publishable` | `sb_publishable_...` |

You'll also see a **Project ID** on the Settings → General page — this is the subdomain in your Project URL.

> **⚠️ Never use the secret key in this file.** The secret key bypasses all RLS and must only ever be used server-side. The publishable key is designed to be safe in client-side code.

### 6. Generate Your Password Hash

The app uses a SHA-256 hash so the actual password never appears in the source code. To generate a hash:

1. Open any browser → press **F12** → go to the **Console** tab
2. Run this (replace `YourChosenPassword` with the password you want your team to use):

```javascript
crypto.subtle.digest('SHA-256', new TextEncoder().encode('YourChosenPassword'))
  .then(b => console.log(Array.from(new Uint8Array(b)).map(x=>x.toString(16).padStart(2,'0')).join('')))
```

3. Copy the long hex string it outputs (e.g. `51d7d2a5d684c038...`)

### 7. Update the HTML File

Open `index.html` and find the **CONFIG** section near the top of the `<script>` tag. Update these three values:

```javascript
// Replace with your Project URL
const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co';

// Replace with your publishable key
const SUPABASE_KEY = 'sb_publishable_YOUR_KEY_HERE';

// Replace with the hash you generated in Step 6
const CORRECT_HASH = 'YOUR_SHA256_HASH_HERE';
```

### 8. Deploy to GitHub Pages

1. Create a **new GitHub repository** (e.g. `caryard-manager`)
2. Make it **private** — this hides your source code from public browsing
3. Push your `index.html` to the `main` branch
4. Go to **Settings → Pages** in the repo
5. Set Source to **Deploy from a branch**, select `main`, folder `/` (root)
6. After a minute or two, your site will be live at:
   ```
   https://yourusername.github.io/caryard-manager/
   ```

> **Important:** Even with a private repo, the GitHub Pages URL is publicly accessible. The password lock screen is what prevents unauthorised access — the private repo just stops people browsing your source code on GitHub.

---

## How It Works

| What happens | How it syncs |
|-------------|-------------|
| Add / edit / delete a car | Writes to Supabase, all devices update via realtime subscription |
| Drag a car to a new location | Updates `location_id` in the database |
| Mark a car as sold | Sets `sold: true`, clears location, records the date |
| Restore a sold car | Returns it to its previous location (or first available) |
| Export backup | Downloads a JSON file of all current data to your device |

The green dot in the header shows sync status — green means connected, amber means syncing, red means connection lost.

---

## Security Summary

| Layer | What it does |
|-------|-------------|
| **Password lock** | Prevents casual access to the UI. SHA-256 hashed — password never in source code |
| **RLS policies** | Controls what the publishable key can do at the database level |
| **Private GitHub repo** | Hides source code from public browsing on GitHub |
| **DB connects after unlock** | No data is fetched until the correct password is entered |
| **Publishable key** | Designed to be safe in client-side code — RLS is the real guard |

---

## Changing the Password

1. Generate a new hash using the console command from Step 6
2. Replace the `CORRECT_HASH` value in `index.html`
3. Push the update to GitHub
4. Tell your team the new password
5. Anyone with "Keep me unlocked" checked will need to clear their browser's localStorage for the site (or use a private/incognito window) to be prompted again

---

## Going Further (Optional)

**Supabase Auth** — Replace the simple password with real user accounts. Each team member signs up, and RLS policies check `auth.uid()`. This is the most secure option but adds login/signup UI.

**Regular backups** — Use the Export button periodically. The free Supabase tier includes daily automatic backups for 7 days.

**Custom domain** — You can point a custom domain to your GitHub Pages site in the repo's Pages settings.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Database read error" toast | RLS policies not set up — run Step 3 SQL |
| "Connection failed" screen | Wrong `SUPABASE_URL` or `SUPABASE_KEY` in the HTML |
| Changes don't sync between devices | Realtime not enabled — run Step 4 SQL |
| Password not working | Make sure the hash matches — regenerate with Step 6 |
| Lock screen doesn't appear | Clear localStorage: open console, run `localStorage.removeItem('cy_unlocked')` |
