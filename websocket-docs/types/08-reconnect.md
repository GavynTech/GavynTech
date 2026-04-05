# Type 08 — Auto-Reconnecting Client

## What It Does

A WebSocket client that automatically re-establishes the connection after a
disconnect — with exponential backoff, jitter, and max retry limits.

This lives entirely on the **client side**. The server doesn't need changes.

---

## When to Use It

- Any production client-side app
- Mobile apps on flaky networks
- Long-running dashboards and monitoring tools
- Apps where the user must not manually refresh to reconnect

---

## Infrastructure Diagram

```
[Client]
  connect()
    ↓
  [connected]
    ↓ (network drops)
  onclose fires
    ↓
  scheduleReconnect()
    ↓ wait (backoff)
  connect() again
    ↓
  [connected again]

Backoff schedule (with jitter):
  attempt 1: ~1s
  attempt 2: ~2s
  attempt 3: ~4s
  attempt 4: ~8s
  attempt 5: ~16s
  attempt 6+: ~30s (max cap)
```

---

## Client Code (Browser — Reusable Class)

```js
// ReconnectingWebSocket.js

class ReconnectingWebSocket {

  constructor(url, options = {}) {
    this.url = url

    // PATTERN: Configurable Reconnect Options
    // Expose all timing constants so callers can tune for their use case.
    this.options = {
      minDelay:        options.minDelay        ?? 1_000,   // 1s minimum wait
      maxDelay:        options.maxDelay        ?? 30_000,  // 30s maximum wait
      maxRetries:      options.maxRetries      ?? Infinity, // infinite by default
      jitter:          options.jitter          ?? 0.3,     // 30% random jitter
      binaryType:      options.binaryType      ?? 'arraybuffer',
      protocols:       options.protocols       ?? [],
    }

    this.retryCount = 0
    this.shouldReconnect = true // set false to permanently close
    this._ws = null
    this._reconnectTimer = null

    // PATTERN: Event Handler Slots
    // Expose the same interface as native WebSocket so callers
    // can assign handlers just like they would on a plain ws.
    this.onopen    = null
    this.onmessage = null
    this.onclose   = null
    this.onerror   = null

    this._connect()
  }

  _connect() {
    // PATTERN: Guard Against Double Connect
    // Don't open a new socket if one is already open or connecting.
    if (this._ws &&
      (this._ws.readyState === WebSocket.OPEN ||
       this._ws.readyState === WebSocket.CONNECTING)) {
      return
    }

    console.log(`[WS] Connecting to ${this.url} (attempt ${this.retryCount + 1})`)

    this._ws = new WebSocket(this.url, this.options.protocols)
    this._ws.binaryType = this.options.binaryType

    this._ws.onopen = (event) => {
      // PATTERN: Reset on Success
      // A successful connection resets the retry counter.
      // Without this, backoff keeps growing even after reconnecting.
      console.log('[WS] Connected')
      this.retryCount = 0
      this._clearReconnectTimer()
      if (this.onopen) this.onopen(event)
    }

    this._ws.onmessage = (event) => {
      if (this.onmessage) this.onmessage(event)
    }

    this._ws.onerror = (event) => {
      console.error('[WS] Error')
      if (this.onerror) this.onerror(event)
      // Note: onerror is always followed by onclose — reconnect happens there
    }

    this._ws.onclose = (event) => {
      console.log(`[WS] Closed. Code: ${event.code}, Reason: ${event.reason}`)
      if (this.onclose) this.onclose(event)

      // PATTERN: Intentional vs Unintentional Close
      // Close code 1000 = normal (user called close()) — don't reconnect.
      // All other codes = unexpected — try to reconnect.
      if (!this.shouldReconnect || event.code === 1000) {
        console.log('[WS] Closed intentionally — not reconnecting')
        return
      }

      this._scheduleReconnect()
    }
  }

  _scheduleReconnect() {
    if (this.retryCount >= this.options.maxRetries) {
      console.error(`[WS] Max retries (${this.options.maxRetries}) reached — giving up`)
      if (this.onclose) this.onclose({ code: 1006, reason: 'Max retries exceeded' })
      return
    }

    // PATTERN: Exponential Backoff with Jitter
    // Base delay doubles each attempt, capped at maxDelay.
    // Jitter adds randomness to prevent "thundering herd" —
    // all clients reconnecting at the exact same moment after a server restart.
    const base = Math.min(
      this.options.minDelay * Math.pow(2, this.retryCount),
      this.options.maxDelay
    )
    const jitterAmount = base * this.options.jitter
    const delay = base + (Math.random() * jitterAmount * 2 - jitterAmount)

    console.log(`[WS] Reconnecting in ${Math.round(delay)}ms (attempt ${this.retryCount + 1})`)

    this._reconnectTimer = setTimeout(() => {
      this.retryCount++
      this._connect()
    }, delay)
  }

  _clearReconnectTimer() {
    if (this._reconnectTimer) {
      clearTimeout(this._reconnectTimer)
      this._reconnectTimer = null
    }
  }

  // PATTERN: Public Send with Guard
  // Callers don't need to check readyState — send() does it.
  send(data) {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      this._ws.send(data)
    } else {
      console.warn('[WS] Cannot send — not connected')
    }
  }

  // PATTERN: Intentional Close
  // Setting shouldReconnect = false prevents the onclose handler
  // from scheduling a reconnect.
  close(code = 1000, reason = '') {
    this.shouldReconnect = false
    this._clearReconnectTimer()
    if (this._ws) this._ws.close(code, reason)
  }

  get readyState() {
    return this._ws ? this._ws.readyState : WebSocket.CLOSED
  }
}
```

---

## Usage (Browser)

```html
<script src="ReconnectingWebSocket.js"></script>
<script>
  // PATTERN: Drop-In Replacement
  // ReconnectingWebSocket has the same interface as native WebSocket.
  // Just swap `new WebSocket(...)` with `new ReconnectingWebSocket(...)`.
  const ws = new ReconnectingWebSocket('ws://localhost:8080', {
    minDelay:   1000,
    maxDelay:   30000,
    maxRetries: 10,
    jitter:     0.3
  })

  ws.onopen    = () => updateStatus('Connected')
  ws.onmessage = (e) => console.log('Message:', e.data)
  ws.onclose   = (e) => updateStatus(`Disconnected (${e.code})`)
  ws.onerror   = (e) => console.error('Error', e)

  function sendMessage(text) {
    ws.send(JSON.stringify({ text }))
  }

  function disconnect() {
    ws.close() // intentional — won't reconnect
  }

  function updateStatus(msg) {
    document.getElementById('status').textContent = msg
  }
</script>
```

---

## Backoff Values Table

```
minDelay = 1000ms, maxDelay = 30000ms, jitter = 30%

Attempt  Base    Jitter Range      Actual Wait
1        1000ms  ± 300ms           700ms – 1300ms
2        2000ms  ± 600ms           1400ms – 2600ms
3        4000ms  ± 1200ms          2800ms – 5200ms
4        8000ms  ± 2400ms          5600ms – 10400ms
5        16000ms ± 4800ms          11200ms – 20800ms
6+       30000ms ± 9000ms          21000ms – 39000ms (capped)
```

---

## Thundering Herd Problem (why jitter matters)

```
WITHOUT jitter:
  Server restarts at t=0
  1000 clients all wait exactly 1000ms
  t=1s: 1000 simultaneous reconnects → server overwhelmed → crashes again

WITH jitter:
  Server restarts at t=0
  1000 clients each wait 700–1300ms (random)
  Reconnects spread over ~600ms → manageable load
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Configurable Reconnect Options** | `options = {}` constructor | Expose timing constants to callers |
| **Event Handler Slots** | `this.onopen = null` | Mirror native WebSocket interface |
| **Guard Against Double Connect** | `readyState === OPEN` check | Don't open two connections |
| **Reset on Success** | `retryCount = 0` on open | Restart backoff after reconnect |
| **Intentional vs Unintentional Close** | `code === 1000` check | Don't reconnect on user-initiated close |
| **Exponential Backoff** | `minDelay * 2^retryCount` | Double wait time each attempt |
| **Jitter** | `base ± random range` | Randomize delays to prevent herd |
| **Thundering Herd Prevention** | jitter on reconnect | Spread reconnects over time |
| **Public Send with Guard** | `readyState === OPEN` in send | Safe send without caller checks |
| **Intentional Close** | `shouldReconnect = false` | Permanently close without reconnect |
| **Drop-In Replacement** | same API as WebSocket | Callers need no API changes |

---

## What to Build Next

- **Reconnect with re-auth** — re-fetch JWT token before each reconnect attempt
- **Offline queue** — buffer messages sent while disconnected, flush on reconnect
- **Reconnect UI** — show user "Reconnecting..." with countdown
- **Move to `09-ratelimit.md`** — protect server from message flooding
