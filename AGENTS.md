# AGENTS.md — PantauKota

> This document is intended for AI coding assistants to understand the architecture,
> conventions, and gotchas of the **PantauKota** project as quickly as possible.

---

## 1. Project Summary

PantauKota is a web-based city infrastructure reporting platform (PWA).
Citizens create reports (with photos, GPS locations, categories), then Admins review,
process, and resolve these reports. Key features:

- **Reporting**: Create reports with photos (Cloudinary), location maps (Leaflet), categories.
- **Voting & Comments**: Citizens can support (upvote) and comment on reports.
- **Automatic Priority**: Reports with high scores (votes × 2 + age in days) are automatically prioritized.
- **Real-time Notifications**: Supabase Realtime (WebSocket) for instant push notifications.
- **Admin Dashboard**: Manage reports, categories, users, statistics, and admin maps.
- **PWA**: Service Worker via Serwist for offline support & installability.

---

## 2. Tech Stack

| Layer         | Technology                                                   |
| ------------- | ------------------------------------------------------------ |
| Framework     | **Next.js 15** (App Router, Server Components, Webpack)      |
| Language      | **TypeScript 5**                                             |
| Styling       | **Tailwind CSS 3** + custom design tokens (see `DESIGN.md`) |
| Database      | **Supabase PostgreSQL** + **Prisma 7** (ORM)                 |
| Auth          | **Supabase Auth** (email/password) + `@supabase/ssr`         |
| Realtime      | **Supabase Realtime** (postgres_changes on `Notifikasi` & `Laporan` tables) |
| Images        | **Cloudinary** (upload via API, delivery via public ID transformation URLs) |
| Maps          | **Leaflet 1.9** + **React-Leaflet 4** (client-side only)     |
| PWA           | **@serwist/next 9** (service worker, precaching)             |
| Icons         | **Lucide React** (all UI icons)                             |
| Charts        | **Recharts** (admin dashboard)                               |
| Forms         | **React Hook Form** + **Zod** (validation)                     |

---

## 3. Directory Structure

```
src/
├── app/                          # Next.js App Router
│   ├── (admin)/                  # Route group: Admin pages (SSR, requires ADMIN role)
│   │   ├── dashboard/            #   Main dashboard + admin report details
│   │   ├── kelola-kategori/      #   Category CRUD
│   │   ├── kelola-laporan/       #   All reports table + filter/search
│   │   ├── kelola-user/          #   User management (active/inactive)
│   │   └── layout.tsx            #   Admin layout (sidebar + mobile header)
│   ├── (auth)/                   # Route group: Login & Register
│   │   └── login/page.tsx        #   Renders AuthScreen component
│   ├── (warga)/                  # Route group: Citizen pages (SSR, requires login)
│   │   ├── beranda/              #   Latest reports feed
│   │   ├── laporan/              #   [id] details, /buat report creation form
│   │   ├── laporan-saya/         #   User's own reports
│   │   ├── notifikasi/           #   Notifications list page
│   │   ├── peta/                 #   Map of all reports
│   │   └── layout.tsx            #   Citizen layout (top navbar)
│   ├── api/                      # Route Handlers (REST API)
│   │   ├── admin/                #   Stats dashboard
│   │   ├── auth/                 #   /register, /session, callback
│   │   ├── kategori/[id]/        #   PATCH, DELETE category
│   │   ├── komentar/[id]/        #   DELETE comment
│   │   ├── laporan/              #   GET list, POST create
│   │   ├── laporan/[id]/         #   GET details, PATCH status, DELETE
│   │   ├── notifikasi/           #   GET, PATCH (mark read), DELETE
│   │   ├── upload/               #   POST (Cloudinary signed upload)
│   │   ├── user/profile/         #   GET/PATCH profile, /[id] admin ops
│   │   └── vote/                 #   POST toggle vote
│   ├── auth/callback/            # Supabase OAuth callback handler
│   ├── sw.ts                     # Service Worker (Serwist)
│   └── layout.tsx                # Root layout
├── components/
│   ├── admin/                    # DashboardCharts, etc.
│   ├── auth/                     # AuthScreen (login/register form)
│   ├── komentar/                 # KomentarSection
│   ├── landing/                  # Landing page (GlobeScene, etc.)
│   ├── laporan/                  # StatusTimeline, DeleteButton, DuplikasiModal, PrioritasScore
│   ├── layout/                   # AdminSidebar, AdminMobileHeader, WargaNavbar, AdminLayoutClient
│   ├── map/                      # MapView, AdminMapView, LocationPicker, + Client wrappers
│   ├── ui/                       # Badge, Button, Toast, VoteButton, Spinner, CameraModal, DynamicIcon
│   ├── NotificationBell.tsx      # Notification bell (header)
│   └── Providers.tsx             # Client providers wrapper
├── hooks/
│   ├── useAuthSession.tsx        # React Context for auth session (client-side)
│   ├── useDebounce.ts            # Debounce hook
│   ├── useGeolocation.ts         # Browser GPS
│   ├── useLaporanMap.ts          # Fetch report data for map
│   ├── useNotifications.ts       # Supabase Realtime listener + notification state
│   ├── useToast.ts               # Toast notification hook
│   └── useVote.ts                # Toggle vote hook
├── lib/
│   ├── auth.ts                   # getCurrentUser(), getCurrentSession() — SERVER ONLY
│   ├── cloudinary.ts             # getCloudinaryImageUrl() — public ID → delivery URL
│   ├── constants.ts              # Centralized magic numbers
│   ├── geo.ts                    # Haversine, reverse geocoding (Nominatim)
│   ├── map.ts                    # Leaflet config (tile URL, default center, icon init)
│   ├── notifications.ts          # kirimNotifikasi(), kirimNotifikasiAdmin() — Prisma insert
│   ├── prisma.ts                 # PrismaClient singleton
│   ├── supabase/
│   │   ├── browser.ts            # createSupabaseBrowserClient() — singleton, CLIENT ONLY
│   │   ├── middleware.ts          # updateSupabaseSession() — refresh token in middleware
│   │   └── server.ts             # createSupabaseServerClient() — SERVER ONLY, uses cookies()
│   └── utils.ts                  # calculatePriorityScore(), cn(), etc.
├── middleware.ts                  # Auth guard: redirects unauthenticated → /login
├── types/
│   ├── laporan.ts                # LaporanMapItem, STATUS_CONFIG, getMarkerColor()
│   └── user.ts                   # CurrentUser, CurrentSession
prisma/
├── schema.prisma                 # Data model (User, Laporan, Kategori, Vote, Komentar, Notifikasi)
├── seed.ts                       # Seed data (categories, admin user)
└── supabase-init.sql             # Enable Realtime publication, RLS policies
```

---

## 4. Important Rules & Gotchas

### 4.1 Next.js 15 Breaking Changes

- **`params` are Promises.** All Server Components and Route Handlers with dynamic segments
  (`[id]`) MUST use `params: Promise<{ id: string }>` and `const { id } = await params;`.
- **`cookies()` is `async`.** Calls to `cookies()` MUST be `await`ed.
  See `src/lib/supabase/server.ts` for examples.
- **`ssr: false` is forbidden in Server Components.** Use the Client Component wrapper pattern:
  create a `*Client.tsx` file with `'use client'` + `dynamic(() => import(...), { ssr: false })`,
  then import that wrapper from the Server Component.
  Example: `MapViewClient.tsx` wraps `MapView.tsx`.

### 4.2 React-Leaflet

- **`reactStrictMode` is disabled** in `next.config.mjs` because react-leaflet crashes
  on double-mount in development (error "Map container is already initialized").
- All map components (`MapView`, `AdminMapView`, `LocationPicker`) MUST be rendered
  client-side only (via `*Client.tsx` wrappers or `next/dynamic` in Client Components).
- Leaflet CSS is imported directly in map components: `import 'leaflet/dist/leaflet.css'`.

### 4.3 Supabase Auth Flow

```
Login → supabase.auth.signInWithPassword()
     → fetch /api/auth/session (server checks Prisma user + isActive)
     → redirect based on role (ADMIN → /dashboard, WARGA → /beranda)
```

- Auth state is refreshed by middleware (`src/middleware.ts` → `updateSupabaseSession()`).
- Users with `isActive: false` will be denied login with an "Account disabled" message.
- Session context (client-side) is provided by `useAuthSession` hook + `SessionProvider`.

### 4.4 Supabase Realtime

- Listener is in `src/hooks/useNotifications.ts`.
- Subscribes to **both tables** in a single channel:
  - `Notifikasi` (INSERT) → add to state + `router.refresh()`
  - `Laporan` (INSERT/UPDATE/DELETE) → `router.refresh()`
- Tables must be registered in the Supabase Realtime publication:
  ```sql
  ALTER PUBLICATION supabase_realtime ADD TABLE "Notifikasi";
  ALTER PUBLICATION supabase_realtime ADD TABLE "Laporan";
  ```
- Channel names MUST be unique (using `Date.now()` + `Math.random()`) to avoid
  "cannot add callbacks after subscribe()" errors.

### 4.5 Cloudinary Images

- Report photos are stored as **Cloudinary public IDs** (e.g., `pantaukota/abc123`),
  NOT full URLs.
- To display images, ALWAYS use `getCloudinaryImageUrl(publicId, options)` from
  `src/lib/cloudinary.ts`. This function is also backward-compatible with legacy full URLs.
- Uploads happen via `/api/upload` (signed upload, server-side).

### 4.6 Report Priority

- Score is calculated as: `(voteCount × 2) + (age in days)`.
- Priority threshold: `PRIORITY_THRESHOLD = 50` (in `src/lib/constants.ts`).
- Reports can also be manually flagged by admins (`prioritas: true`).

### 4.7 Notifications

- In-app notifications are created via `kirimNotifikasi()` / `kirimNotifikasiAdmin()` in `src/lib/notifications.ts`.
  This directly INSERTs into the `Notifikasi` table via Prisma.

### 4.8 PWA / Service Worker

- Uses `@serwist/next` (not `next-pwa`).
- Service Worker source: `src/app/sw.ts`.
- **Disabled in development** (`disable: process.env.NODE_ENV === "development"` in `next.config.mjs`).
- The `sw.ts` file requires `/// <reference lib="webworker" />` on the first line
  to obtain the `ServiceWorkerGlobalScope` type.

---

## 5. Code Patterns & Conventions

### Language

- All variable names, functions, and comments use a **mix of Indonesian and English**.
- Prisma model names: Indonesian (Laporan, Kategori, Komentar, Notifikasi).
- Field names: Indonesian (judul, deskripsi, alamat, dibaca, etc.).
- Helper function names: Indonesian (kirimNotifikasi, tandaiBaca, etc.).

### Server vs Client Components

- **Pages (`page.tsx`)** = Server Components by default.
  They fetch data directly via Prisma and pass it as props.
- **Interactive components** use the `'use client'` directive.
- Do not mix `'use client'` with Prisma queries or `cookies()`.

### API Route Handlers

- All routes in `src/app/api/` use `export const dynamic = 'force-dynamic'`.
- Auth checks: `const session = await getCurrentSession()` + guard clauses.
- Responses: always `NextResponse.json(...)`.

### Styling

- The design system is defined in `DESIGN.md` and `tailwind.config.ts`.
- Colors use CSS custom properties (e.g., `bg-primary`, `text-on-surface`, `bg-surface-container-low`).
- Fonts: `font-display` (headings), `font-sans` (body).

### Error Handling

- APIs: try/catch → `console.error('[API /path METHOD]', error)` → return JSON error.
- Client: `useToast` hook for displaying success/error messages.

---

## 6. Development Commands

```bash
# Start development server
npm run dev

# Build for production
npm run build

# Generate Prisma client
npx prisma generate

# Run migrations
npx prisma migrate dev

# Seed database
npm run seed

# Type check
npx tsc --noEmit

# Lint
npm run lint
```

---

## 7. Environment Variables

See `.env.example` for the full list. Key variables:

| Variable                              | Description                              |
| ------------------------------------- | --------------------------------------- |
| `DATABASE_URL`                        | Supabase pooler (PgBouncer, port 6543)  |
| `DIRECT_URL`                          | Supabase direct (port 5432, for migrations) |
| `NEXT_PUBLIC_SUPABASE_URL`            | Supabase project URL                    |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`| Supabase anon/publishable key           |
| `SUPABASE_SERVICE_ROLE_KEY`           | Supabase service role key (server only) |
| `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME`   | Cloudinary cloud name                   |
| `CLOUDINARY_API_KEY`                  | Cloudinary API key (server only)        |
| `CLOUDINARY_API_SECRET`               | Cloudinary API secret (server only)     |

---

## 8. Data Model (Summary)

```
User (id, name, email, role[WARGA|ADMIN], isActive)
  ├── Laporan (judul, deskripsi, kategoriId, foto[], lat/lng, alamat, status[MENUNGGU|DIPROSES|SELESAI], voteCount, prioritas)
  │     ├── Vote (userId, laporanId) — unique constraint
  │     └── Komentar (userId, laporanId, isi)
  ├── Notifikasi (judul, pesan, laporanId?, dibaca)
  └── Kategori (nama, icon, warna, isActive)
```

---

## 9. Key Files Reference

| File | Role |
| --- | --- |
| `src/lib/auth.ts` | Main auth logic (getCurrentUser, getCurrentSession) |
| `src/lib/supabase/server.ts` | Server-side Supabase client (cookies-based) |
| `src/lib/supabase/browser.ts` | Client-side Supabase client (singleton) |
| `src/hooks/useNotifications.ts` | Realtime listener + notification state management |
| `src/hooks/useAuthSession.tsx` | Auth context provider + session hook |
| `src/lib/cloudinary.ts` | Image URL builder (public ID → delivery URL) |
| `src/lib/notifications.ts` | Server-side notification creator (Prisma insert) |
| `src/middleware.ts` | Auth middleware (protects routes, refreshes session) |
| `next.config.mjs` | Next.js config + Serwist PWA + CSP headers |
| `prisma/schema.prisma` | Database schema |
| `src/types/laporan.ts` | Shared types & status config |
