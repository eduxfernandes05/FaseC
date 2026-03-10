# Task: Architecture Refactor — Streaming Gateway

> Build API gateway/streaming layer for cloud-based game streaming.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Cloud-Native |
| **Estimated Complexity** | Very High |
| **Estimated Effort** | 3 weeks |
| **Dependencies** | Headless video, Headless audio/input, Container system, Session manager |
| **Blocks** | Azure deployment (full streaming) |
| **Phase** | Phase 4 — Cloud-Native |

---

## Objective

Build a streaming gateway that bridges browser clients to headless Quake engine containers, delivering video frames, audio, and relaying input over WebSocket or WebRTC.

---

## Architecture

```
┌─────────────────┐     ┌─────────────────────────────────────┐
│  Browser Client  │     │          Streaming Gateway           │
│                  │     │                                      │
│  WebSocket ──────│────▶│  Connection Manager                  │
│  ↓ Input JSON    │     │    │                                 │
│  ↑ Video frames  │     │    ├── Input Relay                   │
│  ↑ Audio stream  │     │    │   (client input → engine)       │
│                  │     │    ├── Frame Capture                  │
│  OR              │     │    │   (engine framebuffer → encode)  │
│                  │     │    ├── Video Encoder                  │
│  WebRTC ─────────│────▶│    │   (raw pixels → H.264/VP8)      │
│  ↓ DataChannel   │     │    ├── Audio Capture                  │
│  ↑ Video RTP     │     │    │   (engine audio → stream)        │
│  ↑ Audio RTP     │     │    └── Stream Multiplexer             │
│                  │     │        (combine A/V → client)         │
│                  │     │                                      │
└─────────────────┘     └──────────────┬────────────────────────┘
                                       │
                        ┌──────────────▼────────────────────────┐
                        │    Quake Engine Container (Headless)   │
                        │                                        │
                        │  VID_GetFramebuffer() → raw pixels     │
                        │  SND_GetCapturedAudio() → PCM audio    │
                        │  IN_InjectRemoteInput() ← client input │
                        └────────────────────────────────────────┘
```

---

## Data Flow

### Input Path (Client → Engine)

```
Browser → WebSocket message (JSON) → Gateway → Parse input → IN_InjectRemoteInput()
```

**Input message format**:
```json
{
    "type": "input",
    "forward": 400.0,
    "side": 0.0,
    "up": 0.0,
    "pitch": 10.5,
    "yaw": 45.2,
    "roll": 0.0,
    "buttons": 1
}
```

### Video Path (Engine → Client)

```
Engine frame loop → VID_GetFramebuffer() → Color convert (RGB→YUV) → 
  H.264 encode → WebSocket binary frame → Browser → <video> / <canvas>
```

**Frame message format**: Binary WebSocket message with header:
```
[1 byte: type=0x01] [4 bytes: timestamp] [4 bytes: length] [N bytes: encoded frame]
```

### Audio Path (Engine → Client)

```
Engine mix loop → SND_GetCapturedAudio() → Opus/PCM encode →
  WebSocket binary frame → Browser → AudioContext → Speaker
```

**Audio message format**: Binary WebSocket message with header:
```
[1 byte: type=0x02] [4 bytes: timestamp] [4 bytes: length] [N bytes: audio data]
```

---

## Implementation

### Technology Choice

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Gateway server | **Go** | Efficient WebSocket handling, goroutines for concurrency |
| WebSocket | `gorilla/websocket` | Mature Go WebSocket library |
| Video encoding | **FFmpeg C API** (cgo) or subprocess | Industry standard H.264/VP8 |
| Audio encoding | **Opus** (libopus) | Low-latency audio codec |
| WebRTC (future) | **Pion** | Pure Go WebRTC implementation |

### Key Components

#### Connection Manager
```go
type ConnectionManager struct {
    sessions map[string]*StreamSession
    mu       sync.RWMutex
}

type StreamSession struct {
    sessionID   string
    engineAddr  string
    wsConn      *websocket.Conn
    encoder     *VideoEncoder
    audioCap    *AudioCapture
    inputRelay  *InputRelay
    ctx         context.Context
    cancel      context.CancelFunc
}
```

#### Frame Capture Loop
```go
func (ss *StreamSession) captureLoop() {
    ticker := time.NewTicker(time.Second / 30) // 30 FPS
    for {
        select {
        case <-ticker.C:
            // Read framebuffer from engine
            width, height, pixels := ss.getFramebuffer()
            // Encode frame
            encoded := ss.encoder.Encode(pixels, width, height)
            // Send to client
            ss.wsConn.WriteMessage(websocket.BinaryMessage, encoded)
        case <-ss.ctx.Done():
            return
        }
    }
}
```

#### Input Relay
```go
func (ss *StreamSession) inputLoop() {
    for {
        _, msg, err := ss.wsConn.ReadMessage()
        if err != nil { return }
        
        var input InputMessage
        json.Unmarshal(msg, &input)
        
        // Inject into engine
        ss.injectInput(input)
    }
}
```

---

## Engine Integration Points

The gateway communicates with the engine container through:

| API | Source File | Line | Direction |
|-----|-----------|------|-----------|
| `VID_GetFramebuffer()` | `vid_headless.c` | New function | Engine → Gateway |
| `SND_GetCapturedAudio()` | `snd_null.c` (enhanced) | New function | Engine → Gateway |
| `IN_InjectRemoteInput()` | `in_null.c` (enhanced) | New function | Gateway → Engine |

**Communication method**: Shared memory (same container) or Unix socket (sidecar pattern) or HTTP/gRPC (separate container).

---

## Files to Create

| File | Description |
|------|-------------|
| `services/streaming-gateway/main.go` | Entry point |
| `services/streaming-gateway/ws/handler.go` | WebSocket connection handler |
| `services/streaming-gateway/ws/protocol.go` | Message protocol definitions |
| `services/streaming-gateway/stream/session.go` | Stream session management |
| `services/streaming-gateway/stream/capture.go` | Frame/audio capture |
| `services/streaming-gateway/encode/video.go` | Video encoder wrapper |
| `services/streaming-gateway/encode/audio.go` | Audio encoder wrapper |
| `services/streaming-gateway/input/relay.go` | Input relay to engine |
| `services/streaming-gateway/Dockerfile` | Container build |
| `services/streaming-gateway/go.mod` | Go module |
| `services/streaming-gateway/static/client.html` | Browser client (demo) |
| `services/streaming-gateway/static/client.js` | Client-side WebSocket + rendering |

---

## Latency Budget

| Stage | Target | Measurement |
|-------|--------|-------------|
| Frame render (engine) | ≤ 5ms | `R_RenderView()` at `r_main.c:1049` timing |
| Frame extraction | ≤ 1ms | `VID_GetFramebuffer()` copy time |
| Color conversion | ≤ 1ms | RGB → YUV420 |
| Video encoding | ≤ 10ms | H.264 ultrafast preset |
| Network transit | ≤ 10ms | WebSocket send |
| Client decode | ≤ 5ms | Browser MediaSource API |
| Client render | ≤ 1ms | Canvas draw |
| **Total** | **≤ 33ms** | End-to-end |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_ws_connect` | Client connects via WebSocket |
| `test_ws_disconnect` | Clean disconnect doesn't leak resources |
| `test_frame_delivery` | Client receives video frames |
| `test_audio_delivery` | Client receives audio stream |
| `test_input_relay` | Client input reaches engine |
| `test_latency` | End-to-end latency ≤ 50ms |
| `test_concurrent_streams` | 10 simultaneous streams |
| `test_reconnect` | Client reconnects after network interruption |
| `test_adaptive_quality` | Quality degrades under high latency |

---

## Acceptance Criteria

- [ ] Client connects via WebSocket from browser
- [ ] Video frames stream at 30+ FPS
- [ ] Audio streams with ≤ 100ms delay
- [ ] Client input controls game with ≤ 100ms response
- [ ] End-to-end latency ≤ 50ms (P95)
- [ ] Supports 10+ concurrent streams per gateway instance
- [ ] Browser demo page renders streamed gameplay
- [ ] Clean resource cleanup on disconnect
- [ ] Gateway has health endpoint for Kubernetes probes
- [ ] Gateway logs are structured JSON
