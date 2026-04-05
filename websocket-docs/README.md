# WebSocket Documentation — Complete Reference

A learner-focused reference covering every major WebSocket pattern — from a basic echo server to horizontally scaled multi-server architectures.

Each file covers one type of WebSocket usage with full server/client code, annotated infrastructure diagrams, named reusable patterns, and progression suggestions.

---

## Contents

| File | Topic | Level |
|---|---|---|
| [`00-overview.md`](./00-overview.md) | Protocol internals, frame structure, connection lifecycle, close codes | Foundation |
| [`types/01-echo.md`](./types/01-echo.md) | Echo Server | Beginner |
| [`types/02-broadcast.md`](./types/02-broadcast.md) | Broadcast / Chat Room | Beginner |
| [`types/03-pubsub.md`](./types/03-pubsub.md) | Pub/Sub Channels | Intermediate |
| [`types/04-rooms.md`](./types/04-rooms.md) | Room-Based / Namespaced Connections | Intermediate |
| [`types/05-auth.md`](./types/05-auth.md) | Authenticated WebSockets (JWT) | Intermediate |
| [`types/06-binary.md`](./types/06-binary.md) | Binary Data / File Streaming | Intermediate |
| [`types/07-heartbeat.md`](./types/07-heartbeat.md) | Ping/Pong Heartbeat | Intermediate |
| [`types/08-reconnect.md`](./types/08-reconnect.md) | Auto-Reconnecting Client | Intermediate |
| [`types/09-ratelimit.md`](./types/09-ratelimit.md) | Rate Limiting | Advanced |
| [`types/10-redis.md`](./types/10-redis.md) | Scaling with Redis Pub/Sub | Advanced |
| `types/11-patterns.md` *(coming soon)* | Full pattern reference across all types | Reference |

---

## How to Use This

Start with `00-overview.md` to understand the WebSocket protocol itself — handshake, frame structure, close codes, and the full infrastructure stack.

Then read `types/01-echo.md`. Every other type builds on the concepts introduced there.

Each type file follows a consistent structure:

1. What it does
2. When to use it
3. Infrastructure diagram
4. Annotated server code
5. Annotated client code
6. Named patterns you can reuse
7. What to build next

---

## Stack

Server examples use **Node.js** with the `ws` library (and `socket.io` / `ioredis` where applicable).
Client examples use the native browser **WebSocket API** — no libraries required.

---

## Status

- [x] Protocol overview
- [x] 10 type references (echo → Redis scaling)
- [ ] `11-patterns.md` — consolidated pattern reference
