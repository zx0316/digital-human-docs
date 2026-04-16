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

// Create channels
audioIn := make(chan AudioChunk, 100)
textOut := make(chan TextChunk, 100)
llmOut := make(chan TextChunk, 100)
ttsOut := make(chan TTSChunk, 100)

// Launch stages
go asr.Process(ctx, audioIn, textOut)
go agent.Process(ctx, textOut, llmOut)
go tts.Process(ctx, llmOut, ttsOut)

// Handle output
for ttsChunk := range ttsOut {
    // Send to client
    sendToClient(ttsChunk)
}
```

## Key Design Principles

1. **Channel-based communication** - Each stage communicates via channels, no direct function calls
2. **Context propagation** - Context carries cancellation signals through the pipeline
3. **Streaming-first** - Each stage supports streaming input and output
4. **Non-blocking** - Uses goroutines for concurrent processing
5. **Trace-based** - All messages in a round share the same trace_id

## Interrupt Handling

```go
// On receiving interrupt
func handleInterrupt(traceID string) {
    // Cancel context for this trace
    pipeline.Cancel()
    
    // Clean up resources
    close(audioIn)
    close(textOut)
    close(llmOut)
    close(ttsOut)
    
    // Prepare for new trace
    resetPipeline()
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
        AudioIn: make(chan AudioChunk, 100),
        TTSOut:  make(chan TTSChunk, 100),
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
- [ ] Channel-based communication
- [ ] Streaming support in all stages
- [ ] Debug information collection
- [ ] Performance monitoring
- [ ] Error handling and recovery