# Classroom Management System

A full-stack web app for running a school's academic structure: **departments**, the **subjects** they offer, the **classes** teachers run for those subjects, and the **students** enrolled in them.

Admins and teachers use a dashboard to create and browse subjects and classes. Students join a class with its invite code.

> **Project status:** in active development (roughly 35–40% complete). For the current bugs, missing endpoints and remaining work, see [KNOWN_ISSUES.md](KNOWN_ISSUES.md).

---

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Data model](#data-model)
- [API reference](#api-reference)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Scripts](#scripts)
- [Deployment](#deployment)
- [Roadmap](#roadmap)

---

## Features

- **Subjects:** list subjects with search, a department filter and pagination.
- **Classes:** list classes with search and subject/teacher filters. Create a class with a banner image, capacity, status and teacher. View class details.
- **Invite codes:** each class gets a unique invite code that students use to join.
- **Image uploads:** class banners are uploaded straight from the browser to Cloudinary.
- **Authentication:** email-and-password sign-up and sign-in through Better Auth, with three user roles: `admin`, `teacher` and `student`.
- **API protection:** Arcjet provides shield (common attack protection), bot detection and rate limits that depend on the user's role.
- **Monitoring:** Site24x7 APM Insight instruments the backend.
- **Admin UI:** built on Refine, with data tables, forms validated with zod and react-hook-form, light and dark themes, and a command palette (Kbar).

## Tech stack

| Layer | Technology |
|---|---|
| Frontend framework | React 19, Vite 6, TypeScript |
| Admin framework | Refine 5 (`@refinedev/core`, `@refinedev/rest`, `@refinedev/react-router`, `@refinedev/react-table`) |
| UI | Tailwind CSS v4, shadcn/ui (Radix UI), lucide-react, sonner, Recharts |
| Forms and validation | react-hook-form, zod |
| Media | Cloudinary (`@cloudinary/react`) |
| Backend | Node.js, Express 5, TypeScript (ESM), tsx |
| Database | PostgreSQL on Neon (serverless HTTP driver) |
| ORM and migrations | Drizzle ORM, drizzle-kit |
| Auth | Better Auth (Drizzle adapter) |
| Security | Arcjet (shield, bot detection, sliding-window rate limiting) |
| Observability | Site24x7 APM Insight |

## Architecture

```
┌──────────────────────────────┐        HTTPS / JSON         ┌───────────────────────────────┐
│  Frontend  (frontend/)       │  ─────────────────────────▶ │  Backend API  (backend/)      │
│  React + Vite + Refine       │   /api/subjects, /classes,  │  Express 5                    │
│                              │   /users, /auth/*           │                               │
│  • Pages (list/create/show)  │ ◀─────────────────────────  │  • CORS (FRONTEND_URL only)   │
│  • REST data provider        │                             │  • Better Auth  /api/auth/*   │
│    (src/providers/data.ts)   │                             │  • Route handlers             │
│  • zod form schemas          │                             │  • Arcjet security middleware │
└──────────────┬───────────────┘                             └───────────────┬───────────────┘
               │ direct browser upload                                       │ Drizzle ORM
               ▼                                                             ▼ (neon-http)
       ┌───────────────┐                                           ┌───────────────────┐
       │  Cloudinary   │   banner URL + public ID saved on class   │  Neon PostgreSQL  │
       └───────────────┘                                           └───────────────────┘
```

**How a request flows**

1. A Refine page calls a hook such as `useList({ resource: 'classes', filters })`.
2. The REST data provider in `frontend/src/providers/data.ts` turns it into a request such as `GET {VITE_BACKEND_BASE_URL}classes?page=1&limit=10&search=…`.
3. Express checks the origin against `FRONTEND_URL` (CORS) and sends the request to a router in `backend/src/routes/`.
4. The router builds a Drizzle query, which runs on Neon over HTTPS, and returns `{ data, pagination }`.
5. Refine caches the response and renders it in a data table or detail view.

**Authentication:** Better Auth is mounted at `/api/auth/*`. It stores users, sessions and accounts in Postgres through the Drizzle adapter. The `user` table is extended with a `role` field (`student` by default) and an optional `imageCldPubId`.

**Security:** the Arcjet middleware chooses a rate limit from the caller's role (admin 20/min, teacher and student 10/min, guest 5/min) and also applies shield and bot detection.

## Data model

```
departments 1 ──── * subjects 1 ──── * classes * ──── 1 user (teacher)
                                         │
                                         * enrollments * ──── 1 user (student)
```

| Table | Key columns |
|---|---|
| `departments` | `id`, `code` (unique), `name`, `description` |
| `subjects` | `id`, `department_id` → departments, `code` (unique), `name`, `description` |
| `classes` | `id`, `subject_id` → subjects, `teacher_id` → user, `invite_code` (unique), `name`, `description`, `banner_url`, `banner_cld_pub_id`, `capacity` (default 50), `status` (`active` / `inactive` / `archived`), `schedules` (jsonb) |
| `enrollments` | composite primary key (`student_id` → user, `class_id` → classes) |
| `user`, `session`, `account`, `verification` | Better Auth tables. `user` has the extra columns `role` and `image_cld_pub_id` |

Every app table has `created_at` and `updated_at` timestamps. The schema is defined in `backend/src/db/schema/`, and the migrations are in `backend/drizzle/`.

## API reference

Base URL: `http://localhost:8000/api`

| Method | Endpoint | Description | Query parameters |
|---|---|---|---|
| `GET` | `/subjects` | Paginated list of subjects, with their departments | `search`, `department`, `page`, `limit` |
| `GET` | `/classes` | Paginated list of classes, with their subjects and teachers | `search`, `subject`, `teacher`, `page`, `limit` |
| `GET` | `/classes/:id` | Details of one class | — |
| `POST` | `/classes` | Create a class (the invite code is generated) | — |
| `GET` | `/users` | Paginated list of users | `search`, `role`, `page`, `limit` |
| `*` | `/auth/*` | Better Auth endpoints (sign-up, sign-in, session, sign-out) | — |

List responses use this shape:

```json
{
  "data": [ ... ],
  "pagination": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 }
}
```

The endpoints that are planned but not built yet (departments, enrollments, and class update and delete) are listed in [KNOWN_ISSUES.md](KNOWN_ISSUES.md#3-missing-backend-endpoints).

## Project structure

```
class-room-full-system/
├── README.md               # This file
├── KNOWN_ISSUES.md         # Current bugs and remaining work
├── backend/
│   ├── drizzle/            # Generated SQL migrations and snapshots
│   ├── drizzle.config.ts
│   └── src/
│       ├── index.ts        # Express app entry (CORS, auth, routers, middleware)
│       ├── config/arcjet.ts
│       ├── db/
│       │   ├── index.ts    # Neon + Drizzle client
│       │   └── schema/     # app.ts (domain tables), auth.ts (Better Auth tables)
│       ├── lib/auth.ts     # Better Auth configuration
│       ├── middleware/security.ts
│       └── routes/         # classes.ts, subjects.ts, users.ts
└── frontend/
    ├── Dockerfile, vercel.json
    └── src/
        ├── App.tsx         # Refine setup, resources and routes
        ├── providers/data.ts   # REST data provider
        ├── pages/          # dashboard, subjects/*, classes/*
        ├── components/
        │   ├── refine-ui/  # Layout, data table, forms, buttons, views
        │   ├── ui/         # shadcn/ui primitives
        │   └── upload-widget.tsx  # Cloudinary upload
        ├── lib/            # zod schemas, Cloudinary helpers, utils
        ├── constants/      # Environment config and option lists
        └── types/
```

## Getting started

### Prerequisites

- Node.js 20 or later, and npm
- A [Neon](https://neon.tech) Postgres database
- An [Arcjet](https://arcjet.com) key
- A [Cloudinary](https://cloudinary.com) account with an unsigned upload preset

### 1. Clone

```bash
git clone https://github.com/nelsonhano/class-room-full-system.git
cd class-room-full-system
```

### 2. Backend

```bash
cd backend
npm install
# create backend/.env (see Environment variables)
npm run db:migrate
npm run dev            # http://localhost:8000
```

### 3. Frontend

```bash
cd frontend
npm install
# create frontend/.env (see Environment variables)
npm run dev            # http://localhost:5173
```

> If the frontend shows a Tailwind or PostCSS error overlay, the fix is described in [KNOWN_ISSUES.md](KNOWN_ISSUES.md#1-blockers-nothing-works-end-to-end-until-these-are-fixed).

## Environment variables

**`backend/.env`**

| Variable | Description |
|---|---|
| `DATABASE_URL` | Neon Postgres connection string |
| `BETTER_AUTH_SECRET` | Random secret Better Auth uses to sign sessions |
| `FRONTEND_URL` | Frontend origin for CORS and trusted origins, for example `http://localhost:5173` |
| `ARCJET_KEY` | Arcjet site key |
| `ARCJET_ENV` | Set to `development` for local development |

The backend also reads `backend/apminsightnode.json` for the Site24x7 APM license. This file is gitignored, so create it yourself if you use APM.

**`frontend/.env`**

| Variable | Description |
|---|---|
| `VITE_BACKEND_BASE_URL` | API base URL, for example `http://localhost:8000/api/` (keep the trailing slash) |
| `VITE_CLOUDINARY_CLOUD_NAME` | Cloudinary cloud name |
| `VITE_CLOUDINARY_UPLOAD_PRESET` | Unsigned upload preset |
| `VITE_CLOUDINARY_UPLOAD_URL` | Cloudinary upload endpoint |

## Scripts

| Location | Command | What it does |
|---|---|---|
| `backend/` | `npm run dev` | Start the API with hot reload (tsx watch) |
| `backend/` | `npm run build` | Compile TypeScript to `dist/` |
| `backend/` | `npm run db:generate` | Generate a migration from schema changes |
| `backend/` | `npm run db:migrate` | Apply migrations to the database |
| `frontend/` | `npm run dev` | Start the Vite dev server |
| `frontend/` | `npm run build` | Build the production bundle |
| `frontend/` | `npm run start` | Serve the production build |

## Deployment

- **Frontend:** `frontend/vercel.json` rewrites every route to `index.html` so client-side routing works on Vercel. You can also use `frontend/Dockerfile`, which builds the app and serves it with `serve`.
- **Backend:** any Node host works, for example Render, Railway or Fly. Set the backend environment variables there. Before deploying, apply the build and port fixes listed in [KNOWN_ISSUES.md](KNOWN_ISSUES.md#4-bugs-in-existing-backend-code).

## Roadmap

- Department, enrollment and class edit/delete endpoints, with their pages
- Sign-in, sign-up and protected routes in the frontend, plus role-based access control on the API
- A dashboard with statistics and charts
- Pages for faculty, users and enrollments
- Tests for the backend and the frontend

The detailed, prioritized list is in [KNOWN_ISSUES.md](KNOWN_ISSUES.md).
