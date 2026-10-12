# Frontend architecture prompt — React + Vite (SPA)

Copy everything below the line into your AI coding agent. Fill in **Section 0** first; everything else is the reusable architecture and should not be changed per project.

---

## 0. Project brief (fill this in)

- **App name:** `<app name>`
- **What it does / who uses it:** `<one or two sentences>`
- **Backend:** `<base URL, e.g. http://localhost:4000/api/v1>` — response envelope `{ statusCode, message, data, errors }`: `<yes / no>`
- **Auth:** `<e.g. email + password, + emailed code / none>`
- **User roles that can use this app:** `<roles>`
- **Pages / features to build (in order):**
  1. `<page>` — `<what it shows / does>`
  2. `<page>` — `<what it shows / does>`
- **Design source:** `<Figma link / HTML mockup / "use the default UI kit">`

You are a senior frontend engineer. Build the app described in Section 0 using **exactly** the architecture below, so every feature has the same structure, naming and rules. Do not add pages or features that are not listed in Section 0. Where the backend has no endpoint yet, use a clearly marked mock (see §3).

## 1. Stack

- React 19 + TypeScript (strict), built with Vite
- React Router v7 (`createBrowserRouter`, lazy routes)
- TanStack Query v5 for all server state
- Axios — exactly **one** instance
- Tailwind CSS v4 (`@tailwindcss/vite`), theme tokens as CSS variables, light/dark mode
- Vitest (jsdom per file when needed), oxlint, Prettier
- No Redux, no other CSS framework, no component library

## 2. Folder structure

```
src/
  main.tsx
  app/                 # App.tsx, providers/ (QueryClient, Theme, Router), queryClient.ts
  config/env.ts        # the ONLY file that reads import.meta.env (typed + validated)
  constants/           # routes.ts (ROUTES), roles.ts …
  routes/              # publicRoutes.tsx, protectedRoutes.tsx, lazyPages.tsx, pageChunks.ts
  layouts/             # AppShell (sidebar + topbar + <Outlet/>), Sidebar, Topbar, nav.tsx
  features/<feature>/  # one folder per feature (§3)
  components/
    ui/                # generic UI kit: Button, Card, Modal, Table, Select, Skeleton, SkeletonBlocks, Toast …
    common/            # ErrorState, QueryBoundary
    icons/             # one icon per file
  hooks/               # shared hooks (useQueryGate, useLoadingGate, usePageToast …)
  services/api/        # api_helper.ts (Axios + interceptors), api_call_helper.ts (ApiError), url_helper.ts, token_storage.ts
  mocks/               # shared mock data until the backend has endpoints
  types/               # types shared across features
  utils/               # pure helpers (+ *.test.ts)
  styles/index.css     # Tailwind + theme tokens
  theme/               # ThemeProvider, tokens
```

Path alias `@/*` → `src/*`. No `../../..` imports across folders.

## 3. Feature layout

```
src/features/<feature>/
  api/<feature>.api.ts     # calls the backend through api_helper (or a mock, clearly marked)
  hooks/                   # useX (useQuery) / useXActions (useMutation) + <feature>QueryKeys.ts
  components/              # feature components (+ <Feature>Skeleton.tsx)
  pages/<Feature>Page.tsx  # route component: composes hooks + components
  utils/<feature>.ts       # pure logic (+ <feature>.test.ts)
  types.ts
```

- **Boundaries:** features don't import each other. Exception: a *page* may compose another feature's hooks and map the data to its own types. Layouts may read feature hooks (e.g. nav badge counts).
- Shared things move up: generic UI → `components/ui`, shared types → `src/types`, shared mock data → `src/mocks`.
- **Mocks** keep the real function signatures, resolve after a small delay, and carry `TODO(backend): replace with <METHOD> <endpoint>`.

## 4. API layer (one Axios instance)

`services/api/api_helper.ts`:
- `axios.create({ baseURL: env.apiBaseUrl, timeout: env.apiTimeoutMs })`
- request interceptor adds `Authorization: Bearer <accessToken>` from `token_storage`
- response interceptor:
  - **success:** if the body is `{ statusCode, message, data, errors }`, return `data` (other bodies pass through)
  - **401:** refresh **once** (`POST /auth/refresh` with plain axios, also unwrapped); concurrent 401s wait on the same refresh; replay the request; if refresh fails → clear tokens and call a registered `onUnauthorized` handler (registered from `App.tsx`, so this file never imports React or the router)
  - request flags: `skipAuthRefresh` (login/logout), `retriedAfterRefresh` (never retry twice)
  - every error → `normalizeApiError()` → throw `ApiError`
- typed helpers `get<T>`, `post<T, D>`, `put`, `patch`, `del` resolve to the body. Nothing else imports axios.

`services/api/api_call_helper.ts`:
- `class ApiError { kind: 'http'|'network'|'timeout'|'unknown'; status; message; details[] }`
- message priority: backend `errors[]` → backend `message` → friendly default per status. Never show raw 5xx text.
- `getApiErrorMessage(error)`, `isRetryableError(error)` (network, timeout, 429, 5xx)

`url_helper.ts`: every endpoint path as a constant. `token_storage.ts`: tokens in localStorage ("remember me") or sessionStorage.

## 5. Server state (TanStack Query)

- `queryClient.ts`: `staleTime: 30s`, `refetchOnWindowFocus: false`, retry only when `isRetryableError`; `Register { defaultError: ApiError }` so `error` is typed everywhere.
- Query keys per feature (`all`, `list(filters)`, `detail(id)`); mutations invalidate them in `onSuccess`.
- **The URL is the source of truth** for filters, tabs and page (`useSearchParams`).
- Join records by **id**, never by display name.

## 6. Loading, errors, empty states

- **Skeletons** for all data loading — no spinners or full-screen loaders for page data.
- One logic (`hooks/useQueryGate.ts`), two entry points:
  - pages: `const gate = useLoadingGate([query], { skeleton: <XSkeleton /> }); if (gate || !query.data) return gate`
  - sections / cards / modal bodies: `<QueryBoundary queries={[q]} skeleton={…}>{() => content}</QueryBoundary>`
  - behaviour: skeleton on first load and on every visit (cached data refetched once on mount), `ErrorState` with "Try again" on failure, current data kept on screen for later refetches.
- Each feature has a `<Feature>Skeleton.tsx` built from `components/ui/SkeletonBlocks.tsx` (stat card, chips, table with real headers, section card, key/value rows). Static text renders for real.
- `EmptyState` for empty lists; toasts for mutation success/failure.
- Full-screen loader only for app boot (first session check).

## 7. Routing & auth

- `createBrowserRouter` with public routes and protected routes (`RequireAuth` → `AppShell`).
- Pages are lazy chunks, preloaded in the background after the shell mounts; Suspense fallback = generic page skeleton inside the content area (shell stays).
- `RequireAuth` (roles) / `GuestOnly`. A failed session check (network/5xx, not 401) shows a retry screen instead of logging out.
- Session: `GET /auth/me` via `useCurrentUser()`. Sign-out asks for confirmation.
- Navigation defined once in `layouts/nav.tsx`; placeholder routes generated from it for pages not built yet.

## 8. UI, styling, accessibility

- Reuse `components/ui` before writing markup; repeated class strings become a component.
- Tailwind tokens (`bg-surface`, `text-muted`, `border-border` …) as CSS variables with dark values; mobile-first.
- Modal: `role="dialog"`, focus trap + restore, Escape closes, body scroll lock.
- Toggle groups `role="group"` + `aria-pressed`; labelled inputs; `aria-label` on icon buttons; respect `prefers-reduced-motion`.

## 9. Env

- Only `.env` (shared non-secret defaults) and `.env.example` (documents every variable); personal overrides in `.env.local` (git-ignored).
- `VITE_API_BASE_URL` (required; in dev `/api/v1` proxied by `vite.config.ts` to `API_PROXY_TARGET`), `VITE_API_TIMEOUT_MS`, `VITE_APP_NAME`.
- Staging/production set `VITE_API_BASE_URL` in CI/CD; the build fails in CI when it is empty.
- **No secrets in env files** — every `VITE_*` value ships to the browser.

## 10. Quality

- TypeScript strict, no `any`. oxlint with 0 warnings (incl. `rules-of-hooks`, `only-export-components`).
- Vitest for pure helpers and the API layer (fake Axios adapter for 401 → refresh → retry).
- `npm run lint`, `npm test`, `npm run build` pass before every PR.
- Keep `ARCHITECTURE.md` and `README.md` updated.

## 11. Build order

1. **Foundation:** Vite + Tailwind theme, env, API layer, QueryClient, router, AppShell, UI kit, skeleton blocks, QueryBoundary.
2. **Auth** (per Section 0), RequireAuth, sign-out.
3. **The pages in Section 0**, one at a time.

For each feature: types → api (real or mock) → query keys + hooks → skeleton → components → page → route. Run lint, tests and build after each one and fix everything before moving on. Ask before adding any dependency not listed in §1.
