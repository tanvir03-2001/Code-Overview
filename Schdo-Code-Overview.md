# Schdo (SchdoSocial) — Code Overview

[← সূচিপত্র](./README.md)

> **উদ্দেশ্য:** SchdoSocial প্রজেক্টের পুরো আর্কিটেকচার, পেজ, ফাংশন, client/server কোডের মূল অংশ এবং তাদের মধ্যে সম্পর্ক ব্যাখ্যা করে। Git repo শেয়ার না করলেও এই ফাইল দিয়ে প্রজেক্ট বুঝতে সাহায্য করবে।

---

## ১. প্রজেক্ট কাঠামো (High-Level)

```
Schdo/
├── Client/          → Next.js 16 ফ্রন্টএন্ড (React 19, port 3000)
└── Server/          → Express 5 API (port 5000) + Worker process
```

| অংশ | টেকনোলজি | পোর্ট (লোকাল) |
|-----|----------|---------------|
| **Client** | Next.js App Router, Tailwind CSS 4, Radix UI, TanStack Query | `localhost:3000` |
| **Server** | Express 5, Mongoose 9, Agenda job queue, Zod | `localhost:5000` |
| **Worker** | Standalone Agenda worker (`npm run dev:worker`) | — |
| **Database** | MongoDB | — |
| **Integrations** | Facebook Graph API, TikTok API, Cloudinary, Google OAuth | — |

> **প্রজেক্টের উদ্দেশ্য:** Facebook Page ও TikTok account-এ social media post schedule, publish ও manage করার SaaS platform (**SchdoSocial**)।

> **নোট:** `Client/` ও `Server/` আলাদা git repository।

---

## ২. Client ↔ Server কিভাবে যুক্ত

```
┌─────────────────────────────────────────────────────────────────┐
│  Browser (User)                                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  CLIENT — Next.js (Schdo/Client/)                              │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────┐   │
│  │ Pages       │ → │ lib/*.ts     │ → │ apiFetch + cookies  │   │
│  │ (app/)      │   │ api-client   │   │ credentials:include │   │
│  │ Providers   │   │ auth, posts  │   └──────────┬──────────┘   │
│  └─────────────┘   └──────────────┘              │              │
└────────────────────────────────────────────────────┼──────────────┘
                                                     │ HTTP / SSE
┌────────────────────────────────────────────────────▼──────────────┐
│  SERVER — Express (Schdo/Server/)                                │
│  /api/v1/* → Controller → Service → MongoDB                     │
│  + Agenda Queue → Facebook Graph API / TikTok API publish       │
│  + Cloudinary (media) + SSE streams (notifications, admin)      │
└─────────────────────────────────────────────────────────────────┘
```

**মূল নিয়ম:**
- Client সরাসরি Express API-তে কল করে
- Base URL: `NEXT_PUBLIC_API_URL` → `http://localhost:5000/api/v1`
- Auth: `httpOnly` cookie — `apiFetch()` 401-এ auto refresh
- Real-time: SSE streams (notifications, admin monitoring)
- Publishing: Server-side Agenda job queue + poll worker

---

## ৩. Client — ফোল্ডার ও দায়িত্ব

```
Client/src/
├── app/
│   ├── (auth)/             # Login, register, forgot-password
│   ├── (dashboard)/        # User dashboard (guarded)
│   ├── (admin)/            # Admin panel (AdminGuard)
│   ├── privacy/, terms/, data-deletion/
│   ├── accept-terms/
│   ├── layout.tsx          # Root layout + Providers
│   └── page.tsx            # Landing/marketing page
├── components/
│   ├── auth/               # AuthGuard, TermsGuard, OnboardingGuard
│   ├── dashboard/          # Post form, posts table, scheduling UI
│   ├── admin/              # Cloudinary reports, server capacity
│   ├── landing/            # Marketing sections
│   ├── layout/             # Sidebar, header, page selector
│   └── ui/                 # Radix-based design system
├── lib/
│   ├── api-client.ts       # মূল HTTP client
│   ├── auth.ts, posts.ts, facebook-pages.ts, tiktok-accounts.ts
│   ├── scheduling-models.ts, notifications.ts, admin.ts
│   └── ... (35 modules)
├── providers/
│   ├── auth-provider.tsx
│   ├── notifications-provider.tsx  # SSE stream
│   ├── bulk-upload-queue-provider.tsx
│   └── selected-page-provider.tsx
├── hooks/                  # 11 custom hooks
├── i18n/                   # en.ts, bn.ts (English + Bengali)
└── types/                  # Shared TypeScript interfaces
```

---

## ৪. Route Groups ও Guard Chain

| Group | Layout | Guards |
|-------|--------|--------|
| `(auth)` | Public auth layout | — |
| `(dashboard)` | Dashboard layout | `AuthGuard` → `TermsGuard` → `OnboardingGuard` |
| `(admin)` | Admin layout | `AdminGuard` → `TermsGuard` |

```tsx
// app/(dashboard)/dashboard-layout-client.tsx
<AuthGuard>
  <TermsGuard>
    <OnboardingGuard>
      <BulkUploadQueueProvider>
        <PostDeleteQueueProvider>
          {children}
        </PostDeleteQueueProvider>
      </BulkUploadQueueProvider>
    </OnboardingGuard>
  </TermsGuard>
</AuthGuard>
```

---

## ৫. Public & Auth পেজ

| URL | ফাইল | কাজ |
|-----|------|-----|
| `/` | `app/page.tsx` | Landing/marketing page |
| `/privacy` | `app/privacy/page.tsx` | Privacy policy |
| `/terms` | `app/terms/page.tsx` | Terms of service |
| `/data-deletion` | `app/data-deletion/page.tsx` | Meta data deletion |
| `/accept-terms` | `app/accept-terms/page.tsx` | Terms acceptance |
| `/login` | `app/(auth)/login/page.tsx` | Login |
| `/register` | `app/(auth)/register/page.tsx` | Registration |
| `/forgot-password` | `app/(auth)/forgot-password/page.tsx` | Password reset |
| `/reset-password` | `app/(auth)/reset-password/page.tsx` | Reset form |
| `/auth/callback` | `app/(auth)/auth/callback/page.tsx` | OAuth return handler |

---

## ৬. Dashboard পেজ (User)

| URL | কাজ |
|-----|-----|
| `/onboarding` | First-time setup wizard |
| `/dashboard` | Overview / analytics home |
| `/dashboard/posts` | Post list |
| `/dashboard/posts/create` | Create post |
| `/dashboard/posts/[id]/edit` | Edit post |
| `/dashboard/posts/[id]/preview` | Post preview |
| `/dashboard/scheduled` | Calendar / timeline of scheduled posts |
| `/dashboard/scheduling-models` | Scheduling template list |
| `/dashboard/scheduling-models/create` | Create template |
| `/dashboard/scheduling-models/[id]/edit` | Edit template |
| `/dashboard/pages` | Connected Facebook Pages + TikTok accounts |
| `/dashboard/pages/tiktok-callback` | TikTok OAuth popup callback |
| `/dashboard/notifications` | Notification inbox |
| `/dashboard/activity` | User activity log |
| `/dashboard/settings` | Account settings, sessions |

---

## ৭. Admin পেজ

| URL | কাজ |
|-----|-----|
| `/admin` | Admin overview |
| `/admin/analytics` | Platform analytics |
| `/admin/server` | Server capacity metrics (SSE) |
| `/admin/media-pools` | Cloudinary media database pools |
| `/admin/media-pools/[id]` | Pool detail + report |
| `/admin/cloudinary` | Cloudinary usage report |
| `/admin/worker` | Scheduled-post worker monitor (SSE) |
| `/admin/users` | User management |
| `/admin/users/[id]` | User detail |
| `/admin/posts` | All posts (retry/delete) |
| `/admin/pages` | All Facebook pages |
| `/admin/activity` | Platform activity |
| `/admin/audit` | Admin audit log |
| `/admin/settings` | System settings |

**Admin সুরক্ষা:** `AdminGuard` — role `admin` বা `super_user` required।

---

## ৮. Client কোড — মূল অংশ

### ৮.১ API Client (`lib/api-client.ts`)

সব HTTP call-এর কেন্দ্রবিন্দু:

```typescript
// lib/api-client.ts
export const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:5000/api/v1";

export const apiFetch = async <T>(path: string, options: RequestInit = {}, retry = true): Promise<T> => {
  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    credentials: "include",  // JWT cookie পাঠায়
  });
  // 401 → refreshSession() → retry once
  return body.data;
};

export const apiUpload = ...       // Multipart upload with progress
export const apiUploadWithProgress = ...
```

| Client Module | Server Endpoint | কাজ |
|--------------|-----------------|-----|
| `lib/auth.ts` | `/auth/*` | Login, register, OAuth, sessions |
| `lib/posts.ts` | `/posts/*` | Post CRUD, analytics, bulk delete |
| `lib/facebook-pages.ts` | `/facebook-pages/*` | FB page connect/manage |
| `lib/tiktok-accounts.ts` | `/tiktok-accounts/*` | TikTok account connect |
| `lib/scheduling-models.ts` | `/scheduling-models/*` | Scheduling templates |
| `lib/notifications.ts` | `/notifications/*` | In-app notifications |
| `lib/uploads.ts` | `/uploads/images` | Media upload |
| `lib/admin.ts` | `/admin/*` | Admin operations |

---

### ৮.২ Auth Provider (`providers/auth-provider.tsx`)

```typescript
// Mount-এ GET /auth/me → user state set
// Login/register/OAuth → cookies set → refresh user
// logout → POST /auth/logout → clear state
```

---

### ৮.৩ Notifications Provider (`providers/notifications-provider.tsx`)

```typescript
// SSE stream: GET /notifications/stream
// Real-time notification push
// Unread count: GET /notifications/unread-count
```

---

### ৮.৪ Post Form (`components/dashboard/post-form/post-form.tsx`)

মূল post creation/editing UI:
- Text, image, video, reel, story support
- Facebook page selector
- Schedule date/time picker
- Place check-in
- Media upload with progress

**Client flow:**
```
post-form.tsx → lib/posts.ts → POST /api/v1/posts
              → lib/uploads.ts → POST /api/v1/uploads/images
              → Server schedules via Agenda queue
```

---

### ৮.৫ Bulk Upload Queue (`providers/bulk-upload-queue-provider.tsx`)

Background bulk post upload — multiple posts একসাথে upload:
```
BulkUploadQueueProvider → sequential upload → progress tracking
→ GET /posts/upload-progress/:token
```

---

## ৯. Server — ফোল্ডার ও দায়িত্ব

```
Server/src/
├── app.ts                  # Express setup + middleware
├── server.ts               # Bootstrap: DB, queue, scheduler worker
├── worker.ts               # Standalone Agenda worker
├── routes/
│   └── v1.routes.ts        # সব route mount
├── config/
│   ├── env.ts              # Centralized env
│   ├── database.ts         # MongoDB
│   ├── passport.ts         # Google/Facebook OAuth
│   └── cloudinary.ts
├── features/               # Feature modules
│   ├── auth/
│   ├── posts/              # Core post CRUD + publish
│   ├── facebook-pages/
│   ├── tiktok-accounts/
│   ├── scheduling-models/
│   ├── notifications/
│   ├── uploads/
│   ├── admin/
│   ├── activity/
│   └── legal/
├── middleware/
│   ├── authenticate.ts     # JWT from cookie/Bearer
│   ├── authorize.ts        # Role-based access
│   └── validate.ts         # Zod validation
├── models/                 # 15 Mongoose models
├── queue/
│   ├── agenda.client.ts
│   └── post-publish.queue.ts
└── utils/
    ├── facebook-graph/     # FB Graph API client
    ├── tiktok-api/         # TikTok OAuth + API
    └── cloudinary-upload.ts
```

**Feature pattern:**
```
features/posts/
├── posts.route.ts        # Express router
├── posts.controller.ts   # HTTP handlers
├── posts.service.ts      # Business logic
├── posts.schema.ts       # Zod validation
└── posts.scheduler.service.ts  # Poll-based publisher
```

---

## ১০. Server কোড — মূল অংশ

### ১০.১ Route Mounting (`routes/v1.routes.ts`)

```typescript
// routes/v1.routes.ts
router.use("/health", healthRoutes);
router.use("/auth", authRoutes);
router.use("/facebook-pages", facebookPagesRoutes);
router.use("/tiktok-accounts", tiktokAccountsRoutes);
router.use("/posts", postsRoutes);
router.use("/uploads", uploadsRoutes);
router.use("/notifications", notificationsRoutes);
router.use("/scheduling-models", schedulingModelsRoutes);
router.use("/activity", activityRoutes);
router.use("/legal", legalRoutes);
router.use("/admin", adminRoutes);
```

---

### ১০.২ Server Bootstrap (`server.ts`)

```typescript
// server.ts
// 1. Connect MongoDB
// 2. Bootstrap super-admin user
// 3. Initialize Cloudinary media pool
// 4. Start Agenda job queue
// 5. Start poll-based scheduled post worker
// 6. Listen on PORT (5000)
```

---

### ১০.৩ Scheduled Post Publishing

```
User schedules post → POST /api/v1/posts (status: scheduled)
                   → Agenda job created (post-publish.queue.ts)
                   → At scheduled time: posts.scheduler.service.ts
                   → Facebook Graph API publish (utils/facebook-graph/)
                   → Post status → published / failed
                   → Notification sent via SSE
```

---

### ১০.৪ Zod Validation Middleware

```typescript
// middleware/validate.ts
export const validate = (schema: ZodType, source: "body" | "query" | "params") =>
  (req, res, next) => {
    const result = schema.safeParse(req[source]);
    if (!result.success) {
      return next(new AppError("Validation failed", 400, result.error.flatten()));
    }
    req[source] = result.data;
    next();
  };
```

---

## ১১. API Endpoint সংক্ষিপ্ত তালিকা

**Base:** `/api/v1`

### Auth — `/api/v1/auth`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| POST | `/register` | Public | Register page |
| POST | `/login` | Public | Login page |
| POST | `/refresh` | Cookie | apiFetch auto |
| POST | `/logout` | Public | Settings |
| GET | `/me` | Yes | AuthProvider bootstrap |
| POST | `/complete-onboarding` | Yes | Onboarding wizard |
| POST | `/accept-terms` | Yes | TermsGuard |
| GET | `/google`, `/facebook` | Public | OAuth redirect |

### Posts — `/api/v1/posts`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Yes | Posts list |
| POST | `/` | Yes | Create/schedule post |
| GET | `/:id` | Yes | Post detail/edit |
| PATCH | `/:id` | Yes | Update post |
| DELETE | `/:id` | Yes | Delete post |
| GET | `/analytics` | Yes | Dashboard analytics |
| POST | `/bulk-delete` | Yes | Bulk delete queue |
| POST | `/sync` | Yes | Sync from Facebook |

### Facebook Pages — `/api/v1/facebook-pages`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Yes | Connected pages list |
| POST | `/connect-url` | Yes | OAuth popup URL |
| PATCH | `/:id/connect` | Yes | Connect page |
| PATCH | `/:id/disconnect` | Yes | Disconnect page |
| DELETE | `/:id` | Yes | Remove page |

### TikTok Accounts — `/api/v1/tiktok-accounts`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Yes | TikTok accounts list |
| POST | `/connect-url` | Yes | OAuth popup URL |
| PATCH | `/:id/connect` | Yes | Connect account |
| DELETE | `/:id` | Yes | Remove account |

### Scheduling Models — `/api/v1/scheduling-models`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/` | Yes | Template list |
| POST | `/` | Yes | Create template |
| PATCH | `/:id` | Yes | Update template |
| PATCH | `/:id/toggle-active` | Yes | Enable/disable |
| DELETE | `/:id` | Yes | Delete template |

### Notifications — `/api/v1/notifications`
| Method | Endpoint | Auth | Client |
|--------|----------|------|--------|
| GET | `/stream` | Yes | SSE real-time |
| GET | `/unread-count` | Yes | Badge count |
| GET | `/` | Yes | Notification list |
| POST | `/read-all` | Yes | Mark all read |

### Admin — `/api/v1/admin` (admin/super_user only)
| Method | Endpoint | Client |
|--------|----------|--------|
| GET | `/stats` | Admin overview |
| GET | `/system/stream` | SSE server metrics |
| GET | `/worker/stream` | SSE worker monitor |
| GET | `/users`, `/users/:id` | User management |
| GET | `/posts` | All posts |
| POST | `/posts/:id/retry` | Retry failed publish |
| GET | `/media-pools` | Cloudinary pools |
| POST | `/process-scheduled` | Manual scheduler trigger |

---

## ১২. Feature Module ম্যাপিং

| Feature | Client | Server | External |
|---------|--------|--------|----------|
| **Auth** | `auth-provider`, login/register | `features/auth/` | Google/Facebook OAuth |
| **Facebook Pages** | `facebook-pages.ts`, pages dashboard | `features/facebook-pages/` | Facebook Graph API |
| **TikTok** | `tiktok-accounts.ts`, tiktok-callback | `features/tiktok-accounts/` | TikTok Login Kit |
| **Posts** | `post-form`, posts table | `features/posts/` | FB/TikTok publish |
| **Scheduling** | scheduling-models, calendar | `scheduling-models/` + Agenda queue | — |
| **Media Upload** | upload with progress | `features/uploads/` + Cloudinary | Cloudinary |
| **Notifications** | SSE provider | `features/notifications/` | SSE stream |
| **Admin** | admin pages | `features/admin/` | SSE monitoring |
| **Legal** | privacy, terms, data-deletion | `features/legal/` | Meta callbacks |
| **i18n** | `i18n/en.ts`, `i18n/bn.ts` | — | — |

---

## ১৩. Auth Flow

```
১. Register
   Client: register/page.tsx → auth.register()
   Server: POST /auth/register → cookies set

২. Login
   Client: login/page.tsx → auth.login()
   Server: POST /auth/login → accessToken + refreshToken cookies
   Client: AuthProvider → GET /auth/me

৩. OAuth (Google/Facebook)
   Client: OAuth button → redirect /auth/google
   Server: Passport → callback → cookies → redirect /auth/callback
   Client: auth/callback → getMe()

৪. Onboarding
   Client: OnboardingGuard → /onboarding if incomplete
   Server: POST /auth/complete-onboarding

৫. Terms
   Client: TermsGuard → /accept-terms if not accepted
   Server: POST /auth/accept-terms

৬. Token Refresh (401)
   Client: apiFetch() → POST /auth/refresh → retry
   Server: new accessToken cookie

৭. Admin Access
   Client: AdminGuard → role admin/super_user check
   Server: authorize middleware on /admin routes
```

---

## ১৪. Post Publish Flow

```
Create Post                Schedule                  Publish
──────────                 ────────                  ───────
post-form.tsx       →      POST /posts        →      Agenda job queued
  │                          (status: scheduled)         │
  │ upload media             scheduledAt: datetime       │
  │ POST /uploads/images                                 │
  │                                                      ▼
  │                                              posts.scheduler.service
  │                                                      │
  │                                                      ▼
  │                                              Facebook Graph API
  │                                              (publish post/reel/story)
  │                                                      │
  │                                                      ▼
  │                                              status → published/failed
  │                                              SSE notification → client
```

---

## ১৫. MongoDB Models

| Model | উদ্দেশ্য |
|-------|---------|
| `User` | Accounts, roles (`user`, `admin`, `super_user`) |
| `AuthToken` | Refresh token sessions |
| `FacebookConnection` / `FacebookPage` | FB OAuth + page tokens |
| `TikTokAccount` / `TikTokOAuthState` | TikTok integration |
| `Post` | Drafts, scheduled, published, failed posts |
| `SchedulingModel` | Reusable scheduling templates |
| `Notification` | In-app notifications |
| `UserActivityLog` | User action audit |
| `AdminAuditLog` | Admin action audit |
| `CloudinaryPool` / `MediaUploadRecord` | Media storage quotas |
| `DataDeletionRequest` | Meta data-deletion compliance |

---

## ১৬. Environment Variables

### Client (`.env.local`)
```
NEXT_PUBLIC_API_URL=http://localhost:5000/api/v1
```

### Server (`.env`)
```
PORT=5000
MONGODB_URI=mongodb://...
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
CLIENT_URL=http://localhost:3000
CLOUDINARY_*=...
FACEBOOK_APP_ID=...
FACEBOOK_APP_SECRET=...
TIKTOK_CLIENT_KEY=...
TIKTOK_CLIENT_SECRET=...
GOOGLE_CLIENT_ID=...
SMTP_*=...
SUPER_ADMIN_EMAIL=...
SUPER_ADMIN_PASSWORD=...
```

---

## ১৭. চালানোর নিয়ম (Local)

```bash
# Terminal 1 — Server
cd Schdo/Server
npm install
npm run dev          # → http://localhost:5000

# Terminal 2 — Worker (optional, scheduled posts)
cd Schdo/Server
npm run dev:worker

# Terminal 3 — Client
cd Schdo/Client
npm install
npm run dev          # → http://localhost:3000
```

---

## ১৮. গুরুত্বপূর্ণ ফাইল রেফারেন্স

| ফাইল | ভূমিকা |
|------|--------|
| `Client/src/lib/api-client.ts` | মূল HTTP client, token refresh, upload |
| `Client/src/providers/auth-provider.tsx` | Auth state bootstrap |
| `Client/src/providers/notifications-provider.tsx` | SSE notification stream |
| `Client/src/components/dashboard/post-form/post-form.tsx` | Post create/edit UI |
| `Client/src/providers/bulk-upload-queue-provider.tsx` | Background bulk upload |
| `Client/src/app/(dashboard)/dashboard-layout-client.tsx` | Guard chain |
| `Server/src/app.ts` | Express middleware stack |
| `Server/src/server.ts` | Bootstrap: DB, queue, worker |
| `Server/src/routes/v1.routes.ts` | Route mounting |
| `Server/src/features/posts/posts.service.ts` | Core post CRUD + publish |
| `Server/src/features/posts/posts.scheduler.service.ts` | Poll-based publisher |
| `Server/src/queue/post-publish.queue.ts` | Agenda timed jobs |
| `Server/src/utils/facebook-graph/` | Facebook Graph API |
| `Server/src/utils/tiktok-api/` | TikTok OAuth + API |
| `Server/src/config/env.ts` | Centralized env config |

---

## ১৯. তিন প্রজেক্টের তুলনা

| বিষয় | SplasBD | Panzo | Schdo |
|-------|---------|-------|-------|
| **Domain** | E-commerce | E-commerce (Fashion) | Social Media Scheduling |
| **Admin** | Next.js embedded | আলাদা Vite app | Next.js route group |
| **API prefix** | `/api` | `/api/v1` | `/api/v1` |
| **Auth** | JWT cookies | JWT cookies | JWT cookies + OAuth |
| **Job Queue** | — | — | Agenda + poll worker |
| **Real-time** | — | sendBeacon analytics | SSE streams |
| **External APIs** | Steadfast courier | — | Facebook Graph, TikTok |
| **i18n** | — | — | English + Bengali |
| **Permission** | Admin role | Full RBAC | Role-based (admin/super_user) |

---

## ২০. আর্কিটেকচার সিদ্ধান্ত

1. **SaaS platform** — multi-user Facebook/TikTok post scheduling
2. **Agenda job queue** — reliable timed post publishing
3. **Poll worker fallback** — high-concurrency scheduled post processing
4. **SSE real-time** — notifications ও admin monitoring
5. **Guard chain** — Auth → Terms → Onboarding sequential checks
6. **Cloudinary pools** — media quota management per user
7. **Meta compliance** — data-deletion/deauthorize callbacks
8. **Bilingual UI** — English + Bengali i18n support
9. **Bulk operations** — background upload/delete queues
10. **Token encryption** — AES-256-GCM for stored OAuth tokens

---

*শেষ আপডেট: জুলাই ২০২৬ | SchdoSocial — Social Media Scheduling Platform*
