# Testing (pytest plugin)

The plugin ships with anyio (`pytest11` entry point) — no extra install. Decorate
async tests with `@pytest.mark.anyio`:

```python
import pytest


@pytest.mark.anyio
async def test_thing() -> None:
    assert await something() == 42
```

- Mark every async test with `@pytest.mark.anyio` (or set `pytestmark =
  pytest.mark.anyio` at module level to cover the whole file).
- **Async fixtures** are supported and run on the same backend; just define them with
  `@pytest.fixture` and `async def`. They work inside anyio-marked tests.
- Port helpers: `free_tcp_port` / `free_udp_port` (function-scoped),
  `free_tcp_port_factory` / `free_udp_port_factory` (session-scoped).
- Test/introspection helpers: `anyio.wait_all_tasks_blocked()` (deterministic
  scheduling in tests), `anyio.get_current_task()`, `anyio.get_running_tasks()`.
