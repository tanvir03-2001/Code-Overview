# Schdo (SchdoSocial) — Code Overview

[← Table of Contents](./README.md)

> **Purpose:** This document explains the full architecture of the SchdoSocial project — pages, functions, core client/server code, and how they connect. Use this file to understand the project without sharing the git repository.

---

## 1. Project Structure (High-Level)

```
Schdo/
├── Client/          → Next.js 16 frontend (React 19, port 3000)
└── Server/          → Express 5 API (port 5000) + Worker process
```

| Layer | Technology | Port (local) |
|-------|------------|--------------|
| **Client** | Next.js App Router, Tailwind CSS 4, Radix UI, TanStack Query | `localhost:3000` |
| **Server** | Express 5, Mongoose 9, Agenda job queue, Zod | `localhost:5000` |
| **Worker** | Standalone Agenda worker (`npm run dev:worker`) | — |
| **Database** | MongoDB | — |
| **Integrations** | Facebook Graph API, TikTok API, Cloudinary, Google OAuth | — |

> **Project goal:** SaaS platform (**SchdoSocial**) to schedule, publish, and manage social media posts on Facebook Pages and TikTok accounts.

> **Note:** `Client/` and `Server/` are separate git repositories.

---

## 2. How Client ↔ Server Connect

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

**Key rules:**
- Client calls the Express API directly
- Base URL: `NEXT_PUBLIC_API_URL` → `http://localhost:5000/api/v1`
- Auth: `httpOnly` cookies — `apiFetch()` auto-refreshes on 401
- Real-time: SSE streams (notifications, admin monitoring)
- Publishing: Server-side Agenda job queue + poll worker

---

## 3. Client — Folders and Responsibilities

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
│   ├── api-client.ts       # Core HTTP client
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

## 4. Route Groups and Guard Chain

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

## 5. Public & Auth Pages

| URL | File | Purpose |
|-----|------|---------|
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

## 6. Dashboard Pages (User)

| URL | Purpose |
|-----|---------|
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

## 7. Admin Pages

| URL | Purpose |
|-----|---------|
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

**Admin protection:** `AdminGuard` — requires role `admin` or `super_user`.

---

## 8. Client Code — Core Parts

### 8.1 API Client (`lib/api-client.ts`)

Central hub for all HTTP calls:

```typescript
// lib/api-client.ts
export const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:5000/api/v1";

export const apiFetch = async <T>(path: string, options: RequestInit = {}, retry = true): Promise<T> => {
  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    credentials: "include",  // sends JWT cookies
  });
  // 401 → refreshSession() → retry once
  return body.data;
};

export const apiUpload = ...       // Multipart upload with progress
export const apiUploadWithProgress = ...
```

| Client Module | Server Endpoint | Purpose |
|--------------|-----------------|---------|
| `lib/auth.ts` | `/auth/*` | Login, register, OAuth, sessions |
| `lib/posts.ts` | `/posts/*` | Post CRUD, analytics, bulk delete |
| `lib/facebook-pages.ts` | `/facebook-pages/*` | FB page connect/manage |
| `lib/tiktok-accounts.ts` | `/tiktok-accounts/*` | TikTok account connect |
| `lib/scheduling-models.ts` | `/scheduling-models/*` | Scheduling templates |
| `lib/notifications.ts` | `/notifications/*` | In-app notifications |
| `lib/uploads.ts` | `/uploads/images` | Media upload |
| `lib/admin.ts` | `/admin/*` | Admin operations |

---

### 8.2 Auth Provider (`providers/auth-provider.tsx`)

```typescript
// On mount: GET /auth/me → set user state
// Login/register/OAuth → cookies set → refresh user
// logout → POST /auth/logout → clear state
```

---

### 8.3 Notifications Provider (`providers/notifications-provider.tsx`)

```typescript
// SSE stream: GET /notifications/stream
// Real-time notification push
// Unread count: GET /notifications/unread-count
```

---

### 8.4 Post Form (`components/dashboard/post-form/post-form.tsx`)

Main post creation/editing UI:
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

### 8.5 Bulk Upload Queue (`providers/bulk-upload-queue-provider.tsx`)

Background bulk post upload — multiple posts uploaded together:
```
BulkUploadQueueProvider → sequential upload → progress tracking
→ GET /posts/upload-progress/:token
```

---

## 9. Server — Folders and Responsibilities

```
Server/src/
├── app.ts                  # Express setup + middleware
├── server.ts               # Bootstrap: DB, queue, scheduler worker
├── worker.ts               # Standalone Agenda worker
├── routes/
│   └── v1.routes.ts        # All route mounts
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

## 10. Server Code — Core Parts

### 10.1 Route Mounting (`routes/v1.routes.ts`)

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

### 10.2 Server Bootstrap (`server.ts`)

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

### 10.3 Scheduled Post Publishing

```
User schedules post → POST /api/v1/posts (status: scheduled)
                   → Agenda job created (post-publish.queue.ts)
                   → At scheduled time: posts.scheduler.service.ts
                   → Facebook Graph API publish (utils/facebook-graph/)
                   → Post status → published / failed
                   → Notification sent via SSE
```

---

### 10.4 Zod Validation Middleware

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

## 11. API Endpoint Summary

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

## 12. Feature Module Mapping

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

## 13. Auth Flow

```
1. Register
   Client: register/page.tsx → auth.register()
   Server: POST /auth/register → cookies set

2. Login
   Client: login/page.tsx → auth.login()
   Server: POST /auth/login → accessToken + refreshToken cookies
   Client: AuthProvider → GET /auth/me

3. OAuth (Google/Facebook)
   Client: OAuth button → redirect /auth/google
   Server: Passport → callback → cookies → redirect /auth/callback
   Client: auth/callback → getMe()

4. Onboarding
   Client: OnboardingGuard → /onboarding if incomplete
   Server: POST /auth/complete-onboarding

5. Terms
   Client: TermsGuard → /accept-terms if not accepted
   Server: POST /auth/accept-terms

6. Token Refresh (401)
   Client: apiFetch() → POST /auth/refresh → retry
   Server: new accessToken cookie

7. Admin Access
   Client: AdminGuard → role admin/super_user check
   Server: authorize middleware on /admin routes
```

---

## 14. Post Publish Flow

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

## 15. MongoDB Models

| Model | Purpose |
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

## 16. Environment Variables

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

## 17. How to Run Locally

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

## 18. Key File Reference

| File | Role |
|------|------|
| `Client/src/lib/api-client.ts` | Core HTTP client, token refresh, upload |
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

## 19. Three-Project Comparison

| Topic | SplasBD | Panzo | Schdo |
|-------|---------|-------|-------|
| **Domain** | E-commerce | E-commerce (Fashion) | Social Media Scheduling |
| **Admin** | Next.js embedded | Separate Vite app | Next.js route group |
| **API prefix** | `/api` | `/api/v1` | `/api/v1` |
| **Auth** | JWT cookies | JWT cookies | JWT cookies + OAuth |
| **Job Queue** | — | — | Agenda + poll worker |
| **Real-time** | — | sendBeacon analytics | SSE streams |
| **External APIs** | Steadfast courier | — | Facebook Graph, TikTok |
| **i18n** | — | — | English + Bengali |
| **Permission** | Admin role | Full RBAC | Role-based (admin/super_user) |

---

## 20. Code Examples

Real code from the project — shows TypeScript patterns used across client and server.

### Example 1 — Typed API fetch with auto refresh (`Client/src/lib/api-client.ts`)

```typescript
export const apiFetch = async <T>(
  path: string,
  options: RequestInit = {},
  retry = true,
): Promise<T> => {
  const headers = new Headers(options.headers);

  if (!headers.has("Content-Type") && options.body) {
    headers.set("Content-Type", "application/json");
  }

  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    headers,
    credentials: "include",
  });

  const body = (await response.json()) as ApiSuccessResponse<T> | ApiErrorResponse;

  if (response.status === 401 && !body.success) {
    const retried = await handleUnauthorized(path, body, retry, () =>
      apiFetch<T>(path, options, false),
    );

    if (retried !== null) {
      return retried;
    }
  }

  if (!response.ok || !body.success) {
    throw new ApiError(
      body.success ? "Request failed" : parseErrorMessage(body),
      response.status,
      body.success ? undefined : body.errors,
    );
  }

  return body.data;
};
```

### Example 2 — Route guard with loading state (`Client/src/components/auth/auth-guard.tsx`)

```typescript
export function AuthGuard({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      router.replace("/login");
    }
  }, [isAuthenticated, isLoading, router]);

  if (isLoading) {
    return <AuthScreenSkeleton />;
  }

  if (!isAuthenticated) {
    return null;
  }

  return <>{children}</>;
}
```

### Example 3 — Zod validation middleware (`Server/src/middleware/validate.ts`)

```typescript
export const validate =
  (schema: ZodType, source: RequestSource = "body") =>
  (req: Request, _res: Response, next: NextFunction) => {
    const result = schema.safeParse(req[source]);

    if (!result.success) {
      return next(
        new AppError(
          "Validation failed",
          StatusCodes.BAD_REQUEST,
          result.error.flatten(),
        ),
      );
    }

    if (source === "body") {
      req.body = result.data;
    } else if (source === "query") {
      req.validatedQuery = result.data;
    } else {
      req.validatedParams = result.data;
    }

    next();
  };
```

### Example 4 — Scheduled post claim and publish (`Server/src/features/posts/posts.scheduler.service.ts`)

```typescript
const claimDuePosts = async (
  limit: number,
  now: Date,
  claimTtlMs: number,
): Promise<IPost[]> => {
  const staleBefore = new Date(now.getTime() - claimTtlMs);
  const filter = buildDueClaimFilter(now, staleBefore);

  const claims = await Promise.all(
    Array.from({ length: limit }, () =>
      Post.findOneAndUpdate(
        filter,
        { $set: { publishClaimedAt: now, publishStartedAt: now } },
        { sort: { scheduledAt: 1, createdAt: 1 }, new: true },
      ),
    ),
  );

  return claims.filter((post) => post !== null);
};

const getConnectedPage = async (
  post: IPost,
  pageCache: Map<string, IFacebookPage>,
): Promise<IFacebookPage> => {
  const cacheKey = post.facebookPageId.toString();
  const cached = pageCache.get(cacheKey);

  if (cached) {
    return cached;
  }

  const page = await FacebookPage.findOne({
    _id: post.facebookPageId,
    userId: post.userId,
    connected: true,
  }).select("+pageAccessToken");

  if (!page?.pageAccessToken) {
    throw new Error(
      "Connected Facebook page not found or missing access token.",
    );
  }

  pageCache.set(cacheKey, page);
  return page;
};
```

---

## 21. Architecture Decisions

1. **SaaS platform** — multi-user Facebook/TikTok post scheduling
2. **Agenda job queue** — reliable timed post publishing
3. **Poll worker fallback** — high-concurrency scheduled post processing
4. **SSE real-time** — notifications and admin monitoring
5. **Guard chain** — Auth → Terms → Onboarding sequential checks
6. **Cloudinary pools** — media quota management per user
7. **Meta compliance** — data-deletion/deauthorize callbacks
8. **Bilingual UI** — English + Bengali i18n support
9. **Bulk operations** — background upload/delete queues
10. **Token encryption** — AES-256-GCM for stored OAuth tokens

---

*Last updated: July 2026 | SchdoSocial — Social Media Scheduling Platform*
