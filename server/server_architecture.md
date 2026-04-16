# Server Architecture (Go)

## Directory Structure

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

## Core Modules

### WebSocket Gateway (ws)

- **Role**: Handle client connections
- **Responsibilities**:
  - Accept WebSocket connections
  - Route messages to appropriate sessions
  - Handle connection lifecycle

### Session Manager (session)

- **Role**: Manage conversation sessions
- **Responsibilities**:
  - Create/destroy sessions
  - Maintain session context (with cancellation)
  - Route messages to pipeline

### Pipeline Orchestrator (pipeline)

- **Role**: Coordinate pipeline stages
- **Responsibilities**:
  - Create channels for each stage
  - Launch goroutines for each stage
  - Handle context cancellation

### Pipeline Stages

Each stage implements the Stage interface:

```go
type Stage interface {
    Process(ctx context.Context, in <-chan any, out chan<- any)
}
```

#### ASR Stage (asr)

- **Input**: AudioChunk channel
- **Output**: TextChunk channel
- **Responsibilities**:
  - Stream audio to ASR service
  - Stream back partial/final results

#### Agent Stage (agent)

- **Input**: TextChunk channel
- **Output**: TextChunk channel
- **Responsibilities**:
  - Maintain conversation memory
  - Build prompts with context
  - Call LLM for response

#### TTS Stage (tts)

- **Input**: TextChunk channel
- **Output**: TTSChunk channel
- **Responsibilities**:
  - Stream text to TTS service
  - Stream back audio chunks

## Pipeline Data Flow

```
Client WebSocket
      │
      ▼
┌─────────────┐
│   Gateway    │
└─────────────┘
      │
      ▼
┌─────────────┐
│   Session   │
│   Manager   │
└─────────────┘
      │
      ▼
┌─────────────┐
│  ASR Stage │
└─────────────┘
      │
      ▼
┌─────────────┐
│ Agent Stage │
└─────────────┘
      │
      ▼
┌─────────────┐
│  TTS Stage  │
└─────────────┘
      │
      ▼
┌─────────────┐
│   Gateway   │──▶ Client
└─────────────┘
```

## Context Propagation

Each session has its own context:

```go
ctx, cancel := context.WithCancel(context.Background())
```

When interrupt occurs:
1. cancel() is called
2. All stages receive ctx.Done()
3. Stages exit gracefully

## Development Phases

1. **Phase 1 (MVP)**: Mock pipeline (echo back audio)
2. **Phase 2**: Integrate real ASR/TTS APIs
3. **Phase 3**: Add agent with memory