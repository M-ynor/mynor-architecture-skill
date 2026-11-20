# Backend reference (FastAPI + SQLModel)

Two kinds of content, kept apart:
- **Observed conventions**: what the source project's docs say and the code actually does. Follow these by default.
- **Recommended additions**: gaps found in the code review. Apply them in new projects; in an existing repo, propose them first.

---

## Observed conventions

### Stack

Python 3.13, FastAPI, Starlette `BaseHTTPMiddleware`, SQLModel on SQLAlchemy 2, Alembic, Pydantic v2, PyJWT/python-jose, passlib/bcrypt, `requests`, `python-dotenv`, pytest, mypy, and uvicorn (`fastapi run main.py`).

### Bootstrapping

```python
# main.py
from utils.app_factory import create_app
app = create_app()
```

`utils/app_factory.create_app()` does the following, in order:
1. Builds `FastAPI(title=..., version=..., lifespan=lifespan)`.
2. Mounts static directories.
3. Calls `_add_middlewares(app)`, `_add_routes(app)`, `_add_health_check(app)` (`/health` and `/`) and `_add_exception_handlers(app)`.
4. Sets a custom OpenAPI schema (`utils/custom_docs.py`).

`lifespan` starts the in-memory/WebSocket logger, `worker_scheduler.start()` and `initialize_workers()`, then stops the scheduler on shutdown. It skips all of that under pytest.

**Middleware order.** Starlette runs the last-added middleware first. The calls are:

```python
app.add_middleware(CORSMiddleware, allow_origins=general_config.ALLOWED_ORIGINS, allow_credentials=True, ...)
app.add_middleware(AuthMiddleware)            # JWT -> request.state
app.add_middleware(JSONValidationMiddleware)  # 400 on bad JSON / wrong content-type
app.add_middleware(RequestLoggingMiddleware)  # request log table (sanitized headers, truncated body)
app.add_middleware(ErrorLogMiddleware)        # outermost: 4xx/5xx -> app_errors, x-request-id
```

The request therefore passes ErrorLog, then RequestLogging, then JSONValidation, then Auth, then CORS, then the route. CORS sits innermost, so responses produced earlier carry no CORS headers. The project works around this by adding CORS headers by hand in every middleware error response and in the global exception handlers.

**Global exception handlers.** `HTTPException` returns `{"detail": ...}`, and any unhandled `Exception` returns 500 `{"detail": "Internal Server Error"}`. Both handlers add CORS headers.

### Config

`config/__init__.py` calls `load_dotenv(override=True)` and defines one plain class per concern, each with a singleton:

```python
class Config:            # DB_URI, SECRET_KEY, ALGORITHM, token expirations, base URLs, ALLOWED_ORIGINS ...
    DB_URI = os.getenv("DATABASE_URL", "<local default>")
general_config = Config()

class EmailConfig: ...
email_config = EmailConfig()
# one class + singleton per external integration (domain registration, deploy service, ...)
```

Secrets default to a blank string (`" "`). Only non-secret values such as local URLs and ports get real defaults. `utils/environment.py` exposes `is_testing_environment()`, `is_development_environment()` and `is_production_environment()`, all driven by `ENVIRONMENT` and `TESTING`.

### Database

`database/__init__.py` contains:
- `engine = create_engine(DB_URI, poolclass=QueuePool, pool_size=10, max_overflow=20, pool_pre_ping=True, pool_recycle=3600, pool_timeout=30)`
- `SessionLocal = sessionmaker(autocommit=False, autoflush=True, bind=engine, class_=Session)`
- `get_session()`, a generator for `Depends`
- `get_session_sync()`, `close_session(s)`, `rollback_session(s)` and `commit_session(s)`. Each checks `session.is_active`, logs, and re-raises on a failed commit.

`utils/session_manager.py` adds `get_db_session()` (a context manager), the `@safe_session_operation` decorator, the `SessionManager` and `SessionContextManager` classes, and `execute_with_session[_async]`.

In practice:
- Services take `session: Session` as a parameter (about 120 occurrences).
- Self-contained service methods open `with Session(engine)` (about 30 occurrences).
- Controllers call `get_session_sync()` and close in `finally`.

### Models

`database/models/<entity>.py` holds one SQLModel class per file. Classes use plural PascalCase, such as `Users`, `Projects` and `ShortUrls`.

```python
class ShortUrls(SQLModel, table=True):
    __table_args__ = {'extend_existing': True}
    id: int | None = Field(default=None, primary_key=True)
    uuid: str = Field(default_factory=lambda: str(uuid4()))
    user_id: int | None = Field(default=None, foreign_key="users.id")
    is_active: bool = Field(default=True)
    created_at: str = Field(default_factory=lambda: datetime.now().strftime("%Y-%m-%d %H:%M:%S"))
    updated_at: str | None = None
    deleted_at: str | None = None          # soft delete
    user: Optional["Users"] = Relationship(back_populates="short_urls")
```

- Enums are `class PaymentStatus(str, Enum)` and live in the model file.
- `database/models/__init__.py` imports every model so that `SQLModel.metadata` is complete for Alembic and tests.
- A `uuid` column is the public identifier on most tables (17 of 22). A soft-delete column (`deleted_at`, or `soft_delete` in older tables) is on 15 of 22.

### Alembic

`alembic/env.py` reads the URL from `general_config.DB_URI`, imports every module under `database.models` dynamically (`pkgutil`), and targets `SQLModel.metadata`. Migrations are autogenerated. Merge-head revisions exist, which means parallel branches create migrations.

### Routes

```python
orders_router = APIRouter(prefix="/orders", tags=["Orders"])

@orders_router.get("/cost-detail/{project_uuid}",
    name="Get order cost detail",
    response_model=OrderCostDetailResponse,
    description="...",
    responses=generate_error_response_for_statuses([400, 404, 500]))
async def get_order_cost_detail(request: Request, project_uuid: str) -> JSONResponse:
    return await get_order_cost_detail_controller(request, project_uuid)
```

- Every route function takes `request: Request` first.
- Query objects are declared at the route and passed straight through, for example `pagination: PaginationParams = Depends()` and `search: str = Query(None)`.
- `routes/__init__.py` builds `main_router` with one `include_router(router=...)` per domain. Admin routers use the `/admin/<area>` prefix.
- `generate_error_response_for_statuses([..])` maps each status to `{"message": "<reason phrase>"}` and only feeds the OpenAPI docs.

### Controllers

The default template, used in most controllers:

```python
async def get_order_cost_detail_controller(request: Request, project_uuid: str) -> JSONResponse:
    """Docstring: what it returns."""
    session = get_session_sync()
    try:
        detail = order_cost_service.get_order_cost_detail(project_uuid=project_uuid, session=session)
        response = OrderCostDetailResponse(**...)
        return JSONResponse(status_code=200, content=response.model_dump())
    except (NoProjectFound, NoWebsitesFound) as e:
        return JSONResponse(status_code=404, content={"message": str(e)})
    except Exception as e:
        rollback_session(session)
        return JSONResponse(status_code=500, content={"message": f"Internal Server Error: {e}"})
    finally:
        close_session(session)
```

Variants that also appear:
- Some controllers return a Pydantic response model directly and raise `HTTPException` (paginated admin lists).
- Some take `Depends(...)` parameters such as `PaginationParams` or `get_current_user_id`.
- Write flows such as order creation call several services plus an SDK client on one session, roll back on every error branch, and commit once.

Status mapping in use:
- 400: validation or business-rule errors (`ValueError` included)
- 401: authentication
- 403: not allowed, or email not verified
- 404: `No<Entity>Found`
- 500: anything else
- 201: creates

### Services

```python
class OrderCostService:
    """Docstring."""
    def __init__(self) -> None:
        self.plans_services = plans_services          # composes other service singletons
    def get_order_cost_detail(self, project_uuid: str, session: Session) -> OrderCostDetail:
        """Args / Returns / Raises."""
        project = session.exec(select(Projects).where(Projects.uuid == project_uuid,
                                                      Projects.soft_delete == None)).first()
        if not project:
            raise NoProjectFound(f"Project with UUID {project_uuid} not found")
        ...

order_cost_service = OrderCostService()   # singleton at bottom of services/<domain>/__init__.py
```

- There is no base service class and no repository layer. Services query SQLModel directly with `select()` and `session.exec()`.
- There is no DI container. "Dependency injection" means module-level singletons composed in `__init__` (28 of 33 service packages export one), plus FastAPI `Depends` for request parameters.
- Services return models, tuples or dicts and raise domain exceptions.

### Exceptions

`common/exceptions/<domain>/__init__.py` contains flat classes, each extending `Exception` directly:

```python
class NoProjectFound(Exception):
    def __init__(self, message: str):
        super().__init__(message)
        self.message = message
```

Naming follows `No<Entity>Found`, `<Entity>NotFoundError`, `<Thing>AlreadyExists`, `Invalid<Thing>` and `<Thing>ExpiredError`. There is no shared base class and no status code on the exception; the controller decides the status.

### Response models

`common/response_models/<domain>/__init__.py` (sometimes `<domain>.py`) holds Pydantic `BaseModel`s. Names follow `<X>Payload`, `<X>Request`, `<X>Response`, `<X>ResponseList` and `<X>PaginatedResponse`. Models for external services can be grouped under a namespace folder (for example `response_models/<EXTERNAL>/users`). Internal service-to-service payloads go in `common/others/<domain>`.

### Pagination

`utils/pagination.py`:

```python
class PaginationParams:
    def __init__(self, page: int = Query(1, ge=1), limit: int = Query(10, ge=1, le=50)): ...
    @property
    def offset(self) -> int: return (self.page - 1) * self.limit
def paginate_query(query, pagination): return query.offset(pagination.offset).limit(pagination.limit)
```

The response shape is `{total, page, limit, pages, <items>}`. A second helper, `utils/pagination_helper.py`, uses `{total, page, per_page, size, total_pages}`. See the recommendations.

### Auth and permissions

`AuthMiddleware`, for every non-excluded route:
1. Requires `Authorization: Bearer <jwt>`.
2. Resolves the user with `TokenServices.get_user_by_token`.
3. Rejects unverified emails.
4. Sets `request.state.user_id`, `request.state.user_uuid` and `request.state.is_admin`.

On failure it returns 401 `{"error": ...}`.

Public routes are listed in `utils/route_exclusions.py`: `EXCLUDED_PREFIXES`, `EXACT_EXCLUDED_PATHS`, and per-method exceptions inside `is_auth_excluded_route(path, method)`.

On top of that:
- `@require_admin_role()` (`decorators/admin`) wraps controllers. It returns 401 when there is no user and 403 when the user is not an admin.
- `utils/auth.py` provides `get_current_user_id` (optional auth) and `require_auth`, both meant for `Depends` on public-but-personalized routes.

Tokens:
- Access and refresh tokens are JWTs (`SECRET_KEY`, `ALGORITHM`, expiry from config).
- They are persisted in a `Tokens` table, and `/auth/login/refresh-token` issues new ones.
- Passwords are hashed with bcrypt through `utils/password.py`.
- `utils/encryption.py` handles symmetric encryption, with salt and IV taken from env.

### SDK clients

`utils/fetch_client.APIClient(base_url, headers, timeout=60)` wraps `requests` and exposes `get/post/put/delete`. It raises on non-2xx responses and returns `None` on empty ones. Each external service is a package:

```python
# SDK/<service>/__init__.py
class ServiceClient:
    def __init__(self) -> None:
        self.client = APIClient(base_url=service_config.BASE_URL,
                                headers={"Content-Type": "application/json", "X-Api-Key": service_config.API_KEY})
        self.products = Products(self.client)      # SDK/<service>/products.py
    def test_service(self) -> dict | None:
        return self.client.get("/")
service_client = ServiceClient()
```

Sub-module methods parse responses with `<Model>.model_validate(response)`.

### Workers

- `workers/<name>.py` defines a class with async methods plus a module singleton or a `run_<name>()` function.
- `utils/worker_scheduler.WorkerScheduler` keeps `dict[str, asyncio.Task]` and offers `add_task(name, func, interval)`, `remove_task`, `start`, `stop` and `get_task_status`. Each loop iteration is wrapped in try/except so one error doesn't kill the task.
- `utils/worker_initializer.initialize_workers()` registers every worker with its interval in seconds. `get_worker_status()` reports their state.
- Workers handle items one at a time, each in its own try/except, so one bad record doesn't stop the batch.

### Logging and observability

- `ErrorLogMiddleware` captures every 4xx/5xx response (it consumes streaming bodies). It:
  - computes a stable fingerprint;
  - takes the request id from `request.state` or the `x-request-id` header, or generates one, and writes it back as the `x-request-id` response header;
  - persists asynchronously (`asyncio.create_task` plus `run_in_executor`) into `app_errors`;
  - is controlled by `ENABLE_ERROR_LOGS`, `LOG_SAMPLE_RATE` and `SERVICE_NAME`.
- `RequestLoggingMiddleware` stores request logs with sanitized headers, a body truncated to 4,000 characters, duration and tags. An admin "telescope" UI reads them.
- `utils/websocket_logger` is an in-memory ring buffer plus a WebSocket stream at `/logs/ws`. It persists ERROR-level entries.

### Tests

`tests/conftest.py`:
- creates a file-backed SQLite engine and runs `SQLModel.metadata.create_all` once per session;
- defines a `db_session` fixture (function scope) that is rolled back and expunged after each test;
- defines `api_client = TestClient(app)`;
- adds seed fixtures (`seed_test_user`, `seed_admin_user`, `seed_plan`, ...) and an `auth_headers` fixture that logs in through the API.

Test files are `tests/test_<domain>.py`. Service tests are grouped in `class Test<Domain>Services`, and `unittest.mock.patch` stands in for SDK calls. `pytest.ini` sets `asyncio_mode = auto` and `testpaths = tests`.

### Tooling and CI

- **mypy** (`mypy.ini`): `disallow_untyped_defs`, `disallow_incomplete_defs`, `disallow_untyped_calls`, `check_untyped_defs`, `no_implicit_optional` and `warn_return_any`, with the pydantic and sqlalchemy plugins. `tests`, `alembic` and `scripts` are relaxed.
- **CI** (GitHub Actions): on push to `develop` it runs `mypy . --ignore-missing-imports` and `pytest tests/`, then deploys over SSH with `docker compose up -d --build`. A separate workflow tests the `testing` branch.
- **Docker**: `Docker/Dockerfile` (`pip install -r requirements.txt`, `CMD fastapi run main.py`) and `docker-compose.yml` (`env_file: .env`, volume for uploads).
- `requirements.txt` is a full `pip freeze` with pinned versions.

### Docs

`docs/<subsystem>/*.md` covers the layered architecture, workers, SDK and error logging, plus one doc per feature service. Each follows the same outline: Overview, Architecture components, Directory structure, Implementation patterns, Adding new X, Best practices, Testing.

---

## Where docs and code disagree

| Topic | Docs say | Code does |
|---|---|---|
| Service error decorator | `@log_service_errors` in `decorators/service_error_logging.py`, applied to all services | The source file doesn't exist; only a stale `.pyc` remains and nothing uses it |
| SDK errors | SDK doc: catch, print, return `None`. Error-logging doc: log and re-raise | `APIClient` raises on non-2xx and returns `None` on empty; sub-modules vary |
| Error body key | `{"message": ...}` | about 205 controller responses use `message`, 55 use `error`, 6 use `detail`; middleware uses `error`, global handlers use `detail` |
| Python version | 3.13 (mypy, CI) | Dockerfile uses `python:3.11` |
| Logging | "Comprehensive logging" | about 200 `print()` calls versus about 18 `logger` calls in services and controllers |
| Pagination | one pattern | two shapes (`limit/pages` and `per_page/total_pages`), and one admin list paginates in Python after loading every row |

---

## Recommended additions (not in the source; use in new projects)

1. **Response envelope.** Use one shape for everything: success `{"data": ..., "message"?: str}` and error `{"error": {"code": str, "message": str}}`, with no `str(e)` in 500 responses. Build them with a helper in `common/response_models/envelope.py`.
2. **Exception base class.** Add `class AppError(Exception)` with `status_code` and `code` attributes, and have domain exceptions subclass `NotFoundError(AppError)`, `ConflictError`, `ValidationError`, `ForbiddenError`. Register one `@app.exception_handler(AppError)`. Controllers then drop their repeated `except` ladders and only catch what they actually handle.
3. **Middleware order.** Add CORS last so it is outermost. That removes every hand-written CORS header block.
4. **Session per request through `Depends(get_session)`.** Pass it from route to controller to service so tests can override it with `app.dependency_overrides[get_session]`. Today services that use the global `engine` bypass the SQLite test engine.
5. **Repository layer (optional, for larger domains).** Put queries in `repositories/<domain>.py` so services hold only business rules. This also centralizes the `deleted_at IS NULL` filter in a base helper.
6. **Model base.** A `TimestampedModel` mixin with `id`, `uuid`, and `created_at`/`updated_at`/`deleted_at` as timezone-aware `datetime`, not strings.
7. **Config through `pydantic-settings`.** Use `BaseSettings` with types and fail at startup on missing secrets instead of falling back to `" "`.
8. **Async I/O.** Use `httpx.AsyncClient` in `APIClient`, because `requests` blocks the event loop inside async controllers. Or run sync calls in `run_in_threadpool`.
9. **Logging.** Use the standard `logging` module with a JSON formatter and a request id in every record, and ban `print`. Restore or re-implement `@log_service_errors`.
10. **Pagination.** Use a single `Page[T]` generic `{items, total, page, limit, pages}` and paginate in SQL, never in Python.
11. **Auth.** Keep `route_exclusions` as the single allow-list, but prefer a `Depends(require_user)` / `Depends(require_admin)` guard per router, which is explicit and testable. Don't put `str(e)` in 401 details.
12. **Alembic.** Use descriptive revision messages (not `updates_on_database`), review every autogenerated diff, and add a CI check that runs `alembic upgrade head` against a fresh DB.
13. **Tooling.** Add `ruff` (lint and format) and `pre-commit`. Keep `requirements.in` with direct dependencies and pin with `pip-compile`. Add a `.env.example`.
14. **CI secrets.** Never inline env values in workflow YAML; use the platform's secret store. The source repo had test keys and server details written into workflow files.
