# EventHub — Claude Code Build Instructions

You are building a real, working version of the EventHub app — a mobile-first web
application for managing multi-location competition events. A fully functional HTML
prototype already exists that defines the exact UI, data model, scoring logic, and
user flows. Your job is to translate that into a production Next.js + Supabase app.

---

## STACK

- **Framework:** Next.js 14 (App Router)
- **Database + Auth:** Supabase (Postgres + Row Level Security + Realtime)
- **Styling:** Tailwind CSS
- **Deployment:** Vercel (frontend) + Supabase (backend)
- **Language:** TypeScript throughout

Install dependencies:
```
npx create-next-app@latest eventhub --typescript --tailwind --app --no-src-dir
cd eventhub
npm install @supabase/supabase-js @supabase/ssr
```

---

## WHAT TO BUILD — OVERVIEW

The app has two roles:
- **Attendee** (public, read-only): sees leaderboard for their location
- **Organizer** (authenticated): enters results for their assigned location

There is no admin UI needed for the MVP. Admin tasks (creating locations, teams,
activities) can be done directly in the Supabase dashboard.

---

## DATABASE SCHEMA

Run this SQL in your Supabase project (SQL Editor → New Query):

```sql
-- PROFILES (extends Supabase auth)
create table profiles (
  id uuid references auth.users primary key,
  email text,
  role text not null default 'attendee' check (role in ('admin','organizer','attendee')),
  created_at timestamptz default now()
);
alter table profiles enable row level security;
create policy "Users can read own profile" on profiles for select using (auth.uid() = id);
create policy "Admins can read all profiles" on profiles for select using (
  exists (select 1 from profiles where id = auth.uid() and role = 'admin')
);

-- Auto-create profile on signup
create or replace function handle_new_user()
returns trigger as $$
begin
  insert into profiles (id, email) values (new.id, new.email);
  return new;
end;
$$ language plpgsql security definer;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure handle_new_user();

-- LOCATIONS
create table locations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  short_name text not null,
  city text not null,
  venue text,
  accent_color text not null default '#e63946',
  slug text unique not null,
  is_active boolean default true,
  overall_hidden boolean default false,
  created_at timestamptz default now()
);
alter table locations enable row level security;
create policy "Anyone can read active locations" on locations for select using (is_active = true);

-- ORGANIZER_LOCATIONS (which organizers manage which locations)
create table organizer_locations (
  user_id uuid references profiles(id) on delete cascade,
  location_id uuid references locations(id) on delete cascade,
  primary key (user_id, location_id)
);
alter table organizer_locations enable row level security;
create policy "Organizers can read own assignments" on organizer_locations
  for select using (auth.uid() = user_id);

-- TEAMS
create table teams (
  id uuid primary key default gen_random_uuid(),
  location_id uuid references locations(id) on delete cascade not null,
  name text not null,
  number text not null,
  created_at timestamptz default now()
);
alter table teams enable row level security;
create policy "Anyone can read teams" on teams for select using (true);

-- ACTIVITIES
create table activities (
  id uuid primary key default gen_random_uuid(),
  location_id uuid references locations(id) on delete cascade not null,
  name text not null,
  input_type text not null check (input_type in ('time','score')),
  direction text not null check (direction in ('lower','higher')),
  weight numeric not null default 1,
  points_table jsonb not null default '{"1":100,"2":95,"3":90,"4":85,"5":80,"6":75}',
  penalty_rules jsonb not null default '[]',
  is_primary boolean default false,
  show_live boolean default true,
  sort_order int default 0,
  created_at timestamptz default now()
);
alter table activities enable row level security;
create policy "Anyone can read activities" on activities for select using (true);

-- RESULTS
create table results (
  id uuid primary key default gen_random_uuid(),
  activity_id uuid references activities(id) on delete cascade not null,
  team_id uuid references teams(id) on delete cascade not null,
  status text not null default 'incomplete' check (status in ('ok','dnf','dsq','incomplete')),
  raw_value numeric,
  penalties jsonb default '{}',
  submitted_by uuid references profiles(id),
  submitted_at timestamptz default now(),
  updated_at timestamptz default now(),
  unique(activity_id, team_id)
);
alter table results enable row level security;
create policy "Anyone can read results" on results for select using (true);
create policy "Organizers can insert results" on results for insert
  with check (
    exists (
      select 1 from organizer_locations ol
      join activities a on a.location_id = ol.location_id
      where ol.user_id = auth.uid() and a.id = activity_id
    )
  );
create policy "Organizers can update results" on results for update
  using (
    exists (
      select 1 from organizer_locations ol
      join activities a on a.location_id = ol.location_id
      where ol.user_id = auth.uid() and a.id = activity_id
    )
  );
```

**After running the schema**, seed example data:

```sql
-- Dallas location
insert into locations (name, short_name, city, venue, accent_color, slug, overall_hidden)
values ('Dallas Autocross Championship', 'Dallas', 'Dallas, TX', 'Texas Motor Speedway', '#e63946', 'dallas', true);

-- Get the Dallas location id for use in subsequent inserts:
-- select id from locations where slug = 'dallas';
-- Then insert teams and activities using that id.

-- Example teams (replace DALLAS_ID with actual uuid):
-- insert into teams (location_id, name, number) values
--   ('DALLAS_ID', 'Team Rocket', '42'),
--   ('Speed Demons', '7'), ...

-- Example activity:
-- insert into activities (location_id, name, input_type, direction, weight, penalty_rules, is_primary, show_live)
-- values ('DALLAS_ID', 'Autocross Run', 'time', 'lower', 2,
--   '[{"name":"cone","label":"Cones Hit","adj_seconds":2},{"name":"oc","label":"Off Course","adj_seconds":10}]',
--   true, true);
```

---

## PROJECT FILE STRUCTURE

```
eventhub/
├── app/
│   ├── layout.tsx                  # Root layout, fonts, metadata
│   ├── page.tsx                    # Location selector (home screen)
│   ├── [slug]/
│   │   ├── page.tsx                # Leaderboard page for a location
│   │   └── layout.tsx              # Location header + color theming
│   ├── organizer/
│   │   ├── login/page.tsx          # Organizer login page
│   │   └── [slug]/page.tsx         # Organizer result entry (protected)
│   └── api/
│       ├── leaderboard/[slug]/route.ts   # GET computed leaderboard
│       └── results/route.ts             # POST/PUT result entry
├── components/
│   ├── LocationCard.tsx
│   ├── LeaderboardEntry.tsx
│   ├── ActivityTabs.tsx
│   ├── ResultEntryPanel.tsx        # Slide-up panel for organizers
│   └── Toast.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts               # Browser Supabase client
│   │   └── server.ts               # Server Supabase client
│   └── scoring/
│       └── engine.ts               # Scoring logic (see below)
├── types/
│   └── index.ts                    # All TypeScript types
└── middleware.ts                   # Auth guard for /organizer routes
```

---

## TYPES (types/index.ts)

```typescript
export type Role = 'admin' | 'organizer' | 'attendee'

export interface Location {
  id: string
  name: string
  short_name: string
  city: string
  venue: string | null
  accent_color: string
  slug: string
  is_active: boolean
  overall_hidden: boolean
}

export interface Team {
  id: string
  location_id: string
  name: string
  number: string
}

export interface PenaltyRule {
  name: string
  label: string
  adj_seconds: number
}

export interface Activity {
  id: string
  location_id: string
  name: string
  input_type: 'time' | 'score'
  direction: 'lower' | 'higher'
  weight: number
  points_table: Record<string, number>   // e.g. {"1": 100, "2": 95}
  penalty_rules: PenaltyRule[]
  is_primary: boolean
  show_live: boolean
  sort_order: number
}

export interface Result {
  id: string
  activity_id: string
  team_id: string
  status: 'ok' | 'dnf' | 'dsq' | 'incomplete'
  raw_value: number | null
  penalties: Record<string, number>      // e.g. {"cone": 2, "oc": 0}
  submitted_at: string
}

export interface LeaderboardEntry {
  team: Team
  rank: number
  adjusted_value: number | null
  raw_value: number | null
  penalties: Record<string, number>
  points: number
  weighted_points: number
  status: Result['status']
}

export interface OverallEntry {
  team: Team
  rank: number
  total_points: number
  first_place_count: number
  activity_breakdown: Array<{
    activity_name: string
    weighted_points: number
    rank: number
  }>
}
```

---

## SCORING ENGINE (lib/scoring/engine.ts)

Implement these exact functions — they are validated against the working demo:

```typescript
import type { Activity, Result, Team, LeaderboardEntry, OverallEntry } from '@/types'

// Apply time penalties to a result's raw value
export function applyPenalties(result: Result, activity: Activity): number | null {
  if (result.status !== 'ok' || result.raw_value === null) return null
  if (activity.input_type === 'time') {
    const penaltySeconds = activity.penalty_rules.reduce((total, rule) => {
      return total + (result.penalties[rule.name] || 0) * rule.adj_seconds
    }, 0)
    return result.raw_value + penaltySeconds * 1000  // raw_value stored in ms
  }
  // For score type, no penalty adjustment (penalties handled differently if needed)
  return result.raw_value
}

// Get points for a finishing place from a points table
export function getPointsForPlace(place: number, table: Record<string, number>): number {
  if (!isFinite(place)) return 0
  if (table[place] !== undefined) return table[place]
  const maxPlace = Math.max(...Object.keys(table).map(Number))
  const minPoints = table[maxPlace]
  const step = 5  // decrease 5 pts per place beyond the table
  return Math.max(0, minPoints - (place - maxPlace) * step)
}

// Format time in milliseconds to display string (e.g. "1:03.412" or "63.412s")
export function formatTime(ms: number): string {
  const min = Math.floor(ms / 60000)
  const sec = Math.floor((ms % 60000) / 1000)
  const mil = ms % 1000
  if (min > 0) return `${min}:${String(sec).padStart(2,'0')}.${String(mil).padStart(3,'0')}`
  return `${sec}.${String(mil).padStart(3,'0')}s`
}

// Compute leaderboard for a single activity
export function computeActivityLeaderboard(
  activity: Activity,
  results: Result[],
  teams: Team[]
): LeaderboardEntry[] {
  const teamMap = Object.fromEntries(teams.map(t => [t.id, t]))

  const entries = results.map(r => ({
    team: teamMap[r.team_id],
    status: r.status,
    raw_value: r.raw_value,
    adjusted_value: applyPenalties(r, activity),
    penalties: r.penalties,
    points: 0,
    weighted_points: 0,
    rank: Infinity,
    submitted_at: r.submitted_at,
  }))

  // Separate complete and incomplete results
  const complete = entries.filter(e => e.status === 'ok' && e.adjusted_value !== null)
  const incomplete = entries.filter(e => e.status !== 'ok' || e.adjusted_value === null)

  // Sort complete results
  complete.sort((a, b) => {
    const aVal = a.adjusted_value!
    const bVal = b.adjusted_value!
    if (activity.direction === 'lower') return aVal - bVal
    return bVal - aVal
  })

  // Assign ranks (competition ranking: 1,2,2,4)
  let rank = 1
  complete.forEach((e, i) => {
    if (i > 0 && complete[i - 1].adjusted_value === e.adjusted_value) {
      e.rank = complete[i - 1].rank
    } else {
      e.rank = rank
    }
    rank = i + 2
    e.points = getPointsForPlace(e.rank, activity.points_table)
    e.weighted_points = e.points * activity.weight
  })

  incomplete.forEach(e => {
    e.rank = Infinity
    e.points = 0
    e.weighted_points = 0
  })

  return [...complete, ...incomplete]
}

// Compute overall leaderboard across all activities
export function computeOverallLeaderboard(
  activities: Activity[],
  allResults: Result[],
  teams: Team[]
): OverallEntry[] {
  const totals: Record<string, {
    team: Team
    total_points: number
    first_place_count: number
    activity_breakdown: OverallEntry['activity_breakdown']
  }> = {}

  teams.forEach(t => {
    totals[t.id] = { team: t, total_points: 0, first_place_count: 0, activity_breakdown: [] }
  })

  activities.forEach(activity => {
    const actResults = allResults.filter(r => r.activity_id === activity.id)
    const board = computeActivityLeaderboard(activity, actResults, teams)
    board.forEach(entry => {
      if (!totals[entry.team.id]) return
      totals[entry.team.id].total_points += entry.weighted_points
      if (entry.rank === 1) totals[entry.team.id].first_place_count++
      totals[entry.team.id].activity_breakdown.push({
        activity_name: activity.name,
        weighted_points: entry.weighted_points,
        rank: entry.rank,
      })
    })
  })

  return Object.values(totals)
    .sort((a, b) => b.total_points - a.total_points || b.first_place_count - a.first_place_count)
    .map((d, i) => ({ ...d, rank: i + 1 }))
}
```

---

## DESIGN SYSTEM

Match the demo's visual style exactly. Use these CSS custom properties — add them to
`app/globals.css`:

```css
@import url('https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@400;600;700;800;900&family=Barlow:wght@400;500;600&family=JetBrains+Mono:wght@400;600&display=swap');

:root {
  --bg: #0a0a0f;
  --surface: #13131a;
  --surface2: #1c1c26;
  --border: #2a2a3a;
  --text: #e8e8f0;
  --muted: #6b6b88;
  --gold: #ffd700;
  --silver: #c0c0c0;
  --bronze: #cd7f32;
  --radius: 10px;
}
```

Tailwind config additions needed in `tailwind.config.ts`:
```js
fontFamily: {
  display: ['Barlow Condensed', 'sans-serif'],
  body: ['Barlow', 'sans-serif'],
  mono: ['JetBrains Mono', 'monospace'],
}
```

Key design rules:
- Dark background (#0a0a0f) always
- Each location has an accent color — apply it dynamically to headers, tabs, buttons
- Leaderboard rows: gold border for 1st, silver for 2nd, bronze for 3rd
- Rank numbers: gold/silver/bronze colors for top 3, muted for rest
- All headings: Barlow Condensed, uppercase, heavy weight
- Times/scores: JetBrains Mono
- Mobile-first, max content width ~480px centered on desktop
- Organizer entry: slide-up bottom sheet panel with backdrop overlay

---

## KEY PAGES & COMPONENTS

### Home page (app/page.tsx)
- Fetch all active locations from Supabase
- Render a card for each location with: color dot, name, city + venue, live/upcoming badge
- Tapping a card navigates to `/[slug]`
- Hero header with 🏁 and "EventHub" branding

### Location page (app/[slug]/page.tsx)
- Server component: fetch location, teams, activities, results from Supabase
- Pass to client component for interactivity
- Location color header banner with back arrow and "You are viewing: [NAME]" label
- Activity tabs: Overall + one tab per activity (show_live = true only)
- Overall tab: if overall_hidden = true, show locked banner; organizers see "Reveal" button
- Each tab: sorted leaderboard entries using scoring engine

### Organizer login (app/organizer/login/page.tsx)
- Simple email + password form using Supabase Auth
- On success, redirect to `/organizer/[slug]` for their assigned location

### Organizer entry page (app/organizer/[slug]/page.tsx)
- Protected by middleware — redirect to login if not authenticated
- Shows the leaderboard (same as attendee view)
- Floating "✏️ Enter Result" button (bottom right, location accent color)
- Tapping opens slide-up panel with:
  - Activity selector dropdown
  - Team selector dropdown
  - Status buttons: OK / DNF / DSQ / Incomplete
  - Value input (time in seconds for time type, number for score type)
  - Penalty counters (+ / − buttons) for each penalty rule on the activity
  - Save button
- On save: upsert result to Supabase, re-fetch and re-render leaderboard

### ResultEntryPanel component
- Slide-up from bottom, overlays content
- Controlled by open/close state
- Disables value + penalty inputs when status ≠ OK
- Shows penalty adjustment info (+2s per cone, etc.)

---

## API ROUTES

### GET /api/leaderboard/[slug]
```typescript
// Fetches location, teams, activities, results
// Runs scoring engine
// Returns { location, overall: OverallEntry[], activities: { [activityId]: LeaderboardEntry[] } }
```

### POST /api/results
```typescript
// Body: { activity_id, team_id, status, raw_value, penalties }
// Validates organizer has permission for this location's activity
// Upserts result (unique on activity_id + team_id)
// Returns updated result
```

---

## AUTH & MIDDLEWARE

```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  // Protect all /organizer/* routes
  // If no session → redirect to /organizer/login
  // If session but role !== 'organizer' and !== 'admin' → redirect to /
}

export const config = {
  matcher: ['/organizer/:path*']
}
```

---

## SUPABASE CLIENT SETUP

```typescript
// lib/supabase/client.ts  (browser)
import { createBrowserClient } from '@supabase/ssr'
export const supabase = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// lib/supabase/server.ts  (server components / API routes)
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
export function createClient() {
  const cookieStore = cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { getAll: () => cookieStore.getAll(), setAll: (c) => c.forEach(({ name, value, options }) => cookieStore.set(name, value, options)) } }
  )
}
```

```
# .env.local
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
```

---

## REALTIME (nice to have, do after core works)

After the core app works, add live leaderboard updates:

```typescript
// In the leaderboard client component, subscribe to results changes:
supabase
  .channel('results')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'results' }, () => {
    // re-fetch and re-render leaderboard
  })
  .subscribe()
```

---

## BUILD ORDER

Build in this exact order — get each step working before moving to the next:

1. **Supabase setup** — create project, run schema SQL, seed Dallas data manually in dashboard
2. **Next.js project** — create app, install deps, set up env vars, confirm `npm run dev` works
3. **Supabase clients** — `lib/supabase/client.ts` and `lib/supabase/server.ts`
4. **Types** — `types/index.ts`
5. **Scoring engine** — `lib/scoring/engine.ts` — write unit tests with `console.assert` to verify
6. **Home page** — fetch locations, render location cards
7. **Location page** — fetch all data, run scoring engine, render leaderboard with tabs
8. **Organizer login** — Supabase Auth email/password login
9. **Middleware** — protect /organizer routes
10. **Organizer entry page + ResultEntryPanel** — result entry, save to Supabase
11. **Polish** — match demo styles exactly, test on mobile viewport

---

## SETUP INSTRUCTIONS (after building)

1. Go to [supabase.com](https://supabase.com) → New Project
2. SQL Editor → run the schema SQL above
3. Authentication → Settings → enable Email auth, disable "Confirm email" for development
4. Get your Project URL and anon key from Settings → API
5. Run `npm run dev`, open `http://localhost:3000`
6. Sign up for an account, then in Supabase Table Editor set your profile role to `organizer`
7. In Table Editor, add a row to `organizer_locations` linking your user_id to a location
8. Deploy: `npx vercel` and set NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY

---

## NOTES FOR CLAUDE CODE

- The demo HTML file (`event-app-demo.html`) is your UI reference — match it visually
- The scoring engine logic in this file is already validated — implement it exactly
- All data is multi-tenant by location — never show one location's data on another
- Row Level Security handles data access — trust it, don't add redundant client-side filtering
- `raw_value` for time activities is stored in **milliseconds** (e.g. 63.412s = 63412)
- `raw_value` for score activities is stored as the raw score number
- The `penalties` field is a JSON object: `{ "cone": 2, "oc": 0 }`
- Overall standings hidden = attendees see a locked banner; organizers can reveal
- Keep the UI mobile-first — test at 390px width throughout
