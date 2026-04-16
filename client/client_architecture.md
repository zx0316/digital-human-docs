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

### Player Module (With Backpressure Control)

- **Role**: Audio playback with backpressure
- **Features**:
  - **Jitter buffer** - `queue<TTSChunk>`
  - **Streaming playback** (no waiting for complete audio)
  - **Backpressure control** - limit buffer size
  - **Frame dropping** - drop old frames if delay is too high
  - **Support immediate stop on interrupt**

#### Key Constants

```cpp
// Recommended values
const int MAX_BUFFER_MS = 500;    // Maximum buffer duration
const int DROP_THRESHOLD_MS = 300; // Drop frames if delay exceeds this
const int CHUNK_SIZE_MS = 20;      // Audio chunk size
```

#### Buffer Control Logic

```cpp
class AudioPlayer {
private:
    std::queue<TTSChunk> buffer;
    int bufferDurationMs = 0;
    bool isPlaying = false;

public:
    void addChunk(const TTSChunk& chunk) {
        // Calculate chunk duration
        int chunkDuration = calculateDuration(chunk);
        bufferDurationMs += chunkDuration;
        
        // Check buffer size
        if (bufferDurationMs > MAX_BUFFER_MS) {
            dropOldestChunks();
        }
        
        // Add to buffer
        buffer.push(chunk);
    }
    
    void dropOldestChunks() {
        while (bufferDurationMs > MAX_BUFFER_MS && !buffer.empty()) {
            TTSChunk oldest = buffer.front();
            bufferDurationMs -= calculateDuration(oldest);
            buffer.pop();
        }
    }
    
    void checkDelay() {
        int currentDelay = calculateCurrentDelay();
        if (currentDelay > DROP_THRESHOLD_MS) {
            dropOldFrames();
        }
    }
};
```

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
  - Trace ID management (new for each round)
  - State machine (idle, recording, playing, interrupted)
  - Coordinates all other modules
  - **Backpressure coordination**

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

## Backpressure Control

### Client-side Strategies

1. **Buffer size limitation**
   - Maximum buffer duration: 500ms
   - Drop oldest chunks when buffer is full

2. **Delay-based dropping**
   - Drop frames if current delay > 300ms
   - Prioritize real-time over completeness

3. **Playback rhythm control**
   - Adjust playback speed slightly to catch up
   - Maintain natural speech rhythm

4. **Network backpressure**
   - Signal server if client is overloaded
   - Pause receiving when buffer is full

### Implementation Example

```cpp
// In recvLoop
void recvLoop() {
    while (running) {
        // Receive message from server
        Message msg = wsClient.receive();
        
        if (msg.type == "tts_chunk") {
            TTSChunk chunk = parseTTSChunk(msg);
            
            // Add to player with backpressure
            player.addChunk(chunk);
            
            // Check delay and drop if needed
            player.checkDelay();
        }
        else if (msg.type == "interrupt") {
            // Handle interrupt
            player.stop();
            buffer.clear();
        }
    }
}
```

## Trace Management

```cpp
class SessionManager {
private:
    std::string sessionID;
    std::string currentTraceID;
    
public:
    void startNewRound() {
        // Generate new trace ID for each round
        currentTraceID = generateUUID();
    }
    
    std::string getCurrentTraceID() {
        return currentTraceID;
    }
    
    void handleInterrupt() {
        // Cancel current processing
        // Prepare for new trace
        startNewRound();
    }
};
```

## Development Phases

1. **Phase 1 (MVP)**: Mock audio → Mock TTS with backpressure
2. **Phase 2**: Real microphone integration
3. **Phase 3**: Spine rendering integration
4. **Phase 4**: Advanced backpressure optimization

## Key Performance Metrics

- **Buffer duration**: Keep < 500ms
- **Playback delay**: Keep < 300ms
- **Interrupt response**: < 100ms
- **CPU usage**: < 10% on mobile devices

## Implementation Checklist

- [ ] Audio capture with 20ms chunks
- [ ] WebSocket client with auto-reconnect
- [ ] Jitter buffer for TTS chunks
- [ ] Backpressure control with frame dropping
- [ ] Trace ID management
- [ ] Interrupt handling
- [ ] Spine rendering integration
- [ ] Performance monitoring

## Testing Strategy

1. **Load testing**: Simulate high-speed audio input
2. **Latency testing**: Measure end-to-end delay
3. **Interrupt testing**: Test interrupt response time
4. **Stability testing**: Run for extended periods
5. **Network testing**: Test with poor network conditions

## Future Enhancements

- **Adaptive buffer**: Dynamically adjust buffer size based on network conditions
- **Audio quality**: Implement dynamic bitrate adjustment
- **Multi-modal**: Add video streaming support
- **Edge processing**: Move some processing to client for lower latency