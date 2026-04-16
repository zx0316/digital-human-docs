# Realtime Pipeline

## 主链路

1. User speaks
2. Client captures audio (20ms chunk)
3. Send via WebSocket
4. Server ASR streaming
5. Partial text → Agent
6. Memory retrieval
7. LLM streaming output
8. TTS streaming synthesis
9. Return audio chunks
10. Client plays + renders Spine

---

## 关键点

- 全链路 streaming
- audio_chunk为最小单位
- LLM & TTS均允许 partial output
