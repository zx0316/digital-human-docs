# Interrupt Design (Critical)

## Overview

Interrupt capability is essential for natural conversation flow. Users must be able to interrupt the digital human at any time.

## Mechanism

Each session owns a context. Interrupt triggers cancel():

```go
ctx, cancel := context.WithCancel(context.Background())
```

## Stage Requirement

Each pipeline stage must respect context cancellation:

```go
func (s *Stage) Process(ctx context.Context, in <-chan any, out chan<- any) {
    for {
        select {
        case <-ctx.Done():
            return
        case item := <-in:
            // process item
        }
    }
}
```

## Interrupt Flow

1. Client detects: VAD speech start OR user button
2. Client sends interrupt event via WebSocket
3. Server receives interrupt, calls cancel() on session context
4. All pipeline stages receive ctx.Done() and exit gracefully
5. Session state is reset

## Requirements

- **Latency**: Interrupt must be perceived < 100ms
- **Idempotency**: Multiple interrupts must not cause issues
- **Graceful exit**: Stages must clean up resources properly

## Implementation Checklist

- [ ] Session manager supports context cancellation
- [ ] Each pipeline stage checks ctx.Done() in select
- [ ] Audio playback can be stopped immediately
- [ ] TTS generation can be cancelled
- [ ] WebSocket can send interrupt message