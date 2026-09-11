# Build Phases

Ordered so there's something demoable after every phase — never leave the project in a state where nothing runs end-to-end. If time runs short, stop after the phase you're in rather than leaving multiple phases half-done.

## Phase 0 — Foundation (do this first, don't skip)
- Set up the folder structure exactly as in `file-structure.md`.
- Create `database/schema.sql` from `database-schema.md` and run it against a local MySQL instance.
- Build `config/db.php` (PDO connection) and confirm it connects.
- Build `includes/auth-guard.php`, `register.php`, `login.php`, `logout.php`.
- **Demo checkpoint:** you can register as each of the three roles and log in/out.

## Phase 1 — Donor flow (core loop, half 1)
- `donor/post.php` + `api/listings/create.php` — posting a listing, with the category dropdown restricted per `food_categories.donor_selectable` (PRD §7).
- `donor/dashboard.php` + `api/donor/dashboard.php` — list active + past listings.
- `donor/status.php` + `api/listings/status.php` — shows the OTP and a live countdown (polling, per `techstack.md`).
- **Demo checkpoint:** a donor can post a listing and see it sitting in `AVAILABLE` with a visible countdown.

## Phase 2 — NGO flow (core loop, half 2)
- `ngo/feed.php` + `api/listings/feed.php` — the Haversine-sorted, urgency-sorted feed with the Leaflet map.
- `ngo/claim.php` + `api/listings/claim.php` — atomic claim (PRD §8). Test the race condition deliberately (two rapid claim requests) before moving on.
- `ngo/verify.php` + `api/claims/verify.php` — OTP entry, checklist, photo upload, `portions_verified`.
- `api/claims/complete.php` for the final drop-off confirmation.
- **Demo checkpoint:** full loop works — donor posts, NGO claims, NGO verifies with OTP, listing reaches `COMPLETED`. This is the demo you show judges if nothing else gets built.

## Phase 3 — Recycling flow
- Implement the sweep query (`database-schema.md`) so listings actually transition to `EXPIRED_SPOILED` on read.
- `recycling/index.php` + `api/recycling/feed.php`.
- `recycling/claim.php` + `api/recycling/claim.php` + `api/recycling/log-intake.php`.
- **Demo checkpoint:** let a test listing expire (or seed one with a past `expiry_at`), show it landing in the recycling feed, claim it, log intake, watch it reach `RECYCLED`.

## Phase 4 — Public dashboard
- `index.php` + `api/public/metrics.php` — the meals-saved / kg-diverted / CO2e counters using **verified** numbers, per PRD §9.
- `api/public/leaderboard.php` — top donors and top NGOs.
- Chart.js rendering in `dashboard.js`.
- **Demo checkpoint:** the numbers on the landing page actually move as you complete listings in the other flows — this is often the single most persuasive thing to show judges live.

## Phase 5 — Edge cases & trust features
- `api/claims/no-show.php` — donor no-show reporting and trust-score decrement.
- Radius auto-expansion (part of the sweep query) — verify it actually moves a stale listing from 5km to 10km visibility.
- NGO/recycler `verified` gate — an admin approval step (can be a manual DB update for the hackathon; a real admin UI is a stretch goal, see Phase 6).
- **Demo checkpoint:** you can narrate the safety/trust story (checklist, OTP, verified-only claiming) with a live example, not just describe it.

## Phase 6 — Stretch goals (only if time remains)
- Admin UI for NGO/recycler verification approval (instead of manual DB edit).
- Move sweep-query logic to a real cron job instead of check-on-read.
- Basic email verification on signup.
- Responsive polish pass — test every screen at 375px width specifically, since NGO staff will likely use this on a phone in the field.
- CSV export of the leaderboard/metrics for CSR reporting (ties into the revenue model discussed in pitch prep).

## What NOT to do
- Don't build the admin UI before Phase 2's core loop works — judges care about the live demo, not admin tooling.
- Don't add a JS framework or PHP framework partway through to "move faster" — it'll cost more time than it saves this close to a stack decision already made (see `techstack.md`).
- Don't skip the deliberate race-condition test in Phase 2 — "what happens if two NGOs claim at once" is a question judges will ask (see the pitch prep doc), and it's much easier to test now than to debug live during Q&A.
