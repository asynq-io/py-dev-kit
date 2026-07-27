---
name: anyio
description: >-
  Use when writing, reviewing, or debugging async Python that imports `anyio` —
  task groups, cancel scopes, timeouts, memory object streams, sockets, async
  files, threads/processes, or the pytest plugin. Enforces structured
  concurrency and covers the cancellation gotchas that differ from raw asyncio.
---

# Working with AnyIO

AnyIO is a high-level async framework that runs unchanged on **asyncio** or **Trio**.
It enforces *structured concurrency*: every task lives inside a task group, every
timeout/cancel is a scope, and cancellation is *level-triggered* (re-delivered at
every checkpoint until the scope exits) rather than asyncio's edge-triggered one-shot.

When in doubt, prefer AnyIO primitives over `asyncio.*` — mixing them breaks
cancellation semantics and Trio compatibility.

## Golden rules

1. **Never spawn a bare task.** Use a task group; there is no `anyio.create_task`
   that detaches a task from a scope. A task that outlives its spawning scope is a bug.
2. **Always re-raise the cancellation exception.** Catching it and swallowing it
   breaks structured concurrency. Catch with `anyio.get_cancelled_exc_class()` only
   to run cleanup, then `raise`.
3. **`Event.set()`, `CancelScope.cancel()`, `current_time()`, `Lock.release()` are
   synchronous** (no `await`).
4. **Streams raise `EndOfStream`** at EOF — they do not return `b""`/`None`.
5. **Many constructors return awaitables**: `async with await connect_tcp(...)`,
   `async with await open_file(...)`. `create_task_group()`, `CancelScope()`,
   `fail_after()`, `move_on_after()` and the sync primitives do **not** need `await`.

## Core idioms

```python
import anyio

async def main() -> None:
    async with anyio.create_task_group() as tg:   # exits only after ALL children finish
        tg.start_soon(worker, 1)
        tg.start_soon(worker, 2)
    # any failure cancels the siblings; errors surface as an ExceptionGroup (except*)

    with anyio.move_on_after(5) as scope:         # silent timeout (sync context manager)
        await slow()
    if scope.cancelled_caught:
        ...                                       # timed out

    with anyio.fail_after(5):                     # raises TimeoutError on timeout
        await slow()

anyio.run(main)                                   # backend="asyncio" (default) or "trio"
```

`anyio.run(func, *args, backend="asyncio", backend_options={"use_uvloop": True})`.
Event-loop helpers: `await anyio.sleep(seconds)`, `anyio.sleep_forever()`,
`anyio.sleep_until(deadline)`, `anyio.current_time()` (sync, monotonic clock).

Never `yield` inside an open task group in a plain async generator — it is only safe
inside `@asynccontextmanager` (see the lifespan pattern in `references/streams.md`).

## Key exceptions

`EndOfStream`, `BrokenResourceError`, `ClosedResourceError`, `BusyResourceError`,
`WouldBlock` (raised by `*_nowait`), `IncompleteRead`, `DelimiterNotFound`,
`TaskFailed` / `TaskCancelled` / `TaskNotFinished` (from `TaskHandle`),
`NoEventLoopError`, `RunFinishedError`. Get the backend's cancel exception class with
`anyio.get_cancelled_exc_class()`.

## References

Read the file matching the task before writing code:

- `references/tasks-and-cancellation.md` — task group details, spawning APIs,
  `tg.start()` initialization barrier, `TaskHandle`, cancel scopes, shielding,
  synchronization primitives (Event/Lock/Semaphore/Condition/CapacityLimiter).
- `references/streams.md` — memory object streams (the `asyncio.Queue` replacement),
  producer–consumer pattern, FastAPI/ASGI lifespan with background tasks, stream
  wrappers (buffered/text/file/TLS), typed attributes.
- `references/networking-and-files.md` — TCP/UDP/UNIX clients and listeners, async
  file I/O, `anyio.Path`, temp files, signal handling.
- `references/threads-and-processes.md` — `to_thread`/`from_thread`, blocking
  portals, `to_process`, subinterpreters, subprocesses.
- `references/testing.md` — pytest plugin, `@pytest.mark.anyio`, async fixtures,
  port helpers, task introspection.
