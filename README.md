# Code Overview — Table of Contents

> Concise documentation of client project architecture, pages, APIs, and client–server relationships.  
> Use this folder to understand each project without sharing the full git repository.

---

## Project List

Click a project name below to open its full Code Overview.

| # | Project | Type | Document |
|---|---------|------|----------|
| 1 | **SplasBD** | E-commerce (Bangladesh) | [SplasBD Code Overview →](./SplasBD-Code-Overview.md) |
| 2 | **Panzo** | E-commerce (Fashion / Panjabi) | [Panzo Code Overview →](./Panzo-Code-Overview.md) |
| 3 | **Schdo** | Social Media Scheduling (SchdoSocial) | [Schdo Code Overview →](./Schdo-Code-Overview.md) |

---

## Quick Links

- [SplasBD](./SplasBD-Code-Overview.md) — Next.js storefront + embedded admin + Express API  
- [Panzo](./Panzo-Code-Overview.md) — Next.js client + Vite admin + Express API + RBAC  
- [Schdo](./Schdo-Code-Overview.md) — Facebook/TikTok post scheduling SaaS  

---

## At a Glance

| Topic | SplasBD | Panzo | Schdo |
|-------|---------|-------|-------|
| **Domain** | E-commerce | E-commerce | Social scheduling |
| **Client** | Next.js 16 | Next.js 16 | Next.js 16 |
| **Admin** | Next.js (`/admin`) | Vite + React (separate app) | Next.js (`/admin`) |
| **Server** | Express 5 | Express 4 | Express 5 |
| **API Base** | `/api` | `/api/v1` | `/api/v1` |
| **Database** | MongoDB | MongoDB | MongoDB |
| **Auth** | JWT cookies | JWT cookies + RBAC | JWT cookies + OAuth |
| **Notable features** | Steadfast courier, guest checkout | Recommendation engine, analytics, promoter | Agenda queue, Facebook/TikTok publish, SSE |

---

## Folder Structure

```
Code overview/
├── README.md                    ← You are here (table of contents)
├── SplasBD-Code-Overview.md     ← SplasBD details
├── Panzo-Code-Overview.md       ← Panzo details
└── Schdo-Code-Overview.md       ← Schdo details
```

---

## What Each Document Covers

Each Code Overview file typically includes:

1. Project structure (folder layout)
2. Client ↔ Server connection diagram
3. Page URLs and file mapping
4. Core client code (`api.ts`, context, providers)
5. Core server code (routes, controllers, middleware)
6. API endpoint list
7. Feature module mapping (which component uses which API)
8. Auth flow and main user journeys
9. Environment variables and local run instructions
10. Key file reference

---

## Which Document to Read

| If you want to... | Read |
|-------------------|------|
| Understand Bangladesh e-commerce + guest checkout | [SplasBD](./SplasBD-Code-Overview.md) |
| See a separate admin app + permission/RBAC | [Panzo](./Panzo-Code-Overview.md) |
| Learn Facebook/TikTok scheduling + job queues | [Schdo](./Schdo-Code-Overview.md) |

---

*Last updated: July 2026*
