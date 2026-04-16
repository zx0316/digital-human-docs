# Task: client_audio_capture_v1

## Goal

实现客户端音频采集模块

---

## Input

- 系统音频设备（麦克风）

---

## Output

- audio_chunk (PCM 16kHz, 20ms)

---

## Dependencies

- protocol/message_schema.json
- client/client_architecture.md

---

## Requirements

- 16kHz 采样率
- 20ms chunk 大小
- 低延迟采集
- 支持开始/停止控制

---

## Non-Goals

- 不实现音频编码
- 不实现网络传输

---

## Acceptance Criteria

- 可以采集音频数据
- 每 20ms 输出一帧

---

## Trae Prompt

实现一个 C++ 音频采集模块：

- 输入：系统麦克风
- 输出：PCM 16kHz 音频块（20ms）
- 支持开始/停止控制
- 代码结构清晰（class + interface）

参考：
- protocol/message_schema.json (v3.0)
- client/client_architecture.md