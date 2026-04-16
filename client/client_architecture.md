# Client Architecture (C++)

## Core Model

**Dual-loop threading:**

- sendLoop → send audio
- recvLoop → receive tts

## Directory Structure

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

## Key Components

### Audio Module

- **Role**: Audio capture
- **Mock first**: Use mock audio (20ms chunk) for initial development
- **Later**: Replace with real microphone input
- **Requirements**:
  - 16kHz sample rate
  - 20ms chunk size
  - Low latency capture

### Network Module

- **Role**: WebSocket communication
- **Features**:
  - Persistent WebSocket connection
  - Async send/recv
  - Auto-reconnect on disconnect
- **Protocol**: Follow message_schema.json

### Player Module

- **Role**: Audio playback
- **Features**:
  - Buffer queue for received audio chunks
  - Streaming playback (no waiting for complete audio)
  - Support immediate stop on interrupt

### Render Module

- **Role**: Avatar visualization
- **Features**:
  - Spine animation
  - Emotion mapping to facial expressions
  - Synchronized with audio timeline

### Core Module

- **Role**: Session control
- **Features**:
  - Session ID management
  - State machine (idle, recording, playing, interrupted)
  - Coordinates all other modules

## Threading Model

```
┌─────────────┐     ┌─────────────┐
│  sendLoop  │────▶│  WebSocket  │
│  (audio)   │     │   Client    │
└─────────────┘     └─────────────┘
                           │
                           ▼
┌─────────────┐     ┌─────────────┐
│   Player    │◀────│  recvLoop   │
│  (playback) │     │   (tts)     │
└─────────────┘     └─────────────┘
```

## State Machine

```
IDLE ──▶ RECORDING ──▶ PLAYING
  ▲         │              │
  │         ▼              │
  └──── INTERRUPTED ◀──────┘
```

## Development Phases

1. **Phase 1 (MVP)**: Mock audio → Mock TTS
2. **Phase 2**: Real microphone integration
3. **Phase 3**: Spine rendering integration