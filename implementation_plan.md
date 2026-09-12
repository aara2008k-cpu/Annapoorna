# Implementation Plan — Phase 0: Foundation

Phase 0 establishes the bedrock of Annapoorna in accordance with the canonical specs: directory layout, database schema migration, PDO database connection layer, session authentication endpoints, role-based access guard, and the unified authentication UI.

## Spec Adherence Summary

- **Stack**: Vanilla HTML5/CSS3/JavaScript + PHP 8+ / MySQL. No frameworks, bundlers, npm, or composer packages.
- **Database Access**: 100% PDO with prepared statements and typed parameter bindings. Zero string-interpolated SQL.
- **Conventions**: JSON API protocol `{ "success": bool, "data": ..., "error": string|null }` with standard HTTP status codes.
- **Folder Layout**: Exact match to [file-structure.md](file:///c:/Users/Harshil/Downloads/Annapoorna/file-structure.md).

---

## User Review & Environment Prerequisite

> [!IMPORTANT]
> **PHP & MySQL Environment Setup**
> Preliminary system inspection shows that neither `php` nor `mysql` / `mysqld` is currently available in the system `PATH`.
> To run the local dev server (`php -S localhost:8000 -t public`) and host the MySQL database (`Annapoorna`), we need a working PHP and MySQL environment.
>
> Please confirm your preferred setup:
> 1. Do you already have XAMPP, Laragon, WampServer, or Docker installed at a specific directory (e.g. `D:\xampp`, custom path)?
> 2. Or would you like automated assistance downloading and configuring a lightweight portable PHP 8.x zip and MariaDB/MySQL for Windows into a local dev tools directory?
> 3. What are your local MySQL connection credentials (default assumed: `host=127.0.0.1`, `port=3306`, `user=root`, `password=""`, `dbname=Annapoorna`)?

---

## Proposed Changes — Phase 0

### 1. Directory Structure

Establish the directory tree defined in [file-structure.md](file:///c:/Users/Harshil/Downloads/Annapoorna/file-structure.md):

- `database/`
- `config/`
- `public/`
  - `donor/`
  - `ngo/`
  - `recycling/`
  - `api/`
    - `listings/`
    - `claims/`
    - `recycling/`
    - `donor/`
    - `public/`
  - `includes/`
  - `assets/`
    - `css/`
    - `js/`
  - `uploads/`
    - `listings/`
    - `verification/`
    - `docs/`

---

### 2. Database Schema & Migration

#### [NEW] [schema.sql](file:///c:/Users/Harshil/Downloads/Annapoorna/database/schema.sql)
Implement canonical DDL script strictly following [database-schema.md](file:///c:/Users/Harshil/Downloads/Annapoorna/database-schema.md):
- `users` table:
  - `id INT AUTO_INCREMENT PRIMARY KEY`
  - `role ENUM('donor','ngo','recycler','admin') NOT NULL`
  - `org_name VARCHAR(150) NOT NULL`
  - `email VARCHAR(150) UNIQUE NOT NULL`
  - `password_hash VARCHAR(255) NOT NULL`
  - `phone VARCHAR(20) NULL`
  - `lat DECIMAL(10,7) NULL`
  - `lng DECIMAL(10,7) NULL`
  - `verification_doc_path VARCHAR(255) NULL`
  - `verified BOOLEAN DEFAULT FALSE`
  - `trust_score DECIMAL(4,1) DEFAULT 100.0`
  - `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
- `food_categories` table:
  - `id INT AUTO_INCREMENT PRIMARY KEY`
  - `name VARCHAR(100) NOT NULL`
  - `donor_selectable BOOLEAN DEFAULT TRUE`
  - `default_expiry_minutes INT NOT NULL`
  - `radius_expand_minutes INT NOT NULL`
  - Seed initial vetted categories per PRD §7:
    - *Selectable*: "Cooked rice & grains" (expiry: 240 min, expand: 45 min), "Breads & baked items" (expiry: 360 min, expand: 60 min), "Sealed packaged goods" (expiry: 1440 min, expand: 120 min), "Cooked vegetarian curries / meals" (expiry: 180 min, expand: 45 min)
    - *Non-selectable*: "Raw meat / poultry" (donor_selectable: FALSE), "Dairy items requiring cold-chain" (donor_selectable: FALSE), "Cut fruits & fresh salads" (donor_selectable: FALSE)
- `listings` table:
  - `id INT AUTO_INCREMENT PRIMARY KEY`
  - `donor_id INT NOT NULL, FK → users(id)`
  - `category_id INT NOT NULL, FK → food_categories(id)`
  - `portions_posted INT NOT NULL`
  - `portions_verified INT NULL`
  - `photo_path VARCHAR(255) NULL`
  - `cooked_at TIMESTAMP NOT NULL`
  - `expiry_at TIMESTAMP NOT NULL`
  - `lat DECIMAL(10,7) NOT NULL`
  - `lng DECIMAL(10,7) NOT NULL`
  - `status ENUM('AVAILABLE','RESERVED','IN_TRANSIT','COMPLETED','EXPIRED_SPOILED','RECYCLE_CLAIMED','RECYCLED') DEFAULT 'AVAILABLE'`
  - `radius_km DECIMAL(3,1) DEFAULT 5.0`
  - `otp CHAR(4) NOT NULL`
  - `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
- `claims` table:
  - `id INT AUTO_INCREMENT PRIMARY KEY`
  - `listing_id INT NOT NULL, FK → listings(id)`
  - `claimant_id INT NOT NULL, FK → users(id)`
  - `claimant_role ENUM('ngo','recycler') NOT NULL`
  - `claimed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
  - `otp_verified_at TIMESTAMP NULL`
  - `checklist_odor_ok BOOLEAN NULL`
  - `checklist_visual_ok BOOLEAN NULL`
  - `checklist_storage_ok BOOLEAN NULL`
  - `verification_photo_path VARCHAR(255) NULL`
  - `completed_at TIMESTAMP NULL`
  - `bulk_weight_kg DECIMAL(6,2) NULL`
  - `no_show_reported BOOLEAN DEFAULT FALSE`

---

### 3. Backend Core & Configuration

#### [NEW] [db.php](file:///c:/Users/Harshil/Downloads/Annapoorna/config/db.php)
- Returns a singleton or memoized `PDO` instance.
- Reads DB parameters from `$_ENV` or defaults:
  `DB_HOST=127.0.0.1`, `DB_PORT=3306`, `DB_NAME=Annapoorna`, `DB_USER=root`, `DB_PASS=""`.
- Configures options:
  - `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION`
  - `PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC`
  - `PDO::ATTR_EMULATE_PREPARES => false`
- Includes a connection check helper function and JSON error reporting helper `sendJsonResponse($success, $data, $error, $statusCode)`.

#### [NEW] [auth-guard.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/includes/auth-guard.php)
- Starts session safely (`session_status() === PHP_SESSION_NONE`).
- Provides `requireAuth($allowedRoles = [])`:
  - If unauthenticated: redirects to `/auth.php` for browser requests, or emits HTTP 401 `{ "success": false, "error": "unauthorized" }` for API requests.
  - If role does not match `$allowedRoles`: redirects or emits HTTP 403 `{ "success": false, "error": "forbidden" }`.
- Provides `getCurrentUser($pdo)` to retrieve fresh user state (including `verified`, `trust_score`).

---

### 4. Authentication Endpoints

#### [NEW] [register.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/api/register.php)
- Validates inputs: `role` in `('donor','ngo','recycler')`, `org_name`, `email`, `password` (min length 8), `phone`, optional `lat`, `lng`.
- Handles optional verification document file upload (`$_FILES['verification_doc']` or multipart): verifies file extension/MIME (pdf, png, jpg), generates random file name, moves to `public/uploads/docs/`.
- Computes `password_hash($password, PASSWORD_DEFAULT)`.
- If `role` is `ngo` or `recycler`, sets `verified = 0`.
- Inserts via prepared statement. Handles unique constraint violations (`email_already_exists`).
- Returns HTTP 201 `{ "success": true, "data": { "user_id": ..., "role": ..., "verified": ... }, "error": null }`.

#### [NEW] [login.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/api/login.php)
- Body: `{ email, password }` parsed from `php://input`.
- Prepared statement selects user by email.
- Uses `password_verify($password, $user['password_hash'])`.
- If valid, sets `$_SESSION['user_id'] = $user['id']` and `$_SESSION['role'] = $user['role']`.
- Removes `password_hash` from user array and returns HTTP 200 `{ "success": true, "data": { "user": ... }, "error": null }`.
- If invalid, returns HTTP 401 `{ "success": false, "data": null, "error": "invalid_credentials" }`.

#### [NEW] [logout.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/api/logout.php)
- Unsets `$_SESSION`, destroys session, and removes session cookie.
- Returns HTTP 200 `{ "success": true, "data": null, "error": null }`.

---

### 5. UI & Styling Foundation

#### [NEW] [style.css](file:///c:/Users/Harshil/Downloads/Annapoorna/public/assets/css/style.css)
- Mobile-first CSS architecture with CSS custom properties:
  - Theme colors: Emerald/Sage greens (food rescue, fresh), warm Amber (urgency/warnings), deep slate charcoal text, clean card backgrounds.
  - Elevation shadows, responsive container widths, modern typography, glassmorphic accents.
  - Reusable components: forms, buttons, alert banners, role badges, tabs.

#### [NEW] [header.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/includes/header.php)
- Shared HTML header, meta viewport, navigation bar.
- Shows brand logo "Annapoorna", links to public dashboard, and context-sensitive user status (Role badge, Org name, Logout button when logged in; Login/Register links when guest).

#### [NEW] [footer.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/includes/footer.php)
- Shared footer citing EPA WARM model metrics formula (PRD §9) and role links.

#### [NEW] [auth.php](file:///c:/Users/Harshil/Downloads/Annapoorna/public/auth.php)
- Dual-mode card UI: Toggle between **Login** and **Register**.
- Dynamic registration form adjusting fields based on selected role:
  - Donor: Org name, email, password, phone, location (with "Use current location" button via `navigator.geolocation` or lat/lng fields).
  - NGO / Recycler: Includes document upload input for organization verification. Explains that approval is pending for claims.
- Client-side validation + asynchronous `fetch()` API calls to `/api/login.php` and `/api/register.php`.
- Handles machine-readable API error codes (`invalid_credentials`, `email_already_exists`, `unauthorized`) with clear user-facing feedback.
- On login success, seamlessly redirects to the appropriate role screen (`/donor/dashboard.php`, `/ngo/feed.php`, or `/recycling/index.php`).

---

## Verification Plan

### Phase 0 Demo Checkpoint Testing
1. **Database Schema Execution**:
   - Run `schema.sql` on MySQL and confirm all 4 tables (`users`, `food_categories`, `listings`, `claims`) and category seed rows are created with matching types and keys.
2. **Database Connectivity**:
   - Execute a test script to verify `config/db.php` connects via PDO with prepared statements.
3. **Registration Flow**:
   - Register a `donor` user via `/api/register.php`. Verify row in `users` with `verified=1` (or 0 for ngo/recycler) and hashed password.
   - Register an `ngo` user with verification doc. Verify document saved in `uploads/docs/` with random name.
   - Register a `recycler` user.
4. **Login Flow**:
   - Test login with valid donor credentials -> expect HTTP 200, session set, role `donor`.
   - Test login with incorrect password -> expect HTTP 401 `invalid_credentials`.
5. **Session & Auth Guard**:
   - Verify protected endpoints return HTTP 401 when not logged in.
   - Verify role-mismatch triggers HTTP 403 or redirect.
6. **Logout Flow**:
   - Call `/api/logout.php` -> verify session destroyed, subsequent protected access denied.
7. **Auth UI Interactive Check**:
   - Load `/auth.php` in browser (at 375px mobile width and desktop), test tab toggles, role selector changes, registration, and login.
