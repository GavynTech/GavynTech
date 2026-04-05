# Type 02 — Broadcast / Chat Room

## What It Does

Every message any client sends is forwarded to **all connected clients**.
This is the foundation of real-time chat, live feeds, and shared dashboards.

The key shift from echo: the server maintains a **set of all sockets**
and iterates over them to deliver messages.

---

## When to Use It

- Group chat rooms
- Live activity feeds ("User X just signed up")
- Shared dashboards where everyone sees the same data
- Collaborative tools (everyone sees the same cursor, etc.)

---

## Infrastructure Diagram

```
[Client A]  →  "hello"
                        ↘
              [Broadcast Server]  →  "hello" → [Client A]
                        ↗                    → [Client B]
[Client B]  → (connected)                   → [Client C]

All clients share the same wss.clients set on the server.
```

---

## Server Code

```js
// server.js
const { WebSocketServer } = require('ws')

const wss = new WebSocketServer({ port: 8080 })

wss.on('connection', (socket, request) => {

  // PATTERN: Client Registry
  // wss.clients is a built-in Set that ws maintains automatically.
  // It adds the socket on connection and removes it on close.
  // You don't need to manage this manually for basic broadcast.
  console.log(`Client connected. Total: ${wss.clients.size}`)

  // PATTERN: Username Assignment
  // Assign metadata to the socket object directly.
  // This is safe — the socket is just a JS object.
  socket.username = `User${Math.floor(Math.random() * 1000)}`

  // Tell the new client what username they got
  socket.send(JSON.stringify({
    type: 'system',
    text: `You are ${socket.username}`
  }))

  // PATTERN: Join Announcement
  // Announce to all OTHER clients that someone joined.
  broadcast({
    type: 'system',
    text: `${socket.username} joined the room`
  }, socket) // pass `socket` as the "exclude" param (explained below)

  socket.on('message', (data) => {
    let parsed

    // PATTERN: Safe JSON Parse
    // Always try/catch JSON.parse — a bad client can crash your server
    // if you let an exception bubble up from the message handler.
    try {
      parsed = JSON.parse(data.toString())
    } catch {
      socket.send(JSON.stringify({ type: 'error', text: 'Invalid JSON' }))
      return
    }

    // PATTERN: Message Envelope
    // Wrap outgoing messages in a typed object.
    // The `type` field tells clients how to handle the payload.
    const envelope = {
      type: 'message',
      from: socket.username,
      text: parsed.text,
      timestamp: new Date().toISOString()
    }

    // PATTERN: Broadcast to All
    // Send to every connected client, including the sender.
    broadcast(envelope)
  })

  socket.on('close', () => {
    console.log(`${socket.username} disconnected. Total: ${wss.clients.size}`)

    // PATTERN: Leave Announcement
    // Notify remaining clients that someone left.
    broadcast({
      type: 'system',
      text: `${socket.username} left the room`
    })
  })

  socket.on('error', (err) => {
    console.error(`[${socket.username}] Error: ${err.message}`)
  })
})

// PATTERN: Broadcast Function
// Iterates over all connected clients and sends to each one.
// `exclude` is optional — pass a socket to skip it (e.g., don't echo to sender).
// socket.readyState === WebSocket.OPEN is critical:
//   a client might be in the CLOSING state while still in wss.clients
function broadcast(data, exclude = null) {
  const payload = JSON.stringify(data)

  wss.clients.forEach((client) => {
    const { WebSocket } = require('ws')

    if (client !== exclude && client.readyState === WebSocket.OPEN) {
      client.send(payload)
    }
  })
}

wss.on('listening', () => {
  console.log('Broadcast server on ws://localhost:8080')
})
```

---

## Client Code (Browser)

```html
<!-- client.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Chat Room</title>
  <style>
    #messages { height: 300px; overflow-y: auto; border: 1px solid #ccc; padding: 8px; }
    .system { color: gray; font-style: italic; }
    .message { margin: 4px 0; }
  </style>
</head>
<body>
  <div id="messages"></div>
  <input id="input" type="text" placeholder="Message..." />
  <button onclick="sendMessage()">Send</button>

<script>
  const ws = new WebSocket('ws://localhost:8080')
  const messagesDiv = document.getElementById('messages')

  ws.onopen = () => {
    appendMessage('system', 'Connected to chat room')
  }

  ws.onmessage = (event) => {
    // PATTERN: Message Router (Client Side)
    // Parse the envelope and route to different handlers based on `type`.
    const msg = JSON.parse(event.data)

    switch (msg.type) {
      case 'system':
        appendMessage('system', msg.text)
        break
      case 'message':
        appendMessage('message', `[${msg.timestamp.slice(11,19)}] ${msg.from}: ${msg.text}`)
        break
      case 'error':
        appendMessage('system', `Error: ${msg.text}`)
        break
    }
  }

  ws.onclose = () => {
    appendMessage('system', 'Disconnected from chat room')
  }

  ws.onerror = () => {
    appendMessage('system', 'Connection error')
  }

  function sendMessage() {
    const input = document.getElementById('input')
    const text = input.value.trim()
    if (!text || ws.readyState !== WebSocket.OPEN) return

    // PATTERN: Typed Client Message
    // Send a typed envelope so the server knows what kind of message this is.
    ws.send(JSON.stringify({ type: 'message', text }))
    input.value = ''
  }

  // Allow pressing Enter to send
  document.getElementById('input').addEventListener('keydown', (e) => {
    if (e.key === 'Enter') sendMessage()
  })

  function appendMessage(cssClass, text) {
    const div = document.createElement('div')
    div.className = cssClass
    div.textContent = text
    messagesDiv.appendChild(div)
    // PATTERN: Auto-Scroll
    // Keep the chat scrolled to the bottom as messages arrive.
    messagesDiv.scrollTop = messagesDiv.scrollHeight
  }
</script>
</body>
</html>
```

---

## The wss.clients Set — How It Works

```
wss.clients is a Set<WebSocket>

On connection  → socket is added automatically by the ws library
On close       → socket is removed automatically by the ws library

You can:
  wss.clients.size          // count of connected clients
  wss.clients.forEach(...)  // iterate all
  wss.clients.has(socket)   // check if a socket is connected
```

You can also maintain your **own** Map or Set for additional metadata:
```js
const clients = new Map() // socket → { username, room, ... }
```
This is what you'll do in `04-rooms.md`.

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Client Registry** | `wss.clients` | Built-in set of all connected sockets |
| **Socket Metadata** | `socket.username = ...` | Attach data directly to the socket object |
| **Join Announcement** | broadcast on connect | Notify others a user arrived |
| **Leave Announcement** | broadcast on close | Notify others a user left |
| **Safe JSON Parse** | `try/catch JSON.parse` | Protect server from malformed input |
| **Message Envelope** | `{ type, from, text, timestamp }` | Typed wrapper for all messages |
| **Broadcast Function** | `wss.clients.forEach` | Send to all open connections |
| **Exclude Sender** | `client !== exclude` | Skip the original sender if needed |
| **ReadyState Guard** | `readyState === OPEN` | Skip closing/closed sockets in loops |
| **Message Router** | `switch(msg.type)` | Client-side routing by message type |
| **Auto-Scroll** | `scrollTop = scrollHeight` | Keep chat view at bottom |

---

## What to Build Next

- **Add message history** — store last N messages, send to new clients on join
- **User list** — track usernames, broadcast the list on join/leave
- **Typing indicator** — send a `typing` event type, display on other clients
- **Move to `03-pubsub.md`** — let clients subscribe to specific topics
