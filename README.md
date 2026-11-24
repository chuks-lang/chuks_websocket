# @chuks/websocket

Production-ready WebSocket **client and server** for Chuks — 100% pure Chuks, implementing **RFC 6455** on top of `std/net` TCP primitives. No native builtins required.

- **Typed options + fluent builders** — no untyped `any` config.
- **`@threadSafe` server** — concurrent connections, rooms, and broadcast, with the shared registry guarded by `lock(this)`.
- **Full frame codec** — FIN/opcode/mask, 7/16/64-bit payload lengths, **message fragmentation reassembly**, and interleaved control frames.
- **Correct masking** — client→server frames are masked, server→client are not (per spec); a masked-input read pump unmasks transparently.
- **Text and binary messages** — binary arrives as a `Buffer` with every byte intact.
- **Ping/pong keepalive + close handshake** with validated status codes and reasons, and UTF-8 checking of text.
- **Subprotocols, origin checks, `wss://` served directly, and per-connection send queues** with backpressure.
- **Compression** (`permessage-deflate`, RFC 7692), off by default, with a size threshold and every negotiation parameter.
- **Autobahn TestSuite**: server and client pass all 517 cases, compression (sections 12 and 13) included: 512 OK, 2 non-strict, 3 informational, 0 failed.
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
import { createWSClient } from "pkg/@chuks/websocket"

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
    .readTimeoutMs(0)       // each frame must arrive within this; 0 waits for ever (default 0)
    .build()
```

### Client options

```chuks
import { WSClientOptionsBuilder, connect } from "pkg/@chuks/websocket"

var opts = new WSClientOptionsBuilder()
    .handshakeTimeoutMs(10000) // connect + handshake deadline (default 10000)
    .maxMessageSize(1048576)   // max inbound message bytes (default 1 MiB)
    .autoReconnect(true)       // reconnect on unexpected drop (default false)
    .reconnectDelayMs(1000)    // delay between attempts (default 1000)
    .maxReconnectTries(0)      // 0 = unlimited
    .build()

var client = connect("ws://localhost:8080", opts)
```

`readTimeoutMs` closes a connection whose next frame does not arrive in time
(1006, "connection lost"). A client that wants to stay connected through quiet
periods sends something, for example `ping()`, more often than that.

With `autoReconnect(true)`, an unexpected drop triggers a retry loop: the client
re-dials every `reconnectDelayMs` and keeps trying until it reconnects or reaches
`maxReconnectTries` (`0` = unlimited). It therefore survives an outage longer than a
single delay window, e.g. a server restart or a network flap. On a successful
reconnect the attempt counter resets and `onOpen` fires again.

## Rooms & broadcast

The server is `@threadSafe`: connection handlers run on independent tasks, and the connection registry is mutated only under an internal lock. Broadcasts snapshot the live set under the lock, then send outside it so a slow socket never blocks the whole server.

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder } from "pkg/@chuks/websocket"

var wss = new WebSocketServer(new WSServerOptionsBuilder().port(8080).build())
wss.onConnection(function(conn: WebSocketConnection): void {
    conn.join("lobby")
    println(conn.inRoom("lobby"))                       // true
    wss.broadcast("hi all")
    wss.broadcastExcept(conn.id(), "hi others")
    wss.broadcastToRoom("lobby", "hi lobby")
    wss.broadcastToRoomExcept("lobby", conn.id(), "...")
    wss.broadcastJSON({ "type": "tick", "n": 42 })
    println(wss.connectionCount())                      // live connections
    conn.onClose(function(code: int, reason: string): void { conn.leave("lobby") })
})
wss.listen()
```

## Binary messages

`sendBinary` sends a `Buffer` as one binary frame (opcode 2); the peer's
`onBinary` handler receives the same bytes. Text keeps going through `send` and
`onMessage` exactly as before.

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder, connect } from "pkg/@chuks/websocket"
import { Buffer } from "std/buffer"
import { time } from "std/time"

var wss = new WebSocketServer(new WSServerOptionsBuilder().host("127.0.0.1").port(8080).build())
wss.onConnection(function(conn: WebSocketConnection): void {
    conn.onBinary(function(data: Buffer): void {
        conn.sendBinary(data)              // echo the bytes back
        wss.broadcastBinary(data)          // or send them to everyone
    })
})
spawn wss.listen()
time.sleep(200)

var client = connect("ws://127.0.0.1:8080", null)
client.onBinary(function(data: Buffer): void {
    println("got " + string(data.length()) + " bytes, first " + string(data.getByte(0)))
})

var frame: Buffer = Buffer.alloc(6)
frame.writeUInt16BE(513)                   // e.g. a message type
frame.writeUInt32BE(123456)                // and a value
client.sendBinary(frame)                   // the whole buffer goes, whatever its cursor
time.sleep(200)
```

- Binary messages split over continuation frames are reassembled and delivered once.
- `maxMessageSize` applies to binary as to text, and to the whole of a
  fragmented message; a larger one closes the connection with 1009.
- Rooms have binary forms too: `broadcastBinaryExcept`, `broadcastBinaryToRoom`,
  `broadcastBinaryToRoomExcept`. Every broadcast encodes its frame once.
- Without an `onBinary` handler, a binary message goes to `onMessage` as a
  string, as in 1.1, so existing code keeps working. Register `onBinary` to get
  the bytes.

Measured on an Apple M-series Mac, loopback, one server sending 20,000 messages
of 1 KiB to one client (the client decodes base64 back to bytes):

| Payload sent as | Bytes on the wire | Native msg/s | VM msg/s |
| --- | --- | --- | --- |
| binary (`sendBinary`) | 1,024 | 465,000 | 108,000 |
| base64 text (`send`) | 1,368 (+33%) | 99,000 | 52,000 |

## Subprotocols

The client offers protocols in order of preference; the server picks the first
one it also speaks, and both sides can read the choice.

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder, createWSClient } from "pkg/@chuks/websocket"
import { time } from "std/time"

var wss = new WebSocketServer(new WSServerOptionsBuilder().host("127.0.0.1").port(8080).build())
    .protocols(["game.v2", "game.v1"])     // protocols this server speaks
    .requireProtocol(true)                 // refuse (HTTP 400) a client with none of them
wss.onConnection(function(conn: WebSocketConnection): void {
    println("speaking " + conn.protocol())
})
spawn wss.listen()
time.sleep(200)

var client = createWSClient("ws://127.0.0.1:8080", null).protocols(["game.v1"])
client.connect()
println(client.protocol())                 // "game.v1"
```

To decide yourself, pass `selectProtocol(function(offered: []string): string { ... })`;
return one of `offered`, or `""` for none.

## Origin check

Browsers send an `Origin` header. Refuse the ones you do not serve before the
upgrade (HTTP 403):

```chuks
import { WebSocketServer, WSServerOptionsBuilder } from "pkg/@chuks/websocket"
import { strings } from "std/strings"

var wss = new WebSocketServer(new WSServerOptionsBuilder().port(8080).build())
    .allowOrigins(["https://app.example.com"])
// or, for a rule:
//  .checkOrigin(function(origin: string): bool { return strings.endsWith(origin, ".example.com") })
wss.listen()
```

With `allowOrigins`, a request without an `Origin` header (a non-browser
client) is allowed; `checkOrigin` receives `""` for it and decides.

## Serving `wss://`

Pass the certificate and key (PEM strings) and the server listens with TLS
through `std/net`'s `TlsServer`:

```chuks
import { WebSocketServer, WSServerOptionsBuilder, createWSClient } from "pkg/@chuks/websocket"
import { tlsSelfSigned } from "std/net"
import { time } from "std/time"

var certs: any = tlsSelfSigned(["localhost"])   // for a test; use your real cert in production
var wss = new WebSocketServer(new WSServerOptionsBuilder().host("127.0.0.1").port(8443).build())
    .tls({ "cert": certs["cert"], "key": certs["key"] })
spawn wss.listen()
time.sleep(200)

// A client that trusts a private CA passes it the same way TlsConnection takes it.
var client = createWSClient("wss://localhost:8443", null).tls({ "ca": certs["ca"] })
client.connect()
```

## Backpressure

Server sends never block: each connection has a send queue and its own writer
task, so a slow client holds up only its own queue, never a broadcast or other
clients.

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder } from "pkg/@chuks/websocket"
import { Buffer } from "std/buffer"

var wss = new WebSocketServer(new WSServerOptionsBuilder().port(8080).build())
    .highWaterMark(1048576)     // send() returns false above 1 MiB queued (default 1 MiB)
    .maxBufferedBytes(16777216) // close (1008) a client whose queue would pass 16 MiB (default 64 MiB, 0 = no limit)
    .writeTimeoutMs(10000)      // close (1008) a client when one write takes over 10 s (default 0 = none)

wss.onConnection(function(conn: WebSocketConnection): void {
    conn.onMessage(function(msg: string): void {
        var chunk: Buffer = Buffer.alloc(65536)
        var more: bool = conn.sendBinary(chunk)     // false: past the high-water mark
        if (!more) { println("queued " + string(conn.bufferedAmount()) + " bytes; waiting for drain") }
    })
    conn.onDrain(function(): void { println("queue empty, send more") })
})
wss.listen()
```

`bufferedAmount()` is the number of bytes queued and not yet written, like the
browser's `WebSocket.bufferedAmount`. A message is queued even when `send`
returns false; the return value tells you to slow down.

## Compression (permessage-deflate)

Compression is off by default. Turn it on for the server, and for the client
if it is one of yours; browsers offer it on their own.

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder, WSDeflateOptionsBuilder, createWSClient } from "pkg/@chuks/websocket"
import { time } from "std/time"

var wss = new WebSocketServer(new WSServerOptionsBuilder().host("127.0.0.1").port(8080).build())
    .perMessageDeflate(new WSDeflateOptionsBuilder()
        .threshold(1024)                  // messages under 1 KiB go uncompressed (default 1024)
        .level(6)                         // 1 fastest to 9 smallest (default -1, which is 6)
        .build())
wss.onConnection(function(conn: WebSocketConnection): void {
    println("extensions: " + conn.extensions())   // "" when the client did not offer it
    conn.onMessage(function(msg: string): void { conn.send(msg) })
})
spawn wss.listen()
time.sleep(200)

var client = createWSClient("ws://127.0.0.1:8080", null)
    .perMessageDeflate(new WSDeflateOptionsBuilder().build())
client.onMessage(function(msg: string): void { println("echo of " + string(msg.length) + " bytes") })
client.connect()
println(client.extensions())             // "permessage-deflate; client_max_window_bits=15"
client.send("[" + "{\"id\":1,\"kind\":\"danfo\",\"x\":120,\"y\":340}," + "{\"id\":2,\"kind\":\"okada\",\"x\":125,\"y\":338}]")
time.sleep(200)
```

- Text and binary messages at or over the threshold are compressed; the first
  frame carries RSV1, control frames never are. Messages under it go as they are,
  and the two mix freely on one connection.
- A peer that does not offer it (or a server that does not accept it) gets an
  ordinary connection; `extensions()` is then `""`.
- `maxMessageSize` is enforced on the decompressed size while inflating, so a
  small message that would inflate past it (a "zip bomb") closes the connection
  with 1009 before the memory is used. Corrupt compressed data closes with 1007.
- Negotiation follows RFC 7692 section 7: the builder's
  `serverNoContextTakeover`, `clientNoContextTakeover`, `serverMaxWindowBits` and
  `clientMaxWindowBits` (8 to 15) are what the server accepts or the client
  offers. Without `..NoContextTakeover`, each side's compressor keeps its
  history between messages (smaller output, about 32 KiB of memory per
  direction per connection); with it, every message is compressed on its own.
- Each connection compresses its own copy of a broadcast (each has its own
  history), so compression costs CPU per connection.

What it costs, for one tick of an InsideLife-like world (50 vehicles), sent
2,000 times with moving positions, on an Apple M-series Mac:

| Message | Bytes, plain | Bytes, compressed (takeover / none) | Compress per message | Inflate per message |
| --- | --- | --- | --- | --- |
| JSON snapshot | 4,311 | 566 / 885 | 68 / 54 µs | 7 / 9 µs |
| Packed binary snapshot | 608 | 418 / 494 | 32 / 30 µs | 7 / 7 µs |

The times are native; the VM's are within 10% of them, since both run the same
compression engine. End to end over loopback (server to one client, native):

| Message | Plain msg/s | Compressed msg/s |
| --- | --- | --- |
| JSON snapshot | 26,000 | 11,700 |
| Packed binary snapshot | 167,000 | 20,600 |

JSON shrinks to an eighth with context takeover, for about 70 µs of server CPU
per message per connection. Packed binary is already small and gains little:
leave it under the threshold, or leave compression off.

## Close codes and validation

- `close(code, reason)` starts the close handshake: the socket closes when the
  peer answers (or after five seconds), and `onClose` reports the code and
  reason. `closeCode()` and `closeReason()` keep them afterwards.
- A code must be one a peer may receive (`isValidCloseCode`: 1000 to 1003,
  1007 to 1014, 3000 to 4999) or `CLOSE_NO_STATUS`; `close` throws otherwise.
  Reasons are cut to 123 bytes at a character boundary.
- A received close frame with an invalid code fails the connection with 1002;
  a reason that is not UTF-8, with 1007.
- Text messages must be UTF-8 (checked frame by frame, so a character split
  across frames is fine); otherwise the connection closes with 1007. Turn it
  off with `validateUtf8(false)` on the server or client if you trust the peer.
- Protocol errors (reserved bits without an extension, an unmasked client
  frame, a masked server frame, a fragmented or over-long control frame, an
  unknown opcode, a continuation without a start) close with 1002, and a
  message over `maxMessageSize` with 1009.
- The server checks the handshake as RFC 6455 section 4.2.1 requires: `GET`
  over HTTP/1.1, `Host`, `Upgrade: websocket`, `Connection` containing
  `Upgrade` (case-insensitive, comma lists allowed), a 16-byte
  `Sec-WebSocket-Key`, and `Sec-WebSocket-Version: 13` (any other version gets
  426 naming 13). The client checks `Sec-WebSocket-Accept` and refuses a
  protocol or extension it did not offer.

What checking costs, 20,000 ASCII text messages of 1 KiB (same setup as above):

| Text messages | Native msg/s | VM msg/s |
| --- | --- | --- |
| `validateUtf8(true)` (default) | 93,000 | 75,000 |
| `validateUtf8(false)` | 425,000 | 136,000 |

That is about 8 microseconds per 1 KiB message natively and 6 on the VM.

## Events

Send named, JSON-framed events (`{ "event": "...", "data": ... }`):

```chuks
import { WebSocketServer, WebSocketConnection, WSServerOptionsBuilder } from "pkg/@chuks/websocket"

var wss = new WebSocketServer(new WSServerOptionsBuilder().port(8080).build())
wss.onConnection(function(conn: WebSocketConnection): void {
    conn.emit("chat", { "user": "ada", "text": "hello" })
    wss.broadcastEvent("presence", { "online": wss.connectionCount() })
})
wss.listen()
```

## API reference

### `WebSocketServer`

| Method | Description |
| --- | --- |
| `new WebSocketServer(opts: WSServerOptions)` | Construct from typed options. |
| `onConnection(fn(conn))` / `onDisconnect(fn(conn))` | Lifecycle handlers. |
| `listen()` | Bind and accept forever (blocks). |
| `stop()` | Stop accepting and close the listener. |
| `connectionCount()` / `addr()` | Live connection count; the listening address. |
| `broadcast / broadcastJSON / broadcastEvent` | Send to all. |
| `broadcastExcept / broadcastToRoom / broadcastToRoomExcept` | Targeted sends. |
| `broadcastBinary / broadcastBinaryExcept / broadcastBinaryToRoom / broadcastBinaryToRoomExcept` | Binary sends. |
| `protocols(list)` / `selectProtocol(fn)` / `requireProtocol(bool)` | Subprotocol negotiation (chainable, before `listen`). |
| `allowOrigins(list)` / `checkOrigin(fn)` | Origin check, HTTP 403. |
| `tls({ cert, key })` | Serve `wss://`. |
| `highWaterMark(n)` / `maxBufferedBytes(n)` / `writeTimeoutMs(ms)` | Send queue limits. |
| `validateUtf8(bool)` | UTF-8 check of text (default on). |
| `perMessageDeflate(opts)` | Accept compression from clients that offer it. |
| `closeAll()` | Close every connection with `CLOSE_GOING_AWAY`. |

### `WebSocketConnection`

| Method | Description |
| --- | --- |
| `id()` / `remoteAddr()` / `readyState()` / `isOpen()` | Identity & state. |
| `send(msg)` / `sendJSON(data)` / `emit(name, payload)` | Send text; return false above the high-water mark. |
| `sendBinary(buf)` | Send binary; returns as `send`. |
| `bufferedAmount()` | Bytes queued, not yet written. |
| `protocol()` / `extensions()` | The agreed subprotocol and extensions (`""` for none). |
| `ping()` | Send a ping (pong is auto-handled on receive). |
| `close(code, reason)` / `closeCode()` / `closeReason()` | Close handshake; the code and reason once closed. |
| `join / leave / inRoom / rooms` | Room membership. |
| `set(k, v)` / `get(k)` | Per-connection scratch data (userId, session…). |
| `onMessage / onBinary / onClose / onError / onPong / onDrain` | Handlers. |

### `WebSocketClient`

| Method | Description |
| --- | --- |
| `connect()` | Connect + handshake, then pump frames. |
| `send / sendJSON / emit` / `sendBinary(buf)` | Send (frames are masked per spec). |
| `ping()` / `close(code, reason)` / `closeCode()` / `closeReason()` | Control frames; the close code and reason once closed. |
| `readyState()` / `isOpen()` / `protocol()` | State; the subprotocol the server chose. |
| `protocols(list)` / `origin(o)` / `tls(options)` / `validateUtf8(bool)` / `perMessageDeflate(opts)` | Options (chainable, before `connect`). |
| `extensions()` | The extensions the server agreed to (`""` for none). |
| `onOpen / onMessage / onBinary / onClose / onError` | Handlers. |

### Constants

`STATE_CONNECTING/OPEN/CLOSING/CLOSED`, and the RFC 6455 close codes
`CLOSE_NORMAL` (1000), `CLOSE_GOING_AWAY` (1001), `CLOSE_PROTOCOL_ERROR` (1002),
`CLOSE_TOO_LARGE` (1009), `CLOSE_INTERNAL_ERROR` (1011), … plus
`computeAcceptKey(clientKey)` for custom handshakes, `isValidCloseCode(code)`,
and `isValidUtf8(buf)` / `utf8Step(buf, state)` for UTF-8 checks.

## Notes & limitations

- **`wss://` (TLS) client** is supported — `connect("wss://host/path")` dials over TLS (via `std/net`'s `TlsConnection`) and runs the same RFC 6455 handshake and frame codec transparently. Verified against live `wss://` servers including the Discord gateway.
- **Requires Chuks 0.2.0 at commit 873f3a2 or later**, for `std/compress`'s
  streaming `Deflater` and `Inflater`.
- One message is delivered per fragmented sequence. Sending always uses one frame per message.

## Testing

```sh
chuks run tests/index.test.chuks    # VM
chuks build tests/index.test.chuks  # AOT — run the produced binary
```

Covers the RFC 6455 accept-key vector, the option builders, echo, `readTimeoutMs`,
binary (every byte value, the 7/16/64-bit length boundaries, fragments, mixed
with text in order, the size limit, a 500 KB broadcast, client masking),
subprotocols, origins, backpressure, `wss://`, close codes, UTF-8 and handshake
checks, and compression (round trips with and without context takeover, every
window size from 8 to 15, fragmented compressed messages, RSV1 rules, corrupt
data, peers without it, the threshold, and the decompressed-size limit). All
pass on VM and AOT.

## Changelog

### 1.2.0

- Binary messages: `sendBinary`, `onBinary`, and binary broadcasts. Without an
  `onBinary` handler, binary still reaches `onMessage` as before.
- Subprotocols, origin checks, `wss://` served directly, send queues with
  `bufferedAmount`, `onDrain`, and limits for clients that never read.
- Close codes and reasons validated and exposed; UTF-8 checking of text;
  the handshake checks RFC 6455 asks for, on both sides.
- Compression: `permessage-deflate` (RFC 7692) with `WSDeflateOptionsBuilder`,
  `perMessageDeflate()` on server and client, and `extensions()`; the size
  limit applies after inflating. Needs Chuks 0.2.0 at 873f3a2 or later.
- Passes the Autobahn TestSuite, all 517 cases with compression, as server and
  as client.
- Fixed: `readTimeoutMs` was accepted but never applied; each frame now has to
  arrive within it.
- Fixed: close frames went out with code 0 and an empty reason (the payload
  was copied from its end); peers now get the code and reason you pass.
- Fixed: a `Buffer` filled with `write*()` calls went out as zeros.
- Fixed: pongs and close replies were written without the send lock and could
  interleave with a message being sent.
- Fixed: `maxMessageSize` now covers the whole of a fragmented message, and an
  over-size message closes with 1009 instead of dropping the connection.
- Changed: `close()` now runs the close handshake instead of dropping the
  socket at once, and `onClose` fires for a close you start.

## License

MIT
