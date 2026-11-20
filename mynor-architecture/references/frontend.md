# Frontend reference (React 19 + Vite + TanStack)

This file has two kinds of content: **Observed conventions** (what the docs describe and the code actually does; follow these by default) and **Recommended additions** (gaps found in the code review).

---

## Observed conventions

### Stack

| Area | Libraries |
|---|---|
| Core | React 19, TypeScript ~5.7 `strict`, Vite 6 with `@vitejs/plugin-react-swc` |
| Routing and data | `@tanstack/react-router` (file routes via `@tanstack/router-plugin`), `@tanstack/react-query` 5, `@tanstack/react-table` |
| State and HTTP | Zustand 5 (with `persist`), Axios |
| Forms | React Hook Form, Zod, `@hookform/resolvers` |
| UI | Tailwind 3, shadcn/ui ("new-york", CSS variables, Radix), lucide and tabler icons, date-fns, immer |

### Bootstrapping

- `main.tsx` renders `<StrictMode><AppProviders/></StrictMode>`.
- `providers/app-providers.tsx` nests `QueryClientProvider` > `ThemeProvider` > `FontProvider` > `RouterProvider`.
- `config/router.ts` calls `createRouter({ routeTree, context: { queryClient }, defaultPreload: 'intent', defaultPreloadStaleTime: 0 })` and adds the `Register` module augmentation.
- `config/server.ts` maps every `import.meta.env.VITE_*` value into one config object and **throws at startup if any of them is missing**.

### Query client (`config/query-client.ts`)

```ts
new QueryClient({
  defaultOptions: {
    queries: {
      retry: (n, err) => /* none in DEV; up to 3 in PROD; never on 401/403 */,
      refetchOnWindowFocus: import.meta.env.PROD,
      staleTime: 5 * 60_000, gcTime: 10 * 60_000,
      placeholderData: (prev) => prev,
    },
    mutations: { onError: handleServerError },        // global toast
  },
  queryCache: new QueryCache({ onError: (e) => {
    // timeout/no response -> toast; 401 -> toast + resetUser() + navigate('/sign-in', {redirect});
    // 500 -> toast + navigate('/500')
  }}),
})
```

### API layer

**Clients.**
- `api/client/api-client.ts` is the main API: `axios.create({ baseURL, headers: { Accept }, withCredentials: true, timeout: 60000 })` followed by `applyInterceptors(client)`.
- External services each get their own `<name>-client.ts`.

**Interceptors** (`api/client/interceptors.ts`):
- The request interceptor attaches `Authorization: Bearer <token>`.
- The response interceptor handles a 401 by refreshing once. Requests that arrive during the refresh wait in a queue (`isRefreshing` + `failedRequestsQueue`) and replay with the new token. If the refresh fails, the interceptor saves `lastPath` and redirects to `/sign-in`.

**Typed wrapper** (`api/client/http-client.ts`): `createApiWrapper(client)` exports `api.get/post/put/patch/delete<T, B>(url, bodyOrParams?, config?)`.
- Every method returns `T`, meaning `response.data`.
- Each method also has `.withStatus(...)`, which returns `{ data, status }`.
- GET and DELETE send the second argument as `params`.
- POST, PUT and PATCH throw if the body is missing.
- There is one wrapper per client (`api`, `externalApi`, ...).

**Endpoints.** Every URL lives in `api/endpoints`:

```ts
// base-paths.ts
export const BASE_PATHS = { AUTH: '/auth', ORDERS: '/orders', ... }
// index.ts
export const ENDPOINTS = {
  AUTH: createEndpointGroup(BASE_PATHS.AUTH, { SIGN_IN: '/login/', REFRESH_TOKEN: '/login/refresh-token' }),
  PLANS: createEndpointGroup(BASE_PATHS.PLANS, { GET_ALL: '/', GET_BY_ID: param('/') }),
}
```

The builders come from `utils/helpers/endpoint-builder.ts`: `createEndpointGroup`, `param`, `paramWithSuffix` and `multiParam`.

**Service folder** (`api/services/<resource>/`):

```ts
// <resource>-service.ts: object literal of async methods
export const websiteService = {
  async getWebsiteByUUID(uuid?: string): Promise<WebsiteResponse[]> {
    if (!uuid) return []
    try { return await api.get<WebsiteResponse[]>(`${ENDPOINTS.WEBSITE.GET_BY_UUID}${uuid}/`) }
    catch (error) { console.error(getErrorMessage(error, 'default')); return [] }
  },
}
// <resource>-queries.ts
export const useWebsiteByUUID = (uuid?: string) =>
  useQuery<WebsiteResponse[]>({ queryKey: ['website', uuid], queryFn: () => websiteService.getWebsiteByUUID(uuid) })
// <resource>-mutations.ts
export const useCreateOrder = () =>
  useMutation<OrderResponse, unknown, CreateOrderRequest>({ mutationFn: websiteService.createOrder })
// index.ts
export * from './types'; export * from './<resource>-service'; export * from './<resource>-queries'; export * from './<resource>-mutations'
```

- `types.ts` uses the backend's snake_case field names as they arrive (`first_name`, `setup_fee`). The ESLint `naming-convention` rule explicitly allows snake_case properties.
- Hooks are named `use<Verb><Thing>` (`useGetBalance`, `useGetProfileDetails`, `useSignIn`).
- Optional queries are gated with `enabled: options?.enabled !== false && !!id`.

### Routing (`src/routes`, generated `routeTree.gen.ts`)

```
routes/
├── __root.tsx               # createRootRouteWithContext<{queryClient}>(); Outlet + Toaster + devtools (dev only)
│                            # notFoundComponent: NotFoundErrorPage, errorComponent: GeneralErrorPage
├── (auth)/route.tsx         # beforeLoad: checkAuthRedirect (logged-in users leave), AuthLayout
├── (auth)/sign-in.tsx       # component: SignInPage
├── (auth)/sign-up.lazy.tsx
├── (errors)/401|403|404|500|503.lazy.tsx
└── _authenticated/
    ├── route.tsx            # beforeLoad: checkUnauthRedirect(location); <AuthMiddleware><MainLayout><Outlet/>
    ├── index.tsx            # dashboard
    ├── billing/index.lazy.tsx
    └── settings/route.lazy.tsx + account.lazy.tsx ...
```

- A route file only binds a page component, e.g. `createLazyFileRoute('/_authenticated/billing/')({ component: BillingPage })`.
- `middleware/auth-middleware.tsx` is a wrapper component. It loads the profile, syncs it into the auth store, shows `<Loader/>` while loading, and toasts on errors.
- Route `loader`s are not used; data loads inside components through query hooks. `validateSearch` appears once.

### Pages (`pages/<feature>/<name>.page.tsx`)

A page composes module exports (`import { AccountBalanceCard, BillingHistory } from '@/modules/billing'`) and shared components. It calls query hooks and holds page-level UI state such as modal open flags. Error pages live in `pages/errors/*.page.tsx`.

### Modules (`modules/<feature>/`)

- `index.ts` re-exports the component group barrels (`export * from './components/cards/index'`).
- Components are grouped by kind: `cards/`, `tabs/`, `modal/`, `table/`, `sections/`, `shared/`, `feeback|feedback/`. Each async section has a sibling `*-skeleton.tsx`.
- Static config sits in a `data/` folder next to the component that uses it (`table/filters/data/filters-config.ts`, `sidebar/data/sidebar-data.ts`).
- Hooks live in `hooks/use-*.ts` at module or section level.
- Multi-step flows use `<flow>/steps/<step>/{components/, <step>.tsx}` with a barrel `components/index.ts`.
- Larger modules add `constants.ts`, `types.ts` and `utils/`.

### Forms

- **Schema** (`schemas/<feature>/<name>-schema.ts`): `export const signInSchema = z.object({...})` and `export type SignInSchema = z.infer<typeof signInSchema>`. Messages are user-facing strings.
- **Split** every form into `<name>-form.tsx`, which owns `useForm({ resolver: zodResolver(schema), defaultValues, mode })` and the submit logic, and `<name>-fields.tsx`, which renders the shadcn `<FormField>` pieces.
- **Submit** calls a mutation hook with `onSuccess` and `onError` callbacks, sets state in the store, shows a toast (`useToast`) and navigates.
- **Wizards** store their state in Zustand: `stores/ui/wizard-store.ts` holds step ids, visited steps and per-step controls (`onNext`, `onSkip`, disabled and pending flags), and `stores/data/<flow>-store.ts` holds the collected data.

### Stores (Zustand)

```ts
export const useAuthStore = create<AuthState>()(
  persist((set) => ({ user: null, setUser: (user) => set({ user }), resetUser: () => set({ user: null }) }),
          { name: 'auth-storage', storage: createJSONStorage(() => localStorage) }))
```

- Naming: `use<Name>Store`, actions `set<X>`, `reset<X>` and `clear<X>`.
- `stores/data` holds domain or session state; `stores/ui` holds modal and wizard state, usually persisted to `sessionStorage`.
- Code outside React reads stores with `useAuthStore.getState()` (interceptors, query client).

### Shared components

| Folder | Contents |
|---|---|
| `components/ui` | shadcn primitives; generated, so excluded from lint and knip |
| `components/shared` | Generic `data-table/` (TanStack Table with `ColumnProps<T>`, `DataTableProps<T>`, `ColumnMeta.className` augmentation, skeleton), `data-pagination/`, `data-filters/`, `data-list/`, `confirm-dialog`, `password-input`, `international-phone-input`, `date-picker/`, `command-menu`. Each has its own `types.ts` and `index.ts` |
| `components/layouts` | `auth/auth-layout.tsx` and `main/{header, sidebar, container, data}` |
| `components/feedback` | `loader`, `no-data-to-display` |

### Errors (front)

- `utils/formatters/format-error.ts`: `getErrorMessage(error, context)` maps status to a friendly message per context (`'auth' | 'domain-check' | 'default'`). It prefers the backend's `data.message` and handles timeouts.
- `utils/helpers/handle-server-error.ts` is the global mutation `onError` that shows the toast.
- Routes: `errorComponent` and `notFoundComponent` on the root route, plus the `(errors)` status pages.

### Hooks (global `src/hooks`)

`use-debounce`, `use-dialog-state`, `use-mobile`, `use-toast`, `use-auth-status`, `use-filters`, `use-wizard-navigate`, ...: one hook per file. (`use-dialog-state` is a default-exported `function`; most hooks are arrow `const`s.)

### Tooling

**Scripts:**

| Script | Runs |
|---|---|
| `dev` | `vite` |
| `build` | `tsc && vite build` |
| `lint` | `--max-warnings 0` (also split into `lint:large`, `lint:medium` and `lint:small` by folder for memory) |
| `type-check` | `tsc --noEmit` |
| `knip` | dead-code check |
| `gen:routes` | route tree generation |
| `format` | prettier |
| `pipeline` | lint, then tsc, then build |

**ESLint** (flat config) uses typescript-eslint recommended, react-hooks, react-refresh (`only-export-components`), `@tanstack/eslint-plugin-query`, tailwindcss (`no-custom-classname`: error), unicorn (`filename-case: kebabCase`) and cspell (comments only, `cspell.json`). Rules worth knowing:
- `no-explicit-any`, `consistent-type-assertions` and `no-unused-vars` (`^_` allowed) are errors.
- `React.FC` and `React.FunctionComponent` are banned.
- `naming-convention`: types, interfaces, enums and classes in PascalCase; exported functions and parameters in camelCase; properties in camelCase, snake_case, UPPER_CASE or PascalCase.
- In layouts and shared code, components must be arrow functions.

**Prettier:** no semicolons, single quotes (JSX too), 2 spaces, width 80, trailing commas `es5`, LF line endings. It uses `prettier-plugin-tailwindcss` plus `@trivago/prettier-plugin-sort-imports` with this group order: react, then libraries, then `@/assets`, `@/api`, `@/stores`, `@/lib`, `@/utils`, `@/constants`, `@/hooks`, `@/components/layouts`, `@/components/ui`, `@/components`, then relative imports.

**Other config:**
- `tsconfig.app.json`: `strict`, `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `moduleResolution: bundler`, `@/*` paths for every top-level folder.
- `vite.config.ts`: the `@` alias, a dev port from `VITE_PORT` with `strictPort`, `manualChunks` vendor split (react, tanstack, radix, forms, icons, date, maps, vendor), `sourcemap: false`, `minify: 'esbuild'` with `esbuild.drop: ['console','debugger']`, and `optimizeDeps.include` for core libraries.
- `knip.config.ts` ignores `src/components/ui/**` and `routeTree.gen.ts`.
- **Commits** go through commitizen `cz_conventional_commits` (semver, `v$version` tags, changelog on bump).
- **CI** (GitHub Actions, `node:20`) on the `testing` branch runs `tsc:build`, the folder-split lint jobs and the build.

---

## Where docs and code disagree

| Topic | Docs | Code |
|---|---|---|
| Feature folder | README: `features/`, `context/` | `modules/`, `providers/` (ESLint and Prettier still reference `features/`) |
| Constants | Architecture: `utils/constants` | top-level `src/constants/` |
| Query/mutation file names | `service-name-queries.ts` | mostly `<resource>-queries.ts`; one outlier `auth.mutations.ts` |
| Minifier | PERFORMANCE doc: terser `drop_console` | `minify: 'esbuild'` + `esbuild.drop` |
| React.memo | MODULE_REVIEW: 55 files | only about 3 files use `memo(` in the current code |
| Testing | Architecture: unit, integration and E2E strategy | zero test files, no test runner installed |

---

## Recommended additions (not in the source; use in new projects)

1. **Query key factory** per service (`websiteKeys.all`, `.detail(uuid)`), and invalidate in every mutation's `onSuccess`. Only 3 files call `invalidateQueries` today, and keys are inline arrays.
2. **Don't swallow errors in services.** Today a service catches the error and returns `[]` or a zero object, so React Query never sees `isError` and the UI shows empty data. Let errors throw, and map them to UI in the query/mutation layer or with `getErrorMessage`. Avoid `{ OK, messages }` result objects.
3. **Token storage.** Use a single source of truth: the access token in memory or the store, and the refresh token in an httpOnly cookie set by the backend. Today the token is read from localStorage, sessionStorage and the persisted store, which also spreads XSS exposure.
4. **No secrets in `VITE_*`.** Anything prefixed `VITE_` ships to the browser. Third-party API keys must go through the backend.
5. **Route loaders.** Use `loader: ({ context }) => context.queryClient.ensureQueryData(...)` for critical page data, plus `pendingComponent` / `errorComponent` per route group. Add `validateSearch` with Zod for search params.
6. **Store selectors.** Select narrowly (`useAuthStore((s) => s.user)`) rather than `(state) => state` to avoid re-renders.
7. **Tests.** Vitest + Testing Library + MSW for services and hooks, and Playwright for the main flows (auth, wizard). Add `test` to the `pipeline` script and CI.
8. **Module public API.** Import other modules only through `@/modules/<x>` (`index.ts`). Enforce it with `no-restricted-imports` or `eslint-plugin-boundaries`. Cap nesting at 5 levels.
9. **Typed API envelope.** Mirror the backend envelope (`ApiSuccess<T>`, `ApiError`) in `api/client/types.ts`, and parse critical responses with Zod.
10. **Naming.** Fix `feeback/` → `feedback/`, choose one of `-queries.ts` / `.queries.ts`, and update the ESLint and Prettier references from `features/` to `modules/`.
11. **Performance defaults.** Lazy routes (`.lazy.tsx`) for everything past first paint. Use `React.memo`, `useMemo` and `useCallback` for list rows and form sections, `@tanstack/react-virtual` for lists over 100 items, debounced search inputs and lazy images.
