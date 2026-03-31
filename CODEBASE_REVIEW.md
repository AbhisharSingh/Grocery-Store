# Grocery-Store Codebase Review (2026-03-31)

## Scope
- Backend (`backend/`) and frontend (`frontend/`) source files were reviewed.
- Static checks were run for frontend linting and production build.

## Critical / High Priority Bugs

1. **Stripe webhook may continue after signature verification failure**
   - In `stripeWebhooks`, the `catch` block sends a 400 response but does not `return`.
   - Execution can continue to `switch (event.type)` when `event` is undefined, causing runtime errors and duplicate response attempts.
   - File: `backend/controllers/orderController.js`.

2. **CORS origin list contains an empty string and likely blocks production frontend**
   - `allowedOrigins` is `['http://localhost:5173', '']`.
   - Empty-string origin is invalid and there is no environment-driven production origin, which can break auth cookies and API calls outside localhost.
   - File: `backend/server.js`.

3. **Client-side order total trusts mutable product data from cart state and can crash on stale product IDs**
   - `getCartAmount` dereferences `itemInfo.offerPrice` without null checks.
   - If a product was removed from catalog but remains in cart, this can throw and break cart rendering.
   - File: `frontend/src/context/AppContext.jsx`.

4. **Cart page mutates product objects from global state and can throw on missing products**
   - `getCart` sets `product.quantity` directly, mutating objects from `products` state.
   - It also assumes `product` exists and dereferences immediately.
   - This can lead to hard-to-debug state side effects and runtime crashes.
   - File: `frontend/src/pages/Cart.jsx`.

## Medium Priority Bugs / Correctness Risks

5. **Product details related-products effect can crash for invalid route IDs**
   - Effect uses `product.category` without confirming `product` exists.
   - If the URL has a non-existent product ID, this can throw before rendering fallback behavior.
   - File: `frontend/src/pages/ProductDetails.jsx`.

6. **Webhook/payment event handling mismatch risk**
   - Checkout session is created (`checkout.sessions.create`) but webhook switch handles only `payment_intent.*` events.
   - This can miss expected checkout lifecycle events in some Stripe setups and may leave orders unpaid or uncleared.
   - File: `backend/controllers/orderController.js`.

7. **Order schema uses `String` refs for relational fields**
   - `userId`, `items.product`, and `address` are typed as `String` while also using `ref` + `populate`.
   - Works inconsistently and weakens relational integrity compared with `mongoose.Schema.Types.ObjectId`.
   - File: `backend/models/Order.js`.

8. **`GET /api/product/id` reads body payload**
   - Route uses `GET` but controller expects `req.body.id`, which is non-standard and unreliable.
   - Should use route params (`/id/:id`) or query (`/id?id=`).
   - Files: `backend/routes/productRoute.js`, `backend/controllers/productController.js`.

## Code Quality / Maintainability Issues

9. **Frontend lint currently fails (7 errors, 12 warnings)**
   - Includes unused imports/variables and multiple React hook dependency warnings.
   - This reduces reliability and makes regression detection harder.
   - Affected files include `AppContext.jsx`, `Cart.jsx`, `MyOrders.jsx`, `Orders.jsx`, `Loading.jsx`, `Navbar.jsx`, `AddAddress.jsx`, `SellerLogin.jsx`, `ProductDetails.jsx`.

10. **Missing API error status codes across backend controllers/middleware**
   - Most failures return `res.json({ success: false })` with HTTP 200 status.
   - This complicates monitoring, client error handling, retries, and observability.
   - Affected files: multiple controllers and auth middlewares.

## Improvement Recommendations (Actionable)

### 1) Stabilize API contract and security first
- Add proper HTTP status codes (`400`, `401`, `403`, `404`, `500`) for failures.
- Validate request payloads (e.g., with Zod/Joi/express-validator).
- Replace hardcoded CORS list with environment-configured allowlist.
- Enforce `ObjectId` fields for referenced relations in Mongoose schemas.

### 2) Harden payment flow
- Return immediately on webhook signature failure.
- Add idempotency checks to prevent duplicate order mutations.
- Handle Stripe checkout events consistently (e.g., `checkout.session.completed`, `checkout.session.expired`) or explicitly document payment-intent-only design.

### 3) Eliminate frontend crash paths
- Guard null product lookups in cart and total calculations.
- Avoid mutating shared objects (`{ ...product, quantity }` instead of in-place mutation).
- Add invalid-product route fallback in `ProductDetails`.

### 4) Improve developer feedback loops
- Make lint green and enforce it in CI.
- Add backend linting and basic API integration tests (auth, cart, order placement, webhook).
- Add structured logging for payment and auth errors.

## Checks Run
- `npm run dev` (frontend): **starts successfully** (Vite served at `http://localhost:5173`), but this is only a runtime smoke check and does not validate code quality.
- `npm run lint` (frontend): **failed** with 7 errors, 12 warnings. This is the appropriate static-quality gate for reporting correctness and maintainability issues.
- `npm run build` (frontend): **passed** and generated production bundle.
