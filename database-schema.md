# Database Schema

MySQL 8+. Table and column names here are canonical — use them exactly as written across all backend code so multi-session agent work stays consistent.

## `users`
| Column | Type | Notes |
|---|---|---|
| id | INT AUTO_INCREMENT PK | |
| role | ENUM('donor','ngo','recycler','admin') | |
| org_name | VARCHAR(150) | |
| email | VARCHAR(150) UNIQUE | |
| password_hash | VARCHAR(255) | via `password_hash()` |
| phone | VARCHAR(20) | masked from other users except during an active claim |
| lat | DECIMAL(10,7) | |
| lng | DECIMAL(10,7) | |
| verification_doc_path | VARCHAR(255) NULL | required for `ngo` and `recycler` roles |
| verified | BOOLEAN DEFAULT FALSE | admin must approve before `ngo`/`recycler` can claim listings |
| trust_score | DECIMAL(4,1) DEFAULT 100.0 | decremented on no-shows / flagged discrepancies |
| created_at | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | |

## `food_categories`
Lookup table — this is what makes the safety constraint (PRD §7) enforceable server-side, not just a hardcoded list in the frontend.

| Column | Type | Notes |
|---|---|---|
| id | INT AUTO_INCREMENT PK | |
| name | VARCHAR(100) | e.g. "Cooked rice/grains", "Bread", "Sealed packaged goods" |
| donor_selectable | BOOLEAN DEFAULT TRUE | false for anything requiring cold-chain — excluded from the dropdown entirely |
| default_expiry_minutes | INT | drives auto-calculated `expiry_window` on the listing |
| radius_expand_minutes | INT | per-category override of the 45-min default (PRD §5.2, FR8) |

## `listings`
| Column | Type | Notes |
|---|---|---|
| id | INT AUTO_INCREMENT PK | |
| donor_id | INT FK → users.id | |
| category_id | INT FK → food_categories.id | |
| portions_posted | INT | donor's claimed count |
| portions_verified | INT NULL | filled in by NGO/recycler at claim verification — see PRD §9, meals_saved uses this |
| photo_path | VARCHAR(255) NULL | |
| cooked_at | TIMESTAMP | |
| expiry_at | TIMESTAMP | `cooked_at + food_categories.default_expiry_minutes`, computed at insert |
| lat | DECIMAL(10,7) | |
| lng | DECIMAL(10,7) | |
| status | ENUM('AVAILABLE','RESERVED','IN_TRANSIT','COMPLETED','EXPIRED_SPOILED','RECYCLE_CLAIMED','RECYCLED') DEFAULT 'AVAILABLE' | see PRD §4 for the full state diagram |
| radius_km | DECIMAL(3,1) DEFAULT 5.0 | expands to 10.0 per `food_categories.radius_expand_minutes` sweep logic |
| otp | CHAR(4) | generated at insert, shown only to donor |
| created_at | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | |

## `claims`
One row per NGO or recycler claim attempt on a listing — keeps the audit trail separate from the listing itself.

| Column | Type | Notes |
|---|---|---|
| id | INT AUTO_INCREMENT PK | |
| listing_id | INT FK → listings.id | |
| claimant_id | INT FK → users.id | the NGO or recycler |
| claimant_role | ENUM('ngo','recycler') | |
| claimed_at | TIMESTAMP DEFAULT CURRENT_TIMESTAMP | |
| otp_verified_at | TIMESTAMP NULL | set when OTP correctly entered — moves listing to `IN_TRANSIT` |
| checklist_odor_ok | BOOLEAN NULL | |
| checklist_visual_ok | BOOLEAN NULL | |
| checklist_storage_ok | BOOLEAN NULL | |
| verification_photo_path | VARCHAR(255) NULL | |
| completed_at | TIMESTAMP NULL | drop-off confirmation (NGO) or intake logged (recycler) |
| bulk_weight_kg | DECIMAL(6,2) NULL | recycler-only field |
| no_show_reported | BOOLEAN DEFAULT FALSE | releases listing back to `AVAILABLE` |

## Distance query pattern (Haversine, used by the NGO feed)
```sql
SELECT l.*, 
  (6371 * ACOS(
    COS(RADIANS(:ngo_lat)) * COS(RADIANS(l.lat)) *
    COS(RADIANS(l.lng) - RADIANS(:ngo_lng)) +
    SIN(RADIANS(:ngo_lat)) * SIN(RADIANS(l.lat))
  )) AS distance_km
FROM listings l
WHERE l.status = 'AVAILABLE'
HAVING distance_km <= l.radius_km
ORDER BY l.expiry_at ASC;
```

## Sweep query pattern (run before every feed read — see `techstack.md`)
```sql
-- Expand radius for listings unclaimed past their category's window
UPDATE listings l
JOIN food_categories fc ON l.category_id = fc.id
SET l.radius_km = 10.0
WHERE l.status = 'AVAILABLE'
  AND l.radius_km = 5.0
  AND TIMESTAMPDIFF(MINUTE, l.created_at, NOW()) > fc.radius_expand_minutes;

-- Expire listings past their expiry window
UPDATE listings
SET status = 'EXPIRED_SPOILED'
WHERE status = 'AVAILABLE' AND expiry_at < NOW();
```
