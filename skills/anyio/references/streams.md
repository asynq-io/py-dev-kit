# Streams & background-task patterns

## Memory object streams (replace `asyncio.Queue`)

```python
send, receive = anyio.create_memory_object_stream[str](max_buffer_size=10)
```

- `create_memory_object_stream[T](max_buffer_size=0)` — default **0 = rendezvous**
  (sender blocks until a receiver takes the item). Use `math.inf` for unbounded.
- `await send.send(item)` / `send.send_nowait(item)` (sync; raises `WouldBlock`).
  Raises `BrokenResourceError` if all receivers are closed.
- `await receive.receive()` / `async for item in receive:` — the loop ends
  (`EndOfStream` is raised internally) only when **all** send ends are closed.
- `.clone()` either end for multi-producer/consumer; the underlying stream closes only
  when every clone is closed.

## Complete producer–consumer example

Closing the send stream is what lets the consumer's `async for` terminate. Give each
task ownership of its own end with `async with`, and use `.clone()` to fan out.

```python
from __future__ import annotations

import anyio
from anyio.abc import ObjectReceiveStream, ObjectSendStream


async def producer(send: ObjectSendStream[int], start: int) -> None:
    async with send:                       # closing this end signals "no more items"
        for i in range(start, start + 5):
            await send.send(i)


async def consumer(name: str, receive: ObjectReceiveStream[int]) -> None:
    async with receive:                    # closing the clone releases its slot
        async for item in receive:         # ends once ALL send ends are closed
            print(f"{name} got {item}")


async def main() -> None:
    send, receive = anyio.create_memory_object_stream[int](max_buffer_size=10)
    async with anyio.create_task_group() as tg:
        # one consumer per worker — each gets its own receive clone
        for n in range(2):
            tg.start_soon(consumer, f"consumer-{n}", receive.clone())

        # two producers — each gets its own send clone
        async with send:                   # close the ORIGINAL send up front...
            tg.start_soon(producer, send.clone(), 0)
            tg.start_soon(producer, send.clone(), 100)
    # task group exits only after every producer AND consumer has finished


anyio.run(main)
```

The original `send`/`receive` returned by `create_memory_object_stream` count as open
ends. Close the original `send` (here via `async with send`) and hand clones to the
producers, otherwise consumers never see end-of-stream and the group hangs.

## FastAPI / ASGI lifespan with a background task group

A lifespan is an `@asynccontextmanager`, the one place it is safe to `yield` while a
task group is open (the same task enters and exits). Open the group, launch background
tasks, `yield` to run the app, then cancel the scope on shutdown so long-lived tasks
stop cleanly.

```python
from __future__ import annotations

from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

import anyio
from anyio.abc import ObjectReceiveStream, ObjectSendStream
from fastapi import FastAPI


async def queue_worker(receive: ObjectReceiveStream[str]) -> None:
    async with receive:
        async for job in receive:
            await anyio.sleep(0.1)         # ... process the job ...
            print("processed", job)


async def heartbeat() -> None:
    while True:                            # runs until the scope is cancelled
        await anyio.sleep(5)
        print("alive")


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    send, receive = anyio.create_memory_object_stream[str](max_buffer_size=100)
    async with anyio.create_task_group() as tg:
        tg.start_soon(queue_worker, receive)
        tg.start_soon(heartbeat)
        app.state.job_send = send          # request handlers enqueue work via this
        try:
            yield                          # <-- application serves requests here
        finally:
            await send.aclose()            # let the worker drain and exit
            tg.cancel_scope.cancel()       # stop the infinite heartbeat task


app = FastAPI(lifespan=lifespan)


@app.post("/jobs/{job}")
async def submit(job: str) -> dict[str, str]:
    await app.state.job_send.send(job)     # backpressure: blocks if buffer is full
    return {"status": "queued"}
```

Key points:

- The `async with create_task_group()` stays open across `yield`, so the background
  tasks live for the whole application lifetime.
- On shutdown, close the send stream first (so the queue worker drains and returns via
  `EndOfStream`), then `tg.cancel_scope.cancel()` to stop tasks like `heartbeat` that
  loop forever. The task group then exits.
- Request handlers send into the same memory stream — `send.send()` applies
  backpressure (awaits) when the buffer is full, instead of growing unbounded.
- Do **not** stash the task group itself on `app.state` and spawn into it from request
  handlers — that detaches tasks from the structured scope. Communicate via the stream.

## Streams toolbox

- **Stapled**: `StapledByteStream(receive, send)`, `StapledObjectStream(receive, send)` — bidirectional.
- **Buffered**: `BufferedByteReceiveStream(stream)` → `await .receive_exactly(n)`,
  `await .receive_until(delimiter, max_bytes)`.
- **Text**: `TextReceiveStream(byte_stream, encoding="utf-8")` / `TextSendStream(...)`.
- **File**: `await FileReadStream.from_path(path)` / `await FileWriteStream.from_path(path)`.
- **TLS**: `await TLSStream.wrap(stream, ssl_context, server_side=False, server_hostname=...)`,
  `TLSListener(listener, ssl_context, standard_compatible=True)`.
  Set `standard_compatible=False` only for protocols (like HTTP) that skip the TLS close handshake.

Typed metadata uses `stream.extra(SomeAttribute)` (see `anyio.TypedAttributeProvider` /
`TypedAttributeSet` / `typed_attribute`), not asyncio's `get_extra_info()` dict.
