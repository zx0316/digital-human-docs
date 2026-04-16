# Task: client_player_stream_v1

## Goal

实现客户端音频播放模块（带背压控制）

---

## Input

- tts_chunk 消息
- session_id
- trace_id

---

## Output

- 音频播放
- 背压控制

---

## Dependencies

- protocol/message_schema.json
- client/client_architecture.md

---

## Requirements

- 支持流式播放
- 实现 jitter buffer
- 背压控制（MAX_BUFFER_MS = 500）
- 支持 interrupt 停止播放

---

## Non-Goals

- 不实现音频解码

---

## Acceptance Criteria

- 可以播放音频流
- 缓冲区超限时自动丢帧
- interrupt 可以立即停止播放

---

## Trae Prompt

实现一个 C++ 音频播放模块：

- 输入：tts_chunk 音频流
- 输出：音频播放
- 实现 jitter buffer 和背压控制
- 支持立即停止（interrupt）
- 代码结构清晰（class + interface）

参考：
- protocol/message_schema.json (v3.0)
- client/client_architecture.md