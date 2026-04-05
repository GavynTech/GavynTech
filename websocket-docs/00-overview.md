# WebSocket Documentation — Complete Reference

## What This Doc Series Covers

This is a complete, learner-focused WebSocket reference. Each file covers a specific
type of WebSocket usage with full code, annotated infrastructure, and named patterns
you can identify and reuse.

---

## Files in This Series

| File | Type | Difficulty |
|---|---|---|
| `01-echo.md` | Echo Server | Beginner |
| `02-broadcast.md` | Broadcast / Chat Room | Beginner |
| `03-pubsub.md` | Pub/Sub Channels | Intermediate |
| `04-rooms.md` | Room-Based / Namespaced | Intermediate |
| `05-auth.md` | Authenticated WebSockets | Intermediate |
| `06-binary.md` | Binary Data / File Streaming | Intermediate |
| `07-heartbeat.md` | Ping/Pong Heartbeat | Intermediate |
| `08-reconnect.md` | Auto-Reconnecting Client | Intermediate |
| `09-ratelimit.md` | Rate Limiting | Advanced |
| `10-redis.md` | Scaling with Redis Pub/Sub | Advanced |
| `11-patterns.md` | Pattern Reference (all patterns) | Reference |

---

## The WebSocket Protocol — Infrastructure

### 1. The Handshake (HTTP → WebSocket Upgrade)

```
Client → Server  (HTTP Request)
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket          ← asking to upgrade
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNh...  ← random base64 key
Sec-WebSocket-Version: 13

Server → Client  (HTTP 101 Switching Protocols)
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPL...  ← server hashes the key
```

After this handshake, the TCP connection is no longer HTTP.
It is now a raw WebSocket connection — persistent and bidirectional.

---

### 2. The Frame Structure (how data travels)

Every message sent over a WebSocket is a **frame**:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
```

**Key fields:**
- `FIN` — is this the final fragment of a message?
- `opcode` — type of frame: `0x1`=text, `0x2`=binary, `0x8`=close, `0x9`=ping, `0xA`=pong
- `MASK` — clients MUST mask frames to the server (server frames are unmasked)
- `Payload len` — length of the message

You almost never deal with raw frames — libraries handle this. But knowing the
structure helps you understand why WebSockets are efficient (small overhead per frame).

---

### 3. Connection Lifecycle

```
CLIENT                              SERVER
  |                                   |
  |------- HTTP Upgrade Request ----->|
  |<------ 101 Switching Protocols ---|
  |                                   |
  |====== WebSocket Connection ======|
  |                                   |
  |------- text frame ("hello") ----->|   onmessage
  |<------ text frame ("world") ------|   ws.send()
  |                                   |
  |------- ping frame --------------->|   (keepalive)
  |<------ pong frame -----------------|
  |                                   |
  |------- close frame (1000) ------->|   ws.close()
  |<------ close frame (1000) --------|   server confirms
  |                                   |
  |======= TCP Connection Closed =====|
```

---

### 4. Close Codes (what 1000, 1006, etc. mean)

| Code | Meaning |
|---|---|
| 1000 | Normal closure — everything fine |
| 1001 | Going away — server shutting down or browser navigated away |
| 1002 | Protocol error |
| 1003 | Unsupported data type received |
| 1006 | Abnormal closure — connection dropped (no close frame sent) |
| 1008 | Policy violation (e.g., failed auth) |
| 1011 | Server encountered an unexpected error |
| 4000–4999 | **Available for application use** — define your own |

---

### 5. ws:// vs wss://

| Protocol | Transport | Use when |
|---|---|---|
| `ws://` | Plain TCP | Local development only |
| `wss://` | TLS (encrypted) | Always in production |

`wss://` is WebSocket over TLS — same relationship as `http://` vs `https://`.

---

### 6. Infrastructure Stack

```
Browser / Client App
        ↓  wss://
  [TLS Termination] (nginx, Caddy, AWS ALB)
        ↓  ws://
  [WebSocket Server] (Node.js + ws, Socket.IO, etc.)
        ↓
  [App Logic / State]
        ↓
  [Database / Cache / Message Broker]
  (Postgres, Redis, RabbitMQ, etc.)
```

For a single server, you can skip the TLS terminator in dev.
For production at scale, you need a broker (Redis, RabbitMQ) so multiple
server instances can communicate across machines (see `10-redis.md`).

---

### 7. Core Events (all platforms)

| Event | When it fires |
|---|---|
| `open` / `connection` | Connection established |
| `message` | Data received |
| `error` | An error occurred |
| `close` | Connection closed |

These four events are the foundation of every WebSocket pattern.

---

### 8. Libraries Used in This Series

**Server (Node.js):**
```bash
npm install ws          # lightweight, standard
npm install socket.io   # higher-level, rooms/namespaces built-in
npm install ioredis     # Redis client (for scaling examples)
```

**Client (Browser):**
- Native `WebSocket` API — built into every modern browser, no install needed

---

## How to Read These Docs

Each file follows this structure:

1. **What it does** — plain English purpose
2. **When to use it** — real-world scenarios
3. **Infrastructure diagram** — how pieces connect
4. **Server code** — fully annotated
5. **Client code** — fully annotated
6. **Patterns identified** — named, reusable patterns with descriptions
7. **What to build next** — progression suggestions

Start with `01-echo.md`. Every other type builds on the concepts introduced there.
