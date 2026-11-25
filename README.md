# Fretly — Interactive Guitar Learning System

React + TypeScript + Vite frontend, Express + TypeScript API, Supabase (Auth + Postgres + RLS).

> **Read this first — what has and hasn't been verified.**
> This code was written in a sandbox with **no network access**, so `npm install`, `vite build`,
> the Express server, and Supabase were **never run**. What *was* executed successfully:
> the music-theory/chord-data tests (19), the dashboard-statistics tests (9), and the seed generator.
> Everything else was syntax-checked and loosely type-checked against stubs only.
> See [Verification status](#verification-status) and run the checklist there before deploying.

## Architecture & responsibilities

| Layer | Responsibility |
|---|---|
| **Browser (React)** | UI, chord/progression/transposition logic (pure TS, bundled → works offline), audio synthesis, microphone tuner/analyzer (local only), Supabase Auth (sign-up/in/out/reset). |
| **Express API** (`server/`) | Verifies the Supabase access token on every private route, validates input with Zod, rate-limits, computes dashboard stats, and talks to Postgres **as the caller** (their JWT is forwarded, so RLS applies). |
| **Supabase Postgres** | Source of truth. Row Level Security on every user-owned table; transactional SQL functions for multi-step writes; seeded reference data. |

Security decisions:
* **No service-role key anywhere.** The API uses the publishable key + the caller's JWT. Account deletion is therefore a *request* (`profiles.deletion_requested_at`) completed by an admin in the Supabase dashboard.
* User identity always comes from the verified token (`supabase.auth.getUser`), never from request bodies/params.
* Practice-session duration is computed in Postgres (`finish_practice_session`) from server timestamps; clients cannot submit a duration.
* Raw microphone audio never leaves the browser.

## Quick start (local)

Requires Node 20+. The Vite config pins the dev server to port **8443** (set `PORT=5173` to change).

```bash
# 1. install
npm install
npm run server:install

# 2. configure
cp .env.example .env                 # fill in VITE_SUPABASE_URL / VITE_SUPABASE_PUBLISHABLE_KEY
cp server/.env.example server/.env   # fill in SUPABASE_URL / SUPABASE_PUBLISHABLE_KEY

# 3. database: see "Connect Supabase" below

# 4. run (two terminals)
npm run server:dev     # API on http://localhost:3001
npm run dev            # web on http://localhost:8443
```

Without Supabase variables the app still runs in **guest mode** (Tuner, Chord Library, Chord Analyzer).

## Connect Supabase

1. Create a project at supabase.com.
2. **Auth → Providers → Email**: enable; keep "Confirm email" on for email verification.
3. **Auth → URL Configuration**: set *Site URL* to your frontend URL and add redirect URLs
   (`http://localhost:8443`, and your production URL). Password-reset and verification links return there.
4. **Apply migrations** (SQL editor, in order) — or `supabase db push` with the Supabase CLI:
   1. `supabase/migrations/20260101000000_schema.sql`
   2. `supabase/migrations/20260101000100_seed_reference_data.sql`
5. Copy **Project URL** and the **publishable (anon) key** from *Project Settings → API* into both `.env` files.
6. Run `supabase/tests/rls_isolation.sql` in the SQL editor; every line must print `PASS`.

Regenerate the seed after editing `src/data/*`: `npm run db:seed:generate` (idempotent upserts; never deletes user data).

## Tests

```bash
npm test               # music theory, chord data validation, transposition, audio helpers
npm run test:server    # dashboard statistics (timezones, streaks, boundaries)
npm run test:all
npm run typecheck      # frontend tsc
npm --prefix server run typecheck
```

## API (all under `/api`, JSON `{ data, meta? }` / `{ error: { code, message } }`)

| Route | Auth | Purpose |
|---|---|---|
| `GET /health` | – | liveness |
| `GET/PATCH /profile`, `POST/DELETE /profile/deletion-request` | ✔ | profile & deletion request |
| `GET /lessons`, `GET /lessons/:id` | – | lesson catalogue |
| `GET /progress`, `GET/PUT /progress/:lessonId`, `POST /progress/:lessonId/attempts` | ✔ | per-user lesson progress |
| `GET/POST /practice-sessions`, `POST /:id/finish`, `GET/DELETE /:id`, `GET /stats/summary` | ✔ | sessions & analytics |
| `GET /chords`, `GET /chords/:id` | – | dictionary (+ caller's custom voicings if signed in) |
| `GET /chords/favorites/mine`, `PUT/DELETE /chords/favorites/:chordId` | ✔ | favourites |
| `POST /chords/:id/voicings`, `DELETE /chords/voicings/:id` | ✔ | custom voicings |
| `GET/POST /collections`, `PATCH/DELETE /:id`, `PUT /:id/items`, `DELETE /:id/items/:chordId` | ✔ | collections |
| `GET /progressions/templates` | – | template library |
| `GET/POST /progressions`, `GET/PATCH/DELETE /:id`, `POST /:id/duplicate` | ✔ | saved progressions |
| `GET /songs`, `GET /songs/:id` | – | songs |
| `GET /songs/me/state`, `PUT /songs/:id/progress`, `PUT/DELETE /songs/:id/favorite` | ✔ | song status & favourites |

## Deployment

**Frontend (Vercel/Netlify):** build `npm run build`, output `dist/`. Env: `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_API_BASE_URL=https://<your-api>/api`.

**API (Render/Railway/Fly):** root `server/`, build `npm install && npm run build`, start `npm start`.
Env: `NODE_ENV=production`, `PORT` (provided by host), `FRONTEND_URL=https://<your-frontend>` (comma-separate multiple origins, no trailing slash), `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`.
The API trusts one proxy hop in production for correct rate-limit IPs.

**Verify after deploy:** `GET https://<api>/api/health`; register → confirm email → sign in on device A; finish a session and tick a lesson step; sign in on device B and confirm the same data appears.

**Backups:** Supabase Pro includes daily backups/PITR; on the free tier schedule `pg_dump` exports. Seed data is reproducible from the repo.

## Design notes

* The original look is unchanged: new UI reuses existing classes/tokens; new CSS is appended at the end of `src/index.css`.
* Added nav items: **Progressions, Songs, History** (plus a working **Settings**). Original "Octave" filter pill became **Power** (an octave dyad isn't a chord); the original G-octave card was dropped for that reason.
* The dashboard no longer shows invented numbers ("12 day streak", "84 minutes", "Day 12 of 30"). "of 30" was removed because no 30-day curriculum exists in the data.
* Chord voicings are labelled **curated** (hand-entered; notes machine-verified against the chord formula, fingerings **not** teacher-reviewed) or **generated** (movable shapes slid by arithmetic). Nothing is labelled "verified".
* Enharmonic simplification: a few theoretical spellings (e.g. E aug third → "C" rather than "B♯") are shown in simplest form.

## Verification status

| Item | Status |
|---|---|
| Theory, chord data (every voicing's notes checked), transposition, Roman numerals, diatonic chords, audio helpers | ✅ 19 tests **executed and passing** |
| Dashboard stats, streaks, timezone/midnight boundaries | ✅ 9 tests **executed and passing** |
| Seed SQL generation | ✅ executed (220 chords, 261 voicings, 10 progressions, 4 songs, 1 lesson) |
| Frontend/server syntax | ✅ every file parses (esbuild) |
| Frontend/server type-checking | ⚠️ only against hand-written stubs, no real dependency types |
| `vite build`, running UI, visual regression vs original | ❌ **not run** |
| Express server startup, any HTTP route | ❌ **not run** |
| SQL migrations, RLS policies, RPC functions | ❌ **not run** (reviewed by hand; run `supabase/tests/rls_isolation.sql`) |
| Auth flows (register, verify, login, reset), cross-device sync | ❌ **not run** |
| Backend API/auth/RLS automated tests, frontend component tests | ❌ **not written/run** |
| Tuner on a physical guitar | ❌ untested (logic unchanged from original except configurable reference pitch) |

## Feature status

**Implemented (unrun):** auth UI & session handling, profile/settings, lesson progress (cloud), practice session start/finish, dashboard & history analytics, chord dictionary (12 roots × 18 qualities + slash chords), voicing diagrams, detail panel, playback (Web Audio), favourites, collections, custom voicings (note-validated), progression templates, builder (diatonic palette, reorder, bars, voicings, tempo, transposition preview, save/edit/duplicate/delete/favourite, practice), songs (transpose, favourite, status, practice), tuner presets & reference pitch.

**Partial:** only one lesson is seeded (the original Day 12 content); roman-numeral chord-tone analysis for non-diatonic chords is convention-based; quiz scores/attempt endpoints exist but the original lesson has no quiz UI; only three Library "pills" use voicing tags.

**Not done:** user-created songs UI (schema + RLS exist); personal song collections beyond favourites; Supabase Storage/avatars; tuner-usage practice events; chord-collection search/sort UI beyond the API; admin account deletion tooling; backend integration tests; frontend tests.

## Created / modified files

**Modified:** `src/App.tsx` (original markup retained; data wired; shell/gating), `src/main.tsx`, `src/index.css` (append only), `src/vite-env.d.ts`, `package.json`, `.gitignore`.
**Original backup:** none committed; original `App.tsx` was preserved outside the repo during editing.
**Created — frontend:** `src/auth/{AuthContext,AuthScreen}.tsx`, `src/state/AppData.tsx`, `src/lib/{api,supabase,types,tunings}.ts`, `src/lib/theory/{notes,chordEngine,progression}.ts`, `src/lib/audio/chordPlayer.ts`, `src/data/{chordLibrary,progressionTemplates,songs,lessons}.ts`, `src/components/{ChordDiagram,ChordDetail,Icon,SessionBar,Toasts}.tsx`, `src/pages/{Library,Progressions,Songs,History,Settings}.tsx`, `src/lib/theory/__tests__/theory.test.ts`, `.env.example`.
**Created — backend:** `server/{package.json,tsconfig.json,.env.example}`, `server/src/{index,app,env,supabase}.ts`, `server/src/middleware/{auth,errors,validate}.ts`, `server/src/lib/{schemas,stats}.ts`, `server/src/routes/*.ts` (9 modules), `server/test/stats.test.ts`.
**Created — database:** `supabase/migrations/20260101000000_schema.sql`, `supabase/migrations/20260101000100_seed_reference_data.sql` (generated), `supabase/tests/rls_isolation.sql`, `scripts/generate-seed.ts`.
