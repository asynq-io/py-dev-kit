# Task groups, cancellation & synchronization

## Task groups (structured concurrency)

```python
async with anyio.create_task_group() as tg:
    tg.start_soon(worker, 1)              # fire-and-forget; returns a TaskHandle
    tg.start_soon(worker, 2)
# block exits only after ALL child tasks finish
```

- The `async with` block **does not exit until every child task completes**.
- If any task (or the body) raises, **all siblings are cancelled** and the errors
  surface as an `ExceptionGroup` (PEP 654). Handle with `except* SomeError:` (3.11+)
  or the `exceptiongroup` backport. A lone cancellation does not wrap.
- `tg.cancel_scope` is the group's scope; `tg.cancel(reason=None)` is shorthand for
  `tg.cancel_scope.cancel(reason)`.

### Spawning APIs

| Call | Use when |
| --- | --- |
| `tg.start_soon(func, *args, name=None) -> TaskHandle` | normal fire-and-forget |
| `tg.create_task(coro, *, name=None, context=None) -> TaskHandle` | you already have a coroutine object, or need a custom `contextvars.Context` |
| `await tg.start(func, *args, name=None, return_handle=False)` | the task must finish initializing before the caller proceeds |

### Initialization barrier with `start()`

The target must accept `task_status` and call `.started(value)` once ready. `await
tg.start(...)` blocks until then and returns that value (raises `RuntimeError` if the
task finishes without calling `started()`).

```python
from anyio.abc import TaskStatus
from anyio import TASK_STATUS_IGNORED

async def serve(port: int, *, task_status: TaskStatus[int] = TASK_STATUS_IGNORED) -> None:
    listener = await anyio.create_tcp_listener(local_port=port)
    task_status.started(port)            # caller unblocks here
    await listener.serve(handle)

async with anyio.create_task_group() as tg:
    bound_port = await tg.start(serve, 0)
```

### TaskHandle

`start_soon`/`create_task`/`start` return a `TaskHandle` (anyio ≥ 4.14):

- `await handle` → the task's return value. Raises `TaskCancelled` if cancelled,
  `TaskFailed` if it raised another exception.
- `handle.cancel()` — cancel just this task.
- `handle.wait()` — await completion without retrieving the value.
- `handle.status` → `TaskHandle.Status.{PENDING, CANCELLING, CANCELLED, FINISHED, FAILED}`.
- `handle.return_value` / `handle.exception` / `handle.start_value` — raise
  `TaskNotFinished` if still pending; `return_value` raises `TaskCancelled`/`TaskFailed` accordingly.

> Do **not** `yield` while inside `async with create_task_group()` in a plain async
> generator — it breaks structured concurrency. It's only safe inside
> `@asynccontextmanager` (same task drives entry and exit) — which is exactly what the
> FastAPI lifespan pattern in `streams.md` relies on.

## Cancellation & timeouts

Timeout helpers are **sync** context managers returning a `CancelScope`:

```python
with anyio.move_on_after(5) as scope:    # silent: just exits the block on timeout
    await slow()
if scope.cancelled_caught:
    ...                                  # timed out

with anyio.fail_after(5):                # raises TimeoutError on timeout
    await slow()
```

`CancelScope(deadline=math.inf, shield=False)` — manual scope. Properties:
`.cancel(reason=None)`, `.deadline` (get/set), `.cancel_called`, `.cancelled_caught`,
`.shield` (get/set). `anyio.current_effective_deadline()` returns the nearest active deadline.

### Shielding cleanup from outer cancellation

```python
async def task() -> None:
    try:
        await do_work()
    except anyio.get_cancelled_exc_class():
        with anyio.CancelScope(shield=True):     # cleanup survives the cancel
            await resource.aclose()
        raise                                    # ALWAYS re-raise
```

- Cancellation is **level-triggered**: once a scope is cancelled, every checkpoint in
  it keeps raising until the scope is exited. You can't "ignore one" and continue.
- Shielding protects against *scope* cancellation, not a direct `handle.cancel()`.
- Cancel scopes must close in strict LIFO order — don't call `__enter__/__exit__`
  manually or stash them in an `AsyncExitStack` out of order.
- Avoid `asyncio.Condition`/`asyncio.Lock` under AnyIO: they can swallow cancellation
  and busy-loop. Use AnyIO's primitives.

## Synchronization primitives

All are constructed directly (no factory, no `await`) and are **not thread-safe**
(see `threads-and-processes.md` for calling them from a worker thread).

```python
event = anyio.Event(); event.set(); await event.wait()      # single-use, not reusable
lock = anyio.Lock();        async with lock: ...            # only holder may release
sem  = anyio.Semaphore(4);  async with sem: ...
cl   = anyio.CapacityLimiter(10)                            # tokens per borrower
cond = anyio.Condition()
async with cond:
    await cond.wait()                                        # or: await cond.wait_for(pred)
async with cond:
    cond.notify(1)   # or cond.notify_all()
```

- `Event` cannot be reset — make a new one after `set()`.
- `Lock`/`Semaphore` accept `fast_acquire=True` to skip the checkpoint when
  uncontended (risks event-loop starvation in tight loops; use sparingly).
- `ResourceGuard()` — `with guard:` raises `BusyResourceError` if two tasks use a
  serial resource (e.g. a socket) concurrently.
- Each primitive has `.statistics()` for debugging.
