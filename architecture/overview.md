# System Overview

## 高层架构

Client → WebSocket → Server → AI Pipeline → Client

---

## Server Pipeline

Audio Stream
  → ASR
  → Agent (Memory + Prompt)
  → LLM
  → TTS
  → Audio Stream + Emotion

---

## Client Pipeline

Audio Capture
  → Encode (Opus)
  → WebSocket Send

Audio Receive
  → Decode
  → Audio Playback
  → Spine Rendering (Expression)

---

## 核心特性

- Streaming response
- Interrupt support
- Timeline-based rendering
- Session-based memory
