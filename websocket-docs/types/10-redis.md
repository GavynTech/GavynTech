# Type 10 — Scaling with Redis Pub/Sub

## What It Does

When you run multiple WebSocket server instances (for load balancing or high
availability), clients connected to different servers can't normally communicate.

Redis Pub/Sub acts as a **message bus** between servers — a message published on
Server A is delivered to all clients on Server B and C as well.

---

## When to Use It

- Horizontally scaled deployments (multiple server processes/containers)
- Kubernetes pods behind a load balancer
- High availability setups where one server can die without dropping service
- When a single Node.js process can't handle your connection volume

---

## Infrastructure Diagram

```
[Client A] ─── [Server 1]
[Client B] ─── [Server 1]        ← both on server 1

[Client C] ─── [Server 2]
[Client D] ─── [Server 2]        ← both on server 2

Client A sends "hello"
  → Server 1 receives it
  → Server 1 publishes to Redis channel "broadcast"
  → Redis delivers to all subscribers
  → Server 1 receives → sends to Client A, Client B
  → Server 2 receives → sends to Client C, Client D
  ✓ All 4 clients get "hello"

[Load Balancer]  (nginx, AWS ALB — sticky sessions optional)
      ↓                  ↓
  [Server 1]         [Server 2]
      ↓                  ↓
         [Redis]  ← shared message bus
```

---

## Install

```bash
npm install ws ioredis
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')
const Redis = require('ioredis')

const PORT = process.env.PORT || 8080
const SERVER_ID = `server_${PORT}` // unique ID per instance

const wss = new WebSocketServer({ port: PORT })

// PATTERN: Two Redis Clients
// Redis Pub/Sub requires SEPARATE clients for pub and sub.
// A client that has called SUBSCRIBE cannot issue other commands.
// ioredis handles reconnection automatically.
const redisPub = new Redis({ host: 'localhost', port: 6379 })
const redisSub = new Redis({ host: 'localhost', port: 6379 })

const CHANNEL = 'ws:broadcast' // Redis channel name

// PATTERN: Subscribe to Redis Channel
// This server listens on the shared Redis channel.
// All server instances subscribe to the same channel.
redisSub.subscribe(CHANNEL, (err) => {
  if (err) {
    console.error('Redis subscribe error:', err)
    process.exit(1)
  }
  console.log(`[${SERVER_ID}] Subscribed to Redis channel: ${CHANNEL}`)
})

// PATTERN: Redis Message → Local Broadcast
// When a message arrives from Redis, deliver it to all LOCAL clients.
// This is how messages cross server boundaries.
redisSub.on('message', (channel, rawMessage) => {
  if (channel !== CHANNEL) return

  let msg
  try { msg = JSON.parse(rawMessage) }
  catch { return }

  // PATTERN: Skip Own Messages (Optional)
  // If the server published this message itself, it may already have
  // delivered it to local clients. Skip to avoid duplicates.
  // Comment this out if you want servers to receive their own publishes.
  if (msg._serverId === SERVER_ID) return

  console.log(`[${SERVER_ID}] From Redis: ${rawMessage}`)
  broadcastLocal(msg)
})

wss.on('connection', (socket) => {
  console.log(`[${SERVER_ID}] Client connected. Local clients: ${wss.clients.size}`)

  socket.username = `User_${Math.random().toString(36).slice(2, 6)}`

  // Announce join — this goes to Redis so ALL servers distribute it
  publishToRedis({
    type: 'system',
    text: `${socket.username} joined (via ${SERVER_ID})`,
    _serverId: SERVER_ID
  })

  send(socket, { type: 'connected', username: socket.username, server: SERVER_ID })

  socket.on('message', (data) => {
    let msg
    try { msg = JSON.parse(data.toString()) }
    catch { return send(socket, { type: 'error', text: 'Invalid JSON' }) }

    // PATTERN: Publish to Redis (not direct broadcast)
    // Instead of broadcasting locally, publish to Redis.
    // Redis then fans it out to ALL server instances.
    publishToRedis({
      type: 'message',
      from: socket.username,
      text: msg.text,
      timestamp: new Date().toISOString(),
      _serverId: SERVER_ID
    })
  })

  socket.on('close', () => {
    publishToRedis({
      type: 'system',
      text: `${socket.username} left`,
      _serverId: SERVER_ID
    })
  })

  socket.on('error', (err) => console.error(err.message))
})

// PATTERN: Publish to Redis
// Send a message to the Redis channel.
// All servers (including this one) will receive it via redisSub.
function publishToRedis(data) {
  redisPub.publish(CHANNEL, JSON.stringify(data))
    .catch((err) => console.error('Redis publish error:', err))
}

// PATTERN: Local Broadcast
// Deliver a message to all clients connected to THIS server instance.
function broadcastLocal(data) {
  const payload = JSON.stringify(data)
  wss.clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(payload)
    }
  })
}

function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

// PATTERN: Graceful Shutdown
// On SIGTERM (e.g., Kubernetes pod shutdown), cleanly close all connections
// and disconnect from Redis before the process exits.
process.on('SIGTERM', () => {
  console.log(`[${SERVER_ID}] Shutting down...`)
  wss.clients.forEach((client) => client.close(1001, 'Server shutting down'))
  wss.close(() => {
    redisPub.quit()
    redisSub.quit()
    process.exit(0)
  })
})

wss.on('listening', () => {
  console.log(`[${SERVER_ID}] WebSocket server on ws://localhost:${PORT}`)
})
```

---

## Running Multiple Instances (local test)

```bash
# Terminal 1 — Start Redis
redis-server

# Terminal 2 — Server instance 1
PORT=8080 node server.js

# Terminal 3 — Server instance 2
PORT=8081 node server.js

# Open browser tab → connect to ws://localhost:8080
# Open browser tab → connect to ws://localhost:8081
# Message from one tab appears in the other ✓
```

---

## Redis Pub/Sub vs Redis Streams

```
Redis Pub/Sub (used here):
  + Simple, low latency
  - Fire and forget — messages lost if no subscribers at publish time
  - No message history
  Best for: live notifications, chat, real-time updates

Redis Streams:
  + Persistent — messages stored in a log
  + Consumers can replay from any point
  + Consumer groups for parallel processing
  - More complex setup
  Best for: event sourcing, audit logs, guaranteed delivery
```

---

## Sticky Sessions (Load Balancer Config)

```
By default, a load balancer distributes connections randomly.
This means a client may connect to Server 1 for the HTTP upgrade
but get routed to Server 2 for subsequent requests — breaking WebSocket.

Solution: Enable "sticky sessions" (session affinity) so a client
always goes to the same server.

nginx example:
  upstream websocket_servers {
    ip_hash;           ← same client IP always goes to same server
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
  }

AWS ALB: enable "stickiness" in target group settings.
Kubernetes: use a Service with sessionAffinity: ClientIP
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Two Redis Clients** | `redisPub` / `redisSub` | Separate clients for publish and subscribe |
| **Subscribe to Redis Channel** | `redisSub.subscribe(...)` | All servers listen to shared channel |
| **Redis Message → Local Broadcast** | `redisSub.on('message', ...)` | Fan out Redis messages to local clients |
| **Skip Own Messages** | `_serverId === SERVER_ID` | Avoid double-delivery from publisher |
| **Publish to Redis** | `redisPub.publish(...)` | Send to shared channel (not local broadcast) |
| **Local Broadcast** | `wss.clients.forEach(send)` | Deliver to clients on THIS server only |
| **Server Identity** | `SERVER_ID = server_${PORT}` | Unique ID per instance for debug/routing |
| **Graceful Shutdown** | `SIGTERM` handler | Clean disconnect before process exits |

---

## What to Build Next

- **Redis Streams** — persist messages for replay / offline clients
- **Room-based Redis** — publish to `ws:room:roomId` channel per room
- **Presence tracking** — use Redis Sets to track who is online across all servers
- **Move to `11-patterns.md`** — full reference of every pattern identified in this series
