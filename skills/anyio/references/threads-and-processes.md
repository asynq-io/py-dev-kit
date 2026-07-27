# Threads, processes & subprocesses

## Threads, processes, interpreters

```python
# Sync (blocking) code -> worker thread, awaited from the loop
result = await anyio.to_thread.run_sync(blocking_fn, arg)
# abandon_on_cancel=True lets the task be cancelled while the thread runs on detached
result = await anyio.to_thread.run_sync(blocking_fn, abandon_on_cancel=True)

# Async/loop code <- called from inside a worker thread
def in_thread() -> None:
    anyio.from_thread.run(async_fn, arg)      # run a coroutine on the loop
    anyio.from_thread.run_sync(event.set)     # call a loop-thread sync fn safely

# Foreign thread (not spawned by run_sync) needs a portal:
with anyio.from_thread.start_blocking_portal() as portal:
    portal.call(async_fn)
    future = portal.start_task_soon(async_fn)   # -> concurrent.futures.Future
```

- The default thread pool limiter is **40** threads:
  `anyio.to_thread.current_default_thread_limiter().total_tokens = N`.
- `cancellable=` on `run_sync` is the deprecated alias of `abandon_on_cancel=`.
- **CPU-bound work** → `await anyio.to_process.run_sync(fn, *args)` (pickleable
  args/return, importable top-level `fn`, requires `if __name__ == "__main__":` guard,
  no stdio in the worker). On Python 3.13+ `anyio.to_interpreter.run_sync` runs in a
  subinterpreter (experimental, no cancellation).

## Subprocesses

```python
result = await anyio.run_process(["ls", "-l"])   # list = no shell; str = shell
print(result.returncode, result.stdout)

async with await anyio.open_process(["cat"]) as proc:
    await proc.stdin.send(b"hi"); ...             # proc.stdout/stderr are ByteReceiveStreams
    await proc.wait()
```
