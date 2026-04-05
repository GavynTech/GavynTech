# Type 09 — Rate Limiting

## What It Does

Limits how many messages a client can send per time window.
Clients that exceed the limit are warned, then disconnected.

Prevents:
- Spam attacks
- Accidental infinite loops in client code
- Resource exhaustion (CPU from processing, DB write floods)

---

## When to Use It

- Any public-facing WebSocket server
- Chat apps (prevent message floods)
- APIs that trigger expensive operations (DB writes, AI calls)
- Game servers (prevent exploit/spam bots)

---

## Infrastructure Diagram

```
[Client]  → message 1  → allowed  (count: 1/10)
[Client]  → message 2  → allowed  (count: 2/10)
...
[Client]  → message 10 → allowed  (count: 10/10)
[Client]  → message 11 → BLOCKED  → warning sent
[Client]  → message 12 → BLOCKED  → warning sent
[Client]  → message 13 → BLOCKED  → TERMINATED

[window resets every N seconds → count resets to 0]
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')

const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Rate Limit Config
const RATE_LIMIT = {
  windowMs:     10_000,  // 10 second window
  maxMessages:  10,      // max messages per window
  warnAt:       8,       // warn client at this count
  banAfter:     3,       // terminate after this many violations
}

wss.on('connection', (socket) => {

  // PATTERN: Per-Client Rate State
  // Track rate limit data directly on the socket object.
  socket.rateState = {
    count:      0,        // messages sent in current window
    violations: 0,        // number of times limit was exceeded
    windowStart: Date.now()
  }

  socket.on('message', (data) => {

    // PATTERN: Rate Limit Check
    // Run the rate check BEFORE any message processing.
    // If the client is over limit, skip all business logic.
    if (!checkRateLimit(socket)) return

    // --- Normal message processing below ---
    let msg
    try { msg = JSON.parse(data.toString()) }
    catch { return send(socket, { type: 'error', text: 'Invalid JSON' }) }

    send(socket, { type: 'echo', text: msg.text })
  })

  socket.on('close', () => console.log('Client disconnected'))
  socket.on('error', (err) => console.error(err.message))
})

function checkRateLimit(socket) {
  const state = socket.rateState
  const now = Date.now()

  // PATTERN: Sliding Window Reset
  // If the current time is past the window end, reset the counter.
  // This is a "fixed window" algorithm — simple but effective for most cases.
  if (now - state.windowStart >= RATE_LIMIT.windowMs) {
    state.count = 0
    state.windowStart = now
  }

  state.count++

  // PATTERN: Approaching Limit Warning
  // Warn the client before they actually hit the limit.
  // Gives well-behaved clients a chance to slow down.
  if (state.count === RATE_LIMIT.warnAt) {
    send(socket, {
      type: 'rate_warning',
      text: `Slow down — you've sent ${state.count}/${RATE_LIMIT.maxMessages} messages this window`,
      remaining: RATE_LIMIT.maxMessages - state.count
    })
  }

  // PATTERN: Over Limit Violation
  if (state.count > RATE_LIMIT.maxMessages) {
    state.violations++

    send(socket, {
      type: 'rate_limited',
      text: `Rate limit exceeded. Violation ${state.violations}/${RATE_LIMIT.banAfter}`,
      retryAfter: Math.ceil((RATE_LIMIT.windowMs - (now - state.windowStart)) / 1000)
    })

    // PATTERN: Progressive Enforcement
    // First violations = warning. After N violations = disconnect.
    // This avoids punishing clients for one burst while stopping persistent abusers.
    if (state.violations >= RATE_LIMIT.banAfter) {
      console.log('Client banned for rate limit abuse — terminating')
      socket.close(1008, 'Rate limit violated')
    }

    return false // message is blocked
  }

  return true // message is allowed
}

function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

wss.on('listening', () => {
  console.log('Rate-limited server on ws://localhost:8080')
  console.log(`Limit: ${RATE_LIMIT.maxMessages} msgs / ${RATE_LIMIT.windowMs / 1000}s`)
})
```

---

## Token Bucket Algorithm (more advanced)

The fixed window above is simple. The **token bucket** is better for bursty traffic:

```js
// PATTERN: Token Bucket
// Each client has a bucket of `capacity` tokens.
// Each message costs 1 token.
// Tokens refill at `refillRate` tokens per second.
// Bursts are allowed up to `capacity` but sustained rate is limited.

function createTokenBucket({ capacity = 10, refillRate = 2 } = {}) {
  return {
    tokens: capacity,
    capacity,
    refillRate,       // tokens per second
    lastRefill: Date.now()
  }
}

function consumeToken(bucket) {
  const now = Date.now()
  const elapsed = (now - bucket.lastRefill) / 1000 // seconds

  // Refill tokens based on elapsed time
  bucket.tokens = Math.min(
    bucket.capacity,
    bucket.tokens + elapsed * bucket.refillRate
  )
  bucket.lastRefill = now

  if (bucket.tokens >= 1) {
    bucket.tokens -= 1
    return true  // allowed
  }
  return false   // blocked
}

// Usage in connection handler:
wss.on('connection', (socket) => {
  socket.bucket = createTokenBucket({ capacity: 10, refillRate: 2 })

  socket.on('message', (data) => {
    if (!consumeToken(socket.bucket)) {
      return send(socket, { type: 'rate_limited', text: 'Too fast' })
    }
    // ... process message
  })
})
```

---

## Algorithm Comparison

```
Fixed Window:
  + Simple to implement
  + Predictable resets
  - Allows burst at window boundary (10 at t=9.9s + 10 at t=10.1s = 20 in 0.2s)

Sliding Window Log:
  + Most accurate — tracks exact timestamps of all messages
  - Memory: stores N timestamps per client (can be large)
  + No boundary burst problem

Token Bucket:
  + Allows natural bursts up to capacity
  + Sustained rate is always enforced
  + Memory efficient (just 4 numbers per client)
  ± Slightly more complex than fixed window
  Best for: most real-world apps
```

---

## Message Size Limiting

Rate limiting message *count* isn't enough — a client could send 1 huge message.
Always limit message *size* too:

```js
// In the ws server options:
const wss = new WebSocketServer({
  port: 8080,
  maxPayload: 64 * 1024  // 64 KB max message size
                          // Connections sending larger messages are terminated
})

// Or check manually:
socket.on('message', (data) => {
  if (data.length > 64 * 1024) {
    socket.close(1009, 'Message too large')
    return
  }
  // ... process
})
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Rate Limit Config** | top-level constants | Single place to tune all limits |
| **Per-Client Rate State** | `socket.rateState` | Track count/violations per socket |
| **Rate Limit Check** | before message processing | Guard — run first, skip logic if blocked |
| **Sliding Window Reset** | `now - windowStart >= windowMs` | Reset counter when window expires |
| **Approaching Limit Warning** | `count === warnAt` | Warn before hard limit hits |
| **Over Limit Violation** | `count > maxMessages` | Track violations, send retry info |
| **Progressive Enforcement** | violations counter | Warn → warn → disconnect progression |
| **Token Bucket** | `consumeToken(bucket)` | Burst-friendly rate limiting |
| **Message Size Limit** | `maxPayload` option | Prevent single huge message attacks |

---

## What to Build Next

- **Per-route limits** — different limits for different actions (chat vs file upload)
- **IP-based limits** — limit by IP address, not just socket (catches reconnect abuse)
- **Redis-backed limits** — share rate limit state across multiple server instances
- **Move to `10-redis.md`** — scale WebSockets across multiple machines
