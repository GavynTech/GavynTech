# Type 07 — Ping/Pong Heartbeat

## What It Does

Sends periodic ping frames to detect dead connections — clients that dropped
without sending a close frame (network failure, laptop closed, mobile app killed).

Without a heartbeat, dead sockets accumulate in `wss.clients` forever,
consuming memory and causing phantom "connected users" counts.

---

## When to Use It

- Always — in any production WebSocket server
- Mobile apps (frequent background kills, network switches)
- Long-lived connections (dashboards, monitoring tools)
- Any server that tracks "online users"

---

## How Ping/Pong Works at the Protocol Level

```
Server → Client:  [PING frame]   (opcode 0x9)
Client → Server:  [PONG frame]   (opcode 0xA)  ← browser does this automatically

If the server sends a ping and gets no pong within the timeout window,
the connection is assumed dead and terminated.

Browsers handle pong automatically — you do NOT need to write pong logic
on the client side. The browser's WebSocket implementation responds to pings.

For Node.js clients (ws library), you must handle it manually:
  socket.on('ping', () => socket.pong())
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')

const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Heartbeat Constants
const PING_INTERVAL = 30_000  // ping every 30 seconds
const PONG_TIMEOUT  = 10_000  // expect pong within 10 seconds of ping

wss.on('connection', (socket) => {

  // PATTERN: Liveness Flag
  // A boolean on each socket that tracks whether it responded to the last ping.
  // Start as true — assume alive when freshly connected.
  socket.isAlive = true

  // PATTERN: Pong Handler
  // When a pong arrives, mark the socket as alive.
  // The browser sends this automatically in response to our ping.
  socket.on('pong', () => {
    socket.isAlive = true
    console.log('Pong received — connection alive')
  })

  socket.on('message', (data) => {
    // PATTERN: Application-Level Ping
    // Some clients send a JSON { action: 'ping' } instead of using WS protocol pings.
    // Handle both to support web clients that can't send raw ping frames.
    let msg
    try { msg = JSON.parse(data.toString()) }
    catch { return }

    if (msg.action === 'ping') {
      socket.send(JSON.stringify({ type: 'pong', timestamp: Date.now() }))
    }
  })

  socket.on('close', () => console.log('Client disconnected'))
  socket.on('error', (err) => console.error(err.message))
})

// PATTERN: Heartbeat Interval
// A single interval on the SERVER checks ALL connections periodically.
// This is more efficient than one interval per socket.
const heartbeatInterval = setInterval(() => {

  wss.clients.forEach((socket) => {

    // PATTERN: Dead Connection Detection
    // If isAlive is still false from the previous ping cycle,
    // the client never responded → terminate the connection.
    if (!socket.isAlive) {
      console.log('Dead socket detected — terminating')
      socket.terminate() // force-close the TCP connection (no graceful close frame)
      return
    }

    // PATTERN: Reset + Ping
    // Set isAlive to false BEFORE sending the ping.
    // If a pong arrives, the pong handler sets it back to true.
    // If no pong arrives before the next interval, it stays false → terminated.
    socket.isAlive = false
    socket.ping() // sends a WebSocket protocol PING frame
  })

}, PING_INTERVAL)

// PATTERN: Interval Cleanup
// Clear the interval when the server closes to prevent process hang.
wss.on('close', () => {
  clearInterval(heartbeatInterval)
})

wss.on('listening', () => {
  console.log(`Heartbeat server on ws://localhost:8080`)
  console.log(`Ping every ${PING_INTERVAL / 1000}s, timeout ${PONG_TIMEOUT / 1000}s`)
})
```

---

## Client Code (Browser)

```html
<script>
  const ws = new WebSocket('ws://localhost:8080')

  // Browsers handle pong automatically — NO pong code needed here.
  // The browser's WebSocket engine responds to PING frames on its own.

  ws.onopen = () => console.log('Connected')
  ws.onclose = () => console.log('Disconnected')
  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data)
    if (msg.type === 'pong') {
      console.log(`Server pong at ${msg.timestamp}`)
    }
  }

  // PATTERN: Client-Side Application Ping
  // Optionally send your own pings to the server to detect connectivity.
  // Useful when you want to measure round-trip latency.
  let pingInterval = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      const sent = Date.now()
      ws.send(JSON.stringify({ action: 'ping' }))

      // Store sent time to calculate latency when pong arrives
      ws._pingSent = sent
    }
  }, 30000)

  ws.addEventListener('message', (event) => {
    const msg = JSON.parse(event.data)
    if (msg.type === 'pong' && ws._pingSent) {
      const latency = Date.now() - ws._pingSent
      console.log(`Round-trip latency: ${latency}ms`)
    }
  })

  ws.onclose = () => {
    clearInterval(pingInterval)
  }
</script>
```

---

## Node.js Client (must handle pong manually)

```js
// test-client.js
const WebSocket = require('ws')

const ws = new WebSocket('ws://localhost:8080')

// PATTERN: Node Client Pong Handler
// Unlike browsers, Node.js ws clients do NOT auto-respond to pings.
// You must manually call socket.pong() when a ping arrives.
ws.on('ping', () => {
  console.log('Ping received — sending pong')
  ws.pong()
})

ws.on('open', () => console.log('Connected'))
ws.on('close', (code) => console.log(`Closed: ${code}`))
```

---

## Heartbeat Timeline Diagram

```
t=0s    Server sends PING to all clients
t=0s    Client A responds PONG → isAlive = true
         Client B (dead) — no response

t=30s   Server sends PING again
        Client A.isAlive was true → reset to false → send PING
        Client B.isAlive was false → TERMINATE

t=30s   Client A responds PONG → isAlive = true

t=60s   repeat...
```

---

## `socket.ping()` vs `socket.terminate()` vs `socket.close()`

| Method | What it does | Use when |
|---|---|---|
| `socket.ping()` | Send a ping frame | Check if client is alive |
| `socket.close(code, reason)` | Graceful close (sends close frame) | Intentional disconnect |
| `socket.terminate()` | Force-close TCP connection | Dead connection — no response |

Use `terminate()` for dead sockets — `close()` waits for a response that will never come.

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Liveness Flag** | `socket.isAlive = true` | Boolean per socket to track pong receipt |
| **Pong Handler** | `socket.on('pong', ...)` | Reset liveness flag on pong |
| **Heartbeat Interval** | `setInterval` on server | Single interval checks all sockets |
| **Dead Connection Detection** | `if (!isAlive) terminate()` | Kill sockets that missed a pong |
| **Reset + Ping** | `isAlive = false; socket.ping()` | Reset flag then send ping — two-step |
| **Interval Cleanup** | `wss.on('close', clearInterval)` | Prevent process hang on server close |
| **Application-Level Ping** | `{ action: 'ping' }` JSON | App-level keepalive (works in browsers) |
| **Latency Measurement** | `Date.now()` diff | Measure round-trip time with pings |
| **Node Client Pong Handler** | `ws.on('ping', () => ws.pong())` | Required for Node.js clients |

---

## What to Build Next

- **Stagger pings** — ping clients in batches to avoid spikes in CPU
- **Latency dashboard** — track per-client latency over time
- **Reconnection on dead detection** — notify the client before terminating
- **Move to `08-reconnect.md`** — handle disconnects gracefully on the client
