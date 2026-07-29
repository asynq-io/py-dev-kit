---
name: fastapi-best-practices
description: Use when developing or reviewing a FastAPI app — routers, endpoints, Pydantic schemas and settings, Depends() injection, httpx tests, project structure.
---

## Project structure

Organize by domain/feature, not by file type:

```
src/
├── {domain}/            # e.g. auth/, posts/
│   ├── router.py        # endpoints
│   ├── schemas.py        # Pydantic models
│   ├── service.py        # business logic
│   ├── dependencies.py   # route dependencies
│   └── exceptions.py     # domain-specific exceptions
├── config.py
└── main.py
```

Cross-domain imports should name the module explicitly (`from src.auth import service
as auth_service`), never `from src.auth import *`.

## Async vs sync routes

| Route does this                       | Use                                          |
|----------------------------------------|-----------------------------------------------|
| `await`-able non-blocking I/O          | `async def`                                   |
| Blocking I/O (no async client exists)  | `def` (sync — FastAPI runs it in a threadpool)|
| Mix of both                            | `async def` + `run_in_threadpool` for the blocking part |
| CPU-bound work                         | Offload to a background worker, not the request path |

```python
# DON'T — a blocking call inside an async route freezes the whole event loop
@router.get("/bad")
async def bad():
    time.sleep(10)
    return {"ok": True}

# DO — sync route, FastAPI runs it in a threadpool
@router.get("/ok")
def ok():
    time.sleep(10)
    return {"ok": True}

# DO — async route calling a sync library
from fastapi.concurrency import run_in_threadpool

@router.get("/wrap")
async def wrap():
    return await run_in_threadpool(legacy_sync_client.fetch, "id")
```

Don't make a route sync "just in case" — threads are more expensive than coroutines,
and the default threadpool is a shared, limited resource.

## Pydantic (v2)

- Use built-in validated types (`EmailStr`, `AnyUrl`, `Field(min_length=...)`,
  `StrEnum`) instead of hand-rolled validators.
- `json_encoders` and `.dict()` are Pydantic v1 and removed in v2. Use
  `@field_serializer` for custom serialization and `.model_dump()` / `.model_dump_json()`.
- Don't combine a constraint with a contradictory default:
  `Field(ge=18, default=None)` is wrong — pick required (`Field(ge=18)`) or optional
  (`int | None = Field(default=None, ge=18)`).
- `pydantic-settings` is a separate package in v2 (`from pydantic_settings import
  BaseSettings`). Scoping one `BaseSettings` subclass per domain (rather than one
  giant global settings object) keeps config easy to reason about, but follow the
  project's existing convention if one exists.

## Dependencies

- Prefer `Annotated[T, Depends(...)]` over the default-argument form
  `x: T = Depends(...)` — it's the idiomatic style since FastAPI 0.95 and avoids
  default-value gotchas.
- Validate *inside* the dependency, don't just inject raw data and validate in the
  route body:
  ```python
  async def valid_post_id(post_id: UUID) -> dict:
      post = await service.get_by_id(post_id)
      if not post:
          raise PostNotFound()
      return post
  ```
- Dependencies are cached per request — the same `Depends(x)` used multiple times in
  one request only runs once. Chain dependencies to reuse validation logic instead of
  duplicating it.
- Prefer `async def` dependencies over sync ones when the project is otherwise async;
  a sync dependency runs in the threadpool, which is unnecessary overhead for a quick
  check.

## Testing

- Use `httpx.AsyncClient` with `ASGITransport` for in-process async tests instead of
  a synchronous test client:
  ```python
  from httpx import AsyncClient, ASGITransport
  from src.main import app

  async def test_example():
      transport = ASGITransport(app=app)
      async with AsyncClient(transport=transport, base_url="http://test") as client:
          resp = await client.get("/health")
          assert resp.status_code == 200
  ```
- Override dependencies with FastAPI's `app.dependency_overrides`, not by
  monkeypatching internals:
  ```python
  app.dependency_overrides[real_dependency] = fake_dependency
  # ... run tests ...
  app.dependency_overrides.clear()
  ```
- Prefer a real (ephemeral/test) database over mocking the DB layer in integration
  tests — mocked and real behavior tend to drift apart over time.

## Common anti-patterns to flag in review

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Sync/blocking call (`requests.get`, `time.sleep`, sync DB driver, `open()`) inside `async def` | Blocks the entire event loop, not just one request | Use an async equivalent, or `await run_in_threadpool(...)` |
| `model_config = ConfigDict(json_encoders={...})` | Removed/deprecated in Pydantic v2 | `@field_serializer` |
| `Field(ge=18, default=None)` | Constraint contradicts the default | Pick required or optional, not both |
| `def get_x(id: int = Depends(...))` (default-arg form) | Legacy style, default-value gotchas | `x: Annotated[T, Depends(...)]` |
| Bare `except Exception:` around a route body | Hides bugs, turns real errors into silent 200s | Catch the specific exception; raise `HTTPException` with a meaningful status |
| Returning a constructed Pydantic model *and* setting `response_model=` to the same class | Model gets built/validated twice | Return a dict/ORM row and let `response_model` validate, or drop `response_model` |
| Mocking the database in integration tests | Mock and real behavior drift apart | Use a real (test) DB + `dependency_overrides` for external services |
| Deep cross-domain imports (`from src.auth.service.user import ...`) | Tight coupling, hard to refactor | `from src.auth import service as auth_service` |
