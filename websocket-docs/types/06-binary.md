# Type 06 — Binary Data & File Streaming

## What It Does

WebSockets can send raw binary data (Buffers, ArrayBuffers) — not just text.
This enables file uploads, image streaming, audio chunks, and binary protocols
without base64 encoding overhead.

---

## When to Use It

- Real-time image / video feed from a camera or sensor
- File uploads in chunks with progress reporting
- Audio streaming (voice chat, music)
- Binary protocol communication (custom game packets, IoT sensors)
- Canvas/image collaboration tools

---

## Infrastructure Diagram

```
[Client]
  File → slice into chunks (e.g. 64KB each)
  → send: metadata JSON  (filename, size, type)
  → send: binary chunk 1
  → send: binary chunk 2
  → send: binary chunk N
  → send: { action: 'upload_complete' }

[Server]
  → receive metadata → create write stream
  → receive chunks   → write to disk
  → receive complete → close stream, respond
```

---

## Server Code

```js
// server.js
const { WebSocketServer, WebSocket } = require('ws')
const fs = require('fs')
const path = require('path')

const wss = new WebSocketServer({ port: 8080 })

// Ensure upload directory exists
const UPLOAD_DIR = path.join(__dirname, 'uploads')
if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR)

wss.on('connection', (socket) => {

  // PATTERN: Per-Connection Upload State
  // Each client gets its own upload context — they don't interfere.
  let uploadContext = null

  // PATTERN: Mixed Message Handler
  // A single `message` event handles BOTH text (JSON control messages)
  // and binary (file chunk) data. Distinguish using typeof or Buffer.isBuffer().
  socket.on('message', (data, isBinary) => {

    if (!isBinary) {
      // TEXT: control message (JSON)
      let msg
      try { msg = JSON.parse(data.toString()) }
      catch { return send(socket, { type: 'error', text: 'Invalid JSON' }) }

      handleTextMessage(socket, msg)

    } else {
      // BINARY: raw file chunk
      handleBinaryChunk(socket, data)
    }
  })

  function handleTextMessage(socket, msg) {
    switch (msg.action) {

      case 'upload_start':
        // PATTERN: Upload Metadata First
        // Client sends file info before any binary data.
        // Server prepares the write stream and acknowledgement.
        const safeName = path.basename(msg.filename) // prevent path traversal
        const filePath = path.join(UPLOAD_DIR, `${Date.now()}_${safeName}`)

        uploadContext = {
          filename: safeName,
          filePath,
          totalSize: msg.size,
          receivedBytes: 0,
          writeStream: fs.createWriteStream(filePath),
          startTime: Date.now()
        }

        send(socket, { type: 'upload_ready', filename: safeName })
        break

      case 'upload_complete':
        if (!uploadContext) return send(socket, { type: 'error', text: 'No upload in progress' })

        uploadContext.writeStream.end(() => {
          const elapsed = ((Date.now() - uploadContext.startTime) / 1000).toFixed(2)
          const kbps = ((uploadContext.receivedBytes / 1024) / elapsed).toFixed(1)

          send(socket, {
            type: 'upload_success',
            filename: uploadContext.filename,
            bytes: uploadContext.receivedBytes,
            elapsed: `${elapsed}s`,
            speed: `${kbps} KB/s`
          })

          console.log(`Saved: ${uploadContext.filePath} (${uploadContext.receivedBytes} bytes)`)
          uploadContext = null
        })
        break

      default:
        send(socket, { type: 'error', text: `Unknown action: ${msg.action}` })
    }
  }

  function handleBinaryChunk(socket, chunk) {
    if (!uploadContext) {
      return send(socket, { type: 'error', text: 'Send upload_start first' })
    }

    // PATTERN: Streaming Write
    // Write each chunk to disk as it arrives instead of buffering all in memory.
    // Critical for large files — prevents OOM errors.
    uploadContext.writeStream.write(chunk)
    uploadContext.receivedBytes += chunk.length

    // PATTERN: Progress Reporting
    // Send progress percentage back to client after each chunk.
    const percent = Math.round((uploadContext.receivedBytes / uploadContext.totalSize) * 100)
    send(socket, {
      type: 'progress',
      percent,
      received: uploadContext.receivedBytes,
      total: uploadContext.totalSize
    })
  }

  socket.on('close', () => {
    // PATTERN: Interrupted Upload Cleanup
    // If client disconnects mid-upload, close and delete the partial file.
    if (uploadContext) {
      uploadContext.writeStream.destroy()
      fs.unlink(uploadContext.filePath, () => {})
      console.log(`Partial upload deleted: ${uploadContext.filePath}`)
      uploadContext = null
    }
  })

  socket.on('error', (err) => console.error(err.message))
})

// Serve files back (download) — simple example
wss.on('connection', (socket) => {})  // already handled above

function send(socket, data) {
  if (socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify(data))
  }
}

wss.on('listening', () => console.log('Binary server on ws://localhost:8080'))
```

---

## Client Code (Browser)

```html
<!DOCTYPE html>
<html>
<head><title>File Upload via WebSocket</title></head>
<body>
  <input type="file" id="fileInput" />
  <button onclick="uploadFile()">Upload</button>
  <progress id="progress" value="0" max="100" style="width:300px"></progress>
  <div id="status"></div>

<script>
  const ws = new WebSocket('ws://localhost:8080')
  const CHUNK_SIZE = 64 * 1024 // 64 KB chunks

  ws.onopen = () => setStatus('Connected')
  ws.onerror = () => setStatus('Error')
  ws.onclose = () => setStatus('Disconnected')

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data)

    switch (msg.type) {
      case 'upload_ready':
        setStatus(`Server ready — sending chunks...`)
        // PATTERN: Start Sending After Server Ack
        // Wait for server confirmation before sending binary data.
        // This prevents sending chunks before the server is prepared.
        sendChunks()
        break

      case 'progress':
        document.getElementById('progress').value = msg.percent
        setStatus(`${msg.percent}% — ${(msg.received / 1024).toFixed(1)} KB / ${(msg.total / 1024).toFixed(1)} KB`)
        break

      case 'upload_success':
        setStatus(`Done! ${msg.filename} — ${msg.bytes} bytes in ${msg.elapsed} @ ${msg.speed}`)
        document.getElementById('progress').value = 100
        break

      case 'error':
        setStatus(`Error: ${msg.text}`)
        break
    }
  }

  let currentFile = null
  let fileReader = null

  function uploadFile() {
    const input = document.getElementById('fileInput')
    if (!input.files[0]) return setStatus('Select a file first')

    currentFile = input.files[0]

    // PATTERN: Upload Metadata First
    // Send file info as JSON before any binary chunks.
    ws.send(JSON.stringify({
      action: 'upload_start',
      filename: currentFile.name,
      size: currentFile.size,
      type: currentFile.type
    }))

    setStatus(`Initiating upload: ${currentFile.name} (${(currentFile.size / 1024).toFixed(1)} KB)`)
  }

  function sendChunks() {
    let offset = 0

    // PATTERN: Chunked File Reading
    // Use FileReader to read the file in fixed-size slices.
    // This keeps memory usage constant regardless of file size.
    function readNextChunk() {
      if (offset >= currentFile.size) {
        // All chunks sent — signal completion
        ws.send(JSON.stringify({ action: 'upload_complete' }))
        return
      }

      const slice = currentFile.slice(offset, offset + CHUNK_SIZE)
      fileReader = new FileReader()

      fileReader.onload = (e) => {
        // PATTERN: Send ArrayBuffer Directly
        // ws.send() with an ArrayBuffer sends binary (not text).
        // The `isBinary` flag on the server will be true.
        ws.send(e.target.result)
        offset += CHUNK_SIZE
        readNextChunk()
      }

      fileReader.readAsArrayBuffer(slice)
    }

    readNextChunk()
  }

  function setStatus(text) {
    document.getElementById('status').textContent = text
  }
</script>
</body>
</html>
```

---

## Binary vs Text Decision Guide

```
Use TEXT (JSON) for:
  - Control messages (start, stop, config)
  - Small structured data
  - Human-readable protocol messages

Use BINARY for:
  - File contents
  - Image/audio/video data
  - Custom packed binary protocols (game state, sensor readings)
  - Anything where base64 encoding overhead (33%) would matter
```

## Handling Binary on the Client (types)

```js
// Default: string
ws.binaryType = 'blob'          // Blob (default in browsers)
ws.binaryType = 'arraybuffer'   // ArrayBuffer (easier to process)

// Check what arrived:
ws.onmessage = (event) => {
  if (event.data instanceof ArrayBuffer) {
    // binary
    const view = new Uint8Array(event.data)
  } else {
    // text
    const text = event.data
  }
}
```

---

## Patterns Identified

| Pattern Name | Where | Description |
|---|---|---|
| **Mixed Message Handler** | `socket.on('message', (data, isBinary))` | Handle both text and binary in one handler |
| **Upload Metadata First** | `{ action: 'upload_start', filename, size }` | Send file info before binary data |
| **Per-Connection Upload State** | `let uploadContext = null` | Track upload progress per client |
| **Streaming Write** | `writeStream.write(chunk)` | Write chunks to disk as they arrive |
| **Progress Reporting** | `{ type: 'progress', percent }` | Feedback to client on each chunk |
| **Start Sending After Ack** | wait for `upload_ready` | Handshake before binary stream starts |
| **Chunked File Reading** | `FileReader` + `slice()` | Read large files in fixed-size pieces |
| **Send ArrayBuffer Directly** | `ws.send(arrayBuffer)` | Send binary without base64 encoding |
| **Interrupted Upload Cleanup** | `socket.on('close', ...)` | Delete partial files on disconnect |

---

## What to Build Next

- **Resume uploads** — track offset, allow clients to continue after disconnect
- **Image preview** — server processes image and sends a thumbnail back
- **Broadcast binary** — stream camera frames to all connected viewers
- **Move to `07-heartbeat.md`** — keep long-lived connections alive
