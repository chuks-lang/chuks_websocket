# @chuks/websocket

Production-ready WebSocket **client and server** for Chuks — 100% pure Chuks, implementing **RFC 6455** on top of `std/net` TCP primitives. No native builtins required.

- **Typed options + fluent builders** — no untyped `any` config.
- **`@threadSafe` server** — concurrent connections, rooms, and broadcast, with the shared registry guarded by `lock(this)`.
- **Full frame codec** — FIN/opcode/mask, 7/16/64-bit payload lengths, **message fragmentation reassembly**, and interleaved control frames.
- **Correct masking** — client→server frames are masked, server→client are not (per spec); a masked-input read pump unmasks transparently.
- **Ping/pong keepalive + close handshake** with status codes.
- **Verified on both the VM and AOT (native) backends** — see `tests/`.

## Installation

```sh
chuks add @chuks/websocket
```

## Quick start

### Server

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder } from "pkg/@chuks/websocket"

var wss = new WebSocketServer(new WSServerOptionsBuilder().port(8080).build())

wss.onConnection(function(conn: WebSocketConnection): void {
    println("client connected: " + conn.id())

    conn.onMessage(function(msg: string): void {
        conn.send("echo: " + msg)          // reply to this client
        wss.broadcastExcept(conn.id(), msg) // relay to everyone else
    })

    conn.onClose(function(code: int, reason: string): void {
        println("client gone: " + conn.id())
    })
})

wss.listen()   // binds and accepts forever; each client runs on its own task
```

### Client

```chuks
import { connect } from "pkg/@chuks/websocket"

// `connect` returns an already-open client (handshake done) that is pumping frames.
var client = connect("ws://localhost:8080", null)

client.onMessage(function(msg: string): void {
    println("server says: " + msg)
})

client.send("hello server")
```

Register handlers *before* connecting with `createWSClient` instead:

```chuks
import { createWSClient } from "pkg/@chuks/websocket"

var client = createWSClient("ws://localhost:8080", null)
client.onOpen(function(): void { client.send("ready") })
client.onMessage(function(msg: string): void { println(msg) })
client.connect()
```

Both `ws://` and `wss://` (TLS) are supported; a `wss://` URL dials over TLS
transparently and runs the same handshake and frame codec:

```chuks
var client = createWSClient("wss://gateway.discord.gg/?v=10&encoding=json", null)
client.onMessage(function(msg: string): void { println(msg) }) // op 10 Hello, etc.
client.connect()
```

## Configuration

Both server and client take strongly-typed options built with a fluent builder. Every field has a sensible default.

### Server options

```chuks
import { WSServerOptionsBuilder } from "pkg/@chuks/websocket"

var opts = new WSServerOptionsBuilder()
    .host("0.0.0.0")        // bind address (default "0.0.0.0")
    .port(8080)             // listen port (default 8080)
    .path("/ws")            // required handshake path; "" accepts any (default "")
    .maxMessageSize(1048576)// max inbound message bytes (default 1 MiB)
    .build()
```

### Client options

```chuks
import { WSClientOptionsBuilder } from "pkg/@chuks/websocket"

var opts = new WSClientOptionsBuilder()
    .handshakeTimeoutMs(10000) // connect + handshake deadline (default 10000)
    .maxMessageSize(1048576)   // max inbound message bytes (default 1 MiB)
    .autoReconnect(true)       // reconnect on unexpected drop (default false)
    .reconnectDelayMs(1000)    // delay between attempts (default 1000)
    .maxReconnectTries(0)      // 0 = unlimited
    .build()

var client = connect("ws://localhost:8080", opts)
```

With `autoReconnect(true)`, an unexpected drop triggers a retry loop: the client
re-dials every `reconnectDelayMs` and keeps trying until it reconnects or reaches
`maxReconnectTries` (`0` = unlimited). It therefore survives an outage longer than a
single delay window, e.g. a server restart or a network flap. On a successful
reconnect the attempt counter resets and `onOpen` fires again.

## Rooms & broadcast

The server is `@threadSafe`: connection handlers run on independent tasks, and the connection registry is mutated only under an internal lock. Broadcasts snapshot the live set under the lock, then send outside it so a slow socket never blocks the whole server.

```chuks
conn.join("lobby")
conn.leave("lobby")
conn.inRoom("lobby")          // bool

wss.broadcast("hi all")
wss.broadcastExcept(conn.id(), "hi others")
wss.broadcastToRoom("lobby", "hi lobby")
wss.broadcastToRoomExcept("lobby", conn.id(), "...")
wss.broadcastJSON({ "type": "tick", "n": 42 })
wss.connectionCount()         // live connections
```

## Events

Send named, JSON-framed events (`{ "event": "...", "data": ... }`):

```chuks
conn.emit("chat", { "user": "ada", "text": "hello" })
wss.broadcastEvent("presence", { "online": 12 })
```

## API reference

### `WebSocketServer`

| Method | Description |
| --- | --- |
| `new WebSocketServer(opts: WSServerOptions)` | Construct from typed options. |
| `onConnection(fn(conn))` / `onDisconnect(fn(conn))` | Lifecycle handlers. |
| `listen()` | Bind and accept forever (blocks). |
| `stop()` | Stop accepting and close the listener. |
| `connectionCount()` | Live connection count. |
| `broadcast / broadcastJSON / broadcastEvent` | Send to all. |
| `broadcastExcept / broadcastToRoom / broadcastToRoomExcept` | Targeted sends. |
| `closeAll()` | Close every connection with `CLOSE_GOING_AWAY`. |

### `WebSocketConnection`

| Method | Description |
| --- | --- |
| `id()` / `remoteAddr()` / `readyState()` / `isOpen()` | Identity & state. |
| `send(msg)` / `sendJSON(data)` / `emit(name, payload)` | Send text. |
| `ping()` | Send a ping (pong is auto-handled on receive). |
| `close(code, reason)` | Close with a status code. |
| `join / leave / inRoom / rooms` | Room membership. |
| `set(k, v)` / `get(k)` | Per-connection scratch data (userId, session…). |
| `onMessage / onClose / onError / onPong` | Handlers. |

### `WebSocketClient`

| Method | Description |
| --- | --- |
| `connect()` | Connect + handshake, then pump frames. |
| `send / sendJSON / emit` | Send (frames are masked per spec). |
| `ping()` / `close(code, reason)` | Control frames. |
| `readyState()` / `isOpen()` | State. |
| `onOpen / onMessage / onClose / onError` | Handlers. |

### Constants

`STATE_CONNECTING/OPEN/CLOSING/CLOSED`, and the RFC 6455 close codes
`CLOSE_NORMAL` (1000), `CLOSE_GOING_AWAY` (1001), `CLOSE_PROTOCOL_ERROR` (1002),
`CLOSE_TOO_LARGE` (1009), `CLOSE_INTERNAL_ERROR` (1011), … plus
`computeAcceptKey(clientKey)` for custom handshakes.

## Notes & limitations

- **`wss://` (TLS) client** is supported — `connect("wss://host/path")` dials over TLS (via `std/net`'s `TlsConnection`) and runs the same RFC 6455 handshake and frame codec transparently. Verified against live `wss://` servers including the Discord gateway. A **TLS-terminating `wss://` server** is not built in; terminate TLS at a reverse proxy in front of a `ws://` server.
- **Extensions** (`permessage-deflate`) and **subprotocol negotiation** are not implemented.
- One message is delivered per fragmented sequence; binary frames are delivered as their UTF-8 string form via `onMessage`.

## Testing

```sh
chuks run tests/index.test.chuks    # VM
chuks build tests/index.test.chuks  # AOT — run the produced binary
```

Covers the RFC 6455 accept-key vector, the option builders, and a full
loopback handshake → masked send → echo round-trip. All pass on VM and AOT.

## License

MIT
