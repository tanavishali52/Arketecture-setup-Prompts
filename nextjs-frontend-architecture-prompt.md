# Prompt: Build a Next.js frontend (PETS-Services-FE / PETS-Client-FE architecture)

Copy everything below the line into Claude (or any AI coding assistant). Replace the `<…>` placeholders first.

---

You are a senior frontend engineer. Build **<APP NAME>** — <one-line description, e.g. "the role-based portals (groomer, carer, vet) of a pet-services marketplace">. It talks to a NestJS backend at `<BACKEND URL>/api/v1`.

Follow the architecture below exactly. It is the architecture of an existing production codebase (PETS-Services-FE / PETS-Client-FE); keep its structure, naming and rules so every feature looks the same.

> Before writing code, read the docs of the installed Next.js version in `node_modules/next/dist/docs/` — APIs and conventions may differ from older versions. Heed deprecation notices.

## 1. Stack

- **Next.js 16 (App Router) + React 19 + TypeScript (strict)**
- **Redux Toolkit + RTK Query** (`@reduxjs/toolkit`, `react-redux`) — ONE `createApi` for the whole app
- **Tailwind CSS v4** (`@tailwindcss/postcss`), theme tokens as CSS variables, light/dark mode
- **Vitest** + Testing Library, **ESLint** (`eslint-config-next`)
- No other state library, CSS framework or component library.

## 2. Folder structure (project root, no `src/`)

```
app/
  layout.tsx, globals.css, not-found.tsx, page.tsx
  (auth)/              # sign-in, sign-up, … — wrapped in guards/GuestOnly (auth layout)
  (portals)/<role>/    # one folder per portal/role; layout wraps pages in guards/RequireAuth + layouts/PortalShell
  style-guide/
features/              # role-first, then capability (§3)
components/
  ui/                  # generic UI kit: Button, Card, Modal, Table, Input, OtpInput, Skeleton, Toast …
  icons/
layouts/               # PortalShell, Sidebar, Topbar, portalNav.tsx, PortalPlaceholder
guards/                # RequireAuth.tsx, GuestOnly.tsx (client components)
providers/             # AppProviders.tsx (Redux store provider, theme)
store/                 # index.ts (makeStore), provider.tsx, slices/authSlice.ts, crossTabSync.ts
services/api/          # baseApi.ts (the ONE createApi), endpoints
constants/             # env.ts (only place reading process.env.NEXT_PUBLIC_*), routes.ts, apiEndpoints.ts
enums/                 # e.g. portal-role.ts
hooks/                 # redux.ts (typed useAppDispatch/useAppSelector), useHydrated, usePrefersReducedMotion
types/                 # auth.ts, common.ts (shared types)
utils/helper/          # apiError.ts, tokenStorage.ts, statusBadge.ts …
utils/libs/
theme/
test/                  # test helpers, e.g. an in-memory fake of the backend's auth endpoints
docs/ARCHITECTURE.md
```

Path aliases in `tsconfig.json`: `@/*`, `@app/*`, `@components/*`, `@features/*`, `@services/*`, `@store/*`, `@constants/*`, `@utils/*`, `@hooks/*`, `@app-types/*`, `@enums/*` … No `../../..` imports.

## 3. Feature layout (role-first)

```
features/<role>/<capability>/      # e.g. features/groomer/dashboard, features/client/bookings
  components/        # feature-local, presentational
  hooks/             # feature-local hooks (only when needed)
  api.ts             # baseApi.injectEndpoints({...}) — this role+capability's endpoints
  types.ts
  <Feature>Page.tsx  # the connected container the route renders
  index.ts           # the ONLY file others import from (exports the page + public hooks)
```

- Role-first because each portal's data and endpoints are role-scoped. A capability that is truly identical across roles (e.g. `features/auth/*`, `features/verification`) lives in a shared `features/<name>/`.
- `app/(portals)/<role>/<page>/page.tsx` stays thin: it only renders the feature's page from `index.ts`.
- Features import other features only through their `index.ts`.

## 4. API layer: one RTK Query slice

`services/api/baseApi.ts`:
- `fetchBaseQuery({ baseUrl: env.apiBaseUrl, timeout, prepareHeaders })` — adds `Authorization: Bearer <token>` from the auth slice (unless the request already set one) and `Accept: application/json`.
- **Envelope unwrap:** the backend wraps successes in `{ statusCode, message, data, errors }`; wrap the fetch query so every result's `data` is the payload (bodies without the envelope pass through). This covers every endpoint and the refresh call.
- `baseQueryWithReauth`: on 401 (except `NO_REFRESH_ENDPOINTS` like login, verify code, logout) refresh ONCE via `POST /auth/refresh`, share one in-flight refresh across concurrent 401s, replay the request; if refresh fails dispatch `loggedOut()`.
- `export const baseApi = createApi({ reducerPath: 'api', baseQuery: baseQueryWithReauth, tagTypes: [...], endpoints: () => ({}) })` — **never call `createApi` a second time**; features use `baseApi.injectEndpoints`.
- Endpoints the backend doesn't serve yet use a `queryFn` with static/mock data (same hook API, swap later).

`utils/helper/apiError.ts`: `getApiErrorMessage(error, fallback)` → backend `errors[]` first, then `message`, then a friendly default per status (network/timeout messages too); never show raw 5xx text (except user-written 503s).

**Backend proxy:** the browser calls this app's own `/api/*`; `next.config.ts` rewrites it to `API_PROXY_TARGET` (no `NEXT_PUBLIC_` prefix → the backend address is never bundled, no CORS needed).

## 5. State: Redux store

- `store/index.ts` exports `makeStore()` (a new store per request/tab, App Router friendly): reducers `auth` + `baseApi`, middleware `baseApi.middleware` + a listener that, on `tokensReceived`, writes tokens to `tokenStorage` (localStorage when "Remember me", else sessionStorage) and, on `loggedOut`, clears them and `baseApi.util.resetApiState()`.
- `authSlice`: `{ accessToken, refreshToken, remember, restored }` + actions `sessionRestored`, `tokensReceived`, `loggedOut` + selectors.
- `crossTabSync.ts`: sign-in/out in one tab updates the others (storage events).
- Typed hooks in `hooks/redux.ts`. Server data lives in RTK Query only — don't copy it into slices.
- Components that touch storage/window use `"use client"` and `useHydrated()` to avoid hydration mismatches.

## 6. Auth & routing

- Session: `features/auth/session` → `useSession()` returns `{ status: 'loading'|'authenticated'|'unauthenticated'|'error', user, retry }` from `GET /auth/me`; helpers `hasAnyRole`, `canUsePortal`, `portalForRoles`.
- `guards/RequireAuth({ portal })`: unauthenticated → `/sign-in?next=<path>`; signed in to the wrong portal → redirect to their own portal; session check error → retry card (not a logout).
- `guards/GuestOnly`: signed-in users leave the auth pages.
- Sign-in: email + password (+ role/portal) → backend emails a 6-digit code (`challengeToken`) → `TwoFactorCodeStep` (OTP input, resend countdown from `resendAvailableAt`) → tokens → `/auth/me` → portal. Handle `403` codes from login (e.g. `IDENTITY_VERIFICATION_PENDING`) with a clear message and a "send the verification link again" action.
- All routes in `constants/routes.ts` (`ROUTES.auth.signIn`, `ROUTES.portals[role]` …); sidebar items in `layouts/portalNav.tsx`.

## 7. UI, loading, accessibility

- Reuse `components/ui`; repeated class strings become a component.
- Loading: skeletons shaped like the content (`Skeleton`), not spinners, for page data; full-page loader only while the session is restored.
- Errors: one `getApiErrorMessage` everywhere; empty states with `EmptyState`; toasts for mutations.
- Tailwind tokens as CSS variables with dark values; mobile-first; respect `prefers-reduced-motion`.
- Modals trap focus and close on Escape; labelled inputs; `aria-label` on icon buttons.

## 8. Env

- `constants/env.ts` is the only place that reads `process.env.NEXT_PUBLIC_*` (typed): `NEXT_PUBLIC_API_BASE_URL` (default `/api/v1`), timeout, app name.
- `API_PROXY_TARGET` (server only) for the rewrite.
- `.env` not committed (or only non-secret defaults); `.env.example` documents every variable.
- **Never put secrets in `NEXT_PUBLIC_*`** — they ship to the browser.

## 9. Quality

- TypeScript strict, no `any`. `npm run typecheck`, `npm run lint`, `npm test` after every non-trivial change.
- Tests: auth flow against an in-memory fake backend (`test/auth/fakePetsBe.ts` stubbing `fetch`) kept in step with the real contract; pure helpers unit-tested.
- Keep `docs/ARCHITECTURE.md` and `features/README.md` updated; a short `CLAUDE.md` with the rules for AI assistants.

## 10. What to build now

<List the portals/pages, e.g.:>
1. Foundation: Next app, Tailwind theme, env, store + auth slice + token storage, baseApi (unwrap + refresh), guards, PortalShell, UI kit.
2. Auth: sign-in + email code, sign-up, session, sign-out.
3. <role>/<capability> …
4. <role>/<capability> …

Work feature by feature: types → `api.ts` (injectEndpoints, real or `queryFn` mock) → components → page → `index.ts` → thin `app/.../page.tsx`. After each one run typecheck, lint and tests, and fix everything before moving on. Ask me before adding any dependency not listed in §1.
