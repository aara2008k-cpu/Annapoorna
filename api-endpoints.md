# API Endpoints

All endpoints live under `/public/api/` as individual PHP files. All responses are JSON: `{ "success": bool, "data": ..., "error": string|null }`. All POST bodies are JSON, parsed via `json_decode(file_get_contents('php://input'), true)`.

## Auth

### `POST /api/register.php`
Body: `{ role, org_name, email, password, phone, lat, lng, verification_doc? }`
Creates a `users` row. If `role` is `ngo` or `recycler`, `verified` defaults to false and an admin must approve before `login` allows claiming.

### `POST /api/login.php`
Body: `{ email, password }` → sets `$_SESSION['user_id']`, `$_SESSION['role']`. Returns user object (no password hash).

### `POST /api/logout.php`
Clears session.

## Donor

### `POST /api/listings/create.php`
Auth: `role=donor`. Body: `{ category_id, portions_posted, cooked_at, lat, lng, photo? }`.
Server computes `expiry_at` from `food_categories.default_expiry_minutes`, generates a 4-digit `otp`, inserts with `status='AVAILABLE'`. Returns the listing including the OTP (shown only to this donor).

### `GET /api/listings/status.php?id={listing_id}`
Auth: `role=donor`, must own the listing. Returns current status, claimant info (if `RESERVED` or later), and countdown to `expiry_at`.

### `GET /api/donor/dashboard.php`
Auth: `role=donor`. Returns active listings, donation history, and CSR score (`SUM(portions_verified)` across this donor's `COMPLETED` listings).

## NGO

### `GET /api/listings/feed.php?lat={}&lng={}`
Auth: `role=ngo`, `verified=true`. Runs the sweep query (see `database-schema.md`) then the Haversine distance query, filtered to `status='AVAILABLE'`, ordered by `expiry_at ASC`. Returns listings with `distance_km`.

### `POST /api/listings/claim.php`
Auth: `role=ngo`. Body: `{ listing_id }`.
Runs the atomic conditional update: `UPDATE listings SET status='RESERVED' WHERE id=? AND status='AVAILABLE'`. If affected rows = 0, return `{ success: false, error: "already_claimed" }`. Otherwise insert a `claims` row and return success.

### `POST /api/claims/verify.php`
Auth: `role=ngo`, must own the claim. Body: `{ claim_id, otp, portions_verified, checklist_odor_ok, checklist_visual_ok, checklist_storage_ok, photo }`.
Validates `otp` against `listings.otp`. On match: sets `claims.otp_verified_at`, stores checklist + photo + `portions_verified`, updates `listings.status='IN_TRANSIT'` and `listings.portions_verified`. On mismatch: return error, no state change.

### `POST /api/claims/complete.php`
Auth: `role=ngo`. Body: `{ claim_id }`. Sets `claims.completed_at`, `listings.status='COMPLETED'`.

### `POST /api/claims/no-show.php`
Auth: `role=ngo`. Body: `{ claim_id }`. Only allowed if `claimed_at` is more than the grace period (15 min) in the past. Sets `no_show_reported=true`, reverts `listings.status='AVAILABLE'`, decrements the donor's `trust_score`.

## Recycler

### `GET /api/recycling/feed.php`
Auth: `role=recycler`, `verified=true`. Returns listings with `status='EXPIRED_SPOILED'`.

### `POST /api/recycling/claim.php`
Auth: `role=recycler`. Body: `{ listing_id }`. Same atomic-claim pattern as NGO claim, but from `EXPIRED_SPOILED` → `RECYCLE_CLAIMED`.

### `POST /api/recycling/log-intake.php`
Auth: `role=recycler`. Body: `{ claim_id, bulk_weight_kg }`. Sets `claims.bulk_weight_kg`, `claims.completed_at`, `listings.status='RECYCLED'`.

## Public

### `GET /api/public/metrics.php`
No auth. Returns aggregate counters computed per PRD §9:
```json
{
  "meals_saved": "SUM(portions_verified) WHERE status='COMPLETED'",
  "organic_waste_diverted_kg": "SUM(portions_posted) * 0.45 across COMPLETED + RECYCLED",
  "co2e_offset_kg": "organic_waste_diverted_kg * 0.45"
}
```

### `GET /api/public/leaderboard.php?type=donors|ngos`
No auth. Returns top orgs ranked by verified activity (donors: `SUM(portions_verified)`; NGOs: count of `COMPLETED` claims).

## Error conventions
- `401` — not logged in
- `403` — wrong role, or `verified=false` trying to claim
- `409` — conflict (e.g. already-claimed listing)
- `422` — validation failure (e.g. OTP mismatch, missing required field)
- All errors return `{ "success": false, "error": "<machine_readable_code>" }` — the frontend maps codes to user-facing copy, so don't put display text in the API response.
