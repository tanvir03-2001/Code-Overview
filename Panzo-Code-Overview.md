# Panzo — Code Overview

[← Table of Contents](./README.md)

> **Purpose:** This document explains the full architecture of the Panzo E-commerce project — pages, functions, core client/admin/server code, and how they connect. Use this file to understand the project without sharing the git repository.

---

## 1. Project Structure (High-Level)

```
Panzo/
├── client/          → Next.js 16 storefront (React 19, port 3000)
├── admin/           → Vite + React admin panel (port 5173)
├── server/          → Express 4 REST API (port 5000)
└── CASE_STUDY.md    → Existing architecture doc (Bengali)
```

| Layer | Technology | Port (local) |
|-------|------------|--------------|
| **Client** | Next.js App Router, Tailwind CSS 4, Swiper | `localhost:3000` |
| **Admin** | Vite 7, React Router 7, Radix UI, Recharts | `localhost:5173` |
| **Server** | Express 4, Mongoose 8, JWT, Passport OAuth | `localhost:5000` |
| **Database** | MongoDB (`panzo-ecommerce`) | — |

> **Note:** `client/`, `admin/`, and `server/` are **three separate git repositories**. Unlike SplasBD, admin is not embedded in the client — it is a standalone Vite app.

---

## 2. How Client ↔ Admin ↔ Server Connect

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

**Key rules:**
- Both Client and Admin call the Express API directly — no BFF layer
- Base URL: `http://localhost:5000/api/v1`
- Auth: `httpOnly` cookies (`accessToken`, `refreshToken`) via `credentials: 'include'`
- Guest users can use cart and checkout
- Admin panel uses permission-based route guards (RBAC)

---

## 3. Client — Folders and Responsibilities

```
client/src/
├── app/                    # Next.js App Router pages
│   ├── page.tsx            # Home
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
│   └── api.ts              # Core API client (ApiClient class)
├── hooks/                  # useInfiniteProducts, useSearch
├── actions/
│   └── navActions.ts       # Server Actions (cached nav data)
└── lib/                    # metadata, getProduct
```

---

## 4. Client Pages — URL and File Mapping

| URL | File | Purpose |
|-----|------|---------|
| `/` | `app/page.tsx` | Home — hero carousel, categories, product grids |
| `/products` | `app/products/page.tsx` | Product list + filters |
| `/product/[slug]` | `app/product/[slug]/page.tsx` | Product detail (color/size variants) |
| `/category/[slug]` | `app/category/[slug]/page.tsx` | Products by category |
| `/brand/[slug]` | `app/brand/[slug]/page.tsx` | Products by brand |
| `/categories` | `app/categories/page.tsx` | All categories |
| `/search` | `app/search/page.tsx` | Search + autocomplete |
| `/cart` | `app/cart/page.tsx` | Shopping cart |
| `/checkout` | `app/checkout/page.tsx` | Checkout (BD address, COD) |
| `/order-success` | `app/order-success/page.tsx` | Order confirmation |
| `/orders` | `app/orders/page.tsx` | Order history |
| `/orders/[orderId]` | `app/orders/[orderId]/page.tsx` | Order detail |
| `/wishlist` | `app/wishlist/page.tsx` | Saved products |
| `/account` | `app/account/page.tsx` | Profile and addresses |
| `/login`, `/signup` | `app/login/`, `app/signup/` | Auth |
| `/verify-email` | `app/verify-email/page.tsx` | Email verification |

---

## 5. Admin Pages — URL and File Mapping

Admin is a separate Vite app — routes defined with React Router:

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

**Admin protection:** `PermissionRoute` component — checks required permission on each route:

```tsx
// admin/src/App.tsx
<Route path="/" element={
  <PermissionRoute requiredPermission="dashboard.view">
    <Dashboard />
  </PermissionRoute>
} />
```

---

## 6. Client Code — Core Parts

### 6.1 API Client (`client/src/utils/api.ts`)

Central hub for all API calls — `ApiClient` class:

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

### 6.2 ShopProvider (`components/ShopProvider.tsx`)

State managed with React Context + localStorage — no Redux:

```typescript
// ShopProvider — cart, wishlist, user, addresses
// localStorage key: 'panzo_shop_v1'
// Cart add/remove, wishlist toggle, user login state
```

| State | Storage | Server Sync |
|-------|---------|-------------|
| Cart | localStorage | `POST /api/v1/orders` on checkout |
| Wishlist | localStorage | Client-only |
| User | Cookie + context | `GET /api/v1/users/profile` |
| Addresses | localStorage | Account page |

---

### 6.3 Server Actions (`actions/navActions.ts`)

Next.js Server Actions + `unstable_cache` for nav data:

```typescript
'use server';
import { unstable_cache } from 'next/cache';

export async function getNavData() {
  const getCachedCategories = unstable_cache(
    async () => categoryApi.getAllCategories(),
    ['nav-categories'],
    { revalidate: 900, tags: ['nav-categories'] }
  );
  // returns categories + brands
}
```

**Connected Server:** `GET /api/v1/categories`, `GET /api/v1/brands`

---

### 6.4 Analytics Tracking (`utils/analyticsTracking.ts`)

Fire-and-forget tracking — does not block UX:

```typescript
// behaviorApi.trackBehavior() → navigator.sendBeacon()
// POST /api/v1/behavior/track
// Session, page-view, click-event, search tracking
```

---

## 7. Admin Code — Core Parts

### 7.1 Admin API Client (`admin/src/lib/api.ts`)

Same pattern as client + **401 → auto refresh → retry**:

```typescript
// admin/src/lib/api.ts
// ApiClient with token refresh on 401
// VITE_API_URL = http://localhost:5000/api/v1
```

### 7.2 Auth & Permission Context

```
admin/src/contexts/
├── AuthContext.tsx         # Admin login state, token refresh
└── PermissionContext.tsx   # User permissions load

admin/src/components/
├── ProtectedRoute.tsx      # Login required
└── PermissionRoute.tsx     # Specific permission required
```

---

## 8. Server — Folders and Responsibilities

```
server/src/
├── app.ts                  # Express setup + middleware
├── server.ts               # Bootstrap: DB connect, super user init
├── routes/index.ts         # All route registry
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

**Pattern for each feature:**
```
features/product/
├── product.route.ts      # URL + middleware
├── product.controller.ts # Request handling
├── product.service.ts    # Business logic
└── product.model.ts      # Mongoose model
```

---

## 9. Server Code — Core Parts

### 9.1 Route Mounting (`server/src/routes/index.ts`)

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

### 9.2 Feature Controller Pattern

```typescript
// features/product/product.controller.ts
export const getProducts = asyncHandler(async (req, res) => {
  const result = await productService.getAll(true, { page, limit, search, category, brand });
  sendSuccess(res, 200, 'Products fetched successfully.', result);
});
```

---

### 9.3 Recommendation Engine

```typescript
// algorithm/product_view_algorithm/recommendationEngine.ts
// User behavior scoring → personalized product recommendations
// Client: productApi.getRecommendedProducts() → GET /api/v1/products/recommended
```

---

## 10. API Endpoint Summary

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
| POST | `/sync-guest-orders` | Auth | Link guest orders on login |
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

## 11. Feature Module Mapping

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

## 12. Auth Flow

```
1. Signup
   Client: signup/page.tsx → userApi.signup()
   Server: POST /api/v1/users/signup → verification email

2. Login
   Client: login/page.tsx → userApi.login()
   Server: POST /api/v1/users/login → JWT cookies set
   Client: ShopProvider user state update

3. Google/Facebook OAuth
   Client: redirect → /api/v1/users/auth/google
   Server: Passport OAuth → callback → cookies → redirect client

4. Admin Login
   Admin: login page → same /users/login endpoint
   Admin: AuthContext → PermissionContext load permissions
   Admin: PermissionRoute guards each page

5. Token Refresh (401)
   Admin api.ts: auto POST /users/refresh-token → retry
   Client: similar pattern in ApiClient
```

---

## 13. Checkout Flow

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
                           └── Guest order → sync-guest-orders on login
```

---

## 14. Environment Variables

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

## 15. How to Run Locally

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

## 16. Key File Reference

| File | Role |
|------|------|
| `client/src/utils/api.ts` | Core API client + domain APIs |
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

## 17. SplasBD vs Panzo — Key Differences

| Topic | SplasBD | Panzo |
|-------|---------|-------|
| Admin | Embedded in Next.js (`/admin/*`) | Separate Vite app (`:5173`) |
| API prefix | `/api` | `/api/v1` |
| Permission | Simple admin role check | Full RBAC + permission strings |
| Analytics | Basic admin analytics | Full tracking + recommendation engine |
| Finance | — | Expenses + Investments module |
| Promoter | — | Promoter tracking system |
| OAuth | Google only | Google + Facebook |
| State | UserContext | ShopProvider (Context + localStorage) |

---

## 18. Code Examples

Real code from the project — shows TypeScript patterns used across client, admin, and server.

### Example 1 — Reusable API client class (`client/src/utils/api.ts`)

```typescript
class ApiClient {
  private baseURL: string;

  constructor(baseURL: string) {
    this.baseURL = baseURL;
  }

  private async request<T>(
    endpoint: string,
    options: NextFetchRequestInit = {}
  ): Promise<T> {
    const url = `${this.baseURL}${endpoint}`;
    const isServer = typeof window === 'undefined';

    const config: NextFetchRequestInit = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      credentials: 'include', // Include cookies for authentication
      ...(isServer && {
        ...(options.next?.revalidate !== undefined ? {
          next: options.next,
          cache: options.cache || 'force-cache',
        } : {
          cache: options.cache || 'no-store',
          next: options.next || { revalidate: false },
        }),
      }),
      ...(!isServer && {
        cache: 'no-store',
      }),
    };

    const response = await fetch(url, config);
    const data: ApiResponse<T> = await response.json();

    if (!response.ok) {
      const error = new Error(data.message || `HTTP error! status: ${response.status}`);
      (error as any).response = { data };
      (error as any).status = response.status;
      throw error;
    }

    return data.data as T;
  }
}
```

### Example 2 — Server Actions with Next.js cache (`client/src/actions/navActions.ts`)

```typescript
'use server';

import { unstable_cache } from 'next/cache';
import { categoryApi, brandApi, type Category, type Brand } from '@/utils/api';

export async function getNavData(): Promise<{
  categories: Category[];
  brands: Brand[];
}> {
  const getCachedCategories = unstable_cache(
    async () => {
      return await categoryApi.getAllCategories();
    },
    ['nav-categories'],
    {
      revalidate: 900, // 15 minutes
      tags: ['nav-categories'],
    }
  );

  const [categories, brands] = await Promise.all([
    getCachedCategories(),
    getCachedBrands(),
  ]);

  return { categories, brands };
}
```

### Example 3 — Permission-based route guard (`admin/src/components/PermissionRoute.tsx`)

```typescript
export function PermissionRoute({
  children,
  requiredPermission,
  fallback
}: PermissionRouteProps) {
  const { hasPermission, loading } = usePermissions();
  const { user } = useAuth();

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-muted-foreground">Loading permissions...</div>
      </div>
    );
  }

  let hasAccess = hasPermission(requiredPermission);

  // Support both new format (dashboard.view) and old format (view_dashboard)
  if (!hasAccess) {
    if (requiredPermission.includes('.')) {
      const [module, action] = requiredPermission.split('.');
      hasAccess = hasPermission(`${action}_${module}`);
    } else if (requiredPermission.includes('_')) {
      const parts = requiredPermission.split('_');
      if (parts.length >= 2) {
        const action = parts[0];
        const module = parts.slice(1).join('_');
        hasAccess = hasPermission(`${module}.${action}`);
      }
    }
  }

  if (!hasAccess) {
    // Show access denied UI...
  }

  return <>{children}</>;
}
```

### Example 4 — Fire-and-forget analytics tracking (`client/src/utils/api.ts`)

```typescript
export const behaviorApi = {
  trackBehavior: async (payload: TrackBehaviorPayload): Promise<void> => {
    // Use sendBeacon for non-blocking request (fire-and-forget)
    if (typeof window !== 'undefined' && navigator.sendBeacon) {
      const url = `${API_BASE_URL}/behavior/track`;
      const data = JSON.stringify(payload);
      const blob = new Blob([data], { type: 'application/json' });
      navigator.sendBeacon(url, blob);
    } else {
      fetch(`${API_BASE_URL}/behavior/track`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
        keepalive: true,
      }).catch(() => {
        // Silently fail - tracking should not affect user experience
      });
    }
  },
};
```

---

## 19. Architecture Decisions

1. **Three separate apps** — client, admin, server deployed independently
2. **Permission-based admin** — RBAC + legacy permission dual system
3. **Recommendation engine** — personalized products from user behavior scoring
4. **Fire-and-forget analytics** — `sendBeacon` to avoid blocking UX
5. **Guest commerce** — checkout without login; order sync on login
6. **Server Actions caching** — Next.js `unstable_cache` for nav data
7. **Bangladesh-specific** — BD address cascade, COD payment

---

*Last updated: July 2026 | Panzo E-commerce Platform*
