# Digital Human System Docs (v2 - Pipeline Architecture)

## Project Goal

Build a real-time digital human system with full-link control:

- Device-side audio capture (C++)
- Streaming transmission (WebSocket)
- Cloud processing (Go pipeline: ASR → Agent → LLM → TTS)
- Streaming playback + rendering (C++)

**Focus:**

- Low latency
- Streaming experience
- Interrupt capability
- Agent-driven interaction

---

## 🧩 System Architecture

### High-level Flow

```
Client (C++)
  ↓ audio_chunk
Server (Go)
  ↓ ASR (streaming)
  ↓ Agent
  ↓ LLM
  ↓ TTS (streaming)
  ↓ tts_chunk
Client (C++)
  ↓ playback/render
```

---

## 📦 Repository Structure

### 1. digital-human-client (C++)

```
client/
├── audio/        # audio capture (mock → real)
├── network/      # websocket client
├── player/       # audio playback
├── render/       # avatar / lottie
├── core/         # session control
├── main/
└── CMakeLists.txt
```

### 2. digital-human-server (Go)

```
server/
├── cmd/server/main.go
├── internal/
│   ├── ws/
│   ├── session/
│   ├── pipeline/
│   ├── asr/
│   ├── agent/
│   └── tts/
├── pkg/protocol/
└── go.mod
```

### 3. digital-human-docs

```
docs/
├── architecture/
├── protocol/
├── pipeline/
├── client/
├── server/
└── tasks/
```

---

## 🔄 Streaming Protocol (Core)

### Audio Chunk

```json
{
  "type": "audio_chunk",
  "session_id": "string",
  "timestamp": 123456789,
  "seq": 1,
  "data": "base64"
}
```

### TTS Chunk

```json
{
  "type": "tts_chunk",
  "session_id": "string",
  "seq": 1,
  "data": "base64",
  "end": false
}
```

### Control Message (Interrupt)

```json
{
  "type": "interrupt",
  "session_id": "string"
}
```

---

## 🔥 Pipeline Architecture (Server Core)

### Concept

Streaming pipeline using Go channels:

```
audio → asr → agent → llm → tts → output
```

### Data Types

```go
type AudioChunk struct {
    SessionID string
    Seq       int
    Data      []byte
}

type TextChunk struct {
    SessionID string
    Text      string
    Final     bool
}

type TTSChunk struct {
    SessionID string
    Seq       int
    Data      []byte
    End       bool
}
```

### Pipeline Structure

```go
type Pipeline struct {
    AudioIn chan AudioChunk
    TTSOut  chan TTSChunk
}
```

### Stage Interface

```go
type Stage interface {
    Process(ctx context.Context, in <-chan any, out chan<- any)
}
```

### Execution Model

```go
go asr.Process(ctx, audioIn, textOut)
go agent.Process(ctx, textOut, agentOut)
go tts.Process(ctx, agentOut, ttsOut)
```

---

## ⛔ Interrupt Design (Critical)

### Mechanism

Each session owns a context. Interrupt triggers cancel():

```go
ctx, cancel := context.WithCancel(context.Background())
```

### Stage Requirement

```go
select {
case <-ctx.Done():
    return
}
```

---

## 🎧 Client Architecture (C++)

### Core Model

**Dual-loop threading:**

- sendLoop → send audio
- recvLoop → receive tts

### Key Components

- **Audio**: mock first (20ms chunk), later replace with real mic
- **Network**: persistent WebSocket, async send/recv
- **Player**: buffer queue, streaming playback

---

## 🚀 Development Roadmap

### Phase 1 (MVP)

- C++ client sends mock audio
- Go server receives + returns mock TTS
- Full link works

### Phase 2 (Streaming)

- integrate ASR API
- integrate TTS API
- streaming output

### Phase 3 (Agent)

- conversation memory
- intent control
- multi-turn dialogue

### Phase 4 (Advanced)

- interrupt
- emotion / expression
- avatar rendering sync

---

## 🧠 Engineering Principles

1. **Always streaming** - Avoid blocking request-response.

2. **Decouple via channel** - No direct function chaining.

3. **Interrupt-first design** - Must support cancellation from day one.

4. **Protocol consistency** - Single source of truth in docs.

---

## 📌 Task Template (for Trae)

Implement ASR stage in Go pipeline:

- Input: AudioChunk channel
- Output: TextChunk channel
- Mock streaming output
- Respect context cancel

### ✅ Success Criteria

- End-to-end latency < 1s (mock stage)
- Smooth streaming playback
- Interrupt works reliably
- Code structure scalable

---

## 📎 Future Extensions

- WebRTC transport
- Edge inference
- Multi-modal (video + audio)
- Personal memory system

---

*This document is designed for AI-assisted development (Trae) and scalable system evolution.*