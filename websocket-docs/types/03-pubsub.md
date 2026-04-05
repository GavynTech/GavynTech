# Type 03 — Pub/Sub Channels

## What It Does

Clients **subscribe** to named channels (topics). Messages published to a channel
are delivered only to subscribers of that channel — not everyone.

This is the first type where the server maintains meaningful **state per client**.

---

## When to Use It

- News feed with category filters ("sports", "tech", "finance")
- IoT device streams (subscribe to a specific sensor)
- Game events (subscribe to "match:1234" updates)
- Notification topics ("account-alerts", "system-status")

---

## Infrastructure Diagram

```
Client A  →  subscribe("sports")
Client B  →  subscribe("sports")
Client C  →  subscribe("tech")

          [Pub/Sub Server]
          channels = {
            "sports": Set { socketA, socketB },
            "tech":   Set { socketC }
          }

Client A  →  publish("sports", "New game!")
              → socketA gets it  ✓
              → socketB gets it  ✓
              → socketC gets it  ✗  (not subscribed to sports)
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')

const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Channel Registry
// A Map from channel name → Set of subscriber sockets.
// This is the core data structure of any pub/sub system.
const channels = new Map()

// Helper: get or create a channel
function getChannel(name) {
  if (!channels.has(name)) {
    channels.set(name, new Set())
  }
  return channels.get(name)
}

wss.on('connection', (socket) => {

  // PATTERN: Per-Client Subscription State
  // Track which channels THIS socket is subscribed to.
  // Needed so we can unsubscribe cleanly when the client disconnects.
  socket.subscriptions = new Set()

  socket.on('message', (data) => {
    let msg
    try {
      msg = JSON.parse(data.toString())
    } catch {
      return send(socket, { type: 'error', text: 'Invalid JSON' })
    }

    // PATTERN: Action Dispatcher
    // Route incoming messages to handlers based on the `action` field.
    switch (msg.action) {

      case 'subscribe':
        handleSubscribe(socket, msg.channel)
        break

      case 'unsubscribe':
        handleUnsubscribe(socket, msg.channel)
        break

      case 'publish':
        handlePublish(socket, msg.channel, msg.payload)
        break

      default:
        send(socket, { type: 'error', text: `Unknown action: ${msg.action}` })
    }
  })

  // PATTERN: Cleanup on Disconnect
  // When a client disconnects, remove it from ALL channels it was in.
  // Without this, channels fill up with dead sockets (memory leak).
  socket.on('close', () => {
    socket.subscriptions.forEach((channelName) => {
      const channel = channels.get(channelName)
      if (channel) {
        channel.delete(socket)
        // PATTERN: Empty Channel Cleanup
        // Remove the channel entry if no one is subscribed anymore.
        if (channel.size === 0) {
          channels.delete(channelName)
        }
      }
    })
    console.log(`Client disconnected. Channels: ${[...channels.keys()]}`)
  })

  socket.on('error', (err) => console.error(err.message))
})

function handleSubscribe(socket, channelName) {
  if (!channelName) return send(socket, { type: 'error', text: 'channel required' })

  const channel = getChannel(channelName)

  // PATTERN: Idempotent Subscribe
  // Subscribing to a channel you're already in is a no-op.
  if (channel.has(socket)) {
    return send(socket, { type: 'info', text: `Already subscribed to ${channelName}` })
  }

  channel.add(socket)
  socket.subscriptions.add(channelName)

  send(socket, {
    type: 'subscribed',
    channel: channelName,
    subscribers: channel.size
  })

  console.log(`Subscribed to ${channelName}. Subscribers: ${channel.size}`)
}

function handleUnsubscribe(socket, channelName) {
  const channel = channels.get(channelName)
  if (!channel) return send(socket, { type: 'error', text: `Not subscribed to ${channelName}` })

  channel.delete(socket)
  socket.subscriptions.delete(channelName)

  if (channel.size === 0) channels.delete(channelName)

  send(socket, { type: 'unsubscribed', channel: channelName })
}

function handlePublish(socket, channelName, payload) {
  const channel = channels.get(channelName)

  if (!channel || channel.size === 0) {
    return send(socket, { type: 'info', text: `No subscribers on ${channelName}` })
  }

  const envelope = {
    type: 'message',
    channel: channelName,
    payload,
    timestamp: new Date().toISOString()
  }

  // PATTERN: Channel Publish
  // Deliver the message to every subscriber of this channel.
  let delivered = 0
  channel.forEach((subscriber) => {
    if (subscriber.readyState === WebSocket.OPEN) {
      send(subscriber, envelope)
      delivered++
    }
  })

  // PATTERN: Publish Receipt
  // Confirm to the publisher how many clients received the message.
  send(socket, { type: 'published', channel: channelName, delivered })
}

// Helper: send JSON to a socket
function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

wss.on('listening', () => console.log('Pub/Sub server on ws://localhost:8080'))
```

---

## Client Code (Browser)

```html
<!DOCTYPE html>
<html>
<head><title>Pub/Sub Client</title></head>
<body>
  <h3>Pub/Sub WebSocket</h3>

  <div>
    <b>Subscribe:</b>
    <input id="subChannel" placeholder="channel name" />
    <button onclick="subscribe()">Subscribe</button>
    <button onclick="unsubscribe()">Unsubscribe</button>
  </div>

  <div>
    <b>Publish:</b>
    <input id="pubChannel" placeholder="channel name" />
    <input id="pubPayload" placeholder="message" />
    <button onclick="publish()">Publish</button>
  </div>

  <ul id="log"></ul>

<script>
  const ws = new WebSocket('ws://localhost:8080')

  ws.onopen = () => log('Connected')
  ws.onclose = () => log('Disconnected')
  ws.onerror = () => log('Error')

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data)

    // PATTERN: Message Router (Client Side)
    switch (msg.type) {
      case 'subscribed':
        log(`Subscribed to [${msg.channel}] — ${msg.subscribers} subscriber(s)`)
        break
      case 'unsubscribed':
        log(`Unsubscribed from [${msg.channel}]`)
        break
      case 'published':
        log(`Published to [${msg.channel}] — delivered to ${msg.delivered}`)
        break
      case 'message':
        log(`[${msg.channel}] ${JSON.stringify(msg.payload)} @ ${msg.timestamp.slice(11,19)}`)
        break
      case 'error':
      case 'info':
        log(`${msg.type.toUpperCase()}: ${msg.text}`)
        break
    }
  }

  // PATTERN: Action-Based Client Messages
  // Instead of sending raw text, every client message has an `action` field.
  // The server switches on this action field to route the request.

  function subscribe() {
    const channel = document.getElementById('subChannel').value.trim()
    if (channel) ws.send(JSON.stringify({ action: 'subscribe', channel }))
  }

  function unsubscribe() {
    const channel = document.getElementById('subChannel').value.trim()
    if (channel) ws.send(JSON.stringify({ action: 'unsubscribe', channel }))
  }

  function publish() {
    const channel = document.getElementById('pubChannel').value.trim()
    const payload = document.getElementById('pubPayload').value.trim()
    if (channel && payload) {
      ws.send(JSON.stringify({ action: 'publish', channel, payload }))
    }
  }

  function log(text) {
    const li = document.createElement('li')
    li.textContent = text
    document.getElementById('log').prepend(li) // newest at top
  }
</script>
</body>
</html>
```

---

## Channel Registry Data Structure

```
channels = Map {
  "sports"  → Set { socketA, socketB, socketC },
  "tech"    → Set { socketB, socketD },
  "finance" → Set { socketE }
}

socketB.subscriptions = Set { "sports", "tech" }
socketA.subscriptions = Set { "sports" }
```

The bidirectional reference (channel → sockets, socket → channels) allows
O(1) lookup in both directions — fast publish AND fast cleanup on disconnect.

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Channel Registry** | `channels = new Map()` | Map of channel name → subscriber sockets |
| **Per-Client Subscription State** | `socket.subscriptions` | Track subscriptions per socket for cleanup |
| **Action Dispatcher** | `switch(msg.action)` | Route incoming messages to named handlers |
| **Idempotent Subscribe** | `channel.has(socket)` check | Subscribing twice is safe, no duplicates |
| **Cleanup on Disconnect** | `socket.on('close', ...)` | Remove socket from all channels on disconnect |
| **Empty Channel Cleanup** | `if size === 0 delete` | GC empty channels to prevent memory leaks |
| **Channel Publish** | `channel.forEach(send)` | Deliver to all subscribers of one channel |
| **Publish Receipt** | `{ delivered: N }` response | Confirm delivery count to publisher |
| **Action-Based Client Messages** | `{ action: 'subscribe' }` | Client requests have an action field |

---

## What to Build Next

- **Channel metadata** — store description, creator, created-at per channel
- **Private channels** — require a password or token to subscribe
- **Wildcard subscriptions** — subscribe to `"sports.*"` and match `"sports.nba"`, `"sports.nfl"`
- **Move to `04-rooms.md`** — rooms add user identity and access control on top of pub/sub
