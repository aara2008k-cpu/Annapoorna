# File Structure

```
Annapoorna/
├── README.md
├── PRD.md
├── techstack.md
├── database-schema.md
├── api-endpoints.md
├── phases.md
├── file-structure.md
│
├── database/
│   └── schema.sql              # CREATE TABLE statements matching database-schema.md exactly
│
├── config/
│   └── db.php                  # PDO connection, reads from env or hardcoded local creds
│
└── public/                     # this is the web root — point the PHP server / Apache vhost here
    ├── index.php                # Landing / Impact Dashboard (public route: /)
    ├── auth.php                 # Login + registration UI (public route: /auth)
    │
    ├── donor/
    │   ├── dashboard.php         # /donor/dashboard
    │   ├── post.php              # /donor/post
    │   └── status.php            # /donor/status/:id  (reads ?id= query param)
    │
    ├── ngo/
    │   ├── feed.php              # /ngo/feed
    │   ├── claim.php             # /ngo/claim/:id
    │   └── verify.php            # /ngo/verify/:id
    │
    ├── recycling/
    │   ├── index.php             # /recycling
    │   └── claim.php             # /recycling/claim/:id
    │
    ├── api/
    │   ├── register.php
    │   ├── login.php
    │   ├── logout.php
    │   ├── listings/
    │   │   ├── create.php
    │   │   ├── status.php
    │   │   ├── feed.php
    │   │   └── claim.php
    │   ├── claims/
    │   │   ├── verify.php
    │   │   ├── complete.php
    │   │   └── no-show.php
    │   ├── recycling/
    │   │   ├── feed.php
    │   │   ├── claim.php
    │   │   └── log-intake.php
    │   ├── donor/
    │   │   └── dashboard.php
    │   └── public/
    │       ├── metrics.php
    │       └── leaderboard.php
    │
    ├── includes/
    │   ├── header.php            # shared nav, includes session/role check
    │   ├── footer.php
    │   └── auth-guard.php        # role-checking helper, included at top of protected pages
    │
    ├── assets/
    │   ├── css/
    │   │   └── style.css         # single global stylesheet — see techstack.md
    │   └── js/
    │       ├── feed.js           # NGO feed polling + Leaflet map
    │       ├── post-listing.js   # donor post form logic + geolocation capture
    │       ├── status.js         # donor listing status polling
    │       ├── verify.js         # NGO checklist + OTP verification form
    │       └── dashboard.js      # public dashboard Chart.js rendering
    │
    └── uploads/                  # user-submitted photos (listing photos, verification docs, checklist photos)
        ├── listings/
        ├── verification/
        └── docs/
```

## Conventions
- Every file under `public/donor/`, `public/ngo/`, `public/recycling/` starts with an `include 'includes/auth-guard.php'` call that checks `$_SESSION['role']` matches the expected role, and redirects to `/auth.php` if not — this is what makes the route table in PRD §5 actually enforced, not just documented.
- Every file under `public/api/` returns JSON only — no HTML, no `echo` of raw strings outside `json_encode()`.
- `uploads/` should never be trusted for filenames — see `techstack.md` on randomized filenames.
- Don't create a `vendor/` or `node_modules/` — there's no package manager in this stack (see `techstack.md`).
