# POST_MORTEM: Frontend structure, testing infrastructure and TDD on auth logic

Scope: engineering memory for the `assets-exchange-monorepo` (Turborepo; `apps/web` Next.js 14 + Mantine + Redux Toolkit, `apps/api` Express 5 + Vitest + supertest, `packages/contracts`) and for the project being migrated into it (`otc-web`). Reference repo studied for structure: `coin-lab` (Vue 3, domain modules).
Status legend used below: **DECIDED** (implemented in the repo), **RECOMMENDED** (agreed direction, not yet enforced), **OPEN** (decision pending).

---

## Session 2026-10-20 — [mixed] (feature + refactor + debug + research)

Work types in this session: test-infrastructure bootstrap (feature), TDD on routing/auth logic (feature), dependency and lockfile repair (debug), package rename fallout (refactor), frontend folder-structure research (research).

### 1. [Architectural decisions]

#### 1.1 Testing stack in `apps/web` — DECIDED
- Runner: **Jest 30 via `next/jest`** (SWC transform for TS/JSX/ESM is added by the `nextJest` wrapper; there is no `ts-jest`/`babel-jest` in the config). Default environment `jsdom`.
- Vitest stays in `apps/api` and `packages/contracts`. One runner per package; no mixing in the same package.
- Dev dependencies added to `apps/web`: `jest`, `jest-environment-jsdom`, `@testing-library/react`, `@testing-library/dom`, `@testing-library/jest-dom`, `@testing-library/user-event`, `@types/jest`, `msw`, `@edge-runtime/jest-environment`.
  - `@edge-runtime/jest-environment` currently sits in the **root** `package.json`; it belongs to `apps/web` (see backlog).
- Files: `apps/web/jest.config.ts`, `apps/web/test/jest.setup.ts`, `apps/web/test/sanity.test.ts`, `apps/web/test/mantine.test.tsx` (smoke test that Mantine renders under the DOM mocks), tests colocated with code (`src/lib/authRedirect.test.ts`, `src/middleware.test.ts`).
- Scripts: `test: jest`, `test:watch: jest --watch`, `typecheck: tsc --noEmit`. Turbo: `test` depends on `^typecheck`.
- Test status at the last verification: 4 suites, 27 tests green, `tsc --noEmit` clean.

Canonical config (verified):
```ts
// apps/web/jest.config.ts
import nextJest from 'next/jest.js';

export default nextJest({ dir: './' })({
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/test/jest.setup.ts'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
});
```
```ts
// apps/web/test/jest.setup.ts
import '@testing-library/jest-dom';

// DOM mocks only for jsdom; server-side test environments have no `window`
if (typeof window !== 'undefined') {
  Object.defineProperty(window, 'matchMedia', {
    writable: true,
    value: (query: string) => ({
      matches: false, media: query, onchange: null,
      addListener: jest.fn(), removeListener: jest.fn(),
      addEventListener: jest.fn(), removeEventListener: jest.fn(),
      dispatchEvent: jest.fn(),
    }),
  });

  class ResizeObserverMock { observe() {} unobserve() {} disconnect() {} }
  window.ResizeObserver = ResizeObserverMock;
}
```

#### 1.2 Per-file test environments — DECIDED
- Component tests: default `jsdom` + RTL.
- Middleware / server tests: first line of the file `/** @jest-environment @edge-runtime/jest-environment */`. Next middleware runs in the Edge Runtime, so this environment catches use of Node-only APIs that a plain `node` environment would let through. Plain `@jest-environment node` is acceptable for logic that needs only Web API globals (`Request`, `Response`, `Headers`), but it does not verify Edge compatibility.
- Edge compatibility is ultimately verified by `next build` and e2e tests, not by unit tests.

#### 1.3 Functional core / imperative shell for routing logic — DECIDED
- `src/lib/authRedirect.ts` exports the pure function `resolveAuthRedirect(pathname: string, hasSession: boolean): string | null`.
- `src/middleware.ts` is a thin shell: reads the cookie, calls the pure function, returns `NextResponse.redirect(new URL(target, req.url))` or `NextResponse.next()`.
- Pure logic is tested with a table (`it.each`) grouped as NO SESSION / HAS SESSION × PUBLIC PATH / PRIVATE PATH. The shell has a separate test file asserting HTTP-level behaviour only (status 307, `Location` pathname, pass-through = status 200 and no `Location`).
- Public-path matching compares the **first path segment by equality** (so `/signin/`, `/password-recovery/confirm` are public; `/sign`, `/s`, `/recovery`, `/signinfoo` are not).

Canonical implementation:
```ts
const PUBLIC_PATHS = ['/signin', '/signup', '/password-recovery'];
const firstSegment = (path: string) => path.split('/')[1] ?? '';
const PUBLIC_SEGMENTS = PUBLIC_PATHS.map(firstSegment);

function isPublicPath(pathname: string): boolean {
  const segment = firstSegment(pathname);
  return segment !== '' && PUBLIC_SEGMENTS.includes(segment);
}

export function resolveAuthRedirect(pathname: string, hasSession: boolean): string | null {
  const isPublic = isPublicPath(pathname);
  if (!hasSession && !isPublic) return '/signin';
  if (hasSession && isPublic) return '/';
  return null;
}
```
Canonical middleware shell:
```ts
export default function middleware(req: NextRequest) {
  const target = resolveAuthRedirect(req.nextUrl.pathname, req.cookies.has(AUTH_COOKIE));
  return target ? NextResponse.redirect(new URL(target, req.url)) : NextResponse.next();
}
```
Middleware test skeleton:
```ts
/** @jest-environment @edge-runtime/jest-environment */
import { NextRequest } from 'next/server';
import middleware from '@/middleware';

const ORIGIN = 'http://localhost:3000'; // only a base for URL construction; no network is used
const makeReq = (path: string, cookie?: string) =>
  new NextRequest(new URL(path, ORIGIN), { headers: cookie ? { cookie } : {} });
const locationPath = (res: Response) => new URL(res.headers.get('location')!).pathname;
const expectPassThrough = (res: Response) => {
  expect(res.status).toBe(200);
  expect(res.headers.get('location')).toBeNull();
};
```

#### 1.4 Session ownership and next-auth removal — DECIDED (implementation of the backend side pending)
- The backend owns the session (HttpOnly cookie, currently read as `SESSION_ID`). `next-auth` and its route/config were removed from `apps/web`; it duplicated session state and its stubbed Credentials provider ignored input.
- Recommended replacement pattern: Server Action / route handler forwards `Set-Cookie` from the API (or a rewrite `/api/*` → backend for same-origin cookies); Server Components obtain the user through one `getSession()` function (cookie forward + `GET /me`); middleware only does an optimistic cookie-presence check; real validation stays on the backend.
- Cookie contract (name, flags `HttpOnly; Secure; SameSite=Lax; Path=/`) must live in `packages/contracts`. A cookie must never be named after a secret (`SESSION_SECRET` is an env-var name for a signing key, not a cookie name).

#### 1.5 Dependency and workspace state — DECIDED
- Workspace packages were renamed from `@otc/*` to `@assets-exchange-monorepo/*`; all imports, `next.config.mjs` (`transpilePackages`), READMEs and `--filter` commands must use the new scope.
- All `@mantine/*` packages resolve to **7.17.8** (core, hooks, form, dropzone, notifications). The lockfile was regenerated from scratch; `npm ci` on a clean checkout succeeds. Mantine was raised from 7.5.1 within the 7.x line, so a UI smoke check is still owed (backlog).

#### 1.6 Frontend folder-structure research — RECOMMENDED / OPEN
Starting point (`otc-web`): `src/app/(authLayout)/signin/page.tsx` + `src/containers/Auth/{SignIn,SignUp,PasswordRecovery}/…` with a `child/` subfolder per page and barrel `index.ts` at every level (typos: `Sing…` for `Sign…`).

Options evaluated:
| Option | Shape | Strength | Weakness |
|---|---|---|---|
| 1. Improved containers | `containers/Auth/SignIn/components/…`, tests/styles colocated | smallest migration | no place for shared schemas/hooks/api of a domain |
| 2. Feature-based | `features/auth/{screens,components,schemas,hooks,api,lib}` + public `index.ts`, thin `app/` | scales, enforceable by lint, easy refactors | needs discipline about boundaries |
| 3. Colocation in `app/` | `app/(group)/route/{page.tsx,_components,_lib}` | Next-native, local | routing mixed with code; promotion needed on reuse |
| FSD | layers `app, pages, widgets, features, entities, shared` (+ deprecated `processes`) with downward-only imports and slice public APIs | strict, linted (Steiger) | learning curve; feature/entity debates; Next `app`/`pages` name clash |

Observations on `coin-lab` (Vue, inferred from file paths only; imports not inspected): domain modules `modules/<domain>/{constants,domain,features,infrastructure,ui}` + an `app` layer. This is **domain-based with inner DDD/hexagonal-style layering, not FSD**. The word `features` there means composables with scenario logic, which differs from the FSD `features` layer. Risks noted: up to 7 nesting levels, a cross-cutting `useApiFetch` living in `app/features`, no `shared` layer, barrel files at every level.

Direction: target Option 2 or FSD (team decision pending), reach it incrementally through Option 1, keep component-private files colocated. Auth in FSD terms: `pages/sign-in`, `features/auth-by-credentials` (`ui`, `model`, `api`), `entities/session` (holds `resolveAuthRedirect`), `shared/*`.

Next.js + FSD constraint: FSD layers `app` and `pages` collide with Next directories. Real routing goes in the package-root `app/` (thin re-exports from `src/pages/...`), with a root `pages/` stub so Next does not treat `src/pages` as the Pages Router. **Unverified:** where `middleware.ts` must live under this layout; verify against the current Next docs and with a test before committing to it.

#### 1.7 Architecture vocabulary captured
- **Hexagonal / ports & adapters**: core owns interfaces (ports), infrastructure implements them (adapters); dependencies point inward. **DDD**: bounded contexts, ubiquitous language, entities/value objects/aggregates/repositories. Complementary: DDD models the domain, hexagonal controls dependency direction.
- **FSD** organizes frontend code by layer responsibility and then by slice; it does not require ports/adapters. They can be combined (an interface in `shared/api` or `entities/*/model`, implementation injected).
- Frontend consensus: group by feature/domain, give each module a public entry point, enforce import direction by lint. Full hexagonal on the frontend is contested (benefit: framework-obsolescence insulation; cost: ceremony); apply ports only on volatile boundaries (API client, storage).
- Backend conventions (for cross-reference): FastAPI — domain packages under `src/<domain>` for growing projects, file-type layout acceptable for small services; Go — no official layout, prefer flat package-by-feature, `internal/` enforced by the compiler, minimal `main.go`; Dart server — framework dictates structure (Serverpod full-stack with generated client; Dart Frog file-based routes; Shelf minimal); Elixir/Phoenix — contexts as bounded contexts, OTP supervision trees, behaviours as ports, "functional core, imperative shell".

#### 1.8 Next.js / React knowledge consolidated in this session
- Server vs Client Components: server by default; `'use client'` marks a boundary (the file and everything it imports ship to the client); client components are still server-rendered and then hydrated; props crossing the boundary must be serializable; server components can be passed to client components as `children`.
- Pages → App Router mapping: `getServerSideProps` → async server component + `dynamic = 'force-dynamic'` or `cache: 'no-store'`; `getStaticProps`+`revalidate` → `fetch(..., { next: { revalidate } })` / `export const revalidate`; `getStaticPaths` → `generateStaticParams`; `<Head>` → `generateMetadata`; `_app/_document` → `layout.tsx`; `next/router` → `next/navigation`; `req/res` → `cookies()/headers()`. Cache defaults differ by version (Next 14 caches `fetch` by default, Next 15 does not; `params`/`cookies()` become async in 15).
- Incremental migration: `pages/` and `app/` coexist; safety net first (e2e/visual/characterization tests + metric baseline), then foundation (root layout, client provider wrapper, dependency audit), then routes by ascending risk, each behind a flag or per-brand rollout with a revert path; advance on green tests, QA sign-off and non-regressing metrics.
- Multi-brand: brand resolved per request from host; typed, schema-validated config with short cache and `revalidateTag` from the admin, fallback to a default brand; theming through CSS variables; feature flags as a typed helper with safe default (off); i18n by locale segment.
- LCP investigation order: clarify field vs lab data and what changed; identify the LCP element; split into TTFB / resource load delay / resource load duration / render delay; fix the dominant part; verify on RUM. Monitoring: RUM with alerts on p75, release markers, Lighthouse CI budgets in PRs, synthetic checks.
- Docker for Next: multi-stage build, `output: 'standalone'`, runtime env for server variables, build args for `NEXT_PUBLIC_*`, non-root user, healthcheck.

#### 1.9 React patterns practiced
Functional state updates (`setX(prev => …)`), immutable updates, closure snapshots in effects, `AbortController` cleanup against out-of-order responses, `useDebouncedValue`, `useDeferredValue` + `React.memo` + `useMemo` for large lists, SSR-safe `useLocalStorage`, RTL query priorities (role/label over class).

---

### 2. [Production Guardrails]

**Testing and tooling**
1. Run frontend tests per workspace (`npm test -w @assets-exchange-monorepo/web`, or from inside `apps/web`). Never run Jest from the monorepo root; the root has no Jest config, so TypeScript/ESM files fail with "Must use import to load ES Module". Paths passed to Jest are relative to the package.
2. Every path in `jest.config.ts` that points into the package uses `<rootDir>`. `'test/jest.setup.ts'` or `'test/src/$1'` without `<rootDir>` is a defect. The `@/` alias in `moduleNameMapper` must mirror `tsconfig` paths and map to `<rootDir>/src/$1`.
3. `expect` calls live inside `it`/`test`. A file with none is a failed suite.
4. Any test file containing JSX has the `.tsx` extension.
5. A test file that imports `next/server` (middleware, route handlers) declares its environment with a docblock as the first comment in the file: `@edge-runtime/jest-environment` for middleware, `node` only for plain server logic. jsdom has no `Request`/`Response`.
6. Everything in `jest.setup.ts` that touches `window` is guarded by `typeof window !== 'undefined'`; the setup file runs for every environment.
7. DOM mocks for Mantine are fixed: `matchMedia` is a function returning an object implementing the media-query-list interface; `ResizeObserver` is a **class** assigned to `window.ResizeObserver` (not an instance, not `new MediaQueryList()`). Components under test are rendered through `renderWithProviders` (wraps `MantineProvider`).
8. RTL queries by role/label/text in this priority; for required Mantine fields the label contains an asterisk, so match with an anchored regex (`/^email/i`), and anchor `/^password/i` so it does not also match the visibility-toggle button. Interactions through `userEvent.setup()`; asynchronous UI through `findBy*`; absence through `queryBy*`.
9. Tests assert observable contract (HTTP status, `Location` pathname, rendered text, call arguments), not framework internals. Do not assert Next-internal headers (`x-middleware-next`, `x-middleware-rewrite`). Do not hard-code a host in expected values; compare `new URL(location).pathname`.
10. Every test expectation is checked against the written spec before the test is committed; a red test must fail because of missing behaviour, not because the expectation is wrong.

**Security-relevant routing/auth logic**
11. Authorization decisions never use substring checks (`includes`, `startsWith` on raw prefixes). Compare whole path segments by equality.
12. Every allow-list (public paths) has negative tests for prefix-like and suffix-like private paths (`/sign`, `/s`, `/recovery`, `/signinfoo`) and for nested private paths (`/dashboard/settings`).
13. Middleware denies access with `NextResponse.redirect` (307). `NextResponse.rewrite` is never used for auth redirects: it keeps the original URL and returns 200.
14. The auth cookie name is a single exported constant (`AUTH_COOKIE`) used by middleware and tests; presence is checked with `req.cookies.has(...)`.
15. The backend owns the session. No second session library in the frontend. Cookie flags `HttpOnly; Secure; SameSite=Lax; Path=/`; the cookie contract is declared in `packages/contracts`.

**Code hygiene**
16. No `console.log` in committed source or tests.
17. Pure decision logic is extracted from framework wrappers; wrappers stay thin and are tested at the HTTP/render contract level.
18. Test file per module under test (no middleware tests inside `authRedirect.test.ts`).

**Dependencies and monorepo**
19. All `@mantine/*` packages share one exact version; the lockfile is the source of truth. After changing any of them run a clean `npm ci` and `npm ls @mantine/core @mantine/hooks @mantine/form @mantine/notifications @mantine/dropzone`.
20. `--force` and `--legacy-peer-deps` are not used to silence peer conflicts. Fix the version mismatch.
21. Deleting `node_modules` + `package-lock.json` and reinstalling is the approved repair for a corrupted lockfile; the regenerated lock is committed and verified with `npm ci`.
22. A workspace rename is complete only when `grep -rn "<old scope>/"` over `apps`, `packages`, `next.config.*`, READMEs and CI files returns nothing and `npm run typecheck` passes.

**Git workflow**
23. TDD commits: tests first as a separate commit (red), then a commit that changes only source (green), then an optional refactor commit. Do not add new tests inside the fix commit. Commit messages name what the diff actually changes (a commit that edits `authRedirect.test.ts` is not a "middleware" commit). Prefixes: `test`, `feat`, `fix`, `refactor`, `chore`.
24. Typos in names (`Sing…`) are fixed by one mechanical IDE rename commit before migrating code.

**Structure and imports (frontend)**
25. Placement rule: used by one component → next to it; used by several components of one feature → the feature's `components|hooks|lib|schemas`; used by several features without business logic → `components/ui` or `lib`; route glue only → `app/**/page.tsx`. Promote code one level up on its second use.
26. Component folders are `PascalCase`, contain `Component.tsx`, `Component.module.css`, `Component.test.tsx`; folders named `child` are not used (use `components`).
27. Named exports everywhere except files Next requires to be default (`page`, `layout`, `route`, config). Barrel `index.ts` only at a module's public root; never `export *`, never chains of barrels.
28. Cross-module imports use the `@/` alias; relative imports only inside one module/folder. Deep imports into another feature/slice are forbidden (`@/features/*/*`). Lint rules start as `warn` and are promoted to `error`.
29. If FSD is adopted: imports go strictly downward between layers, slices of one layer do not import each other, each slice exposes a public `index.ts`, boundaries are enforced by Steiger or `eslint-plugin-boundaries`. Start with `app, pages, features, shared`; add `entities`/`widgets` when real reuse appears.

**Next.js / React runtime rules**
30. Components are Server Components by default; `'use client'` only at the leaves that need state, effects, handlers or browser APIs. Secrets are read only on the server.
31. Brand/tenant values are not read from `process.env` inside statically rendered pages (they are inlined at build time). Resolve per request (host/headers) or build per brand. `NEXT_PUBLIC_*` values are build-time and are passed as Docker build args.
32. No module-level singleton store on the server (state would leak between requests); create the store per request inside a provider.
33. Each server `fetch` states its caching policy explicitly; the project's Next version determines the default (14: cached, 15: not cached).
34. Theme values (colors, fonts, radii) are CSS custom properties set on `<html>` from a schema-validated config (hex regex), with defaults in `:root`, semantic tokens in between brand tokens and components, contrast checked. React Context is reserved for non-visual data (flags, copy). CDN caches HTML per host.
35. State updates are immutable; any update that depends on previous state uses the functional form; effects that fetch abort in cleanup (`AbortController`) and ignore `AbortError`.
36. Browser-storage hooks return the same value as the server render on first render, read storage in `useEffect`, write only after the initial read, and track which key the loaded value belongs to (`loadedKey`) so a key change cannot copy the old value to the new key.
37. Docker images: multi-stage, `output: 'standalone'`, non-root `USER`, server config via runtime env, `healthcheck` + `depends_on: condition: service_healthy` in compose, compose key is `ports` (not `port`).

---

### 3. [Lessons, Pitfalls and Troubleshooting]

| # | Symptom | Cause | Repair |
|---|---|---|---|
| 1 | `Validation Error: Module <rootDir>/test/jest.setup.ts in setupFilesAfterEnv was not found` | (a) config path written without `<rootDir>`; (b) in a Linux sandbox the file was present but Jest 30's resolver could not load its optional native binding (lockfile created on another platform) | (a) use `<rootDir>/test/jest.setup.ts`. (b) `npm i --no-save @unrs/resolver-binding-linux-x64-gnu`, or regenerate the lock on the target platform |
| 2 | `Must use import to load ES Module: …authRedirect.test.ts` | Jest started from the repo root: no `jest.config.ts` there, so no SWC transform | run via workspace (`npm test -w …`) or `cd apps/web`; pass paths relative to the package |
| 3 | `ReferenceError: Request is not defined` at `import 'next/server'` | the file ran in jsdom, which lacks fetch-API globals | move middleware tests to their own file with a `@jest-environment` docblock as the very first comment |
| 4 | `ReferenceError: window is not defined` from `jest.setup.ts` | setup runs in every environment | wrap DOM mocks in `if (typeof window !== 'undefined')` |
| 5 | `Your test suite must contain at least one test` | `expect(...)` placed outside `it` | wrap in `it('…', () => { … })` |
| 6 | `TS1109: Expression expected` in `mantine.test.ts`; Jest cannot run the suite | JSX inside a `.ts` file | rename to `.tsx` (`git mv`) |
| 7 | `window.matchMedia = () => new MediaQueryList()` / `window.ResizeObserver = new ResizeObserver(...)` throw | `MediaQueryList` is not constructible; `ResizeObserver` does not exist in jsdom and an instance was assigned instead of a class | use the mock from §1.1 |
| 8 | `Cannot find module 'min-indent'` after clean `npm ci` | regenerated lockfile held `apps/web/node_modules/strip-indent` without `resolved`/`integrity` and no `min-indent` entry; old `node_modules` had masked it | `rm -rf node_modules apps/*/node_modules package-lock.json && npm install`, then verify `npm ci` on a clean tree and commit the lock |
| 9 | `ERESOLVE … peer @mantine/core@"7.17.8" from @mantine/notifications@7.17.8` while found `@mantine/core@7.5.1` | `core` pinned to 7.5.1 while `notifications@^7.5.1` resolves to the latest 7.x whose peer requires exactly the same `core`; the previous lock had been produced with forced resolution and hid the mismatch | align all `@mantine/*` to one version, reinstall, verify with `npm ls`; do not use `--force` |
| 10 | `Cannot find module '@otc/contracts'` (TS2307) | packages were renamed, imports were not (`next.config.mjs`, `Rights/interfaces.ts`, `apps/api/src/domain/*`, README) | repo-wide replace of the old scope, then `typecheck` and tests |
| 11 | Middleware returned `undefined` when a redirect was needed; after a quick fix it used `rewrite` | no branch for redirect; `rewrite` swaps content but keeps the URL (status 200) | assert status 307 and `Location` in a test (red), implement `NextResponse.redirect(new URL(target, req.url))` |
| 12 | Allow-list bypass: `/sign`, `/s`, `/in`, `/recovery`, `/pass` treated as public | `publicPath.includes(requestSegment)` (substring) | compare first segments by equality; keep negative tests for prefix-like paths |
| 13 | Test table row `['/', true, '/']` | expectation contradicted the spec (session on a private path → `null`) | review each expectation against the spec before running; grouped table comments help |
| 14 | Debug `console.log` noise in test output | leftover logging in middleware/tests | remove; lint rule `no-console` recommended |
| 15 | `docker-compose` key `port:` | typo for `ports:` | fix; add `healthcheck`/`restart` |
| 16 | Counter shows 1 after three `setCount(count + 1)` | each render closes over its own `count`; the three updates queue the same value | `setCount(c => c + 1)` |
| 17 | `setInterval(() => setCount(count + 1))` with `[]` deps stuck at 1 | stale closure | `setCount(c => c + 1)` with empty deps (adding `count` to deps works but recreates the timer every tick) |
| 18 | `items.push(x); setItems(items)` does not re-render | same reference, React compares with `Object.is` | `setItems(prev => [...prev, x])` |
| 19 | Old user's data appears after fast id switching | out-of-order responses | abort in effect cleanup via `AbortController`, ignore `AbortError` |
| 20 | Hydration mismatch / `localStorage is not defined` in a custom hook | storage read during render (also in `useState` initializer) | initial state = default; read in `useEffect`; write only after read; `loadedKey` on key change |
| 21 | Hook overwrote stored value on mount; old key's value copied to the new key | write effect ran in the same commit as the read effect (and again with old value after key change; Strict Mode double-runs effects) | `ready` flag, then `loadedKey === key` guard |
| 22 | Static page shows the wrong brand | `process.env` is inlined at build time for statically rendered pages | per-request resolution (host/headers) or per-brand build |
| 23 | Typing lag in a 2000-row list | state in parent re-renders and re-filters all rows on each keystroke | profile first; colocate state/debounce; `memo` rows + `useMemo`; `useDeferredValue`; virtualization; server-side search |
| 24 | `GET https://github.com/...` blocked / `could not read Username` | tooling: automated page fetch is disallowed by robots; private repo cannot be cloned | make the repo public or provide the file tree (`git ls-files src`) |

Debug procedure for "tests will not start": (1) confirm the working directory is the package; (2) `npx jest --showConfig` and check `transform`, `setupFilesAfterEnv`, `moduleNameMapper`; (3) read the first line of the error (environment vs transform vs resolution); (4) check the test file extension and docblock; (5) for dependency errors compare a clean `npm ci` with the local `node_modules`.

---

### 4. [FUTURE BACKLOG]

**Auth logic (apps/web)**
1. Refactor `authRedirect.ts`: precompute public segments, make `isPublicPath` return a boolean via `some`/`includes`, document why the empty segment is private.
2. Move the `['/password-recovery/confirm', false, null]` case into the NO SESSION | PUBLIC PATH group; decide and pin by test what happens for a signed-in user on `/password-recovery/confirm` (currently redirects to `/`).
3. `middleware.ts`: confirm no `console.log` remains, extract `AUTH_COOKIE`, use `req.cookies.has`, add test "session on a private path passes through".
4. Move `@edge-runtime/jest-environment` from the root `package.json` to `apps/web` devDependencies.
5. Remove remaining `next-auth` traces (imports, types, env vars) and implement the `getSession()` data-access function.

**LoginForm (next step, RTL)**
6. Create the stub component with contract `onSubmit(values): Promise<void>`, `test/utils.tsx` with `renderWithProviders`, then tests: renders fields/button; empty submit shows "Email is required"/"Password is required" and does not call `onSubmit`; valid submit calls `onSubmit` once with `{ email, password }`; pending state disables the button; rejection renders `role="alert"` "Invalid credentials". Validation via `@mantine/form` + `mantine-form-zod-resolver`, schema in its own file with its own test. Later wire real requests with MSW.
7. Repeat the pattern for sign-up and password-recovery forms.

**Structure**
8. Decide: FSD or domain-based. Then write the placement rules into the repo README, add the path alias and ESLint rules (`no-restricted-imports`, boundaries/Steiger, import sorting) at `warn`, promote to `error` later.
9. If FSD with Next: verify the root `app/` + `pages/` stub layout and the location of `middleware.ts` with a working example before restructuring.
10. During the `otc-web` migration: rename `Sing…` → `Sign…` first, `child` → `components`, colocate tests and styles, then move pages by the boy-scout rule.

**Dependencies / CI**
11. UI smoke test after the Mantine bump 7.5.1 → 7.17.8 (forms, notifications, dropzone); decide pinned vs caret ranges for `@mantine/*` (pinned is safer; with ranges update all five together).
12. Confirm CI runs `npm ci`, then `turbo run typecheck test`; verify the lockfile installs on Linux (optional native bindings).
13. Finish the `@otc` → `@assets-exchange-monorepo` replacement in `apps/api` (sources, tests, README); run the api Vitest suite (not verified here because of a missing native binding in the sandbox).

**Backend (apps/api)**
14. Auth stage from the api README: `PasswordHasher`, `SessionStore`, `Clock`, `Mailer` as fakes/spies; `register`, `login` (same error for unknown email and wrong password), session expiry through the clock, `logout`, `GET /auth/me` 401 without cookie, `forgot` always 200, hashed one-time TTL token, `reset` invalidates sessions; HTTP tests through supertest; cookie contract in `packages/contracts`.

**Knowledge gaps still open (interview preparation)**
15. Not completed: behavioral round (STAR stories, spoken English), own implementation of `useLocalStorage`, task D (`useReducer` todo list), bug-hunt task B (`UserList`), writing a Dockerfile from scratch, TypeScript drill, frontend system-design walk-through.
16. Correct the skills claims to what is verifiable: App Router experience duration (stable only since May 2023), TDD as practiced commercially, Kubernetes depth, Datadog vs Sentry; add monorepo/Turborepo, metrics and testing tools to the CV; remove the photo for US applications.

---

## Appendix A — Canonical placement tree (target, FSD flavour for Next.js)
```
apps/web/
  app/                              # Next routing (package root), thin re-exports only
    (auth)/signin/page.tsx          # export { SignInPage as default } from '@/pages/sign-in'
  pages/README.md                   # stub so Next ignores src/pages as Pages Router
  src/
    app/                            # FSD app layer: providers, global styles, config
    pages/sign-in/{ui/SignInPage.tsx,index.ts}
    features/auth-by-credentials/
      ui/LoginForm/{LoginForm.tsx,LoginForm.module.css,LoginForm.test.tsx,index.ts}
      model/{loginSchema.ts,loginSchema.test.ts,useLogin.ts}
      api/login.ts
      index.ts
    entities/session/{lib/authRedirect.ts,lib/authRedirect.test.ts,index.ts}
    shared/{ui,lib,api,config}
  test/{jest.setup.ts,utils.tsx}
```
(Feature-based alternative: `features/auth/{screens,components,schemas,hooks,api,lib}` + `index.ts`, with `app/**/page.tsx` as thin routes.)

## Appendix B — ESLint skeleton for import discipline
```js
rules: {
  'no-restricted-imports': ['error', { patterns: [
    { group: ['@/features/*/*'], message: 'Import from the feature public API: @/features/<name>' },
    { group: ['../../*'], message: 'Use the @/ alias instead of long relative paths' },
  ]}],
  'import/no-default-export': 'error', // override for page/layout/route/config files
  'no-console': 'error',
}
```
Add `eslint-plugin-boundaries` (or Steiger for FSD), `eslint-plugin-simple-import-sort`, `eslint-plugin-unused-imports`.

## Appendix C — Theming by CSS variables (canonical sketch)
```tsx
// layout.tsx (server): brand resolved from host, validated by schema, defaults on failure
const brand = await getBrand(headers().get('host') ?? '');
const style = brandToCssVars(brand) as CSSProperties; // { '--brand-primary': '#…', … }
return <html lang={brand.locales[0]} style={style}>…</html>;
```
```css
:root { --brand-primary:#1a1a1a; --brand-on-primary:#fff; --color-action-bg:var(--brand-primary); --color-action-fg:var(--brand-on-primary); }
.button { background: var(--color-action-bg); color: var(--color-action-fg); }
```
Config schema must validate colors (`/^#[0-9a-fA-F]{6}$/`) and be fetched with `revalidate` + `revalidateTag`; fall back to a default brand.
