# SplasBD — Code Overview

[← Table of Contents](./README.md)

> **উদ্দেশ্য:** এই ডকুমেন্টটি SplasBD প্রজেক্টের পুরো আর্কিটেকচার, পেজ, ফাংশন, ক্লায়েন্ট ও সার্ভার কোডের মূল অংশ এবং তাদের মধ্যে সম্পর্ক ব্যাখ্যা করে। Git repo শেয়ার না করলেও এই ফাইল দিয়ে প্রজেক্ট বুঝতে সাহায্য করবে।

---

## ১. প্রজেক্ট কাঠামো (High-Level)

```
SplasBD/
├── client/          → Next.js 16 ফ্রন্টএন্ড (React 19, TypeScript)
├── server/          → Express 5 REST API (TypeScript, MongoDB)
└── script/          → ডাটাবেস seed স্ক্রিপ্ট (ঐচ্ছিক)
```

| অংশ | টেকনোলজি | পোর্ট (লোকাল) |
|-----|----------|---------------|
| **Client** | Next.js App Router, Tailwind CSS, Radix UI | `localhost:3000` |
| **Server** | Express 5, Mongoose, JWT, Passport Google OAuth | `localhost:5000` |
| **Database** | MongoDB | — |
| **Deploy** | Vercel (client + serverless API) | — |

> **নোট:** `client/` ও `server/` আলাদা git repository — দুটোই আলাদাভাবে deploy হয়।

---

## ২. Client ↔ Server কিভাবে যুক্ত

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
│  + Cloudinary (ছবি) + Nodemailer (ইমেইল) + Steadfast (কুরিয়ার) │
└─────────────────────────────────────────────────────────────────┘
```

**মূল নিয়ম:**
- Client সরাসরি Express API-তে কল করে — Next.js-এর নিজস্ব API route নেই
- Base URL: `NEXT_PUBLIC_API_URL` অথবা `http://localhost:5000/api`
- Auth: `httpOnly` cookie (`accessToken`, `refreshToken`) — `credentials: 'include'` দিয়ে পাঠানো হয়
- Guest user-ও কার্ট ও অর্ডার করতে পারে (session cookie)

---

## ৩. Client — ফোল্ডার ও দায়িত্ব

```
client/
├── app/                    # সব পেজ (Next.js App Router)
│   ├── (public)/           # স্টোরফ্রন্ট পেজ (URL-এ দেখা যায় না)
│   ├── admin/              # অ্যাডমিন প্যানেল
│   ├── layout.tsx          # রুট লেআউট (UserProvider, SEO)
│   ├── sitemap.ts          # ডায়নামিক sitemap
│   └── robots.ts           # robots.txt
├── components/
│   ├── common/             # Navbar, ProductCard, Cart, Footer...
│   ├── admin/              # AdminSidebar, RichTextEditor...
│   ├── Loading/            # Skeleton loaders
│   └── ui/                 # Button, Dialog, Table (shadcn-style)
├── contexts/
│   └── UserContext.tsx     # গ্লোবাল auth state
└── lib/
    ├── api.ts              # মূল HTTP client + cart/category API
    ├── authApi.ts          # Login, register, profile
    ├── orderApi.ts         # Order + Steadfast
    ├── adminApi.ts         # Dashboard analytics
    └── hooks/              # useInfiniteScroll ইত্যাদি
```

---

## ৪. Public পেজ — URL ও ফাইল ম্যাপিং

| URL | ফাইল | কাজ |
|-----|------|-----|
| `/` | `app/(public)/page.tsx` | হোম — Hero, Category, Promo, Products |
| `/products` | `app/(public)/products/page.tsx` | প্রোডাক্ট লিস্ট + ফিল্টার |
| `/products/[id]` | `app/(public)/products/[id]/page.tsx` | প্রোডাক্ট ডিটেইল |
| `/cart` | `app/(public)/cart/page.tsx` | শপিং কার্ট |
| `/checkout` | `app/(public)/checkout/page.tsx` | চেকআউট (বাংলাদেশি ঠিকানা) |
| `/order-confirmation/[id]` | `app/(public)/order-confirmation/[id]/page.tsx` | অর্ডার কনফার্মেশন |
| `/profile` | `app/(public)/profile/page.tsx` | প্রোফাইল, অর্ডার, রিভিউ |
| `/login`, `/register` | `app/(public)/login|register/page.tsx` | লগইন/রেজিস্ট্রেশন |
| `/auth/callback` | `app/(public)/auth/callback/page.tsx` | Google OAuth callback |
| `/guest-orders` | `app/(public)/guest-orders/page.tsx` | Guest অর্ডার খোঁজা |

---

## ৫. Admin পেজ — URL ও ফাইল ম্যাপিং

| URL | কাজ |
|-----|-----|
| `/admin` | ড্যাশবোর্ড (stats) |
| `/admin/products`, `/add`, `/[id]/edit` | প্রোডাক্ট CRUD |
| `/admin/orders`, `/orders/[id]` | অর্ডার ম্যানেজমেন্ট + Steadfast |
| `/admin/users`, `/users/roles` | ইউজার ও রোল |
| `/admin/banners` | হোমপেজ ব্যানার |
| `/admin/categories` | ক্যাটাগরি/সাবক্যাটাগরি |
| `/admin/promo-codes` | প্রোমো কোড |
| `/admin/shipping` | ঢাকা/ঢাকার বাইরে ডেলিভারি চার্জ |
| `/admin/analytics` | রেভিনিউ/প্রফিট অ্যানালিটিক্স |
| `/admin/settings` | সাইট সেটিংস |

**Admin সুরক্ষা:** `app/admin/layout.tsx` — পেজ লোডে `authApi.getProfile()` চেক করে; `role === ADMIN` না হলে login-এ redirect।

---

## ৬. Client কোড — মূল অংশ

### ৬.১ হোম পেজ (`app/(public)/page.tsx`)

হোম পেজ বিভিন্ন কম্পোনেন্ট একসাথে জোড়ে:

```tsx
// app/(public)/page.tsx — মূল সেকশন
<LeftSidebar />              // ক্যাটাগরি সাইডবার (ডেস্কটপ)
<HeroCarousel />             // ব্যানার ক্যারোসেল → GET /api/banners
<CategoryShowcase />         // ক্যাটাগরি → GET /api/categories
<DynamicPromotionalSections /> // প্রোমো সেকশন → GET /api/promotional-sections
<AllProducts />              // সব প্রোডাক্ট → GET /api/products
<TrustSection />             // ডেলিভারি/ওয়ারেন্টি ট্রাস্ট ব্লক
```

| কম্পোনেন্ট | API Endpoint | Server ফাইল |
|-----------|--------------|-------------|
| `HeroCarousel` | `GET /api/banners` | `features/banner/banner.route.ts` |
| `CategoryShowcase` | `GET /api/categories` | `features/category/category.route.ts` |
| `DynamicPromotionalSections` | `GET /api/promotional-sections` | `features/promotionalSection/` |
| `AllProducts` | `GET /api/products` | `features/product/product.route.ts` |

---

### ৬.২ মূল API Client (`lib/api.ts`)

সব API কলের কেন্দ্রবিন্দু। Token expire হলে auto refresh করে:

```typescript
// lib/api.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:5000/api';

export async function apiRequest<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
  // GET request → cache.ts দিয়ে cache
  // 401 error → refreshAccessToken() → retry
  // fetch(..., { credentials: 'include' }) — cookie পাঠায়
}
```

**সংযুক্ত Server:** যেকোনো `/api/*` route — `server/src/app.ts`-এ mount করা।

---

### ৬.৩ Auth API (`lib/authApi.ts`)

```typescript
// lib/authApi.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:5000/api';

// প্রতিটি request-এ credentials: 'include' — JWT cookie পাঠায়
authApi.login({ email, password })      → POST /api/auth/login
authApi.register({ name, email, password }) → POST /api/auth/register
authApi.getProfile()                    → GET  /api/auth/profile
authApi.refreshToken()                  → POST /api/auth/refresh-token
authApi.logout()                        → POST /api/auth/logout
```

**সংযুক্ত Server:** `server/src/features/auth/auth.route.ts` → `auth.controller.ts`

---

### ৬.৪ User Context (`contexts/UserContext.tsx`)

গ্লোবাল auth state — সব পেজে `UserProvider` দিয়ে wrap:

```typescript
// contexts/UserContext.tsx
export function UserProvider({ children }) {
  // Mount-এ authApi.getProfile() কল
  // Login-এর পর guest order link: orderApi.linkGuestOrders()
  // logout → cookie clear + localStorage clear
}
```

| Client ফাংশন | Server Endpoint | ব্যাখ্যা |
|-------------|-----------------|---------|
| `refreshUser()` | `GET /api/auth/profile` | ইউজার ডেটা লোড |
| `logout()` | `POST /api/auth/logout` | সেশন শেষ |
| Guest order link | `POST /api/orders/link-guest-orders` | Login-এর পর guest অর্ডার যুক্ত |

---

### ৬.৫ Order API (`lib/orderApi.ts`)

```typescript
// lib/orderApi.ts
orderApi.createOrder(data)           → POST /api/orders
orderApi.getMyOrders()               → GET  /api/orders/my-orders
orderApi.linkGuestOrders(orderIds)   → POST /api/orders/link-guest-orders
orderApi.createSteadfastOrder(id)    → POST /api/orders/:id/steadfast  (Admin)
```

**সংযুক্ত Server:** `server/src/features/order/order.controller.ts`

---

## ৭. Server — ফোল্ডার ও দায়িত্ব

```
server/src/
├── app.ts                  # Express setup + সব route mount
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
│   └── ... (১৭টি feature)
└── utils/
    ├── jwt.ts              # Token generate/verify
    ├── email.ts            # Verification email
    └── cloudinary.ts       # Image upload
```

**প্রতিটি feature-এর প্যাটার্ন:**
```
features/product/
├── product.route.ts      # URL + middleware define
├── product.controller.ts # Request handle, validation
├── product.model.ts      # Database query
└── product.schema.ts     # Mongoose schema
```

---

## ৮. Server কোড — মূল অংশ

### ৮.১ Route Mounting (`server/src/app.ts`)

সব API `/api` prefix-এ mount:

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
// ... আরও
```

| Route Prefix | Client API ফাইল | মূল কাজ |
|-------------|-----------------|---------|
| `/api/auth` | `lib/authApi.ts` | Login, register, OAuth |
| `/api/products` | `lib/api.ts` | Product CRUD |
| `/api/cart` | `lib/api.ts` | Session-based cart |
| `/api/orders` | `lib/orderApi.ts` | Order create/manage |
| `/api/admin` | `lib/adminApi.ts` | Dashboard, analytics |

---

### ৮.২ Auth Routes (`features/auth/auth.route.ts`)

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

### ৮.৩ Auth Middleware (`middleware/auth.middleware.ts`)

```typescript
// Cookie থেকে accessToken নেয়
export const authenticateUser = async (req, res, next) => {
  const token = req.cookies?.accessToken || req.headers.authorization?.replace('Bearer ', '');
  const decoded = verifyAccessToken(token);
  const user = await UserModel.findById(decoded.id);
  req.user = decoded;
  next();
};

export const requireAdmin = ...  // role === 'ADMIN' চেক
export const optionalAuth = ...  // logged in থাকলে user attach, না থাকলে চলতে দেয়
```

| Middleware | কোথায় ব্যবহার | Client-এর কোন কল |
|-----------|---------------|-----------------|
| `authenticateUser` | Profile, my-orders | `authApi.getProfile()` |
| `requireAdmin` | Admin CRUD routes | Admin panel API calls |
| `optionalAuth` | Product detail, create order | Guest + logged-in দুটোই |

---

### ৮.৪ Order Create (`features/order/order.controller.ts`)

```typescript
static create = asyncHandler(async (req: AuthRequest, res: Response) => {
  const orderData: CreateOrderDto = req.body;
  const userId = req.user?.id || null;  // Guest হলে null

  // Stock validate
  for (const item of orderData.items) {
    const product = await ProductModel.findById(item.product);
    // size/stock check...
  }

  const order = await OrderModel.create(orderData, userId);
  return ResponseHandler.success(res, order, 'Order created successfully', 201);
});
```

**Client থেকে কল:** `checkout/page.tsx` → `orderApi.createOrder()` → `POST /api/orders`

---

## ৯. সম্পূর্ণ Auth Flow

```
১. Register
   Client: register/page.tsx → authApi.register()
   Server: POST /api/auth/register → email verification পাঠায়

২. Email Verify
   Client: verify-email/page.tsx → authApi.verifyEmail(token)
   Server: POST /api/auth/verify-email

৩. Login
   Client: login/page.tsx → authApi.login()
   Server: POST /api/auth/login → accessToken + refreshToken cookie set
   Client: UserContext.refreshUser() → profile load

৪. Token Expire (401)
   Client: apiRequest() → authApi.refreshToken() → retry
   Server: POST /api/auth/refresh-token → নতুন accessToken cookie

৫. Google OAuth
   Client: "Login with Google" → window.location = /api/auth/google
   Server: Google → callback → cookie set → redirect /auth/callback
   Client: auth/callback/page.tsx → refreshUser()

৬. Admin Access
   Client: admin/layout.tsx → getProfile() → role === ADMIN?
   Server: requireAdmin middleware সব admin route-এ
```

---

## ৯.১ Checkout Flow (Guest + Logged-in)

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

## ১০. API Endpoint সংক্ষিপ্ত তালিকা

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

*(সম্পূর্ণ endpoint তালিকা server `GET /` response-এও পাওয়া যায়)*

---

## ১১. Feature Module ম্যাপিং (Client ↔ Server)

| Feature | Client ফাইল/কম্পোনেন্ট | Server Feature | MongoDB Model |
|---------|------------------------|----------------|---------------|
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

## ১২. Environment Variables

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

## ১৩. চালানোর নিয়ম (Local)

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

## ১৪. গুরুত্বপূর্ণ ফাইল রেফারেন্স

| ফাইল | ভূমিকা |
|------|--------|
| `client/lib/api.ts` | মূল HTTP client, caching, token refresh |
| `client/lib/authApi.ts` | সব auth endpoint |
| `client/contexts/UserContext.tsx` | গ্লোবাল user state |
| `client/app/admin/layout.tsx` | Admin route protection |
| `client/app/(public)/page.tsx` | হোম পেজ composition |
| `server/src/app.ts` | Express app + route mounting |
| `server/src/middleware/auth.middleware.ts` | JWT verification |
| `server/src/features/auth/auth.route.ts` | Auth endpoints |
| `server/src/features/order/order.controller.ts` | Order logic |
| `server/api/index.ts` | Vercel serverless entry |

---

## ১৫. আর্কিটেকচার সিদ্ধান্ত (জানা দরকার)

1. **আলাদা git repo** — client ও server আলাদা deploy
2. **Cookie-based JWT** — localStorage-এ token নেই, শুধু `userSession` flag
3. **Guest commerce** — login ছাড়াই cart/checkout; login-এ অর্ডার link
4. **Feature-sliced server** — প্রতিটি domain আলাদা folder (route → controller → model)
5. **বাংলাদেশ-স্পেসিফিক** — Dhaka/outside shipping, Steadfast courier, বাংলা font
6. **Client-side cache** — GET response `lib/cache.ts`-এ cache; mutation-এ invalidate

---

*শেষ আপডেট: জুলাই ২০২৬ | SplasBD E-commerce Platform*
