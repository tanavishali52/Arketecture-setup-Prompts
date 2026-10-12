# Prompt: Build a React (Vite) frontend (PETS-Admin-FE architecture)

Copy everything below the line into Claude (or any AI coding assistant). Replace the `<…>` placeholders first.

---

You are a senior frontend engineer. Build the frontend for **<APP NAME>** — <one-line description, e.g. "the admin portal for a pet-services marketplace">. It talks to a NestJS backend at `<BACKEND URL>/api/v1`.

Follow the architecture below exactly. It is the architecture of an existing production codebase (PETS-Admin-FE); keep its structure, naming and rules so every feature looks the same.

## 1. Stack

- **React 19 + TypeScript (strict)**, built with **Vite**
- **React Router v7** (`createBrowserRouter`, lazy routes)
- **TanStack Query v5** for all server state
- **Axios** — exactly ONE instance
- **Tailwind CSS v4** (`@tailwindcss/vite`), theme tokens as CSS variables, light/dark mode
- **Vitest** (+ jsdom per file when needed), **oxlint**, **Prettier**
- No Redux, no other CSS framework, no component library.

## 2. Folder structure

```
src/
  main.tsx
  app/                 # App.tsx, providers/ (QueryClientProvider, ThemeProvider, Router), queryClient.ts
  config/env.ts        # the ONLY file that reads import.meta.env (typed + validated)
  constants/           # routes.ts (ROUTES), roles.ts …
  routes/              # publicRoutes.tsx, protectedRoutes.tsx, lazyPages.tsx, pageChunks.ts
  layouts/             # PortalShell (sidebar + topbar + <Outlet/>), Sidebar, Topbar, portalNav.tsx
  features/<feature>/  # one folder per feature (§3)
  components/
    ui/                # generic UI kit: Button, Card, Modal, Table, Select, Skeleton, SkeletonBlocks, Toast …
    common/            # app-level: ErrorState, QueryBoundary
    icons/             # one icon per file
  hooks/               # shared hooks (useQueryGate, useLoadingGate, usePageToast …)
  services/api/        # api_helper.ts (Axios + interceptors), api_call_helper.ts (ApiError), url_helper.ts (endpoints), token_storage.ts
  mocks/               # shared mock data until the backend has endpoints
  types/               # types shared across features
  utils/               # pure helpers (format.ts …) + their *.test.ts
  styles/index.css     # Tailwind + theme tokens
  theme/               # ThemeProvider, tokens
```

Path alias: `@/*` → `src/*`. No `../../..` imports across folders.

## 3. Feature layout

```
src/features/<feature>/
  api/<feature>.api.ts     # functions that call the backend through api_helper (or MOCK, clearly marked)
  hooks/                   # useX (useQuery) / useXActions (useMutation) + <feature>QueryKeys.ts
  components/              # feature components (presentational where possible)
  pages/<Feature>Page.tsx  # route component: composes hooks + components
  utils/<feature>.ts       # pure logic (+ <feature>.test.ts)
  types.ts
```

Rules:
- **Feature boundary:** features don't import each other. Exception: a *page* may compose another feature's hooks (and map the data to its own types). Layouts may read feature hooks (e.g. sidebar badge counts).
- Shared things go up: generic UI → `components/ui`, shared types → `src/types`, shared mock data → `src/mocks`.
- Mock APIs keep the real function signatures, return data after a small delay, and carry `TODO(<BACKEND>): replace with <METHOD> <endpoint>`.

## 4. API layer (one Axios instance)

`services/api/api_helper.ts`:
- `axios.create({ baseURL: env.apiBaseUrl, timeout: env.apiTimeoutMs })`
- request interceptor: `Authorization: Bearer <accessToken>` from `token_storage`
- response interceptor:
  - **success:** unwrap the backend envelope `{ statusCode, message, data, errors }` → return `data` (bodies without an envelope pass through)
  - **401:** refresh ONCE (`POST /auth/refresh` with plain axios, also unwrapped), queue concurrent 401s on the same refresh promise, replay the request; if refresh fails → clear tokens + call the registered `onUnauthorized` handler (set from `App.tsx`, so this file never imports React/router)
  - per-request flags: `skipAuthRefresh` (login/logout), `retriedAfterRefresh` (never retry twice)
  - every error → `normalizeApiError()` → throw `ApiError`
- typed helpers: `get<T>`, `post<T, D>`, `put`, `patch`, `del` → resolve to the body. Nothing else imports axios.

`services/api/api_call_helper.ts`:
- `class ApiError { kind: 'http'|'network'|'timeout'|'unknown'; status; message; details[] }`
- message: backend `errors[]` first (specific reasons), then `message`, then a friendly default per status; never show raw 5xx text (except messages written for users, e.g. 503)
- `getApiErrorMessage(error)`, `isRetryableError(error)` (network, timeout, 429, 5xx)

`url_helper.ts`: every endpoint path as a constant. `token_storage.ts`: access/refresh tokens (localStorage with "remember me", else sessionStorage).

## 5. Server state: TanStack Query

- `queryClient.ts`: `staleTime: 30s`, `refetchOnWindowFocus: false`, retry only when `isRetryableError`; `Register { defaultError: ApiError }` so `error` is typed everywhere.
- Query keys per feature in `<feature>QueryKeys.ts` (`all`, `list(filters)`, `detail(id)`).
- Mutations invalidate the feature's keys in `onSuccess`.
- **URL is the source of truth for filters/tabs/page** (`useSearchParams`), so refresh/back/share keep them.
- Join records by **id**, never by display name.

## 6. Loading, errors, empty states

- **Skeletons everywhere** for data loading — never a spinner or full-screen loader for page data.
- One logic, `hooks/useQueryGate.ts`, used two ways:
  - pages: `const gate = useLoadingGate([query], { skeleton: <XSkeleton/> }); if (gate || !query.data) return gate`
  - sections/cards/modals: `<QueryBoundary queries={[q]} skeleton={…}>{() => content}</QueryBoundary>`
  - shows the skeleton on first load AND on every visit (refetch cached data once on mount), `ErrorState` with "Try again" on failure, keeps data on screen for later refetches.
- Each feature has `components/<Feature>Skeleton.tsx` built from `components/ui/SkeletonBlocks.tsx` (stat card, chips, table with real headers, section card, key/value rows). Static text (intros, headers, labels) renders for real.
- `EmptyState` when a list is empty; toasts (`usePageToast`) for mutation success/failure.
- Full-screen brand loader ONLY for app boot (first session check).

## 7. Routing & auth

- `createBrowserRouter`; public routes (sign-in, style guide) and protected routes (inside `RequireAuth` → `PortalShell`).
- Pages are lazy chunks (`routes/lazyPages.tsx`), preloaded in the background after the shell mounts (`routes/pageChunks.ts`); Suspense fallback = generic page skeleton in the content area (sidebar stays).
- `RequireAuth` (roles) / `GuestOnly`; a failed session check (network/5xx, not 401) shows a retry screen instead of logging out.
- Sign-in: email + password → emailed 6-digit code (`challengeToken`) → tokens → `GET /auth/me` (`useCurrentUser`). Sign-out asks for confirmation.
- Sidebar nav defined once in `layouts/portalNav.tsx`; placeholder routes generated from it for pages not built yet.

## 8. UI kit, styling, accessibility

- Reuse `components/ui` before writing markup; repeated class strings become a component.
- Tailwind tokens (`bg-surface`, `text-muted`, `border-border` …) defined as CSS variables in `styles/index.css` with dark-mode values; mobile-first.
- Modal: `role="dialog"`, focus trap + focus restore, Escape closes, body scroll lock.
- Chip groups `role="group"` + `aria-pressed`; inputs have labels; icon buttons have `aria-label`; respect `prefers-reduced-motion`.

## 9. Env

- Only `.env` (shared, non-secret defaults) and `.env.example` (documents every variable). Personal overrides in `.env.local` (git-ignored).
- `VITE_API_BASE_URL` (required; `/api/v1` proxied in dev via `API_PROXY_TARGET` in `vite.config.ts`), `VITE_API_TIMEOUT_MS`, `VITE_APP_NAME`.
- Staging/production get `VITE_API_BASE_URL` from CI/CD. The build fails in CI if it's empty.
- **Never put secrets in env files** — every `VITE_*` value ships to the browser.

## 10. Quality

- TypeScript strict, no `any`. oxlint 0 warnings (incl. `rules-of-hooks`, `only-export-components` → keep non-components out of component files).
- Vitest for pure helpers and the API layer (fake Axios adapter for the 401 → refresh → retry flow).
- `npm run lint`, `npm test`, `npm run build` must pass before every PR.
- Keep `ARCHITECTURE.md` and `README.md` updated.

## 11. What to build now

<List the pages/features, e.g.:>
1. Foundation: Vite + Tailwind theme, env, API layer, QueryClient, router, PortalShell, UI kit, skeleton blocks, QueryBoundary.
2. Auth: sign-in + email code, RequireAuth, sign-out.
3. <page 1> …
4. <page 2> …

Work feature by feature: types → api (real or mock) → query keys + hooks → skeleton → components → page → route. After each one run lint, tests and build, and fix everything before moving on. Ask me before adding any dependency not listed in §1.
