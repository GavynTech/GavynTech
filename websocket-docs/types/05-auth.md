# Type 05 — Authenticated WebSockets (JWT)

## What It Does

Validates client identity during the WebSocket handshake using a JWT token.
Unauthenticated clients are rejected before any messages are exchanged.

---

## When to Use It

- Any app where users have accounts
- Private APIs — only paying or registered users
- Per-user data streams (show MY notifications, not everyone's)
- Admin dashboards

---

## Infrastructure Diagram

```
[Client]
  1. POST /auth/login  →  [HTTP Auth Server]  →  JWT token
  2. ws://server?token=<JWT>  →  [WebSocket Server]
       ↓ verify token
       ✓ valid   → connection accepted, socket.user = decoded payload
       ✗ invalid → close(1008, "Unauthorized")
```

---

## Install

```bash
npm install ws jsonwebtoken
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')
const jwt = require('jsonwebtoken')
const { parse: parseUrl } = require('url')

const JWT_SECRET = process.env.JWT_SECRET || 'dev-secret-change-in-prod'

const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Handshake Authentication
// The `verifyClient` option runs BEFORE the connection is accepted.
// Return false to reject the connection at the TCP level — no socket is created.
// This is the most efficient place to authenticate: no wasted socket resources.
const wssAuth = new WebSocketServer({
  port: 8081,
  verifyClient: ({ req }, callback) => {

    // PATTERN: Token from Query String
    // The client passes the JWT as ?token=... in the WebSocket URL.
    // Alternative: parse from the Cookie header (more secure for browsers).
    const { query } = parseUrl(req.url, true)
    const token = query.token

    if (!token) {
      // callback(result, httpCode, httpMessage)
      return callback(false, 401, 'Token required')
    }

    jwt.verify(token, JWT_SECRET, (err, decoded) => {
      if (err) {
        return callback(false, 401, 'Invalid token')
      }

      // PATTERN: Attach Decoded User to Request
      // Store decoded payload on req so the connection handler can access it.
      req.user = decoded
      callback(true) // allow connection
    })
  }
})

wssAuth.on('connection', (socket, request) => {

  // PATTERN: Socket User Assignment
  // User identity verified — attach to socket for use in message handlers.
  socket.user = request.user
  console.log(`Authenticated: ${socket.user.username} (${socket.user.role})`)

  send(socket, {
    type: 'authenticated',
    user: {
      id: socket.user.sub,
      username: socket.user.username,
      role: socket.user.role
    }
  })

  socket.on('message', (data) => {
    let msg
    try { msg = JSON.parse(data.toString()) }
    catch { return send(socket, { type: 'error', text: 'Invalid JSON' }) }

    // PATTERN: Role-Based Authorization
    // Check the user's role before allowing privileged actions.
    if (msg.action === 'admin_broadcast') {
      if (socket.user.role !== 'admin') {
        return send(socket, { type: 'error', text: 'Admin only' })
      }
      broadcastAll(wssAuth, { type: 'admin', text: msg.text, from: socket.user.username })
      return
    }

    // Regular authenticated message
    send(socket, {
      type: 'echo',
      text: msg.text,
      receivedBy: socket.user.username
    })
  })

  socket.on('close', () => {
    console.log(`Disconnected: ${socket.user.username}`)
  })

  socket.on('error', (err) => console.error(err.message))
})

function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

function broadcastAll(wss, data) {
  const payload = JSON.stringify(data)
  wss.clients.forEach((s) => {
    if (s.readyState === WebSocket.OPEN) s.send(payload)
  })
}

// ------------------------------------------------------------------
// Separate HTTP endpoint to issue tokens (in a real app this would
// be your existing auth server — Express, Fastify, etc.)
// ------------------------------------------------------------------
const http = require('http')

const authServer = http.createServer((req, res) => {
  // DEMO ONLY: accept any username, issue a token
  // In production: validate credentials against your database
  if (req.method === 'POST' && req.url === '/login') {
    let body = ''
    req.on('data', (chunk) => (body += chunk))
    req.on('end', () => {
      const { username, password } = JSON.parse(body)

      // PATTERN: Token Issuance
      // Sign a JWT with the user's identity and role.
      // `sub` (subject) is the user's unique ID.
      // `exp` controls token lifetime.
      const token = jwt.sign(
        { sub: 'user_123', username, role: username === 'admin' ? 'admin' : 'user' },
        JWT_SECRET,
        { expiresIn: '1h' }
      )

      res.writeHead(200, { 'Content-Type': 'application/json' })
      res.end(JSON.stringify({ token }))
    })
    return
  }
  res.writeHead(404)
  res.end()
})

authServer.listen(3000, () => console.log('Auth server on http://localhost:3000'))
console.log('Auth WebSocket server on ws://localhost:8081')
```

---

## Client Code (Browser)

```js
// Step 1: Login and get a token
async function login(username, password) {
  const res = await fetch('http://localhost:3000/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
  })
  const { token } = await res.json()
  return token
}

// Step 2: Connect to WebSocket with token
async function connect() {
  const token = await login('alice', 'password123')

  // PATTERN: Token in URL
  // Pass JWT as a query parameter.
  // NOTE: For higher security in browsers, use cookies instead:
  //   document.cookie = `token=${token}; Secure; HttpOnly`
  //   Then on server, parse req.headers.cookie
  const ws = new WebSocket(`ws://localhost:8081?token=${token}`)

  ws.onopen = () => console.log('Connected and authenticated')

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data)
    console.log(msg)
  }

  ws.onclose = (event) => {
    // PATTERN: Auth Failure Detection
    // Close code 1008 = policy violation (our auth rejection code)
    if (event.code === 1008 || event.code === 401) {
      console.log('Authentication failed — redirecting to login')
      // window.location.href = '/login'
    }
  }

  return ws
}

connect()
```

---

## Token Refresh Pattern

```
JWT tokens expire. When they do, the WebSocket connection becomes "stale"
(the user is still connected but their token is no longer valid).

Two strategies:

1. Short-lived connections
   - Token expires → server closes socket → client re-authenticates
   - Simple but causes reconnect churn

2. Token refresh over WebSocket
   - Client sends { action: 'refresh_token', refreshToken: '...' }
   - Server validates refresh token, issues new JWT
   - Server updates socket.user with new claims
   - Connection stays open

Strategy 2 example (server side):
```
```js
case 'refresh_token':
  jwt.verify(msg.refreshToken, REFRESH_SECRET, (err, decoded) => {
    if (err) return send(socket, { type: 'error', text: 'Invalid refresh token' })
    const newToken = jwt.sign({ sub: decoded.sub, username: decoded.username, role: decoded.role },
      JWT_SECRET, { expiresIn: '1h' })
    socket.user = { ...socket.user }
    send(socket, { type: 'token_refreshed', token: newToken })
  })
  break
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Handshake Authentication** | `verifyClient` option | Reject connections before socket is created |
| **Token from Query String** | `parseUrl(req.url).query.token` | Pass JWT in WebSocket URL |
| **Attach Decoded User to Request** | `req.user = decoded` | Bridge handshake → connection handler |
| **Socket User Assignment** | `socket.user = request.user` | Attach identity to socket for later use |
| **Role-Based Authorization** | `socket.user.role !== 'admin'` | Gate actions behind role checks |
| **Token Issuance** | `jwt.sign(...)` | Create signed tokens for clients |
| **Auth Failure Detection** | `event.code === 1008` | Client detects rejection by close code |
| **Token Refresh** | refresh action over WS | Extend session without reconnecting |

---

## What to Build Next

- **Cookie-based tokens** — more secure in browsers (no token in URL)
- **Per-user data streams** — only send events relevant to `socket.user.id`
- **Session invalidation** — maintain a token blacklist to force-logout users
- **Move to `06-binary.md`** — send images, audio, or file chunks over WebSocket
