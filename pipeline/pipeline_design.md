# Pipeline Architecture (Server Core)

## Concept

Streaming pipeline using Go channels:

```
audio → asr → agent → llm → tts → output
```

## Data Types

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

## Pipeline Structure

```go
type Pipeline struct {
    AudioIn chan AudioChunk
    TTSOut  chan TTSChunk
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
go asr.Process(ctx, audioIn, textOut)
go agent.Process(ctx, textOut, agentOut)
go tts.Process(ctx, agentOut, ttsOut)
```

## Key Design Principles

1. **Channel-based communication** - Each stage communicates via channels, no direct function calls
2. **Context propagation** - Context carries cancellation signals through the pipeline
3. **Streaming-first** - Each stage supports streaming input and output
4. **Non-blocking** - Uses goroutines for concurrent processing