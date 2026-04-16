# Meta Task: system_bootstrap_v1

## Goal

基于 digital-human-docs，生成完整的：

- client 工程骨架（C++）
- server 工程骨架（Python / Node 均可）
- 基础链路打通（WebSocket + audio_chunk）

---

## Context

阅读以下文档：

- architecture/overview.md
- protocol/message_schema.json
- dataflow/realtime_pipeline.md
- client/client_architecture.md
- server/server_architecture.md

---

## Output

生成两个独立工程：

### 1️⃣ client

结构：

client/
├── audio/
├── network/
├── player/
├── render/
└── main/

---

### 2️⃣ server

结构：

server/
├── ws/
├── asr/
├── agent/
├── llm/
├── tts/
└── session/

---

## Requirements

### Client

- 实现 AudioCapture（mock也可以）
- 实现 WebSocket client
- 每20ms发送 audio_chunk
- 支持 session_id

---

### Server

- WebSocket server
- 接收 audio_chunk
- 打印日志（模拟ASR）
- 返回 mock tts_chunk

---

## Protocol Constraint

严格使用：

protocol/message_schema.json

---

## Acceptance Criteria

- client 能连接 server
- client 持续发送 audio_chunk
- server 能收到并打印
- server 能返回 tts_chunk
- client 能接收并打印

---

## Non-Goals

- 不实现真实 ASR
- 不实现真实 TTS
- 不实现 UI

---

## Trae Prompt（核心）

你是一个全栈系统工程师。

请根据 digital-human-docs：

1. 生成 client（C++）和 server（Node.js 或 Python）
2. 按模块划分代码结构
3. 实现 WebSocket 双向通信
4. client 每20ms发送 audio_chunk
5. server 返回 mock tts_chunk
6. 所有代码可运行

要求：

- 不写伪代码
- 提供完整文件结构
- 每个模块有清晰职责
- 严格遵守 protocol/message_schema.json

输出：

- client 完整代码
- server 完整代码
- 运行说明
- 运行说明