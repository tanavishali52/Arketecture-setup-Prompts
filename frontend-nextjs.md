# Frontend architecture prompt — Next.js (App Router) + Redux Toolkit / RTK Query

Copy everything below the line into your AI coding agent. Fill in **Section 0** first; everything else is the reusable architecture and should not be changed per project.

---

## 0. Project brief (fill this in)

- **App name:** `<app name>`
- **What it does / who uses it:** `<one or two sentences>`
- **Backend:** `<base URL, e.g. http://localhost:4000/api/v1>` — response envelope `{ statusCode, message, data, errors }`: `<yes / no>`
- **Auth:** `<e.g. email + password, + emailed code / none>`
- **Roles / portals in this app:** `<e.g. customer, vendor>`
- **Pages / features to build (in order):**
  1. `<role>/<page>` — `<what it shows / does>`
  2. `<role>/<page>` — `<what it shows / does>`
- **Design source:** `<Figma link / HTML mockup / "use the default UI kit">`

You are a senior frontend engineer. Build the app described in Section 0 using **exactly** the architecture below, so every feature has the same structure, naming and rules. Do not add pages or features that are not listed in Section 0.

> Before writing code, read the docs of the installed Next.js version in `node_modules/next/dist/docs/` — APIs and conventions may differ from older versions. Follow its deprecation notices.

## 1. Stack

- Next.js 16 (App Router) + React 19 + TypeScript (strict)
- Redux Toolkit + RTK Query — **one** `createApi` for the whole app
- Tailwind CSS v4 (`@tailwindcss/postcss`), theme tokens as CSS variables, light/dark mode
- Vitest + Testing Library, ESLint (`eslint-config-next`)
- No other state library, CSS framework or component library

## 2. Folder structure (project root, no `src/`)

```
app/
  layout.tsx, globals.css, not-found.tsx, page.tsx
  (auth)/              # sign-in, sign-up … wrapped in guards/GuestOnly
  (portals)/<role>/    # one folder per role; layout = guards/RequireAuth + layouts/AppShell
features/              # role-first, then capability (§3)
components/
  ui/                  # generic UI kit: Button, Card, Modal, Table, Input, OtpInput, Skeleton, Toast …
  icons/
layouts/               # AppShell, Sidebar, Topbar, nav.tsx, Placeholder
guards/                # RequireAuth.tsx, GuestOnly.tsx (client components)
providers/             # AppProviders.tsx (store provider, theme)
store/                 # index.ts (makeStore), provider.tsx, slices/authSlice.ts, crossTabSync.ts
services/api/          # baseApi.ts (the ONE createApi)
constants/             # env.ts (only place reading NEXT_PUBLIC_*), routes.ts, apiEndpoints.ts
enums/                 # e.g. role.ts
hooks/                 # redux.ts (typed hooks), useHydrated, usePrefersReducedMotion
types/                 # shared types
utils/helper/          # apiError.ts, tokenStorage.ts …
theme/
test/                  # test helpers (e.g. an in-memory fake of the backend auth API)
docs/ARCHITECTURE.md
```

Path aliases (`tsconfig.json`): `@/*`, `@app/*`, `@components/*`, `@features/*`, `@services/*`, `@store/*`, `@constants/*`, `@utils/*`, `@hooks/*`, `@app-types/*`, `@enums/*`. No `../../..` imports.

## 3. Feature layout (role-first)

```
features/<role>/<capability>/      # e.g. features/customer/orders
  components/        # feature-local, presentational
  hooks/             # only when needed
  api.ts             # baseApi.injectEndpoints({...})
  types.ts
  <Feature>Page.tsx  # the connected container the route renders
  index.ts           # the ONLY file others import from
```

- Role-first because each role's data and endpoints are role-scoped. Capabilities truly shared by all roles (auth, profile, verification …) live in `features/<name>/`.
- `app/(portals)/<role>/<page>/page.tsx` stays thin: it only renders the feature page from `index.ts`.
- Features import other features only through `index.ts`.

## 4. API layer (one RTK Query slice)

`services/api/baseApi.ts`:
- `fetchBaseQuery({ baseUrl: env.apiBaseUrl, timeout, prepareHeaders })` — `Authorization: Bearer <token>` from the auth slice (unless already set) and `Accept: application/json`.
- **Envelope unwrap:** if the body is `{ statusCode, message, data, errors }`, the result's `data` becomes the payload (other bodies pass through). Applied to every request including refresh.
- `baseQueryWithReauth`: on 401 (except endpoints listed in `NO_REFRESH_ENDPOINTS`, e.g. login, verify code, logout) refresh **once** via `POST /auth/refresh`, share one in-flight refresh across concurrent 401s, replay; if refresh fails dispatch `loggedOut()`.
- `export const baseApi = createApi({ reducerPath: 'api', baseQuery: baseQueryWithReauth, tagTypes: [...], endpoints: () => ({}) })` — **never call `createApi` again**; features use `injectEndpoints`.
- Endpoints the backend doesn't serve yet use a `queryFn` with mock data (same hook API; swap later).

`utils/helper/apiError.ts`: `getApiErrorMessage(error, fallback)` → backend `errors[]` → backend `message` → friendly default per status (network/timeout included). Never show raw 5xx text.

**Proxy:** the browser calls the app's own `/api/*`; `next.config.ts` rewrites to `API_PROXY_TARGET` (server-only env, no `NEXT_PUBLIC_` prefix → backend address never bundled, no CORS).

## 5. State (Redux store)

- `store/index.ts` exports `makeStore()`: reducers `auth` + `baseApi`; middleware `baseApi.middleware` + a listener that writes tokens to `tokenStorage` on `tokensReceived` (localStorage with "remember me", else sessionStorage) and, on `loggedOut`, clears them and calls `baseApi.util.resetApiState()`.
- `authSlice`: `{ accessToken, refreshToken, remember, restored }`, actions `sessionRestored`, `tokensReceived`, `loggedOut`, plus selectors.
- `crossTabSync.ts`: sign-in/out in one tab updates the others.
- Typed hooks in `hooks/redux.ts`. Server data lives only in RTK Query — never copied into slices.
- Anything touching `window`/storage is a client component and uses `useHydrated()` to avoid hydration mismatches.

## 6. Auth & routing

- `features/auth/session` → `useSession()` returns `{ status: 'loading'|'authenticated'|'unauthenticated'|'error', user, retry }` from `GET /auth/me`, plus helpers `hasAnyRole`, `canUseRole`, `homeForRoles`.
- `guards/RequireAuth({ role })`: unauthenticated → `/sign-in?next=<path>`; wrong role → redirect to the user's own area; session-check error → retry card (not a logout).
- `guards/GuestOnly`: signed-in users leave the auth pages.
- Sign-in flow per Section 0 (e.g. email + password → emailed code step with resend countdown → tokens → `/auth/me` → home). Login `403` responses with a `code` show a clear message and the matching action.
- All paths in `constants/routes.ts`; navigation items in `layouts/nav.tsx`.

## 7. UI, loading, accessibility

- Reuse `components/ui`; repeated class strings become a component.
- Loading: skeletons shaped like the content for page data; full-page loader only while the session is restored.
- One `getApiErrorMessage` everywhere; `EmptyState` for empty lists; toasts for mutations.
- Tailwind tokens as CSS variables with dark values; mobile-first; respect `prefers-reduced-motion`.
- Modals trap focus and close on Escape; labelled inputs; `aria-label` on icon buttons.

## 8. Env

- `constants/env.ts` is the only reader of `process.env.NEXT_PUBLIC_*`: `NEXT_PUBLIC_API_BASE_URL` (default `/api/v1`), timeout, app name.
- `API_PROXY_TARGET` (server only) for the rewrite.
- `.env.example` documents every variable; real secrets are never committed.
- **No secrets in `NEXT_PUBLIC_*`** — they ship to the browser.

## 9. Quality

- TypeScript strict, no `any`. `npm run typecheck`, `npm run lint`, `npm test` after every non-trivial change.
- Auth flow tested against an in-memory fake backend that stubs `fetch` (kept in step with the real contract); pure helpers unit-tested.
- Keep `docs/ARCHITECTURE.md`, `features/README.md` and a short `CLAUDE.md` (rules for AI agents) updated.

## 10. Build order

1. **Foundation:** Next app, Tailwind theme, env, store + auth slice + token storage, `baseApi` (unwrap + refresh), guards, AppShell, UI kit.
2. **Auth** (per Section 0): sign-in, sign-up, session, sign-out.
3. **The pages in Section 0**, one at a time.

For each feature: types → `api.ts` (`injectEndpoints`, real or `queryFn` mock) → components → page → `index.ts` → thin `app/.../page.tsx`. Run typecheck, lint and tests after each one and fix everything before moving on. Ask before adding any dependency not listed in §1.
