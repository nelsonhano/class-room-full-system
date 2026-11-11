# Classroom Full-Stack: Known Issues and Remaining Work

Status from a full local test run on 2026-11-11: every page was opened in a browser and every backend endpoint was called.

**Overall completion: roughly 35–40% (estimate).** The frontend shell, list pages and the class-create form exist. The database connection, authentication, most write endpoints and several pages are missing or broken.

- **Frontend:** React 19 + Vite + Refine + Tailwind v4, at `frontend/`, port 5173
- **Backend:** Express 5 + Drizzle ORM + Neon + Better Auth + Arcjet, at `backend/`, port 8000

---

## 1. Blockers (nothing works end-to-end until these are fixed)

- [ ] **Set a real database URL.** `DATABASE_URL` in `backend/.env` is still the template value (`postgresql://username:...@localhost:5432/database_name`), so every data endpoint returns `500`. The code uses the `drizzle-orm/neon-http` driver, which talks to Neon over HTTPS, so it needs a real **Neon** connection string. A plain local Postgres will not work unless you switch the driver (for example to `drizzle-orm/node-postgres`). After setting it, run `npm run db:migrate` in `backend/`.
- [ ] **Add `BETTER_AUTH_SECRET`** to `backend/.env`. It is missing, and `POST /api/auth/sign-up/email` returns `500`.
- [ ] **Add `ARCJET_ENV=development`** to `backend/.env` for local development. Without it Arcjet logs `Failed to build fingerprint` on every request (see 4.3).
- [ ] **Make the frontend ignore PostCSS configs in parent folders.** Vite looks for a PostCSS config in parent directories. On a machine with a `postcss.config.js` above the project (for example in the user's home folder), it loads Tailwind v3 and the app shows only an error overlay (`@layer base is used but no matching @tailwind base directive`). The project uses `@tailwindcss/vite`, so add this to `frontend/vite.config.ts`:
  ```ts
  css: { postcss: {} },
  ```

## 2. Frontend ↔ backend communication bugs

- [ ] **The teacher filter is dropped.** `useList({ resource: 'users', filters: [{ field: 'role', value: 'teacher' }] })` is sent as `GET /api/users?page=1&limit=100`, without `role=teacher`. `buildQueryParams` in `frontend/src/providers/data.ts` only handles `subjects` and `classes`. As a result, the Teacher dropdowns on Classes → List and Classes → Create show every user, students included. Fix: add a `users` branch that maps `role` → `params.role` and `name`/`email` → `params.search`.
- [ ] **Backend error messages never reach the UI.** The backend responds with `{ error: "..." }`, but `buildHttpError` in `data.ts` reads `payload.message`, so every failure is shown as "Request failed." Either send `message` from the backend or read `payload.error ?? payload.message`.
- [ ] **Errors on create and get-one are not handled.** In `data.ts`, the `create.mapResponse` and `getOne.mapResponse` functions never check `response.ok`, and they return `[]` when `data` is missing.
- [ ] **There is no visible error state.** When a request fails, list pages show an endless spinner and the class details page stays on "Loading class details..." while Refine retries. Show an error message and a retry button.
- [ ] **Sorting is ignored.** Pages send `sorters: [{ field: 'id', order: 'desc' }]`, but the data provider does not forward sorters and no backend route reads a sort parameter.
- [ ] **The department filter matches loosely.** `DEPARTMENT_OPTIONS` are hard-coded names in `frontend/src/constants`, and the backend matches them with `ILIKE %name%`, so they can drift from the real `departments` table. Load departments from the API instead.

## 3. Missing backend endpoints

| Needed for | Endpoint | Status |
|---|---|---|
| Subjects → Create | `POST /api/subjects` | ❌ 404 |
| Department list for forms and filters | `GET /api/departments` (and create) | ❌ 404 |
| Class edit (the Edit button is already shown) | `PUT /api/classes/:id` | ❌ 404 |
| Class delete | `DELETE /api/classes/:id` | ❌ 404 |
| Join class with an invite code | `POST /api/enrollments` or `POST /api/classes/join` | ❌ 404 |
| Class roster | `GET /api/classes/:id/enrollments` | ❌ 404 |
| Subject and user detail, update and delete | `GET/PUT/DELETE /api/subjects/:id`, `/api/users/:id` | ❌ 404 |

## 4. Bugs in existing backend code

1. [ ] **`GET /api/classes/:id` has no `try/catch`** (`backend/src/routes/classes.ts`). On a DB error, Express returns an HTML page that includes the full SQL query. Wrap it in `try/catch` and return JSON.
2. [ ] **`POST /api/classes` has no validation.** It spreads `req.body` straight into the insert. Any column can be set, invalid data reaches the DB, and on failure it returns the raw error object (`{ error: e }`). Validate with zod and pick only the allowed fields. Also, the invite code (`Math.random().toString(36).substring(2, 9)`) can collide with the unique constraint and isn't retried.
3. [ ] **Rate limiting and bot protection never run on API routes.** `app.use(securityMiddleware)` is registered after the routers in `backend/src/index.ts`, so it only protects `GET /`. Ten rapid requests to `/api/*` were all served. Move it above the `app.use('/api/...')` lines.
4. [ ] **The security middleware fails open.** When Arcjet errors (for example a missing IP), the request still proceeds. Decide on the intended behaviour.
5. [ ] **`req.user` is never set,** so the role-based rate limits (admin, teacher, student) always fall back to the guest limit. Add a middleware that reads the Better Auth session and sets `req.user`.
6. [ ] **The production build is broken:**
   - `npm run build` fails with TypeScript errors. `tsconfig.json` uses `"moduleResolution": "node"`, which cannot resolve drizzle-orm's type exports. Use `"module": "NodeNext"` with `"moduleResolution": "NodeNext"` (or `"bundler"`).
   - `npm start` runs `node dist/server.js`, but the entry file is `src/index.ts`, so the output is `dist/index.js`.
7. [ ] **The port is hard-coded** (`const PORT = 8000`). Use `process.env.PORT ?? 8000` for deployment.

## 5. Authentication and security

- [ ] **No API route requires login.** `/api/users` returns every user's name and email to anyone.
- [ ] **Anyone can register as an admin.** `role` is declared with `input: true` in `backend/src/lib/auth.ts`, so a sign-up request can set `"role": "admin"`. Allow only `student` and `teacher` at sign-up, and assign `admin` server-side.
- [ ] **No route checks the user's role.** For example, nothing limits class creation to teachers and admins.
- [ ] **The frontend has no auth wiring.** `sign-in-form.tsx`, `sign-up-form.tsx` and `forgot-password-form.tsx` exist, but nothing uses them yet:
  - no `authProvider` is passed to `<Refine>`
  - there is no Better Auth client (`createAuthClient`)
  - there are no `/login`, `/register` or `/forgot-password` routes (they render blank pages)
  - no route is protected (`<Authenticated>`)
  - requests are not sent with `credentials: 'include'`
- [ ] **Unused legacy settings.** `frontend/src/constants/index.ts` reads `VITE_API_URL`, `VITE_ACCESS_TOKEN_KEY` and `VITE_REFRESH_TOKEN_KEY` and builds a `REFRESH_TOKEN_URL`. None of these are set or used, so remove them or wire them up.

## 6. Incomplete frontend pages and UI

| Page | Issue |
|---|---|
| Dashboard (`/`) | Placeholder: shows only the text "Dashboard". |
| Subjects → Create (`/subjects/create`) | Placeholder: shows only the text "Create". No form, and no backend endpoint. |
| Class details (`/classes/show/:id`) | The "Join Class" button does nothing. The header "Edit" button points to a route that doesn't exist. The invite code and schedules are never shown. |
| Classes → List | The banner fallback `/placeholder-class.png` doesn't exist in `public/`. |
| Classes → Create | No schedule input, although `schedules` is in the schema. The backend overwrites schedules with `[]` anyway. |
| Users / Faculty | `facultySchema` exists, but there are no pages. |
| Enrollments | `enrollmentSchema` exists, but there are no pages. |
| `index.html` | The title and meta tags are still the Refine template defaults. |

**What works:** client-side validation on Classes → Create. Submitting the form empty shows an error for every field.

## 7. Type errors and code quality

- [ ] **The frontend has 3 TypeScript errors** (`npx tsc --noEmit`):
  - `components/upload-widget.tsx`: the component props have no types. Use `UploadWidgetProps` from `@/types`.
  - `pages/classes/create.tsx:129`: a consequence of the untyped `UploadWidget`.
- [ ] **Frontend types don't match the backend:**
  - `Subject.department` is typed as `string`, but the API returns a department object.
  - `ClassDetails.status` is missing `'archived'`.
  - `ClassDetails` declares `courseCode` and `courseName`, which the API never returns.
- [ ] **`backend/src/routes/users.ts` doesn't validate `role`.** It passes the value to the enum with `as any`, so an invalid role becomes a DB error and a `500` instead of a `400`.
- [ ] **Search terms aren't escaped.** The `search` value isn't escaped for `%` and `_` in `ILIKE` (the `subject`, `teacher` and `department` filters are).
- [ ] **Unused constants.** The sample `teachers` and `subjects` arrays and `constants/mock-data.ts` can probably be removed.
- [ ] **No tests.** The backend `npm test` script is a placeholder.

## 8. Running locally

**Backend:**
```bash
cd backend
npm install
# backend/.env needs: DATABASE_URL (Neon), BETTER_AUTH_SECRET, ARCJET_KEY, ARCJET_ENV=development, FRONTEND_URL=http://localhost:5173
npm run db:migrate
npm run dev          # http://localhost:8000
```

**Frontend:**
```bash
cd frontend
npm install
# frontend/.env needs: VITE_BACKEND_BASE_URL=http://localhost:8000/api/, VITE_CLOUDINARY_CLOUD_NAME, VITE_CLOUDINARY_UPLOAD_PRESET, VITE_CLOUDINARY_UPLOAD_URL
npm run dev          # http://localhost:5173
```

> If the frontend shows a Tailwind/PostCSS error overlay, apply the `vite.config.ts` fix in section 1.

## Suggested order of work

1. Fix the blockers in section 1, then run the migrations and seed a few departments, subjects and users.
2. Fix the communication bugs in section 2 and the existing-code bugs in section 4.
3. Wire up authentication and roles (section 5).
4. Build the missing endpoints (section 3) together with their pages (section 6).
5. Clean up types and add tests (section 7).
