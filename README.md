# Code Overview — সূচিপত্র

> Client প্রজেক্টগুলোর architecture, page, API ও client–server সম্পর্কের সংক্ষিপ্ত ডকুমেন্টেশন।  
> Git repo শেয়ার না করলেও এই ফোল্ডার দিয়ে প্রজেক্ট বুঝতে সাহায্য করবে।

---

## প্রজেক্ট তালিকা

নিচের নামে ক্লিক করলে সেই প্রজেক্টের পূর্ণ Code Overview-এ যাবেন।

| # | প্রজেক্ট | ধরন | ডকুমেন্ট |
|---|---------|------|----------|
| ১ | **SplasBD** | E-commerce (Bangladesh) | [SplasBD Code Overview →](./SplasBD-Code-Overview.md) |
| ২ | **Panzo** | E-commerce (Fashion / Panjabi) | [Panzo Code Overview →](./Panzo-Code-Overview.md) |
| ৩ | **Schdo** | Social Media Scheduling (SchdoSocial) | [Schdo Code Overview →](./Schdo-Code-Overview.md) |

---

## দ্রুত লিংক

- 📦 [SplasBD](./SplasBD-Code-Overview.md) — Next.js storefront + embedded admin + Express API  
- 👔 [Panzo](./Panzo-Code-Overview.md) — Next.js client + Vite admin + Express API + RBAC  
- 📱 [Schdo](./Schdo-Code-Overview.md) — Facebook/TikTok post scheduling SaaS  

---

## এক নজরে তুলনা

| বিষয় | SplasBD | Panzo | Schdo |
|-------|---------|-------|-------|
| **Domain** | E-commerce | E-commerce | Social scheduling |
| **Client** | Next.js 16 | Next.js 16 | Next.js 16 |
| **Admin** | Next.js (`/admin`) | Vite + React (আলাদা app) | Next.js (`/admin`) |
| **Server** | Express 5 | Express 4 | Express 5 |
| **API Base** | `/api` | `/api/v1` | `/api/v1` |
| **Database** | MongoDB | MongoDB | MongoDB |
| **Auth** | JWT cookies | JWT cookies + RBAC | JWT cookies + OAuth |
| **বিশেষ ফিচার** | Steadfast courier, guest checkout | Recommendation engine, analytics, promoter | Agenda queue, Facebook/TikTok publish, SSE |

---

## ফোল্ডার কাঠামো

```
Code overview/
├── README.md                    ← আপনি এখানে আছেন (সূচিপত্র)
├── SplasBD-Code-Overview.md     ← SplasBD বিস্তারিত
├── Panzo-Code-Overview.md       ← Panzo বিস্তারিত
└── Schdo-Code-Overview.md       ← Schdo বিস্তারিত
```

---

## প্রতিটি ডকুমেন্টে কী আছে

প্রতিটি Code Overview ফাইলে সাধারণত এই সেকশনগুলো থাকে:

1. প্রজেক্ট কাঠামো (folder structure)
2. Client ↔ Server সংযোগ diagram
3. সব পেজের URL ও ফাইল ম্যাপিং
4. মূল client কোড (`api.ts`, context, providers)
5. মূল server কোড (routes, controllers, middleware)
6. API endpoint তালিকা
7. Feature module ম্যাপিং (কোন component কোন API ব্যবহার করে)
8. Auth flow ও main user journey
9. Environment variables ও local run নির্দেশনা
10. গুরুত্বপূর্ণ ফাইল রেফারেন্স

---

## কোন ডকুমেন্ট কখন পড়বেন

| আপনি যদি... | পড়ুন |
|------------|-------|
| Bangladesh e-commerce + guest checkout বুঝতে চান | [SplasBD](./SplasBD-Code-Overview.md) |
| আলাদা admin app + permission/RBAC দেখতে চান | [Panzo](./Panzo-Code-Overview.md) |
| Facebook/TikTok scheduling + job queue শিখতে চান | [Schdo](./Schdo-Code-Overview.md) |

---

*শেষ আপডেট: জুলাই ২০২৬*
