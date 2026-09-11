# Tech Stack

Keep this stack exactly as specified. No frontend framework (React/Vue/etc.) and no PHP framework (Laravel/Symfony/etc.) — this is intentionally vanilla so it's fast to reason about and easy to demo/deploy without a build step.

## Frontend
- **HTML5** — semantic markup, one `.php` file per screen (PHP is used as a templating layer even for mostly-static pages, so session/role checks can gate access server-side before any HTML renders).
- **CSS3** — a single global stylesheet (`/public/assets/css/style.css`) plus small page-specific overrides where needed. Mobile-first media queries — design for a 375px-wide screen first, then scale up.
- **Vanilla JavaScript (ES6+)** — no jQuery, no bundler. Use the native `fetch()` API for all calls to the PHP backend, which return JSON.
- **Geolocation API** (`navigator.geolocation`) — used on donor signup/posting and NGO feed to get lat/lng.
- **Leaflet.js** (via CDN, no build step) — for the NGO feed's map view and listing pins. Lightweight, no API key required (unlike Google Maps), which matters for a hackathon demo.
- **Chart.js** (via CDN) — for the public impact dashboard's counters/leaderboard visualizations, if time allows.

## Backend
- **PHP 8+** — procedural or lightly object-oriented; no framework. Each API endpoint is its own file under `/public/api/`.
- **PDO with prepared statements** — for all database access, to avoid SQL injection. Never use raw string-interpolated queries.
- **Session-based auth** (`$_SESSION`) — role (`donor` / `ngo` / `recycler`) stored in session after login; every protected endpoint checks `$_SESSION['role']` before acting.
- **`password_hash()` / `password_verify()`** — for credential storage, never plaintext.
- **File uploads** — handled via PHP's `$_FILES`, validated by MIME type and size, stored under `/public/uploads/` with a randomized filename (never trust the original filename).

## Database
- **MySQL 8+** (or MariaDB) — see `database-schema.md` for full schema.
- Distance queries use the **Haversine formula** directly in SQL (no PostGIS/spatial extension needed at this scale) — see `database-schema.md` for the query pattern.

## Real-time-ish behavior
No websockets. Status pages (Listing Status, NGO Feed) use **polling** — a `setInterval` calling a lightweight `fetch()` endpoint every 10–15 seconds to refresh state. This is an accepted, documented tradeoff for a hackathon timeline; note it explicitly if a judge asks about real-time architecture.

## Background/timeout logic
Two time-based behaviors need to run without a user actively on the page:
1. Radius auto-expansion after 45 minutes unclaimed (configurable per category).
2. Auto-transition to `EXPIRED_SPOILED` when `expiry_window` elapses.

Implement these as **checked on read**, not via a cron job: every query that fetches listings (the NGO feed, the recycling hub) first runs a lightweight "sweep" query that updates any listing whose window has elapsed, before returning results. This avoids needing a cron/scheduled task, which is fragile in a typical hackathon hosting environment. If deployment allows a cron job later, this can be moved server-side — note it in `phases.md` Phase 6 as a stretch goal.

## Local development
- `php -S localhost:8000 -t public` to run the app.
- MySQL via XAMPP, MAMP, or a local Docker container — whichever is fastest to set up on the dev machine.
- No `npm install`, no build step, no transpilation. Open and edit files directly.

## Deployment target (for demo day)
Any standard LAMP-compatible host (shared hosting, a small VM, or a platform like Railway/Render with a PHP buildpack) works, since there's no build pipeline. Keep this in mind when choosing libraries — CDN-hosted JS (Leaflet, Chart.js) avoids needing a package manager in production too.
