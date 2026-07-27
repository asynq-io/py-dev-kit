# Networking, files & signals

## Networking

```python
# TCP client (TLS via tls=True or ssl_context/tls_hostname)
async with await anyio.connect_tcp("example.com", 443, tls=True) as stream:
    await stream.send(b"...")
    data = await stream.receive()          # raises EndOfStream at EOF

# TCP server
async def handle(stream: anyio.abc.SocketStream) -> None:
    async with stream:
        data = await stream.receive()
        await stream.send(data)

listener = await anyio.create_tcp_listener(local_host="127.0.0.1", local_port=8000)
await listener.serve(handle)               # serve() runs handlers in its own task group

# UDP
async with await anyio.create_udp_socket(local_port=9000) as udp:
    async for packet, (host, port) in udp:
        await udp.sendto(b"pong", host, port)
```

Also: `connect_unix(path)`, `create_unix_listener(path)` (delete the socket file
yourself), `create_connected_udp_socket(host, port)`, the UNIX datagram variants, and
async `getaddrinfo`/`getnameinfo`. `as_connectable(target, tls=...)` normalizes
`(host, port)` / path / string into a connectable for transport-agnostic clients.

## Async file I/O & paths

```python
async with await anyio.open_file("data.txt", "r") as f:
    contents = await f.read()
    async for line in f:
        ...

p = anyio.Path("dir/file.txt")             # async pathlib
await p.write_text("hi")
text = await p.read_text()
async for child in anyio.Path("dir").iterdir():   # glob() is also an async iterator
    if await child.is_file():
        ...
```

`anyio.Path` mirrors `pathlib.Path`, but every I/O method (`read_text`, `is_file`,
`exists`, `iterdir`, `glob`, …) is async. `wrap_file(fileobj)` wraps an existing file.
Temp helpers: `TemporaryFile`, `NamedTemporaryFile`, `SpooledTemporaryFile`,
`TemporaryDirectory` (async context managers), plus `mkstemp`/`mkdtemp`/`gettempdir`.

## Signals

```python
import signal
with anyio.open_signal_receiver(signal.SIGINT, signal.SIGTERM) as signals:
    async for signum in signals:
        tg.cancel_scope.cancel()
        return
```

Only works in the main thread; SIGTERM/signals are limited on Windows.
