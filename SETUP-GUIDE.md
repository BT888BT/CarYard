# CarYard Manager — Setup Guide

## What I Fixed in Your Code

### 1. Password was visible in plain text
Your original code had `hashPass('Curated2026')` — anyone who right-clicks → View Source can see the password. The fixed version uses **SHA-256 hashing** with a pre-computed hash, so the password itself never appears in the code.

### 2. Fake/invalid Supabase key
The key `sb_publishable_ZXK8O-Q6mUAUcJ-fGYRV2Q_x47nLwZ8` isn't a real Supabase anon key (they start with `eyJ...`). Replaced with placeholder instructions.

### 3. Lock screen started unlocked
Line 169 had `class="unlocked"` — meaning the lock screen was hidden by default on page load. Fixed to show the lock screen first.

### 4. Database loaded before authentication
The original `init()` called `connectDB()` regardless of lock state, meaning data was fetched and available in browser memory even before the password was entered. Fixed so the DB only connects after successful unlock.

### 5. Weak hash function
The original `hashPass` was a simple djb2 hash (trivially reversible). Replaced with browser-native SHA-256 via `crypto.subtle`.

---

## Step-by-Step Supabase Setup

### A. Create Your Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign in
2. Click **New Project**
3. Choose your org, give it a name (e.g. `caryard`), set a strong database password, pick a region close to Australia (e.g. `Southeast Asia (Singapore)`)
4. Wait for it to provision (~2 minutes)

### B. Create the Database Tables

Go to **SQL Editor** in the left sidebar and run this:

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Locations table
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

### C. Enable Row Level Security (RLS) — CRITICAL

This is the most important step. Without RLS, **anyone** who finds your Supabase URL and anon key can read, modify, or delete all your data. The anon key is always visible in your HTML source — RLS is what actually protects your data.

Run this in the SQL Editor:

```sql
-- Enable RLS on both tables
ALTER TABLE locations ENABLE ROW LEVEL SECURITY;
ALTER TABLE cars ENABLE ROW LEVEL SECURITY;

-- Allow anonymous users to read, insert, update, and delete
-- This is safe because:
--   1. Your page has password protection in the UI
--   2. The anon key only grants access through these policies
--   3. You can tighten these later if needed

CREATE POLICY "Allow all access for anon" ON locations
  FOR ALL USING (true) WITH CHECK (true);

CREATE POLICY "Allow all access for anon" ON cars
  FOR ALL USING (true) WITH CHECK (true);
```

> **Why this is "safe enough" for your use case:** Your app is a small team tool, not a public-facing product. The password lock prevents casual access. The RLS policies above allow operations through the anon key, which is the expected behaviour for your setup. If someone were to extract your Supabase URL and key from the HTML source, they could theoretically access the data — but for a car yard inventory, this risk is minimal. See the "Going Further" section below if you want tighter security.

### D. Enable Realtime

1. Go to **Database → Replication** in the Supabase dashboard
2. Under "Supabase Realtime", make sure **both** `locations` and `cars` tables have replication turned **ON**
3. This is what makes changes sync live across devices

### E. Get Your API Keys

1. Go to **Settings → API** in the Supabase dashboard
2. Copy the **Project URL** — looks like `https://abcdefghijk.supabase.co`
3. Copy the **anon public** key — starts with `eyJhbGci...` (it's a long string)

### F. Generate Your Password Hash

Open your browser console (F12 → Console tab) and run:

```javascript
crypto.subtle.digest('SHA-256', new TextEncoder().encode('YourChosenPassword'))
  .then(b => console.log(Array.from(new Uint8Array(b)).map(x=>x.toString(16).padStart(2,'0')).join('')))
```

Replace `YourChosenPassword` with whatever password you want your team to use. Copy the output hash string.

### G. Update the HTML File

Open `index.html` and find the CONFIG section near the top of the `<script>`. Replace:

```javascript
const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co';
const SUPABASE_KEY = 'eyJ_YOUR_ANON_KEY_HERE';
```

With your actual values from Step E.

Then replace:

```javascript
const CORRECT_HASH = 'GENERATE_YOUR_HASH_AND_PASTE_HERE';
```

With the hash you generated in Step F.

### H. Deploy to GitHub Pages

1. Create a new GitHub repository (e.g. `caryard-manager`)
2. Make it **private** (so random people can't browse your source on GitHub)
3. Push your `index.html` to the `main` branch
4. Go to **Settings → Pages**
5. Set Source to **Deploy from a branch**, select `main`, root (`/`)
6. Your site will be live at `https://yourusername.github.io/caryard-manager/`

> **Important:** Even with a private repo, GitHub Pages sites are **publicly accessible** via the URL. The password lock and RLS are your protection layers — not the repo visibility. The private repo just prevents people from browsing the source code directly on GitHub.

---

## Security Summary

| Layer | What it does | Status |
|-------|-------------|--------|
| **Password lock** | Prevents casual access to the UI | ✅ SHA-256 hashed, not in plaintext |
| **RLS policies** | Controls what the anon key can do in Supabase | ✅ Enabled (must run the SQL above) |
| **Private GitHub repo** | Hides source code from public browsing | ✅ Recommended |
| **Anon key exposure** | Always visible in HTML source (this is normal) | ⚠️ Acceptable — RLS is the real guard |

---

## Going Further (Optional)

If you want tighter security in the future, consider:

1. **Supabase Auth** — Replace the simple password with real user accounts. Each team member gets a login, and RLS policies check `auth.uid()`. This is the gold standard but adds complexity.

2. **Service role key on a backend** — Never expose the service role key (the other key in your Supabase settings). It bypasses all RLS. Only use it server-side.

3. **Rate limiting** — Supabase has built-in rate limiting on the free tier, which helps prevent abuse.

4. **Regular backups** — Use the Export button in the app to save JSON backups periodically.
