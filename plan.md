# ArtHub Digital Marketplace — Atomic Step-by-Step Implementation Plan (35 Steps)

> **Specification Reference:** SRS v2.4.0 (`ArtHub_Technical_Specifications_Requirements.pdf`)  
> **Execution Rule:** Each step below is strictly atomic and independent. When commanded *"Do step no X"*, execute **ONLY** that specific step — nothing more, nothing less. Verify completion against the step's *Done Criteria* and stop.

---

## 🧭 Execution Protocol

1. **Strict Scope Isolation:** Do not implement features, files, or configurations that belong to later steps.
2. **Deterministic Outputs:** Each step creates or edits specific target files with exact parameters.
3. **Verification Before Progression:** Never mark a step as finished until all acceptance items in its *Done Criteria* pass.
4. **Command Syntax:** The user triggers execution by saying:
   - `Do step 1` (or `step no 1`)
   - `Do step 2`
   - ...and so on.

---

## 📋 Comprehensive Step Registry

### Phase A: Backend Foundation & Database Architecture

#### Step 1: Initialize Backend Directory Skeleton & Composer Configuration
- **Objective:** Create the standalone `backend/` folder hierarchy and configure `composer.json` for all PHP dependencies.
- **Files to Create:**
  - `backend/composer.json`
  - `backend/public/index.php`
  - `backend/public/.htaccess`
  - Directory placeholders: `backend/app/Controllers/`, `backend/app/Models/`, `backend/app/Services/`, `backend/app/Middleware/`, `backend/app/Helpers/`, `backend/routes/`, `backend/database/`, `backend/storage/logs/`, `backend/storage/invoices/`.
- **Scope Details:**
  - Define dependencies in `composer.json`: `firebase/php-jwt`, `stripe/stripe-php`, `vlucas/phpdotenv`.
  - Set up PSR-4 autoloading: `"App\\": "app/"`.
  - Configure `public/.htaccess` with mod_rewrite to route all requests to `index.php`.
  - Minimal entry-point echo in `public/index.php`.
- **Strict Out of Scope:** Database connection, controllers, routing logic, frontend code.
- **Done Criteria:** Folder structure exists; `composer.json` and `.htaccess` are valid and syntactically correct.

---

#### Step 2: Environment Configuration & Secret Management
- **Objective:** Create environment template files and local configuration with strict client/server secret segregation.
- **Files to Create / Modify:**
  - `backend/.env.example`
  - `backend/.env`
  - `frontend/.env.local`
  - `.gitignore` (ensure `.env` and sensitive files are ignored)
- **Scope Details:**
  - In `backend/.env`: `DB_HOST`, `DB_NAME=arthub_db`, `DB_USER`, `DB_PASS`, `JWT_SECRET_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `IMGBB_API_KEY`, `CORS_ALLOWED_ORIGINS=http://localhost:3000`.
  - In `frontend/.env.local`: `NEXT_PUBLIC_API_BASE_URL=http://localhost:8000/api/v1`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `NEXT_PUBLIC_GOOGLE_CLIENT_ID`.
- **Strict Out of Scope:** Writing application logic, executing database migrations.
- **Done Criteria:** Both `.env` files and `.env.example` are populated with required keys and documented placeholder values.

---

#### Step 3: Relational Database Schema DDL (`schema.sql`)
- **Objective:** Build the complete 3NF normalized MySQL 8.0 schema containing all 9 core tables with strict foreign key constraints and indexes.
- **Files to Create:**
  - `backend/database/schema.sql`
- **Scope Details:**
  - Tables: `users`, `artist_profiles`, `categories`, `artworks`, `orders`, `order_items`, `reviews`, `wishlists`, `webhook_logs`.
  - Proper data types: `BIGINT UNSIGNED` primary keys, `DECIMAL(10,2)` for currency, `JSON` for `shipping_address`, `ENUM` for statuses.
  - Foreign key actions: `ON DELETE CASCADE` or `RESTRICT` as specified in SRS Section 3.
  - Indexes on slug, email, category_id, artist_id, status.
- **Strict Out of Scope:** Populating sample data, PHP PDO code.
- **Done Criteria:** `backend/database/schema.sql` is fully written and syntactically valid SQL for MySQL 8.0.

---

#### Step 4: Database Seeders & Mock Catalog Data (`seeders.sql`)
- **Objective:** Write database seed data to populate initial artwork categories, test accounts (admin, artist, buyer), and sample artworks.
- **Files to Create:**
  - `backend/database/seeders.sql`
- **Scope Details:**
  - 6 Default categories: Oil Painting, Acrylic, Watercolor, Digital Art, Sculpture, Photography.
  - 3 Test users with pre-hashed Bcrypt passwords (`$2y$12$...`): `admin@arthub.com`, `artist@arthub.com`, `buyer@arthub.com`.
  - Initial artist profile for the test artist.
  - At least 6 mock artworks with realistic titles, descriptions, dimensions, prices, and CDN image URLs.
- **Strict Out of Scope:** Executing seeds against live MySQL, creating API endpoints.
- **Done Criteria:** `backend/database/seeders.sql` contains consistent foreign keys linking correctly to `schema.sql`.

---

#### Step 5: Database Connection Singleton & Transaction Utility
- **Objective:** Implement the thread-safe PDO database connection pool and ACID transaction helper.
- **Files to Create:**
  - `backend/app/Services/Database.php`
- **Scope Details:**
  - Singleton pattern returning `PDO` instance.
  - Configuration via environment variables: `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS`.
  - Flags: `ATTR_ERRMODE => ERRMODE_EXCEPTION`, `ATTR_DEFAULT_FETCH_MODE => FETCH_ASSOC`, `ATTR_EMULATE_PREPARES => false`.
  - UTF-8 charset (`utf8mb4`).
- **Strict Out of Scope:** Writing model business logic or API controllers.
- **Done Criteria:** `Database::getInstance()` returns configured PDO instance without connection leaks.

---

#### Step 6: Core REST Router, Response Helper & CORS Middleware
- **Objective:** Build the foundational HTTP pipeline including parameter-aware routing, JSON response standardization, and CORS handling.
- **Files to Create:**
  - `backend/app/Core/Router.php`
  - `backend/app/Helpers/Response.php`
  - `backend/app/Middleware/CorsMiddleware.php`
  - Update `backend/public/index.php` to dispatch routes through `Router`.
- **Scope Details:**
  - Router supporting `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS` with parameterized URL patterns (e.g., `/artworks/{slug}`).
  - Standard JSON response format `{ "status": "success"|"error", "data": ..., "message": ... }`.
  - CORS middleware supporting origins defined in `CORS_ALLOWED_ORIGINS`, handling pre-flight `OPTIONS` requests, credentials, and headers (`Content-Type`, `Authorization`).
- **Strict Out of Scope:** Authentication verification, controller endpoints.
- **Done Criteria:** A test route registered in `public/index.php` returns expected JSON and CORS headers.

---

### Phase B: Backend Authentication & Security

#### Step 7: JWT Service & Password Hashing Engine
- **Objective:** Create stateless JWT token generation/validation utilities and Bcrypt password hashing methods.
- **Files to Create:**
  - `backend/app/Services/AuthService.php`
- **Scope Details:**
  - Bcrypt hashing with cost=12 (`password_hash`, `password_verify`).
  - Access token creation: 15-minute expiration, HMAC-SHA256 signature containing `sub` (user_id), `email`, and `role`.
  - Refresh token creation: 7-day validity.
  - Token decoding and signature verification methods using `JWT_SECRET_KEY`.
- **Strict Out of Scope:** HTTP cookie injection, database operations.
- **Done Criteria:** `AuthService` passes self-contained test generating and verifying valid JWT tokens and rejecting expired/tampered tokens.

---

#### Step 8: Authentication & Role-Based Access Control (RBAC) Middleware
- **Objective:** Create request guard middleware that extracts Bearer JWT tokens, validates claims, and enforces role boundaries.
- **Files to Create:**
  - `backend/app/Middleware/AuthMiddleware.php`
- **Scope Details:**
  - Extract `Authorization: Bearer <token>` from HTTP headers.
  - Return `401 Unauthorized` for missing, expired, or invalid tokens.
  - Attach authenticated user payload (`id`, `role`, `email`) to request context.
  - Role guard method `hasRole(array $allowedRoles)` returning `403 Forbidden` if user role does not match.
- **Strict Out of Scope:** User registration/login database logic.
- **Done Criteria:** Middleware cleanly interrupts unauthenticated or unauthorized requests with standardized JSON errors.

---

#### Step 9: Authentication Controller & Endpoints
- **Objective:** Implement full user registration, login, refresh, and profile endpoints.
- **Files to Create:**
  - `backend/app/Controllers/AuthController.php`
  - `backend/app/Models/User.php`
  - Register auth routes in `backend/routes/api.php`:
    - `POST /api/v1/auth/register`
    - `POST /api/v1/auth/login`
    - `POST /api/v1/auth/refresh`
    - `GET /api/v1/auth/me`
- **Scope Details:**
  - Input validation (valid email, min 8 char password, valid role: `buyer` or `artist`).
  - Auto-create `artist_profiles` entry if registered role is `artist`.
  - Set `HttpOnly`, `SameSite=Strict`, `Secure` refresh cookie on login.
- **Strict Out of Scope:** Google OAuth 2.0 third-party flow (handled in future enhancement), catalog endpoints.
- **Done Criteria:** Can register a new buyer/artist, log in to receive access JWT + cookie, and fetch authenticated `/api/v1/auth/me`.

---

### Phase C: Backend Media Storage & Artwork Catalog

#### Step 10: ImgBB Cloud Media Service Integration
- **Objective:** Implement cURL-based image upload and deletion handler using the ImgBB REST API v1.
- **Files to Create:**
  - `backend/app/Services/ImgBBService.php`
- **Scope Details:**
  - Accept uploaded image file stream or base64 data.
  - Validate file mime-type (`image/jpeg`, `image/png`, `image/webp`) and size (<= 16MB).
  - Execute multipart/form-data cURL POST to `https://api.imgbb.com/1/upload?key={API_KEY}`.
  - Return array containing: `image_url`, `thumbnail_url`, `delete_hash`.
  - Implement delete method via `delete_hash`.
- **Strict Out of Scope:** Database persistence, artwork listing controllers.
- **Done Criteria:** `ImgBBService` returns structured image URLs and deletion hashes upon file upload.

---

#### Step 11: Categories & Catalog Taxonomies Controller
- **Objective:** Implement public endpoints to list active categories with associated artwork counters.
- **Files to Create:**
  - `backend/app/Controllers/CategoryController.php`
  - `backend/app/Models/Category.php`
  - Register route: `GET /api/v1/categories`
- **Scope Details:**
  - Query categories table joined with count of active artworks: `SELECT c.*, COUNT(a.id) as artwork_count ... GROUP BY c.id`.
  - Return formatted JSON list with cache headers.
- **Strict Out of Scope:** Modifying categories (admin only), artwork CRUD.
- **Done Criteria:** `GET /api/v1/categories` returns array of categories with valid artwork counts.

---

#### Step 12: Public Artwork Catalog & Multi-Faceted Filter API
- **Objective:** Implement public artwork discovery endpoint supporting pagination, search, price ranges, and multi-criteria sorting.
- **Files to Create:**
  - `backend/app/Controllers/ArtworkController.php` (Read actions)
  - `backend/app/Models/Artwork.php`
  - Register routes:
    - `GET /api/v1/artworks`
    - `GET /api/v1/artworks/{slug}`
- **Scope Details:**
  - Query parameters: `page`, `limit`, `category`, `medium`, `min_price`, `max_price`, `in_stock`, `q` (search), `sort` (`newest`, `price_asc`, `price_desc`, `popular`).
  - Calculate pagination meta (`total`, `page`, `total_pages`).
  - Detailed slug view includes artist summary (`artist_profiles`) and verified reviews.
  - 100% prepared PDO statements with bound parameters.
- **Strict Out of Scope:** Artwork creation, editing, deletion (Artist/Admin write operations).
- **Done Criteria:** `GET /api/v1/artworks` correctly filters results and `GET /api/v1/artworks/{slug}` returns full detail payload.

---

#### Step 13: Artwork Creation, Update & Deletion API (Artist Protected)
- **Objective:** Implement protected CRUD endpoints for artists to create listings, update pricing/stock, and delete artworks.
- **Files to Modify:**
  - `backend/app/Controllers/ArtworkController.php` (Write actions)
  - Register routes in `backend/routes/api.php`:
    - `POST /api/v1/artworks` (Requires `artist` role)
    - `PUT /api/v1/artworks/{id}` (Requires ownership check)
    - `DELETE /api/v1/artworks/{id}` (Requires ownership check)
- **Scope Details:**
  - Handle multipart image upload via `ImgBBService` in `POST /artworks`.
  - Auto-generate SEO-friendly unique URL slug from title.
  - Strict ownership verification: logged-in artist ID must match artwork's `artist_id` (or user is `admin`).
  - On delete, invoke `ImgBBService::delete()` if `delete_hash` exists and mark record deleted/archived.
- **Strict Out of Scope:** Stripe payments, order processing.
- **Done Criteria:** Artist can create an artwork with image upload, update its price/status, and delete it; non-owners receive `403 Forbidden`.

---

### Phase D: Backend Payments, Orders & Reviews

#### Step 14: Stripe PaymentIntent & Order Intent Service
- **Objective:** Implement checkout initialization endpoint that validates inventory, computes server-side totals, and creates Stripe PaymentIntent.
- **Files to Create:**
  - `backend/app/Services/StripeService.php`
  - `backend/app/Controllers/OrderController.php` (Intent creation)
  - Register route: `POST /api/v1/checkout/create-intent` (Requires authenticated buyer)
- **Scope Details:**
  - Payload: items array `[{ "id": 1, "qty": 1 }]`, shipping address.
  - Validate each artwork exists, is active, and has sufficient `stock_quantity`.
  - Compute total server-side price in USD cents.
  - Call Stripe API: `\Stripe\PaymentIntent::create()` with amount, currency `usd`, and metadata containing `order_number` and `buyer_id`.
  - Return `client_secret` to client.
- **Strict Out of Scope:** Webhook processing, credit card details handling.
- **Done Criteria:** Endpoint returns valid Stripe `client_secret` and rejects out-of-stock items.

---

#### Step 15: Stripe Webhook Listener & Transaction Processor
- **Objective:** Implement secure webhook endpoint to receive Stripe events, verify cryptographic signatures, and mutate database states in an ACID transaction.
- **Files to Create:**
  - `backend/app/Controllers/WebhookController.php`
  - Register route: `POST /api/v1/webhooks/stripe` (Public endpoint, signature verified)
- **Scope Details:**
  - Read raw payload and verify signature header via `\Stripe\Webhook::constructEvent()` using `STRIPE_WEBHOOK_SECRET`.
  - Check `webhook_logs` table for `event_id` to prevent duplicate processing (idempotency).
  - Handle `payment_intent.succeeded`:
    - Open PDO transaction (`beginTransaction`).
    - Insert into `orders` table (`payment_status = 'succeeded'`).
    - Insert into `order_items` table.
    - Decrement `stock_quantity` on purchased artworks (set `status = 'sold'` if 0).
    - Insert record into `webhook_logs`.
    - `commit()`.
- **Strict Out of Scope:** Customer invoice PDF rendering.
- **Done Criteria:** Verified webhook correctly transitions pending intent to completed order in database; invalid signatures return HTTP 400.

---

#### Step 16: Buyer Order History & Artist Fulfillment API
- **Objective:** Implement endpoints for buyers to view their orders and for artists to view and advance fulfillment states.
- **Files to Modify / Create:**
  - `backend/app/Controllers/OrderController.php` (Buyer & Artist actions)
  - Register routes in `backend/routes/api.php`:
    - `GET /api/v1/buyer/orders` (Buyer authenticated)
    - `GET /api/v1/artist/orders` (Artist authenticated)
    - `PUT /api/v1/artist/orders/{id}` (Update status: `packaging`, `shipped`, `delivered`, inject tracking number)
- **Scope Details:**
  - Buyer endpoint returns list of purchases with itemized artwork titles, thumbnails, and fulfillment status.
  - Artist endpoint filters orders to only include order items created by the requesting artist.
  - Update action validates tracking number and carrier name.
- **Strict Out of Scope:** Customer review creation, admin platform ledger.
- **Done Criteria:** Buyer sees personal order history; artist can view pending sales and transition status to `shipped` with tracking details.

---

#### Step 17: Verified Buyer Reviews & Wishlist API
- **Objective:** Implement review submission with verified buyer validation and user wishlist management endpoints.
- **Files to Create:**
  - `backend/app/Controllers/ReviewController.php`
  - `backend/app/Controllers/WishlistController.php`
  - Register routes:
    - `POST /api/v1/reviews` (Buyer authenticated)
    - `GET /api/v1/wishlist` (Buyer authenticated)
    - `POST /api/v1/wishlist` (Add item)
    - `DELETE /api/v1/wishlist/{art_id}` (Remove item)
- **Scope Details:**
  - For `POST /reviews`: Verify buyer has an order with `payment_status = 'succeeded'` containing the target `artwork_id`. Record rating (1-5) and comment. Update `artist_profiles.rating_cache`.
  - For wishlist: Prevent duplicate bookmarks, return full artwork summaries for saved items.
- **Strict Out of Scope:** Admin moderation of reviews.
- **Done Criteria:** Non-purchaser receives `403` when attempting review; wishlist add/remove/list operations function with persistence.

---

#### Step 18: Platform Admin Governance & Platform Analytics API
- **Objective:** Implement administration endpoints for user management, role elevation, content moderation, and GMV calculations.
- **Files to Create:**
  - `backend/app/Controllers/AdminController.php`
  - Register routes (Protected by `admin` role):
    - `GET /api/v1/admin/users` (List/search users)
    - `PUT /api/v1/admin/users/{id}` (Ban/suspend user, elevate role)
    - `GET /api/v1/admin/moderation` (Flagged items and reviews)
    - `GET /api/v1/admin/analytics` (GMV, 10% commission revenue, user stats)
- **Scope Details:**
  - Analytics computes: total GMV (`SUM(total_amount)` from succeeded orders), platform commission (`GMV * 0.10`), active listings count, active artists count.
  - User governance allows updating `status` (`active`, `suspended`, `banned`) or `role` (`buyer`, `artist`, `admin`).
- **Strict Out of Scope:** Frontend UI components.
- **Done Criteria:** Admin endpoints return aggregate financial metrics and correctly update user roles/statuses.

---

### Phase E: Frontend Base & Design System

#### Step 19: Tailwind Design System Tokens & Global Styling
- **Objective:** Configure the ArtHub responsive atomic design tokens, colors, typography, and utility classes in the frontend.
- **Files to Modify / Create:**
  - `frontend/src/app/globals.css`
  - `frontend/src/styles/theme.js` (or tailwind configuration)
  - Install dependencies: `npm install lucide-react clsx tailwind-merge`
- **Scope Details:**
  - Define custom palette:
    - Primary Indigo: `#1E3A8A`
    - Accent Emerald: `#0D9488`
    - Neutral Slate: `#F8FAFC` to `#0F172A`
    - Status Alerts: Red-500, Amber-500, Green-600
  - Configure Inter / System typography and base button/input utility styles.
- **Strict Out of Scope:** Building navigation bars or specific application pages.
- **Done Criteria:** Theme tokens and custom classes compile without Tailwind errors; base typography is visible on page preview.

---

#### Step 20: Responsive Global Navigation Shell & Footer
- **Objective:** Construct the unified application layout shell including desktop navigation, mobile slide-out drawer, search bar, and footer.
- **Files to Create / Modify:**
  - `frontend/src/components/layout/Navbar.jsx`
  - `frontend/src/components/layout/Footer.jsx`
  - `frontend/src/components/layout/MobileNav.jsx`
  - `frontend/src/app/layout.js`
- **Scope Details:**
  - Desktop Navbar: Logo, explore links (Artworks, Artists), search shortcut, wishlist heart with count badge, cart bag with badge, auth buttons / user avatar dropdown.
  - Mobile slide-out drawer with responsive toggle.
  - Clean footer with category links, artist onboarding link, and copyright notices.
- **Strict Out of Scope:** Authentication state logic, cart drawer contents.
- **Done Criteria:** `layout.js` wraps all pages with consistent responsive header and footer across viewport sizes.

---

#### Step 21: API Client & Authentication Context Provider
- **Objective:** Implement the frontend HTTP client service with automatic token attachment and global `AuthContext`.
- **Files to Create:**
  - `frontend/src/services/api.js`
  - `frontend/src/services/authService.js`
  - `frontend/src/context/AuthContext.js`
  - `frontend/src/components/ui/AuthModal.jsx` (Login & Registration modal)
- **Scope Details:**
  - `api.js` Axios or Fetch wrapper intercepting requests to inject `Authorization: Bearer <token>` from memory and handling 401 token refresh automatically.
  - `AuthContext` providing: `user`, `role`, `isAuthenticated`, `login()`, `register()`, `logout()`.
  - `AuthModal` supporting role selection (Buyer vs Artist) on registration.
- **Strict Out of Scope:** Artwork catalog fetching, checkout.
- **Done Criteria:** User can register or log in via modal, state updates in `AuthContext`, and auth token is attached to outgoing requests.

---

### Phase F: Frontend Catalog Discovery & Details

#### Step 22: Artwork Catalog Grid & Responsive Card Components
- **Objective:** Build the primary responsive artwork grid with optimized images, hover states, and shimmer loading skeletons.
- **Files to Create:**
  - `frontend/src/components/artwork/ArtworkCard.jsx`
  - `frontend/src/components/artwork/ArtworkGrid.jsx`
  - `frontend/src/components/ui/SkeletonCard.jsx`
  - `frontend/src/app/artworks/loading.js`
  - `frontend/src/app/artworks/page.js` (Basic layout)
- **Scope Details:**
  - `ArtworkCard` featuring `next/image` with fallback, artist badge, medium tag, price in USD, and quick-save wishlist heart button.
  - Responsive layout: 1 col (mobile), 2 cols (tablet), 3–4 cols (desktop).
  - Shimmer skeleton placeholder cards preventing Cumulative Layout Shift (CLS < 0.1).
- **Strict Out of Scope:** Filter sidebar logic, search input interactions.
- **Done Criteria:** Artwork catalog displays grid of cards with mock or API data and renders skeleton while loading.

---

#### Step 23: Faceted Search & Filter Sidebar
- **Objective:** Implement interactive multi-criteria filtering panel and search controls on the catalog page.
- **Files to Create:**
  - `frontend/src/components/artwork/FilterSidebar.jsx`
  - `frontend/src/components/artwork/SortSelector.jsx`
  - `frontend/src/hooks/useDebounce.js`
  - Update `frontend/src/app/artworks/page.js` with filter state
- **Scope Details:**
  - Filter by price range slider ($50 to $10,000+).
  - Medium checkboxes (Oil, Acrylic, Watercolor, Digital, Sculpture, Photography).
  - Availability toggle (All vs In Stock).
  - Debounced search-as-you-type input across titles and artist names.
  - Sort dropdown: Newest, Price: Low-to-High, Price: High-to-Low, Most Popular.
  - Active filter chips with "Clear All" button.
- **Strict Out of Scope:** Single artwork detail page.
- **Done Criteria:** Adjusting filters or sorting immediately updates catalog API query parameters and refreshes the grid without full page reload.

---

#### Step 24: Dynamic Artwork Detail Page with Image Zoom
- **Objective:** Create the rich single artwork detail view featuring high-res zoom, specifications table, and artist spotlight.
- **Files to Create:**
  - `frontend/src/app/artworks/[slug]/page.js`
  - `frontend/src/components/artwork/ArtworkZoomViewer.jsx`
  - `frontend/src/components/artwork/ArtworkSpecs.jsx`
  - `frontend/src/components/artwork/ArtistSpotlight.jsx`
- **Scope Details:**
  - Interactive image container with hover pan/zoom for inspecting fine details.
  - Item details: medium, dimensions, edition number, framing options, authenticity guarantee badge.
  - Artist bio card with link to artist portfolio.
  - Primary CTA buttons: "Add to Cart" and "Save to Wishlist".
- **Strict Out of Scope:** Cart checkout drawer, review submission form.
- **Done Criteria:** Navigating to `/artworks/{slug}` renders the high-res artwork showcase, zoom viewer, and complete metadata.

---

#### Step 25: Wishlist Context & Saved Items Portal
- **Objective:** Build optimistic client-side wishlist management with backend synchronization and dedicated buyer wishlist page.
- **Files to Create:**
  - `frontend/src/context/WishlistContext.js`
  - `frontend/src/app/buyer/wishlist/page.js`
- **Scope Details:**
  - Optimistic UI updates on heart button click (instant red toggle; revert on API failure).
  - Sync with `/api/v1/wishlist` for authenticated buyers; fallback to `localStorage` for guests.
  - Dedicated wishlist view displaying saved artworks with one-click "Move to Cart" action.
- **Strict Out of Scope:** Stripe checkout integration.
- **Done Criteria:** Clicking heart saves artwork to wishlist, updates navbar counter badge, and reflects on `/buyer/wishlist`.

---

### Phase G: Frontend Cart, Checkout & Buyer Experience

#### Step 26: Slide-out Shopping Cart Drawer & State Management
- **Objective:** Implement slide-out shopping cart drawer with item quantity controls, pricing summary, and local storage fallback.
- **Files to Create:**
  - `frontend/src/context/CartContext.js`
  - `frontend/src/components/cart/CartDrawer.jsx`
  - `frontend/src/components/cart/CartItem.jsx`
- **Scope Details:**
  - Global `CartContext`: `cartItems`, `addToCart()`, `removeFromCart()`, `updateQty()`, `subtotal`, `clearCart()`.
  - Animated slide-out drawer accessible from navbar shopping bag button.
  - Real-time stock validation notification if item quantity exceeds available stock.
  - "Proceed to Checkout" button triggering checkout flow.
- **Strict Out of Scope:** Stripe card inputs, payment submission.
- **Done Criteria:** Adding artwork updates cart badge, opens drawer displaying selected item with price, and persists in browser session.

---

#### Step 27: Embedded Stripe Checkout & Payment Flow
- **Objective:** Integrate Stripe Elements in the frontend for secure credit/debit card checkout supporting SCA 3D-Secure 2.0.
- **Files to Create:**
  - `frontend/src/components/cart/CheckoutModal.jsx`
  - `frontend/src/components/cart/StripePaymentForm.jsx`
  - Install dependencies: `npm install @stripe/stripe-js @stripe/react-stripe-js`
- **Scope Details:**
  - Modal or checkout page embedding Stripe `Elements` provider.
  - Request `client_secret` from backend `POST /api/v1/checkout/create-intent`.
  - Stripe `PaymentElement` for card collection and address block.
  - Error alert handling and 3D-Secure authentication modal.
  - On successful confirmation: clear cart, display success screen with order reference number.
- **Strict Out of Scope:** PDF receipt rendering.
- **Done Criteria:** Buyer can enter test card details, trigger 3D-Secure, complete payment, and see order confirmation screen.

---

#### Step 28: Buyer Order History & Invoice Receipt Portal
- **Objective:** Construct the authenticated buyer orders dashboard rendering itemized past purchases and invoice download links.
- **Files to Create:**
  - `frontend/src/app/buyer/orders/page.js`
  - `frontend/src/components/dashboard/OrderCard.jsx`
- **Scope Details:**
  - Fetch order history from `GET /api/v1/buyer/orders`.
  - Display purchase date, order number (`AH-2026-XXXX`), items purchased, total paid, and fulfillment status badge.
  - Direct carrier tracking button if tracking number is present.
  - "Download PDF Receipt" action button linked to `/api/v1/buyer/orders/{id}/receipt`.
- **Strict Out of Scope:** Review submission form.
- **Done Criteria:** Buyer can view list of past orders with accurate fulfillment status and item breakdowns.

---

#### Step 29: Verified Buyer Review Submission Modal & Ratings Display
- **Objective:** Build star rating picker and review submission interface for verified buyers on artwork and order pages.
- **Files to Create:**
  - `frontend/src/components/artwork/ReviewModal.jsx`
  - `frontend/src/components/artwork/ReviewList.jsx`
  - `frontend/src/components/ui/StarRating.jsx`
- **Scope Details:**
  - Interactive 1 to 5 star rating picker with hover effect.
  - Text area for collector feedback.
  - Post review to `POST /api/v1/reviews`.
  - Display reviews on artwork detail page with "Verified Buyer" badge.
- **Strict Out of Scope:** Artist studio analytics.
- **Done Criteria:** Eligible buyer can submit a star rating and comment; new review appears with verified badge in review list.

---

### Phase H: Frontend Artist Studio & Admin Consoles

#### Step 30: Artist Studio Profile & Artwork Upload Studio (CRUD)
- **Objective:** Build the artist studio portal allowing creators to edit public profile and upload new artwork listings with drag-and-drop imagery.
- **Files to Create:**
  - `frontend/src/app/artist/profile/page.js`
  - `frontend/src/app/artist/artworks/page.js`
  - `frontend/src/components/artist/ArtworkUploadModal.jsx`
- **Scope Details:**
  - Profile management: biography, avatar upload, Instagram handle, portfolio URL.
  - Listing table with active/draft status badges, prices, and edit/delete actions.
  - Upload modal: Drag-and-drop file dropzone (accepting PNG/JPEG/WebP), title, price, category selector, dimensions, medium, stock quantity.
  - Dispatch multipart form data to `POST /api/v1/artworks`.
- **Strict Out of Scope:** Order fulfillment kanban, financial analytics.
- **Done Criteria:** Artist can fill the upload modal, submit an artwork with an image, and see it appear in their studio listings table.

---

#### Step 31: Artist Order Fulfillment Kanban Board
- **Objective:** Build visual order management pipeline for artists to track sales and transition orders through fulfillment stages.
- **Files to Create:**
  - `frontend/src/app/artist/orders/page.js`
  - `frontend/src/components/artist/FulfillmentKanban.jsx`
  - `frontend/src/components/artist/TrackingModal.jsx`
- **Scope Details:**
  - Columnar Kanban or tabbed layout: `Payment Cleared` ➔ `Packaging` ➔ `Shipped` ➔ `Delivered`.
  - Order card showing buyer shipping destination, purchased piece, and timestamp.
  - Action button to mark "Shipped" prompting for courier name (FedEx, DHL, UPS) and tracking code.
  - Dispatches `PUT /api/v1/artist/orders/{id}` to update state.
- **Strict Out of Scope:** Platform-wide admin moderation.
- **Done Criteria:** Artist can drag or advance an order card from 'Packaging' to 'Shipped' with courier details.

---

#### Step 32: Artist Sales Analytics & Revenue Dashboard
- **Objective:** Create visual sales dashboard presenting MTD/YTD revenue metrics, order volumes, and earnings chart.
- **Files to Create:**
  - `frontend/src/app/artist/analytics/page.js`
  - `frontend/src/components/dashboard/MetricCard.jsx`
  - `frontend/src/components/dashboard/RevenueChart.jsx`
  - Install dependencies: `npm install recharts`
- **Scope Details:**
  - KPI Cards: Month-to-Date (MTD) Gross Revenue, Net Payout (after 10% fee), Total Orders, Average Order Value (AOV).
  - Responsive line/bar chart displaying revenue over past 30 days or 12 months.
  - Top-selling artworks ranking table.
- **Strict Out of Scope:** Platform administrator portal.
- **Done Criteria:** Artist sees formatted KPI metrics and rendered revenue graph reflecting their sales history.

---

#### Step 33: Platform Admin Console — User Management & RBAC Governance
- **Objective:** Build platform administrator dashboard to search, inspect, suspend, or elevate user accounts.
- **Files to Create:**
  - `frontend/src/app/admin/layout.js` (Admin sidebar & navigation)
  - `frontend/src/app/admin/users/page.js`
  - `frontend/src/components/admin/UserTable.jsx`
- **Scope Details:**
  - Searchable data table of all registered users with role badges (`Buyer`, `Artist`, `Admin`) and status indicators (`Active`, `Suspended`, `Banned`).
  - Action dropdown: "Elevate to Artist", "Assign Admin", "Suspend Account", "Ban Account".
  - Calls `PUT /api/v1/admin/users/{id}` with instant UI status refresh.
- **Strict Out of Scope:** DMCA content moderation queue.
- **Done Criteria:** Admin can search users, change user roles, and toggle account statuses.

---

#### Step 34: Platform Admin Console — Content Moderation & Webhook Audit Log
- **Objective:** Build content moderation console for flagged listings and real-time Stripe webhook event monitor.
- **Files to Create:**
  - `frontend/src/app/admin/moderation/page.js`
  - `frontend/src/app/admin/analytics/page.js`
  - `frontend/src/components/admin/WebhookLogStream.jsx`
- **Scope Details:**
  - Moderation queue: Inspect flagged artworks with reported reasons, high-res preview, and one-click "Approve" or "Takedown" buttons.
  - Platform Analytics: Platform Gross Merchandise Volume (GMV), 10% marketplace cut, total active transactions.
  - Webhook Monitor: Tabular stream of Stripe events from `webhook_logs` table with color-coded status badges and raw payload viewer.
- **Strict Out of Scope:** Production infrastructure provisioning.
- **Done Criteria:** Admin can inspect and remove reported items, view GMV financial metrics, and examine incoming Stripe webhook logs.

---

### Phase I: Quality Assurance, Hardening & Final Verification

#### Step 35: End-to-End Integration Verification & Security Audit
- **Objective:** Perform full end-to-end user journey tests, security validations, and cross-browser responsiveness checks.
- **Files to Create / Review:**
  - `backend/tests/api_verification.php` (or automated curl test script)
  - Review all database calls for PDO parameter binding.
  - Ensure all input outputs are sanitized with `htmlspecialchars`.
  - Validate Cumulative Layout Shift (CLS < 0.1) and loading skeletons across all routes.
- **Scope Details:**
  - Test Complete Flow:
    1. Register artist ➔ Upload artwork with image.
    2. Register buyer ➔ Search artwork ➔ Add to cart ➔ Complete Stripe checkout.
    3. Simulate Stripe webhook ➔ Verify stock decremented & order completed.
    4. Artist marks order shipped with tracking code.
    5. Buyer leaves verified review.
    6. Admin inspects transaction in GMV ledger.
- **Strict Out of Scope:** Any new feature creep outside the SRS document.
- **Done Criteria:** Full end-to-end transaction completes cleanly; all test assertions pass; zero raw SQL injection vectors found.

---

## 🚀 How to Execute

Whenever you want to start or continue implementation, simply say:

```
Do step no 1
```

I will execute **only** Step 1, verify that its deliverables and done criteria are met, report back with evidence, and pause for your instruction to proceed to the next step.
