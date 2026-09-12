# Product Requirements Document — Annapoorna

## 1. Problem Statement
Surplus edible food from hotels, restaurants, and mess halls is routinely discarded instead of reaching people who need it, because there's no fast, trusted, hyperlocal way to connect a donor with a nearby NGO inside the narrow window before food is no longer safe to eat. Separately, food that genuinely can't be donated (spoiled, past its window) still goes to landfill instead of composting/biogas, because there's no coordination layer for that either.

## 2. Goals
- Let a donor post surplus food in under a minute and have it visible to nearby NGOs immediately.
- Match donors to NGOs within a geofenced radius, prioritizing urgency (shortest time-to-expiry first).
- Provide a verifiable custody chain (OTP handoff + photo + checklist) so both sides can trust the transaction happened safely.
- Automatically reroute anything that goes unclaimed past its safe window to a recycling stream, so nothing is a dead end.
- Show the public a live, credible impact dashboard (meals saved, kg diverted, CO2e offset) to build trust and drive adoption.

## 3. Roles & Core Needs

| Role | Primary need | Key screens |
|---|---|---|
| **Donor** (hotel, hostel, banquet hall, restaurant) | Post surplus fast; know when/who is picking up; get credit for donations | Donor Dashboard, Post Surplus Form, Listing Status |
| **NGO / Shelter** | Find nearby, urgent listings; verify safety before accepting; confirm delivery | NGO Feed, Claim & Navigation, Verification & Dispatch |
| **Recycler** (biogas/composting) | Find expired/inedible listings; log bulk intake | Bio-Waste Exchange, Recycling Log |
| **Public** | See that this is real and working | Landing / Impact Dashboard |

## 4. Core Data Model: Listing Lifecycle

**Edible listing:**
```
AVAILABLE ──(NGO claims)──► RESERVED ──(OTP verified)──► IN_TRANSIT ──(drop-off confirmed)──► COMPLETED
    │
    └──(expiry_window elapsed, unclaimed)──► EXPIRED_SPOILED
```

**Recycling listing (continues from EXPIRED_SPOILED, or posted directly as inedible):**
```
EXPIRED_SPOILED ──(recycler claims)──► RECYCLE_CLAIMED ──(intake logged)──► RECYCLED
```

State transitions are the backbone of the app — see `database-schema.md` for the `status` enum and `api-endpoints.md` for which endpoint is allowed to move a listing between which states. No endpoint should be able to skip a state.

## 5. Functional Requirements by Role

### 5.1 Donor
- FR1: Register with role `donor`, org name, and location (lat/lng, captured via browser geolocation or manual pin).
- FR2: Post a listing: food category (from a **closed, safety-vetted list** — see PRD §7), portion count, prep/cooked timestamp, optional photo.
- FR3: `expiry_window` is **auto-calculated server-side** from the selected category — the donor cannot manually extend it.
- FR4: On listing creation, system generates a 4-digit OTP shown only to the donor.
- FR5: View live status of an active listing: which NGO claimed it (if any), distance, ETA if available, countdown to expiry.
- FR6: View donation history and a running CSR score based on **verified** (not posted) portions delivered.

### 5.2 NGO
- FR7: Register with role `ngo`; must upload a registration/verification document; account is `unverified` until admin approves.
- FR8: Browse a feed of `AVAILABLE` listings within their radius, sorted by urgency (time-to-expiry ascending). Default radius 5 km, auto-expands to 10 km if a listing is unclaimed past 45 minutes (configurable per category — see `database-schema.md`).
- FR9: Claim a listing — this must be an atomic operation (first claim wins; see PRD §8 concurrency requirement).
- FR10: On arrival, complete a mandatory checklist (odor, visual integrity, storage condition) with a required photo, enter the donor's OTP, and confirm **actual received portion count** (may differ from posted count — both numbers are stored).
- FR11: Confirm final drop-off at the beneficiary shelter to move the listing to `COMPLETED`.
- FR12: Report a donor no-show after a grace period, releasing the listing back to `AVAILABLE`.

### 5.3 Recycler
- FR13: Register with role `recycler` (biogas plant, composting hub).
- FR14: Browse a feed of `EXPIRED_SPOILED` listings.
- FR15: Claim a listing, then log intake: bulk weight in kg, processing status.
- FR16: Confirming intake moves the listing to `RECYCLED` and contributes to the public "kg diverted" metric.

### 5.4 Public
- FR17: View live aggregate counters: total meals saved, kg organic waste diverted, kg CO2e offset (see PRD §9 for the formula and its source).
- FR18: View leaderboards: top donor orgs and top NGOs, ranked by **verified** activity.

## 6. Non-Functional Requirements
- **Mobile-first / responsive**: NGO and donor staff will use this on phones in real time — the feed, claim, and verification screens must work well on a small screen.
- **Low posting friction**: the Post Surplus form should be completable in under 60 seconds.
- **Auditability**: every state transition is timestamped and, where relevant, tied to a photo and/or OTP entry — this is the trust layer, not an afterthought.
- **Graceful concurrency**: two NGOs claiming the same listing simultaneously must never both succeed.

## 7. Food Safety Constraints (product-level, not just backend validation)
The category dropdown on the Post Surplus form only offers pre-approved, safe-to-donate categories (e.g. cooked grains, breads, dry/packaged sealed goods, vegetarian curries held above safe temperature). Raw meat, dairy-based items requiring cold-chain, and cut produce are **not selectable options** — this is an intentional product constraint, not just a validation rule, and should be reflected in the UI copy (explain briefly why).

## 8. Concurrency Requirement
Listing claims must use an atomic conditional update (`UPDATE listings SET status='RESERVED', ngo_id=? WHERE id=? AND status='AVAILABLE'`) and check the affected-row count. If zero rows affected, return a "already claimed" error rather than allowing a race.

## 9. Impact Metrics Formulas
- `meals_saved = SUM(verified_portions)` across `COMPLETED` listings — **verified, not posted**.
- `organic_waste_diverted_kg = SUM(portions) × 0.45` (kg per portion assumption).
- `co2e_offset_kg = organic_waste_diverted_kg × 0.45` (avoided-landfill factor, per EPA WARM — see note below).

> **Note carried over from pitch prep:** the original spec used a 2.5x CO2e multiplier; EPA's WARM model puts avoided-landfill emissions closer to a 1:1 ratio (~0.43–0.46 kg CO2e per kg diverted). This PRD uses 0.45 for both, and that source should be cited on the public dashboard for credibility.

## 10. Out of Scope (this build)
- Payments or monetization features
- Native mobile apps
- Automated (non-admin) NGO verification
- Multi-language support
