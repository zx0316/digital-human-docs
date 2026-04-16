# Protocol Design (v3.0)

## Core Principles

The protocol is designed to address 4 key challenges:

- **Ordering** (seq / timestamp)
- **Streaming** (partial / final)
- **Interrupt** (cancel pipeline)
- **Alignment** (audio ↔ text ↔ expression)

## Universal Message Structure

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

## Field Explanation

| Field       | Purpose                              |
|-------------|--------------------------------------|
| session_id  | Conversation session ID              |
| trace_id    | Round ID (one question-answer pair)  |
| seq         | Sequence control (per stream)        |
| timestamp   | Client timestamp                     |
| type        | Message type                         |
| payload     | Message content                      |
| debug       | Optional debug information           |

## Key Design: trace_id

- **Critical for**: interrupt handling, multi-turn conversations, agent memory
- **Changes every round**: New question → new trace_id
- **Links all messages in a round**: audio → ASR → LLM → TTS

## Message Types

### 1. Audio Stream (Client → Server)

```json
{
  "type": "audio_chunk",
  "session_id": "s1",
  "trace_id": "t1",
  "seq": 1,
  "timestamp": 1710000000,
  "payload": {
    "data": "base64",
    "sample_rate": 16000,
    "channels": 1
  }
}
```

**Design Points:**
- 20ms per chunk
- seq increases monotonically
- trace_id = current utterance

### 2. ASR Output (Server → Client)

**Partial Result:**
```json
{
  "type": "asr_partial",
  "trace_id": "t1",
  "seq": 10,
  "payload": {
    "text": "今天天气",
    "final": false
  }
}
```

**Final Result:**
```json
{
  "type": "asr_final",
  "trace_id": "t1",
  "seq": 20,
  "payload": {
    "text": "今天天气怎么样",
    "final": true
  }
}
```

**Critical Design:**
- **Partial can overwrite**: 今 → 今天天 → 今天天气 → final sentence
- **Client must**: Use latest partial to overwrite display (not append)

### 3. LLM Output (Server → Client)

**Streaming Chunk:**
```json
{
  "type": "llm_chunk",
  "trace_id": "t1",
  "seq": 30,
  "payload": {
    "text": "今天天气很好，",
    "end": false
  }
}
```

**End:**
```json
{
  "type": "llm_end",
  "trace_id": "t1",
  "seq": 40,
  "payload": {}
}
```

**Key Point:** Start TTS while LLM is still generating

### 4. TTS Stream (Server → Client)

**Chunk:**
```json
{
  "type": "tts_chunk",
  "trace_id": "t1",
  "seq": 50,
  "payload": {
    "data": "base64",
    "end": false
  }
}
```

**End:**
```json
{
  "type": "tts_end",
  "trace_id": "t1",
  "seq": 60,
  "payload": {}
}
```

**Critical Design:**
- TTS must be slightly slower than LLM, but not wait for LLM to complete
- **Correct Flow**: LLM chunk → immediately trigger TTS → send audio
- **Avoid**: User feels "pause before speaking"

### 5. Interrupt (Client → Server)

```json
{
  "type": "interrupt",
  "session_id": "s1",
  "trace_id": "t1"
}
```

**Behavior Definition:**
- Server cancels current pipeline
- Stops: ASR, LLM, TTS
- Discards all unsent chunks

**Key Detail:**
- **New request must have new trace_id**
- **Otherwise**: Old and new data get mixed (disaster)

## Sequence Diagrams

### Normal Flow

```
Client                Server

audio_chunk ───────→
audio_chunk ───────→

                 → asr_partial
                 → asr_final

                 → llm_chunk
                 → llm_chunk

                 → tts_chunk
                 → tts_chunk
                 → tts_end
```

### Interrupt Flow

```
Client                Server

audio_chunk ───────→
                 → llm_chunk

interrupt ───────→

                 ✂ cancel pipeline
                 ✂ stop tts

(new trace)

audio_chunk ───────→
```

## Latency Model

### Key Metrics

| Stage             | Latency    |
|-------------------|------------|
| Audio Upload      | ~50ms      |
| ASR Partial       | 100~300ms  |
| LLM First Token   | 300~800ms  |
| TTS First Packet  | 500~1000ms |

### Optimization Target

**User speaks → hears response < 1s**

### Optimization Key Points

1. **Early LLM Trigger**
   - Don't wait for final ASR
   - Trigger agent with partial results

2. **LLM → TTS Parallel**
   - Start TTS as soon as LLM produces first tokens

3. **Client Early Playback**
   - Start playing as soon as first TTS chunk arrives

## Debug Capability

Add debug field to messages:

```json
"debug": {
  "stage": "tts",
  "latency_ms": 120
}
```

**Benefits:**
- ASR: 200ms
- LLM: 500ms
- TTS: 300ms
- End-to-end: 1000ms

## Implementation Guidelines

1. **Sequence Management**:
   - Each stream maintains its own seq counter
   - Client and server must handle out-of-order messages

2. **Trace Management**:
   - New trace_id for each question
   - All messages in a round share the same trace_id

3. **Interrupt Handling**:
   - Immediate cancellation
   - Cleanup of all pending operations
   - Ready for new trace

4. **Performance Monitoring**:
   - Add debug info to track latency
   - Monitor each stage's performance

5. **Error Handling**:
   - Graceful degradation on network issues
   - Resume capability after connection loss