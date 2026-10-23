# Phase 1: Platform Foundation and Admin Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the backend foundation and a working operator admin panel that onboards distributors, creates shops, issues one-time activation codes, authenticates edge installations, and records every administrative change in an audit log.

**Architecture:** A FastAPI application with async SQLAlchemy and Alembic migrations. Admin pages are server-rendered Jinja2 templates; admin sessions are database-backed opaque tokens in an HttpOnly cookie. Distributors and shops are first-class records; an installation is activated by a distributor code plus a one-time shop activation code, and receives a revocable per-installation API key.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 async, Alembic, PostgreSQL 16, Redis 7, Jinja2, argon2-cffi, pydantic-settings, pytest, pytest-asyncio, httpx, Docker Compose.

**Spec:** `docs/tech.md` §2-§3, §9; `docs/business.md` §6; `plans/00-roadmap.md`.

## Global Constraints

- Python 3.12, PostgreSQL 16, Redis 7.
- Exact dependency pins only (`==`); no ranges; use releases older than 72 hours with CPython 3.12 wheels.
- No Node toolchain. Admin assets are Jinja2 templates, one CSS file, and one first-party `admin.js`.
- Required settings fail startup when missing. No secret defaults.
- All money is integer cents plus ISO-4217 currency.
- All timestamps are UTC `TIMESTAMPTZ`.
- Admin cookies: `HttpOnly`, `SameSite=Lax`, `Secure` when `APP_ENV=production`.
- Raw activation codes and API keys are never stored; only hashes. Raw values render once.
- Every admin mutation and installation change writes an `audit_logs` row.
- All schema changes via Alembic; tests run against PostgreSQL, not SQLite.

## File Structure

```
backend/
  requirements.txt                      runtime pins
  requirements-dev.txt                  test/lint pins
  pyproject.toml                        pytest + ruff configuration
  alembic.ini
  alembic/env.py
  alembic/versions/0001_core.py
  app/
    main.py                             app factory, middleware, router wiring
    config.py                           Settings + get_settings
    db.py                               engine, session factory, get_session
    security.py                         password/token/API-key/CSRF primitives
    cli.py                              create-admin bootstrap command
    models/
      base.py                           Base, UUID primary key mixin
      admin.py                          AdminUser, AdminSession
      distributor.py                    Distributor
      shop.py                           Shop, ActivationCode
      installation.py                   Installation
      audit.py                          AuditLog
    services/
      audit.py                          write_audit
      auth.py                           authenticate_admin, sessions
      distributors.py                   distributor CRUD + code generation
      shops.py                          shop CRUD
      activations.py                    activation codes + installation activation
      rate_limit.py                     RateLimiter Protocol + Redis/InMemory
    api/
      deps.py                           require_admin, require_installation, csrf
      admin_pages.py                    dashboard
      admin_auth.py                     login/logout routes
      admin_distributors.py             distributor routes
      admin_shops.py                    shop + activation routes
      admin_installations.py            installation list/revoke
      health.py                         /health
      installations.py                  /api/v1/installations/activate, /me
    templates/
      base.html
      login.html
      dashboard.html
      distributors/list.html
      distributors/form.html
      shops/list.html
      shops/form.html
      shops/activations.html
      installations/list.html
    static/admin.css
    static/admin.js
  tests/
    conftest.py
    test_health.py
    test_config.py
    test_security.py
    test_admin_auth.py
    test_admin_distributors.py
    test_admin_shops.py
    test_activations.py
    test_installations.py
    test_rate_limit.py
    test_security_headers.py
docker-compose.yml
.env.example
```

## Interfaces

- `create_app(settings: Settings | None = None) -> FastAPI`
- `get_settings() -> Settings`
- `get_session() -> AsyncIterator[AsyncSession]`
- `require_admin(request: Request, db: AsyncSession) -> AdminUser`
- `require_installation(authorization: str, db: AsyncSession) -> Installation`
- `write_audit(db, *, actor_type, actor_id, action, entity_type, entity_id, metadata=None) -> AuditLog`
- `hash_password(password: str) -> str`, `verify_password(password, password_hash) -> bool`
- `hash_token(raw: str) -> str`, `hash_api_key(raw: str, pepper: str) -> str`
- `create_admin_session(db, admin_user, ttl_minutes) -> tuple[AdminSession, str]`
- `create_distributor(db, *, name, contact_email, commission_rate_bps, activation_bounty_cents, actor) -> Distributor`
- `create_shop(db, *, name, slug, distributor_id, timezone, currency, actor) -> Shop`
- `generate_activation_code(db, shop_id, ttl_hours, actor) -> tuple[ActivationCode, str]`
- `activate_installation(db, *, distributor_code, activation_code, machine_fingerprint, agent_version) -> ActivationResult`
- `verify_installation_key(db, raw_key: str) -> Installation | None`

## Review Focus

1. **Activation replay.** Two installers activate the same code, or the same code is retried after a network timeout, and two installations are created or the second silently reuses the first. Expected: exactly one installation per code, second attempt rejected.
2. **Distributor mismatch.** The installer enters a distributor code that does not own the shop, redirecting commissions. Expected: activation rejected with no installation created.
3. **Admin authorization and CSRF.** A crafted POST or a logged-out browser changes a distributor, shop, or installation. Expected: redirect to login or 403, with no state change.
4. **Key exposure.** A raw API key or activation code appears in the database, logs, or a second render of the page. Expected: only hashes at rest, raw shown once.
5. **Audit gaps.** A mutation succeeds without an audit row, making commission disputes unresolvable. Expected: one audit row per accepted mutation.

---

### Task 1: Backend scaffold, configuration, and health endpoint

**Files:**
- Create: `backend/requirements.txt`
- Create: `backend/requirements-dev.txt`
- Create: `backend/pyproject.toml`
- Create: `backend/app/__init__.py`
- Create: `backend/app/config.py`
- Create: `backend/app/main.py`
- Create: `backend/tests/conftest.py`
- Create: `backend/tests/test_health.py`
- Create: `backend/tests/test_config.py`
- Create: `.env.example`

**Interfaces:**
- Produces: `Settings`, `get_settings() -> Settings`, `create_app(settings: Settings | None = None) -> FastAPI`, `GET /health -> {"status": "ok", "version": "0.1.0"}`.

- [ ] **Step 1: Pin dependencies**

Resolve each package to the latest stable release older than 72 hours that ships a CPython 3.12 wheel. Write exact `==` pins.

`backend/requirements.txt`:
```
fastapi==<resolved>
uvicorn[standard]==<resolved>
sqlalchemy[asyncio]==<resolved>
asyncpg==<resolved>
alembic==<resolved>
pydantic-settings==<resolved>
jinja2==<resolved>
argon2-cffi==<resolved>
python-multipart==<resolved>
redis==<resolved>
celery[redis]==<resolved>
httpx==<resolved>
```

`backend/requirements-dev.txt`:
```
-r requirements.txt
pytest==<resolved>
pytest-asyncio==<resolved>
ruff==<resolved>
```

- [ ] **Step 2: Write the failing tests**

`backend/tests/test_health.py`:
```python
from fastapi.testclient import TestClient

from app.main import create_app


def test_health_returns_ok(client):
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok", "version": "0.1.0"}
```

`backend/tests/test_config.py`:
```python
import pytest
from pydantic import ValidationError

from app.config import Settings


def test_settings_require_database_url(monkeypatch):
    monkeypatch.delenv("DATABASE_URL", raising=False)
    monkeypatch.setenv("API_KEY_PEPPER", "x" * 32)
    with pytest.raises(ValidationError):
        Settings()


def test_settings_reject_empty_pepper(monkeypatch):
    monkeypatch.setenv("DATABASE_URL", "postgresql+asyncpg://u:p@localhost/db")
    monkeypatch.setenv("API_KEY_PEPPER", "")
    with pytest.raises(ValidationError):
        Settings()
```

`backend/tests/conftest.py`:
```python
import pytest
from fastapi.testclient import TestClient

from app.config import Settings
from app.main import create_app


@pytest.fixture
def settings() -> Settings:
    return Settings(
        app_env="test",
        database_url="postgresql+asyncpg://visitmemory:visitmemory@localhost:5432/visitmemory_test",
        redis_url="redis://localhost:6379/1",
        api_key_pepper="test-pepper-0123456789abcdef0123456789abcdef",
        secure_cookies=False,
    )


@pytest.fixture
def client(settings) -> TestClient:
    with TestClient(create_app(settings)) as test_client:
        yield test_client
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_health.py tests/test_config.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'app'` or missing `Settings`.

- [ ] **Step 4: Implement configuration and app factory**

`backend/app/config.py` defines:
```python
class Settings(BaseSettings):
    app_env: Literal["development", "test", "production"] = "development"
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    api_key_pepper: str = Field(min_length=32)
    session_ttl_minutes: int = 720
    secure_cookies: bool = True
    log_level: str = "INFO"
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
```
`backend/app/main.py` defines `create_app(settings=None)` that wires a `/health` route and returns the app. Do not add admin routes yet.

`.env.example` lists every name with placeholder values and no real secrets.

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_health.py tests/test_config.py -v`
Expected: PASS (3 tests).

- [ ] **Step 6: Commit**

```bash
git add backend .env.example
git commit -m "feat: scaffold backend app with validated settings and health endpoint"
```

---

### Task 2: Database engine, session dependency, and Docker Compose

**Files:**
- Create: `docker-compose.yml`
- Create: `backend/alembic.ini`
- Create: `backend/alembic/env.py`
- Create: `backend/alembic/script.py.mako`
- Create: `backend/app/db.py`
- Modify: `backend/tests/conftest.py`
- Create: `backend/tests/test_database.py`

**Interfaces:**
- Consumes: `Settings` from Task 1.
- Produces: `create_engine(settings) -> AsyncEngine`, `create_session_factory(engine) -> async_sessionmaker[AsyncSession]`, `get_session() -> AsyncIterator[AsyncSession]`.

- [ ] **Step 1: Write the failing test**

`backend/tests/test_database.py`:
```python
import pytest
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


@pytest.mark.asyncio
async def test_database_connection(db_session: AsyncSession):
    result = await db_session.execute(text("SELECT 1"))
    assert result.scalar_one() == 1
```

Add a `db_session` fixture to `backend/tests/conftest.py` that creates a session from `TEST_DATABASE_URL` and rolls back after each test.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && pytest tests/test_database.py -v`
Expected: FAIL — missing `db_session` fixture or connection module.

- [ ] **Step 3: Implement engine, session factory, and compose services**

`docker-compose.yml` defines `postgres:16` (user/password/db `visitmemory`, test db `visitmemory_test`) and `redis:7` with healthchecks.

`backend/app/db.py`:
```python
def create_engine(settings: Settings) -> AsyncEngine: ...
def create_session_factory(engine: AsyncEngine) -> async_sessionmaker[AsyncSession]: ...
async def get_session() -> AsyncIterator[AsyncSession]: ...
```

`alembic/env.py` reads `DATABASE_URL`, imports `app.models.Base.metadata`, and supports async migrations.

- [ ] **Step 4: Run test to verify it passes**

Run: `docker compose up -d postgres redis && cd backend && pytest tests/test_database.py -v`
Expected: PASS (1 test).

- [ ] **Step 5: Commit**

```bash
git add docker-compose.yml backend/alembic.ini backend/alembic backend/app/db.py backend/tests
git commit -m "feat: add postgres/redis compose services and async session layer"
```

---

### Task 3: Core models and initial migration

**Files:**
- Create: `backend/app/models/base.py`
- Create: `backend/app/models/admin.py`
- Create: `backend/app/models/distributor.py`
- Create: `backend/app/models/shop.py`
- Create: `backend/app/models/installation.py`
- Create: `backend/app/models/audit.py`
- Create: `backend/app/models/__init__.py`
- Create: `backend/alembic/versions/0001_core.py`
- Create: `backend/tests/models/test_constraints.py`

**Interfaces:**
- Produces SQLAlchemy models: `AdminUser`, `AdminSession`, `Distributor`, `Shop`, `ActivationCode`, `Installation`, `AuditLog` with these fields:
  - `AdminUser(id, email, password_hash, role, is_active, created_at)`
  - `AdminSession(id, admin_user_id, token_hash, csrf_token, expires_at, revoked_at, created_at)`
  - `Distributor(id, code, name, contact_email, commission_rate_bps, activation_bounty_cents, is_active, created_at)`
  - `Shop(id, distributor_id, slug, name, timezone, currency, is_active, created_at)`
  - `ActivationCode(id, shop_id, code_hash, expires_at, used_at, used_by_installation_id, created_at)`
  - `Installation(id, shop_id, machine_fingerprint, agent_version, kind, api_key_prefix, api_key_hash, last_seen_at, revoked_at, created_at)`
  - `AuditLog(id, actor_type, actor_id, action, entity_type, entity_id, metadata, created_at)`

- [ ] **Step 1: Write the failing constraint tests**

`backend/tests/models/test_constraints.py` covers:
```python
async def test_distributor_code_is_unique(db_session): ...
async def test_shop_slug_is_unique(db_session): ...
async def test_activation_code_hash_is_unique(db_session): ...
async def test_admin_email_is_unique_case_insensitive(db_session): ...
```
Each inserts valid rows twice and asserts `IntegrityError`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/models/test_constraints.py -v`
Expected: FAIL — tables or models do not exist.

- [ ] **Step 3: Define models and migration**

Use UUIDv4 primary keys, `TIMESTAMPTZ` server defaults, `Numeric` never for money (use `Integer` cents). Add:
- `installations.kind` string with values `CAMERA|POS_HELPER`, default `CAMERA`, and a check constraint
- unique index on `lower(admin_users.email)`
- unique `distributors.code`, `distributors.name`
- unique `shops.slug`
- unique `activation_codes.code_hash`
- index `installations.api_key_prefix`
- index `audit_logs(entity_type, entity_id)`

- [ ] **Step 4: Apply migration and run tests**

Run: `cd backend && alembic upgrade head && pytest tests/models/test_constraints.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/models backend/alembic/versions/0001_core.py backend/tests/models
git commit -m "feat: add core distributor, shop, activation, installation, and audit models"
```

---

### Task 4: Security primitives

**Files:**
- Create: `backend/app/security.py`
- Create: `backend/tests/test_security.py`

**Interfaces:**
- Produces:
  - `hash_password(password: str) -> str`
  - `verify_password(password: str, password_hash: str) -> bool`
  - `generate_token(nbytes: int = 32) -> str`
  - `hash_token(raw: str) -> str`
  - `hash_api_key(raw: str, pepper: str) -> str`
  - `generate_csrf_token() -> str`
  - `constant_time_compare(left: str, right: str) -> bool`

- [ ] **Step 1: Write the failing tests**

```python
def test_password_hash_roundtrip():
    encoded = hash_password("correct horse battery staple")
    assert encoded != "correct horse battery staple"
    assert verify_password("correct horse battery staple", encoded) is True
    assert verify_password("wrong", encoded) is False


def test_api_key_hash_is_pepper_dependent():
    assert hash_api_key("vm_live_abc", "a" * 32) != hash_api_key("vm_live_abc", "b" * 32)


def test_hash_token_is_stable():
    assert hash_token("abc") == hash_token("abc")
    assert hash_token("abc") != hash_token("abd")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_security.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement primitives**

Use `argon2.PasswordHasher` with Argon2id defaults. Use `secrets.token_urlsafe` for tokens and CSRF values, `hashlib.sha256` for opaque token hashes, and `hmac.new(pepper.encode(), raw.encode(), hashlib.sha256).hexdigest()` for API keys. `verify_password` catches `VerifyMismatchError` and returns `False`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_security.py -v`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/security.py backend/tests/test_security.py
git commit -m "feat: add argon2, opaque token, HMAC API key, and CSRF primitives"
```

---

### Task 5: Admin login, database-backed sessions, and CSRF

**Files:**
- Create: `backend/app/services/audit.py`
- Create: `backend/app/services/auth.py`
- Create: `backend/app/api/admin_auth.py`
- Create: `backend/app/api/deps.py`
- Create: `backend/app/templates/base.html`
- Create: `backend/app/templates/login.html`
- Create: `backend/app/static/admin.css`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_auth.py`

**Interfaces:**
- Consumes: `AdminUser`, `AdminSession`, security primitives.
- Produces:
  - `write_audit(db, *, actor_type, actor_id, action, entity_type, entity_id, metadata=None) -> AuditLog`
  - `authenticate_admin(db, email: str, password: str) -> AdminUser | None`
  - `create_admin_session(db, admin_user: AdminUser, ttl_minutes: int) -> tuple[AdminSession, str]`
  - `resolve_admin_session(db, raw_token: str) -> AdminSession | None`
  - `revoke_admin_session(db, session: AdminSession) -> None`
  - `require_admin(request: Request, db: AsyncSession) -> AdminUser`
  - `verify_csrf(request: Request, form_token: str) -> None` raising HTTP 403 on mismatch
  - Routes: `GET /admin/login`, `POST /admin/login`, `POST /admin/logout`

- [ ] **Step 1: Write the failing auth tests**

`backend/tests/test_admin_auth.py` covers:
```python
async def test_login_sets_httponly_cookie(client, admin_user): ...
async def test_login_with_wrong_password_returns_401(client, admin_user): ...
async def test_admin_page_redirects_when_logged_out(client): ...
async def test_post_without_csrf_returns_403(client, logged_in_client): ...
async def test_logout_revokes_session(client, logged_in_client): ...
async def test_inactive_admin_cannot_login(client, inactive_admin): ...
async def test_expired_session_is_rejected(client, expired_session): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_auth.py -v`
Expected: FAIL — routes and fixtures missing.

- [ ] **Step 3: Implement session auth**

Cookie name is `vm_admin_session`. On login: normalize email to lowercase, verify Argon2 hash, create a 32-byte raw token, store `hash_token(raw)` plus a CSRF token, set cookie `HttpOnly`, `SameSite=Lax`, `Secure` when enabled, `Path=/admin`. `require_admin` resolves the cookie, rejects missing/expired/revoked sessions, and returns the `AdminUser`. Every login success/failure and logout writes an audit row with `actor_type="ADMIN"` and no password data.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_auth.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/audit.py backend/app/services/auth.py backend/app/api backend/app/templates backend/app/static backend/app/main.py backend/tests/test_admin_auth.py
git commit -m "feat: add database-backed admin sessions, login, logout, and CSRF"
```

---

### Task 6: Admin layout, dashboard, and distributor CRUD

**Files:**
- Create: `backend/app/services/distributors.py`
- Create: `backend/app/api/admin_pages.py`
- Create: `backend/app/api/admin_distributors.py`
- Create: `backend/app/templates/dashboard.html`
- Create: `backend/app/templates/distributors/list.html`
- Create: `backend/app/templates/distributors/form.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_distributors.py`

**Interfaces:**
- Consumes: `require_admin`, `verify_csrf`, `write_audit`.
- Produces:
  - `generate_distributor_code(db) -> str` using `D-` plus 8 Crockford base32 characters, retrying on collision.
  - `create_distributor(db, *, name, contact_email, commission_rate_bps, activation_bounty_cents, actor) -> Distributor`
  - `update_distributor(db, distributor_id, *, name, contact_email, commission_rate_bps, activation_bounty_cents, actor) -> Distributor`
  - `set_distributor_active(db, distributor_id, is_active, actor) -> Distributor`
  - Routes: `GET /admin`, `GET /admin/distributors`, `GET/POST /admin/distributors/new`, `GET/POST /admin/distributors/{id}/edit`

- [ ] **Step 1: Write the failing tests**

```python
async def test_create_distributor_generates_code_and_audit(logged_in_client, db_session): ...
async def test_distributor_name_must_be_unique(logged_in_client): ...
async def test_distributor_code_matches_format(logged_in_client): ...
async def test_edit_distributor_updates_commission_rate(logged_in_client): ...
async def test_deactivate_distributor_hides_it_from_active_default(logged_in_client): ...
async def test_dashboard_counts_distributors_and_shops(logged_in_client): ...
async def test_distributor_routes_require_login(client): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_distributors.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement distributor routes and templates**

Forms post `name`, `contact_email`, `commission_rate_bps`, `activation_bounty_cents`, and `csrf_token`. Defaults are 2500 bps and 2000 cents. Validate bps in `[0, 10000]` and bounty `>= 0`. Render active distributors by default; support `?include_inactive=1`. Every create/update/deactivate writes an audit row with the changed field names but no secrets.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_distributors.py -v`
Expected: PASS (7 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/distributors.py backend/app/api/admin_pages.py backend/app/api/admin_distributors.py backend/app/templates backend/tests/test_admin_distributors.py
git commit -m "feat: add admin dashboard and distributor management"
```

---

### Task 7: Shop CRUD and distributor assignment

**Files:**
- Create: `backend/app/services/shops.py`
- Create: `backend/app/api/admin_shops.py`
- Create: `backend/app/templates/shops/list.html`
- Create: `backend/app/templates/shops/form.html`
- Create: `backend/app/templates/shops/activations.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_shops.py`

**Interfaces:**
- Consumes: `Distributor`, `require_admin`.
- Produces:
  - `normalize_slug(name: str) -> str` lower-case, ASCII, hyphenated, max 64 chars.
  - `create_shop(db, *, name, slug, distributor_id, timezone, currency, actor) -> Shop`
  - `update_shop(db, shop_id, *, name, distributor_id, timezone, currency, actor) -> Shop`
  - `set_shop_active(db, shop_id, is_active, actor) -> Shop`
  - Routes: `GET /admin/shops`, `GET/POST /admin/shops/new`, `GET/POST /admin/shops/{id}/edit`, `GET /admin/shops/{id}/activations`

- [ ] **Step 1: Write the failing tests**

```python
async def test_create_shop_assigns_distributor(logged_in_client, distributor): ...
async def test_shop_slug_is_normalized_and_unique(logged_in_client): ...
async def test_shop_requires_existing_distributor(logged_in_client): ...
async def test_reassigning_shop_writes_audit_with_old_and_new_distributor(logged_in_client): ...
async def test_shop_list_filters_by_distributor(logged_in_client): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_shops.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement shop routes**

Default timezone `Asia/Kuala_Lumpur`, default currency `MYR`. Allow a null distributor only when an explicit `direct` checkbox is set; otherwise require one. Audit entries include `distributor_id` before and after on reassignment.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_shops.py -v`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/shops.py backend/app/api/admin_shops.py backend/app/templates/shops backend/tests/test_admin_shops.py
git commit -m "feat: add shop management with distributor assignment"
```

---

### Task 8: One-time activation codes and installation activation API

**Files:**
- Create: `backend/app/services/activations.py`
- Create: `backend/app/api/installations.py`
- Modify: `backend/app/api/deps.py`
- Modify: `backend/app/api/admin_shops.py`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_activations.py`
- Create: `backend/tests/test_installations.py`

**Interfaces:**
- Consumes: `Shop`, `Distributor`, `ActivationCode`, `Installation`, `hash_api_key`, `write_audit`.
- Produces:
  - `generate_activation_code(db, shop_id: UUID, ttl_hours: int, actor: AdminUser) -> tuple[ActivationCode, str]`
  - `ActivationResult` dataclass with `installation_id: UUID`, `shop_id: UUID`, `shop_name: str`, `distributor_id: UUID | None`, `api_key: str`
  - `activate_installation(db, *, distributor_code: str, activation_code: str, machine_fingerprint: str, agent_version: str, installation_kind: Literal["CAMERA", "POS_HELPER"] = "CAMERA") -> ActivationResult`
  - `verify_installation_key(db, raw_key: str) -> Installation | None`
  - `require_installation(authorization: str, db: AsyncSession) -> Installation`
  - `POST /api/v1/installations/activate` returning `201` with `installation_id`, `shop_id`, `shop_name`, `api_key`
  - `GET /api/v1/installations/me` returning shop, distributor, and installation status

- [ ] **Step 1: Write the failing activation tests**

`backend/tests/test_activations.py` covers:
```python
async def test_activation_returns_api_key_once_and_stores_hash(...): ...
async def test_activation_rejects_wrong_distributor(...): ...
async def test_activation_rejects_expired_code(...): ...
async def test_activation_rejects_reused_code(...): ...
async def test_activation_is_atomic_under_concurrent_requests(...): ...
async def test_activation_records_installation_kind(...): ...
async def test_activation_writes_audit_rows(...): ...
```
`backend/tests/test_installations.py` covers:
```python
async def test_me_returns_shop_and_distributor_with_valid_key(...): ...
async def test_me_rejects_unknown_key(...): ...
async def test_me_rejects_revoked_key(...): ...
async def test_me_updates_last_seen_at(...): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_activations.py tests/test_installations.py -v`
Expected: FAIL — activation module and routes missing.

- [ ] **Step 3: Implement activation**

Code format: `A-` plus 10 Crockford base32 characters, displayed once. Store `hash_token(raw_code)`.

Activation transaction:
1. `SELECT ... FOR UPDATE` the activation code by hash.
2. Reject missing (404), expired (410), used (409), or shop distributor code mismatch (404).
3. Generate raw API key `vm_live_` + `secrets.token_urlsafe(32)`; store `api_key_prefix = raw_key[:16]` and `hash_api_key(raw_key, pepper)`.
4. Insert `Installation` with the requested `kind` (default `CAMERA`), set `used_at` and `used_by_installation_id`. The activation request accepts `installation_kind` and rejects any value outside `CAMERA|POS_HELPER`.
5. Commit, then write audit rows for code use and installation creation.

The API key is returned only in this response. Never log the raw key, code, or request body. Apply a placeholder rate limit dependency `activation_rate_limit` that Task 10 backs with Redis.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_activations.py tests/test_installations.py -v`
Expected: PASS (10 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/activations.py backend/app/api/installations.py backend/app/api/deps.py backend/app/api/admin_shops.py backend/app/main.py backend/tests
git commit -m "feat: add one-time activation codes and installation API key auth"
```

---

### Task 9: Admin views for activations and installations

**Files:**
- Modify: `backend/app/api/admin_shops.py`
- Create: `backend/app/api/admin_installations.py`
- Modify: `backend/app/templates/shops/activations.html`
- Create: `backend/app/templates/installations/list.html`
- Modify: `backend/app/main.py`
- Create: `backend/tests/test_admin_installations.py`

**Interfaces:**
- Consumes: `generate_activation_code`, `Installation`.
- Produces:
  - `POST /admin/shops/{id}/activation-codes` rendering the raw code once in the response
  - `GET /admin/installations`
  - `POST /admin/installations/{id}/revoke`
  - `revoke_installation(db, installation_id, actor) -> Installation`

- [ ] **Step 1: Write the failing tests**

```python
async def test_generate_code_shows_raw_value_once_and_never_again(logged_in_client, shop): ...
async def test_revoke_installation_sets_revoked_at_and_audits(logged_in_client, installation): ...
async def test_revoked_key_cannot_call_me(logged_in_client, installation): ...
async def test_activation_list_marks_used_and_expired(logged_in_client, shop): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_admin_installations.py -v`
Expected: FAIL — routes missing.

- [ ] **Step 3: Implement admin views**

The raw code is rendered only in the immediate POST response with a `Cache-Control: no-store` header. Reloading the activations page shows only code prefix, created, expiry, and used status. Revocation writes `revoked_at` and an audit row; a second revoke is idempotent and does not duplicate the audit row.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd backend && pytest tests/test_admin_installations.py -v`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add backend/app/api backend/app/templates backend/tests/test_admin_installations.py
git commit -m "feat: add admin activation code and installation management"
```

---

### Task 10: Rate limiting, security headers, bootstrap CLI, and final hardening

**Files:**
- Create: `backend/app/services/rate_limit.py`
- Create: `backend/app/cli.py`
- Modify: `backend/app/main.py`
- Modify: `backend/app/api/admin_auth.py`
- Modify: `backend/app/api/installations.py`
- Create: `backend/tests/test_rate_limit.py`
- Create: `backend/tests/test_security_headers.py`
- Create: `backend/tests/test_cli.py`

**Interfaces:**
- Consumes: Redis settings.
- Produces:
  - `RateLimiter` Protocol with `hit(key: str, limit: int, window_seconds: int) -> bool`
  - `RedisRateLimiter(redis_client)`
  - `InMemoryRateLimiter()` for tests
  - `create_admin(email: str, password: str) -> AdminUser` used by `python -m app.cli create-admin`
  - Security headers middleware

- [ ] **Step 1: Write the failing tests**

```python
async def test_login_throttles_after_five_failures(client, admin_user): ...
async def test_activation_throttles_after_ten_requests(client): ...
async def test_security_headers_present(client): ...
async def test_cli_rejects_password_shorter_than_twelve_chars(db_session): ...
async def test_cli_is_idempotent_for_same_email(db_session): ...
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd backend && pytest tests/test_rate_limit.py tests/test_security_headers.py tests/test_cli.py -v`
Expected: FAIL — modules missing.

- [ ] **Step 3: Implement limits, headers, and CLI**

Login key: `login:{email_lower}:{client_ip}` with 5 failures per 900 seconds. Activation key: `activation:{client_ip}` with 10 requests per 60 seconds. Successful login clears the counter.

Security headers: `Content-Security-Policy: default-src 'self'; frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `X-Frame-Options: DENY`.

CLI reads the password from `getpass`, requires 12 or more characters, and does not echo it. Creating an existing email updates the password only with `--reset-password`; otherwise it exits non-zero.

- [ ] **Step 4: Run the full phase suite**

Run: `cd backend && pytest -v`
Expected: PASS (all Phase 1 tests).

- [ ] **Step 5: Commit**

```bash
git add backend
git commit -m "feat: add throttling, security headers, and admin bootstrap CLI"
```

---

## Phase 1 Exit Checklist

- [ ] An operator can log in, create a distributor, create a shop assigned to it, and generate a one-time activation code.
- [ ] An unauthenticated activation request with the correct distributor code and activation code returns `201` and an API key that works on `/api/v1/installations/me`.
- [ ] Reusing the code, using a wrong distributor, or revoking the installation is rejected.
- [ ] Raw codes and keys exist only in the one response that created them.
- [ ] `audit_logs` contains an entry for every accepted mutation.
- [ ] `pytest` passes against PostgreSQL with no network access beyond the database.
