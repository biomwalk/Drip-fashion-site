# Drip — Session Handoff (April 29 2026, end of session 13)

## Stack
- **Backend**: Node.js + TypeScript, Express, PostgreSQL (Knex) — `cd drip-backend && npm run dev`
- **App**: Flutter (Provider) — `cd drip-app && flutter run`
- **Production**: Railway (API) + Supabase (PostgreSQL)
- **DB connection (prod only)**: `SUPABASE_URL` in `drip-backend/.env`
- **Redis**: Railway Redis — `REDIS_URL` set in Railway API service env vars (internal URL)

> All backend testing runs against Railway/Supabase only. Never test against local postgres.
> Supabase pooler can return stale reads — always verify schema changes via Supabase SQL editor directly.

## Test User
- Email: `test@test.com` / Password: `password123`
- userId: `407cd7d3-d3bc-43ed-bfff-df8e21d6e214`
- Username: `testuser`

```bash
TOKEN=$(curl -s -X POST https://drip-production-a900.up.railway.app/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"password123"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")
```

## Migration State
Latest sequential: `049_user_profiles_and_follows.ts`
Next: `050_...`

> **Convention note**: Always `cd drip-backend` before creating migrations. Use this to get the correct next number:
> ```bash
> cd drip-backend && python3 -c "
> import os, re
> files = sorted(f for f in os.listdir('src/db/migrations') if re.match(r'^\d{3}_', f) and f.endswith('.ts'))
> print('Next:', str(int(files[-1].split('_')[0]) + 1).zfill(3))
> "
> ```

---

## What Was Done This Session (Session 13)

### 1. User profile schema (migration 049)
- Added to `users` table: `is_public` (bool, default true), `show_wardrobe` (bool, default false), `show_watchlist` (bool, default true), `instagram_id` (text), `tiktok_id` (text)
- Created `user_follows` table: `follower_id`, `followee_id`, `created_at`, composite primary key
- Migration had Supabase pooler issues — columns confirmed via SQL editor, manually inserted into `knex_migrations`

### 2. User profile backend
- `user.service.ts` — `getProfile`, `updateProfile`, `followUser`, `unfollowUser`, `searchUsers`, `getFollowers`, `getFollowing`
- `user.controller.ts` — full CRUD, `String()` cast on all `req.params.id`
- `user.routes.ts` — mounted at `/api/v1/users`
- Routes: `GET /users/me`, `GET /users/:id`, `PATCH /users/me`, `PATCH /users/me/handle`, `POST /users/:id/follow`, `DELETE /users/:id/follow`, `GET /users/:id/followers`, `GET /users/:id/following`, `GET /users/search?q=`
- Privacy gating: non-owners blocked from private profiles. `followed_brands` always visible. `show_watchlist` and `show_wardrobe` respect user settings.

### 3. User profile Flutter
- `lib/models/user_profile.dart` — `UserProfile` + `UserProfileBrand` models
- `lib/services/user_service.dart` — wraps all `/users` endpoints
- `lib/screens/user_profile_screen.dart` — single screen, `isOwner` computed from `AuthProvider.user.id`
  - Own profile: avatar, bio (editable), follower/following counts (tappable → bottom sheet), followed brands grid (tappable → `BrandScreen.fromId`), privacy toggles
  - Other profile: follow/unfollow with optimistic update + rollback, private profile gate
  - Settings gear → pushes existing `ProfileScreen` (account settings)
  - Followers/following → `_FollowListSheet` bottom sheet, tapping navigates to `UserProfileScreen`
- `main.dart` — drawer "My Profile" → `UserProfileScreen(userId: authUser.id)`

### 4. BrandScreen.fromId
- Added `BrandScreen.fromId({required String brandId})` named constructor
- `_BrandScreenState` now has `Brand? _brand` field — populated from `widget.brand` or fetched via `_fetchBrandById()`
- All `widget.brand.` references replaced with `_brand?.` / `_brand!.`
- `user_profile_screen.dart` brand grid taps now navigate to `BrandScreen.fromId(brandId: brand.id)`

### 5. Admin dashboard — confidence tier system
New thresholds in `src/config/index.ts`:
- `FASHION_HARD_BLOCK_THRESHOLD: 10` — hard discard floor, no DB write
- `FASHION_LOW_CONFIDENCE_THRESHOLD: 70` — 10–69% stored as `low_confidence`
- `FASHION_AUTO_APPROVE_THRESHOLD: 85` — 70–84% → `manual_review`, 85%+ → auto-scrape

New service functions: `adminGetLowConfidenceSubmissions()`, `adminBlockBrand(brandId)`, `adminSubmitBrandUrl(url)`
New routes: `GET /brands/admin/low-confidence`, `POST /brands/admin/block/:id`, `POST /brands/admin/submit`

`adminSubmitBrandUrl` runs the full submission pipeline attributed to ops user ID `407cd7d3-d3bc-43ed-bfff-df8e21d6e214`.

Blocked brands: `known_brands.response_category === 'blocked'` returns `not_fashion` immediately on resubmission.

### 6. Admin dashboard — full UI redesign
- **Tab bar**: Review Queue / Submissions / Taxonomy Health
- **Review Queue tab**: Pending brands + Language queue, each in 520px scrollable container
- **Submissions tab**: Low Confidence (sortable ↑↓ confidence, Approve + Block buttons) + Unknown Platform (alpha sort, live search, sortable submissions count)
- **Taxonomy Health tab**: skipped products analysis in scrollable containers
- **URL submit bar**: full-width input at top of dashboard above "BRAND REVIEW" header, hidden until logged in, Enter key supported, inline status feedback with 4s auto-clear
- **localStorage persistence**: token saved on login, restored on page load, Sign Out clears it
- **Score colour coding**: ≥80% green, ≥60% amber, <60% red
- **Logo**: links to homepage
- **DRIP logo → `/`**, admin badge removed from nav

### 7. Homepage updates (index.html)
- Hero title changed from "YOUR STORE" to "YOUR FASHION STORE" (3 lines)
- Nav: Privacy link removed, replaced with Support (mailto) + Admin link
- Admin link in nav is subtle (low opacity) — leads to `admin.html`

---

## Architecture

### Backend (`drip-backend/src/`)
Routes: `auth`, `brand`, `product`, `storefront`, `preferences`, `watchlist`, `address`, `wardrobe`, `notification`, `cart`, `recently-viewed`, `user`

**Key services:**
- `brand.service.ts` — submission pipeline, confidence tiers, known_brands gate (blocked/hype/known), admin functions, enqueues scrape jobs
- `shopify.service.ts` — detect (browser UA, Array.isArray fix), validate, fetch, sale collection check
- `user.service.ts` — profile CRUD, follow graph, search, privacy gating
- `cart.service.ts`, `recently_viewed.service.ts`, `wardrobe.service.ts`, `watchlist.service.ts`
- `queue/scrape.queue.ts` + `queue/scrape.worker.ts` — Bull queue, concurrency 3

### Scrape queue statuses
- `pending` — queued for scraping (85%+ confidence)
- `manual_review` — awaiting ops approval (70–84% confidence or non-English)
- `low_confidence` — stored for investigation (10–69% confidence)
- `processing` — currently being scraped
- `completed` — scrape successful
- `failed` — scrape failed or blocked by ops
- `brand_id IS NULL` + `manual_review` — unknown platform submission

### Known brands — response categories
- `major_retailer`, `luxury`, `marketplace`, `fast_fashion` — return `known_brand`, no scrape
- `hype_brand` — attempt Shopify detection first, fall back if blocked
- `blocked` — return `not_fashion` immediately, never scrape

### Flutter (`drip-app/lib/`)
Providers: `AuthProvider`, `StorefrontProvider`, `PreferencesProvider`, `WatchlistProvider`, `WardrobeProvider`, `CartProvider`, `RecentlyViewedProvider`, `CurrencyService`, `NotificationsProvider`

Services: `ApiService`, `StorefrontService`, `UserService`

**Submit bar states (discover_screen.dart):**
1. `_showSubmitted` — green bar with `_submitSuccessMessage`
2. `_knownBrandName != null` — blue info panel (covers `known_brand` + `hype_brand`)
3. `_notFashionBrandId != null` — pink panel, dispute flow
4. `_notEnglishLang != null` — amber panel
5. `_submitError.isNotEmpty` — red error
6. Default — URL input + submit button

**Login flow (splash_screen.dart):**
1. `silentLogin()` restores session
2. `PreferencesProvider.load()`, `RecentlyViewedProvider.loadForUser(userId)`, `WatchlistProvider.load()`, `CartProvider.loadForUser(userId)` — parallel via `Future.wait<void>`
3. `CurrencyService.init(currency)` then `CurrencyService.loadForUser()`

---

## Notification Types (DB constraint)
`brand_live`, `brand_submission`, `new_products`, `price_drop`, `back_in_stock`, `friend_joined`, `friend_likes`, `follow`, `brand_suggestion`, `exclusive_drop`

Stubbed: `friend_joined`, `friend_likes`, `follow`, `brand_suggestion`, `exclusive_drop`

---

## DB Constraints to Keep in Sync
- `products_gender_check` — must match `ProductGender` in `taxonomy.service.ts`
- `notifications_type_check` — must include all types used in `notify()` calls

---

## Known Issues / TODO

### Immediate:
- [ ] User search UI — `UserProfileScreen` exists but no in-app entry point to find other users yet
- [ ] `adminApproveBrand` doesn't update `low_confidence` queue status to `completed` — approve works on brand row but queue entry stays `low_confidence`. Fix: update queue status in `adminApproveBrand`
- [ ] Test sabahar.com end-to-end with category-scoped admin approval
- [ ] Fix Android AAB build — install Android SDK Command-line Tools, then `flutter doctor --android-licenses`
- [ ] App Store screenshots — on hold until UI finalized

### Instagram / TikTok onboarding graph (Phase 2 — high priority):
- Brand follower backcompute: brand accounts are public, match user platform ID against brand follower lists
- `instagram_id` / `tiktok_id` on `users` table (migration 049)
- Still needed: `instagram_handle` + `tiktok_handle` columns on `brands` table
- Instagram data export requested this session — parse script needed when it arrives

### Android launch prep (Phase 3):
- [ ] Google Play Developer account
- [ ] Google Play Internal Testing
- [ ] `flutter build appbundle --release`
- [ ] End-to-end Android Google Sign-In test

### Stripe / full checkout (Phase 3):
- [ ] Stripe Connect — replace Skimlinks WebView
- [ ] Cart model already correct — only checkout action changes

### Social layer (Phase 2):
- [ ] User search UI in app
- [ ] `convertLikedToWardrobe` — stub in `brand.service.ts`
- [ ] `friend_joined`, `friend_likes`, `follow` notification triggers

### Known limitations:
- [ ] Per-product vibe tagging — brand-level only
- [ ] `CartDrawer` on both `MainShell` and `ProductScreen` — MVP workaround
- [ ] `search_vector` uses 'english' dictionary — non-English brands rank poorly
- [ ] Submission limit: 20/day → 10 (scale) → 5 (long-term)
- [ ] Bull worker runs in same process as API — separate post-launch
- [ ] Hype brands on WooCommerce won't hit attempt-then-fallback

---

## Production Setup
- **API**: Railway (auto-deploys from GitHub main)
- **DB**: Supabase (PostgreSQL)
- **Redis**: Railway Redis (internal)
- **Env vars on Railway**: `JWT_SECRET`, `REFRESH_TOKEN_SECRET`, `SUPABASE_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `ADMIN_TOKEN`, `REDIS_URL`

## Admin Dashboard
- URL: `dripp.fashion/admin.html`
- Token stored in `localStorage` — persists across sessions
- URL submit bar at top of dashboard — runs full submission pipeline via ops user ID
- Three tabs: Review Queue, Submissions (Low Confidence + Unknown Platform), Taxonomy Health

## TestFlight
```bash
cd ~/drip/drip-app && flutter clean && flutter build ipa --release
```
Export compliance: **No**. Current build: 5.

## Android
```bash
cd ~/drip/drip-app && flutter build apk --release        # testing ✓
cd ~/drip/drip-app && flutter build appbundle --release  # Play Store — needs cmdline-tools fix
```
Keystore: `android/app/drip-release.keystore` (gitignored)
Key properties: `android/app/key.properties` (gitignored)

## Git Workflow
```bash
cd ~/drip && git add -A && git commit -m "description" && git push origin main
cd ~/drip-fashion-site && git add -A && git commit -m "description" && git push origin main
```

## Key Identifiers
- Bundle ID (iOS): `com.drip.dripApp`
- Application ID (Android): `com.drip.drip_app`
- Railway API URL: `https://drip-production-a900.up.railway.app/api/v1`
- Skimlinks Publisher ID: `302040X1790029`
- Support email: `biomwalk@gmail.com`
- Domain: `dripp.fashion`
- GitHub (app): github.com/biomwalk/Drip
- GitHub (site): github.com/biomwalk/drip-fashion-site
- Admin dashboard: `dripp.fashion/admin.html`
- Android OAuth Client ID: `151434073874-kl73mdub14h9p83oo53vpj9v71fdiecs.apps.googleusercontent.com`
- Android SHA-1: `67:7E:0A:9C:F7:D3:33:CA:5B:8A:07:A9:72:D7:8D:89:73:59:49:4C`
- Ops user ID (admin submissions): `407cd7d3-d3bc-43ed-bfff-df8e21d6e214`

## Scripts
```bash
cd drip-backend && npx ts-node scripts/backfill_style_tags.ts
```
