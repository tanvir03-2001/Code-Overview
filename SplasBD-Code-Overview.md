# SplasBD — Code Overview

[← Table of Contents](./README.md)

> **Purpose:** This document explains the full architecture of the SplasBD project — pages, functions, core client and server code, and how they connect. Use this file to understand the project without sharing the git repository.

---

## 1. Project Structure (High-Level)

```
SplasBD/
├── client/          → Next.js 16 frontend (React 19, TypeScript)
├── server/          → Express 5 REST API (TypeScript, MongoDB)
└── script/          → Database seed script (optional)
```

| Layer | Technology | Port (local) |
|-------|------------|--------------|
| **Client** | Next.js App Router, Tailwind CSS, Radix UI | `localhost:3000` |
| **Server** | Express 5, Mongoose, JWT, Passport Google OAuth | `localhost:5000` |
| **Database** | MongoDB | — |
| **Deploy** | Vercel (client + serverless API) | — |

> **Note:** `client/` and `server/` are separate git repositories — each is deployed independently.

---

## 2. How Client ↔ Server Connect

```
┌─────────────────────────────────────────────────────────────────┐
│  Browser (User)                                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  CLIENT — Next.js (SplasBD/client/)                              │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────┐   │
│  │ Pages       │ → │ lib/*Api.ts  │ → │ fetch + cookies     │   │
│  │ (app/)      │   │ api.ts       │   │ credentials:include │   │
│  │ Components  │   │ UserContext  │   └──────────┬──────────┘   │
│  └─────────────┘   └──────────────┘              │              │
└────────────────────────────────────────────────────┼──────────────┘
                                                     │ HTTP
┌────────────────────────────────────────────────────▼──────────────┐
│  SERVER — Express (SplasBD/server/)                              │
│  app.ts → /api/* routes → Controller → Model → MongoDB          │
│  + Cloudinary (images) + Nodemailer (email) + Steadfast (courier)│
└─────────────────────────────────────────────────────────────────┘
```

**Key rules:**
- Client calls the Express API directly — no Next.js API routes
- Base URL: `NEXT_PUBLIC_API_URL` or `http://localhost:5000/api`
- Auth: `httpOnly` cookies (`accessToken`, `refreshToken`) sent via `credentials: 'include'`
- Guest users can also use cart and place orders (session cookie)

---

## 3. Client — Folders and Responsibilities

```
client/
├── app/                    # All pages (Next.js App Router)
│   ├── (public)/           # Storefront pages (not visible in URL)
│   ├── admin/              # Admin panel
│   ├── layout.tsx          # Root layout (UserProvider, SEO)
│   ├── sitemap.ts          # Dynamic sitemap
│   └── robots.ts           # robots.txt
├── components/
│   ├── common/             # Navbar, ProductCard, Cart, Footer...
│   ├── admin/              # AdminSidebar, RichTextEditor...
│   ├── Loading/            # Skeleton loaders
│   └── ui/                 # Button, Dialog, Table (shadcn-style)
├── contexts/
│   └── UserContext.tsx     # Global auth state
└── lib/
    ├── api.ts              # Core HTTP client + cart/category API
    ├── authApi.ts          # Login, register, profile
    ├── orderApi.ts         # Orders + Steadfast
    ├── adminApi.ts         # Dashboard analytics
    └── hooks/              # useInfiniteScroll, etc.
```

---

## 4. Public Pages — URL and File Mapping

| URL | File | Purpose |
|-----|------|---------|
| `/` | `app/(public)/page.tsx` | Home — Hero, Category, Promo, Products |
| `/products` | `app/(public)/products/page.tsx` | Product list + filters |
| `/products/[id]` | `app/(public)/products/[id]/page.tsx` | Product detail |
| `/cart` | `app/(public)/cart/page.tsx` | Shopping cart |
| `/checkout` | `app/(public)/checkout/page.tsx` | Checkout (Bangladesh address) |
| `/order-confirmation/[id]` | `app/(public)/order-confirmation/[id]/page.tsx` | Order confirmation |
| `/profile` | `app/(public)/profile/page.tsx` | Profile, orders, reviews |
| `/login`, `/register` | `app/(public)/login|register/page.tsx` | Login / registration |
| `/auth/callback` | `app/(public)/auth/callback/page.tsx` | Google OAuth callback |
| `/guest-orders` | `app/(public)/guest-orders/page.tsx` | Guest order lookup |

---

## 5. Admin Pages — URL and File Mapping

| URL | Purpose |
|-----|---------|
| `/admin` | Dashboard (stats) |
| `/admin/products`, `/add`, `/[id]/edit` | Product CRUD |
| `/admin/orders`, `/orders/[id]` | Order management + Steadfast |
| `/admin/users`, `/users/roles` | Users and roles |
| `/admin/banners` | Homepage banners |
| `/admin/categories` | Category / subcategory |
| `/admin/promo-codes` | Promo codes |
| `/admin/shipping` | Dhaka / outside Dhaka delivery charges |
| `/admin/analytics` | Revenue / profit analytics |
| `/admin/settings` | Site settings |

**Admin protection:** `app/admin/layout.tsx` — on page load calls `authApi.getProfile()`; redirects to login if `role !== ADMIN`.

---

## 6. Client Code — Core Parts

### 6.1 Home Page (`app/(public)/page.tsx`)

The home page composes several components:

```tsx
// app/(public)/page.tsx — main sections
<LeftSidebar />              // Category sidebar (desktop)
<HeroCarousel />             // Banner carousel → GET /api/banners
<CategoryShowcase />         // Categories → GET /api/categories
<DynamicPromotionalSections /> // Promo sections → GET /api/promotional-sections
<AllProducts />              // All products → GET /api/products
<TrustSection />             // Delivery / warranty trust block
```

| Component | API Endpoint | Server File |
|-----------|--------------|-------------|
| `HeroCarousel` | `GET /api/banners` | `features/banner/banner.route.ts` |
| `CategoryShowcase` | `GET /api/categories` | `features/category/category.route.ts` |
| `DynamicPromotionalSections` | `GET /api/promotional-sections` | `features/promotionalSection/` |
| `AllProducts` | `GET /api/products` | `features/product/product.route.ts` |

---

### 6.2 Core API Client (`lib/api.ts`)

Central hub for all API calls. Auto-refreshes token on expiry:

```typescript
// lib/api.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:5000/api';

export async function apiRequest<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
  // GET request → cache via cache.ts
  // 401 error → refreshAccessToken() → retry
  // fetch(..., { credentials: 'include' }) — sends cookies
}
```

**Connected Server:** Any `/api/*` route — mounted in `server/src/app.ts`.

---

### 6.3 Auth API (`lib/authApi.ts`)

```typescript
// lib/authApi.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:5000/api';

// Every request uses credentials: 'include' — sends JWT cookies
authApi.login({ email, password })      → POST /api/auth/login
authApi.register({ name, email, password }) → POST /api/auth/register
authApi.getProfile()                    → GET  /api/auth/profile
authApi.refreshToken()                  → POST /api/auth/refresh-token
authApi.logout()                        → POST /api/auth/logout
```

**Connected Server:** `server/src/features/auth/auth.route.ts` → `auth.controller.ts`

---

### 6.4 User Context (`contexts/UserContext.tsx`)

Global auth state — all pages wrapped with `UserProvider`:

```typescript
// contexts/UserContext.tsx
export function UserProvider({ children }) {
  // On mount: calls authApi.getProfile()
  // After login: links guest orders via orderApi.linkGuestOrders()
  // logout → clears cookies + localStorage
}
```

| Client Function | Server Endpoint | Description |
|----------------|-----------------|-------------|
| `refreshUser()` | `GET /api/auth/profile` | Load user data |
| `logout()` | `POST /api/auth/logout` | End session |
| Guest order link | `POST /api/orders/link-guest-orders` | Link guest orders after login |

---

### 6.5 Order API (`lib/orderApi.ts`)

```typescript
// lib/orderApi.ts
orderApi.createOrder(data)           → POST /api/orders
orderApi.getMyOrders()               → GET  /api/orders/my-orders
orderApi.linkGuestOrders(orderIds)   → POST /api/orders/link-guest-orders
orderApi.createSteadfastOrder(id)    → POST /api/orders/:id/steadfast  (Admin)
```

**Connected Server:** `server/src/features/order/order.controller.ts`

---

## 7. Server — Folders and Responsibilities

```
server/src/
├── app.ts                  # Express setup + all route mounts
├── server.ts               # Standalone server (PORT 5000)
├── config/
│   ├── database.ts         # MongoDB connection
│   └── passport.ts         # Google OAuth
├── middleware/
│   └── auth.middleware.ts  # authenticateUser, requireAdmin
├── features/               # Feature-based modules
│   ├── auth/               # route → controller → model
│   ├── product/
│   ├── order/
│   ├── cart/
│   └── ... (17 features)
└── utils/
    ├── jwt.ts              # Token generate/verify
    ├── email.ts            # Verification email
    └── cloudinary.ts       # Image upload
```

**Pattern for each feature:**
```
features/product/
├── product.route.ts      # URL + middleware definition
├── product.controller.ts # Request handling, validation
├── product.model.ts      # Database queries
└── product.schema.ts     # Mongoose schema
```

---

## 8. Server Code — Core Parts

### 8.1 Route Mounting (`server/src/app.ts`)

All APIs mounted under `/api` prefix:

```typescript
// server/src/app.ts
app.use('/api/auth', authRoutes);
app.use('/api/users', userRoutes);
app.use('/api/products', productRoutes);
app.use('/api/categories', categoryRoutes);
app.use('/api/orders', orderRoutes);
app.use('/api/cart', cartRoutes);
app.use('/api/admin', adminRoutes);
app.use('/api/settings', settingsRoutes);
app.use('/api/banners', bannerRoutes);
app.use('/api/promos', promoRoutes);
app.use('/api/search', searchRoutes);
app.use('/api/reviews', reviewRoutes);
// ... more
```

| Route Prefix | Client API File | Main Purpose |
|-------------|-----------------|--------------|
| `/api/auth` | `lib/authApi.ts` | Login, register, OAuth |
| `/api/products` | `lib/api.ts` | Product CRUD |
| `/api/cart` | `lib/api.ts` | Session-based cart |
| `/api/orders` | `lib/orderApi.ts` | Order create/manage |
| `/api/admin` | `lib/adminApi.ts` | Dashboard, analytics |

---

### 8.2 Auth Routes (`features/auth/auth.route.ts`)

```typescript
// Public routes
router.post('/register', AuthController.register);
router.post('/login', AuthController.login);
router.post('/refresh-token', AuthController.refreshToken);
router.get('/google', passport.authenticate('google'));
router.get('/google/callback', ..., AuthController.googleCallback);

// Protected routes
router.get('/profile', authenticateUser, AuthController.getProfile);
router.put('/profile', authenticateUser, upload.single('photo'), AuthController.updateProfile);
```

---

### 8.3 Auth Middleware (`middleware/auth.middleware.ts`)

```typescript
// Reads accessToken from cookie
export const authenticateUser = async (req, res, next) => {
  const token = req.cookies?.accessToken || req.headers.authorization?.replace('Bearer ', '');
  const decoded = verifyAccessToken(token);
  const user = await UserModel.findById(decoded.id);
  req.user = decoded;
  next();
};

export const requireAdmin = ...  // checks role === 'ADMIN'
export const optionalAuth = ...  // attaches user if logged in, otherwise continues
```

| Middleware | Used On | Client Call |
|-----------|---------|-------------|
| `authenticateUser` | Profile, my-orders | `authApi.getProfile()` |
| `requireAdmin` | Admin CRUD routes | Admin panel API calls |
| `optionalAuth` | Product detail, create order | Guest + logged-in both |

---

### 8.4 Order Create (`features/order/order.controller.ts`)

```typescript
static create = asyncHandler(async (req: AuthRequest, res: Response) => {
  const orderData: CreateOrderDto = req.body;
  const userId = req.user?.id || null;  // null for guests

  // Stock validation
  for (const item of orderData.items) {
    const product = await ProductModel.findById(item.product);
    // size/stock check...
  }

  const order = await OrderModel.create(orderData, userId);
  return ResponseHandler.success(res, order, 'Order created successfully', 201);
});
```

**Called from client:** `checkout/page.tsx` → `orderApi.createOrder()` → `POST /api/orders`

---

## 9. Complete Auth Flow

```
1. Register
   Client: register/page.tsx → authApi.register()
   Server: POST /api/auth/register → sends email verification

2. Email Verify
   Client: verify-email/page.tsx → authApi.verifyEmail(token)
   Server: POST /api/auth/verify-email

3. Login
   Client: login/page.tsx → authApi.login()
   Server: POST /api/auth/login → sets accessToken + refreshToken cookies
   Client: UserContext.refreshUser() → loads profile

4. Token Expire (401)
   Client: apiRequest() → authApi.refreshToken() → retry
   Server: POST /api/auth/refresh-token → new accessToken cookie

5. Google OAuth
   Client: "Login with Google" → window.location = /api/auth/google
   Server: Google → callback → sets cookies → redirect /auth/callback
   Client: auth/callback/page.tsx → refreshUser()

6. Admin Access
   Client: admin/layout.tsx → getProfile() → role === ADMIN?
   Server: requireAdmin middleware on all admin routes
```

---

## 9.1 Checkout Flow (Guest + Logged-in)

```
Cart Page                    Checkout Page                 Server
─────────                    ─────────────                 ──────
cart/page.tsx        →       checkout/page.tsx
  │                            │
  │ GET /api/cart              │ promo verify → POST /api/promos/verify
  │                            │ settings → GET /api/settings (shipping charge)
  │                            │
  └────────────────────────────┼──→ POST /api/orders
                               │      (optionalAuth — guest OK)
                               │
                               └──→ order-confirmation/[id]/page.tsx
```

---

## 10. API Endpoint Summary

### Auth — `/api/auth`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| POST | `/register` | Public | `authApi.register` |
| POST | `/login` | Public | `authApi.login` |
| POST | `/refresh-token` | Cookie | `api.ts` (auto) |
| GET | `/profile` | User | `authApi.getProfile` |
| GET | `/google` | Public | OAuth redirect |

### Products — `/api/products`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Optional | `apiRequest('/products')` |
| GET | `/:id` | Optional | Product detail page |
| POST | `/` | Admin | Admin product add |
| PUT | `/:id` | Admin | Admin product edit |

### Cart — `/api/cart`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Session | Cart page |
| POST | `/add` | Session | ProductCard "Add to Cart" |
| PUT | `/:itemId/quantity` | Session | Cart quantity update |

### Orders — `/api/orders`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| POST | `/` | Optional | Checkout |
| GET | `/my-orders` | User | Profile page |
| POST | `/:id/steadfast` | Admin | Admin order detail |
| POST | `/steadfast/webhook` | Webhook | Steadfast courier callback |

### Admin — `/api/admin`
| Method | Endpoint | Client |
|--------|----------|--------|
| GET | `/dashboard/stats` | Admin dashboard |
| GET | `/analytics` | Admin analytics page |

*(Full endpoint list also available in server `GET /` response)*

---

## 11. Feature Module Mapping (Client ↔ Server)

| Feature | Client File/Component | Server Feature | MongoDB Model |
|---------|----------------------|----------------|---------------|
| **Auth** | `authApi.ts`, `UserContext`, login/register pages | `features/auth/` | User, Admin |
| **Products** | `ProductGrid`, `ProductDetails`, admin products | `features/product/` | Product |
| **Categories** | `CategoryShowcase`, `LeftSidebar` | `features/category/` | Category |
| **Cart** | `CartPage`, `api.ts` cart functions | `features/cart/` | Cart |
| **Orders** | `checkout`, `orderApi.ts`, profile orders | `features/order/` | Order |
| **Reviews** | `ReviewsSection`, `ReviewForm` | `features/review/` | Review |
| **Banners** | `HeroCarousel` | `features/banner/` | Banner |
| **Promo** | Checkout promo input | `features/promo/` | Promo |
| **Search** | `SearchBar` | `features/search/` | Search |
| **Settings** | Footer, checkout shipping | `features/settings/` | Settings |
| **Steadfast** | Admin order "Send to Courier" | `features/steadfast/` | (external API) |
| **CMS** | terms/privacy pages + admin editor | `features/terms/`, `privacy/` | Terms, Privacy |

---

## 12. Environment Variables

### Client (`.env.local`)
```
NEXT_PUBLIC_API_URL=http://localhost:5000/api
```

### Server (`.env`)
```
MONGODB_URI=mongodb://...
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
FRONTEND_URL=http://localhost:3000
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
CLOUDINARY_CLOUD_NAME=...
STEADFAST_API_KEY=...
```

---

## 13. How to Run Locally

```bash
# Terminal 1 — Server
cd SplasBD/server
npm install
npm run dev          # → http://localhost:5000

# Terminal 2 — Client
cd SplasBD/client
npm install
npm run dev          # → http://localhost:3000
```

---

## 14. Key File Reference

| File | Role |
|------|------|
| `client/lib/api.ts` | Core HTTP client, caching, token refresh |
| `client/lib/authApi.ts` | All auth endpoints |
| `client/contexts/UserContext.tsx` | Global user state |
| `client/app/admin/layout.tsx` | Admin route protection |
| `client/app/(public)/page.tsx` | Home page composition |
| `server/src/app.ts` | Express app + route mounting |
| `server/src/middleware/auth.middleware.ts` | JWT verification |
| `server/src/features/auth/auth.route.ts` | Auth endpoints |
| `server/src/features/order/order.controller.ts` | Order logic |
| `server/api/index.ts` | Vercel serverless entry |

---

## 15. Code Examples

Real code from the project — shows TypeScript patterns used across client and server.

### Example 1 — API client with auto token refresh (`client/lib/api.ts`)

```typescript
export async function apiRequest<T>(
  endpoint: string,
  options: RequestInit = {},
  retryCount = 0
): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${endpoint}`, {
    ...options,
    credentials: 'include', // Include cookies for authentication
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  // Handle 401 errors - try to refresh token
  if (response.status === 401 && retryCount === 0) {
    try {
      await refreshAccessToken();
      return apiRequest<T>(endpoint, options, retryCount + 1);
    } catch (refreshError) {
      // Refresh failed - let the error handling below process it
    }
  }
  // ...
}
```

### Example 2 — Guest order linking on login (`client/contexts/UserContext.tsx`)

```typescript
const refreshUser = async (retryCount = 0) => {
  try {
    const userData = await authApi.getProfile();
    setUser(userData);

    if (typeof window !== 'undefined') {
      localStorage.setItem('userSession', 'true');
    }

    // Link guest orders if user just logged in
    const guestOrderIds = getGuestOrders();
    if (guestOrderIds.length > 0) {
      try {
        await orderApi.linkGuestOrders(guestOrderIds);
        clearGuestOrders();
      } catch (error) {
        console.error('Error linking guest orders:', error);
        // Don't fail the login if linking fails
      }
    }
  } catch (error: any) {
    // ...
  }
};
```

### Example 3 — Auth middleware with cookie + Bearer fallback (`server/src/middleware/auth.middleware.ts`)

```typescript
export const authenticateUser = async (
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    const token = req.cookies?.accessToken || req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      throw new AppError('Authentication required', 401);
    }

    const decoded = verifyAccessToken(token);
    const user = await UserModel.findById(decoded.id);

    if (!user) {
      const admin = await AdminModel.findById(decoded.id);
      if (!admin || !admin.isActive) {
        throw new AppError('User not found', 401);
      }
    }

    req.user = decoded;
    next();
  } catch (error: any) {
    // ...
  }
};
```

### Example 4 — Order creation with stock validation (`server/src/features/order/order.controller.ts`)

```typescript
static create = asyncHandler(async (req: AuthRequest, res: Response) => {
  const orderData: CreateOrderDto = req.body;

  if (!orderData.user || !orderData.items || orderData.items.length === 0) {
    throw new AppError('User information and items are required', 400);
  }

  if (orderData.promoCode && !req.user) {
    throw new AppError('Promo codes can only be used by logged-in users', 403);
  }

  const userId = req.user?.id || null;

  for (const item of orderData.items) {
    const product = await ProductModel.findById(item.product);
    if (!product) {
      throw new AppError(`Product ${item.product} not found`, 404);
    }

    if (item.size) {
      const sizeStock = product.sizes.find((s) => s.size === item.size);
      if (!sizeStock || sizeStock.stock < item.quantity) {
        throw new AppError(`Insufficient stock for size ${item.size}`, 400);
      }
    } else if (product.stock < item.quantity) {
      throw new AppError(`Insufficient stock for product ${product.name}`, 400);
    }
  }

  const order = await OrderModel.create(orderData, userId);
  return ResponseHandler.success(res, order, 'Order created successfully', 201);
});
```

---

## 16. Architecture Decisions (Good to Know)

1. **Separate git repos** — client and server deployed independently
2. **Cookie-based JWT** — no token in localStorage, only `userSession` flag
3. **Guest commerce** — cart/checkout without login; orders linked on login
4. **Feature-sliced server** — each domain in its own folder (route → controller → model)
5. **Bangladesh-specific** — Dhaka/outside shipping, Steadfast courier, Bengali font
6. **Client-side cache** — GET responses cached in `lib/cache.ts`; invalidated on mutation

---

*Last updated: July 2026 | SplasBD E-commerce Platform*
