# Task: client_ws_stream_v1

## Goal

实现客户端 WebSocket 音频流发送模块

---

## Input

- audio_chunk (PCM 16k)
- session_id
- trace_id

---

## Output

发送 message:

```json
{
  "type": "audio_chunk",
  "session_id": "...",
  "trace_id": "...",
  "seq": 1,
  "timestamp": 123456789,
  "payload": {
    "data": "base64",
    "sample_rate": 16000,
    "channels": 1
  }
}
```

---

## Dependencies

- protocol/message_schema.json
- client/client_architecture.md

---

## Requirements

- 使用 WebSocket
- 支持断线重连
- 支持 streaming（20ms chunk）
- 遵循 v3.0 协议格式

---

## Non-Goals

- 不实现 UI
- 不实现播放

---

## Acceptance Criteria

- 可以持续发送音频流
- 服务端可正确解析
- 支持 session_id 和 trace_id

---

## Trae Prompt

实现一个 C++ WebSocket 客户端模块：

- 输入：PCM音频流（20ms一帧）
- 输出：按协议发送 audio_chunk（v3.0格式）
- 支持自动重连
- 代码结构清晰（class + interface）

参考：
- protocol/message_schema.json (v3.0)
- client/client_architecture.md