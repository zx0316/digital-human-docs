# Task: client_ws_stream_v1

## Goal

实现客户端 WebSocket 音频流发送模块

---

## Input

- audio_chunk (PCM 16k)
- session_id

---

## Output

发送 message:

{
  "type": "audio_chunk",
  "session_id": "...",
  "timestamp": "...",
  "pcm": "base64"
}

---

## Dependencies

- protocol/message_schema.json
- client/audio_pipeline.md

---

## Requirements

- 使用 WebSocket
- 支持断线重连
- 支持 streaming（20ms chunk）

---

## Non-Goals

- 不实现 UI
- 不实现播放

---

## Acceptance Criteria

- 可以持续发送音频流
- 服务端可正确解析

---

## Trae Prompt

实现一个 C++ WebSocket 客户端模块：

- 输入：PCM音频流（20ms一帧）
- 输出：按协议发送 audio_chunk
- 支持自动重连、
- 代码结构清晰（class + interface）

参考：
- protocol/message_schema.json