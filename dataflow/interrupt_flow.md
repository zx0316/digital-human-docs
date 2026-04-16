# Interrupt Flow

## 触发

Client detects:
- VAD speech start OR user button

---

## 流程

1. Client stops audio playback
2. Client sends interrupt event
3. Server cancels:
   - LLM generation
   - TTS generation
4. Server resets session state

---

## 原则

Interrupt must be:
- < 100ms perceived latency
- idempotent
