
---

## The stack

**Backend:** ASP.NET Core 10 (minimal API style — much cleaner than the old controller-heavy approach you may be used to). Deployed to **Railway** — Vercel is serverless-only and doesn't support persistent .NET processes (no SignalR, no background services, slow cold starts). Render was the other candidate but Railway has better DX and no sleep penalty on paid plans. Railway's usage-based pricing (~$5/mo) may be low-traffic enough to stay near the free credit for a single-user app.

**Frontend:** React Native Web via **Expo**, deployed to **Vercel** as a **PWA (Progressive Web App)**. The target is mobile-first: once deployed, visit the URL in Safari on your iPhone and tap "Add to Home Screen" — it installs with its own icon, runs fullscreen, and behaves like a native app. No App Store, no TestFlight, no $99/year Apple Developer account, no 90-day expiry. Expo supports PWA out of the box. If a real App Store release is ever wanted later, the same codebase can be built for it via EAS — no rewrite needed.

**Database:** Supabase (Postgres). You'll talk to it from the .NET backend only — never directly from the frontend. The frontend only talks to your API.

**Auth:** Supabase Auth, handled by the .NET backend which validates the JWT on every request.

**Calorie pipeline:** OpenFoodFacts (free, no key needed) + a free-tier LLM. The best free option here is **Gemini Flash** (Google's free tier is generous — 1500 requests/day). The flow: user types "chicken tikka masala" → Gemini decomposes it into ingredients + estimated quantities → your backend queries OpenFoodFacts for each → sums the calories → returns the result. No ongoing cost.

---

## Folder structure

```
/healthapp
  /backend                  ← ASP.NET Core 10
    /HealthApp.API
      /Endpoints            ← Minimal API route groups
      /Services             ← Business logic
      /Models               ← DTOs and domain models
      /Data                 ← Supabase/Postgres access (Dapper or EF Core)
      Program.cs
      appsettings.json
  /frontend                 ← Expo (React Native Web)
    /src
      /screens              ← One folder per module
        /gym
        /calories
        /vices
        /dashboard
      /components           ← Shared UI components
      /hooks                ← Data fetching, auth state
      /api                  ← Typed fetch wrappers to your backend
    app.json
    package.json
```

---

## Database schema (outline)

```sql
-- Core
users (id, email, created_at)          ← mirrors Supabase auth.users

-- Gym
workout_plans    (id, user_id, name, created_at)
exercises        (id, name, muscle_group, notes)
plan_exercises   (plan_id, exercise_id, sets, reps, weight_kg)
workout_logs     (id, user_id, plan_id, logged_at)
exercise_logs    (id, workout_log_id, exercise_id, sets, reps, weight_kg)

-- Calories
meal_logs        (id, user_id, description, ingredients_json, total_kcal, logged_at)

-- Vices
vice_logs        (id, user_id, type [smoking|drinking], quantity, unit, logged_at)

-- Progress
weight_logs      (id, user_id, weight_kg, logged_at)
```

`ingredients_json` stores the raw Gemini+OpenFoodFacts breakdown so you can display it to the user without recomputing.

---

## Database schema — updated gym tables

```sql
-- Gym (revised)
workout_plans    (id, user_id, name, created_at)
exercises        (id, name, muscle_group, category, equipment, mechanic, force, instructions, created_at)
plan_exercises   (id, plan_id, exercise_id, order_index, sets, reps, weight_kg)
workout_logs     (id, user_id, plan_id, logged_at)
exercise_logs    (id, workout_log_id, exercise_id, sets, reps, weight_kg)
```

Changes from the original outline:
- `exercises` gets richer columns sourced from the seed data (`category`, `equipment`, `mechanic`, `force`, `instructions`)
- `plan_exercises` gets `order_index` (integer) so exercise order within a plan is explicit and reorderable
- `plan_exercises` gets its own `id` to make individual rows easier to update/delete from the API

---

## Exercise data — seeding strategy

Exercises are seeded once from **[free-exercise-db](https://github.com/yuhonas/free-exercise-db)** — a public domain JSON dataset of ~800 exercises. No API key, no rate limits, no ongoing dependency.

Each exercise entry in the dataset includes:
- `name` — exercise name
- `category` — primary muscle group (e.g. "Chest", "Back")
- `equipment` — required equipment (e.g. "Barbell", "Dumbbell", "Body Only")
- `mechanic` — Compound or Isolation
- `force` — Push / Pull / Static
- `instructions` — array of step-by-step instructions (stored as joined text)

**Seed approach:** a standalone C# console script (`/backend/tools/SeedExercises`) that:
1. Fetches the raw JSON from the free-exercise-db GitHub URL at runtime (or bundles it as an embedded resource for offline use)
2. Maps each entry to an `Exercise` record
3. Bulk-inserts into Supabase via the .NET backend's Postgres connection (Dapper `INSERT ... ON CONFLICT DO NOTHING` so re-running is safe)

The seed script runs once during initial setup and can be re-run safely. User-created exercises (custom additions) can coexist in the same table with a `is_custom` flag and `user_id` nullable column if needed later.

---

## Gym module — feature design

### Workout plans

A user can have multiple named plans (e.g. "Push Day A", "Pull Day B", "Leg Day").

**Create plan:**
- Enter a plan name
- Add exercises one by one: search/filter the `exercises` table by name or muscle group, select one, set default sets / reps / weight for that exercise
- Exercises are ordered; the user can reorder them (stored in `order_index`)

**Edit plan:**
- Rename the plan
- Add, remove, or reorder exercises
- Change sets / reps / weight per exercise (these are the *default* values used when logging; they can be overridden per session)

**Delete plan:** soft-delete or hard-delete — to decide, but workout history referencing the plan should be preserved regardless.

### Logging a session

User picks a plan and starts a session → creates a `workout_logs` row.

For each exercise in the plan (in order):
- The UI shows the exercise name and the plan's default sets/reps/weight
- User can override weight for that session (e.g. they went heavier today)
- User marks each set done, or logs the actual reps/weight if different from the plan
- Each completed exercise → `exercise_logs` row

Session can be abandoned mid-way; partial logs are kept.

### Progress view (Phase 6 scope)

Per-exercise chart: weight over time (last logged weight for that exercise per session). Shows whether the user is progressing. Uses `exercise_logs` joined to `workout_logs`.

---

## Build plan — 6 phases

**Phase 1 — Foundation (1–2 weeks)**
Set up the repo, Supabase project, .NET solution, and Expo app. Get a "hello world" flowing end to end: Expo web → .NET API → Supabase. Configure Supabase Auth and JWT validation in .NET. Deploy the backend to Railway (free tier) and frontend to Vercel. No features yet, just the skeleton working in production.

**Phase 2 — Auth + shell (3–5 days)**
Login/logout screen in Expo. Protected routes. A tab bar with the four sections (Gym, Calories, Vices, Dashboard) — all empty for now. This gives you the navigation skeleton to build into.

**Phase 3 — Weight & dashboard (1 week)**
Start with the simplest data: weight logging. A screen to log today's weight, a line chart of progress over time. This is a good phase to learn the full round trip — Expo form → POST to .NET → insert to Supabase → GET → render chart — without complex business logic.

**Phase 4 — Vices tracker (3–5 days)**
Log cigarettes and drinks. Daily/weekly summary. Streak counters ("X days since last cigarette"). Simple but motivating to use.

**Phase 5 — Calorie tracker (1–2 weeks)**
This is the most complex module. Build the Gemini → OpenFoodFacts pipeline in a dedicated `CalorieService` on the backend. Text input on the frontend, show the ingredient breakdown so you can see and correct it, confirm and log. Daily calorie total on the dashboard.

**Phase 6 — Gym planner & tracker (2 weeks)**
Run the exercise seed script first (one-time). Then: create/edit workout plans (pick exercises from the seeded DB, set order/sets/reps/weight). Log a workout session against a plan, with per-set weight overrides. Progress chart per exercise. See *Gym module — feature design* above for the full breakdown.

---

## A few .NET 10 things worth knowing upfront

Since you're coming from legacy .NET, the biggest shifts are: **minimal APIs** (no controllers, routes are registered directly in `Program.cs` or small endpoint files), **top-level statements** (no `Main` boilerplate), and the DI container is now configured entirely in `Program.cs`. It's actually less ceremony than what you're used to. For data access I'd suggest **Dapper** over EF Core — it's closer to raw SQL which will feel familiar and is a better fit for Supabase's Postgres.

---