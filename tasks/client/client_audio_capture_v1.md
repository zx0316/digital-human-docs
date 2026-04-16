# Task: client_audio_capture_v1

## Goal

实现客户端音频采集模块

---

## Input

- 麦克风输入

---

## Output

- PCM 音频数据（16kHz，20ms chunk）

---

## Dependencies

- client/audio_pipeline.md

---

## Requirements

- 支持 16kHz 采样率
- 支持 20ms 音频块
- 低延迟采集

---

## Non-Goals

- 不实现音频处理
- 不实现网络传输

---

## Acceptance Criteria

- 可以稳定采集音频
- 输出格式符合要求

---

## Trae Prompt

实现一个 C++ 音频采集模块：

- 输入：麦克风
- 输出：PCM 音频流（16kHz，20ms 一帧）
- 支持跨平台（Windows/macOS/Linux）
- 代码结构清晰（class + interface）

参考：
- client/audio_pipeline.md