# Type 01 — Echo Server

## What It Does

The echo server is the "Hello World" of WebSockets.
Every message the client sends, the server immediately sends back.

No state, no rooms, no logic — just the raw connection lifecycle.

---

## When to Use It

- Learning and testing WebSocket connections
- Ping/latency measurement tools
- Verifying a WebSocket server is alive and responding
- Network debugging

---

## Infrastructure Diagram

```
[Browser Client]
      |
      |  "hello"  →
      |  ← "hello"
      |
[Echo Server : port 8080]
      |
  (no database, no state)
  one connection = one loop
```

---

## Install

```bash
npm install ws
```

---

## Server Code

```js
// server.js
const { WebSocketServer } = require('ws')

// PATTERN: Server Init
// Create a WebSocket server bound to a TCP port.
// Every incoming connection gets its own `socket` object.
const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Connection Handler
// This fires once per client that connects.
// `socket` represents THIS specific client's connection.
wss.on('connection', (socket, request) => {

  // `request` is the original HTTP upgrade request.
  // Useful for reading headers, IP address, cookies.
  const clientIP = request.socket.remoteAddress
  console.log(`Client connected: ${clientIP}`)

  // PATTERN: Message Handler
  // Fires every time this client sends a message.
  // `data` is a Buffer by default — convert to string for text.
  socket.on('message', (data) => {
    const message = data.toString()
    console.log(`Received: ${message}`)

    // PATTERN: Echo
    // Send the same message back to the sender.
    // socket.send() targets only THIS client.
    socket.send(message)
  })

  // PATTERN: Disconnect Handler
  // Fires when this client's connection closes.
  // `code` is the close code (1000 = normal, 1006 = abnormal).
  // `reason` is an optional Buffer with a human-readable reason.
  socket.on('close', (code, reason) => {
    console.log(`Client disconnected. Code: ${code}, Reason: ${reason.toString()}`)
  })

  // PATTERN: Error Handler
  // Always handle errors — an unhandled error event crashes Node.
  socket.on('error', (err) => {
    console.error(`Socket error: ${err.message}`)
  })

  // PATTERN: Welcome Message
  // Send a message immediately upon connection.
  // Useful for confirming the connection is ready.
  socket.send('Connected to echo server.')
})

// PATTERN: Server Lifecycle Log
wss.on('listening', () => {
  console.log('Echo server running on ws://localhost:8080')
})

console.log('Starting echo server...')
```

---

## Client Code (Browser)

```html
<!-- client.html -->
<!DOCTYPE html>
<html>
<head><title>Echo Client</title></head>
<body>
  <input id="input" type="text" placeholder="Type a message..." />
  <button onclick="sendMessage()">Send</button>
  <ul id="log"></ul>

<script>
  // PATTERN: Client Connection
  // new WebSocket(url) opens the connection immediately.
  // Use wss:// in production, ws:// for local dev only.
  const ws = new WebSocket('ws://localhost:8080')

  // PATTERN: Client Open Handler
  // Fires when the connection is fully established.
  // Safe to call ws.send() only after this fires.
  ws.onopen = () => {
    log('Connected to server')
  }

  // PATTERN: Client Message Handler
  // Fires when the server sends data.
  // event.data contains the payload (string, Blob, or ArrayBuffer).
  ws.onmessage = (event) => {
    log(`Server says: ${event.data}`)
  }

  // PATTERN: Client Close Handler
  // Fires when the connection closes.
  // event.code and event.reason describe why.
  ws.onclose = (event) => {
    log(`Disconnected. Code: ${event.code}`)
  }

  // PATTERN: Client Error Handler
  ws.onerror = (error) => {
    log(`Error: ${error.message}`)
  }

  // Send a message to the server
  function sendMessage() {
    const input = document.getElementById('input')
    const text = input.value.trim()
    if (!text) return

    // PATTERN: Send Guard
    // Only send if the connection is OPEN (readyState === 1).
    // Sending on a closed socket throws an error.
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(text)
      input.value = ''
    } else {
      log('Not connected!')
    }
  }

  // Helper: append message to the log
  function log(msg) {
    const li = document.createElement('li')
    li.textContent = msg
    document.getElementById('log').appendChild(li)
  }
</script>
</body>
</html>
```

---

## Client Code (Node.js — for testing without a browser)

```js
// test-client.js
const WebSocket = require('ws')

const ws = new WebSocket('ws://localhost:8080')

ws.on('open', () => {
  console.log('Connected')
  ws.send('Hello from test client!')
})

ws.on('message', (data) => {
  console.log(`Echo received: ${data}`)
  ws.close() // close after one echo for this test
})

ws.on('close', () => {
  console.log('Disconnected')
})
```

---

## ReadyState Values (critical to memorize)

```
WebSocket.CONNECTING  = 0   // handshake in progress
WebSocket.OPEN        = 1   // connected, safe to send
WebSocket.CLOSING     = 2   // close frame sent, waiting for confirmation
WebSocket.CLOSED      = 3   // fully closed
```

Always check `ws.readyState === WebSocket.OPEN` before calling `ws.send()`.

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Server Init** | `new WebSocketServer(...)` | Bind server to a port |
| **Connection Handler** | `wss.on('connection', ...)` | Entry point per client |
| **Message Handler** | `socket.on('message', ...)` | Process incoming data |
| **Echo** | `socket.send(data)` | Reflect data back to sender |
| **Disconnect Handler** | `socket.on('close', ...)` | Cleanup per client |
| **Error Handler** | `socket.on('error', ...)` | Prevent crash on error |
| **Welcome Message** | `socket.send(...)` on connect | Confirm connection to client |
| **Client Connection** | `new WebSocket(url)` | Open a WebSocket from client |
| **Client Open Handler** | `ws.onopen` | Ready to send |
| **Client Message Handler** | `ws.onmessage` | Receive data from server |
| **Send Guard** | `readyState === OPEN` | Prevent send on closed socket |

---

## What to Build Next

- **Add message logging** — write all messages to a file or database
- **Add timestamps** — attach a server timestamp to each echo
- **Multiple clients** — open two browser tabs, observe they are independent
- **Move to `02-broadcast.md`** — share messages across all clients
