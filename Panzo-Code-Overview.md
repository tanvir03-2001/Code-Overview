# Panzo — Code Overview

[← সূচিপত্র](./README.md)

> **উদ্দেশ্য:** Panzo E-commerce প্রজেক্টের পুরো আর্কিটেকচার, পেজ, ফাংশন, client/admin/server কোডের মূল অংশ এবং তাদের মধ্যে সম্পর্ক ব্যাখ্যা করে। Git repo শেয়ার না করলেও এই ফাইল দিয়ে প্রজেক্ট বুঝতে সাহায্য করবে।

---

## ১. প্রজেক্ট কাঠামো (High-Level)

```
Panzo/
├── client/          → Next.js 16 স্টোরফ্রন্ট (React 19, port 3000)
├── admin/           → Vite + React অ্যাডমিন প্যানেল (port 5173)
├── server/          → Express 4 REST API (port 5000)
└── CASE_STUDY.md    → বিদ্যমান architecture doc (Bengali)
```

| অংশ | টেকনোলজি | পোর্ট (লোকাল) |
|-----|----------|---------------|
| **Client** | Next.js App Router, Tailwind CSS 4, Swiper | `localhost:3000` |
| **Admin** | Vite 7, React Router 7, Radix UI, Recharts | `localhost:5173` |
| **Server** | Express 4, Mongoose 8, JWT, Passport OAuth | `localhost:5000` |
| **Database** | MongoDB (`panzo-ecommerce`) | — |

> **নোট:** `client/`, `admin/`, `server/` — তিনটোই **আলাদা git repository**। SplasBD-র মতো client-এ admin embedded নেই; admin আলাদা Vite app।

---

## ২. Client ↔ Admin ↔ Server কিভাবে যুক্ত

```
┌─────────────────────┐     ┌─────────────────────┐
│  Client (Next.js)   │     │  Admin (Vite/React) │
│  :3000              │     │  :5173              │
└─────────┬───────────┘     └─────────┬───────────┘
          │  REST + credentials       │
          │  NEXT_PUBLIC_API_URL      │  VITE_API_URL
          │  /api/v1                  │
          └───────────┬───────────────┘
                      ▼
          ┌───────────────────────┐
          │  Express API :5000    │
          │  /api/v1/*            │
          │  JWT httpOnly cookies │
          └───────────┬───────────┘
                      ▼
          ┌───────────────────────┐
          │  MongoDB              │
          │  + Activity Log DB    │
          │  + Cloudinary         │
          └───────────────────────┘
```

**মূল নিয়ম:**
- Client ও Admin দুটোই সরাসরি Express API-তে কল করে — BFF layer নেই
- Base URL: `http://localhost:5000/api/v1`
- Auth: `httpOnly` cookie (`accessToken`, `refreshToken`) — `credentials: 'include'`
- Guest user কার্ট ও checkout করতে পারে
- Admin panel permission-based route guard ব্যবহার করে (RBAC)

---

## ৩. Client — ফোল্ডার ও দায়িত্ব

```
client/src/
├── app/                    # Next.js App Router পেজ
│   ├── page.tsx            # হোম
│   ├── products/, product/[slug]/
│   ├── category/[slug]/, brand/[slug]/
│   ├── cart/, checkout/, order-success/
│   ├── orders/, orders/[orderId]/
│   ├── wishlist/, account/
│   ├── login/, signup/, verify-email/
│   └── sitemap.ts, robots.ts
├── components/
│   ├── ShopProvider.tsx    # Cart, wishlist, user state
│   ├── Header.tsx, Footer.tsx
│   ├── product/            # ProductDetailsClient, gallery
│   └── loading/            # Skeleton loaders
├── utils/
│   └── api.ts              # মূল API client (ApiClient class)
├── hooks/                  # useInfiniteProducts, useSearch
├── actions/
│   └── navActions.ts       # Server Actions (cached nav data)
└── lib/                    # metadata, getProduct
```

---

## ৪. Client পেজ — URL ও ফাইল ম্যাপিং

| URL | ফাইল | কাজ |
|-----|------|-----|
| `/` | `app/page.tsx` | হোম — hero carousel, categories, product grids |
| `/products` | `app/products/page.tsx` | প্রোডাক্ট লিস্ট + ফিল্টার |
| `/product/[slug]` | `app/product/[slug]/page.tsx` | প্রোডাক্ট ডিটেইল (color/size variants) |
| `/category/[slug]` | `app/category/[slug]/page.tsx` | ক্যাটাগরি অনুযায়ী প্রোডাক্ট |
| `/brand/[slug]` | `app/brand/[slug]/page.tsx` | ব্র্যান্ড অনুযায়ী প্রোডাক্ট |
| `/categories` | `app/categories/page.tsx` | সব ক্যাটাগরি |
| `/search` | `app/search/page.tsx` | সার্চ + autocomplete |
| `/cart` | `app/cart/page.tsx` | শপিং কার্ট |
| `/checkout` | `app/checkout/page.tsx` | চেকআউট (BD address, COD) |
| `/order-success` | `app/order-success/page.tsx` | অর্ডার কনফার্মেশন |
| `/orders` | `app/orders/page.tsx` | অর্ডার হিস্ট্রি |
| `/orders/[orderId]` | `app/orders/[orderId]/page.tsx` | অর্ডার ডিটেইল |
| `/wishlist` | `app/wishlist/page.tsx` | সেভ করা প্রোডাক্ট |
| `/account` | `app/account/page.tsx` | প্রোফাইল ও ঠিকানা |
| `/login`, `/signup` | `app/login/`, `app/signup/` | Auth |
| `/verify-email` | `app/verify-email/page.tsx` | ইমেইল verification |

---

## ৫. Admin পেজ — URL ও ফাইল ম্যাপিং

Admin আলাদা Vite app — React Router দিয়ে route define:

| URL | Page | Permission |
|-----|------|------------|
| `/login` | Login | Public |
| `/` | Dashboard | `dashboard.view` |
| `/users` | Users | `view_users` |
| `/categories`, `/categories/new`, `/categories/edit/:id` | Categories CRUD | `view/create/update_categories` |
| `/brands`, `/brands/new`, `/brands/edit/:id` | Brands CRUD | `view/create/update_brands` |
| `/products`, `/products/new`, `/products/edit/:id` | Products CRUD | `view/create/update_products` |
| `/banners`, `/banners/new`, `/banners/edit/:id` | Banners CRUD | `view/create/update_banners` |
| `/orders`, `/orders/new`, `/orders/view/:id` | Orders | `view/create_orders` |
| `/expenses`, `/expenses/new`, `/expenses/dashboard` | Expenses | `view/create_expenses` |
| `/investments` | Investments | `view_investments` |
| `/permissions`, `/permissions/manage/:userId` | RBAC | `view/manage_permissions` |
| `/analytics`, `/analytics/visitor/:identifier` | Analytics | Role-based |
| `/promoter-products` | Promoter links | Promoter role |

**Admin সুরক্ষা:** `PermissionRoute` component — প্রতিটি route-এ required permission check:

```tsx
// admin/src/App.tsx
<Route path="/" element={
  <PermissionRoute requiredPermission="dashboard.view">
    <Dashboard />
  </PermissionRoute>
} />
```

---

## ৬. Client কোড — মূল অংশ

### ৬.১ API Client (`client/src/utils/api.ts`)

সব API কলের কেন্দ্রবিন্দু — `ApiClient` class:

```typescript
// client/src/utils/api.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:5000/api/v1';

class ApiClient {
  // fetch with credentials: 'include'
  // Next.js revalidate caching support
}

// Domain-specific APIs
export const productApi = { getAllProducts, searchProducts, getProductBySlug, getRecommendedProducts, ... };
export const orderApi = { createOrder, getMyOrders, syncGuestOrders, ... };
export const userApi = { login, signup, getProfile, ... };
export const behaviorApi = { trackBehavior };  // sendBeacon for analytics
```

| Client API | Server Endpoint | Server Feature |
|-----------|-----------------|----------------|
| `productApi.getAllProducts()` | `GET /api/v1/products` | `features/product/` |
| `productApi.getRecommendedProducts()` | `GET /api/v1/products/recommended` | `algorithm/recommendationEngine.ts` |
| `orderApi.createOrder()` | `POST /api/v1/orders` | `features/order/` |
| `userApi.login()` | `POST /api/v1/users/login` | `features/user/` |
| `behaviorApi.trackBehavior()` | `POST /api/v1/behavior/track` | `algorithm/product_view_algorithm/` |

---

### ৬.২ ShopProvider (`components/ShopProvider.tsx`)

Redux ছাড়াই React Context + localStorage দিয়ে state manage:

```typescript
// ShopProvider — cart, wishlist, user, addresses
// localStorage key: 'panzo_shop_v1'
// Cart add/remove, wishlist toggle, user login state
```

| State | Storage | Server sync |
|-------|---------|-------------|
| Cart | localStorage | Checkout-এ `POST /api/v1/orders` |
| Wishlist | localStorage | Client-only |
| User | Cookie + context | `GET /api/v1/users/profile` |
| Addresses | localStorage | Account page |

---

### ৬.৩ Server Actions (`actions/navActions.ts`)

Next.js Server Actions + `unstable_cache` দিয়ে nav data cache:

```typescript
'use server';
import { unstable_cache } from 'next/cache';

export async function getNavData() {
  const getCachedCategories = unstable_cache(
    async () => categoryApi.getAllCategories(),
    ['nav-categories'],
    { revalidate: 900, tags: ['nav-categories'] }
  );
  // categories + brands return
}
```

**সংযুক্ত Server:** `GET /api/v1/categories`, `GET /api/v1/brands`

---

### ৬.৪ Analytics Tracking (`utils/analyticsTracking.ts`)

Fire-and-forget tracking — UX block করে না:

```typescript
// behaviorApi.trackBehavior() → navigator.sendBeacon()
// POST /api/v1/behavior/track
// Session, page-view, click-event, search tracking
```

---

## ৭. Admin কোড — মূল অংশ

### ৭.১ Admin API Client (`admin/src/lib/api.ts`)

Client-এর মতো pattern + **401 → auto refresh → retry**:

```typescript
// admin/src/lib/api.ts
// ApiClient with token refresh on 401
// VITE_API_URL = http://localhost:5000/api/v1
```

### ৭.২ Auth & Permission Context

```
admin/src/contexts/
├── AuthContext.tsx         # Admin login state, token refresh
└── PermissionContext.tsx   # User permissions load

admin/src/components/
├── ProtectedRoute.tsx      # Login required
└── PermissionRoute.tsx     # Specific permission required
```

---

## ৮. Server — ফোল্ডার ও দায়িত্ব

```
server/src/
├── app.ts                  # Express setup + middleware
├── server.ts               # Bootstrap: DB connect, super user init
├── routes/index.ts         # সব route registry
├── config/
│   ├── env.ts              # Centralized env config
│   ├── db.ts               # MongoDB connection
│   └── passport.ts         # Google/Facebook OAuth
├── middlewares/
│   ├── auth.middleware.ts  # JWT from cookie
│   ├── permission.middleware.ts
│   └── error.middleware.ts
├── features/               # Feature modules
│   ├── user/               # Auth, OAuth, user CRUD
│   ├── product/            # Catalog, search
│   ├── category/, brand/, banner/
│   ├── order/
│   ├── expense/, investment/
│   ├── dashboard/
│   ├── permission/, rbac/
│   ├── analytics/, tracking/
│   └── activityLog/
├── algorithm/
│   └── product_view_algorithm/
│       ├── recommendationEngine.ts
│       ├── productScoring.ts
│       └── behavior.route.ts
└── utils/                  # jwt, email, cloudinary, apiResponse
```

**প্রতিটি feature-এর প্যাটার্ন:**
```
features/product/
├── product.route.ts      # URL + middleware
├── product.controller.ts # Request handle
├── product.service.ts    # Business logic
└── product.model.ts      # Mongoose model
```

---

## ৯. Server কোড — মূল অংশ

### ৯.১ Route Mounting (`server/src/routes/index.ts`)

```typescript
// server/src/app.ts
app.use('/api/v1', routes);

// routes/index.ts
router.use('/users', userRoutes);
router.use('/products', productRoutes);
router.use('/categories', categoryRoutes);
router.use('/brands', brandRoutes);
router.use('/banners', bannerRoutes);
router.use('/orders', orderRoutes);
router.use('/expenses', expenseRoutes);
router.use('/investments', investmentRoutes);
router.use('/dashboard', dashboardRoutes);
router.use('/permissions', permissionRoutes);
router.use('/rbac', rbacRoutes);
router.use('/analytics', analyticsRoutes);
router.use('/behavior', behaviorRoutes);
router.use('/tracking', trackingRoutes);
router.use('/upload', uploadRoutes);
```

---

### ৯.২ Feature Controller Pattern

```typescript
// features/product/product.controller.ts
export const getProducts = asyncHandler(async (req, res) => {
  const result = await productService.getAll(true, { page, limit, search, category, brand });
  sendSuccess(res, 200, 'Products fetched successfully.', result);
});
```

---

### ৯.৩ Recommendation Engine

```typescript
// algorithm/product_view_algorithm/recommendationEngine.ts
// User behavior scoring → personalized product recommendations
// Client: productApi.getRecommendedProducts() → GET /api/v1/products/recommended
```

---

## ১০. API Endpoint সংক্ষিপ্ত তালিকা

**Base:** `/api/v1`

### Users & Auth — `/api/v1/users`
| Method | Endpoint | Auth | Client/Admin |
|--------|----------|------|--------------|
| POST | `/signup` | Public | Client signup |
| POST | `/login` | Public | Client/Admin login |
| POST | `/refresh-token` | Cookie | Auto refresh |
| GET | `/profile` | Auth | Account page |
| GET | `/auth/google` | Public | Google OAuth |

### Products — `/api/v1/products`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Public | Product listing |
| GET | `/search` | Public | Search page |
| GET | `/suggest` | Public | Autocomplete |
| GET | `/slug/:slug` | Public | Product detail |
| GET | `/recommended` | Public | Home recommended |
| GET | `/admin` | Permission | Admin products |
| POST | `/` | Permission | Admin create |

### Orders — `/api/v1/orders`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| POST | `/` | Optional | Checkout (guest OK) |
| GET | `/my-orders` | Auth | Orders page |
| POST | `/sync-guest-orders` | Auth | Login-এ guest order link |
| GET | `/guest-orders?email=&phone=` | Public | Guest order lookup |

### Analytics — `/api/v1/analytics`
| Method | Endpoint | Auth | Client/Admin |
|--------|----------|------|--------------|
| POST | `/session`, `/page-view`, `/click-event` | Public | Client tracker |
| GET | `/overview`, `/visitors` | Admin | Admin analytics |
| GET | `/promoter/:promoterId` | Promoter | Promoter dashboard |

### RBAC — `/api/v1/rbac`
| Method | Endpoint | Auth | Admin |
|--------|----------|------|-------|
| GET | `/me/permissions` | Auth | Permission load |
| GET | `/permissions`, `/roles` | Admin | Permission management |
| PUT | `/users/:userId/permissions` | Admin | Assign permissions |

---

## ১১. Feature Module ম্যাপিং

| Feature | Client | Admin | Server |
|---------|--------|-------|--------|
| **Catalog** | Browse, filter, infinite scroll | Product/Category/Brand CRUD | Product search, color/size variants |
| **Cart & Checkout** | `ShopProvider` + localStorage | Custom order form | Guest + auth orders, COD |
| **Auth** | Login/signup/OAuth pages | Login + token refresh | JWT cookies, email verify |
| **Recommendations** | Home recommended section | Product analytics tab | `recommendationEngine` |
| **Analytics** | `AnalyticsTracker` | Multi-tab dashboard | Session/page/click/search tracking |
| **Promoter** | Promo visit on product view | Promoter products page | Visit logs |
| **Finance** | — | Expenses, investments | Cost allocation |
| **RBAC** | — | Permission-gated routes | Legacy + RBAC dual system |
| **SEO** | sitemap, structured data | — | Slug-based URLs |

**User Roles:** `user`, `admin`, `moderator`, `super_user`, `reseller`, `product_manager`, `promoter`

---

## ১২. Auth Flow

```
১. Signup
   Client: signup/page.tsx → userApi.signup()
   Server: POST /api/v1/users/signup → verification email

২. Login
   Client: login/page.tsx → userApi.login()
   Server: POST /api/v1/users/login → JWT cookies set
   Client: ShopProvider user state update

৩. Google/Facebook OAuth
   Client: redirect → /api/v1/users/auth/google
   Server: Passport OAuth → callback → cookies → redirect client

৪. Admin Login
   Admin: login page → same /users/login endpoint
   Admin: AuthContext → PermissionContext load permissions
   Admin: PermissionRoute guards each page

৫. Token Refresh (401)
   Admin api.ts: auto POST /users/refresh-token → retry
   Client: similar pattern in ApiClient
```

---

## ১৩. Checkout Flow

```
Cart (ShopProvider)     Checkout Page              Server
─────────────────       ─────────────              ──────
cart/page.tsx    →      checkout/page.tsx
  │                        │
  │ localStorage cart      │ BD address cascade
  │                        │ POST /api/v1/orders (optionalAuth)
  │                        │
  └────────────────────────┼──→ order-success/page.tsx
                           │
                           └── Guest order → login-এ sync-guest-orders
```

---

## ১৪. Environment Variables

### Client (`.env.local`)
```
NEXT_PUBLIC_API_URL=http://localhost:5000/api/v1
```

### Admin (`.env`)
```
VITE_API_URL=http://localhost:5000/api/v1
```

### Server (`.env`)
```
PORT=5000
MONGODB_URI=mongodb://...
MONGODB_ACTIVITY_LOG_URI=mongodb://...
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
CLIENT_URL=http://localhost:3000
ADMIN_URL=http://localhost:5173
CLOUDINARY_*=...
SMTP_*=...
SUPER_USER_EMAIL=...
SUPER_USER_PASSWORD=...
```

---

## ১৫. চালানোর নিয়ম (Local)

```bash
# Terminal 1 — Server
cd Panzo/server
npm install
npm run dev          # → http://localhost:5000

# Terminal 2 — Client
cd Panzo/client
npm install
npm run dev          # → http://localhost:3000

# Terminal 3 — Admin
cd Panzo/admin
npm install
npm run dev          # → http://localhost:5173
```

---

## ১৬. গুরুত্বপূর্ণ ফাইল রেফারেন্স

| ফাইল | ভূমিকা |
|------|--------|
| `client/src/utils/api.ts` | মূল API client + domain APIs |
| `client/src/components/ShopProvider.tsx` | Cart, wishlist, user state |
| `client/src/utils/analyticsTracking.ts` | Client-side tracking |
| `client/src/actions/navActions.ts` | Cached server actions |
| `admin/src/lib/api.ts` | Admin API + token refresh |
| `admin/src/contexts/AuthContext.tsx` | Admin auth state |
| `admin/src/components/PermissionRoute.tsx` | Permission guard |
| `admin/src/App.tsx` | React Router route table |
| `server/src/app.ts` | Express middleware stack |
| `server/src/routes/index.ts` | API route registry |
| `server/src/config/env.ts` | Centralized env |
| `server/src/algorithm/.../recommendationEngine.ts` | Product recommendations |
| `server/scripts/seedDatabase.ts` | DB seeding |

---

## ১৭. SplasBD vs Panzo — মূল পার্থক্য

| বিষয় | SplasBD | Panzo |
|-------|---------|-------|
| Admin | Next.js-এ embedded (`/admin/*`) | আলাদা Vite app (`:5173`) |
| API prefix | `/api` | `/api/v1` |
| Permission | Simple admin role check | Full RBAC + permission strings |
| Analytics | Basic admin analytics | Full tracking + recommendation engine |
| Finance | — | Expenses + Investments module |
| Promoter | — | Promoter tracking system |
| OAuth | Google only | Google + Facebook |
| State | UserContext | ShopProvider (Context + localStorage) |

---

## ১৮. আর্কিটেকচার সিদ্ধান্ত

1. **তিনটি আলাদা app** — client, admin, server আলাদা deploy
2. **Permission-based admin** — RBAC + legacy permission dual system
3. **Recommendation engine** — user behavior scoring থেকে personalized products
4. **Fire-and-forget analytics** — `sendBeacon` দিয়ে UX block না করা
5. **Guest commerce** — login ছাড়াই checkout; login-এ order sync
6. **Server Actions caching** — Next.js `unstable_cache` nav data-র জন্য
7. **Bangladesh-specific** — BD address cascade, COD payment

---

*শেষ আপডেট: জুলাই ২০২৬ | Panzo E-commerce Platform*
