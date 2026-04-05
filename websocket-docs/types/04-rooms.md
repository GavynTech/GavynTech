# Type 04 — Room-Based WebSockets

## What It Does

Clients join named **rooms**. Messages in a room go to everyone in that room.
Unlike pub/sub channels, rooms typically carry **user identity** and enforce
rules like "only N users per room" or "private rooms by invite."

This is how multiplayer games, group DMs, and collaborative workspaces work.

---

## When to Use It

- Multiplayer game lobbies / sessions
- Group messaging (Slack channels, Discord servers)
- Collaborative editing (only people in this doc see changes)
- Video call signaling rooms

---

## Infrastructure Diagram

```
Client A  →  join("room:abc")   ─┐
Client B  →  join("room:abc")   ─┤─→ [room:abc]  { socketA, socketB }
Client C  →  join("room:xyz")   ─┘─→ [room:xyz]  { socketC }

Client A  →  message("room:abc", "hello")
              → socketA ✓
              → socketB ✓
              → socketC ✗  (different room)
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')

const wss = new WebSocketServer({ port: 8080 })

// PATTERN: Room Registry
// Map of roomId → { name, sockets: Set, maxSize, createdAt }
// Richer than a plain pub/sub channel — carries metadata.
const rooms = new Map()

// PATTERN: Client State Map
// Separate map from socket → client info. Keeps socket objects clean.
// Using a Map (not socket properties) makes it easier to serialize/iterate.
const clients = new Map() // socket → { id, username, roomId }

let nextClientId = 1

wss.on('connection', (socket) => {

  // PATTERN: Client Registration
  // Register every new connection immediately with a unique ID.
  const clientId = `client_${nextClientId++}`
  clients.set(socket, {
    id: clientId,
    username: null,     // set when client sends 'identify' action
    roomId: null        // set when client joins a room
  })

  send(socket, { type: 'connected', clientId })

  socket.on('message', (data) => {
    let msg
    try { msg = JSON.parse(data.toString()) }
    catch { return send(socket, { type: 'error', text: 'Invalid JSON' }) }

    const client = clients.get(socket)

    switch (msg.action) {
      case 'identify':    return handleIdentify(socket, client, msg)
      case 'create_room': return handleCreateRoom(socket, client, msg)
      case 'join':        return handleJoin(socket, client, msg)
      case 'leave':       return handleLeave(socket, client)
      case 'message':     return handleMessage(socket, client, msg)
      case 'list_rooms':  return handleListRooms(socket)
      default:
        send(socket, { type: 'error', text: `Unknown action: ${msg.action}` })
    }
  })

  socket.on('close', () => {
    const client = clients.get(socket)
    if (client?.roomId) leaveRoom(socket, client)
    clients.delete(socket)
  })

  socket.on('error', (err) => console.error(err.message))
})

function handleIdentify(socket, client, msg) {
  if (!msg.username?.trim()) {
    return send(socket, { type: 'error', text: 'username required' })
  }
  client.username = msg.username.trim()
  send(socket, { type: 'identified', username: client.username })
}

function handleCreateRoom(socket, client, msg) {
  if (!client.username) return send(socket, { type: 'error', text: 'Identify first' })

  const roomId = `room_${Date.now()}`

  // PATTERN: Room Object
  // A room is an object with identity, capacity, and a socket set.
  rooms.set(roomId, {
    id: roomId,
    name: msg.name || roomId,
    sockets: new Set(),
    maxSize: msg.maxSize || 10,
    createdAt: new Date().toISOString(),
    ownerId: client.id
  })

  send(socket, { type: 'room_created', roomId, name: rooms.get(roomId).name })
  console.log(`Room created: ${roomId}`)
}

function handleJoin(socket, client, msg) {
  if (!client.username) return send(socket, { type: 'error', text: 'Identify first' })

  const room = rooms.get(msg.roomId)
  if (!room) return send(socket, { type: 'error', text: 'Room not found' })

  // PATTERN: Capacity Check
  if (room.sockets.size >= room.maxSize) {
    return send(socket, { type: 'error', text: 'Room is full' })
  }

  // If already in a room, leave it first
  if (client.roomId) leaveRoom(socket, client)

  room.sockets.add(socket)
  client.roomId = msg.roomId

  send(socket, {
    type: 'joined',
    roomId: msg.roomId,
    name: room.name,
    members: getRoomUsernames(room)
  })

  // PATTERN: Room Announcement
  // Broadcast a system message to the room (excluding the joiner).
  broadcastToRoom(room, {
    type: 'system',
    text: `${client.username} joined`,
    members: getRoomUsernames(room)
  }, socket)
}

function handleLeave(socket, client) {
  if (!client.roomId) return send(socket, { type: 'error', text: 'Not in a room' })
  leaveRoom(socket, client)
  send(socket, { type: 'left' })
}

function handleMessage(socket, client, msg) {
  if (!client.roomId) return send(socket, { type: 'error', text: 'Join a room first' })
  const room = rooms.get(client.roomId)
  if (!room) return

  broadcastToRoom(room, {
    type: 'message',
    from: client.username,
    text: msg.text,
    timestamp: new Date().toISOString()
  })
}

function handleListRooms(socket) {
  const list = [...rooms.values()].map((r) => ({
    id: r.id,
    name: r.name,
    members: r.sockets.size,
    maxSize: r.maxSize
  }))
  send(socket, { type: 'room_list', rooms: list })
}

function leaveRoom(socket, client) {
  const room = rooms.get(client.roomId)
  if (!room) return

  room.sockets.delete(socket)
  const username = client.username
  client.roomId = null

  broadcastToRoom(room, {
    type: 'system',
    text: `${username} left`,
    members: getRoomUsernames(room)
  })

  // PATTERN: Empty Room Cleanup
  if (room.sockets.size === 0) {
    rooms.delete(room.id)
    console.log(`Room ${room.id} deleted (empty)`)
  }
}

// PATTERN: Room Broadcast
// Send to everyone in a specific room, with optional exclusion.
function broadcastToRoom(room, data, exclude = null) {
  const payload = JSON.stringify(data)
  room.sockets.forEach((s) => {
    if (s !== exclude && s.readyState === WebSocket.OPEN) {
      s.send(payload)
    }
  })
}

function getRoomUsernames(room) {
  return [...room.sockets].map((s) => clients.get(s)?.username).filter(Boolean)
}

function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

wss.on('listening', () => console.log('Room server on ws://localhost:8080'))
```

---

## Client Interaction Flow

```
ws.send({ action: 'identify',    username: 'Alice' })
ws.send({ action: 'create_room', name: 'Game Room', maxSize: 4 })
ws.send({ action: 'join',        roomId: 'room_...' })
ws.send({ action: 'message',     text: 'Hello room!' })
ws.send({ action: 'leave' })
ws.send({ action: 'list_rooms' })
```

---

## State Snapshot

```
rooms = Map {
  "room_1712345678" → {
    id: "room_1712345678",
    name: "Game Room",
    sockets: Set { socketA, socketB },
    maxSize: 4,
    ownerId: "client_1"
  }
}

clients = Map {
  socketA → { id: "client_1", username: "Alice", roomId: "room_1712345678" },
  socketB → { id: "client_2", username: "Bob",   roomId: "room_1712345678" },
  socketC → { id: "client_3", username: "Carol",  roomId: null }
}
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Room Registry** | `rooms = new Map()` | Map of roomId → room metadata + socket set |
| **Client State Map** | `clients = new Map()` | Separate store for per-socket state |
| **Client Registration** | on `connection` | Assign ID to every new socket immediately |
| **Room Object** | `{ id, name, sockets, maxSize }` | Structured object representing a room |
| **Capacity Check** | `sockets.size >= maxSize` | Reject joins when room is full |
| **Room Announcement** | `broadcastToRoom(...)` | Notify room members of join/leave events |
| **Room Broadcast** | `room.sockets.forEach(send)` | Send to everyone in one room |
| **Empty Room Cleanup** | `if size === 0 delete` | Remove empty rooms to free memory |
| **Members List** | `getRoomUsernames(room)` | Send current member list with announcements |

---

## What to Build Next

- **Room passwords** — require a code to join a private room
- **Room history** — new joiners receive recent message history
- **Roles** — owner can kick members, mute users
- **Move to `05-auth.md`** — add JWT-based identity so usernames can't be spoofed
