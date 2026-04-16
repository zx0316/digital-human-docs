# Pipeline Architecture (Server Core)

## Concept

Streaming pipeline using Go channels with trace-based conversation management:

```
audio → asr → agent → llm → tts → output
```

## Data Types

### Core Types

```go
type AudioChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Data      []byte
    Timestamp int64
}

type TextChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Text      string
    Final     bool
}

type TTSChunk struct {
    SessionID string
    TraceID   string
    Seq       int
    Data      []byte
    End       bool
}
```

### Protocol Message Types

```go
type Message struct {
    Type      string      `json:"type"`
    SessionID string      `json:"session_id"`
    TraceID   string      `json:"trace_id"`
    Seq       int         `json:"seq"`
    Timestamp int64       `json:"timestamp"`
    Payload   interface{} `json:"payload"`
    Debug     *DebugInfo  `json:"debug,omitempty"`
}

type DebugInfo struct {
    Stage      string `json:"stage"`
    LatencyMs  int    `json:"latency_ms"`
}
```

## Pipeline Structure

```go
type Pipeline struct {
    AudioIn chan AudioChunk
    TTSOut  chan TTSChunk
    Ctx     context.Context
    Cancel  context.CancelFunc
}
```

## Channel Buffer Sizes

```go
// Recommended buffer sizes
const (
    AudioBufferSize = 50
    ASRBufferSize   = 20
    LLMBufferSize   = 20
    TTSBufferSize   = 50
)

// Creating channels with buffers
audioIn := make(chan AudioChunk, AudioBufferSize)
asrOut := make(chan TextChunk, ASRBufferSize)
llmOut := make(chan TextChunk, LLMBufferSize)
ttsOut := make(chan TTSChunk, TTSBufferSize)
```

## Stage Interface

```go
type Stage interface {
    Process(ctx context.Context, in <-chan any, out chan<- any)
}
```

## Execution Model

```go
// Create context for interrupt handling
ctx, cancel := context.WithCancel(context.Background())

// Create channels with buffers
audioIn := make(chan AudioChunk, AudioBufferSize)
asrOut := make(chan TextChunk, ASRBufferSize)
llmOut := make(chan TextChunk, LLMBufferSize)
ttsOut := make(chan TTSChunk, TTSBufferSize)

// Launch stages
go asr.Process(ctx, audioIn, asrOut)
go agent.Process(ctx, asrOut, llmOut)
go tts.Process(ctx, llmOut, ttsOut)

// Handle output
for ttsChunk := range ttsOut {
    // Send to client with backpressure control
    sendToClientWithBackpressure(ttsChunk)
}
```

## Key Design Principles

1. **Channel-based communication** - Each stage communicates via channels, no direct function calls
2. **Context propagation** - Context carries cancellation signals through the pipeline
3. **Streaming-first** - Each stage supports streaming input and output
4. **Non-blocking** - Uses goroutines for concurrent processing
5. **Trace-based** - All messages in a round share the same trace_id
6. **Backpressure control** - Every stage implements backpressure strategies

## Backpressure Strategies

### 1. Audio → ASR (Entry Point)

**Problem:** User speaks faster than server can process

**Solution: Drop old data (keep new)**

```go
func sendAudioChunk(chunk AudioChunk) {
    select {
    case audioChan <- chunk:
    default:
        // Drop oldest chunk to make space for new one
        select {
        case <-audioChan:
        default:
        }
        select {
        case audioChan <- chunk:
        default:
            // Still full, skip
        }
    }
}
```

**Principle:** Real-time > Completeness

### 2. ASR → LLM

**Problem:** ASR partial results are very frequent

**Solution: Keep only the latest partial**

```go
func asrToLLMBridge(ctx context.Context, asrOut <-chan TextChunk, llmIn chan<- TextChunk) {
    var latest TextChunk
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case t, ok := <-asrOut:
            if !ok {
                return
            }
            latest = t
        case <-ticker.C:
            // Send latest partial to LLM
            select {
            case llmIn <- latest:
            default:
                // LLM busy, skip
            }
        }
    }
}
```

**Principle:** LLM doesn't need history, only current sentence

### 3. LLM → TTS (Critical Bottleneck)

**Problem:** LLM outputs text faster than TTS can process

**Solution: Rate limiting + chunking**

```go
func llmToTTSBridge(ctx context.Context, llmOut <-chan TextChunk, ttsIn chan<- TextChunk) {
    var textBuffer strings.Builder
    const maxChunkSize = 30 // characters
    
    for {
        select {
        case <-ctx.Done():
            return
        case chunk, ok := <-llmOut:
            if !ok {
                return
            }
            
            textBuffer.WriteString(chunk.Text)
            
            // Send to TTS when buffer reaches threshold or end of text
            if textBuffer.Len() >= maxChunkSize || chunk.Final {
                ttsChunk := TextChunk{
                    SessionID: chunk.SessionID,
                    TraceID:   chunk.TraceID,
                    Seq:       chunk.Seq,
                    Text:      textBuffer.String(),
                    Final:     chunk.Final,
                }
                
                select {
                case ttsIn <- ttsChunk:
                    textBuffer.Reset()
                default:
                    // TTS busy, keep buffer but limit size
                    if textBuffer.Len() > maxChunkSize*2 {
                        textBuffer.Reset()
                    }
                }
            }
        }
    }
}
```

**Principle:** Don't let TTS queue up

### 4. TTS → Client (Often Ignored)

**Problem:** Client playback speed < TTS generation speed

**Solution: Playback queue limit**

```go
func sendToClientWithBackpressure(chunk TTSChunk) {
    // Check client queue status
    if clientQueueSize > MaxClientQueueSize {
        // Drop oldest chunks
        dropOldestClientChunks()
    }
    
    // Send to client
    select {
    case clientChan <- chunk:
    default:
        // Client busy, skip
    }
}
```

**Principle:** Prevent playback delay from increasing

## Interrupt Handling

```go
// On receiving interrupt
func handleInterrupt(traceID string) {
    // Cancel context for this trace
    pipeline.Cancel()
    
    // Clean up resources
    close(audioIn)
    close(asrOut)
    close(llmOut)
    close(ttsOut)
    
    // Prepare for new trace
    resetPipeline()
}
```

## Goroutine Leak Protection

### Proper Context Handling

```go
func (s *Stage) Process(ctx context.Context, in <-chan any, out chan<- any) {
    for {
        select {
        case <-ctx.Done():
            // Clean up and return
            return
        case item, ok := <-in:
            if !ok {
                // Channel closed
                return
            }
            // Process item
        }
    }
}
```

### Channel Close Convention

- **Producer closes output channels**
- **Consumers do not close channels**
- **Use `ok` check to detect channel closure**

## Monitoring

```go
// Log queue sizes periodically
func monitorQueues() {
    ticker := time.NewTicker(1 * time.Second)
    for {
        <-ticker.C
        log.Infof("Queues: audio=%d, asr=%d, llm=%d, tts=%d", 
            len(audioIn), len(asrOut), len(llmOut), len(ttsOut))
        
        // Alert if queues are full
        if len(ttsOut) > TTSBufferSize*0.8 {
            log.Warnf("TTS queue nearly full: %d/%d", len(ttsOut), TTSBufferSize)
        }
    }
}
```

## Client-side Backpressure

### Client Must Implement:

1. **Jitter buffer** - `queue<TTSChunk>`
2. **Playback rhythm control**
3. **Frame dropping strategy**

```cpp
// Client-side buffer control
const int MAX_BUFFER_MS = 500;

void checkBuffer() {
    if (bufferDuration() > MAX_BUFFER_MS) {
        dropOldestAudio();
    }
}

// Drop frames if delay is too high
if (currentDelay() > 300ms) {
    dropOldFrames();
}
```

## Performance Optimization

1. **Early LLM Trigger**
   - Start LLM processing with partial ASR results
   - Don't wait for final ASR

2. **LLM → TTS Parallel**
   - Start TTS as soon as LLM produces first tokens
   - Maintain small buffer between LLM and TTS

3. **Batch Processing**
   - Process multiple audio chunks in parallel
   - Use worker pools for ASR/LLM/TTS

4. **Memory Management**
   - Reuse buffers to reduce GC pressure
   - Set appropriate channel sizes

## Trace Management

```go
type TraceManager struct {
    activeTraces map[string]*Pipeline
    mutex        sync.RWMutex
}

func (tm *TraceManager) CreateTrace(sessionID, traceID string) *Pipeline {
    ctx, cancel := context.WithCancel(context.Background())
    pipeline := &Pipeline{
        AudioIn: make(chan AudioChunk, AudioBufferSize),
        TTSOut:  make(chan TTSChunk, TTSBufferSize),
        Ctx:     ctx,
        Cancel:  cancel,
    }
    
    tm.mutex.Lock()
    tm.activeTraces[traceID] = pipeline
    tm.mutex.Unlock()
    
    return pipeline
}

func (tm *TraceManager) CancelTrace(traceID string) {
    tm.mutex.Lock()
    if pipeline, exists := tm.activeTraces[traceID]; exists {
        pipeline.Cancel()
        delete(tm.activeTraces, traceID)
    }
    tm.mutex.Unlock()
}
```

## Debugging

```go
func addDebugInfo(stage string, start time.Time) *DebugInfo {
    return &DebugInfo{
        Stage:     stage,
        LatencyMs: int(time.Since(start).Milliseconds()),
    }
}

// Usage
start := time.Now()
// process...
debug := addDebugInfo("asr", start)
```

## Implementation Checklist

- [ ] Trace-based pipeline management
- [ ] Context cancellation for interrupts
- [ ] Channel-based communication with buffers
- [ ] Backpressure control in all stages
- [ ] Streaming support in all stages
- [ ] Debug information collection
- [ ] Performance monitoring
- [ ] Error handling and recovery
- [ ] Goroutine leak protection
- [ ] Client-side buffer control

## First Implementation Strategy

**Recommended approach:**
1. Start with minimal pipeline: `audio → mock → tts → client`
2. Implement all backpressure logic from the beginning
3. Add real ASR/LLM later

**Advantage:** System won't crash when you integrate real services

## System Level

With these implementations, your system will have:
- ✔ Real-time streaming
- ✔ Interrupt capability
- ✔ Backpressure control
- ✔ Controllable latency
- ✔ No crashes or deadlocks

**Essentially:** This architecture is close to production-grade voice assistant/AI call systems