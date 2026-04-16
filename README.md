# Digital Human System Docs (v3.0 - Low-latency Streaming Protocol)

## Project Goal

Build a **low-latency, interruptible streaming dialogue system** with:

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
  ↓ audio_chunk (20ms)
Server (Go)
  ↓ ASR (streaming partial)
  ↓ Agent (early trigger)
  ↓ LLM (streaming)
  ↓ TTS (parallel streaming)
  ↓ tts_chunk
Client (C++)
  ↓ playback/render
```

---

## 🔑 Core Protocol Design

### Universal Message Envelope

All messages use a unified envelope:

```json
{
  "type": "audio_chunk",
  "session_id": "uuid",
  "trace_id": "uuid",
  "seq": 12,
  "timestamp": 1710000000,
  "payload": {}
}
```

### Key Fields

| Field       | Purpose                              |
|-------------|--------------------------------------|
| session_id  | Conversation session ID              |
| trace_id    | Round ID (one question-answer pair)  |
| seq         | Sequence control (per stream)        |
| timestamp   | Client timestamp                     |
| type        | Message type                         |

### Message Types

1. **audio_chunk** (Client → Server) - 20ms audio chunks
2. **asr_partial** (Server → Client) - Partial speech recognition
3. **asr_final** (Server → Client) - Final speech recognition
4. **llm_chunk** (Server → Client) - LLM streaming output
5. **llm_end** (Server → Client) - LLM streaming end
6. **tts_chunk** (Server → Client) - TTS streaming audio
7. **tts_end** (Server → Client) - TTS streaming end
8. **interrupt** (Client → Server) - Cancel current processing

---

## 🔥 Pipeline Architecture (Server Core)

### Data Flow

```
audio → asr → agent → llm → tts → output
```

### Key Design Points

1. **Channel-based** - All stages communicate via Go channels
2. **Context propagation** - Interrupt via context cancellation
3. **Trace-based** - All messages in a round share the same trace_id
4. **Early triggering** - LLM starts with partial ASR results
5. **Parallel processing** - LLM and TTS run in parallel

### Stage Interface

```go
type Stage interface {
    Process(ctx context.Context, in <-chan any, out chan<- any)
}
```

---

## ⛔ Interrupt Design

### Mechanism

1. Client sends `interrupt` message with current `trace_id`
2. Server cancels context for that trace
3. All pipeline stages receive `ctx.Done()` and exit
4. Server cleans up resources and prepares for new trace

### Key Requirement

- **New trace_id for new requests** - Prevents data mixing

---

## ⏱ Latency Model

### Key Metrics

| Stage             | Latency    |
|-------------------|------------|
| Audio Upload      | ~50ms      |
| ASR Partial       | 100~300ms  |
| LLM First Token   | 300~800ms  |
| TTS First Packet  | 500~1000ms |

### Optimization Target

**User speaks → hears response < 1s**

### Optimization Strategies

1. **Early LLM trigger** - Start with partial ASR
2. **LLM → TTS parallel** - No waiting for LLM completion
3. **Client early playback** - Start on first TTS chunk

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
│   ├── ws/          # WebSocket gateway
│   ├── session/     # Session management
│   ├── pipeline/    # Pipeline orchestration
│   ├── asr/         # ASR stage
│   ├── agent/       # Agent stage
│   └── tts/         # TTS stage
├── pkg/protocol/    # Protocol definitions
└── go.mod
```

### 3. digital-human-docs

```
docs/
├── architecture/
├── protocol/       # Protocol design
├── pipeline/       # Pipeline architecture
├── client/
├── server/
└── tasks/
```

---

## 🚀 Development Roadmap

### Phase 1 (MVP)

- C++ client sends mock audio
- Go server receives + returns mock TTS
- Full link works

### Phase 2 (Streaming)

- Integrate ASR API (streaming)
- Integrate TTS API (streaming)
- Implement trace-based conversation

### Phase 3 (Agent)

- Add agent with memory
- Implement interrupt capability
- Optimize latency

### Phase 4 (Advanced)

- Emotion / expression
- Avatar rendering sync
- Multi-modal support

---

## 🧠 Engineering Principles

1. **Always streaming** - Avoid blocking request-response
2. **Decouple via channel** - No direct function chaining
3. **Interrupt-first design** - Must support cancellation from day one
4. **Protocol consistency** - Single source of truth in docs
5. **Trace-based** - All messages in a round share the same trace_id

---

## 📌 Task Template (for Trae)

Example: Implement ASR stage in Go pipeline:

- Input: AudioChunk channel with trace_id
- Output: TextChunk channel (partial + final)
- Mock streaming output
- Respect context cancel
- Add debug latency info

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