---
name: mynor-architecture
description: Use when starting or structuring a new general project, or adding a feature to a FastAPI backend or React (Vite + TanStack) frontend; follow Mynor's layered architecture and folder layout.
---

# Mynor's default architecture (FastAPI backend + React frontend)

This is how Mynor likes projects built. It comes from two of his production projects: a FastAPI API and a React/Vite admin panel.

**Precedence.** If you are working in an established repo that already has its own conventions, those win; match them. This skill is the default for new projects, general projects, and repos that have no clear structure yet. When the two conflict in a new project, follow this skill.

Detail lives in:
- [references/backend.md](references/backend.md): full layer code templates, session patterns, SDK clients, workers, error logging, config, tests.
- [references/frontend.md](references/frontend.md): API service template, module layout, routing, stores, forms, lint rules, performance settings.

---

## 1. Backend (Python 3.13, FastAPI, SQLModel, Alembic, Pydantic v2)

### Folder tree

```
project_api/
├── main.py                    # only: from utils.app_factory import create_app; app = create_app()
├── config/__init__.py         # env-backed config classes + singleton instances
├── routes/                    # HTTP layer (thin)
│   ├── __init__.py            # main_router aggregates every domain router
│   ├── <domain>/__init__.py   # <domain>_router = APIRouter(prefix="/<domain>", tags=[...])
│   └── auth/                  # big domains nest sub-routers: auth/login/, auth/register/ ...
├── controllers/               # orchestration + exception -> HTTP status mapping
│   └── <domain>/<verb>_<thing>_controller.py   # one async controller function per file
├── services/                  # business ops + DB access
│   └── <domain>/__init__.py   # class <Domain>Services + module singleton at the bottom
├── database/
│   ├── __init__.py            # engine, get_session_sync, close/commit/rollback helpers
│   └── models/<entity>.py     # SQLModel tables, one entity per file
├── alembic/ + alembic/versions/   # migrations
├── common/
│   ├── response_models/<domain>/__init__.py   # Pydantic request payloads + response models
│   ├── response_models/error_response.py      # generate_error_response_for_statuses()
│   ├── exceptions/<domain>/__init__.py        # domain exceptions
│   ├── others/<domain>/                       # internal (service-to-service) payload models
│   └── email_templates/<template>.py          # functions that return HTML strings
├── SDK/<external_service>/    # one client package per external API
│   ├── __init__.py            # <Service>Client + singleton export
│   └── <operation_group>.py   # sub-modules that receive the shared APIClient
├── middlewares/               # error_log.py (4xx/5xx capture), request_logging.py
├── decorators/                # e.g. admin/ (role gating), service_error_logging.py
├── workers/<name>_worker.py   # background recurring jobs (class + async methods)
├── utils/                     # app_factory, environment, session_manager, fetch_client,
│                              # worker_scheduler, worker_initializer, auth, password,
│                              # pagination, websocket_logger, ...
├── scripts/                   # one-off maintenance/seed scripts (excluded from mypy)
├── tests/                     # conftest.py + test_<domain>.py
├── docs/                      # markdown docs per subsystem (api/, workers/, SDK/, ...)
├── mypy.ini  pytest.ini  requirements.txt  docker-compose.yml  .env (never committed)
```

### Layer rules

| Layer | Owns | Must not |
|---|---|---|
| Route | path, method, `response_model`, `name`/`summary`/`description`, documented error statuses, Pydantic payload validation. Body is one line: `return await <x>_controller(request, payload)` | contain logic, touch DB, catch exceptions |
| Controller | opening/closing the session for the request, calling one or more services/SDK clients, building the response model, mapping each domain exception to a status code with `JSONResponse` | write raw SQL/queries beyond trivial lookups, contain reusable business rules |
| Service | business operations, `select(...)` queries, validation of business rules, raising **domain exceptions**; returns domain objects (SQLModel rows, dataclasses, tuples) | return HTTP responses, know about `Request` |
| SDK client | HTTP calls to one external API through the shared `utils/fetch_client.APIClient`, parsing into Pydantic models | contain business rules |
| Worker | recurring async jobs that reuse services + SDK clients | be called from routes |

### Request flow

```
HTTP Request -> Route -> Controller -> Service -> Database / SDK
Response    <- Route <- Controller <- Service <-
```

Errors bubble up as domain exceptions from services; controllers translate them to HTTP. Middleware logs every 4xx/5xx.

### Adding a backend feature (step by step)

1. **Model**: `database/models/<entity>.py` (SQLModel `table=True`). Include `id`, a public `uuid`, timestamps and a soft-delete column (`deleted_at`) when the entity is user-facing.
2. **Migration**: `alembic revision --autogenerate -m "<descriptive message>"`, review it, `alembic upgrade head`.
3. **Schemas**: `common/response_models/<domain>/__init__.py` with `<Action><Entity>Payload` / `<Entity>Request` and `<Entity>Response` models.
4. **Exceptions**: `common/exceptions/<domain>/__init__.py` (e.g. `NoOrderFound`, `OrderAlreadyPaidError`), each carrying a `message`.
5. **Service**: `services/<domain>/__init__.py`: class `<Domain>Services` with typed methods that take `session: Session`, raise domain exceptions, and filter `deleted_at == None`. End with `<domain>_services = <Domain>Services()`.
6. **Controller**: `controllers/<domain>/<verb>_<thing>_controller.py` with `async def <verb>_<thing>_controller(request: Request, ...) -> JSONResponse` using the try/except/finally template (references/backend.md).
7. **Route**: add to `routes/<domain>/__init__.py`; declare `response_model` and `responses=generate_error_response_for_statuses([...])`.
8. **Register**: `main_router.include_router(router=<domain>_router)` in `routes/__init__.py` (new domains only).
9. **Tests**: `tests/test_<domain>.py` covering route (TestClient), controller and service.
10. **Docs**: if it's a subsystem, add `docs/<subsystem>.md` (overview, components, endpoints, testing).

External API? Add `SDK/<service>/`, config in `config/__init__.py`, export a singleton, add `test_service()`.
Recurring job? Add `workers/<name>.py`, register it in `utils/worker_initializer.py` with an interval.

### Backend conventions (observed)

- **Naming**: modules and functions `snake_case`; classes `PascalCase`; service classes plural `<Domain>Services`; singletons `<domain>_services` / `<service>_client`; routers `<domain>_router`; controllers end in `_controller`.
- **Typing**: every function fully annotated (mypy `disallow_untyped_defs`, `disallow_incomplete_defs`, `warn_return_any`, pydantic + sqlalchemy plugins). Use `X | None`, `list[...]`, `tuple[...]`.
- **Sessions**: the controller owns the request session (`get_session_sync()` ... `commit/rollback` ... `close_session()` in `finally`) and passes `session=` to every service call so one request = one transaction. Services may open `with Session(engine)` for self-contained ops.
- **Public IDs**: expose `uuid` in URLs (`/cost-detail/{project_uuid}`), keep integer `id` internal.
- **Soft delete**: never hard-delete user data; set `deleted_at`, filter it in every query.
- **Auth**: JWT Bearer tokens; auth layer sets `request.state.user_id` that controllers read. Public routes are listed explicitly (route exclusions); role checks via decorators (`decorators/admin`).
- **Docstrings** on service/controller functions describing Args/Returns/Raises.
- **App factory**: `utils/app_factory.create_app()` wires middlewares, routers and lifespan (workers start in lifespan; skipped when `utils/environment.is_testing_environment()`).

### Verified in code (beyond the docs)

- **Middleware stack**, registered in `_add_middlewares` in this order: CORS, Auth, JSONValidation, RequestLogging, ErrorLog. Starlette runs the last-added first, so ErrorLog is outermost.
- **Global exception handlers**: `HTTPException` and catch-all `Exception` both return `{"detail": ...}`.
- **Auth**: `AuthMiddleware` checks the Bearer JWT, rejects unverified emails and sets `request.state.user_id`, `user_uuid` and `is_admin`. Public routes come from the allow-list in `utils/route_exclusions.py` (prefixes, exact paths, method+path). `@require_admin_role()` wraps admin controllers. `Depends(get_current_user_id)` covers optional auth.
- **No DI container, base service or repository layer.** Services are module singletons composed in `__init__` (`self.plans_services = plans_services`), and `Depends` is used only for request params (`PaginationParams`, optional user).
- **Exceptions**: flat classes that each extend `Exception` with `.message`, with no base class or status code.
- **Pagination**: `PaginationParams` (`page>=1`, `limit 1..50`, `.offset`) through `Depends()`. The response is `{total, page, limit, pages, <items>}`.
- **Models**: plural class names, `__table_args__ = {'extend_existing': True}`, `uuid` default factory, `str` enums, and every model imported in `database/models/__init__.py` so Alembic sees it.
- **Engine**: QueuePool (`pool_pre_ping`, `pool_recycle`). `SessionLocal` is a sessionmaker, plus safe commit/rollback/close helpers.
- **Tests**: SQLite test engine, `db_session`, `api_client` (TestClient) and `seed_*` fixtures, `auth_headers` via a real login, `class Test<Domain>Services`, and `unittest.mock.patch` for SDK calls.
- **CI**: mypy, then pytest, then deploy over SSH with `docker compose up -d --build` on push to `develop`.

---

## 2. Frontend (React 19, TypeScript strict, Vite, TanStack Router + Query, Zustand, Axios, Tailwind + shadcn/ui, RHF + Zod)

### Folder tree

```
src/
├── api/
│   ├── client/          # http-client.ts (base axios), api-client.ts (main API),
│   │                    # <external>-client.ts, interceptors.ts (auth header, 401 handling)
│   ├── endpoints/       # base-paths.ts + index.ts: every URL defined once
│   └── services/<resource>/
│       ├── index.ts                    # public exports
│       ├── types.ts                    # request/response interfaces
│       ├── <resource>-service.ts       # plain async functions calling the client
│       ├── <resource>-queries.ts       # useQuery hooks + query keys
│       └── <resource>-mutations.ts     # useMutation hooks + invalidation
├── assets/              # css/, icons/, images/
├── components/
│   ├── ui/              # shadcn primitives (generated; excluded from lint/knip)
│   ├── shared/          # cross-module business components: data-table/, data-pagination/,
│   │                    # data-filters/, confirm-dialog, password-input, date-picker/ ...
│   ├── layouts/         # auth/auth-layout.tsx, main/{header,sidebar,container,data}/
│   └── feedback/        # loader, no-data-to-display
├── config/              # query-client.ts, router.ts, server.ts, fonts.ts
├── constants/           # app-wide constants (pagination.ts, currency.ts, url.ts ...)
├── hooks/               # global hooks: use-debounce.ts, use-mobile.tsx, use-auth-status.ts ...
├── lib/utils.ts         # cn() and tiny helpers (shadcn alias)
├── middleware/          # auth-middleware.tsx (route guard logic)
├── modules/<feature>/   # feature code (see module layout)
├── pages/<feature>/<name>.page.tsx   # page composition: layout + module components
├── providers/           # app-providers.tsx, theme-provider.tsx, ...
├── routes/              # TanStack file routes (thin): (auth)/, (errors)/, _authenticated/, __root.tsx
├── schemas/<feature>/<name>-schema.ts   # Zod schemas (+ common/)
├── stores/
│   ├── data/            # <name>-store.ts: domain/session state (auth, order wizard data)
│   └── ui/              # <name>-store.ts: UI state (modals, wizard step)
├── utils/
│   ├── formatters/      # format-currency.ts, format-date.ts, format-error.ts ...
│   └── helpers/         # handle-server-error.ts, endpoint-builder.ts, logger.ts, forms.ts ...
├── routeTree.gen.ts     # generated; never edit
└── main.tsx
```

### Module layout

```
modules/<feature>/
├── components/          # feature components, grouped by section (cards/, tabs/, modal/, table/)
│   └── <section>/
│       ├── index.tsx    # main component
│       ├── <section>-skeleton.tsx
│       └── data/        # static config for that section (columns, filters-config.ts)
├── hooks/               # use-<feature>-<thing>.ts
├── data/                # static data and configuration
├── utils/               # pure functions
├── constants.ts
├── types.ts
└── index.ts             # public API: other code imports only from here
```

Multi-step flows use `steps/<step>/{components/,<step>.tsx}` (e.g. `auth/sign-up/steps/sign-up`, `steps/validate-otp`). Keep nesting at **5 levels max** and split a module before it becomes a large share of the codebase.

### Layer rules

- **Route file** (`routes/...`): `createFileRoute` / `createLazyFileRoute`, guards (`beforeLoad`), search-param validation, renders a page. No UI logic. Use `.lazy.tsx` for everything not needed on first paint.
- **Page** (`pages/...page.tsx`): composes layout + module components. No data fetching logic of its own beyond calling module hooks.
- **Module components**: UI for one feature; get data from `api/services/*-queries` hooks or module hooks; never call axios directly.
- **API service**: the only place that knows URLs and HTTP; queries/mutations wrap it with TanStack Query.
- **Stores** (Zustand): only state that must survive across routes/components (auth, multi-step wizard data, global modals). Server data lives in React Query, not in stores. Local UI state stays in `useState`.
- **Schemas** (Zod): every form and boundary input; infer TS types with `z.infer`.

### Data flow

User interaction -> component handler / hook -> mutation or store update -> service -> API client (interceptors add auth, normalize errors) -> response typed by `types.ts` -> query cache invalidated -> components re-render.

### Adding a frontend feature (step by step)

1. **Endpoints**: add paths to `api/endpoints`.
2. **Service folder**: `api/services/<resource>/` with `types.ts`, `<resource>-service.ts`, `<resource>-queries.ts`, `<resource>-mutations.ts`, `index.ts`.
3. **Schema**: `schemas/<feature>/<name>-schema.ts` for any form.
4. **Module**: `modules/<feature>/` using the layout above; skeleton component for every async section; empty state via `components/feedback/no-data-to-display`.
5. **Page**: `pages/<feature>/<feature>.page.tsx`.
6. **Route**: `routes/_authenticated/<feature>/index.lazy.tsx` (or `(auth)/` for public); run `npm run gen:routes` if the plugin didn't.
7. **Navigation**: add to `components/layouts/main/sidebar/data/sidebar-data.ts`.
8. **Store** only if state crosses routes: `stores/data/` or `stores/ui/`.
9. Run `npm run lint`, `npm run type-check`, `npm run knip`, `npm run build`.

### Frontend conventions (observed)

- **File names**: kebab-case everywhere (`unicorn/filename-case`); hooks `use-*.ts(x)`; pages `*.page.tsx`; schemas `*-schema.ts`; stores `*-store.ts`.
- **Components**: arrow functions with typed props, `({ a, b }: Props) => JSX.Element`; **no `React.FC`**, no `function` declarations for components in shared/layout code. `React.memo` for heavy list items and form sections; `useMemo`/`useCallback` for expensive values and handlers passed to children.
- **Types**: TS `strict`, no `any` (lint error), no unused vars (prefix `_` to ignore), `consistent-type-assertions`; types/interfaces/enums PascalCase, exported functions and params camelCase.
- **Imports**: absolute `@/…` aliases (`@/api`, `@/modules`, `@/components` ...); import a module through its `index.ts`.
- **Styling**: Tailwind utility classes only (`tailwindcss/no-custom-classname`); `cn()` from `@/lib/utils`; shadcn "new-york" style with CSS variables; icons from lucide / tabler.
- **Forms**: React Hook Form + `zodResolver`; split big forms into memoized section components (50-100 lines each), never a 400-line form.
- **Shared first**: tables, pagination, filters, confirm/action modals and skeletons go in `components/shared` once they repeat twice.
- **Immutable updates** for nested state (helper functions or immer), never mutate.

### Verified in code (beyond the docs)

- **Typed HTTP wrapper**: `api.get/post/put/patch/delete<T, B>(url, bodyOrParams?, config?)` returns `T`; `.withStatus()` returns `{data, status}`. There is one wrapper per axios client.
- **Interceptors**: attach the Bearer token, refresh once on 401 with a queue for concurrent requests, and redirect to `/sign-in` (saving `lastPath`) if the refresh fails.
- **Endpoints**: `ENDPOINTS.<GROUP>.<NAME>` is built with `createEndpointGroup(BASE_PATHS.X, {...})`, and `param('/x')` creates `(id) => '/x/id'`.
- **Service shape**: an object literal (`export const websiteService = { async getX() {...} }`). Query hooks are `use<Verb><Thing>` with inline keys, and `enabled` gates optional ids.
- **Query client**: `staleTime` 5 min, `gcTime` 10 min, `placeholderData` keeps previous data, no retry on 401/403. The global `QueryCache.onError` toasts and redirects (401 to `/sign-in?redirect=`, 500 to `/500`), and mutation `onError` calls `handleServerError`.
- **Router context** carries `{ queryClient }`. The root route sets `errorComponent` and `notFoundComponent`. The `_authenticated/route.tsx` `beforeLoad` guard is wrapped in `<AuthMiddleware>` (profile sync + loader) and `<MainLayout>`.
- **Forms** are split into `<name>-form.tsx` (useForm + zodResolver + mutation) and `<name>-fields.tsx` (FormFields). Schemas export `xSchema` plus `type XSchema = z.infer<...>`.
- **Stores**: `use<Name>Store` with `set/reset/clear` actions and `persist`; data stores use localStorage, UI stores use sessionStorage. Non-React code reads them with `getState()`.
- **`config/server.ts`** throws at startup if any `VITE_*` var is missing.
- **Prettier**: no semicolons, single quotes, width 80, sorted import groups, tailwind class sorting.

---

## 3. Cross-cutting

- **Config/env**: backend reads env vars only in `config/__init__.py` (classes + singletons, e.g. `email_config`); frontend uses `VITE_*` vars read in `src/config`. `.env` is gitignored; ship a `.env.example` with keys only. **Never** put real secrets as `os.getenv` defaults or in docs.
- **Errors**: backend = domain exceptions -> controller status mapping -> `{"message": ...}` body; routes document error statuses. Frontend = interceptors + `utils/helpers/handle-server-error.ts` + `format-error.ts`, toasts for mutation failures, error pages under `routes/(errors)` (401/403/404/500/503).
- **Logging/observability**: middleware captures every 4xx/5xx into an `app_errors` table (fingerprint, request id, method, path, status, stack; no bodies/tokens/PII, truncated) with feature flags (`ENABLE_ERROR_LOGS`, `LOG_SAMPLE_RATE`, `SERVICE_NAME`); workers isolate errors per item; SDK/worker errors should be logged then re-raised, not swallowed (the docs' `@log_service_errors` decorator is not present in code; re-create it in new projects). Request ID returned in response headers. Frontend strips `console`/`debugger` in prod builds.
- **Testing**: backend pytest (`tests/test_<domain>.py`, `asyncio_mode = auto`), test each layer independently. Frontend: Architecture calls for component, service (mocked API) and E2E tests; no framework is set yet, so pick Vitest + Testing Library + Playwright for new projects.
- **Tooling**: backend mypy strict-ish (see `references/backend.md`), Docker Compose with `env_file: .env` and `fastapi run main.py`. Frontend ESLint flat config (typescript-eslint, react-hooks, react-refresh, tanstack-query, tailwindcss, unicorn, cspell), Prettier with import sorting + tailwind plugin, knip for dead code, `pipeline` script = lint + `tsc --noEmit` + build.
- **Commits**: Conventional Commits via commitizen (`cz_conventional_commits`, semver, `tag_format: v$version`, changelog on bump).
- **Docs**: each subsystem gets a markdown doc in `docs/` (overview, components, directory structure, patterns, how to add one, best practices, testing).

---

## 4. Recommended additions (gaps found in the code, not existing conventions)

Apply these by default in **new** projects. In an existing repo, propose them first. Full rationale is in the references.

**Backend**
1. One response envelope (`{data, message?}` / `{error: {code, message}}`) and no `str(e)` in 500s. Today `message`, `error` and `detail` are mixed.
2. An `AppError(Exception)` base with `status_code`/`code`, domain subclasses, and one exception handler. This replaces the repeated `except` ladders in controllers.
3. Add CORS last so it is outermost, which removes the hand-written CORS headers.
4. A session per request through `Depends(get_session)` so tests can override it. Services that use the global `engine` currently bypass the test DB.
5. A `TimestampedModel` mixin (`id`, `uuid`, tz-aware `datetime` `created_at`/`updated_at`/`deleted_at`). Timestamps are strings today.
6. `pydantic-settings` config that fails fast on missing secrets, instead of defaulting to `" "`.
7. `httpx.AsyncClient` in the SDK `APIClient`, since `requests` blocks the event loop.
8. `logging` with JSON output and a request id; no `print` (about 200 calls today).
9. A single `Page[T]` pagination shape (two exist today), paginated in SQL.
10. ruff + pre-commit, a `.env.example`, descriptive Alembic messages, and CI secrets kept only in the secret store.

**Frontend**
1. A query key factory per service, and invalidation on every mutation success.
2. Services throw instead of returning `[]`, zero objects or `{OK, messages}`, so `isError` works.
3. One token source of truth (memory/store, refresh token in an httpOnly cookie) instead of localStorage, sessionStorage and the store.
4. No third-party secrets in `VITE_*`, because everything with that prefix ships to the browser.
5. Route `loader` + `ensureQueryData`, `validateSearch` with Zod, and `pendingComponent`/`errorComponent` per route group.
6. Narrow Zustand selectors; Vitest + Testing Library + MSW + Playwright in CI; module boundary lint (`no-restricted-imports`).

---

## 5. Checklist before you finish

- [ ] Backend: route is one line; controller maps every domain exception; service raises, never returns HTTP.
- [ ] Session opened once per request in the controller, closed in `finally`, rolled back on error.
- [ ] New table has an Alembic migration with a descriptive message; soft delete + uuid where user-facing.
- [ ] Pydantic payload + response model in `common/response_models/<domain>`; errors documented on the route.
- [ ] mypy and pytest pass.
- [ ] Frontend: HTTP only in `api/services`; server state in React Query; Zustand only for cross-route state.
- [ ] Files kebab-case, no `any`, no `React.FC`, Tailwind classes only, imports via `@/` and module `index.ts`.
- [ ] New route is lazy unless it's first-paint; skeleton + empty state for every async section.
- [ ] `lint`, `type-check`, `knip`, `build` pass; commit message follows Conventional Commits.
- [ ] No secrets, tokens or real env values in code, defaults, or docs.
